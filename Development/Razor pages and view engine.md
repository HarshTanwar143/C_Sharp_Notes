## 1. Simple Explanation — Start From Zero

Imagine you want to send a personalized letter to 1000 customers. You don't write 1000 letters manually. You create a **template**:

```
Dear {{CustomerName}},
Your order {{OrderId}} has been shipped.
```

Then you **merge** the template with data to produce the final letter.

**That's exactly what Razor does.**

Razor is a **templating engine** that lets you write HTML mixed with C# code, and at runtime, it **merges your template with data** to produce a final HTML string that is sent to the browser.

---

## 2. Real-World Analogy

Think of a **newspaper printing press**:

- The **metal plate** = Your Razor template (`.cshtml` file)
- The **ink** = Your C# data (the Model)
- The **printed newspaper** = The final HTML sent to the browser

The press doesn't print newspapers one by one by hand. It takes a **template plate**, presses it against **paper (data)**, and produces the **output** efficiently.

Razor Engine is that press. It takes `.cshtml` templates, injects C# data, and produces HTML.

---

## 3. Technical Deep Dive — What Is Razor?

Razor is not a language. Let that sink in.

> **Razor is a markup syntax / templating engine that allows you to embed C# inside HTML.**

It was introduced by **Scott Guthrie at Microsoft in 2011** as part of ASP.NET MVC 3 — specifically to replace the older, more verbose **ASPX (Web Forms)** syntax.

---

### The Three Pillars You Asked About

|Concept|What It Is|Lives In|
|---|---|---|
|**Razor Engine**|The compiler + runtime that processes `.cshtml` files|Microsoft.AspNetCore.Razor|
|**Razor Views**|Templates used in **MVC pattern**|Views/ folder|
|**Razor Pages**|A **page-based** model (alternative to MVC)|Pages/ folder|

These are **three different things** that share the same engine. Most juniors confuse them. Let me untangle them fully.

---

## 4. The Razor Engine — Internal Mechanics

### How Does Razor Work Internally?

When you write this in a `.cshtml` file:

```html
<h1>Hello, @Model.Name</h1>
```

You think this is just "HTML with some C#." But internally, something profound happens.

### Step 1 — Razor Parsing

The Razor parser reads your `.cshtml` file and **converts it into a C# class**.

Your template becomes something like this (simplified):

```csharp
public class YourView : RazorPage<CustomerModel>
{
    public override async Task ExecuteAsync()
    {
        WriteLiteral("<h1>Hello, ");
        Write(Model.Name);           // @Model.Name
        WriteLiteral("</h1>");
    }
}
```

Let that hit you. **Your `.cshtml` file IS a C# class.** It inherits from `RazorPage<TModel>`.

### Step 2 — Roslyn Compilation

This generated C# code is then **compiled by Roslyn** (the C# compiler) into a `.dll`.

In **development**, this happens on first request (or on file change). In **production**, this can be **precompiled** at publish time using:

```xml
<PropertyGroup>
  <RazorCompileOnPublish>true</RazorCompileOnPublish>
</PropertyGroup>
```

### Step 3 — Execution

When a request comes in:

1. The compiled class is instantiated
2. `ExecuteAsync()` is called
3. It writes HTML + data into a `TextWriter`
4. The resulting string is the HTTP response body

### Internal Pipeline Diagram

```
HTTP Request
     ↓
Routing → Controller/PageModel
     ↓
Selects View/Page (.cshtml)
     ↓
Razor Engine reads .cshtml
     ↓
Parses into C# class (RazorPage<T>)
     ↓
Roslyn compiles to IL
     ↓
JIT compiles to native code
     ↓
ExecuteAsync() runs → writes HTML
     ↓
HTML string → HTTP Response
```

---

## 5. Razor Views — The MVC Pattern

### What Are Razor Views?

Razor Views are the **V in MVC** (Model-View-Controller).

In the MVC pattern:

- **Model** → Your data (C# class)
- **View** → Your `.cshtml` template (Razor View)
- **Controller** → Orchestrates between Model and View

### File Structure

```
/Controllers
    CustomerController.cs
/Views
    /Customer
        Index.cshtml       ← View for Index action
        Details.cshtml     ← View for Details action
    /Shared
        _Layout.cshtml     ← Master layout (applied to all)
        _ViewStart.cshtml  ← Runs before every view
        _ViewImports.cshtml← Global using statements
```

### A Complete Example

**Model:**

```csharp
// Models/Customer.cs
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Email { get; set; }
    public List<Order> Orders { get; set; }
}
```

**Controller:**

```csharp
// Controllers/CustomerController.cs
public class CustomerController : Controller
{
    private readonly ICustomerService _customerService;

    // WHY private? → Encapsulation. No external class should touch this.
    // WHY readonly? → After constructor sets it, no accidental reassignment.
    // WHY interface? → Testability. Swap real service with mock in tests.
    // WHY constructor injection? → DI container manages lifetime, not you.
    public CustomerController(ICustomerService customerService)
    {
        _customerService = customerService;
    }

    // Action method — responds to GET /Customer/Details/5
    public async Task<IActionResult> Details(int id)
    {
        // Fetch data from service layer (not directly from DB — separation of concerns)
        var customer = await _customerService.GetByIdAsync(id);

        if (customer == null)
            return NotFound(); // Returns 404 HTTP response

        // Pass the model to the view
        // WHY return View(customer)?
        // → This tells MVC to find Views/Customer/Details.cshtml
        // → And pass customer as Model to that view
        return View(customer);
    }
}
```

**View (Views/Customer/Details.cshtml):**

```html
@* This is a Razor comment — it does NOT render in HTML output *@

@* @model declares the type of Model this view expects *@
@* WHY declare model type? → Strong typing. IntelliSense. Compile-time errors. *@
@model Customer

@* Sets the HTML <title> via the Layout *@
@{
    ViewData["Title"] = "Customer Details";
    // @{ } = C# code block. No output. Just logic.
}

<div class="container">
    @* @ symbol = switch from HTML mode to C# expression mode *@
    <h1>@Model.Name</h1>

    @* WHY not <h1>Model.Name</h1>? → Without @, it's literal text, not C# *@

    <p>Email: @Model.Email</p>

    @* Conditional rendering *@
    @if (Model.Orders.Any())
    {
        <h3>Orders</h3>
        <ul>
            @* Loop — renders one <li> per order *@
            @foreach (var order in Model.Orders)
            {
                <li>Order #@order.Id — @order.Total.ToString("C")</li>
            }
        </ul>
    }
    else
    {
        <p>No orders found.</p>
    }
</div>
```

---

### The `_Layout.cshtml` — Master Template

Every page on your site probably shares a header, footer, navbar. You don't want to repeat that in every view.

```html
@* Views/Shared/_Layout.cshtml *@
<!DOCTYPE html>
<html>
<head>
    <title>@ViewData["Title"] - MyApp</title>
    @* RenderSection = optional section that child views can fill *@
    @RenderSection("Styles", required: false)
</head>
<body>
    <nav><!-- shared navbar --></nav>

    <main>
        @* This is where child view content gets injected *@
        @RenderBody()
    </main>

    <footer>© 2026 MyApp</footer>

    @RenderSection("Scripts", required: false)
</body>
</html>
```

**`_ViewStart.cshtml`** runs before every view and applies the layout:

```csharp
@{
    Layout = "_Layout"; // All views use _Layout.cshtml by default
}
```

**Why this matters:**

- Without layout, every view would repeat `<html>`, `<head>`, etc.
- This is the **Template Method pattern** — skeleton defined in layout, specifics in child views
- DRY principle (Don't Repeat Yourself)

---

### `_ViewImports.cshtml` — Global Namespace Imports

```csharp
@using MyApp.Models
@using MyApp.ViewModels
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers

// WHY @addTagHelper?
// Tag Helpers are C# classes that enhance HTML elements
// <asp:for>, <asp:action>, <form asp-action="Submit"> etc.
// This single line enables ALL built-in Tag Helpers globally
```

Without this, you'd need `@using` at the top of every single view file.

---

## 6. Razor Pages — The Page-Based Model

### WHY Were Razor Pages Created?

Here's the history that most tutorials skip.

In classic ASP.NET MVC, for a simple "Contact Us" page, you needed:

- A `ContactController.cs`
- A `ContactViewModel.cs`
- A `Views/Contact/Index.cshtml`

Three files for one page. The controller had a `GET` method and a `POST` method.

**Microsoft noticed:** Developers were creating Controllers with one or two actions just to serve a single page. The Controller was almost empty boilerplate.

In 2017, with ASP.NET Core 2.0, they introduced **Razor Pages** — which collapses the Controller + View into a single cohesive unit.

> Razor Pages follows the **MVVM or Page Controller pattern**, not the MVC pattern.

### File Structure

```
/Pages
    Index.cshtml           ← Page
    Index.cshtml.cs        ← PageModel (code-behind)
    /Customers
        List.cshtml
        List.cshtml.cs
        Details.cshtml
        Details.cshtml.cs
    /Shared
        _Layout.cshtml
```

### Complete Razor Pages Example

**`Pages/Customers/Details.cshtml.cs` — The PageModel:**

```csharp
// WHY namespace? → Organize code. Avoid naming conflicts.
namespace MyApp.Pages.Customers;

// PageModel is the base class for all Razor Pages code-behind
// It's analogous to a Controller but scoped to a single page
public class DetailsModel : PageModel
{
    private readonly ICustomerService _customerService;

    // Constructor injection — same DI principles as MVC Controller
    public DetailsModel(ICustomerService customerService)
    {
        _customerService = customerService;
    }

    // Public property — accessible from the .cshtml file via @Model.Customer
    // WHY public? → The Razor template needs to read it
    // WHY not [BindProperty]? → GET parameter, not form input
    public Customer Customer { get; private set; }

    // OnGetAsync — handles HTTP GET requests
    // Naming convention: On + HttpVerb + Async
    // WHY async? → Don't block thread while waiting for DB
    public async Task<IActionResult> OnGetAsync(int id)
    {
        Customer = await _customerService.GetByIdAsync(id);

        if (Customer == null)
            return NotFound();

        return Page(); // Render the associated .cshtml file
    }

    // [BindProperty] — model binding for POST data
    // WHY attribute? → Tell the framework: bind form fields to this property
    // WHY not just method parameter? → Complex objects from forms need binding
    [BindProperty]
    public CustomerEditDto EditInput { get; set; }

    // OnPostAsync — handles HTTP POST requests (form submissions)
    public async Task<IActionResult> OnPostAsync(int id)
    {
        // ModelState.IsValid checks all [Required], [MaxLength] etc. annotations
        if (!ModelState.IsValid)
            return Page(); // Re-render with validation errors

        await _customerService.UpdateAsync(id, EditInput);

        // PRG Pattern: Post-Redirect-Get
        // WHY redirect? → Prevent duplicate form submission on browser refresh
        return RedirectToPage("./Details", new { id });
    }
}
```

**`Pages/Customers/Details.cshtml`:**

```html
@page "{id:int}"
@* 
   @page — makes this a Razor Page (not just a view)
   "{id:int}" — route template. URL: /Customers/Details/5
   ":int" — route constraint. Only matches integers.
*@

@model DetailsModel
@* 
   @model here refers to the PageModel class, not a data model
   Different from MVC views where @model is the data class
*@

@{
    ViewData["Title"] = "Customer Details";
}

<h1>@Model.Customer.Name</h1>
<p>@Model.Customer.Email</p>

@* Tag Helper form — generates correct action URL automatically *@
<form method="post">
    @* Anti-forgery token — CSRF protection. Automatically injected. *@
    @Html.AntiForgeryToken()

    <input asp-for="EditInput.Name" />
    @* asp-for = Tag Helper that generates id, name, and validation attributes *@

    <span asp-validation-for="EditInput.Name"></span>
    @* Shows validation error message if Name is invalid *@

    <button type="submit">Save</button>
</form>
```

---

## 7. MVC Views vs Razor Pages — Head-to-Head

|Aspect|MVC Views|Razor Pages|
|---|---|---|
|**Pattern**|MVC (Separation)|Page Controller (Cohesion)|
|**Files per feature**|3+ (Controller, View, ViewModel)|2 (Page + PageModel)|
|**Best for**|Complex apps, APIs + Views|CRUD-heavy, page-focused apps|
|**Routing**|Attribute/Convention-based|File-system based|
|**Handler naming**|Action methods (any name)|`OnGet`, `OnPost`, `OnDelete`|
|**HTTP verb mapping**|`[HttpGet]`, `[HttpPost]`|Convention: `OnGet`, `OnPost`|
|**Testing**|Test Controller actions|Test PageModel handlers|
|**Learning curve**|Higher (understand MVC fully)|Lower (page-centric thinking)|

---

## 8. The `@` Symbol — How Razor Switches Context

This is something 90% of tutorials gloss over but is fundamental.

```
@expression         → Single C# expression, output to HTML
@{ ... }            → C# code block, no output
@if (...) { }       → Control flow
@foreach (...) { }  → Loops
@* ... *@           → Razor comment (not rendered)
@@                  → Literal @ character in output
@:                  → Switch to text output inside a code block
```

**Example of `@:` (important and overlooked):**

```csharp
@foreach (var item in items)
{
    @* Inside code block. Want to output plain text? Use @: *@
    @: Item number @item.Id
    @* Without @:, the compiler doesn't know if "Item number" is code or HTML *@
}
```

---

## 9. Tag Helpers vs HTML Helpers — Why Tag Helpers Won

**Old way (HTML Helpers — still works but considered legacy):**

```csharp
@Html.TextBoxFor(m => m.Name, new { @class = "form-control" })
@Html.ValidationMessageFor(m => m.Name)
@Html.ActionLink("Click Here", "ActionName", "ControllerName")
```

**Problems:**

- Looks like C# inside HTML — confusing to frontend devs
- HTML attributes passed as C# anonymous objects — ugly
- No IntelliSense for the HTML parts

**New way (Tag Helpers):**

```html
<input asp-for="Name" class="form-control" />
<span asp-validation-for="Name"></span>
<a asp-action="ActionName" asp-controller="ControllerName">Click Here</a>
```

**Why Tag Helpers are better:**

- Look like HTML — frontend devs can understand them
- Work with existing CSS frameworks naturally
- Full IntelliSense in VS/Rider
- Can be used by Razor-unaware designers
- Extensible — you can write custom Tag Helpers

---

## 10. Common Mistakes (Bad vs Good)

### ❌ BAD — Logic in the View

```html
@{
    // NEVER do this in a view
    var customers = DbContext.Customers
        .Where(c => c.IsActive)
        .OrderBy(c => c.Name)
        .ToList();
}
@foreach (var c in customers) { ... }
```

**Why bad?**

- Views should be dumb — only display data
- Tight coupling to EF/DB in the presentation layer
- Impossible to unit test
- Violates Single Responsibility Principle

### ✅ GOOD — Controller/PageModel prepares data

```csharp
// In Controller or PageModel
public async Task<IActionResult> OnGetAsync()
{
    Customers = await _customerService.GetActiveCustomersAsync();
    return Page();
}
```

```html
@* View just displays *@
@foreach (var c in Model.Customers) { ... }
```

---

### ❌ BAD — ViewBag overuse

```csharp
// Controller
ViewBag.CustomerName = "John";
ViewBag.OrderCount = 5;
ViewBag.IsVIP = true;
// ViewBag is dynamic — no compile-time safety
```

```html
@* View *@
<h1>@ViewBag.CustomerName</h1>
@* Typo in property name? → Runtime error, not compile error *@
```

### ✅ GOOD — Strongly typed ViewModel

```csharp
public class CustomerDetailsViewModel
{
    public string CustomerName { get; set; }
    public int OrderCount { get; set; }
    public bool IsVip { get; set; }
}
```

```html
@model CustomerDetailsViewModel
<h1>@Model.CustomerName</h1>
@* Typo → Compile error immediately *@
```

---

## 11. Partial Views and View Components

### Partial Views — Reusable View Fragments

```html
@* Reusable partial: Views/Shared/_CustomerCard.cshtml *@
@model Customer
<div class="card">
    <h3>@Model.Name</h3>
    <p>@Model.Email</p>
</div>
```

**Use it:**

```html
@* In a parent view *@
@await Html.PartialAsync("_CustomerCard", someCustomer)

@* Or with Tag Helper *@
<partial name="_CustomerCard" model="someCustomer" />
```

### View Components — More Powerful Partials

When your partial needs its **own service dependencies** (can't just pass model from parent):

```csharp
// ViewComponents/RecentOrdersViewComponent.cs
public class RecentOrdersViewComponent : ViewComponent
{
    private readonly IOrderService _orderService;

    public RecentOrdersViewComponent(IOrderService orderService)
    {
        _orderService = orderService;
    }

    public async Task<IViewComponentResult> InvokeAsync(int customerId)
    {
        var orders = await _orderService.GetRecentAsync(customerId, count: 5);
        return View(orders); // Views/Shared/Components/RecentOrders/Default.cshtml
    }
}
```

**Invoke it:**

```html
@await Component.InvokeAsync("RecentOrders", new { customerId = Model.Id })
```

**Why View Components over Partial Views?**

- View Components have their own DI
- They have their own lifecycle
- Can be unit tested independently
- Partial views only display — no logic, no services

---

## 12. Interview Questions This Topic Generates

1. What is the difference between `@model` in MVC Views vs Razor Pages?
2. What does `@page` directive do and why is it required in Razor Pages?
3. How does the Razor Engine compile `.cshtml` files internally?
4. What's the difference between `@{}` and `@expression`?
5. Why are Tag Helpers preferred over HTML Helpers?
6. What is `_ViewStart.cshtml` and when does it execute?
7. Can you use Razor Pages and MVC in the same application?
8. What is `[BindProperty]` and what does `SupportsGet = true` mean?
9. What is the PRG (Post-Redirect-Get) pattern and why use it?
10. What is the difference between a Partial View and a View Component?

---

## Cross-Questions For You — Answer These Before Moving On

**Question 1 — Conceptual:** If the Razor Engine converts `.cshtml` files into C# classes that inherit from `RazorPage<T>`, what do you think happens to **performance** on the first request after deployment? And how would you fix it in production?

**Question 2 — Scenario:** You have a sidebar that shows "Latest 5 orders for the logged-in user." This sidebar appears on 20 different pages. How would you implement this? Would you use a Partial View or a View Component? Justify your answer technically.

**Question 3 — "What happens if" challenge:**

```csharp
public class DetailsModel : PageModel
{
    public Customer Customer { get; set; }  // No 'private set'

    [BindProperty]
    public Customer Customer { get; set; }  // What if you made Customer a BindProperty?
}
```

What would happen if you decorated the `Customer` property with `[BindProperty]`? What security risk would this introduce?

**Question 4 — Architecture:** A junior developer says: _"Razor Pages is just MVC with fewer files, right? They're basically the same thing."_

Is this statement correct? What would you say to correct or clarify this?

**Question 5 — Deep Razor:** What is `@Html.AntiForgeryToken()` and why does Razor Pages inject it **automatically** in forms using Tag Helpers but you had to call it **manually** in older MVC? What security attack does it prevent?

---

## Mini Assignment

Build mentally (or actually in code) a **Razor Pages CRUD** for a `Product` entity:

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public int StockQuantity { get; set; }
}
```

You need:

1. `Pages/Products/List.cshtml` + `List.cshtml.cs` — Show all products
2. `Pages/Products/Create.cshtml` + `Create.cshtml.cs` — Create form with validation
3. `Pages/Products/Edit.cshtml` + `Edit.cshtml.cs` — Edit form

**For each page, tell me:**

- What properties go on the PageModel?
- Which ones get `[BindProperty]`?
- What do `OnGetAsync` and `OnPostAsync` do?
- How do you prevent a page refresh from re-submitting the form?
