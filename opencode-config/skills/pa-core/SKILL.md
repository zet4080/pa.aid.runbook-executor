---
name: pa-core
description: Use when configuring Pa.Core.* .NET libraries, baseapp Helm chart values, Program.cs registration, NuGet packages, or values.yaml mappings
---
# Skill: pa-core

## Purpose

Quick reference for Pa.Core.* .NET libraries and baseapp Helm chart: NuGet package setup, Program.cs registration, configuration properties, and values.yaml mapping.

## When to Use

- Adding or configuring Pa.Core.* NuGet packages
- Setting up Program.cs or Startup.cs for service using pa.core
- Writing or editing values.yaml for baseapp Helm chart
- Mapping Helm parameters to .NET configuration

---

## Pa.Core Library Reference

Each entry covers: NuGet package name, purpose, `Program.cs` registration, configuration class + `Position` key, all configuration properties, and which `values.yaml` keys inject them automatically.

---

### `Pa.Core.AccessAndEntitlement`

**Purpose:** JWT authentication and entitlement-based authorization against proALPHA Access Management.

**Program.cs registration:**
```csharp
// Service registration
services.UseAccessAndEntitlement(configuration);

// Middleware pipeline (order matters)
app.UseAuthentication();
app.UseAccessAndEntitlement();
app.UseAuthorization();

// Define authorization policies
services.AddAuthorizationBuilder()
    .AddPolicy("canReadData", policy =>
        policy.Requirements.Add(new AuthorizationRequirement("canReadData")));
```

**Configuration class:** `AccessAndEntitlementConfiguration`
**Position key:** `"AccessAndEntitlementConfiguration"`

| Property | Type | Required | Default | Notes |
|---|---|---|---|---|
| `AccessManagementBaseAddress` | `string` | **required** | — | Base URL of the Access Management service. Must be a valid URI. |
| `JwtAuthority` | `string` | **required** | — | Cognito User Pool URL, e.g. `https://cognito-idp.<region>.amazonaws.com/<pool-id>`. Must be a valid URI. |
| `EnableCaching` | `bool` | optional | `true` | Enables in-memory caching of entitlement checks. |

**appsettings.json example:**
```json
{
  "AccessAndEntitlementConfiguration": {
    "AccessManagementBaseAddress": "https://access-management.example.com",
    "JwtAuthority": "https://cognito-idp.eu-central-1.amazonaws.com/eu-central-1_abc123",
    "EnableCaching": true
  }
}
```

**Helm injection:**
- `AccessAndEntitlementConfiguration__JwtAuthority` ← `authentication.issuer` *(automatic)*
- `AccessAndEntitlementConfiguration__AccessManagementBaseAddress` — **must be set manually** via `env:` or `secrets`

---

### `Pa.Core.ApiVersioning`

**Purpose:** Standardised API versioning and Swagger/OpenAPI setup using Asp.Versioning + Swashbuckle.

**Program.cs registration:**
```csharp
services.AddSwaggerAndApiVersioning(
    extendedSwaggerAnnotations: false,  // optional
    forceVersionInApiRoutes: false,     // optional
    useSystemTextJson: false            // optional
);
```

**Configuration:** None — no configuration class, no appsettings keys. Purely code-configured.

**Helm injection:** None.

---

### `Pa.Core.Boot`

**Purpose:** HTTPS/TLS setup including installation of root CA certificates for outgoing calls.

**Program.cs registration:**
```csharp
builder.WebHost.InstallRootCertificate();
```

**Configuration class:** `WebServerConfiguration`
**Position key:** `"WebServerConfiguration"`

| Property | Type | Required | Default | Notes |
|---|---|---|---|---|
| `RootCertificatePath` | `string` | **required** | — | Path to the root CA certificate file. |
| `HttpsPort` | `int` | optional | — | HTTPS port. Injected automatically by Helm. |
| `CertificatePath` | `string` | optional | — | Path to the TLS certificate. Injected automatically by Helm. |

**Helm injection (all automatic):**

| Env Var | Value |
|---|---|
| `WebServerConfiguration__HttpsPort` | `service.internalPort` (default: `5443`) |
| `WebServerConfiguration__CertificatePath` | `/app/cert/tls.crt` (hardcoded) |
| `WebServerConfiguration__RootCertificatePath` | `/app/cert/ca.crt` (hardcoded) |
| `Kestrel__Endpoints__Https__Url` | `https://*:<service.internalPort>` |
| `Kestrel__Endpoints__Https__Certificate__Path` | `/app/cert/tls.crt` (hardcoded) |
| `Kestrel__Endpoints__Https__Certificate__KeyPath` | `/app/cert/tls.key` (hardcoded) |

