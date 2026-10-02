# Security hardening of a production booking & subscription SaaS

*Written by Robin Hmaidan. The product, esthéJob Cowork, was designed and built by the client; I joined on 24 September 2026 as security & integration lead. Everything below describes issues that have been fixed. Snippets are generic illustrations I wrote for this document, not code from the product.*

## Starting point

The app was already live with paying customers: space owners managing rooms, bookings, freelance tenants, invoices and online payments. It is a Next.js App Router application on Vercel, backed by Supabase (Postgres, Auth, Storage) and Stripe Connect, with a long SQL migration history and a solid unit-test suite.

My brief had two parts:

1. **Audit the backend and fix what matters**, without changing product behaviour.
2. **Integrate it with esthéJob**, the marketplace I lead, so both products share one database and one login.

## Approach

### 1. Audit

I reviewed the system from the attacker's and the accountant's point of view at the same time, because in a product that handles money the worst bugs are often correctness bugs with security consequences.

What I looked at:

- **Identity:** sign-up, sign-in, password reset, email confirmation, email change, sign-out, and what each flow trusts.
- **Authorization in the database:** RLS on every table, role privileges, `SECURITY DEFINER` functions, triggers, storage policies.
- **Authorization in the app:** every Server Action and route handler, and which database client it uses.
- **Bearer links:** URLs that grant access without a login (tenant portal, payment links, calendar feeds).
- **Money paths:** Stripe webhooks, subscriptions, refunds, promo codes, invoice numbering and VAT.
- **Public surface:** unauthenticated forms, headers, client bundle, third-party scripts.
- **Operations:** secrets handling, logs, scripts in the repository, monitoring.

### 2. Prioritized findings

I wrote a report with each finding rated by impact and likelihood, a short reproduction description, the proposed fix and how to verify it. The report was the contract: fixes were done in priority order, one concern per commit, with the finding referenced in the commit message.

### 3. Fixes

Small, reviewable commits. Database changes as numbered migrations, each in its own transaction with a lock timeout so a migration can never stall production traffic.

### 4. Verification

A fix is not done until it is verified:

- **Unit tests** around the money logic and webhooks, written *before* touching that code, so behaviour changes would show up.
- **Local migration replay:** every migration replayed from zero on a local Postgres, followed by SQL checks that the intended policies, locks and grants exist.
- **Before/after privilege diffs:** for grant changes, a catalog snapshot before and after, with the diff matching exactly the privileges that were meant to be removed.
- **A production smoke script** that performs real writes against production, checks invariants and column locks, and cleans up after itself.
- **Live browser checks** of public pages and of invalid or expired links, plus CSP violation logs.

## Categories of fixes

### Database privileges and Row Level Security

- Reviewed RLS on every table. Tables that should only be touched by the server have RLS enabled with no policy, which means deny by default.
- **Tightened role privileges:** removed table privileges that the public and authenticated roles never need (for example `TRUNCATE`, `TRIGGER`, `REFERENCES`), verified with a before/after catalog diff.
- **Column locks:** some fields must never be changed by the user who owns the row (for example the contact email on a request, or fields only the server should manage). Triggers reject those updates unless they come from the server role.
- **Pinned search paths** on every `SECURITY DEFINER` function, and execute rights revoked from `public` and granted only where needed.
- **Storage policies** reviewed so that private documents are readable only by the people they belong to.

### Race conditions in promo codes and refunds

Promo codes can be limited ("first 10 customers"). The original flow checked availability and then wrote the reservation as two separate steps, so two simultaneous checkouts could both pass the check. Refunds and expirations had similar windows where a place could be lost or counted twice.

I moved reservation, confirmation, release and refund handling into **atomic PostgreSQL functions** that lock the relevant row, check and write in one transaction. Smoke tests reserve against real carts to prove the limit holds.

A generic illustration of the pattern:

```sql
create function reserve_code(p_code_id uuid, p_order_id uuid)
returns boolean
language plpgsql
as $$
declare v_left int;
begin
  -- Lock the code row: concurrent callers wait here instead of racing.
  select max_uses - used into v_left
  from promo_codes where id = p_code_id
  for update;

  if v_left is null or v_left <= 0 then
    return false;
  end if;

  update promo_codes set used = used + 1 where id = p_code_id;
  insert into promo_reservations (code_id, order_id) values (p_code_id, p_order_id);
  return true;
end;
$$;
```

