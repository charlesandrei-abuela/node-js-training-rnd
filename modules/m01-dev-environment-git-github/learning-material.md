# M01 Learning Material — Dev Environment, Git & GitHub

> **Companion to `brief.md` and `tasks.md`.** The brief tells you *what* to deliver; this document
> teaches the *concepts* you need to get there. It deliberately does **not** contain the finished
> lab: you still create the repo, resolve your own conflict, and write your own `revert` vs `reset`
> explanation. Read it, try the drills, then do the lab.

**Week 1 · Day 1 · ~4 hours**

## 0. How to use this material

| Section | Maps to brief Scope bullet | Maps to Definition of done | Suggested time |
| --- | --- | --- | --- |
| 1. Toolchain | Toolchain | — | 30 min |
| 2. The Git mental model | Git mental model | `revert` vs `reset` narration | 45 min |
| 3. Branching, merging, conflicts | Branching and merging | Merged PR (conflict in history) | 45 min |
| 4. `.gitignore` and secrets | `.gitignore` conventions | `.gitignore`, `.env.example` | 20 min |
| 5. GitHub and collaboration | GitHub | Merged PR, 3+ Conventional Commits | 45 min |
| 6. Guided practice | all | — | 45 min |
| 7. Knowledge check | all | — | 20 min |

Suggested flow: read sections 1–5, do the drills in 6, take the check in 7, then start the lab in
`brief.md`.

**Learning outcomes.** After this module you can:

- Install and verify a Node toolchain pinned by `.nvmrc`.
- Explain working tree → staging → commit and move changes between them.
- Create, merge and resolve conflicts between branches.
- Keep secrets and generated files out of a repo.
- Take a change through a GitHub pull request using Conventional Commits.

---

## 1. Toolchain

### 1.1 What you are installing and why

| Tool | Why you need it |
| --- | --- |
| **VS Code** | The editor used throughout the program; built-in terminal, Git UI, debugger. |
| **Git** | Version control. Everything in this program is delivered through Git history. |
| **Node.js via a version manager** | Lets you switch Node versions per project instead of one global install. |
| **Bruno** | A plain-text API client. Collections live in your repo (used from M05). |
| **A terminal** | Git, Node and npm are driven from the command line. |

### 1.2 Node through a version manager

Installing Node directly from an installer pins your whole machine to one version. A version manager
installs several side by side and switches per project.

| OS | Common choices |
| --- | --- |
| Windows | `nvm-windows`, `fnm`, or `volta` |
| macOS / Linux | `nvm`, `fnm`, or `volta` |

The cohort is pinned to **Node.js 24 LTS**. Your trainer re-verifies the exact version immediately
before the cohort starts, so confirm with them.

### 1.3 `.nvmrc`

A `.nvmrc` file at the repo root contains the Node version the project expects:

```
24
```

Version managers read it so anyone cloning the repo uses the same major version. It is a one-line
file; the value is that it is **committed**.

### 1.4 Terminal basics you will use constantly

| Task | macOS / Linux / Git Bash | PowerShell |
| --- | --- | --- |
| Where am I? | `pwd` | `Get-Location` (`pwd`) |
| List files | `ls -la` | `Get-ChildItem` (`ls`) |
| Change directory | `cd folder` | `cd folder` |
| Make a directory | `mkdir folder` | `mkdir folder` |
| Print a file | `cat file` | `Get-Content file` (`cat`) |
| Clear screen | `clear` | `clear` |

Habits worth building: use Tab to autocomplete, Up-arrow to recall commands, and read the full
error message before searching for it.

### 1.5 Verify your setup

```bash
git --version        # any recent 2.x
node --version       # v24.x
npm --version
code --version       # VS Code on PATH (optional)
```

Configure your Git identity once:

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

### 1.6 Heads-up: native build tools

Later modules install `better-sqlite3`, which compiles native code on some machines. Installing
compiler build tools now (Windows: "Desktop development with C++" build tools / Visual Studio Build
Tools; macOS: `xcode-select --install`; Linux: `build-essential`) avoids a blocked Module 7. Ask
your trainer whether to do this today.

