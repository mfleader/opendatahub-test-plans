---
test_case_id: TC-NEG-001
source_key: RHAIENG-6374
objectives: [2]
priority: P0
status: Draft
automation_status: Not Started
last_updated: "2026-09-21"
upgrade_phase: pre
---
# TC-NEG-001: Unfixed operator reproduces the pgvector persistence crash

**Objective**:

The environment reproduces RHAIENG-6374 with an operator lacking the fix, so a pass in
TC-E2E-002 shows the fix works rather than that the environment cannot fail. Run it before
TC-E2E-002, in its own clean cluster.

**Falsifiability**: this case is itself the control. If the crash does not appear, the
environment cannot reproduce the bug and TC-E2E-002 proves nothing.
**Preconditions**:

- Environment per Section 3.1, except the operator image is built from `main` at `be51587`,
  the fix's parent, without the fix, on a cluster that has never run the fixed operator
- ConfigMap, Secret and the Section 3.5 PostgreSQL fixture as in TC-E2E-002

**Test Steps**:

1. Read the `manager` container image on
   `deployment/ogx-k8s-operator-controller-manager` in `ogx-k8s-operator-system`
2. Apply the `OGXServer` CR from TC-E2E-001 Test Data
3. Wait 120s for `ConfigGenerated=true`, then print the `remote::pgvector` entry from the
   generated ConfigMap
4. Wait 300s for the server pod to reach `Ready`, expecting the wait to time out. The timeout
   matches TC-E2E-002 step 4, so a pod that is merely slow cannot pass as a reproduction
5. Record that pod's phase and `containerStatuses[?(@.name=="ogx")].restartCount`
6. Count `AttributeError: 'NoneType' object has no attribute 'backend'` in the previous
   container's logs, read with `--previous --tail=-1`
7. List events with `reason=BackOff`

**Expected Results**:

- Step 1 prints the unfixed image reference
- Step 3 prints a pgvector entry with the connection fields and no `persistence` key
- Step 4 times out, so the pod never became ready within the same window TC-E2E-002 allows
- Step 5 shows a restart count above `0`
- Step 6 prints `1` or more
- Step 7 lists at least one `BackOff` event
- If any result is absent, the environment does not reproduce the bug and a pass in TC-E2E-002
  proves nothing. Stop and repair the environment