Invoice numbering got the same treatment: numbers are allocated inside an atomic function, so a failed request can never leave a gap or a duplicate.

### Access links and token lifetimes

The product uses bearer links so tenants can reach their portal or pay an invoice without an account.

- **Server-defined lifetimes:** link expiry is set by the server only; the client cannot choose or extend it. Expiring a link never locks a legitimate user out: a fresh link arrives with the next email.
- **Server-only token writes:** tokens are created and rotated by the server role, never with the end user's session.
- **Single purpose:** a payment link pays exactly one invoice and opens nothing else; a portal link is retired as soon as the tenant creates an account.
- **Rate-limited creation**, so link generation can't be abused.
- Calendar feed tokens were regenerated as part of the hardening.

### Authentication flows for a shared account system

Once the app shared its accounts with a second product, every auth flow had to be correct for both. Among the changes:

- **The app sends its own auth emails** (sign-up, recovery, email change), generated server-side and verified on its own callback, so the other product's email templates stay untouched.
- **A GET request never consumes an email link.** Links open a confirmation page and are verified on an explicit action, which stops email security scanners from burning one-time links.
- **Changing a password without the current one** is only possible inside a recovery session.
- **Sign-out closes only the current browser session**, not the user's sessions on the other product.
- **No account enumeration** on activation and sign-up screens.
- Clear feedback for weak or breached passwords.
- **Account linking across products** only links a verified email to a profile that is not already linked.

### Payments and webhooks

- Webhook handling made **idempotent**: a duplicated event or a double-paid subscription is credited once, and duplicates raise an alert for a human.
- Subscription identifiers verified before being used in queries.
- Failures that Stripe can retry are surfaced so Stripe retries them, instead of being swallowed.
- **Per-IP limits** on online payment attempts, and one Stripe session per subscription link.
- VAT on credit notes, refunds and reminders always read from the original invoice's frozen values.

### Web layer

- **Content Security Policy** rolled out in stages: first in report-only mode with a **violation-reporting endpoint**, so real violations could be read in production logs and fixed (including removing `eval` from the client bundle) before tightening.
- **Cloudflare Turnstile** on public forms, verified server-side.
- An AI-assisted document check (Claude API) got a **cost ceiling** with a fallback to manual review, so abuse cannot turn into an unbounded bill.

### Operations and code health

- **Sentry** error tracking with data collection minimized and events scrubbed of personal data.
- Personal data removed from maintenance scripts kept in the repository, and `.env.example` brought in line with every variable the code reads.
- **All 39 React lint errors cleared** (mostly `react-hooks` rules) without changing behaviour, verified by the test suite and before/after checks.
- Contributor rules updated (column locks, how migrations reach production, link-handling rules), so the next change doesn't reintroduce what was fixed.

## Integration: one database, one login

In parallel with the hardening, I moved the app onto esthéJob's Supabase project, into its own Postgres schema, sharing Supabase Auth with the marketplace. It was rehearsed end to end on a local copy proven to match production, with a scripted copy and a scripted rollback, and verified afterwards with table-by-table content hashes. The full story is in the esthéJob repository: [case study: merging two production apps onto one database](https://github.com/Robin-Hmaidan/esthejob/blob/main/docs/case-study-shared-database-merge.md).

## Lessons learned

1. **In a money app, correctness bugs are security bugs.** The most valuable fixes were race conditions and idempotency, not exotic attacks.
2. **Put the invariant in the database.** A check in application code can be raced; a row lock or a constraint cannot.
3. **Bearer links are credentials.** Give them a lifetime, a single purpose and a server-only writer, and design expiry so legitimate users are never stuck.
4. **Roll out strict policies with telemetry first.** Report-only CSP showed exactly what would break before anything was blocked.
5. **Write the tests before touching the money code.** They turned a risky refactor into a routine one.
6. **Verify on production, safely.** A smoke script that writes real data, checks invariants and cleans up gives more confidence than any amount of local testing.
7. **Leave rules behind, not just fixes.** Documented rules and automated checks are what keep a codebase hardened after the auditor leaves.
