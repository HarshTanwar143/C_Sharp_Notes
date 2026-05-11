## 1. Simple Explanation — What is Routing?

Routing is the mechanism that **maps an incoming HTTP request URL to a specific piece of code** (a controller action method) that should handle it.

When a browser or client sends:

```
GET https://myapp.com/products/42
```

Something inside ASP.NET Core has to figure out:

- Which **controller** should handle this?
- Which **method** inside that controller?
- What does `42` mean — is it a parameter?

That "something" is the **routing system**.

---

## 2. Real-World Analogy

Think of a **large hospital**.

When a patient walks in, the **reception desk** looks at their complaint and decides:

- _Chest pain?_ → Cardiology, Room 3, Dr. Ahmed
- _Broken leg?_ → Orthopedics, Room 7, Dr. Khan
- _Fever?_ → General Medicine, Room 1, Dr. Sharma

The **routing system is that reception desk**.

The **URL is the patient's complaint**. The **controller is the department**. The **action method is the specific doctor**.

Without routing, every request would land at the same place and nobody would know what to do with it.

---

## 3. Technical Deep Dive — How Routing Actually Works Internally

### The Big Picture — Request Journey

```
HTTP Request
     ↓
Kestrel (Web Server) — receives raw TCP/HTTP
     ↓
ASP.NET Core Middleware Pipeline
     ↓
[Routing Middleware] — UseRouting()
     ↓
[Endpoint Selection] — matches URL to an endpoint
     ↓
[Authorization Middleware] — UseAuthorization()
     ↓
[Endpoint Execution] — UseEndpoints() / MapControllers()
     ↓
Controller → Action Method → Response
```

This is **critically important** and most junior devs don't know this:

> Routing in ASP.NET Core 3.0+ is **split into two separate middleware steps**:
> 
> 1. **`UseRouting()`** — _Figures out_ which endpoint matches
> 2. **`UseEndpoints()`** — _Executes_ that endpoint

**Why split into two steps?**

Because middleware in the middle (like `UseAuthorization()`) needs to **know which endpoint was matched** before deciding whether to allow access. If routing and execution were one step, authorization middleware couldn't inspect the matched endpoint's metadata (like `[Authorize]` attribute).

---

### 4. Internal Working — What Happens Inside the Runtime

When your app starts (not when a request comes in — **at startup**):

```
App Startup
    ↓
All controllers are scanned via Reflection
    ↓
All [Route] attributes, [HttpGet], [HttpPost] are read
    ↓
A Route Table (called EndpointDataSource) is built in memory
    ↓
A DFA (Deterministic Finite Automaton) tree is constructed
    ↓
This tree is used to match URLs at runtime — O(log n) performance
```

This is why **routing is fast** — the heavy lifting (reflection, tree building) happens **once at startup**, not on every request.

At runtime for each request:

```
Request: GET /products/42
    ↓
RouteConstraintMatcher checks segments
    ↓
DFA traverses the tree
    ↓
Finds: ProductsController.GetById(int id)
    ↓
Populates RouteData dictionary: { "id": "42" }
    ↓
Model Binding converts "42" string → int 42
    ↓
Action method is invoked
```

---

## 5. Two Types of Routing — Convention vs Attribute

### Type 1 — Conventional Routing (MVC Pattern)

```csharp
// Program.cs
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

**Explanation of every part:**

|Part|Meaning|
|---|---|
|`{controller=Home}`|URL segment maps to controller name. Default is `Home`|
|`{action=Index}`|URL segment maps to action method name. Default is `Index`|
|`{id?}`|Optional segment. The `?` means it may or may not be present|
|`name: "default"`|Named route — used for URL generation (e.g., `Url.RouteUrl("default", ...)`)|

**Example URLs this matches:**

```
/                         → HomeController.Index()
/products                 → ProductsController.Index()
/products/details         → ProductsController.Details()
/products/details/42      → ProductsController.Details(id: 42)
```

**Real production controller using conventional routing:**

```csharp
// ProductsController.cs
public class ProductsController : Controller  // inherits Controller, not ControllerBase
{
    private readonly IProductService _productService;

    // WHY private? — No external code should access this field
    // WHY readonly? — After constructor runs, this reference must never change
    //                  Compiler enforces this. Prevents accidental reassignment.
    //                  Thread-safe because the reference itself is immutable.
    // WHY IProductService (interface)? — Decoupled from implementation.
    //                  Unit tests can inject MockProductService.
    //                  Real app injects EF-backed ProductService.
    // WHY underscore? — Convention to distinguish instance fields from local variables.
    //                    C# doesn't have 'this' requirement but underscore makes it explicit.
    private readonly IProductService _productService;

