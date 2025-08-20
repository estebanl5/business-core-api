# PRD — CRM + HR API (Enterprise-Grade, C#-Focused)

## 1) Purpose & Vision
Design an enterprise-ready API that unifies CRM (customers, opportunities) and HR (employees, leaves) under a consistent platform and operating model. The solution must emphasize strong boundaries, observability, and maintainability, enabling multiple product teams to iterate independently while preserving governance and reliability. (Based on the original PRD scope and endpoints.)

---

## 2) Scope
- **In**: Customer master data, opportunities pipeline, employee records, leave management, audit & reporting, webhooks/eventing, API lifecycle management.
- **Out (Phase 1)**: Payroll computations, third-party CRM/HR sync jobs beyond webhook triggers (provide extensibility points).

---

## 3) Architecture Overview (C# Enterprise)
**Style**: Clean Architecture + DDD + CQRS (commands/queries) with MediatR, EF Core, FluentValidation, and Polly for resilience.  
**Runtime**: ASP.NET Core Minimal APIs/Controllers (choose one), .NET 8 LTS.  
**Infrastructure**: Azure App Service or AKS; Azure SQL/SQL MI; Azure API Management; Azure Key Vault; Azure Application Insights; Azure Storage for outbox/state; Azure Service Bus for domain events.

**Layering**
- **Presentation**: ASP.NET Core API (AuthZ filters, exception mapping, versioning, OpenAPI).
- **Application**: Use cases via MediatR handlers, input validation via FluentValidation, pipeline behaviors for cross-cutting (logging, tracing, idempotency).
- **Domain**: Entities/Aggregates (Customer, Opportunity, Employee, Leave) with invariants and domain events.
- **Infrastructure**: EF Core repositories, transactional outbox, Service Bus publisher, caching adapters, time/system abstractions.

**Bounded Contexts**
- CRM Context (Customers, Opportunities)
- HR Context (Employees, Leaves)
- Shared Kernel (Value Objects: Email, Money, DateRange; common abstractions: IClock, CorrelationId)

---

## 4) Non-Functional Requirements (NFRs)

### Security
- OAuth2/OIDC via **Azure AD**; JWT bearer. 
- Role + Claims-based authorization:
  - CRM: `crm.read`, `crm.write`, `crm.admin`
  - HR: `hr.read`, `hr.write`, `hr.admin`
- **Fine-grained ABAC** (Attribute-Based Access Control): e.g., scope + department claim for HR resources.
- TLS 1.2+ enforced end-to-end; HSTS enabled.
- Secrets in **Azure Key Vault**; automatic rotation supported.
- PII encryption at rest; FIPS-compliant algorithms for cryptographic ops.
- **Audit**: who/what/when/where; signed, tamper-evident logs; retention policy (see §12).

### Compatibility
- REST/JSON (OAS 3.0/Swagger); HATEOAS optional.
- **API Versioning**: URL version (`/api/v1`), minimize breaking changes; dual-ship old/new during deprecation window (see §11).
- **Webhooks** for business events (`customer.created`, `employee.created`) with signed payloads (HMAC) + retry + backoff.

### Performance & Resilience
- **Budgets**: GET p50 < 200 ms, p95 < 400 ms; writes p50 < 500 ms, p95 < 900 ms at baseline.
- **Resilience**: Polly policies (retry with jitter, circuit-breaker, timeout, bulkhead); idempotency keys for POST (see §8).
- **Caching**: Response caching for lists; EF Core second-level cache for hot reads; conditional GET (ETags).
- **Scalability**: Horizontal scale on App Service/AKS; stateless API; sticky-free session.
- **Data**: Partitioning strategy for large tables (customers, employees); proper indexing strategy and query plans.

### Observability & Operability
- **OpenTelemetry** tracing + logs + metrics → Application Insights.
- **Correlation**: `X-Correlation-Id` propagated through layers and to Service Bus.
- **SLOs**: Availability 99.9%; Error rate < 1% per rolling 30m; Alerting via Azure Monitor (latency, 5xx, queue DLQ growth).
- **Feature Flags**: Progressive delivery with .NET FeatureManagement.

