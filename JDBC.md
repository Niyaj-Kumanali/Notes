# **JDBC (Java Database Connectivity)**

## **Overview**

-   **JDBC** is an API in Java for connecting and executing queries with a database.
-   Provides a standard method to interact with various relational databases (RDBMS).

---

## **JDBC Architecture**

### 1. **Components**

-   **DriverManager**: Manages JDBC drivers. Establishes the connection.
-   **Driver**: Interface to connect Java with the database.
-   **Connection**: Represents a session with a database.
-   **Statement**: Executes SQL queries.
-   **ResultSet**: Represents data retrieved from a database.

### 2. **Types of JDBC Drivers**

-   **Type 1**: JDBC-ODBC Bridge Driver (deprecated).
-   **Type 2**: Native API Driver (uses database-specific APIs).
-   **Type 3**: Network Protocol Driver (translates JDBC calls into middleware).
-   **Type 4**: Thin Driver (pure Java driver).

---

## **Steps to Work with JDBC**

### **1. Load the Driver**

-   Use `Class.forName` to load the driver class:
    ```java
    Class.forName("com.mysql.cj.jdbc.Driver");
    ```
-   For newer versions of Java, drivers are auto-registered.

---

### **2. Establish a Connection**

-   Use `DriverManager.getConnection()` to connect:
    ```java
    Connection conn = DriverManager.getConnection(url, username, password);
    ```
-   **URL format**:
    ```
    jdbc:<subprotocol>://<host>:<port>/<database>
    ```
-   Example for MySQL:
    ```
    jdbc:mysql://localhost:3306/mydb
    ```

---

### **3. Create a Statement**

-   **Statement**: For executing static SQL.
    ```java
    Statement stmt = conn.createStatement();
    ```
-   **PreparedStatement**: For parameterized queries.
    ```java
    PreparedStatement pstmt = conn.prepareStatement("SELECT * FROM users WHERE id = ?");
    pstmt.setInt(1, 1001);
    ```
-   **CallableStatement**: For stored procedures.
    ```java
    CallableStatement cstmt = conn.prepareCall("{CALL myProcedure(?)}");
    cstmt.setString(1, "parameter");
    ```

---

### **4. Execute SQL Queries**

-   **For SELECT queries**:
    ```java
    ResultSet rs = stmt.executeQuery("SELECT * FROM users");
    ```
-   **For INSERT/UPDATE/DELETE**:
    ```java
    int rows = stmt.executeUpdate("UPDATE users SET name = 'John' WHERE id = 1001");
    ```
-   **For other queries (DDL)**:
    ```java
    boolean result = stmt.execute("CREATE TABLE test (id INT)");
    ```

---

### **5. Process the Results**

-   Use `ResultSet` to fetch query results:
    ```java
    while (rs.next()) {
        System.out.println(rs.getInt("id") + " " + rs.getString("name"));
    }
    ```

---

### **6. Close the Resources**

-   Always close `Connection`, `Statement`, and `ResultSet` to avoid resource leaks:
    ```java
    rs.close();
    stmt.close();
    conn.close();
    ```

---

## **Transaction Management**

-   JDBC uses auto-commit mode by default, where every query is committed automatically.

### **Manual Transaction Management**

-   Disable auto-commit:
    ```java
    conn.setAutoCommit(false);
    ```
-   Commit manually:
    ```java
    conn.commit();
    ```
-   Rollback in case of failure:
    ```java
    conn.rollback();
    ```

---

## **Common JDBC Exceptions**

-   **`SQLException`**: Handles database errors.
    -   Get error details:
        ```java
        e.getMessage();
        e.getSQLState();
        e.getErrorCode();
        ```

---

## **JDBC Objects and Interfaces**

1. **DriverManager**: Manages database connections.
2. **Connection**: Represents a session with the database.
3. **Statement**: Executes SQL queries.
4. **PreparedStatement**: For precompiled SQL with parameters.
5. **CallableStatement**: For stored procedures.
6. **ResultSet**: Represents query results.
7. **SQLException**: Handles database errors.

---

## **Types of Statements**

1. **Statement**:

    - Executes static SQL queries.
    - Example:
        ```java
        Statement stmt = conn.createStatement();
        ResultSet rs = stmt.executeQuery("SELECT * FROM users");
        ```

