## 1. Simple Explanation — What IS Middleware?

Imagine you order food at a restaurant.

Your order doesn't go directly to the chef. It passes through:

1. **Waiter** → writes down your order
2. **Manager** → checks if item is available
3. **Chef** → cooks the food
4. **Quality checker** → verifies presentation
5. **Waiter again** → delivers to your table

Each person in this chain:

- **Receives** the order
- **Does something** with it
- **Passes it forward** (or stops it)
- **Also handles the response** on the way back

**This is exactly what Middleware is.**

In ASP.NET Core, every HTTP request passes through a **pipeline of middleware components** — each one can:

- Inspect the request
- Modify the request
- Short-circuit (stop it from going further)
- Pass it to the next middleware
- Modify the response on the way back

---

## 2. Real-World Analogy — Airport Security

Think of an airport:

```
You (Request) → Check-in → Security → Passport Control → Gate → Plane (Endpoint)
                                                                      ↓
You (Response) ← Baggage Claim ← Exit ← Customs ← Landing ←────────┘
```

Each checkpoint:

- Can **let you pass**
- Can **stop you** (short-circuit)
- Can **stamp your passport** (modify request/response)
- Happens in a **specific order** — you can't go to passport control before security

**Order matters. Critically.**

---

## 3. Technical Deep Dive — The Pipeline

### What does the pipeline look like in code?

```csharp
// Program.cs — The entry point of ASP.NET Core app
var builder = WebApplication.CreateBuilder(args);
var app = builder.Build();

// Each Use___() call REGISTERS a middleware
app.UseHttpsRedirection();   // Middleware 1
app.UseStaticFiles();        // Middleware 2
app.UseRouting();            // Middleware 3
app.UseAuthentication();     // Middleware 4
app.UseAuthorization();      // Middleware 5
app.MapControllers();        // Terminal middleware (Endpoint)

app.Run();
```

Now let me show you what a middleware ACTUALLY is internally:

```csharp
// This is the SIMPLEST possible custom middleware
// Written as a raw RequestDelegate to show you the internals

public class MyLoggingMiddleware
{
    // _next represents the NEXT middleware in the pipeline
    // It is a delegate: Func<HttpContext, Task>
    // Think of it as a "pointer to the next checkpoint in the airport"
    private readonly RequestDelegate _next;

    // Constructor — ASP.NET Core's DI injects _next automatically
    // You NEVER call this constructor yourself
    public MyLoggingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    // Invoke or InvokeAsync — this is called by the PREVIOUS middleware
    // HttpContext contains EVERYTHING about the current request/response
    public async Task InvokeAsync(HttpContext context)
    {
        // ═══════════════════════════════
        // BEFORE — Request is going IN
        // ═══════════════════════════════
        Console.WriteLine($"[REQUEST] {context.Request.Method} {context.Request.Path}");
        var startTime = DateTime.UtcNow;

        // This line is CRITICAL
        // It says: "I'm done with my PRE-processing, now call the next middleware"
        // If you DON'T call this → you SHORT-CIRCUIT the pipeline
        await _next(context);

        // ═══════════════════════════════
        // AFTER — Response is coming OUT
        // ═══════════════════════════════
        var elapsed = DateTime.UtcNow - startTime;
        Console.WriteLine($"[RESPONSE] Status: {context.Response.StatusCode} | Time: {elapsed.TotalMilliseconds}ms");
    }
}
```

**Why `RequestDelegate`?**

```csharp
// RequestDelegate is defined in ASP.NET Core as:
public delegate Task RequestDelegate(HttpContext context);

// It's simply a function signature that says:
// "Give me an HttpContext, I'll return a Task (async operation)"
// Every middleware in the pipeline IS a RequestDelegate
// They are chained together like a linked list
```

---

## 4. Internal Working — How ASP.NET Core Chains Middleware

This is where most developers never look. Let me show you what's happening under the hood.

```
app.Use(MW1)
app.Use(MW2)
app.Use(MW3)
app.Run(FinalHandler)
```

Internally, ASP.NET Core builds this as **nested delegates** — like Russian dolls:

