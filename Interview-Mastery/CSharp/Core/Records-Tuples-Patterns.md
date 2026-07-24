# C# Records, Tuples & Pattern Matching — Interview Mastery

---

## Records (C# 9+)

### What Are Records?

- Records are **reference types** that provide value-based equality semantics
- They are designed for types where **structural equality** matters more than reference equality
- Built-in support for immutability, non-destructive mutation, and concise syntax
- Records automatically generate `Equals`, `GetHashCode`, and `ToString` based on properties
- They are still reference types (allocated on the heap), but behave like value types for equality

```csharp
// Basic record definition
public record Person
{
    public string Name { get; init; }
    public int Age { get; init; }
}

// Usage
var alice = new Person { Name = "Alice", Age = 30 };
var bob = new Person { Name = "Alice", Age = 30 };

Console.WriteLine(alice.Equals(bob));   // True — value-based equality
Console.WriteLine(alice == bob);        // True — uses value-based equality
Console.WriteLine(ReferenceEquals(alice, bob)); // False — different objects
```

### Record vs Class

| Feature | Record | Class |
|---|---|---|
| Type | Reference type | Reference type |
| Equality | Value-based (property-by-property) | Reference-based (default) |
| ToString | Auto-generated with properties | `System.Object.ToString()` |
| Deconstruction | Automatic (positional records) | Must implement manually |
| Immutability | Encouraged (init-only setters) | Mutable by default |
| Inheritance | Supports `record` and `record class` | Supports `class` |
| `with` expressions | Supported | Not supported |
| Sealed by default | No (but can be `sealed record`) | No |

```csharp
// Class — reference equality by default
public class PersonClass
{
    public string Name { get; set; }
    public int Age { get; set; }
}

var a = new PersonClass { Name = "Alice", Age = 30 };
var b = new PersonClass { Name = "Alice", Age = 30 };
Console.WriteLine(a.Equals(b));  // False — reference equality
Console.WriteLine(a == b);       // False — reference equality

// Record — value equality by default
var c = new Person { Name = "Alice", Age = 30 };
var d = new Person { Name = "Alice", Age = 30 };
Console.WriteLine(c.Equals(d));  // True — value equality
Console.WriteLine(c == d);       // True — value equality
```

### Immutability with Init-Only Setters

- Records use `init` accessors instead of `set` to enforce immutability after construction
- You can set values during object initialization, but not afterward
- This is the key mechanism that makes records naturally immutable
- You can still have `{ get; init; }` properties that allow setting during initialization only

```csharp
public record Address(string Street, string City, string ZipCode);

var addr = new Address("123 Main St", "Springfield", "62701");

// This would fail — init-only setters prevent modification
// addr.Street = "456 Oak Ave";  // Compiler error

// Must use 'with' expression for non-destructive mutation
var newAddr = addr with { Street = "456 Oak Ave" };
Console.WriteLine(newAddr);  // Address { Street = "456 Oak Ave", City = "Springfield", ZipCode = "62701" }
Console.WriteLine(addr);     // Address { Street = "123 Main St", City = "Springfield", ZipCode = "62701" } — unchanged
```

### Positional Records

- Positional records use a primary constructor to define properties automatically
- Compiler generates: properties with `{ get; init; }`, a constructor, and `Deconstruct` method
- The primary constructor parameters become the record's properties
- You can add additional properties or methods beyond what the primary constructor defines

```csharp
// Positional record — compact syntax
public record Employee(string Name, string Department, decimal Salary);

// This is equivalent to the verbose version:
public record EmployeeVerbose
{
    public string Name { get; init; }
    public string Department { get; init; }
    public decimal Salary { get; init; }

    public EmployeeVerbose(string name, string department, decimal salary)
    {
        Name = name;
        Department = department;
        Salary = salary;
    }

    // Deconstruct is auto-generated for positional records
    // public void Deconstruct(out string name, out string department, out decimal salary)
    // {
    //     name = Name;
    //     department = Department;
    //     salary = Salary;
    // }
}

// Usage
var emp = new Employee("Bob", "Engineering", 95000m);

// Deconstruction
var (name, department, salary) = emp;
Console.WriteLine($"{name} works in {department} earning {salary:C}");

// With expression
var promoted = emp with { Salary = 105000m };
Console.WriteLine($"{promoted.Name} now earns {promoted.Salary:C}");
```

### Record Structs (C# 10)

- C# 10 introduced `record struct` — a value type with record semantics
- Use `record struct` for lightweight, stack-allocated types with value equality
- Can be `readonly record struct` for full immutability
- Avoids heap allocation for small, short-lived types

```csharp
// Record struct — value type with record equality
public record struct Coordinate(double Latitude, double Longitude);

// Readonly record struct — fully immutable value type
public readonly record struct Temperature(double Value, string Unit);

// Usage
var coord1 = new Coordinate(40.7128, -74.0060);
var coord2 = new Coordinate(40.7128, -74.0060);

Console.WriteLine(coord1.Equals(coord2));  // True — value-based equality
Console.WriteLine(coord1 == coord2);       // True

// Stack-allocated — no GC pressure
var temp = new Temperature(98.6, "F");
var (value, unit) = temp;  // Deconstruction works too
Console.WriteLine($"{value}°{unit}");
```

### With Expressions for Non-Destructive Mutation

- `with` expressions create a **copy** of a record with specified properties changed
- The original record remains unchanged (non-destructive)
- Useful for working with immutable data while producing new modified copies
- Can use `with` on both positional and non-positional records

```csharp
public record Order(string Id, string Product, int Quantity, decimal UnitPrice, DateTime OrderDate);

var original = new Order("ORD-001", "Laptop", 1, 999.99m, DateTime.Now);

// Change a single property
var discounted = original with { UnitPrice = 899.99m };

// Change multiple properties
var updated = original with
{
    Quantity = 2,
    Product = "Laptop Pro"
};

// Copy with no changes (useful for creating a starting point)
var clone = original with { };

Console.WriteLine(original);   // Order { Id = ORD-001, Product = Laptop, ... }
Console.WriteLine(discounted); // Order { Id = ORD-001, Product = Laptop, ..., UnitPrice = 899.99 }
Console.WriteLine(updated);    // Order { Id = ORD-001, Product = Laptop Pro, Quantity = 2, ... }
```

