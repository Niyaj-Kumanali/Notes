# Unit Testing & Integration Testing in .NET (xUnit, NUnit, Moq)

---

## Unit Testing Fundamentals

### What Is a Unit Test

- A unit test is a small, focused piece of code that tests a single "unit" -- typically one method or one class -- in complete isolation from the rest of the application
- The unit under test is exercised directly, and dependencies are replaced with test doubles (mocks, stubs, fakes) so the test validates only the logic you care about
- A single unit test should verify one specific behavior or outcome
- Unit tests are the lowest level of the testing pyramid: fast, numerous, and cheap to write

```csharp
public class Calculator
{
    public int Add(int a, int b) => a + b;
}

public class CalculatorTests
{
    private readonly Calculator _calculator = new();

    [Fact]
    public void Add_TwoPositiveNumbers_ReturnsSum()
    {
        // Arrange
        int a = 2, b = 3;

        // Act
        int result = _calculator.Add(a, b);

        // Assert
        Assert.Equal(5, result);
    }
}
```

- This test exercises exactly one method (`Add`) on one class (`Calculator`) and verifies one behavior

---

### Why Unit Testing Matters

- **Catch bugs early** -- bugs found during development cost a fraction to fix compared to bugs found in production
- **Regression safety** -- when you refactor or add new features, existing unit tests immediately tell you if you broke something
- **Design feedback** -- code that is easy to unit test is almost always well-designed: small classes, clear responsibilities, loose coupling, explicit dependencies
- **Living documentation** -- well-written tests describe exactly what the code is supposed to do, serving as executable specifications
- **Confidence to change** -- a strong test suite lets you refactor aggressively without fear
- **Faster debugging** -- when a test fails, you know exactly which piece of logic broke, narrowing the search space immediately
- **CI/CD foundation** -- unit tests run in seconds and can gate every commit, giving rapid feedback to the whole team

---

### The AAA Pattern (Arrange, Act, Assert)

- Every unit test follows three distinct phases:

- **Arrange** -- set up the test: create objects, configure mocks, define inputs
- **Act** -- invoke the method under test with the arranged inputs
- **Assert** -- verify the result matches expectations

```csharp
[Fact]
public void Divide_ValidNumbers_ReturnsCorrectResult()
{
    // Arrange
    var service = new MathService();
    double numerator = 10.0;
    double denominator = 2.0;

    // Act
    double result = service.Divide(numerator, denominator);

    // Assert
    Assert.Equal(5.0, result);
}
```

```csharp
[Fact]
public void Divide_ByZero_ThrowsDivideByZeroException()
{
    // Arrange
    var service = new MathService();
    double numerator = 10.0;
    double denominator = 0.0;

    // Act & Assert
    Assert.Throws<DivideByZeroException>(() => service.Divide(numerator, denominator));
}
```

- Some teams combine Act and Assert when testing for exceptions -- this is acceptable as long as readability is not harmed
- Comments above each section are optional but helpful for beginners

---

### Test Naming Conventions

- The most common convention is: `Should_ExpectedBehavior_When_Condition`

```csharp
[Fact]
public void CalculateDiscount_WithVIPCustomer_Returns20Percent() { }

[Fact]
public void CalculateDiscount_WithRegularCustomer_Returns5Percent() { }

[Fact]
public void PlaceOrder_WhenStockIsEmpty_ThrowsOutOfStockException() { }

[Fact]
public void SendEmail_WithInvalidAddress_ThrowsFormatException() { }
```

- Alternative conventions include BDD-style: `Given_X_When_Y_Then_Z`
- The key rule: the test name should tell you **what** was tested, **what the expected outcome is**, and **under what conditions** -- so that when it fails, you immediately understand what broke

---

### Characteristics of Good Unit Tests

- **Fast** -- unit tests should run in milliseconds. If a test takes more than a few hundred milliseconds, it is probably an integration test masquerading as a unit test
- **Isolated** -- tests must not depend on each other. Any test can run in any order with no side effects
- **Repeatable** -- the same test, run 1,000 times, must produce the same result every time. No randomness, no time dependency, no flakiness
- **Self-validating** -- a test must have a clear pass/fail outcome. It should not require a human to inspect output to determine if it passed
- **Timely** -- tests should be written at the same time as the production code, or ideally before it (TDD)
- **Focused** -- each test verifies one behavior. If you can remove the test without losing any assertion coverage, it was redundant

---

## xUnit

### What Is xUnit

- xUnit.net is the modern, community-driven testing framework for .NET, and is the **default test framework** for new .NET projects created with `dotnet new`
- It was created by the original authors of NUnit and was designed to fix what they saw as design flaws in NUnit
- xUnit has no `[TestFixture]` attribute -- any public class with test methods is automatically a test class
- xUnit runs tests in parallel by default (each test class runs in its own thread)
- Built-in support for parameterized tests, test collections, and async testing

```bash
dotnet new xunit -n MyTests
dotnet add package xunit
dotnet add package xunit.runner.visualstudio
```

---

### [Fact] vs [Theory] Attributes

- **`[Fact]`** -- marks a simple test with no parameters. It runs once with no input data.

```csharp
[Fact]
public void Add_TwoPositiveNumbers_ReturnsSum()
{
    var calc = new Calculator();
    Assert.Equal(5, calc.Add(2, 3));
}
```

- **`[Theory]`** -- marks a parameterized test. It runs once for each set of input data provided via `[InlineData]`, `[MemberData]`, or `[ClassData]`.

```csharp
[Theory]
[InlineData(1, 1, 2)]
[InlineData(0, 0, 0)]
[InlineData(-1, 1, 0)]
[InlineData(100, 200, 300)]
public void Add_VariousInputs_ReturnsCorrectSum(int a, int b, int expected)
{
    var calc = new Calculator();
    Assert.Equal(expected, calc.Add(a, b));
}
```

- `[Fact]` is for one-off tests; `[Theory]` is for testing the same logic with multiple data sets
- Both attributes support the `Skip` parameter: `[Fact(Skip = "reason")]` or `[Theory(Skip = "reason")]`

---

### [InlineData], [MemberData], [ClassData] for Parameterized Tests

**`[InlineData]`** -- simple inline values directly on the attribute:

```csharp
[Theory]
[InlineData("hello", 5)]
[InlineData("a", 1)]
[InlineData("", 0)]
public void StringLength_ReturnsCorrectLength(string input, int expectedLength)
{
    Assert.Equal(expectedLength, input.Length);
}
```

**`[MemberData]`** -- references a static property or method that returns `IEnumerable<object[]>`:

```csharp
public static IEnumerable<object[]> StringLengthCases =>
    new List<object[]>
    {
        new object[] { "hello", 5 },
        new object[] { "world", 5 },
        new object[] { "", 0 }
    };

[Theory]
[MemberData(nameof(StringLengthCases))]
public void StringLength_WithMemberData_ReturnsCorrectLength(string input, int expected)
{
    Assert.Equal(expected, input.Length);
}
```

**`[ClassData]`** -- references a class that implements `IEnumerable<object[]>`:

```csharp
public class StringLengthTestData : IEnumerable<object[]>
{
    public IEnumerator<object[]> GetEnumerator()
    {
        yield return new object[] { "hello", 5 };
        yield return new object[] { "a", 1 };
        yield return new object[] { "", 0 };
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}

[Theory]
[ClassData(typeof(StringLengthTestData))]
public void StringLength_WithClassData_ReturnsCorrectLength(string input, int expected)
{
    Assert.Equal(expected, input.Length);
}
```

- Use `[InlineData]` for simple, static values
- Use `[MemberData]` or `[ClassData]` when you need complex objects, computed data, or data that lives in a separate file

