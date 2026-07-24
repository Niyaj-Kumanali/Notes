# C# Generics

---

## What are Generics

- **Definition**: Generics allow you to define type parameters — placeholders for types — so you can write classes, methods, interfaces, and delegates that work with **any type** while preserving full type safety at compile time.
- **Why they exist**:
  - **Type safety**: The compiler enforces correct types at compile time, eliminating runtime `InvalidCastException` risks.
  - **Code reuse**: Write one implementation that works for `int`, `string`, `Customer`, or any other type — no code duplication.
  - **Performance**: No boxing/unboxing for value types. A `List<int>` stores `int` values directly on the managed heap, not as boxed `object` references.
- **Generic classes**:
  ```csharp
  public class Repository<T>
  {
      private readonly List<T> _items = new();

      public void Add(T item) => _items.Add(item);
      public T GetById(int index) => _items[index];
  }
  ```
- **Generic methods**:
  ```csharp
  public T Max<T>(T a, T b) where T : IComparable<T>
  {
      return a.CompareTo(b) > 0 ? a : b;
  }
  ```
- **Generic interfaces**:
  ```csharp
  public interface IRepository<T>
  {
      void Add(T entity);
      T? GetById(int id);
      IReadOnlyList<T> GetAll();
  }
  ```
- **Generic delegates**:
  ```csharp
  public delegate TResult Func<in T, out TResult>(T arg);
  public delegate void Action<in T>(T obj);
  public delegate bool Predicate<in T>(T obj);
  ```

---

## Generic Type Parameters

- **Naming conventions**:
  - `T` — single obvious type (e.g., `List<T>`, `Nullable<T>`)
  - `TKey` — dictionary key type
  - `TValue` — dictionary value type
  - `TResult` — return type of a function
  - `TInput` — input type in a converter
  - `TSource` — source collection element type
  - `TElement` — generic element type
  - `TParam` — parameter type
  - The names are only conventions; the compiler does not enforce them.
- **Multiple type parameters**:
  ```csharp
  public class Dictionary<TKey, TValue> { ... }
  public class Tuple<T1, T2, T3> { ... }
  public static TResult Map<TInput, TResult>(TInput input, Func<TInput, TResult> mapper) { ... }
  ```
- **开放类型 vs 封闭类型 (Open vs Closed Generic Types)**:
  - **Open generic type (开放类型)**: A generic type that still has unresolved type parameters.
    - `List<T>` is open — `T` is not yet bound to a concrete type.
    - Cannot be instantiated: `new List<T>()` is a compile error inside a non-generic context.
  - **Closed generic type (封闭类型)**: A generic type where all type parameters have been replaced with concrete types.
    - `List<int>` is closed — `T` is bound to `int`.
    - Can be instantiated: `new List<int>()` is valid.
  - **Constructed generic type**: A generic type that has been partially or fully bound.
    - `Dictionary<string, T>` is partially constructed (still open because of `T`).
    - `Dictionary<string, int>` is fully constructed (closed).
  - **Generic type definition vs constructed type**:
    - `typeof(List<>)` returns the open generic type definition.
    - `typeof(List<int>)` returns the closed constructed type.
  - **Runtime behavior**:
    - CLR creates shared code for reference type arguments (all `List<string>`, `List<Customer>` share the same native code).
    - CLR generates separate native code for each value type argument (`List<int>`, `List<double>` each get their own machine code).
  - **Reflection with open types**:
    ```csharp
    Type openList = typeof(List<>);           // Open generic type definition
    Type closedList = typeof(List<int>);      // Closed constructed type
    Type madeList = typeof(List<>).MakeGenericType(typeof(string));  // Makes List<string>
    ```

---

## Generic Constraints

- Constraints restrict what types can be used as type arguments, and they **unlock capabilities** on the type parameter.
- Without constraints, `T` is treated as `object` — you can only call methods defined on `System.Object`.

### `where T : class` (Reference Type Constraint)

- `T` must be a reference type (class, interface, delegate, array, or `object`).
- Allows null comparisons: `if (item == null)`.
- Allows `as` operator: `var derived = item as DerivedClass`.
- Allows `?` nullable reference annotations.
- **Does NOT** mean `T` cannot be null — `T` can still be `null` unless combined with `notnull`.

### `where T : struct` (Value Type Constraint)

