# Question Generation Heuristics

Use these heuristics to discover strong questions. Do not output these categories mechanically — use only the ones relevant to the PR.

---

## 1. Design Decision

Look for: new abstraction, new component, changed architecture, different control flow, different data flow.

Ask about:
- why this design was chosen
- what alternatives were considered
- what trade-off was accepted

---

## 2. Responsibility Movement

Look for: logic moved between layers, validation moved, state management moved, business rule moved, ownership of data changed.

Ask about:
- why the responsibility belongs there
- what now depends on that location
- what duplication or bypass risk exists

---

## 3. Boundary Change

Look for: new API boundary, database boundary, queue/event boundary, service-to-service call, external dependency, permission boundary.

Ask about:
- failure behavior and retries
- timeout handling
- idempotency
- consistency
- observability

---

## 4. Risky Detail

Look for: transaction handling, concurrency, ordering, caching, migration, deletion, fallback behavior, partial failure, silent failure.

Ask about:
- what can go wrong
- how it is detected
- how it is recovered
- whether rollback is safe

---

## 5. Test Strategy

Look for: tests changed, tests absent, only happy-path tests, mocks replacing real behavior, missing integration coverage.

Ask about:
- what assumption the tests verify
- what failure path is not covered
- what would break without being caught

---

## 6. Operational Impact

Look for: logging changes, metrics changes, alerting changes, rollout strategy, feature flag, migration, rollback path.

Ask about:
- how operators know it works vs. fails
- how to disable or rollback
- what customer impact is expected
