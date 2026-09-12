# Characterization Testing

Pinning what code *does* before you change its shape. Two techniques: hand-written pinning tests for a handful of branches, and a parity harness for code with too many input combinations to enumerate by hand.

## Hand-Written Pinning Tests

The workflow, for each branch the refactor will touch:

1. Call the function with inputs that reach the branch.
2. Print the result. Do not predict it.
3. Paste the printed value into an assertion.
4. Name the test for the behavior, and say "characterizes" if the value looks wrong.

```js
test('usage billing charges only the overage (characterizes unitPrice || default)', () => {
  const out = proc({ ...acct, unitPrice: 0 }, [{ kind: 'usage', qty: 5000 }], null);
  assert.strictEqual(out.sub, 40); // unitPrice 0 falls through to the 0.01 default — pinned, not endorsed
});
```

A pinned bug is the point. It converts "I think that's dead code" into a commit whose diff shows the behavior changing.

**Where to find the branches:** read the code for `if`/`switch`/`||`/`?.`/early returns, and cover each one plus the degenerate inputs — empty array, `undefined`, zero, negative, unknown enum value. Coverage tooling (`node --experimental-test-coverage`, `vitest --coverage`) tells you which lines the existing suite already reaches; refactor freely there, pin everywhere else.

## Parity Harness

For pricing, parsing, formatting, serialization — anything where the interesting behavior lives in the *combinations*. A few dozen lines of harness pins thousands of cases in one run.

**The idea:** keep a copy of the original module, run both implementations over generated inputs, compare serialized results. It is a test that writes its own expectations.

### Snapshot the original first

```sh
mkdir -p test/baseline
git show HEAD:src/invoice.js > test/baseline/invoice.baseline.js
git add test/baseline && git commit -m "test(invoice): snapshot pre-refactor implementation for parity checks"
```

The snapshot is a frozen artifact, never edited again. It is deleted in its own commit once the refactor lands and permanent tests replace it.

### The harness

```js
'use strict';

const test = require('node:test');
const assert = require('node:assert');

const original = require('./baseline/invoice.baseline').proc;
const refactored = require('../src/invoice').proc;

const plans = ['free', 'pro', 'enterprise'];
const states = ['CA', 'NY', 'TX', 'OR', 'WA', undefined];
const kinds = ['seat', 'usage', 'onetime', 'bogus'];
// undefined and NaN belong here: `qty > 0` and `!(qty <= 0)` differ only on these,
// and that is exactly the slip a flatten-the-guards step makes.
const qtys = [0, -1, 1, 10, 49, 50, 99, 100, 49999, 50000, 100000, undefined, NaN];
const coupons = [
  null,
  { type: 'pct', value: 0.1 },
  { type: 'flat', value: 25 },
  { type: 'flat', value: 999999 },
  { type: 'weird', value: 1 },
];

function* cases() {
  for (const plan of plans) {
    for (const state of states) {
      for (const taxExempt of [true, false]) {
        for (const kind of kinds) {
          for (const qty of qtys) {
            for (const coupon of coupons) {
              const account = {
                status: 'active',
                plan,
                seatPrice: 12.5,
                state,
                taxExempt,
                includedUnits: 1000,
                unitPrice: 0.01,
              };

              yield [account, [{ kind, qty, amount: 33.33 }], coupon ? { coupon } : null];
            }
          }
        }
      }
    }
  }

  // degenerate inputs the generator above will never produce
  yield [null, [{ kind: 'seat', qty: 1 }], null];
  yield [{ status: 'closed', plan: 'pro', seatPrice: 10, state: 'CA' }, [{ kind: 'seat', qty: 1 }], null];
  yield [{ status: 'active', plan: 'pro', seatPrice: 10, state: 'CA' }, [], null];
  yield [{ status: 'active', plan: 'pro', seatPrice: 10, state: 'CA' }, undefined, null];
}

// Serialize the outcome, whether it returned or threw — a changed exception is a changed behavior.
function outcome(fn, args) {
  try {
    return JSON.stringify(fn(...structuredClone(args)));
  } catch (err) {
    return `THREW:${err.constructor.name}:${err.message}`;
  }
}

test('refactored invoice matches the pre-refactor implementation', () => {
  let n = 0;

  for (const args of cases()) {
    n++;
    assert.strictEqual(
      outcome(refactored, args),
      outcome(original, args),
      `mismatch for ${JSON.stringify(args)}`,
    );
  }

  assert.ok(n > 1000, `expected a broad sweep, ran ${n}`);
});
```

### Rules that make it real evidence

| Rule | Why |
|---|---|
| `structuredClone` the arguments per call | Otherwise one implementation's mutation of the input poisons the other's run, and the harness is comparing garbage |
| Compare throws, not just returns | A refactor that turns a thrown `TypeError` into a warning object is a behavior change the return-value comparison never sees |
| Include degenerate inputs explicitly | `null`, `undefined`, `[]`, `0`, `-1`, unknown enum values. Cartesian generators only produce the shapes you thought of |
| Assert the case count | A generator bug that yields three cases still passes silently. Pin the sweep size |
| Commit it, and run it per step | It is the net. A harness that exists only in your session protects nothing |
| Keep it fast | A sweep that takes a minute gets skipped. Trim the input lists until it runs in seconds |

### After the refactor lands

The baseline snapshot cannot live in the repo forever — it is a second copy of the logic and it rots. Retire it in one of two ways, in its own commit:

- **Freeze the outputs.** Run the sweep once, write `{ input, output }` pairs to `test/baseline/invoice.golden.json`, and change the harness to compare against the file instead of the module. The evidence survives; the duplicate implementation does not.
- **Promote the interesting cases.** Turn the branches you actually care about into named hand-written tests, then delete both the harness and the snapshot. Fewer cases, better failure messages, no dead copy of the old code.

Freezing is right when the behavior is genuinely combinatorial. Promoting is right when the sweep was scaffolding and a dozen named tests say the same thing more clearly.

## When a Harness Is the Wrong Tool

| Code shape | Instead |
|---|---|
| Hits a database, network, or filesystem | Record real calls once (fixtures, VCR-style cassettes), replay against both implementations |
| Writes files or renders output | Golden-file / approval tests: dump the output, diff against a committed expected file |
| Depends on time, randomness, or ordering | Inject the clock and the seed first — that seam is its own behavior-preserving commit, before the refactor |
| Concurrency and message ordering | A sweep proves nothing here. Pin at a coarser level: integration tests through the public entry point |
| Behavior nobody can characterize because it is genuinely undefined | Say so in the report, and treat the refactor as a behavior change with a reviewer |
