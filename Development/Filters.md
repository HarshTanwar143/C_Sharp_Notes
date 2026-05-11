## 1. Simple Explanation — What Are Filters?

Imagine you have 50 API endpoints. Every endpoint needs to:

- Check if the user is logged in
- Log who called it and when
- Validate the incoming data
- Handle errors gracefully
- Maybe cache the response

**Without filters**, you'd write that logic inside _every single controller action_. 50 endpoints × 5 concerns = 250 places to maintain. Change one thing? Update 250 places. Miss one? Bug in production.

**Filters solve this.** They let you write that logic _once_ and attach it to actions, controllers, or globally — automatically running before/after your action executes.

---

## 2. Real-World Analogy

Think of an airport security system:

```
You (Request) →
  [Ticket Check] →        ← Authorization Filter
  [Baggage Scan] →        ← Resource Filter
  [Boarding Gate] →       ← Action Filter (Before)
  [You board plane] →     ← Your Controller Action
  [Gate closes] →         ← Action Filter (After)
  [Flight logs filed] →   ← Result Filter
  [Emergency protocol]    ← Exception Filter
```

Each checkpoint runs in order. You don't rewrite "check passport" for every flight. It's defined once, applies everywhere.

---

## 3. Technical Deep Dive — The Filter Pipeline

ASP.NET Core has **6 types of filters**, each intercepting a different stage:

```
				HTTP Request
					│
					▼
┌─────────────────────────────────────────────┐
│           MIDDLEWARE PIPELINE               │
│          (Auth, Routing, etc.)              │
└──────────────────┬──────────────────────────┘
                   │
                   ▼
        ┌──────────────────┐
        │  Authorization   │  ← Runs FIRST. No "After" phase.
        │     Filters      │    If denied → short-circuits here.
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │    Resource      │  ← Wraps everything below.
        │     Filters      │    Good for caching.
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │  Model Binding   │  ← Framework binds [FromBody], etc.
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │     Action       │  ← Before AND After action executes.
        │     Filters      │    Most commonly used filter.
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │  YOUR ACTION     │  ← Your actual controller method.
        │    METHOD        │
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │    Exception     │  ← Catches unhandled exceptions.
        │     Filters      │
        └────────┬─────────┘
                 │
                 ▼
        ┌──────────────────┐
        │     Result       │  ← Before AND After IActionResult
        │     Filters      │    executes. (e.g., JSON serialization)
        └────────┬─────────┘
                 │
                 ▼
         HTTP Response
```

---

## 4. Internal Working — How Filters Execute

The ASP.NET Core framework builds a **filter pipeline** at runtime using an `IFilterFactory` and `FilterDescriptor` objects. Here's what happens internally:

```csharp
// Internally, ASP.NET Core does something like this (simplified):
// In ActionInvoker.cs (framework source)

var filters = new IFilterMetadata[]
{
    new AuthorizationFilter(),
    new ResourceFilter(),
    new ActionFilter(),
    new ExceptionFilter(),
    new ResultFilter()
};

// Then executes them as a nested pipeline (like Russian dolls)
```

Each filter implements one or more of these interfaces:

```csharp
// The 6 filter interfaces:
IAuthorizationFilter          // OnAuthorization()
IAsyncAuthorizationFilter     // OnAuthorizationAsync()

IResourceFilter               // OnResourceExecuting() / OnResourceExecuted()
IAsyncResourceFilter          // OnResourceExecutionAsync()

IActionFilter                 // OnActionExecuting() / OnActionExecuted()
IAsyncActionFilter            // OnActionExecutionAsync()

IExceptionFilter              // OnException()
IAsyncExceptionFilter         // OnExceptionAsync()

IResultFilter                 // OnResultExecuting() / OnResultExecuted()
IAsyncResultFilter            // OnResultExecutionAsync()

IAlwaysRunResultFilter        // Runs even when action is short-circuited
```

---

## 5. Code — Every Filter Type With Deep Explanation

### Authorization Filter