2. **PreparedStatement**:

    - Executes parameterized queries, prevents SQL injection.
    - Example:
        ```java
        PreparedStatement pstmt = conn.prepareStatement("INSERT INTO users VALUES (?, ?)");
        pstmt.setInt(1, 1001);
        pstmt.setString(2, "John");
        pstmt.executeUpdate();
        ```

3. **CallableStatement**:
    - Calls stored procedures.
    - Example:
        ```java
        CallableStatement cstmt = conn.prepareCall("{CALL addUser(?, ?)}");
        cstmt.setInt(1, 1001);
        cstmt.setString(2, "John");
        cstmt.execute();
        ```

---

## **Best Practices**

1. Always close resources (`Connection`, `Statement`, `ResultSet`).
2. Use **PreparedStatement** to avoid SQL injection.
3. Use connection pooling for better performance.
4. Use try-with-resources to manage resources automatically:
    ```java
    try (Connection conn = DriverManager.getConnection(...);
         Statement stmt = conn.createStatement()) {
         // Execute SQL
    }
    ```

---

## **Example Program**

```java
import java.sql.*;

public class JDBCExample {
    public static void main(String[] args) {
        String url = "jdbc:mysql://localhost:3306/mydb";
        String username = "root";
        String password = "password";

        try (Connection conn = DriverManager.getConnection(url, username, password);
             Statement stmt = conn.createStatement();
             ResultSet rs = stmt.executeQuery("SELECT * FROM users")) {

            while (rs.next()) {
                System.out.println("ID: " + rs.getInt("id") + ", Name: " + rs.getString("name"));
            }

        } catch (SQLException e) {
            e.printStackTrace();
        }
    }
}
```

---

# **Advanced JDBC Concepts**

## 1. **Batch Processing**

Batch processing allows executing multiple SQL statements in one go, reducing the number of database calls and improving performance.

### **Usage**

-   Useful for performing bulk operations like inserting, updating, or deleting records.
-   Supports both `Statement` and `PreparedStatement`.

### **Steps**

1. Disable auto-commit mode:
    ```java
    conn.setAutoCommit(false);
    ```
2. Add SQL statements to the batch:
    ```java
    stmt.addBatch("INSERT INTO employees VALUES (1, 'Alice')");
    stmt.addBatch("INSERT INTO employees VALUES (2, 'Bob')");
    ```
3. Execute the batch:
    ```java
    int[] results = stmt.executeBatch();
    ```
4. Commit the transaction:
    ```java
    conn.commit();
    ```

### **Example with PreparedStatement**

```java
String sql = "INSERT INTO employees (id, name) VALUES (?, ?)";
try (PreparedStatement pstmt = conn.prepareStatement(sql)) {
    conn.setAutoCommit(false);

    pstmt.setInt(1, 1);
    pstmt.setString(2, "Alice");
    pstmt.addBatch();

    pstmt.setInt(1, 2);
    pstmt.setString(2, "Bob");
    pstmt.addBatch();

    int[] results = pstmt.executeBatch();
    conn.commit();
} catch (SQLException e) {
    conn.rollback();
    e.printStackTrace();
}
```

---

## 2. **Transaction Management**

Transactions ensure that a group of operations are executed atomically. If any operation fails, all changes are rolled back.

### **Key Methods**

-   `setAutoCommit(false)`: Starts a transaction.
-   `commit()`: Commits the transaction.
-   `rollback()`: Rolls back the transaction in case of failure.

### **Example**

```java
try {
    conn.setAutoCommit(false);

    // Perform operations
    stmt.executeUpdate("INSERT INTO accounts (id, balance) VALUES (1, 1000)");
    stmt.executeUpdate("UPDATE accounts SET balance = balance - 200 WHERE id = 1");

    // Commit transaction
    conn.commit();
} catch (SQLException e) {
    // Rollback if any error occurs
    conn.rollback();
    e.printStackTrace();
}
```

### **Savepoints**

-   Savepoints allow partial rollbacks to a specific point in a transaction.
-   Create a savepoint:
    ```java
    Savepoint sp1 = conn.setSavepoint("Savepoint1");
    ```
-   Rollback to a savepoint:
    ```java
    conn.rollback(sp1);
    ```
-   Release savepoints to free resources:
    ```java
    conn.releaseSavepoint(sp1);
    ```

---

## 3. **Metadata**

