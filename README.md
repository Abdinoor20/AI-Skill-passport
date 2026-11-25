# AI Skill Passport

An evidence-based skills passport for emerging talent: candidates build a professional
profile, show real project evidence, and preview a digital passport — while their personal
data stays private to their own account.

This repository is the complete application source: front end, server functions, database
schema, tests, and documentation. It is ready to run locally, upload to GitHub, and deploy
online.

---

## 1. What the app does

| Page | Path | Purpose |
| --- | --- | --- |
| Workspace (home) | `/` | Overview with sample metrics, a labelled reference project, passport journey, and skills at a glance. Works signed-out as a demonstration. |
| Sign in / Sign up | `/auth` | Email + password account creation, sign-in, password reset request, and Google / Lovable social sign-in. |
| Reset password | `/reset-password` | Sets a new password from the emailed reset link. |
| Professional profile | `/profile` | Private, account-owned profile (name, headline, location, bio, GitHub and LinkedIn links) with live validation, completion percentage, and save. Requires sign-in. |
| Sample passport | `/passport` | Printable demonstration passport, clearly labelled as a sample and not an independently verified qualification. |
| Project showcase | `/showcase` | Real open-source reference project (Excalidraw) presented as evidence, with no candidate authorship claimed. |
| Settings | `/settings` | Account summary, sign out, and a plain-language privacy statement. |

**Milestone status (see `roadmap.md`):** Milestone 1 is implemented and tested — account
registration, sign-in, professional profile persistence, the overview page, and a clearly
labelled demonstration preview. Milestones 2–5 (skill/qualification records with evidence
uploads, GitHub analysis and assessments, credential mapping, passport generation and
public sharing) are planned and are **not** built yet.

## 2. Project requirements

**Functional**
1. Visitors can browse a demonstration workspace without an account.
2. Anyone can create an account with email + password (or Google / Lovable sign-in), sign in, sign out, and reset a forgotten password.
3. A signed-in candidate can create and update one professional profile; the data survives a page refresh and is reloaded on the next visit.
4. Profile input is validated before saving: name required (2–100 characters), headline ≤150, location ≤100, bio ≤2000, and GitHub / LinkedIn links must be `https://` links to `github.com` / `linkedin.com`.
5. Sample content (demo metrics, the "Amira Hassan" persona, the Excalidraw reference project) is always visibly labelled as a demonstration and is never written to a real candidate's profile.

**Non-functional / safeguards**
- Every profile read and write goes through an authenticated server function, so the caller's identity is verified on the server.
- Row Level Security in the database enforces the same rule a second time: a user can only select, insert, or update the row whose `id` equals their own account id.
- The service-role key is only used inside server-side modules and is never imported by browser code.
- No secrets, passwords, or API keys belong in this repository (see `.gitignore` and `DEPLOYMENT.md`).

## 3. Technology stack

- **Framework:** TanStack Start v1 (React 19, TypeScript, SSR + server functions), Vite 8
- **Routing:** TanStack Router (file-based routes in `src/routes`)
- **Data:** TanStack Query + Supabase (`@supabase/supabase-js`)
- **Auth:** Supabase Auth (email/password, password recovery, OAuth) via Lovable Cloud
- **Styling:** Tailwind CSS v4 with semantic design tokens in `src/styles.css`, Radix UI primitives (shadcn-style components in `src/components/ui`)
- **Validation:** Zod (`src/lib/profile-schema.ts`)
- **Database migrations:** Drizzle Kit (`drizzle/migrations`)
- **Tests:** Vitest + Testing Library
- **Build target:** Nitro → Cloudflare Worker (`.output/`)

## 4. Prerequisites

- **Node.js 20.19+ or 22+** (and npm 10+) — <https://nodejs.org>. Bun 1.2+ also works and matches the committed `bun.lock`.
- **A Supabase project** (free tier is fine) — <https://supabase.com>. You need its project URL, publishable key, and (optionally) secret/service-role key.
- **Git** — for uploading to GitHub.
- Optional: a **Lovable** account if you want to keep editing the app visually and publish with one click.

## 5. Quick start (run it locally)

```sh
# 1. Install dependencies
npm install          # or: bun install

# 2. Create your environment file (never commit it)
cp .env.example .env
# then open .env and paste your own Supabase URL + keys

# 3. Create the database table (see section 7)
#    Paste drizzle/migrations/0000_create_professional_profiles.sql into the
#    Supabase SQL Editor and run it once.

# 4. Start the dev server
npm run dev
```