---

## 2. The Git mental model

### 2.1 Three areas, one history

Git moves your work through three places:

```
 Working tree  ──git add──▶  Staging area (index)  ──git commit──▶  Repository (history)
 (files on disk)             (what the next commit                  (permanent snapshots)
                              will contain)
        ◀──────── git restore ────────┘◀──── git restore --staged ────┘
```

- **Working tree** — the files you actually edit.
- **Staging area (index)** — a *draft* of your next commit. You choose exactly which changes go in.
- **Repository** — the chain of commits. Each commit is a snapshot with a message, an author and a
  pointer to its parent.

A commit is not a "diff"; it is a snapshot of the whole project at that moment (Git stores it
efficiently). Branches and `HEAD` are just labels pointing at commits.

### 2.2 Everyday commands

| Command | What it does |
| --- | --- |
| `git init` | Turn the current folder into a repo. |
| `git clone <url>` | Copy a remote repo (with history) to your machine. |
| `git status` | Show what is modified, staged and untracked. Run it constantly. |
| `git add <path>` | Stage changes. `git add -p` stages chunk by chunk. |
| `git commit -m "msg"` | Record the staged snapshot. |
| `git log --oneline --graph` | Compact history view. |
| `git diff` | Working tree vs staging. |
| `git diff --staged` | Staging vs last commit. |
| `git restore <file>` | Discard working-tree changes to a file. |
| `git restore --staged <file>` | Unstage a file, keeping your edits. |
| `git stash` / `git stash pop` | Shelve uncommitted work temporarily and bring it back. |

**Good commits** are small, do one thing, and have a message that says *what and why*.

### 2.3 Undoing things

There are several "undo" tools because there are several situations. The key question is always:
**has this commit been shared?**

| Situation | Tool |
| --- | --- |
| Edited a file, want the old version back | `git restore <file>` |
| Staged a file by mistake | `git restore --staged <file>` |
| Need to set work aside for a minute | `git stash` |
| A commit is wrong and **already pushed/shared** | `git revert <commit>` |
| A commit is wrong and **only exists locally** | `git reset` |

Concepts to understand (and then explain in your own words for the lab):

- `git revert` adds a *new* commit that applies the inverse of an earlier commit. History is
  preserved and moves forward.
- `git reset` moves the current branch pointer to a different commit. Depending on the mode it also
  changes the staging area and working tree:

  | Mode | Branch pointer | Staging area | Working tree |
  | --- | --- | --- | --- |
  | `--soft` | moves | unchanged | unchanged |
  | `--mixed` (default) | moves | reset | unchanged |
  | `--hard` | moves | reset | **reset (changes lost)** |

- Think about *why* rewriting shared history causes problems for teammates, and which tool is safe
  on a shared branch.

> Safety net: `git reflog` lists where `HEAD` has been, so even a bad `reset --hard` is usually
> recoverable for a while.

---

## 3. Branching, merging and conflicts

### 3.1 Branches

A branch is a movable label on a commit. Creating one is nearly free, so use one per piece of work.

```bash
git switch -c feat/hello-node     # create and switch
git switch main                   # switch back
git branch                        # list branches
git branch -d feat/hello-node     # delete after merge
```

Why not commit directly to `main`? `main` should always be in a known-good state; branches let you
work, review and discard safely.

### 3.2 Merging

```bash
git switch main
git merge feat/hello-node
```

Two outcomes:

- **Fast-forward** — `main` has no new commits since you branched, so Git just moves the label
  forward. No merge commit.
- **Merge commit** — both branches moved; Git creates a commit with two parents joining them.

### 3.3 Conflicts

A conflict happens when both branches changed the *same lines* (or one deleted what the other
edited). Git cannot decide, so it stops and marks the file:

```
<<<<<<< HEAD
console.log("Hello from main");
=======
console.log("Hello from the feature branch");
>>>>>>> feat/hello-node
```

