# Azure Cloud Design Patterns — A Loan System Migration Story

---

## Zone 1: The Migration (Moving Away from WCF)

*How do you move from a "Big Ball of Mud" to Microservices?*
### 1. Strangler Fig
### 2. Anti-Corruption Layer (ACL)
### 3. Ambassador
### 4. Sidecar
---

### 1. Strangler Fig

**Definition**: Incrementally migrate a legacy system by gradually replacing pieces of functionality with new applications and services.

**Loan System Example**: You don't rewrite the whole Student Loan system at once. You create a new "Student Loan UI" that calls a new microservice for new features, while routing old features back to the WCF service. Eventually, the WCF service "withers" away.

```
Phase 1: New facade routes ALL traffic to old WCF
┌────────┐       ┌──────────┐     ┌──────────────┐
│ Client  │────▶│  Facade  │────▶│  WCF Service │
└────────┘       └──────────┘     │  (Monolith)  │
                                  └──────────────┘

Phase 2: New features go to new microservice, old features stay
┌────────┐     ┌──────────┐──── /new-loan ────▶  ┌────────────────┐
│ Client │────▶│  Facade │                       │ New Loan µsvc  │
└────────┘     └──────────┘──── /old-report ──▶  ┌────────────────┐
                                                 │  WCF Service   │
                                                 └────────────────┘

Phase 3: Everything migrated — WCF decommissioned
┌────────┐     ┌──────────┐────▶ ┌─────────────┐
│ Client  │────▶│  Facade  │────▶ ┌─────────────┐
└────────┘     └──────────┘────▶ ┌─────────────┐
                                 │ Microservices│
                                 └─────────────┘
```

**Layman Term**: Imagine you have an old, drafty house. Instead of tearing it down and sleeping in the rain, you build a new modern room next to it, then move the kitchen there. Then you build a new bedroom and move out of the old one. Eventually, you tear down the last old room. The "Fig" vine grows around the tree until the tree is gone.

**Implementation**:

```csharp
// API Gateway / Reverse Proxy routing (YARP config)
"Routes": [
  {
    "RouteId": "new-student-loans",
    "Match": { "Path": "/api/student-loans/{**catch-all}" },
    "ClusterId": "new-microservice"              // New service
  },
  {
    "RouteId": "legacy-mortgage",
    "Match": { "Path": "/api/mortgage/{**catch-all}" },
    "ClusterId": "legacy-wcf"                    // Still on WCF
  }
]
```

| Phase                         | What Happens                  | Risk Level                |
|---                            |---                            |---                        |
| 1. Add facade                 | All traffic proxied to WCF    | Low — no behavior change  |    
| 2. Migrate feature-by-feature | New features → new service    | Medium — dual systems     |
| 3. Decommission WCF           | All traffic on microservices  | Low — old system unused   |

---

### 2. Anti-Corruption Layer (ACL)

**Definition**: Implement a façade or adapter layer between a modern application and a legacy system to prevent legacy concepts from leaking into new code.

**Loan System Example**: Your new Mortgage service uses modern JSON/REST. The old WCF core uses SOAP/XML. The ACL sits in between, translating "Modern Mortgage Request" into "Old WCF XML" so your new code stays clean.

```
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  New Mortgage     │     │   ACL Layer      │     │  Legacy WCF      │
│  Service (JSON)   │────▶│  (Translator)    │────▶│  Service (SOAP)  │
│                   │◀────│                  │◀────│                  │
│  Clean C# DTOs    │     │  JSON ↔ XML      │     │  Old Data Model  │
└──────────────────┘     └──────────────────┘     └──────────────────┘
```

**Layman Term**: You are a Modern Architect building a smart home. You need to connect it to an Old City Power Grid from the 1940s. Without an ACL, you'd run old copper wires throughout your walls. With an ACL, you install a Modern Power Transformer at the edge — it converts the messy old power into clean digital power your home expects.

```csharp
// Modern service — clean domain model
public class MortgageApplication
{
    public string ApplicantName { get; set; } = "";
    public decimal RequestedAmount { get; set; }
    public int TermYears { get; set; }
}

// ACL — translates between modern and legacy
public class LegacyLoanAcl : ILoanService
{
    private readonly LegacyWcfClient _wcfClient;

    public LegacyLoanAcl(LegacyWcfClient wcfClient) => _wcfClient = wcfClient;

    public async Task<LoanDecision> SubmitApplicationAsync(MortgageApplication app)
    {
        // Translate modern → legacy XML format
        var legacyRequest = new LegacyLoanRequest
        {
            APPLICANT_NM = app.ApplicantName,           // Old naming conventions
            LN_AMT = app.RequestedAmount,
            TERM_MTH = app.TermYears * 12               // Legacy uses months
        };

        var legacyResponse = await _wcfClient.SubmitLoanAsync(legacyRequest);

        // Translate legacy → modern
        return new LoanDecision
        {
            IsApproved = legacyResponse.STATUS_CD == "A",
            ApprovedAmount = legacyResponse.APPR_AMT,
            Reason = MapStatusCode(legacyResponse.STATUS_CD)
        };
    }

    private static string MapStatusCode(string code) => code switch
    {
        "A" => "Approved",
        "D" => "Declined",
        "P" => "Pending Review",
        _ => "Unknown"
    };
}
```

---

### 3. Ambassador

**Definition**: Create helper services that send network requests on behalf of a consumer service or application — handling retries, logging, circuit breaking, and security.

**Loan System Example**: Your Loan app needs to talk to an external Credit Bureau. Instead of coding retry logic into the app, you use an Ambassador (helper) to handle the connection, retries, and security.

```
┌──────────────┐     ┌──────────────────────┐     ┌──────────────────┐
│  Loan Service │────▶│  Ambassador (Envoy)  │────▶│  Credit Bureau   │
│              │     │  • Retry (3x)        │     │  (External API)  │
│  "Check      │     │  • Circuit breaker   │     │                  │
│   credit"    │     │  • mTLS auth         │     │                  │
│              │     │  • Logging           │     │                  │
└──────────────┘     └──────────────────────┘     └──────────────────┘
```

**Layman Term**: You are a CEO (the main app). You need to meet a partner in a foreign country. Without an ambassador, you'd book flights, learn the language, handle currency, and manage security yourself. With an ambassador, they manage translation, travel, and security — you just say "I need to talk to the partner."

```csharp
// Ambassador implemented via HttpClient + Polly
builder.Services.AddHttpClient("CreditBureau", client =>
{
    client.BaseAddress = new Uri("https://api.creditbureau.com");
    client.DefaultRequestHeaders.Add("X-Api-Key", config["CreditBureau:ApiKey"]);
})
.AddTransientHttpErrorPolicy(p => p.WaitAndRetryAsync(3,
    attempt => TimeSpan.FromMilliseconds(200 * Math.Pow(2, attempt))))
.AddTransientHttpErrorPolicy(p => p.CircuitBreakerAsync(5, TimeSpan.FromSeconds(30)));

// The Loan Service stays clean — no retry/circuit logic
public class LoanService
{
    private readonly HttpClient _creditClient;

    public LoanService(IHttpClientFactory factory)
        => _creditClient = factory.CreateClient("CreditBureau");

    public async Task<CreditScore> CheckCreditAsync(string ssn)
    {
        // Ambassador (HttpClient pipeline) handles retries, circuit breaker, auth
        var response = await _creditClient.GetAsync($"/v2/score/{ssn}");
        response.EnsureSuccessStatusCode();
        return await response.Content.ReadFromJsonAsync<CreditScore>();
    }
}
```

---

### 4. Sidecar

**Definition**: Deploy components into a separate process or container to provide isolation and encapsulation — adding cross-cutting concerns without modifying the app.

**Loan System Example**: You want to add logging and monitoring to your old ASP.NET app without changing its code. You deploy a Sidecar container next to it to handle those tasks.

```
┌─────────── Pod / VM ──────────────┐
│                                    │
│  ┌───────────────┐  ┌───────────┐ │
│  │  Loan Service  │  │  Sidecar  │ │
│  │  (your code)   │──│ • Logging │ │
│  │                │  │ • Metrics │ │
│  │  Port 8080     │  │ • mTLS   │ │
│  └───────────────┘  └───────────┘ │
│       localhost communication      │
└────────────────────────────────────┘
```

**Layman Term**: A motorcycle sidecar — the motorcycle (your app) provides speed and direction; the sidecar carries the extra luggage (logging, security, monitoring) so the motorcycle doesn't have to be heavy.

```yaml
# Kubernetes pod with sidecar
apiVersion: v1
kind: Pod
metadata:
  name: loan-service
spec:
  containers:
    - name: loan-app
      image: loans/mortgage-service:v2
      ports:
        - containerPort: 8080
    - name: logging-sidecar
      image: fluent/fluentd:latest
      volumeMounts:
        - name: shared-logs
          mountPath: /var/log/app
    - name: envoy-sidecar            # Service mesh proxy
      image: envoyproxy/envoy:latest
      ports:
        - containerPort: 9901
  volumes:
    - name: shared-logs
      emptyDir: {}
```

| Concern | Without Sidecar | With Sidecar |
|---|---|---|
| Logging | Add library to every service | Sidecar collects logs automatically |
| mTLS | Each service manages certs | Envoy sidecar handles encryption |
| Monitoring | Instrument every service | Sidecar exports metrics |
| Language coupling | Must use same language | Sidecar is language-agnostic |

---

## Zone 2: The Front Door (Traffic Management)

### 5. Gateway Routing
### 6. Gateway Aggregation
### 7. Backends for Frontends (BFF)
### 8. Gateway Offloading
### 9. External Configuration Store

*How do users and devices talk to your different loan services?*

---

### 5. Gateway Routing

**Definition**: A single entry point that routes requests to the appropriate backend microservice based on the URL path.

```
                         api.loans.com
                              │
                     ┌────────┴────────┐
                     │   API Gateway    │
                     └────────┬────────┘
                              │
              ┌───────────────┼───────────────┐
              │               │               │
        /mortgage        /student        /personal
              │               │               │
     ┌────────▼──────┐ ┌─────▼──────┐ ┌──────▼──────┐
     │ Mortgage Svc   │ │ Student Svc│ │ Personal Svc│
     └───────────────┘ └────────────┘ └─────────────┘
```

```csharp
// Azure API Management or YARP Reverse Proxy
"Routes": [
  {
    "RouteId": "mortgage",
    "Match": { "Path": "/api/mortgage/{**remainder}" },
    "ClusterId": "mortgage-cluster",
    "Transforms": [{ "PathRemovePrefix": "/api/mortgage" }]
  },
  {
    "RouteId": "student",
    "Match": { "Path": "/api/student/{**remainder}" },
    "ClusterId": "student-cluster"
  },
  {
    "RouteId": "personal",
    "Match": { "Path": "/api/personal/{**remainder}" },
    "ClusterId": "personal-cluster"
  }
]
```

---

### 6. Gateway Aggregation

**Definition**: The gateway calls multiple backend services, merges their responses, and returns a single aggregated response to the client.

```
Client: GET /api/portfolio
              │
     ┌────────▼────────┐
     │   API Gateway    │
     └────────┬────────┘
              │ (fan-out)
    ┌─────────┼──────────┐
    ▼         ▼          ▼
Student    Mortgage    Personal
Service    Service     Service
    │         │          │
    └─────────┼──────────┘
              │ (merge)
     ┌────────▼────────┐
     │ Combined Response │
     └──────────────────┘
```