```csharp
// IAuthorizationFilter is the FIRST filter that runs.
// It has NO "After" phase — by design.
// WHY? Because if auth fails, nothing else should run.
// Allowing an "after" phase would be a security risk.

public class ApiKeyAuthorizationFilter : IAuthorizationFilter
{
    // WHY private readonly? 
    // private   → encapsulation. Nothing outside touches this.
    // readonly  → assigned once in constructor, never mutated.
    //             Protects against accidental reassignment.
    //             Signals intent: "this is a dependency, not state."
    private readonly string _expectedApiKey;

    // Constructor injection — the CORRECT way.
    // WHY not: _expectedApiKey = "hardcoded-key"?
    // Because that's not testable, not configurable, violates SRP.
    public ApiKeyAuthorizationFilter(IConfiguration configuration)
    {
        // IConfiguration reads from appsettings.json / env vars / secrets
        // WHY not hardcode? Because values differ per environment.
        _expectedApiKey = configuration["ApiKey"] 
            ?? throw new InvalidOperationException("ApiKey not configured");
        // WHY ?? throw? Fail fast. Don't let app start in broken state.
    }

    // OnAuthorization — runs BEFORE action, no after.
    // context: gives you HttpContext, RouteData, ActionDescriptor
    public void OnAuthorization(AuthorizationFilterContext context)
    {
        // TryGetValue — safe dictionary read. No KeyNotFoundException.
        var apiKeyPresent = context.HttpContext.Request.Headers
            .TryGetValue("X-Api-Key", out var extractedApiKey);

        if (!apiKeyPresent || extractedApiKey != _expectedApiKey)
        {
            // Setting Result SHORT-CIRCUITS the pipeline.
            // Nothing below this runs — not action filters, 
            // not your controller, nothing.
            // This is the CORRECT way to block a request.
            context.Result = new UnauthorizedObjectResult(new
            {
                Error = "Invalid or missing API key"
            });
            // WHY not throw exception here?
            // Throwing would go to ExceptionFilter — wrong layer.
            // Authorization failure is expected behavior, not exceptional.
        }
        // If we DON'T set context.Result, pipeline continues.
    }
}
```

---

### Resource Filter — The Caching Use Case

```csharp
// Resource Filters wrap the ENTIRE action pipeline.
// They run BEFORE model binding.
// WHY before model binding? 
// If you have cached response, why even bother reading request body?
// Save CPU, memory, I/O — massive performance gain.

public class CacheResourceFilter : IResourceFilter
{
    private readonly IMemoryCache _cache;
    private readonly ILogger<CacheResourceFilter> _logger;

    public CacheResourceFilter(IMemoryCache cache, 
                                ILogger<CacheResourceFilter> logger)
    {
        _cache = cache;
        _logger = logger;
    }

    // OnResourceExecuting → runs BEFORE model binding + action
    public void OnResourceExecuting(ResourceExecutingContext context)
    {
        // Build cache key from the request path + query string
        var cacheKey = context.HttpContext.Request.Path 
                       + context.HttpContext.Request.QueryString;

        // TryGetValue returns true if found — no exception on miss
        if (_cache.TryGetValue(cacheKey, out string? cachedResponse))
        {
            _logger.LogInformation("Cache HIT for {Key}", cacheKey);
            
            // Short-circuit! Skip action entirely.
            // ContentResult writes string directly to response.
            context.Result = new ContentResult
            {
                Content = cachedResponse,
                ContentType = "application/json",
                StatusCode = 200
            };
            // Pipeline stops here. Action never runs.
            return;
        }

        _logger.LogInformation("Cache MISS for {Key}", cacheKey);
        // Don't set Result → pipeline continues to action
    }

    // OnResourceExecuted → runs AFTER action + result
    public void OnResourceExecuted(ResourceExecutedContext context)
    {
        // Store result in cache for next request
        var cacheKey = context.HttpContext.Request.Path 
                       + context.HttpContext.Request.QueryString;

        if (context.Result is ObjectResult objectResult)
        {
            // Serialize and cache — simplified example
            var serialized = System.Text.Json.JsonSerializer
                .Serialize(objectResult.Value);
            
            _cache.Set(cacheKey, serialized, TimeSpan.FromMinutes(5));
        }
    }
}
```

