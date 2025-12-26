# Architecture: Temporal Parent Library

**Document Status:** Draft
**Last Updated:** 2025-12-26

---

## 1. Overview

The Parent Library is a Java library that abstracts Temporal SDK complexity and provides a simple, annotation-driven interface for scheduled job execution in a multi-tenant environment.

```
+------------------------------------------------------------------+
|                        Microservice                               |
|  +------------------------------------------------------------+  |
|  |                    Business Logic                           |  |
|  |  @ScheduledJob(cron = "0 0 2 * * *")                       |  |
|  |  public void processData(TenantContext ctx) { ... }         |  |
|  +------------------------------------------------------------+  |
|                              |                                    |
|  +------------------------------------------------------------+  |
|  |                  PARENT LIBRARY                             |  |
|  |  +------------------+  +------------------+                 |  |
|  |  | Annotation       |  | Schedule         |                 |  |
|  |  | Processor        |  | Manager          |                 |  |
|  |  +------------------+  +------------------+                 |  |
|  |  +------------------+  +------------------+                 |  |
|  |  | Tenant           |  | Activity         |                 |  |
|  |  | Orchestrator     |  | Executor         |                 |  |
|  |  +------------------+  +------------------+                 |  |
|  |  +------------------------------------------+               |  |
|  |  |           Temporal Client Wrapper        |               |  |
|  |  +------------------------------------------+               |  |
|  +------------------------------------------------------------+  |
+------------------------------------------------------------------+
                              |
                              v
                    +-------------------+
                    |  Temporal Server  |
                    +-------------------+
```

---

## 2. Component Design

### 2.1 Annotation Processor

**Responsibility:** Discover and register scheduled jobs at application startup.

**Input:**
- Classes annotated with `@ScheduledJob`
- Method signatures and parameters

**Output:**
- Job metadata (name, cron, retry config)
- Method references for invocation

**Key Classes:**
```
com.firisbe.scheduler.annotation
├── @ScheduledJob           # Main annotation
├── @Retryable              # Optional retry config
└── ScheduledJobRegistry    # Stores discovered jobs
```

### 2.2 Schedule Manager

**Responsibility:** Create and manage Temporal Schedules.

**Operations:**
- Register schedules on startup
- Update schedules when configuration changes
- Provide API for pause/resume/trigger

**Key Classes:**
```
com.firisbe.scheduler.schedule
├── ScheduleManager         # Main entry point
├── ScheduleRegistrar       # Startup registration
└── ScheduleOperations      # CRUD operations
```

### 2.3 Tenant Orchestrator

**Responsibility:** Execute jobs across all tenants with isolation.

**Flow:**
1. Receive job trigger from Schedule
2. Fetch list of active tenants
3. Create child workflow per tenant
4. Wait for all to complete (or fail independently)
5. Report aggregated results

**Key Classes:**
```
com.firisbe.scheduler.tenant
├── TenantOrchestrator      # Main workflow
├── TenantJobExecutor       # Per-tenant execution
└── TenantContextBridge     # PfContextHolder integration
```

### 2.4 Activity Executor

**Responsibility:** Wrap business logic as Temporal Activity.

**Operations:**
- Set up tenant context before execution
- Invoke actual business method
- Handle exceptions and classify them
- Clean up context after execution

**Key Classes:**
```
com.firisbe.scheduler.activity
├── JobActivityImpl         # Activity implementation
├── ContextSetup            # Context initialization
└── ExceptionClassifier     # Retryable vs non-retryable
```

### 2.5 Temporal Client Wrapper

**Responsibility:** Abstract Temporal SDK connection and configuration.

**Features:**
- Connection lifecycle management
- Configuration via properties
- Health check support

**Key Classes:**
```
com.firisbe.scheduler.temporal
├── TemporalClientFactory   # Client creation
├── TemporalProperties      # Configuration
└── TemporalHealthIndicator # Health check
```

---

## 3. Workflow Design

