# Architecture: Firisbe Scheduler Library

**Document Status:** Draft
**Last Updated:** 2025-12-26

---

## 1. Overview

This is a **custom scheduler library** that builds on top of existing Firisbe infrastructure. We are leveraging existing components like `TenantContextHolder` and `PfContextHolder`.

```
+------------------------------------------------------------------+
|                       KEY PRINCIPLE                               |
+------------------------------------------------------------------+
|                                                                   |
|   We are NOT reinventing the wheel.                              |
|                                                                   |
|   Existing Infrastructure:                                        |
|   - TenantContextHolder.runForeachTenantContext() --> USE IT     |
|   - PfContextHolder                               --> USE IT     |
|   - PfContextBuilder                              --> USE IT     |
|   - ShedLock                                      --> REPLACE IT |
|   - Spring @Scheduled                             --> ENHANCE IT |
|                                                                   |
+------------------------------------------------------------------+
```

---

## 2. Current vs New Architecture

### 2.1 Current Implementation (What Developers Write Today)

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
            
            // Business logic here
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

**Problems with this approach:**
- Boilerplate: Every job repeats `@Scheduled`, `@SchedulerLock`, `runForeachTenantContext`
- No tenant-level retry: If Tenant A fails, all remaining tenants are skipped
- No visibility: Must search logs to find failures
- Manual date handling in every job

### 2.2 New Implementation (What Developers Will Write)

```java
@ScheduledJob(
    name = "vpos-summary",
    cron = "${vpos-summary.scheduled.cron}"
)
public void summaryScheduler(JobContext ctx) {
    // ONLY business logic - nothing else!
    List<SummaryEntity> dataList = vPosTransactionDataService
        .getTransactionSummary(ctx.getBeginDate(), ctx.getEndDate());
    dataList.forEach(k -> k.setDate(ctx.getBeginDate()));
    summaryDataService.saveAll(dataList);
}
```

**What the library handles automatically:**
- Distributed locking (replaces `@SchedulerLock`)
- Multi-tenant iteration (replaces `runForeachTenantContext`)
- Context setup (PfContextHolder, TenantContextHolder)
- Per-tenant retry on failure
- Execution history and visibility

---

## 3. Integration with Existing Infrastructure

### 3.1 TenantContextHolder Integration

We will use the existing `TenantContextHolder` internally:

```
+------------------------------------------------------------------+
|                    MULTI-TENANT FLOW                              |
+------------------------------------------------------------------+

Library internally calls:

TenantContextHolder.runForeachTenantContext(
    PfContextBuilder.builder()
        .username("scheduled")
        .clientChannel(ClientChannel.Other),
    () -> {
        // Library sets up tracking
        // Library invokes user's @ScheduledJob method
        // Library handles retry on failure
        // Library logs execution status
    }
);
```

### 3.2 What We Wrap vs What We Use

```
+------------------------------------------------------------------+
|  EXISTING COMPONENT        |  OUR APPROACH                       |
+------------------------------------------------------------------+
|                            |                                      |
|  TenantContextHolder       |  USE AS-IS (internal call)          |
|  PfContextHolder           |  USE AS-IS (accessed by user)       |
|  PfContextBuilder          |  USE AS-IS (internal call)          |
|  Spring @Scheduled         |  WRAP (user doesn't use directly)   |
|  ShedLock @SchedulerLock   |  REPLACE (our own locking)          |
|                            |                                      |
+------------------------------------------------------------------+
```

---

## 4. High-Level Architecture

```
+------------------------------------------------------------------+
|                    MICROSERVICE                                   |
+------------------------------------------------------------------+
|                                                                   |
|   @ScheduledJob(name = "summary", cron = "...")                  |
|   public void process(JobContext ctx) {                          |
|       // Pure business logic                                      |
|   }                                                               |
|                                                                   |
+------------------------------------------------------------------+
                               |
                               v
+------------------------------------------------------------------+
|              FIRISBE-SCHEDULER-STARTER (Our Library)              |
+------------------------------------------------------------------+
|                                                                   |
|  +------------------+                                             |
|  | Job Registry     |  Discovers @ScheduledJob at startup        |
|  +------------------+                                             |
|           |                                                       |
|           v                                                       |
|  +------------------+  +------------------+                       |
|  | Cron Trigger     |  | Distributed Lock |                      |
|  | (Spring internal)|  | (Our own)        |                      |
|  +------------------+  +------------------+                       |
|           |                    |                                  |
|           v                    v                                  |
|  +--------------------------------------------------+            |
|  |              Execution Engine                     |            |
|  +--------------------------------------------------+            |
|           |                                                       |
|           v                                                       |
|  +--------------------------------------------------+            |
|  |  TenantContextHolder.runForeachTenantContext()   |            |
|  |  (EXISTING INFRASTRUCTURE - we call it)          |            |
|  +--------------------------------------------------+            |
|           |                                                       |
|           +---> Tenant A ---> Execute + Retry if needed          |
|           +---> Tenant B ---> Execute + Retry if needed          |
|           +---> Tenant C ---> Execute + Retry if needed          |
|           |                                                       |
|           v                                                       |
|  +------------------+  +------------------+                       |
|  | State Manager    |  | Retry Engine     |                      |
|  | (DB Persistence) |  | (Per-tenant)     |                      |
|  +------------------+  +------------------+                       |
|                                                                   |
+------------------------------------------------------------------+
```

