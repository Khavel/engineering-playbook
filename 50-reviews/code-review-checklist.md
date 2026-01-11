# Code review checklist

## Quick triage (first 60s)
- [ ] Does the PR title/description explain the intent and risk succinctly?
- [ ] Can I run the change locally or do CI checks pass (build/tests)?
- [ ] Scan for high-risk areas: DB schema, migrations, auth, Kafka topics, secrets

## Correctness
- [ ] Implementation matches the spec/issue exactly (no hidden behavior)
- [ ] Data model changes have clear migrations and backfills documented
- [ ] EF Core mappings match SQL schema (types, nullability, indexes)

## Edge cases & validation
- [ ] Inputs validated server-side (lengths, formats, ranges)
- [ ] Errors are handled explicitly (not swallowed) and return appropriate status codes
- [ ] Boundary conditions covered (empty sets, nulls, max sizes)

## Performance & DB
- [ ] Queries use indexes; N+1 problems avoided (include EXPLAIN if complex)
- [ ] Avoid SELECT *; fetch only needed columns for hot paths
- [ ] Long-running operations moved off request path (background jobs)

## Concurrency / Idempotency (Kafka + general)
- [ ] Consumers handle duplicates and are idempotent or use dedupe keys
- [ ] Offsets/commits handled safely; errors dont cause silent data loss
- [ ] Transactions used where multiple DB writes must be atomic

## Security & privacy
- [ ] No secrets or credentials checked in or logged
- [ ] Input/output encoding to prevent injection (SQL, HTML, logs)
- [ ] Least privilege for DB access and external systems

## Observability
- [ ] Structured logs: include request id, user id (when safe), and correlation id
- [ ] Metrics for important counters/gauges (errors, latency, processing rate)
- [ ] Traces link across services for main request flows and async jobs

## Tests
- [ ] Unit tests cover logic branches; fast and deterministic
- [ ] Integration tests cover DB migrations and external contracts (Kafka/HTTP)
- [ ] Tests assert observable behaviour, not implementation details

## Maintainability
- [ ] Code is idiomatic C#/.NET and follows project conventions
- [ ] Small, focused functions; complex flows decomposed with names
- [ ] No large TODOs or commented-out code left behind

## Documentation / Comments
- [ ] Public APIs and complex algorithms have brief comments or examples
- [ ] Migration/backfill steps and runbooks referenced in PR description

## Common review comments (copy/paste)
- Please add a short test that demonstrates the expected behaviour.
- Can you avoid loading related entities here (N+1); use a projection or include?
- This should be idempotent  what happens if the message is processed twice?
- Move long-running work to a background job to keep request latency low.
- Please add or update a migration and document any required backfill.
- Dont log sensitive data (PII, tokens); redact before logging.
- Prefer explicit DTOs/mappings instead of passing EF entities to callers.
- Add metrics/tracing around this hot path for future debugging.
- Clarify expected error codes in the API contract and tests.
- Use parameterized queries/ORM parameters to avoid SQL injection.