---

### Test Class Lifecycle

- xUnit creates a **new instance** of the test class for each `[Fact]` or `[Theory]` run -- so every test starts with a fresh object
- You can use constructor injection and `IDisposable.Dispose` (or `IAsyncLifetime`) for per-test setup and teardown:

```csharp
public class DatabaseServiceTests : IDisposable
{
    private readonly DbContext _context;

    public DatabaseServiceTests()
    {
        // Constructor runs BEFORE each test -- per-test setup
        var options = new DbContextOptionsBuilder<MyDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;
        _context = new MyDbContext(options);
    }

    [Fact]
    public async Task GetAllUsers_ReturnsSeededUsers()
    {
        _context.Users.Add(new User { Name = "Alice" });
        await _context.SaveChangesAsync();

        var service = new UserService(_context);
        var users = await service.GetAllAsync();

        Assert.Single(users);
    }

    public void Dispose()
    {
        // Runs AFTER each test -- per-test cleanup
        _context.Dispose();
    }
}
```

- For async setup, implement `IAsyncLifetime`:

```csharp
public class AsyncSetupTests : IAsyncLifetime
{
    private HttpClient _client;

    public async Task InitializeAsync()
    {
        var factory = new WebApplicationFactory<Program>();
        _client = factory.CreateClient();
    }

    public async Task DisposeAsync()
    {
        _client?.Dispose();
    }
}
```

- Constructor = `[SetUp]` in NUnit terms
- `Dispose` / `DisposeAsync` = `[TearDown]` in NUnit terms
- Every test gets a fresh instance -- there is no shared state between tests in the same class (by default)

---

### Collection Fixtures

- Sometimes you need to share an expensive resource (like a database container or test server) across multiple test classes
- xUnit provides **collection fixtures** for this purpose:

```csharp
// Shared fixture class
public class TestDatabaseFixture : IDisposable
{
    public DbContext Context { get; }

    public TestDatabaseFixture()
    {
        var options = new DbContextOptionsBuilder<MyDbContext>()
            .UseInMemoryDatabase("SharedTestDb")
            .Options;
        Context = new MyDbContext(options);
        SeedData();
    }

    private void SeedData()
    {
        Context.Users.Add(new User { Name = "Alice" });
        Context.SaveChanges();
    }

    public void Dispose() => Context.Dispose();
}

// Collection definition -- links fixture to a collection name
[CollectionDefinition("Database")]
public class DatabaseCollection : ICollectionFixture<TestDatabaseFixture> { }

// Test class 1
[Collection("Database")]
public class UserServiceTests
{
    private readonly TestDatabaseFixture _fixture;

    public UserServiceTests(TestDatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public void GetAll_ReturnsSeededUsers()
    {
        var service = new UserService(_fixture.Context);
        var users = service.GetAll();
        Assert.NotEmpty(users);
    }
}

// Test class 2 -- same fixture, same database instance
[Collection("Database")]
public class UserRepositoryTests
{
    private readonly TestDatabaseFixture _fixture;

    public UserRepositoryTests(TestDatabaseFixture fixture)
    {
        _fixture = fixture;
    }

    [Fact]
    public void FindByName_ReturnsCorrectUser()
    {
        var repo = new UserRepository(_fixture.Context);
        var user = repo.FindByName("Alice");
        Assert.NotNull(user);
    }
}
```

- Classes in the same collection share the same fixture instance and run **sequentially** (not in parallel)
- Classes in different collections run in parallel with each other
- `ICollectionFixture<T>` is the collection version of `IClassFixture<T>`

---

### Skipping Tests

- You can skip a test with a reason so it appears as "skipped" in test results but does not cause a failure:

```csharp
[Fact(Skip = "Not implemented yet -- waiting for API v2")]
public void NewFeature_ReturnsExpectedResult()
{
    // ...
}

[Theory(Skip = "Known issue #427 -- flaky due to external service")]
[InlineData(1, 2)]
public void ExternalApi_CallsCorrectEndpoint(int a, int b)
{
    // ...
}
```

- Skipped tests show up in test reports as warnings, not failures
- Never commit skipped tests without a tracked issue or clear reason

---

### Assert Class Methods

- xUnit's `Assert` class provides a comprehensive set of assertion methods:

```csharp
// Equality
Assert.Equal(expected, actual);                  // value equality
Assert.NotEqual(expected, actual);               // not equal
Assert.Equal(expected, actual, 2);               // decimal tolerance (2 places)

// Boolean
Assert.True(condition);
Assert.False(condition);

// Null checks
Assert.Null(value);
Assert.NotNull(value);

// Collections
Assert.Empty(collection);
Assert.NotEmpty(collection);
Assert.Single(collection);
Assert.Contains("item", collection);
Assert.DoesNotContain("item", collection);
Assert.Equal(new[] { 1, 2, 3 }, collection);    // ordered comparison
Assert.All(collection, item => Assert.NotNull(item));

// String
Assert.Contains("substring", fullString);
Assert.StartsWith("prefix", fullString);
Assert.EndsWith("suffix", fullString);
Assert.Matches(@"\d+", fullString);
Assert.Empty(fullString);

// Type
Assert.IsType<ConcreteType>(actual);
Assert.IsAssignableFrom<BaseType>(actual);

// Exceptions
Assert.Throws<ExceptionType>(() => { /* code */ });
Assert.ThrowsAsync<ExceptionType>(async () => await SomeAsyncMethod());

// Collection count
Assert.Equal(3, collection.Count());
```

---

### Assert.Throws\<T\> and Assert.ThrowsAsync\<T\>

- Use `Assert.Throws<T>` for synchronous methods that throw exceptions:

```csharp
[Fact]
public void Divide_ByZero_ThrowsDivideByZeroException()
{
    var calc = new Calculator();

    var exception = Assert.Throws<DivideByZeroException>(() => calc.Divide(10, 0));

    Assert.NotNull(exception);
    // You can also inspect the exception message
    Assert.Contains("division by zero", exception.Message);
}
```

- Use `Assert.ThrowsAsync<T>` for async methods that throw exceptions:

```csharp
[Fact]
public async Task GetUserAsync_InvalidId_ThrowsArgumentException()
{
    var service = new UserService(null!);

    var exception = await Assert.ThrowsAsync<ArgumentException>(
        async () => await service.GetUserAsync(-1));

    Assert.Contains("Invalid user ID", exception.Message);
}
```

- You can chain: `var ex = Assert.Throws<T>(() => ...)` to inspect exception properties
- For methods that return `Task`, always use `Assert.ThrowsAsync<T>` -- never wrap in `Result` or use `.GetAwaiter().GetResult()` in a `[Fact]`


---

## NUnit

### What Is NUnit

