This is the advanced version of [MVC — Step-by-Step Learning Guide — V1](MVC%20—%20Step-by-Step%20Learning%20Guide%20—%20V1.md), the previous one builds the base of the topic and this one will go in depth towards topics like DTOs, EF core (used to connect to SQL server instead of using a in memory repository), Mappers, Authentication, Authorization.

## How to think and execute the project:
### Phase 1: Plan on paper first (before touching the keyboard)
Ask yourself three questions:

**1. What data do I need to store?** Write down your entities. For this app: a User and a Task. A Task belongs to a User. Draw that relationship — one User has many Tasks.

**2. What does each page need to show or receive?** List your pages: list of tasks, task form, login form, register form. For each page, write down what fields it needs. This becomes your DTO list.

**3. What are my URLs and what happens at each one?**
```text
GET  /Tasks          → show list
GET  /Tasks/Create   → show form
POST /Tasks/Create   → save task
GET  /Account/Login  → show login
POST /Account/Login  → verify credentials
```

The above routes become our controller actions.

### Phase 2: The exact file creation order

**Step 1 — Create the `.csproj` and install packages**
Install EF Core, Identity, and AutoMapper before writing a single line of business code.
```bash
dotnet add package Microsoft.EntityFrameworkCore.SqlServer 
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore 
dotnet add package AutoMapper.Extensions.Microsoft.DependencyInjection
```

**Step 2 — Write your models**
Models have zero dependencies on anything else in your project. They're just C# classes. Write them first.

Start with `ApplicationUser` because `TodoTask` depends on it (the foreign key). Always write the "parent" before the "child."
```text
Models/ApplicationUser.cs   ← first, no dependencies
Models/TodoTask.cs          ← second, has UserId → ApplicationUser
```
At this point, don't add validation attributes. Those go on DTOs, not models. Models just describe the database shape.

Our models:
```csharp
using Microsoft.AspNetCore.Identity;

namespace mvclearning.Models;

public class ApplicationUser : IdentityUser
{
    // Extra fields beyond what IdentityUser provides
    public string FullName { get; set; } = string.Empty;
    public DateTime JoinedAt { get; set; } = DateTime.UtcNow;

    // Navigation: one user → many tasks
    public ICollection<TodoTask> Tasks { get; set; } = new List<TodoTask>();
}
```

```csharp
namespace mvclearning.Models;

public class TodoTask
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public string? Description { get; set; }

    public bool IsCompleted { get; set; }
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    public Priority Priority { get; set; } = Priority.Medium;
    public DateTime? DueDate { get; set; }

    // ── Foreign key to the Identity User who owns this task ──
    // This is how we enforce per-user data isolation.
    public string UserId { get; set; } = string.Empty;

    public ApplicationUser User { get; set; } = null!;
}

public enum Priority
{
    Low,
    Medium,
    High,
}
```

**Step 3 — Write your DTOs**
Now think about what each operation needs. For every form, you need an input DTO. For every display page, you need an output DTO. Ask yourself: "What fields does this specific page need, and nothing more?"
```text
DTOs/TaskDtos.cs
  CreateTaskDto   ← what the create form submits
  EditTaskDto     ← what the edit form submits
  TaskListDto     ← what the list page displays
  TaskDetailDto   ← what the detail page displays
  RegisterDto     ← what the register form submits
  LoginDto        ← what the login form submits
```
This is where your `[Required]`, `[StringLength]` annotations go — because validation is a UI concern, not a database concern.

