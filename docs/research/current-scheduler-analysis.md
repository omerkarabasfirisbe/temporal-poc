# Research: Current Scheduler Analysis & Parent Library Design

**Purpose:** This document analyzes the existing scheduler implementation and identifies which Temporal features should be wrapped in our custom Parent Library.

**Context:** Based on the VPos Summary Scheduler implementation and the "Parent Library" concept discussed in the Gorkem & Omer meeting.

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

## Parent Library Concept

The goal is not to directly integrate Temporal into every microservice. Instead, we will create a **shared Parent Library** that:

1. Wraps Temporal SDK complexity
2. Provides simple annotations/interfaces for service developers
3. Handles tenant context, retry logic, and error handling automatically
4. Allows microservice developers to focus only on business logic

**Target Developer Experience:**
```java
// Developer only writes this - no Temporal knowledge required
@ScheduledJob(cron = "0 0 2 * * *", name = "vpos-summary")
public void processSummary(TenantContext tenant, LocalDate date) {
    List<SummaryEntity> data = dataService.getTransactionSummary(date);
    summaryService.saveAll(data);
}
```

The Parent Library handles everything else: scheduling, locking, tenant iteration, retries, and visibility.

---

## Temporal Features to Wrap in Parent Library

This section identifies which Temporal features we should leverage internally, hidden from microservice developers.

### 1. Schedules API

**What Temporal Provides:**
- Durable cron-like scheduling
- Pause, resume, backfill capabilities
- Schedule execution history

**What Parent Library Should Expose:**
- Simple `@ScheduledJob(cron = "...")` annotation
- Internal translation to Temporal Schedule
- Automatic schedule registration on application startup

**Internal Responsibility:**
- Create and manage Temporal Schedules
- Handle schedule updates when cron changes
- Provide admin API for pause/resume if needed

---

### 2. Workflow ID Uniqueness (Distributed Locking)

**What Temporal Provides:**
- Single execution guarantee via Workflow ID
- Configurable reuse policies
- No external lock table needed

**What Parent Library Should Expose:**
- Nothing explicit - locking should be automatic
- Job name becomes Workflow ID prefix

**Internal Responsibility:**
- Generate unique Workflow IDs per job + date/tenant
- Configure `WorkflowIdReusePolicy` appropriately
- Handle "already running" scenarios gracefully

---

### 3. Tenant Context Propagation

**What Temporal Provides:**
- Workflow input parameters
- Activity context passing

**What Parent Library Should Expose:**
- `TenantContext` as method parameter (injected automatically)
- Integration with existing `PfContextHolder` and `TenantContextHolder`

**Internal Responsibility:**
- Fetch all tenant IDs at job start
- Create child workflow or activity per tenant
- Set `PfContextHolder` before calling business logic
- Clear context after execution

---

### 4. Child Workflows for Tenant Isolation

**What Temporal Provides:**
- Independent execution per child
- Parallel or sequential execution options
- Per-child retry and visibility

**What Parent Library Should Expose:**
- Configuration option: `parallel = true/false`
- Per-tenant failure does not affect others (automatic)

**Internal Responsibility:**
- Create parent workflow that orchestrates
- Spawn child workflow per tenant
- Aggregate results and report failures

---

### 5. Activity Retry Options

**What Temporal Provides:**
- Configurable retry policies (attempts, backoff, intervals)
- Automatic retry execution
- Non-retryable exception handling

**What Parent Library Should Expose:**
- Annotation-based retry config: `@Retryable(maxAttempts = 5)`
- Or global defaults via configuration

**Internal Responsibility:**
- Translate annotations to `RetryOptions`
- Apply sensible defaults
- Handle exception classification (retryable vs non-retryable)

---

### 6. Visibility and Monitoring

**What Temporal Provides:**
- Temporal UI for workflow inspection
- Query API for programmatic access
- Full execution history

**What Parent Library Should Expose:**
- Optional: REST endpoint for job status
- Logging integration with existing infrastructure

**Internal Responsibility:**
- Configure Temporal client for metrics export
- Optionally expose simplified status API

---

## Features NOT to Expose Initially

Some Temporal features are powerful but add complexity. For the first version of Parent Library, we should NOT expose:

- **Signals:** Event-driven triggering adds complexity
- **Queries:** Real-time workflow state inspection
- **Continue-As-New:** Long-running workflow patterns
- **Saga/Compensation:** Complex rollback scenarios

These can be added in future versions as needed.

---

## Parent Library Components

Based on the above analysis, the Parent Library should contain:

**1. Annotation Processor**
- `@ScheduledJob` - defines a scheduled job
- `@Retryable` - configures retry behavior (optional)

**2. Temporal Client Wrapper**
- Abstracts Temporal SDK setup
- Manages connection lifecycle

**3. Schedule Manager**
- Registers jobs as Temporal Schedules on startup
- Handles schedule CRUD operations

**4. Tenant Orchestrator**
- Fetches tenant list
- Creates child workflows per tenant
- Manages context propagation

**5. Activity Executor**
- Wraps business logic as Temporal Activity
- Applies retry policies
- Handles context setup/teardown

---

## Research Backlog

- [ ] How to integrate with existing `PfContextHolder` and `PfContextBuilder`
- [ ] Annotation processing approach: compile-time vs runtime
- [ ] Temporal client configuration (connection pooling, timeouts)
- [ ] Error reporting integration with existing alerting system
- [ ] Gradual migration strategy for existing scheduled jobs

---

## References

- Temporal Schedules: https://docs.temporal.io/workflows#schedule
- Temporal Java SDK: https://github.com/temporalio/sdk-java
- Activity Retry Options: https://docs.temporal.io/activities#retries
- Child Workflows: https://docs.temporal.io/workflows#child-workflow

---
