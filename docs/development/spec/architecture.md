# Architecture: Firisbe Scheduler Library

**Document Status:** Draft
**Last Updated:** 2025-12-26

---

## 1. Overview

This is a custom scheduler library that replaces `@Scheduled`, `@SchedulerLock`, and `TenantContextHolder.runForeachTenantContext()` boilerplate. Developers write only business logic.

```
+------------------------------------------------------------------+
|                    WHAT LIBRARY HANDLES                           |
+------------------------------------------------------------------+
|                                                                   |
|   1. @Scheduled(cron = "...")         --> Library handles         |
|   2. @SchedulerLock(...)              --> Library handles         |
|   3. Date calculation                 --> Library handles         |
|   4. TenantContextHolder.runForeach   --> Library handles         |
|   5. PfContextHolder setup            --> Library handles         |
|   6. Per-tenant retry                 --> Library handles         |
|                                                                   |
+------------------------------------------------------------------+
|                                                                   |
|   DEVELOPER WRITES ONLY:                                         |
|   - @ScheduledJob annotation                                      |
|   - Business logic for SINGLE TENANT                             |
|                                                                   |
+------------------------------------------------------------------+
```

---

## 2. Before vs After

### 2.1 Current Code (20+ lines of boilerplate)

```java
@Scheduled(cron = "${vpos-summary.scheduled.cron}")
@SchedulerLock(name = "scheduled_vpos_summary", lockAtMostFor = "60m", lockAtLeastFor = "60m")
public void summaryScheduler() {
    log.info("VPos summary scheduler started");

    LocalDate localDate = LocalDate.now().minusDays(1);
    OffsetDateTime beginDate = localDate.atTime(LocalTime.MIN).atOffset(ZoneOffset.UTC);
    OffsetDateTime endDate = localDate.atTime(LocalTime.MAX).atOffset(ZoneOffset.UTC);

    TenantContextHolder.runForeachTenantContext(
        PfContextBuilder.builder()
            .username("scheduled")
            .clientChannel(ClientChannel.Other), 
        () -> {
            log.info("Started for tenant {}", PfContextHolder.get().getTenantId());

            List<SummaryEntity> dataList = vPosTransactionDataService
                .getTransactionSummary(beginDate, endDate);
                
            dataList.forEach(k -> k.setDate(beginDate));
            summaryDataService.saveAll(dataList);
            
            log.info("Saved for tenant {}", PfContextHolder.get().getTenantId());
        }
    );
    log.info("VPos summary scheduler finished");
}
```

### 2.2 New Code (Only business logic)

```java
@ScheduledJob(name = "vpos-summary", cron = "${vpos-summary.scheduled.cron}")
public void summaryScheduler(JobContext ctx) {
    // Library already set up tenant context - PfContextHolder is ready
    log.info("Started for tenant {}", ctx.getTenantId());

    List<SummaryEntity> dataList = vPosTransactionDataService
        .getTransactionSummary(ctx.getBeginDate(), ctx.getEndDate());
        
    dataList.forEach(k -> k.setDate(ctx.getBeginDate()));
    summaryDataService.saveAll(dataList);
    
    log.info("Saved for tenant {}", ctx.getTenantId());
}
```

**What changed:**
- No `@Scheduled` - library handles cron
- No `@SchedulerLock` - library handles locking
- No date calculation - library provides via `ctx.getBeginDate()`
- No `TenantContextHolder.runForeachTenantContext()` - library calls method for each tenant
- No `PfContextBuilder` - library sets up context automatically

---

## 3. High-Level Architecture

```
+------------------------------------------------------------------+
|                    MICROSERVICE                                   |
+------------------------------------------------------------------+
|                                                                   |
|   @ScheduledJob(name = "summary", cron = "0 0 2 * * *")          |
|   public void process(JobContext ctx) {                          |
|       // Business logic for SINGLE TENANT                        |
|       // ctx.getTenantId() - current tenant                      |
|       // ctx.getBeginDate() - calculated date range              |
|       // PfContextHolder.get() - already set by library          |
|   }                                                               |
|                                                                   |
+------------------------------------------------------------------+
                               |
                               v
+------------------------------------------------------------------+
|              FIRISBE-SCHEDULER-STARTER (Our Library)              |
+------------------------------------------------------------------+
|                                                                   |
|  1. STARTUP                                                       |
|     +------------------+                                          |
|     | Job Registry     |  Scans @ScheduledJob, registers cron    |
|     +------------------+                                          |
|                                                                   |
|  2. CRON TRIGGER                                                  |
|     +------------------+                                          |
|     | Cron Scheduler   |  Triggers job at scheduled time         |
|     +------------------+                                          |
|                                                                   |
|  3. LOCKING                                                       |
|     +------------------+                                          |
|     | Lock Manager     |  Acquires distributed lock              |
|     +------------------+  (replaces @SchedulerLock)               |
|                                                                   |
|  4. TENANT ITERATION                                              |
|     +------------------------------------------------------------------+
|     | TenantContextHolder.runForeachTenantContext(                    |
|     |     PfContextBuilder.builder()                                   |
|     |         .username("scheduled")                                   |
|     |         .clientChannel(ClientChannel.Other),                     |
|     |     () -> {                                                      |
|     |         // Library calls user's method HERE                      |
|     |         jobMethod.invoke(buildJobContext());                     |
|     |     }                                                            |
|     | );                                                               |
|     +------------------------------------------------------------------+
|                                                                   |
|  5. PER-TENANT RETRY                                              |
|     +------------------+                                          |
|     | Retry Engine     |  Retries failed tenants                 |
|     +------------------+                                          |
|                                                                   |
|  6. STATE TRACKING                                                |
|     +------------------+                                          |
|     | State Manager    |  Records execution history              |
|     +------------------+                                          |
|                                                                   |
+------------------------------------------------------------------+
```

