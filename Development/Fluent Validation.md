## 1. Simple Explanation — Start From The Beginning

Imagine you're building a registration form. A user submits their name, email, age, and password.

**Before you do anything with that data — save to database, send email, process payment — you need to ask:**

- Is the name empty?
- Is the email a valid format?
- Is the age above 18?
- Is the password strong enough?

This process is called **validation** — confirming that incoming data meets your business rules before you act on it.

**FluentValidation** is a popular .NET library that gives you a clean, expressive, testable way to write those rules.

---

## 2. Real-World Analogy

Think of a **bank loan officer**.

When you apply for a loan, they don't just hand you money. They check:

- Do you have a valid ID?
- Is your credit score above a threshold?
- Is your salary sufficient?
- Are all documents present?

Each of these is a **validation rule**. The loan officer has a **checklist** — a _validator_ — that they run your application through.

FluentValidation lets you build that checklist in code — cleanly, expressively, and separately from your business logic.

---

## 3. The Problem It Solves — Why Does FluentValidation Exist?

### The Old World — Before FluentValidation

Before FluentValidation, .NET developers validated data in 3 messy ways:

---

### ❌ Way 1: Inline Validation (Worst)

```csharp
public IActionResult Register(UserDto dto)
{
    if (string.IsNullOrEmpty(dto.Name))
        return BadRequest("Name is required");

    if (dto.Name.Length > 100)
        return BadRequest("Name too long");

    if (!dto.Email.Contains("@"))
        return BadRequest("Invalid email");

    if (dto.Age < 18)
        return BadRequest("Must be 18+");

    // ... actual logic starts here, buried under validation
}
```

**Problems:**

- Controller becomes a wall of `if` statements
- Validation logic is **mixed with business logic** — violating Single Responsibility Principle
- **Not reusable** — if you have 5 endpoints accepting `UserDto`, you copy-paste this everywhere
- **Not testable in isolation** — you must invoke the full controller to test validation
- **Not readable** — a new developer can't quickly understand rules

---

### ❌ Way 2: Data Annotations

```csharp
public class UserDto
{
    [Required]
    [MaxLength(100)]
    public string Name { get; set; }

    [EmailAddress]
    public string Email { get; set; }

    [Range(18, 120)]
    public int Age { get; set; }
}
```

ASP.NET Core automatically validates these via `ModelState.IsValid`. This looks clean at first. But:

**Problems with Data Annotations:**

- Rules are **on the model itself** — violating Single Responsibility Principle
- What if `UserDto` is used in multiple contexts with **different rules**? Admin can register at any age. Regular users must be 18+. Now what?
- **Complex conditional rules are ugly or impossible**: "Email is required only if PhoneNumber is not provided"
- **No reusable rule composition**
- **Hard to unit test** in isolation
- **Database-coupled** — some annotations like `[MaxLength]` bleed into EF Core schema generation, mixing concerns
- The model knows about its own validation — that's **not its job**

---

### ✅ Way 3: FluentValidation (Correct Approach)

```csharp
public class UserDtoValidator : AbstractValidator<UserDto>
{
    public UserDtoValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty().WithMessage("Name is required")
            .MaximumLength(100).WithMessage("Name cannot exceed 100 characters");

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress().WithMessage("Invalid email format");

        RuleFor(x => x.Age)
            .GreaterThanOrEqualTo(18).WithMessage("Must be 18 or older");
    }
}
```

**What changed:**

- Validation lives in its **own class** — single responsibility
- The `UserDto` model is **clean and dumb** — just carries data
- The validator is **independently unit testable**
- You can have **multiple validators for the same DTO** depending on context
- Rules are **readable like English sentences**
- Complex rules are **easy to express**

---

## 4. Technical Deep Dive

### Setting Up FluentValidation in ASP.NET Core

```bash
dotnet add package FluentValidation.AspNetCore
```

---

### Step 1: The DTO (Data Transfer Object)

```csharp
// UserDto.cs
// Notice: NO validation attributes here. The model is clean.
// Its job is ONLY to carry data across layers.
public class UserDto
{
    public string Name { get; set; }
    public string Email { get; set; }
    public int Age { get; set; }
    public string Password { get; set; }
}
```