Some sample DTO:
```csharp
public class CreateTaskDto
{
    [Required(ErrorMessage = "Title is required.")]
    [StringLength(100, MinimumLength = 3, ErrorMessage = "Title must be 3–100 chars.")]
    public string Title { get; set; } = string.Empty;

    [StringLength(500, ErrorMessage = "Max 500 characters.")]
    public string? Description { get; set; }

    public Priority Priority { get; set; } = Priority.Medium;

    [DataType(DataType.Date)]
    [Display(Name = "Due Date")]
    public DateTime? DueDate { get; set; }
}

// ─────────────────────────────────────────────────────────────
// AUTH DTOs
// ─────────────────────────────────────────────────────────────

/// <summary>
/// Passed from the Register form to AccountController.
/// </summary>
public class RegisterDto
{
    [Required]
    [Display(Name = "Full Name")]
    public string FullName { get; set; } = string.Empty;

    [Required]
    [EmailAddress]
    public string Email { get; set; } = string.Empty;

    [Required]
    [StringLength(100, MinimumLength = 6, ErrorMessage = "Password must be at least 6 characters.")]
    [DataType(DataType.Password)]
    public string Password { get; set; } = string.Empty;

    [DataType(DataType.Password)]
    [Display(Name = "Confirm Password")]
    [Compare("Password", ErrorMessage = "Passwords do not match.")]
    public string ConfirmPassword { get; set; } = string.Empty;
}

/// <summary>
/// Passed from the Login form to AccountController.
/// </summary>
public class LoginDto
{
    [Required]
    [EmailAddress]
    public string Email { get; set; } = string.Empty;

    [Required]
    [DataType(DataType.Password)]
    public string Password { get; set; } = string.Empty;

    [Display(Name = "Remember me")]
    public bool RememberMe { get; set; }
}
```

**Step 4 — Create the DB Context**
Now you have Models, so you can write the DbContext. It just needs to know about your models and how they relate.
```csharp
// ============================================================
// Data/AppDbContext.cs
//
// The DATABASE CONTEXT — the bridge between C# and SQL Server.
//
// IdentityDbContext<ApplicationUser> gives us all Identity
// tables automatically (AspNetUsers, AspNetRoles, etc.)
// plus our custom tables (Tasks).
// ============================================================

using Microsoft.AspNetCore.Identity.EntityFrameworkCore;
using Microsoft.EntityFrameworkCore;
using mvclearning.Models;

namespace mvclearning.Data;

public class AppDbContext : IdentityDbContext<ApplicationUser>
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }

    // DbSet = one table per model class
    // EF will create a "Tasks" table in SQL Server
    public DbSet<TodoTask> Tasks { get; set; }

    protected override void OnModelCreating(ModelBuilder builder)
    {
        // IMPORTANT: must call base first — sets up Identity tables
        base.OnModelCreating(builder);

        builder.Entity<TodoTask>(entity =>
        {
            // Index on UserId for fast per-user queries
            entity.HasIndex(t => t.UserId);

            // Set max length for string columns (SQL VARCHAR size)
            entity.Property(t => t.Title).HasMaxLength(100).IsRequired();
            entity.Property(t => t.Description).HasMaxLength(500);

            // Foreign key: Task.UserId → AspNetUsers.Id
            // DeleteBehavior.Cascade: deleting a user deletes their tasks
            entity
                .HasOne(t => t.User)
                .WithMany(u => u.Tasks)
                .HasForeignKey(t => t.UserId)
                .OnDelete(DeleteBehavior.Cascade);
        });
    }
}
```

**Step 5 — Write the AutoMapper Profile**
You have both Models and DTOs now, so you can define how to convert between them.
For each pair, write one `CreateMap<Source, Destination>()`. If a property name on the DTO doesn't match the model (like `OwnerName` coming from `task.User.FullName`), use `.ForMember()`. If you don't need a field mapped, use `.Ignore()`.
```csharp
using AutoMapper;
using mvclearning.DTOs;
using mvclearning.Models;

namespace mvclearning.Data;

public class MappingProfile : Profile
{
    public MappingProfile()
    {
        // For every 'CreateMap<A, B>()', AutoMapper can convert A → B.

        // ── TodoTask → TaskListDto ──
        // ForMember() handles properties that don't match by name
        CreateMap<TodoTask, TaskListDto>()
            .ForMember(dest => dest.OwnerName, opt => opt.MapFrom(src => src.User.FullName));

        // ── TodoTask → TaskDetailDto ──
        CreateMap<TodoTask, TaskDetailDto>()
            .ForMember(dest => dest.OwnerName, opt => opt.MapFrom(src => src.User.FullName))
            .ForMember(dest => dest.OwnerEmail, opt => opt.MapFrom(src => src.User.Email));

        // ── TodoTask → EditTaskDto (for pre-filling the edit form) ──
        CreateMap<TodoTask, EditTaskDto>();

        // ── CreateTaskDto → TodoTask (when saving a new task) ──
        // UserId and CreatedAt are set manually in the controller
        CreateMap<CreateTaskDto, TodoTask>();

        // ── EditTaskDto → TodoTask (when updating) ──
        CreateMap<EditTaskDto, TodoTask>()
            .ForMember(dest => dest.UserId, opt => opt.Ignore())
            .ForMember(dest => dest.CreatedAt, opt => opt.Ignore())
            .ForMember(dest => dest.User, opt => opt.Ignore());
    }
}
```

