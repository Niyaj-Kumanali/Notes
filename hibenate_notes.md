# Hibernate Notes

===============

## 1. Overview

-   **Definition:** Hibernate is a framework for mapping Java objects to database tables (Object-Relational Mapping or ORM) and managing data persistence.

    -   It simplifies the interaction between Java applications and databases by converting Java objects into database records and vice versa.

-   **Key Features:**

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
            -   **First-level cache:** Automatically enabled and scoped to a session.
            -   **Second-level cache:** Configurable and shared across sessions, improves performance for frequently used data.
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

## 2. Core Components

-   **SessionFactory:**

    -   A factory for `Session` objects.
        -   It is responsible for creating `Session` instances, which are used for interacting with the database.
    -   Created once per application lifecycle and is thread-safe.
        -   Typically created at the start of an application and shared across multiple threads.
    -   Reads configuration from `hibernate.cfg.xml` or `application.properties`.
        -   The `SessionFactory` is configured based on the provided configuration file, which includes database connection details and Hibernate properties.
        -   Example:
            ```java
            SessionFactory factory = new Configuration()
                                       .configure("hibernate.cfg.xml")
                                       .addAnnotatedClass(Employee.class)
                                       .buildSessionFactory();
            ```

-   **Session:**

    -   A lightweight, non-thread-safe object for database operations.
        -   Used to interact with the database and manage entity objects.
    -   Represents a single unit of work.
        -   Operations like save, update, delete, and load are performed within a session.
    -   Common methods:
        -   `save()`: Inserts a new record into the database.
            ```java
            session.save(employee);
            ```
        -   `update()`: Updates an existing record in the database.
            ```java
            session.update(employee);
            ```
        -   `delete()`: Deletes a record from the database.
            ```java
            session.delete(employee);
            ```
        -   `get()`: Retrieves a record eagerly (immediately loads the data).
            ```java
            Employee employee = session.get(Employee.class, 1);
            ```
        -   `load()`: Retrieves a proxy for a record (lazy loading).
            ```java
            Employee employee = session.load(Employee.class, 1);
            ```

-   **Transaction:**

    -   Ensures atomicity of database operations.
        -   Ensures that operations like saving, updating, and deleting are executed as a single transaction and are committed together.
    -   Common methods:
        -   `beginTransaction()`: Starts a new transaction.
            ```java
            Transaction transaction = session.beginTransaction();
            ```
        -   `commit()`: Commits the current transaction, saving the changes.
            ```java
            transaction.commit();
            ```
        -   `rollback()`: Rolls back the current transaction in case of an error.
            ```java
            transaction.rollback();
            ```

-   **Query:**
    -   Used to execute both HQL (Hibernate Query Language) and native SQL queries.
        -   HQL allows querying entity objects, while native SQL can query raw database tables.
    -   Methods:
        -   `createQuery(String hql)`: Creates a query based on HQL.
            ```java
            Query<Employee> query = session.createQuery("FROM Employee WHERE salary > 50000", Employee.class);
            List<Employee> employees = query.getResultList();
            ```
        -   `createSQLQuery(String sql)`: Creates a query based on native SQL.
            ```java
            Query query = session.createSQLQuery("SELECT * FROM employee WHERE salary > 50000");
            List<Object[]> results = query.getResultList();
            ```
        -   `setParameter()`: Used to set parameters in a query.
            ```java
            query.setParameter("salary", 50000);
            ```

## 3. Configuration

