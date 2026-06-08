---
title: "PostgreSQL + TimescaleDB on Huawei Cloud vs Oracle for AMI Projects"
date: "2025-06-10"
summary: "Oracle has been the default choice for AMI platforms for years. TimescaleDB on Huawei Cloud RDS for PostgreSQL offers better time-series performance, native hypertable partitioning, and a dramatically lower total cost. Here's the full comparison."
tags: ["Database", "TimescaleDB", "PostgreSQL", "AMI", "HuaweiCloud"]
slug: "timescaledb-vs-oracle-ami"
lang: "en"
---

# PostgreSQL + TimescaleDB on Huawei Cloud vs Oracle for AMI Projects

## Overview

Advanced Metering Infrastructure (AMI) is fundamentally a time-series problem. Millions of smart meters report readings at 15-minute or hourly intervals, 24 hours a day. The database at the center of an AMI platform must handle continuous high-volume inserts, fast time-range queries, long-term data retention, and growing device counts — all without planned downtime.

Traditional AMI deployments have relied on Oracle in a primary/standby configuration. This document makes the case that **Huawei Cloud RDS for PostgreSQL with TimescaleDB** is a technically superior and dramatically more cost-effective alternative for modern AMI platforms such as Gurux.DLMS.AMI.

---

## Huawei Cloud RDS for PostgreSQL: TimescaleDB Built In

Huawei Cloud RDS for PostgreSQL (versions 12 through 15) ships with TimescaleDB as a built-in plugin. No separate installation, no custom builds, no OS-level access required. Enabling it is a single SQL statement run against your target database:

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;
```

Huawei Cloud handles the `shared_preload_libraries` configuration automatically. From that point, you have the full TimescaleDB feature set available inside a fully managed, cloud-native PostgreSQL instance — with automated backups, high availability, and point-in-time recovery included.

Official documentation: [Huawei Cloud RDS for PostgreSQL Plugin Overview](https://support.huaweicloud.com/intl/en-us/usermanual-rds/rds_09_0043.html)

---

## Why TimescaleDB Is the Right Engine for AMI Data

Meter reading data has a well-defined shape: a device identifier, a timestamp, and one or more measured values. This pattern repeats billions of times across the life of an AMI deployment. TimescaleDB is purpose-built for exactly this workload.

### Hypertables: Automatic Time Partitioning

A TimescaleDB hypertable automatically partitions data into time-based chunks (configurable — e.g., one chunk per week). When a query asks for the last 30 days of readings for a specific meter, the query planner touches only the relevant chunks and skips everything else. On a table with years of history, this is the difference between a millisecond response and a full table scan.

```sql
-- Create the meter readings table as a hypertable
CREATE TABLE meter_readings (
    meter_id    TEXT        NOT NULL,
    ts          TIMESTAMPTZ NOT NULL,
    active_kwh  DOUBLE PRECISION,
    reactive_kvarh DOUBLE PRECISION
);

SELECT create_hypertable('meter_readings', 'ts');
```

### Continuous Aggregates: Pre-computed Rollups at Zero Query Cost

TimescaleDB continuous aggregates maintain pre-computed summaries (hourly, daily, monthly totals) that update automatically as new data arrives. Reporting queries that previously required full scans over billions of rows now read from compact aggregate tables.

### Compression: 90–95% Storage Reduction on Cold Data

Older chunks — readings from last month, last year — are compressed automatically. TimescaleDB's columnar compression routinely achieves 90–95% reduction in storage footprint on time-series data. The data remains queryable; compression is transparent to the application.

---

## Oracle in AMI: The Real Costs

Oracle's licensing model is based on processor cores, with a multiplier applied to physical cores based on the processor type. For a typical AMI deployment running on 4 physical cores:

- Oracle Database Enterprise Edition: ~$47,500 per processor (list price)
- Support and maintenance: 22% of license cost per year
- High Availability (Active Data Guard): additional per-processor license

A modest 4-core primary + 4-core standby deployment routinely exceeds $400,000 USD at list price, before support contracts.

PostgreSQL is open-source. TimescaleDB Community Edition is open-source. Huawei Cloud RDS charges only for compute and storage. The licensing cost delta is not incremental — it is an order of magnitude.

---

## Head-to-Head: TimescaleDB vs Oracle for AMI Workloads

| Capability | PostgreSQL + TimescaleDB | Oracle |
|---|---|---|
| Time-series partitioning | Automatic hypertables | Manual range partitioning |
| Continuous rollup aggregates | Built-in, auto-refreshing | Materialized views, manual refresh |
| Columnar compression | Native, transparent | Advanced Compression (extra cost) |
| Licensing model | Open-source | Per-core, expensive |
| Huawei Cloud managed service | Full support (RDS for PG) | Limited managed options |
| High availability | Streaming replication, built-in | Data Guard (licensed separately) |
| Point-in-time recovery | Standard PostgreSQL feature | Requires additional configuration |

---

## Migration Path from Oracle

A full migration is outside the scope of this document, but the key steps are:

1. **Schema conversion**: Oracle-specific types (`VARCHAR2`, `NUMBER`) map cleanly to PostgreSQL equivalents (`TEXT`, `NUMERIC`). Tools like `ora2pg` automate most of the DDL conversion.
2. **Hypertable setup**: Identify your primary time-series tables and convert them to hypertables. This is a one-time step that does not require application changes.
3. **Continuous aggregate definitions**: Replace Oracle materialized views with TimescaleDB continuous aggregates for auto-refreshing rollups.
4. **Application layer**: PostgreSQL uses standard JDBC/ODBC drivers. Connection string changes are typically the only application modification required.

---

## Conclusion

For AMI deployments, TimescaleDB on Huawei Cloud RDS for PostgreSQL delivers better time-series performance than Oracle at a fraction of the total cost. The hypertable model maps directly to the meter-reading data pattern. Continuous aggregates replace expensive rollup queries. Columnar compression handles the long-tail of historical data efficiently.

Oracle remains a defensible choice for organizations with existing enterprise agreements and Oracle-specific tooling already in place. For new AMI deployments or platforms undergoing architectural modernization, PostgreSQL + TimescaleDB is the technically correct and economically rational choice.
