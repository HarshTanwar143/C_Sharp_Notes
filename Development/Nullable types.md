Nullable are a feature used to represent variables that can either hold a normal value or a `null` value.

There are two distinct types of nullability in .NET:
#### 1. Nullable value types
By default, value types like `int`, `bool`, and `DateTime` cannot be null because they always hold a value. A **nullable value type** wraps these types so they can also represent `null`. You make them nullable by suffixing `?` after the data type.

#### 2. Nullable reference types (c# 8.0+)
Reference types (like `string` or custom classes) have always been nullable in .NET.
- `string`: Interpreted as "this should never be null." The compiler warns you if you try to assign null to it.
- `string?`: Explicitly tells the compiler and other developers that "this can be null," requiring you to check for null before using it.

# Nullable in Models:

## 1. Nullable string
```c#
public string? Name { get; set; }
```
Meaning:
- `Name` is allowed to be `null`
- Compiler will NOT warn if it is unset
- Consumers must handle possible `null`
You can use it like this:
```c#
model.Name = null; // valid
```

Typical use:
- Optional DB columns
- Optional API fields
- Search filters
- Nullable DTO values
## 2. Non-nullable string with null-forgiving operator

```c#
public string Name { get; set; } = null!;
```

Meaning:
- `Name` is declared as NON-nullable
- But you're telling compiler:
	“Trust me, this will be initialized later.”

`null!` suppresses nullable warnings.
The property still starts as `null` at runtime until assigned.
Typical use:
- EF Core entities
- ORM models
- Deserialization scenarios
- Framework-populated properties

Example:
```c#
public class User {  
	public string Name { get; set; } = null!;
}
```

EF Core populates `Name` after materialization.
Without `null!`, compiler warns:
> Non-nullable property must contain a non-null value...

Important:
This can still throw runtime exceptions if not initialized.

```c#
Console.WriteLine(user.Name.Length); // crash if still null
```

## 3. Empty string literal

```c#
public string Name { get; set; } = "";
```
Above you could also have used `string.Empty`. Both `""` and `string.Empty` are same.

Meaning:
- Non-nullable
- Default value is empty string
- Safe immediately

Typical use:
- ViewModels
- DTOs
- Forms
- Simple domain models

This guarantees:
```c#
Name != null
```
## Practical comparison

| Declaration                  | Nullable? | Runtime Initial Value | Compiler Warning? | Typical Use           |
| ---------------------------- | --------- | --------------------- | ----------------- | --------------------- |
| `string? Name`               | Yes       | `null`                | No                | Optional data         |
| `string Name = null!`        | No        | `null`                | Suppressed        | EF Core / serializers |
| `string Name = ""`           | No        | `""`                  | No                | Safe defaults         |

Tags:
#dotnet 
#csharp 
#database 