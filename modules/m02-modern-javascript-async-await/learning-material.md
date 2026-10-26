# M02 Learning Material — Modern JavaScript & Async/Await

> **Companion to `brief.md` and `tasks.md`.** The brief tells you *what* to deliver; this document
> teaches the *concepts* you need to get there. It deliberately does **not** contain the finished
> lab: you still write `sleep`, `retry`, `withTimeout` and `mapLimit` yourself, capture your own
> timing numbers, refactor your own `node:fs` code, and explain the output order in your own words.
> Read it, try the drills, then do the lab.

**Week 1 · Days 2-4 · ~10 hours**

## 0. How to use this material

| Section | Maps to brief Scope bullet | Maps to Definition of done | Suggested time |
| --- | --- | --- | --- |
| 1. Language essentials | Language essentials | — | 2 h |
| 2. How async JavaScript works | Call stack, event loop, queues | Output-order explanation | 1 h |
| 3. Callbacks → Promises → async/await | Callbacks → Promises → `async`/`await` | Callback-to-async refactor | 1 h |
| 4. Concurrency and combinators | `Promise.all`, `allSettled`, `race`, `any` | Parallel beats sequential | 45 min |
| 5. Modern async APIs | `withResolvers`, `Promise.try`, async iteration | — | 30 min |
| 6. Timeouts, cancellation and fetch | `AbortController`, `AbortSignal.timeout`, `fetch` | Timing comparison | 45 min |
| 7. Async pitfalls | Pitfalls | All self-checks pass | 30 min |
| 8. Thinking about the toolkit | Deliverable | All self-checks pass | 30 min |
| 9. Guided practice | all | — | 1.5 h |
| 10. Knowledge check | all | — | 30 min |

Suggested flow: read sections 1–8, do the drills in 9, take the check in 10, then start the lab in
`brief.md`. The lab itself is where most of your 10 hours go, so move through the reading briskly
and come back to sections as you need them.

**Learning outcomes.** After this module you can:

- Write idiomatic modern JavaScript: destructuring, spread, optional chaining, array helpers,
  `Map`/`Set`, classes with private fields.
- Define custom error classes and chain failures with `cause`.
- Describe the call stack, the task queue and the microtask queue, and predict output order.
- Move between callbacks, Promises and `async`/`await`, and handle errors in each.
- Choose between sequential and parallel execution and between `all`, `allSettled`, `race` and `any`.
- Add timeouts and cancellation with `AbortController` and `AbortSignal.timeout`.
- Recognise and avoid the classic async pitfalls.

**Runtime note.** The cohort uses **Node.js 24 LTS** (see M01). Every API in this module — including
`Promise.try`, `Promise.withResolvers`, `Array.fromAsync`, `Object.groupBy` and the new `Set`
methods — is available there. If something is "not a function", check `node --version` first.

---

## 1. Language essentials

Run the snippets in a scratch file (`node scratch.mjs`) or the Node REPL (`node`). Predict the
result *before* running.

### 1.1 `const`, `let`, scope, closures, arrow functions, template literals

- `const` binds a name that cannot be reassigned. It does **not** make the value immutable: a
  `const` array can still be pushed to.
- `let` is block-scoped and reassignable. Prefer `const`, use `let` only when you reassign. Avoid
  `var` (function-scoped, hoisted).
- A **closure** is a function that remembers the variables of the scope where it was created, even
  after that scope has finished.

```js
function makeCounter() {
  let count = 0;
  return () => ++count;   // closes over `count`
}
const next = makeCounter();
next(); next();           // each call sees the same private `count`
```

- **Arrow functions** are shorter and do not have their own `this`: they use the `this` of the
  surrounding code. That makes them good for callbacks and bad for object methods that need
  `this` to be the object.
- **Template literals** use backticks: `` `Hello ${name}, ${items.length} items` ``. They also allow
  multi-line strings.

```js
const timer = {
  label: "job",
  regular() { return this.label; },        // `this` is `timer` when called as timer.regular()
  arrow: () => typeof this,                 // `this` is NOT `timer` here
};
```

