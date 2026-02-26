# Playbook: Self-Testing .NET API Suite

## Overview
This playbook defines the process for implementing a `POST /Health/Test` endpoint that allows an API to validate its own integrity across Dev, UAT, and Production environments.

## Phase 0: Safety & Governance
Before any discovery or implementation, the system must establish mandatory guardrails.

### 0.1 Access Control & Intent
- **Requirement:** The `POST /Test` endpoint MUST be protected by Bearer Token (JWT) or mTLS.
- **Requirement:** Requests MUST include a custom header `X-SelfTest-Intent: run` to prevent accidental triggers.
- **Requirement:** Implement an explicit `mode` query parameter: `?mode=ReadOnly|SafeWrite|Full`.
- **Default Policy:** `Full` mode MUST be disabled in Production environments via Feature Flag.

### 0.2 Tenant & Data Isolation
- **Requirement:** Tests MUST use a dedicated "Test Tenant" or "Sandbox User" context.
- **Requirement:** Any data created during `SafeWrite` or `Full` modes MUST be tagged with a `Test-Correlation-ID` and include an automated cleanup step (or TTL).

## Phase 1: Discovery & Scoping
The executor must first map the application landscape to understand the testing surface area.

### 1.1 Dependency & NuGet Mapping
... (existing content) ...

### 1.2 Configuration & Secret Mapping
... (existing content) ...

### 1.3 Endpoint & Parity Discovery
- **Action:** Inspect Controllers and Minimal API mappings.
- **Parity Audit:** Attempt to locate the legacy system's route manifest (e.g., legacy `swagger.json` or documentation).
- **Requirement:** Create a **Gaps Report** in the checklist identifying any legacy endpoints not found in the new codebase.
- **Requirement:** **FILTER OUT** the `HealthController` and the `/Test` route.

### 1.4 External Data & Side-Effect Identification
... (existing content) ...

### 1.5 Critical Workflow Identification
- **Action:** Group endpoints into "Business Journeys" and identify **Critical Workflows**.
- **Definition of Critical:**
    - **Revenue/Core Logic:** (e.g., Checkouts, Payments, Account Creation).
    - **High-Integrity Writes:** Processes that update multiple tables or external systems.
    - **Security-Sensitive:** Identity management or permission-heavy routes.
- **Requirement:** For each critical workflow, define a "Success Criteria" that goes beyond a 200 OK (e.g., "Order record exists in DB").

## Phase 2: Documentation & Communication
Before implementation, the executor must ensure the user understands the scope.

1. **User Notification:** Provide a summary of all discovered dependencies and endpoints.
2. **Master Checklist:** Generate a `SELF_TEST_CHECKLIST.md` documenting:
    - [ ] DI Container Validation strategy.
    - [ ] Connection health for each identified dependency.
    - [ ] List of all endpoints, categorized by Risk.
    - [ ] **Definition of Critical Workflows and their expected outcomes.**
    - [ ] Serialization compatibility check (Legacy vs. Modern .NET).
    - [ ] Attribute integrity check.
3. **Security Warning:** Explicitly document that `POST /Test` exposes system metadata and DI resolution logic. Recommend future IP whitelisting or Bearer Token protection.

## Phase 3: Implementation Requirements

### 3.1 The Test Endpoint
- **Location:** `HealthController`
- **Route:** `POST /Test`
- **Output:** MUST strictly adhere to the `<Company_Standard_Health_Schema>` for automated aggregation:
```json
{
  "applicationId": "GUID_OR_NAME",
  "status": "PASS|FAIL|PARTIAL",
  "executiveSummary": {
    "migrationReady": true,
    "overallCoverage": 0.85,
    "criticalWorkflowsPass": true
  },
  "diagnostics": {
    "diContainer": { "status": "PASS|FAIL", "errors": [] },
    "dependencies": [
      { "name": "Database", "type": "SQL", "status": "PASS|FAIL", "latencyMs": 12 }
    ],
    "endpoints": {
      "total": 50,
      "passed": 48,
      "timedOut": 1,
      "failed": 1,
      "details": []
    }
  },
  "workflows": [
    { "name": "CheckoutFlow", "status": "PASS|FAIL", "durationMs": 450 }
  ],
  "timestamp": "ISO8601"
}
```
- **Status Code:** Returns `200 OK` only if `status` is `PASS`; otherwise returns `503 Service Unavailable`.

### 3.2 DI Container Validation
- **Logic:** Validate the service provider without expensive or unsafe runtime resolutions.
- **Action:** 
    - Enable `ValidateScopes` and `ValidateOnBuild` in the `Host` configuration.
    - **Targeted Resolution:** Only resolve specific categories within a dedicated `IServiceScope`:
        - All `ControllerBase` types.
        - All `IHostedService` types.
        - A developer-defined "Critical Services" allowlist.
- **Goal:** Catch "Unable to resolve service" errors at startup without side-effect risks.

### 3.3 Connection & Behavioral Health (L0)
- **Logic:** Execute "Sanity" checks for dependencies and internal invariants.
- **Action:** 
    - **Database:** Execute a simple query via the ORM.
    - **Observability (Serilog/Otel):** 
        - Verify `ActivitySource` and `ILogger` are resolved.
        - Check for internal "Self-Diagnostics" errors in the Otel exporter configuration.
    - **Mapping:** Trigger a sample Map (e.g., AutoMapper).

### 3.4 HTTP Pipeline Validation (L1 & L2)
- **Logic:** Execute tests through the full ASP.NET Core middleware pipeline (routing, auth filters, model binding, serialization).
- **Execution Strategy:** 
    - Use `WebApplicationFactory<T>` or `TestServer` for **in-process HTTP calls**.
    - **L1 (Read-Only):** Perform `GET` requests for all discovered endpoints using `ReadOnly` mode.
    - **L2 (Critical Workflows):** Execute "Business Journeys" (e.g., Create -> Get -> Update) using `SafeWrite` mode and a test tenant.
- **Performance Budgeting:** Measure and report execution time for each Critical Workflow.
- **Output (Reporting):** 
    - The JSON response MUST explicitly list any **Timed Out** or **Blocked** tests.
- **Scoring Rules for overallCoverage:**
    - **Pass:** All L0 checks pass + 100% of L2 critical workflows pass + 100% of L1 happy paths pass.
    - **Partial:** All L0 checks pass + 100% of L2 critical workflows pass + <100% of L1 happy paths pass.
    - **Fail:** Any L0 check fails OR any L2 critical workflow fails.

## Phase 4: Plugin Model for Maintainability
The self-test suite MUST avoid bespoke, unmanageable code.

### 4.1 Interface-Based Tests
- **IDependencyCheck:** Implement for each dependency (e.g., `SqlDependencyCheck`, `RedisDependencyCheck`).
- **IEndpointScenario:** For individual L1 happy path tests.
- **IWorkflowScenario:** For grouped L2 business journeys.
- **Goal:** Allow the test runner to automatically discover and execute tests using Reflection.

### 4.2 External Data Configuration
If the executor finds that endpoints require specific data, they must create the following template for the user to populate:

**test-data.template.json**
```json
{
  "Endpoints": {
    "/api/v1/Users/{id}": {
      "id": "REPLACE_WITH_VALID_GUID"
    }
  }
}
```

## Phase 5: Final Validation & Reporting
... (existing Phase 5 content) ...

---
**Note to Executor:** Success is defined as a codebase where the `POST /Test` endpoint reliably validates the happy path of all endpoints and critical workflows **through the full HTTP pipeline** without introducing side-effect risks or production instability.
