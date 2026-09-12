# Worked Examples

Three complete modules written to the `writing-typescript` contract. Each one is the correct version of a task where agents measurably fail without the skill.

They assume the boundary helpers from `SKILL.md` — `isRecord`, `str`, `num` — and the `ParseFailure` / `Parsed<T>` shapes defined there.

## 1. A webhook at the boundary

Untyped third-party body, no schema library, a deadline. Still validated, still immutable, still branchable.

```ts
import type { Order } from './orders.js';

const PAYMENT_EVENTS = ['payment.succeeded', 'payment.failed', 'payment.refunded'] as const;
type PaymentEvent = (typeof PAYMENT_EVENTS)[number];

const isPaymentEvent = (v: string): v is PaymentEvent =>
  (PAYMENT_EVENTS as readonly string[]).includes(v);

const NEXT_STATUS = {
  'payment.succeeded': 'paid',
  'payment.failed': 'failed',
  'payment.refunded': 'refunded',
} as const satisfies Record<PaymentEvent, Order['status']>;

type Payment = {
  eventId: string;
  type: PaymentEvent;
  orderId: string;
  amountCents: number;
};

export type WebhookOutcome =
  | { kind: 'applied'; orderId: string }
  | { kind: 'ignored'; eventType: string }
  | { kind: 'invalid'; error: ParseFailure }
  | { kind: 'order_missing'; orderId: string };

export function parsePaymentEvent(raw: string): Parsed<Payment | null> {
  let json: unknown;

  try {
    json = JSON.parse(raw);
  } catch {
    return { ok: false, error: { kind: 'bad_json' } };
  }

  if (!isRecord(json) || !isRecord(json.data) || !isRecord(json.data.object)) {
    return { ok: false, error: { kind: 'missing_field', path: 'data.object' } };
  }

  const eventId = str(json, 'id');
  const type = str(json, 'type');

  if (eventId === undefined) {
    return { ok: false, error: { kind: 'missing_field', path: 'id' } };
  }

  if (type === undefined) {
    return { ok: false, error: { kind: 'missing_field', path: 'type' } };
  }

  if (!isPaymentEvent(type)) {
    return { ok: true, value: null };   // not ours; not an error
  }

  const object = json.data.object;
  const orderId = str(object, 'order_id');
  const amountCents = num(object, 'amount_cents');

  if (orderId === undefined) {
    return { ok: false, error: { kind: 'missing_field', path: 'data.object.order_id' } };
  }

  if (amountCents === undefined) {
    return {
      ok: false,
      error: {
        kind: 'wrong_type',
        path: 'data.object.amount_cents',
        expected: 'number',
      },
    };
  }

  return {
    ok: true,
    value: {
      eventId,
      type,
      orderId,
      amountCents,
    },
  };
}

function paidCentsAfter(order: Order, event: Payment): number {
  if (event.type === 'payment.succeeded') {
    return event.amountCents;
  }

  return order.paidCents;
}

export function applyPayment(order: Order, event: Payment): Order {
  return {
    ...order,
    status: NEXT_STATUS[event.type],
    paidCents: paidCentsAfter(order, event),
    updatedAt: new Date(),
  };
}
```

| Line of the contract | Where it shows up |
|---|---|
| One boundary | `parsePaymentEvent` is the only function that touches `JSON.parse` output |
| Guards, not casts | `isPaymentEvent` narrows; nothing is asserted |
| Complete validation | Every field `Payment` claims is checked before the value is returned |
| Errors are values | `ParseFailure` and `WebhookOutcome` are both tagged; callers branch on `kind` |
| Immutability | `applyPayment` spreads into a new `Order`; the loaded one is untouched |
| Exhaustiveness | `satisfies` on `NEXT_STATUS` fails to compile when a fourth event type is added |

**Narrow a third-party throw; don't flatten it.** `loadOrder` declares `Promise<Order>` with no failure shape, so a rejection cannot be read as "not found":

```ts
try {
  order = await loadOrder(orderId);
} catch (err) {
  const message = errorMessage(err, 'loading order');

  return { kind: 'load_failed', orderId, message };
}
```

Mapping that rejection to `order_missing` would return a 404 for a database outage, and the sender would stop retrying.

## 2. A per-row parser the caller must survive

50k rows, and the caller collects the bad ones. Throwing is wrong here: a tagged failure per row is the interface.