### 1.2 Destructuring, spread/rest, defaults, `?.`, `??`

```js
const { id, title = "untitled", ...rest } = post;     // object destructuring + default + rest
const [first, , third] = list;                          // array destructuring, skip one
const merged = { ...defaults, ...overrides };           // spread: later keys win
const all = [...a, ...b];
function connect({ host = "localhost", port = 3000 } = {}) { /* ... */ }
```

| Operator | Meaning | Example |
| --- | --- | --- |
| `?.` | Stop and return `undefined` if the left side is `null`/`undefined` | `user?.address?.city` |
| `??` | Use the right side only for `null`/`undefined` | `limit ?? 10` |
| `\|\|` | Use the right side for *any falsy* value | `limit \|\| 10` (replaces `0`!) |
| `??=` | Assign only if the left side is `null`/`undefined` | `opts.retries ??= 3` |

**Truthy/falsy pitfalls.** The falsy values are `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`,
`NaN`. Empty arrays `[]` and objects `{}` are **truthy**. Prefer `===`/`!==`; loose `==` coerces
types (`0 == ""` is `true`, `null == undefined` is `true`). Use `Number.isNaN(x)`, because
`NaN === NaN` is `false`.

> **Why this matters for the lab.** A function like `retry(fn, { retries, delayMs })` has to decide
> what `retries: 0` means. `||` and `??` give different answers. Decide deliberately.

### 1.3 Array methods and friends

| Method | Returns | Mutates? |
| --- | --- | --- |
| `map(fn)` | New array of transformed items | No |
| `filter(fn)` | New array of items where `fn` is truthy | No |
| `reduce(fn, init)` | A single accumulated value | No |
| `find(fn)` / `findLast` | First / last matching item or `undefined` | No |
| `some(fn)` / `every(fn)` | Boolean | No |
| `flatMap(fn)` | Map then flatten one level | No |
| `toSorted(cmp)` / `toReversed()` / `with(i, v)` | Copies with the change applied | **No** |
| `sort(cmp)` / `reverse()` / `splice()` | The same array | **Yes** |

Always pass a comparator to sort numbers: `[10, 9, 1].toSorted((a, b) => a - b)`. The default sort
compares strings.

```js
structuredClone(obj);                                   // deep copy (Dates, Maps, nested objects; not functions)
Object.groupBy(users, (u) => u.role);                   // { admin: [...], user: [...] }  (plain object)
Map.groupBy(users, (u) => u.active);                    // Map { true => [...], false => [...] }
```

### 1.4 `Map`, `Set`, classes, private fields

- `Map` keeps insertion order, accepts **any** value as a key, and has `.size`. Use it instead of a
  plain object when keys are dynamic or not strings.
- `Set` stores unique values. Besides `add/has/delete`, modern Node has set algebra:
  `a.union(b)`, `a.intersection(b)`, `a.difference(b)`, `a.isSubsetOf(b)`.

```js
class Account {
  #balance = 0;                       // truly private: not accessible outside the class
  constructor(owner) { this.owner = owner; }
  get balance() { return this.#balance; }
  set balance(v) { if (v < 0) throw new RangeError("negative"); this.#balance = v; }
  deposit(n) { this.#balance += n; return this; }
}
```

`#balance` is enforced by the language (unlike an `_underscore` convention). Getters/setters look
like properties but run code.

### 1.5 Errors: `try/catch/finally`, custom classes, `cause`

```js
class HttpError extends Error {
  constructor(status, message, options) {
    super(message, options);          // options can carry { cause }
    this.name = "HttpError";
    this.status = status;
  }
}

try {
  try {
    JSON.parse("{ not json");
  } catch (err) {
    throw new HttpError(400, "Bad payload", { cause: err });   // keep the original failure
  }
} catch (err) {
  console.log(err.name, err.message, err.cause?.message);
} finally {
  console.log("runs whether or not something threw");
}
```

