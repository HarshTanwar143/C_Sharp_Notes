A way to access values of private fields. You can't directly access private fields of a class in `csharp` that's why properties were introduced.

Example:
```csharp
using System;

public class Person
{
    private string _firstName;   // backing field, you can even assign a default value here.

    public string FirstName
    {
        get { return _firstName; }
        set
        {
            if (string.IsNullOrWhiteSpace(value))
                throw new ArgumentException("Name cannot be empty");

            _firstName = value.Trim();
        }
    }
}

class Program
{
    static void Main()
    {
        Person person = new Person();

        // Setting property (calls the setter)
        person.FirstName = "  Rahul  ";

        // Getting property (calls the getter)
        Console.WriteLine($"First Name: '{person.FirstName}'");

        // Example of invalid value
        try
        {
            person.FirstName = "   "; // Will throw exception
        }
        catch (ArgumentException ex)
        {
            Console.WriteLine($"Error: {ex.Message}");
        }
    }
}
```

A more simple example:
```csharp
public class Person
{
    public string FirstName { get; set; }           // read-write
    public int Age        { get; set; }
    public string Email   { get; private set; }     // public get, private set
}
```
Above we have used `private set` meaning that you can't assign a value to it. 
If you were to not using any `set` it will become a **read-only** property.

Tags:
#csharp 