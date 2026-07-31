---
title: "Beyond Lift and Shift: Why Cloud Migration Demands Database Code Optimization"
date: "2026-07-31"
summary: "Moving Oracle databases from physical servers to cloud ECS with EVS storage is not just an infrastructure change. Without optimizing SQL and database-interaction code, cloud deployments will hit storage I/O bottlenecks that didn't exist on bare metal. Here's why — and how much effort the fix really takes."
tags: ["Cloud", "Database", "Oracle", "Migration", "Optimization"]
slug: "beyond-lift-and-shift-database-code-optimization"
lang: "en"
---

# Beyond Lift and Shift: Why Cloud Migration Demands Database Code Optimization

> Author: seagaruda | Published: 2026-07-31

---

## The Hidden Trap in Cloud Migration

A familiar story: a company runs its Oracle database on a physical server. The database disks are directly attached — SAS drives in a RAID array, or local NVMe SSDs. The application has worked fine for years. Full-table scans on multi-million-row tables, complex joins without proper indexes, dynamic SQL assembled at runtime — the hardware absorbed it all.

Then the company decides to "go cloud." They provision an ECS (Elastic Cloud Server) on Huawei Cloud or an EC2 instance on AWS, attach an EVS (Elastic Volume Service) or EBS (Elastic Block Store) disk, copy the database over, start the application, and declare migration complete.

The result? Performance collapses. Reports that ran in 30 seconds now take 5 minutes. Batch jobs that finished overnight are still running at noon. Users complain. Management questions the cloud decision. The team scrambles to upgrade the EVS disk to a more expensive tier — and still the performance is worse than the old physical server.

**The problem is not the cloud. The problem is that the code was never optimized for how cloud storage actually works.**

---

## Why Cloud Storage Performs Differently

### Physical Server: Direct-Attached Storage

On a physical server, the Oracle database runs on disks that are directly connected to the motherboard — typically a RAID array of enterprise SAS drives or local NVMe SSDs. The key characteristics:

- **IOPS**: A 12-drive RAID 10 array of 15K SAS drives delivers 1,200–2,400 random IOPS. A single NVMe SSD delivers 500,000+ IOPS. No artificial caps.
- **Throughput**: RAID arrays deliver 1–4 GB/s sequential throughput; NVMe SSDs reach 5–7 GB/s.
- **Latency**: 2–8 ms for SAS, sub-millisecond for NVMe. No network hop.
- **No IOPS provisioning**: The hardware delivers whatever it can, whenever it can. There is no per-disk IOPS quota.

### Cloud ECS with EVS/EBS: Network-Attached Block Storage

Cloud block storage (EVS on Huawei Cloud, EBS on AWS, Managed Disks on Azure) is fundamentally different. The disk is not inside the server — it is a network-attached storage resource accessed over the hypervisor's storage network.

| Metric | Physical Server (RAID 10, 15K SAS) | Cloud EVS (General Purpose SSD) | Cloud EVS (Ultra-high I/O SSD) |
|---|---|---|---|
| Max IOPS | 2,000+ (uncapped) | 20,000 | 128,000 |
| Max Throughput | 1–4 GB/s | 250 MB/s | 1,000 MB/s |
| Latency | 2–8 ms | 1–2 ms | 0.3–0.5 ms |
| IOPS provisioning | None | Per-disk cap | Per-disk cap |

The critical difference is not the peak numbers — it is the **capping mechanism**. On a physical server, the disk delivers its full capability at all times. On cloud block storage, IOPS and throughput are **provisioned and capped per disk**. Exceed the cap, and I/O requests queue up. The disk does not degrade gracefully — it hard-throttles.

### The Multiblock Read Problem

This is where Oracle's I/O architecture collides with cloud storage caps.

Oracle uses `DB_FILE_MULTIBLOCK_READ_COUNT` to read multiple database blocks in a single I/O operation during full table scans. On a physical server with a large OS I/O size (e.g., 1 MB), Oracle can read 128 blocks (at 8 KB block size) in one I/O call. The hardware handles it transparently.

On cloud block storage, this single I/O call may be counted as multiple I/O operations by the storage backend. As AWS migration specialist Yavor Ivanov documented: *"This single IO call from the Oracle engine will be counted as 4 IO operations by EBS. This matters a lot because, no matter if you use IO1/IO2 or GP2/GP3 volumes, you do have a limit of the IOPS, and it is not as high as in Exadata."*