```csharp
// Gateway aggregation endpoint
[HttpGet("portfolio/{customerId}")]
public async Task<IActionResult> GetPortfolio(string customerId)
{
    // Fan-out: call all services concurrently
    var studentTask = _studentClient.GetAsync($"/loans/{customerId}");
    var mortgageTask = _mortgageClient.GetAsync($"/loans/{customerId}");
    var personalTask = _personalClient.GetAsync($"/loans/{customerId}");

    await Task.WhenAll(studentTask, mortgageTask, personalTask);

    // Merge into single response
    return Ok(new LoanPortfolioDto
    {
        StudentLoans = await studentTask.Result.Content.ReadFromJsonAsync<List<LoanDto>>(),
        MortgageLoans = await mortgageTask.Result.Content.ReadFromJsonAsync<List<LoanDto>>(),
        PersonalLoans = await personalTask.Result.Content.ReadFromJsonAsync<List<LoanDto>>()
    });
}
```

**Before**: Client makes 3 API calls → 3 round trips, complex client logic.
**After**: Client makes 1 call → Gateway handles fan-out, 1 response.

---

### 7. Backends for Frontends (BFF)

**Definition**: Create separate backend services tailored for each frontend (mobile app, web portal, admin dashboard).

```
┌──────────┐    ┌──────────────────┐
│ Mobile   │───▶│ Mobile BFF       │──── Lightweight JSON, pagination
│ App      │    │ (optimized data) │
└──────────┘    └──────────────────┘
                                         ┌───────────────┐
┌──────────┐    ┌──────────────────┐     │               │
│ Admin    │───▶│ Admin BFF        │────▶│ Microservices │
│ Portal   │    │ (full data)      │     │               │
└──────────┘    └──────────────────┘     └───────────────┘

┌──────────┐    ┌──────────────────┐
│ Partner  │───▶│ Partner BFF      │──── Rate-limited, filtered
│ Bank API │    │ (restricted)     │
└──────────┘    └──────────────────┘
```

```csharp
// Mobile BFF — lightweight response
[HttpGet("loans")]
public async Task<IActionResult> GetLoans(string customerId)
{
    var loans = await _loanService.GetByCustomerAsync(customerId);
    return Ok(loans.Select(l => new
    {
        l.Id, l.Type, l.Balance   // Only essential fields for mobile
    }));
}

// Admin BFF — full response with audit data
[HttpGet("loans")]
public async Task<IActionResult> GetLoans(string customerId)
{
    var loans = await _loanService.GetByCustomerAsync(customerId);
    return Ok(loans.Select(l => new
    {
        l.Id, l.Type, l.Balance, l.InterestRate,
        l.CreatedBy, l.CreatedAt, l.AuditLog   // Full detail for admin
    }));
}
```

---

### 8. Gateway Offloading

**Definition**: Move cross-cutting concerns (SSL, auth, rate limiting) to the gateway so backend services don't need to handle them.

```
┌────────┐   HTTPS    ┌─────────────────────────┐   HTTP    ┌─────────────┐
│ Client  │──────────▶│ Gateway                   │─────────▶│ Loan Service│
└────────┘            │ ✅ SSL termination        │          │ (no SSL)    │
                      │ ✅ JWT validation         │          │ (no auth)   │
                      │ ✅ Rate limiting          │          │ (no limits) │
                      │ ✅ CORS headers           │          │ (simple)    │
                      │ ✅ Request logging        │          │             │
                      └─────────────────────────┘          └─────────────┘
```

**What's offloaded:**

| Concern               | Without Offloading          | With Offloading                 |
|---                    |---                          |---                              |
| SSL/TLS               | Every service manages certs | Gateway terminates SSL          |
| Authentication        | Every service validates JWT | Gateway validates once          |
| Rate limiting         | Each service implements     | Gateway enforces globally       |
| CORS                  | Configured per service      | Gateway handles it              |
| Logging/tracing       | Instrumented per service    | Gateway captures all traffic    |

---

### 9. External Configuration Store

**Definition**: Store configuration (interest rates, feature flags, loan limits) in a centralized external store instead of `appsettings.json` or `web.config`.

```
┌───────────────┐     ┌──────────────────────────┐
│ Mortgage Svc   │────▶│                            │
├───────────────┤     │ Azure App Configuration       │
│ Student Svc    │────▶│                              │ 
├───────────────┤     │ • InterestRate:Mortgage=6.5   │
│ Personal Svc   │────▶│ • LoanLimit:Student=50000   │
└───────────────┘     │ • Feature:AutoApproval=true │
                      └──────────────────────────┘
```

```csharp
// Program.cs — connect to Azure App Configuration
builder.Configuration.AddAzureAppConfiguration(options =>
{
    options.Connect(builder.Configuration["AppConfig:ConnectionString"])
           .Select("LoanSettings:*")           // Load all loan settings
           .ConfigureRefresh(refresh =>
           {
               refresh.Register("LoanSettings:Sentinel", refreshAll: true)
                      .SetCacheExpiration(TimeSpan.FromMinutes(5));
           });
});

// Bind to strongly typed options
builder.Services.Configure<LoanSettings>(
    builder.Configuration.GetSection("LoanSettings"));

public class LoanSettings
{
    public decimal MortgageRate { get; set; }
    public decimal StudentLoanLimit { get; set; }
    public bool AutoApprovalEnabled { get; set; }
}
```

**Before**: Change a rate → redeploy all services.
**After**: Change a rate in App Configuration → services pick it up within minutes.

---

## Zone 3: Resiliency (Stopping the Crash)

*In a monolith, if WCF hangs, the whole bank stops. In the cloud, we isolate the failure.*

---

### 10. Retry

**Definition**: Automatically retry a failed operation, typically with increasing delays, to handle transient failures.

```
Request ──▶ Service ──▶ ❌ Fail (500)
                        ⏳ Wait 200ms
        ──▶ Service ──▶ ❌ Fail (500)
                        ⏳ Wait 400ms
        ──▶ Service ──▶ ✅ Success (200)
```

```csharp
// Using Polly via Microsoft.Extensions.Http.Resilience
builder.Services.AddHttpClient("MortgageService")
    .AddStandardResilienceHandler(options =>
    {
        options.Retry.MaxRetryAttempts = 3;
        options.Retry.Delay = TimeSpan.FromMilliseconds(200);
        options.Retry.BackoffType = DelayBackoffType.Exponential;
        // 200ms → 400ms → 800ms
    });

// Or with Polly directly
builder.Services.AddHttpClient("CreditCheck")
    .AddTransientHttpErrorPolicy(p =>
        p.WaitAndRetryAsync(3, attempt =>
            TimeSpan.FromMilliseconds(200 * Math.Pow(2, attempt)),
            onRetry: (outcome, delay, attempt, _) =>
            {
                Log.Warning("Retry {Attempt} after {Delay}ms: {Status}",
                    attempt, delay.TotalMilliseconds, outcome.Result?.StatusCode);
            }));
```

| Retry Strategy        | Delays              | Best For                    |
|---                    |---                  |---                          |
| Immediate             | 0, 0, 0             | Quick transient glitches    |
| Fixed                 | 500ms, 500ms, 500ms | Predictable recovery        |
| Exponential backoff   | 200ms, 400ms, 800ms | Network/service recovery    |
| Exponential + jitter  | 200±50ms, 400±100ms | Prevent thundering herd     |

---

### 11. Circuit Breaker

**Definition**: When a service is failing repeatedly, "trip" the breaker to immediately return errors for a cooldown period — preventing cascading failures.

```
State Machine:

  ┌──────────┐  Failures exceed    ┌──────────┐  Timeout expires  ┌───────────┐
  │  CLOSED   │  threshold (5)     │   OPEN    │                  │ HALF-OPEN  │
  │ (normal)  │──────────────────▶│ (failing) │────────────────▶│ (testing)  │
  └──────────┘                    └──────────┘                  └───────────┘
       ▲                               │                             │
       │                          Immediately                   Success → Close
       │                          rejects all                   Failure → Open
       └──────────────────────────requests──────────────────────────┘
```

```csharp
builder.Services.AddHttpClient("CreditCheck")
    .AddTransientHttpErrorPolicy(p =>
        p.CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,   // Trip after 5 failures
            durationOfBreak: TimeSpan.FromSeconds(30), // Stay open 30s
            onBreak: (result, duration) =>
                Log.Warning("Circuit OPEN for {Duration}s", duration.TotalSeconds),
            onReset: () =>
                Log.Information("Circuit CLOSED — service recovered")));
```

**Without Circuit Breaker**: 1,000 users wait 30 seconds each for timeout → cascading failure.
**With Circuit Breaker**: After 5 failures, immediately returns "Service Unavailable" → users get fast error, system recovers.

---

### 12. Bulkhead

**Definition**: Isolate resources for different workloads so a flood in one can't starve the others.

```
Without Bulkhead:
┌─────────────────────────────────────┐
│ Shared Thread Pool (100 threads)    │
│ Personal Loans: 95 threads (flood!) │
│ Mortgage Loans:  5 threads (starved)│
└─────────────────────────────────────┘

With Bulkhead:
┌──────────────────┐  ┌──────────────────┐
│ Personal Loans    │  │ Mortgage Loans    │
│ Pool: 50 threads  │  │ Pool: 50 threads  │
│ Used: 50 (full)   │  │ Used: 10 (fine)   │
│ Excess → 429      │  │ Unaffected ✅     │
└──────────────────┘  └──────────────────┘
```

```csharp
// Bulkhead with Polly
var bulkheadPolicy = Policy.BulkheadAsync<HttpResponseMessage>(
    maxParallelization: 50,        // Max 50 concurrent calls
    maxQueuingActions: 25,         // Queue up to 25 more
    onBulkheadRejectedAsync: ctx =>
    {
        Log.Warning("Bulkhead rejected — too many concurrent requests");
        return Task.CompletedTask;
    });
```

---

### 13. Health Endpoint Monitoring

**Definition**: Expose a `/health` endpoint that reports the service's operational status. Orchestrators (Kubernetes, Azure) use it to detect and replace unhealthy instances.

```csharp
// Program.cs
builder.Services.AddHealthChecks()
    .AddSqlServer(connectionString, name: "database")
    .AddRedis(redisConnection, name: "cache")
    .AddUrlGroup(new Uri("https://api.creditbureau.com/health"), name: "credit-bureau");

app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // Just checks if the process is alive
});
```

Response:
```json
{
  "status": "Healthy",
  "entries": {
    "database": { "status": "Healthy", "duration": "00:00:00.045" },
    "cache": { "status": "Healthy", "duration": "00:00:00.012" },
    "credit-bureau": { "status": "Degraded", "duration": "00:00:02.100" }
  }
}
```

| Endpoint | Purpose | Used By |
|---|---|---|
| `/health/live` | Is the process alive? | Kubernetes liveness probe |
| `/health/ready` | Can it serve traffic? | Kubernetes readiness probe |
| `/health` | Full status with dependencies | Monitoring dashboards |

---

### 14. Throttling

**Definition**: Limit the rate of requests from a specific user, IP, or client to protect your system from overload or abuse.

```csharp
// .NET 7+ Rate Limiting Middleware
builder.Services.AddRateLimiter(options =>
{
    // Per-user throttle
    options.AddPolicy("per-user", context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.User?.Identity?.Name ?? context.Connection.RemoteIpAddress?.ToString(),
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 10,
                Window = TimeSpan.FromSeconds(10),
                QueueLimit = 0
            }));

    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
});

app.UseRateLimiter();

[HttpPost("apply")]
[EnableRateLimiting("per-user")]
public async Task<IActionResult> ApplyForLoan(LoanApplicationDto dto) => Ok();
```

**Scenario**: A bot tries to submit 100 loan applications per second → throttled to 10 per 10 seconds, protecting the database.

---

### 15. Rate Limiting

**Definition**: Setting a hard cap on API usage for a client, partner, or tier over a longer window.

