## Framework

-   A framework represents set of generic programs used by developers in order to enhance an application or software application.
-   Hibernate is also a framework which majorly helps in achieving the concept of ORM(Object Relational Programming).
-   Hibernate was created by `Govin King` and the first release was in year `2001`.

## ORM

-   A ORM is an comcept which represents the process of mapping java objects to respective relational database tables.
-   There are various Java frameworks that are built using concept of ORM.
    1. Hibernate
    2. Ibotis
    3. TopLink
    4. EclipseLink
-   ```java
        class Employee
        {
            private int id;
            private String name;
            private int sal;
        }
    ```
-   One Java object is one row in database table.
-   The hibernate and table together known as `Persistance layer`.

## Maven

-   Maven in java is a software project which is used in build the project, managers the project and also manages the dependencies.
-   It is a built automation tool developed by `Apache Software Foundation`.
-   It uses `pom.xml` file for the configuration which contains the project details and manages the dependencies. (POM --> Project Object Model)

## Components of hibernate layer

    Configuration --> Session Factory --> Session --> DB

-   Configuration contains the information about the database.
-   It provides raw fact to the session session factory.
-   Session factory uses this raw and produces the session.

    **Configuration**

    -   Configuration in hibernate represents a component which is responsible to store and process the configuration information such as database, mapping, entity related etc.
    -   In hibernate, Configuration is a class.
    -   We store the appring information, entity information in an xml file known as `<objectName>.hbm.xml`.
    -   Configuration object is responsible for to read and process the above xml files to perform the database related operations.
    -   Configuration object uses or contains method known as `configure` method in order to read and process the configuration file.
    -   Configure method is an overloaded method.
    -   In hibernate, there is a provision to provide custom names to the configuration files.
    -   In no agument configure method reads the configuration file by default.

    **SessionFactory**

    -   A SessionFactory in hibernate is interface which is present in package `org.hibernate`.
    -   SessionFactory object contains compiled immutable database and mapping information.
    -   We can get sessionfactory instances from configuration object.

    **Session**

    -   Session in hibernate is an interface which is present in package `org.hibernate`.
    -   Session objects are generated using sessionfactory interface.
    -   One sessionfactory can generate n number os sessions intastances.
    -   Technically a session is a single threaded short lived object which represents the `communication between persistance layer and Java application layer`.

        1. **Get method**

        -   Get method in hibernate is an overloaded method which helps in reading the data from the database.
        -   If the data is not their in the databasse then the get method returns `null`.

        2. **Load method**

        -   load method is also same as that of get method which helps in data retrieval but if the data is not present then it will throw `ObjectNotFoundException`.

    **Transaction**

    -   It is also an interface which is present in package `org.hibernate`.
    -   Transaction instances are generated using session.
    -   The session object uses the information of the database from sessionfactory and works with transaction to perform database operation.
    -   `create` value for `hbm2ddl.auto` represents a fresh creation of database tables with no update of data.
    -   In other words create parameter always drops the table if exits and create fresh table with fresh data.
    -   The `update` parameter always updates the data in the database. If the table is not present in the database then update will create a fresh table then inserts the data.

## Relational Mapping

-   **Types**

    1. **one to one**
        - one to one mapping represents the process o mapping one object to only one other object
        - one to one mapping represents with the annotation `@OneToOne`.
    2. **one to many**

        - The process of mapping one object to many objects.
        - It is represented by `@OneToMany`.

    3. **many to one**
        - It is a type of mapping where many objects are mapped to one single object.
        - It is represented by `@ManyToOne`.
    4. **many to many**
        - It is a type of mapping where many objects are mapped across many other objects.
        - It is represented by `@ManyToMany`.

## Lazy and Eager

-   In ORM, the data fetching is classified into two types:

    1. **Lazy loading**
        - It is a design pattern which helps in fetching the data only when user requests for. In Other words the data will not be fetched unless it is requested.
        - **Ex**: @ManyToMany abd @ManyToMany are considered lazy loading.
    2. **Eager loading**
        - It is a design pattern which the data is loaded eagerly even without the request.
        - **Ex**: @OneToOne abd @ManyToOne are considered Eager loading.

    ```java
    public class App{
        public static void main(String[] args){
            Man m = new Man();
            m.setId(1);
            m.setName("Tom");

            Women w = new Women();
            w.setId(1);
            w.setName("Barbie");

            m.setWomen(w);
        }
    }

    @Entity
    public class Man {
        @id
        private int id;
        private String name;

        @OneToOne(fetch= fetchType.Lazy)
        private women w;
    }

    @Entity
    public class Women {
        @id
        private int id;
        private String name;
    }
    ```

