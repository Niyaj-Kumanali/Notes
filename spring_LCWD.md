# Spring Framework

## What is Spring Framework?

The **Spring Framework** is a powerful, lightweight, and flexible framework for building Java applications. It is an open-source framework that provides comprehensive infrastructure support for developing Java applications. Spring enables developers to create high-performance, reusable, and maintainable enterprise-level applications.

Spring is widely used for building **web applications**, **REST APIs**, and **enterprise-level systems**, and it provides various features that help simplify Java development. It follows the principle of **Inversion of Control (IoC)**, which helps in managing application components and their dependencies.

### Key Features of Spring Framework:

1. **Inversion of Control (IoC)**:

    - Spring uses **Dependency Injection (DI)** to manage the components of an application. This allows objects to be created and managed by the Spring container, making it easier to manage the application's dependencies.
    - Example:

        ```java
        public class MyService {
            private MyRepository myRepository;

            // Constructor injection
            @Autowired
            public MyService(MyRepository myRepository) {
                this.myRepository = myRepository;
            }

            public void performAction() {
                myRepository.saveData();
            }
        }
        ```

2. **Aspect-Oriented Programming (AOP)**:

    - Spring allows for the separation of cross-cutting concerns like logging, security, or transaction management, making the application more modular and maintainable.
    - Example:

        ```java
        @Aspect
        @Component
        public class LoggingAspect {

            @Before("execution(* com.example.service.*.*(..))")
            public void logBeforeMethod(JoinPoint joinPoint) {
                System.out.println("Logging before method execution: " + joinPoint.getSignature().getName());
            }
        }
        ```

3. **Spring MVC (Model-View-Controller)**:

    - A web framework built on the Spring framework that simplifies the development of web applications. It provides a flexible architecture and integrates well with various view technologies like JSP, Thymeleaf, and FreeMarker.
    - Example:

        ```java
        @Controller
        public class MyController {

            @GetMapping("/home")
            public String homePage() {
                return "home"; // "home" is the view name (e.g., home.jsp)
            }
        }
        ```

4. **Spring Boot**:

    - A sub-project of Spring that simplifies the configuration and deployment of Spring applications by providing production-ready defaults and embedded servers (e.g., Tomcat, Jetty).
    - Example of Spring Boot application:
        ```java
        @SpringBootApplication
        public class Application {
            public static void main(String[] args) {
                SpringApplication.run(Application.class, args);
            }
        }
        ```

5. **Spring Data**:

    - A set of projects that simplify the use of databases and repositories. It provides an abstraction layer over various database technologies like JPA, MongoDB, Cassandra, etc.
    - Example:
        ```java
        @Repository
        public interface UserRepository extends JpaRepository<User, Long> {
            List<User> findByLastName(String lastName);
        }
        ```

6. **Spring Security**:

    - Provides comprehensive security services for Java applications, including authentication, authorization, and protection against common security vulnerabilities.
    - Example:
        ```java
        @Configuration
        @EnableWebSecurity
        public class SecurityConfig extends WebSecurityConfigurerAdapter {
            @Override
            protected void configure(HttpSecurity http) throws Exception {
                http.authorizeRequests()
                    .antMatchers("/login", "/register").permitAll()
                    .anyRequest().authenticated()
                    .and()
                    .formLogin().loginPage("/login").permitAll()
                    .and()
                    .logout().permitAll();
            }
        }
        ```

7. **Spring Transaction Management**:
    - Spring provides declarative transaction management, which makes it easy to manage transactions across different types of data sources, including relational databases and message queues.
    - Example:
        ```java
        @Service
        @Transactional
        public class UserService {
            public void updateUser(User user) {
                // Transaction is automatically managed
                userRepository.save(user);
            }
        }
        ```

### Advantages of Using Spring Framework:

-   **Loose Coupling**: Spring promotes loose coupling by allowing components to be injected into other components. This makes it easier to change or extend individual components without affecting others.
-   **Simplified Development**: Spring offers a simplified way to configure applications, reducing the boilerplate code and making the development process faster.
-   **Comprehensive Ecosystem**: With projects like Spring Boot, Spring Data, Spring Security, and Spring Cloud, Spring provides a wide range of tools to simplify various aspects of application development.
-   **Testability**: Spring makes it easier to write unit tests for your application by providing features like dependency injection and AOP, which help isolate components for testing.
-   **Integration**: Spring integrates well with various technologies, including Hibernate, JPA, JMS, and more, making it versatile for a wide range of use cases.