**Local development appsettings.json:**
```json
{
  "Kestrel": {
    "Endpoints": {
      "Https": {
        "Url": "https://*:8443",
        "Certificate": {
          "Path": "../certs/service.crt",
          "KeyPath": "../certs/service.key"
        }
      }
    }
  },
  "WebServerConfiguration": {
    "RootCertificatePath": "../certs/rootCA.crt"
  }
}
```

---

### `Pa.Core.EFCore`

**Purpose:** PostgreSQL database access using EF Core with multi-tenancy, RDS IAM auth, and read/write separation.

**Program.cs registration:**
```csharp
services.AddPaDbContext<YourDbContext>(configuration);

// Unit of Work pattern
services.AddScoped<IUnitOfWork>(provider => provider.GetRequiredService<YourDbContext>());
```

**DbContext base classes:**
- `PostgresWriteContext` — writable DbContext, implements `IUnitOfWork`
- `PostgresReadContext` — read-only DbContext, `NoTracking` enabled by default

**Configuration class:** `DatabaseConfiguration`
**Position key:** `"DatabaseConfiguration"`

| Property | Type | Required | Default | Notes |
|---|---|---|---|---|
| `ApplicationName` | `string` | **required** | — | Name of the service; used as DB application name. |
| `ConnectionStringWrite` | `string` | **required** | — | PostgreSQL connection string for write operations. |
| `ConnectionStringRead` | `string` | **required** | — | PostgreSQL connection string for read operations. |
| `ConnectionMethod` | `enum` | optional | `PooledConnectionString` | See values below. |

**`ConnectionMethod` enum values:**

| Value | Behaviour |
|---|---|
| `PooledConnectionString` | Pooled + multi-tenancy via connection string |
| `PooledCommand` | Pooled + multi-tenancy via command *(recommended for RDS IAM)* |
| `ConnectionString` | Non-pooled + multi-tenancy via connection string |
| `Command` | Non-pooled + multi-tenancy via command |

> **RDS IAM token:** Use `{{RDSTOKEN}}` as the password placeholder in connection strings. It is replaced at runtime with a live IAM auth token.

> **Breaking change (chart 3.0.0):** `DatabaseConfiguration__ConnectionString` (single string) is no longer supported. Always use `ConnectionStringRead` and `ConnectionStringWrite`.

**appsettings.json example:**
```json
{
  "DatabaseConfiguration": {
    "ApplicationName": "my-service",
    "ConnectionStringRead": "Host=localhost;Port=5432;Username=postgres;Password=postgres;Database=mydb;Pooling=true",
    "ConnectionStringWrite": "Host=localhost;Port=5432;Username=postgres;Password=postgres;Database=mydb;Pooling=true",
    "ConnectionMethod": "PooledCommand"
  }
}
```

**Helm injection** — requires `databaseconfiguration.enabled: true`:

| Env Var | Source |
|---|---|
| `DatabaseConfiguration__ApplicationName` | service account name |
| `DatabaseConfiguration__ConnectionStringWrite` | built from `databaseconfiguration.*` values |
| `DatabaseConfiguration__ConnectionStringRead` | built from `databaseconfiguration.*` values |

**DbContext implementation example:**
```csharp
public class MyDbContext : PostgresWriteContext
{
    public MyDbContext() : this(new DbContextOptionsBuilder<MyDbContext>().Options) { }
    public MyDbContext(DbContextOptions<MyDbContext> options) : base(options) { }

    public DbSet<MyEntity> MyEntities { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<MyEntity>(e =>
            e.ToTable("my_entities").AddTenantShadowPropertyToModel());
    }
}
```

---

### `Pa.Core.ErrorHandling`

**Purpose:** Standardised error response formatting middleware and controller extension methods using `CSharpFunctionalExtensions`.

**Program.cs registration:**
```csharp
app.UseErrorHandling(); // must be early in the middleware pipeline
```

**Configuration:** None — no configuration class, no appsettings keys.

**Controller usage:**
```csharp
// Return an error directly
return this.ErrorResult(new InvalidArgumentResponse("reason", details));

// Return from Result<T, IErrorResponse>
return this.FromObjectResult(result);  // returns T payload or error response
return this.FromResult(result);         // returns empty 200 or error response
```

---

### `Pa.Core.HealthCheck`

