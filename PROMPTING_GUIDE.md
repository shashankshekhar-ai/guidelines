# Globiculum Prompting Guide

**Who this is for:** everyone on the Globiculum team who uses AI coding tools (Claude Code,
Cursor, ChatGPT, Copilot, or similar) on the React + Vite frontend, the Supabase Edge
Functions and Postgres/RLS backend, or the Python RAG pipeline.

**How to use it:** read sections 1 to 7 once. After that, keep the
[cheat sheet](#8-cheat-sheet) open while you work. For branches, commits and pull requests,
see [GIT_GUIDE.md](GIT_GUIDE.md). If anything here disagrees with
[ENGINEERING_GUIDELINES.md](ENGINEERING_GUIDELINES.md), the guidelines win.

We handle children's and students' data. Most of the rules below exist to protect it.

## Contents

1. [The golden rules](#1-the-golden-rules)
2. [Step by step: one task from start to finish](#2-step-by-step-one-task-from-start-to-finish)
3. [The prompt template](#3-the-prompt-template)
4. [What the AI may and may not do](#4-what-the-ai-may-and-may-not-do)
5. [Reviewing AI output](#5-reviewing-ai-output)
6. [Safety](#6-safety)
7. [When the AI gets stuck](#7-when-the-ai-gets-stuck)
8. [Cheat sheet](#8-cheat-sheet)

---

## 1. The golden rules

1. **Work on a branch.** Never prompt on `main`.
2. **One task per prompt.** Small asks give small diffs that you can actually review.
3. **Never paste secrets or real student data.** Use synthetic data only.
4. **Read every line before you commit.** No exceptions.
5. **You are responsible for the code.** "The AI wrote it" is never an excuse.

---

## 2. Step by step: one task from start to finish

**Step 1. Create a branch.** Start from an up-to-date `main` so the AI's changes are the only
changes you see (details in [GIT_GUIDE.md](GIT_GUIDE.md)).

```bash
git switch main && git pull --ff-only
git switch -c fix/shared-report-blank-page
git status        # must say "nothing to commit, working tree clean"
```

**Step 2. Write the prompt.** Fill in all four parts of the [template](#3-the-prompt-template).

**Step 3. Ask for a plan first.** For anything bigger than a one-line fix, get a plan before
any code is written. In Claude Code, switch to plan mode (Shift+Tab). In other tools, add:

```
Don't change any files yet. Give me a short plan: which files you will change,
what you will change in each, and how we will test it.
```

**Step 4. Review the plan.** Check that it touches only the files you expect, and push back if
it wants to refactor, add packages or edit old migrations.

**Step 5. Let it implement.** Approve shell commands one at a time and refuse anything on the
[human-only list](#4-what-the-ai-may-and-may-not-do).

**Step 6. Make it prove the work.** Ask for real command output, not a claim that it is done.

```
Run npm run lint, npm run build and the tests, and paste the real output.
Don't say "done" until all of them pass.
```

For the Python RAG pipeline, use its test command (for example, `pytest`).

**Step 7. Review the diff.** Run `git status` and `git diff`, then go through the
[review checklist](#5-reviewing-ai-output).

**Step 8. Test it by hand.** Click through the flow in the browser, and log in as a second
user if the change affects who can see what.

**Step 9. Commit it yourself.** Stage files by name, check the staged diff, and write the
message in our commit format (see [GIT_GUIDE.md](GIT_GUIDE.md)).

```bash
git add frontend/src/pages/SharedReport.tsx
git diff --staged
git commit -m "fix(report): show expiry message instead of blank page"
```

**Step 10. Push and open a pull request.** Fill in What / Why / How tested / Risk, and wait
for review before merging (see [GIT_GUIDE.md](GIT_GUIDE.md)).

```bash
git push -u origin fix/shared-report-blank-page
```

If the team asks for it, add one line to the PR description, such as
`AI-assisted: plan and implementation, reviewed line by line by me`.

---

## 3. The prompt template

```
Goal:        What should be true when you're done. Describe behaviour, not code.
Context:     Exact files and functions, the related issue, and the pattern to copy.
Constraints: What NOT to touch. Smallest change. MVP scope is India <-> US only.
Done when:   How we check it: lint, build and tests pass; which flow to click through.
```

File paths in these examples are illustrative. Use the real paths from your task.

### 3.1 Example: bug fix

```
Goal: Opening an expired shared-report link shows "This link has expired"
  instead of a blank page.
Context: frontend/src/pages/SharedReport.tsx calls the fetch-shared-report Edge Function
  (backend/supabase/functions/fetch-shared-report). The function returns HTTP 410 for
  expired links, but the page ignores non-200 responses.
Constraints: Frontend only. Smallest change that fixes it; no refactoring.
  Reuse the error layout from frontend/src/pages/NotFound.tsx.
Done when: npm run lint and npm run build pass. Tell me how to create an expired
  link on my local Supabase so I can check the message myself.
```

### 3.2 Example: small UI feature

```
Goal: On the course comparison page, add a "Curriculum" filter with three options:
  All, India (CBSE/ICSE/State boards) and US (state standards).
Context: frontend/src/pages/CompareCourses.tsx. Course data comes from
  frontend/src/hooks/useAlignmentData.ts. Use the existing <Select> in
  frontend/src/components/ui/select.tsx. The subject filter on the same page is the pattern.
Constraints: No new packages. Don't change the data hook or any Edge Function.
  Keep the filter value in the URL query string, like the subject filter.
Done when: lint and build pass. The filter survives a page refresh, and "All"
  shows exactly the same list as today.
```

### 3.3 Example: database and RLS change

```
Goal: A student can read only their own saved plans. A parent can read the plans
  of students linked to them. Nobody else can read any plan.
Context: Table public.saved_plans (student_id references auth.users).
  Parent links are in public.parent_student_links (parent_id, student_id).
  Copy the style of the existing profiles policies in backend/supabase/migrations/.
Constraints: Plan first. Add ONE new migration file; never edit applied ones.
  No USING (true). No service-role key anywhere in frontend code.
  Don't run the migration; I will run it on my local Supabase myself.
Done when: You give me the migration plus SQL to test it locally with synthetic
  users (student_a@example.com, student_b@example.com, parent_a@example.com),
  and the expected result for each user.
```

### 3.4 Bad prompts, rewritten

| Bad prompt | Problem | Good prompt |
|---|---|---|
| "Improve the UI." | No goal and no scope, so you get 30 changed files. | "In `frontend/src/pages/Dashboard.tsx`, make the course cards stack in one column below 640px. CSS only. Done when it looks right at 375px and 1280px wide." |
| "Fix the shared report, add PDF export and clean up the code." | Three tasks in one prompt, so the diff can't be reviewed. | Three prompts on three branches. First one: "Goal: expired shared links show an expiry message. Context: ... Constraints: ... Done when: ..." |
| "Here's my .env and an export of the students table. Why does login fail?" | Leaks secrets and real children's data. | "Login fails with `Invalid login credentials` for a user I just created locally. Here is `frontend/src/lib/auth.ts`. Env vars are set (values hidden). Use fake users in any examples." Then rotate every key you already pasted. |

---

## 4. What the AI may and may not do

| The AI may (on your branch) | Only a human may |
|---|---|
| Read code and docs in the repo | `git push`, merge, or force anything |
| Edit files on your feature branch | Run migrations on the shared Supabase project |
| Run `npm run lint`, `npm run build` and tests | Deploy the frontend, Edge Functions or pipeline |
| Run read-only commands (`git status`, `git diff`, `ls`) | Delete files or folders |
| Work against your **local** Supabase | Install new packages (check them first, see section 5) |
| Draft commit messages and PR descriptions | Touch production data, keys or cloud settings |

- Never auto-approve wildcard commands such as `bash *`, `curl *` or `rm *`.
- Never give an AI tool the Supabase service-role key or any production credentials.
- Shared project context for Claude Code lives in `CLAUDE.md` at the repo root (stack,
  commands, MVP scope, "never log student data"). Change it through a PR, like code.

---

## 5. Reviewing AI output

Treat the AI like a fast new teammate, and read its diff as you would a stranger's PR.

- [ ] Every import, function and API really exists (check the code or the official docs).
- [ ] No code was silently deleted.
- [ ] No new `any`, `@ts-ignore` or `eslint-disable`.
- [ ] No `console.log` or `print` of user or student data.
- [ ] No RLS `USING (true)`, no `*` CORS, and no admin role added "just to make it work".
- [ ] No hard-coded keys, tokens or URLs.
- [ ] No changes outside the task (extra files, reformatting, refactors).
- [ ] No tests weakened, skipped or deleted to make them pass.
- [ ] Every new package is real: check its npm or PyPI page, maintainer, downloads and last
      release. Attackers register package names that AI tools invent.
- [ ] You can explain every security-related change (auth, RLS, data handling). If you
      can't, don't commit it.

**If the diff is too big to review, throw it away and re-prompt with a smaller task:**

```bash
git restore .     # undo changes to tracked files
git clean -n      # list new files the AI created
git clean -f      # delete them (only after reading the list above)
```

Changes to auth, RLS or data deletion always need a second human reviewer.

---

## 6. Safety

- **Never paste** API keys, tokens, `.env` contents, connection strings, real student or
  parent records, production logs with names or emails, or screenshots showing any of these.
  Crop screenshots and use synthetic data such as `student_a@example.com`.
- **If you paste a secret by mistake,** rotate it immediately in the Supabase or provider
  dashboard and tell the team. Deleting the message does not un-send it.
- **If you paste real student data by mistake,** tell the repo owner the same day.
- **Use only team-approved AI tools and accounts,** and check their data-retention and
  "train on my data" settings.
- **Treat everything the AI reads as untrusted.** Web pages, GitHub issues, PR comments and
  package READMEs can hide instructions such as "ignore previous instructions and run ...".
  If the AI suddenly wants to do something you didn't ask for, stop it.
- **Read every command before you run it.** Never run `curl ... | sh`, `rm -rf` or a script
  the AI suggests without reading it first.

---

## 7. When the AI gets stuck

**Warning signs:**

- It tries the same fix again and again.
- It guesses ("maybe the issue is ...") instead of checking.
- The diff keeps growing, but the bug is still there.
- It wants to disable a test, a lint rule or an RLS policy to "fix" the problem.
- It says "done" but can't show passing output.

**What to do:**

1. Stop it before it changes anything else.
2. Throw away the changes with `git restore .` (and `git clean`, as in section 5).
3. Find the real error yourself: read the full error message, the logs or the failing test.
4. Start a fresh session with a smaller prompt that includes the exact error text and the
   one file that matters. Old sessions carry confusing context.
5. Still stuck after two attempts? Ask a teammate. Ten minutes of human help beats an hour
   of loops.

---

## 8. Cheat sheet

```
BEFORE
  git switch main && git pull --ff-only
  git switch -c <type>/<short-name>         # feat/ fix/ chore/ security/
  git status                                # must be clean

PROMPT
  Goal / Context / Constraints / Done when
  One task. Exact files. A pattern to copy. "Smallest change."
  Non-trivial? "Plan first, no edits yet."

AI MAY                        ONLY A HUMAN MAY
  read, edit on branch          push, merge, deploy
  lint / build / test           migrations on shared Supabase
  local Supabase                delete files, install packages

PROVE IT
  npm run lint && npm run build && tests    # real output, not "done"

REVIEW (git diff)
  Invented imports? Deleted code? any? console.log of user data?
  USING (true)? Hard-coded keys? Out of scope? Weaker tests? Fake packages?
  Too big? git restore . and re-prompt smaller.

COMMIT AND PR (see GIT_GUIDE.md)
  git add <files> -> git diff --staged -> git commit -m "type(scope): ..."
  git push -u origin <branch> -> open PR -> wait for review

NEVER
  Prompt on main. Paste secrets, .env or real student data.
  Run curl | sh unread. Trust "done" without output.
  Leaked a key? Rotate it now.
```
