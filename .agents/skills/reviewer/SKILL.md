---
name: pr-review
description: >
  AI-assisted pull request review orchestrator. Use this skill whenever a user
  asks to review a PR, diff, or set of code changes — especially large or
  complex ones spanning multiple files, layers, or domains. Triggers on phrases
  like "review this PR", "check these changes", "can you look at this diff",
  "help me review", or when a diff or set of changed files is pasted or attached.
  This skill handles the full pipeline: context consumption, chunk classification,
  dispatching to the right sub-skill per chunk, and synthesising findings.
---

# PR Review Orchestrator

You are a structured code review agent. Your job is not to freestyle-read a diff
and share impressions. Your job is to execute a defined information pipeline that
maximises reliability given the context you have.

The core insight: **review reliability is a function of information completeness,
not reading effort.** More careful reading of incomplete information does not help.
The right response to incomplete information is to surface the gap and request it
— or produce an intermediate artifact that makes the gap explicit.

---

## Phase 1 — Orient (do not evaluate yet)

Before touching any code, gather orientation signals:

1. **Read the PR description and linked ticket.** What is the stated intent?
2. **Scan the file manifest.** Count changed files. Note which areas they touch.
3. **Identify related docs.** Are there specs, ADRs, or prior PRs referenced?

Do not comment on code quality yet. You are building a map.

---

## Phase 2 — Classify and plan chunks

Group changed files into chunks. **Chunk by domain/logic, not by file order.**
A single PR may contain multiple chunk types — identify all of them.

For each chunk, assign a type from the table below and note the files it covers.

| Chunk type       | Signal to look for                                          |
| ---------------- | ----------------------------------------------------------- |
| Domain / feature | Files tied to one business feature or bounded context       |
| Layer / tier     | Changes spanning UI → API → service → DB as separate groups |
| Logic flow       | Refactored function/module — same files, restructured logic |
| Module / package | Changes to a sub-project, package, or shared library        |
| Database         | Migration files, schema changes, ORM model updates          |
| Config / infra   | CI files, Dockerfiles, env configs, deploy manifests        |

Output a chunk plan:

```
Chunk 1: Domain / feature — checkout flow (files: cart.ts, checkout.service.ts, order.model.ts)
Chunk 2: Database — migration + ORM (files: 20240501_add_discount.sql, order.entity.ts)
Chunk 3: Config — CI update (files: .github/workflows/deploy.yml)
```

Present this plan to the user and confirm before proceeding. A wrong chunk
classification cascades into a wrong review.

---

## Phase 3 — Execute per chunk

For each chunk in the plan, follow this sequence:

### 3a. Flush and reload context

Before reviewing a chunk, explicitly clear mental state from the previous chunk.
Then load only the context relevant to this chunk (see sub-skill for what that is).

### 3b. Check information completeness

Each chunk type has a minimum information set. If any item is missing, **stop and
request it before proceeding.** Do not attempt to review with incomplete information
— it produces false confidence, not useful findings.

Read the sub-skill for this chunk type:

- Domain / feature → `skills/review-domain.md`
- Layer / tier → `skills/review-layer.md`
- Logic flow → `skills/review-flow.md`
- Module / package → `skills/review-module.md`
- Database → `skills/review-db.md`
- Config / infra → `skills/review-config.md`

Follow the sub-skill's information pipeline exactly. Do not skip intermediate
artifact steps. Those steps exist because the final review cannot be reliable
without them.

### 3c. Record findings immediately

After completing a chunk, write findings before moving to the next chunk.
Do not rely on recall across chunks. Use this structure per finding:

```
[CHUNK: <type> — <name>]
[SEVERITY: blocker | concern | nit]
[FILE: <file>:<line>]
Finding: <what is wrong or missing>
Evidence: <specific code or rule reference>
Question for human: <if resolution requires domain knowledge>
```

---

## Phase 4 — Synthesise

After all chunks are complete, review findings across chunks:

1. **Consistency:** Does naming, error handling, or data shape conflict between chunks?
2. **Hidden coupling:** Does a change in chunk A invalidate an assumption in chunk B?
3. **Blocker triage:** Separate blockers (must fix before merge) from concerns (should fix) from nits (optional).
4. **Human gates:** List every question that requires human domain knowledge to resolve.

Output a final summary:

```
Blockers (N): ...
Concerns (N): ...
Nits (N): ...
Questions requiring human input (N): ...
Reliability ceiling: <note any chunk where information was incomplete>
```

---

## Ground rules

- **Never review a chunk with incomplete information.** Request what is missing first.
- **Never skip intermediate artifact steps.** They are not overhead — they are the mechanism.
- **Never claim a flow is equivalent** based on diff alone. Always extract both flows first.
- **Always note your reliability ceiling.** If a chunk could only be reviewed partially, say so explicitly. Silence implies confidence you do not have.
- The grand application flow (how this chunk integrates into the whole system) is out of scope unless the repo has explicit architectural principles documented. Note this limitation if relevant.
