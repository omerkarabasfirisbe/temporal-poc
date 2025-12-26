# Meeting Notes: Temporal Infrastructure & Scheduler Strategy
**Date:** 2025-12-26
**Participants:** Görkem, Ömer

## Overview
Discussion regarding the integration of Temporal into the existing infrastructure, focusing on multi-tenancy, reliability, and system simplicity.

## Key Principles
- **Minimum Cost, Minimum Setup:** The architecture should prioritize low overhead and ease of deployment.
- **Code-First Configuration:** Avoid making configurations via the UI; prefer programmatic setup.
- **System Simplicity:** Favor lower complexity even if it means broader recovery actions (e.g., restarting the scheduler).

## Decision Summary & Research Areas

### 1. Scheduling & Load
- There is no immediate need to divide the scheduler into multiple parts for load balancing.
- The focus is on a robust single-entry or unified scheduling logic that scales with tenants.

### 2. Multi-tenancy & Isolation
- The system must work across all tenants.
- **Proposal:** Use an annotation-driven approach (e.g., `@SherlockTenant`) to handle tenant context, error handling, and retry mechanisms.

### 3. Error Handling Strategy
- Instead of building complex individual retry/recovery logic, there is a preference for "Parent Library" logic.
- A strategy where the entire scheduler/process restarts is preferred over high-complexity granular management, ensuring better system compatibility.
- Exploration of an **Event-Driven** approach.

### 4. Deep Dive into Temporal
- Research Temporal's core features to identify which specific capabilities (e.g., Workflows, Activities, Signals) should be wrapped into our internal parent library.
- Evaluate how Temporal's built-in retry and state management can replace or augment manual annotation-based logic.

## Action Items
- [ ] Research Temporal architecture in-depth.
- [ ] Identify features compatible with the "Parent Library" concept.
- [ ] Analyze the mapping between Sherlock/Tenant context and Temporal Workflows.