**Purpose:** Custom health checks, including TLS certificate validity checks.

**Program.cs registration:**
```csharp
services.AddHealthChecks()
    .AddCertificateValidityHealthCheck(certificates, delay: TimeSpan.FromSeconds(10));
```

**Configuration:** None — no configuration class, no appsettings keys.

**Helm:** `livenessProbe.livenessPath` defaults to `/health` (port 443).

---

### `Pa.Core.Identity`

**Purpose:** Provides `IUserIdentity` to access the current user's tenant ID, user ID, and JWT token from DI.

**Program.cs registration:** None required when used with `Pa.Core.AccessAndEntitlement` — `IUserIdentity` is registered automatically.

**Usage:**
```csharp
public class MyService(IUserIdentity userIdentity)
{
    var tenantId = userIdentity.TenantId;
    var userId   = userIdentity.UserId;
    var token    = userIdentity.JwtToken;
}
```

**Configuration:** None — no configuration class, no appsettings keys.

---

### `Pa.Core.Logging`

**Purpose:** Serilog-based structured JSON logging with proALPHA conventions.

**Program.cs registration:**
```csharp
builder.Host.UsePaLogging();
// Add this before ConfigureWebHostDefaults or as a Host builder extension
```

**Configuration:** No dedicated configuration class. Uses the standard `Serilog` section in appsettings.json. The library auto-applies the console sink with structured JSON output — no override needed unless customising.

**Helm injection:**
- `Logging__LogLevel__Default` ← `global.loglevel`

**Default Serilog configuration (applied automatically by the library):**
```json
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "Microsoft.Hosting.Lifetime": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "formatter": {
            "type": "Serilog.Templates.ExpressionTemplate, Serilog.Expressions",
            "template": "{ {timestamp: ToString(UtcDateTime(@t),'yyyy-MM-ddTHH:mm:ss.fffZ'), logLevel: @l, message: {text: @m, exception: @x}, SourceContext} }\n"
          }
        }
      }
    ],
    "Enrich": ["FromLogContext"]
  }
}
```

> **Note:** If only `MinimumLevel` is specified in the `Serilog` section without a sink, the library automatically adds the console sink above.

---

### `Pa.Core.MessageBus`

**Purpose:** AWS SQS/SNS messaging via MassTransit with multi-tenancy (publish-subscribe pattern).

**Program.cs registration:**
```csharp
// With consumers (receive endpoint enabled)
services.UseBus(configuration, new Type[] { typeof(MyConsumer) });

// With custom MassTransit config — use with caution, can break multi-tenancy
services.UseBus(configuration, new Type[] { typeof(MyConsumer) }, cfg => { /* custom config */ });

// Publisher-only (no receive endpoint)
services.UseBus(configuration, Array.Empty<Type>());
```

**Configuration class:** `MessageBusConfiguration`
**Position key:** `"MessageBusConfiguration"`

| Property | Type | Required | Default | Notes |
|---|---|---|---|---|
| `EndpointName` | `string` | optional | — | SQS queue name. Omit entirely to disable receive endpoint (publish-only mode). |
| `CognitoJwtAuthority` | `string` | **required** | — | Cognito URL for JWT validation of incoming messages. |
| `AmazonSQSHostOverride` | `string` | optional | — | Override SQS host (e.g. for LocalStack local debugging). |
| `EnableLocalDebugging` | `bool` | optional | — | Routes to `AmazonSQSHostOverride` when `true`. |
| `DisableTokenValidation` | `bool` | optional | — | Disables JWT validation. **Local debug only — never enable in production.** |
| `PersistenceConfiguration.UsePersistence` | `bool` | optional | — | Enables message persistence/auditing. |
| `PersistenceConfiguration.AuditConnectionString` | `string` | optional | — | DB connection string for the audit log. |

**Required environment variable:** `AWS_REGION`

**Helm injection** — requires `busconfiguration.enabled: true`:

| Env Var | Source |
|---|---|
| `MessageBusConfiguration__EndpointName` | `busconfiguration.endpointname` |
| `MessageBusConfiguration__CognitoJwtAuthority` | `authentication.issuer` |
| `MessageBusConfiguration__PersistenceSettings__UsePersistence` | `busconfiguration.persistence.enabled` |
| `AWS_REGION` | `aws.region` |

**Minimal values.yaml:**
```yaml
busconfiguration:
  enabled: true
  endpointname: "my-service"
```

