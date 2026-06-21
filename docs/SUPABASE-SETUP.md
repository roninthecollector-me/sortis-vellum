# Cloud sync setup (Supabase)

This enables a simple login and syncs one world across all your devices. The app works fully **without** this — it's optional. Total time ~10–15 minutes. Steps marked **[Claude Code can do this]** can be driven by the Supabase MCP connector or the Supabase CLI; the rest are quick dashboard clicks.

The architecture: one row per user in a `worlds` table holding the entire world as JSON. Reads/writes are locked to the signed-in user by Row Level Security. The client pulls on sign-in and pushes (debounced) on every change — last write wins.

---

## 1. Create a Supabase project

Dashboard → **New project**. Pick a name and a region near you; save the database password. **[Claude Code can do this]** via the Supabase connector (`create_project`).

## 2. Create the table + security

SQL Editor → run this. It creates the per-user world store and locks each row to its owner.

```sql
create table if not exists public.worlds (
  user_id    uuid primary key references auth.users(id) on delete cascade,
  data       jsonb not null,
  updated_at timestamptz not null default now()
);

alter table public.worlds enable row level security;

create policy "read own world"  on public.worlds
  for select using (auth.uid() = user_id);

create policy "insert own world" on public.worlds
  for insert with check (auth.uid() = user_id);

create policy "update own world" on public.worlds
  for update using (auth.uid() = user_id) with check (auth.uid() = user_id);
```

## 3. Turn on a login method

**Email magic link — simplest, works immediately.** Authentication → Providers → **Email** is on by default. Nothing else needed; users get a sign-in link by email.

**Google — nicer one-click UX, a little more setup.** Authentication → Providers → **Google** → enable, then paste a Google OAuth client ID/secret:
- Google Cloud Console → APIs & Services → Credentials → Create OAuth client ID → *Web application*.
- Authorized redirect URI: the value Supabase shows on the Google provider page (looks like `https://<project-ref>.supabase.co/auth/v1/callback`).
- Copy the client ID + secret back into Supabase.

*Recommendation: ship with magic-link first to get sync working, add Google when you want the one-click button.*

## 4. Set the redirect URLs

Authentication → **URL Configuration**:
- **Site URL**: your deployed URL (e.g. `https://your-app.vercel.app`).
- **Redirect URLs**: add the deployed URL and, for testing, `http://localhost:3000` (or wherever you serve it).

The app calls auth with `redirectTo: window.location.origin`, so the origin it runs on must be listed here.

## 5. Wire the keys into the app

Dashboard → Project Settings → **API**. Copy the **Project URL** and the **anon / public** key into `config.js`:

```js
window.SORTIS_CONFIG = {
  supabaseUrl: "https://YOUR-PROJECT-ref.supabase.co",
  supabaseAnonKey: "YOUR-ANON-PUBLIC-KEY"
};
```

The anon key is meant to be public (it ships in client JS); your data is protected by the RLS policies from step 2. Commit `config.js` — a static site has no build step to inject env vars, so the file must be served.

## 6. Deploy & test

Push to GitHub → Vercel redeploys. Open the site → **Settings → Cloud sync** → sign in. Your local world seeds the cloud on first sign-in; after that it syncs on every change, and signing in on another device pulls it down.

---

### Notes & limits (current, simple version)
- **Last write wins.** If you edit the same world on two devices at the same time, the most recent save wins. Fine for one person across devices; not multi-user co-editing.
- **One world per account.** The schema stores a single world per user. Multiple named worlds per account would be a small schema change (add a `world_id`) when you want it.
- **Local-first still works.** Even signed in, the world is cached locally, so the app keeps working offline and pushes when it can.
- **Panel images (later).** The map-viewer feature adds a Storage bucket (`panels`) with its own policies — that setup lives in `docs/MAP-VIEWER.md`, not here. This doc only covers auth + JSON world sync.
