# Introduction and Environment Setup for ASP.NET Core Web API

---

## 1. What Is a Web API? — REST, SOAP, and GraphQL

A **Web API** (Application Programming Interface) is a set of HTTP endpoints that expose application functionality to external clients — browsers, mobile apps, other services, or IoT devices.

### API Styles Comparison

| Aspect                | REST                                  | SOAP                          | GraphQL                       |
|---                    |---                                    |---                            |---                            |
| **Protocol**          | HTTP (any verb)                       | HTTP, SMTP, TCP (XML-based)   | HTTP (POST usually)           |
| **Data Format**       | JSON (common), XML                    | XML only (SOAP envelope)      | JSON                          |
| **Contract**          | OpenAPI / Swagger (optional)          | WSDL (strict, mandatory)      | Schema (mandatory)            |
| **Versioning**        | URL, header, or query string          | Namespace-based               | Schema evolution              |
| **Caching**           | Built-in HTTP caching | Manual        | Client-side                   |
| **Over-fetching**     | Common (fixed response shapes)        | Common                        | Solved — client selects fields|
| **Under-fetching**    | Requires multiple calls               | Requires multiple calls       | Solved — single query         |
| **Error Handling**    | HTTP status codes | SOAP fault XML    | `errors` array in response    |
| **Learning Curve**    | Low                                   | High                          | Medium                        |
| **Best For**          | Public APIs, microservices, CRUD      | Enterprise, WS-* standards    | Complex UIs, mobile apps      |

### REST (Representational State Transfer)

```
GET    /api/products        → Get all products
GET    /api/products/42     → Get product by ID
POST   /api/products        → Create a product
PUT    /api/products/42     → Replace a product
PATCH  /api/products/42     → Partially update a product
DELETE /api/products/42     → Delete a product
```

**Principles**: Stateless, resource-based URLs, standard HTTP verbs, content negotiation, HATEOAS.

### SOAP (Simple Object Access Protocol)

```xml
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
  <soap:Body>
    <GetProduct xmlns="http://example.com/products">
      <ProductId>42</ProductId>
    </GetProduct>
  </soap:Body>
</soap:Envelope>
```

**Characteristics**: Strict contract (WSDL), built-in WS-Security, WS-ReliableMessaging. Used in banking, healthcare, and legacy enterprise systems.

### GraphQL

```graphql
query {
  product(id: 42) {
    name
    price
    category {
      name
    }
  }
}
```

**Characteristics**: Single endpoint, client specifies exactly what data it needs, schema-first, eliminates over-fetching.

> **ASP.NET Core** is primarily used for **REST APIs** and can also serve GraphQL (via HotChocolate or GraphQL.NET).

---

## 2. Why ASP.NET Core for Web APIs? — Benefits and Use Cases

### Key Benefits

| Benefit                       | Explanation                                                           |
|---                            |---                                                                    |
| **Cross-platform**            | Runs on Windows, Linux, macOS                                         |
| **High performance**          | Kestrel is one of the fastest web servers (TechEmpower benchmarks)    |
| **Open source**               | MIT license, community-driven on GitHub                               | 
| **Unified framework**         | One framework for Web API, MVC, Razor Pages, Blazor, gRPC, SignalR    |
| **Built-in DI**               | First-class dependency injection out of the box                       |
| **Minimal APIs**              | Lightweight endpoint syntax for simple services                       |
| **Configuration system**      | JSON, environment variables, secrets, Azure Key Vault                 |
| **Middleware pipeline**       | Composable request/response processing                                |
| **Security**                  | Built-in authentication, authorization, HTTPS, data protection        |
| **Cloud-native**              | Docker, Kubernetes, Azure App Service, AWS Lambda support             |
| **Long-term support**         | .NET 8 LTS (until Nov 2026), .NET 9 STS                               |

### Common Use Cases

- **RESTful APIs** for web/mobile front-ends (SPA, React, Angular, Flutter)
- **Microservices** communicating via HTTP/gRPC
- **Backend-for-Frontend (BFF)** pattern
- **Real-time APIs** with SignalR (chat, notifications, dashboards)
- **Serverless functions** (Azure Functions, AWS Lambda)
- **IoT backends** processing device telemetry
- **Enterprise line-of-business** APIs