---

## 5. Core Components

### 5.1 Component Overview

```
+------------------------------------------------------------------+
|                       LIBRARY COMPONENTS                          |
+------------------------------------------------------------------+

1. JOB REGISTRY
   - Scans @ScheduledJob annotations at startup
   - Stores job metadata (name, cron, retry config)
   - Validates job definitions

2. CRON TRIGGER
   - Uses Spring @Scheduled internally
   - Triggers jobs based on cron expressions
   - Checks overlapping execution policy

3. DISTRIBUTED LOCK (Replaces ShedLock)
   - Database-based locking
   - Prevents duplicate execution across pods
   - Automatic lock release on failure/timeout

4. EXECUTION ENGINE
   - Orchestrates the entire job execution
   - Calls TenantContextHolder.runForeachTenantContext()
   - Tracks execution state per tenant

5. TENANT EXECUTOR
   - Executes job for single tenant
   - Wraps user's method invocation
   - Handles retry on failure

6. RETRY ENGINE
   - Per-tenant retry logic
   - Configurable retry policy
   - Exponential backoff

7. STATE MANAGER
   - Persists execution state to database
   - Tracks success/failure per tenant
   - Provides execution history
```

---

## 6. Execution Flow

### 6.1 Complete Execution Flow

```
CRON TRIGGERS (02:00 AM)
        |
        v
+------------------+
| Distributed Lock |  Attempt to acquire lock "job-vpos-summary"
+------------------+
        |
        +--- Lock exists, not expired ---> SKIP (another pod running)
        |
        +--- Lock acquired
        |
        v
+------------------+
| State Manager    |  Create execution record
+------------------+  status = RUNNING
        |
        v
+------------------------------------------------------------------+
|  TenantContextHolder.runForeachTenantContext(                    |
|      PfContextBuilder.builder()                                   |
|          .username("scheduled")                                   |
|          .clientChannel(ClientChannel.Other),                     |
|      () -> {                                                      |
|          // For each tenant:                                      |
|          executeTenantWithRetry(currentTenant)                   |
|      }                                                            |
|  );                                                               |
+------------------------------------------------------------------+
        |
        v
+------------------+
| State Manager    |  Update execution record
+------------------+  status = SUCCESS/PARTIAL_FAILURE/FAILED
        |
        v
+------------------+
| Distributed Lock |  Release lock
+------------------+
```

### 6.2 Per-Tenant Execution with Retry

```
TENANT EXECUTION (Tenant A)
        |
        v
+------------------+
| State Manager    |  Create tenant_execution record
+------------------+  status = RUNNING, attempt = 1
        |
        v
+------------------+
| Context Check    |  PfContextHolder already set by
+------------------+  TenantContextHolder.runForeachTenantContext()
        |
        v
+------------------+
| Build JobContext |  ctx.tenantId = current tenant
+------------------+  ctx.beginDate = yesterday start
        |            ctx.endDate = yesterday end
        v
+------------------+
| Invoke User      |  Call @ScheduledJob method
| Method           |  summaryScheduler(ctx)
+------------------+
        |
        +--- Success
        |       |
        |       v
        |   Update status = SUCCESS
        |   Continue to next tenant
        |
        +--- Exception
                |
                v
        +------------------+
        | Exception        |  Is it NonRetryable?
        | Classifier       |
        +------------------+
                |
        +-------+-------+
        |               |
    RETRYABLE      NON-RETRYABLE
        |               |
        v               v
+------------------+  Mark FAILED
| Retry Engine     |  Log error
+------------------+  Continue next tenant
        |
        v
  attempt < max?
        |
  +-----+-----+
  |           |
 YES          NO
  |           |
  v           v
Wait       Mark FAILED
(backoff)  Continue next
  |
  v
Retry from "Invoke User Method"
```

---

## 7. JobContext - What User Receives

```java
public class JobContext {
    
    // Tenant Information (from PfContextHolder)
    private String tenantId;
    private String username;        // "scheduled"
    private ClientChannel channel;  // Other
    
    // Date Range (calculated by library)
    private OffsetDateTime beginDate;  // Yesterday 00:00:00 UTC
    private OffsetDateTime endDate;    // Yesterday 23:59:59 UTC
    
    // Execution Metadata
    private String executionId;
    private int attemptNumber;
    
    // Access to original context if needed
    public PfContext getPfContext() {
        return PfContextHolder.get();
    }
}
```

Developer can access everything they need through `JobContext`:

```java
@ScheduledJob(name = "summary", cron = "0 0 2 * * *")
public void process(JobContext ctx) {
    log.info("Processing tenant: {}", ctx.getTenantId());
    
    List<SummaryEntity> data = dataService.getSummary(
        ctx.getBeginDate(), 
        ctx.getEndDate()
    );
    
    summaryService.saveAll(data);
}
```