### Value-Based Equality

- Records override `Equals` and `GetHashCode` to compare by property values
- Two records are equal if all their properties are equal
- This applies across inheritance hierarchies as well
- Value equality is the default — you can override it if needed

```csharp
public record Point(int X, int Y);

var p1 = new Point(1, 2);
var p2 = new Point(1, 2);
var p3 = new Point(3, 4);

Console.WriteLine(p1.Equals(p2));       // True
Console.WriteLine(p1 == p2);            // True
Console.WriteLine(p1.Equals(p3));       // False
Console.WriteLine(p1.GetHashCode() == p2.GetHashCode()); // True — same values = same hash

// In a HashSet, value equality matters
var points = new HashSet<Point> { p1 };
Console.WriteLine(points.Contains(p2)); // True — HashSet uses value equality for records
```

### When to Use Records

- **DTOs (Data Transfer Objects)** — API request/response models
- **API contracts** — clear, immutable structures for serialization
- **Immutable data** — configuration, settings, messages
- **Pattern matching** — records work well with pattern matching expressions
- **Value objects** — domain-driven design concepts like Money, Address, etc.
- **Data classes** — anything where you care about the data, not the identity

```csharp
// API request/response DTOs
public record CreateOrderRequest(string ProductId, int Quantity, string ShippingAddress);
public record OrderResponse(string OrderId, decimal Total, string Status);

// Domain value objects
public record Money(decimal Amount, string Currency)
{
    public Money Add(Money other)
    {
        if (Currency != other.Currency)
            throw new InvalidOperationException("Cannot add different currencies");
        return this with { Amount = Amount + other.Amount };
    }
}

// Configuration
public record DatabaseConfig(string ConnectionString, int MaxRetries, TimeSpan Timeout);

// Usage
var config = new DatabaseConfig("Server=localhost;Database=MyDb", 3, TimeSpan.FromSeconds(30));
// config is immutable — safe to pass around without worrying about modification
```

### Equality Contracts and Inheritance

- Records support inheritance, but equality semantics can be tricky
- A derived record is only equal to another record of the **same type** with the same property values
- You can override equality behavior if needed using `EqualityContract`
- Sealed records prevent inheritance-related equality issues

```csharp
public record Shape(string Color);
public record Circle(string Color, double Radius) : Shape(Color);
public record Rectangle(string Color, double Width, double Height) : Shape(Color);

var circle1 = new Circle("Red", 5.0);
var circle2 = new Circle("Red", 5.0);
var rect = new Rectangle("Red", 5.0, 10.0);

Console.WriteLine(circle1.Equals(circle2));  // True — same type, same values
Console.WriteLine(circle1.Equals(rect));     // False — different types

// Sealed record — prevents inheritance
public sealed record Coordinate(double X, double Y);
// public record SpecialCoordinate(double X, double Y) : Coordinate(X, Y); // Compiler error if Coordinate is sealed
```

---

## Init-Only Setters (C# 9)

### What Are Init-Only Setters?

- `init` is a property accessor introduced in C# 9 that allows setting a property **only during initialization**
- After the object is fully constructed (constructor or object initializer completes), the property becomes **read-only**
- Provides a middle ground between `{ get; set; }` (fully mutable) and `{ get; }` (fully immutable from construction)

```csharp
public class Person
{
    public string Name { get; init; }  // Can only be set during initialization
    public int Age { get; init; }      // Can only be set during initialization
}

// Valid — setting during object initialization
var person = new Person { Name = "Alice", Age = 30 };

// Invalid — cannot set after initialization
// person.Name = "Bob";  // Compiler error: init-only property can only be assigned in an object initializer
```

### How They Differ from get; set;

- `{ get; set; }` — allows reading and writing at any time, any place
- `{ get; init; }` — allows reading at any time, but writing only during object initialization
- `{ get; }` — allows reading only; must be set in constructor or field initializer
- `init` is a **compile-time enforcement** — no runtime overhead

```csharp
// Fully mutable
public class MutablePerson
{
    public string Name { get; set; }   // Anyone can change this at any time
    public int Age { get; set; }       // Anyone can change this at any time
}

// Immutable after construction
public class ImmutablePerson
{
    public string Name { get; init; }  // Can only be set during initialization
    public int Age { get; init; }      // Can only be set during initialization
}

var mutable = new MutablePerson { Name = "Alice", Age = 30 };
mutable.Name = "Bob";  // Works fine — mutable

var immutable = new ImmutablePerson { Name = "Alice", Age = 30 };
// immutable.Name = "Bob";  // Compiler error — init-only
```

### Why They Are Useful for Immutability

- Immutability simplifies reasoning about code — values don't change unexpectedly
- Thread-safe by default — no need for locks on init-only properties
- Essential for records — records use init-only setters as their default property accessor
- Enables non-destructive mutation via `with` expressions
- Great for API models where you want to define values but prevent modification

```csharp
public record UserSettings(string Theme, bool DarkMode, int FontSize);

// Thread-safe — multiple threads can read without synchronization
var settings = new UserSettings("Monokai", true, 14);

// Create a modified copy for a specific user
var userSettings = settings with { DarkMode = false, FontSize = 16 };

// Original settings remain unchanged
Console.WriteLine(settings.DarkMode);   // True
Console.WriteLine(userSettings.DarkMode); // False

// Safe to pass across thread boundaries — no mutation possible
Task.Run(() => ProcessSettings(settings));
```

### Limitations (Can Only Set During Initialization)

- Init-only setters can only be assigned via object initializers or the constructor
- You cannot set them in methods, events, or after the object is created
- Backing fields are generated as `readonly` fields — the compiler enforces this
- Serialization frameworks may need special configuration to work with init-only setters

