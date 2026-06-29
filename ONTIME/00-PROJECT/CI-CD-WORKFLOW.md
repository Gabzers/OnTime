# CI/CD & Git Workflow

Hub: [[OnTime]] · Maintenance: [[HOW-TO-USE]].

How code gets from a local change to `main` in both `OnTime_Backend` and `OnTime_Frontend`, and
how CI gates that.

## The rule: never push straight to `main`

1. Work happens on a branch (e.g. `chore/topic`, `feat/topic`), never directly on `main`.
2. Push the branch → open a Pull Request on GitHub.
3. GitHub Actions (`.github/workflows/ci.yml` in each repo) runs automatically on every push to a
   PR: backend = `dotnet build` + `dotnet test` (Testcontainers, real Postgres); frontend =
   `npm run test` (Vitest) + `npm run build` (tsc + vite). Frontend PRs also get a free Vercel
   preview deployment + comment automatically (already configured via the Vercel GitHub App).
4. **Merging is always a manual, explicit action by the user on GitHub** (or `gh pr merge` only
   when the user says so in the moment) — even if CI is fully green. No auto-merge, ever, even
   though GitHub's "Allow auto-merge" repo setting exists and was briefly enabled then turned back
   off on 2026-06-27 at the user's explicit request. Don't re-enable it without asking again.
5. After merging on GitHub, sync the local clone: `git checkout main && git pull` in whichever
   repo you're working in. The remote branch is deleted on merge (squash + delete-branch), so
   there's nothing left to clean up locally beyond an occasional `git fetch --prune`.

## What's gated, what isn't

- **Backend CI** (`OnTime_Backend/.github/workflows/ci.yml`): `dotnet restore/build/test` against
  `OnTime.sln`. Needs Docker (Testcontainers spins up real Postgres) — works on GitHub's
  `ubuntu-latest` runners natively, no extra setup.
- **Frontend CI** (`OnTime_Frontend/.github/workflows/ci.yml`): `npm ci`, `npm run test`,
  `npm run build`. **Lint is intentionally not run in CI** — `eslint.config.js` (v9 flat config)
  is missing from the repo, so `npm run lint` fails immediately regardless of code quality. See
  [[KNOWN-ISSUES]]. Re-add the lint step once that's fixed.
- **No branch protection rule exists yet** on either repo's `main` — CI runs and reports status on
  PRs, but nothing currently *blocks* a merge if checks are red. Setting one up (Settings →
  Branches → add rule → "Require status checks to pass" → select `build-and-test`) is a manual
  step still to do, one-time, per repo.
- **Deploy is separate from this CI** and already automatic: Render (backend) and Vercel
  (frontend) both auto-deploy from `main` on every merge — that's *their* own GitHub integration,
  not the Actions workflow. The Actions CI is purely a pre-merge gate; it doesn't deploy anything.

## Local tooling

`gh` (GitHub CLI) is installed (via `winget install --id GitHub.cli`, Chocolatey failed locally
needing admin rights) and authenticated as `Gabzers`. Useful for read-only status checks:
```bash
gh pr list --repo Gabzers/OnTime_Backend
gh pr checks --repo Gabzers/OnTime_Backend <branch-name>
```
**Never use `gh pr merge` (or any merge action) without the user explicitly asking for it in that
exact moment** — confirming CI is green is not the same as authorization to merge.

## Related
[[HOW-TO-USE]] · [[KNOWN-ISSUES]] · [[BEFORE-DEPLOY]] · [[OnTime]]
