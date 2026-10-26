# M04 Learning Material — TypeScript Basics & TypeScript on Node.js

> **Companion to `brief.md` and `tasks.md`.** The brief tells you *what* to deliver; this document
> teaches the *concepts* you need to get there. It deliberately does **not** contain the finished
> lab: you still port your own M02 toolkit, derive the Task-domain types, design `Result<T, E>`, and
> write your own explanations of `unknown` vs `any` and `type` vs `interface`. Read it, try the
> drills, then do the lab.

**Week 2 · Days 1-2 · ~8 hours**

## 0. How to use this material

| Section | Maps to brief Scope bullet | Maps to Definition of done | Suggested time |
| --- | --- | --- | --- |
| 1. Why TypeScript | Why TypeScript; structural typing | — | 20 min |
| 2. Core types | Primitives … `type` vs `interface` | `type` vs `interface` explanation | 40 min |
| 3. Functions and generics | Functions, generics | Toolkit ported with generics | 35 min |
| 4. Narrowing | Narrowing | `Result<T, E>` | 35 min |
| 5. `unknown`, `any`, `never` | `unknown` vs `any` vs `never` | `unknown` vs `any` explanation | 25 min |
| 6. Utility types and friends | Utility types, `satisfies`, `import type` | Task input types | 35 min |
| 7. Project setup | Project setup, `tsconfig.json` | Zero `tsc --noEmit` errors | 30 min |
| 8. Three ways to run TypeScript | `tsx`, native stripping, `tsc` | Runs via `tsx`; stripping attempted | 25 min |
| 9. Typing real Node code | Async, errors, `process.env`, `@types/*` | — | 25 min |
| 10. TypeScript 6 → 7 | TypeScript 6 → 7 | — | 15 min |
| 11. Reading compiler errors | Reading compiler errors | — | 20 min |
| 12. Thinking about the port | Deliverable | All | 15 min |
| 13. Guided practice | all | — | 45 min |
| 14. Knowledge check | all | — | 20 min |

That is about 6.5 hours of reading and practice. The rest of your 8 hours is the lab.

**Before the lab:** have your M02 toolkit (`sleep`, `retry`, `withTimeout`, `mapLimit`) and write-up
at hand. You port *your own* code; this material has no copy of it. Install `@types/node@24` to
match the cohort's Node 24 LTS.

**Learning outcomes.** After this module you can:

- Read and write strict TypeScript with unions, generics, narrowing and utility types.
- Explain `unknown` vs `any` and `type` vs `interface`.
- Configure a project for strict type-checking with `tsc --noEmit`.
- Run TypeScript through `tsx` and say what Node's native type stripping can and cannot do.
- Read a compiler error and find its cause.

**Verified on this toolchain.** The examples and behaviours in sections 7, 8, 10 and 11 were run
with TypeScript 7.0.2, `tsx` 4.23, `@types/node` 24 and Node 24.19. Versions move, so if something
differs on your machine, trust your compiler's output and check `npx tsc --version`.

---

## 1. Why TypeScript

TypeScript is JavaScript plus a **static type system**. The compiler checks your code before it runs
and then **erases** the types; what executes is plain JavaScript. Types exist only at compile time:
they do not validate data coming from a file, a user or a network (that is what Zod does later, in M08).

### 1.1 Structural typing

TypeScript compares the *shape* of values, not their names:

```ts
type Point = { x: number; y: number };
const p = { x: 1, y: 2, label: "origin" };
const q: Point = p;              // fine: p has at least x and y
```

An object is acceptable wherever its shape fits. (Fresh object literals are also checked for extra
properties, which is why `const r: Point = { x: 1, y: 2, label: "o" }` errors.)

### 1.2 Inference vs annotation

```ts
const count = 3;                         // inferred: 3 (literal type for const)
let name = "ada";                        // inferred: string
const ids = [1, 2, 3];                   // inferred: number[]
function double(n: number): number {     // annotate parameters; return type is optional
  return n * 2;
}
```

Let the compiler infer where it can. Annotate **function parameters**, exported function return
types, and places where inference would be too wide or too narrow.

---

## 2. Core types

### 2.1 The building blocks

| Kind | Example |
| --- | --- |
| Primitives | `string`, `number`, `boolean`, `bigint`, `symbol`, `null`, `undefined` |
| Arrays | `number[]` or `Array<number>` |
| Tuples | `[string, number]` — fixed length and order |
| Union | `string \| number` — one of several |
| Intersection | `A & B` — both at once |
| Literal types | `"asc" \| "desc"`, `404` |
| `readonly` | `readonly string[]`, `readonly id: number` |