- Throw `Error` objects (not strings) so you get a stack trace.
- `instanceof` lets callers branch on error type: `if (err instanceof HttpError) ...`.
- `cause` preserves the underlying error so you do not lose the root problem when wrapping.
- `finally` is for cleanup; it runs even if `try` returns or throws.

### 1.6 ES modules: named vs default exports

```js
// math.mjs
export const add = (a, b) => a + b;        // named export
export default function multiply(a, b) { return a * b; }

// app.mjs
import multiply, { add } from "./math.mjs";
import * as math from "./math.mjs";        // namespace import
```

- A file is an ES module if it ends in `.mjs` **or** the nearest `package.json` has
  `"type": "module"`. Otherwise Node treats `.js` as CommonJS.
- Relative imports need the **file extension** in Node (`./math.mjs`, not `./math`).
- **Named exports** are explicit, autocomplete well and refactor safely; many teams prefer them.
  **Default exports** let the importer pick any name, which makes search harder. Choose one
  convention for the toolkit and be ready to justify it in your write-up.
- Top-level `await` is allowed in ES modules.

---

## 2. How async JavaScript works

### 2.1 One thread, one call stack

JavaScript runs your code on a single thread with a **call stack**: calling a function pushes a
frame, returning pops it. While a long synchronous function runs, *nothing else* runs — no timers,
no I/O callbacks, no promise handlers. That is why a `while` loop that spins for 5 seconds freezes a
server.

### 2.2 The event loop and its two queues

Async work (timers, file reads, network) is performed *outside* your JavaScript stack, by the
runtime. When it finishes, a callback is queued. The **event loop** moves queued callbacks onto the
stack, but only when the stack is empty.

```
            ┌─────────────────────────────┐
            │         Call stack          │  ← runs your synchronous code
            └──────────────┬──────────────┘
                           │ stack empty?
        ┌──────────────────┴───────────────────┐
        ▼                                      ▼
┌──────────────────┐                ┌──────────────────────┐
│ Microtask queue  │  drained FIRST │  Task (macro) queue  │  one task per loop turn
│ promise .then,   │  completely    │  setTimeout, I/O,    │
│ await continuation│               │  setImmediate        │
│ queueMicrotask   │                └──────────────────────┘
└──────────────────┘
```

The rule of thumb:

1. Run the current synchronous code to completion.
2. Drain the **entire microtask queue** (including microtasks queued by microtasks).
3. Take **one** task from the task queue, run it, then go to step 2.

So promise callbacks and code after an `await` always run **before** the next `setTimeout`
callback, even `setTimeout(fn, 0)`.

> Node also has `process.nextTick`, which runs even before promise microtasks. You do not need it
> here; just know it exists. Event-loop phases in depth come in M03.

### 2.3 What `await` really does

`await p` pauses the *async function* (not the thread). The rest of the function is scheduled as a
microtask continuation once `p` settles; the caller continues immediately. An `async` function
always returns a Promise.

### 2.4 Worked example — reason through, don't memorise

```js
console.log("A");
setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => console.log("C"));
(async () => {
  console.log("D");
  await null;
  console.log("E");
})();
console.log("F");
```

Walk through it with these steps:

1. Which lines are synchronous and run right now, in order? (Remember the body of an `async`
   function runs synchronously until its first `await`.)
2. What is sitting in the microtask queue when the script ends, and in what order was it queued?
3. What is in the task queue?
4. Apply the rule: drain microtasks, then take one task.

Write your predicted order, then run it. Part of the Definition of done is explaining a snippet like
this *out loud*, so practise the narration: "synchronous first, then microtasks in the order they
were queued, then timers."

---

## 3. Callbacks → Promises → async/await

### 3.1 Callbacks and "callback hell"

Node's original style is **error-first callbacks**: `(err, result) => { ... }`.

```js
import fs from "node:fs";

fs.readFile("a.json", "utf8", (err, text) => {
  if (err) return handle(err);
  fs.readFile("b.json", "utf8", (err2, text2) => {
    if (err2) return handle(err2);
    // ...the pyramid grows with each dependent step
  });
});
```