### ASP.NET Core vs ASP.NET Framework (Legacy)

| Aspect                | ASP.NET Core                          | ASP.NET Framework               |
|---                    |---                                    |---                              |
| Platform              | Cross-platform                        | Windows only                    |
| Hosting               | Kestrel + any reverse proxy           | IIS only                        |
| Performance           | Significantly faster                  | Moderate                        |
| Deployment            | Self-contained or framework-dependent | IIS-dependent                   |
| Side-by-side versions | Yes                                   | No (machine-wide install)       |
| Active development    | Yes                                   | Maintenance mode only           |

---

## 3. ASP.NET Core Web API Architecture — Scalable and Performant

### High-Level Architecture

```
Client (Browser / Mobile / Service)
  │
  │  HTTP Request
  ▼
┌──────────────────────────────────┐
│  Reverse Proxy (IIS / Nginx)     │  ← Optional, handles TLS, load balancing
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────┐
│  Kestrel (Built-in Web Server)   │  ← High-performance, cross-platform
└──────────────┬───────────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────┐
│  Middleware Pipeline                                     │
│  ┌─────────────┐  ┌──────────┐  ┌──────────────────┐     │
│  │ Exception   │→ │ Auth     │→ │ CORS / Routing   │     │
│  │ Handler     │  │          │  │                  │     │
│  └─────────────┘  └──────────┘  └──────────────────┘     │
└──────────────────────────┬───────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────┐
│  MVC / Endpoint Routing                                  │
│  ┌───────────┐  ┌─────────────┐  ┌─────────────────┐     │
│  │ Model     │→ │ Controller  │→ │ Action Filters  │     │
│  │ Binding   │  │ Action      │  │ Result Filters  │     │
│  └───────────┘  └─────────────┘  └─────────────────┘     │
└──────────────────────────┬───────────────────────────────┘
                           │
                           ▼
┌──────────────────────────────────────────────────────────┐
│  Service Layer (Business Logic)                          │
│  └→ Repository / Data Access (EF Core, Dapper, etc.)     │
│       └→ Database (SQL Server, PostgreSQL, MongoDB)      │
└──────────────────────────────────────────────────────────┘
```

### Key Architectural Components

| Component                 | Role |
|---                        |---                                                              |
| **Kestrel**               | Built-in HTTP server — handles connections, TLS, HTTP/2, HTTP/3 |
| **Middleware**            | Composable pipeline — auth, CORS, logging, compression          |
| **Routing**               | Maps URLs to controllers/endpoints                              |
| **Model Binding**         | Converts request data (JSON, query, route) to C# objects        |
| **Controllers**           | Handle requests, call services, return responses                |
| **Dependency Injection**  | Wires up services at startup (built-in IoC container)           |
| **Configuration**         | `appsettings.json`, environment variables, user secrets         |
| **Logging**               | `ILogger<T>` abstraction with pluggable providers               |

### Request Lifecycle

```
1. Kestrel receives HTTP request
2. Middleware pipeline processes (logging, auth, CORS, etc.)
3. Routing matches a controller action
4. Model binding converts request data to action parameters
5. Filters run (Authorization → Resource → Action)
6. Controller action executes → calls services → returns IActionResult
7. Result filters run → response is serialized (JSON)
8. Response flows back through middleware (in reverse)
9. Kestrel sends HTTP response to client
```

---

## 4. Installing .NET SDK and IDEs — Visual Studio and VS Code

### Step 1: Install .NET SDK

