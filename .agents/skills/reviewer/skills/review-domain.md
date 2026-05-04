# Review sub-skill: Domain / feature chunk

Chunk type: **Domain / feature**
Applicable when: files belong to a single business feature or bounded context.

---

## Reliability ceiling

**Medium → High** depending on whether a spec exists as text.
Without explicit AC or spec, any domain review is the agent inferring business
rules from code — unreliable by design. If no spec exists, stop and request one.

---

## Minimum information set

Before starting, confirm all of the following are available:

- [ ] Acceptance criteria or user story (text, not just a ticket number)
- [ ] Domain model or entity definitions (if the feature touches core entities)
- [ ] The diff for this chunk
- [ ] Test files for this chunk

If any item is missing → **stop, request it, do not proceed.**

---

## Information pipeline

### Step 1 — Extract rule inventory (AI, before reading diff)

Read the AC/spec only. Do not touch the diff yet.

Produce a numbered list of every business rule, condition, and threshold:

```
Rule 1: User cannot checkout with an empty cart
Rule 2: Discount applies only when order total > $50
Rule 3: Guest users receive a different pricing tier
Rule 4: Order confirmation email must be sent after successful payment
...
```

Be exhaustive. Implicit rules ("obviously X should happen") must be written
explicitly or they will not be checked.

---

### Step 2 — Map rules to diff (AI)

Read the diff. For each rule, find the code location that implements it.

Produce a mapping table:

```
Rule 1 → cart.service.ts:42 — status: covered
Rule 2 → pricing.util.ts:18 — status: covered (threshold is 50, matches spec)
Rule 3 → pricing.util.ts:31 — status: AMBIGUOUS (condition uses user.type, spec says user.role)
Rule 4 → order.service.ts — status: MISSING (no email dispatch found in diff)
```

Status options:

- `covered` — clear implementation found
- `ambiguous` — implementation exists but wording or condition differs from spec
- `missing` — no matching code found in diff

---

### Step 3 — Human resolves ambiguous and missing (Gate)

Present the ambiguous and missing rules to the human.

For each:

- **Ambiguous:** "Rule 3 uses `user.type` in code but spec says `user.role`. Are these the same field? If so, confirm. If not, this is a bug."
- **Missing:** "Rule 4 has no matching code. Is email dispatch handled elsewhere (e.g. an event listener not in this diff)? If yes, point me to it. If no, this is a blocker."

Wait for human response before continuing.

---

### Step 4 — Test coverage audit (AI)

For each rule marked `covered` or resolved as `covered` by human:

Check the test files. Does a test exist that would catch a regression on this rule?

```
Rule 1 — test: cart.service.spec.ts:88 "throws on empty cart" — COVERED
Rule 2 — test: MISSING — no test for $50 threshold boundary
Rule 3 — test: pricing.spec.ts:41 — COVERED
Rule 4 — test: MISSING — no test for email dispatch
```

Produce a list of rules with no covering test, plus a suggested test scenario
for each gap.

---

### Step 5 — Edge case scan (AI)

For each covered rule, consider: what input would make this condition fail silently?

Common patterns to check:

- Off-by-one on numeric thresholds (> vs >=)
- Null / undefined inputs not guarded before condition check
- Empty string vs null treated differently
- Timezone or locale assumptions in date conditions
- Race conditions in async flows

Flag any found. Mark as `nit` if low risk, `concern` if realistic, `blocker` if
the happy path could be affected.

---

## Output format

```
[CHUNK: Domain — <feature name>]
Rule inventory: <N> rules extracted
Coverage: <N> covered, <N> ambiguous, <N> missing
Test gaps: <N> rules with no covering test
Edge cases flagged: <N>

Findings:
[SEVERITY] [FILE:LINE] — description
...

Questions for human:
1. ...
```
