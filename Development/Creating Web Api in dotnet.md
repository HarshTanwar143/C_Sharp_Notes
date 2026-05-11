You can create web api in dotnet which can serve requests to your frontend. Steps to do so:
#### 1. Setup

**1. Create the Project**
```bash
dotnet new webapi -n BlogApi
cd MyWebApi
```
`BlogApi` can be any name of your choice.
To open in Visual Studio, `start devenv BlogApi.csproj`, In VS code `code .`

By only doing the above you will have a basic api at some endpoint which you can get after `dotnet run`. 

**2. Install NuGet packages**
```bash
# Entity Framework Core + SQL Server
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools

# JWT Authentication
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer

# AutoMapper
dotnet add package AutoMapper.Extensions.Microsoft.DependencyInjection

# Swagger
dotnet add package Swashbuckle.AspNetCore

# Password hashing
dotnet add package BCrypt.Net-Next
```

Our folder structure should something like this:
```text
BlogApi/
├── Controllers/
│   ├── AuthController.cs
│   ├── PostsController.cs
│   └── CommentsController.cs
├── Models/          # EF Core entities
│   ├── User.cs
│   ├── Post.cs
│   └── Comment.cs
├── DTOs/            # Request/Response shapes
│   ├── Auth/
│   ├── Posts/
│   └── Comments/
├── Data/
│   └── AppDbContext.cs
├── Services/
│   ├── IAuthService.cs
│   └── AuthService.cs
├── Mappings/
│   └── MappingProfile.cs
├── Middleware/
│   └── ExceptionMiddleware.cs
└── Program.cs
```

#### 2. models and DB
If you want to serve some specific data you can create model in a `/Models` folder. Say we are creating `User.cs`
```csharp
// '/Models/User.cs'

using System.ComponentModel.DataAnnotations;

namespace BlogApi.Models;

public class User
{
    public int Id { get; set; }

    [Required]
    [MaxLength(50)]
    public string Username { get; set; } = string.Empty;

    [Required]
    [EmailAddress]
    public string Email { get; set; } = string.Empty;

    [Required]
    public string PasswordHash { get; set; } = string.Empty;

    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;

    // Navigation property for related posts
    public ICollection<Post> Posts { get; set; } = new List<Post>();
}
```
One thing to make sure is that `Post` should exist in your model folder for migration to work above. In case you have some circular dependency you can migrate them one by one.

 Create a context.
 ```csharp
 // '/Models/ProductContext.cs'
 
using BlogApi.Models;
using Microsoft.EntityFrameworkCore;

namespace BlogApi.Data;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options)
        : base(options) { }

    public DbSet<User> Users => Set<User>();

    public DbSet<Post> Posts => Set<Post>();

    public DbSet<Comment> Comments => Set<Comment>();
	
	// Below we are creating indexes
    protected override void OnModelCreating(ModelBuilder builder)
    {
        // Unique email
        builder.Entity<User>().HasIndex(user => user.Email).IsUnique();

        // Unique username
        builder.Entity<User>().HasIndex(user => user.Username).IsUnique();
    }
}
 ```

Add `ConnectionStrings__DefaultConnection` to your `appsettings.json` and run migrations.

After adding `ConnectionStrings__DefaultConnection`, we can register `DbContext` to our `Program.cs`.
```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlServer(
        builder.Configuration
               .GetConnectionString("DefaultConnection")));
```

Create and apply migrations:
```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

#### 3. DTOs & AutoMapper
DTOs (Data Transfer Objects) are simple objects used to pass data between the client and the server or between different layers of an application. They act as a 'contract' that defines exactly what data is being sent over the network, separate from the complex business logic or database models. In short you don't have to expose your entire models to the client and your client can just interact with DTOs only.

Sample DTOs for Auth:
```csharp
using System.ComponentModel.DataAnnotations;

namespace BlogApi.DTOs;