Download from [https://dotnet.microsoft.com/download](https://dotnet.microsoft.com/download)

Verify installation:

```bash
dotnet --version
# e.g., 8.0.303

dotnet --list-sdks
# 8.0.303 [C:\Program Files\dotnet\sdk]
```

### Step 2: Choose an IDE

| IDE | Best For | Cost |
|---|---|---|
| **Visual Studio 2022** | Full-featured .NET development, debugging, profiling | Free (Community) / Paid |
| **VS Code** | Lightweight, cross-platform, extensions | Free |
| **JetBrains Rider** | Cross-platform, powerful refactoring | Paid |

### Visual Studio 2022 Setup

1. Download Visual Studio Community (free).
2. In the installer, select the **"ASP.NET and web development"** workload.
3. Ensure **.NET 8 SDK** is checked under Individual Components.

### VS Code Setup

1. Install VS Code.
2. Install extensions:
   - **C# Dev Kit** (Microsoft) — IntelliSense, debugging, project system
   - **REST Client** — send HTTP requests from `.http` files
   - **.NET Install Tool** — manage SDK versions

### Step 3: Create Your First API

```bash
# Create a new Web API project
dotnet new webapi -n MyFirstApi -controllers

# Navigate to the project
cd MyFirstApi

# Run the application
dotnet run
```

Browse to `https://localhost:5001/swagger` to see the Swagger UI.

### Useful CLI Commands

| Command | Purpose |
|---|---|
| `dotnet new webapi -n Name` | Create a new Web API project |
| `dotnet new webapi -minimal` | Create with Minimal API style |
| `dotnet run` | Build and run the project |
| `dotnet watch run` | Run with hot reload |
| `dotnet build` | Build without running |
| `dotnet publish -c Release` | Publish for deployment |
| `dotnet add package Serilog` | Add a NuGet package |
| `dotnet ef migrations add Init` | Create EF Core migration |

---

## 5. Project Folder and File Structure — Roles of Each File

### Default Web API Project Structure

```
MyFirstApi/
├── Controllers/
│   └── WeatherForecastController.cs   ← API controller with endpoints
├── Properties/
│   └── launchSettings.json            ← Dev launch profiles (ports, env)
├── appsettings.json                   ← App configuration (all environments)
├── appsettings.Development.json       ← Dev-specific config overrides
├── MyFirstApi.csproj                  ← Project file (packages, SDK, target framework)
├── MyFirstApi.http                    ← HTTP request file for testing
├── Program.cs                         ← Application entry point and startup config
└── WeatherForecast.cs                 ← Model / DTO class
```

### File-by-File Breakdown

| File / Folder | Purpose |
|---|---|
| **`Program.cs`** | Entry point — configures services, middleware pipeline, and runs the app |
| **`Controllers/`** | API controllers — each maps to a route (e.g., `/api/weatherforecast`) |
| **`appsettings.json`** | App config — connection strings, logging, feature flags |
| **`appsettings.{Env}.json`** | Environment-specific overrides (Development, Staging, Production) |
| **`launchSettings.json`** | Local dev settings — ports, environment variables, launch profiles |
| **`*.csproj`** | MSBuild project file — target framework, NuGet packages, build settings |
| **`*.http`** | HTTP request file — test endpoints from VS Code / Visual Studio |
| **`Properties/`** | Contains `launchSettings.json` |

### `.csproj` File

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
  </ItemGroup>
</Project>
```

### Expanded Real-World Structure

```
MyApi/
├── Controllers/           ← API endpoints
├── Models/                ← Domain entities
├── DTOs/                  ← Data Transfer Objects (request/response shapes)
├── Services/              ← Business logic interfaces and implementations
├── Repositories/          ← Data access layer
├── Middleware/            ← Custom middleware classes
├── Filters/               ← Action/exception filters
├── Extensions/            ← Service registration extension methods
├── Configurations/        ← Strongly-typed options classes
├── Migrations/            ← EF Core database migrations
├── Properties/
│   └── launchSettings.json
├── appsettings.json
├── appsettings.Development.json
├── appsettings.Production.json
├── Program.cs
└── MyApi.csproj
```

---

## 6. Importance of Program.cs — Application Startup Configuration

`Program.cs` is the **single entry point** for the application. It configures services (DI container) and the HTTP middleware pipeline.

### Minimal `Program.cs` (Default Template)

```csharp
var builder = WebApplication.CreateBuilder(args);

// ──────────────── SERVICE REGISTRATION ────────────────
// Add services to the DI container
builder.Services.AddControllers();
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

var app = builder.Build();

// ──────────────── MIDDLEWARE PIPELINE ────────────────
// Configure the HTTP request pipeline
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}