- `T` must be a non-nullable value type (struct, enum, or any type defined with `struct`).
- `T` cannot be nullable (`Nullable<T>` is allowed as a special case since C# 8).
- Enables efficient memory layout — no heap allocation for `T` itself.
- `T` is guaranteed to have a parameterless constructor (value types always do).

### `where T : class?` (Nullable Reference Type Constraint)

- `T` must be a reference type and **may be null**.
- Used in nullable-enabled codebases to explicitly express that `T` can be null.
- Introduced in C# 9 / .NET 5.

### `where T : notnull` (Non-Nullable Constraint)

- `T` must be a non-nullable type.
- For reference types: `T` must not be annotated as nullable.
- For value types: both `Nullable<T>` and non-nullable value types are allowed (C# 8–9), or only non-nullable value types (C# 10+).
- Primarily useful for nullable-enabled codebases.

### `where T : new()` (Parameterless Constructor Constraint)

- `T` must have a public parameterless constructor.
- Allows: `var instance = new T();`
- Commonly used in factory methods, object pools, and activator-based patterns.
- **Limitation**: Cannot specify constructors with parameters — only parameterless.

### `where T : BaseClass` (Inheritance Constraint)

- `T` must derive from (or be) the specified base class.
- Enables calling base class methods and accessing protected members.
- ```csharp
  public void Process<T>(T entity) where T : EntityBase
  {
      entity.Id = Guid.NewGuid();
      entity.CreatedAt = DateTime.UtcNow;
  }
  ```

### `where T : IInterface` (Interface Implementation Constraint)

- `T` must implement the specified interface.
- Enables calling interface methods on `T`.
- Multiple interfaces can be specified.
- ```csharp
  public T Find<T>(IEnumerable<T> items, T target) where T : IEquatable<T>
  {
      return items.FirstOrDefault(item => item.Equals(target));
  }
  ```

### `where T : U` (Type Parameter Constraint)

- `T` must derive from or be the same as another type parameter `U`.
- Enables generic substitution and polymorphism between type parameters.
- ```csharp
  public void Copy<T, U>(T source, U destination)
      where T : U
  {
      // source can be used wherever U is expected
      U copy = source;
  }
  ```
- Useful in patterns like generic decorators, adapters, and middleware chains.

### Multiple Constraints on One Type Parameter

- Separate constraints with commas — no `where` keyword repeats for the same parameter.
- `class` and `struct` are mutually exclusive — cannot combine them.
- Order does not matter, but convention is: interface constraints first, then class/base, then `new()`.
- ```csharp
  public class Cache<T> where T : class, ICacheable, ICloneable, new()
  {
      public T CreateDefault() => new T();
  }
  ```

### Why Constraints Exist

- Without constraints, the compiler treats `T` as `System.Object` (or `object` with no members beyond `ToString`, `Equals`, `GetHashCode`).
- Constraints tell the compiler: "I guarantee `T` has these capabilities" — which allows calling specific methods, properties, or operators.
- Constraints are compile-time only — they do not affect runtime behavior beyond enabling/disabling code paths.
- They serve as documentation: anyone reading the generic definition immediately knows what types are valid.

---

## Generic Methods

- A generic method declares its **own type parameters**, separate from the enclosing class's type parameters.
- ```csharp
  public class Processor
  {
      // This method has its own type parameter T
      public void Log<T>(T value)
      {
          Console.WriteLine(value?.ToString());
      }
  }
  ```
- **Type inference**:
  - The compiler can infer `T` from the arguments passed to the method.
  - `processor.Log(42)` infers `T = int`.
  - `processor.Log("hello")` infers `T = string`.
  - If inference fails, you must specify `T` explicitly: `processor.Log<string>(null)`.
- **When to use generic methods vs generic classes**:
  - **Generic class**: When the entire class operates on a single type consistently (e.g., `Repository<T>`).
  - **Generic method**: When only a single method needs type flexibility, while the class itself is not generic (e.g., a utility method that works with any type).
  - Use a generic method when the type varies **per call**, not per instance.

---

## Variance in Generics

- Variance describes how generic type arguments relate when substituting derived types for base types.
- Three kinds of variance in C#:

### Covariance (`out T`)

- Allows a `Generic<Derived>` to be used where a `Generic<Base>` is expected.
- The type parameter only appears in **output** (return) positions.
- `IEnumerable<out T>` — you can read `T` from the collection, but cannot add to it.
- ```csharp
  IEnumerable<Animal> animals = new List<Dog>(); // Valid: Dog is derived from Animal
  Animal animal = animals.First();               // Reading a Dog as an Animal
  ```
- `out` modifier on the type parameter enables covariance.

### Contravariance (`in T`)

- Allows a `Generic<Base>` to be used where a `Generic<Derived>` is expected.
- The type parameter only appears in **input** (parameter) positions.
- `Action<in T>` — you can pass `Derived` where `Base` is expected.
- ```csharp
  Action<Animal> feedAnimal = (Animal a) => a.Feed();
  Action<Dog> feedDog = feedAnimal;   // Valid: can pass Dog wherever Animal is expected
  feedDog(new Dog());                 // Calls feedAnimal with a Dog
  ```
- `in` modifier on the type parameter enables contravariance.

### Invariance (No Modifier)

- No conversion is allowed between `Generic<Derived>` and `Generic<Base>`.
- `List<T>` is invariant — `List<Dog>` cannot be assigned to `List<Animal>`.
- ```csharp
  List<Animal> animals = new List<Dog>(); // Compile error!
  ```
- The type parameter appears in both input and output positions, or the designer chose not to declare variance.

### Why Arrays Are Covariant but `List<T>` Is Not

- Arrays were designed in .NET 1.0 before generics existed — covariance was baked into the CLR for arrays.
- Array covariance is **unsafe**: `Animal[] animals = new Dog[10]; animals[0] = new Cat();` compiles but throws `ArrayTypeMismatchException` at runtime.
- `List<T>` is invariant by design because it is both read **and** write — making it covariant would allow the same runtime type-safety violation.
- `List<T>` provides `AsReadOnly()` returning `IReadOnlyList<out T>` which is covariant — but only for reading.

### Real-World Examples of Variance

- `IEnumerable<out T>` — covariant: iterate over items as a base type.
- `IReadOnlyList<out T>` — covariant: indexed read access as a base type.
- `IComparer<in T>` — contravariant: compare any two `T` objects.
- `Action<in T>` — contravariant: accept `T` as input.
- `Func<in T, out TResult>` — contravariant in `T`, covariant in `TResult`.
- `IEqualityComparer<T>` — invariant (uses `T` in both input and output).
- `Lazy<out T>` — covariant: lazy evaluation producing a `T`.

---

## Generic Collections

- `List<T>`:
  - Dynamic array backed by `T[]` internally.
  - O(1) indexed access, amortized O(1) `Add`, O(n) `Insert`/`Remove`.
  - Best for indexed access and iteration.
- `Dictionary<TKey, TValue>`:
  - Hash table implementation — O(1) average lookup/insert/delete.
  - Requires `GetHashCode()` and `Equals()` implementations on `TKey`.
  - Use `IEqualityComparer<TKey>` for custom key comparison.
- `HashSet<T>`:
  - Hash set — O(1) average `Add`, `Remove`, `Contains`.
  - No duplicates allowed.
  - Useful for set operations: `UnionWith`, `IntersectWith`, `ExceptWith`.
- `Queue<T>`:
  - FIFO collection — `Enqueue`, `Dequeue`, `Peek`.
  - Backed by a circular array internally.
  - O(1) enqueue and dequeue.
- `Stack<T>`:
  - LIFO collection — `Push`, `Pop`, `Peek`.
  - O(1) push and pop.
- `LinkedList<T>`:
  - Doubly-linked list — O(1) insert/remove at known positions.
  - No indexed access — O(n) traversal.
  - Useful when frequent insertion/removal at arbitrary positions is needed.
- `SortedDictionary<TKey, TValue>`:
  - Red-black tree — O(log n) operations.
  - Keys kept in sorted order.
  - Uses `IComparer<TKey>` for ordering.
- `ConcurrentDictionary<TKey, TValue>`:
  - Thread-safe hash table for concurrent access.
  - Uses fine-grained locking for better throughput.
- **Why generic collections are better than non-generic**:
  - `ArrayList` stores everything as `object` — adding a `Dog` to an `ArrayList` that should contain only `Cat` objects compiles fine but fails at runtime.
  - `Hashtable` has the same problem — no type safety on keys or values.
  - Generic collections enforce type safety at compile time.
- **Performance difference (boxing/unboxing)**:
  - `ArrayList.Add(42)` boxes the `int` — allocates a new object on the heap and copies the value.
  - `List<int>.Add(42)` stores the `int` directly in the internal `int[]` — no heap allocation per element.
  - Boxing cost: memory allocation + GC pressure + potential cache misses.
  - For collections of millions of value-type elements, the performance difference is significant.

---

## Generic Interfaces and Delegates

### `IEqualityComparer<T>`

- Defines `Equals(T, T)` and `GetHashCode(T)` for custom equality comparison.
- Used by `Dictionary<TKey, TValue>`, `HashSet<T>`, and LINQ's `Distinct()`.
- ```csharp
  public class CaseInsensitiveComparer : IEqualityComparer<string>
  {
      public bool Equals(string x, string y) =>
          string.Equals(x, y, StringComparison.OrdinalIgnoreCase);

      public int GetHashCode(string obj) =>
          obj?.ToUpperInvariant().GetHashCode() ?? 0;
  }
  ```

### `IComparer<T>`

- Defines `Compare(T, T)` — returns negative, zero, or positive.
- Used for sorting (`Array.Sort`, `List<T>.Sort`) and ordered collections.
- ```csharp
  public class ReverseComparer<T> : IComparer<T> where T : IComparable<T>
  {
      public int Compare(T x, T y) => y.CompareTo(x);
  }
  ```

### `IEquatable<T>`

- Defines `Equals(T)` — type-safe equality without boxing.
- Preferred over `object.Equals()` for value types.
- `List<T>.Contains`, `Array.IndexOf`, and LINQ `SequenceEqual` use `IEquatable<T>` when available.
- ```csharp
  public struct Point : IEquatable<Point>
  {
      public int X { get; init; }
      public int Y { get; init; }

      public bool Equals(Point other) => X == other.X && Y == other.Y;
      public override bool Equals(object obj) => obj is Point p && Equals(p);
      public override int GetHashCode() => HashCode.Combine(X, Y);
  }
  ```

### `Func<T, TResult>` and `Action<T>`

- `Func<T, TResult>` — delegate that takes `T` and returns `TResult`. Overloaded for up to 16 parameters.
- `Action<T>` — delegate that takes `T` and returns `void`. Overloaded for up to 16 parameters.
- Used extensively in LINQ, event handling, and functional-style C# code.
- ```csharp
  Func<int, int, int> add = (a, b) => a + b;
  Action<string> print = msg => Console.WriteLine(msg);
  ```

### `Predicate<T>`

- Delegate that takes `T` and returns `bool`.
- Used by `List<T>.Find`, `List<T>.Exists`, `List<T>.RemoveAll`.
- ```csharp
  Predicate<int> isEven = n => n % 2 == 0;
  List<int> evens = numbers.FindAll(isEven);
  ```

### `Comparison<T>` and `Converter<TInput, TOutput>`

- `Comparison<T>` — delegate for comparison, used by `List<T>.Sort(Comparison<T>)`.
- `Converter<TInput, TOutput>` — delegate for type conversion, used by `List<T>.ConvertAll(Converter<TInput, TOutput>)`.
- ```csharp
  Comparison<int> desc = (a, b) => b.CompareTo(a);
  numbers.Sort(desc);

  Converter<int, string> toString = n => n.ToString();
  List<string> strings = numbers.ConvertAll(toString);
  ```

---

## Common Mistakes

- **Not using constraints when specific operations are needed**:
  - Writing `public T Create<T>() => new T();` without `where T : new()` causes a compile error.
  - Forgetting `where T : class` when doing null checks or `as` casts on `T`.

- **Overusing `where T : new()`**:
  - Limits flexibility — value types and many reference types do not have public parameterless constructors.
  - Consider factory functions (`Func<T>`) instead of constructor constraints when possible.

- **Confusing covariance and contravariance**:
  - Covariance (`out T`): think "reading" — `IEnumerable<out T>` lets you read items as a base type.
  - Contravariance (`in T`): think "writing" — `Action<in T>` lets you write items of a derived type.
  - Rule of thumb: `out` = returns, `in` = parameters.

- **Generic type inference failures**:
  - `new T()` without constraint — compiler cannot verify `T` has a parameterless constructor.
  - Passing `null` for a value type `T` — inference fails or produces unexpected results.
  - Ambiguous overloads where multiple generic methods match.

- **Boxing when using non-generic collections with value types**:
  - `ArrayList.Add(42)` boxes the `int` — each element is a separate heap allocation.
  - `Hashtable.Add(key, value)` boxes value-type keys and values.
  - Always prefer `List<T>` over `ArrayList`, `Dictionary<TKey, TValue>` over `Hashtable`.

---

## Real-World Usage

- **Generic repository pattern**:
  ```csharp
  public interface IRepository<T> where T : class, IEntity
  {
      Task<T?> GetByIdAsync(int id);
      Task<IReadOnlyList<T>> GetAllAsync();
      Task AddAsync(T entity);
      Task UpdateAsync(T entity);
      Task DeleteAsync(int id);
  }

  public class EfRepository<T> : IRepository<T> where T : class, IEntity
  {
      private readonly DbContext _context;
      private readonly DbSet<T> _dbSet;

      public EfRepository(DbContext context)
      {
          _context = context;
          _dbSet = context.Set<T>();
      }

      public async Task<T?> GetByIdAsync(int id) =>
          await _dbSet.FindAsync(id);

      public async Task<IReadOnlyList<T>> GetAllAsync() =>
          await _dbSet.ToListAsync();

      public async Task AddAsync(T entity) =>
          await _dbSet.AddAsync(entity);

      public async Task UpdateAsync(T entity) =>
          _dbSet.Update(entity);

      public async Task DeleteAsync(int id)
      {
          var entity = await _dbSet.FindAsync(id);
          if (entity != null) _dbSet.Remove(entity);
      }
  }
  ```

- **Generic service layer**:
  ```csharp
  public interface IService<TDto, TEntity> where TEntity : class
  {
      Task<TDto?> GetByIdAsync(int id);
      Task<IReadOnlyList<TDto>> GetAllAsync();
      Task<TDto> CreateAsync(TDto dto);
  }

  public class ProductService : IService<ProductDto, Product>
  {
      private readonly IRepository<Product> _repository;
      private readonly IMapper _mapper;

      public ProductService(IRepository<Product> repository, IMapper mapper)
      {
          _repository = repository;
          _mapper = mapper;
      }

      public async Task<ProductDto?> GetByIdAsync(int id)
      {
          var product = await _repository.GetByIdAsync(id);
          return product == null ? null : _mapper.Map<ProductDto>(product);
      }

      public async Task<IReadOnlyList<ProductDto>> GetAllAsync()
      {
          var products = await _repository.GetAllAsync();
          return products.Select(p => _mapper.Map<ProductDto>(p)).ToList();
      }

      public async Task<ProductDto> CreateAsync(ProductDto dto)
      {
          var product = _mapper.Map<Product>(dto);
          await _repository.AddAsync(product);
          return _mapper.Map<ProductDto>(product);
      }
  }
  ```

- **Generic API response wrapper**:
  ```csharp
  public class ApiResponse<T>
  {
      public bool Success { get; init; }
      public string Message { get; init; } = string.Empty;
      public T? Data { get; init; }
      public List<string> Errors { get; init; } = new();

      public static ApiResponse<T> Ok(T data, string message = "Success") =>
          new() { Success = true, Data = data, Message = message };

      public static ApiResponse<T> Fail(string message, List<string>? errors = null) =>
          new() { Success = false, Message = message, Errors = errors ?? new() };
  }

  public class ApiResponse
  {
      public bool Success { get; init; }
      public string Message { get; init; } = string.Empty;
      public List<string> Errors { get; init; } = new();

      public static ApiResponse Ok(string message = "Success") =>
          new() { Success = true, Message = message };

      public static ApiResponse Fail(string message, List<string>? errors = null) =>
          new() { Success = false, Message = message, Errors = errors ?? new() };
  }
  ```

- **Generic validator**:
  ```csharp
  public interface IValidator<T>
  {
      ValidationResult Validate(T entity);
  }

  public class ValidationResult
  {
      public bool IsValid => !Errors.Any();
      public List<string> Errors { get; } = new();

      public void AddError(string error) => Errors.Add(error);
  }

  public class CustomerValidator : IValidator<Customer>
  {
      public ValidationResult Validate(Customer entity)
      {
          var result = new ValidationResult();

          if (string.IsNullOrWhiteSpace(entity.Name))
              result.AddError("Name is required");

          if (entity.Age < 0 || entity.Age > 150)
              result.AddError("Age must be between 0 and 150");

          if (string.IsNullOrWhiteSpace(entity.Email))
              result.AddError("Email is required");

          return result;
      }
  }
  ```

- **Generic cache**:
  ```csharp
  public interface ICache<T> where T : class
  {
      T? Get(string key);
      void Set(string key, T value, TimeSpan? expiration = null);
      bool TryGet(string key, out T? value);
      void Remove(string key);
  }

  public class MemoryCache<T> : ICache<T> where T : class
  {
      private readonly Dictionary<string, CacheEntry> _cache = new();
      private readonly object _lock = new();

      private class CacheEntry
      {
          public T Value { get; init; } = default!;
          public DateTime? Expiration { get; init; }
      }

      public T? Get(string key)
      {
          lock (_lock)
          {
              if (_cache.TryGetValue(key, out var entry))
              {
                  if (entry.Expiration.HasValue && entry.Expiration.Value < DateTime.UtcNow)
                  {
                      _cache.Remove(key);
                      return null;
                  }
                  return entry.Value;
              }
              return null;
          }
      }

      public void Set(string key, T value, TimeSpan? expiration = null)
      {
          lock (_lock)
          {
              _cache[key] = new CacheEntry
              {
                  Value = value,
                  Expiration = expiration.HasValue
                      ? DateTime.UtcNow.Add(expiration.Value)
                      : null
              };
          }
      }

      public bool TryGet(string key, out T? value)
      {
          value = Get(key);
          return value != null;
      }

      public void Remove(string key)
      {
          lock (_lock)
          {
              _cache.Remove(key);
          }
      }
  }
  ```

---

## Interview Questions

### Question 1: What are generics in C# and why should you use them?

- Generics allow you to write classes, methods, interfaces, and delegates with **type parameters** — placeholders that are replaced with concrete types at compile time.
- You should use them because they provide **type safety** (compile-time checking), **code reuse** (one implementation for all types), and **performance** (no boxing/unboxing for value types).
- Without generics, you would need to use `object` everywhere, losing type safety and incurring boxing costs.

### Question 2: What is the difference between `List<T>` and `ArrayList`?

- `List<T>` is generic — stores `T` directly, no boxing, full type safety at compile time.
- `ArrayList` is non-generic — stores everything as `object`, boxing value types, no compile-time type safety.
- `List<int>` stores `int` values in a contiguous `int[]` — O(1) indexed access without boxing.
- `ArrayList.Add(42)` boxes the `int` into an `object` on the heap — allocates memory and adds GC pressure.

### Question 3: Explain the `where T : class` constraint.

- Restricts `T` to reference types only — classes, interfaces, delegates, arrays.
- Enables null comparisons (`item == null`), `as` casts, and nullable annotations.
- Does NOT prevent `T` from being null — use `where T : class, notnull` for non-null reference types.
- Cannot be combined with `where T : struct`.

### Question 4: What is the difference between `where T : new()` and a factory pattern?

- `new()` constraint requires `T` to have a public parameterless constructor — allows `new T()`.
- Factory pattern (`Func<T>` or `IFactory<T>`) is more flexible — can create instances with constructor parameters, use DI, or return cached instances.
- `new()` is simpler for basic cases but inflexible for complex object creation.

### Question 5: Explain covariance and contravariance with examples.

- **Covariance (`out T`)**: Allows `IEnumerable<Derived>` to be used as `IEnumerable<Base>`. The type parameter only appears in output positions (return types).
- **Contravariance (`in T`)**: Allows `Action<Base>` to be used as `Action<Derived>`. The type parameter only appears in input positions (parameters).
- **Invariance**: No conversion allowed — `List<Derived>` cannot be assigned to `List<Base>`.
- Example: `IEnumerable<Dog>` is covariant, so `IEnumerable<Dog> dogs = new List<Dog>(); IEnumerable<Animal> animals = dogs;` compiles fine.

### Question 6: Why are arrays covariant but `List<T>` is not?

- Arrays were designed before generics (in .NET 1.0) and covariance was added to the CLR.
- Array covariance is unsafe: `Animal[] animals = new Dog[10]; animals[0] = new Cat();` compiles but throws `ArrayTypeMismatchException` at runtime.
- `List<T>` is invariant because it is both readable and writable — making it covariant would allow the same type-unsafe operations.
- `List<T>.AsReadOnly()` returns `IReadOnlyList<out T>` which is covariant but read-only.

### Question 7: What is type inference in generic methods?

- When calling a generic method, the compiler can infer the type parameter from the arguments.
- `Max(3, 5)` infers `T = int`; `Max("a", "z")` infers `T = string`.
- If inference fails (e.g., ambiguous types, no arguments), you must specify `T` explicitly: `Max<int>(3, 5)`.
- Inference uses the **best common type** from all arguments.

### Question 8: What is the difference between an open and a closed generic type?

- **Open generic type**: Type parameters are still unresolved — `List<T>` is open.
- **Closed generic type**: All type parameters are bound to concrete types — `List<int>` is closed.
- You cannot instantiate an open generic type: `new List<T>()` fails outside a generic context.
- At runtime, the CLR generates separate code for each value-type closed type but shares code for reference-type closed types.

### Question 9: Explain the `where T : U` constraint.

- `T` must be the same as or derive from `U`, where `U` is another type parameter.
- Enables generic substitution: `T` can be used wherever `U` is expected.
- Useful in generic decorators, middleware, and adapter patterns.
- Example: `public void Copy<T, U>(T source, U target) where T : U` — `T` can be assigned to `U`.

### Question 10: How do generic constraints unlock capabilities on type parameters?

- Without constraints, `T` is treated as `System.Object` — only `ToString()`, `Equals()`, `GetHashCode()` are available.
- `where T : IComparable<T>` enables `a.CompareTo(b)`.
- `where T : new()` enables `new T()`.
- `where T : BaseEntity` enables `entity.Id`, `entity.CreatedAt`, etc.
- Constraints are compile-time only — they do not affect runtime behavior beyond enabling/disabling code paths.

### Question 11: What is the performance difference between generic and non-generic collections?

- Generic collections avoid **boxing/unboxing** for value types.
- `List<int>.Add(42)` stores the `int` directly — no heap allocation.
- `ArrayList.Add(42)` boxes the `int` — allocates a new `object` on the heap, copies the value.
- For millions of elements, boxing can cause significant memory pressure and GC overhead.
- Generic collections also have better JIT optimization because the runtime can generate type-specific code.

### Question 12: When would you use a generic method instead of a generic class?

- **Generic class**: When the entire class consistently operates on one type — e.g., `Repository<T>`, `Cache<T>`, `Service<T>`.
- **Generic method**: When only one method needs type flexibility — e.g., `T Max<T>(T a, T b)`, `void Log<T>(T value)`.
- Use a generic method when the type varies **per call**, not per instance.

### Question 13: Explain `IEqualityComparer<T>` vs `IEquatable<T>`.

- `IEquatable<T>`: Defines `Equals(T)` — used by the object itself to compare equality. Preferred for value types to avoid boxing.
- `IEqualityComparer<T>`: Defines `Equals(T, T)` and `GetHashCode(T)` — an external comparer that can be injected into collections.
- `Dictionary<TKey, TValue>` uses `IEqualityComparer<TKey>` for key comparison.
- `List<T>.Contains` uses `IEquatable<T>` when available, otherwise falls back to `object.Equals`.

### Question 14: What are the common generic delegate types in C#?

- `Func<T, TResult>` — takes `T`, returns `TResult`. Up to 16 type parameters.
- `Action<T>` — takes `T`, returns `void`. Up to 16 type parameters.
- `Predicate<T>` — takes `T`, returns `bool`.
- `Comparison<T>` — takes two `T`, returns `int`. Used by `List<T>.Sort`.
- `Converter<TInput, TOutput>` — takes `TInput`, returns `TOutput`. Used by `List<T>.ConvertAll`.

### Question 15: Can you constrain a type parameter to multiple interfaces?

- Yes — separate constraints with commas, no repeated `where` keyword.
- ```csharp
  public void Process<T>(T item) where T : IComparable<T>, ICloneable, IDisposable
  ```
- `class` and `struct` cannot be combined with each other.
- `new()` can be combined with interface and base class constraints.

### Question 16: What happens at the runtime level with generic types?

- The CLR handles generic types differently for value types vs reference types.
- For **reference type** arguments, the CLR shares a single implementation — `List<string>` and `List<Customer>` share the same native code.
- For **value type** arguments, the CLR generates **separate native code** for each closed type — `List<int>` and `List<double>` each get their own machine code.
- This is why `List<int>` is more efficient than `List<object>` — no boxing, no indirection.

### Question 17: How does the generic repository pattern improve code?

- Eliminates repetitive CRUD code for each entity type.
- Provides a consistent interface across all entity types.
- Makes it easy to add cross-cutting concerns (logging, caching, validation) in one place.
- Testable — mock `IRepository<T>` in unit tests.
- Works seamlessly with dependency injection frameworks.
