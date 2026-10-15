# M01 Write-Up — Dev Environment, Git & GitHub

> Fill this in as you go, not after the fact. Write it so a trainee starting this stage next cohort
> could follow your path on their own.

## What I built

For M01 I built the **learning material** that supports the stage, not the lab repo itself:
`modules/m01-dev-environment-git-github/learning-material.md`. It is a companion to `brief.md` and
`tasks.md` (which define the deliverables) and teaches the concepts a trainee needs to reach them:

- **Toolchain:** VS Code, Git, Node through a version manager, `.nvmrc`, Bruno, terminal basics.
- **Git mental model:** working tree → staging → commit, the everyday commands, and the undo tools.
- **Branching and conflicts:** branches, fast-forward vs merge commit, resolving conflicts, merge vs
  rebase (conceptual).
- **`.gitignore` and secrets:** the Node-project ignore list and the `.env.example` habit.
- **GitHub collaboration:** remotes, authentication, Issues, pull requests, GitHub Flow,
  Conventional Commits.
- **Exercises:** 7 sandbox drills and a 13-question knowledge check, with an answer key for the
  multiple-choice and drills only.

I produced it by running the Nexus E→P→A→V cycle: `/evaluate` to orient, `/plan` for the blueprint,
`/apply` to write the file, and `/validate` to check it against the plan.

## Why it's built this way (key decisions)

- **A separate file, not edits to the brief.** The program is build-to-teach: `brief.md` is owned by
  the trainer and contains no lesson or answers. Putting the lesson in its own file leaves the brief,
  tasks and write-up template intact.
- **Concepts without the lab answers.** The material explains how Git works but leaves the graded
  items (the `revert` vs `reset` narration, resolving your own conflict) for the trainee to do, so
  the lab still proves understanding.
- **Structure follows the brief's Scope and Definition of done.** Each section maps to a Scope
  bullet, so nothing is taught that the module doesn't assess and nothing assessed is untaught.
- **Practice in a sandbox.** The drills use a throwaway repo so trainees can break things safely
  before the real lab.
- **Single file.** For a 4-hour module, one document is easier to follow than splitting lessons and
  exercises.
- **EPAV process.** Evaluating first (graph and docs) surfaced the build-to-teach constraint and the
  `better-sqlite3` build-tools risk before any writing, which shaped the plan.

## How to build it (teach it to the next trainee)

1. **Evaluate.** Run `/evaluate <module>`. Query the knowledge graph, then read the brief, tasks,
   write-up template and the relevant arch-doc sections. Note constraints (no solutions, plain
   JavaScript, pinned Node version) and risks.
2. **Plan.** Run `/plan` with what you want produced. Check blast radius, list files created and
   modified, state constraints, and resolve open decisions (for example single file vs split).
   Wait for approval.
3. **Apply.** Run `/apply`. Write exactly what the plan says and no more. Skip anything the
   reviewer excluded (here, the README update).
4. **Validate.** Run `/validate`. Check each plan criterion, classify issues as BLOCKER, FIX NOW or
   BACKLOG, and confirm no other files changed.
5. **Fill the write-up** as the last step, from what the cycle actually produced.

Reasoning: evaluating before planning prevents writing material that contradicts the program's
rules, and validating afterwards confirms the scope held.

## Concepts worth explaining

**Working tree → staging → commit.** Git keeps your work in three places. The working tree is the
files on disk. The staging area is a draft of the next commit, where you choose exactly what goes in.
A commit is a permanent snapshot in history. `git add` moves changes into staging and `git commit`
records them. This is why `git diff` and `git diff --staged` show different things.

**The E→P→A→V cycle.** Evaluate, Plan, Apply, Validate is a loop that separates understanding,
deciding, doing and checking. Keeping them apart stops a task from jumping to code before the
constraints are known, and gives a review point (plan approval) before anything is written.

## What tripped me up

- The tension between "write a complete learning material" and the program's build-to-teach rule
  that the brief has no lesson or answers. Resolved by creating a separate file and withholding
  answers for the graded lab items.
- Confusing the two artifacts: the learning material is reference content, while the write-up is the
  trainee's own teaching content. They have different authors and purposes.

## Checkpoint evidence

- **Learning material:** `modules/m01-dev-environment-git-github/learning-material.md`
  (about 550 lines, 10 sections).
- **Exercises:** 7 guided drills (section 6) and a 13-question knowledge check (section 7).
- **Validation result:** 7 of 7 plan criteria passed; `README.md`, `brief.md` and `tasks.md` unchanged.
- **Graph:** `graphify` shows the new file as its own community within M01, with no unexpected
  cross-references.

## What I'd do differently

- Trim the `revert` vs `reset` reference table so it doesn't narrow what trainees must work out for
  the lab.
- Reconfirm the pinned Node version with the trainer before writing it into the material.