public class RegisterDto
{
    [Required]
    [MaxLength(50)]
    public string Username { get; set; } = string.Empty;

    [Required]
    [EmailAddress]
    public string Email { get; set; } = string.Empty;

    [Required]
    [MinLength(6)]
    public string Password { get; set; } = "";
}

// DTOs/Auth/LoginDto.cs
public class LoginDto
{
    [Required]
    public string Email { get; set; } = "";

    [Required]
    public string Password { get; set; } = "";
}

// DTOs/Auth/AuthResponseDto.cs
public class AuthResponseDto
{
    public string Token { get; set; } = "";
    public string Username { get; set; } = "";
    public string Email { get; set; } = "";
}
```
Note that you have applied Validations on DTO.

Create mapping or `automapper`. `AutoMapper` will basically convert one dto to another dto or to a model and vice-versa. In below conversions, `CreateMap<A, B>()` `A` is converted to `B`.

```csharp
using AutoMapper;
using BlogApi.DTOs;
using BlogApi.Models;
using Microsoft.Data.SqlClient;

namespace BlogApi.Mappings;

// 'Profile' is an AutoMapper class used to define mappings.
public class MappingProfile : Profile
{
    public MappingProfile()
    {
        // User maps
        CreateMap<RegisterDto, User>();
        CreateMap<User, AuthResponseDto>();

        // Post maps
        CreateMap<CreatePostDto, Post>();
        CreateMap<Post, PostResponseDto>()
            .ForMember(dest => dest.AuthorUsername, opt => opt.MapFrom(src => src.User.Username))
            .ForMember(dest => dest.CommentCount, opt => opt.MapFrom(src => src.Comments.Count));

        CreateMap<CreateCommentDto, Comment>();
        CreateMap<Comment, CommentResponseDto>()
            .ForMember(dest => dest.AuthorUsername, opt => opt.MapFrom(src => src.User.Username));
    }
}

```

#### 4. CRUD controllers
You can scaffold your controller using steps that are according to either visual studio (`/controllers -> right click -> new Scaffold item`) or in vs code use below commands:
```bash
dotnet add package Microsoft.VisualStudio.Web.CodeGeneration.Design
dotnet add package Microsoft.EntityFrameworkCore.Design
dotnet add package Microsoft.EntityFrameworkCore.SqlServer
dotnet add package Microsoft.EntityFrameworkCore.Tools
dotnet tool uninstall -g dotnet-aspnet-codegenerator
dotnet tool install -g dotnet-aspnet-codegenerator
dotnet tool update -g dotnet-aspnet-codegenerator
```
Above commands will add all the necessary packages.

Now you can scaffold the controller using the following command:
```bash
dotnet aspnet-codegenerator controller -name ProductController -async -api -m Product -dc ProdcutContext -outDir Controllers
```

The created controller will look something like follows:
```csharp
using System;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Http;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using MyWebApi.Models; 

namespace MyWebApi.Controllers
{
    [Route("api/[controller]")]
    [ApiController]
    public class ProductsController : ControllerBase
    {
        private readonly ProductContext _context;  

        public ProductsController(ProductContext context)
        {
            _context = context;
        } 

        // GET: api/Products
        [HttpGet]
        public async Task<ActionResult<IEnumerable<Product>>> GetProducts()
        {
            return await _context.Products.ToListAsync();
        } 

        // GET: api/Products/5
        [HttpGet("{id}")]
        public async Task<ActionResult<Product>> GetProduct(int id)
        {
            var product = await _context.Products.FindAsync(id);

            if (product == null)
            {
                return NotFound();
            } 

            return product;
        }

        // PUT: api/Products/5
        // To protect from overposting attacks, see https://go.microsoft.com/fwlink/?linkid=2123754
        [HttpPut("{id}")]
        public async Task<IActionResult> PutProduct(int id, Product product)
        {
            if (id != product.Id)
            {
                return BadRequest();
            } 

            _context.Entry(product).State = EntityState.Modified; 

            try
            {
                await _context.SaveChangesAsync();
            }
            catch (DbUpdateConcurrencyException)
            {
                if (!ProductExists(id))
                {
                    return NotFound();
                }
                else
                {
                    throw;
                }
            }  

            return NoContent();
        }  