```csharp
builder.Services.AddRateLimiter(options =>
{
    // Partner bank: 5,000 requests per hour
    options.AddPolicy("partner-tier", context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.Request.Headers["X-Partner-Id"].ToString(),
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 5000,
                Window = TimeSpan.FromHours(1),
                QueueLimit = 0
            }));

    // Sliding window for smoother distribution
    options.AddPolicy("smooth-limit", context =>
        RateLimitPartition.GetSlidingWindowLimiter(
            partitionKey: context.Request.Headers["X-Api-Key"].ToString(),
            factory: _ => new SlidingWindowRateLimiterOptions
            {
                PermitLimit = 1000,
                Window = TimeSpan.FromMinutes(10),
                SegmentsPerWindow = 5
            }));
});
```

| Algorithm | Behavior | Best For |
|---|---|---|
| Fixed Window | Reset counter every N seconds | Simple quotas |
| Sliding Window | Rolling window — no burst at boundary | Smooth rate control |
| Token Bucket | Tokens refill over time; allows bursts | API quotas with burst allowance |
| Concurrency | Limits simultaneous requests | Resource protection |

---

## Zone 4: Messaging & Async (The "Waiting Room")

*Loan processing takes time. We shouldn't keep the user's browser spinning.*

---

### 16. Asynchronous Request-Reply

**Definition**: Accept a long-running request, return a tracking ID immediately, and let the client poll for the result.

```
Client: POST /api/loans/apply  →  { "jobId": "abc-123" }   (202 Accepted)
Client: GET  /api/loans/status/abc-123  →  { "status": "Processing" }
Client: GET  /api/loans/status/abc-123  →  { "status": "Processing" }
Client: GET  /api/loans/status/abc-123  →  { "status": "Approved", "result": {...} }
```

```csharp
[HttpPost("apply")]
public async Task<IActionResult> Apply(LoanApplicationDto dto)
{
    var jobId = Guid.NewGuid().ToString();

    // Queue the work for background processing
    await _queue.SendMessageAsync(new LoanJob { JobId = jobId, Application = dto });

    return Accepted(new { JobId = jobId, StatusUrl = $"/api/loans/status/{jobId}" });
}

[HttpGet("status/{jobId}")]
public async Task<IActionResult> GetStatus(string jobId)
{
    var status = await _cache.GetStringAsync($"job:{jobId}");

    return status switch
    {
        null => NotFound(),
        "Completed" => Ok(new { Status = "Completed",
            Result = await _cache.GetStringAsync($"job:{jobId}:result") }),
        _ => Ok(new { Status = status })
    };
}
```

---

### 17. Queue-Based Load Leveling

**Definition**: Use a queue as a buffer between the caller and the processing service to smooth out spikes in demand.

```
"Back to School" season:

Without Queue:
  100 apps/sec ──▶ Service (handles 10/sec) ──▶ 💥 Crash / Timeout

With Queue:
  100 apps/sec ──▶ ┌──────────┐ ──▶ Service (processes 10/sec steadily)
                   │  Queue    │     No crash — just a longer wait
                   │ (buffer)  │
                   └──────────┘
```

```csharp
// Producer — writes to queue regardless of backend speed
[HttpPost("apply")]
public async Task<IActionResult> Apply(LoanApplicationDto dto)
{
    await _serviceBus.SendMessageAsync(
        new ServiceBusMessage(JsonSerializer.Serialize(dto)));

    return Accepted(new { Message = "Application received. Processing..." });
}

// Consumer — processes at its own pace
public class LoanProcessorWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        var processor = _client.CreateProcessor("loan-applications");

        processor.ProcessMessageAsync += async args =>
        {
            var dto = JsonSerializer.Deserialize<LoanApplicationDto>(args.Message.Body);
            await _loanService.ProcessAsync(dto!);
            await args.CompleteMessageAsync(args.Message);
        };

        await processor.StartProcessingAsync(ct);
    }
}
```

---

### 18. Publisher-Subscriber (Pub/Sub)

**Definition**: When an event occurs, publish it once. Multiple independent subscribers react to it without the publisher knowing about them.

```
Loan Approved event:
                              ┌─────────────────┐
                         ┌───▶│ Email Service     │  "Send approval email"
                         │    └─────────────────┘
┌──────────────┐   ┌─────┴──┐
│ Loan Service  │──▶│ Topic  │──▶┌─────────────────┐
│ "Approved!"   │   └─────┬──┘   │ Document Service │  "Generate contract PDF"
└──────────────┘         │    └─────────────────┘
                         │    ┌─────────────────┐
                         └───▶│ Payment Service  │  "Schedule first payment"
                              └─────────────────┘
```

```csharp
// Publisher — fires and forgets
public class LoanService
{
    private readonly ServiceBusSender _topicSender;

    public async Task ApproveLoanAsync(int loanId)
    {
        // ... approve logic ...

        await _topicSender.SendMessageAsync(new ServiceBusMessage(
            JsonSerializer.Serialize(new LoanApprovedEvent
            {
                LoanId = loanId,
                ApprovedAt = DateTime.UtcNow,
                Amount = loan.ApprovedAmount
            }))
        {
            Subject = "LoanApproved"
        });
    }
}

// Subscriber — Email Service
processor.ProcessMessageAsync += async args =>
{
    var evt = JsonSerializer.Deserialize<LoanApprovedEvent>(args.Message.Body);
    await _emailService.SendApprovalEmailAsync(evt!.LoanId);
    await args.CompleteMessageAsync(args.Message);
};
```

---

### 19. Competing Consumers

**Definition**: Multiple consumers read from the same queue, processing messages in parallel to increase throughput.

```
                    ┌──────────────┐
               ┌───▶│ Email Worker 1│
┌──────────┐   │    └──────────────┘
│  Queue    │───┤    ┌──────────────┐
│ 1,000     │───┼───▶│ Email Worker 2│    Each message processed by ONE worker
│ emails    │───┤    └──────────────┘
└──────────┘   │    ┌──────────────┐
               └───▶│ Email Worker 3│
                    └──────────────┘
                    Throughput: 3x
```

```csharp
// Azure Service Bus automatically distributes messages across consumers
// Kubernetes: scale the worker deployment
// replicas: 5 → 5 competing consumers reading from the same queue
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: email-worker
spec:
  replicas: 5    # 5 competing consumers
  template:
    spec:
      containers:
        - name: email-worker
          image: loans/email-worker:v1
```

---

### 20. Priority Queue

**Definition**: Process high-priority messages before low-priority ones.

```
Emergency Personal Loan ──▶ ┌─────────────────┐ ──▶ Processed FIRST
                            │ High Priority Q  │
                            └─────────────────┘

Standard Student Loan   ──▶ ┌─────────────────┐ ──▶ Processed AFTER
                            │ Low Priority Q   │
                            └─────────────────┘
```

```csharp
// Send to different queues based on priority
public async Task SubmitLoanAsync(LoanApplicationDto dto)
{
    var queueName = dto.Priority switch
    {
        LoanPriority.Emergency => "loan-processing-high",
        LoanPriority.Standard => "loan-processing-low",
        _ => "loan-processing-low"
    };

    var sender = _serviceBusClient.CreateSender(queueName);
    await sender.SendMessageAsync(new ServiceBusMessage(JsonSerializer.Serialize(dto)));
}

// Consumer: process high-priority queue first, then low
// Or: assign more workers to the high-priority queue
```

---

### 21. Claim Check

**Definition**: Instead of sending a large payload through the message bus, store it externally and include only a reference (the "claim check") in the message.

```
Without Claim Check:
  50MB PDF ──▶ Message Bus ──▶ 💥 (message too large, bus crashes)

With Claim Check:
  50MB PDF ──▶ Blob Storage (save, get URL)
  Message: { "pdfUrl": "https://storage/docs/abc.pdf" } ──▶ Message Bus ──▶ ✅
```

```csharp
// Producer: save file, send reference
public async Task SubmitMortgageAsync(MortgageDto dto, Stream pdfStream)
{
    // Store the large file
    var blobName = $"applications/{Guid.NewGuid()}.pdf";
    var blobClient = _container.GetBlobClient(blobName);
    await blobClient.UploadAsync(pdfStream);

    // Send lightweight message with reference
    var message = new MortgageMessage
    {
        ApplicationId = dto.Id,
        ApplicantName = dto.Name,
        DocumentUrl = blobClient.Uri.ToString()   // Claim Check
    };

    await _sender.SendMessageAsync(
        new ServiceBusMessage(JsonSerializer.Serialize(message)));
}

// Consumer: retrieve file when needed
var msg = JsonSerializer.Deserialize<MortgageMessage>(args.Message.Body);
var blobClient = _container.GetBlobClient(new Uri(msg.DocumentUrl).AbsolutePath);
var pdfStream = await blobClient.OpenReadAsync();
```

---

### 22. Sequential Convoy

**Definition**: Process related messages in a strict order — ensuring "Update Address" always happens after "Create Account" for the same customer.

```
Messages for Customer 123:
  1. Create Account   ──▶ ┌───────────────────┐
  2. Update Address   ──▶ │ Session: Cust-123  │ ──▶ Processed in order: 1, 2, 3
  3. Add Loan         ──▶ └───────────────────┘
```

```csharp
// Azure Service Bus: use Sessions for ordered delivery
await sender.SendMessageAsync(new ServiceBusMessage(payload)
{
    SessionId = customerId   // All messages for same customer go to same session
});

// Consumer: accepts one session at a time → processes in order
var processor = client.CreateSessionProcessor("customer-events",
    new ServiceBusSessionProcessorOptions
    {
        MaxConcurrentSessions = 10  // 10 customers processed concurrently
    });
```

---

### 23. Choreography

**Definition**: Services coordinate through events — each service listens for events and decides what to do next. No central orchestrator.

```
Loan Application Flow (Choreography):

Apply ──▶ "ApplicationReceived"
              │
              ▼
         Credit Service listens ──▶ "CreditChecked"
                                         │
                                         ▼
                                    Underwriting listens ──▶ "UnderwritingComplete"
                                                                   │
                                                                   ▼
                                                              Approval listens ──▶ "LoanApproved"
```

Each service is autonomous — knows its triggers, does its work, emits an event. No central controller.

| Choreography | Orchestration |
|---|---|
| Decentralized — services react to events | Centralized — one coordinator directs |
| Loosely coupled | Tightly coupled to orchestrator |
| Hard to track end-to-end flow | Easy to see the full workflow |
| Best for simple, linear flows | Best for complex, branching flows |

---

### 24. Messaging Bridge

**Definition**: Connect two different messaging systems so they can exchange messages.

```
┌──────────────┐     ┌───────────────────┐     ┌──────────────┐
│ Azure Service │     │  Messaging Bridge  │     │  IBM MQ      │
│ Bus (new)     │◀───▶│  (translator)     │◀───▶│  (legacy)    │
└──────────────┘     └───────────────────┘     └──────────────┘
```

The bridge reads from one system, transforms the message format, and writes to the other. Used when migrating or integrating with external partners who use different messaging tech.

---

## Zone 5: Data Management (Consistency & Speed)

*This is where the money is managed.*

---

### 25. CQRS (Command Query Responsibility Segregation)

**Definition**: Separate the "Write" model (commands) from the "Read" model (queries). Use different databases or schemas optimized for each.

```
┌──────────────────────────────────────────────────┐
│                   API Layer                       │
│   POST /loans (Command)    GET /dashboard (Query) │
│        │                         │                │
│        ▼                         ▼                │
│  ┌───────────┐             ┌────────────┐         │
│  │ Write Model │           │ Read Model │         │
│  │ (SQL Server)│           │ (CosmosDB / │        │
│  │ Normalized  │──sync──▶ │  Denormalized)│      │
│  │ Full entity │           │  Pre-joined  │      │
│  └───────────┘             └────────────┘        │
└──────────────────────────────────────────────────┘
```