**Consumer implementation example:**
```csharp
public sealed class MyMessageConsumer : IConsumer<MyMessage>
{
    public async Task Consume(ConsumeContext<MyMessage> context)
    {
        // handle message
    }
}
```

> **Note:** Message objects must implement `CorrelatedBy<T>` for correlation ID support.

---

### `Pa.Core.Redis`

**Purpose:** Redis/ElastiCache distributed cache integration with TokenLib IAM authentication.

**Configuration class:** `RedisConfiguration`
**Position key:** `"RedisConfiguration"`

| Property | Type | Required | Default | Notes |
|---|---|---|---|---|
| `ApplicationName` | `string` | **required** | — | Used as cache key prefix/namespace. |
| `CacheName` | `string` | **required** | — | Name of the Redis cache. |
| `ClusterAddress` | `string` | **required** | — | Redis cluster endpoint. |
| `IsServerless` | `bool` | optional | — | Set `true` for AWS ElastiCache Serverless. |
| `EnableLocalDebugging` | `bool` | optional | — | For local development. |

**Helm injection** — requires `redis.enabled: true`:

| Env Var | Source |
|---|---|
| `RedisConfiguration__ApplicationName` | service account name |
| `RedisConfiguration__CacheName` | `redis.cachename` |
| `RedisConfiguration__ClusterAddress` | `redis.clusteraddress` |
| `RedisConfiguration__IsServerless` | `redis.serverless` |

**Minimal values.yaml:**
```yaml
redis:
  enabled: true
  cachename: "my-cache"
  clusteraddress: "my-cache.abc.use1.cache.amazonaws.com"
  serverless: false
```

---

### `Pa.Core.Telemetry`

**Purpose:** OpenTelemetry metrics export via OTLP.

**Program.cs registration:**
```csharp
services.AddPaOpenTelemetry(configuration);
```

**Configuration class:** `OpenTelemetry`
**Position key:** `"OpenTelemetry"`

| Property | Type | Required | Default | Notes |
|---|---|---|---|---|
| `Endpoint` | `string` | **required** | — | OTLP collector endpoint, e.g. `http://otel-collector:4317`. |
| `ServiceName` | `string` | **required** | — | Name of the service used for telemetry labelling. |

**appsettings.json example:**
```json
{
  "OpenTelemetry": {
    "Endpoint": "http://otel-collector:4317",
    "ServiceName": "my-service"
  }
}
```

**Helm injection:** No automatic injection. Must be set manually via `env:`:
```yaml
env:
  OpenTelemetry__Endpoint: "http://otel-collector:4317"
  OpenTelemetry__ServiceName: "my-service"
```

---

## baseapp Helm Chart Reference

The `baseapp` chart (`ccoe/base.charts.helm/baseapp`) is the standard base Helm chart for all proALPHA .NET microservices. It is used as a chart dependency, not deployed standalone.

---

### Full `values.yaml` Structure

```yaml
global:
  environment: Production       # → ASPNETCORE_ENVIRONMENT
  loglevel: "warning"           # → Logging__LogLevel__Default

image:
  repository: ""
  tag: ""
  pullPolicy: IfNotPresent

replicaCount: 1
revisionHistoryLimit: 3

serviceAccount:
  create: true
  iamRoleArn: ""                # MUST be overridden — IAM role ARN for pod identity

authentication:
  issuer: ""                    # Cognito User Pool URL — required for AccessAndEntitlement + MessageBus
  tokenvendorendpoint: ""       # TokenLib endpoint → TokenLib__TokenEndpoint

databaseconfiguration:
  enabled: false
  usemigration: false
  migrationrole: ""
  migrationuser: "migration-user-rdsiam"
  database: ""
  searchpath: "public"
  additional: "Pooling=true"
  connection:                   # if set, overrides RDS IAM-based connection string
  migrationconnection:          # if set, overrides migration connection string

busconfiguration:
  enabled: false
  endpointname: ""
  persistence:
    enabled: false

redis:
  enabled: false
  cachename: ""
  clusteraddress: ""
  serverless: false

aws:
  region: eu-central-1         # → AWS_REGION

service:
  type: ClusterIP
  port: 443
  targetPort: https
  internalPort: 5443            # → WebServerConfiguration__HttpsPort + Kestrel HTTPS port

certificate:
  issuer: app-issuer

configmap:
  restartpod: true
  app: {}                       # dict: arbitrary key/value → injected as env vars into the container
  externals: []                 # list of existing ConfigMap names → all keys injected as env vars
  featureFlags:
    mount: false                # mount feature-management ConfigMap as a JSON file
    mountPath: /app/config/feature_flags.json
  extraMounts: []               # list of {name, mountPath, subPath, configMapName}

secrets: []                     # list of Kubernetes Secret names → all keys injected as env vars

env: {}                         # dict of arbitrary env vars added directly to the container

autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80

livenessProbe:
  enable: true
  livenessPath: /health         # default health check path (port 443)
  periodSeconds: 60
  initialDelaySeconds: 20

ingress:
  enabled: false
  yarp:                         # YARP reverse proxy annotations
    session-affinity: ...
```

