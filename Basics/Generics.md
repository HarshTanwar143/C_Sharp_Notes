Generics allows you to write **type-safe,** **reusable** code without knowing the exact type at compile time.

**Main idea:**
Instead of writing the same logic multiple times for int, string, Customer, Order, etc., you write it once using a placeholder (type parameter), and the compiler creates specialized versions when needed.

##### Comparison - without vs with generics:

**without**
```csharp
class IntStack
{
    private int[] items = new int[100];
    private int top = -1;
    public void Push(int value) { ... }
    public int Pop() { ... }
}

class StringStack
{
    private string[] items = new string[100];
    // almost identical code
}
```

**with**
```csharp
class Stack<T>                  // T is the type parameter
{
    private T[] items = new T[100];
    private int top = -1;

    public void Push(T value)
    {
        if (top >= items.Length - 1) throw new InvalidOperationException();
        items[++top] = value;
    }

    public T Pop()
    {
        if (top < 0) throw new InvalidOperationException();
        return items[top--];
    }
}

// Usage
var intStack    = new Stack<int>();
var nameStack   = new Stack<string>();
var orderStack  = new Stack<Order>();

intStack.Push(42);
string s = nameStack.Pop();     // type-safe – compiler prevents wrong types
```

#### Most Important Generic Patterns in 2025

| Pattern                          | Example Type                          | What it solves / typical usage                          |
|----------------------------------|---------------------------------------|----------------------------------------------------------|
| Generic collection               | `List<T>`, `Dictionary<TKey,TValue>`  | Almost every modern collection                           |
| Generic repository               | `IRepository<T>`                      | Entity Framework, Dapper, custom data access             |
| Generic service / manager        | `IService<T>`                         | CRUD operations per entity type                          |
| Generic result / wrapper         | `Result<T>`, `Maybe<T>`               | Functional style error handling / optional values        |
| Generic comparer / equality      | `IComparer<T>`, `IEqualityComparer<T>`| Custom sorting, `HashSet<T>`, `Dictionary` keys          |
| Generic constraint examples      | `where T : class, new()`              | Restrict what types can be used                          |
| Generic interface + default impl | `IValidator<T>`                       | FluentValidation style, minimal API validators           |
#### Examples

##### 1. Generic with interface

```csharp
// 1. Interface
// Any class implementing this must define Add and GetById for type T.
public interface IRepository<T>
{
    void Add(T item);
    T GetById(int id);
}

// 2. Generic class implementing interface
using System;
using System.Collections.Generic;

// Generic class implements generic interface
public class Repository<T> : IRepository<T>
{
    private readonly List<T> _items = new List<T>();

    public void Add(T item)
    {
        _items.Add(item);
        Console.WriteLine($"{typeof(T).Name} added.");
    }

    public T GetById(int id)
    {
        // Just demo logic (not real ID logic)
        if (id < 0 || id >= _items.Count)
            return default;

        return _items[id];
    }
}

// 3. Usage
public class Customer
{
    public string Name { get; set; }
}

class Program
{
    static void Main()
    {
        IRepository<Customer> customerRepo = new Repository<Customer>();

        customerRepo.Add(new Customer { Name = "Rahul" });

        Customer customer = customerRepo.GetById(0);

        Console.WriteLine(customer.Name);
    }
}
```

#### 2. Result Wrapper (very popular in APIs)

```csharp
public class Result<T>
{
    public bool IsSuccess { get; }
    public T? Value { get; }
    public string? Error { get; }

    private Result(T value) => (IsSuccess, Value) = (true, value);
    private Result(string error) => (IsSuccess, Error) = (false, error);

    public static Result<T> Success(T value) => new(value);
    public static Result<T> Failure(string error) => new(error);
}

// Usage
Result<User> result = await userService.GetUserAsync(id);
if (result.IsSuccess)
{
    Console.WriteLine(result.Value.Name);
}
```

#### 3. Generic Cache with constraint

```csharp
public class MemoryCache<T> where T : class
{
    private readonly ConcurrentDictionary<string, T> _cache = new();

    public T GetOrCreate(string key, Func<T> factory)
    {
        return _cache.GetOrAdd(key, _ => factory());
    }
}
```

#### 4. Generic method example

```csharp
public static T Max<T>(T a, T b) where T : IComparable<T>
{
    return a.CompareTo(b) >= 0 ? a : b;
}

// works with int, string, DateTime, custom types that implement IComparable<T>
```

### Quick decision table – when to use generics

| Situation                                 | Use generics? | Alternative if no generics                     |
|-------------------------------------------|---------------|------------------------------------------------|
| Same logic for many different types       | Yes           | Boxing + `object` (slow, not type-safe)        |
| Collection / container of items           | Yes           | `ArrayList`, `Hashtable` (avoid in 2025)       |
| Need compile-time type checking           | Yes           | Reflection hell                                |
| Writing base class/interface for entities | Yes           | Copy-paste per entity type                     |
| Only 2–3 types needed                     | Maybe not     | Separate classes sometimes clearer             |

- [Collections, Generic and Non-Generic](Collections,%20Generic%20and%20Non-Generic.md)

Tags:
#csharp 