### Usability
- Developer portal via **Azure API Management**; examples, SDK stubs (C#, TypeScript).
- Sandbox with seeded data + predictable test IDs.

---

## 5) Domain Model (DDD Summary)
- **Customer (Aggregate)**: Id, Name, PrimaryEmail, Status, Addresses[], Contacts[]; Events: `CustomerCreated`, `CustomerDeactivated`.
- **Opportunity (Aggregate)**: Id, CustomerId, Stage, Owner, Amount(Money), CloseDate; Events: `OpportunityStageChanged`, `OpportunityWon`.
- **Employee (Aggregate)**: Id, PersonName, Email, Department, ManagerId, HireDate, Status; Events: `EmployeeCreated`, `EmployeeStatusChanged`.
- **Leave (Entity)**: Id, EmployeeId, DateRange, Type, Status (Pending/Approved/Rejected); Event: `LeaveRequested`, `LeaveApproved`.

Invariants live in aggregates; cross-aggregate coordination uses domain events + handlers inside the same context, integration events across contexts.

---

## 6) Data Design
- **SQL** (normalized cores + read-optimized indexes). Soft deletes for PII-heavy tables; row-version columns for concurrency.
- EF Core configurations: precision for Money, owned types for Value Objects (Email, DateRange), shadow properties for audit.
- **Outbox** table: `OutboxMessages(Id, AggregateId, Type, Payload, OccurredAtUtc, ProcessedAtUtc, Attempts)`.

---

## 7) API Design (Resources & Endpoints)
**Base**: `/api/v1`

### CRM — Customers
- `GET /customers` (list, filter: `status`, `email`, `createdFrom`, `createdTo`, pagination)
- `GET /customers/{id}`
- `POST /customers`
- `PUT /customers/{id}`
- `PATCH /customers/{id}`
- `DELETE /customers/{id}`

### CRM — Opportunities
- `GET /opportunities` (filter: `customerId`, `stage`, `owner`, `from`, `to`, pagination)
- `GET /opportunities/{id}`
- `POST /opportunities`
- `PATCH /opportunities/{id}`
- `DELETE /opportunities/{id}`

### HR — Employees
- `GET /employees` (filter: `department`, `status`, pagination)
- `GET /employees/{id}`
- `POST /employees`
- `PUT /employees/{id}`
- `PATCH /employees/{id}`
- `DELETE /employees/{id}`

### HR — Leaves
- `GET /employees/{id}/leaves`
- `POST /employees/{id}/leaves`
- `PATCH /leaves/{id}`
- `DELETE /leaves/{id}`

**Design Rules**
- **Pagination**: `?page[size]=25&page[number]=1` (default size 25, max 200); include `totalCount` header.
- **Sorting**: `?sort=-createdAt,name`.
- **Filtering**: consistent operators (`eq`, `lt`, `gt`, `in`).
- **Idempotency**: `Idempotency-Key` header on POST; server stores request hash+response for TTL (see §8).
- **Optimistic Concurrency**: `If-Match` with ETag or `RowVersion` field on updates; return `412 Precondition Failed` on mismatch.
- **Errors**: RFC 7807 Problem Details; consistent error codes (see §10).

---

## 8) Critical Cross-Cutting Designs

### Idempotency (POST/Commands)
- Accept `Idempotency-Key`; dedupe via store (SQL/Redis).
- Persist request body hash + outcome; return same response on replay.
- Enforced in a **MediatR Pipeline Behavior**.

### Transactional Outbox
- Wrap aggregate changes + outbox write in one DB transaction.
- Background worker publishes outbox messages to **Azure Service Bus**.
- Publisher uses retry/backoff; moves poison to DLQ; metrics on publish latency.

### Webhooks
- Subscriptions per event type; secret per subscriber; **HMAC-SHA256** signature in header.
- Retries with exponential backoff; dead-letter on permanent failures; subscriber self-service test endpoint.

### Caching
- HTTP cache headers on GET; ETags; APIM response cache for list endpoints.
- In-proc cache only for ephemeral data; Redis when multi-instance.

### Multi-Tenancy (optional)
- Tenant resolution via header `X-Tenant-Id`; row-level security (SQL) or per-tenant schema depending on isolation needs.
- Tenant in all domain keys if shared schema.

---

## 9) Security & Authorization Model
- JWT scopes map to coarse permissions; resource-level AuthZ policies (Owner/Manager/Department).
- PII field-level redaction for unauthorized scopes.
- **Data Access Logs**: log subject, resource, action, decision, reason.

---

## 10) Error Handling (Standardized)
- Use **Problem Details** (`application/problem+json`).
- Error categories:
  - `validation_error` (400)
  - `unauthorized` (401), `forbidden` (403)
  - `not_found` (404)
  - `conflict` (409) — concurrency/idempotency violations
  - `unprocessable_entity` (422) — business rule failure
  - `too_many_requests` (429)
  - `server_error` (5xx)
- Include `traceId` and `correlationId` in every response.

---

## 11) Versioning & Deprecation
- Sunsetting policy: ≥ 6 months overlap for breaking changes.
- `Deprecation` & `Sunset` headers; APIM portal announcements; migration guides and SDK diffs.
- Backward-compatible changes preferred (additive fields, new endpoints).

---

## 12) Data Governance
- **Retention**: Raw audit logs 13 months (configurable); PII purge workflows; GDPR/CCPA “right to be forgotten”.
- **Backups**: Point-in-time restore; encrypted; tested via quarterly drills.
- **Access**: RBAC least privilege; just-in-time elevation; break-glass procedures.

---

## 13) CI/CD & Quality Gates
- **Pipelines**: Build → Test → SCA → IaC Validate → Deploy to Dev → Int → Staging → Prod with approvals.
- **Quality**: static analysis (Roslyn analyzers), code coverage gate (≥85% for Application layer), dependency vulnerability scan.
- **Infra as Code**: Bicep/Terraform for Azure infra. Blue-green or canary on App Service/AKS.
- **DB**: EF Core migrations with pre/post flight checks; backward-compatible deploys first.

---

## 14) Testing Strategy
- **Unit**: Domain invariants; validators; handlers.
- **Integration**: EF Core against Testcontainers (SQL container); API against WireMock for external deps.
- **Contract**: OpenAPI-driven consumer/provider tests; webhook signature tests.
- **E2E**: Synthetic journeys for CRUD + authz paths; load tests with k6.

---

## 15) Sample C# Patterns (Escaped Blocks)

**MediatR Command (CreateCustomer)**
```csharp
public sealed record CreateCustomerCommand(string Name, string Email) : IRequest<CustomerId>;

public sealed class CreateCustomerHandler : IRequestHandler<CreateCustomerCommand, CustomerId>
{
    private readonly ICustomerRepository _repo;
    private readonly IUnitOfWork _uow;

    public CreateCustomerHandler(ICustomerRepository repo, IUnitOfWork uow)
    {
        _repo = repo; _uow = uow;
    }

    public async Task<CustomerId> Handle(CreateCustomerCommand cmd, CancellationToken ct)
    {
        var email = Email.Create(cmd.Email); // VO with validation
        var customer = Customer.Create(cmd.Name, email);
        await _repo.AddAsync(customer, ct);
        await _uow.SaveChangesAsync(ct); // writes outbox inside the tx
        return customer.Id;
    }
}
```

**Pipeline Behavior (Idempotency)**
```csharp
public sealed class IdempotencyBehavior<TRequest, TResponse> : IPipelineBehavior<TRequest, TResponse>
    where TRequest : notnull
{
    private readonly IIdempotencyStore _store;
    private readonly IHttpContextAccessor _http;

    public IdempotencyBehavior(IIdempotencyStore store, IHttpContextAccessor http)
    { _store = store; _http = http; }

    public async Task<TResponse> Handle(TRequest request, RequestHandlerDelegate<TResponse> next, CancellationToken ct)
    {
        var key = _http.HttpContext?.Request.Headers["Idempotency-Key"].ToString();
        if (string.IsNullOrWhiteSpace(key)) return await next();

        if (await _store.TryGetAsync<TResponse>(key, request, ct) is { Found: true } hit)
            return hit.Value;

        var response = await next();
        await _store.SaveAsync(key!, request, response, TimeSpan.FromHours(24), ct);
        return response;
    }
}
```

**Problem Details Middleware (Mapping Exceptions)**
```csharp
app.UseExceptionHandler(errApp =>
{
    errApp.Run(async ctx =>
    {
        var feature = ctx.Features.Get<IExceptionHandlerFeature>();
        var (status, type, title) = ExceptionMapper.Map(feature?.Error);
        ctx.Response.StatusCode = status;
        ctx.Response.ContentType = "application/problem+json";
        var problem = new ProblemDetails
        {
            Type = type,
            Title = title,
            Status = status,
            Instance = ctx.Request.Path
        };
        problem.Extensions["traceId"] = Activity.Current?.Id ?? ctx.TraceIdentifier;
        problem.Extensions["correlationId"] = ctx.Request.Headers["X-Correlation-Id"].ToString();
        await ctx.Response.WriteAsJsonAsync(problem);
    });
});
```

**EF Core Entity Configuration (Value Objects & Concurrency)**
```csharp
public sealed class CustomerConfig : IEntityTypeConfiguration<Customer>
{
    public void Configure(EntityTypeBuilder<Customer> b)
    {
        b.ToTable("Customers");
        b.HasKey(x => x.Id);
        b.Property(x => x.RowVersion).IsRowVersion();

        b.OwnsOne(x => x.PrimaryEmail, email =>
        {
            email.Property(e => e.Value)
                 .HasColumnName("PrimaryEmail")
                 .HasMaxLength(254)
                 .IsRequired();
        });

        b.Property(x => x.Name)
         .HasMaxLength(200)
         .IsRequired();

        b.HasIndex(x => x.Name);
        b.HasIndex("PrimaryEmail").IsUnique();
    }
}
```

**Polly Resilience Handler for Service Bus/Webhook Post**
```csharp
var jitter = new Random();
var policy = Policy.Handle<HttpRequestException>()
    .OrResult<HttpResponseMessage>(r => (int)r.StatusCode >= 500 || r.StatusCode == HttpStatusCode.RequestTimeout)
    .WaitAndRetryAsync(5, attempt => TimeSpan.FromSeconds(Math.Pow(2, attempt)) + TimeSpan.FromMilliseconds(jitter.Next(0, 250)))
    .WrapAsync(Policy.TimeoutAsync<HttpResponseMessage>(TimeSpan.FromSeconds(10)))
    .WrapAsync(Policy.BulkheadAsync<HttpResponseMessage>(maxParallelization: 50, maxQueuingActions: 100));
```

---

## 16) Security Testing & Compliance
- Automated DAST on staging; SAST on PR; dependency scanning (GitHub Dependabot/OWASP).
- Threat modeling (STRIDE) at epic kickoff; mitigation backlog tracked to closure.
- Periodic pen-tests; evidence stored for audits (SOC 2/ISO 27001 alignment).

---

## 17) Rollout & Support
- Phased rollout per bounded context; dark-launch read endpoints first.
- Operational runbooks (oncall, dashboards, SLOs, incident comms).
- Error budgets feed into release cadence.

---

## 18) Acceptance Criteria (Samples)
- Creating a Customer with valid email returns `201` and emits `customer.created` within 5s.
- Concurrent PATCH with stale ETag returns `412`.
- POST with same `Idempotency-Key` returns identical response within 24h.
- Webhook retries up to 24h with exponential backoff, then DLQ.

## 19) Database Schema (Draft ERD + SQL DDL)

### Entity Relationship Overview
- **Customer (1) → (M) Opportunity**
- **Employee (1) → (M) Leave**
- **Employee (1) → (M) Employee (Manager relation)**

### Draft Schema (SQL Server / EF Core Compatible)
```sql
CREATE TABLE Customers (
  Id UNIQUEIDENTIFIER PRIMARY KEY,
  Name NVARCHAR(200) NOT NULL,
  PrimaryEmail NVARCHAR(254) NOT NULL UNIQUE,
  Status NVARCHAR(50) NOT NULL,
  CreatedAtUtc DATETIME2 NOT NULL DEFAULT SYSUTCDATETIME(),
  RowVersion ROWVERSION
);

CREATE TABLE Opportunities (
  Id UNIQUEIDENTIFIER PRIMARY KEY,
  CustomerId UNIQUEIDENTIFIER NOT NULL,
  Stage NVARCHAR(50) NOT NULL,
  OwnerId UNIQUEIDENTIFIER NOT NULL,
  Amount DECIMAL(18,2) NOT NULL,
  CloseDate DATE,
  RowVersion ROWVERSION,
  FOREIGN KEY (CustomerId) REFERENCES Customers(Id)
);

CREATE TABLE Employees (
  Id UNIQUEIDENTIFIER PRIMARY KEY,
  Name NVARCHAR(200) NOT NULL,
  Email NVARCHAR(254) NOT NULL UNIQUE,
  Department NVARCHAR(100),
  ManagerId UNIQUEIDENTIFIER NULL,
  HireDate DATE NOT NULL,
  Status NVARCHAR(50) NOT NULL,
  RowVersion ROWVERSION,
  FOREIGN KEY (ManagerId) REFERENCES Employees(Id)
);

CREATE TABLE Leaves (
  Id UNIQUEIDENTIFIER PRIMARY KEY,
  EmployeeId UNIQUEIDENTIFIER NOT NULL,
  StartDate DATE NOT NULL,
  EndDate DATE NOT NULL,
  Type NVARCHAR(50) NOT NULL,
  Status NVARCHAR(20) NOT NULL,
  FOREIGN KEY (EmployeeId) REFERENCES Employees(Id)
);
```

## 20) OpenAPI Contract Snippets (OAS 3.0)
```
paths:
  /customers:
    get:
      summary: List customers
      parameters:
        - in: query
          name: status
          schema: { type: string }
        - in: query
          name: page[size]
          schema: { type: integer, default: 25 }
        - in: query
          name: page[number]
          schema: { type: integer, default: 1 }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: "#/components/schemas/Customer"
    post:
      summary: Create a new customer
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/CustomerCreateRequest"
      responses:
        "201":
          description: Created
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Customer"
  /customers/{id}:
    get:
      summary: Get customer by Id
      parameters:
        - in: path
          name: id
          required: true
          schema: { type: string, format: uuid }
      responses:
        "200":
          description: OK
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/Customer"
				
components:
  schemas:
    Customer:
      type: object
      properties:
        id: { type: string, format: uuid }
        name: { type: string }
        primaryEmail: { type: string, format: email }
        status: { type: string }
        createdAtUtc: { type: string, format: date-time }

    CustomerCreateRequest:
      type: object
      required: [name, primaryEmail]
      properties:
        name: { type: string }
        primaryEmail: { type: string, format: email }
```

## 21) Solution / Folder Structure
```
src/
  Api/                      # ASP.NET Core entrypoint (controllers, filters, ProblemDetails middleware)
    Controllers/
    Filters/
    Middlewares/
  Application/              # CQRS layer (MediatR commands, queries, validators)
    Commands/
    Queries/
    Behaviors/
  Domain/                   # Entities, Aggregates, Value Objects, Domain Events
    Customers/
    Opportunities/
    Employees/
    Leaves/
  Infrastructure/           # EF Core, repositories, service bus, caching, outbox
    Persistence/
    Messaging/
    Configurations/
  Shared/                   # Cross-cutting abstractions (IClock, CorrelationId, base classes)

tests/
  UnitTests/                # Domain + Application tests
  IntegrationTests/         # EF Core + API tests with Testcontainers
  ContractTests/            # OpenAPI + Webhook signature tests

build/
  pipelines/                # CI/CD templates, infra as code (Bicep/Terraform)
```

## 22) Example Payloads (for Testing & Cursor)
Request
```json
POST /api/v1/customers
{
  "name": "Acme Corporation",
  "primaryEmail": "info@acme.com"
}
```
Response
```json
{
  "id": "c2b2f4a2-7d4a-44a0-9d92-2f9e12a7f4e0",
  "name": "Acme Corporation",
  "primaryEmail": "info@acme.com",
  "status": "Active",
  "createdAtUtc": "2025-08-20T17:42:00Z"
}
```