```
RequestDelegate pipeline = 
    MW1(
        MW2(
            MW3(
                FinalHandler
            )
        )
    );
```

When a request comes in, the runtime calls `pipeline(httpContext)`.

MW1 runs → calls `_next` → MW2 runs → calls `_next` → MW3 runs → calls `_next` → FinalHandler runs → returns → MW3 resumes after `await _next` → MW2 resumes → MW1 resumes.

**This is a call stack. Literally.**

```
Call Stack (Request going in):
    MW1.InvokeAsync()
        → MW2.InvokeAsync()
            → MW3.InvokeAsync()
                → FinalHandler()
                ← returns
            ← resumes after await
        ← resumes after await
    ← resumes after await
```

---

## 5. The Three Types of Middleware Registration

```csharp
// ══════════════════════════════════════════════
// TYPE 1: app.Use() — Standard middleware
// Runs code BEFORE and AFTER the next middleware
// ══════════════════════════════════════════════
app.Use(async (context, next) =>
{
    Console.WriteLine("Before");
    await next.Invoke();          // calls next middleware
    Console.WriteLine("After");  // runs when coming back
});


// ══════════════════════════════════════════════
// TYPE 2: app.Run() — Terminal middleware
// NEVER calls next — pipeline ENDS here
// Use only at the very end
// ══════════════════════════════════════════════
app.Run(async (context) =>
{
    // Notice: no 'next' parameter
    // This is the END of the line
    await context.Response.WriteAsync("Hello World");
});


// ══════════════════════════════════════════════
// TYPE 3: app.Map() — Branching middleware
// Splits the pipeline based on URL path
// Creates a completely SEPARATE pipeline branch
// ══════════════════════════════════════════════
app.Map("/api", apiApp =>
{
    // This branch ONLY runs for requests starting with /api
    apiApp.Use(async (context, next) =>
    {
        Console.WriteLine("API-specific middleware");
        await next.Invoke();
    });
    
    apiApp.Run(async context =>
    {
        await context.Response.WriteAsync("API Response");
    });
});
// Requests NOT starting with /api skip this branch entirely
```

---

## 6. Production-Level Custom Middleware — Full Pattern

```csharp
// ════════════════════════════════════════════════
// THE FULL PRODUCTION PATTERN
// Every senior .NET dev writes middleware this way
// ════════════════════════════════════════════════

public class GlobalExceptionHandlingMiddleware
{
    private readonly RequestDelegate _next;
    
    // You can inject SINGLETON services here in constructor
    // IMPORTANT: Only inject SINGLETON services in constructor
    // For Scoped/Transient services → use InvokeAsync parameters
    private readonly ILogger<GlobalExceptionHandlingMiddleware> _logger;

    public GlobalExceptionHandlingMiddleware(
        RequestDelegate next,
        ILogger<GlobalExceptionHandlingMiddleware> logger)  // Singleton — OK here
    {
        _next = next;
        _logger = logger;
    }

    // Scoped services (like DbContext) go as METHOD parameters, NOT constructor
    // ASP.NET Core is smart enough to inject them per-request here
    public async Task InvokeAsync(
        HttpContext context,
        IUserService userService)  // Scoped — injected per request here ✅
    {
        try
        {
            await _next(context);
        }
        catch (ValidationException ex)
        {
            _logger.LogWarning(ex, "Validation error occurred");
            await HandleExceptionAsync(context, ex, StatusCodes.Status400BadRequest);
        }
        catch (UnauthorizedAccessException ex)
        {
            _logger.LogWarning(ex, "Unauthorized access attempt");
            await HandleExceptionAsync(context, ex, StatusCodes.Status401Unauthorized);
        }
        catch (Exception ex)
        {
            // Catch-all — never let an exception bubble up unhandled
            _logger.LogError(ex, "An unhandled exception occurred");
            await HandleExceptionAsync(context, ex, StatusCodes.Status500InternalServerError);
        }
    }

    private static async Task HandleExceptionAsync(
        HttpContext context, 
        Exception exception, 
        int statusCode)
    {
        context.Response.ContentType = "application/json";
        context.Response.StatusCode = statusCode;

        var errorResponse = new
        {
            StatusCode = statusCode,
            Message = exception.Message,
            // Never expose stack trace in production!
            TraceId = context.TraceIdentifier
        };

        await context.Response.WriteAsJsonAsync(errorResponse);
    }
}

// ════════════════════════════════════════════
// EXTENSION METHOD — Industry Best Practice
// This is HOW you make middleware feel "native"
// Like UseAuthentication(), UseRouting() etc.
// ════════════════════════════════════════════
public static class MiddlewareExtensions
{
    // 'this IApplicationBuilder app' — Extension method on IApplicationBuilder
    // This is how UseAuthentication() etc. are implemented in ASP.NET Core source
    public static IApplicationBuilder UseGlobalExceptionHandling(
        this IApplicationBuilder app)
    {
        return app.UseMiddleware<GlobalExceptionHandlingMiddleware>();
        // UseMiddleware<T>() is the built-in method that:
        // 1. Reads T's constructor
        // 2. Resolves RequestDelegate + DI services
        // 3. Wraps it as a RequestDelegate in the pipeline
    }
}

// ═══════════════════════
// Usage in Program.cs
// ═══════════════════════
app.UseGlobalExceptionHandling();  // Clean, discoverable, testable
```