Problems: nesting, repeated error checks, hard to run in parallel, easy to call a callback twice or
never.

### 3.2 Promises

A **Promise** is an object for a value that will exist later. States: *pending* → *fulfilled* or
*rejected* (settled once, never changes).

```js
fetchSomething()
  .then((value) => transform(value))      // returning a value → next .then receives it
  .then((next) => save(next))             // returning a Promise → chain waits for it
  .catch((err) => report(err))            // catches a rejection from any step above
  .finally(() => cleanup());              // runs either way; its return value is ignored
```

Key points:

- `.then` returns a **new** promise, which is what makes chaining work.
- A thrown error inside `.then` becomes a rejection of the returned promise.
- Forgetting to `return` inside a `.then` breaks the chain (next step gets `undefined`).
- `new Promise((resolve, reject) => { ... })` wraps callback-style APIs. Call `resolve` or `reject`
  exactly once.

### 3.3 Converting a callback API

Two ways to turn error-first callback code into promise code — learn both, pick for the lab:

- Wrap manually with `new Promise` (good for understanding).
- Use what Node already provides: the promise-based API `import fs from "node:fs/promises"`, or
  `util.promisify` for callback functions.

### 3.4 `async` / `await`

```js
import { readFile } from "node:fs/promises";

async function loadConfig(path) {
  try {
    const text = await readFile(path, "utf8");
    return JSON.parse(text);
  } catch (err) {
    throw new Error(`Cannot load ${path}`, { cause: err });
  } finally {
    // cleanup
  }
}
```

- `await` can only be used inside an `async` function (or at the top level of an ES module).
- `try/catch` around `await` catches **rejections** as if they were thrown errors.
- A rejection nobody handles becomes an `unhandledRejection`, which **crashes Node** by default.
- Returning a value from an `async` function resolves its promise; throwing rejects it.

### 3.5 Error handling differences

| Style | How a failure is signalled | How you handle it |
| --- | --- | --- |
| Callback | First argument `err` | `if (err)` in every callback |
| Promise | Rejection | `.catch()` somewhere on the chain |
| `async`/`await` | Rejection → thrown at the `await` | `try/catch` |

A `try/catch` does **not** catch an error thrown inside a plain callback (e.g. inside `setTimeout`)
because that callback runs later, on a different stack.

---

## 4. Concurrency and combinators

### 4.1 Sequential vs parallel

```js
// Sequential: each request waits for the previous one. Total ≈ sum of durations.
for (const id of ids) results.push(await getItem(id));

// Parallel: start everything first, then wait. Total ≈ the slowest one.
const results = await Promise.all(ids.map((id) => getItem(id)));
```

Why is parallel faster for independent network calls? Because the waiting time is spent *outside*
JavaScript; starting all requests before awaiting lets the waits overlap. Parallel is **not** better
when steps depend on each other, or when firing hundreds of requests at once would overload the
server — that is the problem a concurrency limit solves.

### 4.2 The four combinators

| Combinator | Fulfils when | Rejects when | Result | Use it when |
| --- | --- | --- | --- | --- |
| `Promise.all` | **All** fulfil | **Any** rejects (fails fast) | Array of values, same order as input | All results are required |
| `Promise.allSettled` | **All** settle | Never | Array of `{ status, value \| reason }` | You want every outcome, even failures |
| `Promise.race` | The **first** settles | The first settles as a rejection | That first outcome | Timeouts, "first response wins" |
| `Promise.any` | The **first fulfils** | **All** reject (`AggregateError`) | First fulfilled value | Redundant sources; ignore failures |

Notes:

- `Promise.all` keeps **input order** in its result, regardless of completion order.
- When `Promise.all` rejects, the other promises are *not cancelled* — they keep running; you just
  stop waiting for them.
- `Promise.race` of an empty array never settles.

---

## 5. Modern async APIs