**Step 6 — Configure `appsettings.json`**
Add your connection string here in `appsettings.json`.

**Step 7 — Wire everything up in `Program.cs`**
Now you have everything the DI container needs to know about. Register them all here. The order inside `Program.cs` matters:
```csharp
// Register DbContext first — Identity depends on it
builder.Services.AddDbContext<AppDbContext>(...);

// Register Identity — depends on DbContext
builder.Services.AddIdentity<ApplicationUser, IdentityRole>(...)
    .AddEntityFrameworkStores<AppDbContext>();

// Configure the login/logout redirect paths
builder.Services.ConfigureApplicationCookie(...);

// Register AutoMapper — scans your MappingProfile
builder.Services.AddAutoMapper(typeof(MappingProfile));

// Register your services
builder.Services.AddScoped<ITaskService, TaskService>();

// Register MVC last
builder.Services.AddControllersWithViews();
```

**Step 8 — Run the migration**
Before writing a single controller or view, make your database exist:
```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

If this fails, your DbContext or models have a bug.

**Step 9 — Write the Service Interface and Implementation**
Define the interface first — it's the contract. Then implement it. This order matters because the controller will depend on the interface, not the concrete class.
```
Services/ITaskService.cs   ← interface (what the service can do)
Services/TaskService.cs    ← implementation (how it does it)
```
Inside each service method, always filter by `userId`. This is where authorization logic for data ownership lives — not in the controller.

It is the Services job to talk to database, our controllers will just call these services.
```csharp
using AutoMapper;
using Microsoft.EntityFrameworkCore;
using mvclearning.Data;
using mvclearning.DTOs;
using mvclearning.Models;

namespace mvclearning.Services;

public interface ITaskService
{
    Task<List<TaskListDto>> GetUserTasksAsync(string userId, string? filter);
    Task<TaskDetailDto?> GetTaskDetailAsync(int id, string userId);
    Task<EditTaskDto?> GetTaskForEditAsync(int id, string userId);
    Task<bool> CreateTaskAsync(CreateTaskDto dto, string userId);
    Task<bool> UpdateTaskAsync(EditTaskDto dto, string userId);
    Task<bool> DeleteTaskAsync(int id, string userId);
    Task<bool> ToggleCompleteAsync(int id, string userId);
}

public class TaskService : ITaskService
{
	// Our way to access database
    private readonly AppDbContext _db;
    // Interface by Automapper
    private readonly IMapper _mapper;

    public TaskService(AppDbContext db, IMapper mapper)
    {
        _db = db;
        _mapper = mapper;
    }

    public async Task<List<TaskListDto>> GetUserTasksAsync(string userId, string? filter)
    {
        // Start with an IQueryable — nothing hits the DB yet
        var query = _db
            .Tasks.Include(t => t.User) // JOIN with AspNetUsers
            .Where(t => t.UserId == userId) // AUTHORIZATION: only this user's tasks
            .AsQueryable();

        // Apply filter
        query = filter switch
        {
            "completed" => query.Where(t => t.IsCompleted),
            "pending" => query.Where(t => !t.IsCompleted),
            "high" => query.Where(t => t.Priority == Priority.High),
            _ => query,
        };

        // Execute query, then map to DTOs
        var tasks = await query.OrderByDescending(t => t.CreatedAt).ToListAsync();
        return _mapper.Map<List<TaskListDto>>(tasks);
    }

    public async Task<TaskDetailDto?> GetTaskDetailAsync(int id, string userId)
    {
        var task = await _db
            .Tasks.Include(t => t.User)
            .FirstOrDefaultAsync(t => t.Id == id && t.UserId == userId);

        return task == null ? null : _mapper.Map<TaskDetailDto>(task);
    }

    public async Task<EditTaskDto?> GetTaskForEditAsync(int id, string userId)
    {
        var task = await _db.Tasks.FirstOrDefaultAsync(t => t.Id == id && t.UserId == userId);

        return task == null ? null : _mapper.Map<EditTaskDto>(task);
    }

