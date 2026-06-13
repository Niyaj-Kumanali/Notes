# Text Blocks

## Overview

- **Definition** — A text block is a multi-line string literal delimited by triple double-quotes (`"""`) that preserves line breaks while stripping incidental leading whitespace.
- **Why It Exists** — To eliminate the need for explicit newline characters (`\n`), string concatenation (`+`), and escape sequences when writing multi-line strings such as SQL, JSON, HTML, or XML.
- **Historical Context** — Previewed in JDY 13, received a second preview in JDK 14, and was finalized in JDK 15.
- **Key Concepts** — **Delimiter rules**: opening `"""` must be followed by a line terminator; closing `"""` determines indentation by its position; **leading whitespace** is stripped using the indent algorithm (common leading whitespace across all lines is removed); **escape sequences** still work inside text blocks; **new escape sequences**: `\s` forces a trailing space, `\linebreak` suppresses the line break for line continuation; **formatted()** for interpolation instead of native string interpolation.

## Core Concepts

- Text block delimiter: opening `"""` must be followed by a line terminator (newline). The content starts on the next line.

  ```java
  String html = """
    <html>
      <body>
        <p>Hello</p>
      </body>
    </html>
    """;
  ```

- The closing `"""` position determines the stripping of incidental leading whitespace. The compiler removes the common leading whitespace shared by the closing `"""` and all content lines.
- Indent algorithm:
  - Determines the column position of the closing `"""`.
  - Calculates the minimum indentation across all non-blank content lines.
  - Removes that minimum indentation from every content line.
  - Trailing whitespace on each line is stripped.
- Escape sequences work inside text blocks:

  ```java
  String quote = """
    He said, "Hello."
    Tab:\tindent
    """;
  ```

- New escape sequences introduced for text blocks:
  - `\s` — forces a trailing space (trailing whitespace is normally stripped).
  - `\linebreak` (or `\<newline>`) — suppresses the line break for line continuation:

  ```java
  String longQuery = """
    SELECT id, name, email \
    FROM users \
    WHERE active = true
    """;
  ```

- Java does not have native string interpolation. Use `formatted()` or `String.format()` for variable substitution:

  ```java
  String name = "Alice";
  int age = 30;
  String json = """
    {
      "name": "%s",
      "age": %d
    }
    """.formatted(name, age);
  ```

## Common Mistakes

