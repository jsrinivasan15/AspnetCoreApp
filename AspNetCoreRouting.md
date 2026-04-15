# Routing in ASP.NET Core Web API

---

## 1. What Is Routing? — How HTTP Requests Reach Controller Actions

**Routing** is the system that matches incoming HTTP requests to **endpoint handlers** (controller actions or minimal API delegates) based on the request URL and HTTP method.

### How It Works

```
HTTP Request: GET /api/products/42
                    │
                    ▼
┌──────────────────────────────────┐
│  Routing Middleware              │
│  Matches URL against registered  │
│  route templates                 │
└──────────────┬───────────────────┘
               │  Match found: ProductsController.GetById(int id = 42)
               ▼
┌──────────────────────────────────┐
│  Endpoint Middleware             │
│  Executes the matched action     │
└──────────────────────────────────┘
```

### Routing in the Middleware Pipeline

```csharp
var app = builder.Build();

app.UseRouting();          // 1. Matches the request to an endpoint
app.UseAuthentication();   // 2. Auth runs BETWEEN routing and endpoint
app.UseAuthorization();    // 3. Authorization checks
app.MapControllers();      // 4. Registers controller endpoints (executes matched endpoint)
```

> In .NET 6+, `UseRouting()` is called **implicitly** by the framework when you call `MapControllers()`. You only need to call it explicitly when inserting middleware between routing and endpoint execution.

### Key Concepts

| Concept | Explanation |
|---|---|
| **Route template** | A pattern like `"api/products/{id}"` that defines URL structure |
| **Route parameter** | A named placeholder (`{id}`) extracted from the URL |
| **Endpoint** | The controller action or delegate that handles the matched request |
| **Route matching** | The process of comparing a request URL to registered route templates |
| **HTTP method matching** | `[HttpGet]`, `[HttpPost]`, etc. constrain which verb an action responds to |

---

## 2. Types of Routing — Attribute Routing vs Conventional Routing

### Attribute Routing

Routes are defined **directly on controllers and actions** using attributes.

```csharp
[ApiController]
[Route("api/[controller]")]         // → /api/products
public class ProductsController : ControllerBase
{
    [HttpGet]                        // GET /api/products
    public IActionResult GetAll() => Ok(_products);

    [HttpGet("{id}")]                // GET /api/products/42
    public IActionResult GetById(int id) => Ok(_products.Find(id));

    [HttpPost]                       // POST /api/products
    public IActionResult Create(ProductDto dto) => Created(...);
}
```

### Conventional Routing

Routes are defined **centrally** in `Program.cs` using a route table. Common in MVC apps, **rarely used for Web APIs**.

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

This maps `GET /Products/GetById/42` → `ProductsController.GetById(42)`.

### Comparison

| Aspect | Attribute Routing | Conventional Routing |
|---|---|---|
| **Definition** | On controllers / actions | Centralized in `Program.cs` |
| **Flexibility** | Full control per action | Pattern-based, less granular |
| **RESTful URLs** | Easy (`api/products/{id}`) | Awkward (controller/action/id) |
| **Discoverability** | Route visible next to the code | Must check `Program.cs` |
| **Web API standard** | Yes — **recommended** | No — designed for MVC views |
| **`[ApiController]` support** | Required | Not compatible |
| **Overriding per action** | Easy | Requires separate routes |
| **Maintenance** | Scales well — route lives with code | Central file grows complex |

### When to Use Each

| Scenario | Recommendation |
|---|---|
| Web API (RESTful) | **Attribute routing** (always) |
| MVC with views | Conventional routing or attribute routing |
| Mixed API + MVC | Attribute routing for API, conventional for MVC pages |

> **Best Practice**: For Web APIs, **always use attribute routing**. The `[ApiController]` attribute requires it.

---

## 3. Defining Routes in Controllers and Actions

### Controller-Level Route (Base Route)

```csharp
[ApiController]
[Route("api/[controller]")]   // [controller] → "products" (strips "Controller" suffix)
public class ProductsController : ControllerBase
{
    // All actions inherit the base route: /api/products/...
}
```

### Action-Level Routes