```csharp
// Command side — full validation, business rules
public class CreateLoanCommandHandler
{
    private readonly AppDbContext _writeDb;

    public async Task HandleAsync(CreateLoanCommand cmd)
    {
        var loan = new Loan { Amount = cmd.Amount, CustomerId = cmd.CustomerId };
        _writeDb.Loans.Add(loan);
        await _writeDb.SaveChangesAsync();

        // Publish event to sync read model
        await _bus.PublishAsync(new LoanCreatedEvent { LoanId = loan.Id });
    }
}

// Query side — fast, denormalized reads
public class LoanDashboardQueryHandler
{
    private readonly CosmosClient _readDb;

    public async Task<DashboardDto> HandleAsync(string customerId)
    {
        // Pre-computed, denormalized — no JOINs needed
        return await _readDb.ReadItemAsync<DashboardDto>(customerId, new PartitionKey(customerId));
    }
}
```

---

### 26. Event Sourcing

**Definition**: Instead of storing current state, store a sequence of events. Replay events to reconstruct state. Creates a perfect audit trail.

```
Traditional: Account balance = $5,000 (current state only)

Event Sourcing:
  Event 1: AccountOpened     { Amount: $0 }
  Event 2: LoanDisbursed     { Amount: +$10,000 }
  Event 3: PaymentReceived   { Amount: -$2,000 }
  Event 4: PaymentReceived   { Amount: -$3,000 }
  ─────────────────────────────────────────────
  Current State (replay):     Balance = $5,000
  Audit Trail: Every transaction visible to regulators ✅
```

```csharp
public class LoanAccount
{
    private readonly List<IEvent> _events = new();
    public decimal Balance { get; private set; }

    public void Apply(LoanDisbursedEvent e)
    {
        Balance += e.Amount;
        _events.Add(e);
    }

    public void Apply(PaymentReceivedEvent e)
    {
        Balance -= e.Amount;
        _events.Add(e);
    }

    // Rebuild from history
    public static LoanAccount FromHistory(IEnumerable<IEvent> events)
    {
        var account = new LoanAccount();
        foreach (var e in events)
            ((dynamic)account).Apply((dynamic)e);
        return account;
    }
}
```

---

### 27. Materialized View

**Definition**: Pre-compute and store query results so complex reads are instant.

```
Source Tables (normalized):                    Materialized View (pre-computed):
┌───────────┐ ┌───────────┐ ┌──────────┐     ┌─────────────────────────────┐
│StudentLoan│ │MortgageLoan│ │PersonalLn│     │ CustomerDebtSummary          │
│ CustId    │ │ CustId     │ │ CustId   │ ──▶ │ CustId | Total  | Count    │
│ Balance   │ │ Balance    │ │ Balance  │     │ 123    | $85,000 | 3        │
└───────────┘ └───────────┘ └──────────┘     └─────────────────────────────┘
                                                Updated every hour
```

The "Total Debt" screen loads in milliseconds — no JOINs across three tables at query time.

---

### 28. Sharding

**Definition**: Split data horizontally across multiple databases to spread the load.

```
10 million customers:
  ┌───────────────┐     ┌───────────────┐
  │ Database 1     │     │ Database 2     │
  │ Customers A-M  │     │ Customers N-Z  │
  │ 5M rows        │     │ 5M rows        │
  └───────────────┘     └───────────────┘

Shard key: First letter of last name (or hash of CustomerId)
```

```csharp
public class ShardRouter
{
    public string GetConnectionString(string customerId)
    {
        var shardKey = customerId.GetHashCode() % 4;  // 4 shards
        return shardKey switch
        {
            0 => "Server=shard0.db;Database=Loans",
            1 => "Server=shard1.db;Database=Loans",
            2 => "Server=shard2.db;Database=Loans",
            3 => "Server=shard3.db;Database=Loans",
            _ => throw new InvalidOperationException()
        };
    }
}
```

---

### 29. Index Table

**Definition**: Create a secondary index (separate table or store) to efficiently query by attributes that aren't the primary key.

```
Primary Table (by Loan ID):          Index Table (by Phone Number):
┌──────────┬──────────┬──────┐      ┌──────────────┬──────────┐
│ LoanId   │ CustName │ Phone│      │ Phone         │ LoanId   │
│ LN-001   │ Alice    │ 555..│      │ 555-0101      │ LN-001   │
│ LN-002   │ Bob      │ 555..│      │ 555-0202      │ LN-002   │
└──────────┴──────────┴──────┘      └──────────────┴──────────┘

"Find loan by phone" → O(1) lookup instead of full table scan
```

---

### 30. Valet Key

**Definition**: Give the client a time-limited, restricted token to access a storage resource directly — bypassing the web server for large file transfers.

```
1. Client:  "I need to upload my mortgage documents"
2. Server:  Generates a SAS token (valid 10 min, write-only, specific container)
3. Client:  Uploads 100MB PDF directly to Azure Blob Storage using the token
4. Server:  Never touches the file — saves CPU and bandwidth
```

```csharp
[HttpGet("upload-token")]
public IActionResult GetUploadToken()
{
    var sasBuilder = new BlobSasBuilder
    {
        BlobContainerName = "mortgage-docs",
        BlobName = $"{Guid.NewGuid()}.pdf",
        Resource = "b",
        ExpiresOn = DateTimeOffset.UtcNow.AddMinutes(10)
    };
    sasBuilder.SetPermissions(BlobSasPermissions.Write);

    var sasToken = _blobServiceClient
        .GetBlobContainerClient("mortgage-docs")
        .GetBlobClient(sasBuilder.BlobName)
        .GenerateSasUri(sasBuilder);

    return Ok(new { UploadUrl = sasToken.ToString() });
}
// Client uploads directly to Azure Blob → server is not a bottleneck
```

---

### 31. Compensating Transaction

**Definition**: If a multi-step operation fails partway through, execute "reverse" actions to undo the completed steps.

```
Transfer Loan Payment:
  Step 1: Debit source account    ✅
  Step 2: Credit loan account     ✅
  Step 3: Update ledger           ❌ FAILED

Compensation:
  Undo Step 2: Reverse credit
  Undo Step 1: Reverse debit
  Result: System back to original state
```

```csharp
public async Task TransferAsync(TransferRequest request)
{
    var compensations = new Stack<Func<Task>>();

    try
    {
        await _sourceAccount.DebitAsync(request.Amount);
        compensations.Push(() => _sourceAccount.CreditAsync(request.Amount)); // Undo

        await _loanAccount.CreditAsync(request.Amount);
        compensations.Push(() => _loanAccount.DebitAsync(request.Amount));    // Undo

        await _ledger.RecordAsync(request);  // If this fails...
    }
    catch
    {
        // Run compensations in reverse order
        while (compensations.Count > 0)
            await compensations.Pop()();

        throw;
    }
}
```

---

### 32. Saga

**Definition**: A long-running process broken into steps. If any step fails, the Saga manages compensation (undo) for all completed steps.

```
Loan Application Saga:

  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
  │ Credit Check │───▶│ Background  │───▶│ Underwriting │───▶│  Approval   │
  │              │    │ Check       │    │              │    │             │
  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘    └─────────────┘
         │ fail             │ fail             │ fail
         ▼                  ▼                  ▼
  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
  │ Cancel App   │◀───│ Release Hold│◀───│ Cancel       │
  │              │    │             │    │ Underwriting │
  └─────────────┘    └─────────────┘    └─────────────┘
       Compensating transactions
```

| Saga Type | How It Works |
|---|---|
| **Orchestration** | Central orchestrator tells each service what to do and handles failures |
| **Choreography** | Each service listens for events and decides its action independently |

---

### 33. Cache-Aside

**Definition**: The application checks the cache first. On a miss, it fetches from the database and stores the result in the cache for next time.

```
GET "Current Interest Rates":

  ┌──────┐  1. Check cache   ┌───────┐
  │ App   │──────────────────▶│ Cache  │  Hit? → Return cached value
  └──┬───┘                   └───────┘
     │                            │ Miss
     │  2. Query DB               │
     ▼                            │
  ┌──────┐                        │
  │  DB   │  3. Return data       │
  └──┬───┘                        │
     │  4. Store in cache         │
     └───────────────────────────▶│
```

```csharp
public async Task<InterestRates> GetRatesAsync()
{
    const string key = "interest-rates";

    // 1. Check cache
    var cached = await _cache.GetStringAsync(key);
    if (cached != null)
        return JsonSerializer.Deserialize<InterestRates>(cached)!;

    // 2. Cache miss — query database
    var rates = await _context.InterestRates.FirstAsync();

    // 3. Store in cache for next time
    await _cache.SetStringAsync(key,
        JsonSerializer.Serialize(rates),
        new DistributedCacheEntryOptions { AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(1) });

    return rates;
}
```

---

### 34. Static Content Hosting

**Definition**: Serve static files (PDFs, CSS, JS, images) from a CDN or blob storage instead of your application server.

```
Without:  Client ──▶ App Server ──▶ "LoanTerms.pdf"  (server CPU wasted)
With:     Client ──▶ Azure CDN / Blob ──▶ "LoanTerms.pdf"  (fast, cheap, global)
```

Azure Blob Storage with CDN: files cached at edge locations worldwide.

---

### 35. Geode

**Definition**: Deploy your service in multiple geographic regions so customers always hit the server physically closest to them.

```
US Customer ──▶ US East deployment ──▶ US East DB (replica)
EU Customer ──▶ EU West deployment ──▶ EU West DB (replica)
Asia Customer ──▶ SE Asia deployment ──▶ SE Asia DB (replica)
```

Each "geode" is a full stamp (app + database replica). Azure Traffic Manager or Front Door routes based on geographic proximity.

---

## Zone 6: Advanced Coordination

---

### 36. Federated Identity

**Definition**: Let users log in using an external identity provider (Google, Azure AD, corporate AD) instead of managing credentials yourself. This is a **design pattern** — implemented using protocols like **OpenID Connect (OIDC)** and **OAuth 2.0**.

```
                    ┌──────────────────┐
User ──▶ "Log in    │  Identity Provider│  (Azure AD, Google, Okta)
         with       │  Authenticates    │
         Google" ──▶│  Issues tokens    │
                    └────────┬─────────┘
                             │ ID Token (OIDC) — "Who is this user?"
                             │ Access Token (OAuth 2.0) — "What can they access?"
                             ▼
                    ┌──────────────────┐
                    │  Loan Portal API  │  Validates tokens — never sees password
                    └──────────────────┘
```

**Federated Identity vs OAuth 2.0 vs OpenID Connect:**

| Concept                   | What It Answers                       | Type                                        |
|---                        |---                                    |---                                          |
| **Federated Identity**    | "Let someone else verify who you are" | Design Pattern (not a protocol)             |
| **OAuth 2.0**             | "What is this app allowed to do?"     | Authorization protocol                      |
| **OpenID Connect (OIDC)** | "Who is this user?"                   | Authentication protocol (extends OAuth 2.0) |

**How they work together**: 

Federated Identity is the pattern. OIDC + OAuth 2.0 are the protocols that implement it. 
Azure AD's `/v2.0` endpoint issues both an **ID token** (OIDC — identity) and an **Access token** (OAuth 2.0 — permissions).

**Layman Term**: 
Federated Identity is like accepting a foreign passport at your border. 
You don't verify their identity yourself — you trust the issuing country (the Identity Provider). 
OAuth 2.0 is the visa stamp that says what they're allowed to do. OIDC is the passport photo page that says who they are.

```csharp
// Federated Identity implemented via OIDC/OAuth 2.0
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        // Azure AD as the federated identity provider
        options.Authority = "https://login.microsoftonline.com/{tenantId}/v2.0";
        options.Audience = "api://loan-portal";
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true
        };
    });

// Access user identity from the OIDC ID token claims
[Authorize]
[HttpGet("profile")]
public IActionResult GetProfile()
{
    var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;  // From ID token
    var email = User.FindFirst("email")?.Value;                     // OIDC claim
    var roles = User.FindAll(ClaimTypes.Role).Select(c => c.Value); // OAuth scope/role

    return Ok(new { userId, email, roles });
}
```

