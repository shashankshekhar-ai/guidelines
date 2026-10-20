# Globiculum Git Guide

**Who this is for:** everyone who writes code for Globiculum (React + Vite frontend, Supabase backend), including people who are new to Git.

**How to use it:** do [section 2](#2-one-time-setup-every-team-member) once on your computer. The repo owner does [section 3](#3-repo-admin-setup-owner-only-once) once. After that, follow [section 4](#4-daily-workflow) for every task and keep the [cheat sheet](#8-cheat-sheet) open. When something goes wrong, read [section 7](#7-oops-fixing-common-mistakes) before you try anything else.

## Contents

1. [The 5 golden rules](#1-the-5-golden-rules)
2. [One-time setup (every team member)](#2-one-time-setup-every-team-member)
3. [Repo admin setup (owner only, once)](#3-repo-admin-setup-owner-only-once)
4. [Daily workflow](#4-daily-workflow)
5. [Keeping your branch up to date](#5-keeping-your-branch-up-to-date)
6. [Reviewing a pull request](#6-reviewing-a-pull-request)
7. [Oops: fixing common mistakes](#7-oops-fixing-common-mistakes)
8. [Cheat sheet](#8-cheat-sheet)

## Words you will see

- **Repo** (repository): the project folder that Git tracks. Ours lives on GitHub.
- **Branch**: your own line of work, separate from everyone else's.
- **`main`**: the main branch. It holds the code we ship. Nobody changes it directly.
- **Commit**: a saved snapshot of your changes, with a short message.
- **Push / pull**: send your commits to GitHub / get the latest commits from GitHub.
- **Pull Request (PR)**: a request to merge your branch into `main`. GitLab calls this a Merge Request (MR). On GitHub it is called a Pull Request, so this guide says PR.

## How work flows

```
main     ──●───────────────────────────●──────▶   (always safe, protected)
           │                           ▲
           │ 1. create branch          │ 5. squash merge
           ▼                           │
feat/x     ●──●──●  ──▶  PR  ──▶  review + approve
           2. commit  3. push  4. open PR
```

---

## 1. The 5 golden rules

1. **Never push to `main`.** Every change goes through a PR.
2. **One branch per task.** New task, new branch.
3. **Keep PRs small.** Small PRs are easier to review and hide fewer bugs. Aim for under 400 changed lines.
4. **Pull before you start.** Always start from the latest `main`.
5. **Never commit secrets.** No `.env` files, keys, passwords or real student data.

---

## 2. One-time setup (every team member)

### 2.1 Install Git

Install Git for your system, then check that it works.

- macOS: run `xcode-select --install` (or `brew install git` if you use Homebrew).
- Windows: download it from https://git-scm.com and accept the defaults.

```sh
git --version
# Expect something like: git version 2.45.0
```

### 2.2 Tell Git who you are

Git puts your name and email on every commit, so use the email of your GitHub account.

```sh
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

### 2.3 Keep history clean

These settings make `git pull` avoid messy "Merge branch..." commits and name new repos' first branch `main`.

```sh
git config --global pull.rebase true
git config --global init.defaultBranch main
```

### 2.4 Connect to GitHub with an SSH key

An SSH key lets you push to GitHub without typing a password. Create one and print it (press Enter for every question).

```sh
ssh-keygen -t ed25519 -C "you@example.com"
cat ~/.ssh/id_ed25519.pub
```

Copy the printed line. On GitHub, click your photo → **Settings** → **SSH and GPG keys** → **New SSH key**, paste it and click **Add SSH key**. Then test the connection (type `yes` if asked about the host).

```sh
ssh -T git@github.com
# Expect: Hi <your-username>! You've successfully authenticated, but GitHub does not provide shell access.
```

Also turn on two-factor authentication (2FA): GitHub → **Settings** → **Password and authentication**.

### 2.5 Clone the repo

Cloning downloads the repo to your computer. On the repo page, click **Code** → **SSH**, copy the URL and use it below.

```sh
git clone git@github.com:<org>/<repo>.git
cd <repo>
```

---

## 3. Repo admin setup (owner only, once)

### 3.1 Protect main

This stops anyone (admins included) from pushing to `main`, force-pushing or deleting it. On the repo page, go to **Settings** → **Rules** → **Rulesets** → **New ruleset** → **New branch ruleset**, then:

1. **Ruleset name:** `protect-main`.
2. **Enforcement status:** `Active`.
3. **Bypass list:** leave it empty, so admins follow the rules too.
4. **Target branches:** **Add target** → **Include default branch**.
5. Tick these rules:
   - **Restrict deletions**: nobody can delete `main`.
   - **Block force pushes**: nobody can rewrite the history of `main`.
   - **Require a pull request before merging**: set **Required approvals** to `1` and tick **Dismiss stale pull request approvals when new commits are pushed**. Under **Allowed merge methods**, keep only **Squash**.
6. Click **Create**.

On older screens, use **Settings** → **Branches** → **Add branch protection rule**, enter `main` and tick the same options.

> Note: on **private** repos, rulesets and branch protection need a paid plan (GitHub Pro or Team). If you cannot turn them on, the team must follow golden rule 1 by hand.

### 3.2 Allow squash merge only

Squash merge turns all commits of a PR into one clean commit on `main`. Go to **Settings** → **General** → **Pull Requests** and:

- Untick **Allow merge commits**.
- Keep **Allow squash merging** ticked and set its default message to **Pull request title**.
- Untick **Allow rebase merging**.
- Tick **Automatically delete head branches**, so GitHub deletes a branch after its PR is merged.

### 3.3 Add team members

Go to **Settings** → **Collaborators** (or **Collaborators and teams**) → **Add people**, type the GitHub username and choose the **Write** role. Only owners get **Admin**.

---

## 4. Daily workflow

### 4.1 Update your main

Switch to `main` and download the latest changes, so you start from fresh code.

```sh
git switch main
git pull
```

### 4.2 Create a branch

Name your branch `type/short-description`, in lowercase with words joined by `-` (this style is called kebab-case).

| Type | Use for | Example |
|---|---|---|
| `feat/` | a new feature | `feat/us-state-math-courses` |
| `fix/` | a bug fix | `fix/shared-report-blank-page` |
| `docs/` | documentation only | `docs/update-readme-setup` |
| `chore/` | tooling, config, cleanup | `chore/add-prettier` |
| `security/` | security fixes, key rotation | `security/rotate-supabase-keys` |

Create the branch and move onto it in one command.

```sh
git switch -c fix/shared-report-blank-page
# Expect: Switched to a new branch 'fix/shared-report-blank-page'
```

### 4.3 Make your changes

Edit the code, run the app and test your change in the browser.

### 4.4 Check what changed

Look at which files changed and at the exact lines, before you save anything.

```sh
git status
git diff
```

Make sure there is no `.env`, `node_modules/`, `dist/` or any file you did not mean to touch.

### 4.5 Stage the files you want

Staging picks which changes go into the next commit. Name the files one by one instead of using `git add .`, then check the final result.

```sh
git add frontend/src/pages/Report.tsx frontend/src/lib/report.ts
git diff --staged
```

### 4.6 Commit

A commit message has the form `type(area): what you did`: lowercase, short, no full stop, written as an order ("add", not "added").

```sh
git commit -m "fix(report): show an error instead of a blank page"
```

Good messages:

```
feat(assessment): add grade-specific US subjects
fix(share): let only the report owner create a share link
docs: explain local Supabase setup in README
```

Bad messages:

```
updtes                                          # says nothing
UI merge                                        # what changed, and why?
fixed stuff and added login and cleaned css     # three changes: make three commits
```

You can commit many times on your branch. They become one commit when the PR is squash merged.

### 4.7 Push your branch

Pushing uploads your branch to GitHub. The first push needs `-u` to link your branch to GitHub; after that, plain `git push` is enough.

```sh
git push -u origin fix/shared-report-blank-page
git push
```

### 4.8 Open a pull request

On GitHub, click the yellow **Compare & pull request** button (or go to **Pull requests** → **New pull request**). Set **base** to `main` and **compare** to your branch.

- **Title:** write it like a commit message, because it becomes the commit on `main`. Example: `fix(report): show an error instead of a blank page`.
- **Description:** say what, why and how you tested it.

```markdown
## What
Show a friendly error instead of a blank page when a report is missing.

## Why
Parents saw a blank screen on old share links. Fixes #42.

## How tested
- `npm run lint` and `npm run build` pass
- Opened a deleted report link: the error message shows
- Screenshots attached (desktop and mobile)
```

- **Reviewers** (right side): pick one teammate. Changes to login, sharing, the database or security rules need the backend owner.

### 4.9 Respond to review

Make the requested fixes on the same branch and push again. The PR updates by itself, so do not open a new one.

```sh
git add frontend/src/pages/Report.tsx
git commit -m "fix(report): handle a missing report id"
git push
```

Reply to each comment with "done" or a short reason why not.

### 4.10 Squash and merge

When the PR is approved, click **Squash and merge**, check the title and click **Confirm squash and merge**.

### 4.11 Clean up

Go back to `main`, download the merged work and delete your local branch. Use `-D` (capital D) here: after a squash merge, Git cannot tell the branch was merged, so `-d` refuses. This is safe because your work is now in `main`.

```sh
git switch main
git pull
git branch -D fix/shared-report-blank-page
```

Next task? Go back to [4.1](#41-update-your-main).

---

## 5. Keeping your branch up to date

### 5.1 Rebase (normal way)

Others merge into `main` while you work. Rebase replays your commits on top of the latest `main`; do this before opening a PR or when GitHub says the branch is out of date. A rebase rewrites your branch, so the push needs `--force-with-lease` (only ever on your own branch).

```sh
git switch fix/shared-report-blank-page
git fetch origin
git rebase origin/main
git push --force-with-lease
```

### 5.2 Merge (simple fallback)

If rebase feels scary, merge `main` into your branch instead. It needs no force push, and the extra merge commit disappears when the PR is squashed.

```sh
git switch fix/shared-report-blank-page
git pull --no-rebase origin main
git push
```

### 5.3 Resolve a conflict

A conflict happens when you and someone else changed the same lines. Git stops and prints `CONFLICT (content): Merge conflict in <file>`.

1. Run `git status` to list the files marked **both modified**.
2. Open each file and find the conflict markers:

   ```
   <<<<<<< (one side)
   code from one side
   =======
   code from the other side
   >>>>>>> (other side)
   ```

3. Edit the block into the correct final code and delete the three marker lines. VS Code's **Accept Current / Incoming / Both** buttons can help.
4. Save the file and mark it as fixed with `git add <file>`.
5. Continue: run `git rebase --continue` if you were rebasing, or `git commit` if you were merging. Repeat steps 1 to 4 if Git stops again.
6. Run the app to check it still works, then push.

### 5.4 Cancel and ask for help

If you get stuck, cancel the rebase or merge to go back to exactly where you were, then ask a teammate.

```sh
git rebase --abort
git merge --abort
```

Run only the one that matches what you were doing.

---

## 6. Reviewing a pull request

Open the PR and go to the **Files changed** tab. Check:

- [ ] I understand what it does and why (the description is filled in).
- [ ] It does one thing and is not too big.
- [ ] The diff has no secrets, `.env` files, keys or real student data.
- [ ] Login, sharing and database changes keep data private: only the right user can see it.
- [ ] Lint and build pass (green checks, once CI is set up).
- [ ] UI changes include screenshots.
- [ ] Names and code are easy to read.

Start each comment with `blocker:` (must fix), `suggestion:`, `question:` or `nit:` (tiny and optional). Finish with **Review changes** → **Approve** or **Request changes**. Try to review within one working day.

---

## 7. Oops: fixing common mistakes

Before anything risky, save a backup branch. It costs nothing and can be deleted later.

```sh
git branch backup/my-work
```

### 7.1 I committed to main by mistake (not pushed)

Create a new branch that keeps your commits, then move your local `main` back to match GitHub.

```sh
git switch -c fix/my-work
git fetch origin
git branch -f main origin/main
```

Now push `fix/my-work` and open a PR as usual. (If `main` is protected, GitHub would have rejected a push anyway.)

### 7.2 Wrong message on my last commit

Rewrite the message of the last commit. If you already pushed it, update your own branch with `--force-with-lease`.

```sh
git commit --amend -m "fix(report): correct message here"
git push --force-with-lease
```

### 7.3 Undo my last commit but keep the changes

This removes the last commit and keeps your changes staged, ready to commit again.

```sh
git reset --soft HEAD~1
```

To unstage a file but keep its changes, run `git restore --staged <file>`.

### 7.4 I committed a secret

This covers API keys, the Supabase `service_role` key, passwords and `.env` files.

1. **Rotate the key first.** Create a new key in the service (for example Supabase dashboard → **Project Settings** → **API**) and disable the old one. Deleting the file does not help: the key is already in history and on other people's computers.
2. Tell the repo owner right away.
3. Remove the secret from the code, keep it in `.env` and make sure `.env` is listed in `.gitignore`.
4. If you have not pushed yet, undo the commit, unstage the secret file and commit again without it:

   ```sh
   git reset --soft HEAD~1
   git restore --staged .env
   git commit -m "fix(config): read api key from env"
   ```

5. If you already pushed, rotation is the real fix. Ask the owner before any history cleanup.

### 7.5 My branch is a mess: start fresh

Save the old branch, start a new one from the latest `main` and copy back only the files you want.

```sh
git branch backup/my-task
git switch main
git pull
git switch -c feat/my-task-v2
git restore --source=backup/my-task -- frontend/src/pages/Report.tsx
```

### 7.6 Force push rules

Never force-push `main` or a branch other people push to. On your own branch, always use `git push --force-with-lease`, never plain `--force`: it refuses if someone else pushed in the meantime, so you cannot wipe their work.

---

## 8. Cheat sheet

| I want to... | Command |
|---|---|
| Get the latest `main` | `git switch main` then `git pull` |
| Start a task | `git switch -c feat/short-name` |
| See what changed | `git status` and `git diff` |
| Stage a file | `git add <file>` |
| Check staged changes | `git diff --staged` |
| Commit | `git commit -m "type(area): what you did"` |
| Push the first time | `git push -u origin <branch>` |
| Push again | `git push` |
| Update my branch from `main` | `git fetch origin` then `git rebase origin/main` |
| Push after a rebase (own branch) | `git push --force-with-lease` |
| Cancel a rebase | `git rebase --abort` |
| Fix the last commit message | `git commit --amend -m "new message"` |
| Undo the last commit, keep changes | `git reset --soft HEAD~1` |
| Unstage a file | `git restore --staged <file>` |
| Save a backup | `git branch backup/<name>` |
| List my branches | `git branch` |
| Delete a branch after its PR is merged | `git branch -D <branch>` |
| See recent history | `git log --oneline -10` |
