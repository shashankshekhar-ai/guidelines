# Globiculum Engineering Guidelines

Status: proposed, 2026-10-20. Covers the repo review, coding style, the Git workflow, prompting for AI coding tools and generic security rules.
Owner: whole team. Change this file by PR like any other code.

---

## 1. Repo review: where we are

Snapshot taken 2026-10-20 on `main` (`781c097`).

### What is working

- The structure is clear: `frontend/` (React + Vite + TS + shadcn), `backend/supabase/` (Edge Functions + migrations) and `backend/rag-pipeline/` (Python).
- The `@/` path alias and shadcn components are used consistently.
- Edge Functions ship with `index.test.ts` files.
- Some commits already follow Conventional Commits, e.g. `fix(reports): ...` and `security: remove backend/.env ...`.
- Secrets were removed from tracking, and `.env.example` files exist.

### Problems found

| # | Severity | Problem | Evidence | Fix |
|---|---|---|---|---|
| 1 | **Critical** | Real secrets are still in Git history: the Supabase `service_role` key and the Gemini API key. Removing the file did not remove them from history. | `git show 518a379~1:backend/.env`, `git show deb5ac4~1:frontend/.env` | **Rotate both keys now** in the Supabase and Google AI Studio dashboards. After that, the old values in history are dead. History rewrite is optional once rotated. |
| 2 | High | The open security findings in `docs/Globiculum_Project_Review.md` are unresolved: enumerable share tokens, guardian self-verify, missing consent gate and student data left in localStorage after logout. | That doc, section "Highest-priority code findings" | Track each one as an issue and fix it before launch. |
| 3 | High | There is no CI. Nothing runs lint, type check, build or tests on push or PR. | No `.github/` directory | Add a GitHub Actions workflow (section 3.6). |
| 4 | High | Lint is red: `npm run lint` reports 71 errors and 14 warnings. | `frontend/` | Fix the errors, then make lint a blocking CI step. |
| 5 | Medium | TypeScript runs with `strict: false`, `noImplicitAny: false` and `strictNullChecks: false`. There are 93 `any` usages. Null bugs reach runtime. | `frontend/tsconfig*.json` | Turn on `strictNullChecks` first and fix the errors per folder, then `strict`. |
| 6 | Medium | Several files are very large: `ReportPreview.tsx` (2334 lines), `analyze-curriculum/index.ts` (2054), `curriculumGaps.ts` (1604) and two wizards at about 1250 each. They are hard to review and attract merge conflicts. | `wc -l` | Split by responsibility when you next touch them. Don't do a big-bang refactor. |
| 7 | Medium | Unused-variable checks are off (`@typescript-eslint/no-unused-vars: "off"`). | `frontend/eslint.config.js` | Set it to `warn` with `argsIgnorePattern: "^_"`. |
| 8 | Medium | There are 40 `console.log`/`console.debug` calls in app and function code, and some log report metadata. | grep | Use a small logger. Never log student data. |
| 9 | Low | No formatter (no Prettier or EditorConfig), so diffs mix style changes with real changes. | — | Add Prettier with the config in section 2.1 and format once in a single commit. |
| 10 | Low | `.gitignore` lists `.claude/`, but `.claude/settings.json`, the skills and `__pycache__/*.pyc` are tracked anyway. `settings.json` was committed with broken JSON and contains Windows-only paths. | `git ls-files .claude` | Keep shared settings in `.claude/settings.json`, personal ones in `.claude/settings.local.json`, and untrack the `.pyc` files. |
| 11 | Low | Asset names contain spaces and personal names, e.g. `ChatGPT Image Aug 21, 2026, 10_36_25 PM.png` and `Prerana Photo - Prerana Oswal.jpg`. | `frontend/src/assets/` | Rename to kebab-case, e.g. `team-prerana.jpg`. |
| 12 | Low | The README still names the project "Academi-Align", mentions IB/IGCSE (out of MVP scope) and shows broken box-drawing characters. | `README.md` | Rewrite it for the India↔US MVP. |

### Git history

