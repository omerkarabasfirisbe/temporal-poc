# Research: Temporal Lifecycle & Modern Orchestration

**Purpose:** This document provides a deep dive into Temporal for developers familiar with traditional scheduling (like Spring Scheduler) but new to the concept of Durable Execution.

**Context:** Prepared in alignment with the principles established during the Gorkem & Omer meeting.

---

## What is Temporal?

### The "Video Game" Analogy

Imagine you are playing a complex video game:

**Classic Approach (Spring Scheduler):** It's like a game with no save points. If your console crashes or power goes out while you are on level 10, you have to start the entire game from level 1 again. You lose all your progress.

**Temporal Approach:** It's like a game with auto-save after every single move. If the power goes out, the moment you turn the console back on, you are exactly where you left off. The world state, your items, and your position are perfectly preserved.

### Definition

Temporal is a "Durable Execution" Workflow Engine. It ensures that your code runs to completion, regardless of what happens to the underlying servers, network, or external APIs.

---

## Core Concepts

### A. Workflow (The Brain)

Workflow is a simple function written in your favorite language (Java, Go, etc.). It defines the sequence of steps (business logic).

```java
// Example: Order Processing Workflow
public class OrderWorkflowImpl implements OrderWorkflow {
    
    @Override
    public OrderResult processOrder(Order order) {
        // 1. Charge the customer
        PaymentResult payment = activities.chargeCustomer(order);
        
        // 2. If successful, send welcome email
        if (payment.isSuccessful()) {
            activities.sendWelcomeEmail(order.getCustomerEmail());
        }
        
        // 3. Update the database
        return activities.updateDatabase(order);
    }
}
```

**Key Point:** Temporal makes this function "immortal." If the server running this code dies, Temporal moves the "brain" to another server and continues.

### B. Activity (The Muscles)

Activities are individual tasks that the Workflow calls. They interact with the outside world (Database, REST API, File System).

**Activity Types:**
- **Database Activity:** PostgreSQL, MongoDB
- **API Activity:** REST, gRPC calls
- **File Activity:** File system operations
- **Notification Activity:** Email, SMS sending

```java
@ActivityInterface
public interface PaymentActivities {
    
    @ActivityMethod
    PaymentResult chargeCustomer(Order order);
    
    @ActivityMethod
    void sendWelcomeEmail(String email);
}
```

**Key Point:** Unlike Workflows, Activities are expected to fail (due to network issues, API timeouts). Temporal automatically retries these "muscles" based on rules you define.

---

## Spring @Scheduled vs Temporal

In a small application, `@Scheduled` works fine. But in a professional, multi-tenant system, it becomes a liability.

### I. Durability — The "What if it crashes?" Problem

**Spring @Scheduled:**
- Pod restart while processing 1000 users: Remaining 500 users are ignored.
- Recovery logic: You must write complex "resume" code yourself.
- Flow: User 1 -> User 2 -> User 500 -> CRASH -> User 501-1000 LOST

**Temporal:**
- Pod restart while processing 1000 users: Knows exactly which user it stopped at.
- Recovery logic: Automatic, zero data loss.
- Flow: User 1 -> User 2 -> User 500 -> CRASH -> RECOVER -> User 501 -> ... -> User 1000 (Complete)

### II. State Management — The "Where was I?" Problem

**Spring @Scheduled:**
- State tracking: Manual DB records (`INITIATED`, `IN_PROGRESS`, `COMPLETED`)
- Code ratio: 70% state management / 30% business logic

**Temporal:**
- State tracking: Automatic history
- Code ratio: 100% business logic

```java
// Spring Scheduler - State management clutter
@Scheduled(cron = "0 0 * * * *")
public void processOrders() {
    List<Order> orders = orderRepo.findByStatus("PENDING");
    for (Order order : orders) {
        try {
            order.setStatus("IN_PROGRESS");
            orderRepo.save(order);
            
            processPayment(order);
            
            order.setStatus("COMPLETED");
            orderRepo.save(order);
        } catch (Exception e) {
            order.setStatus("FAILED");
            order.setErrorMessage(e.getMessage());
            orderRepo.save(order);
        }
    }
}
```

