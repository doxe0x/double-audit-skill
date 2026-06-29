---
name: double-audit
description: Run a rigorous two-pass audit of a codebase, product repo, bot, automation, or security-sensitive workflow. Use this when the user asks for an audit, security review, code review, launch readiness review, risk review, hardening pass, PR review, bug hunt, or asks whether a repo is safe to ship. The skill combines discovery, adversarial verification, prioritized findings, fix execution, regression tests, security tooling, documentation updates, and a final handoff. It is designed for practical audits that produce working fixes, not vague recommendations.
---

# Double Audit

## Core idea

A useful audit is not a list of scary possibilities. It is a verified map of what can actually break, what matters most, and what was fixed. The skill runs in two passes:

1. **Identify**: inspect the system, map trust boundaries, list plausible failure modes, and separate real risks from noise.
2. **Adversarial verify**: try to disprove each finding. Reproduce it, trace the code path, check assumptions, and downgrade or delete weak findings.

The output should leave the repo safer and the owner less confused. If the user asks you to fix issues too, the job is not done until code is changed, tests pass, security checks run, and docs explain what changed.

## When to use

Use this skill for:

- security audits
- codebase audits
- bot and automation reviews
- payment, subscription, referral, or billing reviews
- deploy and ops safety checks
- launch readiness reviews
- PR review before merge
- hardening follow-up PRs
- "what else is risky here?"
- "are these all the fixes?"
- "continue fixing the problematic places you found"

Do not use it for shallow style review, general brainstorming, or pure content editing.

## Operating modes

Pick one mode from the user's request. If unclear, default to AUDIT ONLY and ask before writing code.

### 1. AUDIT ONLY

Inspect and report. No repo writes unless the user explicitly asks for fixes.

Output:
- executive summary
- scope and assumptions
- findings table
- evidence for each real finding
- false positives or deprioritized items
- recommended fix order
- verification commands to run after fixes

### 2. AUDIT AND FIX

Inspect, patch, test, and document. This is the mode for "fix the problematic places" or "make it cleaner".

Output:
- branch or commit info
- changed files
- findings fixed
- verification results with real command output
- remaining risks
- PR link if pushed

### 3. PR REVIEW

Review a branch or pull request. Do not rewrite the whole project. Focus on delta risk.

Output:
- verdict: approve, request changes, or comment only
- blocking issues
- non-blocking suggestions
- test gaps
- exact files and lines where possible

### 4. POST-FIX SECURITY POSTURE

Run after primary fixes. This mode is for cleanup that makes the repo easier to trust:
- remove static analyzer noise
- replace production asserts with explicit errors
- remove silent exception swallowing
- tighten deploy scripts
- add tests for fixed behavior
- update handoff docs

## Audit workflow

### Step 0: establish scope

Before touching code, determine:
- repository path and remote
- current branch and worktree status
- whether writes are allowed
- whether remote writes are allowed
- production risk: bot, payments, auth, wallets, data, deploy, cron, external APIs
- runtime stack: language, framework, DB, workers, service manager, deploy path

Commands often useful:

```bash
git status --short --branch
git remote -v
git log --oneline -5
find . -maxdepth 2 -type f | sort | sed 's#^./##'
```

If the repo has uncommitted user work, do not overwrite it. Inspect first.

### Step 1: map the system

Build a compact mental model before looking for bugs:
- entrypoints
- background loops and scheduled jobs
- external APIs
- database schema and migrations
- config and secrets
- payment or billing flows
- admin-only flows
- deploy scripts
- tests and CI

Create a one-page map in your notes. Do not report every file to the user unless useful.

### Step 2: identify trust boundaries

For each boundary, ask what input crosses it and what would happen if it is wrong, duplicated, stale, malicious, or delayed.

Common boundaries:
- user chat input
- callback data
- webhooks
- external API responses
- database rows from older schema versions
- env vars
- admin commands
- payment provider status
- referral or balance state
- deploy scripts running as root
- files restored from backup