```csharp
public record Person(string Name, int Age);

var person = new Person { Name = "Alice", Age = 30 };  // Valid

// These all fail:
// person.Name = "Bob";                    // Compiler error
// person with { Name = "Bob" };           // Works — creates new record (non-destructive mutation)

// Serialization note: System.Text.Json supports init-only setters
// Newtonsoft.Json requires [JsonConstructor] or special settings
```

### init vs readonly in Constructors

- `readonly` fields can only be set in the constructor and remain immutable thereafter
- `init` setters can be set in the constructor **and** in object initializers
- `init` is more flexible because it works with object initializer syntax
- Both provide immutability guarantees after construction is complete

```csharp
// readonly fields — constructor only
public class PersonReadonly
{
    public readonly string Name;
    public readonly int Age;

    public PersonReadonly(string name, int age)
    {
        Name = name;
        Age = age;
    }
}

// init setters — constructor + object initializers
public class PersonInit
{
    public string Name { get; init; }
    public int Age { get; init; }
}

// Object initializer syntax — only works with init
var person1 = new PersonInit { Name = "Alice", Age = 30 };  // Clean syntax

// Constructor syntax — works with both
var person2 = new PersonReadonly("Alice", 30);
var person3 = new PersonInit { Name = "Alice", Age = 30 };
```

---

## Tuples

### ValueTuple<T1,T2> (C# 7+) vs Tuple<T1,T2> (Pre-C# 7)

- `Tuple<T1, T2>` (System.Tuple) — reference type, heap-allocated, immutable, verbose syntax
- `ValueTuple<T1, T2>` (System.ValueTuple) — value type, stack-allocated (usually), lightweight
- C# 7 introduced tuple syntax: `(string name, int age)` — syntactic sugar for `ValueTuple`
- `ValueTuple` is the preferred choice for performance and ergonomics

```csharp
// Old-style Tuple (reference type, heap-allocated)
Tuple<string, int> oldTuple = Tuple.Create("Alice", 30);
Console.WriteLine(oldTuple.Item1);  // "Alice" — not descriptive
Console.WriteLine(oldTuple.Item2);  // 30 — not descriptive

// New-style ValueTuple (value type, stack-allocated)
(string name, int age) newTuple = ("Alice", 30);
Console.WriteLine(newTuple.name);   // "Alice" — descriptive
Console.WriteLine(newTuple.age);    // 30 — descriptive

// Shorthand syntax
var shortTuple = ("Alice", 30);  // Type is (string, int)
```

### Named Tuples vs Unnamed Tuples

- Named tuples have descriptive element names: `(string Name, int Age)`
- Unnamed tuples use default names: `Item1`, `Item2`, etc.
- Named tuples improve readability and reduce bugs
- Both are `ValueTuple` under the hood

```csharp
// Named tuple — descriptive, readable
(string Name, int Age, string Department) employee = ("Bob", 25, "Engineering");
Console.WriteLine(employee.Name);        // "Bob"
Console.WriteLine(employee.Department);  // "Engineering"

// Unnamed tuple — less readable
(string, int, string) unnamed = ("Bob", 25, "Engineering");
Console.WriteLine(unnamed.Item1);        // "Bob" — what is Item1?
Console.WriteLine(unnamed.Item3);        // "Engineering" — error-prone

// Named tuple as return type
public (string FullName, int Age, bool IsActive) GetEmployee(int id)
{
    return ("Alice Smith", 30, true);
}

var emp = GetEmployee(1);
Console.WriteLine($"{emp.FullName} is {(emp.IsActive ? "active" : "inactive")}");
```

### Deconstruction of Tuples

- Tuples support deconstruction — extracting elements into separate variables
- Can deconstruct into existing variables or new variables
- Works with both named and unnamed tuples

```csharp
// Deconstruction into new variables
var person = ("Alice", 30);
var (name, age) = person;
Console.WriteLine($"{name} is {age} years old");

// Deconstruction into existing variables
string n;
int a;
(n, a) = person;
Console.WriteLine($"{n} is {a} years old");

// Deconstruction in method parameters
public void PrintPerson((string Name, int Age) person)
{
    var (name, age) = person;
    Console.WriteLine($"{name} is {age}");
}

// Deconstruction in foreach
var people = new[] { ("Alice", 30), ("Bob", 25) };
foreach (var (personName, personAge) in people)
{
    Console.WriteLine($"{personName} is {personAge}");
}

// Deconstruction with discard
var (_, ageOnly) = ("Alice", 30);
Console.WriteLine(ageOnly);  // 30
```

### Tuple Equality

- `ValueTuple` supports `==` and `!=` operators in C# 13+ (previously required `Equals`)
- Comparison is element-by-element
- Named and unnamed tuples can be compared (names are ignored for equality)

```csharp
var tuple1 = (Name: "Alice", Age: 30);
var tuple2 = (Name: "Alice", Age: 30);
var tuple3 = (Name: "Bob", Age: 25);

// C# 13+ supports direct == comparison
Console.WriteLine(tuple1 == tuple2);  // True — all elements match
Console.WriteLine(tuple1 == tuple3);  // False

// Pre-C# 13: use Equals
Console.WriteLine(tuple1.Equals(tuple2));  // True

// Different named tuples can still be compared
(string a, int b) unnamed = ("Alice", 30);
(string name, int age) named = ("Alice", 30);
Console.WriteLine(unnamed.Equals(named));  // True — names don't affect equality
```

### When to Use Tuples vs DTOs

- **Tuples** — quick, temporary groupings of related values (internal use)
- **DTOs** — public API contracts, serialized data, persistent models
- Tuples lack documentation — no XML comments on elements
- Tuples don't support methods, validation, or computed properties
- DTOs (records) provide better encapsulation and domain modeling