        // POST: api/Products
        // To protect from overposting attacks, see https://go.microsoft.com/fwlink/?linkid=2123754
        [HttpPost]
        public async Task<ActionResult<Product>> PostProduct(Product product)
        {
            _context.Products.Add(product);
            await _context.SaveChangesAsync();

            // return CreatedAtAction("GetProduct", new { id = product.Id }, product);
            return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
        }

        // DELETE: api/Products/5
        [HttpDelete("{id}")]
        public async Task<IActionResult> DeleteProduct(int id)
        {
            var product = await _context.Products.FindAsync(id);
            if (product == null)
            {
                return NotFound();
            } 

            _context.Products.Remove(product);
            await _context.SaveChangesAsync();  

            return NoContent();
        }  

        private bool ProductExists(int id)
        {
            return _context.Products.Any(e => e.Id == id);
        }
    }
}
```

You can also create your own custom controller with things like authorization and custom return type.
```c#
using System.Security.Claims;
using AutoMapper;
using BlogApi.Data;
using BlogApi.DTOs;
using BlogApi.Mappings;
using BlogApi.Models;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;

namespace BlogApi.Controllers;

[ApiController]
[Route("api/[controller]")] // api/posts
public class PostsController : ControllerBase
{
    // ControllerBase -> no views, only JSON responses

    // Lets us access the database
    private readonly AppDbContext _db;

    // converts between entity models and DTOs
    private readonly IMapper _mapper;

    public PostsController(AppDbContext db, IMapper mapper)
    {
        _db = db;
        _mapper = mapper;
    }

    // Get api/posts
    [HttpGet]
    public async Task<ActionResult<List<PostResponseDto>>> GetAll()
    {
        // Fetch all post and include the user and comments, than sort by 'CreatedAt' in ascending order.
        var posts = await _db
            .Posts.Include(p => p.User)
            .Include(p => p.Comments)
            .OrderByDescending(p => p.CreatedAt)
            .ToListAsync();

        // Convert 'posts' to List<dto> ('PostResponseDto')
        return Ok(_mapper.Map<List<PostResponseDto>>(posts));
    }

    // Get api/posts/5
    [HttpGet("{id}")]
    public async Task<ActionResult<PostResponseDto>> GetById(int id)
    {
        // Find first post with given ID.
        var post = await _db
            .Posts.Include(p => p.User)
            .Include(p => p.Comments)
            .FirstOrDefaultAsync(p => p.Id == id);

        // Returns 404 (not found)
        if (post == null)
            return NotFound();

        // Converts 'post' to dto ('PostResponseDto')
        return Ok(_mapper.Map<PostResponseDto>(post));
    }

