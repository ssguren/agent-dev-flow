Generate a deployment checklist for the current sprint / release. Analyze the changes and produce a comprehensive pre-deployment verification list.

## Step 1 — Analyze what changed

Read:
1. `git log <last-deploy-tag>..HEAD --oneline` or ask for the commit range
2. `specs/api-specification.md` — look for new or changed endpoints
3. `adr/` — any ADRs from this sprint that have deployment implications
4. Check for database migration files (look for `migration/`, `flyway/`, `liquibase/`, `*.sql` files)
5. Check for config changes (`.yml`, `.properties`, `.env.example` diffs)

## Step 2 — Categorize changes

Identify which of these change types are present:
- **New endpoints** → requires: API gateway config, firewall rules (if applicable)
- **Modified response shape** → requires: frontend coordination, client version check
- **New config keys** → requires: Nacos / env var configuration on all environments
- **DB schema changes** → requires: migration script, backup plan
- **New external service dependency** → requires: credentials, network access, approval
- **New cron jobs / scheduled tasks** → requires: scheduler config
- **Breaking changes** → requires: coordinated deployment or versioning

## Step 3 — Generate checklist

```markdown
# Deployment Checklist — Sprint N / YYYY-MM-DD

**Deploying**: [service name]
**Commit**: [hash]
**Deployer**: ___
**Scheduled time**: ___

---

## Pre-deployment

### Code & Build
- [ ] Build passes on CI
- [ ] All tests pass (test report linked: ___)
- [ ] No P0/P1 open bugs
- [ ] Code review complete (all PRs merged)

### Configuration
[For each new/changed config key found:]
- [ ] `<key>` set on staging ✓ / production ___
- [ ] Default value acceptable? (If no default, MUST be set before deploy)

### Database
[If migration files found:]
- [ ] Migration script reviewed: `<filename>`
- [ ] Migration is idempotent (safe to re-run)?
- [ ] Tested on staging DB?
- [ ] Backup taken before running?
- [ ] Estimated migration duration: ___ (flag if > 5 min — needs maintenance window)

### External Dependencies
[For each new external service:]
- [ ] API credentials configured in production
- [ ] Network access verified (firewall / VPC rules)
- [ ] External approval obtained (if required): ___

### Frontend Coordination
[If API response shape changed:]
- [ ] Frontend team notified?
- [ ] Frontend version compatible with this API version?
- [ ] Deploy frontend first or backend first? ___

---

## Deployment Execution

- [ ] Announce deploy start in team channel
- [ ] Disable health check if needed
- [ ] Run migration: `<command>`
- [ ] Deploy new version
- [ ] Verify service started (health endpoint: `GET /actuator/health`)
- [ ] Re-enable health check

---

## Post-deployment (first 15 minutes)

- [ ] Error rate normal (baseline: <0.1%)
- [ ] Response time normal (baseline: <200ms p95)
- [ ] No unexpected log errors
- [ ] Smoke test: [link to smoke test checklist]

---

## Rollback Plan

**Trigger condition**: Error rate > X% or [specific symptom]
**Decision maker**: [Tech Lead name]

Steps:
1. `<rollback command>`
2. [If migration ran: describe data rollback or accept forward-only]
3. Announce rollback in team channel

**Rollback time estimate**: ___ minutes
**Data rollback possible?**: Yes / No / Partial (explain)

---

## Sign-off

| Role | Name | Confirmed |
|------|------|-----------|
| Backend Dev (changes owner) | | |
| DevOps (deployment executor) | | |
| Tech Lead (go/no-go decision) | | |
```

## Output instruction

After generating the checklist, print:
```
Checklist generated
===================
Change types detected: [list]
Risk level: LOW / MEDIUM / HIGH

⚠️ Attention items:
[Any HIGH risk items that need special attention]

Recommended deploy time: [business hours / off-peak / maintenance window]
```