    // WHY constructor injection? — DI container calls this constructor.
    //   You don't new() up IProductService. The container owns the lifetime.
    //   If you did: new ProductService() — you coupled yourself to the implementation.
    //   You broke testability. You broke the Dependency Inversion Principle.
    public ProductsController(IProductService productService)
    {
        _productService = productService;
    }

    // Conventional routing maps: GET /Products/Index
    // 'Index' is the method name — convention routing uses method name as action
    public IActionResult Index()
    {
        var products = _productService.GetAll();
        return View(products);  // MVC returns a View
    }

    // Conventional routing maps: GET /Products/Details/42
    // 'id' matches the {id?} segment in the route template
    public IActionResult Details(int id)
    {
        var product = _productService.GetById(id);
        if (product == null) return NotFound();
        return View(product);
    }
}
```

**WHY does conventional routing exist?**

Historically, before REST APIs became dominant, web apps were page-based. The URL `/Products/Details/5` naturally described _"go to Products section, show Details page, for item 5"_. This was **intuitive for MVC web applications** where controllers had many actions with predictable names.

**Problems with conventional routing:**

1. You can accidentally expose actions you didn't intend to expose
2. Order of route registration matters — can cause wrong route to match
3. Hard to have non-standard URLs like `/api/v2/products/featured`
4. Tight coupling between method name and URL

---

### Type 2 — Attribute Routing (Web API Pattern)

```csharp
// ProductsController.cs — Web API Style
[ApiController]           // WHY? Enables automatic model validation, problem details, etc.
[Route("api/[controller]")] // WHY? [controller] is a token replaced with class name minus "Controller"
                            // ProductsController → "products" (lowercased automatically)
                            // Result: all actions under /api/products