    // Post api/posts [Authorize]
    [HttpPost]
    [Authorize]
    public async Task<ActionResult<PostResponseDto>> Create(CreatePostDto dto)
    {
        // Extract logged-in user ID from JWT token
        var userId = int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier)!);

        // converts incoming dto to post
        var post = _mapper.Map<Post>(dto);
        // assigns current user as owner
        post.UserId = userId;

        // adds post to database
        _db.Posts.Add(post);
        await _db.SaveChangesAsync();

        // explicitly loads related user data
        await _db.Entry(post).Reference(p => p.User).LoadAsync();

        // returns 201 (created)
        // includes location of new resource
        // created post data as 'PostResponseDto'
        return CreatedAtAction(
            nameof(GetById),
            new { id = post.Id },
            _mapper.Map<PostResponseDto>(post)
        );
    }

    // Put api/posts/5 [Authorize]
    [HttpPut("{id}")]
    [Authorize]
    public async Task<IActionResult> Update(int id, UpdatePostDto dto)
    {
        var userId = int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier)!);

        var post = await _db.Posts.FindAsync(id);

        if (post == null)
            return NotFound();

        // Only the user that created this post can modify this, else forbid 403
        if (post.UserId != userId)
            return Forbid();

        if (dto.Title != null)
            post.Title = dto.Title;
        if (dto.Content != null)
            post.Content = dto.Content;

        post.UpdatedAt = DateTime.UtcNow;

        await _db.SaveChangesAsync();
        return NoContent();
    }

    // Delete api/posts/5 [Authorize]
    [HttpDelete("{id}")]
    [Authorize]
    public async Task<IActionResult> Delete(int id)
    {
        var userId = int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier)!);

        var post = await _db.Posts.FindAsync(id);
        if (post == null)
            return NotFound();
        if (post.UserId != userId)
            return Forbid();

        _db.Posts.Remove(post);
        await _db.SaveChangesAsync();
        return NoContent();
    }
}
```

Important things to know about above code:
##### • `var userId = int.Parse(User.FindFirstValue(ClaimTypes.NameIdentifier)!);`:
Above line retrieves the currently logged-in user's ID from the JWT/authentication claims and converts it to an integer. If you have added more than one claim to your JWT (will add in later step), you can also access them.
Say you have added `Username` and `Email`, you can access them using:
```c#
new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
new Claim(ClaimTypes.Name, user.Username),
new Claim(ClaimTypes.Email, user.Email),
```

The full chain visualized:
```text
Client sends request
  └── Header: Authorization: Bearer eyJhbGci...

ASP.NET Core JWT Middleware
  └── Decodes token
  └── Validates signature, issuer, audience, expiry
  └── Rebuilds claims into ClaimsPrincipal
  └── Assigns it to HttpContext.User

