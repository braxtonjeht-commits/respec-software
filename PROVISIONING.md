# Respec Platform — Company Provisioning Runbook

How to manually onboard a new client company to the multi-tenant client portal
(`respecsoftware.com/login/`). Provisioning is **admin-only** — there is no public
self-serve signup.

- **Supabase project:** `respec-platform` — ref `cnpyltipddxfjyucigxj`
- **Dashboard:** https://supabase.com/dashboard/project/cnpyltipddxfjyucigxj
- **Not** to be confused with `ltmakawhazxasgvkygso` (Josh / Relentless Motorsports'
  existing single-tenant system). That project is separate and untouched.

## Data model

| Table | Purpose |
|---|---|
| `public.companies` | One row per client business. `company_code` is what they type in the **Company ID** field at login. |
| `public.company_users` | Links a Supabase Auth user (`auth.users.id`) to a company. This is what actually grants access. |
| `public.my_company` | Convenience view — returns the signed-in user's own company + role. Runs with `security_invoker`, so RLS applies. |

RLS is on for both tables: a signed-in user can read **only** their own company row
and their own membership row. There are no client-side insert/update/delete policies —
all provisioning happens through the dashboard or the service role.

> **Company ID is not a secret.** It's a human-typeable code. The password is the
> secret. The login page requires the typed Company ID to match the user's real
> `company_users` row — a mismatch is treated as a failed login.

## Provisioning a new company

### Step 1 — Create the company row

SQL Editor → run (adjust values):

```sql
insert into public.companies (company_code, name, manage_url, hud_url, client_url)
values (
  'RELENTLESS-TX',                            -- uppercase, no spaces; typed at login
  'Relentless Motorsports TX',                -- display name
  'https://josh-ecu-dashboard.pages.dev',     -- Manage box
  'https://josh-ecu-hud.pages.dev',           -- HUD box
  'https://josh-ecu-intake.pages.dev'         -- Client box
)
returning id, company_code;
```

Copy the returned `id`.

Any of the three URLs may be left `null` — the login hub hides a box that has no URL,
rather than showing a dead link.

### Step 2 — Create the Auth user

Dashboard → **Authentication → Users → Add user → Create new user**

- **Email:** the person's real email (this is what they type in the **Username** field)
- **Password:** set one, send it to them over a secure channel
- **Auto Confirm User:** ✅ **ON** — otherwise they cannot sign in until they confirm

Copy the new user's UUID.

### Step 3 — Link the user to the company

```sql
insert into public.company_users (user_id, company_id, role)
values (
  '<auth user uuid from step 2>',
  '<company id from step 1>',
  'owner'          -- 'owner' | 'member'
);
```

### Step 4 — Verify

Go to https://respecsoftware.com/login/ and sign in with the email, password, and
Company ID. You should land on the hub with the three boxes pointing at that
company's URLs. Reload — the session should persist. Sign out — you should return
to the login form.

## One-step helper

To do steps 1 and 3 in one go **after** the Auth user exists (step 2 must be done in
the dashboard, since password hashing is handled by Auth):

```sql
select public.provision_company(
  'RELENTLESS-TX',                          -- company_code
  'Relentless Motorsports TX',              -- name
  'owner@relentless.example',               -- email of an EXISTING auth user
  'https://josh-ecu-dashboard.pages.dev',   -- manage_url
  'https://josh-ecu-hud.pages.dev',         -- hud_url
  'https://josh-ecu-intake.pages.dev'       -- client_url
);
```

Returns the new company's UUID. It is idempotent on `company_code` (re-running
updates the URLs rather than erroring) and will raise a clear error if no auth user
exists for the given email. It runs as `security definer` and is **revoked from
`anon` and `authenticated`** — it is only callable from the SQL editor / service role.

## Updating a company's destinations

```sql
update public.companies
set manage_url = 'https://new-dashboard.example',
    hud_url    = 'https://new-hud.example',
    client_url = 'https://new-intake.example'
where company_code = 'RELENTLESS-TX';
```

Changes take effect on the user's next sign-in or page reload — no deploy needed.

## Removing access

```sql
-- revoke one user's access (leaves the company and the auth user intact)
delete from public.company_users where user_id = '<uuid>';
```

To fully offboard, also delete the Auth user in the dashboard. Deleting a
`companies` row cascades to its `company_users` rows.

## Notes / current limits

- The three destination apps (`josh-ecu-*`) are still single-tenant. The
  `manage_url` / `hud_url` / `client_url` columns are a **temporary bridge** so each
  company can point at its own app. Making those apps company-aware (reading the
  session + `company_id` and applying per-company branding/config) is a later phase.
- `companies.config` (jsonb) and `logo_url` / `brand_color` exist for per-company
  branding but are not consumed by the login page yet.
- The publishable key in `login/index.html` is safe to commit — RLS is what enforces
  isolation. The **service role key must never** be committed or used client-side.