---

### Automatic Env Var Injection — Complete Map

The baseapp chart uses `__` (double underscore) as the .NET configuration hierarchy separator.

| Env Var | Source in values.yaml | Library |
|---|---|---|
| `ASPNETCORE_ENVIRONMENT` | `global.environment` | ASP.NET Core |
| `Logging__LogLevel__Default` | `global.loglevel` | `Pa.Core.Logging` |
| `AWS_REGION` | `aws.region` | `Pa.Core.MessageBus` |
| `TokenLib__TokenEndpoint` | `authentication.tokenvendorendpoint` | TokenLib |
| `AccessAndEntitlementConfiguration__JwtAuthority` | `authentication.issuer` | `Pa.Core.AccessAndEntitlement` |
| `WebServerConfiguration__HttpsPort` | `service.internalPort` | `Pa.Core.Boot` |
| `WebServerConfiguration__CertificatePath` | `/app/cert/tls.crt` (hardcoded) | `Pa.Core.Boot` |
| `WebServerConfiguration__RootCertificatePath` | `/app/cert/ca.crt` (hardcoded) | `Pa.Core.Boot` |
| `Kestrel__Endpoints__Https__Url` | `https://*:<service.internalPort>` | ASP.NET Core |
| `Kestrel__Endpoints__Https__Certificate__Path` | `/app/cert/tls.crt` (hardcoded) | ASP.NET Core |
| `Kestrel__Endpoints__Https__Certificate__KeyPath` | `/app/cert/tls.key` (hardcoded) | ASP.NET Core |
| `DatabaseConfiguration__ApplicationName` | service account name | `Pa.Core.EFCore` |
| `DatabaseConfiguration__ConnectionStringWrite` | built from `databaseconfiguration.*` | `Pa.Core.EFCore` |
| `DatabaseConfiguration__ConnectionStringRead` | built from `databaseconfiguration.*` | `Pa.Core.EFCore` |
| `MessageBusConfiguration__EndpointName` | `busconfiguration.endpointname` | `Pa.Core.MessageBus` |
| `MessageBusConfiguration__CognitoJwtAuthority` | `authentication.issuer` | `Pa.Core.MessageBus` |
| `MessageBusConfiguration__PersistenceSettings__UsePersistence` | `busconfiguration.persistence.enabled` | `Pa.Core.MessageBus` |
| `RedisConfiguration__ApplicationName` | service account name | `Pa.Core.Redis` |
| `RedisConfiguration__CacheName` | `redis.cachename` | `Pa.Core.Redis` |
| `RedisConfiguration__ClusterAddress` | `redis.clusteraddress` | `Pa.Core.Redis` |
| `RedisConfiguration__IsServerless` | `redis.serverless` | `Pa.Core.Redis` |

---

### Injecting Custom Configuration

For any configuration key not automatically injected by the chart, use one of these four mechanisms:

#### 1. `env` — inject key/value pairs directly as container environment variables *(preferred)*
The preferred way to set additional .NET config keys not wired automatically by the chart. Values are set directly on the container without creating any additional Kubernetes objects.
```yaml
env:
  OpenTelemetry__Endpoint: "http://otel-collector:4317"
  OpenTelemetry__ServiceName: "my-service"
  AccessAndEntitlementConfiguration__AccessManagementBaseAddress: "https://am.example.com"
  AccessAndEntitlementConfiguration__EnableCaching: "false"
```

#### 2. `secrets` — reference a Kubernetes Secret (all keys become env vars)
```yaml
secrets:
  - my-service-secrets
```

#### 3. `configmap.externals` — reference an externally managed ConfigMap (e.g. from Terraform)
References **existing** ConfigMaps created outside the chart (e.g. by Terraform) and mounts them as `envFrom`.
```yaml
configmap:
  externals:
    - shared-platform-config
```