Open the URL the terminal prints (Vite's default is `http://localhost:5173`; the Lovable
preview environment uses port `8080`).

Then verify it end to end:

1. Open `/` — the sample workspace renders.
2. Go to `/auth` → **Create an account** (8+ character password). If your Supabase project requires email confirmation, confirm the email first, then sign in.
3. You land on `/profile`. Fill in the fields, watch the completion percentage change, and press **Save**.
4. Refresh the page — your saved values are still there (that proves the database round-trip works).
5. Open `/settings` → **Sign out**, then open `/profile` — you are sent back to sign-in.

## 6. Environment configuration

Copy `.env.example` to `.env` and fill in real values. `.env` is ignored by git.

| Variable | Where it is read | Required | What it is |
| --- | --- | --- | --- |
| `VITE_SUPABASE_URL` | browser | yes | Your Supabase project URL, e.g. `https://xxxx.supabase.co` |
| `VITE_SUPABASE_PUBLISHABLE_KEY` | browser | yes | The **publishable** key (`sb_publishable_…`) |
| `SUPABASE_URL` | server | yes | The same project URL (used by server functions and SSR) |
| `SUPABASE_PUBLISHABLE_KEY` | server | yes | The same publishable key |
| `SUPABASE_SERVICE_ROLE_KEY` | server only | only if you use admin code | The **secret** key. Used by `src/integrations/supabase/client.server.ts`, which bypasses row-level security — keep it out of the browser bundle and out of git. |
| `LOVABLE_CRON_SECRET` | server | no | Only needed if you add scheduled jobs (`src/integrations/supabase/cron-auth.ts`) |

Notes:
- `.gitignore` in this copy does **not** yet list `.env`, so before your first git commit add
  it: `printf '\n.env\n.env.*\n!.env.example\n' >> .gitignore` (see `GITHUB-UPLOAD.md`, step 0).
- `VITE_`-prefixed variables are baked into the browser bundle at build time — they are
  public by design, so only put the publishable key there.
- The app throws a clear error at startup if a required variable is missing:
  `Missing Supabase environment variable(s): …`.

## 7. Database setup

The whole schema for this milestone is one table plus its security rules, in
`drizzle/migrations/0000_create_professional_profiles.sql`.

**Apply it (easiest way):** Supabase dashboard → **SQL Editor** → paste the file's contents
→ **Run**. It is safe to run once on a fresh project.

What it creates:

- `public.profiles` — one row per candidate: `id` (the account's user id, primary key),
  `full_name`, `headline`, `location`, `bio`, `github_url`, `linkedin_url`, `created_at`,
  `updated_at`.
- **Row Level Security** policies: `Read own profile`, `Create own profile`,
  `Update own profile` — each compares the row's `id` with `auth.uid()`, so no account can
  read or change another account's row.
- `GRANT SELECT, INSERT, UPDATE … TO authenticated` and full rights to `service_role`.
- A trigger `on_candidate_signup` + function `handle_new_candidate()` that inserts an empty
  profile row automatically whenever a new account is created, so the profile page always
  has something to open.

**Verify it worked** (SQL Editor):

```sql
select relname, relrowsecurity from pg_class where relname = 'profiles';
select polname from pg_policy where polrelid = 'public.profiles'::regclass;
```

You should see `relrowsecurity = true` and three policies listed.

**Changing the schema later:** edit `drizzle/schema.ts` and run
`npx drizzle-kit generate` to write a new migration file, then apply that file in the SQL
Editor. `drizzle.config.ts` reads the connection string from `LOVABLE_DB_MIGRATION_URL`;
on Lovable, ask the assistant to make schema changes instead so the migration is recorded
correctly.

## 8. Project structure

```
├── drizzle/
│   ├── migrations/0000_create_professional_profiles.sql   # schema + RLS + signup trigger
│   └── schema.ts
├── public/                     # favicon, robots.txt
├── src/
│   ├── assets/                 # CDN pointers for the demo portrait / project image
│   ├── components/
│   │   ├── auth-provider.tsx   # single app-wide sign-in listener (clears cache on sign-out)
│   │   ├── workspace.tsx       # shared app shell / navigation
│   │   ├── passport-card.tsx   # passport visual
│   │   └── ui/                 # shadcn-style primitives (button, card, input …)
│   ├── integrations/supabase/  # generated clients, auth middleware, cron guard
│   ├── lib/
│   │   ├── profile-schema.ts   # Zod validation + completion score (browser-safe)
│   │   ├── profile.functions.ts# authenticated server functions: getProfile / saveProfile
│   │   ├── demo.ts             # labelled demonstration content
│   │   └── page-meta.ts        # per-page <head> metadata
│   ├── routes/                 # file-based pages: /, /auth, /profile, /passport,
│   │                           # /showcase, /settings, /reset-password
│   ├── styles.css              # Tailwind v4 theme tokens
│   └── test/                   # Vitest suites
├── .env.example                # safe template — copy to .env
├── GITHUB-UPLOAD.md            # step-by-step GitHub guide
├── DEPLOYMENT.md               # step-by-step hosting guide
└── roadmap.md                  # milestone plan
```

## 9. Commands

| Command | What it does |
| --- | --- |
| `npm run dev` | Start the dev server with hot reload |
| `npm run build` | Production build (Nitro → Cloudflare Worker output in `.output/`) |
| `npm run preview` | Serve the production build locally |
| `npm test` | Run the test suite (Vitest) |
| `npm run lint` | ESLint over the whole project |
| `npm run format` | Prettier write |

## 10. Testing and quality checks

`npm test` currently runs 4 assertions in 2 files:

- `src/test/profile-schema.test.ts` — profile validation rules: a valid profile passes,
  short names are rejected, non-`https` or wrong-domain GitHub/LinkedIn links are rejected,
  and the completion score is calculated correctly.
- `src/test/app-routing.test.tsx` — the router's page graph is intact (every page is
  reachable and correctly named).