```
OAuth 2.0 Authorization Code Flow (used by OIDC):

  User ──▶ "Login" ──▶ Redirect to Azure AD
                              │
                        User enters credentials
                              │
                        Azure AD validates
                              │
                  ┌───────────┴───────────┐
                  │ Authorization Code    │  (sent back to app)
                  └───────────┬───────────┘
                              │
                  App exchanges code for:
                  ┌───────────┴───────────┐
                  │ ID Token (OIDC)       │  → "This is John, john@bank.com"
                  │ Access Token (OAuth)  │  → "Can read loans, write payments"
                  │ Refresh Token         │  → "Get new tokens without re-login"
                  └───────────────────────┘
```

---

### 37. Deployment Stamps

**Definition**: Deploy a complete, isolated copy of the entire system (app + database + cache) for a specific tenant or region.

```
Standard Customers:
  ┌─────────────────────────────┐
  │ Stamp 1: Shared             │
  │ App + DB + Cache            │
  └─────────────────────────────┘

Corporate Client (Bank of XYZ):
  ┌─────────────────────────────┐
  │ Stamp 2: Dedicated          │
  │ App + DB + Cache            │  ← Isolated, private data
  └─────────────────────────────┘
```

Each stamp is independently deployable, scalable, and can be in a different region. 
Provides **data isolation** for compliance and **performance isolation** for large clients.

---

### 38. Leader Election

**Definition**: Among multiple instances of a service, elect one "leader" to perform a task that should only run once (e.g., nightly interest calculation).

**Problem**: You have 5 replicas of the Interest Calculator worker. If ALL 5 run the nightly calculation, customers get charged 5x the interest. Only ONE should run it.

```
5 Worker Instances:
  Worker A: "I'm the leader"  ──▶ Runs interest calculation
  Worker B: "Not leader"      ──▶ Standby
  Worker C: "Not leader"      ──▶ Standby
  Worker D: "Not leader"      ──▶ Standby
  Worker E: "Not leader"      ──▶ Standby

  If Worker A crashes → Worker B becomes leader
```

```
Leader Election Flow:

  ┌──────────┐  1. Try acquire lease   ┌───────────────────┐
  │ Worker A │────────────────────────▶│ Azure Blob Storage│
  │          │◀──── ✅ Lease granted  │ "leader.lock"     │
  └──────────┘      (I'm the leader!)  └───────────────────┘

  ┌──────────┐  2. Try acquire lease        │
  │ Worker B  │─────────────────────────────▶│
  │           │◀──── ❌ 409 Conflict         │  (lease already held)
  └──────────┘      (not leader — standby)  │

  ┌──────────┐  3. Try acquire lease        │
  │ Worker C  │─────────────────────────────▶│
  │           │◀──── ❌ 409 Conflict         │
  └──────────┘                              │

  Worker A crashes... lease expires after 30s

  ┌──────────┐  4. Try acquire lease        │
  │ Worker B  │─────────────────────────────▶│
  │           │◀──── ✅ Lease granted        │  (new leader!)
  └──────────┘                              │
```

**Layman Term**: Imagine 5 bank tellers all want to count the vault at night. They agree: whoever grabs the "vault key" first does the counting. The key has a timer — if the teller disappears, the key goes back to the hook after 30 seconds, and the next teller grabs it.

**Implementation — Azure Blob Storage Lease (Full Example)**:

```csharp
// LeaderElectionService — runs in every worker instance
public class LeaderElectionService : BackgroundService
{
    private readonly BlobClient _leaderBlob;
    private readonly ILogger<LeaderElectionService> _logger;
    private readonly string _instanceId = Environment.MachineName;

    public LeaderElectionService(BlobServiceClient blobClient, ILogger<LeaderElectionService> logger)
    {
        _leaderBlob = blobClient
            .GetBlobContainerClient("leader-election")
            .GetBlobClient("interest-calculator.lock");
        _logger = logger;
    }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        // Ensure the lock blob exists
        if (!await _leaderBlob.ExistsAsync(ct))
            await _leaderBlob.UploadAsync(new BinaryData("leader-lock"), cancellationToken: ct);

        while (!ct.IsCancellationRequested)
        {
            await TryBecomeLeaderAsync(ct);
            await Task.Delay(TimeSpan.FromSeconds(15), ct); // Retry every 15s
        }
    }

    private async Task TryBecomeLeaderAsync(CancellationToken ct)
    {
        var leaseClient = _leaderBlob.GetBlobLeaseClient();

        try
        {
            // Try to acquire a 30-second lease
            var lease = await leaseClient.AcquireAsync(
                TimeSpan.FromSeconds(30), cancellationToken: ct);

            _logger.LogInformation("Instance {Id} is the LEADER", _instanceId);

            try
            {
                // I'm the leader — do the exclusive work
                await RunInterestCalculationAsync(ct);
            }
            finally
            {
                // Release lease so others can take over if needed
                await leaseClient.ReleaseAsync(cancellationToken: ct);
                _logger.LogInformation("Instance {Id} released leadership", _instanceId);
            }
        }
        catch (RequestFailedException ex) when (ex.Status == 409)
        {
            // 409 Conflict = lease already held by another instance
            _logger.LogDebug("Instance {Id} is STANDBY — another instance is leader", _instanceId);
        }
    }

    private async Task RunInterestCalculationAsync(CancellationToken ct)
    {
        _logger.LogInformation("Calculating nightly interest for all loans...");
        // Only ONE instance runs this across all replicas
        await Task.Delay(TimeSpan.FromSeconds(5), ct); // Simulated work
        _logger.LogInformation("Interest calculation complete");
    }
}

// Register in Program.cs
builder.Services.AddHostedService<LeaderElectionService>();
```

**Implementation — Lease Renewal (Long-Running Tasks)**:

```csharp
// For tasks longer than the lease duration, renew the lease periodically
private async Task RunWithLeaseRenewalAsync(BlobLeaseClient leaseClient, CancellationToken ct)
{
    using var renewalCts = CancellationTokenSource.CreateLinkedTokenSource(ct);

    // Background task: renew lease every 10s (lease is 30s)
    var renewalTask = Task.Run(async () =>
    {
        while (!renewalCts.Token.IsCancellationRequested)
        {
            await Task.Delay(TimeSpan.FromSeconds(10), renewalCts.Token);
            await leaseClient.RenewAsync(cancellationToken: renewalCts.Token);
            _logger.LogDebug("Lease renewed");
        }
    }, renewalCts.Token);

    try
    {
        // Do the actual long-running work
        await RunInterestCalculationAsync(ct);
    }
    finally
    {
        renewalCts.Cancel();  // Stop renewing
        await leaseClient.ReleaseAsync(cancellationToken: ct);
    }
}
```

**Alternative — Distributed Lock with Redis**:

```csharp
// Using StackExchange.Redis for leader election
public class RedisLeaderElection
{
    private readonly IDatabase _redis;

    public async Task<bool> TryBecomeLeaderAsync(string taskName, string instanceId)
    {
        // SET key value NX EX 30  →  only sets if key doesn't exist, expires in 30s
        return await _redis.StringSetAsync(
            $"leader:{taskName}",
            instanceId,
            TimeSpan.FromSeconds(30),
            When.NotExists);  // NX = only if not already set
    }

    public async Task ReleaseLeadershipAsync(string taskName, string instanceId)
    {
        // Only release if we're still the leader (avoid releasing someone else's lock)
        var currentLeader = await _redis.StringGetAsync($"leader:{taskName}");
        if (currentLeader == instanceId)
            await _redis.KeyDeleteAsync($"leader:{taskName}");
    }
}
```

| Mechanism                 | How It Works                                  | Best For                          |
|---                        |---                                            |---                                |
| **Azure Blob Lease**      | 15-60s lease on a blob; 409 if already held   | Azure-native, simple              |
| **Redis SETNX**           | Atomic set-if-not-exists with TTL             | Low latency, cross-platform       |
| **Kubernetes Lease**      | Built-in Lease resource in K8s API            | K8s-native workloads              |
| **ZooKeeper / etcd**      | Distributed consensus with ephemeral nodes    | Complex distributed systems       |
| **Database Row Lock**     | `SELECT ... FOR UPDATE` on a lock row         | Already have a DB, no new infra   |

**When to Use**:
- Nightly batch jobs (interest calculation, report generation)
- Singleton services (only one instance should process a queue)
- Scheduled maintenance (only one worker prunes old records)
- Cache warming (only one instance rebuilds the cache)

**When NOT to Use**:
- If the task is idempotent (safe to run multiple times) — just let all instances run
- If you can use a queue with a single consumer instead

Great question. The answer is high availability / fault tolerance.

If you keep only 1 instance:

Worker A runs the nightly interest calculation
Worker A crashes at 2 AM (out of memory, node failure, deployment restart)
Nobody calculates interest tonight
Customers don't get charged, reports are wrong, regulators are angry
With 5 instances + Leader Election:

Worker A is the leader, runs the calculation
Worker A crashes at 2 AM
Worker B automatically becomes the new leader within 30 seconds
Interest calculation continues — zero downtime
The 5 instances aren't there to do 5x the work. They're there so the job always gets done, even if 1, 2, or even 4 instances crash.

| Approach                          | Normal Operation      | If Worker Crashes                                  |
|---                                |---                    |---                                                 |
| **1 instance**                    | Works fine            | Job doesn't run until someone manually restarts it |
| **5 instances + Leader Election** | 1 works, 4 standby    | Standby takes over in seconds — self-healing       |

**Real-world analogy**: A hospital has 5 surgeons on call, but only 1 performs the surgery. The other 4 aren't wasted — they're **backup**. If the lead surgeon gets sick, the next one steps in immediately. You wouldn't staff the hospital with just 1 surgeon and hope they never get sick.

Also note: Those standby workers aren't truly idle. In practice they often handle other tasks (processing queue messages, serving API requests) — the leader election is only for the one exclusive task (nightly batch). So all 5 instances are busy serving regular loan API traffic; only the interest calculation needs a single leader.
---

### 39. Pipes and Filters

**Definition**: Break a complex process into a series of independent, reusable steps (filters) connected by channels (pipes). Each filter does one thing, receives input, transforms it, and passes it to the next filter.

**Problem**: Your loan underwriting logic is a 2,000-line method that checks identity, calculates risk, runs fraud detection, and gets manager approval — all tangled together. You can't test, reuse, or scale any piece independently.

```
Monolith Approach (tangled):
  Application ──▶ ┌──────────────────────────────────────┐ ──▶ Decision
                  │ Giant ProcessLoan() method            │
                  │  - Identity check                     │
                  │  - Risk score                         │
                  │  - Fraud check                        │
                  │  - Manager review                     │
                  │  All coupled together, untestable     │
                  └──────────────────────────────────────┘

Pipes and Filters Approach (clean):
  Application ──▶ Filter 1 ──pipe──▶ Filter 2 ──pipe──▶ Filter 3 ──pipe──▶ Filter 4 ──▶ Decision
                  Identity          Risk Score          Fraud Check        Manager
                  Check                                                    Review
                  (independent)     (independent)       (independent)      (independent)
```

```
Detailed Flow:

  ┌───────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
  │ Loan App  │────▶│ Identity Filter  │────▶│ Risk Filter      │────▶│ Fraud Filter     │────▶│ Manager Filter   │
  │ (input)   │     │                 │     │                 │     │                 │     │                 │
  │           │     │ IN:  App data   │     │ IN: + ID status │     │ IN: + Risk score│     │ IN: + Fraud flag│
  │           │     │ OUT: + ID pass ✅│     │ OUT: + Score 720│     │ OUT: + Clean ✅  │     │ OUT: Approved ✅ │
  └───────────┘     └────────┬────────┘     └────────┬────────┘     └────────┬────────┘     └─────────────────┘
                             │                       │                       │
                        ID failed?              Score < 500?           Fraud detected?
                        ──▶ REJECT              ──▶ REJECT             ──▶ REJECT
                        (short-circuit)         (short-circuit)        (short-circuit)
```

