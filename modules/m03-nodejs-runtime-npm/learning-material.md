# M03 Learning Material — Node.js Runtime & npm

> **Companion to `brief.md` and `tasks.md`.** The brief tells you *what* to deliver; this document
> teaches the *concepts* you need to get there. It deliberately does **not** contain the finished
> lab: you still build the `log-report` CLI, the bare `node:http` server, and your own explanation
> of what Express would remove pain from. Read it, try the drills, then do the lab.

**Week 1 · Days 4-5 · ~6 hours · the Week 1 Friday gate**

## 0. How to use this material

| Section | Maps to brief Scope bullet | Maps to Definition of done | Suggested time |
| --- | --- | --- | --- |
| 1. What Node is | What Node is | — | 25 min |
| 2. Modules | Modules | — | 20 min |
| 3. Core APIs | Core APIs | — | 30 min |
| 4. Streams | `node:stream` | CLI genuinely streams | 40 min |
| 5. `node:http` by hand | `node:http` | Explain what Express removes | 40 min |
| 6. `process` | `process` | — | 20 min |
| 7. Config, ergonomics, globals | Config, Globals | — | 15 min |
| 8. npm | npm | — | 30 min |
| 9. Supply-chain hygiene | Supply-chain hygiene | — | 20 min |
| 10. Debugging | Debugging | — | 15 min |
| 11. Thinking about the two programs | Deliverable | Both merged by PR | 15 min |
| 12. Guided practice | all | — | 45 min |
| 13. Knowledge check | all | — | 20 min |

That is about 5.5 hours of reading and practice, so budget the rest of your 6 hours carefully: the
lab itself is where most of your time goes. Read sections 1–10, skim the ones you already know, do
the drills in 12, take the check in 13, then start the lab.

**Before the lab:** get the synthetic ~100 MB log file (or its generator script) from your trainer.
The drills below use tiny throwaway files; never test your streaming claims on a tiny sample only.

**Learning outcomes.** After this module you can:

- Explain what Node.js is made of and why blocking the event loop matters.
- Use ES modules and the `node:` core modules confidently.
- Process a file larger than you want in memory using streams.
- Write a bare HTTP server and see where a framework would help.
- Read arguments, environment variables and signals through `process`.
- Manage dependencies with npm and judge a package before installing it.
- Debug with `console`, stack traces and `--inspect`.

---

## 1. What Node is

### 1.1 The pieces

| Piece | Role |
| --- | --- |
| **V8** | Google's JavaScript engine. Compiles and runs your JS. |
| **libuv** | A C library providing the event loop, async file/network I/O and a **thread pool**. |
| **Core APIs** | `fs`, `http`, `crypto`, `path`, `stream`… the built-in modules, written in JS and C++. |

Node = V8 running your code + libuv handling I/O + the core modules to talk to the operating system.

### 1.2 Single-threaded event loop, plus a thread pool

Your JavaScript runs on **one thread** (the event loop from M02). When you ask for I/O, Node hands
it to the OS or to libuv's **thread pool** (default 4 threads: some file-system calls, DNS lookups,
`crypto`, compression), keeps running other code, and queues your callback when it finishes.

### 1.3 Blocking vs non-blocking

```js
import { readFileSync } from "node:fs";
import { readFile } from "node:fs/promises";

const a = readFileSync("big.log");        // blocking: nothing else runs until it finishes
const b = await readFile("big.log");      // non-blocking: the loop stays free while it reads
```

A synchronous call or a heavy CPU loop **blocks every request** on a server. Note that
`await readFile(...)` is non-blocking but still loads the *whole file into memory* — which is a
different problem, and the subject of section 4.

### 1.4 When Node fits (and doesn't)

| Good fit | Poor fit |
| --- | --- |
| APIs and web servers with lots of concurrent I/O | Heavy CPU work (video encoding, big number-crunching) on the main thread |
| Real-time features, proxies, tooling and CLIs | Workloads that need many threads sharing memory |
| Same language front to back | Tiny scripts where another tool is simpler |

CPU-heavy work can still be done in Node with `worker_threads` or separate processes; you don't need
them in this module.

---

## 2. Modules

### 2.1 ESM vs CommonJS