---

## 4. Execution Flow

### 4.1 Complete Flow

```
02:00 AM - Cron Triggers
        |
        v
+------------------+
| Lock Manager     |  Try to acquire lock "job-vpos-summary"
+------------------+
        |
        +--- Lock exists (another pod) ---> SKIP
        |
        +--- Lock acquired
        |
        v
+------------------+
| State Manager    |  Create execution record (status = RUNNING)
+------------------+
        |
        v
+------------------+
| Date Calculator  |  Calculate beginDate, endDate
+------------------+  (yesterday 00:00 to 23:59 UTC)
        |
        v
+------------------------------------------------------------------+
|  LIBRARY INTERNALLY CALLS:                                        |
|                                                                   |
|  TenantContextHolder.runForeachTenantContext(                    |
|      PfContextBuilder.builder()                                   |
|          .username("scheduled")                                   |
|          .clientChannel(ClientChannel.Other),                     |
|      () -> {                                                      |
|          // For each tenant, library does:                        |
|          JobContext ctx = buildContext(currentTenant, dates);     |
|          try {                                                    |
|              userMethod.invoke(ctx);  // Call user's business code|
|              recordSuccess(tenant);                               |
|          } catch (Exception e) {                                  |
|              retryEngine.handleFailure(tenant, e);               |
|          }                                                        |
|      }                                                            |
|  );                                                               |
+------------------------------------------------------------------+
        |
        v
+------------------+
| State Manager    |  Update final status
+------------------+
        |
        v
+------------------+
| Lock Manager     |  Release lock
+------------------+
        |
        v
    COMPLETE
```

### 4.2 Per-Tenant Retry Flow

```
TENANT EXECUTION (Tenant A)
        |
        v
+------------------+
| JobContext       |  ctx.tenantId = "tenant-a"
| Creation         |  ctx.beginDate = yesterday start
+------------------+  ctx.endDate = yesterday end
        |
        v
+------------------+
| Invoke User      |  summaryScheduler(ctx)
| Method           |  (PfContextHolder already set by runForeachTenantContext)
+------------------+
        |
        +--- Success ---> Record success, continue to Tenant B
        |
        +--- Exception
                |
                v
        +------------------+
        | Exception        |  Retryable?
        | Classifier       |
        +------------------+
                |
        +-------+-------+
        |               |
    RETRYABLE      NON-RETRYABLE
        |               |
        v               v
   Retry Engine    Mark FAILED
        |          Continue to Tenant B
        v
   attempt < max?
        |
   +----+----+
   |         |
  YES        NO
   |         |
   v         v
 Wait     Mark FAILED
(backoff) Continue
   |
   v
 Retry invoke
```

---

## 5. JobContext - What User Receives

```java
public class JobContext {
    
    // Current tenant (from PfContextHolder)
    private String tenantId;
    
    // Date range (calculated by library)
    private OffsetDateTime beginDate;  // Yesterday 00:00:00 UTC
    private OffsetDateTime endDate;    // Yesterday 23:59:59 UTC
    
    // Execution metadata
    private String executionId;
    private int attemptNumber;
    
    // Access to original PfContext if needed
    public PfContext getPfContext() {
        return PfContextHolder.get();
    }
    
    // Access to tenant ID shortcut
    public String getTenantId() {
        return PfContextHolder.get().getTenantId();
    }
}
```

---

## 6. Core Components

```
+------------------------------------------------------------------+
|                       LIBRARY COMPONENTS                          |
+------------------------------------------------------------------+

1. JOB REGISTRY
   - Scans @ScheduledJob at startup
   - Stores job metadata (name, cron, retry config)
   - Registers Spring @Scheduled trigger internally

2. LOCK MANAGER (Replaces ShedLock)
   - Database-based distributed locking
   - Prevents duplicate execution across pods
   - Auto-release on timeout/failure

3. EXECUTION ENGINE
   - Calls TenantContextHolder.runForeachTenantContext()
   - Invokes user method for each tenant
   - Coordinates retry and state tracking

4. DATE CALCULATOR
   - Calculates beginDate, endDate based on config
   - Default: yesterday (00:00 to 23:59 UTC)
   - Configurable: today, custom offset

5. RETRY ENGINE
   - Per-tenant retry on failure
   - Configurable attempts, backoff
   - Exception classification

6. STATE MANAGER
   - Persists execution state to database
   - Tracks success/failure per tenant
   - Provides execution history
```

