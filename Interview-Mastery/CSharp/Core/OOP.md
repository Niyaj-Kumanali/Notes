# C# OOP — Interview Mastery

---

## Class and Object

### What is a Class

- A class is a user-defined data type that acts as a blueprint or template for creating objects
- It bundles data (fields, properties) and behavior (methods, events) into a single unit
- A class defines what an object will look like and what it can do, but it is not the object itself
- Memory is not allocated for a class until an object (instance) is created from it
- A class can contain constants, fields, constructors, methods, properties, indexers, events, operators, and nested types
- Classes are reference types — a variable of class type holds a reference (pointer) to the object on the heap, not the object itself

### What is an Object

- An object is a runtime instance of a class — it is the actual entity created from the blueprint
- When you create an object, the CLR allocates memory on the managed heap and returns a reference to that memory
- Each object has its own copy of instance fields, but shares the class's method definitions
- Objects are garbage collected — when no references point to an object, the GC eventually reclaims its memory
- Two variables can reference the same object — modifying the object through one variable is visible through the other

### How Objects Are Created

- The `new` keyword allocates memory on the heap, calls the constructor, and returns the reference
- The constructor initializes the object's state — it is not a method and does not return a value
- `Person p = new Person("Alice");` — `new` allocates, `Person("Alice")` invokes the constructor, `p` holds the reference
- Without `new`, you cannot instantiate a class (except via reflection or `FormatterServices.GetUninitializedObject`)
- `new` can also be used to hide inherited members (discussed under Inheritance)

### Why Classes Are Blueprints

- A blueprint defines the structure and behavior, but you build many houses from one blueprint
- Similarly, one `Person` class can create thousands of `Person` objects, each with its own name, age, etc.
- This separation of definition from instantiation is the foundation of object-oriented design
- It allows you to model real-world concepts as code entities with both state and behavior

### Types of Constructors

#### Default Constructor

- A constructor with no parameters, provided automatically by the compiler only if no other constructor is defined
- If you define any constructor, the compiler does NOT generate a default constructor
- The default constructor calls the parameterless base class constructor implicitly
- It typically initializes fields to their default values (0, null, false)

```csharp
public class Employee
{
    public string Name; // null by default

    // Compiler generates this if no other constructor exists:
    // public Employee() { }
}
```

#### Parameterized Constructor

- A constructor that accepts parameters to initialize object state at creation time
- Allows objects to be created with meaningful initial values
- Overloading constructors gives flexibility in how objects are created

```csharp
public class Employee
{
    public string Name;
    public int Age;

    public Employee(string name, int age)
    {
        Name = name;
        Age = age;
    }
}
```

#### Copy Constructor

- A constructor that creates a new object as a copy of an existing object of the same class
- Takes a single parameter — an instance of the same class
- Performs a shallow copy by default (copies reference types by reference, not deep copy)
- Used when you want to create independent copies of objects

```csharp
public class Employee
{
    public string Name;
    public Address Address;

    public Employee(Employee other)
    {
        Name = other.Name;
        Address = other.Address; // shallow copy — both share same Address
    }
}
```

#### Static Constructor

- A constructor that runs once, before any instance is created or any static member is accessed
- Cannot have any access modifier (technically it is `private static`) — you cannot call it directly
- Cannot accept parameters — it initializes static fields only
- The CLR guarantees it runs only once, even in multi-threaded environments (thread-safe by the runtime)
- Execution order: static constructor runs before the first instance is created or any static member is accessed

```csharp
public class Config
{
    public static readonly string ConnectionString;

    static Config()
    {
        ConnectionString = ConfigurationManager.ConnectionStrings["Main"].ConnectionString;
    }
}
```

### Constructor Chaining

#### Using `this()` — Chaining to Another Constructor in the Same Class

- Allows one constructor to call another constructor of the same class to avoid code duplication
- The called constructor must be the first statement in the calling constructor
- Useful when you have multiple ways to create an object but want a single initialization point

```csharp
public class Employee
{
    public string Name;
    public int Age;
    public string Department;

    public Employee(string name) : this(name, 0, "Unknown")
    {
        // Name, Age, Department all set by the other constructor
    }

    public Employee(string name, int age, string dept)
    {
        Name = name;
        Age = age;
        Department = dept;
    }
}
```

#### Using `base()` — Chaining to a Base Class Constructor

- Allows a derived class constructor to call a specific base class constructor
- If not specified, the compiler implicitly calls the parameterless base constructor `: base()`
- The `base()` call must be the first statement in the derived constructor
- Essential when the base class has no parameterless constructor

```csharp
public class Person
{
    public string Name;

    public Person(string name)
    {
        Name = name;
    }
}

public class Employee : Person
{
    public int EmployeeId;

    public Employee(string name, int id) : base(name)
    {
        EmployeeId = id;
    }
}
```

### Constructor Invocation Order in Inheritance