```csharp
// Tuples — good for internal, temporary data
public (int Min, int Max, double Average) CalculateStatistics(int[] numbers)
{
    return (numbers.Min(), numbers.Max(), numbers.Average());
}

var stats = CalculateStatistics(new[] { 1, 2, 3, 4, 5 });
Console.WriteLine($"Min: {stats.Min}, Max: {stats.Max}, Avg: {stats.Average}");

// Records — better for public APIs and persistent data
public record ProductResponse(string Id, string Name, decimal Price, int Stock);

// Records support validation, methods, documentation
public record Money(decimal Amount, string Currency)
{
    public override string ToString() => $"{Amount} {Currency}";
}
```

### Performance (Stack-Allocated vs Heap-Allocated)

- `ValueTuple` is typically stack-allocated — no GC pressure
- `Tuple` is always heap-allocated — adds GC pressure
- `ValueTuple` is a value type — copying creates a new copy on the stack
- For large tuples, consider whether stack space is a concern

```csharp
// ValueTuple — stack-allocated (value type)
// Typically no heap allocation
var small = (1, "hello");  // Stack-allocated

// Tuple — heap-allocated (reference type)
// Creates an object on the heap
var oldStyle = Tuple.Create(1, "hello");  // Heap-allocated

// Value tuples with many elements may spill to heap
// when captured by closures or async methods
var huge = (1, 2, 3, 4, 5, 6, 7, 8, 9, 10);  // May use heap depending on context
```

---

## Pattern Matching (C# 7+)

### is Pattern

- `is` pattern tests if a value matches a type and optionally extracts it into a variable
- Commonly used with null checks and type checks
- Eliminates verbose casting patterns

```csharp
// Type pattern — check type and cast in one step
object obj = "Hello";
if (obj is string s)
{
    Console.WriteLine(s.Length);  // 5 — 's' is already cast to string
}

// Constant pattern — check against a specific value
if (obj is "Hello")
{
    Console.WriteLine("Exact match!");
}

// Null check pattern
if (obj is not null)
{
    Console.WriteLine("Object is not null");
}

// Property pattern (C# 8+)
if (obj is string { Length: > 5 } longString)
{
    Console.WriteLine($"Long string: {longString}");
}
```

### switch Expression

- Switch expressions provide a concise, functional-style alternative to switch statements
- Must be exhaustive — must handle all possible cases or use `_` discard
- Returns a value — can be assigned to a variable or used inline
- Supports pattern matching in each arm

```csharp
// Basic switch expression
string GetDayName(int day) => day switch
{
    1 => "Monday",
    2 => "Tuesday",
    3 => "Wednesday",
    4 => "Thursday",
    5 => "Friday",
    6 => "Saturday",
    7 => "Sunday",
    _ => "Invalid day"  // Discard — matches everything else
};

// With pattern matching
string ClassifyNumber(int number) => number switch
{
    > 0 and < 10 => "Small positive",
    >= 10 and < 100 => "Medium positive",
    >= 100 => "Large positive",
    0 => "Zero",
    < 0 => "Negative"
};

// Using with records
public record Shape(string Type, double Dimension1, double Dimension2);

double CalculateArea(Shape shape) => shape switch
{
    Shape("Circle", var radius, _) => Math.PI * radius * radius,
    Shape("Rectangle", var w, var h) => w * h,
    Shape("Triangle", var b, var h) => 0.5 * b * h,
    _ => throw new ArgumentException("Unknown shape")
};
```

### Property Patterns

- Property patterns match on properties of an object
- Use `{ PropertyName: pattern }` syntax
- Can be nested for complex objects
- Available from C# 8+

```csharp
// Simple property pattern
if (person is { Name: "Alice" })
{
    Console.WriteLine("Found Alice!");
}

// Multiple properties
if (person is { Name: "Alice", Age: > 25 })
{
    Console.WriteLine("Alice is older than 25");
}

// Nested property patterns
public record Address(string City, string ZipCode);
public record PersonWithAddress(string Name, Address HomeAddress);

if (person is { HomeAddress: { City: "Springfield" } })
{
    Console.WriteLine("Lives in Springfield");
}

// Property pattern in switch expression
string DescribePerson(PersonWithAddress p) => p switch
{
    { Name: "Admin", HomeAddress: { City: "HQ" } } => "Admin at headquarters",
    { Name: "Admin" } => "Admin elsewhere",
    { Age: > 65 } => "Senior",
    { Age: < 18 } => "Minor",
    _ => "Regular person"
};
```

### Positional Patterns

- Positional patterns use the `Deconstruct` method of a type to match on positional elements
- Works with records, tuples, and any type with a `Deconstruct` method
- Use `(pattern1, pattern2, ...)` syntax

```csharp
// With records
public record Point(double X, double Y);

string DescribePoint(Point p) => p switch
{
    (0, 0) => "Origin",
    (var x, 0) => $"On X-axis at {x}",
    (0, var y) => $"On Y-axis at {y}",
    (var x, var y) when x == y => $"On diagonal at ({x}, {y})",
    (var x, var y) => $"At ({x}, {y})"
};

// With tuples
string ClassifyTuple((int X, int Y) point) => point switch
{
    (0, 0) => "Origin",
    (var x, > 0) => $"Positive X quadrant: {x}",
    (< 0, var y) => $"Negative X quadrant: {y}",
    _ => "Other"
};

// Nested positional patterns
public record Line(Point Start, Point End);

string DescribeLine(Line line) => line switch
{
    Line((0, 0), var end) => $"From origin to {end}",
    Line(var start, (0, 0)) => $"From {start} to origin",
    _ => $"Line from {line.Start} to {line.End}"
};
```

### Relational Patterns

- Relational patterns use comparison operators: `<`, `>`, `<=`, `>=`
- Chain multiple relational patterns with `and` and `or`
- Available from C# 9+