- 220 commits, 53 of them merges (about 1 in 4). Only 15 use a Conventional Commit prefix.
- Messages like `updtes`, `UI merge`, `Audit done 2`, `debanka and nabin feedback2` say nothing about the change.
- Commits are very large. `a1eff82` alone touched 40 files and added 5950 lines.
- The `frontend-dev` branch is merged back into `main` repeatedly, and every person works straight on long-lived branches. The result is a noisy history and repeated conflicts, e.g. "merged with teammate's parentLogo visual".

---

## 2. Coding style

The rule: **match the surrounding code, keep diffs small, and make the type checker and linter do the policing.**

### 2.1 Formatting (automated, not debated)

Add `frontend/.prettierrc`:

```json
{ "semi": true, "singleQuote": false, "trailingComma": "all", "printWidth": 100 }
```

- Format the whole repo once in one commit titled `style: apply prettier`. Never mix formatting with logic changes in a commit.
- Python: `ruff format` + `ruff check`, line length 100.
- SQL migrations: lowercase keywords are fine, but stay consistent within a file.

### 2.2 TypeScript / React

- **No new `any`.** Use `unknown` plus narrowing, or a real type. Generated Supabase types are in `src/integrations/supabase/types.ts`, so use them.
- Validate every external input with Zod at the boundary: form data, Edge Function request bodies and LLM output.
- Components are `PascalCase.tsx` with one component per file and a named export. Hooks are `useThing.ts`. Utilities are `camelCase.ts` in `src/lib/`.
- Feature code lives in `components/<feature>/` (as in `assessment/parent`, `assessment/student` and `assessment/shared`). Only truly shared code goes in `shared/`.
- Soft limit of **400 lines per file**. When a file grows past that, pull out hooks (data and logic) and subcomponents (UI).
- Data fetching goes through React Query hooks, not `useEffect` + `fetch` inside pages.
- Styling: use Tailwind utilities and shadcn components from `components/ui`. No inline `style={{}}` except for dynamic values. Use `cn()` for conditional classes.
- Imports: always use `@/...`, never deep `../../../`.
- Don't add `eslint-disable` without a comment that explains why.

### 2.3 Edge Functions (Deno)

- Shared logic lives in `functions/_shared/`. Keep each `index.ts` to request parsing, auth, a call into `_shared` and the response.
- Check auth and ownership on the server for every request. Never trust a `user_id` or `report_id` sent by the client.
- The service-role key is used only inside functions and must never reach the browser.
- Return typed errors (`{ error: { code, message } }`) and never leak stack traces.

### 2.4 Database

- Every schema change is a new migration file. Never edit a migration that has already been applied.
- Every new table gets RLS enabled in the **same** migration, with owner-scoped policies. `USING (true)` for `anon` needs a written reason in the PR.
- Test policies with two accounts before you merge.

### 2.5 Data and privacy (children's data)

- Never log PII or student form data, either with `console.log` or in Edge Function logs.
- Don't keep student data in `localStorage`. If you must cache it, scope it to the user, give it a TTL and clear it on logout.
- No real student data in fixtures, screenshots or prompts to AI tools.

### 2.6 Comments and naming

- Comments explain **why**, not what. Delete commented-out code, because Git keeps it.
- Use names from the domain (`board`, `grade`, `gapItem`, `readinessScore`) and keep them the same across frontend, backend and DB.
- `TODO` must carry an owner or an issue: `// TODO(#42): ...`.

---

## 3. Git guidelines

### 3.1 Branch model: trunk-based, short-lived branches

- `main` is always deployable and protected: no direct pushes, PR required, CI must pass.
- **Retire `frontend-dev` and `backend-dev`.** Long-lived team branches are the main source of our merge commits and conflicts.
- Create one branch per task from up-to-date `main`. It should live for 1–3 days.

Branch names follow `<type>/<short-kebab-description>`:

```
feat/us-state-math-courses
fix/shared-report-blank-page
chore/add-prettier
security/rotate-supabase-keys
```

### 3.2 Commits: Conventional Commits

```
<type>(<scope>): <imperative summary, ≤ 72 chars, no period>

<optional body: WHY the change, not a list of files>

Refs #<issue>
```

