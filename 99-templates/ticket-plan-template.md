# Ticket plan (copy into ticket folder as plan.md)

## Quick start (fill first)
- [ ] Header: ticket key + short title
- [ ] Goal / DoD: one clear sentence
- [ ] Affected schemas/topics/endpoints
- [ ] Migrations/backfill needed? (y/n)
- [ ] Tests: unit / integration / manual
- [ ] Rollout: feature flag or gradual

---

## 1) Header
- Ticket: [TICKET-123]
- Title: Short, imperative summary

## 2) Goal / Definition of Done
- Goal: One-line measurable outcome.
- DoD (checkable):
  - [ ] All unit tests pass
  - [ ] Integration tests for DB/Kafka pass
  - [ ] Deployed behind feature flag (if applicable)
  - [ ] No schema-breaking regressions in prod

## 3) Scope (In / Out)
- In:
  - Short bulleted list of what this ticket will change
- Out:
  - What is explicitly NOT part of this ticket

## 4) Inputs & dependencies
- Tables/columns: list names (e.g., `Orders`, `OrderEvents`)
- Kafka topics: input/output topics
- HTTP endpoints: affected APIs (method + path)
- Config / feature flags / external services
- Teams or owners to notify/review

## 5) Approach (high-level steps)
- [ ] Draft: small bullets of steps (no code)
  - e.g., add migration  update producer  add consumer handler
- [ ] Keep changes small and reversible; prefer feature flags

## 6) Data & backward compatibility
- Migration: add migration file, note downtime (none/minimal)
- Backfill: required? (yes/no)  if yes, brief plan and estimate
- Contracts: API/Kafka schema versioning strategy

## 7) Risks & mitigations (top 35)
- Risk: e.g., long migration lock  Mitigation: online migration strategy
- Risk: duplicate messages  Mitigation: idempotency keys
- Risk: sensitive data exposure  Mitigation: redact in logs

## 8) Test plan
- Unit tests: what to assert (edge cases)
- Integration tests: DB migrations, consumer/producer contract tests
- Manual checks: example requests, sample Kafka messages
- Observability checks: logs, metrics, traces to verify

## 9) Rollout plan
- Feature flag: yes / no. If yes, initial rollout % and monitoring window
- Gradual rollout steps and rollback criteria (errors, latency, business metrics)
- Post-deploy checks and who will monitor (pager/team)

## 10) Notes / open questions
- Blockers / who to ask
- Links to related tickets or runbooks

