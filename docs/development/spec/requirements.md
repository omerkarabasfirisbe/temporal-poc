# Requirements: Temporal Parent Library

**Document Status:** Draft
**Last Updated:** 2025-12-26

---

## 1. Functional Requirements

### FR-1: Scheduled Job Execution

**FR-1.1:** The library must support cron-based job scheduling.
- Jobs should be defined using a simple annotation
- Cron expressions must be externalizable via configuration

**FR-1.2:** The library must guarantee single execution per scheduled run.
- No duplicate executions across multiple pods
- No external lock table required

**FR-1.3:** The library must support manual job triggering.
- Jobs can be triggered on-demand via API
- Manual runs should not interfere with scheduled runs

### FR-2: Multi-Tenant Support

**FR-2.1:** The library must execute jobs for all active tenants.
- Automatically iterate through tenant list
- Each tenant processed in isolation

**FR-2.2:** The library must propagate tenant context to business logic.
- Integration with existing `PfContextHolder`
- Integration with existing `TenantContextHolder`
- Context must be set before and cleared after execution

**FR-2.3:** Tenant failures must be isolated.
- Failure in Tenant A must not affect Tenant B
- Failed tenants should be retried independently

### FR-3: Retry and Error Handling

**FR-3.1:** The library must provide automatic retry for failed jobs.
- Configurable retry attempts
- Exponential backoff support
- Maximum retry interval configurable

**FR-3.2:** The library must distinguish retryable vs non-retryable errors.
- Network errors: retryable
- Business logic errors: configurable
- Validation errors: non-retryable

**FR-3.3:** The library must report failures after retry exhaustion.
- Integration with existing alerting system
- Detailed error information in logs

### FR-4: Visibility and Monitoring

**FR-4.1:** The library must provide job execution status.
- Currently running jobs
- Recently completed jobs
- Failed jobs with error details

**FR-4.2:** The library must support existing logging infrastructure.
- Correlation ID propagation
- Tenant ID in log context
- Job name in log context

---

## 2. Non-Functional Requirements

### NFR-1: Performance

**NFR-1.1:** Job scheduling overhead must be minimal.
- Less than 100ms added latency per job start

**NFR-1.2:** The library must support parallel tenant processing.
- Configurable parallelism level
- Default: sequential for safety

### NFR-2: Reliability

**NFR-2.1:** Jobs must survive pod restarts.
- In-progress jobs resume after restart
- No manual intervention required

**NFR-2.2:** The library must handle Temporal server unavailability gracefully.
- Connection retry with backoff
- Clear error reporting

### NFR-3: Developer Experience

**NFR-3.1:** Minimal boilerplate for job definition.
- Single annotation to define a job
- No Temporal SDK knowledge required

**NFR-3.2:** Clear error messages for misconfiguration.
- Validation at startup
- Descriptive exception messages

### NFR-4: Operations

**NFR-4.1:** The library must support configuration via environment variables.
- Temporal server address
- Default retry policies
- Parallelism settings

**NFR-4.2:** The library must be deployable without Temporal for local development.
- Mock mode for unit testing
- In-memory execution option

---

## 3. Constraints

**C-1:** Must use Temporal Java SDK (not alternative workflow engines)

**C-2:** Must be compatible with Java 17+

**C-3:** Must be compatible with Spring Boot 3.x

**C-4:** Must not require changes to existing database schema

**C-5:** Library size must be reasonable (avoid unnecessary dependencies)

---

## 4. Assumptions

**A-1:** Temporal Server will be deployed and managed separately

**A-2:** All microservices use the same tenant management system

**A-3:** Existing `PfContextHolder` and `TenantContextHolder` APIs are stable

---

## 5. Out of Scope (v1.0)

- UI for job management (use Temporal UI directly)
- Complex workflow patterns (sagas, compensations)
- Event-driven job triggering (signals)
- Real-time job state queries

---