```ts
type Status = "todo" | "doing" | "done";          // union of literals: your "enum"
const pair: [string, number] = ["age", 30];
type Named = { name: string };
type Aged = { age: number };
type Person = Named & Aged;                        // must have both
```

### 2.2 `as const`

`as const` freezes a literal to its narrowest, `readonly` type:

```ts
const levels = ["debug", "info", "error"] as const;   // readonly ["debug", "info", "error"]
type Level = (typeof levels)[number];                 // "debug" | "info" | "error"
```

### 2.3 `type` vs `interface`

Both describe object shapes. Know the differences, then decide which this course should use and why:

| | `type` | `interface` |
| --- | --- | --- |
| Object shapes | Yes | Yes |
| Unions, tuples, primitives, mapped/conditional types | **Yes** | No |
| Extending | `&` intersection | `extends` |
| Re-opening and merging (declaration merging) | No (duplicate is an error) | **Yes** |

Optional and read-only properties work in both: `{ id: number; nickname?: string; readonly createdAt: Date }`.

### 2.4 Unions instead of `enum`

This course uses string-literal unions, not `enum`:

```ts
type Role = "admin" | "member";            // erased completely at runtime
```

An `enum` generates real JavaScript code, which is not "erasable" syntax. Section 8 shows what happens
when you try to run one with Node's native type stripping.

---

## 3. Functions and generics

### 3.1 Typing functions

```ts
function greet(name: string, greeting = "hello"): string {      // default → optional
  return `${greeting} ${name}`;
}
const add = (a: number, b: number): number => a + b;
type Handler = (input: string) => void;                           // a function type
function log(message: string, ...tags: string[]) {}               // rest parameter
```

### 3.2 Generics

A **generic** is a type placeholder that the caller fills in, so one function works for many types
without losing information:

```ts
function first<T>(items: T[]): T | undefined {
  return items[0];
}
const n = first([1, 2, 3]);        // T inferred as number → n: number | undefined
const s = first(["a", "b"]);       // T inferred as string
```

Contrast with `any`: a function returning `any` forgets what it was given; a generic remembers it.

### 3.3 Constraints and defaults

```ts
function longest<T extends { length: number }>(a: T, b: T): T {   // T must have .length
  return a.length >= b.length ? a : b;
}
type Box<T = string> = { value: T };                                // default type argument
```

### 3.4 Generics and `Promise`

`Promise<T>` is generic: the type of the value inside. `async` functions always return a `Promise`:

```ts
async function load<T>(fn: () => Promise<T>): Promise<T> {
  return await fn();
}
```

Reading signatures like `<T>(fn: () => Promise<T>) => Promise<T>` fluently is most of what the
toolkit port needs.

---

## 4. Narrowing

Narrowing means the compiler knows a *more specific* type inside a branch.

| Technique | Example |
| --- | --- |
| `typeof` | `if (typeof x === "string") { x.toUpperCase() }` |
| Truthiness / equality | `if (x !== undefined) { … }` |
| `in` | `if ("radius" in shape) { shape.radius }` |
| `instanceof` | `if (err instanceof Error) { err.message }` |
| Discriminated union | check a shared literal field (below) |
| Type guard | a function returning `x is T` |

### 4.1 Discriminated unions

A union whose members share a literal "tag" property lets the tag decide the type:

```ts
type Shape =
  | { kind: "circle"; radius: number }
  | { kind: "square"; size: number };

function area(s: Shape): number {
  switch (s.kind) {
    case "circle": return Math.PI * s.radius ** 2;     // s is the circle member here
    case "square": return s.size ** 2;
  }
}
```

If you add a third member and forget a `case`, the compiler can warn you (section 5.3).

### 4.2 User-defined type guards

```ts
function isString(value: unknown): value is string {
  return typeof value === "string";
}
```

The `value is string` return type is a promise *you* make; the compiler trusts it, so write it
carefully.

---

## 5. `unknown`, `any`, `never`

### 5.1 `any`

`any` switches the type checker **off** for that value. Anything is allowed, nothing is checked, and
it spreads: values derived from `any` are often `any` too.

### 5.2 `unknown`

`unknown` means "a value of a type I don't know yet". You may hold it, pass it around and compare it,
but you must **narrow it before using it**:

```ts
function describe(x: unknown) {
  // x.length;                    // error: 'x' is of type 'unknown'
  if (typeof x === "string") return x.length;       // narrowed to string
  return 0;
}
```

This is why caught errors and parsed JSON should be `unknown`, not `any`. Writing the difference up in
your own words is part of the Definition of done, so work out for yourself *when you would reach for
each* and what goes wrong with `any`.

### 5.3 `never`

`never` is the type of values that cannot exist: a function that always throws, or the leftover
case after every union member is handled.

```ts
function assertNever(x: never): never {
  throw new Error(`Unhandled: ${JSON.stringify(x)}`);
}
// in a switch's default branch: assertNever(shape)
// if a union member is unhandled, shape is not `never` and the compiler errors
```

---

## 6. Utility types, `satisfies`, `import type`

### 6.1 The utility types

Built-in generic helpers that derive new types from existing ones, so you don't repeat yourself.
Shown here with a deliberately unrelated type:

```ts
type User = { id: number; name: string; email?: string; role: "admin" | "member" };

type A = Partial<User>;                  // all properties optional
type B = Required<User>;                 // all properties required (email too)
type C = Pick<User, "id" | "name">;      // only these keys
type D = Omit<User, "id">;               // everything except these keys
type E = Record<"a" | "b", number>;      // { a: number; b: number }
type F = Readonly<User>;                 // all properties readonly
type G = ReturnType<typeof Date.now>;    // number — the return type of a function type
type H = Awaited<Promise<string>>;       // string — unwrap a Promise
```

Utility types compose: `Partial<Omit<User, "id">>` means "everything except `id`, all optional".
When one type should follow another, **derive** it instead of copying it — a change to the source
then flows through automatically.

### 6.2 `satisfies`

`satisfies` checks that a value fits a type **without widening** its inferred type:

```ts
const ports = { http: 80, https: 443 } satisfies Record<string, number>;
ports.http;        // still known to be a number, and 'ports.ftp' is an error
```

### 6.3 `import type`

```ts
import type { Server } from "node:http";      // erased entirely; no runtime import
```

Under `verbatimModuleSyntax` (section 7), anything that is **only a type** must be imported with
`import type` (or the `type` modifier: `import { type Server, createServer }`).

---

## 7. Project setup

### 7.1 Install

```bash
npm init -y
npm pkg set type=module
npm i -D typescript@7 tsx @types/node@24
```

`@types/node` supplies the types for `node:fs`, `process`, and so on. **Match its major version to
your Node major** (24), or you'll type APIs your runtime doesn't have.

### 7.2 `tsconfig.json` essentials

```json
{
  "compilerOptions": {
    "target": "es2023",
    "module": "nodenext",
    "moduleResolution": "nodenext",
    "strict": true,
    "verbatimModuleSyntax": true,
    "erasableSyntaxOnly": true,
    "noUncheckedIndexedAccess": true,
    "noEmit": true,
    "skipLibCheck": true,
    "types": ["node"]
  },
  "include": ["src"]
}
```

| Option | What it does |
| --- | --- |
| `strict` | Turns on the strict family (`noImplicitAny`, `strictNullChecks`, …). The point of TypeScript |
| `module` / `moduleResolution: nodenext` | Resolve imports the way modern Node does (ESM, extensions, `package.json` `type`) |
| `verbatimModuleSyntax` | Imports/exports are kept or dropped exactly as written; type-only imports need `type` |
| `erasableSyntaxOnly` | Errors on syntax that needs code generation (`enum`, `namespace`, parameter properties) |
| `noUncheckedIndexedAccess` | `arr[i]` and `obj[key]` include `undefined` in their type |
| `noEmit` | Type-check only; don't write `.js` files |
| `skipLibCheck` | Skip type-checking of `.d.ts` files in `node_modules` (faster; hides library-internal errors) |

You will likely need **one more** option depending on how you write imports — see section 8.3.

### 7.3 The type-check gate

```json
{ "scripts": { "typecheck": "tsc --noEmit" } }
```

`npm run typecheck` exits `0` when there are no errors and non-zero otherwise, so CI can gate on it.
This is the check the brief's Definition of done refers to. Running your code with `tsx` does **not**
type-check it (section 8).

---

## 8. Three ways to run TypeScript

| Way | What it does | Type-checks? | Role in this course |
| --- | --- | --- | --- |
| **`tsx`** | Strips/transforms types on the fly, then runs | **No** | The dev runner (`tsx file.ts`, `tsx watch`) |
| **Node native type stripping** | Node removes erasable type syntax and runs | **No** | Awareness; fully stable only on Node 26 |
| **`tsc` build** | Compiles to `.js`, then you run the output | **Yes** (type-check) | Awareness only |