### 5.1 `Promise.withResolvers`

Gives you a promise plus its `resolve`/`reject` functions without nesting everything in an executor
callback:

```js
const { promise, resolve, reject } = Promise.withResolvers();
emitter.once("ready", resolve);
emitter.once("error", reject);
await promise;
```

### 5.2 `Promise.try`

Starts a chain from a function that may throw synchronously **or** return a promise, treating both
the same:

```js
Promise.try(() => maybeSyncOrAsync(input))
  .then(handleResult)
  .catch(handleError);       // a synchronous throw also lands here
```

### 5.3 Async iteration

```js
async function* pages() {            // async generator: yields values over time
  for (let n = 1; n <= 3; n++) {
    yield await loadPage(n);
  }
}

for await (const page of pages()) {  // waits for each value in turn
  console.log(page);
}

const items = await Array.fromAsync(pages());   // collect an async iterable into an array
```

`for await…of` also works on a plain array of promises and on Node streams (used later).

---

## 6. Timeouts, cancellation and `fetch`

### 6.1 Cancelling work

A promise by itself **cannot be cancelled**. Cancellation is cooperative: the work must accept an
`AbortSignal` and stop when it fires.

```js
const controller = new AbortController();
const { signal } = controller;

fetch(url, { signal }).catch((err) => {
  if (err.name === "AbortError") console.log("cancelled");
});

controller.abort();                  // from somewhere else: ask the work to stop
```

- `AbortSignal.timeout(ms)` gives a signal that aborts itself after `ms` milliseconds (the error
  name is `TimeoutError`).
- `AbortSignal.any([a, b])` combines signals.
- Many Node APIs accept `{ signal }` — including `fetch`, and `setTimeout` from
  `node:timers/promises`.

### 6.2 Timeouts: reject vs actually stop

Racing a promise against a timer *rejects the caller* after `ms`, but the original work may keep
running in the background, holding sockets, timers and memory. Ask yourself, when you build
`withTimeout`:

- Does a timer left running keep the process alive after the work finishes?
- Should the underlying operation be cancelled too, and how would it know?

### 6.3 `fetch` and JSON

`fetch` is built into Node 24.

```js
const response = await fetch("https://jsonplaceholder.typicode.com/posts/1", {
  signal: AbortSignal.timeout(5000),
});
if (!response.ok) {                         // 404/500 do NOT reject on their own!
  throw new Error(`HTTP ${response.status}`);
}
const post = await response.json();         // parsing is async too
```

Important details:

- `fetch` only rejects on **network failure** or abort. An HTTP 404 resolves successfully with
  `response.ok === false`.
- `response.json()` and `response.text()` return promises; the body can be read **once**.
- To send JSON: `method: "POST"`, `headers: { "content-type": "application/json" }`,
  `body: JSON.stringify(data)`.

### 6.4 Measuring time

For your timing comparison, use a monotonic clock:

```js
const start = performance.now();
// ...work...
const elapsedMs = performance.now() - start;
```

`Date.now()` can jump if the system clock changes; `performance.now()` cannot. Run each variant a
few times — the first request includes DNS and connection set-up, so a single measurement can
mislead. Report your actual numbers and what you ran.

### 6.5 Awareness: Temporal

`Date` is famously awkward (mutable, month index from 0, weak time zone support). **Temporal** is the
upcoming replacement. You do not need to use it in this module; just know it exists so you are not
surprised when you see it.

---

## 7. Async pitfalls