# Dependency Injection (DI)

## What is Dependency Injection (DI)?

**Dependency Injection (DI)** is a design pattern used in object-oriented programming to achieve **Inversion of Control (IoC)** between classes and their dependencies. The idea behind DI is that instead of a class creating its dependencies internally, the dependencies are provided (injected) to the class from the outside. This pattern helps in decoupling components and enhances testability, flexibility, and maintainability of the code.

In Spring, DI is one of the core concepts, and it is implemented through **Inversion of Control (IoC) containers**. The Spring container is responsible for managing the lifecycle and dependencies of beans (objects) in the application.

Dependency Injection is a core concept in Spring that helps decouple classes and their dependencies, leading to more modular, flexible, and maintainable code. By using DI, we can manage the complexity of applications, improve testability, and promote the reusability of components. The Spring container manages the creation and wiring of beans based on configuration, making it easy to build complex applications in a modular way.

### Types of Dependency Injection

1. **Constructor Injection**:

    - In constructor injection, dependencies are provided to a class through its constructor. This is the most preferred method in Spring as it ensures that all required dependencies are provided when the object is created.
    - Example:

        ```java
        @Component
        public class MyService {
            private final MyRepository myRepository;

            // Constructor injection
            @Autowired
            public MyService(MyRepository myRepository) {
                this.myRepository = myRepository;
            }

            public void performAction() {
                myRepository.saveData();
            }
        }
        ```

2. **Setter Injection**:

    - In setter injection, dependencies are provided via setter methods. This approach allows dependencies to be injected after the object is created.
    - Example:

        ```java
        @Component
        public class MyService {
            private MyRepository myRepository;

            // Setter injection
            @Autowired
            public void setMyRepository(MyRepository myRepository) {
                this.myRepository = myRepository;
            }

            public void performAction() {
                myRepository.saveData();
            }
        }
        ```

3. **Field Injection**:

    - In field injection, dependencies are injected directly into the fields of a class, without the need for constructors or setter methods. This is the least preferred method due to its lack of visibility for dependencies and testability.
    - Example:

        ```java
        @Component
        public class MyService {

            @Autowired
            private MyRepository myRepository;

            public void performAction() {
                myRepository.saveData();
            }
        }
        ```

### Benefits of Dependency Injection

1. **Loose Coupling**:

    - DI reduces the dependency between objects. Classes do not need to know how their dependencies are created or managed, which makes the code more flexible and easier to modify.

2. **Testability**:

    - With DI, dependencies can be mocked or stubbed easily during testing, allowing for better unit testing. This is especially important for writing unit tests for classes that interact with external services like databases or APIs.

3. **Separation of Concerns**:

    - DI helps separate the concerns of how objects are created and how they interact with each other. The creation and management of dependencies are handled by the Spring container, not by the business logic of the classes.

4. **Reusability**:

    - By providing dependencies through DI, components become reusable. The same object can be injected into different classes without modification.

5. **Maintainability**:
    - As the system grows, DI helps keep the codebase more maintainable. Changes to one class do not require changes to other classes, making it easier to update, replace, or add new components.

### Spring DI Container

The Spring container is responsible for managing the objects (beans) and their dependencies. The container uses XML configuration or annotations to define how beans are created and wired together.

-   **Bean Definition**: A Spring bean is a Java object that is instantiated, assembled, and managed by the Spring container.
-   **Container Types**:
    1. **ApplicationContext**: The central interface for accessing the Spring container.
    2. **BeanFactory**: A more basic interface, mostly used in non-web applications.

### Example of Dependency Injection in Spring

Here’s an example of how DI works in a Spring application:

1.  **Bean Definitions** (`@Component` annotation):

        ```java
        @Component
        public class MyRepository {
            public void saveData() {
                System.out.println("Data saved!");
            }
        }

        @Component
        public class MyService {
            private final MyRepository myRepository;

            // Constructor Injection
            @Autowired
            public MyService(MyRepository myRepository) {
                this.myRepository = myRepository;
            }

            public void performAction() {
                myRepository.saveData();
            }
        }
        ```

        ```java

        @Configuration
        public class AppConfig {

            @Bean
            public MyRepository myRepository() {
                return new MyRepository();
            }

            @Bean
            public MyService myService() {
                return new MyService(myRepository());
            }

        }```

        ```java
        @SpringBootApplication
        public class Application {
            public static void main(String[] args) {
                AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

                // Retrieve the MyService bean from the container
                MyService myService = context.getBean(MyService.class);

                // Perform action
                myService.performAction();

                context.close();
            }
        }```

# Ambiguity Problem and Its Solution with Constructor Injection

## 1. What is the Ambiguity Problem?

The **ambiguity problem** occurs when Spring has multiple beans of the same type and can't decide which one to inject into a dependent class. This commonly happens with **constructor injection** when multiple candidate beans of the required type exist.

## 2. Example of Ambiguity

```java
@Component
public class Employee {
    private Service service;

    // Constructor Injection
    @Autowired
    public Employee(Service service) {
        this.service = service;
    }
}

@Component
public class ServiceA implements Service { ... }

@Component
public class ServiceB implements Service { ... }
```

If both ServiceA and ServiceB are present, Spring won't know which one to inject into the Employee constructor.

## 3. Solutions to Resolve Ambiguity

### 3.1 Use @Qualifier

    The @Qualifier annotation specifies which bean to inject by name.
    ```java
    @Autowired
    public Employee(@Qualifier("serviceA") Service service) {
        this.service = service;
    }```

### 3.2 Use @Primary

    The @Primary annotation marks one bean as the default when multiple candidates are available.
    ```java
    @Component
    @Primary
    public class ServiceA implements Service { ... }
    ```

### 3.3 Use Multiple Constructors

    If you have multiple constructors, use @Autowired on the one you want Spring to use for injection.
    ```java
    @Autowired
    public Employee(Service service) { ... }
    ```

# 4. Best Practices

-   Use @Qualifier when you have multiple beans of the same type.
-   Use @Primary for a default bean when one is preferred over others.
-   Avoid multiple constructors unless necessary to simplify the code.

# Inversion of Control (IoC)

## What is Inversion of Control (IoC)?

**Inversion of Control (IoC)** is a design principle in software development in which the control of object creation and the flow of execution is transferred from the programmer to a container or framework. This principle helps decouple the components of an application and makes the system more flexible and easier to manage.

In the context of Spring, IoC refers to the Spring **IoC container** that is responsible for managing the lifecycle and dependencies of application components (beans). Spring’s IoC container takes over the process of instantiating objects, managing their dependencies, and injecting the dependencies where required.

Inversion of Control is a powerful design principle that helps in creating flexible, modular, and testable applications. In Spring, IoC is implemented through the IoC container, which manages the lifecycle and dependencies of beans. By following IoC, Spring promotes loose coupling, better maintainability, and improved testability in applications.

### Key Concepts of IoC

1. **Beans**:
    - In Spring, objects that are managed by the IoC container are called beans. These beans are created, configured, and managed by the Spring container.
2. **IoC Container**:
    - The Spring IoC container is responsible for instantiating beans, configuring them, and assembling their dependencies. The container can be configured using XML, annotations, or Java configuration classes.
3. **Dependency Injection (DI)**:
    - IoC is closely related to **Dependency Injection (DI)**. DI is a mechanism used by the IoC container to provide dependencies to beans. Instead of the bean creating its own dependencies, the container injects the required dependencies into the bean.

### Types of IoC in Spring

1. **Constructor-based Injection**:

    - The IoC container injects dependencies into a bean through its constructor.
    - Example:

        ```java
        @Component
        public class MyService {
            private final MyRepository myRepository;

            @Autowired
            public MyService(MyRepository myRepository) {
                this.myRepository = myRepository;
            }

            public void performAction() {
                myRepository.saveData();
            }
        }
        ```

2. **Setter-based Injection**:

    - The IoC container injects dependencies through setter methods after the bean has been created.
    - Example:

        ```java
        @Component
        public class MyService {
            private MyRepository myRepository;

            @Autowired
            public void setMyRepository(MyRepository myRepository) {
                this.myRepository = myRepository;
            }

            public void performAction() {
                myRepository.saveData();
            }
        }
        ```