```csharp
// Basic relational patterns
string GetTemperatureDescription(double temp) => temp switch
{
    < 0 => "Freezing",
    >= 0 and < 15 => "Cold",
    >= 15 and < 25 => "Comfortable",
    >= 25 and < 35 => "Warm",
    >= 35 => "Hot"
};

// Grading system
char GetGrade(int score) => score switch
{
    >= 90 => 'A',
    >= 80 and < 90 => 'B',
    >= 70 and < 80 => 'C',
    >= 60 and < 70 => 'D',
    < 60 => 'F'
};

// Shipping cost calculator
decimal CalculateShipping(double weight) => weight switch
{
    < 1 => 5.00m,
    >= 1 and < 5 => 10.00m,
    >= 5 and < 20 => 20.00m,
    >= 20 => 50.00m
};
```

### Logical Patterns

- Logical patterns combine patterns using `and`, `or`, `not` keywords
- `and` — both patterns must match
- `or` — at least one pattern must match
- `not` — the pattern must NOT match
- Available from C# 9+

```csharp
// 'and' pattern — both conditions must match
if (age is >= 18 and <= 65)
{
    Console.WriteLine("Working age");
}

// 'or' pattern — either condition matches
if (day is "Saturday" or "Sunday")
{
    Console.WriteLine("Weekend!");
}

// 'not' pattern — negation
if (input is not null and not "")
{
    Console.WriteLine("Valid input");
}

// Combining logical patterns
string GetDiscount(decimal total, bool isMember) => (total, isMember) switch
{
    (> 100, true) => "20% discount",
    (> 100, false) => "10% discount",
    (<= 100, true) => "5% discount",
    _ => "No discount"
};

// Complex logical patterns
bool IsWeekday(DayOfWeek day) => day is >= DayOfWeek.Monday and <= DayOfWeek.Friday;
bool IsWeekend(DayOfWeek day) => day is DayOfWeek.Saturday or DayOfWeek.Sunday;
bool IsBusinessHours(int hour) => hour is >= 9 and <= 17;
```

### Type Patterns

- Type patterns check if a value is of a specific type
- Can optionally extract the value into a variable
- Eliminates explicit casts

```csharp
// Basic type pattern
object obj = 42;
if (obj is int number)
{
    Console.WriteLine($"Integer: {number}");
}

// Multiple type patterns
string DescribeObject(object obj) => obj switch
{
    int i when i > 0 => $"Positive integer: {i}",
    int i when i < 0 => $"Negative integer: {i}",
    int => "Zero",
    string s => $"String of length {s.Length}",
    double d => $"Double: {d:F2}",
    bool b => $"Boolean: {b}",
    null => "Null",
    _ => $"Unknown type: {obj.GetType().Name}"
};

// Type pattern with 'not'
if (obj is not string)
{
    Console.WriteLine("Not a string");
}

// Type pattern in collections
void ProcessItems(object[] items)
{
    foreach (var item in items)
    {
        switch (item)
        {
            case int n:
                Console.WriteLine($"Number: {n}");
                break;
            case string s:
                Console.WriteLine($"Text: {s}");
                break;
        }
    }
}
```

### var Patterns

- `var` pattern always matches and captures the value in a variable
- Useful for default cases where you need to access the value
- Combined with `when` clauses for conditional logic

```csharp
// Basic var pattern — always matches
if (obj is var value)
{
    Console.WriteLine($"Got value: {value}");
}

// var pattern with when clause
string DescribeNumber(int n) => n switch
{
    var x when x < 0 => $"Negative: {x}",
    var x when x == 0 => "Zero",
    var x when x % 2 == 0 => $"Even positive: {x}",
    var x => $"Odd positive: {x}"
};

// var pattern in switch expression for default case
string Classify(object obj) => obj switch
{
    int i => $"Integer: {i}",
    string s => $"String: {s}",
    var other => $"Other: {other}"  // Catch-all that captures the value
};
```

### Discards

- Discards (`_`) match any value and throw it away
- Used when you don't need a particular value from a pattern
- Essential for making switch expressions exhaustive

```csharp
// Discard in deconstruction
var (name, _, age) = ("Alice", "Smith", 30);
Console.WriteLine($"{name}, age {age}");  // "Alice, age 30" — "Smith" discarded

// Discard in pattern matching
if (obj is string { Length: var _ })
{
    Console.WriteLine("It's a string of any length");
}

// Discard in switch expression (exhaustive pattern)
int Classify(int x) => x switch
{
    0 => 1,
    1 => 2,
    _ => 0  // Matches anything else
};

// Multiple discards
var (_, _, third) = (1, 2, 3);
Console.WriteLine(third);  // 3
```

---

## C# 8+ Features

### Switch Expressions

- More concise than traditional switch statements
- Must be exhaustive — compiler enforces all cases are covered
- Return a value — ideal for assignments
- Support pattern matching in each arm

```csharp
// Traditional switch statement (verbose)
string GetSeasonOld(int month)
{
    switch (month)
    {
        case 3:
        case 4:
        case 5:
            return "Spring";
        case 6:
        case 7:
        case 8:
            return "Summer";
        case 9:
        case 10:
        case 11:
            return "Autumn";
        case 12:
        case 1:
        case 2:
            return "Winter";
        default:
            throw new ArgumentOutOfRangeException(nameof(month));
    }
}

// Switch expression (concise)
string GetSeason(int month) => month switch
{
    >= 3 and <= 5 => "Spring",
    >= 6 and <= 8 => "Summer",
    >= 9 and <= 11 => "Autumn",
    12 or 1 or 2 => "Winter",
    _ => throw new ArgumentOutOfRangeException(nameof(month))
};
```

### Nullable Reference Types

- Nullable reference types provide compile-time warnings for potential null references
- Enable with `<Nullable>enable</Nullable>` in .csproj or `#nullable enable` in file
- Reference types are treated as non-nullable by default
- Use `?` to explicitly mark nullable reference types

