# Open Questions: Error Handling Strategy

**Status:** Needs Discussion
**Created:** 2025-12-26

---

## Overview

This document lists open questions regarding error handling in the scheduler library. These need to be discussed and resolved before implementation.

---

## Scenario 1: Single Tenant Failure

**Situation:**
```
Tenant A → Success
Tenant B → FAIL (Database timeout)
Tenant C → Success
Tenant D → Success
```

**Questions:**

1. Should we retry Tenant B immediately?
2. How many retry attempts before giving up?
3. Should we continue with other tenants (C, D) while Tenant B is retrying?
4. Should retry happen synchronously (block other tenants) or asynchronously?

---

## Scenario 2: Retry Exhaustion

**Situation:**
```
Tenant B → Attempt 1 → FAIL
         → Attempt 2 → FAIL
         → Attempt 3 → FAIL (max attempts reached)
```

**Questions:**

1. What happens after max retries are exhausted?
2. Should we send an alert/notification? To whom? How?
3. Should we retry again on the next scheduled run?
4. Should there be a manual retry mechanism (API endpoint)?
5. How do we track failed tenants for later investigation?

---

## Scenario 3: Exception Classification

**Situation:**
```
Tenant A → IOException (Network issue)
Tenant B → NullPointerException (Bug in code)
Tenant C → BusinessValidationException (Invalid data)
```

**Questions:**

1. How do we classify exceptions as retryable vs non-retryable?
2. Default classifications:
   - Network errors → Retryable?
   - Database errors → Retryable?
   - NullPointerException → Non-retryable (bug)?
   - Business exceptions → Non-retryable?
3. Should developers be able to override classification via annotation?
4. Example: `@NonRetryable(BusinessValidationException.class)`

---

## Scenario 4: Complete Job Failure

**Situation:**
```
Lock acquisition failed (another pod crashed with lock)
OR
Database is completely down
OR
TenantContextHolder.runForeachTenantContext() throws exception
```

**Questions:**

1. How do we handle infrastructure-level failures?
2. Should we implement circuit breaker pattern?
3. What alerts should be triggered for critical failures?
4. Should the job be marked as "needs attention" for manual review?

---

## Scenario 5: Partial Success

**Situation:**
```
Total tenants: 100
Success: 95
Failed: 5
```

**Questions:**

1. What is the overall job status? SUCCESS or PARTIAL_FAILURE?
2. Should partial failure trigger an alert?
3. How do we report which tenants failed?
4. Is there a threshold (e.g., >10% failure = critical alert)?

---

## Scenario 6: Long Running Tenant

**Situation:**
```
Tenant A → Takes 5 minutes (very large dataset)
Tenant B → Takes 1 second
...
Lock timeout is 60 minutes
```

**Questions:**

1. Should there be a timeout per tenant execution?
2. What happens if tenant execution exceeds timeout?
3. Should slow tenants be logged/alerted?

---

## Scenario 7: Retry Backoff Strategy

**Questions:**

1. What backoff strategy should we use?
   - Fixed interval (e.g., always wait 5 seconds)
   - Exponential backoff (1s, 2s, 4s, 8s...)
   - Exponential with jitter (randomization)
   
2. What should be the default values?
   - Initial interval: ?
   - Max interval: ?
   - Backoff multiplier: ?
   - Max attempts: ?

---

## Scenario 8: Alerting Integration

**Questions:**

1. How should alerts be sent?
   - Existing alerting system integration?
   - Log-based alerts (log level ERROR/CRITICAL)?
   - Webhook/notification service?
   
2. What information should be included in alerts?
   - Job name
   - Tenant ID
   - Error message
   - Stack trace?
   - Attempt count

---

## Scenario 9: Execution History and Audit

**Questions:**

1. How long should we keep execution history?
2. Should failed executions be kept longer than successful ones?
3. Should there be an API to query execution history?
4. Should we provide a simple dashboard/UI for monitoring?

---

## Summary: Key Decisions Needed

| Question | Options | Decision |
|----------|---------|----------|
| Continue on tenant failure? | Yes / No | ? |
| Retry strategy | Fixed / Exponential / With jitter | ? |
| Default max attempts | 1 / 3 / 5 / Configurable | ? |
| Alert on partial failure? | Yes / No / Threshold-based | ? |
| Manual retry API? | Yes / No | ? |
| Per-tenant timeout? | Yes / No | ? |
| Exception classification | Annotation / Config / Both | ? |

---
