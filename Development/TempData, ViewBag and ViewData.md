## 1. Simple Explanation — What Are These?

When a controller processes a request, it needs to **pass data to a View** (or sometimes to another action). These three mechanisms are different ways to do that.

Think of them as **three different types of envelopes** you use to send a letter:

|Mechanism|Scope|Type Safety|Storage|
|---|---|---|---|
|`ViewData`|Current request only|❌ No (object)|Dictionary|
|`ViewBag`|Current request only|❌ No (dynamic)|Wraps ViewData|
|`TempData`|Current + ONE next request|❌ No (object)|Session/Cookie|

---

## 2. Real-World Analogy

Imagine you're a **restaurant manager** (Controller) handing information to a **waiter** (View):

- **ViewData** = A sticky note 📝 — You write "Table 5 wants extra sauce" on paper and give it to the waiter. It only works for that one shift (request). Tomorrow it's gone.
    
- **ViewBag** = A whiteboard 🪧 — Same information, but now written on a whiteboard. Easier to read (dynamic), but still only lasts that one shift.
    
- **TempData** = A locker 🔐 — You lock a message inside a locker. The waiter can read it **today OR tomorrow** but after they read it, it's gone forever (unless they say "keep it").
    

---

## 3. Technical Deep Dive

### ViewData — The Foundation

```csharp
// In Controller
public IActionResult Index()
{
    // ViewData is of type ViewDataDictionary
    // which inherits from IDictionary<string, object>
    ViewData["PageTitle"] = "Welcome to My App";
    ViewData["UserCount"] = 42;
    ViewData["CurrentUser"] = new User { Name = "Ali" };
    
    return View();
}
```

```html
<!-- In View (.cshtml) -->
<h1>@ViewData["PageTitle"]</h1>
<!-- WHY the cast? Because ViewData stores as 'object' -->
<p>Users: @((int)ViewData["UserCount"])</p>

@{
    // You must cast to get the real type back
    var user = ViewData["CurrentUser"] as User;
}
<p>Hello @user?.Name</p>
```

**WHY is it stored as `object`?** Because `ViewDataDictionary` was designed to hold ANY type of data. The dictionary is `IDictionary<string, object>`. This means everything is **boxed** into `object` when stored, and must be **unboxed/cast** when retrieved. This has a performance cost (boxing/unboxing) and a safety cost (no compile-time checking).

---

### ViewBag — The Syntactic Sugar

```csharp
// In Controller
public IActionResult Index()
{
    // ViewBag is a dynamic wrapper OVER ViewData
    // They share the SAME underlying dictionary!
    ViewBag.PageTitle = "Welcome";
    ViewBag.UserCount = 42;
    
    // THIS IS THE SAME AS:
    ViewData["PageTitle"] = "Welcome";
    ViewData["UserCount"] = 42;
    
    return View();
}
```

```html
<!-- In View -->
<!-- No cast needed because it's dynamic -->
<h1>@ViewBag.PageTitle</h1>
<p>Users: @ViewBag.UserCount</p>
```

**WHY does ViewBag exist if ViewData already exists?**

ViewBag was introduced in MVC 3 to provide a **cleaner dot-notation syntax**. Compare:

```csharp
// Ugly — string key, requires cast
var title = (string)ViewData["PageTitle"];

// Clean — property-like access, no cast
var title = ViewBag.PageTitle;
```

But here's the critical insight — **ViewBag is literally just a dynamic wrapper over ViewData**. Look at the internal implementation:

```csharp
// Simplified internal implementation of ViewBag
public dynamic ViewBag
{
    get
    {
        // DynamicViewData wraps the ViewData dictionary
        return new DynamicViewData(() => ViewData);
    }
}
```

So when you do `ViewBag.Title = "Hello"`, internally it calls `ViewData["Title"] = "Hello"`.

**Proof:**

```csharp
ViewBag.Message = "Hello from ViewBag";
Console.WriteLine(ViewData["Message"]); // Prints: Hello from ViewBag

ViewData["Name"] = "Ali";
Console.WriteLine(ViewBag.Name); // Prints: Ali
```