public class ProductsController : ControllerBase  // WHY ControllerBase not Controller?
                                                   // ControllerBase has no View() support
                                                   // APIs don't return Views — they return JSON/XML
                                                   // Controller inherits ControllerBase + View support
                                                   // Using Controller for APIs adds unnecessary overhead
{
    private readonly IProductService _productService;
    private readonly ILogger<ProductsController> _logger;
    // WHY ILogger<ProductsController>? 
    // Generic type parameter is used for log category name
    // Logs will show: "ProductsController" as source category
    // This helps filter logs in production by controller name

    public ProductsController(
        IProductService productService,
        ILogger<ProductsController> logger)
    {
        _productService = productService;
        _logger = logger;
    }

    // GET api/products
    [HttpGet]                    // WHY explicit HttpGet? — In Web API, you MUST declare HTTP verb
                                 // Convention routing guesses from method name (Get*, Post*, etc.)
                                 // Attribute routing forces you to be explicit — no ambiguity
    [ProducesResponseType(typeof(IEnumerable<ProductDto>), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status500InternalServerError)]
    // WHY ProducesResponseType? — Documents the API (Swagger/OpenAPI reads this)
    //                             Tells consumers what to expect
    //                             Not enforced at runtime — purely documentary + Swagger
    public async Task<ActionResult<IEnumerable<ProductDto>>> GetAll()
    {
        // WHY ActionResult<T> instead of just T?
        // ActionResult<T> allows returning EITHER:
        //   - Ok(data)        → 200 with body
        //   - NotFound()      → 404 with no body
        //   - BadRequest()    → 400
        //   - Problem(...)    → RFC 7807 problem details
        // If you returned just IEnumerable<ProductDto>, you lose this flexibility
        
        _logger.LogInformation("Fetching all products");
        var products = await _productService.GetAllAsync();
        return Ok(products);
    }

    // GET api/products/42
    [HttpGet("{id:int}")]   // WHY {id:int}? — Route CONSTRAINT
                            // :int means this route only matches if id segment is an integer
                            // GET /api/products/abc — will NOT match this route (no int)
                            // GET /api/products/42  — WILL match
                            // Without :int, "abc" would match and fail at model binding — worse error
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status200OK)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<ActionResult<ProductDto>> GetById(int id)
    {
        var product = await _productService.GetByIdAsync(id);
        
        if (product is null)
        {
            _logger.LogWarning("Product {ProductId} not found", id);
            // WHY not string interpolation $"Product {id} not found"?
            // Structured logging — {ProductId} becomes a searchable property in Seq/Splunk/ELK
            // String interpolation loses the structured data — just becomes a flat string
            return NotFound();
        }
        
        return Ok(product);
    }

    // POST api/products
    [HttpPost]
    [ProducesResponseType(typeof(ProductDto), StatusCodes.Status201Created)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    public async Task<ActionResult<ProductDto>> Create(
        [FromBody] CreateProductRequest request)
        // WHY [FromBody]? — Tells model binder: read this from HTTP request body (JSON)
        // Alternatives:
        //   [FromQuery]  → ?name=laptop&price=999 (query string)
        //   [FromRoute]  → /products/{name} (URL segment)
        //   [FromHeader] → X-Custom-Header: value
        //   [FromForm]   → multipart/form-data (file uploads, HTML forms)
        // With [ApiController], [FromBody] is inferred for complex types
        // But being explicit is better for clarity and documentation
    {
        // With [ApiController], if ModelState is invalid, 
        // framework automatically returns 400 BEFORE your code runs
        // You don't need: if (!ModelState.IsValid) return BadRequest(...)
        // This is one of the major benefits of [ApiController]
        
        var created = await _productService.CreateAsync(request);
        
        // WHY CreatedAtAction instead of Ok()?
        // REST standard: POST that creates resource should return 201 Created
        // AND include Location header pointing to the new resource
        // Location: https://myapi.com/api/products/43
        // This is HATEOAS — Hypermedia As The Engine Of Application State
        // Clients discover the URL of the new resource without hardcoding it
        return CreatedAtAction(
            nameof(GetById),      // Which action generates the URL?
            new { id = created.Id }, // Route values for that action
            created);             // The response body
    }

    // PUT api/products/42
    [HttpPut("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status400BadRequest)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Update(
        int id, 
        [FromBody] UpdateProductRequest request)
        // WHY IActionResult instead of ActionResult<T>?
        // 204 No Content returns no body — there's no T to return
        // ActionResult<T> is only valuable when you have a T to return on success
    {
        if (id != request.Id)
            return BadRequest("Route ID and body ID mismatch");
        
        var success = await _productService.UpdateAsync(id, request);
        if (!success) return NotFound();
        
        return NoContent(); // 204 — success but nothing to return
    }

    // DELETE api/products/42
    [HttpDelete("{id:int}")]
    [ProducesResponseType(StatusCodes.Status204NoContent)]
    [ProducesResponseType(StatusCodes.Status404NotFound)]
    public async Task<IActionResult> Delete(int id)
    {
        var success = await _productService.DeleteAsync(id);
        if (!success) return NotFound();
        return NoContent();
    }
}
```

---

## 6. Route Constraints — Deep Dive

Route constraints are **guards** that filter which requests a route will match.

```csharp
// Built-in constraints:
[HttpGet("{id:int}")]           // Must be integer
[HttpGet("{id:guid}")]          // Must be GUID: 3fa85f64-5717-4562-b3fc-2c963f66afa6
[HttpGet("{id:long}")]          // Must be long integer
[HttpGet("{slug:alpha}")]       // Only letters (no numbers)
[HttpGet("{age:range(1,120)}")]  // Integer between 1 and 120
[HttpGet("{name:minlength(3)}")] // String minimum 3 characters
[HttpGet("{name:maxlength(50)}")] // String maximum 50 characters
[HttpGet("{id:int:min(1)}")]    // Multiple constraints — id must be int AND >= 1

// Custom constraint example:
public class EvenNumberConstraint : IRouteConstraint
{
    public bool Match(HttpContext? httpContext, IRouter? route, 
                      string routeKey, RouteValueDictionary values, 
                      RouteDirection routeDirection)
    {
        if (values.TryGetValue(routeKey, out var value))
        {
            if (int.TryParse(value?.ToString(), out int intValue))
                return intValue % 2 == 0; // Only matches even numbers
        }
        return false;
    }
}

// Register custom constraint:
builder.Services.AddRouting(options =>
{
    options.ConstraintMap.Add("even", typeof(EvenNumberConstraint));
});

// Use it:
[HttpGet("{id:even}")] // Only matches /products/2, /products/4, etc.
```

---

## 7. Versioning — Real Production Pattern

```csharp
// Version 1
[ApiController]
[Route("api/v{version:apiVersion}/products")]
[ApiVersion("1.0")]
public class ProductsV1Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() 
    {
        return Ok(new[] { "Product V1 format" });
    }
}

// Version 2 — new format, backward compatible
[ApiController]
[Route("api/v{version:apiVersion}/products")]
[ApiVersion("2.0")]
public class ProductsV2Controller : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll()
    {
        return Ok(new[] { new { Id = 1, Name = "Product", Category = "V2 has categories" } });
    }
}

// Requests:
// GET /api/v1/products → V1 controller
// GET /api/v2/products → V2 controller
```

---

## 8. Why Industry Uses Attribute Routing for APIs

|Concern|Conventional Routing|Attribute Routing|
|---|---|---|
|**Explicit**|No — guessed from method name|Yes — you declare exactly|
|**REST-friendly**|Hard — `/Products/GetAll` is not REST|Easy — `/api/products`|
|**Versioning**|Painful|Natural — `/api/v2/products`|
|**Discoverability**|You must read code to know URLs|Attribute on method tells you|
|**Accidental exposure**|High risk|Zero — must be explicitly decorated|
|**Non-standard URLs**|Hard|Easy — `/api/products/featured/bestsellers`|

---

## 9. Common Mistakes & Anti-Patterns

### Mistake 1 — Mixing Conventional and Attribute Routing

```csharp
// BAD — Don't mix in the same controller
public class ProductsController : ControllerBase
{
    // This action uses attribute routing
    [HttpGet("api/products")]
    public IActionResult GetAll() { ... }
    
