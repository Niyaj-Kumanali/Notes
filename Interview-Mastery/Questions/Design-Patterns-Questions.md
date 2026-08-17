# Design Patterns Questions

## Questions

1. What are design principles and design patterns?
2. Explain the SOLID principles.
3. What are the DRY and KISS principles?
4. Explain the Factory design pattern.
5. Explain the Singleton design pattern.
6. Explain the Strategy design pattern.
7. Explain the Observer design pattern.
8. Explain the Adapter design pattern.

---

## Answers

1. What are design principles and design patterns?
   - **Answer:**
      - Design principles are general guidelines for writing clean, maintainable software — like SOLID, DRY, and KISS
      - Design patterns are proven, reusable solutions to recurring design problems in code — like Factory, Singleton, Strategy, Observer, and Adapter
      - Principles tell me *how to think* about design; patterns give me *ready-made templates* to solve specific problems
   - **If asked more:**
      - I can explain that principles are language-agnostic and guide every design decision, while patterns are more concrete and often solve one specific problem
      - Patterns are split into creational (object creation — Factory, Singleton), structural (object composition — Adapter, Decorator), and behavioral (object interaction — Strategy, Observer)
      - I can also explain that patterns should be applied when the problem matches, not forced into every class
2. Explain the SOLID principles.
   - **Answer:**
      - SOLID is five design principles:
         - **S**ingle Responsibility — a class should have one reason to change
         - **O**pen/Closed — classes open for extension, closed for modification
         - **L**iskov Substitution — subclasses must be replaceable by their parent without breaking behavior
         - **I**nterface Segregation — don't force clients to depend on methods they don't use
         - **D**ependency Inversion — depend on abstractions, not concrete classes
      - In my service layer, I keep each service focused on one domain (single responsibility) and inject interfaces like `PartnerRepository` rather than concrete classes (dependency inversion)
   - **If asked more:**
      - I can give an example of each: SRP (splitting a fat `InventoryService` into `InventoryValidationService` and `InventoryReportService`), OCP (adding new alert types without editing the alert dispatcher — via polymorphism), LSP (a square class breaking rectangle behavior), ISP (splitting a huge `UserService` interface into `UserReader` and `UserWriter`), and DIP (using Spring DI to inject interfaces)
      - I can also explain that violating SOLID usually shows up as code that is hard to test and hard to extend
3. What are the DRY and KISS principles?
   - **Answer:**
      - DRY (Don't Repeat Yourself) means avoid duplicating logic — extract it into a method, class, or constant and reuse it
      - KISS (Keep It Simple, Stupid) means prefer the simplest solution that works rather than over-engineering
      - I apply DRY by putting validation rules in a single utility method, and KISS by choosing a straightforward loop or stream over a complex pattern when the simple version is enough
   - **If asked more:**
      - I can explain the balance between DRY and KISS — forcing DRY too aggressively can create abstractions that are hard to follow, which violates KISS
      - I'd also mention that a small amount of duplication (like a literal in two unrelated places) is often better than a premature abstraction
      - I can give a real example where I extracted a serial-number validation helper once and reused it across validation and reporting code
4. Explain the Factory design pattern.
   - **Answer:**
      - Factory is a creational pattern that provides an interface for creating objects, letting a subclass or a central factory method decide which concrete class to instantiate
      - Instead of calling `new ConcreteClass()` in client code, I ask a factory to create it — this decouples client code from concrete implementations
      - In my projects, a factory could decide which report generator (`PDFReportGenerator`, `ExcelReportGenerator`) to create based on the requested report format
   - **If asked more:**
      - I can explain the three variants: Simple Factory (a single static method), Factory Method (subclasses override a creation method), and Abstract Factory (families of related objects)
      - I'd explain the benefit — adding a new product type means adding a new class and a new factory case, without changing client code (Open/Closed)
      - I can also mention when a factory is overkill, like when there is only one implementation
5. Explain the Singleton design pattern.
   - **Answer:**
      - Singleton ensures a class has only one instance and provides a global access point to it
      - It is typically implemented with a private constructor and a static `getInstance()` method, with either eager initialization (`private static final` instance) or lazy initialization with `synchronized` or a static holder class for thread safety
      - I would use it for a shared resource like a database connection manager or a configuration reader
   - **If asked more:**
      - I can explain thread-safety: a naive lazy singleton is broken under concurrency, so I use either eager init, double-checked locking, or the Bill Pugh static holder class (which is the cleanest)
      - I'd also mention that Singletons are often criticized because they act like global state and hurt testability — in Spring, the singleton scope gives the same single-instance guarantee but through the IoC container, so I rarely hand-roll a Singleton in Spring apps
6. Explain the Strategy design pattern.
   - **Answer:**
      - Strategy is a behavioral pattern that lets me define a family of algorithms, put each in its own class, and make them interchangeable at runtime
      - The client holds an interface reference and the algorithm can be swapped without changing the client
      - In my cold-chain project, I could define alert strategies (`ThresholdAlertStrategy`, `TrendAlertStrategy`) and switch which one is used per sensor type
   - **If asked more:**
      - I can explain the three roles — Context (holds the strategy reference), Strategy (interface), and Concrete Strategies (implementations)
      - I can show how this replaces long `if-else` chains with polymorphic dispatch, and how it combines with the Factory pattern (factory picks the strategy, client uses it)
      - I can also mention that in Java, lambdas can act as lightweight strategies for single-method algorithms
7. Explain the Observer design pattern.
   - **Answer:**
      - Observer is a behavioral pattern where one object (the subject) maintains a list of dependents (observers) and notifies them automatically when its state changes
      - This decouples the source of events from the code that reacts to them
      - In my cold-chain project, when a temperature reading exceeded a threshold, the subject could notify observers like an alert service and a dashboard updater without the sensor code knowing about them
   - **If asked more:**
      - I can explain the two roles — Subject/Observable (tracks observers, calls `notifyObservers()`) and Observer (implements an `update()` method)
      - I can mention that Java has built-in support (`java.util.Observable`, `java.util.Observer`) which is deprecated, and that the pattern is the conceptual foundation of event listeners, publish-subscribe systems, and Spring's `ApplicationEvent` mechanism
      - I can also compare Observer with pub/sub — Observer is typically in-process and synchronous, pub/sub usually has a broker
8. Explain the Adapter design pattern.
   - **Answer:**
      - Adapter is a structural pattern that converts the interface of a class into an interface the client expects, allowing incompatible classes to work together
      - It wraps an existing class and translates method calls
      - In my projects, if a third-party reporting library exposes a different API than my code expects, I create an adapter class implementing my expected interface and delegate to the third-party class inside
   - **If asked more:**
      - I can explain the two forms: Class Adapter (uses inheritance — adapter extends the adaptee) and Object Adapter (uses composition — adapter holds the adaptee, which is generally preferred)
      - I can give a real example like making a `CSVDataReader` conform to a common `DataReader` interface, or using an adapter to bridge a legacy LDAP client API to my `AuthenticationService` interface
      - I'd also explain how Adapter differs from Facade — Adapter changes an interface to match, Facade simplifies a subsystem