---

### Action Filter — Most Commonly Used

```csharp
// Action Filters run AFTER model binding but BEFORE action execution.
// They have full access to action arguments.
// Most common uses:
// - Validation
// - Logging
// - Timing
// - Modifying action arguments

public class ValidationActionFilter : IActionFilter
{
    private readonly ILogger<ValidationActionFilter> _logger;

    public ValidationActionFilter(ILogger<ValidationActionFilter> logger)
    {
        _logger = logger;
    }

    // Runs BEFORE the action method
    // context.ActionArguments → the bound parameters of your action
    public void OnActionExecuting(ActionExecutingContext context)
    {
        // ModelState is populated by model binding BEFORE this runs.
        // That's WHY action filters are perfect for validation.
        if (!context.ModelState.IsValid)
        {
            // Extract all validation errors into a clean dictionary
            var errors = context.ModelState
                .Where(x => x.Value?.Errors.Count > 0)
                .ToDictionary(
                    kvp => kvp.Key,                    // Field name
                    kvp => kvp.Value!.Errors           // Errors for that field
                              .Select(e => e.ErrorMessage)
                              .ToArray()
                );

            // Short-circuit with 400 Bad Request
            context.Result = new BadRequestObjectResult(new
            {
                Message = "Validation failed",
                Errors = errors
            });
            return;
        }

        // Log who is calling what
        var actionName = context.ActionDescriptor.DisplayName;
        var user = context.HttpContext.User.Identity?.Name ?? "Anonymous";
        _logger.LogInformation(
            "Action {Action} called by {User} at {Time}", 
            actionName, user, DateTime.UtcNow);
    }

    // Runs AFTER the action method completes
    // context.Result → what the action returned
    public void OnActionExecuted(ActionExecutedContext context)
    {
        // context.Exception → if action threw, it's here
        // context.ExceptionHandled → set to true to swallow it

        if (context.Exception != null)
        {
            _logger.LogError(context.Exception, 
                "Action threw an unhandled exception");
            // Don't set ExceptionHandled = true here
            // Let ExceptionFilter handle it properly
        }

        _logger.LogInformation(
            "Action completed. Result type: {Type}", 
            context.Result?.GetType().Name);
    }
}
```

---

### Exception Filter — Centralized Error Handling

```csharp
// Exception Filters catch UNHANDLED exceptions from:
// - Action Filters
// - Model Binding
// - Action Methods
// They do NOT catch exceptions from:
// - Result Filters
// - Resource Filters (before model binding)
// - Middleware (outside filter pipeline)

// WHY not just use try-catch in every action?
// Imagine 100 actions all with:
// try { ... } catch (Exception ex) { return StatusCode(500); }
// That's 100 copies of the same code. DRY violation.
// What if you want to change error format? Update 100 places.

public class GlobalExceptionFilter : IExceptionFilter
{
    private readonly ILogger<GlobalExceptionFilter> _logger;
    private readonly IWebHostEnvironment _env;

    public GlobalExceptionFilter(
        ILogger<GlobalExceptionFilter> logger,
        IWebHostEnvironment env)
    {
        _logger = logger;
        _env = env;
    }

    public void OnException(ExceptionContext context)
    {
        // context.Exception → the actual exception
        // context.ExceptionHandled → set true to stop propagation

        _logger.LogError(
            context.Exception,
            "Unhandled exception for request {Path}",
            context.HttpContext.Request.Path);

        // Pattern: switch on exception type for different responses
        var (statusCode, message) = context.Exception switch
        {
            // Domain-level exceptions mapped to HTTP status codes
            UnauthorizedAccessException => 
                (StatusCodes.Status403Forbidden, "Access denied"),
            
            KeyNotFoundException => 
                (StatusCodes.Status404NotFound, "Resource not found"),
            
            ArgumentException ex => 
                (StatusCodes.Status400BadRequest, ex.Message),
            
            // Catch-all for unexpected errors
            _ => (StatusCodes.Status500InternalServerError, 
                  "An unexpected error occurred")
        };

        // In Development, expose full details for debugging
        // In Production, NEVER expose internal error details
        // WHY? Security. Stack traces reveal architecture to attackers.
        var detail = _env.IsDevelopment() 
            ? context.Exception.ToString() 
            : null;

        context.Result = new ObjectResult(new
        {
            StatusCode = statusCode,
            Message = message,
            Detail = detail
        })
        {
            StatusCode = statusCode
        };

        // CRITICAL: Mark as handled. 
        // Without this, ASP.NET Core will ALSO process it — double handling.
        context.ExceptionHandled = true;
    }
}
```