They are **the same storage**, two different access styles.

---

### The `dynamic` Keyword — Why It's Dangerous

`ViewBag` uses `dynamic`, which means:

- No IntelliSense ❌
- No compile-time error ❌
- Runtime exceptions instead ❌

```csharp
ViewBag.UsreName = "Ali"; // Typo! But NO compile error
```

```html
@ViewBag.UserName <!-- This returns null silently. Bug! -->
```

You won't catch this until your app runs. This is why senior engineers **avoid ViewBag** in production code.

---

### TempData — Surviving Across Requests

```csharp
// Action 1 — Sets TempData
[HttpPost]
public IActionResult CreateUser(UserDto dto)
{
    // Save user to DB...
    _userService.Create(dto);
    
    // Store success message in TempData
    TempData["SuccessMessage"] = "User created successfully!";
    
    // REDIRECT — this ends the current request
    return RedirectToAction("Index");
}

// Action 2 — Reads TempData (after redirect)
public IActionResult Index()
{
    // TempData["SuccessMessage"] is still available here!
    // But after this request, it's GONE unless you call Keep()
    return View();
}
```

```html
<!-- In Index.cshtml -->
@if (TempData["SuccessMessage"] != null)
{
    <div class="alert alert-success">
        @TempData["SuccessMessage"]
    </div>
}
```

---

## 4. Internal Working — How TempData Works

This is where it gets really interesting. TempData has to **survive across HTTP requests**, which are stateless by design. How does it do this?

### The PRG Pattern (Post-Redirect-Get)

```
Browser                    Server
  |                          |
  |-- POST /CreateUser ----→ |  (Request 1)
  |                          |  TempData["msg"] = "Success"
  |← 302 Redirect to /Index--|  Data stored in Session/Cookie
  |                          |
  |-- GET /Index -----------→|  (Request 2)
  |                          |  TempData reads "msg" from storage
  |                          |  marks it for deletion after this request
  |← 200 OK with message ----|
```

### ITempDataProvider — The Plug

TempData doesn't store data itself. It uses an **`ITempDataProvider`** interface. By default, ASP.NET Core uses:
```csharp
// Default: CookieTempDataProvider (stores in cookie)
// Alternative: SessionStateTempDataProvider (stores in session)

// In Program.cs — switching to Session-based TempData
builder.Services.AddControllersWithViews()
    .AddSessionStateTempDataProvider();

builder.Services.AddSession(); // Required for session
```

**Cookie-based (default):**

- Data is serialized to JSON and stored in a browser cookie
- Cookie name: `.AspNetCore.Mvc.CookieTempDataProvider`
- Encrypted using ASP.NET Core Data Protection
- Limited by cookie size (~4KB)
- No server-side storage needed

**Session-based:**

- Data stored on server (in memory, Redis, SQL, etc.)
- Session ID sent via cookie
- No size limitation (practically)
- Requires distributed cache for multi-server scenarios

### TempData Read = Mark For Deletion

```csharp
// Internal lifecycle of TempData:

// Request 1: Write
TempData["key"] = "value";
// → Serialized and saved to cookie/session at end of request

// Request 2: Read
var val = TempData["key"]; 
// → Loaded from cookie/session
// → Key is now MARKED for deletion

// End of Request 2:
// → Marked keys are DELETED from storage
// → Cookie/session is updated
```

### Keep() and Peek() — Advanced TempData

```csharp
// PEEK — read WITHOUT marking for deletion
var msg = TempData.Peek("SuccessMessage"); 
// Data survives to the next request

// KEEP — after reading, un-mark for deletion
var msg = TempData["SuccessMessage"];
TempData.Keep("SuccessMessage"); 
// Data will survive one MORE request

// KEEP ALL
TempData.Keep(); // Keeps everything
```

**When would you use `Keep()`?** Imagine a wizard/multi-step form where you need the message to survive 3 steps, not just 1. You'd call `Keep()` at each step.

---

## 5. Why Industry Uses This Approach

### ViewData/ViewBag — The Problem They Solve

