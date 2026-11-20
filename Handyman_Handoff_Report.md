# Handyman: Handoff Report (v0.2 prototype)

## 1. Where we are
- `handyman.html` is a working front-end prototype with sign-in/sign-up (email or phone + password) and three role-based areas: Customer, Handyman and Admin (superuser control board). Data lives in the browser only (localStorage) and Paystack is simulated.
- Demo logins (password `demo123`): `ama@handyman.app` (customer), `kofi@handyman.app` (handyman). Admin: `admin@handyman.app` / `admin123`. Remove these before any real launch.
- It proves the flows and screens. It is **not yet safe for real money or real ID documents**: that needs the backend in section 3.

## 2. What the prototype covers
| Requirement | Status |
|---|---|
| Search/catalog by service, area, rating, tier | Done (prototype) |
| Handyman profile, job history, ratings visible | Done |
| Booking, payment step, order tracking (Paid, Accepted, In progress, Completed) | Done (payment simulated) |
| Ratings and written reviews | Done |
| Complaints and refunds (admin decision) | Done (refund simulated) |
| Handyman registration with ID/certificates, admin approval before going live | Done (file upload not stored yet) |
| Merit system (points and tiers, both sides) | Done (rules in section 6) |
| 10% commission reporting | Done |
| Login and sign-up with email or phone, role-based access | Done (demo only: passwords use a toy hash in the browser) |
| Admin control board: overview metrics, users, approvals, complaints, finance ledger | Done (prototype) |
| Invoicing: numbered invoice per completed job, business details, tax, terms, print/PDF | Done (prototype) |
| Admin settings: commission rate, invoice details, activity log | Done |
| Mobile-friendly, light/dark | Done |
| Real authentication (phone/email codes, password reset), secure database, real Paystack, notifications, maps | **Not built** (sections 3 to 5) |
| Export (CSV/Excel), date-range filters, handyman commission statements | **Not built** (section 10) |

## 3. Recommended production stack
- **Supabase** (hosted PostgreSQL + Auth + Storage + Row Level Security). Fast to launch, scales well, and lets us keep security rules inside the database.
- **Server functions** (Supabase Edge Functions) for everything involving money. The Paystack **secret key never goes in the web page**.
- Front end: keep this HTML design, then move to React/Next.js when the team grows.
- Hosting: Netlify, Vercel or GitHub Pages for the front end (matches the Mega Tag website plan).