---

## 8. Database Schema

```
+------------------------------------------------------------------+
|                      DATABASE TABLES                              |
+------------------------------------------------------------------+

TABLE: scheduler_job
─────────────────────
| Column          | Type         | Description                     |
|-----------------|--------------|----------------------------------|
| id              | VARCHAR(64)  | Primary key (job name)          |
| cron_expression | VARCHAR(100) | Cron schedule                   |
| enabled         | BOOLEAN      | Is job enabled                  |
| retry_attempts  | INT          | Max retry attempts              |
| created_at      | TIMESTAMP    | Creation time                   |
| updated_at      | TIMESTAMP    | Last update time                |


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
| started_at      | TIMESTAMP    | Execution start time            |
| finished_at     | TIMESTAMP    | Execution end time              |


TABLE: scheduler_tenant_execution
─────────────────────────────────
| Column          | Type         | Description                     |
|-----------------|--------------|----------------------------------|
| id              | UUID         | Primary key                     |
| execution_id    | UUID         | FK to scheduler_execution       |
| tenant_id       | VARCHAR(64)  | Tenant identifier               |
| status          | VARCHAR(20)  | RUNNING/SUCCESS/FAILED          |
| attempt_count   | INT          | Current attempt number          |
| started_at      | TIMESTAMP    | Tenant execution start          |
| finished_at     | TIMESTAMP    | Tenant execution end            |
| error_message   | TEXT         | Error details if failed         |


TABLE: scheduler_lock
─────────────────────
| Column          | Type         | Description                     |
|-----------------|--------------|----------------------------------|
| lock_name       | VARCHAR(64)  | Primary key (job name)          |
| locked_by       | VARCHAR(100) | Pod/instance identifier         |
| locked_at       | TIMESTAMP    | Lock acquisition time           |
| expires_at      | TIMESTAMP    | Lock expiry time                |
```

---

## 9. Configuration

```yaml
firisbe:
  scheduler:
    enabled: true
    
    # Context configuration
    context:
      username: "scheduled"
      client-channel: Other
      
    # Date range (what period to process)
    date-range:
      type: YESTERDAY          # YESTERDAY, TODAY, CUSTOM
      offset-days: -1          # For CUSTOM type
      
    # Retry configuration
    retry:
      max-attempts: 3
      initial-interval: 5s
      max-interval: 1m
      backoff-multiplier: 2.0
      
    # Lock configuration
    lock:
      timeout: 60m             # Max lock hold time
      
    # Tenant processing
    tenant:
      continue-on-failure: true  # Continue with other tenants if one fails
```

---

## 10. Package Structure

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
|
+-- execution/
|   +-- ExecutionEngine.java
|   +-- TenantExecutor.java
|   +-- JobInvoker.java
|
+-- retry/
|   +-- RetryEngine.java
|   +-- RetryPolicy.java
|   +-- ExceptionClassifier.java
|
+-- lock/
|   +-- DistributedLockManager.java
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

## 11. Migration Example

### Before (Current Code)

```java
@Scheduled(cron = "${vpos-summary.scheduled.cron}")
@SchedulerLock(name = "scheduled_vpos_summary", lockAtMostFor = "60m", lockAtLeastFor = "60m")
public void summaryScheduler() {
    LocalDate localDate = LocalDate.now().minusDays(1);
    OffsetDateTime beginDate = localDate.atTime(LocalTime.MIN).atOffset(ZoneOffset.UTC);
    OffsetDateTime endDate = localDate.atTime(LocalTime.MAX).atOffset(ZoneOffset.UTC);

    TenantContextHolder.runForeachTenantContext(
        PfContextBuilder.builder().username("scheduled").clientChannel(ClientChannel.Other), 
        () -> {
            List<SummaryEntity> dataList = vPosTransactionDataService
                .getTransactionSummary(beginDate, endDate);
            dataList.forEach(k -> k.setDate(beginDate));
            summaryDataService.saveAll(dataList);
        }
    );
}
```

### After (With Our Library)

```java
@ScheduledJob(name = "vpos-summary", cron = "${vpos-summary.scheduled.cron}")
public void summaryScheduler(JobContext ctx) {
    List<SummaryEntity> dataList = vPosTransactionDataService
        .getTransactionSummary(ctx.getBeginDate(), ctx.getEndDate());
    dataList.forEach(k -> k.setDate(ctx.getBeginDate()));
    summaryDataService.saveAll(dataList);
}
```

**Lines of code reduced:** 15 → 5 (67% reduction)
**Boilerplate eliminated:** Locking, tenant iteration, date calculation, context setup

---

## 12. Next: Detailed Component Design

- [ ] Section 13: Job Registry Design
- [ ] Section 14: Distributed Lock Design
- [ ] Section 15: Execution Engine Design
- [ ] Section 16: Retry Engine Design
- [ ] Section 17: State Manager Design

---
