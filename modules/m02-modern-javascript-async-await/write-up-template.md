# M02 Write-Up — Modern JavaScript & Async/Await

> Fill this in as you go, not after the fact. Write it so a trainee starting this stage next cohort
> could follow your path on their own.

## What I built

For M02 I built the **learning material** that supports the stage, not the lab deliverables
themselves: `modules/m02-modern-javascript-async-await/learning-material.md`. It is a companion to
`brief.md` and `tasks.md` and teaches the concepts a trainee needs before building the Async toolkit.
It follows the same shape as the M01 material and is about 770 lines:

- **Language essentials:** `const`/`let`, closures, arrow functions and `this`, destructuring,
  spread/rest, `?.`, `??`/`??=`, array methods and non-mutating helpers, `Map`/`Set`, `#private`
  fields, custom errors with `cause`, and named vs default ES module exports.
- **How async works:** call stack, event loop, task vs microtask queue, what `await` does, and a
  worked output-order example the trainee reasons through.
- **Callbacks → Promises → async/await:** error handling in each style and how to convert callback
  code.
- **Concurrency:** sequential vs parallel, and `all` / `allSettled` / `race` / `any` in a comparison
  table.
- **Modern APIs:** `Promise.withResolvers`, `Promise.try`, `for await…of`, `Array.fromAsync`.
- **Cancellation and `fetch`:** `AbortController`, `AbortSignal.timeout`, checking `response.ok`,
  measuring with `performance.now()`.
- **Pitfalls:** forgotten `await`, `await` in loops, `forEach(async …)`, unhandled rejections and
  swallowed errors, each with a cause and a fix.
- **Toolkit design questions:** prompts for `sleep`, `retry`, `withTimeout` and `mapLimit`, with no
  solutions.
- **Exercises:** 9 sandbox drills and a 13-question knowledge check with an answer key for the
  multiple-choice questions and drill hints only.

I did **not** build the Async toolkit, the self-checks, the JSONPlaceholder timing comparison or the
`node:fs` refactor. Those are the trainee's lab work, so there are no timing numbers in this write-up.

I produced it with the Nexus E→P→A→V cycle: `/evaluate`, `/plan`, `/apply`, then `/validate`.

## Why it's built this way (key decisions)

- **A separate file, not edits to the brief.** The program is build-to-teach and `brief.md` has no
  lesson or answers. A separate file leaves the brief, tasks and template as they were.
- **Concepts without the lab answers.** The material explains how `AbortSignal`, combinators and
  the event loop work but leaves `retry`, `withTimeout` and `mapLimit` unwritten. Section 8 asks
  design questions (what does `retries: 3` mean, does `withTimeout` cancel the work, how is order
  kept in `mapLimit`) so the trainee has to decide.
- **Structure follows `tasks.md`.** Each of the 14 Research bullets maps to a subsection, so nothing
  is taught that the module doesn't assess and nothing assessed is untaught.
- **Code claims were run, not assumed.** In `/validate` I ran the worked example, quiz Q4, the
  abort error names, the 404 behaviour of `fetch`, `Promise.any` and the newer APIs on Node 24.19.0.
- **Sandbox drills.** They use throwaway scripts and files, not the lab's `fs` code, so the lab
  refactor stays the trainee's own.
- **Single file.** One document is easier to follow than splitting lessons and exercises, as in M01.
- **Left `tasks.md` alone.** M01's `tasks.md` doesn't link its learning material, so I matched that.

## How to build it (teach it to the next trainee)

1. **Evaluate.** Run `/evaluate <module>`. Query the graph, then read `brief.md`, `tasks.md`, the
   write-up template and the arch-doc rows. Note the constraints (no solutions file, plain
   JavaScript, `node:assert` only, JSONPlaceholder) and the risks.
2. **Plan.** Run `/plan` and point it at a reference to copy (here, the M01 material) and at
   `tasks.md` as the source of topics. Check the blast radius, list files created and modified, and
   wait for approval.
3. **Apply.** Run `/apply`. Write exactly what the plan says. Check the reference to decide small
   things, such as whether `tasks.md` links to the material.
4. **Validate.** Run `/validate`. Re-run the code samples and check the answer key against them,
   then classify issues as BLOCKER, FIX NOW or BACKLOG.
5. **Fill the write-up** last, from what the cycle actually produced, and say what was not built.

Reasoning: reading `tasks.md` first keeps the material tied to what is assessed, and running the
samples catches wrong claims before a trainee trusts them.

## Concepts worth explaining

**Event loop and the microtask queue.** JavaScript runs one call stack. Async work finishes outside
the stack and queues a callback. When the stack is empty, the runtime drains the whole microtask
queue (promise `.then` callbacks and the code after an `await`), then takes one task, such as a
`setTimeout` callback, and repeats. That is why a promise callback runs before `setTimeout(fn, 0)`.
In the worked example the output is `A D F C E B`: synchronous lines first (the `async` function body
runs until its first `await`), then the promise and the continuation after `await`, then the timer.

**`Promise.all` vs `allSettled`, `race` and `any`.** `all` needs every result and rejects as soon as
one fails, though the other promises keep running. `allSettled` never rejects and reports every
outcome. `race` follows whichever settles first, which is useful for timeouts. `any` takes the first
success and rejects with an `AggregateError` only if all fail.

## What tripped me up

- **A promise can't be cancelled.** `Promise.all` rejecting, or a timeout winning a race, doesn't stop
  the work. I wrote this into the material as cooperative cancellation through `AbortSignal`.
- **`fetch` doesn't reject on 404.** I confirmed it fulfils with `ok === false`, which is a common
  source of silent bugs.
- **Teaching without solving.** The hardest part was explaining `retry`, `withTimeout` and `mapLimit`
  well enough to be useful without writing them. Design questions were the compromise.

## Checkpoint evidence

- **Learning material:** `modules/m02-modern-javascript-async-await/learning-material.md`
  (about 770 lines, 13 sections).
- **Exercises:** 9 guided drills (section 9) and a 13-question knowledge check (section 10).
- **Validation result:** 6 of 6 criteria passed, with the `tasks.md` link skipped on purpose. Code
  samples were run on Node 24.19.0.
- **Not applicable here:** `node:assert` self-checks for the toolkit, the sequential-vs-parallel
  timings and the lab's output-order explanation. Those belong to the trainee's lab.

## What I'd do differently

- Trim the material. It is longer and section 11 and the drills could be
  shorter for a 10-hour module.
- Add a footnote that `Object.groupBy` returns a null-prototype object.
- Run `graphify update .` after writing, so the new file is in the knowledge graph.