Your Controller
  └── User  →  HttpContext.User  (ClaimsPrincipal)
        └── .FindFirstValue(ClaimTypes.NameIdentifier)
                └── searches Claims collection
                └── returns "3"  (the user's Id as string)

You parse it
  └── int.Parse("3")  →  3
```
##### • `ActionResult` VS `IActionResult`:
The simple rule:
```text
Has a response body  →  ActionResult<T>
No response body     →  IActionResult
```
In our `PostsController`
```c#
// Returns data — use ActionResult<T>
public async Task<ActionResult<List<PostResponseDto>>> GetAll()     // returns a list
public async Task<ActionResult<PostResponseDto>>      GetById()     // returns one post
public async Task<ActionResult<PostResponseDto>>      Create()      // returns created post

// Returns no data — use IActionResult
public async Task<IActionResult> Update()    // returns 204 No Content
public async Task<IActionResult> Delete()    // returns 204 No Content
```

What each one actually is:
```c#
// IActionResult
// ─────────────
// A plain interface. Represents "some HTTP response".
// No type information — the compiler has no idea what's inside.
// You can return Ok(), NotFound(), BadRequest(), NoContent(),
// CreatedAtAction() — anything. All are valid.

public async Task<IActionResult> Update(int id, UpdatePostDto dto)
{
    // ...
    return NoContent();   // 204, no body — perfectly fine
    return NotFound();    // 404, no body — perfectly fine
    return Forbid();      // 403, no body — perfectly fine
}


// ActionResult<T>
// ───────────────
// A generic wrapper. Tells the compiler (and Swagger) exactly
// what type lives in the response body on success.
// Still lets you return NotFound(), BadRequest() etc. for errors.

public async Task<ActionResult<PostResponseDto>> GetById(int id)
{
    var post = await _db.Posts.FindAsync(id);

    if (post == null) return NotFound();         // error — no body, fine
    return Ok(_mapper.Map<PostResponseDto>(post)); // success — typed body
}
```
One other benefit of `ActionResult<T>` is in Swagger:
This is the most practical reason to use `ActionResult<T>` whenever you have a response body. Swagger reads the `<T>` to generate accurate documentation automatically:
```csharp
// With IActionResult — Swagger sees nothing
public async Task<IActionResult> GetById(int id)
// Swagger documents this as returning: "No schema"
// Developers consuming your API have no idea what shape the response is


// With ActionResult<T> — Swagger sees everything
public async Task<ActionResult<PostResponseDto>> GetById(int id)
// Swagger documents this as returning:
// {
//   "id": 0,
//   "title": "string",
//   "content": "string",
//   "authorUsername": "string",
//   "createdAt": "2026-01-01T00:00:00Z",
//   "commentCount": 0
// }
```
`ActionResult<T>` can even implicitly convert a `T` directly, so you don't even need `Ok()` in some cases.

Quick decision guide for every endpoint you write:
```text
Writing GET by id?         →  ActionResult<PostResponseDto>
Writing GET all?           →  ActionResult<List<PostResponseDto>>
Writing POST (create)?     →  ActionResult<PostResponseDto>   (returns created resource)
Writing PUT (update)?      →  IActionResult                   (204, no body)
Writing PATCH?             →  IActionResult                   (204, no body)
Writing DELETE?            →  IActionResult                   (204, no body)
Writing auth login?        →  ActionResult<AuthResponseDto>   (returns token)
```
The pattern is consistent across every API you'll ever build — if something comes back in the body, type it with `ActionResult<T>`. If the success case is just a status code with no body, use `IActionResult`.
##### • `Controller` VS `ControllerBase`:

| Feature                        | `ControllerBase` | `Controller`            |
| ------------------------------ | ---------------- | ----------------------- |
| Intended for                   | APIs             | MVC apps with Views     |
| Supports Views/Razor           | ❌ No             | ✅ Yes                   |
| Includes API helper methods    | ✅ Yes            | ✅ Yes                   |
| Includes View-related features | ❌ No             | ✅ Yes                   |
| Lightweight                    | ✅ Yes            | ❌ Slightly heavier      |
| Common use case                | REST APIs        | Web apps returning HTML |
Inheritance Relationship:
```text
Object
   └── ControllerBase
           └── Controller
```

**What can `ControllerBase` do:**
`ControllerBase` is the **minimal base class** for API controllers.
It provides:
- Http Context access
- Request/Response handling
- Model binding
- Validation
- Helper methods like:
	- `Ok()`
	- `BadRequest()`
	- `NotFound()`
	- `CreatedAtAction()`
	- `File()`
	- `Json()`
But it does not support:
- Razor views
- `View()`
- `PartialView()`
- `ViewBag`
- `TempData`

**What can `Controller` do:**
`Controller` is used for **MVC applications** that return HTML views.
It includes everything from `ControllerBase` plus:
- `View()`
- `PartialView()`
- `ViewBag`
- `ViewData`
- `TempData`


#### 5. JWT auth
First we will create services which will handle authentication work (register and login).
```c#
// Services/IAuthService.cs
using BlogApi.DTOs;

namespace BlogApi.Services;

public interface IAuthService
{
    Task<AuthResponseDto> Register(RegisterDto dto);
    Task<AuthResponseDto> Login(LoginDto dto);
}
```

```c#
// Services/AuthService.cs
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Security.Cryptography;
using System.Text;
using AutoMapper;
using BlogApi.Data;
using BlogApi.DTOs;
using BlogApi.Models;
using Microsoft.EntityFrameworkCore;
using Microsoft.IdentityModel.Tokens;

namespace BlogApi.Services;

public class AuthService : IAuthService
{
	// A connection to database
    private readonly AppDbContext _db;
    // A way to read appsettings
    private readonly IConfiguration _config;
    // AutoMapper
    private readonly IMapper _mapper;

    public AuthService(AppDbContext db, IConfiguration config, IMapper mapper)
    {
        _db = db;
        _config = config;
        _mapper = mapper;
    }

    public async Task<AuthResponseDto> Register(RegisterDto dto)
    {
        // checks if user already exists
        if (await _db.Users.AnyAsync(u => u.Email == dto.Email))
            throw new InvalidOperationException("User already exists.");

        // map dto to user entity
        var user = _mapper.Map<User>(dto);
        // hash the password
        user.PasswordHash = BCrypt.Net.BCrypt.HashPassword(dto.Password);

        // save user to database
        _db.Users.Add(user);
        await _db.SaveChangesAsync();

        // return response with jwt token
        var response = _mapper.Map<AuthResponseDto>(user);
        response.Token = GenerateToken(user);
        return response;
    }

    public async Task<AuthResponseDto> Login(LoginDto dto)
    {
        // find user by email, if not found, throw error
        var user =
            await _db.Users.FirstOrDefaultAsync(u => u.Email == dto.Email)
            ?? throw new UnauthorizedAccessException("Invalid credentials");

        // matches hashed password and the password given to us now. done by BCrypt
        if (!BCrypt.Net.BCrypt.Verify(dto.Password, user.PasswordHash))
            throw new UnauthorizedAccessException("Invalid credentials");

        // if valid, generate token and return response
        var response = _mapper.Map<AuthResponseDto>(user);
        response.Token = GenerateToken(user);
        return response;
    }

    private string GenerateToken(User user)
    {
        // get secret key from config.
        var secret = _config["JwtSettings:Secret"];
        // create signing key
        var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secret!));

        // create claims, user info inside token.
        // these claims are stored inside the jwt, and can be used for authorization.
        var claims = new[]
        {
            new Claim(ClaimTypes.NameIdentifier, user.Id.ToString()),
            new Claim(ClaimTypes.Name, user.Username),
            new Claim(ClaimTypes.Email, user.Email),
        };

        // create token
        var token = new JwtSecurityToken(
            issuer: _config["JwtSettings:Issuer"],
            audience: _config["JwtSettings:Audience"],
            claims: claims,
            expires: DateTime.UtcNow.AddMinutes(int.Parse(_config["JwtSettings:ExpiryMinutes"]!)),
            signingCredentials: new SigningCredentials(key, SecurityAlgorithms.HmacSha256)
        );

        // convert token to string
        return new JwtSecurityTokenHandler().WriteToken(token);
    }
}
```

In general cases your Services folder will contain your business logic. The main ideology is to keep your controllers thin and pass down the work to services and repositories.

Add controller that will use the above `AuthService`.
```c#
// Controller/AuthController
using BlogApi.DTOs;
using BlogApi.Services;
using Microsoft.AspNetCore.Mvc;