- **Placing the opening """ on the same line as content**
  - If `"""` is followed by non-whitespace on the same line, the compiler treats everything after `"""` as content on an empty line, causing unexpected indentation or content.
  - **Why it looks correct:** Regular strings start and end on the same line.
  - The fix: always put the opening `"""` at the end of a line with nothing after it and start the content on the next line.

- **Misunderstanding whitespace stripping with the closing """ position**
  - The position of the closing `"""` determines how much leading whitespace is stripped. If the closing `"""` is not aligned with the content, the output will not match expectations.
  - **Why it looks correct:** The closing `"""` looks like a closing bracket that could be anywhere.
  - The fix: align the closing `"""` with the leftmost column of content lines. All content lines must have the same base indentation for predictable results.

- **Expecting runtime interpolation like JavaScript template literals**
  - Text blocks do not support `${variable}` syntax. They are compile-time constants.
  - **Why it looks correct:** Other languages use `${}` inside multi-line strings for interpolation.
  - The fix: use `textBlock.formatted(args)` or `String.format(textBlock, args)` for runtime substitution.

- **Forgetting that trailing whitespace is stripped**
  - Text blocks strip trailing whitespace from every line. If you need a trailing space, use the `\s` escape.
  - **Why it looks correct:** In normal strings, trailing spaces are preserved.
  - The fix: add `\s` at the end of any line that requires a trailing space.

- **Using + concatenation inside a text block instead of formatted()**
  - You can concatenate text blocks with `+`, but it breaks the multi-line readability advantage.
  - **Why it looks correct:** Concatenation is the traditional way to build multi-line strings in Java.
  - The fix: use `formatted()` for variable substitution and keep the text block as a single literal.

## Real-World Scenarios

### SQL Query Formatting

- Embed multi-line SQL queries without concatenation or `+` signs:

  ```java
  String query = """
    SELECT u.id, u.name, o.total
    FROM users u
    JOIN orders o ON u.id = o.user_id
    WHERE o.status = 'ACTIVE'
    ORDER BY o.total DESC
    LIMIT ?
    """;
  PreparedStatement ps = connection.prepareStatement(query);
  ```

### JSON Payload Construction

- Write JSON literals directly without escaping every quote:

  ```java
  String payload = """
    {
      "user": {
        "name": "Alice",
        "email": "alice@example.com",
        "roles": ["admin", "user"]
      }
    }
    """;
  ```

### HTML Email Templates

- Define multi-line HTML templates inline without concatenation:

  ```java
  String emailHtml = """
    <!DOCTYPE html>
    <html>
    <body>
      <h1>Welcome, %s!</h1>
      <p>Click <a href="%s">here</a> to verify your account.</p>
    </body>
    </html>
    """.formatted(userName, verificationLink);
  ```

## Scenario-Based Questions

**Q: You are writing a SQL query with a dynamic WHERE clause that depends on user input. How would you use a text block while safely interpolating user-controlled values?**

- Use a text block with `?` placeholders and `formatted()` for non-user parts, or better, use `PreparedStatement` with `?` placeholders inside the text block and pass parameters separately to prevent SQL injection. Text blocks are compile-time constants, so runtime values must be injected via `formatted()` or set as prepared statement parameters.

  ```java
  String query = """
    SELECT * FROM users
    WHERE active = ?
    AND role = ?
    """;
  PreparedStatement ps = conn.prepareStatement(query);
  ps.setBoolean(1, true);
  ps.setString(2, role);
  ```

- **Interview follow-up:** What security concern arises if you use `formatted()` to insert raw user input into a SQL text block?

**Q: A junior developer writes a text block with the closing """ directly after the last line like this: `"""content"""`. What output do they get and why?**

- The closing `"""` is not on its own line, so the compiler treats it as part of the content. The resulting string will include the trailing `"""` as literal characters. Text blocks require the opening `"""` to be followed by a line terminator, and the closing `"""` to determine indentation from its line.
- **Interview follow-up:** How would you correctly write a single-line text block?

## Interview Questions

- **What determines how much leading whitespace is stripped from a text block?**
  - The position of the closing `"""`. The compiler measures the column offset of the closing `"""` and subtracts that amount of leading whitespace from every non-blank content line. Additional leading whitespace may also be stripped to the minimum common indentation across all lines.

- **What is the purpose of the \s and \<linebreak> escape sequences in text blocks?**
  - `\s` forces a trailing space at the end of a line that would otherwise have its trailing whitespace stripped. `\<linebreak>` (a backslash followed by a newline) suppresses the line break for line continuation, allowing you to split a long line across multiple source lines without adding a newline to the string.

- **Can you use text blocks to represent a regular expression?**
  - Yes, but you must be careful with backslashes. Inside a text block, `\\` represents a single backslash, just like in a regular string literal. Text blocks make multi-line regex patterns more readable.

- **Are text blocks compile-time constants?**
  - Yes. A text block is a constant expression of type `String`. It is computed at compile time and stored in the constant pool, just like a regular string literal.

## Developer Recommendations

- **Use text blocks for all multi-line SQL, JSON, HTML, and XML literals**
  - Eliminates the need for concatenation operators and escape sequences, making the code match the actual content format.
  - Place the closing `"""` at the indentation level that matches the desired left margin of the content.

- **Use formatted() for variable substitution instead of concatenation**
  - Keeps the multi-line structure intact and makes the substitution points visible.
  - For prepared SQL statements, use `?` placeholders in the text block and set parameters via the `PreparedStatement` API to avoid SQL injection.

- **Align closing """ with the leftmost content column**
  - Prevents confusion about whitespace stripping. If all content lines have the same indentation, align the closing `"""` with them.
  - For embedded content (e.g., JSON inside a method), indent the text block relative to the surrounding code and align the closing `"""` accordingly.

- **Use \s for trailing spaces and \<newline> for long lines**
  - When a line must end with a space (e.g., before a unit in a formatted message), append `\s`.
  - Use `\` at the end of a line to break an extremely long text block line without introducing an actual newline.
  - **Production story:** A team maintaining a reporting module replaced 200 lines of string concatenation for SQL and JSON generation with text blocks, reducing accidental syntax errors and making the queries directly copy-pasteable into a database console for debugging.
