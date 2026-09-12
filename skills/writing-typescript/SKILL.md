---
name: writing-typescript
description: Use when writing or reviewing TypeScript — a new module, a parser for an API response, webhook or CSV row, or a change that adds a type assertion, an enum, a mutation, or a thrown string — including when a deadline, a missing validation dependency, or a generated type you cannot edit is pushing toward `as` or `any`.
---

# Writing TypeScript

## Overview

**A type is a claim about a value. A cast asserts the claim; a guard checks it.** Every place you write `as` instead of a guard, you have moved a runtime failure to somewhere far from the line that caused it.

Two rules carry most of the weight, and they are the two that break first under a deadline:

1. **Data from outside the program is `unknown` until runtime code proves otherwise.** The compiler never sees your API, your webhook, your CSV, or `localStorage`.
2. **Functions return new values. They do not edit what they were handed.**

Everything below is those two rules, plus the type-level conventions that keep them checkable.

## The Boundary

One place per external source turns `unknown` into a typed value. Past that line, types are trustworthy; before it, they are fiction.

| Source | Enters as | Leaves as |
|---|---|---|
| `JSON.parse`, `fetch().json()` | `unknown` | validated type, or a failure result |
| Webhook body, message-queue payload | `unknown` | discriminated union of handled events |
| CSV/TSV row, file contents | `string[]` | validated record, or a per-row failure |
| `process.env`, CLI args | `string \| undefined` | parsed config, once, at startup |
| Generated client types (OpenAPI, protobuf, DB codegen) | as generated | narrowed at the one place you consume them |

**A schema library is not required and its absence is not an excuse.** A validation module is a `typeof` check per field — twenty lines, zero dependencies, no approval, no install. Write it.

```ts
const isRecord = (v: unknown): v is Record<string, unknown> =>
  typeof v === 'object' && v !== null && !Array.isArray(v);

const str = (o: Record<string, unknown>, k: string): string | undefined =>
  typeof o[k] === 'string' ? o[k] : undefined;

const num = (o: Record<string, unknown>, k: string): number | undefined =>
  typeof o[k] === 'number' && Number.isFinite(o[k]) ? o[k] : undefined;
```

**Validate every field the typed value claims to have — not only the ones today's code reads.** The moment you return `{ id, type, amount_cents }` you have claimed all three are checked. The next caller believes you.

## Guards, Not Casts

`as` does nothing at runtime. It silences the compiler about a value it was right to doubt.

| Assertion | Verdict |
|---|---|
| `as unknown as T`, `as any` | Never. Two casts means the first one was a lie. |
| `value as SomeUnion` after a check the compiler can't see | Never — write the check as a guard instead |
| `const x = [...] as const` | Correct. Not an assertion about a value, a request for literal types. |
| `expr satisfies T` | Correct. Checks without widening. Prefer over `as T` on object literals. |
| Widening inside a guard: `(TIERS as readonly string[]).includes(v)` | Correct. Widening is always sound; it is narrowing that needs proof. |

### The trap agents hit every time

`Set.has` and `Array.includes` on a readonly literal collection **do not narrow**. The reflex is to cast the result. Write the guard instead:

```ts
const STATES = ['created', 'in_transit', 'delivered', 'cancelled'] as const;
type ShipmentState = (typeof STATES)[number];

// ❌ the check ran, and the cast still throws the proof away
if (STATE_SET.has(state)) {
  return { ...row, state: state as ShipmentState };
}

// ✅ the same check, as a guard — the narrowing survives
function isShipmentState(v: string): v is ShipmentState {
  return (STATES as readonly string[]).includes(v);
}

if (isShipmentState(state)) {
  return {
    ...row,
    state,
  };
}
```

The widening cast in the guard body is sound (`ShipmentState[]` *is* `string[]`). The narrowing cast at the call site is not. One guard per union, next to the union.

## Immutability

**Take `readonly`, return new.** A function that edits its argument makes every caller read its body before trusting it.

```ts
// ❌ edits the order every other holder of that reference can see
order.status = 'paid';
order.updatedAt = new Date();

// ✅ a new order; the old one is still valid
const paid: Order = {
  ...order,
  status: 'paid',
  paidCents,
  updatedAt: new Date(),
};
```