**Why no annotations?** Because the model's job is data transport. Validation is a **separate concern**.

---

### Step 2: The Validator

```csharp
// UserDtoValidator.cs
using FluentValidation;

public class UserDtoValidator : AbstractValidator<UserDto>
{
    public UserDtoValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty()
                .WithMessage("Name is required")
            .MinimumLength(2)
                .WithMessage("Name must be at least 2 characters")
            .MaximumLength(100)
                .WithMessage("Name cannot exceed 100 characters")
            .Matches(@"^[a-zA-Z\s]+$")
                .WithMessage("Name can only contain letters and spaces");

        RuleFor(x => x.Email)
            .NotEmpty()
                .WithMessage("Email is required")
            .EmailAddress()
                .WithMessage("A valid email address is required");

        RuleFor(x => x.Age)
            .InclusiveBetween(18, 120)
                .WithMessage("Age must be between 18 and 120");

        RuleFor(x => x.Password)
            .NotEmpty()
                .WithMessage("Password is required")
            .MinimumLength(8)
                .WithMessage("Password must be at least 8 characters")
            .Matches(@"[A-Z]")
                .WithMessage("Password must contain at least one uppercase letter")
            .Matches(@"[0-9]")
                .WithMessage("Password must contain at least one number");
    }
}
```

**Every keyword explained:**

|Keyword|Why it exists|
|---|---|
|`AbstractValidator<T>`|Base class from FluentValidation. It gives you `RuleFor`, `Validate()`, and the validation pipeline. `T` is the type you're validating.|
|`RuleFor(x => x.Name)`|Lambda expression that **selects the property**. The library uses this to build property name for error messages and to access the value. It's an **Expression Tree**, not a delegate — FluentValidation reads the property name from it.|
|`.NotEmpty()`|Checks that the value is not null, empty string, or whitespace. For strings: it's `string.IsNullOrWhiteSpace()`. For collections: checks Count > 0.|
|`.WithMessage()`|Overrides the default error message. Without this, you get generic messages like "Name must not be empty."|
|`.EmailAddress()`|Uses a built-in regex pattern to validate email format.|
|`.Matches(@"...")`|Custom regex validation. The `@` prefix is a verbatim string literal in C# — backslashes are treated literally, not as escape sequences.|
|`.InclusiveBetween(18, 120)`|Validates that a value is >= 18 AND <= 120, inclusive.|

---

### Step 3: Register With DI Container

```csharp
// Program.cs
builder.Services.AddControllers();

// Option A: Manual registration (more control)
builder.Services.AddScoped<IValidator<UserDto>, UserDtoValidator>();

// Option B: Auto-registration (scans assembly)
builder.Services.AddValidatorsFromAssemblyContaining<UserDtoValidator>();
// This scans the assembly for ALL classes that extend AbstractValidator<T>
// and registers them automatically. Cleaner for large projects.
```

**Why `AddScoped`?** Validators are typically **stateless** — they don't hold data between requests. But because they can have **injected dependencies** (like a database check), `Scoped` is the safe default. A new validator instance per HTTP request.

Why not `Singleton`? Because if your validator injects a `DbContext` (which is Scoped), you'll get a **captive dependency** — a Singleton holding a Scoped service, causing bugs. We'll cover this later.

---

### Step 4: Use in Controller

```csharp
// Approach 1: Inject and manually validate
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IValidator<UserDto> _validator;
    private readonly IUserService _userService;

    public UsersController(
        IValidator<UserDto> validator,
        IUserService userService)
    {
        _validator = validator;
        _userService = userService;
    }

    [HttpPost]
    public async Task<IActionResult> Register(UserDto dto)
    {
        // Step 1: Validate
        ValidationResult result = await _validator.ValidateAsync(dto);

        if (!result.IsValid)
        {
            // result.Errors is a List<ValidationFailure>
            // Each ValidationFailure has .PropertyName and .ErrorMessage
            return BadRequest(result.Errors
                .GroupBy(e => e.PropertyName)
                .ToDictionary(
                    g => g.Key,
                    g => g.Select(e => e.ErrorMessage).ToArray()
                ));
        }

        // Step 2: Act on valid data
        await _userService.RegisterAsync(dto);
        return Ok();
    }
}
```

