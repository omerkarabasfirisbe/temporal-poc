# Research: Current Scheduler Analysis & Temporal Migration Path

**Purpose:** This document analyzes the existing scheduler implementation and explores which Temporal features can replace or enhance the current approach.

**Context:** Based on the VPos Summary Scheduler implementation in `vposgw-srv` module.

---

## Current Implementation Overview

The VPos Summary Scheduler is a daily job that calculates and stores transaction summaries for all tenants.

**Technologies Used:**
- Spring `@Scheduled` with externalized cron expression for timing
- ShedLock (`@SchedulerLock`) for distributed locking across pods (60 minute lock)
- `TenantContextHolder.runForeachTenantContext()` for multi-tenant iteration

**Execution Flow:**
1. Job triggers based on cron schedule
2. ShedLock acquires distributed lock
3. Iterates through all tenants sequentially
4. For each tenant: fetches yesterday's transactions, calculates summary, saves to database

**Current Limitations:**
- All-or-Nothing: If job fails at Tenant 50 of 100, remaining tenants are skipped
- No Visibility: Cannot see which tenant is processing or failed without searching logs
- Sequential Only: No parallel processing, long execution time for many tenants
- Lock Complexity: Requires lock table, fixed duration may be too short or long

---

## Temporal Features for Migration

### 1. Schedules (Replaces @Scheduled + Cron)

Temporal Schedules provide a more powerful alternative to Spring @Scheduled:

**Current Approach:**
```java
@Scheduled(cron = "${vpos-summary.scheduled.cron}")
```

**Temporal Approach:**
```java
// Creating a Schedule
ScheduleClient scheduleClient = ScheduleClient.newInstance(WorkflowServiceStubs.newInstance());

Schedule schedule = Schedule.newBuilder()
    .setAction(ScheduleActionStartWorkflow.newBuilder()
        .setWorkflowType(VPosSummaryWorkflow.class)
        .setOptions(WorkflowOptions.newBuilder()
            .setTaskQueue("vpos-summary-queue")
            .build())
        .build())
    .setSpec(ScheduleSpec.newBuilder()
        .setCronExpressions(List.of("0 0 2 * * *")) // Daily at 2 AM
        .build())
    .build();

scheduleClient.createSchedule("vpos-summary-schedule", schedule, ScheduleOptions.newBuilder().build());
```

**Benefits:**
- No external cron daemon needed
- Schedule state is durable (survives restarts)
- Can pause, resume, backfill, and trigger manually via UI or API
- Full visibility into schedule execution history

---

### 2. Built-in Distributed Locking (Replaces ShedLock)

Temporal provides automatic distributed execution without external locking:

**Current Approach:**
```java
@SchedulerLock(name = "scheduled_vpos_summary", lockAtMostFor = "60m", lockAtLeastFor = "60m")
```

**Temporal Approach:**
- No explicit locking needed
- Workflow ID uniqueness guarantees single execution
- If a Workflow with the same ID is already running, new execution is rejected or queued

```java
WorkflowOptions options = WorkflowOptions.newBuilder()
    .setWorkflowId("vpos-summary-" + LocalDate.now()) // Unique per day
    .setTaskQueue("vpos-summary-queue")
    .setWorkflowIdReusePolicy(WorkflowIdReusePolicy.WORKFLOW_ID_REUSE_POLICY_REJECT_DUPLICATE)
    .build();
```

**Benefits:**
- No database table required for locks
- No lock expiry issues
- Automatic handling of pod crashes

---

### 3. Child Workflows (Replaces runForeachTenantContext)

For multi-tenant processing, Temporal Child Workflows provide isolation and parallelism:

**Current Approach:**
```java
TenantContextHolder.runForeachTenantContext(context, () -> {
    // Process single tenant
});
```

**Temporal Approach:**
```java
// Parent Workflow
@WorkflowInterface
public interface VPosSummaryWorkflow {
    @WorkflowMethod
    void processSummary(LocalDate date);
}

public class VPosSummaryWorkflowImpl implements VPosSummaryWorkflow {
    @Override
    public void processSummary(LocalDate date) {
        List<String> tenantIds = activities.getAllTenantIds();
        
        List<Promise<Void>> childPromises = new ArrayList<>();
        for (String tenantId : tenantIds) {
            // Start child workflow for each tenant (parallel execution)
            TenantSummaryWorkflow child = Workflow.newChildWorkflowStub(
                TenantSummaryWorkflow.class,
                ChildWorkflowOptions.newBuilder()
                    .setWorkflowId("summary-" + date + "-" + tenantId)
                    .build()
            );
            childPromises.add(Workflow.async(child::processTenant, tenantId, date));
        }
        
        // Wait for all tenants to complete
        Promise.allOf(childPromises).get();
    }
}

// Child Workflow (per tenant)
@WorkflowInterface
public interface TenantSummaryWorkflow {
    @WorkflowMethod
    void processTenant(String tenantId, LocalDate date);
}
```

