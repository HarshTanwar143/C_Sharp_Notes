## 1. SIMPLE EXPLANATION — What is OOP?

OOP is a **way of organizing code** around _things_ (objects) rather than _procedures_ (steps).

Instead of saying:

> "Do step 1, step 2, step 3..."

You say:

> "Here are the _things_ in my system. Each thing knows its own _data_ and its own _behavior_."

---

## 2. REAL-WORLD ANALOGY — The Bank

Think of a **real bank**.

A bank has:

- **Customers** — they have a name, account number, balance
- **Accounts** — they can deposit, withdraw, check balance
- **Tellers** — they process transactions
- **Loans** — they have interest rates, durations

Each of these is a **thing** with:

- Its own **data** (state)
- Its own **behavior** (what it can do)

Now imagine trying to write a bank system as just a list of functions:

```
ProcessDeposit()
ProcessWithdrawal()
CalculateInterest()
ValidateCustomer()
```

Within weeks, this becomes **spaghetti**. Nobody knows which data belongs to which function. Functions start sharing global state. Bugs explode.

OOP solves this by **grouping data + behavior together into a single unit** — the **class/object**.

---

## 3. THE FOUR PILLARS — But Taught the Right Way

Most tutorials list them. I will explain **WHY they exist and what problem each solves.**

---

### PILLAR 1 — ENCAPSULATION

#### What is it?

Hiding internal data and only exposing what is necessary.

#### What problem does it solve?

Without encapsulation, any part of your code can modify any data directly. This makes bugs **untraceable**.

#### Bad Practice — No Encapsulation

```csharp
// BAD: Everything is public. Anyone can set anything.
public class BankAccount
{
    public string Owner;       // Anyone can change the owner
    public decimal Balance;    // Anyone can set Balance = 999999
    public string AccountNumber;
}

// Somewhere in your codebase...
BankAccount acc = new BankAccount();
acc.Balance = -50000; // Nothing stops this. This is a disaster.
```

**Why is this dangerous?**

- Any developer, any class, any method can corrupt the data
- You have **zero control** over how your object's state changes
- You cannot add **validation logic** — there's no place to put it
- This is called **anemic domain model** — data bag with no behavior

#### Good Practice — Encapsulation

```csharp
public class BankAccount
{
    // private: ONLY this class can touch this field directly
    // readonly on _accountNumber: set once at construction, never changed
    private readonly string _accountNumber;
    
    // private backing field: nobody outside can directly touch balance
    private decimal _balance;
    
    // public property: controlled read access
    // No setter = nobody outside can set this directly
    public decimal Balance => _balance;
    
    public string AccountNumber => _accountNumber;
    
    // Constructor: the ONLY way to create a valid object
    // Forces you to provide required data upfront
    public BankAccount(string accountNumber, decimal initialBalance)
    {
        // Validation happens HERE, at the boundary
        if (string.IsNullOrWhiteSpace(accountNumber))
            throw new ArgumentException("Account number cannot be empty");
        
        if (initialBalance < 0)
            throw new ArgumentException("Initial balance cannot be negative");
        
        _accountNumber = accountNumber;
        _balance = initialBalance;
    }
    
    // Behavior is a METHOD — not a public field
    // This is the ONLY way to deposit
    public void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Deposit amount must be positive");
        
        _balance += amount;
        // Future: fire domain event, log audit trail, notify observers
    }
    
    public void Withdraw(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentException("Withdrawal amount must be positive");
        
        if (amount > _balance)
            throw new InvalidOperationException("Insufficient funds");
        
        _balance -= amount;
    }
}
```

**Now let's explain EVERY keyword:**

```csharp
private readonly string _accountNumber;
```

|Keyword|Why it exists|What breaks if removed|
|---|---|---|
|`private`|Only this class can access it. Enforcement of ownership.|Any external code modifies it directly. Validation bypassed.|
|`readonly`|Can only be assigned in constructor or declaration. Immutability guarantee.|Someone reassigns `_accountNumber` mid-lifecycle. Data corruption.|
|`string`|The type. CLR allocates a reference type on heap.|N/A|
|`_accountNumber`|Convention: underscore = private field. Distinguishes from parameters/locals.|Code confusion between `accountNumber` param and `accountNumber` field.|

---

#### Internal Mechanics — What actually happens in memory?

```csharp
var acc = new BankAccount("ACC001", 1000m);
```

1. CLR allocates memory on the **managed heap** for the `BankAccount` object
2. The constructor runs
3. `_accountNumber` gets a reference to the string `"ACC001"` (strings are interned or heap-allocated)
4. `_balance` gets value `1000` stored directly in the object's memory layout (decimal is a value type, stored inline)
5. `acc` (the variable) is a **reference** on the **stack** pointing to the heap object