    // -- Create --
    public async Task<bool> CreateTaskAsync(CreateTaskDto dto, string userId)
    {
        var task = _mapper.Map<TodoTask>(dto);

        task.UserId = userId;
        task.CreatedAt = DateTime.UtcNow;

        _db.Tasks.Add(task);
        await _db.SaveChangesAsync(); // executes INSERT
        return true;
    }

    public async Task<bool> UpdateTaskAsync(EditTaskDto dto, string userId)
    {
        var task = await _db.Tasks.FirstOrDefaultAsync(t => t.Id == dto.Id && t.UserId == userId);

        if (task is null)
            return false;

        _mapper.Map<TodoTask>(dto);

        await _db.SaveChangesAsync(); // executes UPDATE
        return true;
    }

    public async Task<bool> DeleteTaskAsync(int id, string userId)
    {
        var task = await _db.Tasks.FirstOrDefaultAsync(t => t.Id == id && t.UserId == userId);

        if (task is null)
            return false;

        _db.Tasks.Remove(task);
        await _db.SaveChangesAsync(); // executes DELETE
        return true;
    }

    public async Task<bool> ToggleCompleteAsync(int id, string userId)
    {
        var task = await _db.Tasks.FirstOrDefaultAsync(t => t.Id == id && t.UserId == userId);

        if (task is null)
            return false;

        task.IsCompleted = !task.IsCompleted;
        await _db.SaveChangesAsync();
        return true;
    }
}

```

**Step 10 — Write Controllers**
Now write controllers. They should be thin — receive input, call the service, return a view. If you find yourself writing DB queries in a controller, stop and move that to the service.
```
Controllers/AccountController.cs   ← Register, Login, Logout
Controllers/TasksController.cs     ← CRUD, all marked [Authorize]
Controllers/HomeController.cs      ← public home page
```

Write `AccountController` before `TasksController` because you need to be able to log in to test the task controller.

**`AccountController.cs`**
```csharp
// ============================================================
// Controllers/AccountController.cs
//
// Handles Authentication:
//   Register → creates a new user in AspNetUsers table
//   Login    → validates credentials, issues auth cookie
//   Logout   → deletes the auth cookie
//
// Key services injected:
//   UserManager<T>   — create/find/update users in the DB
//   SignInManager<T> — issue/remove authentication cookies
// ============================================================

using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Identity;
using Microsoft.AspNetCore.Mvc;
using mvclearning.DTOs;
using mvclearning.Models;

namespace mvclearning.Controllers;

public class AccountController : Controller
{
	// Provided by AspNetCore.Identity that provides high level APIs for managing user accounts.
    private readonly UserManager<ApplicationUser> _userManager;
    // Provided by AspNetCore.Identity that handles the actual process of authenticating users and managing their sign-in state.
    private readonly SignInManager<ApplicationUser> _signInManager;

    public AccountController(
        UserManager<ApplicationUser> userManager,
        SignInManager<ApplicationUser> signInManager
    )
    {
        _userManager = userManager;
        _signInManager = signInManager;
    }

    // ─────────────────────────────────────────
    // REGISTER
    // ─────────────────────────────────────────

    // GET /Account/Register
    [AllowAnonymous] // explicitly allow unauthenticated access
    public IActionResult Register()
    {
        // If already logged in, go home
        if (_signInManager.IsSignedIn(User))
            return RedirectToAction("Index", "Home");
        return View();
    }

    // POST /Account/Register
    [HttpPost]
    [AllowAnonymous]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Register(RegisterDto dto)
    {
        if (!ModelState.IsValid)
            return View(dto);

        // Map RegisterDto → ApplicationUser
        var user = new ApplicationUser
        {
            FullName = dto.FullName,
            UserName = dto.Email, // Identity uses UserName internally
            Email = dto.Email,
            JoinedAt = DateTime.UtcNow,
        };

        // CreateAsync hashes the password and INSERTs into AspNetUsers
        var result = await _userManager.CreateAsync(user, dto.Password);

        if (result.Succeeded)
        {
            // Automatically sign in after registration
            await _signInManager.SignInAsync(user, isPersistent: false);
            TempData["Success"] = $"Welcome, {user.FullName}! Account created.";
            return RedirectToAction("Index", "Tasks");
        }

        // Identity returns specific errors (e.g., "Email already taken")
        foreach (var error in result.Errors)
            ModelState.AddModelError(string.Empty, error.Description);

        return View(dto);
    }

    // ─────────────────────────────────────────
    // LOGIN
    // ─────────────────────────────────────────