| | ES modules (ESM) | CommonJS (CJS) |
| --- | --- | --- |
| Syntax | `import x from "./x.js"` / `export` | `const x = require("./x")` / `module.exports` |
| Loading | Static, asynchronous; supports top-level `await` | Synchronous |
| Enabled by | `.mjs` or `"type": "module"` in `package.json` | `.cjs` or no `"type"` (the default) |
| Paths | Include the file extension | Extension optional |

This course uses **ESM**. Set `"type": "module"` in `package.json` and `.js` files become ESM.

### 2.2 No `__dirname` in ESM

```js
import path from "node:path";

import.meta.dirname;                                   // directory of this file
import.meta.filename;                                  // full path of this file
path.join(import.meta.dirname, "data", "sample.log");   // build paths relative to the file
```

### 2.3 The `node:` prefix

`import fs from "node:fs"` says unambiguously "the built-in module", never a package with the same
name from `node_modules`. Use it for every core module.

---

## 3. Core APIs

| Module | Use it for | Small example |
| --- | --- | --- |
| `node:fs/promises` | Read/write/list files with promises | `await readFile(p, "utf8")`, `await writeFile(p, text)`, `await readdir(dir)` |
| `node:path` | Build paths portably | `path.join("a", "b.txt")`, `path.extname(f)`, `path.basename(f)` |
| `node:os` | Machine information | `os.cpus().length`, `os.tmpdir()`, `os.EOL` |
| `node:events` | The `EventEmitter` base class | `emitter.on("x", fn)`, `emitter.emit("x", data)`, `once(emitter, "x")` |
| `node:crypto` | Hashing, random values, UUIDs | `randomUUID()` |
| `node:timers/promises` | Promise-based timers | `await setTimeout(100)` (import under another name to avoid clashing with the global) |
| `node:util` | `parseArgs`, `promisify`, `inspect` | see below |

### 3.1 `util.parseArgs`

```js
import { parseArgs } from "node:util";

const { values, positionals } = parseArgs({
  options: {
    out:     { type: "string",  short: "o" },
    verbose: { type: "boolean", short: "v" },
  },
  allowPositionals: true,
});
// node cli.js -v --out report.json input.log
// values → { verbose: true, out: "report.json" }, positionals → ["input.log"]
```

Things to decide in your CLI: which options are required, what the defaults are, and what happens
on an unknown flag (`parseArgs` throws by default — `strict: true`).

### 3.2 `EventEmitter`

Many Node objects are emitters: streams, HTTP servers, `process`. You subscribe with `.on` and the
emitter calls you back. If an emitter emits `"error"` and nobody listens, Node **throws** — which is
why you will see error handlers attached to streams.

---

## 4. Streams

### 4.1 Why they exist

`readFile` on a 100 MB log loads 100 MB (plus overhead) into memory before you see one line. Ten
concurrent requests would mean a gigabyte. A **stream** delivers data in small chunks, so memory use
stays roughly flat regardless of file size.

| Stream type | Example |
| --- | --- |
| **Readable** | `createReadStream(path)`, an incoming HTTP request, `process.stdin` |
| **Writable** | `createWriteStream(path)`, an HTTP response, `process.stdout` |
| **Transform** | A stage that reads, changes and passes on data (e.g. compression) |

### 4.2 Streams are async iterables

Chunks (not lines!) arrive from a readable stream, and you can loop over them with `for await` — the
same tool you learned in M02:

```js
import { createReadStream } from "node:fs";

let bytes = 0;
for await (const chunk of createReadStream("sample.log")) {
  bytes += chunk.length;               // a Buffer of up to ~64 KB at a time
}
```

A chunk boundary can fall in the middle of a line. Reading line by line is a common need, and Node
has a helper for it:

```js
import { createInterface } from "node:readline";

const lines = createInterface({ input: createReadStream("sample.log"), crlfDelay: Infinity });
for await (const line of lines) {
  // one complete line at a time
}
```

### 4.3 Backpressure and `pipeline`

If a writer is slower than the reader, data piles up in memory unless the reader pauses.
Streams handle this "backpressure" for you **when you connect them correctly**:

```js
import { pipeline } from "node:stream/promises";

await pipeline(sourceStream, transformStream, destinationStream);   // wires up, cleans up, propagates errors
```