```java
// Temporal - Just business logic
public OrderResult processOrder(Order order) {
    PaymentResult payment = activities.processPayment(order);
    activities.sendNotification(order);
    return activities.completeOrder(order);
    // State management? Temporal handles it!
}
```

### III. Retries — The "External API is down" Problem

**Spring @Scheduled:**
- Retry logic: `try-catch`, exponential backoff libraries
- If 5th attempt also fails: Manual handling required

**Temporal:**
- Retry logic: Declarative policy
- If 5th attempt also fails: Automatic management

```java
// Spring - Manual retry complexity
public void callExternalApi() {
    int maxRetries = 5;
    int attempt = 0;
    while (attempt < maxRetries) {
        try {
            externalApiClient.call();
            return;
        } catch (Exception e) {
            attempt++;
            Thread.sleep((long) Math.pow(2, attempt) * 1000);
            if (attempt >= maxRetries) {
                throw new MaxRetriesExceededException(e);
            }
        }
    }
}
```

```java
// Temporal - Declarative retry
@ActivityInterface
public interface ExternalApiActivities {
    
    @ActivityMethod(
        startToCloseTimeout = "30s",
        retryPolicy = @RetryPolicy(
            maxAttempts = 5,
            backoffCoefficient = 2.0
        )
    )
    void callExternalApi();
}
```

### IV. Visibility — The "Is it working?" Problem

**Spring @Scheduled:**
- Error investigation: Search through thousands of log lines
- Tenant-based tracking: Manual implementation required

**Temporal:**
- Error investigation: Temporal UI dashboard provides full visibility
- Tenant-based tracking: Namespace/Task Queue isolation built-in

---

## Why Temporal for This Project?

For our PoC, Temporal solves three critical needs:

### 1. Multi-Tenancy
- Each tenant's work can be isolated into separate Namespaces or Task Queues.
- Heavy load from Tenant A does not slow down Tenant B.

### 2. Parent Library Logic
- Instead of building a complex scheduler for every microservice, we can build a "Parent Library" that wraps Temporal (as discussed with Gorkem).
- Other teams just write their business logic, while the library handles "immortality" and retries.

### 3. Auditability
- Every step of a Temporal workflow is recorded.
- We have an automatic audit log of every business process for free.
- We know exactly who did what and when.

---

## Alignment with Meeting Decisions

This section shows how Temporal aligns with the principles established in the 2025-12-26 meeting:

**Minimum Cost, Minimum Setup**
- Temporal Equivalent: Single command setup with Docker Compose
- Status: Aligned

**Code-First Configuration**
- Temporal Equivalent: All retry/timeout policies defined in code
- Status: Aligned

**System Simplicity**
- Temporal Equivalent: Temporal's own state recovery instead of granular recovery
- Status: Aligned

**Parent Library Logic**
- Temporal Equivalent: Workflow/Activity wrappers in shared-library
- Status: Aligned

**Multi-Tenancy**
- Temporal Equivalent: Namespace or Task Queue for tenant isolation
- Status: Aligned

**Event-Driven Approach**
- Temporal Equivalent: Temporal Signals & Queries
- Status: To Be Researched

### Research Backlog

- @SherlockTenant to Temporal Tenant Context Mapping
- Workflow vs Activity: Which goes in Parent Library?
- Using Signals for Event-Driven triggering
- Scheduler restart strategy (Full system vs Granular)

---

## Summary

Temporal is not just a "better cron job." It is a fundamental shift in how we write reliable distributed systems. It removes the "plumbing" (retries, state, persistence) from the developer's plate, allowing us to focus 100% on Business Logic.

**What Temporal handles:**
- Retry Logic
- State Management
- Persistence
- Failure Recovery
- Audit Logging

**What developers focus on:**
- 100% Business Logic

---

## References

- Temporal Docs: https://docs.temporal.io
- Temporal Java SDK: https://github.com/temporalio/sdk-java
- Meeting Notes: [2025-12-26 Gorkem & Omer Meeting](../meetings/2025-12-26-meeting-with-gorkem.md)

---

**Next Step:** Start building the practical implementation to put these concepts into practice.
