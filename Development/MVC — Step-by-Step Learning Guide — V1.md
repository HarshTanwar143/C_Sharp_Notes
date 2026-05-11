## What is MVC?

MVC stands for **Model-View-Controller** — a pattern that separates your app into 3 layers:

|Layer|Responsibility|Files in this project|
|---|---|---|
|Model|Data + business rules|`Models/TodoTask.cs`, `Data/InMemoryTaskRepository.cs`|
|View|HTML rendering|`Views/**/*.cshtml`|
|Controller|HTTP handling + glue|`Controllers/*.cs`|

---

## Prerequisites

- [.NET 9 SDK](https://dotnet.microsoft.com/download) — check with `dotnet --version`
- Any editor: [Visual Studio 2022](https://visualstudio.microsoft.com/) or [VS Code](https://code.visualstudio.com/)

---

## Step 0: Initialize the Project

```bash
dotnet new mvc -n mvclearning
```

Above command creates a mvc project with name `mvclearning`.

---

## Step 1: Understand the Request Lifecycle

When you visit `https://localhost:5001/Tasks/Index`:

```
Browser  ──GET /Tasks/Index──►  Router
                                  │
                          matches: Tasks = TasksController
                                  │        Index = Index()
                                  ▼
                           TasksController.Index()
                                  │
                          calls: _repo.GetAll()
                                  │
                          calls: return View(tasks)
                                  │
                                  ▼
                          Views/Tasks/Index.cshtml
                          (renders HTML with @model data)
                                  │
                                  ▼
Browser  ◄──HTML response──────────
```

> **Study file:** `Program.cs` — look at `app.MapControllerRoute()`

---

## Step 2: Models (`Models/TodoTask.cs`)

A **Model** is just a C# class. It defines the shape of your data.

Key concepts:

- **Data Annotations** (`[Required]`, `[StringLength]`) — automatic validation
- **Enums** — for fixed sets like Priority levels

**Try this:** Add a `DueDate` property to `TodoTask`, then display it in the views.

```csharp
[DataType(DataType.Date)]
public DateTime? DueDate { get; set; }
```

Sample Model:
```csharp
using System.ComponentModel.DataAnnotations; 

namespace mvclearning.Models; 

public class TodoTask
{
    // Primary key, is required
    public int Id { get; set; }  

    // This field is required, and if it's not provided, the error message "Title is required." will be shown
    [Required(ErrorMessage = "Title is required.")]
    [StringLength(
        100,
        MinimumLength = 3,
        ErrorMessage = "Title cannot be longer than 100 characters."
    )]
    public string Title { get; set; } = string.Empty;  

    // It is not required, but if it is provided, it cannot be longer than 500 characters. If it is longer, the error message "Description cannot be longer than 500 characters." will be shown
    [StringLength(500, ErrorMessage = "Description cannot be longer than 500 characters.")]
    public string? Description { get; set; } = string.Empty;
  
    public bool IsCompleted { get; set; } = false;  

    public DateTime CreatedAt { get; set; } = DateTime.Now;  

    public Priority Priority { get; set; } = Priority.Medium;
} 

public enum Priority
{
    Low,
    Medium,
    High,
}
```

---

## Step 3: The Data Layer (`Data/InMemoryTaskRepository.cs`)

This replaces a real database for learning. It stores tasks in a `List<TodoTask>`.

In a real app you'd use **Entity Framework Core**:

```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
```

Key patterns here:

- `GetAll()` → SELECT *
- `GetById(id)` → SELECT WHERE id = ?
- `Add(task)` → INSERT
- `Update(task)` → UPDATE
- `Delete(id)` → DELETE

Sample data layer:
```csharp
using MvcLearning.Models;

namespace MvcLearning.Data;

/// <summary>
/// This acts as our "database". It stores tasks in memory.
/// Because it's registered as a Singleton in Program.cs,
/// the same instance is reused across all requests.
/// </summary>
public class InMemoryTaskRepository
{
    private readonly List<TodoTask> _tasks = new();
    private int _nextId = 1;

    // Seed with some starter tasks so the app isn't empty
    public InMemoryTaskRepository()
    {
        _tasks.AddRange(new[]
        {
            new TodoTask { Id = _nextId++, Title = "Learn what MVC means", Description = "Model-View-Controller pattern", Priority = Priority.High, IsCompleted = true },
            new TodoTask { Id = _nextId++, Title = "Understand Controllers", Description = "Controllers handle HTTP requests and return responses", Priority = Priority.High },
            new TodoTask { Id = _nextId++, Title = "Study Razor Views", Description = "Views render HTML using C# code", Priority = Priority.Medium },
            new TodoTask { Id = _nextId++, Title = "Build a Todo app", Description = "Put it all together!", Priority = Priority.Medium },
        });
    }

    // READ — get all tasks
    public List<TodoTask> GetAll() => _tasks.ToList();

    // READ — get a single task by ID
    public TodoTask? GetById(int id) => _tasks.FirstOrDefault(t => t.Id == id);

    // CREATE — add a new task
    public void Add(TodoTask task)
    {
        task.Id = _nextId++;
        task.CreatedAt = DateTime.Now;
        _tasks.Add(task);
    }

    // UPDATE — replace an existing task
    public bool Update(TodoTask updated)
    {
        var existing = _tasks.FirstOrDefault(t => t.Id == updated.Id);
        if (existing is null) return false;

        existing.Title = updated.Title;
        existing.Description = updated.Description;
        existing.Priority = updated.Priority;
        existing.IsCompleted = updated.IsCompleted;
        return true;
    }

    // DELETE — remove a task by ID
    public bool Delete(int id)
    {
        var task = _tasks.FirstOrDefault(t => t.Id == id);
        if (task is null) return false;
        _tasks.Remove(task);
        return true;
    }

    // TOGGLE — flip the IsCompleted flag
    public bool ToggleComplete(int id)
    {
        var task = _tasks.FirstOrDefault(t => t.Id == id);
        if (task is null) return false;
        task.IsCompleted = !task.IsCompleted;
        return true;
    }
}
```

---

## Step 4: Controllers (`Controllers/TasksController.cs`)

Controllers contain **Actions** — public methods that handle HTTP requests. Your MVC views will redirect actions (through forms) to these actions. These actions will be responsible for sending views and performing CRUD applications.
### Action Return Types

|Return|When to use|
|---|---|
|`View()`|Render an HTML page|
|`View(model)`|Render with data|
|`RedirectToAction("Index")`|Redirect after POST|
|`NotFound()`|404 response|
|`BadRequest()`|400 response|
|`Json(data)`|Return JSON (APIs)|

### The POST-Redirect-GET Pattern

```
GET  /Tasks/Create → show empty form
POST /Tasks/Create → validate → save → RedirectToAction("Index")
GET  /Tasks        → show updated list
```

This prevents duplicate submissions on browser refresh.

Sample controller:
```csharp
using Microsoft.AspNetCore.Mvc;
using MvcLearning.Data;
using MvcLearning.Models;

namespace MvcLearning.Controllers;

/// <summary>
/// TasksController handles all task-related pages and actions.
/// 
/// CRUD map:
///   Index   → List all tasks        (GET  /Tasks)
///   Details → Show one task         (GET  /Tasks/Details/5)
///   Create  → Show empty form       (GET  /Tasks/Create)
///   Create  → Save new task         (POST /Tasks/Create)
///   Edit    → Show pre-filled form  (GET  /Tasks/Edit/5)
///   Edit    → Save changes          (POST /Tasks/Edit/5)
///   Delete  → Confirm deletion      (GET  /Tasks/Delete/5)
///   Delete  → Actually delete       (POST /Tasks/Delete/5)
/// </summary>
public class TasksController : Controller
{
    // Dependency Injection: MVC automatically provides the repository
    // We declared it as Singleton in Program.cs
    private readonly InMemoryTaskRepository _repo;

    public TasksController(InMemoryTaskRepository repo)
    {
        _repo = repo;
    }

    // ─────────────────────────────────────────
    // READ: List all tasks
    // ─────────────────────────────────────────

    // GET /Tasks
    public IActionResult Index(string? filter)
    {
        var tasks = _repo.GetAll();

        // Apply optional filter from query string: /Tasks?filter=completed
        tasks = filter switch
        {
            "completed" => tasks.Where(t => t.IsCompleted).ToList(),
            "pending"   => tasks.Where(t => !t.IsCompleted).ToList(),
            "high"      => tasks.Where(t => t.Priority == Priority.High).ToList(),
            _           => tasks
        };

        // ViewBag passes the current filter back to the view
        ViewBag.CurrentFilter = filter ?? "all";
        ViewBag.TotalCount = _repo.GetAll().Count;
        ViewBag.CompletedCount = _repo.GetAll().Count(t => t.IsCompleted);

        // Pass the list of tasks as the "Model" to the view
        return View(tasks);
    }

    // ─────────────────────────────────────────
    // READ: Show a single task
    // ─────────────────────────────────────────

    // GET /Tasks/Details/5
    public IActionResult Details(int id)
    {
        var task = _repo.GetById(id);

        // If task not found, return a 404 Not Found response
        if (task is null)
            return NotFound();

        return View(task);
    }

    // ─────────────────────────────────────────
    // CREATE: Show the empty form
    // ─────────────────────────────────────────

    // GET /Tasks/Create
    public IActionResult Create()
    {
        // Return an empty form — no model needed
        return View();
    }

    // POST /Tasks/Create
    // [HttpPost] means this only handles POST requests (form submissions)
    // [ValidateAntiForgeryToken] protects against CSRF attacks
    [HttpPost]
    [ValidateAntiForgeryToken]
    public IActionResult Create(TodoTask task)
    {
        // ModelState.IsValid checks all [Required], [StringLength] etc. annotations
        if (!ModelState.IsValid)
        {
            // Validation failed — redisplay the form with error messages
            return View(task);
        }

        _repo.Add(task);

        // TempData survives one redirect — perfect for success messages
        TempData["Success"] = $"Task '{task.Title}' created successfully!";

        // RedirectToAction sends the browser to a different action
        return RedirectToAction(nameof(Index));
    }

    // ─────────────────────────────────────────
    // UPDATE: Show the pre-filled edit form
    // ─────────────────────────────────────────

    // GET /Tasks/Edit/5
    public IActionResult Edit(int id)
    {
        var task = _repo.GetById(id);
        if (task is null) return NotFound();

        // Pass the existing task so the form shows current values
        return View(task);
    }

    // POST /Tasks/Edit/5
    [HttpPost]
    [ValidateAntiForgeryToken]
    public IActionResult Edit(int id, TodoTask task)
    {
        // Ensure the route ID matches the form data (security check)
        if (id != task.Id) return BadRequest();

        if (!ModelState.IsValid)
            return View(task);

        var updated = _repo.Update(task);
        if (!updated) return NotFound();

        TempData["Success"] = $"Task '{task.Title}' updated!";
        return RedirectToAction(nameof(Index));
    }

    // ─────────────────────────────────────────
    // DELETE: Confirm page
    // ─────────────────────────────────────────

    // GET /Tasks/Delete/5
    public IActionResult Delete(int id)
    {
        var task = _repo.GetById(id);
        if (task is null) return NotFound();
        return View(task);
    }

    // POST /Tasks/Delete/5
    // ActionName("Delete") lets us keep the method name different
    [HttpPost, ActionName("Delete")]
    [ValidateAntiForgeryToken]
    public IActionResult DeleteConfirmed(int id)
    {
        var task = _repo.GetById(id);
        var title = task?.Title ?? "Task";
        _repo.Delete(id);
        TempData["Success"] = $"'{title}' deleted.";
        return RedirectToAction(nameof(Index));
    }

    // ─────────────────────────────────────────
    // EXTRA: Toggle complete via AJAX-friendly POST
    // ─────────────────────────────────────────

    // POST /Tasks/Toggle/5
    [HttpPost]
    [ValidateAntiForgeryToken]
    public IActionResult Toggle(int id)
    {
        _repo.ToggleComplete(id);
        return RedirectToAction(nameof(Index));
    }
}
```

### Dependency Injection

```csharp
// In Program.cs:
builder.Services.AddSingleton<InMemoryTaskRepository>();

// In controller constructor:
public TasksController(InMemoryTaskRepository repo)
{
    _repo = repo;  // MVC injects this automatically!
}
```

---

## Step 5: Views (`Views/**/*.cshtml`)

Views use **Razor syntax** — HTML + C# mixed together.

### Razor Cheatsheet

```cshtml
@Model.Title                    <!-- Output a value -->
@if (Model.IsCompleted) { }     <!-- Conditional -->
@foreach (var t in Model) { }   <!-- Loop -->
@{ var x = 5; }                 <!-- Code block -->
@(x + 1)                        <!-- Expression -->
```

### Tag Helpers (preferred over Html.*)

```cshtml
<!-- Link to an action -->
<a asp-controller="Tasks" asp-action="Edit" asp-route-id="@task.Id">Edit</a>

<!-- Form that posts to an action -->
<form asp-action="Create" method="post">

<!-- Input bound to a model property -->
<input asp-for="Title" />
<span asp-validation-for="Title"></span>  <!-- Shows error message -->
```

### Passing Data: Controller → View

|Method|Best for|
|---|---|
|`return View(model)`|Main data, strongly typed|
|`ViewBag.Foo = value`|Small extras, dynamic|
|`ViewData["Foo"]`|Same as ViewBag, string keys|
|`TempData["Foo"]`|One-time messages across redirects|

---

## Step 6: Layout & Shared Files

```
Views/
  _ViewStart.cshtml     ← runs before every view, sets Layout
  _ViewImports.cshtml   ← shared @using and @addTagHelper
  Shared/
    _Layout.cshtml      ← master HTML template (nav, footer, etc.)
```

`@RenderBody()` in the layout is where page content gets injected.

How the `_ViewImports.cshtml` will look like in our case:
```cshtml
@using mvclearning
@using mvclearning.Models
@addTagHelper *, Microsoft.AspNetCore.Mvc.TagHelpers
```

---
## Key MVC Conventions

ASP.NET Core MVC is **convention over configuration**:

- Controller class name ends in `Controller` → `TasksController`
- Action named `Index` in `TasksController` maps to `/Tasks/Index`
- View for `TasksController.Index()` lives at `Views/Tasks/Index.cshtml`
- `_Layout.cshtml` in `Views/Shared/` is auto-discovered

You don't configure any of this — it just works by naming things correctly.

The above part just uses basic MVC, models, in memory repository, controllers. But we can also use things like SQL server, DTOs, Automapper, Authentication and Authorization which have been discussed in the following note.
[MVC — Step-by-Step Learning Guide — V2](MVC%20—%20Step-by-Step%20Learning%20Guide%20—%20V2.md)


Tags:
#development 
#dotnet 