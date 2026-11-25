# Deploying AI Skill Passport online

Two supported routes. Pick one:

- **Route A — Lovable (fastest, no server admin).** The project is already wired for Lovable
  Cloud: publishing handles the build, the hosting, the Supabase connection, and the
  environment variables for you.
- **Route B — Cloudflare Workers (you control the hosting).** You build the app yourself and
  push the Worker bundle to Cloudflare, then supply the Supabase variables as secrets.

Both routes need a Supabase project whose URL and publishable key you know, and whose
database already has the `profiles` table (see README section 7).

---

## Route A — Publish with Lovable

1. Open your project in the [Lovable editor](https://lovable.dev).
2. **Connect Supabase** if it is not connected yet: the Cloud panel provisions the database
   and injects `SUPABASE_URL`, `SUPABASE_PUBLISHABLE_KEY`, `VITE_*` equivalents, and the
   service-role key automatically — you never paste a key into code.
3. **Apply the database schema:** open the Supabase SQL Editor for that project and run
   `drizzle/migrations/0000_create_professional_profiles.sql` once (or ask Lovable to apply
   the migration for you).
4. Click **Publish**. Lovable builds the app (Vite + Nitro → Cloudflare Worker) and deploys it
   to a `*.lovable.app` address.
5. **Add your domain (optional):** Settings → Domain → connect a custom domain and follow the
   DNS records it shows.
6. **Configure auth redirects:** in Supabase → Authentication → URL configuration, add the
   published URL (e.g. `https://your-app.lovable.app`) as a Site URL and redirect URL, so
   sign-in and password-reset links land on the right host.
7. Re-publish after any code change.

Costs: Lovable's hosting and Supabase's free tier are enough to start; check both dashboards
for current limits.

---

## Route B — Deploy to Cloudflare Workers yourself

### B1. Build

```sh
npm install
npm run build          # writes a Cloudflare Worker bundle under .output/
```

If the build stops with a missing variable, that is expected for a production build:
set the `VITE_` values first (B2) and rebuild.

### B2. Environment variables

| Variable | Where to put it | Why |
| --- | --- | --- |
| `VITE_SUPABASE_URL`, `VITE_SUPABASE_PUBLISHABLE_KEY` | build-time environment (a `.env` file locally, or the platform's build variables) | They are compiled into the browser bundle |
| `SUPABASE_URL`, `SUPABASE_PUBLISHABLE_KEY` | runtime secret | Server functions and SSR read them |
| `SUPABASE_SERVICE_ROLE_KEY` | runtime secret, only if you use admin code | Bypasses row-level security — never expose it to the browser |

Locally:

```sh
cp .env.example .env    # fill in real values
npm run build
```

On Cloudflare, after the first deploy, store the runtime values as secrets:

```sh
npx wrangler secret put SUPABASE_URL
npx wrangler secret put SUPABASE_PUBLISHABLE_KEY
npx wrangler secret put SUPABASE_SERVICE_ROLE_KEY   # only if needed
```

### B3. Create the Worker configuration

The repository intentionally ships without a `wrangler.toml` (it belongs to your hosting
account). Create one in the project root:

```toml
name = "ai-skill-passport"
main = ".output/server/index.mjs"
compatibility_date = "2026-01-01"
compatibility_flags = ["nodejs_compat"]

[assets]
directory = ".output/public"
binding = "ASSETS"
```

Check the two paths against what your build actually produced (`ls .output`); if Nitro
wrote them elsewhere, point `main` and `directory` at the real files.

### B4. Deploy

```sh
npx wrangler login
npx wrangler deploy
```

Wrangler prints the live URL, e.g. `https://ai-skill-passport.<your-subdomain>.workers.dev`.

### B5. Point auth at the new URL

Supabase → Authentication → URL configuration:
- **Site URL** = your live URL
- **Additional redirect URLs** = your live URL plus `/reset-password`

Without this, sign-in and password-reset links fail or redirect to the wrong host.

### B6. Alternative hosts

- **Vercel / Netlify:** import the GitHub repository, set the build command `npm run build`
  and the output directory to Nitro's output, and add the same variables in the project's
  environment settings. The default build target is Cloudflare, so a non-Cloudflare host may
  need its Nitro preset changed before it works — if the deploy fails immediately, that is
  the first thing to check.
- **Any Node host:** the same Worker bundle can run under a Node-compatible server; you still
  supply the same variables.

---

## Verify the deployment (do this every time)

1. Open the live URL. The sample workspace renders with no blank page.
2. Open the browser console (F12) — no red errors on load.
3. `/auth` → create a real account with your own email, confirm it if Supabase requires it,
   then sign in.
4. On `/profile`, fill in every field with real values and press **Save** → success toast.
5. **Refresh the page.** The values are still there — the database round-trip works in production.
6. Open the site in a private window (or a second browser profile) and sign in with a
   **different** account. Its profile must be empty — that proves row-level security works
   on the deployed database, not just locally.
7. `/settings` → **Sign out** → open `/profile` directly → you are sent to sign-in.
8. `/passport` and `/showcase` still show the "sample / reference project" labels.
9. Request a password reset from `/auth` and follow the emailed link to `/reset-password`.

## Common deployment errors

| Error | What it means | Fix |
| --- | --- | --- |
| Blank page, console shows `Missing Supabase environment variable(s)` | The `VITE_` values were not present when the bundle was built | Set them in the host's build environment and rebuild/redeploy |
| `401` / `Invalid API key` | Wrong key or key type in the environment | Publishable key in `*_PUBLISHABLE_KEY`; secret key only in `SUPABASE_SERVICE_ROLE_KEY` |
| `relation "public.profiles" does not exist` | The migration was never applied to the production database | Run `drizzle/migrations/0000_create_professional_profiles.sql` in that project's SQL Editor |
| Sign-in works but the profile page is empty after refresh | Redirect URL mismatch, or the session belongs to a different project | Check Supabase URL configuration and that all variables point at the same project |
| `new row violates row-level security policy` | You are signed in but the trigger did not create your profile row, or you inserted with the wrong id | Confirm the `on_candidate_signup` trigger exists; save the profile from the signed-in page (the server sets the id) |
| CORS error on `supabase.co` | Your live URL is not allowed in Supabase | Add it under Authentication → URL configuration |
| `wrangler deploy` fails with `main` not found | `.output` paths in `wrangler.toml` do not match the build output | Inspect `.output` and correct `main` / `directory` |
| Worker runs out of memory / CPU | Unusual load or a heavy request | Split the work, cache reads, or raise the plan limit in Cloudflare |
| The deployed site is an older version | The build cache reused an old bundle | Trigger a clean rebuild (clear the host's build cache) and redeploy |

## Rollback

- **Lovable:** re-publish from an earlier commit (the editor keeps project history).
- **Cloudflare:** `npx wrangler deployments list` then `npx wrangler rollback` to pick a
  previous deployment.
- **Database:** the migration only adds a table, so a rollback of code does not need a
  database change. Never drop `public.profiles` — it holds candidate data.
