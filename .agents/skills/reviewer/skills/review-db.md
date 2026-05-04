# Review sub-skill: Database / migration chunk

Chunk type: **Database / migration**
Applicable when: diff includes migration files, schema changes, ORM model
updates, or significant query changes.

---

## Reliability ceiling

**Medium.** Structural safety is fully automatable. Lock risk and data correctness
at production scale require human knowledge of real table sizes, data distributions,
and production DB constraints. This ceiling cannot be raised by the agent alone.

---

## Minimum information set

- [ ] Migration file(s) — up and down
- [ ] Current full schema (not just the affected table — foreign keys reach across)
- [ ] ORM model files for affected entities
- [ ] Table row count estimates for affected tables (approximate is sufficient)
- [ ] DB engine and version (e.g. PostgreSQL 15, MySQL 8.0)

If row counts are unavailable → explicitly note that lock risk assessment will be
incomplete. Request estimates. Do not silently skip this.

---

## Information pipeline

### Step 1 — Classify each migration operation (AI)

Read the migration file. For each DDL operation, assign a safety class:

| Operation                             | Class       | Reason                         |
| ------------------------------------- | ----------- | ------------------------------ |
| ADD COLUMN (nullable or with default) | Additive    | Safe on all engines            |
| ADD COLUMN (NOT NULL, no default)     | Destructive | Will fail if table has rows    |
| DROP COLUMN                           | Destructive | Data permanently lost          |
| RENAME COLUMN                         | Structural  | Breaks all code using old name |
| ADD INDEX                             | Structural  | May lock table during creation |
| DROP INDEX                            | Additive    | Usually safe                   |
| CREATE TABLE                          | Additive    | Safe                           |
| DROP TABLE                            | Destructive | Data permanently lost          |
| ALTER COLUMN type                     | Structural  | May require full table rewrite |
| ADD FOREIGN KEY                       | Structural  | Validates all existing rows    |
| ADD CONSTRAINT (NOT NULL, CHECK)      | Structural  | Validates all existing rows    |

```
Migration 20240501_add_discount.sql:
  ADD COLUMN discount_code VARCHAR(50) NULL    → Additive ✓
  ADD INDEX idx_orders_discount ON orders(discount_code) → Structural ⚠
  ADD COLUMN applied_at TIMESTAMP NOT NULL     → Destructive ✗ (no default, table has rows)
```

---

### Step 2 — Lock risk assessment (AI)

For each `Structural` or `Destructive` operation, combine with row count estimate:

```
ADD INDEX on orders — estimated 8M rows
  MySQL 8.0: online DDL available (ALGORITHM=INPLACE) — LOW lock risk if specified
  Without ALGORITHM=INPLACE: MEDIUM lock risk (shared lock during build)
  Recommendation: add ALGORITHM=INPLACE, LOCK=NONE to migration

ADD COLUMN applied_at NOT NULL — 8M rows, no default
  BLOCKER: will fail immediately on MySQL if table has rows
  Fix: add DEFAULT NOW() or make nullable, backfill separately
```

Produce a risk matrix per operation.

---

### Step 3 — Down migration verification (AI)

Read the rollback / down migration. For each up operation, verify the down
migration is the correct structural inverse:

```
UP:   ADD COLUMN discount_code
DOWN: DROP COLUMN discount_code ✓

UP:   ADD INDEX idx_orders_discount
DOWN: DROP INDEX idx_orders_discount ✓

UP:   ADD COLUMN applied_at NOT NULL
DOWN: DROP COLUMN applied_at ✓ (but UP is already a blocker)
```

Flag any up operation with no corresponding down step.
Flag any down step that would cause data loss beyond what the up step implies.

---

### Step 4 — ORM model consistency check (AI)

Compare the new schema shape against ORM entity/model files:

```
Schema after migration:
  orders: { ..., discount_code VARCHAR(50) NULL, applied_at TIMESTAMP NOT NULL }

ORM Order entity:
  discountCode?: string       ✓ matches (nullable)
  appliedAt: Date             ✓ present
  [MISSING: no @Column for appliedAt] — Flag: ORM field exists but no decorator
```

Check for:

- Fields in schema not in ORM model
- Fields in ORM model not in schema
- Type mismatches (DB VARCHAR(50) vs ORM `number`)
- Nullable mismatch (DB NULL vs ORM non-optional)
- Index definitions in ORM that do not match new schema index

---

### Step 5 — Query impact check (AI)

Scan the diff and related service/repository files for queries that touch
the changed tables. Flag any query that:

- References a renamed or dropped column
- Does not handle a newly nullable column
- Would return different row counts due to new constraints
- Performs a full table scan on a table that now has a relevant index available

---

### Step 6 — Human validates production risk (Gate)

Present the lock risk matrix and any destructive operations to the human.

> "ADD COLUMN applied_at NOT NULL with no default will fail on orders table
> (estimated 8M rows). This is a blocker. Options: (a) add DEFAULT NOW() to
> migration, (b) make nullable and backfill, (c) add in two separate migrations."

> "ADD INDEX will acquire a shared lock. With 8M rows and ALGORITHM=INPLACE,
> estimated 2-5 min at low traffic. Run during maintenance window or use
> pt-online-schema-change?"

Human decides: fix, defer, split, or accept risk.

---

## Output format

```
[CHUNK: Database — <migration name>]
Operations: <N> additive, <N> structural, <N> destructive
Lock risk: <HIGH/MEDIUM/LOW/UNKNOWN if row counts missing>
ORM drift issues: <N>
Down migration gaps: <N>

Findings:
[SEVERITY] [FILE:LINE] — description
...

Questions for human:
1. ...

Reliability note: Lock risk assessment based on estimated row counts.
Actual production behaviour may differ. Human must validate before merge.
```