- NUnit is one of the oldest and most mature .NET testing frameworks, originally ported from JUnit
- It has been widely used in the .NET community since the early 2000s and remains popular in enterprise environments
- NUnit uses explicit attributes to mark tests, fixtures, and lifecycle methods
- NUnit runs tests sequentially within a fixture by default (unlike xUnit's parallel execution)

---

### [Test] vs [TestCase] Attributes

- **`[Test]`** -- marks a simple test method (equivalent to xUnit's `[Fact]`):

```csharp
[TestFixture]
public class CalculatorTests
{
    [Test]
    public void Add_TwoPositiveNumbers_ReturnsSum()
    {
        var calc = new Calculator();
        Assert.That(calc.Add(2, 3), Is.EqualTo(5));
    }
}
```

- **`[TestCase]`** -- marks a parameterized test (equivalent to xUnit's `[Theory]` with `[InlineData]`):

```csharp
[TestFixture]
public class CalculatorTests
{
    [TestCase(1, 1, 2)]
    [TestCase(0, 0, 0)]
    [TestCase(-1, 1, 0)]
    [TestCase(100, 200, 300)]
    public void Add_VariousInputs_ReturnsCorrectSum(int a, int b, int expected)
    {
        var calc = new Calculator();
        Assert.That(calc.Add(a, b), Is.EqualTo(expected));
    }
}
```

- `[TestCase]` also supports named parameters: `[TestCase(1, 2, ExpectedResult = 3, Description = "adds two numbers")]`

---

### [SetUp] and [TearDown]

- **`[SetUp]`** -- runs before each test in the fixture (similar to xUnit constructor):

```csharp
[TestFixture]
public class UserServiceTests
{
    private UserService _service;
    private MyDbContext _context;

    [SetUp]
    public void SetUp()
    {
        var options = new DbContextOptionsBuilder<MyDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;
        _context = new MyDbContext(options);
        _service = new UserService(_context);
    }

    [TearDown]
    public void TearDown()
    {
        _context?.Dispose();
    }

    [Test]
    public void GetAll_ReturnsSeededUsers()
    {
        _context.Users.Add(new User { Name = "Alice" });
        _context.SaveChanges();

        var result = _service.GetAll();

        Assert.That(result, Is.Not.Empty);
    }
}
```

- **`[TearDown]`** -- runs after each test (similar to xUnit `IDisposable.Dispose`)
- NUnit shares the fixture instance across all `[Test]` methods in that class -- unlike xUnit which creates a new instance per test

---

### [OneTimeSetUp] and [OneTimeTearDown]

- **`[OneTimeSetUp]`** -- runs once before all tests in the fixture (like a static initializer):

```csharp
[TestFixture]
public class DatabaseIntegrationTests
{
    private static TestServer _server;
    private static HttpClient _client;

    [OneTimeSetUp]
    public void OneTimeSetUp()
    {
        _server = new WebApplicationFactory<Program>()
            .WithWebHostBuilder(builder =>
            {
                builder.ConfigureServices(services =>
                {
                    services.AddDbContext<MyDbContext>(options =>
                        options.UseInMemoryDatabase("IntegrationTestDb"));
                });
            });
        _client = _server.CreateClient();
    }

    [OneTimeTearDown]
    public void OneTimeTearDown()
    {
        _client?.Dispose();
        _server?.Dispose();
    }

    [Test]
    public async Task Get_Health_ReturnsOk()
    {
        var response = await _client.GetAsync("/health");
        Assert.That(response.StatusCode, Is.EqualTo(HttpStatusCode.OK));
    }
}
```

- **`[OneTimeTearDown]`** -- runs once after all tests in the fixture complete
- Useful for expensive one-time setup like spinning up a test database or test server

---

### [TestFixture] Attribute

- In NUnit, the `[TestFixture]` attribute explicitly marks a class as containing tests
- It is optional in modern NUnit (classes with `[Test]` methods are automatically treated as fixtures), but it is good practice to include it:

```csharp
[TestFixture]
public class PaymentServiceTests
{
    [Test]
    public void ProcessPayment_ValidCard_ReturnsSuccess()
    {
        // ...
    }
}
```

- The attribute also supports constructor parameters for data-driven fixtures:

```csharp
[TestFixture(100)]
[TestFixture(500)]
[TestFixture(1000)]
public class ThresholdTests
{
    private readonly int _threshold;

    public ThresholdTests(int threshold)
    {
        _threshold = threshold;
    }

    [Test]
    public void ValueAboveThreshold_ThrowsException()
    {
        Assert.Throws<ArgumentOutOfRangeException>(() => Validate(_threshold + 1));
    }
}
```

---

### [Category] and [Explicit] Attributes

- **`[Category]`** -- groups tests into categories for filtering:

```csharp
[Test]
[Category("Unit")]
public void Add_ReturnsSum()
{
    // ...
}

[Test]
[Category("Integration")]
[Category("Slow")]
public async Task DatabaseQuery_ReturnsUsers()
{
    // ...
}
```

- Run only a category: `dotnet test --filter "Category=Unit"`

- **`[Explicit]`** -- marks a test that should only run when explicitly selected (never runs as part of a full suite):

```csharp
[Test]
[Explicit("Requires production database credentials")]
public void ProductionQuery_ReturnsCorrectData()
{
    // ...
}
```

- `[Explicit]` is similar to xUnit's `Skip` but with the intention that it **can** be run on demand

---

### Constraint Model: Assert.That

- NUnit's modern assertion style uses the **constraint model** with `Assert.That`:

```csharp
// Equality
Assert.That(result, Is.EqualTo(42));
Assert.That(result, Is.Not.EqualTo(0));

// Comparisons
Assert.That(value, Is.GreaterThan(10));
Assert.That(value, Is.LessThanOrEqualTo(100));
Assert.That(value, Is.InRange(1, 100));

// Collections
Assert.That(collection, Is.Not.Empty);
Assert.That(collection, Has.Count.EqualTo(3));
Assert.That(collection, Contains.Item("hello"));
Assert.That(collection, Does.Contain("hello"));
Assert.That(collection, Does.Not.Contain("world"));

// Strings
Assert.That(name, Does.StartWith("John"));
Assert.That(name, Does.EndWith("son"));
Assert.That(name, Does.Contain("oh"));
Assert.That(name, Does.Match(@"^J\w+$"));

// Types
Assert.That(obj, Is.InstanceOf<string>());
Assert.That(obj, Is.TypeOf<string>());

// Exceptions
Assert.That(() => calc.Divide(10, 0), Throws.TypeOf<DivideByZeroException>());
Assert.That(() => calc.Divide(10, 0), Throws.InstanceOf<ArgumentException>());
Assert.That(() => calc.Divide(10, 0), Throws.TypeOf<DivideByZeroException>()
    .With.Message.Contains("division by zero"));

// Property constraints
Assert.That(user, Has.Property("Name").EqualTo("Alice"));
Assert.That(user, Has.Property("Age").GreaterThan(18));

// Logical
Assert.That(value, Is.GreaterThan(0).And.LessThan(100));
Assert.That(value, Is.EqualTo(1).Or.EqualTo(2).Or.EqualTo(3));
```

- The constraint model is more expressive and readable than the classic `Assert.AreEqual` style
- NUnit also supports the classic style (`Assert.AreEqual`, `Assert.IsTrue`, etc.) but the constraint model is preferred

---

### NUnit vs xUnit Key Differences

| Feature | NUnit | xUnit |
|---|---|---|
| Test attribute | `[Test]` | `[Fact]` / `[Theory]` |
| Fixture attribute | `[TestFixture]` (optional) | None (auto-detected) |
| Per-test setup | `[SetUp]` | Constructor |
| Per-test teardown | `[TearDown]` | `IDisposable.Dispose` |
| Suite-level setup | `[OneTimeSetUp]` | `IClassFixture<T>` / `ICollectionFixture<T>` |
| Parameterized tests | `[TestCase]` | `[Theory]` + `[InlineData]` |
| Fixture instance sharing | Same instance across tests | New instance per test (by default) |
| Parallel execution | Sequential by default | Parallel by default |
| Assertion model | `Assert.That` (constraint) | `Assert.Equal`, etc. (classic) |
| Community momentum | Stable, legacy projects | Active, default for new .NET |
| Origin | Ported from JUnit | Created by NUnit authors |


---

## Mocking with Moq

### What Is Mocking

- Mocking is the practice of creating **test doubles** -- fake implementations of dependencies that your code relies on
- A mock object records how it was called during the test so you can **verify** that your code called it correctly
- Mocks let you test your business logic **without** involving real databases, APIs, file systems, or other external dependencies

```csharp
// Real dependency
public interface IEmailService
{
    void Send(string to, string subject, string body);
}

// Mock in test
var mockEmailService = new Mock<IEmailService>();
```

---

### Why Mocking Matters

- **Test business logic, not infrastructure** -- your order processing logic should not fail because the email server is down
- **Speed** -- mocks return instantly, while real database calls or HTTP requests can take seconds or minutes
- **Determinism** -- mocks return exactly what you tell them to, eliminating non-deterministic behavior
- **Isolation** -- mocks let you test a single class without spinning up its entire dependency graph
- **Edge case simulation** -- you can make a mock throw an exception to test error handling paths that are hard to reproduce with real dependencies

---

### Moq Setup

- Install Moq: `dotnet add package Moq`
- Create a mock with `new Mock<IService>()`:

```csharp
public interface IUserRepository
{
    User GetById(int id);
    IEnumerable<User> GetAll();
    void Add(User user);
}

public class UserService
{
    private readonly IUserRepository _repository;

    public UserService(IUserRepository repository)
    {
        _repository = repository;
    }

    public User GetUser(int id)
    {
        var user = _repository.GetById(id);
        if (user == null)
            throw new NotFoundException($"User {id} not found");
        return user;
    }
}
```

```csharp
[Fact]
public void GetUser_ExistingId_ReturnsUser()
{
    // Arrange
    var mockRepo = new Mock<IUserRepository>();
    mockRepo.Setup(r => r.GetById(1))
        .Returns(new User { Id = 1, Name = "Alice" });

    var service = new UserService(mockRepo.Object);

    // Act
    var result = service.GetUser(1);

    // Assert
    Assert.Equal("Alice", result.Name);
}
```

---

### Setup/Returns for Methods

- `Setup` configures how a mock responds to method calls:
- `Returns` specifies the return value:

```csharp
var mockRepo = new Mock<IUserRepository>();

// Return a specific value for a specific argument
mockRepo.Setup(r => r.GetById(1))
    .Returns(new User { Id = 1, Name = "Alice" });

// Return different values for different arguments
mockRepo.Setup(r => r.GetById(It.IsAny<int>()))
    .Returns((int id) => new User { Id = id, Name = $"User{id}" });

// Return a sequence of values (call 1 returns first, call 2 returns second, etc.)
mockRepo.Setup(r => r.GetAll())
    .Returns(new[]
    {
        new User { Id = 1, Name = "Alice" },
        new User { Id = 2, Name = "Bob" }
    });

// ReturnsAsync for async methods
mockRepo.Setup(r => r.GetByIdAsync(1))
    .ReturnsAsync(new User { Id = 1, Name = "Alice" });
```

---

### SetupGet for Properties

- Use `SetupGet` (or the property syntax) to configure property behavior:

```csharp
var mockService = new Mock<IConfiguration>();

// SetupGet syntax
mockService.SetupGet(s => s.ConnectionString)
    .Returns("Server=localhost;Database=TestDb");

// Property syntax (preferred in modern Moq)
mockService.Setup(s => s.ConnectionString)
    .Returns("Server=localhost;Database=TestDb");

// Lazily computed property
var callCount = 0;
mockService.Setup(s => s.RequestCount)
    .Returns(() => ++callCount);  // increments each time accessed
```

---

### SetupSet for Property Setters

- Use `SetupSet` to verify or configure when a property is assigned:

```csharp
var mockLogger = new Mock<ILogger>();

// Verify that a property was set to a specific value
mockLogger.SetupSet(l => l.LogLevel = LogLevel.Error);

// Later, verify it was called
mockLogger.VerifySet(l => l.LogLevel = LogLevel.Error, Times.Once());
```

---

### Verify: Assert a Method Was Called

- `Verify` asserts that a method on the mock was called with specific arguments:

```csharp
[Fact]
public void DeleteUser_ExistingId_CallsRepositoryDelete()
{
    // Arrange
    var mockRepo = new Mock<IUserRepository>();
    var service = new UserService(mockRepo.Object);

    // Act
    service.DeleteUser(1);

    // Assert -- verify the repository's Delete method was called
    mockRepo.Verify(r => r.Delete(1), Times.Once());
}

[Fact]
public void DeleteUser_NonExistingId_DoesNotCallRepositoryDelete()
{
    var mockRepo = new Mock<IUserRepository>();
    mockRepo.Setup(r => r.GetById(It.IsAny<int>())).Returns((User)null);

    var service = new UserService(mockRepo.Object);

    service.DeleteUser(999);

    // Verify Delete was never called
    mockRepo.Verify(r => r.Delete(It.IsAny<int>()), Times.Never());
}
```

---

### VerifyTimes: Assert Exact Call Count

- `Times` options control how many times a method should have been called:

```csharp
mockRepo.Verify(r => r.GetById(1), Times.Once());       // exactly once
mockRepo.Verify(r => r.GetById(1), Times.Never());      // never called
mockRepo.Verify(r => r.GetById(1), Times.Exactly(3));   // exactly 3 times
mockRepo.Verify(r => r.GetById(1), Times.AtLeast(1));   // at least once
mockRepo.Verify(r => r.GetById(1), Times.AtMost(5));    // at most 5 times
mockRepo.Verify(r => r.GetById(1), Times.AtLeastOnce()); // at least once (shorthand)
```

---

### Callback: Side Effects During Test

- `Callback` lets you execute custom logic when a mock method is called:

```csharp
var mockRepo = new Mock<IUserRepository>();
var savedUsers = new List<User>();

mockRepo.Setup(r => r.Add(It.IsAny<User>()))
    .Callback<User>(user => savedUsers.Add(user));  // capture the argument

var service = new UserService(mockRepo.Object);

service.CreateUser("Alice");
service.CreateUser("Bob");

Assert.Equal(2, savedUsers.Count);
Assert.Equal("Alice", savedUsers[0].Name);
Assert.Equal("Bob", savedUsers[1].Name);
```

- Callback is useful for capturing arguments, tracking call order, or simulating side effects

---

### Strict vs Loose Mocks

- **Loose mock** (default) -- returns default values for methods that are not set up, and does not fail on unexpected calls:

```csharp
var looseMock = new Mock<IUserRepository>();  // loose by default
var user = looseMock.Object.GetById(999);  // returns null, no error
```

- **Strict mock** -- throws an exception for any call that was not explicitly set up:

```csharp
var strictMock = new Mock<IUserRepository>(MockBehavior.Strict);
var user = strictMock.Object.GetById(999);  // throws MockException!
```

- Use strict mocks when you want to ensure your code only calls the dependencies you expect
- Use loose mocks for convenience when unexpected calls are acceptable
- Most teams default to loose mocks and switch to strict only for specific scenarios

---

### Moq Matchers

- Moq provides matchers to create flexible argument expectations:

```csharp
// It.IsAny<T> -- matches any value of type T
mock.Setup(r => r.GetById(It.IsAny<int>()))
    .Returns(new User { Id = 1 });

// It.Is<T> -- matches values satisfying a predicate
mock.Setup(r => r.GetById(It.Is<int>(id => id > 0)))
    .Returns(new User { Id = 1 });

// It.IsInRange<T> -- matches values within a range
mock.Setup(r => r.GetById(It.IsInRange(1, 100, Range.Inclusive)))
    .Returns(new User { Id = 1 });

// It.IsIn<T> -- matches if the value is in a set
mock.Setup(r => r.GetById(It.IsIn(1, 2, 3)))
    .Returns(new User { Id = 1 });

// It.IsNotIn<T> -- matches if the value is NOT in a set
mock.Setup(r => r.GetById(It.IsNotIn(0, -1)))
    .Returns(new User { Id = 1 });

// It.IsRegex -- matches strings against a regex pattern
mock.Setup(r => r.GetByEmail(It.IsRegex(@"^.*@example\.com$")))
    .Returns(new User { Email = "test@example.com" });
```

---

### Full Moq Example

```csharp
public interface IOrderRepository
{
    Order GetById(int id);
    void Save(Order order);
}

public interface IPaymentGateway
{
    PaymentResult Charge(decimal amount, string cardToken);
}

public interface IEmailService
{
    void SendConfirmation(string email, int orderId);
}

public class OrderService
{
    private readonly IOrderRepository _orderRepo;
    private readonly IPaymentGateway _paymentGateway;
    private readonly IEmailService _emailService;

    public OrderService(
        IOrderRepository orderRepo,
        IPaymentGateway paymentGateway,
        IEmailService emailService)
    {
        _orderRepo = orderRepo;
        _paymentGateway = paymentGateway;
        _emailService = emailService;
    }

    public OrderResult PlaceOrder(int productId, decimal price, string cardToken, string email)
    {
        var paymentResult = _paymentGateway.Charge(price, cardToken);
        if (!paymentResult.Success)
            return OrderResult.PaymentFailed;

        var order = new Order { ProductId = productId, Price = price, Email = email };
        _orderRepo.Save(order);

        _emailService.SendConfirmation(email, order.Id);

        return OrderResult.Success;
    }
}
```

```csharp
[Fact]
public void PlaceOrder_PaymentSucceeds_SavesOrderAndSendsEmail()
{
    // Arrange
    var mockOrderRepo = new Mock<IOrderRepository>();
    var mockPayment = new Mock<IPaymentGateway>();
    var mockEmail = new Mock<IEmailService>();

    mockPayment.Setup(p => p.Charge(99.99m, "tok_visa"))
        .Returns(new PaymentResult { Success = true });

    var service = new OrderService(mockOrderRepo.Object, mockPayment.Object, mockEmail.Object);

    // Act
    var result = service.PlaceOrder(1, 99.99m, "tok_visa", "alice@example.com");

    // Assert
    Assert.Equal(OrderResult.Success, result);
    mockOrderRepo.Verify(r => r.Save(It.IsAny<Order>()), Times.Once());
    mockEmail.Verify(e => e.SendConfirmation("alice@example.com", It.IsAny<int>()), Times.Once());
}

[Fact]
public void PlaceOrder_PaymentFails_DoesNotSaveOrder()
{
    // Arrange
    var mockOrderRepo = new Mock<IOrderRepository>();
    var mockPayment = new Mock<IPaymentGateway>();
    var mockEmail = new Mock<IEmailService>();

    mockPayment.Setup(p => p.Charge(It.IsAny<decimal>(), It.IsAny<string>()))
        .Returns(new PaymentResult { Success = false });

    var service = new OrderService(mockOrderRepo.Object, mockPayment.Object, mockEmail.Object);

    // Act
    var result = service.PlaceOrder(1, 99.99m, "tok_visa", "alice@example.com");

    // Assert
    Assert.Equal(OrderResult.PaymentFailed, result);
    mockOrderRepo.Verify(r => r.Save(It.IsAny<Order>()), Times.Never());
    mockEmail.Verify(e => e.SendConfirmation(It.IsAny<string>(), It.IsAny<int>()), Times.Never());
}
```


---

## Moq with Async Methods

### SetupAsync and ReturnsAsync

- Use `ReturnsAsync` to configure mock behavior for async methods:

```csharp
public interface IUserRepository
{
    Task<User> GetByIdAsync(int id);
    Task<IEnumerable<User>> GetAllAsync();
    Task AddAsync(User user);
}
```

```csharp
[Fact]
public async Task GetUserAsync_ExistingId_ReturnsUser()
{
    // Arrange
    var mockRepo = new Mock<IUserRepository>();
    mockRepo.Setup(r => r.GetByIdAsync(1))
        .ReturnsAsync(new User { Id = 1, Name = "Alice" });

    var service = new UserService(mockRepo.Object);

    // Act
    var result = await service.GetUserAsync(1);

    // Assert
    Assert.Equal("Alice", result.Name);
}
```

- You can also use `Returns(Task.FromResult(...))` which is equivalent:

```csharp
mockRepo.Setup(r => r.GetByIdAsync(1))
    .Returns(Task.FromResult(new User { Id = 1, Name = "Alice" }));
```

---

### Callback with Async Methods

- Use `Callback` before `ReturnsAsync` to intercept arguments in async setups:

```csharp
var savedUsers = new List<User>();

mockRepo.Setup(r => r.AddAsync(It.IsAny<User>()))
    .Callback<User>(user => savedUsers.Add(user))
    .Returns(Task.CompletedTask);

await service.CreateUserAsync("Alice");

Assert.Single(savedUsers);
Assert.Equal("Alice", savedUsers[0].Name);
```

---

### Verify Async Methods

- Verify async method calls the same way as synchronous:

```csharp
mockRepo.Verify(r => r.AddAsync(It.IsAny<User>()), Times.Once());
mockRepo.Verify(r => r.GetByIdAsync(It.IsAny<int>()), Times.Never());
```

---

### Testing async void Methods

- `async void` methods cannot be awaited directly -- use `Task.Run` or the `FluentAssertions` equivalent to observe completion:

```csharp
// Production code (avoid async void, but sometimes unavoidable in event handlers)
public class OrderPlacedHandler
{
    private readonly IEmailService _emailService;

    public OrderPlacedHandler(IEmailService emailService)
    {
        _emailService = emailService;
    }

    public async void Handle(OrderPlacedEvent evt)
    {
        await _emailService.SendConfirmationAsync(evt.UserEmail, evt.OrderId);
    }
}
```

```csharp
[Fact]
public async Task Handle_OrderPlaced_SendsEmail()
{
    // Arrange
    var mockEmail = new Mock<IEmailService>();
    var handler = new OrderPlacedHandler(mockEmail.Object);

    // Act -- wrap async void in Task.Run to observe completion
    await Task.Run(() => handler.Handle(new OrderPlacedEvent
    {
        UserEmail = "alice@example.com",
        OrderId = 42
    }));

    // Small delay to allow async void to complete
    await Task.Delay(100);

    // Assert
    mockEmail.Verify(
        e => e.SendConfirmationAsync("alice@example.com", 42),
        Times.Once());
}
```

- In production code, prefer `async Task` over `async void` wherever possible -- `async void` is only appropriate for event handlers and fire-and-forget scenarios

---

## Integration Testing

### What Is Integration Testing

- Integration testing verifies that **multiple components work together correctly** -- unlike unit tests which isolate a single class
- Integration tests exercise real dependencies: databases, file systems, HTTP clients, message queues, external APIs
- They sit in the middle of the testing pyramid: fewer than unit tests, more than end-to-end tests
- Integration tests catch issues that unit tests cannot: incorrect SQL queries, serialization mismatches, middleware misconfiguration, dependency wiring errors

---

### TestServer in ASP.NET Core (WebApplicationFactory\<T\>)

- `WebApplicationFactory<TStartup>` creates an in-memory test server that hosts your ASP.NET Core app without binding to a real network port
- It replaces real services with test doubles or in-memory alternatives

```csharp
public class BasicIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public BasicIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task Get_Home_ReturnsOk()
    {
        var response = await _client.GetAsync("/");

        response.EnsureSuccessStatusCode();
        var content = await response.Content.ReadAsStringAsync();
        Assert.Contains("Welcome", content);
    }
}
```

---

### CustomWebApplicationFactory\<T\> Pattern

- For more control over the test environment, create a custom factory:

```csharp
public class CustomWebApplicationFactory<TProgram> : WebApplicationFactory<TProgram>
    where TProgram : class
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Remove the real database registration
            var descriptor = services.SingleOrDefault(
                d => d.ServiceType == typeof(DbContextOptions<MyDbContext>));
            if (descriptor != null)
                services.Remove(descriptor);

            // Add in-memory database
            services.AddDbContext<MyDbContext>(options =>
                options.UseInMemoryDatabase("IntegrationTestDb"));

            // Replace external services with mocks
            var mockEmailService = new Mock<IEmailService>();
            services.AddSingleton(mockEmailService.Object);
        });

        builder.UseEnvironment("Testing");
    }
}
```

```csharp
public class UserApiTests : IClassFixture<CustomWebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public UserApiTests(CustomWebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetUsers_ReturnsOk()
    {
        var response = await _client.GetAsync("/api/users");

        response.EnsureSuccessStatusCode();
        var users = await response.Content.ReadFromJsonAsync<List<UserDto>>();
        Assert.NotNull(users);
    }
}
```

---

### In-Memory Database with EF Core

- EF Core's `UseInMemoryDatabase` provider lets you test data access without a real database server:

```csharp
public class UserRepositoryTests : IDisposable
{
    private readonly MyDbContext _context;
    private readonly UserRepository _repository;

    public UserRepositoryTests()
    {
        var options = new DbContextOptionsBuilder<MyDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())  // unique per test
            .Options;
        _context = new MyDbContext(options);
        _repository = new UserRepository(_context);
    }

    [Fact]
    public async Task AddAsync_SavesUser()
    {
        var user = new User { Name = "Alice", Email = "alice@example.com" };

        await _repository.AddAsync(user);

        var result = await _context.Users.FindAsync(user.Id);
        Assert.NotNull(result);
        Assert.Equal("Alice", result.Name);
    }

    [Fact]
    public async Task GetByIdAsync_ReturnsUser()
    {
        var user = new User { Name = "Bob", Email = "bob@example.com" };
        _context.Users.Add(user);
        await _context.SaveChangesAsync();

        var result = await _repository.GetByIdAsync(user.Id);

        Assert.Equal("Bob", result.Name);
    }

    public void Dispose() => _context.Dispose();
}
```

- Always use `Guid.NewGuid().ToString()` as the database name to ensure test isolation
- The in-memory provider does not support transactions or relational features -- use `SQLite` in-memory mode if you need those

---

### HttpClient with TestServer

- Use `CreateClient()` on `WebApplicationFactory` to get an `HttpClient` that targets the in-memory server:

```csharp
[Fact]
public async Task Post_ValidOrder_ReturnsCreatedAtAction()
{
    // Arrange
    var order = new CreateOrderRequest
    {
        ProductId = 1,
        Quantity = 2,
        CardToken = "tok_visa"
    };

    // Act
    var response = await _client.PostAsJsonAsync("/api/orders", order);

    // Assert
    Assert.Equal(HttpStatusCode.Created, response.StatusCode);
    var created = await response.Content.ReadFromJsonAsync<OrderDto>();
    Assert.NotNull(created);
    Assert.Equal(1, created.ProductId);
}

[Fact]
public async Task Get_NonExistentOrder_ReturnsNotFound()
{
    var response = await _client.GetAsync("/api/orders/99999");

    Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
}
```

---

### Testing Middleware Pipeline

- Integration tests can verify middleware behavior end-to-end:

```csharp
[Fact]
public async Task Request_WithoutAuthHeader_ReturnsUnauthorized()
{
    // No Authorization header set
    var response = await _client.GetAsync("/api/protected/resource");

    Assert.Equal(HttpStatusCode.Unauthorized, response.StatusCode);
}

[Fact]
public async Task Request_WithValidToken_ReturnsOk()
{
    _client.DefaultRequestHeaders.Authorization =
        new AuthenticationHeaderValue("Bearer", "valid-test-token");

    var response = await _client.GetAsync("/api/protected/resource");

    Assert.Equal(HttpStatusCode.OK, response.StatusCode);
}
```

---

### Testing Authentication/Authorization in Integration Tests

- Configure the test server to use a test authentication scheme:

```csharp
protected override void ConfigureWebHost(IWebHostBuilder builder)
{
    builder.ConfigureServices(services =>
    {
        services.AddAuthentication("Test")
            .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>(
                "Test", options => { });

        services.AddAuthorization();
    });
}
```

```csharp
public class TestAuthHandler : AuthenticationHandler<AuthenticationSchemeOptions>
{
    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var claims = new[]
        {
            new Claim(ClaimTypes.Name, "TestUser"),
            new Claim(ClaimTypes.Role, "Admin")
        };

        var identity = new ClaimsIdentity(claims, "Test");
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, "Test");

        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

---

### Seeding Test Data

- Seed data in your custom factory or test setup to create consistent test environments:

```csharp
protected override void ConfigureWebHost(IWebHostBuilder builder)
{
    builder.ConfigureServices(services =>
    {
        services.AddDbContext<MyDbContext>(options =>
            options.UseInMemoryDatabase("SeededTestDb"));

        var sp = services.BuildServiceProvider();
        using var scope = sp.CreateScope();
        var context = scope.ServiceProvider.GetRequiredService<MyDbContext>();
        context.Database.EnsureCreated();

        // Seed data
        context.Users.AddRange(
            new User { Id = 1, Name = "Alice", Email = "alice@test.com" },
            new User { Id = 2, Name = "Bob", Email = "bob@test.com" },
            new User { Id = 3, Name = "Charlie", Email = "charlie@test.com" }
        );
        context.SaveChanges();
    });
}
```

```csharp
[Fact]
public async Task GetAllUsers_ReturnsSeededUsers()
{
    var response = await _client.GetAsync("/api/users");

    response.EnsureSuccessStatusCode();
    var users = await response.Content.ReadFromJsonAsync<List<UserDto>>();

    Assert.Equal(3, users.Count);
    Assert.Contains(users, u => u.Name == "Alice");
}
```

---

## Test Organization

### Test Project Structure

- Mirror the source project structure in your test project:

```
src/
  MyProject/
    Services/
      UserService.cs
      OrderService.cs
    Repositories/
      UserRepository.cs
    Models/
      User.cs

tests/
  MyProject.Tests/
    Services/
      UserServiceTests.cs
      OrderServiceTests.cs
    Repositories/
      UserRepositoryTests.cs
```

- This makes it easy to find the test for any given class
- Some teams also use a `UnitTests` / `IntegrationTests` project split:

```
tests/
  MyProject.UnitTests/       -- fast, isolated, mock-heavy
  MyProject.IntegrationTests/ -- real DB, real HTTP, WebApplicationFactory
```

---

### Test Naming Conventions

- Consistent naming helps when filtering, searching, and reviewing test output
- Common patterns:

```csharp
// Method_Scenario_ExpectedResult
[Fact]
public void GetUser_InvalidId_ThrowsNotFoundException() { }

// Should_ExpectedBehavior_When_Condition
[Fact]
public void Should_ReturnEmptyList_When_NoUsersExist() { }

// BDD style
[Fact]
public void Given_InvalidEmail_When_CreatingUser_Then_ThrowsValidationException() { }
```

- Pick one convention and stick to it across the entire team

---

### Shared Test Fixtures

- xUnit: `IClassFixture<T>` for per-class sharing, `ICollectionFixture<T>` for cross-class sharing
- NUnit: `[OneTimeSetUp]` for per-fixture sharing, `[SetUp]` / `[TearDown]` for per-test sharing
- Be careful with shared state -- mutable shared fixtures can cause test interdependence

---

### Test Categories / Trait Filtering

- xUnit uses `[Trait]` for categorization:

```csharp
[Trait("Category", "Unit")]
[Trait("Priority", "High")]
[Fact]
public void CriticalPath_ReturnsExpectedResult() { }
```

```bash
dotnet test --filter "Category=Unit"
dotnet test --filter "Priority=High"
dotnet test --filter "FullyQualifiedName~UserService"
```

- NUnit uses `[Category]`:

```csharp
[Test]
[Category("Slow")]
[Category("Database")]
public async Task LargeQuery_ReturnsAllResults() { }
```

```bash
dotnet test --filter "Category!=Slow"
```

---

### Test Execution Order

- xUnit randomizes test execution order **by design** -- this forces you to write isolated tests
- NUnit runs tests in declaration order within a fixture by default
- Never rely on test execution order -- each test must set up its own state
- If tests pass in isolation but fail when run together, they share mutable state (a common bug)


---

## Common Mistakes

### Testing Implementation Details Instead of Behavior

- **Bad:** asserting that a specific internal method was called or a private field has a certain value
- **Good:** asserting that the public output or side effect matches expectations
- If you change the implementation but the behavior is the same, tests should still pass
- Tests that break when you refactor (but behavior is unchanged) are testing implementation details

```csharp
// BAD -- tests implementation detail (internal cache)
[Fact]
public void GetUser_CachesResult()
{
    var service = new UserService();
    service.GetUser(1);
    Assert.True(service.Cache.ContainsKey(1));  // fragile!
}

// GOOD -- tests behavior (returns same user for same ID)
[Fact]
public void GetUser_CalledTwiceWithSameId_ReturnsSameUser()
{
    var service = new UserService();
    var first = service.GetUser(1);
    var second = service.GetUser(1);
    Assert.Equal(first.Id, second.Id);
    Assert.Equal(first.Name, second.Name);
}
```

---

### Too Many Assertions Per Test

- Each test should verify **one behavior** -- multiple assertions are acceptable if they verify the same logical outcome
- If a test has 10 unrelated assertions, split it into 10 tests
- Each assertion in a test should fail for the **same reason**

```csharp
// BAD -- tests too many unrelated things
[Fact]
public void CreateOrder_EverythingWorks()
{
    var order = service.CreateOrder(product, user, card);
    Assert.NotNull(order);
    Assert.Equal(1, order.Items.Count);
    Assert.Equal(99.99m, order.Total);
    Assert.Equal("Shipped", order.Status);
    Assert.Equal(3, user.OrderHistory.Count);
    Assert.Equal(1, mockEmail.Invocations.Count);
}

// GOOD -- each test verifies one thing
[Fact]
public void CreateOrder_ValidInput_ReturnsNonNullOrder() { }

[Fact]
public void CreateOrder_SingleProduct_HasOneItem() { }

[Fact]
public void CreateOrder_ValidProduct_CalculatesCorrectTotal() { }

[Fact]
public void CreateOrder_NewOrder_HasShippedStatus() { }
```

---

### Not Cleaning Up Test Data

- Tests that create data in a database must clean up after themselves
- Use `Guid.NewGuid().ToString()` for database names in `UseInMemoryDatabase` to guarantee isolation
- For shared databases, use transactions that roll back after each test:

```csharp
public class DatabaseTests : IAsyncLifetime
{
    private readonly MyDbContext _context;

    public DatabaseTests()
    {
        var options = new DbContextOptionsBuilder<MyDbContext>()
            .UseInMemoryDatabase(databaseName: Guid.NewGuid().ToString())
            .Options;
        _context = new MyDbContext(options);
    }

    public Task InitializeAsync() => Task.CompletedTask;

    public async Task DisposeAsync()
    {
        await _context.Database.EnsureDeletedAsync();
        await _context.DisposeAsync();
    }
}
```

---

### Tests That Depend on Other Tests

- Tests must be completely independent -- each test must pass on its own
- Never create state in one test and rely on it in the next test
- xUnit randomizes execution order, so dependent tests will break unpredictably

```csharp
// BAD -- test B depends on test A having run first
[Fact]
public void TestA_CreatesUser()
{
    _context.Users.Add(new User { Name = "Alice" });
    _context.SaveChanges();
}

[Fact]
public void TestB_FindsUser()
{
    var user = _context.Users.First();  // assumes Alice exists from TestA
    Assert.Equal("Alice", user.Name);
}
```

---

### Mocking Too Much (Over-Mocking)

- When you mock everything, you are testing that your test setup works, not that your code works
- If a class requires 5+ mocks to test, it probably has too many dependencies -- consider refactoring
- Integration tests sometimes work better than heavy mocking for testing component interaction
- Only mock the **seams** -- external I/O, network calls, and infrastructure boundaries

---

### Not Testing Edge Cases and Error Paths

- Tests for the happy path are necessary but insufficient
- Always test:

```csharp
[Theory]
[InlineData(null)]
[InlineData("")]
[InlineData("   ")]
public void ValidateName_EmptyOrNull_ThrowsArgumentException(string name)
{
    Assert.Throws<ArgumentException>(() => validator.ValidateName(name));
}

[Fact]
public void GetUser_NonExistentId_ThrowsNotFoundException()
{
    mockRepo.Setup(r => r.GetByIdAsync(999)).ReturnsAsync((User)null);
    Assert.ThrowsAsync<NotFoundException>(() => service.GetUserAsync(999));
}

[Theory]
[InlineData(-1)]
[InlineData(0)]
[InlineData(int.MaxValue)]
public void ProcessPayment_InvalidAmount_ThrowsArgumentOutOfRangeException(decimal amount)
{
    Assert.Throws<ArgumentOutOfRangeException>(() => service.ProcessPayment(amount));
}
```

---

### Flaky Tests

- Flaky tests pass sometimes and fail sometimes, destroying confidence in the test suite
- Common causes and fixes:

| Cause | Fix |
|---|---|
| `Thread.Sleep` / timing | Use `Task.Delay` with polling, or `EventWaitHandle` |
| Shared mutable state | Use unique database names per test |
| External services | Mock all external dependencies |
| System clock | Inject `IClock` interface instead of `DateTime.Now` |
| Test ordering dependency | Make each test self-contained |
| Parallel test conflicts | Use test collections to serialize conflicting tests |

```csharp
// BAD -- flaky due to timing
[Fact]
public async Task BackgroundJob_ProcessesWithin10Seconds()
{
    var task = BackgroundJob.StartAsync();
    await Task.Delay(10000);
    Assert.True(task.IsCompleted);
}

// GOOD -- poll with timeout
[Fact]
public async Task BackgroundJob_CompletesSuccessfully()
{
    var task = BackgroundJob.StartAsync();
    var timeout = TimeSpan.FromSeconds(30);
    var stopwatch = Stopwatch.StartNew();

    while (!task.IsCompleted && stopwatch.Elapsed < timeout)
    {
        await Task.Delay(100);
    }

    Assert.True(task.IsCompleted);
}
```

---

### Testing Framework Code Instead of Application Code

- Do not write tests for things the framework guarantees -- ASP.NET Core routing, EF Core behavior, etc.
- Focus on testing **your business logic**, **your validation rules**, **your custom middleware**, and **your integration glue**
- If you are testing that `List.Add` adds an item, you are testing the framework, not your code

---

## Interview Questions

### Question 1: What is the AAA pattern and why is it important?

- **Answer:** AAA stands for Arrange, Act, Assert. Arrange sets up the test (objects, mocks, inputs), Act invokes the method under test, and Assert verifies the expected outcome. It provides a clear, consistent structure that makes tests readable and easy to maintain.

---

### Question 2: What is the difference between [Fact] and [Theory] in xUnit?

- **Answer:** `[Fact]` marks a simple test that runs once with no parameters. `[Theory]` marks a parameterized test that runs once per set of input data provided via `[InlineData]`, `[MemberData]`, or `[ClassData]`. Use `[Fact]` for one-off tests and `[Theory]` when you need to test the same logic with multiple inputs.

---

### Question 3: How does xUnit handle test class lifecycle compared to NUnit?

- **Answer:** xUnit creates a new instance of the test class for each `[Fact]` or `[Theory]`, with the constructor for setup and `IDisposable.Dispose` for teardown. NUnit shares a single fixture instance across all `[Test]` methods and uses `[SetUp]` / `[TearDown]` for per-test lifecycle and `[OneTimeSetUp]` / `[OneTimeTearDown]` for suite-level lifecycle.

---

### Question 4: Explain the purpose of mocking and when you should use it.

- **Answer:** Mocking creates fake implementations of dependencies so you can test business logic in isolation. You should mock external I/O (databases, file systems, HTTP clients), message queues, and infrastructure services. You should NOT mock value objects, simple data classes, or things you don't own. Mock the seams, not the internals.

---

### Question 5: What is the difference between MockBehavior.Strict and MockBehavior.Loose in Moq?

- **Answer:** A strict mock throws `MockException` for any method call that was not explicitly set up. A loose mock (default) returns default values for unconfigured calls and silently allows unexpected calls. Strict mocks catch unintended dependencies; loose mocks are more convenient for general use.

---

### Question 6: How do you verify that a method was called on a mock in Moq?

- **Answer:** Use `mock.Verify(expression, times)` where `expression` specifies the method and expected arguments, and `times` specifies the expected call count (e.g., `Times.Once()`, `Times.Never()`, `Times.AtLeastOnce()`). If the verification fails, Moq throws a `MockException` with details about what was expected vs. what actually happened.

---

### Question 7: What is WebApplicationFactory and how is it used for integration testing in ASP.NET Core?

- **Answer:** `WebApplicationFactory<TProgram>` creates an in-memory test server that hosts the ASP.NET Core app without binding to a real network port. It lets you send HTTP requests via `HttpClient` and test the full middleware pipeline, routing, model binding, and dependency injection. You can customize the test environment by overriding `ConfigureWebHost` to replace services with test doubles.

---

### Question 8: When would you choose integration tests over unit tests?

- **Answer:** Integration tests are appropriate when you need to verify that multiple components work together correctly -- e.g., testing that a repository correctly queries a database, that middleware pipeline configuration is correct, or that serialization/deserialization works end-to-end. Unit tests are better for testing isolated business logic where dependencies can be cleanly mocked.

---

### Question 9: What are the characteristics of a good unit test?

- **Answer:** Fast (runs in milliseconds), isolated (no dependencies on other tests), repeatable (same result every time), self-validating (clear pass/fail), timely (written alongside production code), and focused (verifies one behavior). A good test serves as documentation and provides confidence to refactor.

---

### Question 10: What is a collection fixture in xUnit and when would you use one?

- **Answer:** A collection fixture shares a single instance of a fixture across multiple test classes via `ICollectionFixture<T>` and `[Collection("name")]`. Use it when multiple test classes need to share an expensive resource like a test database or test server. Classes in the same collection run sequentially; classes in different collections run in parallel.

---

### Question 11: How do you test async methods that throw exceptions in xUnit?

- **Answer:** Use `Assert.ThrowsAsync<T>()` which accepts an async delegate and returns the exception for inspection:

```csharp
var ex = await Assert.ThrowsAsync<InvalidOperationException>(
    async () => await service.DoWorkAsync());
Assert.Contains("error message", ex.Message);
```

Never use `.Result` or `.GetAwaiter().GetResult()` in a `[Fact]` method as this can cause deadlocks.

---

### Question 12: What is the difference between mocking, stubbing, and faking?

- **Answer:** A **mock** records interactions and is verified after the test (e.g., "was `SendEmail` called?"). A **stub** provides canned answers to calls during the test (e.g., "return this user for any ID"). A **fake** is a working but lightweight implementation (e.g., `InMemoryDatabase`). Moq creates both mocks and stubs depending on whether you use `Verify` or `Setup/Returns`.

---

### Question 13: How do you handle flaky tests in a test suite?

- **Answer:** Identify the root cause (timing, shared state, external dependencies, system clock), then fix it: replace `Thread.Sleep` with polling, use unique database names per test, mock external services, inject `IClock` instead of `DateTime.Now`, and make each test self-contained. If a flaky test cannot be fixed immediately, mark it with `[Trait("Flaky", "true")]` and track it in a backlog.

---

### Question 14: When should you NOT mock something?

- **Answer:** Do not mock value objects, DTOs, simple data classes, or primitives. Do not mock classes you don't own (framework classes). Do not mock everything in sight -- if you mock all collaborators, you are verifying test setup, not application behavior. Consider using real implementations (like `InMemoryDatabase`) instead of mocks when testing component interaction.

---

### Question 15: How do [InlineData], [MemberData], and [ClassData] differ in xUnit?

- **Answer:** `[InlineData]` takes literal values as attribute arguments -- best for simple, static data. `[MemberData]` references a static property or method returning `IEnumerable<object[]>` -- best for computed or dynamic data. `[ClassData]` references a class implementing `IEnumerable<object[]>` -- best for complex test data that lives in its own class.

---

### Question 16: Explain how [SetUp] and [TearDown] work in NUnit and their xUnit equivalents.

- **Answer:** In NUnit, `[SetUp]` runs before each test and `[TearDown]` runs after each test within the same fixture instance. In xUnit, the constructor replaces `[SetUp]` and `IDisposable.Dispose()` (or `IAsyncLifetime.DisposeAsync()`) replaces `[TearDown]`. For suite-level setup in NUnit, use `[OneTimeSetUp]` / `[OneTimeTearDown]`; in xUnit, use `IClassFixture<T>` or `ICollectionFixture<T>`.

---

### Question 17: How do you test a service that depends on multiple dependencies using Moq?

- **Answer:** Create a mock for each dependency, configure each with `Setup`/`Returns`, inject them into the service constructor via `.Object`, execute the method, then verify each mock's interactions separately. Only set up the methods your test code path actually calls -- leave unexpected calls unconfigured (or use strict mocks to catch them).

---

### Question 18: What is the purpose of the constraint model in NUnit assertions?

- **Answer:** The constraint model (`Assert.That(value, constraint)`) provides a fluent, expressive assertion API that produces clear failure messages. It supports rich constraints like `Is.EqualTo`, `Has.Count.GreaterThan`, `Does.Contain`, `Throws.TypeOf`, and logical composition with `.And` / `.Or`. It is preferred over the classic `Assert.AreEqual` style because of better readability and more descriptive failure messages.

---

### Question 19: How do you test middleware in an ASP.NET Core application?

- **Answer:** Use `WebApplicationFactory` to create an in-memory test server, then send HTTP requests through `HttpClient` and assert on the response headers, status codes, and body content. For example, to test a rate-limiting middleware, send multiple requests and verify that later requests return 429 Too Many Requests. You can also configure the test pipeline to include only the middleware under test.

---

### Question 20: What is the testing pyramid and where do unit and integration tests fit?

- **Answer:** The testing pyramid is a model suggesting you should have many fast unit tests at the base (cheapest, fastest, most numerous), fewer integration tests in the middle (verify component interaction), and very few end-to-end/UI tests at the top (slowest, most brittle). Unit tests verify isolated logic; integration tests verify that components work together correctly with real dependencies.

