# pgvector Persistence Config

Test plan for the fix to [RHAIENG-6374](https://redhat.atlassian.net/browse/RHAIENG-6374): the OGX
operator's config generation drops the `persistence` block from the pgvector `vector_io` provider
config when a declarative v1beta1 `OGXServer` CR uses `providers.vectorIo.remote.pgvector`, which
crashes the OGX server pod on startup with
`AttributeError: 'NoneType' object has no attribute 'backend'`.

- **Strategy**: [RHAIENG-6374](https://redhat.atlassian.net/browse/RHAIENG-6374)
- **ADR**: none provided
- **Test Plan**: [TestPlan.md](TestPlan.md)
- **Test Cases**: [test_cases/INDEX.md](test_cases/INDEX.md). 7 test cases (P0: 3, P1: 3, P2: 1)
- **Automated tests destination**: `opendatahub-tests` (OGX component test directory)