**Nothing but `tsc --noEmit` checks your types.** A file with a type error still runs under `tsx`.

### 8.1 `tsx`

```bash
npx tsx src/main.ts
npx tsx watch src/main.ts
```

### 8.2 Native type stripping on Node 24

On the Node 24.19 used to write this material, plain `node file.ts` ran a file using only erasable
syntax (type annotations, `type`/`interface`, `import type`) with no flag. It does **not** handle syntax
that needs generated code: an `enum` fails with `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX` ("TypeScript enum
is not supported in strip-only mode"). That is why `erasableSyntaxOnly` exists: it makes `tsc`
reject the same syntax up front. Your task is to attempt it yourself on your own code and **record
what you actually saw**, including whether your version needed a flag or printed a warning.

### 8.3 Imports and extensions

Node's ESM resolution wants explicit file extensions. This produces a real tension, worth
understanding before you hit it:

- `import { x } from "./helper"` — `tsc` errors under `nodenext` (`TS2835`, "Relative import paths
  need explicit file extensions"), and Node can't resolve it.
- `import { x } from "./helper.js"` — `tsc` accepts it (it maps `.js` to `helper.ts`), and `tsx`
  runs it, but **native stripping fails** because no `helper.js` exists on disk.
- `import { x } from "./helper.ts"` — native stripping runs it, but `tsc` reports `TS5097`
  until you enable `allowImportingTsExtensions` (allowed here because `noEmit` is on).

So the setup you choose decides which of the three runners work. Decide deliberately and note it in
your write-up.

---

## 9. Typing real Node code

### 9.1 Async

```ts
import { readFile } from "node:fs/promises";

async function loadJson(path: string): Promise<unknown> {     // JSON.parse results are untrusted
  return JSON.parse(await readFile(path, "utf8"));
}
```

Prefer to return `unknown` from boundaries (files, network) and narrow or validate it.

### 9.2 Caught errors

With `strict`, `catch (e)` gives `e` the type `unknown`:

```ts
try {
  await loadJson("x.json");
} catch (e: unknown) {
  const message = e instanceof Error ? e.message : String(e);
}
```

Anything can be thrown in JavaScript (a string, `undefined`), so check before reading `.message`.

### 9.3 `process.env`

`process.env.PORT` has type `string | undefined`. Convert and default explicitly:

```ts
const port = Number(process.env.PORT ?? 3000);
```

### 9.4 `@types/*` and `.d.ts`

A `.d.ts` file contains only type declarations. Packages written in JavaScript ship types through
`@types/<name>` (the `DefinitelyTyped` project) or bundle their own. If a package has no types you can
write a small `.d.ts` declaring its shape.

---

## 10. TypeScript 6 → 7

TypeScript 7 is a **native-compiler** rewrite: the compiler is no longer JavaScript, which makes it
much faster. What the brief wants you to know, and what we confirmed on 7.0.2:

- **Strict by default.** With no `tsconfig.json` at all, `tsc --noEmit file.ts` already reported an
  implicit-`any` parameter and a `null` assigned to `number`.
- **Legacy options are removed.** Setting `target: "es5"`, `baseUrl`, or `moduleResolution: "node10"`
  is now an error (`TS5108`/`TS5102`) rather than a deprecation warning. Old tsconfigs may need cleaning.
- **No stable programmatic API yet.** Tools that import the compiler as a library (some Jest
  transformers, linters) need workarounds. The program meets this again in M09, so note it now.
- **TypeScript 6 remains available** (`npm i -D typescript@6`) if a tool needs it.

---

## 11. Reading compiler errors

Format: `file(line,col): error TSxxxx: message`. Read the **first** error first — later ones are often
consequences. Errors seen while writing this material:

| Code | Message (shortened) | Usual cause |
| --- | --- | --- |
| `TS2322` | `Type 'string' is not assignable to type 'number'` | Wrong value for a declared type; also `number \| undefined` after an indexed access |
| `TS7006` | `Parameter 'a' implicitly has an 'any' type` | Missing parameter annotation under `strict` |
| `TS18046` | `'x' is of type 'unknown'` | Using an `unknown` value without narrowing |
| `TS18048` | `'s' is possibly 'undefined'` | Using a value that might be `undefined` without a check |
| `TS2339` | `Property 'b' does not exist on type …` | Typo, or the property isn't on that type |
| `TS1294` | `This syntax is not allowed when 'erasableSyntaxOnly' is enabled` | You used `enum`, `namespace`, etc. |
| `TS2835` | `Relative import paths need explicit file extensions…` | Missing extension under `nodenext` |
| `TS5097` | `An import path can only end with a '.ts' extension when …` | `allowImportingTsExtensions` is off |

Habit: read the message, go to the line, ask "what type does the compiler think this is?" (hover in
VS Code, or assign to a variable with a deliberately wrong type to make the compiler tell you).
Resist silencing errors with `any` or `as`.

---

## 12. Thinking about the port

Questions only — answers are your design.

**Porting the toolkit**
- What are the type parameters of `retry`, `mapLimit` and `withTimeout`, and what should be inferred
  rather than annotated?
- What type should `retry`'s options have, and which fields are optional?
- In `mapLimit<T, R>`, how does the callback's return type determine the result's type?
- What type does `sleep` return, and what should an `AbortSignal` parameter look like?
- Where could `noUncheckedIndexedAccess` change code that was fine in JavaScript?

**Task domain**
- Which fields belong on `Task`? Which are required on creation and which are generated?
- Which utility types would derive `CreateTaskInput` and `UpdateTaskInput` from `Task` so that you
  never write the field list twice?

**`Result<T, E>`**
- What is the tag property, and what does narrowing on it let a caller do?
- When would you return a `Result` instead of throwing?

**Delivery**
- Which import style will you use (section 8.3), and which runners does that allow?

---

## 13. Guided practice (throwaway project)

Use a scratch folder, **not** your lab repo:

```bash
mkdir ts-sandbox && cd ts-sandbox && npm init -y
npm pkg set type=module && npm i -D typescript@7 tsx @types/node@24
```

Create the `tsconfig.json` from 7.2 and a `src/` folder.

### Drill 1 — Inference
Declare variables with `const` and `let` without annotations and hover to see the inferred types.
Why is `const x = "a"` typed `"a"` but `let y = "a"` typed `string`?

### Drill 2 — Unions and narrowing
Write a function taking `string | number | undefined` and returning a string for every case. Add a
`never` check so adding a type to the union breaks the build until you handle it.

### Drill 3 — Discriminated union
Model a `Shape` union with three variants and write `area`. Add a fourth variant and watch the
compiler complain in `area`.

### Drill 4 — Generics
Write `pluck<T, K extends keyof T>(items: T[], key: K): T[K][]`. Call it and check the inferred
result type. Try a key that doesn't exist.

### Drill 5 — Utility types
For a `Product` type with five fields, derive: a creation type without `id`, an update type that's
the creation type with everything optional, and a summary type with two fields. Never repeat a
field name.

### Drill 6 — `unknown` vs `any`
Write a function that parses JSON and returns `unknown`, then narrows it to `{ name: string }` with a
type guard. Rewrite it using `any` and note what the compiler no longer tells you.

### Drill 7 — Read the errors
Deliberately create four errors from the table in section 11. For each, predict the code and
message before running `npx tsc --noEmit`.

### Drill 8 — Three runners
Write a small file with a type error and run it with `tsx`, then `node file.ts`, then
`npx tsc --noEmit`. Which ones notice the error? Add an `enum` and repeat. Try the three import styles
from 8.3 and record which combination works with which runner.

### Drill 9 — Index access
Turn `noUncheckedIndexedAccess` off, write `const v: number = arr[10]`, then turn it on. What changes
and how do you fix it properly?

---

## 14. Knowledge check

Answer without looking at the material first. Answers to questions 1–10 are at the end; questions
11–13 are open-ended and have **no answer key** because they feed your write-up.

### Multiple choice

**1.** What happens to TypeScript types at runtime?
- a) They validate incoming data  b) They are erased  c) They become JSON schemas  d) They slow startup

