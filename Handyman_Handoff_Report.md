# Handyman: Handoff Report (v0.3 prototype)

## 1. Where we are
- `handyman.html` is a working front-end prototype with sign-in/sign-up (email or phone + password) and three role-based areas: Customer, Handyman and Admin (superuser control board). Data lives in the browser only (localStorage) and Paystack is simulated.
- v0.3 adds: commission and fee-payer rules per category, transport hailing for Ghana with admin-managed areas, invoices with platform-fee and tax lines, account verification, password reset and a 5-attempt lockout, CSV/Excel export, date filters and handyman commission statements. Notifications are deliberately not included (see section 9).
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
| Admin settings: invoice details, Paystack fee %, tax %, activity log | Done |
| Commission % and who pays the Paystack processing fee, set per category (rates saved on each order) | Done |
| Transport hailing: ride booking (taxi, motorbike, tricycle), fare by distance, driver accepts, tracked like any order; admin adds/edits/turns off areas | Done (fares typed by distance, no maps or live GPS) |
| Invoices show a separate platform-fee line and the tax rate | Done |
| Email/phone verification, password reset, lock after 5 wrong attempts | Done (demo: the code is shown on screen; nothing is sent) |
| CSV and Excel export, date filters, handyman commission statements | Done |
| Mobile-friendly, light/dark | Done |
| Real code sending, server-side authentication, secure database, real Paystack, maps and live GPS | **Not built** (sections 3 to 5) |

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

-- v0.3: per-category fees, snapshot of rates on each order, transport areas
alter table categories add column fee_payer text not null default 'platform' check (fee_payer in ('platform','handyman','customer'));
alter table app_settings add column processing_pct numeric(4,2) not null default 1.95;  -- check the Paystack dashboard
alter table orders
  add column category_id int references categories(id),
  add column commission_pct numeric(4,2), add column fee_payer text, add column processing_pct numeric(4,2),
  add column processing_fee numeric(10,2) default 0,
  add column kind text not null default 'job' check (kind in ('job','ride')),
  add column area_id int, add column pickup text, add column dropoff text, add column distance_km numeric(6,1);
create table areas (
  id serial primary key, name text unique not null, region text,
  base_fare numeric(8,2) not null, per_km numeric(8,2) not null, min_fare numeric(8,2) not null default 0,
  active boolean not null default true
);
alter table orders add foreign key (area_id) references areas(id);
alter table providers add column area_id int references areas(id);
create table login_attempts (identifier text primary key, fails int not null default 0, locked_until timestamptz);
alter table areas enable row level security;
create policy "areas readable" on areas for select using (active or is_admin());
create policy "areas admin write" on areas for all using (is_admin()) with check (is_admin());
alter table login_attempts enable row level security;   -- no policies: only server functions touch it
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

**v0.3 security rules** (prototype behaviour, to be enforced on the server):
- New accounts must verify the 6-digit code (valid 10 minutes) sent by SMS or email before first sign-in. Supabase Auth does this natively; for SMS use a provider that covers Ghana.
- Password reset sends a code to the registered email or phone. Passwords need 8 characters or more.
- After 5 wrong passwords an identifier is locked for 15 minutes (`login_attempts`, checked by an Edge Function before sign-in). The prototype also locks unknown identifiers; keep that to stop guessing of which accounts exist.
- Never reveal whether an email or phone is registered in the sign-in error message.

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

### 4d. Commission and fee model (per category)
- Each category has a commission % and a "processing fee payer" (platform, handyman or customer). The Paystack processing % lives in settings.
- When an order is created the server copies these three values onto the order, so later changes only affect new orders.
- Per order: commission = amount x commission %; processing fee = amount x processing %. If the customer pays the fee it is added to their charge; if the handyman pays it comes off their payout; if the platform pays it comes off the commission. Net revenue = commission minus any fee the platform absorbs.
- Verify the real Paystack Ghana fee structure (cards vs mobile money may differ) before choosing defaults.

