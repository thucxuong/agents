# Review sub-skill: Logic flow / refactor chunk

Chunk type: **Logic flow / refactor**
Applicable when: a function or module has been restructured — same behaviour
intended, different implementation. The diff shows movement, extraction,
renaming, or reordering of logic.

---

## Reliability ceiling

**Medium, even with all steps followed.**

Semantic equivalence between two structurally different implementations is a
hard limit. The pipeline below maximises reliability by forcing explicit flow
representation before any comparison — but the final equivalence judgment
at the flow delta step requires a human who knows the original intent.

Never claim equivalence based on diff alone. The diff is the worst possible
input for this chunk type.

---

## Minimum information set

- [ ] **Full original function/module** — the entire file or function, not a snippet
- [ ] The diff for this chunk
- [ ] Existing tests for this code

The full original is non-negotiable. If only the diff is available → **stop,
request the full original file before proceeding.**

---

## Information pipeline

### Step 1 — Extract old flow as pseudocode (AI, read original only)

Read the full original code. Do **not** look at the diff yet.

Produce numbered pseudocode capturing every branch, condition, and exit point.
Be mechanical — do not interpret or summarise. Every `if`, `else`, `catch`,
`return`, `throw` must appear.

```
Old flow — processOrder():
1. Validate input (throws ValidationError if invalid)
2. If cart is empty → throw EmptyCartError
3. Fetch user from DB
4. If user.role === 'guest' → apply guest pricing tier
5a. If total > 50 → apply discount (10%)
5b. Else → no discount
6. Create order record
7. If payment gateway returns error → rollback order, throw PaymentError
8. Send confirmation email (async, fire-and-forget)
9. Return { orderId, total, status: 'confirmed' }
```

---

### Step 2 — Human validates old flow (Gate — critical)

**This is the most important gate in the entire pipeline.**

Present the pseudocode to the human. Ask them to confirm:

- Is every branch captured?
- Is the ordering correct?
- Are any implicit behaviours missing?

A wrong old flow makes the entire comparison worthless. Do not skip this gate.
Do not proceed until human confirms.

---

### Step 3 — Extract new flow as pseudocode (AI, read diff)

Now read the diff. Produce the same pseudocode format for the new implementation.

Use the same numbering scheme where possible to make comparison easier.
Where a step is new (no equivalent in old flow), mark it with `[NEW]`.

```
New flow — processOrder():
1. Validate input (throws ValidationError if invalid)
2. If cart is empty → throw EmptyCartError
3. Fetch user from DB
4. Determine pricing tier (extracted to getPricingTier(user))
   4a. If user.role === 'guest' → guest tier
   4b. Else → standard tier
5. Apply discount via applyDiscount(total, tier)  ← [changed: was inline condition]
6. Create order record
7. If payment gateway returns error → rollback order, throw PaymentError
8. [MISSING — no email dispatch found]
9. Return { orderId, total, status: 'confirmed' }
```

---

### Step 4 — Diff the two pseudocode flows (AI)

Compare old vs new step by step. Produce a delta report:

```
EQUIVALENT:  Steps 1, 2, 3, 6, 7, 9 — identical behaviour
REFACTORED:  Step 4 — guest pricing logic extracted to getPricingTier()
             → verify extracted function covers same conditions
REFACTORED:  Step 5 — discount logic extracted to applyDiscount()
             → verify threshold is still > 50 (not >= 50)
MISSING:     Step 8 — confirmation email. Present in old flow, absent in new.
             → Intentional removal? Or silent regression?
```

Focus on:

- Steps in old flow with **no equivalent** in new flow
- Conditions that **changed** (operator, threshold, variable name)
- New steps with **no old equivalent** (what was added, and why?)

---

### Step 5 — Verify extracted functions (AI)

For each step marked `REFACTORED` — where logic was moved to a new function:

Read the extracted function. Does it reproduce the original condition exactly?

Common failure modes:

- Off-by-one: `> 50` became `>= 50`
- Inverted condition: `=== 'guest'` became `!== 'standard'` (may differ for new roles)
- Missing case: original had 3 conditions, extracted function has 2
- Default changed: original defaulted to X, extracted function defaults to Y

---

### Step 6 — Test coverage check (AI)

For each branch in the old flow: does a test exist that would catch a regression?
For each `MISSING` or `CHANGED` item in the delta: is there a test that would
have caught this if it were a bug?

```
Step 4 (guest pricing) — test exists: order.spec.ts:112 ✓
Step 5 threshold — NO TEST for boundary value ($50 exact)
Step 8 email — NO TEST (fire-and-forget, but regression risk is real)
```

---

### Step 7 — Human judges each delta item (Gate)

For every `MISSING` and `CHANGED` item, present to human:

> "Step 8 (confirmation email) is in the old flow but not in the new flow.
> Is this an intentional removal, or a regression? If intentional, was it
> moved elsewhere?"

Do not assume missing = intentional. Do not assume changed condition = bug.
Human intent is the only source of truth here.

---

## Output format

```
[CHUNK: Logic flow — <function/module name>]
Old flow steps: <N>
New flow steps: <N>
Equivalent: <N> | Refactored: <N> | Missing: <N> | New: <N>
Test gaps: <N>

Findings:
[SEVERITY] [FILE:LINE] — description
...

Flow delta (requires human review):
- MISSING: Step N — <description>
- CHANGED: Step N — old: <condition> / new: <condition>

Questions for human:
1. ...

Reliability note: Semantic equivalence has not been confirmed. Human must
validate the flow delta above before this chunk can be considered reviewed.
```