**2.** Which type best fits a value that could be one of `"asc"` or `"desc"`?
- a) `string`  b) `"asc" \| "desc"`  c) `enum`  d) `any`

**3.** What does `Omit<User, "id">` produce?
- a) `User` with `id` optional  b) `User` without the `id` property  c) Only `id`  d) `never`

**4.** Which `tsconfig` option makes `arr[i]` include `undefined`?
- a) `strict`  b) `noEmit`  c) `noUncheckedIndexedAccess`  d) `skipLibCheck`

**5.** What type is `e` in `catch (e)` under `strict`?
- a) `any`  b) `Error`  c) `unknown`  d) `never`

**6.** A function returns `value is string`. This is a:
- a) Generic constraint  b) User-defined type guard  c) Template literal type  d) Utility type

**7.** Which command checks types without writing files?
- a) `tsx src/main.ts`  b) `node src/main.ts`  c) `tsc --noEmit`  d) `npm install`

**8.** You run `tsx app.ts` and it works, but `tsc --noEmit` reports errors. Why?
- a) `tsx` is wrong  b) `tsx` doesn't type-check, only strips types  c) `tsc` is slower  d) The errors are fake

**9.** Which syntax is **not** supported by Node's native type stripping?
- a) `interface`  b) `import type`  c) `enum`  d) type annotations on parameters

