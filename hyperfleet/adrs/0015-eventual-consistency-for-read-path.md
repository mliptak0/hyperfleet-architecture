---
Status: Active
Owner: HyperFleet Team
Last Updated: 2026-05-08
---

# 0015 — Eventual Consistency for the API Read Path

## Context

The HyperFleet API LIST endpoints execute COUNT and SELECT as two independent queries without a shared database transaction. Under concurrent modifications (inserts or deletes between the two queries), the reported `total` may not match the actual number of items returned, and pagination may skip or duplicate records.

The transaction middleware skips transaction creation for read operations (GET requests). Sentinels, adapters, and observability collectors poll the API continuously, resulting in high read traffic volume.

HYPERFLEET-870 investigated whether this inconsistency should be fixed and at what cost.

## Decision

Accept eventual consistency on the read path. LIST queries are not wrapped in a transaction and do not use window functions to guarantee COUNT/SELECT consistency within a single request.

## Consequences

**Gains:** No additional latency on LIST endpoints. No additional database connection pool pressure from read transactions.

**Trade-offs:** Under concurrent modifications, clients may observe inaccurate total counts, skipped records, or duplicate records across pages.

## Alternatives Considered

| Alternative | Why Rejected |
|-------------|--------------|
| Window function (`COUNT(*) OVER()`) | 32% p99 latency regression, 48% throughput reduction. Postgres scans all matching rows for the window count regardless of page size. |
| Read-only transaction (`BEGIN READ ONLY` wrapping COUNT + SELECT) | 12% p99 latency regression, 22% throughput reduction. Adds overhead to every LIST call. |
| Cursor-based pagination | Eliminates COUNT entirely. Requires a breaking API contract change (removes `total` field, removes direct page access). |

## Benchmark Data

Local benchmarks: 10,000 clusters, 1,000 requests, 10 concurrency, page size 10, 15 runs per approach.

| Approach | p99 Latency | Throughput |
|----------|-------------|------------|
| No transaction (current) | 31.4ms | 789 req/sec |
| Window function | 41.5ms (+32%) | 409 req/sec (-48%) |
| Read-only transaction | 35.1ms (+12%) | 618 req/sec (-22%) |