```
Stack                    Heap
------                   ----
acc  ──────────────────► [BankAccount Object]
                          _accountNumber ──► "ACC001" (string on heap)
                          _balance: 1000  (inline, value type)
```

---

### PILLAR 2 — INHERITANCE

#### What is it?

A class can **inherit** data and behavior from another class.

#### What problem does it solve?

Code reuse. When multiple types share common behavior, don't repeat it.

#### Real-world analogy

A `SavingsAccount` and `CurrentAccount` are both `BankAccount`s. They share common behavior (deposit, withdraw) but have unique behavior (savings earns interest, current has overdraft).

```csharp
// BASE CLASS — common behavior for ALL account types
public class BankAccount
{
    protected decimal _balance; // protected: accessible in derived classes
    
    public decimal Balance => _balance;
    
    public BankAccount(decimal initialBalance)
    {
        if (initialBalance < 0)
            throw new ArgumentException("Initial balance cannot be negative");
        _balance = initialBalance;
    }
    
    // virtual: derived classes CAN override this behavior
    // but don't HAVE to
    public virtual void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive");
        _balance += amount;
    }
    
    public virtual void Withdraw(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive");
        if (amount > _balance) throw new InvalidOperationException("Insufficient funds");
        _balance -= amount;
    }
    
    // ToString override — everyone inherits this from object
    public override string ToString() => $"Balance: {_balance:C}";
}

// DERIVED CLASS — SavingsAccount IS-A BankAccount
public class SavingsAccount : BankAccount
{
    private decimal _interestRate;
    
    // base(...) calls the parent constructor
    // You MUST initialize the parent before the child
    public SavingsAccount(decimal initialBalance, decimal interestRate)
        : base(initialBalance)
    {
        _interestRate = interestRate;
    }
    
    // Additional behavior unique to savings
    public void ApplyInterest()
    {
        decimal interest = _balance * _interestRate;
        _balance += interest; // protected field from parent is accessible here
    }
    
    // Override Withdraw — savings may have withdrawal limits
    public override void Withdraw(decimal amount)
    {
        if (amount > _balance * 0.9m) // Can't withdraw more than 90%
            throw new InvalidOperationException("Savings withdrawal limit exceeded");
        
        base.Withdraw(amount); // Call parent's logic too — DRY principle
    }
}

// DERIVED CLASS — CurrentAccount IS-A BankAccount
public class CurrentAccount : BankAccount
{
    private decimal _overdraftLimit;
    
    public CurrentAccount(decimal initialBalance, decimal overdraftLimit)
        : base(initialBalance)
    {
        _overdraftLimit = overdraftLimit;
    }
    
    // Override: CurrentAccount allows overdraft
    public override void Withdraw(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive");
        
        // Different rule: can go negative up to overdraft limit
        if (amount > _balance + _overdraftLimit)
            throw new InvalidOperationException("Overdraft limit exceeded");
        
        _balance -= amount; // Note: balance can go negative here
    }
}
```

**Every keyword explained:**

|Keyword|Why|
|---|---|
|`protected`|Accessible in this class AND derived classes. Not to outside world.|
|`virtual`|Tells CLR: this method CAN be overridden. Without it, `override` won't compile.|
|`override`|Tells CLR: replace parent's implementation with this one for this type.|
|`base(...)`|Call parent constructor. Ensures parent is initialized first.|
|`base.Withdraw(amount)`|Call parent's version of method inside override. Reuse parent logic.|
|`: BankAccount`|This class inherits from BankAccount. Gets all non-private members.|

---

#### Internal Mechanics — Virtual Dispatch Table (vTable)

This is where it gets deeply interesting.

When you declare a method as `virtual`, the CLR creates a **virtual dispatch table (vTable)** for each type.

```csharp
BankAccount account = new SavingsAccount(1000m, 0.05m);
account.Withdraw(100m); // Which Withdraw is called?
```

At **runtime**, the CLR:

1. Looks at the **actual type** of the object (`SavingsAccount`)
2. Looks up its vTable
3. Finds `SavingsAccount.Withdraw`
4. Calls it — NOT `BankAccount.Withdraw`

This is called **polymorphic dispatch** or **late binding**.

```
BankAccount vTable:          SavingsAccount vTable:
┌──────────────┐             ┌──────────────────────┐
│ Deposit      │──► BankAcc  │ Deposit              │──► BankAcc (inherited)
│ Withdraw     │──► BankAcc  │ Withdraw             │──► SavingsAccount (overridden!)
│ ToString     │──► BankAcc  │ ToString             │──► BankAcc (inherited)
│ ApplyInterest│──► N/A      │ ApplyInterest        │──► SavingsAccount
└──────────────┘             └──────────────────────┘
```

**Performance note:** Virtual calls have a tiny overhead compared to non-virtual calls because of this table lookup. In 99.9% of applications this is irrelevant. But in extremely hot paths (millions of calls per second), you'd consider `sealed` classes.

