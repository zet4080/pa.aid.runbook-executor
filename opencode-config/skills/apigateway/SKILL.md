---
name: apigateway
description: Use when configuring proALPHA API Gateway YARP routes, authentication, rate limiting, CORS, Swagger aggregation, maintenance mode, or appsettings/values mappings
---

# Skill: API Gateway Configuration

## Purpose

Quick reference for configuring proALPHA API Gateway: YARP routes, authentication (JWT Bearer), CORS, rate limiting, Swagger aggregation, maintenance mode, appsettings/values mappings.

## When to Use

- Configuring routes for new or existing backend services
- Setting up authentication, CORS, rate limiting
- Setting up Swagger/OpenAPI aggregation
- Enabling maintenance mode or custom error pages
- Generating appsettings.json for deployment
- Configuring SSL/TLS

---

## Repository Map

| Repository | Workspace Path | Purpose |
|------------|---------------|---------|
| `pa.routing.apigateway.cs` | `/workspace/inthub/pa.routing.apigateway.cs` | API Gateway application |
| `pa.routing.swagger.cs` | `/workspace/inthub/pa.routing.swagger.cs` | Custom Swagger aggregation library |

**Key files:**

| File | Path | Purpose |
|------|------|---------|
| `appsettings.json` | `src/Pa.ApiGateway/appsettings.json` | Main configuration template |
| `GatewayExtensions.cs` | `src/Pa.ApiGateway/GatewayExtensions.cs` | Service registration logic |
| `WebApplicationExtensions.cs` | `src/Pa.ApiGateway/WebApplicationExtensions.cs` | Middleware pipeline |
| `ApiGatewayConfiguration.cs` | `src/Pa.ApiGateway/Configuration/` | Root config model |
| `AuthenticationConfiguration.cs` | `src/Pa.ApiGateway/Configuration/` | Auth config model |
| `RateLimitConfiguration.cs` | `src/Pa.ApiGateway/Configuration/` | Rate limit config model |
| Helm chart | `helm/apigateway/` | Kubernetes deployment values |

---

## Architecture Overview

```
Client Request
      │
      ▼
┌─────────────────────────────────────────────────┐
│               API Gateway (YARP)                 │
│                                                  │
│  ┌─────────────┐   ┌──────────────────────────┐ │
│  │  Maintenance │   │  Error Page Middleware    │ │
│  │  Middleware  │   │  (502/503/504 → HTML)     │ │
│  └─────────────┘   └──────────────────────────┘ │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │          YARP Reverse Proxy               │   │
│  │  Routes → Clusters → Destinations        │   │
│  │  + Authentication + CORS + Rate Limits   │   │
│  └──────────────────────────────────────────┘   │
│                                                  │
│  ┌──────────────────────────────────────────┐   │
│  │   Swagger Aggregation (Pa.Routing.Swagger)│   │
│  │   Fetches & merges downstream Swagger    │   │
│  └──────────────────────────────────────────┘   │
└─────────────────────────────────────────────────┘
         │                        │
    Backend Services         SwaggerUI
```

---

## Configuration Output Format

The output is always an **`appsettings.json`** file. The full schema:

```json
{
  "RootCertificatePath": "./certs/rootCA.crt",
  "Cors": {
    "default": [
      "https://app.example.com",
      "https://localhost:4200"
    ]
  },
  "Authentication": {
    "myPolicy": {
      "Issuer": "https://cognito-idp.eu-central-1.amazonaws.com/eu-central-1_XXXXX",
      "Audience": "my-client-id"
    }
  },
  "RateLimits": {
    "myRatePolicy": {
      "Type": "FixedWindow",
      "PermitLimit": 100,
      "WindowSeconds": 60
    }
  },
  "ReverseProxy": {
    "Routes": {
      "MyService": {
        "ClusterId": "MyService",
        "SwaggerKey": "MyService",
        "AuthorizationPolicy": "myPolicy",
        "Match": {
          "Path": "myservice/{**catch-all}"
        },
        "Transforms": [
          { "PathPattern": "api/{**catch-all}" }
        ]
      }
    },
    "Clusters": {
      "MyService": {
        "Destinations": {
          "Default": {
            "Address": "http://my-backend-service:8080"
          }
        }
      }
    }
  },
  "SwaggerConfig": {
    "BaseUrl": "api.example.com",
    "Services": {
      "MyService": {
        "Swaggers": {
          "MyService-v1": {
            "URL": "/swagger/v1/swagger.json",
            "PrefixPath": "/myservice"
          }
        }
      }
    }
  },
  "Kestrel": {
    "Endpoints": {
      "Http": { "Url": "http://*:8000" }
    }
  }
}
```