3. **Field-based Injection**:

    - The IoC container directly injects dependencies into fields of a class.
    - Example:

        ```java
        @Component
        public class MyService {

            @Autowired
            private MyRepository myRepository;

            public void performAction() {
                myRepository.saveData();
            }
        }
        ```

### Advantages of Inversion of Control

1. **Loose Coupling**:

    - IoC helps decouple the components of an application. Components do not need to know how to instantiate their dependencies or manage their lifecycle. The IoC container manages the creation and wiring of dependencies, which reduces the coupling between classes.

2. **Testability**:
    - IoC promotes testability by allowing the dependencies of a class to be easily injected. During unit testing, you can mock or replace dependencies, making the testing process easier and more isolated.
3. **Reusability**:

    - By decoupling components, IoC allows for better reuse. You can easily swap one implementation of a dependency for another without affecting the rest of the application.

4. **Flexibility and Maintainability**:
    - IoC increases the flexibility and maintainability of the application. If a dependency changes, you only need to change it in the configuration or the container, rather than making changes throughout the entire codebase.
5. **Separation of Concerns**:
    - With IoC, the responsibility of creating objects and managing dependencies is shifted to the Spring container. This separation of concerns allows developers to focus on the core business logic, while the container handles object creation and wiring.

### Spring IoC Container

Spring provides several types of containers for managing beans and their lifecycle:

1. **BeanFactory**:

    - The most basic container in Spring. It provides the fundamental features of the IoC container, such as bean creation and dependency injection. BeanFactory is typically used in non-web applications or when resources are limited.

2. **ApplicationContext**:
    - The more advanced and commonly used container. It extends BeanFactory and provides additional features such as event handling, internationalization, and support for annotating beans. `ApplicationContext` is typically used in web applications and other large-scale applications.

### Example of IoC in Spring

Here is an example that demonstrates the concept of IoC in Spring:

1. **Bean Definitions** (`@Component` annotation):

    ```java
    @Component
    public class MyRepository {
        public void saveData() {
            System.out.println("Data saved!");
        }
    }

    @Component
    public class MyService {
        private final MyRepository myRepository;

        // Constructor Injection
        @Autowired
        public MyService(MyRepository myRepository) {
            this.myRepository = myRepository;
        }

        public void performAction() {
            myRepository.saveData();
        }
    }
    ```

    ```java
    @Configuration
    public class AppConfig {

        @Bean
        public MyRepository myRepository() {
            return new MyRepository();
        }

        @Bean
        public MyService myService() {
            return new MyService(myRepository());
        }
    }
    ```

    ```java
    @SpringBootApplication
    public class Application {
        public static void main(String[] args) {
            AnnotationConfigApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

            // Retrieve the MyService bean from the container
            MyService myService = context.getBean(MyService.class);

            // Perform action
            myService.performAction();

            context.close();
        }
    }
    ```

    In the example above, the IoC container manages the creation of the MyRepository and MyService beans. The MyService bean depends on the MyRepository bean, and the Spring container injects the MyRepository instance into the MyService instance automatically. This is achieved through constructor injection, making the code more flexible and maintainable.

# Introduction to Spring Modules

The **Spring Framework** is a comprehensive, modular framework that provides a wide variety of tools for building Java-based enterprise applications. The framework is organized into different modules, each designed to address specific concerns in application development.

Each module in the Spring Framework can be used independently or in combination, depending on the needs of the application. Some modules provide core functionality like dependency injection, while others offer specific features like transaction management, web development, and data access.

The key advantage of Spring is its flexibility and modularity—developers can use only the necessary modules based on their project requirements.

The Spring Framework offers a wide variety of modules that cater to different aspects of application development, from core functionalities like Dependency Injection to advanced topics like web development, security, and messaging. By offering modularity, Spring allows developers to pick and choose only the modules they need, making it a highly flexible and efficient framework for building enterprise-grade applications.

Each module in Spring focuses on a specific concern, allowing developers to build clean, maintainable, and scalable applications. This modular approach also enables easy integration with other technologies, making Spring a powerful tool in Java development.

Here are the major Spring modules:

# Core Container Modules

