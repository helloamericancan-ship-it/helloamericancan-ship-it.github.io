# techrenew — Backend Project Brief

## Goal
Turn the existing front-end prototype (`index.html` = customer offer site, `admin.html` = admin panel)
into a real, working site with:
- Real accounts (sign up / sign in)
- A real database for categories, lots (with photos), and offers
- Real transactional email (new offer → notify admin; accepted/declined → notify buyer; signup confirmation)

## Recommended stack
- **Backend**: Node.js + Express (simple REST API)
- **Database + file storage**: Supabase (managed Postgres, plus Supabase Storage for lot photos instead of storing base64 in the database)
- **Auth**: Supabase Auth (handles sign up / sign in / password hashing — no need to build this by hand)
- **Email**: Resend (transactional email API)
- **Hosting**: Render or Railway for the API; Vercel/Netlify or Supabase Storage for the static front end (or keep GitHub Pages for the front end and just point it at the API)

## Existing front-end reference
Two files are already built as a working prototype (localStorage-only, no backend):
- `index.html` — customer site: category browsing, sign in/up, "make an offer" form
- `admin.html` — admin panel: add/edit/delete lots per category with photo upload, view + accept/decline offers, mock email log

These define the UI/UX and the exact data shape already in use (see below) — the backend should match
these field names so the front end needs minimal rework, just swapping `localStorage` calls for `fetch()`
calls to the new API.

## Database schema (Postgres / Supabase)

```sql
create table categories (
  id uuid primary key default gen_random_uuid(),
  name text not null,
  slug text not null unique
);

create table lots (
  id text primary key,              -- e.g. "TR-DSK-0142"
  category_id uuid references categories(id),
  title text not null,
  spec text,
  grade text check (grade in ('A','B','C')),
  asking_price integer not null,
  offer_deadline timestamptz,
  status text default 'open',       -- open | closed
  created_at timestamptz default now()
);

create table lot_images (
  id uuid primary key default gen_random_uuid(),
  lot_id text references lots(id) on delete cascade,
  url text not null,
  sort_order int default 0
);

-- users table: use Supabase Auth's built-in auth.users for login (email/password),
-- plus a profiles table for app-specific fields Supabase Auth doesn't store:
create table profiles (
  id uuid primary key references auth.users(id) on delete cascade,
  name text,
  address text,
  city text,
  state text,
  zip text,
  country text,
  notify_new_lots boolean default false,
  is_admin boolean default false          -- checked server-side and in RLS policies; never trust a client-sent flag
);

create table offers (
  id uuid primary key default gen_random_uuid(),
  lot_id text references lots(id),
  user_id uuid references auth.users(id),
  amount integer not null,
  note text,
  status text default 'Pending review',  -- Pending review | Accepted | Declined
  submitted_at timestamptz default now()
);
```

## API endpoints to build

| Method | Path | Purpose |
|---|---|---|
| GET | `/api/lots?category=` | List lots, optionally filtered by category |
| POST | `/api/admin/lots` | Create a lot (admin only) |
| PUT | `/api/admin/lots/:id` | Edit a lot (admin only) |
| DELETE | `/api/admin/lots/:id` | Delete a lot (admin only) |
| POST | `/api/lots/:id/offers` | Submit an offer (requires signed-in user) → triggers "new offer" email to admin |
| GET | `/api/admin/offers` | List all offers (admin only) |
| PUT | `/api/admin/offers/:id` | Accept/decline an offer (admin only) → triggers email to buyer |

Auth (`/api/auth/*`) and sign-in/sign-up can mostly be handled directly via the Supabase JS client from
the front end — no custom auth endpoints needed.

## Environment variables needed
```
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
RESEND_API_KEY=
ADMIN_EMAIL=            # where "new offer" notifications go
ADMIN_PASSCODE=         # replace the hardcoded admin1234 with this, checked server-side
```