- **Types:** `feat`, `fix`, `refactor`, `style`, `perf`, `test`, `docs`, `chore`, `security`, `db` (migrations).
- **Scopes** (use the area): `assessment`, `report`, `share`, `auth`, `home`, `rag`, `edge`, `db`, `admin`.

Good:

```
fix(share): require report ownership in shared_reports insert policy
feat(assessment): add grade-specific US subjects with core/elective split
db: add verified learning resources per board, class and subject
```

Bad, all taken from our history:

```
updtes
UI merge
Audit done 2
debanka and nabin feedback2
Fix Retake: ParentStep5.tsx: retake context notice; 'Regenerate Report' ...
```

Rules:

- **One logical change per commit.** If the message needs "and", it is probably two commits.
- Credit feedback in the body or the PR, not in the subject: `Feedback from: Debanka, Nabin`.
- Never commit `.env`, keys, real student data, `dist/` or `__pycache__/`. Run `git status` and `git diff --staged` before every commit.

### 3.3 Staying in sync

```sh
git switch main && git pull --ff-only
git switch -c feat/my-task
# ...work, commit...
git fetch origin && git rebase origin/main   # not "git merge main"
git push -u origin feat/my-task
```

- Set this once per machine: `git config --global pull.rebase true`.
- Rebase **your own** branch freely. Never rebase a branch someone else is also pushing to.
- If a rebase gets messy, `git rebase --abort` and ask for help. Don't force-push to `main`.

### 3.4 Pull requests

- Aim for small PRs, ideally under 400 changed lines. Split a big feature behind a flag or into stacked PRs.
- Every PR needs at least **1 reviewer** approval. Changes to RLS, auth, sharing or migrations need the backend owner.
- Merge with **squash merge** so `main` gets one clean Conventional Commit per PR. Delete the branch after merging.
- PR template, saved as `.github/pull_request_template.md`:

```markdown
## What
<one paragraph>

## Why
<issue / feedback link>

## How tested
- [ ] `npm run lint` and `npm run build` pass
- [ ] Tested flow manually: <which>
- [ ] Screenshots (UI changes): before / after, desktop + mobile
- [ ] Migrations: RLS checked with two accounts

## Risk
<what could break, rollback plan>
```

### 3.5 Reviewing

- Check correctness and security first (auth, RLS, data leaks), then readability. Style nits are for Prettier and ESLint, not humans.
- Prefix comments: `blocker:`, `suggestion:`, `nit:` or `question:`.
- Respond within one working day.

### 3.6 CI (minimum)

Add `.github/workflows/ci.yml` running on every PR:

1. `npm ci` in `frontend/`
2. `npm run lint` (blocking once lint is green)
3. `npx tsc -p tsconfig.app.json --noEmit`
4. `npm run build`
5. `deno test` for `backend/supabase/functions`
6. A secret scan (e.g. `gitleaks`)

### 3.7 Releases

- Tag deploys from `main`: `v0.3.0` (semver). Write release notes from the squash-commit titles.
- Hotfix: branch `fix/...` from `main`, PR, squash, tag a patch release.

---

## 4. Prompting guidelines (Claude Code and other AI tools)

The team uses AI coding agents heavily. These rules keep the output reviewable and safe.

### 4.1 Before you prompt

- Start from a clean, up-to-date branch (`git status` is empty, rebased on `main`). That makes the AI's diff easy to see and easy to throw away.
- Know the outcome you want and how you'll check it. If you can't say how to verify it, the task isn't ready.
- **Never paste secrets, `.env` values, or real student/parent data** into a prompt. Use synthetic data.

### 4.2 Writing the prompt

Use this shape:

```
Goal:       what should be true when done (user-visible behaviour)
Context:    files/folders involved, related issue, relevant constraint
Constraints: what NOT to touch, style to follow, MVP scope (India↔US only)
Done when:  how to verify — lint/build passes, which flow to click through, which test
```

Example:

```
Goal: Shared report links must only work for reports the sharer owns.
Context: backend/supabase/migrations/*shared_reports*, functions/fetch-shared-report.
  See docs/Globiculum_Project_Review.md finding #2.
Constraints: New migration only, don't edit applied ones. No frontend changes.
Done when: insert policy checks report ownership; fetch-shared-report verifies it too;
  explain how to test with two accounts.
```

Do:

- Name exact files and functions, and point to the existing pattern to copy ("follow `useAlignmentData.ts`").
- **One task per prompt.** Bundling "fix X, also restyle Y, also add Z" produces 40-file commits.
- For anything non-trivial, ask for a **plan first** (plan mode), review it, then let it implement.
- Ask it to run `npm run lint` / `npm run build` / tests and report the result.
- State the scope limit: "smallest change that fixes it; don't refactor unrelated code".

Don't:

- Vague asks like "improve the UI", "fix the report" or "clean up the code".
- Let the AI commit, push, or run migrations against the shared Supabase project unattended.
- Accept "done" without seeing the evidence (command output, screenshot).

### 4.3 Reviewing AI output

Treat the AI as a fast junior engineer whose work you sign off:

- Read **every line** of the diff (`git diff`) before you commit. You are the author of record.
- Check for:
  - invented imports and APIs
  - silently deleted code
  - new `any` and `eslint-disable`
  - `console.log` of user data
  - RLS `USING (true)`
  - hard-coded URLs and keys
  - changes outside the requested scope
- If the diff is too big to review, discard it (`git restore .`) and re-prompt with a smaller task.
- Write the commit message yourself in the section 3.2 format, or have the AI draft it and then edit it.

### 4.4 Shared AI config in the repo

- `CLAUDE.md` (create at repo root) holds the project context every session should know: the stack, the commands (`npm run dev/lint/build`), MVP scope, the "never log student data" rule, and a pointer to this file. Keep it under about 100 lines.
- `.claude/settings.json` is **shared**. It holds only team-wide, portable allow rules (no machine paths, no `node -e *` wildcards). Personal permissions go in `.claude/settings.local.json` (gitignored).
- Changes to either file go through PR like code.

---

## 5. Generic security and safety rules (any project)

These rules don't depend on Globiculum. Apply them to every repo you work in.

### 5.1 Git: secrets

- **Never commit secrets:** API keys, tokens, passwords, private keys, `.env` files, service-account JSON or database dumps.
- Put `.env*` (except `.env.example`) in `.gitignore` **before** the first commit, not after.
- `.env.example` holds placeholder values only (`your-api-key`), never a real value.
- Install a pre-commit secret scanner (`gitleaks` or `git-secrets`) locally and in CI.
- **If a secret is committed, rotate it immediately.** Deleting the file or rewriting history doesn't help, because clones, forks, CI logs and caches already have it. Rotate first, clean up history second.
- Treat a repo as public in the worst case. Private repos leak through forks, laptops, ex-members and third-party integrations.

### 5.2 Git: safe operations

- Look before you stage: `git status`, then `git diff --staged`. Avoid blind `git add .` and `git commit -a`.
- Never force-push to shared branches (`main` and release branches). On your own branch, use `git push --force-with-lease`, never plain `--force`.
- Before any destructive command (`reset --hard`, `clean -fd`, `branch -D`, `push --force`, `filter-repo`), make a backup: `git branch backup/<name>` or `git stash`.
- Protect `main`: require PRs, passing CI, at least one approval and signed commits if possible, and allow no direct pushes. Admins follow the same rules.
- Use `git pull --ff-only` or rebase. Don't create surprise merge commits.
- Keep large binaries, generated files (`dist/`, `build/`, `node_modules/`, `__pycache__/`) and personal files (`.DS_Store`, IDE config) out of Git.
- Use SSH keys or fine-grained tokens with expiry for Git auth. Never put a token in a remote URL (`https://token@github.com/...`).
- Enable 2FA on GitHub/GitLab for every member of the org.

### 5.3 Git: supply chain