```csharp
[ApiController]
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpGet]                          // GET /api/products
    public IActionResult GetAll() => Ok();

    [HttpGet("{id}")]                  // GET /api/products/5
    public IActionResult GetById(int id) => Ok();

    [HttpGet("featured")]              // GET /api/products/featured
    public IActionResult GetFeatured() => Ok();

    [HttpGet("category/{categoryId}")] // GET /api/products/category/3
    public IActionResult GetByCategory(int categoryId) => Ok();

    [HttpPost]                         // POST /api/products
    public IActionResult Create(ProductDto dto) => Created();

    [HttpPut("{id}")]                  // PUT /api/products/5
    public IActionResult Update(int id, ProductDto dto) => NoContent();

    [HttpPatch("{id}")]                // PATCH /api/products/5
    public IActionResult PartialUpdate(int id, JsonPatchDocument<ProductDto> patch) => NoContent();

    [HttpDelete("{id}")]               // DELETE /api/products/5
    public IActionResult Delete(int id) => NoContent();
}
```

### Token Replacement

| Token | Replaced With | Example |
|---|---|---|
| `[controller]` | Controller class name (without "Controller" suffix) | `ProductsController` → `products` |
| `[action]` | Action method name | `GetAll` → `getall` |
| `[area]` | Area name (if using areas) | `Admin` → `admin` |

```csharp
[Route("api/[controller]/[action]")]   // → /api/products/getall
```

> For RESTful APIs, avoid `[action]` in routes. Use HTTP verbs to differentiate actions.

### Overriding the Base Route

Prefix a route with `/` or `~/` to **ignore** the controller-level route.

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]                           // GET /api/products
    public IActionResult GetAll() => Ok();

    [HttpGet("/api/inventory")]         // GET /api/inventory (overrides base route)
    public IActionResult GetInventory() => Ok();

    [HttpGet("~/api/catalog")]          // GET /api/catalog (also overrides)
    public IActionResult GetCatalog() => Ok();
}
```

### Multiple Routes on a Single Action

```csharp
[HttpGet("{id}")]
[HttpGet("by-id/{id}")]               // Both /api/products/5 and /api/products/by-id/5 work
public IActionResult GetById(int id) => Ok();
```

### Named Routes (for URL Generation)

```csharp
[HttpGet("{id}", Name = "GetProductById")]
public IActionResult GetById(int id)
{
    var product = _service.GetById(id);
    return Ok(product);
}

[HttpPost]
public IActionResult Create(ProductDto dto)
{
    var product = _service.Create(dto);
    // Generates URL: /api/products/5
    return CreatedAtRoute("GetProductById", new { id = product.Id }, product);
}
```

---

## 4. Route Parameters and Constraints — Capture and Validate URL Data

### Basic Route Parameters

```csharp
[HttpGet("{id}")]                        // /api/products/42 → id = 42
public IActionResult GetById(int id) => Ok();

[HttpGet("{category}/{id}")]             // /api/products/electronics/42
public IActionResult Get(string category, int id) => Ok();
```

### Route Constraints

Constraints **restrict** which values a route parameter will match. If the value doesn't satisfy the constraint, the route doesn't match (returns 404, not 400).

```csharp
[HttpGet("{id:int}")]                    // Only matches integers
public IActionResult GetById(int id) => Ok();

[HttpGet("{slug:alpha}")]                // Only matches alphabetic strings
public IActionResult GetBySlug(string slug) => Ok();
```

### Built-in Constraints

| Constraint | Example | Matches | Does NOT Match |
|---|---|---|---|
| `int` | `{id:int}` | `42`, `-1` | `abc`, `3.14` |
| `long` | `{id:long}` | `9223372036854775807` | `abc` |
| `decimal` | `{price:decimal}` | `9.99`, `100` | `abc` |
| `float` | `{val:float}` | `3.14` | `abc` |
| `double` | `{val:double}` | `3.14159` | `abc` |
| `bool` | `{active:bool}` | `true`, `false` | `yes`, `42` |
| `guid` | `{id:guid}` | `CD2C1638-1638-72D5-1638-DEADBEEF1638` | `abc` |
| `datetime` | `{date:datetime}` | `2026-04-11` | `not-a-date` |
| `alpha` | `{name:alpha}` | `hello` | `hello123` |
| `minlength(n)` | `{name:minlength(3)}` | `abc` | `ab` |
| `maxlength(n)` | `{name:maxlength(10)}` | `short` | `toolongstring` |
| `length(n)` | `{code:length(5)}` | `AB123` | `AB12` |
| `length(m,n)` | `{code:length(3,5)}` | `ABC`, `ABCDE` | `AB` |
| `min(n)` | `{age:min(18)}` | `18`, `25` | `17` |
| `max(n)` | `{qty:max(100)}` | `50` | `101` |
| `range(m,n)` | `{page:range(1,100)}` | `1`, `50` | `0`, `101` |
| `regex(expr)` | `{code:regex(^[A-Z]{{3}}$)}` | `ABC` | `abc`, `AB` |
| `required` | `{name:required}` | `anything` | (empty) |

### Multiple Constraints

Chain constraints with `:`.

```csharp
[HttpGet("{id:int:min(1)}")]             // Must be an integer >= 1
public IActionResult GetById(int id) => Ok();

