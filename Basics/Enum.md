are a lightweight way to define **a set of named constants** that belong together. They make code much more **readable**, **type-safe**, and **self-documenting** compared to using raw numbers or strings. Enums are **value types** and by default use int as their underlying type.

A basic example:
```csharp
public enum DayOfWeek
{
    Sunday,     // = 0 (default)
    Monday,     // = 1
    Tuesday,    // = 2
    Wednesday,  // = 3
    Thursday,   // = 4
    Friday,     // = 5
    Saturday    // = 6
}

// How to access above
Console.WriteLine(DayOfWeek.Sunday); // 'Sunday'
Console.WriteLine((int)DayOfWeek.Monday) // 1
```

You can even define them inside a class, let's say:
```csharp
using System;

public class Person{
	public enum DayOfWeek
	{
	    Sunday,     // = 0 (default)
	    Monday,     // = 1
	    Tuesday,    // = 2
	    Wednesday,  // = 3
	    Thursday,   // = 4
	    Friday,     // = 5
	    Saturday    // = 6
	}

    public String Name {get; set;}
}

class Program
{
    static void Main()
    {
        Person p = new Person();
        p.Name = "Mahesh";
        Console.WriteLine(Person.DayOfWeek.Sunday);
        // Can't do
        Console.WriteLine(p.DayOfWeek.Sunday);
    }
}

```

Explicit value:
```csharp
public enum HttpStatusCode
{
    OK                  = 200,
    Created             = 201,
    BadRequest          = 400,
    Unauthorized        = 401,
    NotFound            = 404,
    InternalServerError = 500
}

Console.WriteLine(HttpStatusCode.OK); // OK
Console.WriteLine((int)HttpStatusCode.OK); // 200

// If you were to stop giving integer values above say at 'Created = 201', than 'BadRequest' would have '202'
```

Tags:
#csharp 