### Step 3: run baseline checks

Use the repo's own tooling first. Then add general checks.

Python baseline:

```bash
python -m compileall -q .
python -m pytest -q
python -m pip_audit -r requirements.txt
bandit -q -r . -x ./.venv,./venv,./tests,./docs-site
```

Node baseline:

```bash
npm test
npm run lint
npm audit --audit-level=moderate
npm run build
```

Generic:

```bash
git diff --check
```

If a tool is missing, install it only if appropriate for the environment. Otherwise state that it was unavailable and continue with alternatives.

### Step 4: inspect high-risk paths manually

Do not rely only on scanners. Review the paths where real money, auth, or production state changes.

Payment and billing checklist:
- idempotency around payment finalization
- atomicity of balance mutations
- duplicate callback or poll handling
- stale invoice status
- refund or rejection paths
- promo/referral interactions
- crash window between activation and commission accrual
- manual admin approval double-click behavior

DB checklist:
- transaction boundaries
- migration idempotency
- schema backfill correctness
- dynamic SQL and allow-lists
- race conditions on counters and caps
- unique constraints for idempotency
- old rows with NULL or missing columns

Bot and automation checklist:
- admin checks on every privileged callback
- callback data tampering
- blocked users and retry logic
- background task crash behavior
- graceful shutdown and restart behavior
- rate limits
- polling cadence and duplicate processing

Deploy checklist:
- backup before schema changes
- WAL checkpoint for SQLite
- backup integrity check
- rollback path
- service restart on failure
- secrets not printed
- root-only scripts guarded
- dependency install conditions

### Step 5: classify findings

Every finding must have:
- severity: Critical, High, Medium, Low
- confidence: Confirmed, Likely, Hypothesis
- impact
- evidence
- reproduction or code path
- recommended fix
- test to prove the fix

Prefer fewer real findings over a long weak list.

Severity guide:
- **Critical**: direct loss of funds, auth bypass, remote code execution, irreversible data corruption.
- **High**: payment bypass, admin bypass, broad data exposure, production crash loop, unsafe deploy causing likely data loss.
- **Medium**: race condition with limited blast radius, missed commission, stale state, dependency vulnerability with realistic exploit path.
- **Low**: hardening, scanner noise, logging gaps, edge-case cleanup.

Confidence guide:
- **Confirmed**: reproduced or traced end-to-end.
- **Likely**: strong code evidence, not reproduced.
- **Hypothesis**: plausible, needs more proof.

Hypotheses should not be presented as facts.

### Step 6: adversarial verification

For each finding, try to kill it:
- Is there an existing guard elsewhere?
- Is the input actually user-controlled?
- Is the code path reachable in production?
- Does a DB constraint already prevent it?
- Does a background reconciler repair it?
- Is the scanner flag a false positive?
- Is the exploit economically meaningful?

If the finding survives, keep it. If not, downgrade or delete it.

### Step 7: fix order

Fix in this order:
1. data loss or funds risk
2. auth/admin bypass
3. atomicity and idempotency gaps
4. deploy safety
5. dependency vulnerabilities
6. recovery/reconciliation loops
7. scanner noise and code hygiene
8. docs and runbooks

Do not start with low-severity style cleanup while a payment race is open.

## Fix workflow

When the user asks you to fix findings:

1. Create or confirm a branch.
2. Patch the smallest safe change.
3. Add or update regression tests.
4. Run focused tests.
5. Run full tests and security checks.
6. Update docs only after code is verified.
7. Commit with a clear message.
8. Push and open PR if the user authorized remote writes.
9. Check PR status.

Example branch names:
- `fix/audit-hardening`
- `fix/security-posture-cleanup`
- `fix/payment-idempotency`
- `chore/audit-docs-refresh`

## Common fixes

### Atomic balance mutation