- Base class static constructor runs first (once, before anything else)
- Derived class static constructor runs next (once, before first instance)
- Base class instance constructor runs BEFORE derived class instance constructor
- Derived class constructor body runs AFTER base class constructor body
- The full order: `Derived static → Base static → Base instance → Derived instance`
- This means the base class initializes its state before the derived class runs — the derived constructor sees a fully initialized base

```csharp
public class A
{
    public A() { Console.WriteLine("A instance ctor"); }
    static A() { Console.WriteLine("A static ctor"); }
}

public class B : A
{
    public B() { Console.WriteLine("B instance ctor"); }
    static B() { Console.WriteLine("B static ctor"); }
}

// new B() outputs:
// A static ctor
// B static ctor
// A instance ctor
// B instance ctor
```

---

## Encapsulation

### What is Encapsulation

- Encapsulation is the principle of bundling data (fields) and the methods that operate on that data into a single unit (class), while restricting direct access to some of the object's components
- It is one of the four pillars of OOP alongside inheritance, polymorphism, and abstraction
- The goal is to protect the internal state of an object from invalid or unintended modifications
- External code interacts with the object through a controlled public interface (methods and properties)
- This creates a "black box" — consumers use the object without knowing or depending on its internal implementation

### How Access Modifiers Work

- Access modifiers control the visibility and accessibility of types and members
- C# has six access modifiers:

| Modifier | Accessible From |
|---|---|
| `public` | Anywhere — no restrictions |
| `private` | Only within the same class or struct |
| `protected` | The declaring class and its derived classes |
| `internal` | Any code within the same assembly (project) |
| `protected internal` | Derived classes OR any code in the same assembly (union) |
| `private protected` | Derived classes AND within the same assembly (intersection) |

- `protected internal` is more permissive — it is the OR of `protected` and `internal`
- `private protected` is more restrictive — it is the AND of `protected` and `internal`
- Default access for class members is `private`; default for top-level types is `internal`

### Why We Use Properties with Private Setters

- A public getter allows read access while a private setter restricts write access to within the class
- This enforces encapsulation — external code can read the value but cannot arbitrarily change it
- The class retains full control over when and how the value changes (validation, side effects, logging)

```csharp
public class BankAccount
{
    public decimal Balance { get; private set; }

    public BankAccount(decimal initial)
    {
        Balance = initial;
    }

    public void Deposit(decimal amount)
    {
        if (amount <= 0) throw new ArgumentException("Amount must be positive");
        Balance += amount; // only the class can modify Balance
    }
}
```

- Without the private setter, any external code could set `Balance` to any value, bypassing validation
- In C# 6+, you can also use `{ get; }` (init-only setter) for values set only during initialization

### Why Encapsulation Matters

- **Maintainability**: Internal implementation can change without breaking external code — as long as the public interface stays the same
- **Validation**: The class can enforce invariants — every method that modifies state can validate inputs
- **Testing**: Encapsulated classes are easier to test because you interact through a defined interface
- **Reduced coupling**: Consumers depend on behavior (methods), not structure (fields), leading to looser coupling
- **Thread safety**: Controlled access points make it easier to add synchronization later
- **Debugging**: When state changes only through methods, it is easier to trace when and why a value changed

---

## Inheritance

### What is Inheritance

- Inheritance is a mechanism where a derived (child) class acquires the fields, properties, methods, and events of a base (parent) class
- It establishes an "is-a" relationship — `Employee` is a `Person`, `Car` is a `Vehicle`
- The derived class can add new members and modify (override) existing behavior of the base class
- Inheritance promotes code reuse — shared logic lives in the base class and is inherited by all derived classes
- Inheritance forms a hierarchy — a class can inherit from one class, which inherits from another, forming a chain

### How Single Inheritance Works in C#

- C# supports single inheritance only — a class can inherit from exactly one base class
- A derived class extends one base class but can implement multiple interfaces
- This avoids the "diamond problem" that occurs with multiple class inheritance
- The derived class has access to all non-private members of the base class (public, protected, internal, protected internal)

```csharp
public class Animal
{
    public virtual void Speak() { Console.WriteLine("..."); }
}

public class Dog : Animal
{
    public override void Speak() { Console.WriteLine("Woof!"); }
}
```

### Why C# Does Not Support Multiple Class Inheritance

- Multiple class inheritance creates ambiguity — if two base classes define the same method, which one does the derived class inherit?
- This is known as the "diamond problem" — two base classes share a common ancestor, and the derived class has two copies of the ancestor's members
- C# avoids this by allowing single class inheritance + multiple interface implementation
- Interfaces provide the contract without implementation, avoiding the ambiguity
- Microsoft's design decision prioritizes simplicity and clarity over the flexibility of multiple inheritance

### The `base` Keyword

- `base` refers to the base class instance — it provides access to base class members from a derived class
- `base.Method()` calls the base class version of a method (useful when you override but still need the base behavior)
- `base(propertyName)` passes parameters to a base class constructor via `: base(args)`
- `base` cannot be used in static members — it is an instance reference
- You cannot use `base` to access members that are `private` in the base class

