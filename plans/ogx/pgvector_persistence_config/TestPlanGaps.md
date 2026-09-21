---
feature: pgvector_persistence_config
source_key: RHAIENG-6374
status: Open
gap_count: 4
last_updated: '2026-09-21'
---
# Gaps: pgvector_persistence_config

Four follow-ups are named in TestPlan.md Section 1.2 as out of scope and none has a Jira key yet.

- Milvus and Qdrant persistence, and the missing Milvus token key
- The pgvector `vector_index` shape mismatch with the OGX schema
- Residual `metadata_store` cases in praxis mode
- More than one declarative pgvector provider on one OGXServer, which would share vector store
  records because the namespace is hardcoded
