# Double Audit playbook

This file backs the main `SKILL.md`. Read it when you need deeper checklists, audit commands, finding taxonomy, examples, or PR templates.

## 1. Audit mental model

A good audit has three outputs:

1. **Truth**: what is actually risky, not what merely looks risky.
2. **Order**: what to fix first.
3. **Proof**: what command, test, reproduction, or code path proves the claim.

The most common audit failure is stopping after the first output. A repo owner does not need a scary list. They need a reliable fix plan and proof that the fix worked.

## 2. The double-pass method

### Pass A: identify

Look for possible failure modes quickly and broadly:
- obvious bugs
- missing checks
- unsafe dependency pins
- suspicious scanner output
- race conditions
- missing tests
- fragile deploy scripts
- unclear runbooks

Do not over-explain yet. Collect candidates.

### Pass B: adversarial verify

For each candidate, try to disprove it:
- Is there a guard before this code path?
- Is the input truly attacker-controlled?
- Is the function reachable from production?
- Does a DB constraint or unique index already make it safe?
- Is a background reconciler fixing the crash window?
- Is the scanner warning only a static-analysis false positive?
- Would this actually matter economically?

Only surviving issues become findings.

## 3. Scope template

Use this at the top of serious audits:

```markdown
## Scope
- Repo:
- Branch/commit:
- Runtime:
- Production status:
- Areas reviewed:
- Areas not reviewed:
- Writes allowed:
- Remote writes allowed:
```

If the user did not authorize writes, keep the audit read-only.

## 4. Command packs

### Python repo

```bash
git status --short --branch
git log --oneline -5
python -m compileall -q .
python -m pytest -q
python -m pip_audit -r requirements.txt
bandit -q -r . -x ./.venv,./venv,./tests,./docs-site
ruff check .
mypy .
git diff --check
```

Run only commands that fit the repo. If a project does not use mypy or ruff, do not invent their output.

### Node repo

```bash
git status --short --branch
npm ci
npm test
npm run lint
npm run typecheck
npm run build
npm audit --audit-level=moderate
git diff --check
```

### SQLite service

```bash
sqlite3 bot.db 'PRAGMA integrity_check;'
sqlite3 bot.db 'PRAGMA journal_mode;'
sqlite3 bot.db 'PRAGMA foreign_keys;'
sqlite3 bot.db '.schema'
```

For live production DBs, do not run write statements. Prefer copies or read-only checks.

### GitHub PR

```bash
gh pr view <number> --json title,state,baseRefName,headRefName,mergeable,isDraft,url
gh pr diff <number>
gh pr checks <number>
```

## 5. Finding taxonomy

### Atomicity

Symptoms:
- check then mutate in separate transactions
- balance deducted after payment row creation
- cap checked in UI but not at insert
- multiple writes with no transaction around them

Fix pattern:
- one transaction
- conditional update
- unique idempotency key
- final re-check at mutation point

Test pattern:
- duplicate request
- insufficient balance
- concurrent-ish repeated call
- crash window if possible

### Idempotency

Symptoms:
- payment poller can finalize the same invoice twice
- callback double click creates duplicate side effects
- referral commission can accrue twice
- retry creates duplicate rows

Fix pattern:
- status transition guarded by old status
- unique index on side effect key
- `INSERT OR IGNORE` for recoverable side effects
- claim step returns true only once

Test pattern:
- call finalizer twice
- replay callback
- run reconciliation after finalization

### Authorization

Symptoms:
- admin callback does not re-check admin
- action trusts user id from callback data
- endpoint checks role in UI only
- support/staff binding by username instead of immutable id

Fix pattern:
- re-check actor on every privileged action
- bind privileged users by stable numeric id
- do not trust callback payload for ownership

Test pattern:
- non-admin tries callback
- user tries another user's id

### Data migrations

Symptoms:
- migration not idempotent
- partial migration leaves broken schema
- `ALTER TABLE` assumptions about missing columns
- destructive rebuild with foreign keys enabled incorrectly
- no test for old schema

Fix pattern:
- detect current schema
- additive migrations where possible
- scratch tables with preserved ids
- explicit commits around SQLite FK toggles
- tests from old schema to new schema

Test pattern:
- fresh DB
- old DB migration
- partial migration rerun if feasible

### External API trust

Symptoms:
- assumes JSON success on HTTP 500
- no timeout
- no retry/backoff
- treats missing fields as real zero values
- no suspicious-data guard

Fix pattern:
- timeout
- HTTP status check
- parse failure handling
- schema validation at boundary
- cache and skip suspicious empty responses

Test pattern:
- HTTP 500
- non-JSON body
- missing fields
- empty list after non-empty previous state

### Deploy safety

Symptoms:
- `git pull` while service is running during schema changes
- no DB backup
- plain `cp` of SQLite WAL DB
- no backup integrity check
- service left stopped on script failure
- scripts require root but do not check root
- secrets printed to logs