Bad pattern:
- check balance
- create payment row
- deduct balance later

Better pattern:
- single DB transaction
- conditional update where balance is sufficient
- insert payment only if update succeeded
- unique idempotency key if the action can retry

### Payment finalization

Finalization should be idempotent:
- claim pending payment with a conditional status update
- activate subscription only after claim succeeds
- mark completed once
- referral/promo side effects protected by unique constraints
- recovery loop for side effects that can be missed after a crash

### Global cap or quota checks

Quota checks need a final check at the mutation point. A pre-check in UI is not enough. Re-check inside the DB transaction or immediately before insert.

### Dynamic SQL

Prefer explicit SQL maps:

```python
FIELD_SQL = {
    "alert_on_new": "UPDATE users SET alert_on_new = ? WHERE id = ?",
    "min_value_usd": "UPDATE user_wallets SET min_value_usd = ? WHERE id = ?",
}

if field not in FIELD_SQL:
    raise ValueError("unknown field")
conn.execute(FIELD_SQL[field], (value, row_id))
```

Do not interpolate user-controlled table, column, sort, or where fragments. If a fragment must be dynamic, it must come from a closed allow-list.

### Runtime asserts

Do not rely on `assert` for production invariants. Python can strip asserts with optimization. Use explicit exceptions:

```python
if status not in {"paid", "rejected"}:
    raise ValueError(f"unsupported status: {status}")
```

### Silent exceptions

Silent exception swallowing is only acceptable for cancellation and only when the behavior is intentional. Otherwise log it:

```python
try:
    await task
except asyncio.CancelledError:
    pass
except Exception:
    logger.debug("background task raised during shutdown", exc_info=True)
```

### SQLite deploy safety

For SQLite-backed services:
- stop service first
- checkpoint WAL
- create backup with `.backup`, not plain `cp`
- run `PRAGMA integrity_check` on backup
- restart service on deploy failure
- prune backups only after a valid new backup exists

## Output templates

### Audit report

```markdown
## Verdict
<ship / do not ship / ship after fixes>

## Scope
- Repo: `<owner>/<repo>`
- Branch: `<branch>`
- Areas reviewed: ...
- Out of scope: ...

## Findings
| ID | Severity | Confidence | Area | Finding | Fix |
|---|---|---|---|---|---|
| A1 | High | Confirmed | Payments | ... | ... |

## Details
### A1: <title>
- Impact:
- Evidence:
- Reproduction/code path:
- Fix:
- Test:

## False positives / deferred
- ...

## Recommended order
1. ...
```

### Fix report

````markdown
## Done
- ...

## Verification
```text
pytest: ...
pip-audit: ...
bandit: ...
```

## Changed files
- ...

## Remaining risk
- ...

## PR
<link>
````

## Documentation handoff

After fixes, update the repo docs with:
- what was fixed
- current verification commands
- current test count
- deployment notes
- known deferred risks
- PR or branch references
- what should happen next

Good files to update:
- `HANDOFF.md`
- `CLAUDE.md`
- `README.md`
- `docs/security.md`
- runbooks under `deploy/`

Do not update docs first. Docs should reflect verified code, not intentions.

## Rules

- Never invent scan output. Run the command or say it was not run.
- Never claim a finding is exploitable unless you verified reachability.
- Never print secrets from `.env`, logs, CI, or config.
- Never run live destructive actions without explicit approval.
- Prefer small, reviewable PRs over one giant patch.
- Keep false positives visible, but do not pad the report with them.
- If you fix a scanner warning by suppression, explain why it is safe. Prefer code changes over suppression.
- Every serious finding needs a regression test or a clear reason why it cannot be tested.
- When the audit is about a live bot or production service, inspect first and avoid changing runtime state unless authorized.

## Reference files

- `references/audit-playbook.md`: detailed checklists, command packs, finding taxonomy, PR body template, and examples from payment/bot/SQLite audits.