---

## Section-by-Section Reference

### 1. Root-Level Settings

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `EnableIngressController` | `bool` | `false` | Enable Kubernetes Ingress Controller mode (uses YARP K8s integration) |
| `EnableGlobalMaintenance` | `bool` | `false` | Serve 503 maintenance page for all requests (except `/health`, `/static`) |
| `RootCertificatePath` | `string` | `null` | Path to a root CA certificate to trust for upstream TLS connections |

---

### 2. CORS (`Cors`)

Dictionary of named CORS policies. The key `"default"` is **required**.

```json
"Cors": {
  "default": [
    "https://app.example.com",
    "https://localhost:4200"
  ],
  "internal": [
    "https://admin.example.com"
  ]
}
```

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `Cors.default` | `string[]` | **Yes** | Allowed origins. This policy is automatically applied as the ASP.NET Core default CORS policy — **all routes use it unless a different `CorsPolicy` is explicitly set on the route** |
| `Cors.{policyName}` | `string[]` | No | Additional named CORS policies for routes that need different origins |

> **Default behavior:** `app.UseCors()` is called without arguments, which applies the `"default"` CORS policy globally. A route only needs `CorsPolicy` set when it must use a **different** named policy.

> All CORS policies are configured with: `AllowAnyHeader`, `AllowAnyMethod`, `AllowCredentials`, `WithExposedHeaders("x-total-count")`, and wildcard subdomain support.

---

### 3. Authentication (`Authentication`)

Dictionary of named JWT Bearer authentication/authorization policies.

```json
"Authentication": {
  "myAuthPolicy": {
    "Issuer": "https://cognito-idp.eu-central-1.amazonaws.com/eu-central-1_XXXXX",
    "Audience": "my-app-client-id"
  }
}
```

| Property | Type | Default | Required | Description |
|----------|------|---------|----------|-------------|
| `Type` | `string` | `"Bearer"` | No | Authentication scheme type. Currently only `"Bearer"` is supported |
| `Issuer` | `string` | — | **Yes** | JWT token issuer URL (e.g. Cognito User Pool URL) |
| `Audience` | `string` | `null` | No | Expected JWT audience (app client ID) |
| `ValidateIssuer` | `bool` | `true` | No | Whether to validate the issuer claim |
| `ValidateAudience` | `bool` | `false` | No | Whether to validate the audience claim |

> **How it links to routes:** A route uses `"AuthorizationPolicy": "myAuthPolicy"` to require authentication.

> **Special `"default"` policy:** When an entry named `"default"` exists under `Authentication`, the gateway automatically sets it as the ASP.NET Core **FallbackPolicy** — every route that has no explicit `AuthorizationPolicy` set will require authentication. Use this to secure all routes by default.

---

### 4. Rate Limiting (`RateLimits`)

Dictionary of named rate limit policies.

```json
"RateLimits": {
  "defaultPolicy": {
    "Type": "FixedWindow",
    "PermitLimit": 100,
    "WindowSeconds": 60,
    "QueueLimit": 10
  },
  "strictPolicy": {
    "Type": "SlidingWindow",
    "PermitLimit": 20,
    "WindowSeconds": 10,
    "SegmentsPerWindow": 5
  },
  "tokenPolicy": {
    "Type": "TokenBucket",
    "TokenLimit": 50,
    "TokensPerPeriod": 10,
    "ReplenishmentPeriodSeconds": 5
  },
  "concurrentPolicy": {
    "Type": "ConcurrencyLimiter",
    "PermitLimit": 10,
    "QueueLimit": 5
  }
}
```

#### Supported Rate Limiter Types