-   **Hibernate Configuration File:**

    -   `hibernate.cfg.xml`: Central configuration file for Hibernate.

        -   This file contains necessary settings to connect Hibernate with a database and control its behavior during runtime.
        -   Example (`hibernate.cfg.xml`):

            ```xml
            <hibernate-configuration>
                <session-factory>
                    <!-- Database connection settings -->
                    <property name="hibernate.dialect">org.hibernate.dialect.MySQLDialect</property>
                    <property name="hibernate.connection.url">jdbc:mysql://localhost:3306/mydb</property>
                    <property name="hibernate.connection.username">root</property>
                    <property name="hibernate.connection.password">password</property>

                    <!-- JDBC connection pool settings -->
                    <property name="hibernate.c3p0.min_size">5</property>
                    <property name="hibernate.c3p0.max_size">20</property>

                    <!-- Schema management -->
                    <property name="hibernate.hbm2ddl.auto">update</property>

                    <!-- Enable SQL logging -->
                    <property name="hibernate.show_sql">true</property>
                    <property name="hibernate.format_sql">true</property>
                </session-factory>
            </hibernate-configuration>
            ```

    -   Key properties:
        -   `hibernate.dialect`: Database dialect (e.g., `org.hibernate.dialect.MySQLDialect`).
            -   Defines the SQL dialect Hibernate should use to communicate with the database.
        -   `hibernate.connection.url`: Database connection URL.
            -   The URL to the database, e.g., `jdbc:mysql://localhost:3306/mydb`.
        -   `hibernate.connection.username`: Database username.
            -   The username used to authenticate the database connection.
        -   `hibernate.connection.password`: Database password.
            -   The password associated with the database username.
        -   `hibernate.hbm2ddl.auto`: Schema management (`update`, `validate`, `create`, `create-drop`).
            -   Controls schema generation and updating strategies:
                -   `update`: Automatically updates the schema to match the entity model.
                -   `create`: Drops and re-creates the schema on startup.
                -   `validate`: Validates the schema with the current mapping.
                -   `create-drop`: Drops the schema when the session factory is closed.

-   **Mapping Files:**

    -   `*.hbm.xml`: Maps Java classes to database tables.

        -   These files are used to configure object-relational mapping (ORM) outside annotations, though annotations are commonly used now.
        -   Example (`Employee.hbm.xml`):
            ```xml
            <hibernate-mapping>
                <class name="com.example.Employee" table="employee">
                    <id name="id" column="employee_id">
                        <generator class="identity"/>
                    </id>
                    <property name="name" column="employee_name"/>
                    <property name="salary" column="employee_salary"/>
                </class>
            </hibernate-mapping>
            ```

    -   Contains `<class>` tags with attributes like `<id>` and `<property>`.
        -   `<class>`: Defines a class mapping to a database table.
        -   `<id>`: Specifies the primary key.
        -   `<property>`: Defines a property mapping to a column in the table.

-   **Logging Configuration:**
    -   Enable SQL logging using `hibernate.show_sql` and `hibernate.format_sql`.
        -   `hibernate.show_sql`: Enables output of generated SQL to the console.
        -   `hibernate.format_sql`: Formats the SQL for better readability.
        -   Example:
            ```xml
            <property name="hibernate.show_sql">true</property>
            <property name="hibernate.format_sql">true</property>
            ```

---

## 4. Hibernate Annotations

-   Replace XML configurations with annotations directly in Java classes.

    -   Annotations in Hibernate provide an easy way to configure entity mappings without needing XML files, making the code more concise and readable.

-   Common Annotations:

    -   `@Entity`: Marks a class as an entity.

        ```java
        @Entity
        public class Employee {
            @Id
            private Long id;
            private String name;
            private double salary;
        }
        ```

    -   `@Table`: Specifies the database table name.

        ```java
        @Entity
        @Table(name = "employee_table")
        public class Employee {
            @Id
            private Long id;
            private String name;
            private double salary;
        }
        ```

    -   `@Id`: Marks the primary key.

        ```java
        @Entity
        public class Employee {
            @Id
            @GeneratedValue(strategy = GenerationType.IDENTITY)
            private Long id;
            private String name;
            private double salary;
        }
        ```

    -   `@GeneratedValue`: Specifies the primary key generation strategy.

        ```java
        @Entity
        public class Employee {
            @Id
            @GeneratedValue(strategy = GenerationType.IDENTITY)
            private Long id;
            private String name;
            private double salary;
        }
        ```

    -   `@Column`: Maps a field to a database column.

        ```java
        @Entity
        public class Employee {
            @Id
            private Long id;

            @Column(name = "emp_name", length = 100, nullable = false)
            private String name;
        }
        ```

    -   `@OneToOne`, `@OneToMany`, `@ManyToOne`, `@ManyToMany`: Define relationships between entities.

        -   `@OneToOne` Example:

            ```java
            @Entity
            public class Employee {
                @Id
                private Long id;

                @OneToOne
                @JoinColumn(name = "profile_id")
                private Profile profile;
            }
            ```

        -   `@OneToMany` Example:

            ```java
            @Entity
            public class Department {
                @Id
                private Long id;

                @OneToMany(mappedBy = "department")
                private List<Employee> employees;
            }
            ```

        -   `@ManyToOne` Example:

            ```java
            @Entity
            public class Employee {
                @Id
                private Long id;

                @ManyToOne
                @JoinColumn(name = "department_id")
                private Department department;
            }
            ```

        -   `@ManyToMany` Example:

            ```java
            @Entity
            public class Student {
                @Id
                private Long id;

                @ManyToMany
                @JoinTable(
                    name = "student_course",
                    joinColumns = @JoinColumn(name = "student_id"),
                    inverseJoinColumns = @JoinColumn(name = "course_id")
                )
                private List<Course> courses;
            }
            ```

