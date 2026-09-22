# Test Case Index: pgvector_persistence_config

Parent test plan: [TestPlan.md](../TestPlan.md)

## End-to-End Test Cases

| Test Case ID | Title | Priority |
|--------------|-------|----------|
| [TC-E2E-001](TC-E2E-001.md) | Declarative pgvector CR emits the persistence block in the generated ConfigMap | P0 |
| [TC-E2E-002](TC-E2E-002.md) | OGX server pod reaches Running with declarative pgvector against live PostgreSQL | P0 |
| [TC-E2E-005](TC-E2E-005.md) | Release gate: configgen golden fixture for pgvector includes persistence and passes | P1 |
| [TC-E2E-006](TC-E2E-006.md) | Release gate: pkg/config unit test asserts the pgvector persistence block | P1 |

## Negative Test Cases

| Test Case ID | Title | Priority |
|--------------|-------|----------|
| [TC-NEG-001](TC-NEG-001.md) | Unfixed operator reproduces the pgvector persistence crash | P0 |
| [TC-NEG-002](TC-NEG-002.md) | Base config without storage.backends still emits persistence and the pod starts | P2 |

## Upgrade Testing

| Test Case ID | Title | Priority |
|--------------|-------|----------|
| [TC-UPG-001](TC-UPG-001.md) | Operator upgrade regenerates config with persistence for an existing pgvector CR | P1 |
