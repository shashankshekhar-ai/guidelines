# Globiculum Guidelines

How the Globiculum team writes code, uses Git and works with AI coding tools.

Read these before your first commit. They are short on purpose.

## The guides

| Guide | Read it when | What's inside |
|---|---|---|
| [Git Guide](GIT_GUIDE.md) | You are new to the team, or new to Git | One-time setup, keeping `main` safe, the branch → commit → Pull Request → merge workflow, fixing mistakes, cheat sheet |
| [Prompting Guide](PROMPTING_GUIDE.md) | You use Claude Code, Cursor, ChatGPT or any AI coding tool | Prompt template with examples, plan-first workflow, what the AI may and may not do, how to review AI code, safety rules |
| [Engineering Guidelines](ENGINEERING_GUIDELINES.md) | You want the full reference | Repo review, coding style for React / Supabase / database, privacy rules for student data, CI, rollout plan |

New here? Read them in this order: **Git Guide → Prompting Guide → Engineering Guidelines** (sections 2 and 5).

## The rules that matter most

### Git

1. **Never push to `main`.** `main` is protected. Every change goes through a Pull Request.
2. **One branch per task.** Create it from an up-to-date `main` and name it like `feat/us-math-courses` or `fix/report-blank-page`.
3. **Keep Pull Requests small.** Aim for less than 400 changed lines, with at least one reviewer.
4. **Pull before you start.** Run `git switch main && git pull` before you create a branch.
5. **Never commit secrets.** That covers `.env` files, API keys, tokens and real student data. If one slips in, rotate the key right away.

### Prompting AI tools

1. **Work on a branch,** never on `main`, so every AI change is easy to see and easy to throw away.
2. **One task per prompt.** Use the template: Goal / Context / Constraints / Done when.
3. **Ask for a plan first** for anything bigger than a small fix.
4. **Never paste secrets or real student or parent data** into any AI tool.
5. **Read every line before you commit.** You are the author, not the AI.

## Daily workflow at a glance

```
main ──●───────────────────────────●──▶  (protected, always deployable)
        \                         /
         └─ feat/my-task ─●─●─●──┘
            1. branch   2. commit   3. push + open PR   4. review   5. squash merge
```

```sh
git switch main && git pull
git switch -c feat/my-task
# ...make changes...
git add <files>
git commit -m "feat(report): add grade filter"
git push -u origin feat/my-task
# open the Pull Request on GitHub, get a review, squash merge, delete the branch
```

The [Git Guide](GIT_GUIDE.md) walks through every step.

## Changing these guidelines

These documents follow their own rules. To change one, create a branch, edit it and open a Pull Request. Anyone on the team can propose a change.