    // GET /Account/Login
    [AllowAnonymous]
    public IActionResult Login(string? returnUrl = null)
    {
        if (_signInManager.IsSignedIn(User))
            return RedirectToAction("Index", "Home");
        ViewBag.ReturnUrl = returnUrl;
        return View();
    }

    // POST /Account/Login
    [HttpPost]
    [AllowAnonymous]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Login(LoginDto dto, string? returnUrl = null)
    {
        if (!ModelState.IsValid)
            return View(dto);

        // PasswordSignInAsync:
        //  - looks up user by email
        //  - verifies password hash
        //  - issues an encrypted auth cookie if valid
        var result = await _signInManager.PasswordSignInAsync(
            dto.Email,
            dto.Password,
            isPersistent: dto.RememberMe, // "Remember me" = longer cookie lifetime
            lockoutOnFailure: true
        ); // increment lockout counter on failure

        if (result.Succeeded)
        {
            TempData["Success"] = "Logged in successfully!";
            // returnUrl is where the user was trying to go before being redirected to login
            if (!string.IsNullOrEmpty(returnUrl) && Url.IsLocalUrl(returnUrl))
                return Redirect(returnUrl);
            return RedirectToAction("Index", "Tasks");
        }

        if (result.IsLockedOut)
        {
            ModelState.AddModelError(
                string.Empty,
                "Account locked due to too many failed attempts. Try again in 10 minutes."
            );
            return View(dto);
        }

        ModelState.AddModelError(string.Empty, "Invalid email or password.");
        return View(dto);
    }

    // ─────────────────────────────────────────
    // LOGOUT
    // ─────────────────────────────────────────

    // POST /Account/Logout (always POST to prevent CSRF logout attacks)
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Logout()
    {
        await _signInManager.SignOutAsync(); // clears the auth cookie
        TempData["Success"] = "You have been logged out.";
        return RedirectToAction("Index", "Home");
    }

    // GET /Account/AccessDenied
    public IActionResult AccessDenied() => View();
}
```

**`TasksController`**
```csharp
// ============================================================
// Controllers/TasksController.cs
//
// Changes from v1:
//  • [Authorize] — entire controller requires login
//  • Uses ITaskService (not repository directly)
//  • Accepts/returns DTOs (not raw models)
//  • Gets userId from User.FindFirstValue() (the logged-in user)
//  • Authorization enforced in service (userId filter on all queries)
// ============================================================

using System.Security.Claims;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using mvclearning.DTOs;
using mvclearning.Services;

namespace mvclearning.Controllers;

// [Authorize] on the class = every action requires authentication.
// If not logged in, ASP.NET redirects to /Account/Login automatically.
[Authorize]
public class TasksController : Controller
{
    private readonly ITaskService _taskService;

    public TasksController(ITaskService taskService)
    {
        _taskService = taskService;
    }

    // Helper: gets the current user's ID from their auth cookie claims
    private string GetUserId() =>
        User.FindFirstValue(ClaimTypes.NameIdentifier)
        ?? throw new InvalidOperationException("User not authenticated");

    // ─────────────────────────────────────────
    // READ: List all tasks
    // ─────────────────────────────────────────

    // GET /Tasks
    public async Task<IActionResult> Index(string? filter)
    {
        var userId = GetUserId();
        var tasks = await _taskService.GetUserTasksAsync(userId, filter);

        ViewBag.CurrentFilter = filter ?? "all";
        ViewBag.TotalCount = tasks.Count;
        ViewBag.CompletedCount = tasks.Count(t => t.IsCompleted);

        // View receives List<TaskListDto> — not the raw Model
        return View(tasks);
    }

    // ─────────────────────────────────────────
    // READ: Show a single task
    // ─────────────────────────────────────────

    // GET /Tasks/Details/5
    public async Task<IActionResult> Details(int id)
    {
        var dto = await _taskService.GetTaskDetailAsync(id, GetUserId());

        if (dto is null)
            return NotFound();

        // View receives TaskDetailDto — not the raw Model
        return View(dto);
    }

    // ─────────────────────────────────────────
    // CREATE: Show the empty form
    // ─────────────────────────────────────────

    // GET /Tasks/Create
    public IActionResult Create()
    {
        return View(new CreateTaskDto());
    }