Fix pattern:
- stop service
- checkpoint WAL
- `.backup` command
- `PRAGMA integrity_check`
- `trap` restart on failure
- root guard
- backup retention guard

Test pattern:
- `bash -n`
- dry-run where possible
- manual review of failure path

## 6. Scanner interpretation

### Bandit

Bandit is useful for finding classes of risk, but not all warnings are findings.

Treat as likely real:
- shell injection with user input
- unsafe deserialization
- hardcoded secrets
- weak crypto in auth or payment code
- broad `subprocess(..., shell=True)`

Often false positive if verified:
- dynamic SQL where fragment is from a closed allow-list
- `urlopen` where scheme and host are pinned
- asserts in tests

Preferred order:
1. rewrite code so scanner no longer flags it
2. if impossible, add a narrow suppression with a reason
3. never globally disable a rule because one line is safe

### pip-audit or npm audit

Classify dependency findings by exploitability:
- Is the vulnerable package imported at runtime?
- Is the vulnerable feature used?
- Is the service exposed to untrusted input?
- Can the package be upgraded without breaking pins?
- Is the dependency only transitive through an unused library?

If a stale library pins a vulnerable dependency, consider removing or replacing that library.

## 7. Fix PR body template

```markdown
## Summary
- fixed ...
- added regression coverage for ...
- updated docs/runbook for ...

## Findings fixed
| ID | Severity | Area | Fix |
|---|---|---|---|
| A1 | High | Payments | ... |

## Verification
- [x] `python -m pytest -q`
- [x] `python -m pip_audit -r requirements.txt`
- [x] `bandit -q -r . -x ./.venv,./tests`
- [x] `git diff --check`

## Notes
- Deferred: ...
- Manual deploy step: ...
```

## 8. Audit report template

```markdown
## Verdict
<ship / do not ship / ship after fixes>

## Executive summary
- ...

## Findings
| ID | Severity | Confidence | Area | Finding | Recommendation |
|---|---|---|---|---|---|
| A1 | High | Confirmed | Payments | ... | ... |

## Details
### A1: <title>
- Impact:
- Evidence:
- Code path:
- Fix:
- Test:

## False positives or deferred
- ...

## Suggested order
1. ...
```

## 9. Post-fix posture checklist

After the main vulnerabilities are fixed, run a cleanup pass:

- static analyzer baseline clean or documented
- dependency audit clean or documented
- production asserts removed
- silent exception swallowing removed
- dynamic SQL minimized
- HTTP clients handle status, timeout, and non-JSON responses
- deploy scripts have safe failure behavior
- docs reflect exact current commands and test counts
- PR body includes verification commands

This pass matters because it lowers future review cost. A clean baseline lets the next real issue stand out.

## 10. Example: payment bot audit

High-signal findings to look for:

- Balance payment created without atomic deduction.
- Invoice finalization can run twice.
- Referral commission can be missed if process crashes after subscription activation.
- Manual payment approval can be double-clicked.
- Crypto payment library pins vulnerable dependencies.
- Global wallet cap checked only before user confirmation.
- Deploy script pulls schema changes without DB backup.

Good fix sequence:

1. Make balance payment atomic.
2. Make finalization idempotent.
3. Add final global cap re-check.
4. Replace vulnerable payment dependency.
5. Add referral reconciliation loop.
6. Add safe deploy-with-backup script.
7. Clean static analyzer baseline.
8. Update handoff docs.

## 11. Example: bot callback audit

Questions:
- Does every callback re-check the actor?
- Is the callback payload enough to act on another user's object?
- Is the user id taken from `update.effective_user`, not from payload?
- Are admin buttons bound to admin checks at callback time?
- Are blocked users marked and skipped?
- Are retries bounded?

## 12. Example: SQLite migration audit

Questions:
- Can migration run twice?
- What happens if process dies halfway?
- Are foreign keys toggled outside a transaction?
- Are old ids preserved?
- Does migration handle missing columns from old versions?
- Is there a test that starts from old schema and opens the app with new code?

## 13. Language for risk

Use precise labels:

- "confirmed" when reproduced or traced end-to-end
- "likely" when code evidence is strong but not reproduced
- "hypothesis" when plausible but unproven
- "false positive" when a tool flagged it but manual review proved it safe
- "deferred" when real but not worth fixing now

Avoid dramatic wording unless the evidence supports it.

## 14. Final response checklist

Before reporting back:
- Did you satisfy the user's requested mode?
- Did you run real commands for every verification claim?
- Did you avoid printing secrets?
- Did you separate fixed from remaining issues?
- Did you update docs after code, not before?
- If you pushed, did you include the PR link and status?
- If CI is absent, did you say that checks were not reported?
