---
name: refactoring-safely
description: Use when restructuring code that already works — a long function, a nested conditional pyramid, duplicated branches, cryptic names, a module someone wants split up — including when the suite is slow, flaky, or missing, when a deadline says "just make it presentable", or when a bug fix is riding along with the cleanup.
---

# Refactoring Safely

## Overview

**Refactoring is changing the shape of code without changing what it does. The restructuring is the easy half; the proof is the deliverable.**

Two rules carry the weight, and they are the two that break first under a deadline:

1. **You may only refactor behavior that a test pins down.** No test on that branch means the first move is writing one, not reading the code harder.
2. **A commit either changes behavior or changes shape. Never both.**

Everything below is those two rules and the mechanics that make them cheap.

**Reading the code is not verification.** Agents restructure legacy code competently and then report "behavior preserved" on the strength of having read it. Sometimes that is even true. It is still not a result anybody can check, re-run, or trust on the next change.

## The Loop

One step is one pass through this. Not one session, not one file.

```
1. Pin      test covers the behavior you are about to move?   no → write it, commit it
2. Move     one named transformation, nothing else
3. Verify   run the tests                                     red → revert, don't debug
4. Commit   refactor-only message, working tree clean
```

**Step 3 is not optional when the suite is slow, and it is mandatory when the suite is flaky.** Flaky means you do not know what green means — that is a reason to run *more* often, on a smaller diff, where a failure is attributable. A red suite after a 200-line step tells you nothing; after a 20-line step it names the line.

**Red after a step: revert, do not debug.** `git checkout -- <file>` costs seconds and the step was small by construction. Debugging a broken refactor is how a shape change quietly becomes a behavior change.

## Step Zero: The Safety Net

Before the first edit, run the suite and look at what it actually covers.

| What you find | What you do |
|---|---|
| Green suite, covers the code you're touching | Go |
| Green suite, **the branch you're touching is uncovered** | Write characterization tests for that branch. Commit them alone |
| Suite is red before you start | Stop. You have no net. Fix or pin the failures first, and say so |
| No tests at all | Write characterization tests for the paths you will touch. Commit them alone |
| Suite is slow | Narrow it: run the one file per step, the full suite before each commit |

**A characterization test pins what the code does, not what it should do.** You are not judging the behavior — you are making it impossible to change by accident.

```js
// Don't reason about what the coupon math "should" produce. Run it and write down the number.
test('pct coupon discounts the subtotal (characterizes current rounding)', () => {
  const out = proc(acct, [{ kind: 'seat', qty: 10 }], { coupon: { type: 'pct', value: 0.335 } });
  assert.strictEqual(out.disc, 33.5);
  assert.strictEqual(out.total, 72.32);
});
```

If a value looks wrong, **pin it anyway and note it**. A characterization test that encodes a bug is doing its job: it makes the bug's removal a visible, separate, reviewable commit instead of a side effect of a rename.

For code with many input combinations — pricing, parsing, formatting — a parity harness pins far more behavior per minute than hand-written cases: [references/characterization-testing.md](references/characterization-testing.md).

**Commit the net.** A parity script or a set of pinning tests that lives only in your session is not a safety net; it is a memory of one. Every agent in baseline testing verified with a scratch script and deleted it, leaving a 200-line restructuring backed by nothing the next person can run.

## One Step, One Commit

| | A step | Not a step |
|---|---|---|
| Size | One transformation, typically under ~40 lines touched | "Extract the helpers, flatten the guards, and rename the locals" |
| Shape | Named: extract, inline, rename, flatten, table-ise | "Clean up `proc`" |
| End state | Tests green, working tree clean | 200 unstaged lines and a report |

Commit messages say what moved, not what improved: `refactor(invoice): extract seat pricing into priceSeat`, `refactor(invoice): flatten proc guards`, `refactor(invoice): rename it/amt/res to item/amount/invoice`.

**"I'll commit once it's all done" is the failure.** The commits are the rollback points; one commit at the end means the only rollback point is "before I started", and the only unit of review is the whole rewrite. Under a demo deadline this inverts: small commits are what let you hand over something half-finished and still working.

Splitting a file is not exempt — `git mv` first, commit, then edit. A delete plus an untracked directory is the worst state to hand anyone, and `git log --follow` loses the file's history.

## The Moves

Real transformations of the same function, one per commit.

### Flatten with guard clauses

Deep nesting is nearly always an early-return refusing to be written. Take the refusals out first — it usually removes an indentation level per commit and shrinks everything that follows.