Prefer `pipeline` over chaining `.pipe()` by hand: it destroys all streams if one fails, and the
promise version rejects so you can `try/catch` it.

### 4.4 Proving that you stream

"It works on a small file" proves nothing about memory. To check, compare memory while processing:

```js
const mb = (n) => Math.round(n / 1024 / 1024);
const { rss, heapUsed } = process.memoryUsage();
console.log(`rss ${mb(rss)} MB, heap ${mb(heapUsed)} MB`);
```

Log it at intervals during the run, or measure the process from outside. Compare against a deliberate
`readFile` version on the same file and see the difference for yourself.

### 4.5 Streams and errors

Always plan for: the file not existing (`ENOENT`), a stream error mid-way, and what to do with a
malformed line. Decide whether bad lines are skipped, counted, or fatal — and say so in the report.

---

## 5. `node:http` by hand

### 5.1 The model

```js
import http from "node:http";

const server = http.createServer(async (req, res) => {
  // req: incoming message (a readable stream) — method, url, headers, body
  // res: server response (a writable stream) — status, headers, body
  res.writeHead(200, { "content-type": "application/json" });
  res.end(JSON.stringify({ ok: true }));
});

server.listen(3000, () => console.log("listening on http://localhost:3000"));
```

### 5.2 What you do yourself

| Task | The raw material Node gives you |
| --- | --- |
| Routing | `req.method` and `req.url` (a string including the query) — you decide what to do |
| Query string | Parse with `new URL(req.url, "http://localhost")` and `url.searchParams` |
| Reading a JSON body | `req` is a stream: collect chunks, then parse; the body is **not** parsed for you |
| Sending JSON | Set the `content-type` header, `JSON.stringify`, `res.end` |
| Status codes | You choose 200/201/400/404/405/500 deliberately |
| Errors | An uncaught throw in a handler can crash the process or leave the request hanging |

A taste of the body-reading part (for the shape of the problem, not a finished solution):

```js
let body = "";
for await (const chunk of req) body += chunk;     // request is an async iterable
// body is now a string — is it JSON? Is it too large? Is the header right? Is it empty?
```

### 5.3 Questions to answer while you write it

Keep a list of everything that felt repetitive, fragile or easy to forget. You need that list for
the Definition of done:

- How did you pick the handler for a given method + path? What happens for an unknown path? For a
  known path with the wrong method?
- What happens when the body is invalid JSON, empty, or enormous?
- How do you make sure every response gets a correct `content-type`?
- Where does a thrown error end up? Does the client get an answer?
- What would you have to copy into every new endpoint?

Your answers are the raw material for explaining what a framework removes — don't skip this.

---

## 6. `process`

| Property | Meaning |
| --- | --- |
| `process.argv` | `[nodePath, scriptPath, ...yourArgs]` — your arguments start at index 2 |
| `process.env` | Environment variables; values are always strings or `undefined` |
| `process.exitCode = 1` | Set the exit status and let Node finish cleanly |
| `process.exit(1)` | Exit immediately (may cut off pending output) |
| `process.cwd()` | The directory the command was run from |
| `process.stdout` / `stderr` | Standard output and error streams |

**Exit codes:** `0` means success, non-zero means failure. Shells and CI use them, so a CLI that
fails should not exit `0`. Print errors to `stderr` (`console.error`), results to `stdout`.

**Signals.** The OS asks a program to stop with signals: `SIGINT` (Ctrl+C) and `SIGTERM` (e.g. from
a process manager). By default Node exits; you can handle them to clean up first:

```js
process.on("SIGTERM", () => {
  server.close(() => process.exit(0));     // stop accepting connections, finish in-flight ones
});
```

---

## 7. Config, ergonomics and globals

### 7.1 Environment variables and `.env`

Config such as ports and credentials belongs in the environment, not in code. A `.env` file holds
local values and must be **git-ignored** (commit `.env.example`, as in M01).

```bash
node --env-file=.env app.js        # loads .env into process.env (no dotenv package needed)
node --watch app.js                # restarts when files change
node --run dev                     # runs the "dev" script from package.json (faster than npm run)
```

```js
const port = Number(process.env.PORT ?? 3000);     // env values are strings: convert, and use ?? not ||
```

### 7.2 Globals worth knowing