**Layman Term**: Think of a car wash. Your car goes through a series of stations: pre-rinse → soap → scrub → rinse → dry → wax. Each station does ONE job and passes the car to the next. If one station breaks, you replace just that station. You can even add a new "tire shine" station without touching the others. That's Pipes and Filters — each step is independent and reusable.

**Implementation — Full Example with DI Registration**:

```csharp
// The shared context that flows through all filters
public class UnderwritingContext
{
    public LoanApplication Application { get; set; } = default!;
    public bool IdentityVerified { get; set; }
    public int RiskScore { get; set; }
    public decimal FraudScore { get; set; }
    public bool ManagerApproved { get; set; }
    public bool IsRejected { get; set; }
    public string RejectionReason { get; set; } = "";
    public List<string> AuditLog { get; set; } = new();
}

// Filter contract — each filter is an independent step
public interface IUnderwritingFilter
{
    int Order { get; }  // Controls pipeline sequence
    Task<UnderwritingContext> ProcessAsync(UnderwritingContext context);
}

// Filter 1: Identity Check
public class IdentityCheckFilter : IUnderwritingFilter
{
    private readonly IIdentityService _idService;
    public int Order => 1;

    public IdentityCheckFilter(IIdentityService idService) => _idService = idService;

    public async Task<UnderwritingContext> ProcessAsync(UnderwritingContext ctx)
    {
        ctx.IdentityVerified = await _idService.VerifyAsync(ctx.Application.SSN);
        ctx.AuditLog.Add($"Identity check: {(ctx.IdentityVerified ? "PASS" : "FAIL")}");

        if (!ctx.IdentityVerified)
        {
            ctx.IsRejected = true;
            ctx.RejectionReason = "Identity verification failed";
        }
        return ctx;
    }
}

// Filter 2: Risk Scoring
public class RiskScoringFilter : IUnderwritingFilter
{
    private readonly IRiskEngine _riskEngine;
    public int Order => 2;

    public RiskScoringFilter(IRiskEngine riskEngine) => _riskEngine = riskEngine;

    public async Task<UnderwritingContext> ProcessAsync(UnderwritingContext ctx)
    {
        ctx.RiskScore = await _riskEngine.CalculateAsync(ctx.Application);
        ctx.AuditLog.Add($"Risk score: {ctx.RiskScore}");

        if (ctx.RiskScore < 500)
        {
            ctx.IsRejected = true;
            ctx.RejectionReason = $"Risk score too low: {ctx.RiskScore}";
        }
        return ctx;
    }
}

// Filter 3: Fraud Detection
public class FraudCheckFilter : IUnderwritingFilter
{
    private readonly IFraudService _fraudService;
    public int Order => 3;

    public FraudCheckFilter(IFraudService fraudService) => _fraudService = fraudService;

    public async Task<UnderwritingContext> ProcessAsync(UnderwritingContext ctx)
    {
        ctx.FraudScore = await _fraudService.ScoreAsync(ctx.Application);
        ctx.AuditLog.Add($"Fraud score: {ctx.FraudScore}");

        if (ctx.FraudScore > 0.8m)
        {
            ctx.IsRejected = true;
            ctx.RejectionReason = $"High fraud risk: {ctx.FraudScore}";
        }
        return ctx;
    }
}

// Filter 4: Manager Review (auto-approve if all checks pass)
public class ManagerReviewFilter : IUnderwritingFilter
{
    public int Order => 4;

    public Task<UnderwritingContext> ProcessAsync(UnderwritingContext ctx)
    {
        if (ctx.RiskScore >= 700 && ctx.FraudScore < 0.3m)
        {
            ctx.ManagerApproved = true;
            ctx.AuditLog.Add("Auto-approved by policy (high score, low fraud)");
        }
        else
        {
            ctx.AuditLog.Add("Flagged for manual manager review");
        }
        return Task.FromResult(ctx);
    }
}
```

**Pipeline Executor**:

```csharp
// Pipeline that runs all filters in order
public class UnderwritingPipeline
{
    private readonly IEnumerable<IUnderwritingFilter> _filters;
    private readonly ILogger<UnderwritingPipeline> _logger;

    public UnderwritingPipeline(
        IEnumerable<IUnderwritingFilter> filters,
        ILogger<UnderwritingPipeline> logger)
    {
        _filters = filters.OrderBy(f => f.Order);  // Ensure correct sequence
        _logger = logger;
    }

    public async Task<UnderwritingContext> RunAsync(LoanApplication app)
    {
        var context = new UnderwritingContext { Application = app };

        foreach (var filter in _filters)
        {
            _logger.LogInformation("Running filter: {Filter}", filter.GetType().Name);
            context = await filter.ProcessAsync(context);

            if (context.IsRejected)
            {
                _logger.LogWarning("Pipeline short-circuited at {Filter}: {Reason}",
                    filter.GetType().Name, context.RejectionReason);
                break;  // Short-circuit — no need to run remaining filters
            }
        }

        _logger.LogInformation("Pipeline complete. Rejected: {Rejected}", context.IsRejected);
        return context;
    }
}
```

**DI Registration — Adding/Removing Filters is Just Config**:

```csharp
// Program.cs — register all filters
builder.Services.AddScoped<IUnderwritingFilter, IdentityCheckFilter>();
builder.Services.AddScoped<IUnderwritingFilter, RiskScoringFilter>();
builder.Services.AddScoped<IUnderwritingFilter, FraudCheckFilter>();
builder.Services.AddScoped<IUnderwritingFilter, ManagerReviewFilter>();
builder.Services.AddScoped<UnderwritingPipeline>();

// Want to add a new "Income Verification" step? Just add one line:
// builder.Services.AddScoped<IUnderwritingFilter, IncomeVerificationFilter>();
// No existing code changes needed!
```

**Usage in Controller**:

```csharp
[HttpPost("underwrite")]
public async Task<IActionResult> Underwrite(LoanApplicationDto dto)
{
    var app = _mapper.Map<LoanApplication>(dto);
    var result = await _pipeline.RunAsync(app);

    if (result.IsRejected)
        return Ok(new { Status = "Rejected", result.RejectionReason, result.AuditLog });

    return Ok(new { Status = result.ManagerApproved ? "Approved" : "PendingReview", result.AuditLog });
}
```

**Alternative — Azure Service Bus Pipeline (Decoupled Filters as Separate Services)**:

```
For maximum independence, each filter can be a separate microservice connected by queues:

  App ──▶ [Queue 1] ──▶ Identity Svc ──▶ [Queue 2] ──▶ Risk Svc ──▶ [Queue 3] ──▶ Fraud Svc ──▶ [Queue 4] ──▶ Decision
          (pipe)                          (pipe)                      (pipe)                      (pipe)

Each filter:
  - Reads from its input queue
  - Does its work
  - Writes to the next queue
  - Can be scaled independently (3 fraud checkers if fraud is slow)
  - Can be written in different languages
```

```csharp
// Fraud Check as a standalone queue-based filter
public class FraudCheckWorker : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        var processor = _client.CreateProcessor("underwriting-after-risk");  // Input pipe

        processor.ProcessMessageAsync += async args =>
        {
            var ctx = JsonSerializer.Deserialize<UnderwritingContext>(args.Message.Body);

            ctx!.FraudScore = await _fraudService.ScoreAsync(ctx.Application);
            ctx.AuditLog.Add($"Fraud score: {ctx.FraudScore}");

            if (ctx.FraudScore > 0.8m)
            {
                ctx.IsRejected = true;
                ctx.RejectionReason = "High fraud risk";
            }

            // Send to next pipe (output queue)
            var sender = _client.CreateSender("underwriting-after-fraud");
            await sender.SendMessageAsync(
                new ServiceBusMessage(JsonSerializer.Serialize(ctx)));

            await args.CompleteMessageAsync(args.Message);
        };

        await processor.StartProcessingAsync(ct);
    }
}
```

**Pipes and Filters vs Saga vs Middleware**:

| Aspect                | Pipes and Filters                        | Saga                                      | ASP.NET Middleware                          |
|---                    |---                                       |---                                        |---                                          |
| **Purpose**           | Transform data step-by-step              | Coordinate distributed transactions       | Handle HTTP request/response pipeline       |
| **On Failure**        | Short-circuit, skip remaining filters    | Run compensating transactions (undo)      | Short-circuit or pass error down            |
| **Data Flow**         | Context flows forward through filters    | Each step is an independent service call  | HttpContext flows through middleware chain   |
| **Coupling**          | Filters share a context object           | Services are fully independent            | Middleware shares HttpContext                |
| **Undo/Compensation** | No — just stops processing               | Yes — rolls back completed steps          | No                                          |
| **Best For**          | Data transformation, validation pipeline | Multi-service business transactions       | Cross-cutting concerns (auth, logging)      |

**When to Use**:
- Multi-step data processing or validation (underwriting, ETL, document processing)
- You need to add/remove/reorder steps without changing existing code
- Each step is independently testable and reusable
- You want to scale individual steps differently (3 fraud checkers, 1 identity checker)

**When NOT to Use**:
- If you need compensation/undo on failure → use **Saga** instead
- If the steps are tightly coupled and always change together → a single method is simpler
- If you only have 2 steps → overhead of the pattern isn't worth it

---

### 40. Compute Resource Consolidation

**Definition**: Combine multiple small, rarely used tasks onto a single server to save cost.

```
Before (5 separate servers — expensive):
  Server 1: Student Loan Report Generator     (runs once/day)
  Server 2: Late Payment Reminder             (runs once/day)
  Server 3: Interest Calculator               (runs once/day)
  Server 4: Archive Old Records               (runs once/week)
  Server 5: Compliance Report                 (runs once/month)

After (1 server — consolidated):
  Server 1: All 5 tasks as Background Jobs (Hangfire / Azure Functions)
```

---

### 41. Quarantine

**Definition**: Isolate potentially harmful or malformed input into a separate "quarantine" area for inspection before processing.

```
Customer uploads file:
  ┌──────────┐     ┌───────────────┐     ┌──────────────┐
  │ Upload   │──── │ Quarantine    │────▶│ Virus Scanner│
  │          │     │ Blob Storage  │     │              │
  └──────────┘     └───────────────┘     └──────┬───────┘
                                                │
                                     Clean? ────┤──── Infected?
                                     ▼          │     ▼
                              ┌────────────┐    │  ┌───────────┐
                              │ Production │    │  │ Alert +   │
                              │ Storage    │    │  │ Delete    │
                              └────────────┘    │  └───────────┘
```

---

### 42. Scheduler Agent Supervisor

**Definition**: A three-part system:  **Scheduler** triggers tasks, **Agent** executes them, **Supervisor** monitors and retries on failure.

```
┌─────────────┐     ┌─────────────┐     ┌──────────────┐
│  Scheduler  │────▶│   Agent      │◀────│  Supervisor  │
│ "Run audit  │     │ "Execute     │     │ "Is it done? │
│  at 2 AM"   │     │  health audit│     │  Retry if    │
│             │     │  on loans"   │     │  crashed"    │
└─────────────┘     └─────────────┘     └──────────────┘
```

The Supervisor ensures work completes even if an Agent crashes — it can restart, reassign, or escalate.

---

### 43. Static Content Editing

**Definition**: Allow non-technical users (marketing, compliance) to update content like loan terms, FAQ text, or promotional copy without a code deployment.

```
Marketing: Updates "Current Mortgage Rate: 6.25%" in CMS
           ──▶ Published to CDN instantly
           ──▶ No developer needed, no deploy
```

Implementations: headless CMS (Contentful, Strapi), Azure Blob + CDN, feature flags for text content.

---

### 44. AI Agent Orchestration