```csharp
#nullable enable

// Non-nullable — compiler warns if not initialized
public class Person
{
    public string Name { get; set; } = "";  // Must initialize or use null-forgiving operator
    public int Age { get; set; }
}

// Nullable reference type
public class PersonWithNullable
{
    public string? Name { get; set; }  // Can be null
    public int Age { get; set; }
}

// Null-forgiving operator (!)
string name = null!;  // Suppresses warning — use with caution

// Null-conditional operator (?.)
int? length = name?.Length;  // null if name is null

// Null-coalescing operator (??)
string displayName = name ?? "Unknown";
```

### Ranges and Indexes

- Ranges and indexes provide concise syntax for working with arrays and collections
- `^1` means "last element", `^2` means "second to last", etc.
- `[1..3]` means elements at index 1 and 2 (exclusive upper bound)
- Available from C# 8+ with `Index` and `Range` types

```csharp
int[] numbers = { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 };

// Index from end
int last = numbers[^1];       // 9
int secondLast = numbers[^2]; // 8

// Range
int[] slice1 = numbers[1..4];    // { 1, 2, 3 }
int[] slice2 = numbers[..4];     // { 0, 1, 2, 3 } — from start
int[] slice3 = numbers[6..];     // { 6, 7, 8, 9 } — to end
int[] slice4 = numbers[^3..^1];  // { 7, 8 }

// Works with strings too
string text = "Hello, World!";
string substring = text[7..12];  // "World"
string lastChars = text[^6..];   // "World!"

// Works with Span<T> for zero-allocation slicing
Span<int> span = numbers.AsSpan()[1..4];
```

### Using Declarations

- `using` declarations simplify resource management by auto-disposing at scope end
- No need for explicit `using` blocks — resource is disposed when the variable goes out of scope
- Available from C# 8+

```csharp
// Traditional using block (C# 7 and earlier)
using (var reader = new StreamReader("file.txt"))
{
    var content = reader.ReadToEnd();
    Console.WriteLine(content);
}  // reader.Dispose() called here

// Using declaration (C# 8+)
var reader = new StreamReader("file.txt");
// No explicit using block needed

// Or combined with file reading
using var stream = new FileStream("data.bin", FileMode.Open);
using var binaryReader = new BinaryReader(stream);
var data = binaryReader.ReadBytes(1024);
// Both stream and binaryReader disposed when scope ends
```

### Static Local Functions

- Static local functions cannot capture variables from the enclosing method
- This makes them more predictable and potentially more performant
- Use when the local function doesn't need access to enclosing scope variables

```csharp
// Non-static local function — can capture variables
int Factorial(int n)
{
    int result = 1;

    int Calculate(int x)  // Captures 'result' implicitly
    {
        result *= x;
        return result;
    }

    for (int i = 1; i <= n; i++)
        Calculate(i);

    return result;
}

// Static local function — cannot capture variables
int FactorialStatic(int n)
{
    return Calculate(n);  // Must pass all needed data as parameters

    static int Calculate(int x)
    {
        if (x <= 1) return 1;
        return x * Calculate(x - 1);  // Pure function — no captured state
    }
}
```

---

## Real-World Usage

### Records for API Request/Response DTOs

```csharp
// Request DTOs — immutable, clear contract
public record CreateOrderRequest(
    string ProductId,
    int Quantity,
    string ShippingAddress,
    string? PromoCode
);

public record UpdateOrderRequest(
    string OrderId,
    int? Quantity,
    string? ShippingAddress
);

// Response DTOs — consistent, serializable
public record OrderResponse(
    string OrderId,
    string ProductId,
    int Quantity,
    decimal TotalPrice,
    string Status,
    DateTime CreatedAt
);

public record ErrorResponse(
    string Code,
    string Message,
    string? Details
);

// Usage in API controller
public record OrderController
{
    public OrderResponse CreateOrder(CreateOrderRequest request)
    {
        // Process order
        return new OrderResponse(
            OrderId: Guid.NewGuid().ToString(),
            ProductId: request.ProductId,
            Quantity: request.Quantity,
            TotalPrice: request.Quantity * 9.99m,
            Status: "Created",
            CreatedAt: DateTime.UtcNow
        );
    }
}
```

### Tuples for Method Returns with Multiple Values

```csharp
// Good use of tuples — internal calculation results
public (int Min, int Max, double Average, int Count) AnalyzeData(int[] data)
{
    return (data.Min(), data.Max(), data.Average(), data.Length);
}

// Good use of tuples — parsing with success indicator
public (bool Success, string? Value, string? Error) ParseInput(string input)
{
    if (string.IsNullOrWhiteSpace(input))
        return (false, null, "Input cannot be empty");

    if (int.TryParse(input, out int result))
        return (true, result.ToString(), null);

    return (false, null, $"Invalid input: {input}");
}

// Usage
var (success, value, error) = ParseInput("42");
if (success)
    Console.WriteLine($"Parsed: {value}");
else
    Console.WriteLine($"Error: {error}");

// Tuple in loop
var items = new[] { 1, 2, 3, 4, 5 };
foreach (var (index, item) in items.Select((item, index) => (index, item)))
{
    Console.WriteLine($"[{index}] = {item}");
}
```

### Pattern Matching in Authorization Logic

```csharp
public record User(string Name, string Role, bool IsActive, int Age);
public record Resource(string OwnerId, string Type, bool IsPublic);

public record AccessRequest(User User, Resource Resource);

public record AccessResult(bool Allowed, string Reason);

public AccessResult EvaluateAccess(AccessRequest request) => request switch
{
    // Inactive users are always denied
    { User: { IsActive: false } } => new(false, "User account is inactive"),

    // Admins can access everything
    { User: { Role: "Admin" } } => new(true, "Admin access granted"),

    // Public resources are accessible to everyone
    { Resource: { IsPublic: true } } => new(true, "Public resource access"),

    // Resource owners can access their own resources
    { User: var u, Resource: var r } when u.Name == r.OwnerId
        => new(true, "Owner access"),

    // Age restriction for certain resources
    { User: { Age: < 18 }, Resource: { Type: "Adult" } }
        => new(false, "Age restriction: must be 18+"),

    // Default deny
    _ => new(false, "Access denied")
};
```