## Hibernate Caching

-   Hibernate caching is a mechanism which is built in order to increase the application performance and reduce number of database hits.
-   There are two types of caching

    1. **First level cache**
        - It is default caching mechanism in hibernate.
        - It is associated with session objects.
        - It gets the life when the session is opened and looses the life when session is closed. In other words, the objects that are cached in first level cache of level 1 is not visible to first level cache of any other session.
        - Their is a provision to delete or remove the objects from the cache memory.
        - Once the session is closed all the objects present in first level cache will be lost.
    2. **Second level cache**

        - Second level cache memory or level 2 cache is another type of caching mechanism in hibernate.
        - It is always associated with the SessionFactory.
        - It is always shared among the number of session got created.
        - SessionFactory is responsible for creation and destruction of second level cache memory.
        - From SessionFactory point of view(POV) second level cache memory is the point of contact.
        - When a client triggers a request from the session POV it is always checks for first level cache memory.
        - It the data is not present in first level cache memory then the session works for second level cache memory.
        - If the data is not present even in second level cache memory then a query gets fired to the database server and correspondingly the response copy will be stored in first level and second level cache memory.
        - Under the some SessionFactory if a new session get created and try to access the same data as previous session then the session object first looks for first level cache memory(Since a copy of data is present in the level 2 cache from previous cache) and retrieves the data.

        **Steps to enable second level cache**

        1. Download ehCache library from using maven dependencies.
        2. Their are various second level cache memory providers like ehCache, osCache, swarmp etc.
        3. Download hibernate ehCache dependency from maven.
        4. Add a property tag in `hibernate.cfg.xml` in order to enable second level cache memory and to specify the region factory class of hibernate ehCache dependency.
        5. The region factory class acts as a bridge between the hibernate and cache providers.
        6. The hibernate framework will not have any information related to cache providers. Hence region factory class acts a bridge.
        7. Add `@Cachable` and `@Cache` annotations with concurrency strategy on top of each entity classes that has to be part of second level cache memory.

        **Cacheable**

        - This annotation helps the entity class to get notified as a cached object in second level cache memory in other words it helps in notifying the hibernate that the entity class is cacheable in second level cache memory.

        **Cache**

        - This annotation helps the second level cache memory to store the entity object in the second level cache memory based on cache con-current strategy.
        - There is 5 different types of cache-currency strategy in hibernate.
            1. NONE
            2. READ_ONLY
            3. READ_WRITE
            4. NONRESTRICTED_READ_WRITE
            5. TRANSACTION

## Hibernate Query Language(HQL)

-   It is an object oriented query language which is similar to that of SQL but instead of table name we use class name, and column names with variables names.
-   HQL is case sensitive with respect to class names and variables names.
-   HQL supports various clauses like `FROM`, `SELECT`, `JOIN`, `aggregate functions`, `WHERE`, `AS`, `ORDER BY`, `GROUP BY`, `DELETE`, and `UPDATE` etc.
-   Generally developers are recommended to use HQL because it is very close to object resemebalance.
-   There are three types of queries which helps in performing database operations
    1. **Query**
        - It is an interface which represents an object orientation of a query.
        - The object of query interface is created using create query method.
        - Their are certain methods in query interface, some of them are `public int ExecuteUpdate()`, `public list list()` etc.
        - The create query method is invoked using session object.
    2. **Native Query**
    3. **Criteria API Query**
        - It provides object oriented approach of data retrieval from database.
        - Hibernate Criteria API is always used to bulk results.
        - We can not use hibernate criteria API fr data updatation and deletion or any DDL statements.
        - Hibernate criteria API fetches the result from the database using object oriented approach.
        - We have `create` criteria method which helps in fetching the data from the database based on entity class specified.
        - Perform CRUD operation using native query in hibernate CUD.
        - `WhP` in order to fetch a specific record using criteria API.
        - Perform an R and D on projections in criteria API.

## Java Persistance API(JPA)

-   **API**: Application programming Interface.
-   JPA only provides specification, it does not provide any implementation.
-   The specification of JPA are kind of set of rules and guidelines for implementing ORM.
-   Hibernate is know for implementation of JPA is present in `javax.persistence.package`.
-   Hibernate JPA internally uses JPQL(Java Persistence Query Language) in order to task to the database.
-   JPA uses entity manager factory interface in order to initiate the database connection process.
-   All the configuration related to the database connectivity are stored in `persistence.xml` file.
-   entity manager factory interface helps in creating the instances of entity manager interface which internally creates transaction instance and performs the database related operations.
-   entity manager factory is based on a design pattern known as factory design pattern.
