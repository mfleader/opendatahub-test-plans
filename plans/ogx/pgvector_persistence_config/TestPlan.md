---
feature: pgvector_persistence_config
source_key: RHAIENG-6374
source_type: issue
status: Draft
author: OGX QE
components:
- OGX Core
additional_docs:
- https://github.com/ogx-ai/ogx-k8s-operator/blob/be5158708a5d955063a859e29104a87f8bed19a4/config/crd/bases/ogx.io_ogxservers.yaml
- https://github.com/ogx-ai/ogx-k8s-operator/blob/be5158708a5d955063a859e29104a87f8bed19a4/config/samples/starter-config-configmap.yaml
- https://github.com/ogx-ai/ogx-k8s-operator/blob/be5158708a5d955063a859e29104a87f8bed19a4/cmd/configgen/testdata/p1-pgvector-config/cr.yaml
- https://github.com/ogx-ai/ogx-k8s-operator/blob/be5158708a5d955063a859e29104a87f8bed19a4/pkg/config/generator.go
last_updated: '2026-09-21'
version: 1.0.0
reviewers: []
---
# pgvector_persistence_config Test Plan

**OGX QE**: **Declarative pgvector Persistence Fix Verification**

**Strategy**: [RHAIENG-6374](https://redhat.atlassian.net/browse/RHAIENG-6374)

---

## 1. Executive Summary

### 1.1 Purpose

RHAIENG-6374: when an `OGXServer` CR declares `spec.providers.vectorIo.remote.pgvector`,
`expandPgvectorProvider` (`pkg/config/provider.go`) omits the `persistence` block, the reference
naming which key-value backend and namespace hold the provider's vector store records, from the
generated provider configuration, and the server crashes at startup with
`AttributeError: 'NoneType' object has no attribute 'backend'`, raised in `kvstore.py` from the
`kvstore_impl(self.config.persistence)` call in `pgvector.py`. The fix
emits `persistence` with `backend: kv_default` and a namespace of `vector_io::` followed by the
provider's explicit `id`, so two instances on one OGXServer do not share vector store records. A CR
that sets no `id` gets `vector_io::pgvector`, the value every shipped base config uses, so records
written before the move to the declarative path stay reachable.

### 1.2 Scope

#### In Scope (OGX QE Responsibilities)

Objectives #1 to #5 in Section 1.3. The generated ConfigMap named in
`status.configGeneration.configMapName` is the verification mechanism.

#### Out of Scope (Other Teams)

- The pgvector `vector_index` shape mismatch with the OGX schema
- A base config that defines `storage.backends` without `kv_default`. The emitted reference is
  then unresolved and the server fails at startup with an unknown backend error

### 1.3 Test Objectives

1. The generated pgvector provider config carries `persistence`, read from the ConfigMap named in
   `status.configGeneration.configMapName` (AC: #1 - pgvector provider config must contain
   persistence with backend kv_default and namespace vector_io::pgvector)
2. Against a reachable PostgreSQL with the vector extension, the server pod starts clean (AC: #2 -
   pod must reach Running with no AttributeError on persistence given a reachable PostgreSQL with
   the vector extension)
3. The golden fixture carries the block and `go test ./cmd/configgen/...` passes (AC: #3 -
   cmd/configgen testdata for the pgvector case must include the persistence block and the
   configgen test must pass)
4. `go test ./pkg/config/... -run TestExpandProviders_PgvectorPersistence` passes (AC: #4 - a unit
   test in pkg/config must assert the persistence block for pgvector)
5. The `${env.*}` path resolved through `spec.workload.overrides.env` still works (AC: #5 - base
   config providers and the env-override path must be unchanged)

---

## 2. Test Strategy

### 2.1 Test Levels

- **E2E System Testing**. The fix changes a controller configuration path with no dashboard
  surface, so there is no user interface testing.

### 2.2 Test Types

Positive and negative. See [test_cases/INDEX.md](test_cases/INDEX.md).

### 2.3 Test Priorities

P0 is the crash-fix scenario itself (Objective: #1) (Objective: #2). P1 is the two release gates
and the non-regression checks (Objective: #3) (Objective: #4) (Objective: #5). P2 is the two edge
cases the fix exposes (Objective: #1).

---

## 3. Test Environment

### 3.1 Infrastructure & Configuration

- OpenShift Container Platform on AWS, RHOAI 3.5 Early Access 2, per the bug environment field. A kind
  cluster is acceptable: nothing in the fix or the crash is specific to OpenShift.
- `OGXServer` CRD `v1beta1`. The CR shape and the status fields depend on this version.
- An operator image built from the fix branch. The fix is committed on
  `fix/rhaieng-6374-pgvector-persistence` but not merged, and the latest release tag is
  `v0.15.0`, which carries the praxis merge but not the fix, so no released image has it:

  ```bash
  git checkout fix/rhaieng-6374-pgvector-persistence
  make image-build image-push IMG=<your-registry>/ogx-operator:rhaieng-6374
  make deploy IMG=<your-registry>/ogx-operator:rhaieng-6374
  ```

- A second image built from `main` at `be51587`, the fix's parent, without the fix, for
  TC-NEG-001.
- OGX server image, set through `spec.distribution.image`, because `spec.distribution.name: rh`
  is not a key in `distributions.json` and `validateDistributionName` rejects it. The digest below
  is the build the bug was reported on. An equivalent `quay.io` tag is acceptable where
  `registry.redhat.io` credentials are unavailable, and the case run records which was used:

  ```text
  registry.redhat.io/rhoai/odh-ogx-core-rhel9@sha256:fd71c3ed486398002dac8aa09e1e4162e2bf740d33e192e1bbc2f3233d4f766a
  ```

- PostgreSQL with the `vector` extension, reachable from the OGXServer namespace (Section 3.5).
- Base config ConfigMap and password Secret in that namespace, both labelled
  `ogx.io/watch: "true"`. The base config must declare an `inference` provider, because every
  vector_io provider spec declares `api_dependencies=[Api.inference]`.
- Workload storage: PVC-backed mount at `/.ogx`, size `5Gi`, replicas `1`.
- Legacy mode. Every case leaves `spec.praxisMode` unset.

### 3.2 Test Data Requirements

| Data | Example | Purpose |
|------|---------|---------|
| Declarative pgvector CR | `host`, `port`, `db`, `user`, `distanceMetric`, `vectorIndex`. Only `password`, a SecretKeyRef, is required | Objectives #1 and #2 |
| The same CR with `id: pgvector` | The reproduction CR, against host `postgres` | Confirms the env var becomes `OGX_PGVECTOR_PASSWORD` |
| Base config ConfigMap, labelled `ogx.io/watch: "true"` | `version`, `distro_name`, `apis`, an `inference` provider | Generation input, and the Objective #5 comparison fixture |
| pgvector password Secret | Opaque, key `password`, labelled for watch | Satisfies the required field |
| Golden fixture | `cmd/configgen/testdata/p1-pgvector-config/want-config.yaml` | Objectives #1 and #3 |
| Pre-fix baseline | `vector_io[].config` with no `persistence` key | Expected output of TC-NEG-001 |

### 3.4 Test Tools

`oc` or `kubectl` for the CR, ConfigMap, Secret and status fields. `yq` to read
`persistence.backend` and `persistence.namespace` from the generated ConfigMap. `oc logs` and
`oc get events` for the crash signature. `psql` for the `vector` extension. `go test` at the head of
`fix/rhaieng-6374-pgvector-persistence` for the two release gates.

---

### 3.5 PostgreSQL Fixture

The cluster cases assume an in-namespace PostgreSQL reachable as host `postgres` on port `5432`,
database `postgres`, user `postgres`, with the password in Secret `postgres-secret` and the
`vector` extension installed. Deploy `docker.io/pgvector/pgvector:pg16` with that Secret and a
Service named `postgres`, then run
`psql -U postgres -d postgres -c 'CREATE EXTENSION IF NOT EXISTS vector;'` in the pod.

---

## 4. Interfaces Under Test

| Interface | Type | Purpose |
|-----------|------|---------|
| `OGXServer` v1beta1 `spec.providers.vectorIo.remote.pgvector`, `spec.baseConfig` | CRD | Declare the provider and base config the operator expands |
| `OGXServer` v1beta1 `status.configGeneration.configMapName`, `status.phase`, `status.conditions` | CRD | Locate the generated ConfigMap and confirm generation and pod phase |
| `oc get configmap <generated-name>` | CLI | Assert the `persistence` block under `providers.vector_io[].config` |
| `oc logs` and `oc get events` on the OGXServer pod | CLI | Detect or confirm the absence of the `AttributeError` |
| `go test ./cmd/configgen/...` | CLI | Release gate: the golden fixture carries the block |
| `go test ./pkg/config/... -run TestExpandProviders_PgvectorPersistence` | CLI | Release gate: the unit test asserts the block |

## 5. Test Cases

Seven test cases live in [test_cases/](test_cases/), indexed in
[test_cases/INDEX.md](test_cases/INDEX.md).

---

## 7. Non-Functional Requirements

### 7.2 Upgrade/Migration

Covered by TC-UPG-001 (Objective: #1) (Objective: #5)

### 7.5 Security

**Not Applicable**. Credential handling is unchanged. The generated config still points at the
password through an `${env.*}` reference rather than embedding it. The variable name follows the
provider id, so `OGX_PGVECTOR_PASSWORD` for `id: pgvector` and `OGX_REMOTE_PGVECTOR_PASSWORD`
for a CR that sets no `id`. The password variable and the persistence namespace fall back to
different defaults, `remote-pgvector` and `pgvector`, because only the namespace keys stored
records.

---

## 8. Risks and Mitigation

| Risk | Impact | Probability | Mitigation |
|------|--------|-------------|------------|
| Declaring a `vector_io` provider replaces the base config's `vector_io` list (`MergeProviders`), whereas `buildFinalConfig` copies `vector_stores` through. A base config copied from `starter-config-configmap.yaml` therefore points `default_provider_id: faiss` at a provider that is gone | High | Medium | Assert the generated list (TC-E2E-001), and use a base config with no `vector_stores` block for the startup case (TC-E2E-002). A dangling `default_provider_id` is not a regression of this fix (Objective: #1) (Objective: #2) |
| The operator sets no `SQLITE_STORE_DIR`, so a `kv_default` `db_path` copied from `starter-config-configmap.yaml` resolves outside the PVC mounted at `/.ogx`, to a path that may be read-only | Medium | Medium | The cluster cases use a base config whose `db_path` values default to `/.ogx` (TC-E2E-001). File the unset `SQLITE_STORE_DIR` separately. The missing setting is a base-config defect (Objective: #2) |

---

## 9. Appendix

### 9.1 Test Case Summary

| Category | Total | P0 | P1 | P2 |
|----------|-------|----|----|-----|
| TC-E2E | 4 | 2 | 2 | 0 |
| TC-NEG | 2 | 1 | 0 | 1 |
| TC-UPG | 1 | 0 | 1 | 0 |
| **Total** | **7** | **3** | **3** | **1** |

---

## End of Test Plan