| Global | Notes |
| --- | --- |
| `fetch` | HTTP client (from M02) |
| `URL`, `URLSearchParams` | Parse and build URLs |
| `AbortController` | Cancellation (from M02) |
| `structuredClone` | Deep copy |

---

## 8. npm

### 8.1 `package.json` anatomy

```json
{
  "name": "log-report",
  "version": "1.0.0",
  "type": "module",
  "engines": { "node": ">=24" },
  "scripts": { "start": "node src/cli.js", "dev": "node --watch src/server.js" },
  "dependencies": {},
  "devDependencies": {}
}
```

| Field | Purpose |
| --- | --- |
| `type` | `"module"` makes `.js` files ES modules |
| `scripts` | Named commands: `npm run dev` (or `node --run dev`) |
| `dependencies` | Needed to run the app |
| `devDependencies` | Needed only to build/test (e.g. test tools, linters) |
| `engines` | The Node versions the project supports (advisory unless configured to enforce) |

### 8.2 Semver ranges

Versions are `MAJOR.MINOR.PATCH`. Major = breaking changes, minor = new features, patch = fixes.

| Range | Allows |
| --- | --- |
| `^1.2.3` | `>=1.2.3 <2.0.0` — minor and patch updates |
| `~1.2.3` | `>=1.2.3 <1.3.0` — patch updates only |
| `1.2.3` | Exactly that version |

### 8.3 The lockfile and the commands

`package-lock.json` records the **exact** resolved version of every dependency (and transitive ones).
Commit it so everyone, and CI, gets the same tree.

| Command | What it does |
| --- | --- |
| `npm install` | Install per `package.json`, updating the lockfile if needed |
| `npm install <pkg>` / `-D <pkg>` | Add a dependency / dev dependency |
| `npm ci` | Clean install **strictly from the lockfile**; fails if they disagree. Use in CI |
| `npm outdated` | Show which dependencies have newer versions |
| `npm update` | Update within the allowed ranges |
| `npm uninstall <pkg>` | Remove one |
| `npx <tool>` | Run a package's binary, downloading it temporarily if needed |
| `npm audit` | Report known vulnerabilities in your tree |

`node_modules` is generated: git-ignore it and never edit it.

---

## 9. Supply-chain hygiene

Every dependency is code you run with your permissions, and so are *its* dependencies. Habits:

- **Vet before you install.** Check the package name carefully (typosquatting — `lodahs` vs
  `lodash`), weekly downloads, maintainers, last publish date, repository link, open issues.
  Do you need a package at all when `node:` has the feature built in?
- **Review lockfile diffs.** A pull request that changes `package-lock.json` can add packages you
  never asked for; read the diff, don't just scroll past it.
- **Understand lifecycle scripts.** Packages can run commands on install (`preinstall`, `install`,
  `postinstall`). `npm install --ignore-scripts` skips them when you want to inspect first.
- **Run `npm audit`, but read the result.** Not every advisory is exploitable in your use; not
  every real risk has an advisory yet.
- **Pin with the lockfile, update deliberately.** Update on purpose and test, not blindly.

---

## 10. Debugging

### 10.1 `console`

| Method | Use |
| --- | --- |
| `console.log` / `console.error` | Normal output to stdout / errors to stderr |
| `console.table(arr)` | Tabular view of arrays of objects |
| `console.dir(obj, { depth: null })` | Print deeply nested objects fully |
| `console.time("x")` / `console.timeEnd("x")` | Quick timing |

### 10.2 The debugger

```bash
node --inspect-brk app.js        # pause on the first line and wait for a debugger
```

In VS Code, use **Run and Debug → JavaScript Debug Terminal**, then run `node app.js` and set
breakpoints in the editor. You can step, inspect variables and view the call stack.

### 10.3 Reading stack traces

Read from the **top**: the first line is the error message, the next frame is where it was thrown.
Find the first frame inside *your* code (not `node:internal/…`). For async errors the stack may be
short; an `Error` `cause` (from M02) shows the underlying failure.

### 10.4 Awareness only

Node ships a built-in test runner (`node:test`) and a built-in SQLite module (`node:sqlite`). This
course deliberately uses **Jest** and **Prisma** instead, so you don't need to use either here.

---

## 11. Thinking about the two programs

Questions only — answers are your design.