```js
// ❌ the actual work lives four levels in, and the reasons are at the bottom
function proc(acct, items, opts) {
  if (acct) {
    if (acct.status !== 'closed') {
      if (items && items.length > 0) {
        // ... 60 lines of pricing ...
      } else {
        res.warnings.push('no items');
      }
    } else {
      res.warnings.push('account closed');
    }
  } else {
    res.warnings.push('no account');
  }
  return res;
}

// ✅ reasons first, work at the top level
function proc(acct, items, opts) {
  if (!acct) {
    return empty(['no account']);
  }

  if (acct.status === 'closed') {
    return empty(['account closed']);
  }

  if (!items || items.length === 0) {
    return empty(['no items']);
  }

  // ... pricing, at one level of indentation ...
}
```

Inverting a condition is where behavior leaks. `!(qty > 0)` is not `qty <= 0` — they differ on `undefined` and `NaN`, and that divergence shipped in baseline testing. **When you invert, keep the original predicate and negate it whole**; simplify it in a later commit, if at all.

### Extract a method

Pull out a span that computes one thing, and name it for the answer it produces.

```js
// ✅ the tier ladder, extracted and named for its result
function volumeMultiplier(plan, qty) {
  if (plan === 'enterprise') {
    if (qty >= 100) {
      return 0.85;
    }

    if (qty >= 50) {
      return 0.9;
    }
  }

  if (plan === 'pro' && qty >= 50) {
    return 0.95;
  }

  return 1;
}
```

| Rule | Why |
|---|---|
| Move the code, don't retype it | Retyping is how `>=` becomes `>`. Cut, paste, adjust the edges |
| One extraction per commit | Two at once and a red suite names neither |
| Name the return value, not the steps | `volumeMultiplier`, not `handleTiers` |
| Keep the extracted body dumb | Pure in, pure out. No new logging, no new guards, no "while I'm here" |
| Don't extract what one caller uses once and reads fine inline | A helper per five lines is a different unreadable |

**Duplicate branches are not automatically one function.** The seat ladder and the usage ladder in this file look identical and use different thresholds — 50/100 seats against 50000/100000 units. Collapsing them needs both pinned first, and a parameterized table is honest only if the tests for both still pass unchanged.

### Rename

Renaming is the highest-value move and the one with a real mechanical risk: a missed occurrence in a dynamic language is a runtime error nobody sees until production.

```
grep -rn '\bamt\b' src | wc -l     # 7 — before
# rename
grep -rn '\bamt\b' src | wc -l     # 0 — after, and the old name is gone everywhere
npm test
```

Names describe what the value *is*, not how it was computed or what type it has: `amount` over `amt`, `invoice` over `res`, `volumeMultiplier` over `calcDiscountNum`.

**Locals are yours. The public surface is not.** Exported names and returned keys — `proc`, `{ sub, disc, tax, total }` — are an API: renaming one is a breaking change with callers to update, and nobody asked for it. Leave it, name it in your report, and rename only if the answer comes back yes. If you do, it is its own commit, with the call sites, and no assertion edited.

### Replace a ladder with a table

Three or more branches choosing a value is data, not control flow — and a table is checkable at a glance.

```js
const TIERS = {
  enterprise: [{ min: 100, mult: 0.85 }, { min: 50, mult: 0.9 }],
  pro: [{ min: 50, mult: 0.95 }],
};
```

Do this only after the ladder is extracted, named, and green. Table-ising an inline ladder is two steps in one commit.

## Behavior Changes Are Separate Commits

The cleanup is where bug fixes ask to ride along. Every baseline agent took the offer, and each described the fix as part of the cleanup.

```js
// the first line is dead — overwritten before it is read
res.disc = Math.round(res.sub * opts.coupon.value) / 100 * 100;
res.disc = Math.round(res.sub * opts.coupon.value * 100) / 100;
```

Deleting that is correct. Deleting it *inside* an extraction commit is not: if the parity harness goes red, nothing tells you whether the extraction or the deletion did it, and a reviewer reading `refactor: extract computeDiscount` has no reason to check the arithmetic.

| Situation | Order |
|---|---|
| Found a bug mid-refactor | Finish the current step, commit it. Then a test that fails on the bug, then the fix, then a `fix:` commit |
| Asked to refactor *and* fix in one task | Both, in that order, in separate commits. Say which commit is which |
| Dead code you believe is unreachable | Pin the surrounding behavior, delete, confirm parity holds — its own commit |
| Behavior looks wrong but was not in scope | Leave it. Pin it, name it in your report |

"Output is unchanged, so it's part of the refactor" is the trap. If output is genuinely unchanged, the commit costs one line and one message — there is no reason to bundle it. If output *did* change, bundling is exactly the thing that hides it.

## Let the Machine Do It

You do not have an IDE's Extract Method, but the equivalents are cheap and they catch more than rereading does.

| Want | Use |
|---|---|
| Rename across a file or repo | `grep -rn '\bold\b'` for a before-count, edit, re-grep for zero, run tests |
| Move or split a file | `git mv` in its own commit, before any edit |
| A large mechanical transformation | A codemod (`ast-grep`, `jscodeshift`), then read the diff — not fifty hand edits |
| Fast feedback between test runs | The typechecker and linter. `tsc --noEmit` costs seconds and catches the arity slip the tests would take minutes to reach |
| Confidence on combinatorial code | A parity harness against the pre-refactor version — see the reference |
| Review your own step | `git diff` before every commit. A refactor diff that adds behavior is visible the moment you look |