Before calling any change done, run:

```sh
npm test && npm run lint && npm run build
```

Then click through the flow in section 5 (sign up → save profile → refresh → sign out).

## 11. Troubleshooting

| Symptom | Cause and fix |
| --- | --- |
| `Missing Supabase environment variable(s): SUPABASE_URL …` at startup | `.env` is missing or the names are misspelled. `cp .env.example .env` and fill in real values, then restart `npm run dev`. |
| `Failed to fetch` / CORS errors on sign-in | The Supabase URL in `.env` must be the project's own URL, and the URL you open in the browser must be allowed under Supabase → Authentication → **URL configuration** (add `http://localhost:5173` and your future domain). |
| Sign-up says "check your email" but nothing arrives | Supabase's email confirmation is on. Confirm the address, or (for local testing only) disable "Confirm email" in Authentication → Providers → Email. |
| Profile page shows "Your profile could not be loaded" | The `profiles` table or its RLS policies are missing — re-run the migration SQL in section 7, and make sure you are signed in. |
| Profile page redirects to `/auth` even when signed in | The session expired or the token isn't attached. Sign out and back in; check that `SUPABASE_PUBLISHABLE_KEY` matches the browser key. |
| "Use a github.com profile link" / "Use an https link" toast | Expected validation: paste a full `https://github.com/yourname` (or `linkedin.com/in/yourname`) link. |
| `npm run build` fails on a memory limit | Close other processes, or build with `NODE_OPTIONS=--max-old-space-size=4096 npm run build`. |
| Build works locally but the deployed site shows a blank page | The `VITE_` variables were not set in the hosting platform before the build — set them and rebuild (see `DEPLOYMENT.md`). |
| A deployed page returns 500 with `Unauthorized: No authorization header provided` in the logs | A protected page was opened directly without a session; sign in first. Server functions intentionally reject anonymous calls. |
| Google sign-in does nothing when hosted yourself | The OAuth providers are wired through Lovable Cloud. Either host on Lovable, or add your own Google OAuth configuration in Supabase → Authentication → Providers → Google. |

## 12. Next steps

- Upload the code to GitHub: follow **`GITHUB-UPLOAD.md`**.
- Put it online: follow **`DEPLOYMENT.md`** (Lovable publish, or Cloudflare Workers).
- Keep building: the next milestone in `roadmap.md` is skill/qualification records with
  evidence uploads and repository validation.

## 13. Known limitations (stated honestly)

- Only Milestone 1 is built; the passport, showcase, and metrics you can see are labelled
  demonstration content, not verified candidate data.
- Profile data is private to the account that owns it — there is no public sharing,
  recruiter search, or admin review yet.
- OAuth sign-in depends on Lovable Cloud; email/password sign-in works anywhere.
- Deploying on your own requires a Supabase project and its keys; the ZIP deliberately
  contains no credentials, so nothing is exposed if you share it.