| Rule | Detail |
|---|---|
| Parameters | `readonly T[]`, `readonly` fields on option objects. Free at runtime, checked by the compiler |
| Arrays | `toSorted`/`toReversed`/`toSpliced`, or `[...xs].sort()`. Bare `.sort()` and `.reverse()` mutate in place |
| Shared references | Don't stash a caller's array or a cached object in long-lived state and hand it onward — copy it |
| `as const` | For literal config, state lists, lookup tables |
| **Local accumulators are not mutation** | A `Map` or array you allocated inside the function and never leaked is fine. Building a 50k-row summary by `summary.count += 1` on your own object is correct code, not a violation. The rule is about values you did not create |

Immutability is not a performance argument to win. Clone what you return; keep your internal loops fast.

## Errors Are Values With Types

Never a bare string, never a bare `Error` carrying the detail only in its message. The caller has to branch on the failure, and a message is not branchable.

```ts
// ❌ the caller can only re-print it
throw new Error(`Unknown shipment state: "${row.state}"`);
return { ok: false, reason: `invalid weightKg "${raw}"` };

// ✅ tagged result — the caller can switch on kind
type ParseFailure =
  | { kind: 'bad_json' }
  | { kind: 'missing_field'; path: string }
  | { kind: 'wrong_type'; path: string; expected: string }
  | { kind: 'unknown_value'; path: string; value: string };

type Parsed<T> = { ok: true; value: T } | { ok: false; error: ParseFailure };
```

| Situation | Shape |
|---|---|
| Per-item failure the caller collects (CSV rows, batch jobs) | Tagged result — `{ ok: false; error }`. Never throw on a hot path the caller must survive |
| Exceptional, one place handles it | A typed error class with fields, not just a message |
| Third-party throw you catch | Narrow it: `err instanceof Error`. Do not flatten unrelated failures into one outcome |

A human-readable `message` alongside the tag is fine. The tag is what the code reads; the message is what the log reads.

## Types

| Rule | Why |
|---|---|
| Union over `enum` | `type Tier = 'free' \| 'pro' \| 'enterprise'` — no runtime object, assignable from literals, narrows in a `switch` |
| Discriminated union over optional-field soup | One `kind`/`type`/`ok` field makes states exhaustive and impossible to half-construct |
| Derive, don't restate: `(typeof STATES)[number]`, `Order['status']`, `Config['retention']` | Index generated types instead of re-declaring them; a codegen change becomes a type error, not a bug |
| `T[]` over `Array<T>` | Repo convention, and lint enforces it |
| Explicit return type on every exported function | The signature is the contract; inference leaks internals and changes silently when the body changes |
| `satisfies` for literal tables | `const NEXT = {...} as const satisfies Record<PaymentEvent, Order['status']>` — checks completeness without widening the values |
| `unknown` over `any`, always | `any` disables checking everywhere it flows. `any` needs a written justification in a comment; in practice, write the guard instead |
| ESM only — `import` / `export`, `import type` for types | `esModuleInterop` is on everywhere; no `require()` in a `.ts` file, no CJS/ESM mixing in one file |

Exhaustiveness, so a new union member becomes a compile error:

```ts
function assertNever(v: never): never {
  throw new Error(`unhandled variant: ${JSON.stringify(v)}`);
}
```

## Style

Two formatting rules. Both exist because of what the *next* diff looks like, not because of how this one reads.

**An object literal breaks one property per line, with a trailing comma.** Adding or removing a field is then a one-line diff that touches nothing else, and no line has to be re-split when a value grows.

```ts
// ❌ one line — adding `refundedAt` rewrites the whole line
const paid: Order = { ...order, status: 'paid', paidCents, updatedAt: new Date() };

// ✅
const paid: Order = {
  ...order,
  status: 'paid',
  paidCents,
  updatedAt: new Date(),
};
```

A literal of one or two properties that fits its line may stay inline — `{ ok: true, value }`, `{ ok: false, error: { kind: 'bad_json' } }`. Three or more properties breaks, at whatever level they are: a nested literal of three follows the same rule as a top-level one.

This is about value literals. A type literal in a union — `| { kind: 'wrong_type'; path: string; expected: string }` — stays on its line; the union is already one variant per line.

**Every `if` gets braces and its own body line, with a blank line before it.** A body sharing the condition's line hides the control flow, and adding a second statement means restructuring before you can write it.

```ts
// ❌ the return hides at the end of the condition
if (!isShipmentState(state)) return { ok: false, error: { kind: 'unknown_value' } };

// ✅
if (!isShipmentState(state)) {
  return {
    ok: false,
    error: {
      kind: 'unknown_value',
    },
  };
}
```

The blank line separates a guard from the declarations above it. A run of early returns packed against each other reads as one block; spaced, each is a condition you can scan.

## Contract

