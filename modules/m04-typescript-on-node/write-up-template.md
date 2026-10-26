# M04 Write-Up — TypeScript Basics & TypeScript on Node.js

> Fill this in as you go, not after the fact. Write it so a trainee starting this stage next cohort
> could follow your path on their own.

## What I built

For M04 I built the **learning material** that supports the stage, not the lab deliverables
themselves: `modules/m04-typescript-on-node/learning-material.md`. It is a companion to `brief.md`
and `tasks.md` and teaches the concepts a trainee needs before porting the M02 toolkit. It follows the
M01-M03 shape and is 684 lines:

- **Core TypeScript:** structural typing, inference vs annotation, unions, tuples, literal types,
  `as const`, `type` vs `interface`, and unions instead of `enum`.
- **Generics and narrowing:** function generics, constraints and defaults, `Promise<T>`, `typeof`/`in`/
  `instanceof`, discriminated unions and type guards.
- **`unknown`, `any`, `never`,** with an exhaustiveness example.
- **Utility types,** `satisfies` and `import type`, shown on a generic `User` type so the trainee still
  derives the Task types themselves.
- **Project setup:** installing TypeScript 7 and `@types/node@24`, a `tsconfig.json` flag-by-flag table,
  and the `typecheck` script.
- **Three ways to run TypeScript:** `tsx`, Node's native type stripping and `tsc`, including the
  import-extension tension between them.
- **Node specifics:** typing async code, caught errors, `process.env` and `@types/*`; TypeScript 6 → 7;
  and a table of real compiler errors.
- **Exercises:** 9 sandbox drills and a 13-question knowledge check with an answer key for the
  multiple-choice questions and drill hints only.

I did **not** port the toolkit, write the Task types or `Result<T, E>`, or produce a `tsc --noEmit`
run for a lab project. The M02 toolkit also does not exist in this repo, so there was nothing to port.
Those are the trainee's lab work, and this write-up has no `tsc` output of its own to show.

I produced it with the Nexus E→P→A→V cycle: `/evaluate`, `/plan`, `/apply`, then `/validate`.

## Why it's built this way (key decisions)

- **A separate file, not edits to the brief.** The program is build-to-teach and `brief.md` has no
  lesson or answers, so the lesson lives in its own file.
- **Concepts without the lab answers.** §6 teaches utility types on an unrelated `User` type, and §12
  asks design questions about `retry`, `mapLimit`, `Task` and `Result<T, E>` without answering them.
  The Definition-of-done explanations (`unknown` vs `any`, `type` vs `interface`) are taught with
  contrasts but left for the trainee to write.
- **Tested against the real toolchain.** The registry has TypeScript 7.0.2, `tsx` 4.23.15 and
  `@types/node@24`. I ran every `tsconfig` flag and snippet in a scratch project on Node 24.19. That
  showed things the brief only implies:
  - TypeScript 7 is strict by default with no `tsconfig`.
  - `target: es5`, `baseUrl` and `moduleResolution: node10` are now errors.
  - `tsx` and native stripping don't type-check; only `tsc --noEmit` does.
  - Native stripping on 24.19 runs erasable syntax with no flag and rejects `enum`.
- **A whole section on import extensions.** Extensionless imports fail in both `tsc` and Node. `./x.js`
  passes `tsc` and `tsx` but fails natively, while `./x.ts` runs natively but needs
  `allowImportingTsExtensions`. I judged that a trainee would hit this early, so §8.3 explains it.
- **Structure follows `tasks.md`.** The 15 Research bullets map to sections.
- **Time budget fixed during validation.** The first draft summed to about 7.1 h in an 8 h module. I
  trimmed it to about 6.4 h.
- **Left `tasks.md` alone,** matching M01-M03.

## How to build it (teach it to the next trainee)

1. **Evaluate.** Run `/evaluate <module>`. Query the graph and read the brief, tasks, write-up
   template and arch-doc rows. Note the version pins (TypeScript 7, Node 24) and risks, such as the
   missing M02 toolkit.
2. **Plan.** Run `/plan`, pointing at the earlier modules' material as the pattern. Check the registry for
   the pinned versions before promising anything about them.
3. **Apply.** Install the toolchain in a scratch folder outside the repo, run every snippet and flag you
   plan to describe, then write the file from what you observed.
4. **Validate.** Re-run what you didn't test, check the answer key and drill hints, and check that the
   time budget adds up.
5. **Fill the write-up** last, from what the cycle actually produced, and say what was not built.

Reasoning: new tooling is the easiest place to teach something wrong, so every behaviour claim in the
material is backed by a run, not memory.

## Concepts worth explaining

**`tsx` vs native stripping vs `tsc`.** Only `tsc --noEmit` checks types. `tsx` and Node's native
type stripping both just remove the types and run the JavaScript, so a file with a type error still
runs. The difference between those two is syntax: native stripping on Node 24 handles only erasable
syntax (annotations, `type`, `interface`, `import type`) and rejects `enum`, which is why
`erasableSyntaxOnly` exists and why this course uses unions of string literals.

**Discriminated unions and `never`.** A union whose members share a literal tag lets the compiler
narrow the type inside each `case`. Assigning the leftover value to a `never` variable turns a forgotten
case into a compile error, so adding a union member breaks the build until you handle it.

## What tripped me up

- **Types look like runtime checks.** They are erased, so they don't validate data from files or the
  network. That is Zod's job in M08.
- **Import extensions.** I expected `./helper` or `./helper.js` to work everywhere, but each of the
  three runners accepts a different style.
- **A subtle `import type` rule.** A class used only as a type compiles without `type`, while an
  interface doesn't. My first wording was too broad.
- **Time budget.** My first estimate didn't add up to the module length.

## Checkpoint evidence

- **Learning material:** `modules/m04-typescript-on-node/learning-material.md`
  (684 lines, 17 sections).
- **Exercises:** 9 guided drills (§13) and a 13-question knowledge check (§14).
- **Validation result:** 7 of 7 criteria passed, with one issue fixed (the time budget). Behaviour was
  run on TypeScript 7.0.2, `tsx` 4.23.15 and Node 24.19.0.
- **Not applicable here:** zero `tsc --noEmit` errors for the ported toolkit, the toolkit running via
  `tsx`, and the trainee's own native-stripping attempt. Those belong to the trainee's lab.

## What I'd do differently

- Add a note to §6.3 that a class used only as a type compiles without `import type`, while an
  interface needs it.
- Say earlier in the material where a trainee without an M02 toolkit should get one.
- Mention npm's `allow-scripts` / `approve-scripts` behaviour somewhere, since it appeared during install
  and bears on the lifecycle-scripts discussion in M03.
