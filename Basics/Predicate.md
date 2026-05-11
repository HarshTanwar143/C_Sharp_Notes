`Predicate<T>` is a **built-in delegate** that represents a method that:
- Takes **one parameter**
- Returns a **bool**

Internally (in definition) it looks like this:
```csharp
public delegate bool Predicate<T>(T obj);
```
A method that takes a `T` and return `true`  or `false`.

Example:
```csharp
using System;
// For LINQ
using System.Collections.Generic;

public class HelloWorld
{
    public static void Main(string[] args)
    {
        var list = new List<int>();
        
        for(int i = 1; i <= 10; i++){
            list.Add(i);
        }
		
		// Accepts a int and returns bool.
        Predicate<int> isEven = p => p % 2 == 0;

		// 'FindAll' will filter out only those elements which met the condition.
        var res = list.FindAll(isEven);

        foreach(var i in res){
            Console.WriteLine(i);
        }
    }
}
```

Tags:
#csharp 