| Type | Required Fields | Description |
|------|----------------|-------------|
| `FixedWindow` | `PermitLimit`, `WindowSeconds` | Max N requests per fixed time window |
| `SlidingWindow` | `PermitLimit`, `WindowSeconds`, `SegmentsPerWindow` | Sliding window with configurable segments |
| `TokenBucket` | `TokenLimit`, `TokensPerPeriod`, `ReplenishmentPeriodSeconds` | Token bucket algorithm |
| `Concurrency` / `ConcurrencyLimiter` | `PermitLimit` | Max N concurrent requests |

#### All Rate Limit Properties

| Property | Type | Applies To | Description |
|----------|------|-----------|-------------|
| `Type` | `string` | All | Limiter algorithm (see table above) |
| `PermitLimit` | `int` | FixedWindow, SlidingWindow, Concurrency | Max requests in window / concurrent |
| `WindowSeconds` | `int` | FixedWindow, SlidingWindow | Time window in seconds |
| `QueueLimit` | `int` | All | Max requests to queue when limit hit (0 = reject immediately) |
| `SegmentsPerWindow` | `int` | SlidingWindow | Number of segments to divide window (default: 1) |
| `TokenLimit` | `int` | TokenBucket | Max tokens in the bucket |
| `TokensPerPeriod` | `int` | TokenBucket | Tokens added per replenishment period |
| `ReplenishmentPeriodSeconds` | `int` | TokenBucket | Replenishment period in seconds |

> On rejection the gateway returns **HTTP 429** with body: `"Rate limit exceeded. Please try again later."`

> **How it links to routes:** Use the same policy name in `AuthorizationPolicy` on the route. Rate limiting and auth policies share the same policy name namespace.

---

### 5. YARP Routes (`ReverseProxy.Routes`)

Dictionary of named routes. Each route matches incoming requests and forwards them to a cluster.

```json
"ReverseProxy": {
  "Routes": {
    "DMS": {
      "ClusterId": "DMS",
      "SwaggerKey": "DMS",
      "AuthorizationPolicy": "myPolicy",
      "Match": {
        "Path": "dms/{**catch-all}",
        "Hosts": ["api.example.com"]
      },
      "Transforms": [
        { "PathPattern": "api/dms/{**catch-all}" }
      ],
      "Order": 0,
      "MaxRequestBodySize": 104857600
    }
  }
}
```

#### Route Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `ClusterId` | `string` | **Yes** | References the cluster to forward matching requests to |
| `SwaggerKey` | `string` | No | Links this route to a `SwaggerConfig.Services` entry for Swagger aggregation |
| `CorsPolicy` | `string` | No | Override the CORS policy for this route. **Omit this field to use the `"default"` CORS policy automatically.** Only set when a different named policy is needed |
| `AuthorizationPolicy` | `string` | No | Name of the auth/rate-limit policy to apply |
| `Match.Path` | `string` | Yes* | URL path pattern. Use `{**catch-all}` to match all sub-paths |
| `Match.Hosts` | `string[]` | No | Host-based routing (virtual hosting). Matches the `Host` header |
| `Transforms` | `array` | No | Request/response transforms (see Transforms section below) |
| `Order` | `int` | No | Route priority. Lower numbers match first |
| `MaxRequestBodySize` | `long` | No | Maximum request body size in bytes |
| `TimeoutPolicy` | `string` | No | Named timeout policy |
| `Timeout` | `string` | No | Direct timeout as TimeSpan string (e.g. `"00:01:30"`) |
| `Metadata` | `object` | No | Custom key-value metadata dictionary |

*One of `Match.Path` or `Match.Hosts` is required.

#### Common Path Patterns

| Pattern | Matches |
|---------|---------|
| `api/v1/{**catch-all}` | All paths starting with `api/v1/` |
| `myservice/{**catch-all}` | All paths starting with `myservice/` |
| `{**catch-all}` | All paths (catch-all route, use with `Order`) |
| `api/v1/users` | Only the exact path `api/v1/users` |

#### Transforms

Transforms modify the request before forwarding or modify the response.