#### 4. `configmap.app` — creates a ConfigMap object with the given key/values, injected as envFrom
Valid but less direct than `env:`. Useful when you want the config values visible as a named ConfigMap in the cluster.
```yaml
configmap:
  app:
    SomeSection__SomeKey: "value"
```

---

### Breaking Change: Chart 3.0.0

`DatabaseConfiguration__ConnectionString` (single connection string) is **no longer supported**.

Always use both:
- `DatabaseConfiguration__ConnectionStringRead`
- `DatabaseConfiguration__ConnectionStringWrite`

---

## Quick-Start: Canonical `Program.cs`

Full `Program.cs` using all pa.core libraries in the correct registration order:

```csharp
var builder = WebApplication.CreateBuilder(args);

// 1. Logging — add first so all subsequent setup is logged
builder.Host.UsePaLogging();

// 2. TLS/HTTPS — install root CA for outgoing calls
builder.WebHost.InstallRootCertificate();

// 3. API versioning + Swagger
builder.Services.AddSwaggerAndApiVersioning(extendedSwaggerAnnotations: true);

// 4. Authentication + Authorization
builder.Services.UseAccessAndEntitlement(builder.Configuration);

builder.Services.AddAuthorizationBuilder()
    .AddPolicy("canReadData", policy =>
        policy.Requirements.Add(new AuthorizationRequirement("canReadData")));

// 5. Database (EF Core + PostgreSQL)
builder.Services.AddPaDbContext<MyDbContext>(builder.Configuration);
builder.Services.AddScoped<IUnitOfWork>(p => p.GetRequiredService<MyDbContext>());

// 6. Message Bus (SQS/SNS via MassTransit)
builder.Services.UseBus(builder.Configuration, new[] { typeof(MyConsumer) });

// 7. Telemetry (OpenTelemetry / OTLP)
builder.Services.AddPaOpenTelemetry(builder.Configuration);

var app = builder.Build();

// Middleware pipeline — order matters
app.UseErrorHandling();         // must be early

app.UseAuthentication();
app.UseAccessAndEntitlement();
app.UseAuthorization();

app.MapControllers();
app.Run();
```

---

## Configuration Properties — Quick-Reference Summary

| Library | Config Class | Position Key | Required Properties |
|---|---|---|---|
| `Pa.Core.AccessAndEntitlement` | `AccessAndEntitlementConfiguration` | `"AccessAndEntitlementConfiguration"` | `AccessManagementBaseAddress`, `JwtAuthority` |
| `Pa.Core.ApiVersioning` | — | — | *(none)* |
| `Pa.Core.Boot` | `WebServerConfiguration` | `"WebServerConfiguration"` | `RootCertificatePath` |
| `Pa.Core.EFCore` | `DatabaseConfiguration` | `"DatabaseConfiguration"` | `ApplicationName`, `ConnectionStringWrite`, `ConnectionStringRead` |
| `Pa.Core.ErrorHandling` | — | — | *(none)* |
| `Pa.Core.HealthCheck` | — | — | *(none)* |
| `Pa.Core.Identity` | — | — | *(none)* |
| `Pa.Core.Logging` | — | *(Serilog section)* | *(none — auto-configured)* |
| `Pa.Core.MessageBus` | `MessageBusConfiguration` | `"MessageBusConfiguration"` | `CognitoJwtAuthority` + env `AWS_REGION` |
| `Pa.Core.Redis` | `RedisConfiguration` | `"RedisConfiguration"` | `ApplicationName`, `CacheName`, `ClusterAddress` |
| `Pa.Core.Telemetry` | `OpenTelemetry` | `"OpenTelemetry"` | `Endpoint`, `ServiceName` |

---

## Helm values.yaml — Which Block Enables Which Library

| values.yaml block | `enabled` flag needed | Library / effect |
|---|---|---|
| `databaseconfiguration` | `enabled: true` | `Pa.Core.EFCore` — injects all `DatabaseConfiguration__*` env vars |
| `busconfiguration` | `enabled: true` | `Pa.Core.MessageBus` — injects all `MessageBusConfiguration__*` env vars |
| `redis` | `enabled: true` | `Pa.Core.Redis` — injects all `RedisConfiguration__*` env vars |
| `authentication.issuer` | *(always active when set)* | `Pa.Core.AccessAndEntitlement` + `Pa.Core.MessageBus` |
| `global.loglevel` | *(always active when set)* | `Pa.Core.Logging` |
| `env` | *(always active)* | Preferred manual injection for Telemetry, AM base address, and other service-specific config keys |