---

## 7. WHY Order Matters — The Most Critical Concept

This is where **bugs happen in production** when developers don't understand middleware.

```csharp
// ❌ WRONG ORDER — This will break your app in production
app.UseAuthorization();      // Runs BEFORE authentication!
app.UseAuthentication();     // User identity set AFTER authorization checked
app.UseRouting();            // Route not resolved when above middleware runs

// ✅ CORRECT ORDER — Microsoft's recommended order
app.UseExceptionHandler();   // 1st — Catch ALL exceptions from everything below
app.UseHttpsRedirection();   // 2nd — Redirect before doing any real work
app.UseStaticFiles();        // 3rd — Serve static files early, skip pipeline
app.UseRouting();            // 4th — Resolve WHICH endpoint will handle this
app.UseAuthentication();     // 5th — WHO are you? (sets HttpContext.User)
app.UseAuthorization();      // 6th — Are you ALLOWED? (reads HttpContext.User)
app.UseRateLimiter();        // 7th — Are you making too many requests?
app.MapControllers();        // Last — Actually execute the controller action
```

**WHY does UseAuthentication come before UseAuthorization?**

Because `UseAuthentication` reads the token/cookie and sets `HttpContext.User`. `UseAuthorization` READS `HttpContext.User` to decide if you're allowed.

If you flip them, `HttpContext.User` is null when authorization runs — everyone gets rejected, or worse, everyone gets access.

**WHY does UseExceptionHandler come first?**

Because it wraps everything BELOW it in a try-catch. If it came last, exceptions from middleware above it would escape uncaught.

---

## 8. Short-Circuiting — A Critical Security Concept

```csharp
// This middleware completely STOPS the pipeline
// The request never reaches your controllers
public class IpBlockingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly HashSet<string> _blockedIps = new() { "192.168.1.100", "10.0.0.1" };

    public IpBlockingMiddleware(RequestDelegate next) => _next = next;

    public async Task InvokeAsync(HttpContext context)
    {
        var clientIp = context.Connection.RemoteIpAddress?.ToString();

        if (_blockedIps.Contains(clientIp ?? ""))
        {
            // ⚡ SHORT-CIRCUIT: We write response and RETURN
            // _next is NEVER called
            // Pipeline STOPS here
            context.Response.StatusCode = StatusCodes.Status403Forbidden;
            await context.Response.WriteAsync("Access Denied");
            return;  // ← This is the short-circuit
        }

        // Only reaches here for non-blocked IPs
        await _next(context);
    }
}
```

**What happens in memory when you short-circuit?**

The call stack unwinds immediately. All middleware ABOVE this one in the stack resumes their "after" code (code after `await _next(context)`). But nothing BELOW this middleware ever executes.

---

## 9. Bad Practice vs Good Practice