**Path rewrite:**
```json
"Transforms": [
  { "PathPattern": "api/{**catch-all}" }
]
```
This rewrites the path. If route matches `myservice/foo/bar` with path pattern `myservice/{**catch-all}`, and the transform is `{ "PathPattern": "api/{**catch-all}" }`, the upstream receives `api/foo/bar`.

**Header transforms:**
```json
"Transforms": [
  { "RequestHeader": "X-Custom-Header", "Set": "my-value" },
  { "RequestHeaderRemove": "Cookie" }
]
```

**X-Forwarded headers:**
```json
"Transforms": [
  { "X-Forwarded": "Set", "For": "true", "Proto": "true", "Host": "true", "Prefix": "true" }
]
```

For the full list of YARP transforms see: https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/yarp/transforms

---

### 6. YARP Clusters (`ReverseProxy.Clusters`)

Dictionary of named clusters (backend service pools). Each cluster has one or more destinations.

```json
"ReverseProxy": {
  "Clusters": {
    "MyService": {
      "Destinations": {
        "Default": {
          "Address": "http://my-backend:8080"
        },
        "Replica2": {
          "Address": "http://my-backend-2:8080"
        }
      }
    }
  }
}
```

#### Cluster Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `Destinations` | `object` | **Yes** | Dictionary of named destination addresses |
| `Destinations.{name}.Address` | `string` | **Yes** | Backend URL. Supports `http://` and `https://` |
| `Destinations.{name}.AccessTokenClientName` | `string` | No | Named OAuth client to use for token-based auth to backend |

> Multiple destinations in a cluster enable **load balancing**. YARP uses round-robin by default.

---

### 7. Swagger Configuration (`SwaggerConfig`)

> 💡 **Ask the user for every service being configured:** *"Does this service expose a Swagger/OpenAPI definition? If so, should it be aggregated into the gateway's SwaggerUI?"*
>
> - If **yes**: add `SwaggerKey` to the route and configure a matching entry under `SwaggerConfig.Services` (see below).
> - If **no**: omit `SwaggerKey` from the route and do not add an entry under `SwaggerConfig.Services`.
>
> Also ask: *"Does the downstream Swagger URL use a path prefix that differs from what the gateway exposes?"* — if so, use `PrefixPath` and/or `RemovePrefix` to rewrite the paths.

Configures the Swagger aggregation layer. The gateway fetches Swagger JSON from downstream services, rewrites paths, and serves a unified SwaggerUI.

```json
"SwaggerConfig": {
  "BaseUrl": "api.example.com",
  "Services": {
    "DMS": {
      "Swaggers": {
        "DMS-v1": {
          "URL": "/swagger/v1/swagger.json",
          "PrefixPath": "/dms",
          "RemovePrefix": null
        }
      }
    },
    "Barcode": {
      "Swaggers": {
        "Barcode-internal": {
          "URL": "http://barcode-svc:8080/swagger/v1/swagger.json",
          "PrefixPath": "/barcode",
          "RemovePrefix": "/api/barcode"
        }
      }
    }
  }
}
```

#### SwaggerConfig Properties

| Property | Type | Required | Description |
|----------|------|----------|-------------|
| `BaseUrl` | `string` | No | Filter SwaggerUI endpoints by request hostname. Only serves swagger when request host matches this value |
| `Services.{key}` | `object` | — | Service group. The key must match the `SwaggerKey` on the corresponding route |
| `Services.{key}.Swaggers.{name}.URL` | `string` | **Yes** | URL to fetch the downstream Swagger JSON. Can be relative (fetched via the gateway itself) or absolute |
| `Services.{key}.Swaggers.{name}.PrefixPath` | `string` | No | Path prefix to prepend to all paths in the downstream Swagger document |
| `Services.{key}.Swaggers.{name}.RemovePrefix` | `string` | No | Path prefix to strip from all downstream Swagger paths before adding `PrefixPath` |

#### Path Rewriting Logic

The downstream swagger path is rewritten as:
```
final_path = PrefixPath + (downstream_path with RemovePrefix stripped)
```

**Example:**
- Downstream path: `/api/barcode/scan`
- `RemovePrefix`: `/api/barcode`
- `PrefixPath`: `/barcode`
- Result: `/barcode/scan`

#### Linking Routes to Swagger