    // POST /Tasks/Create
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Create(CreateTaskDto dto)
    {
        if (!ModelState.IsValid)
            return View(dto);

        await _taskService.CreateTaskAsync(dto, GetUserId());

        TempData["Success"] = $"Task '{dto.Title}' created successfully!";

        return RedirectToAction(nameof(Index));
    }

    // ─────────────────────────────────────────
    // UPDATE: Show the pre-filled edit form
    // ─────────────────────────────────────────

    // GET /Tasks/Edit/5
    public async Task<IActionResult> Edit(int id)
    {
        var dto = await _taskService.GetTaskForEditAsync(id, GetUserId());

        if (dto is null)
            return NotFound();

        return View(dto);
    }

    // POST /Tasks/Edit/5
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Edit(int id, EditTaskDto dto)
    {
        if (id != dto.Id)
            return BadRequest();

        if (!ModelState.IsValid)
            return View(dto);

        var updated = await _taskService.UpdateTaskAsync(dto, GetUserId());
        if (!updated)
            return NotFound();

        TempData["Success"] = $"Task '{dto.Title}' updated!";

        return RedirectToAction(nameof(Index));
    }

    // ─────────────────────────────────────────
    // DELETE: Confirm page
    // ─────────────────────────────────────────

    // GET /Tasks/Delete/5
    public async Task<IActionResult> Delete(int id)
    {
        var dto = await _taskService.GetTaskDetailAsync(id, GetUserId());

        if (dto is null)
            return NotFound();

        return View(dto);
    }

    [HttpPost, ActionName("Delete")]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> DeleteConfirmed(int id)
    {
        var task = await _taskService.GetTaskDetailAsync(id, GetUserId());
        var title = task?.Title ?? "Task";
        await _taskService.DeleteTaskAsync(id, GetUserId());

        TempData["Success"] = $"Task '{title}' deleted!";

        return RedirectToAction(nameof(Index));
    }

    // ─────────────────────────────────────────
    // EXTRA: Toggle complete
    // ─────────────────────────────────────────

    // POST /Tasks/Toggle/5
    [HttpPost]
    [ValidateAntiForgeryToken]
    public async Task<IActionResult> Toggle(int id)
    {
        await _taskService.ToggleCompleteAsync(id, GetUserId());
        return RedirectToAction(nameof(Index));
    }
}

```

**Step 11 — Write Views (last)**
Views are last because they just display what the controller sends. Each view starts with `@model YourDtoType` — the specific DTO for that page.
```
Views/_ViewImports.cshtml      ← add @using and @addTagHelper (once)
Views/_ViewStart.cshtml        ← set Layout = "_Layout" (once)
Views/Shared/_Layout.cshtml    ← shared nav/footer (write this early)
Views/Account/Register.cshtml  ← @model RegisterDto
Views/Account/Login.cshtml     ← @model LoginDto
Views/Tasks/Index.cshtml       ← @model List<TaskListDto>
Views/Tasks/Create.cshtml      ← @model CreateTaskDto
Views/Tasks/Edit.cshtml        ← @model EditTaskDto
Views/Tasks/Details.cshtml     ← @model TaskDetailDto
Views/Tasks/Delete.cshtml      ← @model TaskDetailDto
```

---

### How the files connect — the dependency map

```
appsettings.json
      ↓ (connection string read by)
Program.cs
      ↓ (registers)
      ├── AppDbContext  ←──────────── Models (ApplicationUser, TodoTask)
      │        ↑
      │   Identity tables (AspNetUsers etc.) live here
      │
      ├── AutoMapper  ←──────────── MappingProfile
      │        ↑                        ↑
      │      Models ←────────────── DTOs
      │
      └── ITaskService → TaskService
               ↓ (uses)
          AppDbContext + IMapper
               ↓
      TasksController
               ↓ (uses)
          ITaskService + DTOs
               ↓ (passes DTO to)
            Views  (typed with @model DTO)
```

---

### The rule for when something feels wrong

If your controller is more than ~40 lines, business logic leaked in — move it to the service. If your model has `[Required]` attributes, validation leaked in — move it to the DTO. If your view has C# logic beyond simple loops and conditionals, display logic leaked in — move it to a ViewModel property or a helper method. Each layer should only do its own job.


In all our constructors, how do our class get access to various variables like say `SignInManager`, `UserManager`, `ITaskService`, this is done by Dependency Injection.

Tags:
#development 
#dotnet 