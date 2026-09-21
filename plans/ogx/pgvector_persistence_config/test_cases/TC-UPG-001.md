---
test_case_id: TC-UPG-001
source_key: RHAIENG-6374
objectives: [1, 5]
priority: P1
status: Draft
automation_status: Not Started
last_updated: "2026-09-21"
upgrade_phase: post
---
# TC-UPG-001: Changed generated content produces a new ConfigMap the pod mounts

**Objective**: With the fixed operator, changing a field that alters generated content makes the
operator write a second ConfigMap under a new content hash, keep the old one, and mount the new
one on the server pod. This is the regeneration path an operator upgrade relies on.

**Preconditions**:

- Environment per Section 3.1, and the Section 3.5 PostgreSQL fixture as in TC-E2E-002
- The TC-E2E-001 CR applied and `Ready`

**Test Steps**:

1. Record `status.configGeneration.configMapName` as `CM_OLD`, and the ConfigMap name mounted on
   the server pod
2. Edit the CR to change a field that alters generated content, such as `db`. A connection field
   must still point at something real, so create the target database and its `vector` extension
   first, or the server fails with `InvalidCatalogNameError` for a reason unrelated to this fix
3. Wait 120s for `status.configGeneration.generatedAt` to advance
4. Record the new `configMapName` as `CM_NEW`, and its pgvector `config.persistence`
5. Confirm `$CM_OLD` still exists
6. Wait 300s for the pod to reach `Ready`, then read the ConfigMap name it mounts
7. Count the crash signature in the pod logs

**Expected Results**:

- Step 4 yields a `CM_NEW` differing from `CM_OLD`, since the name carries a content hash, and
  `persistence` with `backend: kv_default` and `namespace: vector_io::pgvector`
- Step 5 still lists `$CM_OLD`. The old ConfigMap is **not** deleted, because
  `configMapRetention = 2` keeps both after one regeneration
- Step 6 shows the pod mounting `$CM_NEW`
- Step 7 prints `0`