```csharp
// ❌ BAD: Putting business logic in middleware
public async Task InvokeAsync(HttpContext context)
{
    // DON'T DO THIS — middleware is for cross-cutting concerns only
    var userId = context.User.FindFirst("sub")?.Value;
    var user = await _dbContext.Users.FindAsync(userId);
    user.LastSeen = DateTime.UtcNow;
    await _dbContext.SaveChangesAsync();
    
    await _next(context);
}

// ✅ GOOD: Middleware for cross-cutting concerns only
// Logging, Exception Handling, Auth, Rate Limiting, Correlation IDs
public async Task InvokeAsync(HttpContext context)
{
    // Assign a unique ID to track this request across logs
    var correlationId = Guid.NewGuid().ToString();
    context.Items["CorrelationId"] = correlationId;
    context.Response.Headers.Append("X-Correlation-Id", correlationId);
    
    await _next(context);
}
```

---

## 10. What Are Cross-Cutting Concerns?

These are things that **every request needs**, regardless of what the request is doing:

|Concern|Middleware|
|---|---|
|Logging all requests|Custom logging middleware|
|Catching all exceptions|`UseExceptionHandler`|
|Verifying HTTPS|`UseHttpsRedirection`|
|Who is the user?|`UseAuthentication`|
|Can they access this?|`UseAuthorization`|
|Compress responses|`UseResponseCompression`|
|Track request timing|Custom timing middleware|
|Rate limiting|`UseRateLimiter`|
|CORS headers|`UseCors`|

**None of these belong in your controllers.** If you put them in controllers, you'd repeat the code everywhere — and one day forget it somewhere.

---

## Interview Questions You Must Know

1. **What is the difference between `app.Use()` and `app.Run()`?**
2. **What happens if you don't call `await _next(context)` in middleware?**
3. **Why can't you inject a Scoped service in the middleware constructor?**
4. **What is the difference between `app.Map()` and `app.MapWhen()`?**
5. **If exception handling middleware is registered LAST, what happens?**
6. **What is `HttpContext.Items` and why is it useful in middleware?**
7. **Can middleware access the response body? What are the limitations?**

---

## 🔥 Cross Questions — Answer These Before Moving On

**Question 1:** You have this pipeline:

```
MW_A → MW_B → MW_C → Controller
```

MW_B throws an exception. What happens? Which middleware "sees" this exception? Trace the exact execution path.

**Question 2:** A developer writes this:

```csharp
public class BadMiddleware
{
    private readonly IOrderService _orderService; // Scoped service

    public BadMiddleware(RequestDelegate next, IOrderService orderService)
    {
        _next = next;
        _orderService = orderService; // ← Is this correct?
    }
}
```

What is the exact problem? What error will you get at runtime? WHY does this error occur?

**Question 3:** What will this print and in what order?

```csharp
app.Use(async (ctx, next) => {
    Console.WriteLine("A-before");
    await next();
    Console.WriteLine("A-after");
});

app.Use(async (ctx, next) => {
    Console.WriteLine("B-before");
    await next();
    Console.WriteLine("B-after");
});

app.Run(async ctx => {
    Console.WriteLine("C - terminal");
    await ctx.Response.WriteAsync("Done");
});
```

**Question 4 — Scenario:** Your app is receiving 10,000 requests/second. Your logging middleware is doing `await File.WriteAllTextAsync(...)` for every single request to write to a log file. What problems will this cause? How would you fix it?

**Question 5 — Predict the behavior:**

```csharp
app.Use(async (ctx, next) => {
    await next();
    ctx.Response.Headers.Append("X-Custom", "value"); // AFTER next()
});
```

On a production server, this sometimes works and sometimes throws an exception. Why? What is the exact condition that causes it to fail?

---

## 🎯 Mini Assignment

Build a middleware called `RequestTimingMiddleware` that:

1. Records the time when the request arrived
2. Passes to next middleware
3. After response is ready, calculates elapsed milliseconds
4. Adds a response header: `X-Response-Time-Ms: 42`
5. Logs: `GET /api/users → 200 in 42ms`
6. If elapsed > 2000ms → logs a WARNING instead of INFO

Write it using the full class-based pattern with an extension method.

Then tell me: **Should this middleware be registered first or last? Why?**