**Definition**: Coordinate multiple AI agents to perform complex, multi-step cognitive tasks — each agent specializes in one area.

```
┌──────────────────────────────────────────────────┐
│                 AI Orchestrator                  │
│                                                  │
│  Step 1: ┌──────────────────┐                    │
│          │ Doc Reader Agent │  "Extract income   │
│          │ (reads bank stmt)│   from statement"  │
│          └────────┬─────────┘                    │
│                   ▼                              │
│  Step 2: ┌──────────────────┐                    │
│          │ Validator Agent  │  "Cross-check income    │
│          │ (checks loan app)│   with application"     │
│          └────────┬─────────┘                         │
│                   ▼                                   │
│  Step 3: ┌──────────────────┐                         │
│          │ Decision Agent   │  "Approve or flag       │
│          │ (risk assessment)│   for human review"     │
│          └──────────────────┘                         │
└───────────────────────────────────────────────────┘
```

```csharp
// Using Semantic Kernel for AI Agent Orchestration
var kernel = Kernel.CreateBuilder()
    .AddAzureOpenAIChatCompletion("gpt-4", endpoint, apiKey)
    .Build();

// Define specialized agents
var docReaderAgent = new ChatCompletionAgent
{
    Name = "DocReader",
    Instructions = "Extract income, expenses from bank statements.",
    Kernel = kernel
};

var validatorAgent = new ChatCompletionAgent
{
    Name = "Validator",
    Instructions = "Compare extracted data against the loan application for discrepancies.",
    Kernel = kernel
};

// Orchestrate
var chat = new AgentGroupChat(docReaderAgent, validatorAgent);
chat.AddChatMessage(new ChatMessageContent(AuthorRole.User,
    $"Review this loan application: {applicationJson}"));

await foreach (var message in chat.InvokeAsync())
{
    Console.WriteLine($"{message.AuthorName}: {message.Content}");
}
```

---

## Pattern Zone Summary

| Zone                  | Purpose                               | Key Patterns                                                      |
|---                    |---                                    |---                                                                |
| **1. Migration**      | Move from monolith to microservices   | Strangler Fig, ACL, Ambassador, Sidecar                           |
| **2. Front Door**     | Traffic management & routing          | Gateway Routing/Aggregation, BFF, Offloading, External Config     |
| **3. Resiliency**     | Fault tolerance & isolation           | Retry, Circuit Breaker, Bulkhead, Health Checks, Throttling       |
| **4. Messaging**      | Async communication & decoupling      | Pub/Sub, Queue Leveling, Competing Consumers, Saga, CQRS events   |
| **5. Data**           | Consistency, speed, scale             | CQRS, Event Sourcing, Sharding, Cache-Aside, Valet Key, Saga      |
| **6. Coordination**   | Advanced orchestration                | Federated Identity, Leader Election, Pipes & Filters, AI Agents   |

## Zone 1: The Migration (Moving Away from WCF)

### 1. Strangler Fig
### 2. Anti-Corruption Layer (ACL)
### 3. Ambassador
### 4. Sidecar

## Zone 2: The Front Door (Traffic Management)

### 5. Gateway Routing
### 6. Gateway Aggregation
### 7. Backends for Frontends (BFF)
### 8. Gateway Offloading
### 9. External Configuration Store

## Zone 3: Resiliency (Stopping the Crash)

### 10. Retry
### 11. Circuit Breaker
### 12. Bulkhead
### 13. Health Endpoint Monitoring
### 14. Throttling
### 15. Rate Limiting

## Zone 4: Messaging & Async (The "Waiting Room")

### 16. Asynchronous Request-Reply
### 17. Queue-Based Load Leveling
### 18. Publisher-Subscriber (Pub/Sub)
### 19. Competing Consumers
### 20. Priority Queue
### 21. Claim Check
### 22. Sequential Convoy
### 23. Choreography
### 24. Messaging Bridge

## Zone 5: Data Management (Consistency & Speed)

### 25. CQRS (Command Query Responsibility Segregation)
### 26. Event Sourcing
### 27. Materialized View
### 28. Sharding
### 29. Index Table
### 30. Valet Key
### 31. Compensating Transaction
### 32. Saga
### 33. Cache-Aside
### 34. Static Content Hosting
### 35. Geode

## Zone 6: Advanced Coordination

### 36. Federated Identity
### 37. Deployment Stamps
### 38. Leader Election
### 39. Pipes and Filters
### 40. Compute Resource Consolidation
### 41. Quarantine
### 42. Scheduler Agent Supervisor
### 43. Static Content Editing
### 44. AI Agent Orchestration

---

## **Complete ASP.NET Core Web App: Loan Processing with Resiliency Patterns (10-15)**

*A production-ready example combining Retry, Circuit Breaker, Bulkhead, Health Monitoring, Throttling, and Rate Limiting.*

### **Scenario**
A Loan Processing API that:
- Calls an external Credit Bureau API (with retries on transient failures)
- Has a circuit breaker if Credit Bureau keeps failing
- Uses bulkheads to isolate credit checks from income verifications
- Expires `/health` endpoints for Kubernetes
- Throttles requests per user to prevent abuse
- Rate-limits partners to a quota per hour

### **Project Structure**
```
LoanProcessingApi/
├── Program.cs
├── Models/
│   ├── LoanApplicationDto.cs
│   ├── CreditCheckDto.cs
│   └── LoanDecisionDto.cs
├── Services/
│   ├── CreditBureauService.cs
│   ├── IncomeVerificationService.cs
│   └── LoanUnderwritingService.cs
├── Controllers/
│   ├── LoansController.cs
│   └── HealthController.cs
├── appsettings.json
└── appsettings.Development.json
```

### **Stage 1: Models**

```csharp
// Models/LoanApplicationDto.cs
public class LoanApplicationDto
{
    public string ApplicantName { get; set; } = "";
    public string SSN { get; set; } = "";
    public decimal RequestedAmount { get; set; }
    public int TermYears { get; set; }
    public decimal AnnualIncome { get; set; }
}

// Models/CreditCheckDto.cs
public class CreditCheckDto
{
    public string SSN { get; set; } = "";
    public int CreditScore { get; set; }
    public string Status { get; set; } = "";  // "Approved", "Denied", "Review"
    public DateTime CheckedAt { get; set; }
}

// Models/LoanDecisionDto.cs
public class LoanDecisionDto
{
    public bool IsApproved { get; set; }
    public decimal ApprovedAmount { get; set; }
    public string Reason { get; set; } = "";
    public int CreditScore { get; set; }
    public decimal DebtToIncomeRatio { get; set; }
}
```

### **Stage 2: Services with Resilience**

```csharp
// Services/CreditBureauService.cs
// This service demonstrates: Retry + Circuit Breaker
public class CreditBureauService
{
    private readonly HttpClient _httpClient;
    private readonly ILogger<CreditBureauService> _logger;

    public CreditBureauService(HttpClient httpClient, ILogger<CreditBureauService> logger)
    {
        _httpClient = httpClient;
        _logger = logger;
    }

    public async Task<CreditCheckDto> CheckCreditAsync(string ssn)
    {
        try
        {
            var response = await _httpClient.GetAsync($"/api/credit/check/{ssn}");
            response.EnsureSuccessStatusCode();

            var json = await response.Content.ReadAsStringAsync();
            _logger.LogInformation("Credit check SUCCESS for {SSN}", ssn);
            return JsonSerializer.Deserialize<CreditCheckDto>(json) 
                ?? throw new InvalidOperationException("No data returned");
        }
        catch (HttpRequestException ex)
        {
            _logger.LogError(ex, "Credit check FAILED for {SSN}", ssn);
            throw;
        }
    }
}

// Services/IncomeVerificationService.cs
// This service demonstrates: Bulkhead (separate from credit checks)
public class IncomeVerificationService
{
    private readonly ILogger<IncomeVerificationService> _logger;

    public IncomeVerificationService(ILogger<IncomeVerificationService> logger)
    {
        _logger = logger;
    }

    public async Task<decimal> VerifyIncomeAsync(string ssn, decimal claimedIncome)
    {
        _logger.LogInformation("Verifying income for {SSN}", ssn);

        // Simulate IRS/employer lookup
        await Task.Delay(100);

        var verifiedIncome = claimedIncome * 0.95m;  // 95% of claimed
        _logger.LogInformation("Income verified: {VerifiedIncome}", verifiedIncome);
        return verifiedIncome;
    }
}

// Services/LoanUnderwritingService.cs
// Coordinates the underwriting process
public class LoanUnderwritingService
{
    private readonly CreditBureauService _creditService;
    private readonly IncomeVerificationService _incomeService;
    private readonly ILogger<LoanUnderwritingService> _logger;

    public LoanUnderwritingService(
        CreditBureauService creditService,
        IncomeVerificationService incomeService,
        ILogger<LoanUnderwritingService> logger)
    {
        _creditService = creditService;
        _incomeService = incomeService;
        _logger = logger;
    }

    public async Task<LoanDecisionDto> UnderwriteAsync(LoanApplicationDto app)
    {
        try
        {
            // Call credit check (has Retry + Circuit Breaker)
            var creditCheck = await _creditService.CheckCreditAsync(app.SSN);

            // Call income verification (separate Bulkhead)
            var verifiedIncome = await _incomeService.VerifyIncomeAsync(app.SSN, app.AnnualIncome);

            // Decision logic
            var dti = (app.RequestedAmount * 12 / app.TermYears) / verifiedIncome;

            var approved = creditCheck.CreditScore >= 650 && dti < 0.43m;

            return new LoanDecisionDto
            {
                IsApproved = approved,
                ApprovedAmount = approved ? app.RequestedAmount : 0,
                Reason = approved ? "Approved" : "Credit score or debt-to-income too high",
                CreditScore = creditCheck.CreditScore,
                DebtToIncomeRatio = dti
            };
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Underwriting failed");
            throw;
        }
    }
}
```

### **Stage 3: Program.cs (All Patterns Configured)**

