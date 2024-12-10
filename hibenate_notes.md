# Hibernate Notes

## 1. Overview

### _Definition_: Hibernate is a framework for mapping Java objects to database tables (Object-Relational Mapping or ORM) and managing data persistence.

-   It simplifies the interaction between Java applications and databases by converting Java objects into database records and vice versa.

### _Key Features_:

-   **Simplifies database interactions by eliminating boilerplate SQL code.**

    -   Hibernate automatically generates SQL for CRUD (Create, Read, Update, Delete) operations.
        ```java
        session.save(employee);
        ```

-   **Supports HQL (Hibernate Query Language) for complex queries.**

    -   HQL allows you to write database queries in an object-oriented manner, using entity objects instead of database tables.
        ```java
        Query<Employee> query = session.createQuery("FROM Employee WHERE salary > 50000", Employee.class);
        List<Employee> employees = query.getResultList();
        ```

-   **Provides caching mechanisms for performance optimization.**

    -   Hibernate supports caching to reduce the load on the database by storing frequently accessed data in memory.
        -   First-level cache: Automatically enabled and scoped to a session.
        -   Second-level cache: Configurable and shared across sessions, improves performance for frequently used data.
        ```xml
        <!-- Sample configuration for second-level cache (EHCache) -->
        <property name="hibernate.cache.provider_class">org.hibernate.cache.EhCacheProvider</property>
        ```

-   **Automatically generates SQL queries.**

    -   Hibernate can automatically generate SQL queries based on the entity class mappings, eliminating the need to manually write SQL.
        ```java
        session.save(employee);
        ```

-   **Works with multiple databases without changing the code.**
    -   Hibernate abstracts the underlying database, allowing the same Java code to run on different databases (e.g., MySQL, PostgreSQL, Oracle).
        ```xml
        <!-- Hibernate configuration for MySQL -->
        <property name="hibernate.dialect">org.hibernate.dialect.MySQLDialect</property>
        <property name="hibernate.connection.url">jdbc:mysql://localhost:3306/mydb</property>
        <property name="hibernate.connection.username">root</property>
        <property name="hibernate.connection.password">password</property>
        ```

---