In other words, a single Oracle multiblock read can consume 4× the IOPS budget on cloud storage compared to what it consumed on physical hardware. Full table scans — which were tolerable on bare metal because the disk had no IOPS cap — become IOPS-eating monsters in the cloud.

---

## Real-World Evidence

### Case 1: Oracle to Azure Cloud — Storage Bottlenecking

A migration team moving Oracle databases to Azure documented that *"I/O latency increased during replay tests using Database Replay (RAT) compared to capture in the on-prem environment. The culprit was storage bottlenecking — the chosen Premium SSD simply couldn't deliver the I/O throughput the on-prem SAN provided."* The fix was not just upgrading to Ultra Disk — it was also rewriting the worst-performing SQL queries to reduce I/O volume.

### Case 2: The $100K Lift-and-Shift Failure

A company migrated its monolithic application to the cloud using pure lift-and-shift. The result: *"The architecture didn't change. Only the hosting bill did. Migration is not modernization. Moving a monolith to the cloud gives you a cloud-hosted monolith."* The application's database queries — designed for local SAN storage — saturated the cloud disk's IOPS cap within hours. The company spent $100K before realizing that code-level SQL optimization was the missing step.

### Case 3: British Airways — The Cautionary Tale

The 2017 British Airways outage, while not solely a database issue, illustrated the core lesson: moving workloads without adapting them to the new environment creates cascading failures. A lift-and-shift of its booking system to a hybrid cloud contributed to 20 hours of downtime, 672 canceled flights, and an estimated $100 million in losses.

### Case 4: Oracle on AWS — IOPS Saturation in Production

When migrating Oracle Exadata workloads to AWS, engineers found that multiblock reads saturate cloud storage in two ways simultaneously: *"by sheer volume of data transferred — hit the throughput limit — or consuming all the IOPS you have, provisioned or not."* On Exadata, smart scans offload I/O to storage cells. On EBS, there is no such offload. Every full scan hits the disk cap directly.

---

## What Needs to Change in the Code

Cloud migration is not just about moving the database — it is about **optimizing how the application talks to the database**. Here are the key code-level changes required:

### 1. Eliminate Full Table Scans

On physical servers, `SELECT COUNT(*) FROM large_table` or `SELECT * FROM orders WHERE status = 'PENDING'` without an index was tolerable — the disk was fast enough. In the cloud, these queries eat IOPS budget and block other operations.

**Action**: Audit all SQL statements. Add appropriate B-tree indexes. Use index-only scans where possible. For analytical queries, consider materialized views.

### 2. Replace Dynamic SQL with Parameterized Queries

Dynamic SQL (string concatenation of query fragments) prevents Oracle from caching execution plans. On physical hardware, the cost of re-parsing was absorbed by CPU. In the cloud, suboptimal execution plans lead to full scans that hit IOPS caps.

**Action**: Use bind variables. Pre-compile frequently-used query patterns. Enable cursor sharing.

### 3. Implement Pagination at the Database Level