| Pitfall | What it looks like | Why it bites | Fix |
| --- | --- | --- | --- |
| **Forgotten `await`** | `const user = getUser(id); user.name` → `undefined` | `user` is a pending Promise, not the value | `await` it, or lint with `@typescript-eslint/no-floating-promises` later |
| **`await` in a loop** | `for (...) { await fetchItem(i) }` for independent items | Runs one at a time; slow | `Promise.all(items.map(...))`, or a concurrency limit |
| **`forEach(async …)`** | `items.forEach(async (i) => { await save(i) }); done()` | `forEach` ignores returned promises; `done()` runs too early and errors are lost | `for…of` with `await`, or `await Promise.all(items.map(...))` |
| **Unhandled rejection** | A promise rejects with no `.catch` / `try/catch` | Node terminates with an error | Always await or catch; add `.catch` to fire-and-forget calls |
| **Swallowed error** | `catch (e) {}` or `catch (e) { console.log(e) }` then continuing as if fine | Failure hidden; later code works on bad data | Handle it or rethrow (with `cause`) |
| **Catch that returns nothing** | `.catch(log)` at the end of a chain whose callers expect rejection | The chain now *fulfils* with `undefined` | Rethrow when callers need to know |
| **Creating the promise too early** | `const p = work(); await sleep(1000); await p;` | `work()` started immediately; its rejection may go unhandled while you wait | Create the promise where you await it, or attach a handler |
| **Missing `response.ok` check** | Parsing an error page as JSON | `fetch` doesn't reject on 4xx/5xx | Check `ok` before `json()` |

Practise spotting each one before you build; your write-up asks which one you designed around.

---

## 8. Thinking about the toolkit

These are **questions to answer yourself**, not solutions. Sketch your answers before you code, then
check them against your self-checks.

**`sleep(ms)`**
- What does it return? What should happen if `ms` is `0`, negative, or not a number?
- Could it accept an `AbortSignal` so a caller can stop waiting?

**`retry(fn, { retries, delayMs })`**
- Does `retries: 3` mean 3 attempts total or 1 + 3? Write the definition down and test it.
- What happens when all attempts fail — which error is thrown, and can you keep the others
  reachable (hint: `cause`, `AggregateError`)?
- Do you wait after the *last* failure? What is the default for missing options?
- How will your self-check prove that a function failing twice then succeeding is called exactly
  three times, without waiting real seconds?

**`withTimeout(promise, ms)`**
- What error is thrown on timeout, and how can a caller tell it from the promise's own failure?
- If the promise settles first, is the timer cleared? If the timeout fires first, what happens to
  the promise's later result or rejection?
- How would you assert the timeout case without making the test slow?

**`mapLimit(items, limit, fn)`**
- At most `limit` calls of `fn` are in flight at once — how will you *prove* that in a self-check?
- Results must line up with the input order even though calls finish in a different order.
- What happens if one call rejects? Do you stop starting new ones? Document your decision.
- What do `limit` values of `0`, `1`, and larger than `items.length` mean?
- Which pitfall from section 7 (`forEach(async …)`, `await` in a loop) would give you a wrong
  design here?

**Self-checks with `node:assert`**

```js
import assert from "node:assert/strict";

assert.equal(1 + 1, 2);
assert.deepEqual([1, 2], [1, 2]);
await assert.rejects(somePromise, { name: "TimeoutError" });   // asserts a promise rejects
await assert.doesNotReject(otherPromise);
assert.throws(() => badSyncCall(), /message/);
```

Use `assert/strict` so `equal` means `===`. Keep delays tiny (10–50 ms) in checks so the whole file
runs in a second or two, and make each check print a line when it passes so you can see what ran.

---

## 9. Guided practice (throwaway scripts)

Do these in a scratch folder, **not** in your lab file. Create one:

```bash
mkdir async-sandbox && cd async-sandbox
```

### Drill 1 — Scope and closures
Write a function `makeLogger(prefix)` that returns a function logging `prefix: message`. Create two
loggers with different prefixes and confirm each remembers its own prefix.

### Drill 2 — Destructuring and defaults
Write a function taking `{ retries, delayMs }` with defaults `3` and `100`. Call it with `{}`, with
`{ retries: 0 }`, and with nothing. Compare the result of using `??` versus `||` for the defaults.

### Drill 3 — Custom errors
Create `class ValidationError extends Error` with a `field` property. Throw it wrapped around a
`JSON.parse` failure using `cause`. Print the whole chain with `err.cause`.