- Between `<<<<<<<` and `=======` is **your current branch** (`HEAD`).
- Between `=======` and `>>>>>>>` is **the branch being merged in**.

General approach to resolving (you choose the actual resolution):

1. `git status` — see which files are "both modified".
2. Open each file, decide what the final content should be (either side, both, or something new),
   and remove all three marker lines.
3. Run/test the result — a conflict-free file can still be wrong.
4. `git add <file>` to mark it resolved.
5. `git commit` (or `git merge --continue`) to finish.

`git merge --abort` returns you to before the merge if you want to start over.

### 3.4 Merge vs rebase (conceptual)

| | Merge | Rebase |
| --- | --- | --- |
| What it does | Joins histories with a merge commit | Replays your commits on top of another branch |
| History shape | Preserves true history, branches visible | Linear, cleaner-looking |
| Rewrites commits? | No | Yes (new commit IDs) |
| Risk | Noisy history | Dangerous on shared branches |

At this stage you only need the idea. The rule of thumb to remember: **do not rebase commits other
people already have.**

---

## 4. `.gitignore` and secrets

### 4.1 What `.gitignore` does

It lists patterns of files Git should not track. It only affects **untracked** files: if a file is
already committed, adding it to `.gitignore` does not remove it (see drill 5).

### 4.2 What belongs in a Node project's `.gitignore`

| Entry | Why |
| --- | --- |
| `node_modules/` | Installed dependencies are reproducible from `package.json`; huge. |
| `.env` | Local configuration and **secrets**. |
| `*.db` | Local SQLite database files (used from M06). |
| Generated Prisma client | Output of `prisma generate`; rebuilt, not committed (used from M07). |
| `dist/` | Compiled build output (TypeScript from M04). |
| `coverage/` | Test coverage reports (M09). |

Useful pattern syntax: `*.log` (wildcard), `dir/` (directory), `!keep.txt` (negate), `#` comments.

### 4.3 Never commit secrets

Commit a `.env.example` instead: the same keys as `.env` with **placeholder** values and no real
secrets. It documents what a new developer must configure.

```
# .env.example
PORT=3000
DATABASE_URL=file:./dev.db
API_KEY=replace-me
```

Even when there is nothing to configure yet, create the file: the point is the habit.

### 4.4 If you commit a secret anyway

1. Treat the secret as **compromised** and rotate/revoke it immediately.
2. Removing it in a later commit is not enough; it remains in history.
3. History-rewriting tools exist but rotating the credential is the actual fix.

---

## 5. GitHub and collaboration

### 5.1 Remotes

Git is distributed; GitHub is one place to host a copy. A **remote** is a named URL.

```bash
git remote add origin <url>
git remote -v
git push -u origin main        # first push, sets upstream
git pull                       # fetch + integrate
git fetch                      # download only
```

### 5.2 Authentication options

| Method | Notes |
| --- | --- |
| GitHub CLI (`gh auth login`) | Easiest; handles the credential flow. |
| Credential manager (HTTPS) | Git Credential Manager stores a token for you. |
| SSH keys | Generate a key, add the public key to GitHub; no password prompts. |

Passwords are not accepted for Git over HTTPS; use a token via one of the above. Pick one and stay
consistent.

### 5.3 Issues, pull requests and review

- **Issue** — a tracked unit of work or bug report.
- **Pull request (PR)** — a request to merge one branch into another, with a discussion thread,
  diff, and checks.
- **Review comments** — line-level feedback; reviewer can comment, approve, or request changes.

A good PR: small, focused, with a title and description saying what changed and why, and linked to
an issue when relevant.

### 5.4 GitHub Flow

1. Branch from `main`.
2. Commit on the branch.
3. Open a PR.
4. Review and discuss; push more commits as needed.
5. Merge to `main`.
6. Delete the branch.

`main` is always deployable/healthy. This is the workflow used for every stage of the program.

### 5.5 Conventional Commits

A convention for commit messages:

```
<type>(<optional scope>): <short description>

<optional body>

<optional footer, e.g. BREAKING CHANGE: ...>
```