---

### PILLAR 3 — ABSTRACTION

#### What is it?

Hiding **how** something works and only exposing **what** it does.

#### What problem does it solve?

Consumers of your class don't need to know your implementation. They just need to know: "I can call Deposit(), Withdraw(), check Balance. I don't care how it works internally."

Abstraction is achieved via:

1. **Abstract classes** — partial implementation, some methods must be overridden
2. **Interfaces** — pure contract, zero implementation (pre C# 8)

```csharp
// ABSTRACT CLASS — "I am a BankAccount but I'm incomplete"
// You CANNOT instantiate this directly
// new BankAccount() → compiler error
public abstract class BankAccount
{
    protected decimal _balance;
    
    public decimal Balance => _balance;
    
    // Concrete method — implementation provided
    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive");
        _balance += amount;
    }
    
    // Abstract method — NO implementation
    // Derived class MUST implement this
    // Forces every account type to define its own withdrawal rules
    public abstract void Withdraw(decimal amount);
    
    // Abstract property — derived class must implement
    public abstract string AccountType { get; }
}

// INTERFACE — pure contract
// "If you implement me, you MUST provide these behaviors"
public interface IInterestBearing
{
    decimal InterestRate { get; }
    void ApplyInterest();
}

public interface IAuditable
{
    IReadOnlyList<string> GetAuditLog();
}

// SavingsAccount fulfills MULTIPLE contracts
public class SavingsAccount : BankAccount, IInterestBearing, IAuditable
{
    private readonly decimal _interestRate;
    private readonly List<string> _auditLog = new();
    
    public SavingsAccount(decimal initialBalance, decimal interestRate)
    {
        _balance = initialBalance;
        _interestRate = interestRate;
    }
    
    // MUST implement abstract member
    public override void Withdraw(decimal amount)
    {
        if (amount > _balance * 0.9m)
            throw new InvalidOperationException("Savings withdrawal limit exceeded");
        
        _balance -= amount;
        _auditLog.Add($"Withdrew {amount:C} at {DateTime.UtcNow}");
    }
    
    // MUST implement abstract property
    public override string AccountType => "Savings";
    
    // IInterestBearing implementation
    public decimal InterestRate => _interestRate;
    
    public void ApplyInterest()
    {
        decimal interest = _balance * _interestRate;
        _balance += interest;
        _auditLog.Add($"Applied interest {interest:C} at {DateTime.UtcNow}");
    }
    
    // IAuditable implementation
    public IReadOnlyList<string> GetAuditLog() => _auditLog.AsReadOnly();
}
```

---

#### Abstract Class vs Interface — The Deep Distinction

This is a **senior-level question** in every interview.

||Abstract Class|Interface|
|---|---|---|
|**Purpose**|Partial base implementation|Pure behavioral contract|
|**Instantiation**|Cannot instantiate directly|Cannot instantiate directly|
|**Methods**|Can have concrete + abstract|C# 8+: can have default implementations; historically: none|
|**State**|Can have fields, state|Cannot have instance fields (pre C# 11)|
|**Inheritance**|Single only (C# single inheritance)|Multiple interfaces allowed|
|**Constructor**|Can have constructor|Cannot have constructor|
|**Use when**|Shared base behavior + force override|Define capability contract|

**Real question: Why does C# not support multiple inheritance of classes?**

The **Diamond Problem**:

```
        Animal
       /      \
    Dog        Cat
       \      /
        DogCat (???)
```

If both `Dog` and `Cat` override `MakeSound()`, and `DogCat` inherits from both — which `MakeSound()` does `DogCat` use? This is ambiguous and leads to bugs. C++ allows it and it's notoriously complex. C# designers chose to forbid it for classes. **Interfaces solve this** because they traditionally had no state or implementation to conflict.

---

### PILLAR 4 — POLYMORPHISM

#### What is it?

One interface, many behaviors. The same method call behaves differently based on the actual type at runtime.

```csharp
// The power of polymorphism
public class Bank
{
    private readonly List<BankAccount> _accounts = new();
    
    public void AddAccount(BankAccount account) => _accounts.Add(account);
    
    // This method doesn't know or care if it's Savings, Current, etc.
    // It just calls Withdraw on each — runtime decides which implementation runs
    public void ProcessMonthEndWithdrawals(decimal amount)
    {
        foreach (var account in _accounts)
        {
            try
            {
                account.Withdraw(amount); // POLYMORPHIC CALL
                Console.WriteLine($"{account.AccountType}: Withdrew {amount:C}");
            }
            catch (InvalidOperationException ex)
            {
                Console.WriteLine($"{account.AccountType}: Failed — {ex.Message}");
            }
        }
    }
}

// Usage
var bank = new Bank();
bank.AddAccount(new SavingsAccount(1000m, 0.05m));
bank.AddAccount(new CurrentAccount(500m, 200m));

bank.ProcessMonthEndWithdrawals(450m);
// Output varies per account type — SAME code, DIFFERENT behavior
```

---

## 4. WHY THE INDUSTRY USES OOP THIS WAY

1. **Maintainability** — Changes to `SavingsAccount` don't affect `CurrentAccount`
2. **Extensibility** — Add `FixedDepositAccount` without touching existing code (Open/Closed Principle)
3. **Testability** — Mock `IInterestBearing` in tests without real database
4. **Domain modeling** — Code maps to real-world concepts. New developers understand it faster.
5. **Team scalability** — Teams can work on different classes simultaneously without conflicts

---

## 5. COMMON MISTAKES — What junior developers do wrong

```csharp
// MISTAKE 1: Using inheritance for code reuse when there's no IS-A relationship
// Wrong: Logger is NOT-A BankAccount
public class BankAccountWithLogging : BankAccount
{
    // Just wanted to reuse logging... but this is WRONG
    // Use composition instead
}

// CORRECT: Compose, don't inherit for unrelated behavior
public class BankAccount
{
    private readonly ILogger<BankAccount> _logger; // INJECT the logger
    
    public BankAccount(ILogger<BankAccount> logger)
    {
        _logger = logger;
    }
}

// MISTAKE 2: Making everything public "to be safe"
public class BankAccount
{
    public decimal _balance; // NEVER. This is not encapsulation.
}

// MISTAKE 3: Anemic domain model
// Data class with no behavior
public class BankAccount
{
    public decimal Balance { get; set; }
    public string AccountNumber { get; set; }
}

// Then a SEPARATE class does all the work
public class BankAccountService
{
    public void Withdraw(BankAccount account, decimal amount)
    {
        account.Balance -= amount; // Business logic OUTSIDE the domain object
    }
}
// This is procedural programming disguised as OOP
```

---

## 6. INTERVIEW QUESTIONS

1. What is the difference between `abstract` and `virtual`?
2. Can an abstract class have a constructor? Why?
3. Can you instantiate an interface? Why not?
4. What is the diamond problem?
5. What's the difference between overriding and overloading?
6. What is the difference between `is-a` and `has-a` relationships?
7. What does `sealed` keyword do and why would you use it?
8. Why is composition often preferred over inheritance?
9. What happens internally when a virtual method is called?
10. What does `protected internal` mean?

---

## 🔥 CROSS-QUESTIONS FOR YOU — Answer these before we go further

**Question 1:**

```csharp
BankAccount account = new SavingsAccount(1000m, 0.05m);
account.Withdraw(100m);
```

_Which `Withdraw` method gets called — `BankAccount.Withdraw` or `SavingsAccount.Withdraw`? WHY? Explain the internal mechanism._

---

**Question 2:** I have this code:

```csharp
public class BankAccount
{
    public decimal Balance;  // public field, no encapsulation
}
```

_What are THREE specific problems this causes in a production application? Think about validation, thread safety, and future changes._

---

**Question 3:** _When should you use an **abstract class** instead of an **interface**? Give me a real-world scenario where choosing the wrong one causes problems._

---

**Question 4:**

```csharp
public class SavingsAccount : BankAccount
{
    public new void Withdraw(decimal amount) // Notice: 'new', not 'override'
    {
        Console.WriteLine("SavingsAccount Withdraw");
    }
}

BankAccount acc = new SavingsAccount(1000m, 0.05m);
acc.Withdraw(100m);
```

_What does `new` do here instead of `override`? What gets printed? Why is this dangerous?_

---

**Question 5 — Scenario:** Your team lead says:

> _"Let's make all our domain classes inherit from a single `BaseEntity` class that has logging, auditing, and database ID built in."_

_What are the problems with this approach? What would YOU recommend instead?_

---

## 🎯 MINI ASSIGNMENT

Build this in code:

1. An abstract class `Vehicle` with:
    
    - Properties: `Make`, `Model`, `FuelLevel`
    - Concrete method: `Refuel(decimal amount)`
    - Abstract method: `StartEngine()`
    - Abstract property: `VehicleType`
2. Two derived classes: `Car` and `ElectricCar`
    
    - `Car`: starts with fuel check, logs "Vroom"
    - `ElectricCar`: starts with battery check, overrides `Refuel` to throw exception (electric cars don't use fuel)
3. An interface `INavigable` with method `Navigate(string destination)`
    
4. Make `Car` implement `INavigable`
    
5. Write a method that accepts `List<Vehicle>` and calls `StartEngine()` on all — demonstrating polymorphism
    

**Predict before you code:** What happens when you call `Refuel()` on an `ElectricCar` reference stored as `Vehicle`?

---

Answer my cross-questions first. Then show me your assignment code. I will review it line by line and challenge you on it.