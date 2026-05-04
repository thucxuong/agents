# Review sub-skill: Config / infra chunk

Chunk type: **Config / infra**
Applicable when: diff includes CI/CD config, Dockerfiles, environment variable
files, deploy manifests, or infrastructure-as-code.

---

## Reliability ceiling

**Medium → High** for completeness and hygiene checks.
Whether a new env var is actually provisioned in live infrastructure is a hard
human gate — the agent has no access to the real environment.

---

## Minimum information set

- [ ] All environment config files (dev, staging, prod, CI) — not just the changed one
- [ ] Existing env var inventory (`.env.example`, `docker-compose.yml`, docs, or grep output)
- [ ] The diff for this chunk
- [ ] Dockerfile(s) if changed

If only one environment's config is provided → **stop, request all environments.**
Cross-environment completeness is the primary value of this review — checking
one env in isolation misses the most common failure mode.

---

## Information pipeline

### Step 1 — Build env var matrix (AI)

Read all environment config files. Build a matrix: variable name × environment.

```
Variable               | dev | staging | prod | CI  |
----------------------|-----|---------|------|-----|
DATABASE_URL          |  ✓  |    ✓    |  ✓   |  ✓  |
REDIS_URL             |  ✓  |    ✓    |  ✓   |  –  |  ← missing in CI
STRIPE_SECRET_KEY     |  ✓  |    ✓    |  ✓   |  ✓  |
NEW_DISCOUNT_API_KEY  |  ✓  |    –    |  –   |  –  |  ← NEW, only in dev
SENDGRID_API_KEY      |  –  |    ✓    |  ✓   |  –  |  ← missing in dev, CI
```

Flag every cell that is:

- **Missing** in an environment where it is likely needed
- **Present only in dev** (new variable not yet provisioned elsewhere)
- **Present in prod but not in staging** (staging drift)

---

### Step 2 — New variable risk assessment (AI)

For each variable added in the diff:

```
NEW_DISCOUNT_API_KEY:
  Present in: dev only
  Used in: discount.service.ts (required, will throw if missing)
  Risk: BLOCKER — application will fail in staging/prod without this var
  Action needed: provision in staging and prod secret manager before merge
```

For each new variable, determine:

- Is it required or optional? (Check usage in code — is there a fallback?)
- Which environments need it?
- Is it a secret (should be in secret manager) or a config value (can be in env file)?

---

### Step 3 — Secret hygiene scan (AI)

Scan all config files in the diff for:

- Hardcoded credential patterns: `password=`, `secret=`, `key=`, `token=`, `api_key=`
  followed by a non-placeholder value
- Real-looking values: UUIDs, long alphanumeric strings, `sk_live_`, `pk_live_`
- `.env` files committed with real values (not `.env.example`)

```
.env.staging — LINE 12: STRIPE_SECRET_KEY=sk_live_abc123... — BLOCKER: real secret committed
docker-compose.yml — LINE 34: POSTGRES_PASSWORD=devpass — nit: weak dev password, acceptable if dev-only
```

---

### Step 4 — CI pipeline logic check (AI)

If CI config changed, read the step sequence and verify logical ordering:

Required ordering rules:

- Install dependencies before build/test
- Build before test (if build step exists)
- Test before deploy
- Secrets/env vars available before steps that consume them
- Caching steps before the steps they cache for

```
.github/workflows/deploy.yml:
  Step order: install → test → build → deploy ⚠
  Issue: test runs before build — if tests import built artifacts, this will fail
  Recommended: install → build → test → deploy
```

Also check:

- Are test steps allowed to fail silently (`continue-on-error: true` on test steps)?
- Are deploy steps gated on test success?
- Is there a rollback or manual approval step for production deploys?

---

### Step 5 — Docker consistency check (AI, if Dockerfile changed)

If Dockerfile changed:

- Is the base image pinned to a specific version or using `latest`?
- Are build args consistent with env var inventory?
- Does the exposed port match what the application config expects?
- Are secrets used in build args (bad — they appear in image layers)?
- Is a `.dockerignore` present and does it exclude `.env` files?

---

### Step 6 — Human validates live provisioning (Gate)

Present the env var matrix with gaps highlighted.

For each new variable missing from an environment:

> "NEW_DISCOUNT_API_KEY is used in discount.service.ts and will throw if missing.
> It is only present in dev. Before this PR merges, it must be provisioned in
> staging and prod. Can you confirm this has been done or is scheduled?"

The agent cannot verify what is actually provisioned in a live secret manager.
This gate is mandatory.

---

## Output format

```
[CHUNK: Config — <files changed>]
Environments covered: <N>
New variables: <N>
Missing across environments: <N> gaps
Secret hygiene issues: <N>
CI ordering issues: <N>

Findings:
[SEVERITY] [FILE:LINE] — description
...

Env var matrix: [attach matrix]

Questions for human:
1. ...

Reliability note: Completeness checks are reliable. Whether variables are
actually provisioned in live infrastructure requires human confirmation.
```