[HttpGet("{code:alpha:length(3,10)}")]   // Alphabetic, 3-10 chars
public IActionResult GetByCode(string code) => Ok();

[HttpGet("{page:int:range(1,100)}")]     // Integer between 1 and 100
public IActionResult GetPage(int page) => Ok();
```

### Catch-All Parameter

Captures the **remaining** URL path (including slashes).

```csharp
[HttpGet("files/{**path}")]              // /api/files/images/photos/cat.jpg → path = "images/photos/cat.jpg"
public IActionResult GetFile(string path) => Ok();
```

### Custom Route Constraint

```csharp
public class EvenNumberConstraint : IRouteConstraint
{
    public bool Match(HttpContext? httpContext, IRouter? route, string routeKey,
        RouteValueDictionary values, RouteDirection routeDirection)
    {
        if (values.TryGetValue(routeKey, out var value) && int.TryParse(value?.ToString(), out var number))
        {
            return number % 2 == 0;
        }
        return false;
    }
}
```

Register:

```csharp
builder.Services.AddRouting(options =>
{
    options.ConstraintMap.Add("even", typeof(EvenNumberConstraint));
});
```

Use:

```csharp
[HttpGet("{id:even}")]                   // Only matches even numbers
public IActionResult GetEven(int id) => Ok();
```

> **Important**: Route constraints validate **URL matching**, not input validation. For data validation, use model validation (`[Required]`, `FluentValidation`, etc.).

---

## 5. Default Values and Optional Parameters

### Optional Parameters

Add `?` to make a route parameter optional.

```csharp
[HttpGet("search/{category?}")]
public IActionResult Search(string? category)
{
    // GET /api/products/search          → category = null
    // GET /api/products/search/electronics → category = "electronics"

    if (category == null)
        return Ok(_products);

    return Ok(_products.Where(p => p.Category == category));
}
```

### Default Values

Assign a default value in the route template.

```csharp
[HttpGet("list/{page=1}")]
public IActionResult List(int page)
{
    // GET /api/products/list         → page = 1 (default)
    // GET /api/products/list/3       → page = 3
    return Ok(_products.Skip((page - 1) * 10).Take(10));
}
```

### Default + Constraint

```csharp
[HttpGet("list/{page:int=1}")]           // Default 1, must be int
public IActionResult List(int page) => Ok();
```

### C# Parameter Defaults (Alternative)

```csharp
[HttpGet("search")]
public IActionResult Search(
    [FromQuery] string? category = null,
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 10)
{
    // GET /api/products/search?category=electronics&page=2&pageSize=20
    // All parameters are optional with defaults
    return Ok();
}
```

### Optional vs Default — When to Use

| Feature | Syntax | Value When Absent | Best For |
|---|---|---|---|
| **Optional** | `{id?}` | `null` | When "no value" has distinct meaning |
| **Default** | `{page=1}` | The default value | When a sensible default exists |
| **Query defaults** | `int page = 1` | C# default | Query string parameters |

### Common Pattern: Paged List

```csharp
[HttpGet]
public IActionResult GetAll(
    [FromQuery] int page = 1,
    [FromQuery] int pageSize = 20,
    [FromQuery] string? sortBy = "name",
    [FromQuery] string? order = "asc")
{
    // GET /api/products
    // GET /api/products?page=2&pageSize=50&sortBy=price&order=desc
    return Ok();
}
```

---

## 6. Customizing Routes — Advanced Templates and Patterns

### Area-Based Routing

Group controllers by feature area.

```csharp
[ApiController]
[Area("Admin")]
[Route("api/[area]/[controller]")]       // → /api/admin/users
public class UsersController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() => Ok();
}
```

### API Versioning via Routes

```csharp
[ApiController]
[Route("api/v1/products")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() => Ok(new { version = 1, products = _list });
}

[ApiController]
[Route("api/v2/products")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() => Ok(new { version = 2, data = _list, total = _list.Count });
}
```

### Nested Resources

```csharp
[ApiController]
[Route("api/customers/{customerId}/orders")]
public class CustomerOrdersController : ControllerBase
{
    [HttpGet]                                    // GET /api/customers/5/orders
    public IActionResult GetOrders(int customerId) => Ok();