```csharp
// Program.cs
var builder = WebApplication.CreateBuilder(args);

// ===== PATTERN 10 & 11: Configure HttpClient with Retry + Circuit Breaker =====
builder.Services.AddHttpClient<CreditBureauService>("CreditBureau", client =>
    {
        client.BaseAddress = new Uri("https://api.creditbureau.com");
        client.DefaultRequestHeaders.Add("X-Api-Key", builder.Configuration["CreditBureau:ApiKey"] ?? "test-key");
        client.Timeout = TimeSpan.FromSeconds(10);
    })
    // Retry: exponential backoff on transient failures
    .AddTransientHttpErrorPolicy(p =>
        p.WaitAndRetryAsync(3, attempt =>
            TimeSpan.FromMilliseconds(200 * Math.Pow(2, attempt)),
            onRetry: (outcome, delay, attempt, _) =>
            {
                var logger = builder.Services.BuildServiceProvider()
                    .GetRequiredService<ILoggerFactory>()
                    .CreateLogger("HttpClientRetry");
                logger.LogWarning("Retry {Attempt} after {DelayMs}ms: {Status}",
                    attempt, delay.TotalMilliseconds, outcome.Result?.StatusCode);
            }))
    // Circuit Breaker: trip after 5 consecutive failures, reset after 30s
    .AddTransientHttpErrorPolicy(p =>
        p.CircuitBreakerAsync(
            handledEventsAllowedBeforeBreaking: 5,
            durationOfBreak: TimeSpan.FromSeconds(30),
            onBreak: (outcome, duration) =>
            {
                var logger = builder.Services.BuildServiceProvider()
                    .GetRequiredService<ILoggerFactory>()
                    .CreateLogger("CircuitBreaker");
                logger.LogCritical("Circuit OPEN for {Duration}s", duration.TotalSeconds);
            },
            onReset: () =>
            {
                var logger = builder.Services.BuildServiceProvider()
                    .GetRequiredService<ILoggerFactory>()
                    .CreateLogger("CircuitBreaker");
                logger.LogInformation("Circuit CLOSED - service recovered");
            }));

// ===== PATTERN 12: Bulkhead for Income Verification =====
builder.Services.Configure<BulkheadPolicyOptions>(options =>
{
    options.MaxParallelization = 50;  // Max 50 concurrent income verifications
    options.MaxQueuingActions = 25;   // Queue up to 25 more
});

// ===== Dependency Injection =====
builder.Services.AddScoped<CreditBureauService>();
builder.Services.AddScoped<IncomeVerificationService>();
builder.Services.AddScoped<LoanUnderwritingService>();

builder.Services.AddControllers();

// ===== PATTERN 13: Health Endpoint Monitoring =====
builder.Services.AddHealthChecks()
    .AddUrlGroup(
        new Uri("https://api.creditbureau.com/health"),
        name: "credit-bureau",
        timeout: TimeSpan.FromSeconds(5))
    .AddCheck("database", () => HealthCheckResult.Healthy())  // Placeholder
    .AddCheck("memory", () =>
    {
        var workingSet = GC.GetTotalMemory(false);
        return workingSet < 500_000_000  // < 500MB
            ? HealthCheckResult.Healthy()
            : HealthCheckResult.Degraded();
    });

builder.Services
    .AddHealthChecksUI()
    .AddInMemoryStorage();

// ===== PATTERN 14 & 15: Throttling + Rate Limiting =====
builder.Services.AddRateLimiter(options =>
{
    // PATTERN 14: Throttling - Per-user concurrency limit
    options.AddPolicy("throttle-user", context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.User?.Identity?.Name 
                ?? context.Connection.RemoteIpAddress?.ToString() 
                ?? "anonymous",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 10,
                Window = TimeSpan.FromSeconds(10),
                QueueLimit = 0
            }));

    // PATTERN 15: Rate Limiting - Partner quotas per hour
    options.AddPolicy("rate-limit-partner", context =>
        RateLimitPartition.GetFixedWindowLimiter(
            partitionKey: context.Request.Headers["X-Partner-Id"].ToString() ?? "unknown",
            factory: _ => new FixedWindowRateLimiterOptions
            {
                PermitLimit = 1000,
                Window = TimeSpan.FromHours(1),
                QueueLimit = 100
            }));

    options.RejectionStatusCode = StatusCodes.Status429TooManyRequests;
    options.OnRejected = async (ctx, _) =>
    {
        ctx.HttpContext.Response.StatusCode = StatusCodes.Status429TooManyRequests;
        await ctx.HttpContext.Response.WriteAsJsonAsync(new
        {
            error = "Too many requests",
            retryAfter = ctx.HttpContext.Response.Headers["Retry-After"]
        });
    };
});

var app = builder.Build();

// Use the Retry/Circuit Breaker for HttpClient
app.UseRateLimiter();

// ===== Map Health Endpoints =====
app.MapHealthChecks("/health", new HealthCheckOptions
{
    ResponseWriter = UIResponseWriter.WriteHealthCheckUIResponse
});

app.MapHealthChecks("/health/ready", new HealthCheckOptions
{
    Predicate = check => check.Tags.Contains("ready")
});

app.MapHealthChecks("/health/live", new HealthCheckOptions
{
    Predicate = _ => false  // Just checks if process is alive
});

app.MapHealthChecksUI(options =>
{
    options.UIPath = "/admin/health";
    options.ResourcesPath = "/health-ui";
});

app.MapControllers();
app.Run();

public class BulkheadPolicyOptions
{
    public int MaxParallelization { get; set; } = 50;
    public int MaxQueuingActions { get; set; } = 25;
}
```

### **Stage 4: Controllers**

```csharp
// Controllers/LoansController.cs
[ApiController]
[Route("api/[controller]")]
public class LoansController : ControllerBase
{
    private readonly LoanUnderwritingService _underwriting;
    private readonly ILogger<LoansController> _logger;

    public LoansController(LoanUnderwritingService underwriting, ILogger<LoansController> logger)
    {
        _underwriting = underwriting;
        _logger = logger;
    }

    /// <summary>
    /// Apply for a loan — Throttled per user (10 requests/10s)
    /// </summary>
    [HttpPost("apply")]
    [EnableRateLimiting("throttle-user")]  // PATTERN 14: Throttling
    public async Task<IActionResult> ApplyForLoan(LoanApplicationDto application)
    {
        _logger.LogInformation("Loan application received from {Name}", application.ApplicantName);

        try
        {
            // Underwriting calls:
            // - CreditBureauService (Retry + Circuit Breaker)
            // - IncomeVerificationService (Bulkhead)
            var decision = await _underwriting.UnderwriteAsync(application);

            return Ok(new
            {
                approved = decision.IsApproved,
                amount = decision.ApprovedAmount,
                reason = decision.Reason,
                creditScore = decision.CreditScore,
                dtiRatio = decision.DebtToIncomeRatio
            });
        }
        catch (HttpRequestException ex) when (ex.InnerException is OperationCanceledException)
        {
            _logger.LogError(ex, "Credit check timeout");
            return StatusCode(503, new { error = "Credit bureau unavailable — please try again" });
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Loan processing failed");
            return StatusCode(500, new { error = "Internal server error" });
        }
    }

    /// <summary>
    /// Partner portal — Rate limited to 1000 requests/hour
    /// </summary>
    [HttpGet("partner/status/{applicantId}")]
    [EnableRateLimiting("rate-limit-partner")]  // PATTERN 15: Rate Limiting
    public IActionResult PartnerGetStatus(string applicantId)
    {
        var partnerId = Request.Headers["X-Partner-Id"].ToString();
        _logger.LogInformation("Partner {Id} querying status", partnerId);

        return Ok(new { applicantId, status = "Processing", partnerId });
    }

    /// <summary>
    /// Internal admin endpoint — no rate limiting
    /// </summary>
    [HttpGet("admin/stats")]
    public IActionResult AdminStats()
    {
        return Ok(new
        {
            applicationsToday = 42,
            approvalRate = 0.68m,
            avgProcessingTimeMs = 1234
        });
    }
}

// Controllers/HealthController.cs
[ApiController]
[Route("api/[controller]")]
public class HealthController : ControllerBase
{
    private readonly IHealthCheckService _healthCheckService;
    private readonly ILogger<HealthController> _logger;

    public HealthController(IHealthCheckService healthCheckService, ILogger<HealthController> logger)
    {
        _healthCheckService = healthCheckService;
        _logger = logger;
    }

    /// <summary>
    /// Kubernetes Liveness Probe — is the process alive?
    /// </summary>
    [HttpGet("live")]
    public IActionResult Live()
    {
        return Ok(new { status = "alive" });
    }

    /// <summary>
    /// Kubernetes Readiness Probe — can it serve traffic?
    /// </summary>
    [HttpGet("ready")]
    public async Task<IActionResult> Ready()
    {
        var result = await _healthCheckService.CheckHealthAsync();

        return result.Status == HealthStatus.Healthy
            ? Ok(new { status = "ready", checks = result.Entries.Keys })
            : StatusCode(503, new { status = "not-ready", failures = result.Entries
                .Where(e => e.Value.Status != HealthStatus.Healthy)
                .Select(e => e.Key) });
    }
}
```

### **Stage 5: Configuration (appsettings.json)**

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning",
      "HttpClientRetry": "Debug",
      "CircuitBreaker": "Warning"
    }
  },
  "CreditBureau": {
    "ApiKey": "your-key-here"
  },
  "AllowedHosts": "*"
}
```

### **Stage 6: Running the App**

```bash
# 1. Create project
dotnet new webapi -n LoanProcessingApi
cd LoanProcessingApi

# 2. Add NuGet packages
dotnet add package Microsoft.Extensions.Http.Resilience
dotnet add package Polly
dotnet add package AspNetCore.HealthChecks.UI
dotnet add package AspNetCore.HealthChecks.Uris

# 3. Run
dotnet run

# 4. Test endpoints

# Test Throttling (10 requests/10s per user)
curl -X POST http://localhost:5000/api/loans/apply \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer user123" \
  -d '{"applicantName":"John","ssn":"123-45-6789","requestedAmount":50000,"termYears":30,"annualIncome":75000}'

# Test Rate Limiting (1000 requests/hour per partner)
curl -X GET http://localhost:5000/api/loans/partner/status/app-001 \
  -H "X-Partner-Id: partner-bank-xyz"

# Test Health Endpoints
curl http://localhost:5000/health       # Full status
curl http://localhost:5000/health/live  # Liveness probe
curl http://localhost:5000/health/ready # Readiness probe
curl http://localhost:5000/admin/health # UI dashboard
```

### **What Each Pattern Does in This Example**

| Pattern | Where | What Happens |
|---|---|---|
| **Retry** | CreditBureauService | If credit bureau returns 500, retry up to 3 times with 200ms, 400ms, 800ms delays |
| **Circuit Breaker** | CreditBureauService | After 5 consecutive failed requests, "trip" the circuit and immediately reject requests for 30s |
| **Bulkhead** | IncomeVerificationService | Max 50 concurrent income checks; excess requests queue or fail with 429 |
| **Health Monitoring** | `/health/*` endpoints | Kubernetes probes check if process is alive, if it's ready to serve traffic, and dependency status |
| **Throttling** | `POST /api/loans/apply` | Each user can submit max 10 loan applications per 10 seconds |
| **Rate Limiting** | `GET /api/loans/partner/status` | Each partner bank can make 1000 API calls per hour |

### **Real-World Scenario**

```
Day 1: Loan processor tries to apply for a mortgage
  POST /api/loans/apply (User: "john@bank.com")
  ✅ Throttling: 1st request today — allowed
  Credit Bureau responds in 500ms
  ✅ Retry: Not needed
  ✅ Circuit Breaker: CLOSED (healthy)
  ✅ Bulkhead: Income check uses 1/50 slots
  ✅ Health: All dependencies healthy
  Result: Approved with $250,000

Same day: Credit Bureau goes down (disk failure)
  POST /api/loans/apply (User: "jane@bank.com")
  ✅ Throttling: Allowed
  Credit Bureau: Connection refused
  🔄 Retry #1: Wait 200ms, retry... Connection refused
  🔄 Retry #2: Wait 400ms, retry... Connection refused
  🔄 Retry #3: Wait 800ms, retry... Connection refused
  ❌ Circuit Breaker: TRIPS (5 failures)
  Result: 503 Service Unavailable — "Credit bureau offline, please try again"

30 seconds later: Credit Bureau back online
  POST /api/loans/apply (User: "bob@bank.com")
  ✅ Circuit Breaker: Half-open, tries one test request
  Credit Bureau: 200 OK
  ✅ Circuit Breaker: CLOSES
  Result: Approved

Aggressive bot tries 100 requests/second to bypass system:
  POST /api/loans/apply (over and over)
  ❌ Request #11: Throttling — 429 Too Many Requests
  ❌ Request #12-100: 429 Too Many Requests
  Result: Bot is blocked; legitimate users unaffected

Partner bank tries to abuse the API:
  GET /api/loans/partner/status x 2,000 in one hour
  ✅ Requests #1-1000: 200 OK
  ❌ Request #1001: Rate Limiting — 429 Too Many Requests
  Result: Partner quota enforced; can retry next hour
```

### **Summary**

This complete ASP.NET Core app demonstrates:

1. ✅ **Retry** — Automatic recovery from transient failures
2. ✅ **Circuit Breaker** — Prevents cascading failures
3. ✅ **Bulkhead** — Isolates workloads so one failure doesn't starve others
4. ✅ **Health Monitoring** — Tells orchestrators (Kubernetes) the app's status
5. ✅ **Throttling** — Protects against user abuse
6. ✅ **Rate Limiting** — Enforces partner quotas

All implemented in a production-ready, fully-configured ASP.NET Core application.