In MVC, the **controller and view are separate classes**. They can't directly call each other's methods. The view is rendered AFTER the controller finishes. So you need a **shared bag of data** that both sides can access during the same request.

ASP.NET Core makes `ViewData` available in both:

- `Controller` (via `this.ViewData`)
- `RazorPage`/View (via `ViewData` directly accessible in `.cshtml`)

They share the same dictionary because the **controller passes its `ViewData` to the View when rendering**.

### TempData — The Problem It Solves

HTTP is **stateless**. After `return RedirectToAction(...)`, the server has no memory of the previous request. But users expect to see "Record saved!" after being redirected. TempData bridges this stateless gap.

---

## 6. Bad Practices vs Good Practices

### ❌ BAD — Overusing ViewBag for Complex Data

```csharp
// Controller
public IActionResult Index()
{
    ViewBag.Users = _db.Users.ToList();
    ViewBag.TotalCount = _db.Users.Count();
    ViewBag.AdminCount = _db.Users.Where(u => u.IsAdmin).Count();
    ViewBag.PageTitle = "User Management";
    ViewBag.CurrentPage = 1;
    // This becomes unmaintainable!
    return View();
}
```

```html
<!-- View — no type safety, no intellisense -->
@foreach (var user in ViewBag.Users) // What type is user??
{
    <tr><td>@user.Name</td></tr> // Runtime error if Name doesn't exist
}
```

### ✅ GOOD — Use a ViewModel

```csharp
// Create a dedicated ViewModel
public class UserIndexViewModel
{
    public List<UserDto> Users { get; set; }
    public int TotalCount { get; set; }
    public int AdminCount { get; set; }
    public string PageTitle { get; set; }
    public int CurrentPage { get; set; }
}

// Controller
public IActionResult Index()
{
    var viewModel = new UserIndexViewModel
    {
        Users = _userService.GetAll(),
        TotalCount = _userService.GetCount(),
        AdminCount = _userService.GetAdminCount(),
        PageTitle = "User Management",
        CurrentPage = 1
    };
    
    return View(viewModel); // Strongly typed!
}
```

```html
<!-- View — fully typed, IntelliSense works! -->
@model UserIndexViewModel

<h1>@Model.PageTitle</h1>
@foreach (var user in Model.Users) // Compiler knows the type!
{
    <tr><td>@user.Name</td></tr>
}
```

**ViewModel is ALWAYS preferred over ViewBag/ViewData for complex data.**

---

### ❌ BAD — Using TempData for Large Data

```csharp
// NEVER do this!
TempData["AllUsers"] = JsonSerializer.Serialize(_db.Users.ToList());
// Thousands of users in a cookie? Cookie size limit exceeded!
// Performance nightmare!
```

### ✅ GOOD — TempData for Small Status Messages Only

```csharp
// TempData is designed for THIS:
TempData["Success"] = "Order placed successfully!";
TempData["Error"] = "Payment failed. Try again.";
TempData["Info"] = "Email verification sent.";
```

---

### ❌ BAD — Reading TempData in Controller Logic

```csharp
public IActionResult Index()
{
    // Business logic should NOT depend on TempData
    if ((string)TempData["UserRole"] == "Admin")
    {
        // This is wrong — use Claims/Authorization instead
    }
}
```

---

## 7. Comparison Table — When to Use What

|Scenario|Use|
|---|---|
|Pass a list of users to view|ViewModel ✅|
|Pass a page title|ViewData/ViewBag (acceptable)|
|Show success message after redirect|TempData ✅|
|Pass complex form data between actions|TempData (small) or Session|
|Strongly-typed data to view|ViewModel always ✅|
|Flash messages (success/error/warning)|TempData ✅|

---

## 8. Memory and Performance Implications

### ViewData/ViewBag:
- Lives on the **heap** as a dictionary
- Garbage collected after request ends (very fast)
- Boxing/unboxing has a **minor CPU cost**
- No I/O involved — pure in-memory

### TempData (Cookie-based):
- Data must be **serialized to JSON**
- JSON is then **encrypted** (Data Protection API)
- Sent over the wire in HTTP headers on EVERY request
- Adds bytes to HTTP response/request
- Deserializes on every request even if you don't use it

