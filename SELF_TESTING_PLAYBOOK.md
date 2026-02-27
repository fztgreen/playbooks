# Playbook: Self-Testing .NET 10 API Suite (Enterprise Governance Edition)

## Phase 0: Safety, Governance & Audit (MANDATORY)
The `POST /Test` endpoint is an enterprise diagnostic tool and MUST be hardened against abuse.

### 0.1 Access, Intent & Rate Limiting
- **Auth:** MUST require Bearer Token (JWT) or mTLS with specific `Diagnostics.Run` scope.
- **Intent Header:** MUST require `X-SelfTest-Intent: run`.
- **Rate Limiting:** MUST implement per-client (e.g., 1/min) and global (e.g., 5/min) rate limits.
- **Audit Log:** EVERY invocation MUST be logged to a non-repudiable audit sink (User, Mode, Tenant, RunId, Result).

### 0.2 Mode Gating & Tenant Isolation
- **Mode Toggle:** Support `?mode=ReadOnly|SafeWrite|Full`.
    - `ReadOnly`: L0 and L1 (Idempotent/Read-Only) only.
    - `SafeWrite`: L0, L1, and L2 (with test tenant isolation).
    - `Full`: Destructive/Legacy writes (DISABLED in Prod by default).
- **Tenant Isolation:** All tests MUST execute within a dedicated `Test-Tenant-ID`.
- **Cleanup:** L2 scenarios MUST include an automated `cleanup` step or TTL-based data records.

## Phase 1: Discovery & Scoping

### 1.1 Dependency & NuGet Mapping
- **Action:** Analyze `Program.cs`, `Startup.cs`, and `*.csproj` files.
- **Requirement:** Identify major version jumps in critical libraries (e.g., EF Core, Serilog, Otel).
- **Goal:** Define an L0 "Behavioral Smoke Test" for each.

### 1.2 Configuration & Secret Inventory
- **Action:** Scan codebase for `IOptions<T>` and `IConfiguration` usage.
- **Requirement:** Cross-reference against Kubernetes ConfigMap keys and Environment Variable Secrets.

### 1.3 Endpoint & Parity Discovery
- **Action:** Map new routes against legacy manifests (Swagger/Logs).
- **Requirement:** Generate a **Gaps Report** for any missing legacy functionality.
- **Filter:** Exclude `HealthController` and `/Test` from parity enumeration and scenario testing.

### 1.4 Scenario Classification (L1 vs L2)
- **L1 (Read-Only):** Identify idempotent scenarios with NO side effects (method-agnostic, e.g., GET or POST-Search).
- **L2 (Workflows):** Identify "Business Journeys" (e.g., Create -> Process -> Validate).

## Phase 2: Documentation & Accountability
1. **SELF_TEST_CHECKLIST.md:** Track DI layers, L0-L2 coverage, and failure taxonomy for every endpoint.
2. **test-data.template.json:** Define required GUIDs, IDs, and payloads for L2 scenarios.
3. **Security Warning:** Document metadata exposure risks and mandatory governance gates.

## Phase 3: Implementation Requirements (.NET 10)

### 3.1 Strict Enterprise Schema
Output MUST strictly follow the `<Company_Standard_Health_Schema>`:
```json
{
  "runId": "GUID",
  "mode": "ReadOnly|SafeWrite|Full",
  "environment": "Prod|UAT|Dev",
  "applicationId": "NAME",
  "status": "PASS|FAIL|PARTIAL",
  "failureCategories": ["Auth", "Routing", "ModelBinding", "Validation", "Serialization", "Dependency", "Timeout", "UnhandledException", "ContractMismatch", "ConfigurationMissing"],
  "executiveSummary": {
    "migrationReady": true,
    "overallCoverage": 0.95,
    "criticalWorkflowsPass": true
  },
  "diagnostics": {
    "diContainer": { "status": "PASS|FAIL", "layers": ["Graph", "Activation"] },
    "dependencies": [
      { "name": "Database", "status": "PASS", "latencyMs": 12 }
    ],
    "endpoints": {
      "total": 50, "passed": 48, "details": [
        { "method": "GET", "route": "/api/v1/user", "scenario": "GetProfile", "category": "Routing", "status": "FAIL", "message": "404 Not Found" }
      ]
    }
  },
  "timestamp": "ISO8601"
}
```

### 3.2 Two-Layer DI Validation
1. **Layer 1: Graph Validation (No Instantiation):** Use `ValidateScopes` and `ValidateOnBuild`. Verify `IOptions` binding.
2. **Layer 2: Activation Smoke (Targeted):** Resolve an **Allowlist** (Controllers, HostedServices) within a dedicated `IServiceScope`.

### 3.3 L0: Connection & Observability Proof
- **ORM Proof:** Execute a basic query (e.g., `db.Users.AnyAsync()`) to verify translation.
- **Otel Proof:** Verify `ActivitySource.HasListeners()`, presence of OTLP endpoint, and zero internal exporter errors.
- **Serilog Proof:** Verify `ILogger` resolution, `Sinks.Count > 0`, and emit a correlation-tagged test event.

### 3.4 L1 & L2: HTTP Pipeline Execution
- **Execution:** Use `WebApplicationFactory<T>` for **in-process HTTP calls**.
- **L1 (Idempotent):** Execute all read-only scenarios through the pipeline to validate Routing, Model Binding, and Serialization.
- **L2 (Workflows):** Sequence HTTP calls (POST -> GET) passing state between them.
- **Snapshot Validation:** Canonicalize (sort keys, normalize dates/GUIDs) and diff. DO NOT normalize away `null` vs `missing` or precision changes.

## Phase 4: Maintainability & Clean Code
- **Naming Standards:** Use simple, clear, and descriptive names for all methods. Avoid cryptic abbreviations.
- **Self-Documenting Code:** Follow Clean Code principles; prioritize well-named functions and expressive logic over inline comments to explain "what" or "how".
- **Interfaces:** Implement `IDependencyCheck`, `IEndpointScenario`, and `IWorkflowScenario` to ensure modularity.
- **Parallelism:** Execute scenarios in parallel with a 30s `CancellationToken` and failure taxonomy reporting.

## Phase 5: Final Validation & Reporting
1. **Deterministic Snapshots:** Ensure canonicalization rules are strictly applied before comparison.
2. **Configuration Audit:** Verify 100% of discovered keys have values in the runtime environment.
3. **EXECUTIVE_SUMMARY.md:** Final "Go/No-Go" report.
4. **Readiness Rule:** `migrationReady=true` requires 100% L0/L2 PASS and L1 Coverage ≥ 90% with ZERO `5xx` or `ContractMismatch` failures.
5. **Build Integrity:** Validate that the .NET API and all associated test projects build successfully (`dotnet build`) without errors before final completion.

---
**Note to Executor:** Direct controller invocation is FORBIDDEN for L1/L2. All endpoint testing MUST exercise the full middleware pipeline.