    [HttpGet("{orderId}")]                       // GET /api/customers/5/orders/10
    public IActionResult GetOrder(int customerId, int orderId) => Ok();

    [HttpPost]                                   // POST /api/customers/5/orders
    public IActionResult CreateOrder(int customerId, OrderDto dto) => Created();
}
```

### Constant Segments and Mixed Templates

```csharp
[Route("api/reports")]
public class ReportsController : ControllerBase
{
    [HttpGet("sales/monthly/{year:int}/{month:range(1,12)}")]
    // GET /api/reports/sales/monthly/2026/4
    public IActionResult MonthlySales(int year, int month) => Ok();

    [HttpGet("sales/annual/{year:int}")]
    // GET /api/reports/sales/annual/2026
    public IActionResult AnnualSales(int year) => Ok();

    [HttpGet("export/{format:regex(^(csv|pdf|xlsx)$)}")]
    // GET /api/reports/export/csv
    public IActionResult Export(string format) => Ok();
}
```

### Slug-Based Routes

```csharp
[HttpGet("{slug:alpha:minlength(3)}")]
// GET /api/products/wireless-mouse
public IActionResult GetBySlug(string slug) => Ok();
```

### Route Prefix for All Controllers

Use a convention to prefix every controller route.

```csharp
public class ApiRoutePrefixConvention : IApplicationModelConvention
{
    private readonly AttributeRouteModel _routePrefix;

    public ApiRoutePrefixConvention(string prefix)
    {
        _routePrefix = new AttributeRouteModel(new RouteAttribute(prefix));
    }

    public void Apply(ApplicationModel application)
    {
        foreach (var controller in application.Controllers)
        {
            foreach (var selector in controller.Selectors)
            {
                selector.AttributeRouteModel = selector.AttributeRouteModel != null
                    ? AttributeRouteModel.CombineAttributeRouteModel(_routePrefix, selector.AttributeRouteModel)
                    : _routePrefix;
            }
        }
    }
}
```

```csharp
builder.Services.AddControllers(options =>
{
    options.Conventions.Add(new ApiRoutePrefixConvention("api/v1"));
});
```

Now `[Route("[controller]")]` becomes `/api/v1/products`.

### Route Transformation (Kebab-Case URLs)

```csharp
builder.Services.AddRouting(options =>
{
    options.LowercaseUrls = true;
    options.LowercaseQueryStrings = true;
});
```

For kebab-case (`product-categories` instead of `productcategories`):

```csharp
public class KebabCaseParameterTransformer : IOutboundParameterTransformer
{
    public string? TransformOutbound(object? value)
    {
        return value?.ToString() != null
            ? Regex.Replace(value.ToString()!, "([a-z])([A-Z])", "$1-$2").ToLower()
            : null;
    }
}
```

```csharp
builder.Services.AddControllers(options =>
{
    options.Conventions.Add(new RouteTokenTransformerConvention(new KebabCaseParameterTransformer()));
});
```

`ProductCategoriesController` → `/api/product-categories`

---

## 7. Route Precedence and Ordering — How ASP.NET Core Selects the Correct Route

### How Route Matching Works

When multiple routes could match a request, ASP.NET Core uses a **precedence algorithm** to select the best match.

### Precedence Rules (Highest to Lowest)

| Priority | Pattern Type | Example | Explanation |
|---|---|---|---|
| 1 | **Literal segment** | `products/featured` | Exact text match |
| 2 | **Constrained parameter** | `{id:int}` | Parameter with constraint |
| 3 | **Unconstrained parameter** | `{id}` | Any value matches |
| 4 | **Optional / Default** | `{id?}`, `{page=1}` | May not be present |
| 5 | **Catch-all** | `{**path}` | Matches everything remaining |

### Example: Precedence in Action

```csharp
[Route("api/products")]
public class ProductsController : ControllerBase
{
    [HttpGet("featured")]              // 1. Literal → highest priority
    public IActionResult GetFeatured() => Ok("featured");

    [HttpGet("{id:int}")]              // 2. Constrained parameter
    public IActionResult GetById(int id) => Ok($"by id: {id}");

    [HttpGet("{slug}")]                // 3. Unconstrained parameter
    public IActionResult GetBySlug(string slug) => Ok($"by slug: {slug}");