The `SwaggerKey` on a route **must match** the key in `SwaggerConfig.Services`. If they don't match, a `SwaggerKeyMismatchException` is thrown.

```json
"Routes": {
  "DMS": {
    "SwaggerKey": "DMS",   // ← must match SwaggerConfig.Services key
    ...
  }
},
"SwaggerConfig": {
  "Services": {
    "DMS": { ... }         // ← this key
  }
}
```

---

### 8. Static Content (`StaticContent`)

Configuration for serving static files (HTML pages for maintenance/error pages).

```json
"StaticContent": {
  "Path": "StaticContent",
  "RequestPath": "/static"
}
```

| Property | Type | Default | Description |
|----------|------|---------|-------------|
| `Path` | `string` | `"StaticContent"` | File system path to the static files directory |
| `RequestPath` | `string` | `"/static"` | URL path prefix for serving static files |

---

### 9. Kestrel Configuration

```json
"Kestrel": {
  "Endpoints": {
    "Http": {
      "Url": "http://*:8000"
    },
    "Https": {
      "Url": "https://*:8023",
      "Certificate": {
        "Path": "./certs/server.crt",
        "KeyPath": "./certs/server.key"
      }
    }
  }
}
```

---

### 10. SSL / TLS Configuration

> ⚠️ **Always ask the user whether SSL/TLS is needed before generating configuration.** TLS affects both the inbound (gateway) and outbound (cluster destinations) sides independently.

#### 10.1 Inbound TLS — Exposing the Gateway over HTTPS

To have the gateway accept HTTPS connections, configure a Kestrel HTTPS endpoint **and** provide a valid server certificate. The certificate must be in PEM format (separate `.crt` / `.key` files).

> ⚠️ **Inform the user before enabling HTTPS:** A server certificate and its private key must be available on the host/container at the configured paths before the gateway starts. Without them the gateway will fail to start.

```json
"Kestrel": {
  "Endpoints": {
    "Http": { "Url": "http://*:8000" },
    "Https": {
      "Url": "https://*:8443",
      "Certificate": {
        "Path": "./certs/server.crt",
        "KeyPath": "./certs/server.key"
      }
    }
  }
}
```

**Checklist before enabling inbound HTTPS:**
- [ ] Server certificate file (`.crt`) is available at the configured path
- [ ] Private key file (`.key`) is available at the configured path
- [ ] The certificate covers the hostname clients will use (CN or SAN)
- [ ] Certificate is not self-signed, or clients are configured to trust it

---

#### 10.2 Outbound TLS — Forwarding to HTTPS Downstream Services

When a cluster `Destination.Address` uses `https://`, YARP validates the downstream server's certificate against the system trust store.

> ⚠️ **If the downstream service uses a self-signed or private CA certificate**, the gateway will reject the connection unless one of the following is done:

| Option | When to use | How to configure |
|--------|-------------|-----------------|
| **Trust via system store** | Certificate issued by a public/enterprise CA already trusted by the OS | No config change needed — works automatically |
| **`RootCertificatePath`** | Certificate issued by a private/internal CA | Set `RootCertificatePath` to the CA certificate file path |

```json
{
  "RootCertificatePath": "./certs/rootCA.crt",
  "ReverseProxy": {
    "Clusters": {
      "MyService": {
        "Destinations": {
          "Default": { "Address": "https://my-backend:8443" }
        }
      }
    }
  }
}
```

**Checklist before using HTTPS cluster destinations:**
- [ ] Confirm whether the downstream certificate is from a trusted public CA, an enterprise CA, or self-signed
- [ ] If private/self-signed: obtain the CA certificate (`.crt`) and set `RootCertificatePath`
- [ ] Verify the CA cert file is available at the configured path on the host/container

---

#### 10.3 SSL Decision Checklist (ask the user)

Before generating any configuration, confirm:

1. **Does the gateway itself need to serve HTTPS?**
   - Yes → configure `Kestrel.Endpoints.Https` with a certificate; make sure the cert files will be present
   - No → only configure `Kestrel.Endpoints.Http`

