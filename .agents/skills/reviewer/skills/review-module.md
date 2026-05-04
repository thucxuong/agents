# Review sub-skill: Module / package chunk

Chunk type: **Module / package**
Applicable when: changes are scoped to a sub-project, shared library, or
package with a defined public API surface (barrel/index file, package.json exports).

---

## Reliability ceiling

**High.** Module contracts are structurally visible in exports.
Consumer impact analysis is the high-value intermediate step — do not skip it.

---

## Minimum information set

- [ ] Index / barrel file(s) for changed modules (before and after)
- [ ] List of modules/packages that import this one (from dependency graph, IDE, or grep)
- [ ] The diff for this chunk
- [ ] Test files for the changed module

If the consumer list is unavailable → run `grep -r "from '../../<module-name>'"` or
equivalent across the repo. This is mandatory — consumer impact cannot be assessed
without it.

---

## Information pipeline

### Step 1 — Snapshot public API surface: before (AI)

Read the current index/barrel file(s). List every exported symbol with its type signature:

```
Before — @app/shared/utils exports:
- formatDate(date: Date, locale: string): string
- calculateTotal(items: CartItem[]): number
- createUser(name: string): Promise<User>
- UserRole (enum)
- CartItem (type)
```

---

### Step 2 — Snapshot public API surface: after (AI)

Read the diff and produce the same snapshot for the new state:

```
After — @app/shared/utils exports:
- formatDate(date: Date, locale: string, tz?: string): string  ← signature changed
- calculateTotal(items: CartItem[], coupon?: Coupon): number    ← signature changed
- createUser(name: string, options?: UserOptions): Promise<User> ← signature changed
- deleteUser(id: string): Promise<void>                        ← ADDED
- UserRole (enum)
- CartItem (type)
- UserOptions (type)                                           ← ADDED
[REMOVED: no removals]
```

---

### Step 3 — Classify changes (AI)

For each changed export, classify:

| Export         | Change type                              | Breaking? | Reason                               |
| -------------- | ---------------------------------------- | --------- | ------------------------------------ |
| formatDate     | Signature widened (optional param added) | No        | Optional param, backwards compatible |
| calculateTotal | Signature widened                        | No        | Optional param                       |
| createUser     | Signature widened                        | No        | Optional param                       |
| deleteUser     | Added                                    | No        | New export, no existing consumers    |
| UserOptions    | Added                                    | No        | New type                             |

Breaking = any consumer would fail to compile or behave differently without changes.
Widening with optional params is usually non-breaking. Removing exports or changing
required params is always breaking.

---

### Step 4 — Consumer impact analysis (AI)

For each changed or added export, scan all consumer files.

```
formatDate — consumers: checkout.component.ts:34, order-summary.tsx:12, invoice.service.ts:88
  → all calls use (date, locale) — no tz arg — safe, no changes needed

calculateTotal — consumers: cart.service.ts:51
  → call uses (items) only — safe

deleteUser — consumers: none yet — new export, no impact
```

Flag any consumer that:

- Passes arguments that no longer match the new signature
- Relies on a return type that has changed shape
- Would be affected by a removed export

---

### Step 5 — Test isolation check (AI)

Read the module's test files. Flag any test that:

- Imports from outside the module boundary (cross-module test dependency)
- Mocks a module it should not know about
- Tests implementation details rather than the public interface

```
shared/utils/__tests__/format.spec.ts — imports from @app/order — FLAG: cross-module dependency
shared/utils/__tests__/calc.spec.ts — clean, isolated ✓
```

---

### Step 6 — Internal consistency check (AI)

Within the changed module, check:

- Naming consistency: do new exports follow the same naming pattern as existing ones?
- Error handling: do new functions throw the same error types as existing ones?
- Return shape: do similar functions return similar shapes?

---

### Step 7 — Human confirms breaking change intent (Gate)

Present the breakage report. For each breaking or potentially breaking change:

> "calculateTotal signature changed — existing callers do not pass the new coupon
> param so they are safe, but if any caller needs coupon support, they must update.
> Is this intentional? Should we bump the minor version?"

Human decides: intentional (version bump) or unintended regression.

---

## Output format

```
[CHUNK: Module — <module name>]
Exports before: <N> | after: <N>
Added: <N> | Removed: <N> | Signature changed: <N>
Breaking changes: <N>
Consumers affected: <N> files
Test isolation issues: <N>

Findings:
[SEVERITY] [FILE:LINE] — description
...

Questions for human:
1. ...
```