## Suggested folder structure
```
techrenew-backend/
  server.js
  routes/
    lots.js
    offers.js
    admin.js
  lib/
    supabase.js
    email.js
  .env.example
  package.json
frontend/
  index.html   (adapted from the current prototype)
  admin.html   (adapted from the current prototype)
```

## Migration notes from the prototype
- Replace every `localStorage.getItem/setItem` call for `tr_listings`, `tr_offers`, `tr_accounts` with
  `fetch()` calls to the new API.
- Replace the hardcoded `admin1234` passcode check with a real server-side check (or just gate the whole
  admin page behind Supabase Auth with an `is_admin` flag on the user).
- Photo uploads: switch from FileReader → base64 (fine for a demo, bad for a real database) to uploading
  the file to Supabase Storage and saving the returned URL in `lot_images`.
- Countdown/deadline logic can stay client-side exactly as it is, just reading `offer_deadline` from the
  API response instead of localStorage.
- **Timezone handling for "Closes at":** the admin's date/time picker for `offer_deadline` currently treats
  Eastern Time as a fixed UTC-5 offset and does not adjust for daylight saving (EDT is UTC-4 in summer).
  In the real backend, convert the admin's picked local time to UTC using a proper timezone-aware library
  (e.g. `Intl.DateTimeFormat` with `timeZone: "America/New_York"` in Node, or `luxon`/`date-fns-tz`) so it
  adjusts automatically across DST changes, rather than hardcoding the offset.

## Security considerations

The prototype (as it stands, running purely in the browser via localStorage) has **no real security** —
none of the following are fixed with a config tweak; they require the actual backend described above.
Treat the checklist below as required work, not optional hardening, before any real customer data touches
this system.

- **Never store or check passwords in the frontend.** The prototype's `tr_accounts` stores passwords in
  plain text in the browser. Replace this entirely with Supabase Auth (or an equivalent), which handles
  password hashing, session tokens, and password-reset flows — do not hand-roll any of this.
- **Never gate admin access with a hardcoded passcode.** `admin1234` is visible to anyone who views the
  page source. Instead, add an `is_admin` (or `role`) column to the users table, check it server-side on
  every admin API call, and use Supabase Row Level Security (RLS) policies so the database itself refuses
  admin-only reads/writes from non-admin sessions — not just the UI hiding buttons.
- **Enforce access control at the database layer (RLS), not just in API route logic.** Example policies:
  a signed-in user can read/write only their own row in `accounts`/`offers`; only `is_admin = true` users
  can read the full `accounts` or `offers` tables; anyone (including logged-out visitors) can read `lots`.
- **Validate and sanitize all input server-side**, even though the frontend already validates (required
  fields, offer amount > 0, etc.) — frontend validation is a UX nicety, not a security boundary. Re-check
  types, lengths, and ranges in the API before writing to the database.
- **Keep all secrets server-side only**, in environment variables — `SUPABASE_SERVICE_ROLE_KEY`,
  `RESEND_API_KEY`, database credentials. None of these should ever appear in frontend code, a public repo,
  or browser devtools. The frontend should only ever use Supabase's public anon key, which RLS policies
  make safe to expose.
- **Rate-limit sensitive endpoints** — sign-in attempts, sign-up, and offer submission — to slow down
  brute-force and spam/bot abuse (Render/Railway or an API gateway can add this without custom code).
- **Restrict CORS** on the API to the site's actual domain(s) rather than allowing all origins (`*`).
- **Don't log personal data** (passwords, full addresses, emails) in plaintext server logs; log IDs and
  event types instead.
- **HTTPS is already covered** by the recommended hosts (GitHub Pages, Supabase, Render/Railway all serve
  over HTTPS by default) — no extra setup needed there.
- **Have a privacy policy / data-handling statement** once real customer PII (name, address, email) is
  collected — this is a legal/compliance point worth a lawyer's input if selling in the US, not something
  code alone solves.
- **Keep dependencies updated** (`npm audit`, Dependabot on the GitHub repo) once the backend has a
  `package.json` full of real dependencies.
