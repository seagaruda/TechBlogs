---
title: "From Monolith to Distributed: Best Practices and Effort Estimation"
date: "2026-06-10"
summary: "Traditional industry software — factory MES, hospital HIS, power SCADA, AMI metering platforms — was built for a different era. Here's a grounded roadmap for migrating to distributed architecture, with realistic effort estimates for each stage."
tags: ["Architecture", "Distributed Systems", "Migration", "AMI", "DevOps"]
slug: "monolith-to-distributed-architecture"
lang: "en"
---

# From Monolith to Distributed: Best Practices and Effort Estimation

> Author: seagaruda | Published: 2026-06-10

---

## Background

Traditional industry software — factory MES, hospital HIS, power SCADA, AMI metering platforms — was born in the monolith era. One deployment package, one database, one server. At the time, that was the most practical choice.

As systems age, connected devices multiply, and user loads grow, the cracks start showing:

- **Scaling ceiling**: traffic spikes require vertical scaling of the entire machine — expensive and inflexible
- **Single point of failure**: one server goes down, the whole system stops
- **Downtime deploys**: every release requires a maintenance window; operations teams hate it
- **Resource waste**: servers idle at off-peak hours, yet can't handle peak loads

The knee-jerk reaction is often "should we go microservices?" This article argues: **distributed deployment with load balancing is the right first step for traditional industry systems**. It solves the majority of real problems at a manageable cost. Microservices are a later conversation — when team size and business complexity actually justify it.

---

## 1. Target State: What the System Looks Like After Migration

After migration, the system should reach this topology:

```
                    ┌──────────────────────────────────────┐
Users / Devices ──▶ │   Nginx Load Balancer / API Gateway   │
                    └───────────┬──────────────┬────────────┘
                                │              │
                       ┌────────▼───┐  ┌───────▼────┐
                       │  App Node 1 │  │  App Node 2 │  ← scale horizontally
                       └────────┬───┘  └───────┬────┘
                                │              │
              ┌─────────────────▼──────────────▼──────────────┐
              │   Shared Database (primary/replica read split)  │
              │   Primary (writes) + Read Replica (queries)     │
              └────────────────────────────────────────────────┘
              ┌────────────────────────────────────────────────┐
              │   Redis (shared session / cache / dist. locks)  │
              └────────────────────────────────────────────────┘
```

Core gains:
- **High availability**: one node crashes, traffic routes to others — users notice nothing
- **Horizontal scaling**: add a machine, start an instance, capacity grows linearly
- **Rolling deploys**: replace nodes one at a time, no maintenance windows
- **Database relief**: read/write split means reporting queries no longer block the primary

---

## 2. The Core Work: Making the Application Stateless

**Multi-instance deployment requires a stateless application** — any instance must be able to handle any request without depending on data local to a specific machine.

Monolith systems typically carry four categories of state that need to be addressed.

### 2.1 Session Migration

**Problem**: Sessions live in JVM heap memory. After login, requests must hit the same machine. Add a second instance and sessions break.

**Fix**: Move sessions to Redis, shared across all instances.

```java
// Spring Boot: one dependency + config, almost no code change
// pom.xml
<dependency>
    <groupId>org.springframework.session</groupId>
    <artifactId>spring-session-data-redis</artifactId>
</dependency>

// application.yml
spring:
  session:
    store-type: redis
  redis:
    host: redis-cluster-host
    port: 6379
```

**Watch out**: objects stored in session must implement `Serializable`. Check your user info objects — they're the most common offender.

**Effort**: 2–5 days (including testing)

---

### 2.2 Local File Storage Migration

**Problem**: uploaded files, generated reports, exported spreadsheets all live on the local disk. Instance A uploads a file; Instance B can't find it.

**Fix**: route all file I/O through object storage (MinIO for self-hosted, OSS/S3 on cloud).

```java
// Before: direct local path
File file = new File("/data/uploads/" + filename);
FileUtils.copyInputStreamToFile(inputStream, file);

// After: unified storage service
storageService.upload(inputStream, filename);  // delegates to MinIO/OSS
String url = storageService.getUrl(filename);
```

Audit tip: search the codebase for `new File(`, `FileInputStream`, `FileOutputStream`, `MultipartFile.transferTo` — every hit is a candidate for migration.

**Effort**: 3–7 days (depends on how scattered file operations are)

---

### 2.3 Scheduled Job Deduplication

**Problem**: every instance runs every scheduled job. Three instances running the nightly reconciliation job simultaneously means triple processing and triple notifications.

**Fix** (pick one based on complexity):

**Option A — Distributed lock** (simple tasks, low job count):
```java
@Scheduled(cron = "0 0 1 * * ?")
public void dailySettlement() {
    boolean locked = redisLock.tryLock("daily-settlement", 60);
    if (!locked) return;  // another instance already running
    try {
        doSettlement();
    } finally {
        redisLock.unlock("daily-settlement");
    }
}
```