# 5. Hibernate Query Language (HQL)

-   A database-independent query language.

    -   HQL allows you to write queries in terms of your Java objects (entities), rather than database tables. It is object-oriented and works with Java entity classes.

-   Operates on entity objects, not tables.

    -   Unlike SQL, which works with tables, HQL works directly with entities and their properties.

-   Common Examples:

    -   **Select**:
        ```java
        List<Employee> employees = session.createQuery("FROM Employee WHERE salary > 50000").list();
        ```
    -   **Insert**:
        ```java
        session.createQuery("INSERT INTO Employee (id, name) VALUES (1, 'John')")
               .executeUpdate();
        ```
    -   **Update**:
        ```java
        session.createQuery("UPDATE Employee SET salary = 60000 WHERE id = 1")
               .executeUpdate();
        ```
    -   **Delete**:
        ```java
        session.createQuery("DELETE FROM Employee WHERE id = 1")
               .executeUpdate();
        ```

-   Named Queries:

    -   Define reusable queries using `@NamedQuery` annotation to avoid writing the same query multiple times in the code.
    -   Example:

        ```java
        @Entity
        @NamedQuery(name = "Employee.findBySalary", query = "FROM Employee WHERE salary > :salary")
        public class Employee {
            @Id
            private Long id;
            private String name;
            private double salary;
        }

        Query query = session.getNamedQuery("Employee.findBySalary");
        query.setParameter("salary", 50000);
        List<Employee> employees = query.list();
        ```

# 6. Caching in Hibernate

-   **First-Level Cache**:

    -   Enabled by default in Hibernate.
    -   Scoped to a `Session`, meaning it is valid only during the lifecycle of that session.
    -   Used to reduce the number of database queries by storing objects in memory within the current session.
    -   **Example**:
        ```java
        Session session = sessionFactory.openSession();
        Employee emp1 = session.get(Employee.class, 1); // First-level cache stores emp1
        Employee emp2 = session.get(Employee.class, 1); // emp2 is fetched from first-level cache, not DB
        session.close();
        ```

-   **Second-Level Cache**:

    -   Requires explicit configuration in `hibernate.cfg.xml` or `persistence.xml`.
    -   Shared across multiple sessions and persists data beyond the scope of a single session.
    -   Useful for caching data that is frequently accessed across different sessions.
    -   Popular providers include EHCache, Redis, and Infinispan.
    -   **Example of enabling second-level cache in `hibernate.cfg.xml`**:
        ```xml
        <hibernate-configuration>
            <session-factory>
                <property name="hibernate.cache.use_second_level_cache">true</property>
                <property name="hibernate.cache.region.factory_class">org.hibernate.cache.ehcache.EhCacheRegionFactory</property>
            </session-factory>
        </hibernate-configuration>
        ```

-   **Query Cache**:
    -   Caches query results to avoid repeated execution of the same query.
    -   Needs second-level cache to be enabled.
    -   **Example of enabling query cache**:
        ```java
        Session session = sessionFactory.openSession();
        session.createQuery("FROM Employee WHERE salary > 50000")
               .setCacheable(true)
               .list();
        session.close();
        ```

# 7. Relationships in Hibernate

-   **One-to-One**:

    -   Represents a one-to-one relationship between two entities.
    -   **Example**:

        ```java
        @Entity
        public class User {
            @Id
            @GeneratedValue
            private Long id;

            @OneToOne(mappedBy = "user")
            private Profile profile;
        }

        @Entity
        public class Profile {
            @Id
            @GeneratedValue
            private Long id;

            @OneToOne
            @JoinColumn(name = "user_id")
            private User user;
        }
        ```