The **Core Container** of the Spring Framework consists of four primary modules that provide fundamental features such as Dependency Injection (DI), beans management, and AOP (Aspect-Oriented Programming).

1. **Core Module**:

    - The **Core** module is the foundation of the Spring Framework. It provides essential features like Dependency Injection (DI) and the BeanFactory.
    - **BeanFactory** and **ApplicationContext** are the key interfaces in this module. The **ApplicationContext** is the more advanced container used for configuring beans and integrating with other Spring modules.

2. **Beans Module**:

    - The **Beans** module is responsible for managing the beans in the Spring container. It provides functionality for defining, configuring, and managing beans using configuration metadata (XML, annotations, or Java Config).
    - It also supports the lifecycle management of beans and their dependencies.

3. **Context Module**:

    - The **Context** module provides the means for accessing and interacting with the Spring container. It builds upon the **Beans** module by adding features like event propagation, internationalization, and access to the bean factory.
    - The **ApplicationContext** interface, which extends the **BeanFactory**, provides a more feature-rich version of the bean container, including capabilities for event listeners, resource loading, and more.

4. **SpEL (Spring Expression Language) Module**:
    - **SpEL** is a powerful expression language that can be used for querying and manipulating objects at runtime. It is commonly used in Spring configurations (XML, Java, and annotations) to provide flexibility in bean configuration and management.

# Data Access and Integration Modules

Spring also provides a suite of modules for data access and integration with different data sources, such as relational databases, messaging systems, and transaction management.

1. **JDBC Module**:

    - The **JDBC** module simplifies database interaction by providing a **JdbcTemplate** class that handles low-level details of database connections, such as exception handling, connection management, and prepared statement management.
    - It helps in reducing boilerplate code when working with JDBC.

2. **ORM Module**:

    - The **ORM** (Object-Relational Mapping) module integrates Spring with popular ORM frameworks like Hibernate, JPA (Java Persistence API), and JDO (Java Data Objects).
    - It provides a consistent programming model for data access and transaction management across different ORM frameworks.

3. **JMS Module**:

    - The **JMS** (Java Message Service) module provides an abstraction layer for integrating with messaging systems, such as ActiveMQ or IBM MQ.
    - It simplifies the use of JMS APIs, allowing for message-driven beans (MDBs) and JMS templates to interact with queues and topics.

4. **Transaction Module**:
    - The **Transaction** module provides support for programmatic and declarative transaction management.
    - It allows developers to manage transactions through a simple and consistent API, regardless of the underlying transaction management system (JTA, JDBC, Hibernate, etc.).

# Web Modules

Spring provides robust support for building web applications. The web modules are designed for building both traditional web applications (servlet-based) and modern web services (RESTful APIs).

1. **Web Module**:

    - The **Web** module provides fundamental support for web applications, such as handling requests, responses, and managing session data.
    - It contains utilities for working with **servlets**, **filters**, and **listeners**. It can be used to configure and set up a **DispatcherServlet**, the central controller in Spring-based web applications.

2. **Web MVC Module**:

    - The **Web MVC** module provides the Spring MVC framework, which is used for building web applications based on the Model-View-Controller (MVC) design pattern.
    - It includes features like controllers, views, form handling, validation, and more. It is one of the most commonly used modules for building web applications in Spring.

3. **Web WebSocket Module**:

    - The **WebSocket** module provides support for building real-time, bidirectional communication applications using WebSocket protocols.
    - It integrates WebSocket communication into Spring, enabling the creation of WebSocket endpoints and handlers.

4. **WebFlux Module**:
    - The **WebFlux** module is a reactive programming model for building non-blocking, asynchronous web applications.
    - It supports reactive stream processing and can be used for building modern, scalable web services that handle many concurrent connections.

# Security Module

The **Spring Security** module provides comprehensive security features for Java applications, including authentication, authorization, and protection against common attacks like CSRF (Cross-Site Request Forgery) and session fixation.

-   **Spring Security** integrates with both web applications and REST APIs, providing a flexible and extensible security framework for controlling access to resources.
-   It offers authentication mechanisms such as username/password, OAuth2, LDAP, and Single Sign-On (SSO).
-   It also provides authorization features for securing methods, endpoints, and resources, based on user roles or other conditions.

# Aspect-Oriented Programming (AOP) Module