    [HttpGet("{**path}")]              // 4. Catch-all → lowest priority
    public IActionResult CatchAll(string path) => Ok($"catch-all: {path}");
}
```

Request resolution:

```
GET /api/products/featured     → GetFeatured()     (literal match wins)
GET /api/products/42           → GetById(42)        (int constraint matches)
GET /api/products/widget       → GetBySlug("widget") (unconstrained matches)
GET /api/products/a/b/c        → CatchAll("a/b/c")  (catch-all matches)
```

### Ambiguous Routes — Error

If two routes have **equal** precedence and both match, ASP.NET Core throws an `AmbiguousMatchException`.

```csharp
// ❌ AMBIGUOUS — both are unconstrained parameters at the same position
[HttpGet("{id}")]
public IActionResult GetById(string id) => Ok();

[HttpGet("{slug}")]
public IActionResult GetBySlug(string slug) => Ok();
// AmbiguousMatchException at runtime!
```

**Fix**: Add constraints to differentiate.

```csharp
// ✅ No ambiguity — int constraint differentiates
[HttpGet("{id:int}")]
public IActionResult GetById(int id) => Ok();

[HttpGet("{slug:alpha}")]
public IActionResult GetBySlug(string slug) => Ok();
```

### HTTP Method Differentiation

Routes with the same template but different HTTP methods are **not ambiguous**.

```csharp
[HttpGet("{id}")]              // GET /api/products/5
public IActionResult Get(int id) => Ok();

[HttpDelete("{id}")]           // DELETE /api/products/5
public IActionResult Delete(int id) => NoContent();
// No conflict — different HTTP methods
```

### Route Order Attribute

Use `[Order]` to explicitly prioritize when automatic ordering isn't sufficient.

```csharp
builder.Services.AddControllers(options =>
{
    options.Conventions.Add(new RouteTokenTransformerConvention(new KebabCaseParameterTransformer()));
});
```

### Segment-by-Segment Comparison

ASP.NET Core compares routes **segment by segment** from left to right.

```
Template A: api/products/{id:int}
Template B: api/products/{name}
Template C: api/{controller}/{action}

For request: GET /api/products/42

Segment 1: "api" — all match (literal)
Segment 2: "products" — A and B match (literal), C matches (parameter)
            → A and B win (literal > parameter)
Segment 3: "42" — A matches (int constraint), B matches (unconstrained)
            → A wins (constrained > unconstrained)

Result: Template A selected
```

### Best Practices for Avoiding Conflicts

| Practice | Example |
|---|---|
| Use constraints to differentiate | `{id:int}` vs `{slug:alpha}` |
| Use literal segments for fixed routes | `products/featured` instead of `{type}` |
| Keep catch-all routes on separate controllers | Avoid mixing `{**path}` with specific routes |
| Use consistent URL patterns | Don't mix `/api/getProducts` with `/api/products` |
| Test with conflicting values | Ensure `GET /api/products/featured` doesn't match `{id}` |

### Debugging Route Issues

```csharp
// Log all registered endpoints at startup
app.MapControllers();

var endpoints = app as IEndpointRouteBuilder;
foreach (var endpoint in endpoints.DataSources.SelectMany(ds => ds.Endpoints))
{
    Console.WriteLine($"{endpoint.DisplayName} → {(endpoint as RouteEndpoint)?.RoutePattern}");
}
```

Or use the **Routing Analyzer** middleware in development:

```csharp
if (app.Environment.IsDevelopment())
{
    app.Use(async (context, next) =>
    {
        var endpoint = context.GetEndpoint();
        Console.WriteLine($"Matched: {endpoint?.DisplayName}");
        await next(context);
    });
}
```

---

## Quick Reference Cheat Sheet

| Topic | Key Takeaway |
|---|---|
| What is routing | Maps HTTP requests (URL + method) to controller actions |
| Attribute vs Conventional | Web APIs always use attribute routing; conventional is for MVC views |
| Route templates | `[Route("api/[controller]")]` + `[HttpGet("{id}")]` on actions |
| Parameters | `{id}` captures URL segments; bind to method parameters by name |
| Constraints | `{id:int}`, `{page:range(1,100)}`, `{code:regex(...)}` — for route matching |
| Optional / Default | `{id?}` for optional; `{page=1}` for default values |
| Customization | Nested resources, versioning, kebab-case transformer, global prefix |
| Precedence | Literal > Constrained > Unconstrained > Optional > Catch-all |
| Ambiguity | Same-precedence routes = `AmbiguousMatchException`; fix with constraints |
