| Feature                                   | Abstract Class                                             | Interface                                                     | Winner / When to choose     |
| ----------------------------------------- | ---------------------------------------------------------- | ------------------------------------------------------------- | --------------------------- |
| Can have fields / state                   | Yes (fields, properties with backing field)                | No (only properties without implementation until C# 8)        | Abstract class              |
| Can have constructors                     | Yes                                                        | No                                                            | Abstract class              |
| Can have concrete methods                 | Yes (both abstract + implemented)                          | Yes (default implementations since C# 8)                      | Both (but different intent) |
| Can have private / protected members      | Yes                                                        | No (everything is public by default)                          | Abstract class              |
| Multiple inheritance                      | No (a class can inherit only one abstract class)           | Yes (a class can implement many interfaces)                   | Interface                   |
| Access modifiers on members               | Full control (private, protected, internal, public)        | All members are public (until C# 8 static/private were added) | Abstract class              |
| Can be used as base for fields/properties | Yes (can enforce state + behavior)                         | No (cannot enforce state)                                     | Abstract class              |
| "Is-a" vs "Can-do"                        | Represents **"is a"** relationship                         | Represents **"can do"** / capability                          | Depends on modeling         |
| Versioning / evolution                    | Harder (adding abstract member breaks all derived classes) | Easier (default interface methods since C# 8)                 | Interface (modern C#)       |
| Performance (very minor)                  | Slightly faster virtual call dispatch                      | Slightly slower (interface dispatch)                          | Almost never matters        |

### When to choose what (real rules used in 2024–2025)

| You want to...                                               | Use                                             | Example from real projects                          |
| ------------------------------------------------------------ | ----------------------------------------------- | --------------------------------------------------- |
| Share state + common behavior                                | Abstract class                                  | `ControllerBase`, `DbContext`, `Animal`             |
| Enforce a contract / capability                              | Interface                                       | `IDisposable`, `IEnumerable<T>`, `ILogger`          |
| Allow a class to have many roles/behaviors                   | Interface                                       | `IComparable<T>`, `IAsyncDisposable`, `IRepository` |
| Provide some default behavior but allow override             | Abstract class or interface with default method | `IEqualityComparer<T>` (default impl since C# 8)    |
| Build a family of related classes with shared implementation | Abstract class                                  | `Stream` → `MemoryStream`, `FileStream`             |
| Want future-proof API (add methods later)                    | Interface + default methods                     | Most new .NET libraries after ~2019                 |

### Code comparison – same scenario, both styles

**Scenario**: We want different kinds of notifications (email, sms, push)

#### Using abstract class

```csharp
public abstract class NotificationSender
{
    // State
    protected string SenderName { get; }

    // Constructor
    protected NotificationSender(string senderName)
    {
        SenderName = senderName ?? throw new ArgumentNullException();
    }

    // Concrete method
    protected void LogSendAttempt(string recipient)
    {
        Console.WriteLine($"[{SenderName}] Attempting to send to {recipient}");
    }

    // Must be implemented
    public abstract void SendAsync(string recipient, string message);
}

public class EmailSender : NotificationSender
{
	// parent class constructor
    public EmailSender() : base("noreply@company.com") { }
	
	// override is a must for inheritance
    public override void SendAsync(string recipient, string message)
    {
        LogSendAttempt(recipient);
        Console.WriteLine($"Email sent to {recipient}");
    }
}
```

#### Using interface + default implementation (modern style)

```csharp
public interface INotificationSender
{
    string SenderName { get; }

    // Default implementation (C# 8+)
    Task LogSendAttemptAsync(string recipient)
    {
        Console.WriteLine($"[{SenderName}] Attempting to send to {recipient}");
        return Task.CompletedTask;
    }

    Task SendAsync(string recipient, string message);
}

public class EmailSender : INotificationSender
{
    public string SenderName => "noreply@company.com";

    public async Task SendAsync(string recipient, string message)
    {
        await LogSendAttemptAsync(recipient);   // can use default
        await Task.Delay(300);
        Console.WriteLine($"Email sent to {recipient}");
    }
}
```

### Quick decision flowchart 

1. Do you need to share **state** (fields) or **protected helper methods**?  
   → **Yes** → Abstract class

2. Do many unrelated classes need to support the same **capability**?  
   → **Yes** → Interface  
   (logging, disposable, comparable, repository, etc.)

3. Are you building a public library/API that will evolve over years?  
   → **Prefer interface** + default implementations

4. Are you modelling a clear **"is-a"** hierarchy with shared base behaviour?  
   → **Abstract class** (Animal → Mammal → Dog)

5. Do you need both?  
   → Very common: abstract base class **+** one or more interfaces  
   Example: `DbContext` is abstract class + implements `IDbContext`, `IInfrastructure<IServiceProvider>`, etc.

Bottom line in 2025:

- **Abstract classes** → for family of classes with shared **implementation + state**  
- **Interfaces** → for **contracts**, **capabilities**, **multiple behaviours**, and **future-friendly APIs**

Which style are you currently trying to decide between in your project?  
I can help you pick the better one for your specific case.

[Multiple Inheritance](Multiple%20Inheritance.md)

Tags:
#csharp 