**`log-report` CLI**
- What does one line of the log look like, and which fields do you aggregate (e.g. counts by status
  code, by path, by hour)?
- How do you receive the input path and output path, and what happens if they're missing?
- What do you do with a line that doesn't parse?
- What does the JSON report contain, and how do you write it (once at the end)?
- How will you demonstrate memory behaviour on the full file, and what would the non-streaming
  version have shown?
- What exit code do you return on failure?

**Bare `node:http` server**
- Which three JSON endpoints, and which methods?
- How do you structure routing so it is readable, without a framework?
- How do you read and validate the body safely?
- What does a consistent error response look like, and how do you avoid repeating it?

**Delivery**
- Both programs go to `main` through a pull request, with Conventional Commits (M01). Plan your
  branch and commits before you start.

---

## 12. Guided practice (throwaway scripts)

Use a scratch folder, **not** your lab repo, and only small generated data:

```bash
mkdir node-sandbox && cd node-sandbox && npm init -y
```

### Drill 1 — Blocking vs non-blocking
Write a `setInterval` that logs a tick every 100 ms. Add a synchronous busy loop for 1 second and
watch the ticks stop. Replace it with `await sleep(1000)` from `node:timers/promises` and compare.

### Drill 2 — Modules
Create an ESM project (`"type": "module"`), import a helper with a relative path, and print
`import.meta.dirname`. Then try `require` and read the error message.

### Drill 3 — Generate and stream
Write a script that generates a file of 200,000 short lines. Read it three ways — `readFile`,
`createReadStream` with `for await`, and `readline` — counting lines. Print `process.memoryUsage()`
before and after each. Which is smallest, and why? Why can chunk counting differ from line counting?

### Drill 4 — `parseArgs`
Write a tiny script accepting `--name`, `-v` and a positional argument. Run it with an unknown flag
and read the error. Set `process.exitCode` to `1` when an argument is missing and check it with
`echo $?` (or `$LASTEXITCODE` in PowerShell).

### Drill 5 — Signals
Start a `setInterval` and handle `SIGINT` by printing a message, clearing the timer and letting the
process exit. Press Ctrl+C. Then remove the handler and compare.

### Drill 6 — Hello server
Build a server that returns JSON for `GET /ping`, a 404 JSON for other paths, and echoes the length
of a posted body for `POST /echo`. Test with Bruno or `curl`. Send invalid JSON to a route that
parses it and see what happens.

### Drill 7 — npm
Install a small, well-known package. Inspect `package.json` and `package-lock.json`. Delete
`node_modules` and run `npm ci`. Edit `package.json` to disagree with the lockfile and run `npm ci`
again. Run `npm outdated` and `npm audit`.

### Drill 8 — Debugging
Write a script with a deliberate bug in a loop. Start it with `--inspect-brk`, attach VS Code, set a
breakpoint, and find the bad value by stepping.

---

## 13. Knowledge check

Answer without looking at the material first. Answers to questions 1–10 are at the end; questions
11–13 are open-ended and have **no answer key** because they feed your write-up.

### Multiple choice

**1.** Which component provides Node's event loop and thread pool?
- a) V8  b) libuv  c) npm  d) `node:http`

**2.** Why is `readFileSync` on a busy server a problem?
- a) It uses too much disk space
- b) It blocks the event loop, so no other request runs meanwhile
- c) It can't read text files
- d) It only works on Windows

**3.** How do you get the directory of the current file in an ES module?
- a) `__dirname`  b) `process.cwd()`  c) `import.meta.dirname`  d) `require.main`

**4.** What does the `node:` prefix guarantee?
- a) A faster import
- b) The import refers to the built-in module, never a package from `node_modules`
- c) The module is ESM only
- d) The module is async

**5.** Why prefer a stream over `readFile` for a 100 MB log?
- a) Streams are always faster
- b) Memory use stays roughly constant instead of growing with the file
- c) `readFile` cannot read large files
- d) Streams parse JSON automatically

**6.** What does `^1.4.2` allow?
- a) Only `1.4.2`  b) `>=1.4.2 <1.5.0`  c) `>=1.4.2 <2.0.0`  d) Any version

**7.** When should CI use `npm ci` instead of `npm install`?
- a) Always, for reproducible installs strictly from the lockfile
- b) Only on Windows
- c) Only when adding a package
- d) Never