[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly IAuthService _authService;

    public AuthController(IAuthService authService)
    {
        _authService = authService;
    }

    [HttpPost("register")]
    public async Task<ActionResult<AuthResponseDto>> Register(RegisterDto dto)
    {
        var result = await _authService.Register(dto);

        return Ok(result);
    }

    [HttpPost("login")]
    public async Task<ActionResult<AuthResponseDto>> Login(LoginDto dto)
    {
        var result = await _authService.Login(dto);

        return Ok(result);
    }
}
```

Somethings to know about above controller:
##### • Why inject interface only and not the implementation:
You inject the interface instead of the implementation class because of:
- abstraction
- loose coupling
- easier testing
- flexibility
- maintainability
If you inject the implementation directly, your controller becomes tightly coupled to that exact class. Controller depends on a specific implementation. The controller only cares about `I need something that behaves like a post service`. 
It is also easy to swap implementation, you just have to change your `Program.cs`
```csharp
AddScoped<IPostService, PostService>();
```
to 
```csharp
AddScoped<IPostService, CachedPostService>();
```

##### • Why `private readonly`?
`private` means that only this class can access it. `readonly` means that value can be assigned once. Value assignment is usually done in constructor, this prevents accidental reassignment after constructor finishes.
Another reason for `readonly` is that dependencies should not change while controller is running.

Now you need to add JWT auth in your `Program.cs`
Use dotnet 9, 10 have many issues regarding Swagger.
```c#
// Dependency Injection
builder.Services.AddScoped<IAuthService, AuthService>();

