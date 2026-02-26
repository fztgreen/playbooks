# Playbook: Self-Testing .NET API Suite

## Overview
This playbook defines the process for implementing a `POST /Health/Test` endpoint that allows an API to validate its own integrity across Dev, UAT, and Production environments.

## Phase 1: Discovery & Scoping
The executor must first map the application landscape to understand the testing surface area.

### 1.1 Dependency Mapping
- **Action:** Analyze `Program.cs`, `Startup.cs`, and configuration files (e.g., `appsettings.json`) to identify all external dependencies.
- **Scope:** Databases (SQL, NoSQL), Message Brokers (RabbitMQ, Service Bus), Cache (Redis), and External APIs.
- **Requirement:** For each dependency, the executor must determine the standard health check mechanism (e.g., `Microsoft.Extensions.Diagnostics.HealthChecks`).

### 1.2 Endpoint Discovery
- **Action:** Inspect Controllers and Minimal API mappings.
- **Assumption:** The application supports OpenAPI/Swagger. Use `IApiDescriptionGroupCollectionProvider` or reflection to list all routable endpoints.

### 1.3 External Data Identification
- **Action:** Identify endpoints requiring specific identifiers (GUIDs, IDs) or 3rd-party data to execute successfully.
- **Requirement:** **STOP and notify the user** if external data is missing. Generate a `test-data.template.json` for the user to populate.

## Phase 2: Documentation & Communication
Before implementation, the executor must ensure the user understands the scope.

1. **User Notification:** Provide a summary of all discovered dependencies and endpoints.
2. **Master Checklist:** Generate a `SELF_TEST_CHECKLIST.md` documenting:
    - [ ] DI Container Validation strategy.
    - [ ] Connection health for each identified dependency.
    - [ ] List of all endpoints to be pinged/tested.
3. **Security Warning:** Explicitly document that `POST /Test` exposes system metadata and DI resolution logic. Recommend future IP whitelisting or Bearer Token protection.

## Phase 3: Implementation Requirements

### 3.1 The Test Endpoint
- **Location:** `HealthController`
- **Route:** `POST /Test`
- **Output:** A detailed JSON report containing `Pass/Fail` status for each category.

### 3.2 DI Container Validation
- **Logic:** Iterate through the `IServiceCollection` registered in the `IServiceProvider`.
- **Action:** Attempt to resolve every registered service (or at least all Controllers/Services).
- **Goal:** Catch "Unable to resolve service for type..." errors that usually only appear at runtime.

### 3.3 Connection Health
- **Logic:** Execute a "ping" or "select 1" style query for all mapped dependencies.
- **Action:** Use existing .NET HealthCheck abstractions where possible.

### 3.4 Endpoint Connectivity
- **Logic:** For every discovered route, perform a low-impact request (e.g., `GET` or a `POST` with the test data provided in the config template).
- **Validation:** Check for `2xx` or `4xx` (auth/business logic) vs `5xx` (server crash).

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

---
**Note to Executor:** Proceed with implementation only after the user has acknowledged the checklist and provided necessary data in the template.