Applications that fetch all rows and paginate in memory (e.g., `SELECT * FROM huge_table` then slice in Java/C#) worked when disk I/O was free. In the cloud, this is catastrophic.

**Action**: Use `OFFSET ... FETCH FIRST N ROWS ONLY` (Oracle 12c+) or `ROWNUM`-based pagination. Never fetch more rows than the page needs.

### 4. Optimize Batch Operations

Batch jobs that loop through rows one-by-one, issuing individual `UPDATE` or `INSERT` statements, generate thousands of small random I/O operations — the worst-case scenario for cloud storage.

**Action**: Use bulk operations (`FORALL` in PL/SQL, batch JDBC in Java). Use `MERGE` instead of separate UPDATE/INSERT logic. Reduce round-trips.

### 5. Add Connection Pooling and Statement Caching

On a physical server with a local database, connection overhead was negligible (sub-millisecond TCP on localhost). In the cloud, even with the database on the same ECS, connection establishment and cursor allocation consume resources that compound under load.

**Action**: Use connection pooling (Oracle UCP, HikariCP). Enable implicit statement caching. Set appropriate pool sizes based on ECS vCPU count.

### 6. Partition Large Tables

Tables with tens of millions of rows that are scanned entirely for reporting queries are prime candidates for partition pruning. On cloud storage, partition pruning reduces I/O by orders of magnitude.

**Action**: Implement range or list partitioning on large tables. Ensure queries include partition-key predicates. Use local indexes.

### 7. Re-evaluate PL/SQL Bulk Collect Limits

`BULK COLLECT` with `LIMIT` clause controls how many rows Oracle fetches per round-trip. On physical servers, a large LIMIT (e.g., 1000) was fine. On cloud storage, the I/O pattern of large bulk collects may hit throughput caps.

**Action**: Benchmark different LIMIT values (100, 500, 1000) in the cloud environment. Tune based on actual cloud storage behavior, not on-prem habits.

---

## The Optimization Workflow

```
Phase 1: Baseline          Phase 2: Audit           Phase 3: Optimize
─────────────────          ──────────────          ───────────────
  Capture AWR reports       Identify top SQL         Add indexes
  Record response times     by elapsed time          Rewrite queries
  Profile I/O patterns     Find full table scans     Add partitioning
  Document baseline        Check missing indexes     Batch operations
                            Review execution plans   Pool connections

Phase 4: Validate          Phase 5: Monitor
─────────────────          ───────────────
  Re-run workload          Set up Cloud Eye
  Compare to baseline      /CloudWatch alerts
  Load test               Monitor IOPS utilization
  Regression test         Track SQL performance
```

---

## Effort Estimation for Database Code Refactoring

Based on industry migration case studies and real-world Oracle-to-cloud projects, here is a realistic effort breakdown for a mid-sized application (50–100 database tables, 200–500 SQL statements to audit):

### Assessment Phase

| Task | Effort | Description |
|---|---|---|
| AWR capture & analysis | 2–3 person-days | Capture workload snapshots during peak and batch periods; identify top SQL by elapsed time, I/O, and CPU |
| SQL inventory & classification | 3–5 person-days | Catalog all SQL statements from application code, stored procedures, and reports; classify by criticality |
| Execution plan review | 3–5 person-days | Generate `EXPLAIN PLAN` for top 50–100 SQL; identify full scans, Cartesian joins, inefficient access paths |
| Infrastructure baseline | 1–2 person-days | Document current IOPS, throughput, latency; map to cloud EVS disk type |

**Subtotal: 9–15 person-days**

### Optimization Phase

| Task | Effort | Description |
|---|---|---|
| Index creation & tuning | 5–8 person-days | Design and create B-tree, bitmap, and function-based indexes; validate with real query patterns |
| SQL rewriting | 8–15 person-days | Rewrite top-N problematic queries: eliminate full scans, add bind variables, implement pagination |
| Stored procedure optimization | 5–10 person-days | Optimize PL/SQL: BULK COLLECT tuning, FORALL adoption, cursor management |
| Batch job refactoring | 3–7 person-days | Convert row-by-row loops to bulk operations; implement parallelism where appropriate |
| Partitioning implementation | 3–5 person-days | Design partitioning strategy; migrate large tables; build local indexes |
| Connection pooling setup | 1–2 person-days | Configure UCP/HikariCP; tune pool sizes; enable statement caching |

**Subtotal: 25–47 person-days**

### Validation & Deployment Phase

| Task | Effort | Description |
|---|---|---|
| Performance regression testing | 3–5 person-days | Re-run all SQL against cloud environment; compare to baseline; document improvements |
| Load testing | 2–3 person-days | Simulate peak load; verify IOPS headroom; identify remaining bottlenecks |
| Monitoring setup | 1–2 person-days | Configure cloud monitoring (Cloud Eye/CloudWatch); set IOPS and latency alerts |
| Documentation & runbooks | 2–3 person-days | Document optimization decisions; create operational runbooks for future tuning |

**Subtotal: 8–13 person-days**

### Summary

| Phase | Effort (person-days) |
|---|---|
| Assessment | 9 – 15 |
| Optimization | 25 – 47 |
| Validation & Deployment | 8 – 13 |
| **Total** | **42 – 75 person-days** |

For a typical team of 2 developers + 1 DBA working in parallel, this translates to approximately **6–12 weeks** of dedicated effort.

### Key Variables That Shift the Estimate

- **Application size**: 50 tables vs. 500 tables shifts the estimate by 3–5×.
- **Code quality**: Well-structured code with clear data access layers is faster to optimize than scattered inline SQL across hundreds of source files.
- **SQL complexity**: Simple CRUD queries take 0.5–1 hour each. Complex analytical queries with multiple subqueries and UNIONs can take 1–2 days each.
- **Testing infrastructure**: If a staging environment with production-like data volumes exists, validation is faster. If not, add 5–10 person-days for test data setup.
- **Team experience**: Developers familiar with Oracle execution plans and AWR analysis work 2–3× faster than those learning on the job.

---

## Cost of Inaction

Failing to optimize database code during cloud migration is not free — it just defers the cost:

| Consequence | Impact |
|---|---|
| Over-provisioned cloud storage | Paying for Ultra-high I/O EVS (3–5× the cost of General Purpose SSD) to compensate for bad queries |
| Slow user experience | Productivity loss, user complaints, SLA violations |
| Batch window overruns | Nightly jobs bleeding into business hours, cascading delays |
| Emergency firefighting | Unplanned optimization under production-down pressure — always more expensive than planned work |
| Cloud reputation damage | "The cloud is slow" narrative, when the real issue is unoptimized code |

As one practitioner put it: *"When you move a monolithic SQL Server or Oracle database into an AWS EC2 or Azure VM environment without refactoring, you are essentially paying a premium to rent someone else's hardware to run inefficient, legacy code."*

---

## Conclusion

Cloud migration is not a copy operation. It is a transformation that demands changes at every layer — infrastructure, architecture, and code. The database, in particular, requires careful attention because cloud storage behaves fundamentally differently from direct-attached disks:

1. **IOPS are capped**, not unlimited. Full table scans that were tolerable on bare metal become IOPS-eating bottlenecks in the cloud.
2. **Multiblock reads are counted differently**. A single Oracle I/O call may consume multiple IOPS on cloud storage, accelerating cap exhaustion.
3. **Throughput has hard limits**. Cloud EVS/EBS disks throttle when the limit is hit — they don't degrade gracefully.

The solution is not to throw more expensive storage at the problem. It is to **optimize the code**: eliminate unnecessary full scans, add indexes, use bind variables, implement pagination, batch operations, and partition large tables.

For a mid-sized application, this optimization effort is **6–12 weeks of dedicated work** — a fraction of the cost of running an over-provisioned, under-performing cloud deployment for even a single year.

**The cloud does not make your database faster. It makes your optimization decisions matter.**

---

## References

- Oracle Help Center, "DB_FILE_MULTIBLOCK_READ_COUNT" — [docs.oracle.com](https://docs.oracle.com/en/database/oracle/oracle-database/19/refrn/DB_FILE_MULTIBLOCK_READ_COUNT.html)
- Yavor Ivanov, "Migrating Exadata Workloads to AWS, Part Five: EBS Volumes for Oracle DBAs" — [LinkedIn](https://www.linkedin.com/pulse/migrating-exadata-workloads-aws-part-five-ebs-volumes-yavor-ivanov)
- DBA Insight, "Migrating Oracle Databases to Azure Cloud — Performance Validation and Lessons Learned" — [dbainsight.com](https://dbainsight.com/2025/11/oracle-database-migration-azure-cloud/)
- Thomas Nys, "The $100K Cloud Migration Mistake" — [thomasnys.com](https://thomasnys.com/posts/the-100k-cloud-migration-mistake/)
- Huawei Cloud, "EVS Disk Types and Performance" — [support.huaweicloud.com](https://support.huaweicloud.com/intl/en-us/productdesc-evs/en-us_topic_0014580744.html)
- AWS, "Determining IOPS Needs for Oracle Database on AWS" — [docs.aws.amazon.com](https://docs.aws.amazon.com/whitepapers/latest/determining-iops-needs-oracle-db-on-aws/)
- Oracle, "Moving Databases to Oracle Cloud: Performance Best Practices" — [oracle.com](https://www.oracle.com/technetwork/oem/db-mgmt/con6980-moving-to-cloud-perf-best-p-3398018.pdf)
- PerformanceOne, "Beyond the Lift and Shift: Cloud Migration Cost Optimization" — [performanceonedatasolutions.com](https://performanceonedatasolutions.com/blogs/prevent-post-migration-cloud-bill-shock/)
- Jose Carlos Moreira, "Best Practices for Optimizing Oracle Database Performance in Cloud Environments" — [Medium](https://medium.com/@deco_92728/best-practices-for-optimizing-oracle-database-performance-in-cloud-environments-a5ea667bc647)