-   **One-to-Many**:

    -   Represents a one-to-many relationship where one entity is related to multiple entities.
    -   **Example**:

        ```java
        @Entity
        public class Department {
            @Id
            @GeneratedValue
            private Long id;

            @OneToMany(mappedBy = "department")
            private List<Employee> employees = new ArrayList<>();
        }

        @Entity
        public class Employee {
            @Id
            @GeneratedValue
            private Long id;

            @ManyToOne
            @JoinColumn(name = "department_id")
            private Department department;
        }
        ```

-   **Many-to-One**:

    -   Represents a many-to-one relationship where many entities are related to a single entity.
    -   **Example**:

        ```java
        @Entity
        public class Employee {
            @Id
            @GeneratedValue
            private Long id;

            @ManyToOne
            @JoinColumn(name = "department_id")
            private Department department;
        }

        @Entity
        public class Department {
            @Id
            @GeneratedValue
            private Long id;
        }
        ```

-   **Many-to-Many**:

    -   Represents a many-to-many relationship where multiple entities are related to multiple entities.
    -   **Example**:

        ```java
        @Entity
        public class Student {
            @Id
            @GeneratedValue
            private Long id;

            @ManyToMany
            @JoinTable(
                name = "student_course",
                joinColumns = @JoinColumn(name = "student_id"),
                inverseJoinColumns = @JoinColumn(name = "course_id")
            )
            private List<Course> courses = new ArrayList<>();
        }

        @Entity
        public class Course {
            @Id
            @GeneratedValue
            private Long id;

            @ManyToMany(mappedBy = "courses")
            private List<Student> students = new ArrayList<>();
        }
        ```

# 8. Lifecycle of an Entity

-   **Transient**:

    -   Object created but not associated with a `Session` or `Database`.
    -   **Example**:
        ```java
        Employee emp = new Employee();
        emp.setName("John Doe"); // Object is in the transient state.
        ```

-   **Persistent**:

    -   Object associated with a `Session` and synchronized with the `Database`.
    -   **Example**:
        ```java
        Session session = sessionFactory.openSession();
        Transaction tx = session.beginTransaction();
        session.save(emp); // Object becomes persistent.
        tx.commit();
        session.close();
        ```

-   **Detached**:

    -   Object previously associated with a `Session` but now disconnected.
    -   The object still exists in the `Database` but is no longer managed by the `Session`.
    -   **Example**:
        ```java
        session.close(); // The `emp` object is now detached.
        emp.setName("John Updated"); // Modifications will not be saved to the database.
        ```

-   **Removed**:
    -   Object marked for deletion from the `Database`.
    -   Object is still associated with the `Session` until the transaction is committed.
    -   **Example**:
        ```java
        Session session = sessionFactory.openSession();
        Transaction tx = session.beginTransaction();
        session.delete(emp); // Object is marked for removal.
        tx.commit(); // Object is removed from the database.
        session.close();
        ```

# 9. Common Issues and Solutions

-   **LazyInitializationException**:

    -   **Cause**: Accessing uninitialized proxy objects outside a `Session`.
        -   Hibernate uses proxy objects for lazy loading, which can cause this exception if you try to access them after the `Session` has been closed.
    -   **Solution**: Use eager fetching or ensure the `Session` is open when accessing the entity.
        -   **Eager fetching** loads the association when the parent entity is loaded, thus avoiding the exception.
            ```java
            @OneToMany(fetch = FetchType.EAGER)
            private List<Address> addresses;
            ```
        -   Alternatively, keep the `Session` open during the data access.
            ```java
            session.beginTransaction();
            Employee employee = session.get(Employee.class, 1);
            System.out.println(employee.getAddresses());
            session.getTransaction().commit();
            ```

-   **DuplicateRecordException**:

    -   **Cause**: Re-saving detached objects.
        -   This happens when an object that was previously persisted (and now detached) is re-saved using `save()` or `update()`, which can lead to duplication.
    -   **Solution**: Use `merge()` instead of `save()` or `update()`.
        -   The `merge()` method will check if the entity already exists in the database and update it if necessary, avoiding duplication.
            ```java
            session.merge(employee);
            ```