| Type | Use for |
| --- | --- |
| `feat` | A new feature |
| `fix` | A bug fix |
| `docs` | Documentation only |
| `style` | Formatting, no logic change |
| `refactor` | Code change that is neither fix nor feature |
| `test` | Adding or fixing tests |
| `chore` | Tooling, config, housekeeping |

Examples:

```
feat: add hello-node script
docs: add setup notes to README
chore: add .gitignore and .env.example
fix(greeting): handle empty name argument
```

Rules of thumb: imperative mood ("add", not "added"), lowercase type, no trailing period, subject
around 50–72 characters.

### 5.6 README basics

A README answers: what is this, how do I run it, how do I contribute? For this module a few lines
are enough: project name, purpose, and how to run `hello-node`.

### 5.7 Why a public repo?

The program convention is a **public** training repo for every module. Branch protection on private
repos depends on the GitHub plan; a public repo avoids that dependency. (Because it is public:
never commit secrets or personal data.)

---

## 6. Guided practice (throwaway repo)

These drills use a sandbox folder, **not** your lab repo. Create one:

```bash
mkdir git-sandbox && cd git-sandbox && git init
```

### Drill 1 — Staging and unstaging
1. Create `a.txt` and `b.txt` with some text. Stage both.
2. Unstage only `b.txt` without losing its contents.
3. Commit only `a.txt`. Confirm with `git status` and `git log --oneline`.

### Drill 2 — Discard vs stash
1. Edit `a.txt` and commit.
2. Edit it again, then stash the change; confirm the file returned to its committed state.
3. Pop the stash. Then discard the same edit permanently using the appropriate command.

### Drill 3 — Create and resolve a conflict
1. On `main`, create `greeting.txt` with one line and commit.
2. Create branch `one`, change that line, commit. Switch back to `main`.
3. Create branch `two` from `main`, change the *same line* differently, commit.
4. Merge `one` into `main`, then merge `two`. Resolve the conflict, finish the merge, and view the
   result with `git log --oneline --graph`.

### Drill 4 — Fix the last commit message
Make a commit with a bad message ("stuff"). Rewrite it to a proper Conventional Commit message
using a command that *amends* the most recent commit. Why is this safe only before pushing?

### Drill 5 — Ignore an already-tracked file
1. Commit a file called `secret.env`.
2. Add it to `.gitignore` and observe that Git still tracks it.
3. Make Git stop tracking it *without* deleting your local copy, then commit.

### Drill 6 — Conventional Commits series
On a new branch, make three separate small changes and commit each with a correct Conventional
Commit message using at least two different types. Review with `git log --oneline`.

### Drill 7 — Undo with revert and reset (sandbox only)
1. Make three commits `A`, `B`, `C`.
2. Use the safe-on-shared-history tool to undo `B` and observe the log.
3. In a *fresh* sandbox, make three commits and use the other tool to move back by one commit in
   each of the three modes. Compare `git status` after each (recreate the commits between tries).

---

## 7. Knowledge check

Answer without looking at the material first. Answers to questions 1–10 are at the end; questions
11–13 are open-ended and have **no answer key** because they feed your write-up.

### Multiple choice

**1.** Which area holds the draft of your next commit?
- a) Working tree  b) Staging area  c) Remote  d) Stash

**2.** You staged `config.js` by mistake and want to keep your edits but unstage it. Which command?
- a) `git restore config.js`
- b) `git restore --staged config.js`
- c) `git reset --hard`
- d) `git stash drop`

**3.** What does `git diff --staged` compare?
- a) Working tree vs staging
- b) Staging vs last commit
- c) Local vs remote
- d) Two branches

**4.** A merge completes with no new commit and `main` simply moves forward. This is a:
- a) Rebase  b) Squash  c) Fast-forward  d) Cherry-pick

**5.** In a conflict block, the section between `<<<<<<< HEAD` and `=======` is:
- a) The incoming branch's version
- b) The common ancestor
- c) Your current branch's version
- d) The stash