## 4. Secure database (PostgreSQL, run in Supabase SQL editor)
```sql
create extension if not exists pgcrypto;
create type user_role as enum ('customer','handyman','admin');
create type pro_status as enum ('pending','approved','rejected','suspended');
create type order_status as enum ('pending_payment','paid','accepted','in_progress','completed','disputed','refunded','cancelled');

create table profiles (
  id uuid primary key references auth.users on delete cascade,
  role user_role not null default 'customer',
  full_name text not null, phone text, city text,
  merit_points int not null default 0,
  created_at timestamptz default now()
);
alter table profiles add column active boolean not null default true;   -- admin can suspend
alter table profiles add column email text unique, add column phone text unique;
create table categories (id serial primary key, name text unique not null, commission_pct numeric(4,2) not null default 10);

create table providers (
  id uuid primary key references profiles(id) on delete cascade,
  category_id int references categories(id),
  bio text, base_price numeric(10,2) check (base_price > 0),
  status pro_status not null default 'pending',
  approved_by uuid references profiles(id), approved_at timestamptz,
  rating_sum int not null default 0, rating_count int not null default 0,
  jobs_completed int not null default 0,
  payout_subaccount text  -- Paystack subaccount/recipient code
);
-- ID and certificates: private storage bucket "kyc", never public
create table provider_documents (
  id uuid primary key default gen_random_uuid(),
  provider_id uuid references providers(id) on delete cascade,
  doc_type text not null,              -- ghana_card, licence, certificate
  id_number_enc bytea,                 -- encrypted (pgcrypto), never plain text
  storage_path text not null, verified boolean default false,
  uploaded_at timestamptz default now()
);
create table catalog_items (
  id uuid primary key default gen_random_uuid(),
  provider_id uuid references providers(id) on delete cascade,
  title text not null, price numeric(10,2) not null, active boolean default true
);
create table orders (
  id uuid primary key default gen_random_uuid(),
  ref text unique not null default 'H'||floor(random()*1e8)::text,
  customer_id uuid references profiles(id), provider_id uuid references providers(id),
  note text, amount numeric(10,2) not null check (amount > 0),
  commission numeric(10,2) not null default 0,
  status order_status not null default 'pending_payment',
  paystack_ref text unique, created_at timestamptz default now()
);
create table order_events (  -- tracking history, append-only
  id bigserial primary key, order_id uuid references orders(id) on delete cascade,
  status order_status not null, actor uuid, at timestamptz default now()
);
create table payments (       -- written by server only
  id uuid primary key default gen_random_uuid(),
  order_id uuid references orders(id), paystack_ref text unique not null,
  amount numeric(10,2) not null, currency text default 'GHS',
  status text not null, raw jsonb, created_at timestamptz default now()
);
create table reviews (
  id uuid primary key default gen_random_uuid(),
  order_id uuid unique references orders(id),   -- one review per completed order
  customer_id uuid references profiles(id), provider_id uuid references providers(id),
  stars int check (stars between 1 and 5), body text, created_at timestamptz default now()
);
create table complaints (
  id uuid primary key default gen_random_uuid(),
  order_id uuid references orders(id), raised_by uuid references profiles(id),
  body text not null, status text default 'open', resolution text,
  resolved_by uuid references profiles(id), created_at timestamptz default now()
);
create table refunds (
  id uuid primary key default gen_random_uuid(),
  complaint_id uuid references complaints(id), order_id uuid references orders(id),
  amount numeric(10,2) not null, paystack_refund_ref text, created_at timestamptz default now()
);
create table audit_log (id bigserial primary key, actor uuid, action text, target text, at timestamptz default now());
-- Admin settings (single row) and invoices
create table app_settings (
  id int primary key default 1 check (id = 1),
  commission_pct numeric(4,2) not null default 10,
  biz_name text not null default 'Handyman', biz_address text, biz_phone text, biz_email text, tax_id text,
  tax_pct numeric(4,2) not null default 0,      -- tax already included in prices
  invoice_prefix text not null default 'HM', payment_terms text, invoice_footer text
);
insert into app_settings default values;
create table invoices (
  id uuid primary key default gen_random_uuid(),
  number text unique not null,                  -- e.g. HM-2026-00041, from a sequence
  order_id uuid unique references orders(id),
  customer_id uuid references profiles(id),
  subtotal numeric(10,2) not null, tax numeric(10,2) not null default 0, total numeric(10,2) not null,
  settings_snapshot jsonb not null,             -- business details as at issue date, so old invoices never change
  issued_at timestamptz default now()
);
create sequence invoice_seq;
create index on orders (customer_id), (provider_id), (status);
create index on providers (category_id, status);

-- ROW LEVEL SECURITY: nothing is readable unless a policy allows it
create or replace function is_admin() returns boolean language sql stable security definer as
$$ select exists(select 1 from profiles where id = auth.uid() and role = 'admin') $$;

alter table profiles enable row level security;
alter table providers enable row level security;
alter table provider_documents enable row level security;
alter table orders enable row level security;
alter table payments enable row level security;
alter table reviews enable row level security;
alter table complaints enable row level security;
alter table refunds enable row level security;
alter table invoices enable row level security;
alter table app_settings enable row level security;
alter table audit_log enable row level security;

create policy "own profile" on profiles for select using (id = auth.uid() or is_admin());
create policy "public approved providers" on providers for select using (status = 'approved' or id = auth.uid() or is_admin());
create policy "provider edits own" on providers for update using (id = auth.uid()) with check (id = auth.uid() and status = 'pending' or is_admin());
create policy "docs owner or admin" on provider_documents for all using (provider_id = auth.uid() or is_admin());
create policy "order parties" on orders for select using (customer_id = auth.uid() or provider_id = auth.uid() or is_admin());
create policy "customer creates order" on orders for insert with check (customer_id = auth.uid() and status = 'pending_payment');
create policy "payments admin only" on payments for select using (is_admin());   -- writes: service role only
create policy "reviews public" on reviews for select using (true);
create policy "review own completed order" on reviews for insert with check (
  customer_id = auth.uid() and exists(select 1 from orders o where o.id = order_id and o.customer_id = auth.uid() and o.status = 'completed'));
create policy "complaint parties" on complaints for select using (raised_by = auth.uid() or is_admin());
create policy "complaint create" on complaints for insert with check (raised_by = auth.uid());
create policy "admin refunds" on refunds for all using (is_admin());
create policy "invoice parties" on invoices for select using (customer_id = auth.uid() or is_admin());
create policy "settings admin" on app_settings for all using (is_admin()) with check (is_admin());
create policy "audit admin read" on audit_log for select using (is_admin());
create policy "admin reads all profiles" on profiles for update using (is_admin());   -- suspend/reactivate
```
Status changes on `orders` (paid, completed, refunded) must happen **only** through server functions, never directly from the browser.

### 4a. Login and roles
- Use **Supabase Auth**: email + password, phone + SMS OTP (Ghana SMS provider such as Twilio or an alternative), and password reset. Never store passwords yourself.
- On sign-up a database trigger creates the `profiles` row (role `customer` or `handyman`). The role `admin` is **never** self-selectable: create admin users manually in the SQL editor, and turn on two-factor authentication for them.
- Handyman sign-up creates a `providers` row with status `pending`; they stay hidden from search until an admin approves.
- Suspended users (`profiles.active = false`) are blocked by policy and by the server functions.

### 4b. Admin control board
Metrics in the prototype and where they come from in production:
- Commission, job value, refunds, average order, escrow held: `orders` and `payments`.
- Customers, handymen live/total, new users, awaiting approval: `profiles`, `providers`.
- Open complaints, ratings: `complaints`, `reviews`.
Build these as SQL **views** or functions callable only by admins, so the heavy maths stays in the database. Every admin action (approve, suspend, refund, settings change) writes to `audit_log`. The commission rate must be copied onto each order when it is paid so later rate changes do not rewrite history.