2. **Do any downstream (cluster) services use HTTPS?**
   - Yes → check whether their certificate is trusted by the system
     - Trusted public CA → no extra config needed
     - Private / self-signed CA → set `RootCertificatePath` to the CA cert

---

## Common Configuration Patterns

### Pattern 1: Simple Proxy (no auth, no swagger)

```json
{
  "Cors": {
    "default": ["https://app.example.com"]
  },
  "ReverseProxy": {
    "Routes": {
      "MyApi": {
        "ClusterId": "MyApi",
        "Match": { "Path": "api/{**catch-all}" }
      }
    },
    "Clusters": {
      "MyApi": {
        "Destinations": {
          "Default": { "Address": "http://my-api:8080" }
        }
      }
    }
  }
}
```

### Pattern 2: Proxy with Path Rewrite

When the backend uses a different base path than the gateway exposes:

```json
{
  "ReverseProxy": {
    "Routes": {
      "Barcode": {
        "ClusterId": "Barcode",
        "SwaggerKey": "Barcode",
        "Match": { "Path": "barcode/{**catch-all}" },
        "Transforms": [
          { "PathPattern": "api/barcode/{**catch-all}" }
        ]
      }
    },
    "Clusters": {
      "Barcode": {
        "Destinations": {
          "Default": { "Address": "http://barcode-svc:8080" }
        }
      }
    }
  },
  "SwaggerConfig": {
    "Services": {
      "Barcode": {
        "Swaggers": {
          "Barcode-v1": {
            "URL": "/swagger/v1/swagger.json",
            "PrefixPath": "/barcode",
            "RemovePrefix": "/api/barcode"
          }
        }
      }
    }
  }
}
```

### Pattern 3: Route with JWT Authentication

```json
{
  "Authentication": {
    "appAuth": {
      "Issuer": "https://cognito-idp.eu-central-1.amazonaws.com/eu-central-1_XXXXX",
      "Audience": "my-app-client-id",
      "ValidateAudience": true
    }
  },
  "ReverseProxy": {
    "Routes": {
      "ProtectedApi": {
        "ClusterId": "ProtectedApi",
        "AuthorizationPolicy": "appAuth",
        "Match": { "Path": "protected/{**catch-all}" }
      }
    },
    "Clusters": {
      "ProtectedApi": {
        "Destinations": {
          "Default": { "Address": "http://protected-api:8080" }
        }
      }
    }
  }
}
```

### Pattern 4: Rate Limiting

```json
{
  "RateLimits": {
    "publicApi": {
      "Type": "FixedWindow",
      "PermitLimit": 60,
      "WindowSeconds": 60
    }
  },
  "ReverseProxy": {
    "Routes": {
      "PublicApi": {
        "ClusterId": "PublicApi",
        "AuthorizationPolicy": "publicApi",
        "Match": { "Path": "public/{**catch-all}" }
      }
    },
    "Clusters": {
      "PublicApi": {
        "Destinations": {
          "Default": { "Address": "http://public-api:8080" }
        }
      }
    }
  }
}
```

### Pattern 5: Multiple Services with Swagger Aggregation

```json
{
  "Cors": {
    "default": ["https://app.example.com"]
  },
  "ReverseProxy": {
    "Routes": {
      "ServiceA": {
        "ClusterId": "ServiceA",
        "SwaggerKey": "ServiceA",
        "Match": { "Path": "service-a/{**catch-all}" }
      },
      "ServiceB": {
        "ClusterId": "ServiceB",
        "SwaggerKey": "ServiceB",
        "Match": { "Path": "service-b/{**catch-all}" },
        "Transforms": [
          { "PathPattern": "api/v2/{**catch-all}" }
        ]
      }
    },
    "Clusters": {
      "ServiceA": {
        "Destinations": {
          "Default": { "Address": "http://service-a:8080" }
        }
      },
      "ServiceB": {
        "Destinations": {
          "Default": { "Address": "http://service-b:8080" }
        }
      }
    }
  },
  "SwaggerConfig": {
    "BaseUrl": "api.example.com",
    "Services": {
      "ServiceA": {
        "Swaggers": {
          "ServiceA-v1": {
            "URL": "/swagger/v1/swagger.json",
            "PrefixPath": "/service-a"
          }
        }
      },
      "ServiceB": {
        "Swaggers": {
          "ServiceB-v1": {
            "URL": "/swagger/v1/swagger.json",
            "PrefixPath": "/service-b",
            "RemovePrefix": "/api/v2"
          }
        }
      }
    }
  }
}
```