Metadata provides information about the database structure and query results.

### **DatabaseMetaData**

-   Retrieves database-level information.
-   Example:
    ```java
    DatabaseMetaData dbMeta = conn.getMetaData();
    System.out.println("Database Product: " + dbMeta.getDatabaseProductName());
    System.out.println("Driver Version: " + dbMeta.getDriverVersion());
    ```

### **ResultSetMetaData**

-   Retrieves information about the columns in a `ResultSet`.
-   Example:

    ```java
    ResultSet rs = stmt.executeQuery("SELECT * FROM employees");
    ResultSetMetaData rsMeta = rs.getMetaData();

    int columnCount = rsMeta.getColumnCount();
    for (int i = 1; i <= columnCount; i++) {
        System.out.println("Column Name: " + rsMeta.getColumnName(i));
        System.out.println("Column Type: " + rsMeta.getColumnTypeName(i));
    }
    ```

---

## 4. **RowSet**

RowSet is an advanced version of `ResultSet` with additional functionality.

### **Types of RowSet**

1. **JdbcRowSet**: Connected RowSet (requires active connection).
2. **CachedRowSet**: Disconnected RowSet, useful for offline processing.
3. **WebRowSet**: Serializable RowSet for web applications.

### **Example: CachedRowSet**

```java
import javax.sql.rowset.CachedRowSet;
import com.sun.rowset.CachedRowSetImpl;

CachedRowSet crs = new CachedRowSetImpl();
crs.setUrl("jdbc:mysql://localhost:3306/mydb");
crs.setUsername("root");
crs.setPassword("password");
crs.setCommand("SELECT * FROM employees");
crs.execute();

while (crs.next()) {
    System.out.println(crs.getInt("id") + " " + crs.getString("name"));
}
```

---

## 5. **CallableStatement (Stored Procedures)**

Stored procedures allow executing precompiled SQL on the database.

### **Syntax**

1. Create a stored procedure:
    ```sql
    DELIMITER //
    CREATE PROCEDURE getEmployee(IN empId INT, OUT empName VARCHAR(100))
    BEGIN
        SELECT name INTO empName FROM employees WHERE id = empId;
    END //
    DELIMITER ;
    ```
2. Call the procedure in Java:

    ```java
    CallableStatement cstmt = conn.prepareCall("{CALL getEmployee(?, ?)}");
    cstmt.setInt(1, 1); // Input parameter
    cstmt.registerOutParameter(2, Types.VARCHAR); // Output parameter
    cstmt.execute();

    String name = cstmt.getString(2);
    System.out.println("Employee Name: " + name);
    ```

---

## 6. **Connection Pooling**

Connection pooling improves performance by reusing database connections.

### **How It Works**

-   A pool of connections is maintained.
-   Applications borrow a connection from the pool and return it after use.

### **Libraries for Connection Pooling**

1. **Apache DBCP**

    - Example:

        ```java
        BasicDataSource ds = new BasicDataSource();
        ds.setUrl("jdbc:mysql://localhost:3306/mydb");
        ds.setUsername("root");
        ds.setPassword("password");
        ds.setMaxTotal(10); // Maximum connections

        Connection conn = ds.getConnection();
        ```

2. **HikariCP**

    - Lightweight and fast.
    - Example:

        ```java
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        config.setUsername("root");
        config.setPassword("password");
        HikariDataSource ds = new HikariDataSource(config);

        Connection conn = ds.getConnection();
        ```

---

## 7. **Advanced Error Handling**

### **Handling SQLException**

-   Retrieve detailed error information:
    ```java
    try {
        // Database operations
    } catch (SQLException e) {
        System.out.println("Error Code: " + e.getErrorCode());
        System.out.println("SQL State: " + e.getSQLState());
        System.out.println("Message: " + e.getMessage());
    }
    ```

### **Logging Errors**

-   Use logging frameworks (e.g., Log4j) for tracking database errors.
-   Example:
    ```java
    Logger logger = Logger.getLogger("DatabaseLogger");
    try {
        // Database operations
    } catch (SQLException e) {
        logger.error("SQL Error", e);
    }
    ```

---

## 8. **Distributed Transactions (XA Transactions)**

For handling transactions across multiple databases.

### **XA Transactions Example**

-   Use **JTA** (Java Transaction API) for managing distributed transactions.

---