### 4c. Invoicing
- An invoice is issued by a server function when an order becomes `completed`: number from `invoice_seq` with the prefix and year from settings, tax split from the price (prices include tax), and a snapshot of the business details saved on the invoice.
- Show it as printable HTML first; later generate a PDF on the server and email or SMS the link to the customer.
- Check Ghana tax rules (VAT and levies, GRA e-invoicing if it applies to you) with your accountant before setting the tax rate.

## 5. Paystack integration plan (for now; keep it behind one module so it can be swapped)
1. Customer places order, server creates it as `pending_payment` and calls Paystack **Initialize Transaction** (amount in pesewas, currency GHS, order ref in metadata).
2. Browser opens Paystack Popup with the returned access code.
3. Paystack calls our **webhook**. The server checks the `x-paystack-signature` header (HMAC SHA512 with the secret key), then calls **Verify Transaction**, confirms amount and ref match, and only then sets the order to `paid`.
4. Money sits with us as escrow. When the job is `completed` (customer confirms or auto-confirms after a set window), the server computes `commission = amount x commission_pct` and pays the rest to the handyman via **Transfer Recipients + Transfers** (mobile money or bank). Alternative: Paystack **Subaccounts** with split payments.
5. Refunds go through Paystack **Refund API**, triggered only by an admin decision.
6. Wrap all of it in a `PaymentProvider` interface (`initialize`, `verify`, `payout`, `refund`) so switching providers later touches one file.
7. Test with Paystack test keys first. Confirm Ghana mobile money and card support and fee rates on your Paystack dashboard before pricing the commission.

## 6. Merit system (current rules, easy to tune)
- Handyman: +10 per completed job, +5 per star above 3 on each review. Customer: +10 per completed job, +5 per review.
- Tiers: Bronze, Silver (150), Gold (400), Platinum (1000). Higher tier ranks higher in search.
- Suggested additions: penalties for cancellations and upheld complaints, monthly decay, badges for response time.

## 7. Security checklist before launch
- Secret keys only in server environment variables; rotate if ever exposed.
- Enforce RLS on every table (above); test each policy with a non-admin user.
- Store ID documents in a **private** bucket with signed, short-lived URLs; encrypt ID numbers; restrict who can open them.
- HTTPS everywhere, email/phone verification, rate limiting on login and booking, CAPTCHA on registration.
- Data protection: register with Ghana's Data Protection Commission and publish a privacy policy covering ID documents and location data.
- Daily database backups and a tested restore. Keep the audit log.
- Legal pages: Terms (commission, refund policy, dispute rules) and handyman agreement. Get a lawyer to review, and consider insurance for transport services.

## 8. Roadmap (updated)
1. **Phase 1 (4 to 6 weeks)**: Supabase project and the schema above, real auth (phone OTP + email), connect this UI to the database, KYC upload to the private bucket, admin approval, admin metrics as views, invoice function, Paystack test mode end to end.
2. **Phase 2**: live Paystack, payouts, notifications (SMS/WhatsApp/push), in-app chat, photos on jobs.
3. **Phase 3**: transport hailing (live location, fare estimate, driver matching), scheduled jobs, PWA install, mobile apps, multi-city.

## 9. Decisions needed from you
1. Commission: flat 10% or per category? Who pays Paystack fees, you or the handyman?
2. Escrow release: customer confirmation only, or auto-release after N hours?
3. Refund policy and dispute time limits.
4. Launch city and first 3 categories (focus makes launch easier than "everything at once").
5. Transport hailing is a separate, heavier product (live tracking, regulation). Treat it as Phase 3.

## 10. Known gaps in this prototype
- Admin checks happen only in the browser. In production every admin action must be enforced on the server (RLS and `is_admin()`).
- No password reset, no email/phone verification, no rate limiting.
- Certificates chosen at sign-up are not uploaded or stored.
- Changing the commission rate recalculates past figures; fix by storing the rate on each order.
- No CSV/Excel export, date-range filters, handyman commission statements, or notifications.

## 11. Notes for the next developer or Claude session
- Single source of truth for the prototype is `handyman.html`. Constants `CATS`, `CUR` are at the top of the script; commission is `S.cfg.fee` (copied to `FEE`); tier thresholds are in `tier()`.
- Structure: `S` is the whole state (users, pros, orders, comps, reviews, cfg, log, sess). Views are functions in the `V` object (customer: `find`, `orders`, `merit`; handyman: `jobs`, `earn`; admin: `ov`, `users`, `approve`, `disp`, `fin`, `inv`, `cfg`). All clicks go through one handler using `data-*` attributes.
- `rev()` and `reg()` are leftover functions from v0.1 and can be deleted.
- To go live, replace the `S` state object and `save()` with Supabase calls and replace the "Complete test payment" button with the Paystack Popup flow in section 5.
