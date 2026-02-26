# Playbook: Self-Testing .NET API Suite (Enterprise Edition)

## Phase 0: Safety & Governance (MANDATORY)
The `POST /Test` endpoint is a powerful diagnostic tool; it must be hardened for Production.

### 0.1 Access & Intent Gates
- **Auth:** MUST require Bearer Token (JWT) or mTLS.
- **Intent Header:** MUST require `X-SelfTest-Intent: run`.
- **Mode Toggle:** MUST support `?mode=ReadOnly|SafeWrite|Full`.
    - `ReadOnly`: L0 and L1 (GET) only.
    - `SafeWrite`: L0, L1, and L2 (with test tenant isolation).
    - `Full`: Destructive/Legacy writes (DISABLED in Prod by default via Feature Flag).

### 0.2 Tenant & Data Isolation
- **Tenant Isolation:** All tests MUST run under a dedicated `Test-Tenant-ID`.
- **Correlation:** Every log/trace emitted by the test MUST include a `Test-Correlation-ID`.
- **Cleanup:** L2 tests MUST implement a `cleanup` step or utilize TTL-based test records.

## Phase 1: Discovery & Scoping

### 1.1 Dependency & NuGet Mapping
- **Action:** Analyze `Program.cs`, `Startup.cs`, and `*.csproj` files.
- **Requirement:** Identify major version jumps (e.g., EF Core, Serilog, Otel).
- **Goal:** Define an L0 "Behavioral Smoke Test" for each critical library.

### 1.2 Configuration & Secret Inventory
- **Action:** Scan codebase for `IOptions<T>` and `IConfiguration` usage.
- **Requirement:** Cross-reference against Kubernetes ConfigMap keys and Environment Variable Secrets.

### 1.3 Endpoint & Parity Discovery
- **Action:** Map new routes against legacy `swagger.json` or documentation.
- **Requirement:** Generate a **Gaps Report** for any missing legacy functionality.
- **Filter:** **FILTER OUT** the `HealthController` and the `/Test` route from testing.

### 1.4 Critical Workflow Identification
- **Action:** Group endpoints into "Business Journeys" and identify **Critical Workflows**.
- **Definition:** Revenue-impacting logic, high-integrity writes, or security-sensitive paths.

## Phase 2: Documentation & Accountability
1. **SELF_TEST_CHECKLIST.md:** Document DI strategy, L0-L2 coverage, and categorization of every endpoint.
2. **test-data.template.json:** Define required GUIDs, IDs, and payloads for L2 workflows.
3. **Security Warning:** Explicitly document metadata exposure risks and the requirement for auth gates.

## Phase 3: Implementation Requirements (.NET 10 Spec)

### 3.1 Strict Schema & Scoring
Output MUST be a single JSON object following the `<Company_Standard_Health_Schema>`:
- `migrationReady`: Boolean. `True` ONLY if:
    - 100% of L0 (Dependencies/DI) PASS.
    - 100% of L2 (Critical Workflows) PASS.
    - L1 (Endpoint) Coverage ≥ 90% with zero 5xx errors.
- `status`: `PASS` (Ready), `PARTIAL` (L0/L2 pass, L1 < 90%), `FAIL` (Any L0/L2 failure).

### 3.2 DI Strategy (Safe & Targeted)
- **Logic:** Validate the graph without side-effect resolution.
- **Action:** 
    - Enable `ValidateScopes` and `ValidateOnBuild` in the Host.
    - Resolve an **Allowlist** within a dedicated `IServiceScope`: `ControllerBase`, `IHostedService`, and explicit "Critical Application Services."

### 3.3 L0: Connection & Observability Proof
- **Connectivity:** Execute a basic ORM query (e.g., `db.Users.AnyAsync()`) to verify query translation.
- **Otel Proof:** Verify `ActivitySource` is initialized AND check for the absence of exporter configuration errors.
- **Serilog Proof:** Verify `ILogger` resolution and that at least one sink is registered.

### 3.4 L1 & L2: HTTP Pipeline Execution (The "Real" Test)
- **Execution:** Use `WebApplicationFactory<T>` or `TestServer` for **in-process HTTP calls**.
- **L1 (Discovery):** Hit all `GET` routes through the pipeline to validate Routing, Model Binding, and Serialization.
- **L2 (Workflows):** Sequence multiple HTTP calls (POST -> GET) passing state between them to test state transitions.
- **Snapshot Validation:** Canonicalize (sort keys, normalize dates) and diff against "Golden JSON" snapshots.
- **Parallelism:** Execute scenarios in parallel with a mandatory 30s `CancellationToken` and detailed timeout reporting.

## Phase 4: Plugin Model for Maintainability
- **Interfaces:** Implement `IDependencyCheck`, `IEndpointScenario`, and `IWorkflowScenario`.
- **Discovery:** Use Reflection to auto-discover and register scenarios in the test runner.

## Phase 5: Final Validation & Reporting
1. **Deterministic Snapshots:** Ensure all JSON is normalized before comparison to prevent flakiness.
2. **Configuration Audit:** Verify 100% of discovered keys have values in the runtime environment.
3. **EXECUTIVE_SUMMARY.md:** Final "Go/No-Go" based on weighted coverage and L2 journey success.

---
**Note to Executor:** Direct controller invocation is forbidden for L1/L2 tests. You MUST use the HTTP pipeline (via `HttpClient`) to ensure all routing, filters, and serialization logic are validated for the .NET 10 migration.