**What is `ValidationResult`?** It's an object returned by `.Validate()` or `.ValidateAsync()`. It has:

- `IsValid` — `bool`: true if all rules passed
- `Errors` — `List<ValidationFailure>`: list of failures, each containing:
    - `PropertyName` — which property failed
    - `ErrorMessage` — the message
    - `AttemptedValue` — what value was submitted
    - `ErrorCode` — identifier for the rule that failed

---

### Step 5: Automatic Validation (Cleaner)

Instead of manually calling `_validator.ValidateAsync()` in every controller, you can plug FluentValidation into the **ASP.NET Core model validation pipeline**:

```csharp
// Program.cs
builder.Services.AddFluentValidationAutoValidation();
builder.Services.AddValidatorsFromAssemblyContaining<UserDtoValidator>();
```

Now your controller becomes:

```csharp
[HttpPost]
public async Task<IActionResult> Register(UserDto dto)
{
    // If we get here, dto is already valid.
    // ASP.NET Core ran the validator BEFORE calling this method.
    // If invalid, it automatically returned 400 Bad Request.

    await _userService.RegisterAsync(dto);
    return Ok();
}
```

**How does this work internally?** FluentValidation hooks into ASP.NET Core's `IModelValidator` pipeline. After **model binding** (parsing the HTTP request into `UserDto`), but before executing your action, ASP.NET Core runs all registered `IModelValidator` instances. FluentValidation registers itself as one. If validation fails, `ModelState.IsValid` is false, and ASP.NET Core's built-in behavior returns `400 Bad Request` with the errors — before your action even runs.

This is the **Filter pipeline** in action. More specifically, it's triggered during `ResourceFilter` / `ActionFilter` execution before your action method.

---

## 5. Internal Working — How FluentValidation Works Under The Hood

### Expression Trees — The Magic Behind `RuleFor`

When you write:

```csharp
RuleFor(x => x.Name)
```

This lambda is NOT compiled to a regular delegate. It's compiled to an **Expression Tree** — a data structure that represents the code as data.

FluentValidation uses this to:

1. **Extract the property name** (`"Name"`) for error messages — without you hardcoding it
2. **Compile it to a delegate** for actual property access at runtime

```csharp
// FluentValidation does something like this internally:
Expression<Func<UserDto, string>> expr = x => x.Name;

// Extract property name:
var memberExpr = (MemberExpression)expr.Body;
string propertyName = memberExpr.Member.Name; // "Name"

// Compile for runtime use:
Func<UserDto, string> getter = expr.Compile();
string value = getter(dto); // gets the actual value
```

**Why does this matter?** If you rename `Name` to `FullName` in your DTO, the error message automatically updates to `"FullName"`. You never hardcode property names as strings.

---

### The Rule Chain — Builder Pattern

```csharp
RuleFor(x => x.Name)
    .NotEmpty()
    .MaximumLength(100)
    .Matches(@"^[a-zA-Z]+$")
```

This is the **Builder Pattern**. Each method (`NotEmpty()`, `MaximumLength()`) adds a `IPropertyValidator` to an internal list on the `IRuleBuilder`. When `.Validate()` is called, FluentValidation iterates through this list and runs each validator in order.

Internally, the rule chain is a `List<IPropertyValidator>`. Think of it as a pipeline:

```
NotEmpty → MaximumLength → Matches
   ↓              ↓            ↓
 Pass?          Pass?        Pass?
   ↓              ↓            ↓
Continue       Continue    Add Error
```

By default, all rules run even if an earlier one fails. You can change this with **CascadeMode**:

```csharp
// Stop running rules for a property after first failure
RuleFor(x => x.Name)
    .Cascade(CascadeMode.Stop)
    .NotEmpty()
    .MaximumLength(100); // Won't run if NotEmpty fails
```

---

## 6. Advanced FluentValidation Patterns

### Conditional Validation

```csharp
// Only validate PhoneNumber if Email is not provided
RuleFor(x => x.PhoneNumber)
    .NotEmpty()
    .When(x => string.IsNullOrEmpty(x.Email))
    .WithMessage("Phone number is required when email is not provided");
```

**This is impossible to express cleanly with Data Annotations.**

---

### Custom Validators

