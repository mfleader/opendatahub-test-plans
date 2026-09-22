---
feature: pgvector_persistence_config
source_key: RHAIENG-6374
status: Open
gap_count: 2
last_updated: '2026-09-21'
---
# Gaps: pgvector_persistence_config

Two known defects in the pgvector path are out of scope for this plan, per TestPlan.md Section 1.2.

- The pgvector `vector_index` shape mismatch with the OGX schema
- A base config that defines `storage.backends` without `kv_default`, leaving the emitted
  reference unresolved