**8.** In `process.argv`, where do your own command-line arguments start?
- a) Index 0  b) Index 1  c) Index 2  d) Index 3

**9.** What exit code signals success?
- a) `0`  b) `1`  c) `-1`  d) `255`

**10.** What does an incoming `http` request (`req`) look like to your code?
- a) A parsed JSON object
- b) A readable stream you must read the body from
- c) A string
- d) A Promise

### Short answer / explain

**11.** Explain, in your own words, why loading a 100 MB file with `readFile` is wasteful and what
a stream changes. How did you confirm the CLI really streams?

**12.** From the bare `node:http` server you wrote, list specific things a framework like Express
would take off your hands, and say which one hurt the most.

**13.** A teammate suggests adding an unfamiliar npm package that has a `postinstall` script. What
do you check before installing it, and why?

<details>
<summary>Answer key (questions 1–10 and drill hints only)</summary>

**Multiple choice:** 1-b, 2-b, 3-c, 4-b, 5-b, 6-c, 7-a, 8-c, 9-a, 10-b.

**Drill hints (verify by running them):**

- Drill 1: the interval stops ticking during the synchronous loop but keeps ticking during `await`.
- Drill 2: `require` is not defined in an ES module scope.
- Drill 3: `readFile` shows the largest memory jump; chunks are arbitrary-size Buffers that can split
  a line, which is what `readline` fixes.
- Drill 4: `parseArgs` throws on unknown options unless `strict: false`.
- Drill 5: with a handler, Node waits for you to exit; without one, it exits immediately.
- Drill 6: an unguarded `JSON.parse` on bad input throws inside the handler.
- Drill 7: `npm ci` errors when `package.json` and the lockfile disagree.

</details>

---

## 14. Common pitfalls and error messages

| Symptom | Likely cause / fix |
| --- | --- |
| `ReferenceError: require is not defined` | You are in an ES module; use `import`. |
| `ReferenceError: __dirname is not defined` | Use `import.meta.dirname` in ESM. |
| `ERR_MODULE_NOT_FOUND` | Missing file extension in a relative import. |
| `ENOENT: no such file or directory` | Wrong path; build it from `import.meta.dirname` or check `process.cwd()`. |
| `EADDRINUSE` | Another process holds the port; stop it or pick another port. |
| Server hangs on a request | A code path never calls `res.end()`. |
| Memory keeps growing with file size | You are loading everything (e.g. `readFile`, or collecting all lines in an array). |
| Unhandled `'error'` event on a stream | Attach an error handler or use `pipeline`. |
| Exit code is `0` after a failure | Set `process.exitCode = 1` in the error path. |
| `process.env.PORT` behaves like text | Env values are strings; convert with `Number(...)`. |
| `npm ci` fails | `package.json` and the lockfile disagree; run `npm install` and commit the lockfile. |
| Output order surprises | Revisit M02's event-loop section. |

## 15. Further reading

- Node.js documentation: <https://nodejs.org/api/>
- Node.js — Learn (guides): <https://nodejs.org/en/learn>
- Node.js — Backpressuring in Streams: <https://nodejs.org/en/learn/modules/backpressuring-in-streams>
- Node.js — `util.parseArgs`: <https://nodejs.org/api/util.html#utilparseargsconfig>
- npm documentation: <https://docs.npmjs.com>
- Semantic Versioning: <https://semver.org>
- Node.js — Debugging guide: <https://nodejs.org/en/learn/getting-started/debugging>

## 16. Ready for the lab?

Before starting the lab in `brief.md`, you should be able to:

- [ ] Explain V8, libuv, the event loop and the thread pool, and why blocking is harmful.
- [ ] Use ES modules with `import.meta.dirname` and the `node:` prefix.
- [ ] Read a file line by line without loading it all into memory, and measure the difference.
- [ ] Write a bare HTTP server that reads a JSON body and returns correct status codes.
- [ ] Parse arguments with `parseArgs` and exit with a meaningful code.
- [ ] Explain `^`/`~`, the lockfile, and `npm ci` vs `npm install`.
- [ ] Say what you check before adding a dependency.
- [ ] Have the trainer's log file (or generator) on hand.

Then proceed to `tasks.md` and fill in `write-up-template.md` as you build.