```csharp
// Inline custom rule
RuleFor(x => x.Username)
    .Must(username => !username.Contains("admin"))
    .WithMessage("Username cannot contain 'admin'");

// Custom validator class (reusable)
public class StrongPasswordValidator : PropertyValidator<UserDto, string>
{
    public override string Name => "StrongPasswordValidator";

    protected override bool IsValid(ValidationContext<UserDto> context, string value)
    {
        if (value == null) return false;

        bool hasUpper = value.Any(char.IsUpper);
        bool hasDigit = value.Any(char.IsDigit);
        bool hasSpecial = value.Any(ch => !char.IsLetterOrDigit(ch));
        bool longEnough = value.Length >= 8;

        return hasUpper && hasDigit && hasSpecial && longEnough;
    }

    protected override string GetDefaultMessageTemplate(string errorCode)
        => "Password must be at least 8 chars with uppercase, digit, and special character";
}

// Use it
RuleFor(x => x.Password)
    .SetValidator(new StrongPasswordValidator());
```

---

### Async Validation — Database Checks

```csharp
public class UserDtoValidator : AbstractValidator<UserDto>
{
    private readonly IUserRepository _userRepository;

    // Note: Validator has its own constructor — it supports DI
    public UserDtoValidator(IUserRepository userRepository)
    {
        _userRepository = userRepository;

        RuleFor(x => x.Email)
            .NotEmpty()
            .EmailAddress()
            .MustAsync(async (email, cancellationToken) =>
            {
                // Hit the database to check uniqueness
                bool exists = await _userRepository.EmailExistsAsync(email);
                return !exists;
            })
            .WithMessage("This email is already registered");
    }
}
```

**Why is this powerful?** You can validate business rules that require database access — all in one place, before your service layer even runs. No more "throw an exception from the service when email exists."

**Important:** This is why validators are registered as `Scoped` — they can inject `DbContext` and other scoped services safely.

---

### Nested Object Validation

```csharp
public class OrderDto
{
    public string OrderNumber { get; set; }
    public AddressDto ShippingAddress { get; set; }
    public List<OrderItemDto> Items { get; set; }
}

public class AddressDto
{
    public string Street { get; set; }
    public string City { get; set; }
    public string PostalCode { get; set; }
}

public class AddressDtoValidator : AbstractValidator<AddressDto>
{
    public AddressDtoValidator()
    {
        RuleFor(x => x.Street).NotEmpty();
        RuleFor(x => x.City).NotEmpty();
        RuleFor(x => x.PostalCode)
            .NotEmpty()
            .Matches(@"^\d{6}$").WithMessage("Invalid Indian postal code");
    }
}

public class OrderDtoValidator : AbstractValidator<OrderDto>
{
    public OrderDtoValidator()
    {
        RuleFor(x => x.OrderNumber).NotEmpty();

        // Delegate to child validator
        RuleFor(x => x.ShippingAddress)
            .NotNull()
            .SetValidator(new AddressDtoValidator());

        // Validate each item in the collection
        RuleForEach(x => x.Items)
            .SetValidator(new OrderItemDtoValidator());
    }
}
```

**`RuleForEach`** — validates every element of a collection. Errors report as `Items[0].ProductId`, `Items[1].Quantity`, etc.

---

### Validation Groups (RuleSets)

```csharp
public class UserDtoValidator : AbstractValidator<UserDto>
{
    public UserDtoValidator()
    {
        // Run always
        RuleFor(x => x.Email).NotEmpty().EmailAddress();

        // Only run during creation
        RuleSet("Create", () =>
        {
            RuleFor(x => x.Password)
                .NotEmpty()
                .MinimumLength(8);
        });

        // Only run during update
        RuleSet("Update", () =>
        {
            RuleFor(x => x.Name).NotEmpty();
        });
    }
}

// Usage
var result = await _validator.ValidateAsync(
    dto,
    options => options.IncludeRuleSets("Create")
);
```

**Use case:** Same DTO used for create and update, but different rules apply.

---

## 7. Clean Architecture Placement

```
Solution/
├── API/                          ← Controllers, Program.cs
│   └── Program.cs                ← Register validators here
│
├── Application/                  ← Validators live here
│   ├── DTOs/
│   │   └── UserDto.cs
│   ├── Validators/
│   │   └── UserDtoValidator.cs   ← Validator in Application layer
│   └── Services/
│       └── UserService.cs
│
├── Domain/                       ← Business entities, no validators
│
└── Infrastructure/               ← DbContext, Repositories
```

