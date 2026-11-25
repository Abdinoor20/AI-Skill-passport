# Uploading AI Skill Passport to GitHub

Follow this top to bottom. Every command is copied exactly as you should type it, with the
output you should expect. Stop at any step that looks different and fix that before
continuing.

## Step 0 — Confirm no secrets are in the code (2 minutes)

The ZIP you received has **no** `.env` file — only `.env.example`, which contains placeholders.
Keep it that way. First make sure git is told to ignore your real environment file, because
the project's `.gitignore` does not list it yet:

```sh
printf '\n# Real credentials — never commit\n.env\n.env.*\n!.env.example\n' >> .gitignore
```

Before the first push, check that git will not carry a secret:

```sh
# From the project folder
git status --short                # nothing named .env should appear
git check-ignore -v .env          # should print the .gitignore rule that ignores it
grep -RInE "sb_secret_|sb_publishable_|service_role|eyJ[A-Za-z0-9_-]{20,}" . --exclude-dir=node_modules
```

The last command searches the whole project for secret-shaped strings. The expected result
is **no matches** (or matches only inside comments/example placeholders). If a real key ever
appears in a file, move it to `.env`, and if it was already pushed, rotate it in the Supabase
dashboard — a pushed key is a compromised key.

## Step 1 — Create the repository on GitHub

1. Sign in to <https://github.com> → **New repository** (top-right `+`).
2. Repository name: `ai-skill-passport`.
3. Visibility: **Private** while you are still learning; **Public** if you want it as portfolio work.
4. **Do not** tick "Add a README", ".gitignore", or "license" — your project already has them, and ticking them makes the first push collide.
5. Click **Create repository**. GitHub shows you the exact remote URL, e.g.
   `https://github.com/YOUR-USERNAME/ai-skill-passport.git`.

## Step 2 — Turn the project folder into a git repository

```sh
cd path/to/AI-Skill-Passport     # the folder you unzipped
git init -b main
git add .
git status                        # review: no .env, no node_modules, no dist
git commit -m "feat: AI Skill Passport milestone 1 (accounts, profile, sample passport)"
```

If `git status` lists `node_modules/` or `.env`, stop and fix `.gitignore` before committing.
`node_modules` and `dist` are already ignored by the project's `.gitignore`.

## Step 3 — Connect and push

```sh
git remote add origin https://github.com/YOUR-USERNAME/ai-skill-passport.git
git remote -v                     # confirms the URL
git push -u origin main
```

- **GitHub asks for credentials:** a password will not work. Use a **Personal Access Token**
  (GitHub → Settings → Developer settings → Personal access tokens → generate one with
  `repo` scope) as the password, or use the **GitHub CLI** (`gh auth login`) or an SSH remote
  (`git@github.com:YOUR-USERNAME/ai-skill-passport.git`).
- **`! [rejected] … (fetch first)`:** GitHub created files you don't have. Run
  `git pull --rebase origin main`, resolve any conflict, then `git push -u origin main`.
- **`fatal: not a git repository`:** you are in the wrong folder — `cd` into the project first.

Expected result: `github.com/YOUR-USERNAME/ai-skill-passport` shows `src/`, `drizzle/`,
`README.md`, and **no** `.env`.

## Step 4 — Everyday workflow

```sh
git status                        # what changed
git add .                         # stage changes
git commit -m "fix: describe the change"
git push
```

Work on a branch instead of `main` when the change is big:

```sh
git switch -c feat/skills-records
# … edit, commit …
git push -u origin feat/skills-records     # then open a Pull Request on GitHub
```

## Step 5 — Invite a collaborator (optional)

Repository → **Settings** → **Collaborators** → **Add people** → type their GitHub username.
They clone with `git clone https://github.com/YOUR-USERNAME/ai-skill-passport.git` and create
their **own** `.env` from `.env.example`. Never send a `.env` file by email or chat.

## Step 6 — Protect the repository

- **Branch protection:** Settings → Branches → add a rule for `main` so changes arrive through
  pull requests.
- **Secret scanning:** Settings → Code security and analysis → enable **Secret scanning**
  (free on public repos) so a pasted key is flagged.
- **If a secret is ever committed:** remove it from the file, then rewrite history
  (`git filter-repo` or BFG) **and rotate the key in Supabase**. Rotation is the step that
  actually makes it safe.

## Step 7 — Keep the repository clean

The project's `.gitignore` already excludes `node_modules`, `dist`, `.output`, `.wrangler`,
`*.local`, and `.env`. If you add a new tool that writes cache files, add its folder to
`.gitignore` in the same commit.
