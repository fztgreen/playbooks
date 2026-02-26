# Playbook: Self-Testing .NET API Suite

## Overview
This playbook defines the process for implementing a `POST /Health/Test` endpoint that allows an API to validate its own integrity across Dev, UAT, and Production environments.

## Phase 1: Discovery & Scoping
The executor must first map the application landscape to understand the testing surface area.

### 1.1 Dependency & NuGet Mapping
- **Action:** Analyze `Program.cs`, `Startup.cs`, and `*.csproj` files.
- **Scope:** Databases, Brokers, Cache, and **Critical NuGet Packages** (e.g., EF Core, Serilog, OpenTelemetry).
- **Requirement:** Identify packages that underwent major version jumps. The executor must determine a "Smoke Test" for each.

### 1.2 Configuration & Secret Mapping
- **Action:** Perform a recursive scan of the codebase for configuration consumption.
- **Patterns to Identify:**
    - `IOptions<T>`, `IOptionsSnapshot<T>`, `IOptionsMonitor<T>` (Options Pattern).
    - `Configuration["Key"]` or `_configuration.GetValue<T>("Key")` (Direct Access).
- **Goal:** Build a master list of all required keys that must be present in the ConfigMap or Environment Variables.

### 1.2 Endpoint & Parity Discovery
- **Action:** Inspect Controllers and Minimal API mappings.
- **Parity Audit:** Attempt to locate the legacy system's route manifest (e.g., legacy `swagger.json` or documentation).
- **Requirement:** Create a **Gaps Report** in the checklist identifying any legacy endpoints not found in the new codebase.
- **Requirement:** **FILTER OUT** the `HealthController` and the `/Test` route.

### 1.3 External Data & Side-Effect Identification
...
### 1.4 Critical Workflow Identification
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
- **Logic:** Iterate through the `IServiceCollection` registered in the `IServiceProvider`.
- **Action:** Attempt to resolve every registered service (or at least all Controllers/Services).
- **Goal:** Catch "Unable to resolve service for type..." errors that usually only appear at runtime.

### 3.3 Connection & Behavioral Health
- **Logic:** Execute a "ping" AND a "basic operation" for all mapped dependencies.
- **Action:** 
    - **Database:** Execute a simple query via the ORM (e.g., EF Core).
    - **Observability (Serilog):** Emit a "Health Test" log and verify the `ILogger` is resolved correctly from DI.
    - **Observability (Otel):** Start a test `Activity` or `Span` via the Otel/Grafana SDK to ensure the telemetry pipeline is initialized.
    - **Mapping:** Trigger a sample Map (e.g., AutoMapper) to verify profiles are still valid in the new runtime.

### 3.4 In-Process Endpoint & Workflow Validation
- **Logic:** Resolve Controllers and invoke actions directly.
- **Testing Priorities:**
    - **100% Happy Path:** Every endpoint must have a successful "basic" execution.
    - **Critical Workflow Sequencing:** Pass state between steps for "Crown Jewel" journeys.
    - **Performance Budgeting:** Measure and report execution time for each Critical Workflow.
    - **Graceful Failure Check:** Verify the Global Exception Handler.
- **Execution Strategy:**
    - **Parallel Execution:** Utilize `Task.WhenAll` or a `SemaphoreSlim` to execute test scenarios in parallel.
    - **Safety Timeouts:** Implement a mandatory timeout (e.g., 30s) per scenario or for the entire suite.
    - **Timeout Handling:** If a test exceeds the timeout, the system must **cancel the task**, capture the specific scenario that hung, and notify the user in the report.
- **Output (Reporting Timeouts):**
    - The JSON response must explicitly list any **Timed Out** tests.
    - Include the reason (e.g., "Dependency Latency" or "Deadlock suspected") based on the last captured trace before the timeout.

## Phase 4: External Data Configuration
If the executor finds that endpoints require specific data, they must create the following template:

**test-data.template.json**
```json
{
  "Endpoints": {
    "/api/v1/Users/{id}": {
      "id": "REPLACE_WITH_VALID_GUID"
    },
    "/api/v1/Orders/Process": {
      "body": { "orderId": "REPLACE_WITH_DATA" }
    }
  }
}
```

## Phase 5: Final Validation & Observability

### 5.1 Snapshot Regression
- **Action:** If available, compare the JSON output of the `POST /Test` against a "Golden Snapshot" from the legacy system.
- **Goal:** Detect subtle serialization shifts (e.g., `DateTime` ISO formats, `null` vs. `empty` arrays).

### 5.2 Configuration & Secret Audit
- **Action:** Cross-reference the master list of keys from Phase 1.2 against the runtime environment.
- **Verification:**
    - **ConfigMaps:** Ensure non-sensitive keys exist in the injected configuration providers.
    - **Secrets:** Verify that sensitive keys (e.g., API keys, Connection Strings) are present as Environment Variables.
- **Goal:** Prevent "Missing Configuration" errors that often occur during migration when names change (e.g., `ConnectionStrings:Db` vs `DB_CONNECTION`).

### 5.3 Observability Verification
- **Action:** The `/Test` endpoint must emit a specific "Self-Test-Success" log and trace.
- **Goal:** Confirm that the migration didn't break the logging and telemetry pipeline.

### 5.4 Executive Sign-Off Report
- **Action:** The executor must generate a final `EXECUTIVE_SUMMARY.md` based on the codebase validation findings.
- **Mandatory Content:**
    - **Migration Readiness:** [READY/NOT READY]
    - **Feature Parity:** [X] of [Y] legacy features validated.
    - **Critical Workflows:** Results of the identified business journeys.
    - **Residual Risks:** List any endpoints or dependencies that were skipped and why.
    - **Observability Proof:** Verification that Serilog and Otel traces were successfully emitted during the test.

---
**Note to Executor:** Success is defined as a codebase where the `POST /Test` endpoint reliably validates the happy path of all endpoints and critical workflows within the application process.