```ts
export const STATES = ['created', 'in_transit', 'delivered', 'cancelled'] as const;
export type ShipmentState = (typeof STATES)[number];

export function isShipmentState(v: string): v is ShipmentState {
  return (STATES as readonly string[]).includes(v);
}

export type RowFailure =
  | { kind: 'wrong_column_count'; expected: number; actual: number }
  | { kind: 'missing_field'; path: string }
  | { kind: 'wrong_type'; path: string; expected: string; value: string }
  | { kind: 'unknown_value'; path: string; value: string };

export type ParsedRow =
  | { ok: true; value: Shipment }
  | { ok: false; error: RowFailure };

const EXPECTED_COLUMNS = 7;

export function parseShipmentCsvRow(row: readonly string[]): ParsedRow {
  if (row.length !== EXPECTED_COLUMNS) {
    return {
      ok: false,
      error: {
        kind: 'wrong_column_count',
        expected: EXPECTED_COLUMNS,
        actual: row.length,
      },
    };
  }

  const [id, stateRaw, weightRaw, country, postcode, tagsRaw, deliveredAtRaw] = row;

  if (id === '') {
    return { ok: false, error: { kind: 'missing_field', path: 'id' } };
  }

  if (!isShipmentState(stateRaw)) {
    return {
      ok: false,
      error: {
        kind: 'unknown_value',
        path: 'state',
        value: stateRaw,
      },
    };
  }

  const weightKg = Number(weightRaw);

  if (weightRaw === '' || !Number.isFinite(weightKg)) {
    return {
      ok: false,
      error: {
        kind: 'wrong_type',
        path: 'weightKg',
        expected: 'number',
        value: weightRaw,
      },
    };
  }

  if (country === '') {
    return { ok: false, error: { kind: 'missing_field', path: 'destination.country' } };
  }

  if (postcode === '') {
    return { ok: false, error: { kind: 'missing_field', path: 'destination.postcode' } };
  }

  let deliveredAt: string | undefined;

  if (deliveredAtRaw !== '') {
    if (Number.isNaN(Date.parse(deliveredAtRaw))) {
      return {
        ok: false,
        error: {
          kind: 'wrong_type',
          path: 'deliveredAt',
          expected: 'ISO timestamp',
          value: deliveredAtRaw,
        },
      };
    }

    deliveredAt = deliveredAtRaw;
  }

  const tags = tagsRaw === '' ? [] : tagsRaw.split(';');

  return {
    ok: true,
    value: {
      id,
      state: stateRaw,
      weightKg,
      destination: { country, postcode },
      tags,
      ...(deliveredAt !== undefined ? { deliveredAt } : {}),
    },
  };
}
```

Notes worth carrying to other parsers:

- **`isShipmentState` is why `state: stateRaw` compiles.** The guard, not a cast, is what makes the narrowed value usable in the returned object.
- **Every field the `Shipment` type declares is checked**, not only the ones this report reads. Returning the type is a claim about all of them.
- **The optional field is spread conditionally**, so `deliveredAt` is absent rather than present-and-`undefined` — correct under `exactOptionalPropertyTypes`.
- **The failure carries `path` and `value`**, so the caller can aggregate "4,000 rows had an unknown state" instead of grepping messages.

## 3. Immutable input, mutable accumulator

Both halves of the immutability rule in one function. The caller's array is never touched; the `Map` and the summaries inside it are allocated here and mutated freely.

```ts
export interface CountrySummary {
  country: string;
  count: number;
  totalWeightKg: number;
  deliveredCount: number;
}

export function summarizeByCountry(
  shipments: readonly Shipment[],
): Map<string, CountrySummary> {
  const summaries = new Map<string, CountrySummary>();

  for (const shipment of shipments) {
    const country = shipment.destination.country;
    let summary = summaries.get(country);

    if (summary === undefined) {
      summary = {
        country,
        count: 0,
        totalWeightKg: 0,
        deliveredCount: 0,
      };

      summaries.set(country, summary);
    }

    // Allocated in this loop and not yet leaked — a local accumulator,
    // not an edit of anything the caller handed us.
    summary.count += 1;
    summary.totalWeightKg += shipment.weightKg;

    if (shipment.state === 'delivered') {
      summary.deliveredCount += 1;
    }
  }

  return summaries;
}
```

The same rule applied to sorting, where the reflex is to mutate:

```ts
// ❌ reorders the caller's array
shipments.sort(byDeliveredAt);

// ✅ sorts a copy this function owns
return [...shipments].sort(byDeliveredAt);
```

Cloning what you return costs one allocation. Cloning inside a 50k-iteration loop is what you avoid — not the same thing, and not a reason to hand back a mutated input.