```csharp
public class Derived : Base
{
    public override void DoWork()
    {
        base.DoWork(); // call base implementation first
        // additional derived behavior
    }
}
```

### Constructor Chaining in Inheritance

- When creating a derived object, the base class constructor always runs first
- The derived constructor can specify which base constructor to call using `: base(args)`
- If no `base()` is specified, the compiler implicitly calls `: base()` (parameterless)
- If the base class has no parameterless constructor, the derived class MUST explicitly specify a `base()` call

```csharp
public class Base
{
    public Base(string msg) { Console.WriteLine(msg); }
}

public class Derived : Base
{
    // MUST call base explicitly — no parameterless base ctor exists
    public Derived() : base("Hello from base") { }
}
```

### Method Hiding with `new` vs `override`

- `new` keyword hides the base class method — the derived method completely replaces it from the derived type's perspective
- `override` keyword provides a new implementation for a `virtual` or `abstract` base method — it participates in polymorphism
- With `new`, the method called depends on the declared type of the variable, not the actual object type
- With `override`, the method called depends on the actual runtime type of the object
- The compiler generates a warning when you use `new` without explicitly specifying it — this is a potential design smell

```csharp
public class Base
{
    public void Foo() { Console.WriteLine("Base.Foo"); }
    public virtual void Bar() { Console.WriteLine("Base.Bar"); }
}

public class Derived : Base
{
    public new void Foo() { Console.WriteLine("Derived.Foo"); }
    public override void Bar() { Console.WriteLine("Derived.Bar"); }
}

Base obj = new Derived();
obj.Foo(); // "Base.Foo" — compile-time type determines call (new hides)
obj.Bar(); // "Derived.Bar" — runtime type determines call (override polymorphic)
```

---

## Polymorphism

### Compile-Time Polymorphism (Method Overloading)

- Method overloading allows multiple methods with the same name but different parameter lists in the same class
- The compiler resolves which method to call at compile time based on the argument types and count
- Overload resolution follows specific rules — exact match is preferred, then implicit conversions, then params arrays
- Return type alone is not sufficient to distinguish overloads — parameter types and count must differ
- This is sometimes called "static polymorphism" because the binding happens at compile time

```csharp
public class Calculator
{
    public int Add(int a, int b) => a + b;
    public double Add(double a, double b) => a + b;
    public int Add(int a, int b, int c) => a + b + c;
}
```

### Runtime Polymorphism (Method Overriding)

- Method overriding allows a derived class to provide a specific implementation of a method defined in the base class
- The base method must be marked `virtual`, `abstract`, or already `override`
- The derived method uses the `override` keyword to indicate it replaces the base implementation
- The CLR uses a virtual method table (vtable) to resolve which method to call at runtime
- The actual type of the object determines which implementation runs, not the variable's declared type

```csharp
public class Shape
{
    public virtual double Area() => 0;
}

public class Circle : Shape
{
    public double Radius { get; set; }
    public override double Area() => Math.PI * Radius * Radius;
}
```

### Why the `virtual` Keyword is Needed

- By default, non-virtual methods are statically bound — the compiler decides which method to call based on the declared type
- The `virtual` keyword opts a method into dynamic dispatch — the runtime looks up the actual type's implementation
- Without `virtual`, marking a derived method `override` causes a compile error — you cannot override what is not virtual
- This is a deliberate design choice — it prevents accidental polymorphic behavior and makes performance predictable
- Virtual method calls are slightly slower than non-virtual calls due to the vtable lookup, but the difference is negligible in most applications

### Why `sealed` Prevents Overriding

- The `sealed` keyword on a method prevents further overriding in derived classes
- When applied to a class, it prevents the class from being inherited at all
- Sealing methods tells the compiler it can devirtualize the call — the exact implementation is known
- This is useful when you override a method in a derived class and want to ensure no further derived class changes it
- Sealing a class is common for final implementations — `string` in .NET is a sealed class

```csharp
public class Base
{
    public virtual void Foo() { }
}

public class Derived : Base
{
    public sealed override void Foo() { } // no further overriding allowed
}

public class MoreDerived : Derived
{
    // public override void Foo() { } // COMPILE ERROR — Foo is sealed
}
```

### How the CLR Resolves Virtual Calls (vtable)

- When a class is loaded, the CLR builds a virtual method table (vtable) for that type
- Each entry in the vtable points to the most-derived implementation of a virtual method
- When `override` is used, the derived type's vtable entry for that method is replaced with the new implementation
- At runtime, a virtual call indexes into the vtable of the actual object's type and invokes the method at that slot
- This is why virtual dispatch works even when the variable is typed as the base class — the vtable of the actual object is used
- Non-virtual calls skip the vtable entirely — the method address is resolved at JIT compile time

### `new` Keyword for Hiding vs `override`