### Pattern Matching for State Machines

```csharp
public record ConnectionState(string Name);

public record Disconnected() : ConnectionState("Disconnected");
public record Connecting(string Address) : ConnectionState("Connecting");
public record Connected(string Address, DateTime ConnectedAt) : ConnectionState("Connected");
public record Disconnecting(string Address, string Reason) : ConnectionState("Disconnecting");

public ConnectionState Transition(ConnectionState current, string @event) => (current, @event) switch
{
    (Disconnected, "connect") => new Connecting("server.example.com"),
    (Connecting var c, "connected") => new Connected(c.Address, DateTime.UtcNow),
    (Connected var c, "disconnect") => new Disconnecting(c.Address, "User request"),
    (Disconnecting, "disconnected") => new Disconnected(),
    (Connecting, "timeout") => new Disconnected(),
    (Disconnecting var d, "error") => new Disconnected(),
    _ => current  // Invalid transition — stay in current state
};

// Usage
var state = new ConnectionState("Disconnected");
state = Transition(state, "connect");      // Connecting
state = Transition(state, "connected");    // Connected
state = Transition(state, "disconnect");   // Disconnecting
state = Transition(state, "disconnected"); // Disconnected
```

---

## Interview Questions

### Question 1: What is the difference between a record and a class?

**Answer:** A record is a reference type with value-based equality semantics, while a class uses reference-based equality by default. Records automatically generate `Equals`, `GetHashCode`, and `ToString` based on property values. Records support `with` expressions for non-destructive mutation, while classes do not. Records are ideal for DTOs, API contracts, and immutable data, while classes are better for behavior-rich objects with mutable state.

```csharp
var a = new PersonRecord("Alice", 30);
var b = new PersonRecord("Alice", 30);
Console.WriteLine(a == b);  // True — value equality

var c = new PersonClass("Alice", 30);
var d = new PersonClass("Alice", 30);
Console.WriteLine(c == d);  // False — reference equality
```

### Question 2: How do init-only setters work and why are they useful?

**Answer:** Init-only setters (`{ get; init; }`) allow setting a property only during object initialization (constructor or object initializer). After construction, the property becomes read-only. They enforce immutability at compile time without the verbosity of readonly fields. They are essential for records, which use them by default. Unlike `{ get; set; }`, init-only setters prevent modification after construction, making objects thread-safe and easier to reason about.

```csharp
public record Person(string Name, int Age);

var p = new Person("Alice", 30);
// p.Name = "Bob";  // Compiler error — init-only

// Useful for configuration that shouldn't change
public record Config(string Url, int Timeout)
{
    public string? ApiKey { get; init; }  // Optional, set only during creation
}
```

### Question 3: What is the difference between `ValueTuple` and `Tuple`?

**Answer:** `ValueTuple` is a value type (stack-allocated, no GC pressure), while `Tuple` is a reference type (heap-allocated). ValueTuple supports named elements (`(string Name, int Age)`), while Tuple uses `Item1`, `Item2`. ValueTuple is more performant and ergonomic. Tuple is legacy from before C# 7 and should generally be avoided in new code.

```csharp
// ValueTuple — preferred
(string Name, int Age) person = ("Alice", 30);

// Tuple — legacy, avoid
Tuple<string, int> oldPerson = Tuple.Create("Alice", 30);
Console.WriteLine(oldPerson.Item1);  // Less readable
```

### Question 4: How do you deconstruct a tuple?

**Answer:** Tuples can be deconstructed into separate variables using tuple assignment syntax. You can deconstruct into new variables or existing variables. Discards (`_`) can be used to ignore elements you don't need.

```csharp
var person = ("Alice", 30, "Engineer");

// Deconstruct into new variables
var (name, age, role) = person;

// Deconstruct into existing variables
string n;
int a;
(n, a, _) = person;

// Use discard for unwanted elements
var (_, _, jobTitle) = person;
```

### Question 5: What types of pattern matching are available in C#?

**Answer:** C# supports several pattern types: **is pattern** (`obj is string s`), **type pattern** (`case int x`), **constant pattern** (`case 42`), **property pattern** (`{ Name: "Alice" }`), **positional pattern** (`(x, y)`), **relational pattern** (`> 10`, `< 5`), **logical pattern** (`and`, `or`, `not`), **var pattern** (`var x`), and **discard** (`_`). These can be combined for expressive, readable logic.

```csharp
string Describe(object obj) => obj switch
{
    int x and > 0 => $"Positive: {x}",
    string { Length: > 5 } s => $"Long string: {s}",
    (var a, var b) => $"Tuple: {a}, {b}",
    null => "Null",
    _ => "Unknown"
};
```

### Question 6: When should you use records vs classes?

**Answer:** Use records for data-centric types where value equality matters: DTOs, API contracts, immutable configuration, value objects, and return types. Use classes for behavior-rich objects with mutable state, complex lifecycle management, or when you need reference semantics. Records are ideal when you want to compare by content, while classes are better when identity matters.

```csharp
// Record — data-focused
public record Product(string Id, string Name, decimal Price);

// Class — behavior-focused
public class ShoppingCart
{
    private readonly List<Product> _items = new();
    public void Add(Product item) => _items.Add(item);
    public decimal Total => _items.Sum(p => p.Price);
}
```

### Question 7: What are the limitations of init-only setters?

**Answer:** Init-only setters can only be assigned during object initialization (constructor or object initializer). They cannot be set in methods, after construction, or by derived classes. The backing field is `readonly`. Some serialization frameworks (like older Newtonsoft.Json versions) may need special configuration. They don't work well with frameworks that modify objects after creation (like some ORMs).

```csharp
public record Person(string Name, int Age);

var p = new Person("Alice", 30);
// p.Name = "Bob";  // Error — init-only after construction
// p with { Name = "Bob" } works — creates a new record
```

### Question 8: How do with expressions work?