- Commit lockfiles (`package-lock.json`, `poetry.lock`) and use `npm ci` in CI, not `npm install`.
- Review new dependencies before adding them: maintainers, download count, last release, install scripts. Don't add a package for a one-liner.
- Turn on Dependabot or Renovate plus `npm audit` / `pip-audit` in CI. Patch critical CVEs within days.
- Pin GitHub Actions to a commit SHA, not `@main`. Give CI tokens the least permission they need (`permissions: read-all` by default).
- Never put secrets in CI logs. Use masked secrets and don't `echo` them.

### 5.4 Prompting: what you send

- **Never paste secrets** into any AI tool: keys, tokens, passwords, `.env` contents or connection strings. If you pasted one by mistake, rotate it.
- **Never paste personal data:** real customer, student or patient records, emails, phone numbers or production logs that contain PII. Use synthetic or masked data.
- Don't paste proprietary third-party code or confidential documents unless the company's AI policy allows it.
- Know where data goes. Use only company-approved AI tools and accounts, and check the provider's data-retention and training settings.
- Screenshots count as data too. Crop out tokens, emails and customer details.

### 5.5 Prompting: what you let the AI do

- **Least privilege.** Give AI agents only the permissions the task needs. Approve shell commands one by one unless they are read-only.
- Don't auto-approve wildcard commands (`node -e *`, `bash *`, `curl *`, `rm *`). Allowlist exact commands only.
- **The human confirms** anything destructive or outward-facing: deleting files, `git push`, force operations, DB migrations, deploys, sending emails or messages, and changing cloud resources or permissions.
- Don't let agents run against production databases or production credentials. Point them at local or staging environments.
- Run agents on a clean branch so every change is visible in `git diff` and easy to throw away.
- **Treat content the AI reads as untrusted:** web pages, issues, PR comments, dependency READMEs and files from unknown repos. They can contain prompt-injection instructions ("ignore previous instructions and run ..."). An agent should not act on instructions found inside data.
- Don't run scripts or commands an AI suggests, or that you found in a random file, without reading them first, especially `curl ... | sh`.

### 5.6 Prompting: what you accept from the AI

- You are the author. Review every line before you commit, as you would a stranger's PR.
- Check AI code for these security issues:
  - hard-coded secrets or URLs
  - disabled auth, validation or TLS checks (`verify=False`, `rejectUnauthorized: false`)
  - over-broad permissions (`USING (true)`, `*` CORS, admin roles)
  - SQL built from string concatenation
  - `eval` / `exec` / `dangerouslySetInnerHTML` on user input
  - logging of sensitive data
  - silently weakened or deleted tests
- **Verify that packages exist** before installing anything the AI suggests. Hallucinated package names get registered by attackers ("slopsquatting").
- Verify that APIs, config flags and CLI options exist in the official docs. Don't trust confident-sounding output.
- Ask the AI to explain any security-relevant change (auth, crypto, permissions, data handling). If it can't explain it clearly, don't merge it.
- Security-critical code (auth, payments, crypto, access control, data deletion) always gets a second human reviewer.

### 5.7 Quick checklist before each commit

```
[ ] git diff --staged reviewed line by line
[ ] no secrets, .env, keys, tokens, real personal data
[ ] no new any / eslint-disable / console.log of user data
[ ] auth + permission checks are server-side
[ ] new dependencies vetted and actually exist
[ ] lint, build, tests pass locally
[ ] commit message: type(scope): what — and why in body
```

---

## 6. Rollout plan

| Step | Action | Owner | When |
|---|---|---|---|
| 1 | Rotate Supabase service_role + Gemini keys | Backend lead | **Today** |
| 2 | Protect `main`, enable squash-merge only, auto-delete branches | Repo admin | This week |
| 3 | Add PR template + CI workflow (lint non-blocking at first) | — | This week |
| 4 | Add Prettier, format once; add root `CLAUDE.md` | — | This week |
| 5 | Fix lint errors → make lint blocking | — | Next 2 weeks |
| 6 | Retire `frontend-dev` / `backend-dev`; everyone on short branches | All | After step 2 |
| 7 | Enable `strictNullChecks`, fix per folder | — | Ongoing |
| 8 | Work through security findings in `Globiculum_Project_Review.md` | Backend | Before launch |