---

### Result Filter — Post-Processing Responses

```csharp
// Result Filters wrap IActionResult execution.
// They run AFTER action filter's OnActionExecuted.
// Use for:
// - Adding response headers
// - Wrapping responses in standard envelope
// - Logging response details

public class ResponseEnvelopeResultFilter : IResultFilter
{
    // Runs BEFORE IActionResult.ExecuteResultAsync()
    public void OnResultExecuting(ResultExecutingContext context)
    {
        // Add standard security/caching headers to every response
        context.HttpContext.Response.Headers.Add(
            "X-Content-Type-Options", "nosniff");
        context.HttpContext.Response.Headers.Add(
            "X-Frame-Options", "DENY");
    }

    // Runs AFTER IActionResult writes to response
    public void OnResultExecuted(ResultExecutedContext context)
    {
        // Response is already written to stream here.
        // You CAN'T modify response body (headers are sent).
        // Common use: logging response status code
    }
}
```

---

## 6. Registering Filters — 3 Scopes

```csharp
// ═══════════════════════════════════════════
// SCOPE 1: GLOBAL — applies to ALL endpoints
// ═══════════════════════════════════════════
// In Program.cs:
builder.Services.AddControllers(options =>
{
    // Added to ALL controllers, ALL actions, globally.
    // Order matters — filters added first run first (for same type).
    options.Filters.Add<GlobalExceptionFilter>();
    options.Filters.Add<ValidationActionFilter>();
});

// WHY register filters in DI AND in options?
// Because filters can have constructor dependencies.
// If you just do: options.Filters.Add(new MyFilter())
// → you lose DI injection. The filter's dependencies are null.
// The correct way: options.Filters.Add<MyFilter>()
// → ASP.NET Core creates it through DI container.

// ═══════════════════════════════════════════
// SCOPE 2: CONTROLLER — applies to all actions in one controller
// ═══════════════════════════════════════════

[ApiController]
[Route("api/[controller]")]
[ServiceFilter(typeof(CacheResourceFilter))] // ← applied to whole controller
public class ProductsController : ControllerBase
{
    // All actions in this controller get CacheResourceFilter

    [HttpGet]
    public IActionResult GetAll() { ... }

    [HttpGet("{id}")]
    public IActionResult GetById(int id) { ... }
}

// ═══════════════════════════════════════════
// SCOPE 3: ACTION — applies to one specific action
// ═══════════════════════════════════════════

[HttpDelete("{id}")]
[ServiceFilter(typeof(ApiKeyAuthorizationFilter))] // ← only this action
public IActionResult Delete(int id)
{
    ...
}
```

---

## 7. ServiceFilter vs TypeFilter vs Direct Attribute