**Benefits:**
- Each tenant has its own workflow with independent retry policy
- Tenant A failure does not affect Tenant B
- Parallel processing possible
- Full visibility per tenant in Temporal UI

---

### 4. Activities with Retry (Replaces Manual Error Handling)

**Current Approach:**
- No explicit retry logic
- If database call fails, entire job fails

**Temporal Approach:**
```java
@ActivityInterface
public interface SummaryActivities {
    
    @ActivityMethod
    List<SummaryEntity> getTransactionSummary(String tenantId, OffsetDateTime begin, OffsetDateTime end);
    
    @ActivityMethod
    void saveSummaryData(String tenantId, List<SummaryEntity> data);
}

// Activity options with retry
ActivityOptions options = ActivityOptions.newBuilder()
    .setStartToCloseTimeout(Duration.ofMinutes(5))
    .setRetryOptions(RetryOptions.newBuilder()
        .setInitialInterval(Duration.ofSeconds(1))
        .setMaximumInterval(Duration.ofMinutes(1))
        .setBackoffCoefficient(2.0)
        .setMaximumAttempts(5)
        .build())
    .build();
```

**Benefits:**
- Automatic retry with exponential backoff
- Configurable per activity
- No try-catch clutter in business logic

---

### 5. Task Queues (Multi-Tenant Isolation)

**Current Approach:**
- All tenants share same execution context
- No resource isolation

**Temporal Approach:**
```java
// Option A: Separate queue per tenant
String taskQueue = "summary-queue-" + tenantId;

// Option B: Single queue with workflow-level isolation
WorkflowOptions.newBuilder()
    .setTaskQueue("summary-queue")
    .setWorkflowId("summary-" + tenantId + "-" + date)
    .build();
```

**Benefits:**
- Tenant A heavy load does not block Tenant B
- Can scale workers per tenant if needed
- Clear visibility and metrics per queue

---

### 6. Signals (Event-Driven Triggering)

**Use Case:** Trigger summary calculation on-demand, not just on schedule

**Temporal Approach:**
```java
@WorkflowInterface
public interface VPosSummaryWorkflow {
    @WorkflowMethod
    void run();
    
    @SignalMethod
    void triggerManualRun(LocalDate date);
}
```

**Benefits:**
- Can trigger workflow from external event
- Supports hybrid approach: scheduled + event-driven

---

## Mapping: Current vs Temporal

**Spring @Scheduled + Cron**
- Temporal Equivalent: Schedules API
- Benefit: Durable, observable, controllable

**ShedLock (@SchedulerLock)**
- Temporal Equivalent: Workflow ID uniqueness
- Benefit: No external dependency, no lock table

**TenantContextHolder.runForeachTenantContext**
- Temporal Equivalent: Child Workflows
- Benefit: Parallel execution, per-tenant visibility

**Manual try-catch for retries**
- Temporal Equivalent: Activity RetryOptions
- Benefit: Declarative, automatic

**Log-based debugging**
- Temporal Equivalent: Temporal UI + Query API
- Benefit: Real-time visibility, full history

---

## Research Backlog

- [ ] How to pass tenant context (PfContextHolder) into Temporal Workflow/Activity
- [ ] Integration with existing PfContextBuilder
- [ ] Performance comparison: Sequential vs Parallel tenant processing
- [ ] Temporal Server deployment options (self-hosted vs cloud)
- [ ] Migration strategy: Gradual rollout vs Big Bang

---

## Proposed Architecture

```
                     Temporal Schedule
                           |
                           v
                 +-------------------+
                 | VPosSummaryWorkflow|
                 | (Parent)           |
                 +-------------------+
                           |
         +-----------------+-----------------+
         |                 |                 |
         v                 v                 v
+---------------+  +---------------+  +---------------+
| TenantWorkflow|  | TenantWorkflow|  | TenantWorkflow|
| (tenant-a)    |  | (tenant-b)    |  | (tenant-c)    |
+---------------+  +---------------+  +---------------+
         |                 |                 |
         v                 v                 v
   +----------+      +----------+      +----------+
   | Activity |      | Activity |      | Activity |
   | DB Query |      | DB Query |      | DB Query |
   +----------+      +----------+      +----------+
         |                 |                 |
         v                 v                 v
   +----------+      +----------+      +----------+
   | Activity |      | Activity |      | Activity |
   | DB Save  |      | DB Save  |      | DB Save  |
   +----------+      +----------+      +----------+
```

---

## References

- Temporal Schedules: https://docs.temporal.io/workflows#schedule
- Temporal Java SDK: https://github.com/temporalio/sdk-java
- Activity Retry Options: https://docs.temporal.io/activities#retries
- Child Workflows: https://docs.temporal.io/workflows#child-workflow

---