The **AOP** module provides support for Aspect-Oriented Programming in Spring applications. AOP enables cross-cutting concerns (such as logging, security, or transaction management) to be modularized into separate concerns (aspects).

-   **Spring AOP** allows developers to define aspects that can be applied to multiple classes and methods, making it easier to implement concerns like logging or transaction management across the application.
-   It works by creating **advice** (the action to be performed) that is associated with certain **join points** (methods or execution points in the program).

Common uses of AOP in Spring include:

-   Transaction management
-   Security enforcement
-   Logging and monitoring

# Messaging Module

The **Spring Messaging** module provides integration with messaging systems, supporting a wide range of messaging protocols and message brokers.

-   It simplifies messaging integration by abstracting the underlying protocols, such as **JMS**, **AMQP** (Advanced Message Queuing Protocol), and others.
-   This module enables developers to build applications that can send and receive messages asynchronously and reliably, often in the context of enterprise message-driven architecture.

Some features of Spring Messaging:

-   **Message handlers**: Allow for processing incoming messages.
-   **Message channels**: Facilitate communication between message-producing and consuming components.
-   **Message converters**: Enable easy conversion of messages between different formats.

# Test Module

The **Spring Test** module provides comprehensive testing support for Spring-based applications. It integrates with JUnit and other testing frameworks to help developers write unit and integration tests for their Spring beans, services, and controllers.

-   The **@SpringBootTest** annotation is commonly used to bootstrap the entire Spring application context during tests.
-   **MockMvc** is used to test Spring MVC controllers in isolation, providing a mock HTTP environment for integration testing.

Spring also provides **@MockBean** and **@Autowired** to inject dependencies and mock beans, which can be particularly useful for unit testing.

# Life Cycle Methods in Spring Framework

In Spring, the life cycle of a bean refers to the stages a bean goes through during its existence in the Spring container. These stages include bean creation, initialization, and destruction. Spring provides multiple ways to manage the life cycle of beans, using XML, interfaces, and annotations.

## 1. Life Cycle Methods using XML

### 1.1 Using `<init-method>` and `<destroy-method>` in XML Configuration

In XML-based configuration, we can specify the initialization and destruction methods for a bean using the `init-method` and `destroy-method` attributes.

#### Example:

```xml
<bean id="myBean" class="com.example.MyBean" init-method="init" destroy-method="cleanup">
</bean>
```

-   `init-method`: Specifies the method to be called after the bean is initialized.
-   `destroy-method`: Specifies the method to be called before the bean is destroyed.

#### Example Bean Class:

```java
public class MyBean {
    public void init() {
        System.out.println("Bean is initialized");
    }

    public void cleanup() {
        System.out.println("Bean is destroyed");
    }
}
```

## 2. Life Cycle Methods using Interfaces

### 2.1 Implementing `InitializingBean` and `DisposableBean`

Spring provides two interfaces, `InitializingBean` and `DisposableBean`, that you can implement in your bean classes to handle initialization and destruction.

-   `InitializingBean`: The `afterPropertiesSet()` method is called after the properties are set on the bean.
-   `DisposableBean`: The `destroy()` method is called when the bean is destroyed.

#### Example:

```java
import org.springframework.beans.factory.InitializingBean;
import org.springframework.beans.factory.DisposableBean;

public class MyBean implements InitializingBean, DisposableBean {

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("Bean is initialized using InitializingBean");
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("Bean is destroyed using DisposableBean");
    }
}
```

#### XML Configuration:

```xml
<bean id="myBean" class="com.example.MyBean"/>
```

## 3. Life Cycle Methods using Annotations

### 3.1 Using `@PostConstruct` and `@PreDestroy` Annotations

Spring also supports lifecycle methods using annotations. The `@PostConstruct` annotation is used to specify the method to be called after the bean's properties are set. The `@PreDestroy` annotation is used to specify the method to be called before the bean is destroyed.

#### Example:

```java
import javax.annotation.PostConstruct;
import javax.annotation.PreDestroy;

public class MyBean {

    @PostConstruct
    public void init() {
        System.out.println("Bean is initialized using @PostConstruct");
    }

    @PreDestroy
    public void cleanup() {
        System.out.println("Bean is destroyed using @PreDestroy");
    }
}
```

