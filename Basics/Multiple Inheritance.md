C# doesn't support multiple inheritance using classes. It can only do so using interfaces.

**Inheritance with two interfaces**
```csharp
using System;

// Contract 1: can fly
public interface IFlyable
{
    void Fly();
    // getter property
    double MaxAltitude { get; }
}

// Contract 2: can swim
public interface ISwimmable
{
    void Swim();
    // getter property
    double MaxDepth { get; }
}

// A concrete class that does BOTH
public class FlyingFish : IFlyable, ISwimmable
{
    public double MaxAltitude => 50.0;   
    public double MaxDepth => 200.0;     

    public void Fly()
    {
        Console.WriteLine("Gliding above the water at 40 km/h!");
    }

    public void Swim()
    {
        Console.WriteLine("Swimming fast like a torpedo!");
    }
}

// Program entry point
class Program
{
    static void Main()
    {
        // Using concrete type
        FlyingFish fish = new FlyingFish();

        Console.WriteLine($"Max altitude: {fish.MaxAltitude} meters");
        Console.WriteLine($"Max depth: {fish.MaxDepth} meters");

        fish.Fly();
        fish.Swim();

        Console.WriteLine();

        // Using interface references
        // If you were to use 'Swim' here, it will give error saying it don't exist on 'IFlyable'.
        IFlyable flyable = fish;
        // If you were to use 'Fly' here, it will give error saying it don't exist on 'ISwimmable'.
        ISwimmable swimmable = fish;

        Console.WriteLine($"(Via IFlyable) Max altitude: {flyable.MaxAltitude}");
        flyable.Fly();

        Console.WriteLine($"(Via ISwimmable) Max depth: {swimmable.MaxDepth}");
        swimmable.Swim();
    }
}
```

**Multiple inheritance with abstract and two interfaces**
```csharp
using System;

// Base abstract class
public abstract class EntityBase
{
    public int Id { get; protected set; }
    // By default it is set to current time (utc)
    public DateTime CreatedAt { get; protected set; } = DateTime.UtcNow;
    public DateTime UpdatedAt { get; protected set; }
}

public interface IAuditable
{
    void MarkAsUpdated();
}

public interface ISoftDeletable
{
    bool IsDeleted { get; }
    void SoftDelete();
}

// Real entity using both base class + multiple interfaces
public class Customer : EntityBase, IAuditable, ISoftDeletable
{
	// an empty string
    public string Name { get; set; } = string.Empty;
    // false
    public bool IsDeleted { get; private set; } = false;

    public void MarkAsUpdated()
    {
        UpdatedAt = DateTime.UtcNow;
    }

    public void SoftDelete()
    {
        IsDeleted = true;
    }
}

// Program entry point
class Program
{
    static void Main()
    {
        Console.WriteLine("=== Using concrete type ===");

        Customer customer = new Customer
        {
            Name = "Rahul"
        };

        Console.WriteLine($"Name: {customer.Name}");
        Console.WriteLine($"CreatedAt: {customer.CreatedAt}");
        Console.WriteLine($"IsDeleted: {customer.IsDeleted}");

        // Mark updated
        customer.MarkAsUpdated();
        Console.WriteLine($"UpdatedAt after update: {customer.UpdatedAt}");

        // Soft delete
        customer.SoftDelete();
        Console.WriteLine($"IsDeleted after soft delete: {customer.IsDeleted}");

        Console.WriteLine();
        Console.WriteLine("=== Using interface references ===");

        // Using as IAuditable
        IAuditable auditable = customer;
        auditable.MarkAsUpdated();
        Console.WriteLine($"UpdatedAt via IAuditable: {customer.UpdatedAt}");

        // Using as ISoftDeletable
        ISoftDeletable deletable = customer;
        deletable.SoftDelete();
        Console.WriteLine($"IsDeleted via ISoftDeletable: {deletable.IsDeleted}");
    }
}
```

[Abstract class and Interface](Abstract%20class%20and%20Interface.md)

#csharp 
#oops