app.UseHttpsRedirection();
app.UseAuthorization();
app.MapControllers();

app.Run();
```

### Two Phases

| Phase | Purpose | Code Section |
|---|---|---|
| **1. Service Registration** | Register dependencies in the DI container | `builder.Services.Add...()` |
| **2. Middleware Pipeline** | Configure request/response processing order | `app.Use...()` / `app.Map...()` |

### Real-World `Program.cs`

```csharp
var builder = WebApplication.CreateBuilder(args);

// ── Services ──
builder.Services.AddControllers(options =>
{
    options.Filters.Add<ValidationFilter>();
});

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Database
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

// Custom services
builder.Services.AddScoped<IProductService, ProductService>();
builder.Services.AddScoped<IOrderService, OrderService>();

// Authentication
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidIssuer = builder.Configuration["Jwt:Issuer"],
            ValidateAudience = true,
            ValidAudience = builder.Configuration["Jwt:Audience"],
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(builder.Configuration["Jwt:Key"]!))
        };
    });

// CORS
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
        policy.WithOrigins("https://myapp.com")
              .AllowAnyMethod()
              .AllowAnyHeader());
});

// Caching
builder.Services.AddMemoryCache();
builder.Services.AddResponseCaching();

// Logging (Serilog)
builder.Host.UseSerilog((context, config) =>
    config.ReadFrom.Configuration(context.Configuration));

var app = builder.Build();

// ── Middleware Pipeline (ORDER MATTERS) ──
if (app.Environment.IsDevelopment())
{
    app.UseDeveloperExceptionPage();
    app.UseSwagger();
    app.UseSwaggerUI();
}
else
{
    app.UseExceptionHandler("/error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseCors("AllowFrontend");
app.UseAuthentication();
app.UseAuthorization();
app.UseResponseCaching();
app.UseSerilogRequestLogging();
app.MapControllers();

app.Run();
```

### `WebApplication.CreateBuilder` — What It Does

| Responsibility | Details |
|---|---|
| **Configuration** | Loads `appsettings.json`, environment variables, user secrets, command-line args |
| **Logging** | Adds Console, Debug, EventSource providers |
| **DI Container** | Creates the built-in service container |
| **Kestrel** | Configures the HTTP server |
| **Environment** | Reads `ASPNETCORE_ENVIRONMENT` (Development, Staging, Production) |

---

## 7. Running and Testing the API — Postman, Swagger, and Fiddler

### Running the Application

```bash
# Standard run
dotnet run

# With hot reload (auto-restart on file changes)
dotnet watch run

# Specify environment
dotnet run --environment Development

# Specify URL
dotnet run --urls "https://localhost:5001;http://localhost:5000"
```

### Tool 1: Swagger / Swashbuckle (Built-in)

Swagger provides an **interactive UI** to explore and test your API directly in the browser.

```csharp
// Program.cs
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title = "My API",
        Version = "v1",
        Description = "Product management API"
    });
});

if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

Browse to: `https://localhost:5001/swagger`

### Tool 2: Postman

| Feature | Use |
|---|---|
| **Collections** | Group related API requests |
| **Environments** | Switch between dev/staging/prod URLs |
| **Variables** | Store tokens, IDs for reuse |
| **Tests** | Write JavaScript assertions on responses |
| **Pre-request scripts** | Auto-generate tokens before each request |

Example request:
```
POST https://localhost:5001/api/products
Headers:
  Content-Type: application/json
  Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
Body:
{
  "name": "Widget",
  "price": 9.99
}
```

### Tool 3: `.http` Files (VS Code / Visual Studio)

Built into the IDE — no external tool needed.

```http
### Get all products
GET https://localhost:5001/api/products
Accept: application/json

### Create a product
POST https://localhost:5001/api/products
Content-Type: application/json
Authorization: Bearer {{token}}

{
  "name": "Widget",
  "price": 9.99
}

### Get by ID
GET https://localhost:5001/api/products/1
```