### Pattern 6: Adding a Service — Swagger Decision Flow

When adding any new service, follow this decision flow:

```
Does the service expose a Swagger/OpenAPI definition?
│
├─ NO  → Route only (no SwaggerKey, no SwaggerConfig entry)
│         "ReverseProxy": {
│           "Routes": { "MyService": { "ClusterId": "MyService", "Match": { "Path": "myservice/{**catch-all}" } } },
│           "Clusters": { "MyService": { "Destinations": { "Default": { "Address": "http://my-svc:8080" } } } }
│         }
│
└─ YES → Does the downstream Swagger path match what the gateway exposes?
          │
          ├─ YES (no path difference) → Use PrefixPath only
          │    "SwaggerKey": "MyService"  (on route)
          │    "SwaggerConfig": { "Services": { "MyService": { "Swaggers": { "MyService-v1": {
          │      "URL": "/swagger/v1/swagger.json", "PrefixPath": "/myservice" } } } } }
          │
          └─ NO (gateway path ≠ backend path) → Use PrefixPath + RemovePrefix
               "SwaggerKey": "MyService"  (on route)
               "SwaggerConfig": { "Services": { "MyService": { "Swaggers": { "MyService-v1": {
                 "URL": "/swagger/v1/swagger.json",
                 "PrefixPath": "/myservice",
                 "RemovePrefix": "/api/myservice" } } } } }
```

**Questions to ask the user:**
1. Does the service expose a Swagger/OpenAPI JSON endpoint? *(If no, skip SwaggerConfig entirely)*
2. What is the URL of the Swagger JSON? *(Relative like `/swagger/v1/swagger.json` or absolute `http://...`)*
3. Under what path does the gateway expose this service? *(becomes `PrefixPath`)*
4. Does the backend's own Swagger use a different path prefix than what the gateway exposes? *(If yes, set `RemovePrefix` to the backend's prefix)*

---

### Pattern 7: Maintenance Mode

```json
{
  "EnableGlobalMaintenance": true,
  "Cors": { "default": ["https://app.example.com"] },
  "ReverseProxy": { "Routes": {}, "Clusters": {} }
}
```

---

## Validation Rules

The gateway validates configuration on startup using FluentValidation. Errors prevent startup.

| Section | Rule |
|---------|------|
| `Cors` | Must have at least a `"default"` policy |
| `Cors.{policy}` | Must have at least one allowed origin |
| `Authentication.{policy}.Issuer` | Must not be empty if section is present |
| `RateLimits.{policy}.Type` | Must be one of: `FixedWindow`, `SlidingWindow`, `TokenBucket`, `Concurrency`, `ConcurrencyLimiter` |
| `StaticContent.RequestPath` | Must start with `/` |
| `RootCertificatePath` | File must exist if specified |

---

## Quick Reference: "I need to..."

| Goal | What to configure |
|------|------------------|
| Add a new backend service route | Add entry to `ReverseProxy.Routes` + `ReverseProxy.Clusters` |
| Rewrite the URL path | Add `Transforms: [{ "PathPattern": "..." }]` to the route |
| Require JWT auth on a route | Add `Authentication.{policyName}` + set `AuthorizationPolicy` on route |
| Allow all routes to require auth by default | Name the auth policy `"default"` |
| Add CORS to a route | Add origins to `Cors.default`. Only set `CorsPolicy` on a route if it needs **different** origins than the default policy |
| Limit requests per time window | Add `RateLimits.{policyName}` + set `AuthorizationPolicy` on route |
| Aggregate Swagger docs for a service | **Ask the user first** if the service has a Swagger definition and whether it should appear in SwaggerUI. If yes: add `SwaggerKey` to route + add matching entry under `SwaggerConfig.Services` |
| Strip a path prefix for downstream Swagger | Use `RemovePrefix` in `SwaggerConfig.Services.{key}.Swaggers.{name}` |
| Enable maintenance mode | Set `EnableGlobalMaintenance: true` |
| Trust a custom CA certificate | Set `RootCertificatePath` to the CA cert file path |
| Expose the gateway over HTTPS | Configure `Kestrel.Endpoints.Https` with `Certificate.Path` + `Certificate.KeyPath`; ensure cert files exist before starting |
| Forward traffic to an HTTPS backend | Use `https://` in `Clusters.{name}.Destinations.{name}.Address`; set `RootCertificatePath` if the backend uses a private/self-signed CA |
| Route by hostname (virtual hosting) | Add `Match.Hosts: ["hostname"]` to the route |
| Set route priority | Set `Order` on the route (lower = higher priority) |
| Limit max upload size | Set `MaxRequestBodySize` on the route (in bytes) |
| Load balance across multiple backends | Add multiple entries under `Clusters.{name}.Destinations` |

