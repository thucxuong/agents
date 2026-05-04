# Review sub-skill: Layer / tier chunk

Chunk type: **Layer / tier**
Applicable when: changes span multiple technical layers (UI → API → service → DB)
reviewed as separate groups, each with its own contract surface.

---

## Reliability ceiling

**High.** Layer contracts are explicit in types, interfaces, and DTOs.
Agent can verify mechanically if given the interface surface.
The one human gate is architectural judgment on flagged misplacements.

---

## Minimum information set

- [ ] Type definition / interface files for all affected layer boundaries
- [ ] API schema or OpenAPI spec (if API layer is involved)
- [ ] The diff for this chunk
- [ ] Naming convention document or inferable from existing code

If type files are missing → **stop, request them.**

---

## Information pipeline

### Step 1 — Identify layer boundaries in this chunk (AI)

List every layer boundary touched by the diff. A boundary is any point where
data crosses from one layer to another.

```
Boundary A: UI component → API call (CheckoutForm → POST /orders)
Boundary B: API handler → service (OrderController → OrderService.create)
Boundary C: service → repository (OrderService → OrderRepository.save)
```

---

### Step 2 — Extract contract snapshot: before (AI)

Read the current type/interface files (not the diff). For each boundary,
record the exact data shape flowing in and out.

```
Boundary A (before):
  in:  { items: CartItem[], userId: string }
  out: { orderId: string, status: OrderStatus }

Boundary B (before):
  in:  CreateOrderDto { items, userId, couponCode? }
  out: OrderResponseDto { id, total, status }
```

---

### Step 3 — Extract contract snapshot: after (AI)

Read the diff and produce the same snapshot for the new state.

```
Boundary A (after):
  in:  { items: CartItem[], userId: string, guestToken?: string }
  out: { orderId: string, status: OrderStatus }

Boundary B (after):
  in:  CreateOrderDto { items, userId, couponCode?, guestToken? }
  out: OrderResponseDto { id, total, status, discountApplied? }
```

---

### Step 4 — Diff the two snapshots (AI)

Compare before vs after for each boundary. Flag:

- **Added fields:** Are they optional? Do all consumers handle the new field?
- **Removed fields:** Will any caller break?
- **Type changes:** Widening (safe) vs narrowing (potentially breaking)?
- **Shape mismatch:** Does what layer A sends match what layer B now expects?

```
Boundary A: added guestToken? — optional, safe
Boundary B in: added guestToken? — matches Boundary A, consistent
Boundary B out: added discountApplied? — UI must handle this or it's lost silently
```

---

### Step 5 — Logic placement audit (AI)

Scan each layer's changed files. Flag logic that belongs in a different layer:

| Found in     | Should be in | Example                              |
| ------------ | ------------ | ------------------------------------ |
| Controller   | Service      | Business rule calculation            |
| UI component | API/service  | Direct DB call or complex validation |
| Repository   | Service      | Conditional business logic           |
| Service      | Repository   | Raw query construction               |

Produce a list of misplacements with suggested correct layer.

---

### Step 6 — Error propagation check (AI)

For each boundary: does the error handling shape match across layers?

- If service throws `OrderNotFoundError`, does the controller catch and map it to HTTP 404?
- If API returns a 422, does the UI handle it or swallow it silently?
- Are error types consistent (typed errors vs untyped strings)?

---

### Step 7 — Human reviews architectural flags (Gate)

Present misplacement flags and shape mismatches. Human decides:

- Is this misplacement an acceptable trade-off or must it move?
- Is the missing `discountApplied` handler in UI a blocker or will it be addressed separately?

---

## Output format

```
[CHUNK: Layer — <layers covered>]
Boundaries reviewed: <N>
Contract changes: <N> added fields, <N> removed, <N> type changes
Misplacements flagged: <N>
Error propagation gaps: <N>

Findings:
[SEVERITY] [FILE:LINE] — description
...

Questions for human:
1. ...
```
