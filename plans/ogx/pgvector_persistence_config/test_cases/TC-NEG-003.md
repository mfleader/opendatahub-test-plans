---
test_case_id: TC-NEG-003
source_key: RHAIENG-6374
objectives: [1]
priority: P2
status: Draft
automation_status: Not Started
last_updated: "2026-09-21"
upgrade_phase: post
---
# TC-NEG-003: Base ConfigMap without the watch label fails config generation

**Objective**: When the base config ConfigMap lacks the label `ogx.io/watch: "true"`, the operator
cannot read it through its label-filtered cache (`newCacheOptions` in `main.go`) and reports
`ConfigGenerationFailed`, so no generated ConfigMap and no persistence block are produced.

**Preconditions**:

- Environment per Section 3.1 of the plan
- ConfigMap `ogx-base-config-unlabeled` (key `config.yaml`) exists in `ogx-pgvector-test` with
  the `config/samples/starter-config-configmap.yaml` content and **without** the `ogx.io/watch`
  label
- Secret `postgres-secret` (key `password`) exists in `ogx-pgvector-test` with the label
  `ogx.io/watch: "true"`

**Test Steps**:

1. Read `metadata.labels` on ConfigMap `ogx-base-config-unlabeled`
2. Apply the `OGXServer` CR from Test Data
3. Wait 120s for condition `ConfigGenerated=false` on `ogxserver/ogxserver-unlabeled-base`
4. Read the `reason` and `message` of the `ConfigGenerated` condition
5. Read `status.configGeneration.configMapName`, and count ConfigMaps labelled
   `ogx.io/generated-config` whose name begins `ogxserver-unlabeled-base-config-`
6. Label the same ConfigMap `ogx.io/watch=true`, wait up to 180s for `ConfigGenerated=true`, then
   print the `remote::pgvector` entry's `config.persistence` from the generated ConfigMap

**Expected Results**:

- Step 1 prints no `ogx.io/watch` key
- Step 4 prints reason `ConfigGenerationFailed` and the message
  `failed to reconcile base ConfigMap: failed to find referenced base ConfigMap`
  followed by the namespace and name
- Step 5 prints an empty ConfigMap name and a count of `0`
- Step 6 prints `backend: kv_default` and `namespace: vector_io::pgvector`, confirming the
  missing label alone caused the failure

**Test Data**:

The TC-E2E-001 CR with `metadata.name: ogxserver-unlabeled-base` and
`spec.baseConfig.name: ogx-base-config-unlabeled`.