```csharp
// ─────────────────────────────────────────────────────
// Option A: Direct Attribute (BAD for filters with dependencies)
// ─────────────────────────────────────────────────────
// This CANNOT use constructor injection from DI container.
// All parameters must be compile-time constants.
[MyFilter("hardcoded-value")]   // ← only works with no DI dependencies

// ─────────────────────────────────────────────────────
// Option B: ServiceFilter (RECOMMENDED — uses DI fully)
// ─────────────────────────────────────────────────────
// The filter MUST be registered in DI container.
// ASP.NET Core resolves it through the container.
// Respects lifetime (scoped, transient, singleton).

// Register in DI:
builder.Services.AddScoped<ValidationActionFilter>();

// Use on controller/action:
[ServiceFilter(typeof(ValidationActionFilter))]
public class OrdersController : ControllerBase { ... }

// ─────────────────────────────────────────────────────
// Option C: TypeFilter (DI but WITHOUT pre-registration)
// ─────────────────────────────────────────────────────
// Creates filter instance using DI, but you don't need to
// register the filter itself. You CAN pass extra constructor args.
[TypeFilter(typeof(CacheResourceFilter), Arguments = new object[] { 300 })]
public IActionResult GetProducts() { ... }

// WHEN to use which?
// ServiceFilter  → filter has DI deps, no extra args, pre-registered
// TypeFilter     → filter has DI deps + extra constructor args
// Direct attr    → filter has NO DI dependencies (avoid usually)
```

---

## 8. Async Filters — Why and When

```csharp
// Sync filter interface → two separate methods (Before + After)
// Async filter interface → one method with delegate (more control)

// WHY async?
// If your filter does I/O (DB query, HTTP call, cache lookup)
// using sync filter BLOCKS the thread.
// In high-throughput apps, blocking threads = performance killer.

public class AsyncLoggingActionFilter : IAsyncActionFilter
{
    private readonly ILogger<AsyncLoggingActionFilter> _logger;
    private readonly IUserActivityRepository _repo;

    public AsyncLoggingActionFilter(
        ILogger<AsyncLoggingActionFilter> logger,
        IUserActivityRepository repo)
    {
        _logger = logger;
        _repo = repo;
    }

    // Single method — YOU control when "next" runs
    public async Task OnActionExecutionAsync(
        ActionExecutingContext context,
        ActionExecutionDelegate next)  // ← "next" = rest of pipeline
    {
        // ────── BEFORE ACTION ──────
        var stopwatch = Stopwatch.StartNew();
        var actionName = context.ActionDescriptor.DisplayName;

        _logger.LogInformation("Starting: {Action}", actionName);

        // ────── EXECUTE ACTION + ALL FILTERS AFTER THIS ──────
        // "next()" runs the action method AND the next filters.
        // CRITICAL: If you don't call next(), action never executes.
        // This is how you short-circuit in async filters.
        var executedContext = await next();

        // ────── AFTER ACTION ──────
        stopwatch.Stop();

        // Log to database ASYNCHRONOUSLY — no thread blocking
        await _repo.LogActivityAsync(new UserActivity
        {
            ActionName = actionName,
            DurationMs = stopwatch.ElapsedMilliseconds,
            UserId = context.HttpContext.User.Identity?.Name,
            StatusCode = context.HttpContext.Response.StatusCode,
            ExceptionOccurred = executedContext.Exception != null,
            Timestamp = DateTime.UtcNow
        });

        _logger.LogInformation(
            "Completed: {Action} in {Ms}ms", 
            actionName, stopwatch.ElapsedMilliseconds);
    }
}
```

---

## 9. Filter Execution Order

```csharp
// Multiple filters of the SAME type → Order property controls sequence.
// Default Order = 0. Lower number = runs FIRST (outer layer).

// Example: Two Action Filters
[ServiceFilter(typeof(LoggingFilter))]    // Order = 0 (default)
[ServiceFilter(typeof(ValidationFilter))] // Order = 0 (default)
public class OrdersController { }

// Execution:
// LoggingFilter.OnActionExecuting
//   ValidationFilter.OnActionExecuting
//     → ACTION RUNS
//   ValidationFilter.OnActionExecuted
// LoggingFilter.OnActionExecuted

// To control order explicitly:
public class LoggingFilter : IActionFilter, IOrderedFilter
{
    public int Order => -1; // Runs BEFORE ValidationFilter (Order=0)
    // Lower = outer = runs first before, runs last after
    public void OnActionExecuting(ActionExecutingContext context) { }
    public void OnActionExecuted(ActionExecutedContext context) { }
}
```