- `new` is compile-time hiding — it creates a separate method that shadows the base method
- `override` is runtime polymorphism — it replaces the base method in the virtual dispatch chain
- With `new`, casting to the base type exposes the base method (hiding is lost)
- With `override`, casting to the base type still calls the derived method (polymorphism is preserved)
- The compiler warns when `new` is used implicitly — always be explicit about your intent

---

## Abstraction

### Abstract Classes vs Interfaces

- An abstract class can provide partial implementation — it can have fields, constructors, and concrete methods
- An interface (pre-C#8) was purely a contract — it defined only method signatures, properties, and events with no implementation
- An abstract class uses the `abstract` keyword and cannot be instantiated directly
- A class can inherit from only one abstract class but can implement many interfaces
- Abstract classes can have access modifiers on members; interface members are `public` by default (pre-C#8)
- Abstract classes can have constructors (to initialize state for derived classes); interfaces cannot have constructors

### Abstract Methods vs Virtual Methods

- An abstract method has no body — it MUST be overridden by a non-abstract derived class
- A virtual method has a default implementation — it CAN be overridden but does not have to be
- An abstract method implicitly behaves as if it were virtual — you use `override` in the derived class
- Abstract methods can only exist in abstract classes
- A class with any abstract member must itself be declared `abstract`

```csharp
public abstract class Vehicle
{
    public abstract void Start();   // no body — derived MUST implement
    public virtual void Honk() { Console.WriteLine("Beep!"); } // has body — derived MAY override
}

public class Car : Vehicle
{
    public override void Start() { Console.WriteLine("Engine started"); }
    // Honk() is inherited as-is from Vehicle
}
```

### Why Abstract Classes Can Have Fields but Interfaces Cannot (Pre-C#8)

- Fields represent storage — they define the memory layout of an object
- A class has a single memory layout determined by its class hierarchy — multiple inheritance of state would cause conflicts
- Interfaces define behavior contracts, not storage — they say "you can do this," not "you store this here"
- If interfaces had fields, a class implementing two interfaces with the same field name would have an ambiguous memory layout
- Abstract classes can have fields because they participate in the single-inheritance memory model — the derived class has exactly one copy of each base field
- Since C# 8, interfaces can have static abstract members and default implementations, but still cannot have instance fields

### When to Use Abstract Class vs Interface

- Use an abstract class when related classes share common state (fields) and behavior (methods)
- Use an interface when unrelated classes need to adhere to a common contract
- Use an abstract class when you need to provide a partial implementation and leave some methods for derived classes
- Use an interface when you need to define a capability that can be applied to multiple inheritance hierarchies
- Use an abstract class when you need constructors, access modifiers, or non-public members
- Use an interface when you are designing for composition over inheritance
- A common pattern: define an interface for the contract, provide an abstract base class with shared logic, and let concrete classes inherit from the base class

### C# 8+ Interface Default Implementations

- C# 8 introduced default interface methods — an interface method can now have a body
- This allows adding new members to an interface without breaking existing implementations
- A class implementing the interface can choose to override the default or use it as-is
- Default interface methods are called through the interface type, not through the implementing class
- This is primarily for backward compatibility — it allows interface evolution without a major version break

```csharp
public interface ILogger
{
    void Log(string message);

    // Default implementation added in C# 8
    void LogError(string message)
    {
        Log($"ERROR: {message}");
    }
}

public class ConsoleLogger : ILogger
{
    public void Log(string message) => Console.WriteLine(message);
    // LogError is inherited from the interface default
}
```

- Important caveat: default interface methods have limitations — they cannot access instance state of the implementing class

---

## Interfaces

### What is an Interface

- An interface is a contract that defines a set of members (methods, properties, events, indexers) that a implementing type must provide
- It specifies what a type can do, not how it does it
- A class or struct that implements an interface must provide implementations for all interface members (unless using default implementations)
- Interfaces enable loose coupling — code can depend on an interface rather than a concrete type
- An interface name conventionally starts with `I` (e.g., `IComparable`, `IDisposable`)

### Interface vs Abstract Class — Detailed Comparison

| Feature | Interface | Abstract Class |
|---|---|---|
| Instantiation | Cannot be instantiated | Cannot be instantiated |
| Inheritance | A class can implement many | A class can inherit one |
| Fields | No instance fields | Can have instance fields |
| Constructors | No constructors | Can have constructors |
| Access modifiers | All members public (pre-C#8) | Any access modifier |
| Default implementation | Yes (C# 8+) | Yes (concrete methods) |
| Static members | Yes (C# 11+, static abstract too) | Yes |
| Events | Yes | Yes |
| Properties | Yes | Yes |
| State | No state (no instance fields) | Can hold state |
| Evolution | Adding members breaks implementers (pre-C#8) | Adding members can break derived if no default |
| Design purpose | Defines capability/contract | Provides base implementation |

### Can an Interface Have Constructors?

- No — interfaces cannot have constructors of any kind
- An interface does not represent a concrete type — it is a contract, not a thing you construct
- You cannot write `new IComparable()` — it makes no semantic sense
- If you need initialization logic, put it in an abstract class or a concrete class

### Can an Interface Have Fields?

- No — interfaces cannot have instance fields or instance variables
- Interfaces can have `const` fields (which are implicitly `static`) — these are compile-time constants
- Interfaces can have `static readonly` fields since C# 11
- The reason: fields represent storage, and interfaces do not define memory layout
- If you need to share state, use an abstract class instead

### Default Interface Methods (C# 8+)

- Allows an interface method to have a body — a default implementation
- Existing classes do not need to implement the new method — they inherit the default
- Useful for adding functionality to an interface without breaking all implementers
- Default methods cannot be called on the implementing class directly — they must be called through the interface

```csharp
public interface IRepository<T>
{
    void Add(T item);
    void Remove(T item);

    // Default implementation
    void AddRange(IEnumerable<T> items)
    {
        foreach (var item in items)
            Add(item);
    }
}
```

- A class can still override the default: `public void AddRange(IEnumerable<T> items) { /* custom */ }`

### Explicit Interface Implementation

- Explicit implementation ties a method to a specific interface — it can only be called through the interface type
- Useful when a class implements multiple interfaces that have methods with the same name
- The method is not accessible through the class type — you must cast to the interface first
- Allows a class to provide different implementations for the same method signature from different interfaces

```csharp
public interface IFlyable
{
    void Move(); // flies
}

public interface ISwimmable
{
    void Move(); // swims
}

public class Duck : IFlyable, ISwimmable
{
    void IFlyable.Move() { Console.WriteLine("Flying"); }
    void ISwimmable.Move() { Console.WriteLine("Swimming"); }
}

Duck d = new Duck();
// d.Move(); // COMPILE ERROR — ambiguous
((IFlyable)d).Move(); // "Flying"
((ISwimmable)d).Move(); // "Swimming"
```

### Multiple Interface Implementation

- A class or struct can implement any number of interfaces
- This is C#'s answer to multiple inheritance — you get multiple contracts without the diamond problem
- When a class implements multiple interfaces, it must provide implementations for all members from all interfaces
- If two interfaces define a method with the same signature, explicit implementation resolves the ambiguity

```csharp
public interface IDrawable { void Draw(); }
public interface IResizable { void Resize(); }

public class Widget : IDrawable, IResizable
{
    public void Draw() { /* ... */ }
    public void Resize() { /* ... */ }
}
```

### Real Interface Examples

#### IComparable

- Defines a method `int CompareTo(object obj)` (or `int CompareTo(T other)` for the generic version)
- Used by sorting algorithms and collection methods to determine ordering
- Returns negative if `this` is less, zero if equal, positive if greater

```csharp
public class Employee : IComparable<Employee>
{
    public int Id { get; set; }

    public int CompareTo(Employee other)
    {
        if (other is null) return 1;
        return Id.CompareTo(other.Id);
    }
}
```

#### IDisposable

- Defines a single method `void Dispose()` for releasing unmanaged resources (file handles, database connections, etc.)
- Enables the `using` statement which calls `Dispose()` automatically at the end of the block
- The dispose pattern often includes a `Dispose(bool disposing)` method and a finalizer

```csharp
public class DatabaseConnection : IDisposable
{
    private SqlConnection _connection;

    public void Dispose()
    {
        _connection?.Close();
        _connection?.Dispose();
    }
}

// Usage:
using (var db = new DatabaseConnection())
{
    // use the connection
} // Dispose() called automatically here
```

#### IEnumerable

- Defines `IEnumerator GetEnumerator()` for iterating over a collection
- Enables `foreach` loops — the compiler calls `GetEnumerator()` behind the scenes
- The generic version `IEnumerable<T>` provides type-safe enumeration

```csharp
public class NumberCollection : IEnumerable<int>
{
    private int[] _numbers = { 1, 2, 3, 4, 5 };

    public IEnumerator<int> GetEnumerator()
    {
        foreach (var n in _numbers)
            yield return n;
    }

    System.Collections.IEnumerator System.Collections.IEnumerable.GetEnumerator()
    {
        return GetEnumerator();
    }
}
```

---

## Polymorphism Deep Dive

### Covariance (`out T`)

- Covariance preserves the inheritance relationship when going from derived to base
- `out` keyword on a generic type parameter means T can only appear in output (return) positions
- Allows a method returning `IEnumerable<Derived>` to be assigned to a variable of type `IEnumerable<Base>`
- The direction is "outward" — from specific to general — hence "co" (with) variance

```csharp
// IEnumerable<out T> — T is covariant
IEnumerable<Derived> derived = new List<Derived>();
IEnumerable<Base> baseEnumerable = derived; // OK — covariant
```

### Contravariance (`in T`)

- Contravariance reverses the inheritance relationship — base can be used where derived is expected
- `in` keyword on a generic type parameter means T can only appear in input (parameter) positions
- Allows a method accepting `Action<Base>` to be assigned to a variable of type `Action<Derived>`
- The direction is "inward" — from general to specific — hence "contra" (against) variance

```csharp
// Action<in T> — T is contravariant
Action<Base> baseAction = (Base b) => Console.WriteLine(b.ToString());
Action<Derived> derivedAction = baseAction; // OK — contravariant
derivedAction(new Derived()); // works — passes Derived (which is a Base) to baseAction
```

### Why Arrays Are Covariant but Generic Collections Are Not

- Arrays were made covariant in C# for compatibility with earlier .NET languages and design decisions
- Array covariance is unsound — it compiles but can throw `ArrayTypeMismatchException` at runtime

```csharp
Base[] arr = new Derived[10]; // compiles — covariant
arr[0] = new Base(); // ArrayTypeMismatchException at runtime!
// the array is actually a Derived[], but you just tried to put a Base in it
```

- Generic collections (like `List<T>`) are invariant by default — `List<Derived>` is NOT assignable to `List<Base>`
- This prevents runtime type errors — `List<T>` checks types at compile time
- You can opt into covariance/contravariance with `out`/`in` on interface type parameters (e.g., `IEnumerable<out T>`, `Action<in T>`)
- The key difference: arrays are reified (they know their element type at runtime), while generic variance is enforced at compile time and by the CLR for interfaces and delegates

### Sealed Classes and Sealed Methods

- A `sealed` class cannot be inherited — it is the final node in the inheritance hierarchy
- A `sealed` method (on an unsealed class) prevents further overriding in derived classes
- Sealing classes allows the JIT compiler to devirtualize method calls — performance optimization
- Commonly used for utility classes, final implementations, and security-sensitive classes
- `string` is a sealed class in .NET — you cannot inherit from it
- Sealing a class is also useful when you want to ensure the behavior cannot be tampered with by malicious derived classes

```csharp
public sealed class FinalImplementation : Base
{
    public override void Foo() { } // no one can override this further
}
```

- `sealed override` is useful when you override a virtual method in a derived class and want to lock it down:

```csharp
public class B : A
{
    public sealed override void DoWork() { /* locked — no further override */ }
}

public class C : B
{
    // public override void DoWork() { } // COMPILE ERROR — sealed
}
```

---

## Real-World Usage Patterns

### Repository Pattern

- Abstracts data access logic behind an interface — business code depends on `IRepository<T>`, not on `DbContext` or SQL directly
- Makes it possible to swap data access implementations (in-memory for testing, SQL Server for production)
- Centralizes common CRUD operations

```csharp
public interface IRepository<T> where T : class
{
    T GetById(int id);
    IEnumerable<T> GetAll();
    void Add(T entity);
    void Update(T entity);
    void Delete(T entity);
}

public class EmployeeRepository : IRepository<Employee>
{
    private readonly AppDbContext _context;

    public EmployeeRepository(AppDbContext context) => _context = context;

    public Employee GetById(int id) => _context.Employees.Find(id);
    public IEnumerable<Employee> GetAll() => _context.Employees.ToList();
    public void Add(Employee entity) => _context.Employees.Add(entity);
    public void Update(Employee entity) => _context.Employees.Update(entity);
    public void Delete(Employee entity) => _context.Employees.Remove(entity);
}
```

### Template Method Pattern

- A base class defines the skeleton of an algorithm in a method, deferring some steps to derived classes
- Uses `abstract` and `virtual` methods — the base class controls the flow, derived classes fill in the details
- The base method is often non-virtual (or sealed) to prevent derived classes from changing the algorithm structure

```csharp
public abstract class ReportGenerator
{
    // Template method — defines the algorithm skeleton
    public void GenerateReport()
    {
        var data = FetchData();
        var formatted = FormatData(data);
        Export(formatted);
    }

    protected abstract object FetchData();       // derived MUST implement
    protected virtual string FormatData(object data) => data.ToString(); // derived MAY override
    protected abstract void Export(string content); // derived MUST implement
}

public class SalesReport : ReportGenerator
{
    protected override object FetchData() => GetSalesData();
    protected override void Export(string content) => File.WriteAllText("sales.txt", content);
    // Uses default FormatData
}
```

### Strategy Pattern Using Interfaces

- Defines a family of algorithms, encapsulates each one behind an interface, and makes them interchangeable
- The context class holds a reference to an `IStrategy` and delegates the algorithm to it
- The strategy can be changed at runtime — different behavior without modifying the context class

```csharp
public interface IPricingStrategy
{
    decimal CalculatePrice(decimal basePrice);
}

public class RegularPricing : IPricingStrategy
{
    public decimal CalculatePrice(decimal basePrice) => basePrice;
}

public class PremiumPricing : IPricingStrategy
{
    public decimal CalculatePrice(decimal basePrice) => basePrice * 0.9m; // 10% discount
}

public class Order
{
    private IPricingStrategy _pricingStrategy;

    public Order(IPricingStrategy strategy) => _pricingStrategy = strategy;

    public decimal Total(decimal basePrice) => _pricingStrategy.CalculatePrice(basePrice);
}

// Usage:
var regularOrder = new Order(new RegularPricing());
var premiumOrder = new Order(new PremiumPricing());
```

### Factory Pattern

- Creates objects without exposing the instantiation logic to the caller
- The factory method returns an interface or base class type, hiding the concrete type
- Useful when object creation is complex, depends on configuration, or you want to decouple creation from usage

```csharp
public interface INotificationService
{
    void Send(string message);
}

public class EmailNotification : INotificationService
{
    public void Send(string message) => Console.WriteLine($"Email: {message}");
}

public class SmsNotification : INotificationService
{
    public void Send(string message) => Console.WriteLine($"SMS: {message}");
}

public static class NotificationFactory
{
    public static INotificationService Create(string type)
    {
        return type.ToLower() switch
        {
            "email" => new EmailNotification(),
            "sms" => new SmsNotification(),
            _ => throw new ArgumentException($"Unknown type: {type}")
        };
    }
}

// Usage:
var notifier = NotificationFactory.Create("email");
notifier.Send("Hello!"); // caller doesn't know the concrete type
```

---

## Interview Questions

### Q1: What is the difference between an abstract class and an interface?

- An abstract class can have fields, constructors, access modifiers, and partial implementation; an interface (pre-C#8) is a pure contract with no implementation
- A class can inherit one abstract class but implement many interfaces
- Abstract classes define an "is-a" relationship; interfaces define a "can-do" relationship
- Abstract classes can have state (fields); interfaces cannot (only constants)
- Since C# 8, interfaces can have default method implementations, narrowing the gap, but still cannot have instance fields or constructors

### Q2: What is the difference between `virtual`, `override`, and `new`?

- `virtual` marks a base class method as overridable — it opts into dynamic dispatch
- `override` provides a new implementation of a virtual/abstract method in a derived class — it participates in polymorphism
- `new` hides a base class method — it creates a separate method that shadows the base, and does NOT participate in polymorphism
- With `override`, the runtime type determines which method is called; with `new`, the compile-time (declared) type determines it

### Q3: What does the `sealed` keyword do?

- On a class: prevents the class from being inherited at all
- On a method: prevents further overriding of that method in derived classes
- Sealing allows the JIT to devirtualize calls — a minor performance optimization
- Sealing is also a design signal: "this is the final implementation"

### Q4: What is the constructor execution order in an inheritance hierarchy?

- Static constructors of the base and derived classes run once, before any instance is created
- Instance constructors execute base-first: base class constructor runs, then derived class constructor body runs
- Full order: base static → derived static → base instance → derived instance
- If you chain constructors with `this()`, the chain resolves before the constructor body runs

### Q5: Can a C# class inherit from multiple base classes?

- No — C# supports single class inheritance only
- A class can implement multiple interfaces, which is the C# alternative to multiple inheritance
- This design avoids the diamond problem and ambiguities that arise with multiple class inheritance

### Q6: What is explicit interface implementation and when would you use it?

- Explicit implementation ties a method to a specific interface — it can only be called by casting to that interface
- Use it when a class implements two interfaces that have methods with the same name but different semantics
- It resolves ambiguity and allows the same method name to have different behaviors depending on the interface context
- The method is not accessible through the class type directly

### Q7: Can an interface have a constructor?

- No — interfaces cannot have constructors of any kind
- Interfaces are contracts, not concrete types — you cannot instantiate an interface
- If you need shared initialization logic, use an abstract class

### Q8: Can an interface have fields?

- No — interfaces cannot have instance fields
- Interfaces can have `const` (implicitly static) fields
- Since C# 11, interfaces can also have `static readonly` fields
- Fields represent storage; interfaces define behavior — they intentionally do not define memory layout

### Q9: What are default interface methods and when were they introduced?

- Default interface methods were introduced in C# 8.0
- They allow an interface method to have a body (implementation)
- Existing implementations do not need to implement the new method — they inherit the default
- Useful for evolving interfaces without breaking all implementers
- Limitation: default methods cannot access instance state of the implementing class

### Q10: What is covariance and contravariance? Give examples.

- Covariance (`out T`): preserves the "is-a" relationship in the output direction — `IEnumerable<Derived>` is assignable to `IEnumerable<Base>`
- Contravariance (`in T`): reverses the "is-a" relationship in the input direction — `Action<Base>` is assignable to `Action<Derived>`
- Covariance: the generic type can only appear in output positions (return types)
- Contravariance: the generic type can only appear in input positions (parameter types)
- Example: `Func<out TResult>` is covariant in `TResult`; `Action<in T>` is contravariant in `T`

### Q11: Why are arrays covariant in C# but `List<T>` is not?

- Arrays are covariant for backward compatibility with earlier .NET designs
- Array covariance is type-unsafe — it compiles but can throw `ArrayTypeMismatchException` at runtime
- `List<T>` is invariant by default — `List<Derived>` is not assignable to `List<Base>` — which prevents runtime errors
- You can opt into variance on interfaces with `in`/`out` keywords (e.g., `IEnumerable<out T>`, `IComparer<in T>`)

### Q12: How does the CLR resolve virtual method calls?

- The CLR builds a virtual method table (vtable) for each type when it is loaded
- Each vtable entry points to the most-derived implementation of that virtual method
- When a virtual call is made, the CLR indexes into the vtable of the actual runtime type
- This is why `obj.Method()` calls the derived implementation even when `obj` is typed as the base class
- Non-virtual calls bypass the vtable — the method address is resolved statically at JIT time

### Q13: What is the difference between `protected` and `private protected`?

- `protected`: accessible from the declaring class AND any derived class (regardless of assembly)
- `private protected`: accessible from the declaring class AND derived classes, but ONLY within the same assembly
- `private protected` is more restrictive — it is the intersection of `protected` and `internal`
- Use `private protected` when you want derived classes to access a member but not outside the assembly

### Q14: What is the difference between `internal` and `protected internal`?

- `internal`: accessible from anywhere within the same assembly
- `protected internal`: accessible from derived classes OR from anywhere in the same assembly (union of `protected` and `internal`)
- `protected internal` is more permissive than `internal` — it adds cross-assembly access for derived classes
- If you want cross-assembly access only for derived classes, use `private protected`

### Q15: Can you override a non-virtual method?

- No — you cannot override a method that is not marked `virtual`, `abstract`, or already `override`
- You can hide a non-virtual method using the `new` keyword, but this is not polymorphism
- The `new` keyword creates a new method that shadows the base — it does not replace it in the vtable
- The compiler generates a warning when `new` is used implicitly — always use it explicitly to signal intent

### Q16: What is the purpose of the `base` keyword?

- `base.Method()` calls the base class implementation of an overridden method
- `base(args)` in a constructor calls a specific base class constructor
- `base` provides access to non-private base class members from within a derived class
- `base` cannot be used in static contexts — it references the base portion of the current instance

### Q17: Can an abstract class have a constructor?

- Yes — abstract classes can have constructors, including parameterized constructors
- The abstract class constructor runs when a derived class constructor is invoked (via `base()` or implicit call)
- It initializes the state of the abstract class portion of the derived object
- You cannot write `new AbstractClass()` — the constructor is only called indirectly through derived classes

### Q18: When would you use the Strategy pattern over simple inheritance?

- When you need to swap algorithms at runtime — inheritance fixes the behavior at compile time
- When you have many variants of a behavior and do not want a class explosion from combinatorial inheritance
- When you want to follow the Open/Closed Principle — add new strategies without modifying existing code
- When different objects of the same class need different behaviors — inheritance gives the same behavior to all instances of a derived class

### Q19: What happens if you do not mark a base method as `virtual` but try to `override` it in a derived class?

- You get a compile error: the method is not marked as virtual, abstract, or override
- You must mark the base method as `virtual` to allow overriding
- If you intended to hide the method instead, use the `new` keyword — but be aware this is not polymorphism

### Q20: What is the diamond problem and how does C# avoid it?

- The diamond problem occurs when a class inherits from two classes that both inherit from a common base — creating a diamond-shaped inheritance graph
- It causes ambiguity: which base class implementation does the derived class inherit?
- C# avoids it by allowing single class inheritance only — a class can inherit from exactly one base class
- Multiple interfaces can be implemented, but interfaces (pre-C#8) have no implementation to conflict
- C# 8 default interface methods reintroduce a limited form of this problem — if two interfaces have the same default method, the implementing class must provide an explicit implementation to resolve the ambiguity

### Q21: What is the difference between `IComparable` and `IComparer<T>`?

- `IComparable<T>` is implemented by the object being compared — it defines how `this` compares to another object: `int CompareTo(T other)`
- `IComparer<T>` is a separate object that compares two other objects: `int Compare(T x, T y)`
- Use `IComparable<T>` when the natural ordering is intrinsic to the class (e.g., `Employee` sorts by `Id`)
- Use `IComparer<T>` when you need external or multiple sorting strategies (e.g., sort employees by name, by salary, by department)

### Q22: Can a class be both abstract and sealed?

- No — this is a logical contradiction: abstract means "must be inherited" and sealed means "cannot be inherited"
- The compiler will reject this combination with a compile error
- You can have a class that is neither abstract nor sealed — a concrete, inheritable class

### Q23: What is the difference between `object.ToString()` and overriding it?

- `object.ToString()` returns the fully qualified type name by default: `"Namespace.ClassName"`
- Override `ToString()` to provide a meaningful string representation of your object
- Override `ToString()` in derived classes to customize what gets logged, displayed, or serialized
- `string.Format`, string interpolation (`$"{obj}"`), and `Console.WriteLine(obj)` all call `ToString()` implicitly
