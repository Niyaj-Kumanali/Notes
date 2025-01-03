
# JDBC (Java Database Connectivity) Notes

## 1. Java Database Connectivity (JDBC) Overview
### **Info:**
- JDBC is an API that enables Java applications to interact with relational databases. It allows the execution of SQL queries, updating data, and retrieving results.
- It provides a uniform interface to interact with multiple database systems such as MySQL, Oracle, and PostgreSQL.

### **Key Features:**
- **Database independence:** JDBC abstracts away database-specific details.
- **Dynamic SQL execution:** You can execute SQL queries dynamically at runtime.
- **Transaction support:** JDBC supports transaction management, ensuring atomic operations.
- **Metadata support:** Retrieves information about the database and the result set.
- **Batch processing:** Optimizes performance by sending multiple SQL statements in one request.

---

## 2. JDBC Architecture

### **Info:**
- **JDBC API:** Provides interfaces for Java applications to interact with databases.
- **JDBC Drivers:** A set of classes that translate JDBC calls to the database-specific calls.
- **JDBC Manager:** Acts as the mediator between the API and the database drivers.
- **Database Server:** The backend where data is stored and managed.

### **Example:**
```plaintext
Application -> JDBC API -> DriverManager -> Database Driver -> Database
```

---

## 3. JDBC API Components

### **1. DriverManager**
#### **Info:**
- **DriverManager** is a central class that manages database drivers.
- It selects the appropriate database driver based on the connection URL.

#### **Example:**
```java
Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/testdb", "root", "password");
System.out.println("Connection established successfully!");
```

### **2. Connection**
#### **Info:**
- Represents a database session.
- Allows you to create **Statements** for executing SQL queries and manage transactions.

#### **Example:**
```java
Connection conn = DriverManager.getConnection("jdbc:oracle:thin:@localhost:1521:orcl", "user", "password");
conn.setAutoCommit(false); // Manual transaction management
conn.close(); // Always close connection
```

### **3. Statement**
#### **Info:**
- Used to execute simple SQL queries that don’t require parameters.
- It returns a **ResultSet** containing the query results.

#### **Example:**
```java
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT * FROM products");
while (rs.next()) {
    System.out.println("Product Name: " + rs.getString("name"));
}
```

### **4. PreparedStatement**
#### **Info:**
- Used for executing precompiled SQL queries with input parameters.
- More secure and efficient for repeated executions.

#### **Example:**
```java
PreparedStatement pstmt = conn.prepareStatement("INSERT INTO products (id, name) VALUES (?, ?)");
pstmt.setInt(1, 1);
pstmt.setString(2, "Laptop");
pstmt.executeUpdate();
```

### **5. CallableStatement**
#### **Info:**
- Used to execute stored procedures in the database.
- Supports input, output, and input-output parameters.

#### **Example:**
```java
CallableStatement cstmt = conn.prepareCall("{call updateStock(?, ?)}");
cstmt.setInt(1, 101);
cstmt.setInt(2, 50);
cstmt.execute();
```

### **6. ResultSet**
#### **Info:**
- Represents the result set returned by executing a query.
- Provides methods to navigate through the data (e.g., `next()`, `getString()`).

#### **Example:**
```java
ResultSet rs = stmt.executeQuery("SELECT * FROM employees");
while (rs.next()) {
    System.out.println("Name: " + rs.getString("name"));
    System.out.println("Age: " + rs.getInt("age"));
}
```

---

## 4. JDBC Driver Types

### **1. Type-1 (JDBC-ODBC Bridge)**
#### **Info:**
- Translates JDBC calls into ODBC calls.
- Requires ODBC driver setup.
- Deprecated due to poor performance.

#### **Example:**
```java
Class.forName("sun.jdbc.odbc.JdbcOdbcDriver");
Connection conn = DriverManager.getConnection("jdbc:odbc:myDSN");
```

### **2. Type-2 (Native API)**
#### **Info:**
- Converts JDBC calls to database-specific native API calls.
- Requires native libraries.

#### **Example:**
```java
Class.forName("oracle.jdbc.driver.OracleDriver");
Connection conn = DriverManager.getConnection("jdbc:oracle:thin:@localhost:1521:xe", "user", "password");
```

### **3. Type-3 (Network Protocol Driver)**
#### **Info:**
- Converts JDBC calls to network protocol calls that are translated by middleware.
- Suitable for internet-based applications.

### **4. Type-4 (Thin Driver)**
#### **Info:**
- Pure Java driver that translates JDBC calls to database-specific protocol.
- Does not require any external libraries.

#### **Example:**
```java
Class.forName("com.mysql.cj.jdbc.Driver");
Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/testdb", "root", "password");
```

---

## 5. Transaction Management

### **1. Transactions**
#### **Info:**
- A transaction groups multiple operations as a single unit.
- Ensures **ACID** properties (Atomicity, Consistency, Isolation, Durability).

### **2. Auto-Commit Mode**
#### **Info:**
- By default, JDBC commits changes automatically after each SQL statement.

#### **Example:**
```java
conn.setAutoCommit(false); // Disable auto-commit
```

### **3. Commit and Rollback**
#### **Example:**
```java
conn.commit();   // Commit changes
conn.rollback(); // Rollback changes
```

### **4. Savepoints**
#### **Info:**
- Savepoints mark specific points in a transaction, allowing partial rollbacks.

#### **Example:**
```java
Savepoint sp = conn.setSavepoint();
conn.rollback(sp); // Rollback to savepoint
conn.commit();
```

---

## 6. Advanced Features

### **1. Batch Processing**
#### **Info:**
- Batch processing allows executing multiple SQL queries in one request, reducing database interaction time.

#### **Example:**
```java
Statement stmt = conn.createStatement();
stmt.addBatch("INSERT INTO employees VALUES (1, 'Alice')");
stmt.addBatch("INSERT INTO employees VALUES (2, 'Bob')");
stmt.executeBatch();
```

### **2. Metadata**
#### **1. DatabaseMetaData**
#### **Info:**
- Provides detailed information about the database (e.g., product name, version, supported features).

#### **Example:**
```java
DatabaseMetaData metaData = conn.getMetaData();
System.out.println("Database Product: " + metaData.getDatabaseProductName());
```

#### **2. ResultSetMetaData**
#### **Info:**
- Provides information about the columns in a **ResultSet** (e.g., column count, data type).

#### **Example:**
```java
ResultSetMetaData rsMeta = rs.getMetaData();
System.out.println("Number of Columns: " + rsMeta.getColumnCount());
```

### **3. Connection Pooling**
#### **Info:**
- Connection pooling improves performance by reusing database connections instead of opening new ones for every request.

#### **Example (HikariCP):**
```java
HikariConfig config = new HikariConfig();
config.setJdbcUrl("jdbc:mysql://localhost:3306/testdb");
config.setUsername("root");
config.setPassword("password");
HikariDataSource ds = new HikariDataSource(config);
Connection conn = ds.getConnection();
```

---

## 7. Security and Best Practices

### **1. Avoid Hardcoding Credentials:**
- Use environment variables or configuration files to securely manage credentials.

### **2. Use PreparedStatements:**
- **PreparedStatements** protect against SQL injection attacks by using parameterized queries.

### **3. Close Resources:**
- Always close `Connection`, `Statement`, and `ResultSet` objects to avoid resource leaks.

#### **Example:**
```java
stmt.close();
conn.close();
rs.close();
```

### **4. Use Connection Pooling:**
- For better performance, always use connection pooling libraries like HikariCP or Apache DBCP.

### **5. Validate Input:**
- Always validate user inputs before passing them into SQL queries to avoid injection attacks.