---

## 10. Real Production Example — Putting It All Together

```csharp
// Program.cs — Production-grade filter setup
builder.Services.AddScoped<ValidationActionFilter>();
builder.Services.AddScoped<GlobalExceptionFilter>();
builder.Services.AddScoped<ApiKeyAuthorizationFilter>();
builder.Services.AddScoped<AsyncLoggingActionFilter>();
builder.Services.AddSingleton<CacheResourceFilter>();
// WHY Singleton for cache filter? 
// Cache itself is singleton, filter just wraps it.
// No per-request state in the filter itself.

builder.Services.AddControllers(options =>
{
    options.Filters.Add<GlobalExceptionFilter>();    // Global exception handling
    options.Filters.Add<ValidationActionFilter>();   // Global validation
    options.Filters.Add<AsyncLoggingActionFilter>(); // Global audit logging
});

// Controller using specific filters
[ApiController]
[Route("api/[controller]")]
[ServiceFilter(typeof(ApiKeyAuthorizationFilter))]
public class AdminController : ControllerBase
{
    private readonly IAdminService _adminService;

    public AdminController(IAdminService adminService)
    {
        _adminService = adminService;
    }

    [HttpGet("reports")]
    [ServiceFilter(typeof(CacheResourceFilter))] // Only cache this action
    public async Task<IActionResult> GetReports()
    {
        var reports = await _adminService.GetReportsAsync();
        return Ok(reports);
    }

    [HttpDelete("users/{id}")]
    // No cache here — deletions should never be cached
    public async Task<IActionResult> DeleteUser(int id)
    {
        await _adminService.DeleteUserAsync(id);
        return NoContent();
    }
}
```

---

## 11. Filters vs Middleware — When to Use What

||**Filters**|**Middleware**|
|---|---|---|
|Scope|MVC/API pipeline only|Entire HTTP pipeline|
|Access to action args|✅ Yes|❌ No|
|Access to ModelState|✅ Yes|❌ No|
|Can short-circuit|✅ Yes|✅ Yes|
|DI Friendly|✅ Yes|✅ Yes|
|Works with Razor Pages|✅ Yes|✅ Yes|
|Works with static files|❌ No|✅ Yes|
|Best for|Action-specific logic|Cross-cutting HTTP concerns|

```
Rule of Thumb:
- Need access to controller/action context? → Filter
- Need to run for static files, SignalR, gRPC, etc.? → Middleware
- Global auth/CORS/HTTPS redirect? → Middleware
- Validation, logging per-action, caching per-endpoint? → Filter
```

---

## 12. Common Mistakes (Anti-Patterns)

```csharp
// ❌ BAD: Creating service directly in filter
public class BadFilter : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        var service = new UserService(); // Hard dependency. Not testable.
        // What lifetime is this? What if UserService needs DbContext?
        // Memory leak risk. No DI. No mocking in tests.
    }
}

// ✅ GOOD: Constructor injection
public class GoodFilter : IActionFilter
{
    private readonly IUserService _userService;
    public GoodFilter(IUserService userService) => _userService = userService;
    public void OnActionExecuting(ActionExecutingContext context) { ... }
}

// ❌ BAD: Storing request-scoped state in singleton filter
public class BadSingletonFilter : IActionFilter  // registered as Singleton
{
    private string _currentUser; // DANGEROUS: shared across ALL requests

    public void OnActionExecuting(ActionExecutingContext context)
    {
        _currentUser = context.HttpContext.User.Identity.Name;
        // Request A sets _currentUser = "Alice"
        // Request B sets _currentUser = "Bob"
        // Request A reads _currentUser → "Bob"!!! RACE CONDITION!!!
    }
}

// ✅ GOOD: Register as Scoped, or use context directly
public class GoodScopedFilter : IActionFilter // registered as Scoped
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        var currentUser = context.HttpContext.User.Identity?.Name;
        // context is per-request. No shared mutable state.
    }
}

// ❌ BAD: Exception handling in action instead of filter
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(int id)
{
    try
    {
        var order = await _service.GetAsync(id);
        return Ok(order);
    }
    catch (KeyNotFoundException)
    {
        return NotFound(); // Copy-pasted in 50 actions
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "...");
        return StatusCode(500);
    }
}

// ✅ GOOD: Let GlobalExceptionFilter handle it. Action stays clean.
[HttpGet("{id}")]
public async Task<IActionResult> GetOrder(int id)
{
    var order = await _service.GetAsync(id);
    return Ok(order);
    // If KeyNotFoundException throws → ExceptionFilter maps it to 404
    // If anything else throws → ExceptionFilter maps it to 500
}
```

