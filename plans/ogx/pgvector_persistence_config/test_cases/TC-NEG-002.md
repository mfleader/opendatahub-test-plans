---
test_case_id: TC-NEG-002
source_key: RHAIENG-6374
objectives: [1]
priority: P2
status: Draft
automation_status: Not Started
last_updated: "2026-09-21"
upgrade_phase: post
---
# TC-NEG-002: Base config without storage.backends still emits persistence and the pod starts

**Objective**: With a base config that defines no `storage.backends`, the operator still emits
`persistence.backend: kv_default` and the server pod starts, because the server defaults the
referenced backend rather than failing on it.

**Preconditions**:

- Environment per Section 3.1 of the plan
- ConfigMap `ogx-minimal-base-config` (key `config.yaml`) exists in `ogx-pgvector-test` with
  the label `ogx.io/watch: "true"` and the minimal content from Test Data (no `storage` key)
- Secret `postgres-secret` and the PostgreSQL fixture from Section 3.5 of the test plan, as in
  TC-E2E-002

**Test Steps**:

The server pod carries the label
`app.kubernetes.io/instance=ogxserver-minimal-base`.

1. Apply the `OGXServer` CR from Test Data
2. Wait 120s for `ConfigGenerated=true`
3. From the generated `config.yaml`, print the top-level `storage` key and the
   `remote::pgvector` entry's `config.persistence`
4. Wait 300s for the server pod to reach `Ready`, then record its phase and
   `containerStatuses[0].restartCount`
5. Count `AttributeError: 'NoneType' object has no attribute 'backend'` in the pod logs
6. Only if the pod is not `Running`, capture the first line of its logs matching `Error` or
   `error`

**Expected Results**:

- Step 2 reports reason `ConfigGenerationSucceeded`
- Step 3 prints `persistence` with `backend: kv_default` and `namespace: vector_io::pgvector`,
  and `storage: null`, since the operator emits no storage section when neither `spec.storage`
  nor the base config defines one (`buildStorageSection`)
- Step 4 shows `Running` with restart count `0`, since the server manufactures the backend
  through `_default_backends()` in the OGX server, not the operator
- Step 5 prints `0`
- Step 6 should not execute. If the pod fails, the anticipated cause is the unwritable default
  sqlite path (Section 8, risk 2). File it separately, it is not RHAIENG-6374

**Test Data**:

```yaml
# ogx-minimal-base-config, key config.yaml. No storage section, which is what this
# case exercises. The inference provider is required, per Section 3.1.
version: '2'
distro_name: rh
apis:
- inference
- vector_io
providers:
  inference:
  - provider_id: sentence-transformers
    provider_type: inline::sentence-transformers
    config:
      trust_remote_code: false
```

The CR is the TC-E2E-001 CR with `metadata.name: ogxserver-minimal-base` and
`spec.baseConfig.name: ogx-minimal-base-config`.