**Why Application layer?** Validators enforce **application-level rules** (not domain rules). Domain rules live in domain entities (e.g., `Order.AddItem()` throws if quantity is 0). Application validators check HTTP-layer concerns: is the data format correct? Is the email valid? These are input validation concerns, not domain invariants.

---

## 8. Bad Practices vs Good Practices

```csharp
// ❌ BAD: Validation in controller action
public IActionResult Register(UserDto dto)
{
    if (string.IsNullOrEmpty(dto.Name)) return BadRequest("Name required");
    if (dto.Age < 18) return BadRequest("Too young");
    // ...100 more lines of validation
    _userService.Register(dto);
    return Ok();
}

// ❌ BAD: Validation in service
public async Task RegisterAsync(UserDto dto)
{
    if (string.IsNullOrEmpty(dto.Name))
        throw new ArgumentException("Name required");
    // Services should trust that input is already validated
}

// ❌ BAD: Data annotations on domain entities
public class User
{
    [Required] // ← mixing domain model with annotation concern
    public string Name { get; set; }
}

// ✅ GOOD: Dedicated validator, clean controller, clean service
public class UserDtoValidator : AbstractValidator<UserDto> { ... }

[HttpPost]
public async Task<IActionResult> Register(UserDto dto)
{
    // Validation already happened — either manually or auto-validation
    await _userService.RegisterAsync(dto);
    return Ok();
}
```

---

## 9. Interview Questions

1. **What is the difference between FluentValidation and Data Annotations?**
2. **Why are validators registered as Scoped and not Singleton?**
3. **How does `RuleFor(x => x.Name)` know the property is called "Name" without you hardcoding it?**
4. **What is CascadeMode and when would you use it?**
5. **How does FluentValidation integrate with ASP.NET Core's model validation pipeline?**
6. **Can you have multiple validators for the same DTO? When would you?**
7. **How would you validate that an email is unique without FluentValidation hitting the DB in a controller?**
8. **What happens internally when `.Validate()` is called?**
9. **What is the difference between `.Must()` and `.MustAsync()`?**
10. **Where should validators live in Clean Architecture and why?**

---

## 🔥 Cross-Questions For You

Now I turn the table. Answer these before moving on:

**Q1.** You have a `ProductDto` with a `Discount` field. The rule is: `Discount` can only be set if the user has the role `"Admin"`. How would you implement this in FluentValidation? Think about what you'd need access to.

**Q2.** If a validator is registered as `Singleton` and it injects a `DbContext` (which is `Scoped`) — what specific problem occurs? What is the name of this anti-pattern?

**Q3.** You have `UserDto` used for both registration and profile update. During registration, `Password` is required. During update, it's optional. What FluentValidation feature would you use? Sketch out the approach.

**Q4.** Look at this code:

```csharp
RuleFor(x => x.Email)
    .NotEmpty()
    .EmailAddress()
    .MustAsync(async (email, ct) => !await _repo.EmailExistsAsync(email));
```

If `NotEmpty()` fails, does `MustAsync()` still execute? What would happen? What should you add to prevent unnecessary DB calls?

**Q5.** Why is `RuleFor(x => x.Name)` using a **lambda expression** and not just a **string** like `RuleFor("Name")`? What advantage does the lambda give you that a string cannot?

---

## 🏋️ Mini Assignment

Build a `ProductDto` validator with these requirements:

1. `Name` — required, 3–200 chars, letters/spaces/hyphens only
2. `Price` — required, greater than 0, less than 1,000,000
3. `CategoryId` — required, must be positive integer
4. `Description` — optional, but if provided must be max 2000 chars
5. `DiscountPercent` — optional, but if provided must be between 0 and 90
6. `Tags` — optional list, but if provided, each tag must be non-empty and max 50 chars

**Bonus:** Add an async rule that checks `CategoryId` exists in the database via `ICategoryRepository.ExistsAsync(int id)`.

Write the full validator class. Then explain: _why did you choose `Scoped` registration? What would break if you chose `Singleton`?_