    // This action relies on conventional routing
    // Result: AMBIGUITY, unpredictable behavior
    public IActionResult Delete(int id) { ... }
}
```

### Mistake 2 — Returning Wrong Status Codes

```csharp
// BAD
[HttpPost]
public IActionResult Create(CreateProductRequest request)
{
    var product = _service.Create(request);
    return Ok(product); // 200 — Wrong! POST that creates should be 201
}

// GOOD
[HttpPost]
public IActionResult Create(CreateProductRequest request)
{
    var product = _service.Create(request);
    return CreatedAtAction(nameof(GetById), new { id = product.Id }, product); // 201
}
```

### Mistake 3 — Not Using Route Constraints

```csharp
// BAD — Both routes can match /api/products/something
[HttpGet("{id}")]
[HttpGet("{name}")]

// GOOD — Constraints disambiguate
[HttpGet("{id:int}")]      // Only integers
[HttpGet("{name:alpha}")]  // Only letters
```

### Mistake 4 — Hardcoding Controller Name in Route

```csharp
// BAD — If you rename the controller, the URL breaks silently
[Route("api/products")]

// GOOD — Token replaces itself with actual controller name
[Route("api/[controller]")]
// ProductsController → /api/products automatically
```

---

## 10. Interview Questions You Must Know Cold

1. **What is the difference between `UseRouting()` and `UseEndpoints()`? Why are they separate?**
2. **Why does middleware order matter in ASP.NET Core? What breaks if `UseAuthorization()` is before `UseRouting()`?**
3. **What is the difference between `Controller` and `ControllerBase`? When do you use each?**
4. **What does `[ApiController]` attribute actually do? Name 3 specific behaviors it enables.**
5. **What is the difference between `IActionResult` and `ActionResult<T>`?**
6. **Why does `CreatedAtAction` exist? What HTTP header does it set?**
7. **How does ASP.NET Core routing achieve O(log n) performance?**
8. **What is the difference between `[FromBody]`, `[FromQuery]`, `[FromRoute]`?**
9. **When would you use conventional routing over attribute routing in 2024+?**
10. **What happens if two routes match the same URL?**

---

## Now I Cross-Examine You 🎯

Answer these before we move forward. Think carefully — don't guess:

### Question 1 — Conceptual

If I put `UseAuthorization()` **before** `UseRouting()` in my middleware pipeline, what will happen? Will authorization work correctly? Why or why not?

### Question 2 — Predict the Behavior

```csharp
[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult GetById(int id) { ... }
    
    [HttpGet("{customerId}/orders")]
    public IActionResult GetByCustomer(string customerId) { ... }
}
```

If I send `GET /api/orders/42/orders` — which action gets called? Why?

### Question 3 — Design Decision

You are building an e-commerce API. You need this URL:

```
GET /api/products/electronics/laptops/featured?maxPrice=2000&inStock=true
```

How would you design the controller routing for this? What would you use for `electronics`, `laptops`, `featured`, `maxPrice`, and `inStock`? Which binding source for each?

### Question 4 — What Breaks?

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet("{id:int}")]
    public IActionResult GetById(int id) { ... }
}
```

What happens when a client sends `GET /api/products/abc`? Trace the exact behavior step by step.

### Question 5 — Architecture Decision

A junior developer says: _"Why do we need `[Route]` on the controller AND `[HttpGet]` on the method? Why not just put the full route on `[HttpGet]`?"_

What would you tell them? Is the junior developer wrong? Are there cases where their approach is valid?

---

## Mini Assignment 🔨

Build a `OrdersController` with full attribute routing for this domain:

- `GET /api/orders` — Get all orders (with optional `?status=pending` query filter)
- `GET /api/orders/{id:int}` — Get specific order
- `GET /api/orders/{orderId:int}/items` — Get all items in an order
- `POST /api/orders` — Create order, return 201 with Location header
- `PUT /api/orders/{id:int}/status` — Update only the status (not full update)
- `DELETE /api/orders/{id:int}` — Soft delete (mark as cancelled, not DB delete)

Rules:

- Use correct HTTP status codes for every scenario
- Use `ActionResult<T>` where body is returned
- Use `IActionResult` where no body is returned
- Add meaningful `[ProducesResponseType]` attributes
- Handle not-found cases correctly
- Use structured logging for at least 2 actions