### Drill 4 — Predict the order
Write a script mixing `console.log`, `setTimeout(…, 0)`, `Promise.resolve().then`, `queueMicrotask`
and an `async` function with two `await`s. Write your prediction on paper first, then run it. Repeat
with a deliberately different arrangement until you predict correctly three times in a row.

### Drill 5 — Callbacks to promises
Using a throwaway text file (not the lab's `fs` code), read it with `fs.readFile` and a callback;
then rewrite it three ways: manual `new Promise` wrapper, `util.promisify`, and `node:fs/promises`
with `async`/`await`. Add a missing-file case and handle the error in each style.

### Drill 6 — Combinators
Create three promise-returning helpers that resolve or reject after different delays. Run them
through `all`, `allSettled`, `race` and `any` with (a) all succeeding, (b) one failing quickly,
(c) all failing. Record the result of each of the 12 runs in a small table.

### Drill 7 — Reproduce the pitfalls
Write a failing example of each: forgotten `await`, `forEach(async …)` (show that the "done" line
prints first), a swallowed error, and an unhandled rejection (watch Node's output). Then fix each.

### Drill 8 — Cancellation
Start a `fetch` to JSONPlaceholder with `AbortSignal.timeout(1)` and catch the `TimeoutError`. Then
create an `AbortController`, start a `fetch`, call `abort()` after a few milliseconds, and compare
the error `name` you see.

### Drill 9 — Modern APIs
Rewrite the callback event wait from §5.1 using `Promise.withResolvers`. Then write an async
generator that yields 1, 2, 3 with a short delay between values and consume it with `for await` and
with `Array.fromAsync`.

---

## 10. Knowledge check

Answer without looking at the material first. Answers to questions 1–10 are at the end; questions
11–13 are open-ended and have **no answer key** because they feed your write-up.

### Multiple choice

**1.** Which expression returns `10` when `limit` is `0`?
- a) `limit || 10`  b) `limit ?? 10`  c) `limit ?. 10`  d) `limit && 10`

**2.** What does `Object.groupBy(users, fn)` return?
- a) A `Map`  b) An array of arrays  c) A plain object of arrays  d) A `Set`

**3.** Which array method does **not** mutate the original array?
- a) `sort`  b) `reverse`  c) `splice`  d) `toSorted`

**4.** In what order does this print?
```js
setTimeout(() => console.log("T"), 0);
Promise.resolve().then(() => console.log("P"));
console.log("S");
```
- a) T, P, S  b) S, T, P  c) S, P, T  d) P, S, T

**5.** `await somePromise` inside an `async` function does what?
- a) Blocks the whole thread until it settles
- b) Pauses only that function and lets other code run
- c) Cancels the promise if it takes too long
- d) Converts the promise into a callback

**6.** One of five `Promise.all` inputs rejects. What happens?
- a) It waits for all five, then returns the failures
- b) It rejects with that error, and the others keep running
- c) It cancels the other four
- d) It returns `undefined` for the failed slot

**7.** Which combinator gives you the outcome of *every* promise, including failures, without ever
rejecting?
- a) `all`  b) `race`  c) `any`  d) `allSettled`

**8.** `fetch("/missing")` gets an HTTP 404 response. What happens?
- a) The promise rejects with an `HttpError`
- b) The promise fulfils with `response.ok === false`
- c) The promise never settles
- d) Node throws synchronously

**9.** Why is `items.forEach(async (i) => { await save(i); })` a bug if you need all saves to finish?
- a) `forEach` doesn't exist on arrays
- b) `await` isn't allowed in arrow functions
- c) `forEach` ignores the returned promises, so nothing waits for them
- d) It runs the saves in reverse order

**10.** What is the safest way to give a caller a way to cancel an operation?
- a) Return a promise with a `.cancel()` method
- b) Accept an `AbortSignal` and stop when it fires
- c) Call `process.exit()`
- d) Wrap it in `try/catch`

### Short answer / explain

