# M03 Write-Up — Node.js Runtime & npm

> Fill this in as you go, not after the fact. Write it so a trainee starting this stage next cohort
> could follow your path on their own.

## What I built

For M03 I built the **learning material** that supports the stage, not the lab deliverables
themselves: `modules/m03-nodejs-runtime-npm/learning-material.md`. It is a companion to `brief.md`
and `tasks.md` and teaches the concepts a trainee needs before building the `log-report` CLI and the
bare `node:http` server. It follows the M01 and M02 shape and is 639 lines:

- **What Node is:** V8 + libuv + core APIs, the event loop and thread pool, blocking vs
  non-blocking, and when Node does and doesn't fit.
- **Modules and core APIs:** ESM vs CommonJS, `import.meta.dirname`, the `node:` prefix, and a table
  of `fs/promises`, `path`, `os`, `events`, `crypto`, `timers/promises` and `util` (`parseArgs`).
- **Streams:** why they exist, readable/writable/transform, `for await` and `readline`,
  backpressure and `pipeline`, and how to measure memory so "it streams" can be proven.
- **`node:http` by hand:** the request/response model, and a list of what the trainee must do
  themselves (routing, body reading, status codes, errors).
- **`process`, config and ergonomics:** `argv`, `env`, exit codes, signals, `--env-file`, `--watch`,
  `--run`, and the globals.
- **npm and supply-chain hygiene:** `package.json`, semver, the lockfile, `install` vs `ci`, vetting
  packages, lockfile diffs and lifecycle scripts.
- **Debugging:** `console`, `--inspect`, reading stack traces.
- **Exercises:** 8 sandbox drills and a 13-question knowledge check with an answer key for the
  multiple-choice questions and drill hints only.

I did **not** build the `log-report` CLI or the bare server, so there are no PRs, memory
measurements or endpoint results in this write-up. Those are the trainee's lab work. I also did not
write the explanation of what Express would remove pain from.

I produced it with the Nexus E→P→A→V cycle: `/evaluate`, `/plan`, `/apply`, then `/validate`.

## Why it's built this way (key decisions)

- **A separate file, not edits to the brief.** The program is build-to-teach and `brief.md` has no
  lesson or answers, so the lesson lives in its own file.
- **Concepts without the lab answers.** The material explains streams and `node:http`, but sections
  5.3 and 11 ask design questions instead of giving the CLI or the server. The "pain is the point"
  rule means the Express explanation has to come from the trainee's own experience.
- **Memory is something you prove.** The brief's Definition of done asks for genuine streaming, so
  §4.4 shows how to measure memory and §12 Drill 3 compares `readFile`, a stream and `readline`
  on a small generated file. The drills never use the trainer's log file.
- **Structure follows `tasks.md`.** The 11 Research bullets map to sections, so nothing is taught
  that the module doesn't assess.
- **Code claims were run.** I ran the stream, `parseArgs`, `http`, `--env-file` and `--run`
  examples on Node 24.19.0 before writing, then re-checked the ESM errors, exit codes, `argv` index
  and unguarded-JSON crash during `/validate`.
- **Tighter than M02.** I kept it to 639 lines after noting M02 (766) ran long.
- **Left `tasks.md` alone.** M01 and M02 don't link their learning material either.

## How to build it (teach it to the next trainee)

1. **Evaluate.** Run `/evaluate <module>`. Query the graph and read `brief.md`, `tasks.md`, the
   write-up template and the arch-doc row. Note the constraints (plain JavaScript, no Express, no
   solutions file) and risks, such as the log file coming from the trainer.
2. **Plan.** Run `/plan` and point it at the earlier modules' material as the pattern and at
   `tasks.md` as the topic list. Check the blast radius and wait for approval.
3. **Apply.** Test the code samples in a scratch folder first, then write the file. Keep generated
   data out of the repo.
4. **Validate.** Re-run any claims you didn't test, check the answer key, and classify issues as
   BLOCKER, FIX NOW or BACKLOG.
5. **Fill the write-up** last, from what the cycle actually produced, and say what was not built.

Reasoning: testing the samples before and after writing keeps wrong claims out of material that
trainees will trust.

## Concepts worth explaining

**Streams.** `readFile` loads a whole file, so memory grows with its size: about 100 MB for the log
file, and more with several requests at once. A stream hands over small chunks, so memory stays
roughly flat. Chunks are arbitrary-size Buffers that can split a line, which is why `readline` is
useful. `pipeline` connects streams, applies backpressure and cleans up when one fails.

**Event loop vs thread pool.** JavaScript runs on one thread. Waiting on I/O doesn't block it,
because the work goes to the OS or to libuv's thread pool (default 4 threads) and a callback is queued
when it's done. Blocking the loop with a synchronous call or a CPU loop stalls every request. Note
that `await readFile(...)` doesn't block the loop but still loads the whole file, which is a
separate problem.

## What tripped me up

- **Blocking vs memory.** `await readFile` is non-blocking but not memory-safe. Keeping those two
  ideas apart took care in the writing.
- **Teaching without solving.** I had to describe how a bare server reads a body without handing
  over the finished routing and parsing. I used the shape of the problem plus questions.
- **What I didn't run.** I didn't run the `--inspect-brk` flow or the npm commands, so those
  sections come from knowledge, not a run.

## Checkpoint evidence

- **Learning material:** `modules/m03-nodejs-runtime-npm/learning-material.md`
  (639 lines, 16 sections).
- **Exercises:** 8 guided drills (§12) and a 13-question knowledge check (§13).
- **Validation result:** 8 of 8 criteria passed. Behaviour claims were run on Node 24.19.0; the
  knowledge graph includes the new file as its own community.
- **Not applicable here:** both PRs merged to `main`, the CLI's memory behaviour on the full log
  file, the bare server's three endpoints, and the Express-pain explanation. Those belong to the
  trainee's lab.

## What I'd do differently

- Add a note to Drill 5 that signal behaviour on Windows differs from macOS and Linux.
- Run the `--inspect-brk` and npm drills myself before calling the material validated.
- Reconcile §14, where one row says a server "hangs" and Drill 6 shows an unguarded bad body crashing.
