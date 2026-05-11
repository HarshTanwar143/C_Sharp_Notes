A delegate is a type-safe function pointer. It lets you treat methods as values that you can pass around, store in variables, invoke later, combine, etc.
Think of it as:
- A contract saying: "a method that matches this exact signature"
- Once you have a variable of delegate type → you can assign any compatible method to it
- You can call it like a normal method → but which actual method runs is decided at runtime

### Simple Analogy
Imagine a TV remote. The remote does not contain the TV logic.
It only knows:
- which button calls which action
Delegate works similarly:
- stores method reference
- invokes method later


Example:
**1. Basic Syntax : Declaration & Usage** 
```csharp
// Step 1: Declare the delegate type (signature)
public delegate void SimpleAction();                  // no parameters, no return
public delegate int MathOperation(int a, int b);      // two ints → int
public delegate bool Predicate<T>(T item);            // generic version
```
```csharp
// Step 2: Methods that match the signature
static void SayHello()
{
    Console.WriteLine("Hello from delegate!");
}

static int Add(int x, int y) => x + y;
static int Multiply(int x, int y) => x * y;
```
```csharp
// Step 3: Using it
static void Main()
{
    // Create delegate instance (old style - still works)
    SimpleAction action1 = new SimpleAction(SayHello);

    // Modern style (method group conversion - preferred)
    SimpleAction action2 = SayHello;           // no new, no ()
    
    action1();     // Hello from delegate!
    action2();     // Hello from delegate!

    // With parameters & return
    MathOperation op = Add;
    Console.WriteLine(op(10, 5));          // 15

    op = Multiply;
    Console.WriteLine(op(10, 5));          // 50
}
```

One thing to make sure is that methods which you want to assign to a delegate should have same signature(same number of parameters, same type of parameters, same return value, order of parameters).

You can also attach multiple methods to a delegate.

**Basic Example (Multiple delegate)**
```csharp
using System;

public delegate void MyDelegate();

class Program
{
    static void MethodA()
    {
        Console.WriteLine("Method A executed");
    }

    static void MethodB()
    {
        Console.WriteLine("Method B executed");
    }

    static void Main()
    {
        MyDelegate del = MethodA;   // Add first method
        del += MethodB;             // Add second method

        del(); // Calls BOTH MethodA and MethodB
        del -= MethodA;
        
        del(); // Calls only MethodB
    }
}
```

### Different Types of Delegates
#### 1. Custom delegates
These are manually declared.
```csharp
public delegate void Logger(string message);

Logger log = Console.WriteLine;
log("Hello");
```

#### 2. Action delegate
It represents methods with **NO** return value.
```csharp
Action<string> log = Console.WriteLine;
log("Hello");

// Above is equivalent to:
delegate void Something(string value);
```

It is mandatory that `Action<>` always return `void`.
`Action<>` is very common in callbacks, middleware, event handlers, background processing.

##### 3. Func delegate
It represents methods that **returns** a value.
```csharp
Func<int, int, int> add = (a, b) => a + b;
```

Meaning:
```
Input: int, int
Return: int
```
Last type parameter is always return type.

#### 4. Predicate delegate
It represents methods that return a bool. Used for conditions.
```csharp
Predicate<int> isEven = x => x % 2 == 0;

// Above is equal to:
Func<int, bool>
```

#### 5. Multicast delegate
A delegate that points to multiple methods.
```csharp
Action action = Method1;
action += Method2;
action += Method3;

action(); // All methods are executed sequentially.
```

#### 6. Anonymous delegates
Old syntax before lambdas.
```c#
Func<int, int> square = delegate(int x) {
	return x * x;
};
```

#### 7. Lambda expressions
Modern shorthand syntax. This is not technically a delegate type, it is syntax that creates delegates.
```c#
x => x * x
```

#### 8. Event delegate
Special delegates used by events.
```c#
public delegate void ButtonClick(object sender, EventArgs e);
```

#### 9. Generic delegate
```c#
Func<T>
Action<T>
Predicate<T>
```


### Quick Summary Table — What to Remember

| Question                         | Answer / Best Practice                            |
| -------------------------------- | ------------------------------------------------- |
| What is delegate?                | Type-safe function pointer / method reference     |
| When to declare custom delegate? | Almost never — use `Action`, `Func`, `Predicate`  |
| Multicast means?                 | One delegate → many methods (+=, -=)              |
| Events are?                      | Special multicast delegates + protection          |
| Lambda vs delegate?              | Lambda = short syntax to create delegate instance |
| Covariance / contravariance?     | `Func<object>` ← can assign `Func<string>` (out)  |
| Most used in practice?           | Event handlers + LINQ + callbacks + async/await   |
Backlinks:
[Predicate](Predicate.md)

Tags:
#csharp 