---

## 13. Interview Questions You Must Know

1. What is the difference between `IActionFilter` and `IAsyncActionFilter`? Which should you prefer and why?
2. Can an `ExceptionFilter` catch exceptions thrown inside a `ResultFilter`?
3. What is the difference between `ServiceFilter` and `TypeFilter`?
4. If you register the same filter globally AND on a controller, does it run twice?
5. What happens if you DON'T call `next()` in an async filter?
6. What's the difference between short-circuiting in an `AuthorizationFilter` vs an `ActionFilter`?
7. Why can't `ResourceFilter` access `ModelState`?
8. What is `IAlwaysRunResultFilter` and when would you need it?
9. How does filter order work when filters are registered at different scopes (global vs controller vs action)?
10. What's the execution order when both a controller-level filter and action-level filter of the same type exist?

---

## Cross Questions For You — Answer These Before We Continue

**Question 1:** You have a `CacheFilter` registered as **Singleton**, and inside the `OnResourceExecuting`, you're reading `context.HttpContext.Request.Headers["Authorization"]`. What is the potential problem here?

**Question 2:** You have an `IAsyncActionFilter`. Inside `OnActionExecutionAsync`, you write some logging code BEFORE `await next()`. But you realize the log is being written even when `AuthorizationFilter` short-circuits the request. Is this possible? Why or why not?

**Question 3:** You want to write a filter that:

- Checks if the response will be a `200 OK`
- If yes, adds a custom header `X-Success: true`

Which filter type would you use and WHY? Could you use `ActionFilter` for this?

**Question 4 (Scenario):** You have a microservice with 80 endpoints. The team lead says: "Every response must be wrapped in `{ "data": ..., "timestamp": ..., "requestId": ... }`."

Which filter would you use? How would you implement it without touching all 80 controllers? What edge cases would you handle (what if the action returns `404`? What if it throws an exception?)?

**Question 5 (Predict the behavior):**

```csharp
public class FilterA : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        context.Result = new OkObjectResult("Short circuited by A");
    }
    public void OnActionExecuted(ActionExecutedContext context)
    {
        Console.WriteLine("FilterA After");
    }
}

public class FilterB : IActionFilter
{
    public void OnActionExecuting(ActionExecutingContext context)
    {
        Console.WriteLine("FilterB Before");
    }
    public void OnActionExecuted(ActionExecutedContext context)
    {
        Console.WriteLine("FilterB After");
    }
}

// Applied: [FilterA] [FilterB] on the same action
// FilterA runs first (inner), FilterB runs second (outer)
```

What gets printed? Does the action run? Does `FilterB.OnActionExecuting` run? Does `FilterA.OnActionExecuted` run?

---

## Mini Assignment

Build a `RequestAuditFilter` that:

1. Captures: action name, HTTP method, request path, query string, user identity, start time
2. After action: captures duration, response status code, whether an exception occurred
3. Logs it as structured JSON using `ILogger`
4. Must be async (in case you extend it to write to DB later)
5. Must handle the case where the action short-circuits (e.g., validation fails)
6. Register it globally