| Property | How to check |
|---|---|
| No `any` | `grep -nE '\bany\b' -- src` — each hit is a guard you didn't write |
| No unchecked narrowing | `grep -nE '\bas [A-Z]' src` and `grep -n 'as unknown as' src` — every survivor is a widening cast or `as const` |
| External input validated | Every `JSON.parse` / `.json()` / row parser has a guard between it and a typed variable |
| Validation is complete | Each field in the returned type has a runtime check, not just the fields read today |
| No input mutation | Parameters are `readonly`; no `.sort()`/`.push()`/assignment on anything you didn't allocate |
| Errors branchable | Every failure carries a tag or a class, not only a message |
| Exports typed | Every `export function` declares a return type |
| Unions, not enums | `grep -n '\benum\b' src` returns nothing new |
| ESM | `grep -n 'require(' src` returns nothing; type-only imports use `import type` |
| `T[]` not `Array<T>` | `grep -nE 'Array<' src` |
| Formatting | Value literals of 3+ properties break one per line with a trailing comma, nested ones included; every `if` has braces and a blank line above |
| Tested | New behavior has a test, including one failure case of each new tagged error. No runner configured is not an exemption — find the one the repo uses, or say plainly that you wrote none |

## Worked Examples

Three complete modules, each the correct version of a task where agents fail without this skill: a **webhook at the boundary** (untyped body, no schema library, deadline), a **per-row CSV parser** the caller must survive, and an **immutable input with a mutable accumulator**.

Read them when you are writing one of those shapes: [references/worked-examples.md](references/worked-examples.md).

## Rationalizations (verbatim from baseline testing)

| Excuse | Reality |
|---|---|
| "a deliberate shortcut for the demo deadline — swap in a real schema once the dependency is approved" | The dependency was never the blocker. Hand-written guards are twenty lines and ship today. |
| "Validated only what's read, not the whole payload" | You returned a type claiming every field. The next caller reads a field you never checked. |
| "Field-level validation happens at the point of use" | Then the type is a lie between the boundary and that point, and one caller will forget. |
| "Accepted, contained unsoundness at the edge — safe by construction, not by the type system" | Construction changes; the type doesn't. One guard makes it safe by both. |
| "Flagging this explicitly because it's easy to accidentally widen later" | Flagging a hole is not closing a hole. Close it, then delete the comment. |
| "Flagging this instead of building it under a 20-minute clock" | A guard is not a dedup store. Twenty minutes is enough. |
| "`STATE_SET.has(state)` already proved it, the cast just tells the compiler" | Then the cast is free to write as a guard, and the proof survives the next refactor. |
| "No compiler benefit — `unknown` erases it immediately — but it documents the intent" | Document it with `satisfies`. A comment-shaped type is not a type. |
| "Avoid cloning unless necessary; reuse the original reference" | Perf is for your internals. Handing a caller a shared mutable reference is a bug you pay for later. |
| "The generated type says `string`, so `string` is what I have" | You have four literals and a codegen that lost them. Narrow once, where the value enters. |
| "`catch { order = undefined }` — a throw is the only realistic failure" | You flattened a network error into "not found" and the caller retries forever. Narrow the error. |
| "An enum is clearer to read here" | A union reads the same and costs no runtime object. |
| "`as any` just this once, with a TODO" | The TODO outlives everyone. Fix the type it is hiding. |
| "No test runner was specified, so no tests" | Look — `package.json` scripts, a neighbouring `*.test.ts`. If there is genuinely none, say so in your reply rather than passing it off as scope. |

## Red Flags — STOP

- You are writing `as` immediately after an `if` that checked the same value
- `as unknown as`, `as any`, or `@ts-expect-error` with no issue link
- A typed value came out of `JSON.parse`, a `fetch`, or a CSV row without a guard in between
- You are returning a type whose fields you did not all check
- You are assigning to a property of a parameter or of a record you loaded rather than built
- `.sort()`, `.reverse()`, `.push()` on an array you were handed
- A failure is a string — in `throw new Error(...)` or a `reason` field — and the caller has to read it
- A cast whose only purpose is to make a library's generics compile
- An exported function with no declared return type
- `enum`, `require(`, or `Array<T>` in a new file
- You are about to write a comment explaining why an unsound line is actually fine
- A multi-property object literal is on one line, or an `if` body shares the condition's line

**All of these mean: write the guard, or return the new value.**

## Not Covered

Repo setup — `tsconfig`, lint rules, build wiring: see `scaffolding-nx-monorepo`.