**11.** Explain, step by step and in your own words, the output order of a snippet that mixes
`setTimeout`, a promise `.then`, and an `await`. Name the queue each callback waits in.

**12.** You must fetch 20 independent resources. Compare doing it sequentially, with `Promise.all`,
and with a concurrency limit of 5. When would you choose each, and what are the risks of each?

**13.** Which async pitfall from section 7 are you most likely to hit in this module's lab, and what
will you do in your design to avoid it?

<details>
<summary>Answer key (questions 1–10 and drill hints only)</summary>

**Multiple choice:** 1-a, 2-c, 3-d, 4-c, 5-b, 6-b, 7-d, 8-b, 9-c, 10-b.

**Drill hints (verify by running them):**

- Drill 2: `||` replaces `0` with the default; `??` keeps `0`.
- Drill 3: `new ValidationError("bad", { cause: err })` — make sure your constructor forwards the
  options to `super`.
- Drill 4: synchronous logs first, then everything queued as microtasks (in queue order), then
  timers.
- Drill 5: `util.promisify(fs.readFile)` works because `readFile` uses error-first callbacks.
- Drill 6: `all` rejects fast on (b) and (c); `any` rejects only in (c) with an `AggregateError`;
  `race` follows whichever settles first.
- Drill 7: the "done" line from `forEach(async …)` prints before the saves finish.
- Drill 8: a timeout-triggered abort has `name === "TimeoutError"`; a manual `abort()` gives
  `"AbortError"`.

</details>

---

## 11. Common pitfalls and error messages

| Symptom | Likely cause / fix |
| --- | --- |
| `SyntaxError: Cannot use import statement outside a module` | File is CommonJS; rename to `.mjs` or set `"type": "module"` in `package.json`. |
| `ERR_MODULE_NOT_FOUND` for a relative import | Missing file extension in the import path. |
| `SyntaxError: await is only valid in async functions` | Use `await` inside an `async` function or at module top level. |
| Output is `Promise { <pending> }` | You forgot to `await`. |
| `UnhandledPromiseRejection` crash | A rejection with no handler; add `try/catch` or `.catch`. |
| `TypeError: x is not a function` for `Promise.try` etc. | Old Node version; run `node --version` and switch to 24. |
| `TypeError: fetch failed` | Network failure; check connection, proxy, and the URL. `err.cause` has details. |
| Script hangs after finishing | A timer or open handle is keeping Node alive; make sure timers are cleared. |
| Timing results vary wildly | Single runs include connection set-up; repeat and compare medians. |
| `AssertionError` message hard to read | Use `assert/strict` and pass a message as the last argument. |

## 12. Further reading

- MDN — JavaScript Guide and Reference: <https://developer.mozilla.org/docs/Web/JavaScript>
- MDN — Using Promises: <https://developer.mozilla.org/docs/Web/JavaScript/Guide/Using_promises>
- MDN — `AbortSignal`: <https://developer.mozilla.org/docs/Web/API/AbortSignal>
- javascript.info — Promises, async/await: <https://javascript.info/async>
- Node.js — The event loop, timers and `nextTick`:
  <https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick>
- Node.js — `assert` module: <https://nodejs.org/api/assert.html>
- JSONPlaceholder: <https://jsonplaceholder.typicode.com>

## 13. Ready for the lab?

Before starting the lab in `brief.md`, you should be able to:

- [ ] Use destructuring, defaults, `?.` and `??` and say when `??` differs from `||`.
- [ ] Write a custom error class and wrap a failure with `cause`.
- [ ] Explain the call stack, task queue and microtask queue, and predict output order.
- [ ] Convert callback code to promises and to `async`/`await`.
- [ ] Choose between `all`, `allSettled`, `race` and `any`, and between sequential and parallel.
- [ ] Add a timeout or cancellation with `AbortSignal`, and check `response.ok` after `fetch`.
- [ ] Name the five async pitfalls and how to avoid each.

Then proceed to `tasks.md` and fill in `write-up-template.md` as you build.
