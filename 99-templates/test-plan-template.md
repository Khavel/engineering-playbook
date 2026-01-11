# Test plan (copy into ticket folder as tests.md)

## Header
- Ticket: [TICKET-123]
- Scope (one line): short description of change

## 1) Test strategy overview
- Types needed (brief): unit, integration (DB/Kafka/API), manual, observability
- Why: focus on correctness of logic, contracts, and production verification

## 2) Unit tests
- What to cover (brief bullets):
  - Pure logic branches and transformations
  - Validation and edge cases
  - Error paths and exception handling
- Determinism/mocks: list heavy deps to mock or fake

## 3) Integration tests
- DB:
  - Migrations apply cleanly, queries return expected shapes
  - Constraints/index behaviour for critical queries
- Kafka:
  - Consumer/producer contract tests (schema, headers)
  - Idempotency/dup handling tests
- HTTP/API:
  - Contract tests for endpoints (status codes, payloads)

## 4) Manual / exploratory tests
- Steps to verify locally or in test env (short checklist):
  - [ ] Run service locally and hit X endpoint with Y payload
  - [ ] Produce sample message to topic and observe consumer
  - [ ] Validate migration applied and data looks correct
- Example requests/messages (paste one or two examples)

## 5) Observability checks
- Logs: search for structured log entries, request/trace ids
- Metrics: increase in success/failure counters, latency gauges
- Traces: end-to-end trace present for example flow

## 6) Evidence (fill after implementation)
| Type | Link / note |
|---|---|
| Unit tests | |
| Integration tests | |
| Manual verification (screenshots/logs) | |
| Dashboards / metrics | |

## 7) Gaps / follow-ups
- Known missing coverage (brief) and why
- Deferred tests with acceptance criteria and owner

---

Execution checklist (mark when done):
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] Manual verification completed and evidence linked
- [ ] Observability validated in test/staging