-   **StaleObjectStateException**:
    -   **Cause**: Concurrent updates to the same data.
        -   This exception occurs when multiple transactions try to update the same record in the database at the same time, causing a conflict.
    -   **Solution**: Use versioning with `@Version` annotation.
        -   The `@Version` annotation ensures that the version number is checked before updating the entity, preventing conflicting updates.
            ```java
            @Version
            private int version;
            ```
        -   The version field is automatically managed by Hibernate, and any concurrent updates to the same record will result in an exception if the version number has changed.

# 10. Advanced Topics

-   **Criteria API**:

    -   **Programmatically create queries**.
        -   The Criteria API is a flexible way to build queries dynamically in a type-safe manner, avoiding the need for string-based queries.
    -   **Example**:
        ```java
        CriteriaBuilder builder = session.getCriteriaBuilder();
        CriteriaQuery<Employee> query = builder.createQuery(Employee.class);
        Root<Employee> root = query.from(Employee.class);
        query.select(root).where(builder.greaterThan(root.get("salary"), 50000));
        List<Employee> employees = session.createQuery(query).getResultList();
        ```
    -   This allows you to build complex queries programmatically, such as joins, filtering, and ordering, with compile-time type checking.

-   **Batch Processing**:

    -   **Use JDBC Batch Processing for large datasets**.
        -   Batch processing allows you to insert or update large numbers of entities in a single database operation, improving performance.
    -   **Example**:
        ```java
        for (int i = 0; i < employees.size(); i++) {
            session.save(employees.get(i));
            if (i % 50 == 0) {
                session.flush(); // Write changes to the database
                session.clear(); // Clear session to release memory
            }
        }
        ```
    -   By using `flush()` and `clear()`, you can efficiently process large datasets without overwhelming memory.

-   **Interceptors**:

    -   **Customize Hibernate behavior by implementing the `Interceptor` interface**.
        -   Interceptors allow you to hook into Hibernate’s internal operations (e.g., save, update) and modify behavior, such as logging, auditing, or filtering data.
    -   **Example**:

        ```java
        public class MyInterceptor implements Interceptor {
            @Override
            public boolean onSave(Object entity, Serializable id, Object[] state, String[] propertyNames, Type[] types) {
                if (entity instanceof Employee) {
                    System.out.println("Saving employee with ID: " + id);
                }
                return false; // Continue with default behavior
            }
            // Other overridden methods (e.g., onLoad, onUpdate) can be implemented similarly.
        }

        sessionFactory.getCurrentSession().setInterceptor(new MyInterceptor());
        ```

    -   This allows for custom operations, like automatically logging entity changes or modifying entity state before it’s persisted.

---

# 11. Tools and Frameworks for Hibernate

-   **Spring Data JPA**:

    -   **Simplifies the use of JPA-based repositories with Hibernate**.
        -   Spring Data JPA provides an abstraction layer over Hibernate, simplifying CRUD operations and queries.
        -   It reduces boilerplate code by automatically implementing repository interfaces, allowing easy interaction with the database.
    -   **Example**:
        ```java
        @Repository
        public interface EmployeeRepository extends JpaRepository<Employee, Long> {
            List<Employee> findBySalaryGreaterThan(Double salary);
        }
        ```
        -   The repository interface automatically provides implementations for standard database operations, such as `save()`, `findById()`, and custom queries like `findBySalaryGreaterThan()`.

-   **Liquibase/Flyway**:
    -   **Database version control tools that integrate well with Hibernate**.
        -   **Liquibase** and **Flyway** are tools for managing database schema migrations. They track changes to the database schema and apply them automatically during application startup.
        -   These tools ensure that the database schema stays consistent across different environments, handling versioning and rollback capabilities.
    -   **Liquibase**:
        -   Uses XML, YAML, or JSON changelogs to define database changes.
        -   Can be integrated with Hibernate for automated schema migrations.
    -   **Flyway**:
        -   Uses SQL-based migrations to track changes.
        -   Integrates seamlessly with Hibernate to handle database versioning.
    -   **Example**:
        -   **Liquibase**:
            ```xml
            <databaseChangeLog>
                <changeSet author="developer" id="1" >
                    <addColumn tableName="employee">
                        <column name="salary" type="double"/>
                    </addColumn>
                </changeSet>
            </databaseChangeLog>
            ```
        -   This adds a new column `salary` to the `employee` table, and Liquibase will ensure it gets applied to the database during the migration process.