**Option B — Dedicated scheduler** (many jobs, complex dependencies): extract scheduled jobs to a single dedicated instance that doesn't participate in load balancing, or adopt a job scheduling framework (XXL-JOB, Quartz Cluster).

**Effort**: 3–10 days (depends on job count and business criticality)

---

### 2.4 In-Process Cache Migration

**Problem**: `HashMap` or Guava `LoadingCache` caches live in each instance's memory. After a cache update on Instance A, Instance B still serves stale data.

**Fix**: replace local caches used for cross-request sharing with Redis. Keep local caches only for data that is truly read-only and process-scoped (e.g., static config loaded at startup).

```java
// Before: local Guava cache
LoadingCache<String, Device> deviceCache = CacheBuilder.newBuilder()
    .expireAfterWrite(5, TimeUnit.MINUTES)
    .build(key -> deviceRepo.findById(key));

// After: Redis-backed, consistent across instances
@Cacheable(value = "device", key = "#deviceId")
public Device getDevice(String deviceId) {
    return deviceRepo.findById(deviceId);
}
```

**Effort**: 2–5 days

---

## 3. Infrastructure: Load Balancer + Database Read/Write Split

### 3.1 Nginx Load Balancing

A minimal working config:

```nginx
upstream app_cluster {
    least_conn;  # route to the instance with fewest active connections
    server 10.0.0.1:8080;
    server 10.0.0.2:8080;
    keepalive 32;
}

server {
    listen 80;
    location / {
        proxy_pass http://app_cluster;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }
}
```

**Health check**: configure Nginx `health_check` or use a cloud load balancer (ALB/CLB) with active health probes. Remove unhealthy instances from the pool automatically.

**Effort**: 1–2 days

---

### 3.2 Database Read/Write Split

```java
// Route writes to primary, reads to replica via AbstractRoutingDataSource
public class ReadWriteRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
            ? "replica" : "primary";
    }
}

// Mark read-only transactions explicitly
@Transactional(readOnly = true)
public List<MeterReading> queryReadings(String meterId, LocalDate from, LocalDate to) {
    return meterReadingRepo.findByMeterIdAndDateBetween(meterId, from, to);
}
```

**Effort**: 3–5 days

---

## 4. Effort Estimation Summary

| Work Item | Effort | Risk |
|-----------|--------|------|
| Session → Redis | 2–5 days | Low — Spring Boot makes this nearly config-only |
| File storage → Object Storage | 3–7 days | Medium — scattered file ops need a full audit |
| Scheduled job deduplication | 3–10 days | Medium–High — business logic sensitivity varies |
| Local cache → Redis | 2–5 days | Low |
| Nginx load balancer setup | 1–2 days | Low |
| DB read/write split | 3–5 days | Medium |
| Integration testing | 5–10 days | — |
| Staged rollout + rollback plan | 3–5 days | — |
| **Total** | **22–49 days** | |

---

## 5. Migration Path: Three Phases

### Phase 1 — Stateless First (Weeks 1–3)
Fix all four stateful problems: session, files, scheduled jobs, local cache. Deploy to a test environment and validate thoroughly before touching production.

### Phase 2 — Multi-Instance + Load Balancer (Weeks 4–5)
Start with two instances behind Nginx. Validate session sharing, file access, and job deduplication under real traffic. Keep rollback ready.

### Phase 3 — Database Read/Write Split (Week 6+)
Set up primary/replica replication. Enable read routing gradually — start with low-risk query-only modules. Monitor replica lag.

---

## 6. Common Pitfalls

**Don't skip the stateless audit.** The most common failure is deploying multiple instances before fixing session or file storage, then debugging mysterious "random" login failures in production.

**Test scheduled jobs with two instances running simultaneously** before go-live. A reconciliation job firing twice is a serious data integrity issue.

**Replica lag is real.** After a write, a subsequent read routed to the replica may return stale data. For operations that read immediately after writing (e.g., "save and display"), force the read to primary.

**Microservices are not the next step.** After completing this migration, the system handles the vast majority of scaling scenarios most traditional industry platforms will ever face. Microservices introduce distributed transactions, service mesh, independent CI/CD per service, and significant operational overhead. Revisit that decision when the team has grown and the current architecture is a proven bottleneck — not before.

---

## Conclusion

The migration from monolith to distributed deployment is a concrete, achievable project for most traditional industry software teams — typically 4–8 weeks of focused engineering. The payoff is high availability, horizontal scaling, and zero-downtime deploys.

The architecture described here — stateless app nodes, shared Redis, load balancer, primary/replica database — is battle-tested across factory MES, hospital systems, and utility metering platforms. It is not a stepping stone to something else. It is a stable, long-term architecture that most teams can run comfortably for years.