### Tool 4: Fiddler

| Feature | Use |
|---|---|
| **Traffic capture** | Inspect all HTTP/HTTPS traffic |
| **Breakpoints** | Pause and modify requests/responses |
| **Auto-responder** | Mock API responses |
| **Composer** | Build and send custom requests |

Best for debugging proxy issues, inspecting headers, and diagnosing TLS problems.

### Tool 5: `curl` (Command Line)

```bash
# GET
curl -X GET https://localhost:5001/api/products

# POST with JSON body
curl -X POST https://localhost:5001/api/products \
  -H "Content-Type: application/json" \
  -d '{"name": "Widget", "price": 9.99}'

# With Bearer token
curl -H "Authorization: Bearer <token>" https://localhost:5001/api/products
```

### Tool Comparison

| Tool | Type | Best For |
|---|---|---|
| **Swagger UI** | Browser-based | Quick testing during development |
| **Postman** | Desktop/Web app | Full-featured testing, collections, CI/CD |
| **`.http` files** | IDE-integrated | Version-controlled requests, quick tests |
| **Fiddler** | Traffic inspector | Debugging HTTP traffic, proxying |
| **curl** | CLI | Scripts, CI pipelines |

---

## 8. Basic Configuration of `launchSettings.json`

`launchSettings.json` lives in `Properties/` and controls **local development** behavior. It is **not deployed** to production.

### Default `launchSettings.json`

```json
{
  "$schema": "https://json.schemastore.org/launchsettings.json",
  "profiles": {
    "http": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "https": {
      "commandName": "Project",
      "dotnetRunMessages": true,
      "launchBrowser": true,
      "launchUrl": "swagger",
      "applicationUrl": "https://localhost:5001;http://localhost:5000",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "IIS Express": {
      "commandName": "IISExpress",
      "launchBrowser": true,
      "launchUrl": "swagger",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    }
  },
  "iisSettings": {
    "windowsAuthentication": false,
    "anonymousAuthentication": true,
    "iisExpress": {
      "applicationUrl": "http://localhost:12345",
      "sslPort": 44300
    }
  }
}
```

### Key Properties

| Property | Purpose | Example |
|---|---|---|
| `commandName` | How the app is launched | `"Project"`, `"IISExpress"`, `"Executable"` |
| `applicationUrl` | URLs the server listens on | `"https://localhost:5001;http://localhost:5000"` |
| `launchBrowser` | Open browser on start | `true` / `false` |
| `launchUrl` | Default page to open | `"swagger"` |
| `environmentVariables` | Env vars for the process | `"ASPNETCORE_ENVIRONMENT": "Development"` |
| `dotnetRunMessages` | Show `dotnet run` output | `true` / `false` |

### Multiple Environment Profiles

```json
{
  "profiles": {
    "Development": {
      "commandName": "Project",
      "applicationUrl": "https://localhost:5001",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Development"
      }
    },
    "Staging": {
      "commandName": "Project",
      "applicationUrl": "https://localhost:5002",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Staging",
        "ConnectionStrings__Default": "Server=staging-db;..."
      }
    },
    "Production": {
      "commandName": "Project",
      "applicationUrl": "https://localhost:5003",
      "environmentVariables": {
        "ASPNETCORE_ENVIRONMENT": "Production"
      }
    }
  }
}
```

Run a specific profile:

```bash
dotnet run --launch-profile Staging
```

### How Configuration Loads (Priority Order)

```
1. appsettings.json                    (lowest priority)
2. appsettings.{Environment}.json
3. User Secrets (Development only)
4. Environment variables
5. Command-line arguments              (highest priority — wins)
```

Later sources **override** earlier ones for the same key.

---

## 9. Overview of Hosting Models — In-Process vs Out-of-Process

### What Is Hosting?

The **hosting model** determines how ASP.NET Core runs relative to the web server (IIS, Nginx, etc.).

### In-Process Hosting

The ASP.NET Core app runs **inside the IIS worker process** (`w3wp.exe`).

```
Client → IIS (w3wp.exe) → ASP.NET Core Module → App Code
         └── Same process ──────────────────────┘
```