// 'JwtSettings:Issuer' and 'JwtSettings:Audience' are present in appsettings.json
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.TokenValidationParameters =
            new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ValidIssuer = builder.Configuration[
                "JwtSettings:Issuer"],
            ValidAudience = builder.Configuration[
                "JwtSettings:Audience"],
            IssuerSigningKey = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(
                    builder.Configuration[
                        "JwtSettings:Secret"]!))
        };
    });

builder.Services.AddAuthorization();

// In the middleware pipeline (order matters!)
app.UseAuthentication();   // before
app.UseAuthorization();    // after
```
#### 6. Swagger & Errors
We will add a custom middleware which will be responsible for handling exceptions.
```csharp
// Middleware/ExceptionMiddleware.cs
using Microsoft.AspNetCore.Mvc;

public class ExceptionMiddleware
{
	// a function that processes an HTTP request. It must accept an 'HttpContext' and return a 'Task'.
    private readonly RequestDelegate _next;
    // 'ILogger' is prebuilt loggin interface.
    private readonly ILogger<ExceptionMiddleware> _logger;

    public ExceptionMiddleware(RequestDelegate next, ILogger<ExceptionMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, ex.Message);
            await HandleExceptionAsync(context, ex);
        }
    }

    private static async Task HandleExceptionAsync(HttpContext context, Exception ex)
    {
        context.Response.ContentType = "application/json";

        context.Response.StatusCode = ex switch
        {
            UnauthorizedAccessException => 401,
            InvalidOperationException => 400,
            KeyNotFoundException => 404,
            _ => 500,
        };

        var problem = new ProblemDetails
        {
            Status = context.Response.StatusCode,
            Title = GetTitle(ex),
            Detail = ex.Message,
        };

        await context.Response.WriteAsJsonAsync(problem);
    }

    private static string GetTitle(Exception ex) =>
        ex switch
        {
            UnauthorizedAccessException => "Unauthorized",
            InvalidOperationException => "Bad request",
            KeyNotFoundException => "Not found",
            _ => "Server error",
        };
}
```

Now we can add `Swagger` with `JWT`  support in our `Program.cs`.
```csharp
// Program.cs
using Microsoft.OpenApi.Models

builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(options =>
{
    options.SwaggerDoc("v1", new OpenApiInfo
    {
        Title   = "Blog API",
        Version = "v1",
        Description = "A blog platform API"
    });

    // Add JWT bearer input to Swagger UI
    options.AddSecurityDefinition("Bearer",
        new OpenApiSecurityScheme
    {
        Name   = "Authorization",
        Type   = SecuritySchemeType.Http,
        Scheme = "Bearer",
        In     = ParameterLocation.Header,
        Description = "Enter your JWT token"
    });

	// Apply the JWT requirement to all endpoints globally
    options.AddSecurityRequirement(new
        OpenApiSecurityRequirement
    {
        {
            new OpenApiSecurityScheme {
                Reference = new OpenApiReference {
                    Type = ReferenceType.SecurityScheme,
                    Id   = "Bearer"
                }
            },
            Array.Empty<string>()
        }
    });
});

// Global error handler, catches all unhandled exceptions from everything below it.
app.UseMiddleware<ExceptionMiddleware>();

// Enable Swagger UI in development
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(c => c.SwaggerEndpoint(
        "/swagger/v1/swagger.json", "Blog API v1"));
}
```

#### 7. Run your program using `dotnet run`
Now, you can simply do `dotnet run` and you will get a url `localhost:<port>` in your terminal. You can append `/swagger` to visualize your api.

[Connecting Razor Pages to web Api](Connecting%20Razor%20Pages%20to%20web%20Api.md)


Tags:
#development 
#dotnet 
#database 