### TempData (Session-based):
- Data stored in server memory (or Redis/SQL)
- Lookup by Session ID
- Network I/O if using distributed session
- More overhead but more storage space

---

## 9. Interview Questions

1. **What is the difference between ViewBag and ViewData?** → ViewBag is a dynamic wrapper over ViewData. They share the same underlying dictionary. ViewBag offers dot-notation access but no compile-time safety.
    
2. **Can ViewData survive a redirect?** → No. ViewData/ViewBag only survive for the current HTTP request. After `RedirectToAction`, a new request begins and ViewData is empty.
    
3. **How does TempData survive across requests?** → TempData uses `ITempDataProvider`. By default, it serializes data to a cookie using JSON + Data Protection encryption. Alternatively, session can be used.
    
4. **What is the PRG pattern and how does TempData fit in?** → Post-Redirect-Get pattern prevents form resubmission. TempData allows carrying a success/error message from the POST action to the GET action after redirect.
    
5. **What happens if you read TempData["key"] twice?** → Reading it once marks it for deletion. If you read it a second time in the same request, the value is still there (deletion happens at end of request). But the next request won't have it.
    
6. **What's the difference between `Peek()` and `Keep()` in TempData?** → `Peek()` is used to read from `TempData` without marking it for deletion. `Keep()` un-mark data for deletion after reading.
    
7. **Why is ViewBag considered a code smell in production applications?** 
Ans: The core problem: **no type safety.**
**Five quick reasons:**
- No IntelliSense — you're guessing property names
- No compile-time errors — typos become runtime bugs
- No refactoring support — rename a property and nothing updates
- Unreadable — what data does this view expect? You have to hunt through the controller
- Untestable — you can't assert on a dynamic object cleanly in unit tests
The fix is always a **ViewModel**.
    
---

## 10. Cross Questions For You 🎯

Now it's your turn. Think carefully before answering:

**Q1.** If `ViewBag` and `ViewData` share the same dictionary, what do you think happens if you set the same key in both?

```csharp
ViewData["Name"] = "Ali";
ViewBag.Name = "Rahul";
// What is ViewData["Name"] now? What is ViewBag.Name?
```

Ans: `ViewBag` is just a dynamic wrapper over `ViewData`. It means that `Viewata["Name"]` will just be overwritten.

**Q2.** Your colleague writes this code:

```csharp
[HttpPost]
public IActionResult SaveOrder(OrderDto dto)
{
    _orderService.Save(dto);
    ViewBag.Message = "Order saved!";
    return RedirectToAction("Confirmation");
}

public IActionResult Confirmation()
{
    // Will ViewBag.Message be here?
    return View();
}
```

Will `ViewBag.Message` be available in the Confirmation view? **Why or why not?** What would you use instead?

Ans: `ViewBag.Message` will not be available in the Confirmation view as `ViewBag` only persists data in current request. The correct approach would be to use `TempData` because it is designed for persisting data over redirects.

**Q3.** `TempData` uses cookies by default. What is a **security risk** this introduces? How does ASP.NET Core mitigate it?
Ans: The main security risk is that cookies are visible to the user in the browser. A malicious user can modify these to try and gain access to our application. ASP.Net core mitigates this by using the **Data Protection API** for the cookie provider. It encrypts the cookie content and signs the cookie to prevent tampering.

**Q4.** What do you think happens to TempData if the user opens the POST page in **two browser tabs** simultaneously?

**Q5.** A junior developer says:

> "I'll use TempData to store the entire shopping cart so it persists across pages."

What problems do you foresee? What would you recommend instead?
Ans: 

---

## Mini Assignment 🏋️

**Task:** Create a simple ASP.NET Core MVC flow:

1. A form that accepts a user's name
2. On POST: save the name (just in memory for now), add a TempData success message, redirect to a list page
3. On the list page: show the success message from TempData AND show the submitted name using a ViewModel
4. Make sure the success message disappears after ONE refresh of the list page

This will test your understanding of:

- PRG pattern
- TempData lifecycle
- ViewModel vs ViewBag distinction
- When TempData gets deleted