---

## 7. Database Schema

```
TABLE: scheduler_job
─────────────────────
| Column          | Type         | Description                     |
|-----------------|--------------|----------------------------------|
| id              | VARCHAR(64)  | Primary key (job name)          |
| cron_expression | VARCHAR(100) | Cron schedule                   |
| enabled         | BOOLEAN      | Is job enabled                  |
| retry_attempts  | INT          | Max retry attempts              |

TABLE: scheduler_execution
──────────────────────────
| Column          | Type         | Description                     |
|-----------------|--------------|----------------------------------|
| id              | UUID         | Primary key                     |
| job_id          | VARCHAR(64)  | FK to scheduler_job             |
| status          | VARCHAR(20)  | RUNNING/SUCCESS/PARTIAL/FAILED  |
| total_tenants   | INT          | Total tenants processed         |
| success_count   | INT          | Successful tenants              |
| failed_count    | INT          | Failed tenants                  |
| started_at      | TIMESTAMP    | Execution start                 |
| finished_at     | TIMESTAMP    | Execution end                   |

TABLE: scheduler_tenant_execution
─────────────────────────────────
| Column          | Type         | Description                     |
|-----------------|--------------|----------------------------------|
| id              | UUID         | Primary key                     |
| execution_id    | UUID         | FK to scheduler_execution       |
| tenant_id       | VARCHAR(64)  | Tenant identifier               |
| status          | VARCHAR(20)  | RUNNING/SUCCESS/FAILED          |
| attempt_count   | INT          | Current attempt number          |
| error_message   | TEXT         | Error details if failed         |

TABLE: scheduler_lock
─────────────────────
| Column          | Type         | Description                     |
|-----------------|--------------|----------------------------------|
| lock_name       | VARCHAR(64)  | Primary key (job name)          |
| locked_by       | VARCHAR(100) | Pod identifier                  |
| locked_at       | TIMESTAMP    | Lock acquisition time           |
| expires_at      | TIMESTAMP    | Lock expiry time                |
```

---

## 8. Package Structure

```
com.firisbe.scheduler
|
+-- annotation/
|   +-- ScheduledJob.java
|   +-- NonRetryable.java
|   +-- EnableFirisbeScheduler.java
|
+-- autoconfigure/
|   +-- SchedulerAutoConfiguration.java
|   +-- SchedulerProperties.java
|
+-- core/
|   +-- JobRegistry.java
|   +-- JobMetadata.java
|   +-- JobContext.java
|   +-- DateCalculator.java
|
+-- execution/
|   +-- ExecutionEngine.java
|   +-- JobInvoker.java
|
+-- retry/
|   +-- RetryEngine.java
|   +-- RetryPolicy.java
|   +-- ExceptionClassifier.java
|
+-- lock/
|   +-- LockManager.java
|   +-- LockRepository.java
|
+-- state/
|   +-- StateManager.java
|   +-- ExecutionRepository.java
|   +-- TenantExecutionRepository.java
|
+-- entity/
|   +-- JobEntity.java
|   +-- ExecutionEntity.java
|   +-- TenantExecutionEntity.java
|   +-- LockEntity.java
```

---

## 9. Configuration

```yaml
firisbe:
  scheduler:
    enabled: true
    
    # Context configuration (used when calling runForeachTenantContext)
    context:
      username: "scheduled"
      client-channel: Other
      
    # Date range calculation
    date-range:
      type: YESTERDAY       # YESTERDAY, TODAY, CUSTOM
      offset-days: -1       # For CUSTOM type
      
    # Retry configuration
    retry:
      max-attempts: 3
      initial-interval: 5s
      max-interval: 1m
      backoff-multiplier: 2.0
      
    # Lock configuration
    lock:
      timeout: 60m
```

---

## 10. Summary

```
+------------------------------------------------------------------+
|                LIBRARY RESPONSIBILITY MATRIX                      |
+------------------------------------------------------------------+
|                                                                   |
|   FEATURE                         HANDLED BY                      |
|   ───────────────────────────────────────────────────────────    |
|   Cron scheduling                 Library (internal @Scheduled)   |
|   Distributed locking             Library (replaces ShedLock)     |
|   Date calculation                Library (provides via ctx)      |
|   Tenant iteration                Library (calls runForeach)      |
|   Context setup                   Library (via runForeach)        |
|   Per-tenant retry                Library                         |
|   Execution history               Library (database)              |
|   Business logic                  DEVELOPER                       |
|                                                                   |
+------------------------------------------------------------------+
```

---

## 11. Next: Detailed Component Design

- [ ] Section 12: Job Registry Design
- [ ] Section 13: Lock Manager Design
- [ ] Section 14: Execution Engine Design
- [ ] Section 15: Retry Engine Design
- [ ] Section 16: State Manager Design

---