### 3.1 Parent Workflow

The main workflow that orchestrates tenant execution.

```
ParentWorkflow
    |
    +-- Activity: FetchTenantList
    |
    +-- For each tenant (parallel or sequential):
    |       |
    |       +-- ChildWorkflow: TenantJobWorkflow
    |               |
    |               +-- Activity: SetupContext
    |               +-- Activity: ExecuteJob
    |               +-- Activity: CleanupContext
    |
    +-- Activity: ReportResults
```

### 3.2 Workflow IDs

**Format:** `{job-name}-{date}-{run-id}`

**Examples:**
- `vpos-summary-2025-12-26-scheduled`
- `vpos-summary-2025-12-26-manual-abc123`

**Child Workflow ID:** `{parent-id}-{tenant-id}`

---

## 4. Package Structure

```
com.firisbe.scheduler
├── annotation/
│   ├── ScheduledJob.java
│   ├── Retryable.java
│   └── EnableScheduler.java
├── config/
│   ├── SchedulerAutoConfiguration.java
│   ├── SchedulerProperties.java
│   └── TemporalConfiguration.java
├── core/
│   ├── JobRegistry.java
│   ├── JobMetadata.java
│   └── JobExecutionContext.java
├── schedule/
│   ├── ScheduleManager.java
│   ├── ScheduleRegistrar.java
│   └── ScheduleController.java (optional REST API)
├── tenant/
│   ├── TenantOrchestrator.java
│   ├── TenantProvider.java (interface)
│   └── TenantContextBridge.java
├── workflow/
│   ├── ParentJobWorkflow.java
│   ├── ParentJobWorkflowImpl.java
│   ├── TenantJobWorkflow.java
│   └── TenantJobWorkflowImpl.java
├── activity/
│   ├── JobActivity.java
│   ├── JobActivityImpl.java
│   └── TenantActivity.java
└── exception/
    ├── SchedulerException.java
    ├── NonRetryableException.java
    └── TenantExecutionException.java
```

---

## 5. Configuration

### 5.1 Application Properties

```yaml
firisbe:
  scheduler:
    temporal:
      server-address: localhost:7233
      namespace: default
      task-queue: scheduler-queue
    defaults:
      retry-attempts: 3
      retry-interval: 1s
      retry-max-interval: 1m
      retry-backoff: 2.0
    tenant:
      parallel: false
      max-parallelism: 10
```

### 5.2 Environment Variables

```
TEMPORAL_SERVER_ADDRESS=temporal:7233
TEMPORAL_NAMESPACE=production
SCHEDULER_RETRY_ATTEMPTS=5
```

---

## 6. Integration Points

### 6.1 Tenant Provider Interface

Microservices must implement this interface:

```java
public interface TenantProvider {
    List<String> getActiveTenantIds();
}
```

### 6.2 Context Bridge

Integration with existing context holders:

```java
public interface ContextBridge {
    void setupContext(String tenantId, JobExecutionContext context);
    void clearContext();
}
```

Default implementation uses `PfContextHolder` and `TenantContextHolder`.

---

## 7. Error Handling

### 7.1 Exception Classification

```
Exception
├── NonRetryableException (do not retry)
│   ├── ValidationException
│   └── BusinessRuleException
└── RetryableException (retry automatically)
    ├── NetworkException
    ├── DatabaseException
    └── TemporaryException
```

### 7.2 Failure Reporting

After retry exhaustion:
1. Log error with full context
2. Mark tenant as failed in workflow
3. Continue with remaining tenants
4. Send alert via existing alerting system

---

## 8. Deployment Considerations

### 8.1 Temporal Server

- Must be deployed separately
- Recommended: Temporal Cloud or self-hosted cluster
- Single Temporal cluster can serve multiple microservices

### 8.2 Worker Deployment

- Each microservice runs its own Temporal Worker
- Worker is embedded in the application (Spring Boot starter)
- No separate worker deployment needed

---