#### Enabling Annotations in XML:

To enable annotations, the Spring configuration should include the `<context:annotation-config>` tag.

```xml
<context:annotation-config/>
<bean id="myBean" class="com.example.MyBean"/>
```

Alternatively, using Java-based configuration:

```java
@Configuration
@ComponentScan(basePackages = "com.example")
public class AppConfig {
}
```

## 4. Summary of Life Cycle Methods

| Method Type    | XML Configuration  | Interface Implementation | Annotation Approach |
| -------------- | ------------------ | ------------------------ | ------------------- |
| Initialization | `<init-method>`    | `afterPropertiesSet()`   | `@PostConstruct`    |
| Destruction    | `<destroy-method>` | `destroy()`              | `@PreDestroy`       |

## 5. Conclusion

Spring provides multiple ways to manage the lifecycle of beans, whether through XML configuration, interface implementation, or annotations. The approach you choose will depend on the type of configuration (XML, annotation-based, or Java-based) and your project needs.

Each method provides flexibility in handling initialization and destruction processes for Spring beans, ensuring a well-managed lifecycle.

# Autowiring in Spring Framework

Autowiring in Spring Framework is a mechanism that allows Spring to automatically inject the required dependencies into a bean. It eliminates the need for explicit `setter` or `constructor` injection by the developer.

### Types of Autowiring

1. **Autowiring by Type (`@Autowired`)**
2. **Autowiring by Name**
3. **Autowiring by Constructor**
4. **Autowiring by Qualifier**

### 1. Autowiring by Type

When Spring injects a dependency by type, it matches the property of the bean with the type of the autowired bean. This is the default mode of autowiring in Spring.

#### Example using `@Autowired` for type-based autowiring

```java
import org.springframework.beans.factory.annotation.Autowired;

public class Car {

    @Autowired
    private Engine engine;

    public void start() {
        engine.run();
    }
}
```

### 2. Autowiring by Name

Autowiring by name works by matching the name of the property to the name of the bean in the container.

#### Example

```java
<bean id="engine" class="com.example.Engine" />
<bean id="car" class="com.example.Car" autowire="byName" />
```

### 3. Autowiring by Constructor

Autowiring by constructor works by matching the constructor argument types with the available beans in the container.

#### Example

```java
import org.springframework.beans.factory.annotation.Autowired;

public class Car {

    private Engine engine;

    @Autowired
    public Car(Engine engine) {
        this.engine = engine;
    }

    public void start() {
        engine.run();
    }
}
```

### 4. Autowiring with `@Qualifier`

In case there are multiple beans of the same type, you can use the `@Qualifier` annotation to specify which bean should be injected.

#### Example

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;

public class Car {

    @Autowired
    @Qualifier("v8Engine")
    private Engine engine;

    public void start() {
        engine.run();
    }
}
```

### Summary

-   **Autowiring by Type**: Injects dependencies based on the type of the property.
-   **Autowiring by Name**: Matches the property name with the bean name.
-   **Autowiring by Constructor**: Uses constructor arguments to autowire dependencies.
-   **`@Qualifier`**: Used to resolve ambiguity when multiple beans of the same type are available.

# Autowiring Using XML

Autowiring using XML configuration is a way of injecting dependencies into beans using the `autowire` attribute in the Spring XML configuration file.

### 1. Autowire by Type

In this mode, Spring will automatically inject dependencies based on the type of the property.

#### Example

```xml
<bean id="engine" class="com.example.Engine" />
<bean id="car" class="com.example.Car" autowire="byType" />
```

### 2. Autowire by Name

This mode matches the property name with the bean name in the container.

#### Example

```xml
<bean id="engine" class="com.example.Engine" />
<bean id="car" class="com.example.Car" autowire="byName" />
```

### 3. Autowire by Constructor

In this mode, Spring will inject dependencies through the constructor of the bean.

#### Example

```xml
<bean id="engine" class="com.example.Engine" />
<bean id="car" class="com.example.Car" autowire="constructor" />
```

### 4. Summary

-   **Autowire by Type**: Injects dependencies by matching the type of the property with available beans.
-   **Autowire by Name**: Injects dependencies by matching the property name with the bean name.
-   **Autowire by Constructor**: Injects dependencies through the constructor of the bean.
