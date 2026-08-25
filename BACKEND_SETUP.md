# Opt-in database setup (Supabase)

This site is static (GitHub Pages), so opt-ins from `landing.html` are stored in a
[Supabase](https://supabase.com) project: a hosted Postgres database with a free
tier, a private dashboard only you can log into, and a REST API the landing page
calls directly. No automation platform, no middleman.

Takes about 5 minutes.

## 1. Create the project

1. Go to [supabase.com](https://supabase.com) and create a free account/project.
2. Wait for the project to finish provisioning.

## 2. Create the `leads` table

In the Supabase dashboard, open **SQL Editor** and run:

```sql
create table public.leads (
  id uuid primary key default gen_random_uuid(),
  created_at timestamptz not null default now(),
  name text not null,
  email text not null,
  phone text not null,
  tag text,
  source text
);

alter table public.leads enable row level security;

-- The public landing page can only INSERT new leads.
create policy "Anyone can submit a lead"
  on public.leads
  for insert
  to anon
  with check (true);

-- Only a signed-in user (you, via admin.html) can read leads.
create policy "Only authenticated users can view leads"
  on public.leads
  for select
  to authenticated
  using (true);
```

There are deliberately no `update` or `delete` policies for the `anon` or
`authenticated` roles — submitted leads can't be altered or removed through the
public API. You can still edit or delete rows yourself anytime from the
Supabase dashboard's **Table Editor**, which always has full access regardless
of these policies.

## 3. Create your own login (for `admin.html`)

1. In the dashboard, go to **Authentication → Users → Add user**.
2. Create a user with your own email and a password. This is the only account
   that will ever be able to read the `leads` table from outside the dashboard.

## 4. Get your API keys

Go to **Project Settings → API**. Copy:

- **Project URL** (looks like `https://xxxxxxxxxxxx.supabase.co`)
- **anon public** key (a long JWT string — this is safe to expose client-side;
  it can only insert leads or read them after a valid login, per the policies
  above)

## 5. Wire them into the site

Paste both values into two places:

- `landing.html` — inside the `<script>` block near the top, replace
  `SUPABASE_URL` and `SUPABASE_ANON_KEY`.
- `admin.html` — same two constants, near the top of its `<script>` block.

Commit and push. Do **not** use the "service role" key anywhere in these files —
only the "anon public" key, which is the one designed to be exposed in
client-side code.

## 6. View your opt-ins

Open `admin.html` on the live site and sign in with the account from step 3.
It lists every opt-in (name, email, phone, source, timestamp), newest first.
You can also always view/export the raw table from the Supabase dashboard's
Table Editor.

## Optional: also keep the ESP/CRM webhook

`landing.html` still has a `WEBHOOK_URL` placeholder if you want opt-ins to
also trigger an email sequence in ActiveCampaign, GoHighLevel, etc. It fires
in parallel with the Supabase insert and is independent of it — leave it as
the placeholder to skip it entirely.
