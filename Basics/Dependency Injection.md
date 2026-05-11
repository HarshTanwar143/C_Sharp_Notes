**Dependency injection** is a simple way to make our C# code more flexible and easier to test.

#### Explanation
Imagine we are building a house. Instead of making the electrician inside the house, we bring the electrician from outside when needed. This way, if we want to change the electrician (maybe hire a better one), we don't have to break the whole house.

##### Why do we need it?
Without DI (Bad way):
```csharp
public class Car
{
    private Engine engine = new Engine();   // Hard-coded inside

    public void Start()
    {
        engine.StartEngine();
    }
}
```
Problem:
- You cannot change or test the `Engine` easily.
- If `Engine` has problems, `Car` is also affected.
- Very difficult to use a different engine (like ElectricEngine)

With DI
```csharp
// 1. First, create an interface (contract)
public interface IEngine
{
    void StartEngine();
}

// 2. Create classes that follow this contract
public class PetrolEngine : IEngine
{
    public void StartEngine()
    {
        Console.WriteLine("Petrol engine started");
    }
}

public class ElectricEngine : IEngine
{
    public void StartEngine()
    {
        Console.WriteLine("Electric engine started with battery");
    }
}

// 3. Car class now receives engine from outside
public class Car
{
    private readonly IEngine _engine;

    // Constructor Injection (most common and recommended)
    public Car(IEngine engine)   // ← Injection happens here
    {
        _engine = engine;
    }

    public void Start()
    {
        _engine.StartEngine();
    }
}
```

How to use it:
```csharp
class Program
{
    static void Main()
    {
        // We decide which engine to give to the car
        IEngine myEngine = new ElectricEngine();
        
        Car myCar = new Car(myEngine);   // Injecting dependency
        myCar.Start();
    }
}
```

#### Dependency Injection Lifecycle
In dependency injection, lifecycle means: How long should one object live? When should it be created? When should it be destroyed?

There are 3 main lifecycles:
##### 1. Transient (Shortest Life)
- Created every single time it is requested.
- New object -> every injection
Example -> Think of a pen. Every time you need to write something, you get a new pen.

```csharp
services.AddTransient<ILogger, ConsoleLogger>();
```
**When to use?**
- Lightweight services
- No state (doesn't remember anything)
- Cheap to create

##### 2. Scoped (Medium life)
- Created once per request (in web apps).
- Same object is used throughout one HTTP request.
Example -> Think of a **Shopping Cart**. While you are browsing one session, you should get the same cart, not a new one every time.
```csharp
services.AddScoped<IUserService, UserService>();
services.AddScoped<ICartRepository, CartRepository>();
```
**When to use?**
- In a web API, each user request gets its own Scoped objects.
- Perfect for database contexts (`DbContext`)

##### 3. Singleton (Longest life)
- Created only once in the entire application
- Same object is shared everywhere until the app shuts down.
Example -> Think of a printer in an office. There is only one printer, and everyone uses the same printer.
```csharp
services.AddSingleton<ICacheService, MemoryCacheService>();
services.AddSingleton<IConfiguration, Configuration>();
```
**When to use?**
- Heavy objects
- Needs to maintain state (like cache, configuration, connection pools)
- Should be thread safe

#### Visual Summary (Very Simple)

| Lifecycle     | New Object Created When? | How many objects in app? | Best Example                 |
| ------------- | ------------------------ | ------------------------ | ---------------------------- |
| **Transient** | Every time injected      | Many                     | Pen, Calculator              |
| **Scoped**    | Once per HTTP request    | One per request          | Shopping Cart, DbContext     |
| **Singleton** | Only once (app start)    | Only 1                   | Cache, Logger, Configuration |
**How to register in ASP.NET Core (`Program.cs`)**
```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddTransient<ITransientService, TransientService>();
builder.Services.AddScoped<IScopedService, ScopedService>();
builder.Services.AddSingleton<ISingletonService, SingletonService>();

var app = builder.Build();
```

Tags:
#csharp 