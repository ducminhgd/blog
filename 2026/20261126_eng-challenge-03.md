---
tags: Advanced, Database Engineering, Performance Debugging
levels: Senior Engineer, Technical Architect
---
# **Daily Engineering Challenge #03 — The PostgreSQL Query That Suddenly Became 100x Slower**

## **1. Scenario**

You are the Technical Architect of a multi-tenant SaaS ERP platform.

Your backend consists of:

- Python services using PostgreSQL 16.
- Several Go microservices.
- PostgreSQL with shared tables across tenants.
- PgBouncer for connection pooling.
- Kubernetes for application deployment.
- Approximately 2,000 active tenants.

A production incident occurs.

An API endpoint that normally responds in 40–80 ms starts taking 3–8 seconds for certain customers.

The incident affects only some tenants. Other tenants continue operating normally.

No deployment occurred during the incident.

The engineering team suspects PostgreSQL.

## **2. System architecture**

[PgBouncer at Scale: 10K+ Connections Multi-Tenant Postgres](https://images.openai.com/static-rsc-4/0n2iX8zMsEzJBnTKDBm5Z_UpTIkoH-lR552XW7mhT9-s1xvXplwRd-qVgO-6rmcUG69XRKn7g-xLItjA_c0yCe7pGUzoJGHZYoMlSv2RZkoi6B40fwdI-1CAArL-cFwPh_m1AjxLxJtr-MsCCCDro-11IST_hnJOADjxjM3Vg2E?purpose=inline)

[dzone.com](https://dzone.com/articles/database-connection-pooling-at-scale-pgbouncer-mul?utm_source=chatgpt.com)

The affected endpoint retrieves recent invoices with their customer information.

Simplified SQL:

```
SELECT
    i.id,
    i.invoice_number,
    i.status,
    i.created_at,
    c.name AS customer_name,
    SUM(il.quantity * il.unit_price) AS total_amount
FROM invoices i
JOIN customers c
    ON c.id = i.customer_id
LEFT JOIN invoice_lines il
    ON il.invoice_id = i.id
WHERE
    i.tenant_id = $1
    AND i.status = $2
GROUP BY
    i.id, c.id
ORDER BY
    i.created_at DESC
LIMIT 50;
```

Existing indexes:

```
CREATE INDEX idx_invoices_tenant
ON invoices (tenant_id);

CREATE INDEX idx_invoices_tenant_status
ON invoices (tenant_id, status);

CREATE INDEX idx_invoice_lines_invoice
ON invoice_lines (invoice_id);
```

## **3. Production metrics**

BEFORE AND DURING INCIDENT

| Metric               | Normal | Incident |
| -------------------- | ------ | -------- |
| API p50              | 35 ms  | 55 ms    |
| API p95              | 70 ms  | 1,200 ms |
| API p99              | 110 ms | 7,800 ms |
| PostgreSQL CPU       | 35%    | 88%      |
| Database connections | 120    | 450      |
| Cache hit rate       | 92%    | 91%      |
| Database disk IOPS   | 2,000  | 2,200    |
| Error rate           | 0.1%   | 0.3%     |

Additional observations:

- Small tenants experience almost no degradation.
- Three large tenants account for most slow requests.
- The affected endpoint is read-only.
- Other database queries remain relatively healthy.
- Restarting application instances does not resolve the issue.

## **4. Database characteristics**

Your production database has the following characteristics:

| Table         | Approximate rows |
| ------------- | ---------------- |
| invoices      | 180 million      |
| invoice_lines | 900 million      |
| customers     | 25 million       |

Tenant distribution is highly uneven:

| Tenant   | Invoice count |
| -------- | ------------- |
| Tenant A | 2,000         |
| Tenant B | 50,000        |
| Tenant C | 2 million     |
| Tenant D | 25 million    |
| Tenant E | 40 million    |

The database uses shared tables and a `tenant_id` column for logical isolation.

The application uses prepared statements.

PostgreSQL's default query planner statistics target has not been customized for these tables.

## **5. Investigation evidence**

The database team captures two execution plans for the same logical query.

PLAN A — FAST EXECUTION

```
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...
```

Simplified output:

```
Limit
  (actual time=12.4..12.8 rows=50)

  -> Sort
       Sort Method: top-N heapsort

       -> Nested Loop
            -> Index Scan using
               idx_invoices_tenant_status
               (actual rows=50)

            -> Index Scan using
               idx_invoice_lines_invoice
               (actual rows=3)

Execution Time: 13.2 ms
```

PLAN B — SLOW EXECUTION

```
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...
```

Simplified output:

```
Limit
  (actual time=7200..7800 rows=50)

  -> Sort
       Sort Method: external merge
       Disk: 850MB

       -> Hash Join
            -> Seq Scan on invoices
               (actual rows=25000000)

            -> Hash
                 -> Seq Scan on customers

Execution Time: 7810 ms
```

The query text is identical.

The parameter values are different.

## **6. Your investigation challenges**

### **Challenge A — Root cause analysis**

Identify the most plausible reasons for the radically different execution plans.

Investigate these concepts:

- PostgreSQL query planner.
- Custom plans versus generic plans.
- Prepared statement plan caching.
- Parameter sensitivity.
- Cardinality estimation.
- Data distribution and tenant skew.
- Column statistics and extended statistics.

Determine which observations are evidence and which are merely correlations.

### **Challenge B — Index engineering**

Evaluate the current indexes.

Consider:

1. Would a composite index containing `tenant_id`, `status`, and `created_at` improve this query?
2. Should the index use ascending or descending order?
3. Would a covering index with `INCLUDE` columns help?
4. Would a partial index be appropriate?
5. What are the write amplification and storage costs?

Do not assume adding indexes always improves performance.

### **Challenge C — Query planner investigation**

Design a systematic investigation using:

```
EXPLAIN (ANALYZE, BUFFERS, SETTINGS)
```

Consider:

- `pg_stat_statements`
- `pg_prepared_statements`
- `pg_stats`
- `pg_statistic`
- `ANALYZE`
- `auto_explain`

Determine how you would distinguish between:

- Incorrect cardinality estimates.
- Bad join order.
- Inefficient index selection.
- Plan caching problems.
- Memory pressure and disk spills.

### **Challenge D — Multi-tenant architecture**

The system currently uses shared tables.

Evaluate three architectural alternatives:

| Architecture        | Description                          |
| ------------------- | ------------------------------------ |
| Shared tables       | All tenants share the same tables    |
| Schema per tenant   | Each tenant has a separate schema    |
| Database per tenant | Each tenant has an isolated database |

Compare their implications for:

- Query planning.
- Index maintenance.
- Connection management.
- Operational complexity.
- Tenant isolation.
- Large-tenant performance.

Would you introduce a separate database for exceptionally large tenants?

### **Challenge E — Production remediation**

Design a remediation strategy that does not require taking the system offline.

Address:

- Immediate mitigation.
- Query-plan stabilization.
- Statistics maintenance.
- Index deployment without blocking writes.
- PgBouncer prepared statement behavior.
- Monitoring and regression detection.
- Rollback strategy.

## **7. Additional constraints**

Your solution must account for these production requirements:

1. No downtime.
2. No changes to existing API contracts.
3. No data loss.
4. No cross-tenant data leakage.
5. Large tenants must not degrade smaller tenants.
6. Write throughput must remain stable.
7. The system must support continuous schema migrations.

## **8. Engineering considerations**

A senior engineer should distinguish between several easily confused concepts:

- Query execution time versus planning time.
- Index selectivity versus index usefulness.
- Estimated rows versus actual rows.
- Database CPU saturation versus I/O saturation.
- Connection pool exhaustion versus slow query execution.
- Generic plan stability versus parameter-specific optimality.

**The central question:**

Why can identical SQL produce radically different execution times in PostgreSQL, and how do you build a multi-tenant database architecture that remains predictable under extreme data skew?

**Knowledge areas covered**

PostgreSQL internals

Advanced

Query optimization

Advanced

Multi-tenant architecture

Advanced

Production incident response

Advanced

No submission required. The challenge is for independent study. The solution remains undisclosed unless explicitly requested.

---

# **Solution — Daily Engineering Challenge #03**

**PostgreSQL query performance degradation in a multi-tenant SaaS architecture**

Solution

Advanced

PostgreSQL 16

## **1. Root cause analysis**

The most likely root cause is a combination of **parameter-sensitive query planning, tenant data skew, and inaccurate cardinality estimation**.

The key observation is that identical SQL produces radically different execution plans for different parameter values.

PostgreSQL does not necessarily execute prepared statements using a newly optimized plan for every parameter set.

### **1.1 Generic plans versus custom plans**

PostgreSQL supports two relevant prepared statement planning strategies.

| Plan type    | Behavior                                                            |
| ------------ | ------------------------------------------------------------------- |
| Custom plan  | Planner considers actual parameter values                           |
| Generic plan | Planner reuses a plan without considering specific parameter values |

Consider:

```
PREPARE invoice_query(bigint, text) AS
SELECT ...
WHERE tenant_id = $1
AND status = $2;
```

PostgreSQL can initially generate custom plans. After several executions, it can compare the estimated cost of custom plans with a generic plan and choose whether to reuse the generic plan.

This can be problematic when tenant distributions are highly uneven.

For example:

- Tenant A has 2,000 invoices.
- Tenant D has 25 million invoices.

A plan that works well for Tenant A might be extremely inefficient for Tenant D.

The planner might choose an index scan for one execution and a sequential scan for another. Neither strategy is universally optimal.

**Important distinction:** A generic plan is not automatically the cause. A custom plan can also be poor if statistics or cardinality estimates are inaccurate.

### **1.2 Why tenant data skew matters**

Suppose the planner estimates:

```
Estimated rows: 10,000
Actual rows:    25,000,000
```

This is a 2,500x estimation error.

The planner may select a nested loop because it expects a small number of matching invoices.

In reality, the query processes millions of rows.

The resulting execution plan can involve:

- Excessive random index access.
- Large intermediate result sets.
- Expensive joins.
- Large sorting operations.
- Temporary disk usage.

The query then becomes much slower.

### **1.3 Why the execution plan spills to disk**

The slow plan reports:

```
Sort Method: external merge
Disk: 850MB
```

This indicates that the sort operation exceeded its available memory and PostgreSQL used temporary files.

The relevant configuration is:

```
SHOW work_mem;
```

`work_mem` applies per sort or hash operation, not per connection.

A query with several sorts and hash operations can consume multiple allocations.

Increasing `work_mem` globally is dangerous because hundreds of concurrent queries could multiply memory consumption.

### **1.4 What the evidence actually tells us**

| Observation                   | Interpretation                                                  |
| ----------------------------- | --------------------------------------------------------------- |
| Large tenants suffer          | Tenant data skew is relevant                                    |
| Small tenants remain healthy  | Not a uniform infrastructure failure                            |
| CPU rises to 88%              | Database workload is computationally expensive                  |
| Disk IOPS barely change       | Not strong evidence of storage saturation                       |
| Cache hit rate remains stable | Buffer cache is probably not the primary issue                  |
| External merge sort           | Memory insufficient for that sort                               |
| Seq scan over 25M rows        | Potentially inappropriate access path, depending on selectivity |

One additional concern: the supplied execution plans are simplified. The actual plan must be examined for estimated versus actual rows, loops, buffers, and planning time before concluding exactly where the bad decision originated.

## **2. Index engineering**

The current indexes are:

```
CREATE INDEX idx_invoices_tenant
ON invoices (tenant_id);

CREATE INDEX idx_invoices_tenant_status
ON invoices (tenant_id, status);

CREATE INDEX idx_invoice_lines_invoice
ON invoice_lines (invoice_id);
```

The biggest opportunity is to support filtering and ordering together.

### **2.1 Composite index**

Recommended starting point:

```
CREATE INDEX CONCURRENTLY
idx_invoices_tenant_status_created
ON invoices (
    tenant_id,
    status,
    created_at DESC
);
```

Why this order?

PostgreSQL B-tree indexes are most effective when equality predicates constrain the leading columns.

The query has:

```
tenant_id = $1
AND status = $2
ORDER BY created_at DESC
LIMIT 50
```

The index supports:

1. Tenant filtering.
2. Status filtering.
3. Newest-first ordering.
4. Early termination after retrieving 50 matching invoices.

This can avoid scanning and sorting a much larger result set.

Note that the existing index `(tenant_id, status)` is a prefix of the new index and may become redundant for some workloads. Validate other queries before dropping it.

### **2.2 Covering index**

You could consider:

```
CREATE INDEX CONCURRENTLY
idx_invoices_covering
ON invoices (
    tenant_id,
    status,
    created_at DESC
)
INCLUDE (
    id,
    customer_id,
    invoice_number
);
```

This can enable index-only scans when PostgreSQL can satisfy visibility checks through the visibility map.

However, there are trade-offs:

- Larger indexes.
- Higher write amplification.
- More storage.
- More expensive vacuuming and index maintenance.
- Potentially lower cache efficiency.

Also, the invoice-line aggregation still requires accessing `invoice_lines`.

Do not include every selected column indiscriminately.

### **2.3 Partial index**

If the endpoint frequently retrieves only issued invoices:

```
CREATE INDEX CONCURRENTLY
idx_invoices_issued
ON invoices (
    tenant_id,
    created_at DESC
)
WHERE status = 'issued';
```

This can be smaller than a full index.

However, PostgreSQL must be able to prove that the query predicate implies the partial-index predicate.

Parameterized queries using a generic plan can complicate that determination because the planner does not know the actual status parameter.

This is another reason to understand prepared statement behavior before relying on partial indexes.

### **2.4 Index design trade-offs**

| Index strategy   | Benefit                        | Cost                       |
| ---------------- | ------------------------------ | -------------------------- |
| Composite B-tree | Filtering and ordering         | Storage and write overhead |
| Covering index   | Potential index-only scans     | Larger index               |
| Partial index    | Smaller, specialized index     | Predicate restrictions     |
| Separate indexes | Flexible for different queries | May require bitmap scans   |

**Recommendation:** Start with the composite index. Measure its impact. Add covering or partial indexes only when execution plans justify them.

## **3. Query planner investigation**

The objective is to identify why PostgreSQL selected the wrong execution plan, not simply force it to use an index.

### **3.1 Inspect actual execution plans**

Start with:

```
EXPLAIN (
    ANALYZE,
    BUFFERS,
    SETTINGS,
    VERBOSE,
    SUMMARY
)
SELECT ...;
```

Focus on these fields:

| Field                  | What to investigate                |
| ---------------------- | ---------------------------------- |
| `cost`                 | Planner's estimated execution cost |
| `rows`                 | Estimated output rows              |
| `actual rows`          | Actual output rows                 |
| `loops`                | Number of node executions          |
| `Buffers: shared hit`  | Data served from shared buffers    |
| `Buffers: shared read` | Data read into shared buffers      |
| `temp read/write`      | Temporary file activity            |
| `Planning Time`        | Time spent generating the plan     |
| `Execution Time`       | Actual execution duration          |

The most important diagnostic is:

**Estimated rows versus actual rows.**

For example:

```
Index Scan
  (cost=0.43..500 rows=100)
  (actual time=0.5..2500 rows=25000000)
```

This indicates a severe cardinality estimation problem.

### **3.2 Investigate prepared statements**

Inspect prepared statements:

```
SELECT
    name,
    statement,
    parameter_types,
    generic_plans,
    custom_plans
FROM pg_prepared_statements;
```

The `generic_plans` and `custom_plans` counters help establish whether PostgreSQL is using generic or custom plans.

Test both strategies:

```
SET plan_cache_mode = force_custom_plan;

EXPLAIN (ANALYZE, BUFFERS)
EXECUTE invoice_query(123, 'issued');
```

Then:

```
SET plan_cache_mode = force_generic_plan;

EXPLAIN (ANALYZE, BUFFERS)
EXECUTE invoice_query(123, 'issued');
```

Compare execution plans for:

- Small tenants.
- Medium tenants.
- Very large tenants.

Do not leave `force_custom_plan` enabled globally without workload testing.

It can increase planning overhead substantially.

### **3.3 Investigate statistics**

Inspect column statistics:

```
SELECT
    attname,
    n_distinct,
    most_common_vals,
    most_common_freqs,
    histogram_bounds
FROM pg_stats
WHERE tablename = 'invoices';
```

Potential issue:

PostgreSQL may understand the distribution of `tenant_id` and `status` independently but fail to understand their correlation.

For example, certain tenants might have overwhelmingly more issued invoices than others.

Create extended statistics:

```
CREATE STATISTICS invoices_tenant_status_stats
    (mcv, dependencies)
ON tenant_id, status
FROM invoices;
```

Then:

```
ANALYZE invoices;
```

This provides additional information about correlated column distributions.

You can also increase the statistics target:

```
ALTER TABLE invoices
ALTER COLUMN tenant_id
SET STATISTICS 1000;

ANALYZE invoices;
```

A higher target can improve estimates at the cost of additional analysis work and statistics storage.

It does not guarantee accurate estimates for every highly skewed distribution.

### **3.4 Instrumentation**

I would enable the following monitoring capabilities:

| Tool                   | Purpose                          |
| ---------------------- | -------------------------------- |
| `pg_stat_statements`   | Query-level execution statistics |
| `auto_explain`         | Capture slow execution plans     |
| `pg_stat_activity`     | Active queries and wait events   |
| `pg_stat_database`     | Database-level statistics        |
| `pg_stat_user_tables`  | Table scans and tuple activity   |
| `pg_stat_user_indexes` | Index usage statistics           |

Example:

```
SELECT
    queryid,
    calls,
    mean_exec_time,
    max_exec_time,
    total_exec_time,
    rows,
    shared_blks_hit,
    shared_blks_read,
    temp_blks_written
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 20;
```

For production, configure `auto_explain` carefully:

```
LOAD 'auto_explain';

SET auto_explain.log_min_duration = '1s';
SET auto_explain.log_analyze = on;
SET auto_explain.log_buffers = on;
SET auto_explain.log_timing = off;
```

`log_analyze` adds overhead. Enable it selectively and evaluate the impact.

### **3.5 Diagnostic decision matrix**

| Finding                         | Likely problem                      | Investigation                 |
| ------------------------------- | ----------------------------------- | ----------------------------- |
| Estimated rows far below actual | Bad statistics or estimation        | `pg_stats`, `ANALYZE`         |
| Generic plan consistently poor  | Parameter sensitivity               | Compare custom and generic    |
| Large temp writes               | Sort/hash memory pressure           | `work_mem`, plan structure    |
| High nested-loop iterations     | Poor join strategy or estimates     | Join cardinality              |
| High shared reads               | Buffer misses / working set         | Buffer and I/O metrics        |
| High planning time              | Complex query or excessive planning | Planning metrics              |
| Many waiting connections        | Pooling or contention               | `pg_stat_activity`, PgBouncer |

## **4. Multi-tenant architecture**

The existing shared-table architecture is not inherently wrong.

The problem is that tenant workloads have dramatically different sizes and characteristics.

This is commonly called the *noisy neighbor problem*.

### **4.1 Architecture comparison**

**Option A — Shared tables**

[Kubernetes Resource Units Explained - Ajit Singh](https://images.openai.com/static-rsc-4/WCjtI8EgBltvV-KX6ZtFoG_1_8lp6_hav1YoBC14tczGWFMhUc-AWb9psXWRsLOOmoC-JrnyJbLIXAzkegLUIy9qP0LuyoLEv3ii0u65XgvWV4QW9c3B23oW7D5hKJw73sMwqz5Lkzcr73HK_q3zQEwulxinCInOSJeS6bS4rbA?purpose=inline)

[singhajit.com](https://singhajit.com/explainer/kubernetes-resource-units/?utm_source=chatgpt.com)

All tenants share the same tables. Every row contains `tenant_id`.

Advantages:

- Lowest infrastructure overhead.
- Efficient resource utilization.
- Simple centralized migrations.
- Easy cross-tenant operational reporting.

Disadvantages:

- Data skew affects query planning.
- Noisy neighbor risks.
- Complex tenant-specific optimization.
- Indexes serve multiple workloads.

  **Option B — Schema per tenant**

[Multi-Tenant SaaS with Spring Boot and PostgreSQL](https://images.openai.com/static-rsc-4/dlkKmFiEf57GVqUXa0xw70YGnbk1-lEUERqHBCKxz9S5Y6zQWOgtcYYsd4W1upleJioLpsZLQ_3AAO9UYCL1MIiMRC-4uPLo_3IPLvURF-NvzHi9Bjgtu2868p8KbeD5QlaGq9wxDqLLb0z7mwxgAlP7bAthE31gR_6VcFx-LK4?purpose=inline)

[blog.techxplore.io](https://blog.techxplore.io/multi-tenant-architecture-patterns-trade-offs-and-best-practices?utm_source=chatgpt.com)

Each tenant gets an isolated schema inside a shared database.

Advantages:

- Independent indexes and table structures.
- Easier tenant-specific schema customization.
- Logical organization.

Disadvantages:

- Thousands of schemas create operational complexity.
- Schema migrations become expensive.
- Connection search-path management.
- Shared database resources remain shared.

  **Option C — Database per tenant**

[Implementing Multitenancy with NestJS, TypeORM, and PostgreSQL | Renan Damé de Almeida posted on the topic | LinkedIn](https://images.openai.com/static-rsc-4/XPbbNUefjPGzvq6TmLizmyaXY8Gjm3CswUhr9aOuIiRHotpJ-k-k7F8H9-XvM5gzFToleXfosqTSFN_V7B-wjiVMab-2MLeGGHCFUKBXzDU0Ahv770mNRAC-HNf2RT3RAhZVuTRu97rrIE-g7p8WJ55hhTqU10kxWH_gE7oCTIE?purpose=inline)

[linkedin.com](https://www.linkedin.com/posts/renan-dame-de-almeida_construindo-uma-api-multitenant-com-nestjs-activity-7425518142736666624-ZuE7?utm_source=chatgpt.com)

Each tenant receives an independent database.

Advantages:

- Stronger resource and operational isolation.
- Independent scaling.
- Independent maintenance windows.
- Easier tenant-specific tuning.

Disadvantages:

- Higher infrastructure cost.
- More connection management.
- More complicated migrations.
- Backup, monitoring and disaster recovery complexity.

### **4.2 My recommendation: Hybrid multi-tenancy**

I would not migrate all 2,000 tenants to separate databases.

I would implement a tiered architecture.

## **Hybrid tenant architecture**

**Tier 1 — Standard tenants**

Shared PostgreSQL cluster · Shared tables · tenant_id isolation

Majority of tenants

**Tier 2 — Large tenants**

Dedicated database or dedicated cluster, depending on resource requirements

High-volume customers

**Tier 3 — Enterprise tenants**

Dedicated cluster, customized capacity, isolation and operational policies

Enterprise isolation

Introduce a tenant routing layer:

```
def resolve_database(tenant_id):    tenant = tenant_registry.get(tenant_id)    return database_pool.get(        tenant.database_location    )
```

The tenant registry stores the database location and deployment tier.

This allows tenant migration without changing application APIs.

Important implementation requirements:

- Enforce tenant isolation at the database and application layers.
- Consider PostgreSQL Row-Level Security as defense in depth.
- Keep routing metadata authoritative.
- Avoid allowing clients to choose arbitrary database identifiers.
- Support migration between database locations.
- Maintain consistent schema versions across tiers.

### **4.3 Should large tenants receive dedicated databases?**

Yes, when their resource consumption or isolation requirements justify the cost.

Useful migration triggers include:

- Sustained CPU consumption.
- High database I/O consumption.
- Excessive table and index growth.
- Large vacuum workloads.
- Significant impact on other tenants.
- Dedicated backup or recovery requirements.

Do not use invoice count alone. Workload characteristics matter more than raw row counts.

## **5. Production remediation**

I would divide remediation into three phases.

### **Phase 1 — Immediate mitigation**

Incident response

1. Identify affected tenants and parameter combinations.
2. Capture execution plans for representative tenants.
3. Check whether generic plans are responsible.
4. Test custom plans in controlled sessions.
5. Temporarily route the affected workload to an isolated connection pool if connection contention is contributing.
6. Reduce unnecessary concurrent execution of the expensive query.

If generic plans are confirmed as the cause, test:

```
SET LOCAL plan_cache_mode = force_custom_plan;
```

This can be applied to a dedicated transaction or workload-specific database connection.

Do not apply it indiscriminately to every query.

### **Phase 2 — Structural optimization**

Performance engineering

#### **A. Add the composite index**

```
CREATE INDEX CONCURRENTLY
idx_invoices_tenant_status_created
ON invoices (
    tenant_id,
    status,
    created_at DESC
);
```

`CONCURRENTLY` avoids blocking ordinary writes for the duration of index construction, although it introduces additional I/O and has operational caveats.

Monitor index creation and validate it afterward.

#### **B. Refresh statistics**

```
ANALYZE invoices;
ANALYZE invoice_lines;
ANALYZE customers;
```

Create extended statistics where evidence supports them.

#### **C. Rewrite the query**

This is an important optimization opportunity.

The original query joins invoices to invoice lines, aggregates all matching rows, and sorts the grouped results.

But the endpoint only needs the latest 50 invoices.

You can retrieve those invoices first, then aggregate their invoice lines.

```
WITH recent_invoices AS (
    SELECT
        id,
        customer_id,
        invoice_number,
        status,
        created_at
    FROM invoices
    WHERE tenant_id = $1
      AND status = $2
    ORDER BY created_at DESC, id DESC
    LIMIT 50
)
SELECT
    i.id,
    i.invoice_number,
    i.status,
    i.created_at,
    c.name AS customer_name,
    COALESCE(
        SUM(il.quantity * il.unit_price),
        0
    ) AS total_amount
FROM recent_invoices i
JOIN customers c
    ON c.id = i.customer_id
LEFT JOIN invoice_lines il
    ON il.invoice_id = i.id
GROUP BY
    i.id,
    i.customer_id,
    i.invoice_number,
    i.status,
    i.created_at,
    c.name
ORDER BY
    i.created_at DESC,
    i.id DESC;
```

Why is this better?

The original query can process invoice lines for a large population of invoices before returning 50 groups.

The rewritten query limits the invoice population before joining invoice lines.

Instead of aggregating millions of invoices, it aggregates at most 50 invoices.

This transformation is valid because the ordering and filtering depend only on invoice columns, not on the aggregated invoice-line values. If business requirements change to filter or order by aggregate values, the transformation needs to be reconsidered.

The `id DESC` tie-breaker also makes pagination deterministic when invoices have identical timestamps.

#### **D. Consider precomputed aggregates**

If invoice totals are read frequently and invoice lines rarely change after posting, maintain a materialized invoice total:

```
ALTER TABLE invoices
ADD COLUMN total_amount NUMERIC(18, 2);
```

Update it transactionally when invoice lines change, or maintain it through a reliable event-driven process.

Trade-offs:

- Faster reads.
- More complex writes.
- Potential consistency problems with asynchronous updates.
- Backfill and reconciliation requirements.

For financial data, avoid an eventually consistent total unless the API clearly defines its consistency guarantees.

### **Phase 3 — Long-term optimization**

- Introduce tenant workload classification.
- Isolate high-volume tenants.
- Consider time-based partitioning if retention and query patterns justify it.
- Implement query performance SLOs.
- Establish automatic slow-query detection.
- Introduce regression testing with production-like tenant distributions.

## **6. PgBouncer and connection pooling**

This is an important part of the investigation because the application uses PgBouncer.

### **6.1 Transaction pooling considerations**

PgBouncer supports three pooling modes:

| Mode        | Connection behavior                                   |
| ----------- | ----------------------------------------------------- |
| Session     | Backend connection remains assigned to client session |
| Transaction | Backend connection assigned only during transaction   |
| Statement   | Backend connection assigned per statement             |

Transaction pooling is generally suitable for stateless backend services.

However, traditional session-level prepared statements introduce compatibility issues with transaction pooling.

Modern PgBouncer versions support protocol-level named prepared statements when `max_prepared_statements` is configured appropriately.

You must distinguish:

- SQL-level `PREPARE` statements.
- Extended-protocol prepared statements.
- PgBouncer's prepared statement tracking and rewriting.

Do not assume all prepared statement behavior is incompatible with transaction pooling.

### **6.2 What I would inspect**

```
[pgbouncer]

pool_mode = transaction

max_prepared_statements = 200

default_pool_size = 30
reserve_pool_size = 10
```

These are illustrative values, not recommended production defaults.

Inspect:

- PgBouncer version.
- Prepared statement configuration.
- Application driver behavior.
- Connection reuse.
- Pool saturation.
- Transaction duration.
- Server-side connection counts.

### **6.3 Connection exhaustion**

The incident shows:

```
Normal connections:   120
Incident connections: 450
```

This deserves investigation.

However, more connections do not necessarily mean more throughput.

Excessive database concurrency can cause:

- CPU context switching.
- Lock contention.
- Memory pressure.
- More concurrent sorts and hash operations.
- Increased tail latency.

I would establish a database concurrency budget and distribute it across application pools.

Use PgBouncer metrics:

```
SHOW POOLS;
SHOW STATS;
SHOW CLIENTS;
SHOW SERVERS;
```

Do not solve a slow-query problem simply by increasing the connection pool.

## **7. Partitioning strategy**

Partitioning deserves consideration, especially for the 25-million-invoice tenant.

However, partitioning is not an automatic solution to bad query plans.

### **Option A: Hash partitioning by tenant**

```
CREATE TABLE invoices (
    id BIGINT NOT NULL,
    tenant_id BIGINT NOT NULL,
    status TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (tenant_id, id)
)
PARTITION BY HASH (tenant_id);
```

You could create 32 or 64 partitions.

Advantages:

- Tenant filtering can enable partition pruning.
- Smaller indexes per partition.
- Better maintenance isolation at partition level.

Disadvantages:

- A single large tenant remains concentrated in one partition.
- More complicated global index management.
- Partitioning does not automatically solve parameter sensitivity.

### **Option B: Range partitioning by time**

```
PARTITION BY RANGE (created_at);
```

Advantages:

- Efficient time-based queries.
- Easier retention management.
- Partition pruning for date-filtered workloads.

Disadvantages:

- Queries without date filters may scan many partitions.
- Every partition needs maintenance.
- Unique constraints have partition-key implications.

### **My recommendation**

Use time partitioning when data retention and time-based access patterns justify it.

Use tenant-based isolation for exceptionally large tenants.

Do not create thousands of partitions simply because you have thousands of tenants.

## **8. Production monitoring and regression prevention**

I would establish the following metrics.

| Category    | Metric                      | Purpose                  |
| ----------- | --------------------------- | ------------------------ |
| Latency     | Query p50/p95/p99           | Detect regressions       |
| Query plans | Estimated/actual rows ratio | Detect estimation errors |
| Database    | CPU utilization             | Identify CPU saturation  |
| Database    | Temp bytes written          | Detect spills            |
| Database    | Buffer reads                | Detect cache pressure    |
| Connections | Active/waiting connections  | Detect pool pressure     |
| Tenants     | Query latency per tenant    | Detect noisy neighbors   |
| Storage     | Table/index growth          | Capacity planning        |
| Operations  | Vacuum/analyze duration     | Maintenance health       |

### **Define SLOs**

For example:

**Invoice listing API**

p95 latency

**Under 200 ms**

p99 latency

**Under 500 ms**

Availability

**99.9%**

Error rate

**Under 0.1%**

Illustrative targets. Establish actual SLOs using workload measurements and customer requirements.

Crucially, monitor latency by tenant size and workload class, not just globally.

A global p95 can conceal severe degradation affecting a small number of enterprise customers.

## **9. Rollout and rollback strategy**

I would deploy the remediation in this order:

| Step | Action                           | Risk control                  |
| ---- | -------------------------------- | ----------------------------- |
| 1    | Capture baseline execution plans | Preserve evidence             |
| 2    | Add composite index concurrently | Monitor I/O and write latency |
| 3    | Update statistics                | Compare execution plans       |
| 4    | Deploy query rewrite             | Canary deployment             |
| 5    | Tune prepared statement behavior | Workload-specific testing     |
| 6    | Introduce tenant isolation       | Gradual tenant migration      |
| 7    | Validate SLOs                    | Compare before/after          |

For every step, retain the ability to roll back independently.

For example:

- Query rewrite: revert application release.
- Index: drop concurrently if necessary.
- Planner setting: restore previous setting.
- Tenant migration: route the tenant back after validating replication or data synchronization.

Never drop the old index immediately after creating a replacement. First establish whether other queries still depend on it.

# **10. Final architectural recommendation**

If I were accountable for this production system, I would prioritize the following:

## **Priority 1 — Fix the immediate performance problem**

1. Identify generic versus custom plan behavior.
2. Compare actual and estimated cardinalities.
3. Create the composite index.
4. Rewrite the query to limit invoices before aggregation.
5. Validate with production-like data distributions.

Highest priority

## **Priority 2 — Improve database predictability**

1. Improve column and extended statistics.
2. Monitor prepared statements.
3. Establish database concurrency limits.
4. Track tenant-specific query performance.
5. Introduce query-plan regression tests.

Medium-term

## **Priority 3 — Improve multi-tenant scalability**

1. Classify tenants by resource consumption.
2. Keep standard tenants on shared infrastructure.
3. Move high-volume tenants to dedicated databases where justified.
4. Introduce tenant-aware database routing.
5. Standardize migrations and operational procedures across database tiers.

Long-term

## **11. The most important engineering lessons**

There are five takeaways from this incident.

**1. Identical SQL does not imply identical execution plans.**

Parameter values, statistics, prepared statement planning, and data distribution all influence query planning.

**2. Indexes cannot compensate for every architectural problem.**

A query that aggregates millions of records before applying a limit may need a structural rewrite, not another index.

**3. Average database performance is misleading in multi-tenant systems.**

Monitor performance by tenant and workload class.

**4. Connection pooling is a concurrency management mechanism.**

More connections do not automatically produce more throughput.

**5. Multi-tenancy requires resource isolation, not just data isolation.**

`tenant_id` prevents logical data mixing when correctly enforced. It does not prevent one tenant from consuming disproportionate database resources.

## **12. Further reading — Official documentation**

These are the primary references I would use to deepen the investigation.

1. [PostgreSQL — PREPARE](https://www.postgresql.org/docs/16/sql-prepare.html?utm_source=chatgpt.com): Generic and custom plans.
2. [PostgreSQL — Query Planning Configuration](https://www.postgresql.org/docs/16/runtime-config-query.html?utm_source=chatgpt.com): `plan_cache_mode`, planner behavior.
3. [PostgreSQL — Planner Statistics](https://www.postgresql.org/docs/16/planner-stats.html?utm_source=chatgpt.com): Statistics and extended statistics.
4. [PostgreSQL — Multicolumn Indexes](https://www.postgresql.org/docs/16/indexes-multicolumn.html?utm_source=chatgpt.com).
5. [PostgreSQL — Using EXPLAIN](https://www.postgresql.org/docs/16/using-explain.html?utm_source=chatgpt.com).
6. [PgBouncer — Configuration](https://www.pgbouncer.org/config.html?utm_source=chatgpt.com): Pooling and prepared statements.

**My overall conclusion:** The best solution is not simply to force PostgreSQL to use an index. It is to make the query workload predictable, improve cardinality estimation, reduce unnecessary aggregation, and isolate disproportionately expensive tenants.