## Smell → Move

| Smell | First move |
|---|---|
| Nesting three levels deep | Guard clauses for the refusal branches |
| Function over ~50 lines | Extract the spans that compute one value each |
| Same branch ladder twice | Pin both, extract each, then merge only if the thresholds truly match |
| `res`, `amt`, `it`, `tmp`, `data` | Rename, grep-verified |
| Ladder choosing between values | Extract, then a lookup table |
| Comment explaining a block | Extract the block, use the comment as the name |
| Flags threaded through a function | Split into the two functions the flag was choosing between |
| A class holding unrelated state | Split along the state each method actually touches — last, not first |

## When Not to Refactor

- **The code is about to be deleted or rewritten for a real requirement.** Refactor then, as part of it.
- **You are in the middle of a behavior change.** Land it, then refactor, in that order.
- **You cannot get the code under test at all** — untestable I/O tangle, no seam. Then the first commit is the seam, and it is a behavior-preserving one.
- **Nobody has read the code in a year and nothing is wrong with it.** Untouched code is not a smell.

## Contract

| Property | How to check |
|---|---|
| Net exists before the first edit | The pinning tests are a commit that precedes every refactor commit |
| Uncovered branches got pinned | Every branch the diff touches appears in a test that existed before the step |
| Steps are small | `git log --stat` — each refactor commit is one named move |
| Every step was verified | The suite was run per step, not once at the end |
| Tree is clean | `git status --short` is empty; no untracked split-out directory, no deleted-file limbo |
| Behavior unchanged | Tests that existed before the refactor pass **unmodified**. A test you edited to match the new code is not evidence |
| Shape and behavior are separate | No commit message says "refactor" and contains a fix, or vice versa |
| Public surface intact | Exported names and returned keys unchanged, or the change is its own commit with its callers |
| Proof is re-runnable | The parity harness or pinning tests are committed, not a deleted scratch file |
| Reported honestly | If you skipped a step, say which. "Verified manually" names what you ran |

## Rationalizations (verbatim from baseline testing)

| Excuse | Reality |
|---|---|
| "Per your note about the suite being slow/flaky, I did not loop on it" | Flaky means green is uninformative — so make the diff small enough that red is attributable. Running less is the opposite move. |
| "I did not add new tests, since none existed before and the ask was cleanup, not new coverage" | The tests are not coverage, they are the net. Refactoring uncovered code is a rewrite with no way to tell. |
| "The first line was dead code, clearly leftover, not a behavior difference" | Then proving it costs one test and one commit. "Clearly" is what you have instead of evidence. |
| "I verified those paths manually against the original logic" | With a script you deleted. Nobody can re-run manual. |
| "Fuzz-compared across 176 combinations — zero mismatches" | Good work, thrown away. Commit the harness and it protects the next change too. |
| "Output is unchanged, so it belongs in the refactor commit" | If it's unchanged, a separate commit costs nothing. If it isn't, bundling is what hides it. |
| "I assumed `value` is a fraction, since that's consistent with the surviving line" | An assumption about untested behavior. Pin it, and the assumption becomes a fact or a failing test. |
| "It's a 15-minute job, commits would slow me down" | Three commits cost 30 seconds and are the only thing that makes 15 minutes recoverable. |
| "I kept the public API identical so no callers need changing" | Good — that is the bar, not the achievement. It says nothing about the internals you moved. |
| "The lead wants it modular, so one big restructuring is the task" | The destination is one big restructuring. The route is still one move per commit. |
| "Tests pass, so behavior is preserved" | Four tests over a function with twelve branches. Passing means the four are fine. |
| "I'll clean up the test file afterwards to match the new structure" | Editing the tests to fit the new code deletes the evidence. Tests change in their own commit, after. |

## Red Flags — STOP

- You are editing a branch that no test executes
- The suite has not been run since you started editing
- You ran the suite once, and it was after the last edit
- `git status` shows more than one logical change, or the first commit is still ahead of you
- The step you are in the middle of has no name — you are "cleaning up"
- You are about to fix something in a commit whose message starts with `refactor`
- You inverted a condition and simplified it in the same edit
- You retyped a block instead of moving it
- A test file is open in the same change as the code it tests
- Your evidence is a script that will not exist in five minutes
- You are deleting code because it looks unreachable
- The suite went red and you started reading the new code instead of reverting

**All of these mean: stop, revert to the last green commit, and take a smaller step.**

## Not Covered

Commit message wording: see `writing-commit-messages`. TypeScript-specific shape rules — guards over casts, immutability, tagged errors: see `writing-typescript`.