---

## Full appsettings.json Template (All Features)

```json
{
  "EnableIngressController": false,
  "EnableGlobalMaintenance": false,
  "RootCertificatePath": null,
  "StaticContent": {
    "Path": "StaticContent",
    "RequestPath": "/static"
  },
  "Cors": {
    "default": [
      "https://app.example.com",
      "https://localhost:4200"
    ],
    "adminPolicy": [
      "https://admin.example.com"
    ]
  },
  "Authentication": {
    "appAuth": {
      "Type": "Bearer",
      "Issuer": "https://cognito-idp.eu-central-1.amazonaws.com/eu-central-1_XXXXX",
      "Audience": "your-app-client-id",
      "ValidateIssuer": true,
      "ValidateAudience": false
    }
  },
  "RateLimits": {
    "publicLimit": {
      "Type": "FixedWindow",
      "PermitLimit": 60,
      "WindowSeconds": 60,
      "QueueLimit": 10
    }
  },
  "ReverseProxy": {
    "Routes": {
      "ServiceA": {
        "ClusterId": "ServiceA",
        "SwaggerKey": "ServiceA",
        "AuthorizationPolicy": "appAuth",
        "Match": {
          "Path": "service-a/{**catch-all}"
        }
      },
      "ServiceB": {
        "ClusterId": "ServiceB",
        "SwaggerKey": "ServiceB",
        "Match": {
          "Path": "service-b/{**catch-all}"
        },
        "Transforms": [
          { "PathPattern": "api/service-b/{**catch-all}" }
        ]
      },
      "PublicApi": {
        "ClusterId": "PublicApi",
        "AuthorizationPolicy": "publicLimit",
        "Match": {
          "Path": "public/{**catch-all}"
        }
      }
    },
    "Clusters": {
      "ServiceA": {
        "Destinations": {
          "Default": { "Address": "http://service-a:8080" }
        }
      },
      "ServiceB": {
        "Destinations": {
          "Default": { "Address": "http://service-b:8080" }
        }
      },
      "PublicApi": {
        "Destinations": {
          "Primary": { "Address": "http://public-api-1:8080" },
          "Replica": { "Address": "http://public-api-2:8080" }
        }
      }
    }
  },
  "SwaggerConfig": {
    "BaseUrl": "api.example.com",
    "Services": {
      "ServiceA": {
        "Swaggers": {
          "ServiceA-v1": {
            "URL": "/swagger/v1/swagger.json",
            "PrefixPath": "/service-a"
          }
        }
      },
      "ServiceB": {
        "Swaggers": {
          "ServiceB-v1": {
            "URL": "/swagger/v1/swagger.json",
            "PrefixPath": "/service-b",
            "RemovePrefix": "/api/service-b"
          }
        }
      }
    }
  },
  "Kestrel": {
    "Endpoints": {
      "Http": { "Url": "http://*:8000" }
    }
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

---

## External References

- [YARP Documentation](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/yarp/yarp-overview?view=aspnetcore-10.0)
- [YARP Route Configuration](https://microsoft.github.io/reverse-proxy/articles/config-files.html)
- [YARP Transforms Reference](https://learn.microsoft.com/en-us/aspnet/core/fundamentals/servers/yarp/transforms?view=aspnetcore-10.0)