### 4e. Transport hailing (Ghana)
- Areas (city and region, base fare, per-km rate, minimum fare) are rows in `areas`; admins add, edit and turn them off without code changes. The starting values in the prototype (Accra, Tema, Kumasi, Takoradi, Cape Coast, Tamale) are placeholders, not researched fares.
- Vehicle types: taxi, motorbike (okada), tricycle, with a fare multiplier. Fare = max(minimum, (base + per-km x distance) x multiplier).
- Flow: customer requests and pays, the ride appears to approved Transport handymen in the same area, the first to accept gets it, then it follows the normal order stages and rating.
- Not in the prototype and needed for a real service: live GPS, route and distance from a maps provider (Google Maps or Mapbox), driver availability and matching, cancellations and no-show rules, safety features (trip sharing, SOS), driver vehicle and licence checks, and the transport regulations for Ghana (check with the relevant authorities before launch).

### 4f. Exports and statements
- CSV and Excel exports: transactions, invoices, users, and per-handyman commission statements, all filtered by the selected date range. In production, generate large exports on the server.
- Cells starting with =, +, - or @ are escaped to prevent spreadsheet formula injection.
- A statement lists each completed job with amount, rate, commission, processing fee charged to the handyman and payout, with totals.

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
2. **Phase 2**: live Paystack, payouts, in-app chat, photos on jobs, and notifications only if you decide to include them (section 9).
3. **Phase 3**: transport upgrades (maps, live GPS, driver matching, safety features), scheduled jobs, PWA install, mobile apps, more areas.

## 9. Decisions still needed from you
1. **Escrow release:** pay the handyman only when the customer confirms, or automatically after N hours? (Rides usually release at trip end.)
2. **Refunds:** refund policy and dispute time limits, including cancellation fees for rides.
3. **Launch:** first city and first three service categories (transport is already built for the six starter areas).
4. **Real values:** default commission % per category, who pays the processing fee in each, the tax rate, and the base fare, per-km rate and minimum fare for each area. The prototype uses placeholders (10% commission, platform pays fees, 0% tax).
5. **Notifications:** you wrote "no notifications". I left them out and show verification codes on screen in the demo. A live app cannot send codes without SMS or email sending, so please confirm whether that sending is allowed while other notifications (order updates) stay out.
6. **Platform fee on invoices:** I show it as an included line (taken from the handyman's payout). Should it instead be added on top of the customer's bill?
7. **Transport rules:** do drivers need extra documents (vehicle registration, driver's licence, insurance) before approval?

## 10. Known gaps in this prototype
- Admin checks happen only in the browser. In production every admin action must be enforced on the server (RLS and `is_admin()`).
- Login, verification codes, password reset and the 5-attempt lockout run in the browser with a toy password hash. Real sending of codes by SMS or email and server-side enforcement are needed.
- Certificates and vehicle documents chosen at sign-up are not uploaded or stored.
- Transport has no maps, live GPS, driver matching, cancellations, tipping or safety features. Distance is typed in by the customer.
- Excel and CSV downloads need the viewer to approve a save prompt, and depend on the hosting page allowing downloads.
- No real database (Supabase) and no live Paystack or payouts yet.
- No notifications (by your instruction), no in-app chat, no dispute evidence uploads.

## 11. Notes for the next developer or Claude session
- Single source of truth for the prototype is `handyman.html`. `S` is the whole state (users, pros, orders, comps, reviews, cats, areas, cfg, log, fails, codes, sess). Views are functions in the `V` object (customer: `find`, `ride`, `orders`, `merit`; handyman: `jobs`, `rides`, `earn`, `stmt`; admin: `ov`, `users`, `approve`, `disp`, `fin`, `stmt`, `inv`, `areas`, `cfg`). All clicks go through one handler using `data-*` attributes.
- Money rules live in one function, `calc(order)`. Orders store `rate`, `payer` and `ps` (processing %) at creation; always read those, never the current settings, for past orders.
- Exports: `rowsFor(kind)` builds the table, `exp(kind, format)` saves it using the `downloads` capability; the page is published with that capability declared. Excel uses SheetJS 0.18.5 from cdnjs.
- Auth state machine: `authMode` is `in`, `up`, `verify`, `forgot` or `reset`; `sendCode()` shows the demo code on screen.
- Ride flow: `ride()` and the `rd_go` / `rq` handlers create an order with `kind` ride and no handyman; `take` assigns the first eligible Transport driver in the same area.