**10.** What does `satisfies` do?
- a) Casts a value to a type
- b) Checks a value fits a type without widening its inferred type
- c) Imports a type
- d) Makes a property optional

### Short answer / explain

**11.** In your own words, explain `unknown` vs `any`. Give one place in the toolkit or Task domain
where `unknown` is the right choice.

**12.** In your own words, explain `type` vs `interface`, which you used and why.

**13.** What happened when you tried native type stripping on Node 24 with your own code? What did you
have to change, and what does that tell you about `tsx` vs native stripping?

<details>
<summary>Answer key (questions 1–10 and drill hints only)</summary>

**Multiple choice:** 1-b, 2-b, 3-b, 4-c, 5-c, 6-b, 7-c, 8-b, 9-c, 10-b.

**Drill hints (verify by running them):**

- Drill 1: `const` can't change, so its literal type is kept; `let` can, so it widens to `string`.
- Drill 2: end the narrowing with `const _exhaustive: never = value`.
- Drill 4: the constraint `K extends keyof T` rejects unknown keys.
- Drill 6: with `any`, `result.nmae` compiles; with `unknown` you must narrow first.
- Drill 8: only `tsc` reports type errors; `enum` fails natively but runs under `tsx`.
- Drill 9: `arr[10]` becomes `number | undefined`; fix with a check, a default, or `?.`.

</details>

---

## 15. Common pitfalls and errors

| Symptom | Likely cause / fix |
| --- | --- |
| Code runs but `tsc` complains | `tsx` and `node file.ts` don't type-check; run `npm run typecheck`. |
| `TS2835` on a relative import | Add an extension, or choose a consistent import style (section 8.3). |
| `TS5097` on a `.ts` import | Enable `allowImportingTsExtensions` (requires `noEmit`). |
| `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX` | You used `enum`/`namespace`; use a union of literals. |
| `TS1294` | Same cause, caught earlier by `erasableSyntaxOnly`. |
| `Cannot find module 'node:fs'` types | Install `@types/node` and check `"types": ["node"]`. |
| `@types/node` mismatch | Match its major to your Node major (24). |
| `x is of type 'unknown'` | Narrow with `typeof`, `instanceof`, or a type guard. |
| Fixing errors with `as any` | Narrow, type properly, or use `unknown` — `any` hides the next bug. |
| Old config fails on TypeScript 7 | Remove `baseUrl`, `target: es5`, `moduleResolution: node10`. |

## 16. Further reading

- TypeScript Handbook: <https://www.typescriptlang.org/docs/handbook/intro.html>
- TypeScript `tsconfig` reference: <https://www.typescriptlang.org/tsconfig>
- Utility types: <https://www.typescriptlang.org/docs/handbook/utility-types.html>
- Narrowing: <https://www.typescriptlang.org/docs/handbook/2/narrowing.html>
- Node.js — Running TypeScript natively: <https://nodejs.org/api/typescript.html>
- `tsx`: <https://tsx.is>
- Total TypeScript tutorials (free): <https://www.totaltypescript.com>

## 17. Ready for the lab?

Before starting the lab in `brief.md`, you should be able to:

- [ ] Read a generic signature like `<T, R>(items: T[], fn: (t: T) => Promise<R>) => Promise<R[]>`.
- [ ] Narrow a discriminated union and use `never` for exhaustiveness.
- [ ] Derive one type from another with `Pick`, `Omit` and `Partial` instead of retyping fields.
- [ ] Say what each `tsconfig` option in section 7.2 does.
- [ ] Explain why `tsx` running a file doesn't mean it type-checks.
- [ ] Have your M02 toolkit and write-up ready to port.

Then proceed to `tasks.md` and fill in `write-up-template.md` as you build.