**6.** Which file should be committed so a new developer knows which settings to configure?
- a) `.env`  b) `.env.example`  c) `node_modules`  d) `.nvmrc.bak`

**7.** You add `dist/` to `.gitignore`, but `dist/app.js` was already committed. What happens?
- a) Git removes it automatically
- b) It is ignored from now on and deleted from history
- c) It is still tracked until you untrack it
- d) Git errors on the next commit

**8.** Which is a valid Conventional Commit?
- a) `Fixed the login bug.`
- b) `fix: handle expired token on login`
- c) `FIX - login`
- d) `update stuff`

**9.** Which statement about `git rebase` is most accurate?
- a) It never changes commit IDs
- b) It replays commits and creates new ones, so avoid it on shared history
- c) It is the same as `git merge`
- d) It deletes the branch

**10.** Why does this program use a public training repo?
- a) GitHub bills public repos less
- b) Branch protection on private repos depends on the plan; public avoids it
- c) Private repos cannot hold Node projects
- d) Public repos run faster

### Short answer / explain

**11.** Describe, in your own words, a situation where you would choose `git revert` over
`git reset`, and one where the reverse is true.

**12.** Walk through what happens, step by step, between editing a file and seeing it on GitHub
in a merged PR. Mention each Git area and GitHub concept involved.

**13.** A teammate says "I'll just commit my `.env` — the repo is private." Explain why that is
still a bad idea and what to do instead.

<details>
<summary>Answer key (questions 1–10 and drills only)</summary>

**Multiple choice:** 1-b, 2-b, 3-b, 4-c, 5-c, 6-b, 7-c, 8-b, 9-b, 10-b.

**Drill hints (verify by running them):**

- Drill 1: `git restore --staged b.txt`, then `git commit -m "..."` (only `a.txt` is staged).
- Drill 2: `git stash`, `git stash pop`; discard with `git restore a.txt`.
- Drill 3: after the second merge, edit the marked file, `git add greeting.txt`, then
  `git commit` (or `git merge --continue`).
- Drill 4: `git commit --amend -m "chore: ..."`. Amending rewrites the commit, so only do it before
  the commit is shared.
- Drill 5: `git rm --cached secret.env`, then commit.
- Drill 6: e.g. `feat: ...`, `docs: ...`, `chore: ...`.
- Drill 7: `git revert <B>` adds a new commit; `git reset --soft/--mixed/--hard HEAD~1` differ in
  what is left staged and in the working tree (see table in 2.3).

</details>

---

## 8. Common pitfalls

| Symptom | Likely cause / fix |
| --- | --- |
| `fatal: not a git repository` | You are not inside a repo folder; `cd` into it or `git init`. |
| `Permission denied (publickey)` | SSH key not added to GitHub or agent. |
| Push rejected (non-fast-forward) | Remote has commits you lack; `git pull` first. |
| `.gitignore` seems ignored | The file is already tracked; `git rm --cached <file>`. |
| Committed to `main` by accident | Create a branch from it, then move `main` back before pushing. |
| `node` is the wrong version | Run your version manager's `use` command / check `.nvmrc`. |
| Detached HEAD | You checked out a commit, not a branch; `git switch -c <name>` to keep work. |

## 9. Further reading

- Pro Git book (free): <https://git-scm.com/book>
- Git reference: <https://git-scm.com/docs>
- GitHub Docs — GitHub flow: <https://docs.github.com/get-started/using-github/github-flow>
- Conventional Commits: <https://www.conventionalcommits.org>
- Learn Git Branching (interactive): <https://learngitbranching.js.org>

## 10. Ready for the lab?

Before starting the lab in `brief.md`, you should be able to:

- [ ] Explain working tree, staging and commit, and move changes between them.
- [ ] Create a branch, make a conflict on purpose and resolve it.
- [ ] Say which `.gitignore` entries a Node project needs and why `.env.example` exists.
- [ ] Write a Conventional Commit without looking it up.
- [ ] Describe the GitHub Flow from branch to merged PR.

Then proceed to `tasks.md` and fill in `write-up-template.md` as you build.