```xml
<!-- .csproj -->
<PropertyGroup>
  <AspNetCoreHostingModel>InProcess</AspNetCoreHostingModel>
</PropertyGroup>
```

### Out-of-Process Hosting

The ASP.NET Core app runs in its **own Kestrel process**, and IIS/Nginx acts as a **reverse proxy**.

```
Client → IIS / Nginx (reverse proxy) → HTTP → Kestrel → App Code
         └── Process 1 ──────────────┘       └── Process 2 ────┘
```

```xml
<!-- .csproj -->
<PropertyGroup>
  <AspNetCoreHostingModel>OutOfProcess</AspNetCoreHostingModel>
</PropertyGroup>
```

### Comparison

| Aspect | In-Process | Out-of-Process |
|---|---|---|
| **Performance** | Faster (no inter-process communication) | Slightly slower (HTTP proxy hop) |
| **Server** | IIS HTTP Server (`IISHttpServer`) | Kestrel |
| **Process** | Inside `w3wp.exe` | Separate `dotnet.exe` process |
| **Platform** | Windows + IIS only | Cross-platform |
| **Reliability** | Tied to IIS process lifecycle | Independent process, can self-restart |
| **Debugging** | Slightly more complex | Standard Kestrel debugging |
| **Default (.NET 8)** | Yes (when hosted in IIS) | Used with Nginx, Docker, Kestrel-only |

### Kestrel-Only (Self-Hosted)

For containerized or cross-platform deployments, Kestrel runs directly without a reverse proxy.

```
Client → Kestrel → ASP.NET Core App
```

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.WebHost.ConfigureKestrel(options =>
{
    options.ListenAnyIP(5000);                              // HTTP
    options.ListenAnyIP(5001, listenOptions =>
    {
        listenOptions.UseHttps("cert.pfx", "password");     // HTTPS
    });
});
```

### Kestrel + Reverse Proxy (Recommended for Production)

```
Internet → Nginx / IIS / Apache → Kestrel → App
```

**Why use a reverse proxy?**

| Benefit | Explanation |
|---|---|
| TLS termination | Reverse proxy handles HTTPS certificates |
| Load balancing | Distribute traffic across multiple Kestrel instances |
| Static file serving | Nginx serves static files directly |
| Request filtering | Block malicious requests before they reach the app |
| Process management | IIS/systemd restarts the app on crash |

### Nginx Reverse Proxy Example

```nginx
server {
    listen 80;
    server_name api.myapp.com;

    location / {
        proxy_pass http://localhost:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Hosting Model Decision Guide

| Scenario | Recommended |
|---|---|
| Windows + IIS | In-Process (best performance) |
| Linux + Nginx | Out-of-Process (Kestrel + reverse proxy) |
| Docker / Kubernetes | Kestrel-only or with Nginx sidecar |
| Azure App Service (Windows) | In-Process (default) |
| Azure App Service (Linux) | Kestrel-only |
| Development | Kestrel-only (`dotnet run`) |

---

## Quick Reference Cheat Sheet

| Topic | Key Takeaway |
|---|---|
| Web API | HTTP endpoints exposing app functionality; REST is the most common style |
| REST vs SOAP vs GraphQL | REST = simple/HTTP-based; SOAP = strict/XML; GraphQL = flexible queries |
| Why ASP.NET Core | Cross-platform, high-perf, built-in DI, unified framework |
| Architecture | Kestrel → Middleware → Routing → Controller → Service → Database |
| SDK & IDE | `dotnet new webapi`; Visual Studio or VS Code + C# Dev Kit |
| Project structure | `Program.cs` (entry), Controllers, appsettings, .csproj |
| Program.cs | Two phases: service registration → middleware pipeline |
| Testing tools | Swagger (browser), Postman (desktop), `.http` files (IDE), Fiddler (traffic) |
| launchSettings.json | Local dev profiles — ports, env vars, launch URL (not deployed) |
| Hosting | In-Process (IIS, fastest); Out-of-Process (Kestrel + proxy, cross-platform) |