**Answer:** `with` expressions create a shallow copy of a record with specified properties modified. The original record remains unchanged. This enables non-destructive mutation of immutable data. Works with both positional and non-positional records.

```csharp
var original = new Person("Alice", 30);
var modified = original with { Age = 31 };

Console.WriteLine(original.Age);  // 30 — unchanged
Console.WriteLine(modified.Age);  // 31 — new copy
```

### Question 9: What is value-based equality and how does it work in records?

**Answer:** Value-based equality means two objects are considered equal if their property values are equal, not if they are the same reference. Records automatically implement `Equals` and `GetHashCode` using all properties. This makes records behave like value types for equality comparisons, even though they are reference types.

```csharp
var a = new Point(1, 2);
var b = new Point(1, 2);
Console.WriteLine(a.Equals(b));       // True — same property values
Console.WriteLine(a.GetHashCode() == b.GetHashCode());  // True
```

### Question 10: Can record structs have mutable properties?

**Answer:** Yes, regular `record struct` can have mutable properties with `{ get; set; }`. However, `readonly record struct` enforces immutability like records. The choice depends on whether you need mutability for the value type.

```csharp
public record struct MutablePoint(double X, double Y);          // Mutable
public readonly record struct ImmutablePoint(double X, double Y);  // Immutable

var mutable = new MutablePoint(1, 2);
mutable.X = 3;  // Works — mutable record struct

var immutable = new ImmutablePoint(1, 2);
// immutable.X = 3;  // Error — readonly record struct
```

### Question 11: How do property patterns work in pattern matching?

**Answer:** Property patterns match on properties of an object using `{ PropertyName: pattern }` syntax. They can be nested for complex objects. Available from C# 8+, they enable concise matching without explicit null checks or property access.

```csharp
if (person is { Name: "Alice", Age: > 25 })
{
    Console.WriteLine("Found Alice over 25");
}

// Nested property pattern
if (order is { Customer: { IsVip: true } })
{
    Console.WriteLine("VIP customer order");
}
```

### Question 12: What is the difference between `and`, `or`, and `not` patterns?

**Answer:** `and` requires both patterns to match, `or` requires at least one to match, and `not` negates a pattern. These logical patterns enable complex conditions in pattern matching expressions.

```csharp
// and — both conditions
if (age is >= 18 and <= 65) { }  // Working age

// or — either condition
if (day is "Saturday" or "Sunday") { }  // Weekend

// not — negation
if (input is not null and not "") { }  // Valid input
```

### Question 13: When would you use tuples instead of records?

**Answer:** Use tuples for temporary, internal groupings of values where you don't need documentation, validation, or methods. Use records for public APIs, persistent data, or when you need named types with behavior. Tuples are lightweight but lack the expressiveness of named types.

```csharp
// Tuple — quick internal calculation
public (int Min, int Max) GetRange(int[] data) => (data.Min(), data.Max());

// Record — public API contract
public record ProductResponse(string Id, string Name, decimal Price);
```

### Question 14: How do positional patterns work?

**Answer:** Positional patterns use the `Deconstruct` method to match on positional elements. They work with records, tuples, and any type with a `Deconstruct` method. Use `(pattern1, pattern2)` syntax.

```csharp
public record Point(double X, double Y);

string Describe(Point p) => p switch
{
    (0, 0) => "Origin",
    (var x, 0) => $"On X-axis at {x}",
    (var x, var y) => $"At ({x}, {y})"
};
```

### Question 15: What are the benefits of using pattern matching over traditional if-else chains?

**Answer:** Pattern matching provides more concise, readable, and expressive code. It eliminates redundant type checks and casts, supports exhaustive checking (compiler ensures all cases are covered), and works seamlessly with records and tuples. Switch expressions return values, making them ideal for assignments and method returns.

```csharp
// Traditional — verbose, repetitive
if (obj is int)
{
    int i = (int)obj;
    if (i > 0) return $"Positive: {i}";
    else if (i < 0) return $"Negative: {i}";
    else return "Zero";
}

// Pattern matching — concise, expressive
return obj switch
{
    int i and > 0 => $"Positive: {i}",
    int i and < 0 => $"Negative: {i}",
    int => "Zero",
    _ => "Unknown"
};
```

### Question 16: Can you inherit from records and what happens with equality?

**Answer:** Records support inheritance, but equality is type-specific. A derived record is only equal to another record of the **same derived type** with the same property values. A `Circle` will never equal a `Rectangle` even if they have the same properties. You can use `sealed record` to prevent inheritance.

```csharp
public record Shape(string Color);
public record Circle(string Color, double Radius) : Shape(Color);

var c1 = new Circle("Red", 5.0);
var c2 = new Circle("Red", 5.0);
Console.WriteLine(c1.Equals(c2));  // True — same type, same values
```

### Question 17: What is the var pattern and when is it useful?

**Answer:** The `var` pattern always matches and captures the value in a variable. It's useful in default switch cases where you need to access the value, or with `when` clauses for conditional logic that requires the captured value.

```csharp
string Classify(int x) => x switch
{
    > 0 and < 100 => "Small positive",
    var n when n % 2 == 0 => $"Even: {n}",
    var n => $"Odd: {n}"
};
```

### Question 18: How do ranges and indexes work with collections?

**Answer:** Ranges (`[1..3]`) and indexes (`[^1]`) provide concise syntax for slicing collections. `^1` means last element, `[1..3]` means elements at index 1 and 2 (exclusive upper bound). Works with arrays, strings, and `Span<T>`.

```csharp
int[] arr = { 0, 1, 2, 3, 4, 5 };
Console.WriteLine(arr[^1]);     // 5 (last element)
Console.WriteLine(arr[1..4]);   // { 1, 2, 3 }
Console.WriteLine(arr[..3]);    // { 0, 1, 2 }
```

---

*Records, tuples, and pattern matching are essential modern C# features that enable cleaner, safer, and more expressive code. Master these for interviews and production development.*
