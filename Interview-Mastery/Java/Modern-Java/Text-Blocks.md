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

## Use Cases

Text blocks eliminate the readability tax of multi-line string literals — no more `\n`, broken indentation, or escaped quotes obscuring the actual content.

- **SQL queries** — Write multi-line queries with natural indentation and without concatenation noise.
  - Open a text block, write the SQL as you would in a database console, and close. Example: a DAO layer with complex JOIN queries for reporting.
  - **Avoid when:** the query is dynamic — use `?` placeholders with `PreparedStatement` and set parameters separately to prevent injection.

- **HTML/XML templates** — Embed markup directly in code with proper structure visible to the developer.
  - Compose email bodies or API response templates as text blocks with `%s` or `%s`/`formatted()` for substitution. Example: a verification email with HTML styling.
  - **Avoid when:** the template is large or changes frequently — move to a dedicated template file (Thymeleaf, FreeMarker) and keep Java code for logic.

- **JSON or YAML constants** — Include sample payloads, test fixtures, or configuration snippets without escaping every quote.
  - Paste a JSON document verbatim into a text block and strip incidental indentation with `stripIndent()`. Example: test data for a REST API integration test.
  - **Avoid when:** the constant must be compiled once and reused across modules — define it in a resource file instead.

- **Shell scripts or command strings** — Embed small command sequences or multi-line instructions.
  - Use a text block for a series of shell commands passed to `ProcessBuilder` or SSH exec calls. Example: a deployment script that creates directories, copies files, and restarts a service.
  - **Avoid when:** the script is platform-specific and long — prefer a separate script file to avoid mixing languages in the same source file.

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

**Q: You need to embed a JSON payload in a Java test file. How would you ensure that the text block indentation does not add extra whitespace to the JSON string?**

- Indent the text block content to the same level as the surrounding Java code. The compiler strips the common leading whitespace based on the closing `"""` position. Place the closing `"""` at the same indentation as the content lines to remove the incidental whitespace.

  ```java
  String json = """
      {
        "name": "test",
        "value": 42
      }
      """;
  ```

- **Interview follow-up:** What happens if some content lines have different indentation than others?

**Q: You are writing a multi-line SQL query with dynamic parameters. How do you combine text blocks with prepared statement placeholders?**

- Write the SQL as a text block with `?` placeholders and pass it to `PreparedStatement`. The text block makes the SQL readable. Never use `formatted()` with user input to avoid SQL injection.

  ```java
  String sql = """
      SELECT id, name, email
      FROM users
      WHERE status = ?
      AND created_at > ?
      """;
  ```

- **Interview follow-up:** How would you handle an IN clause with a dynamic number of parameters?

**Q: Your logging framework produces multi-line log messages that are hard to read. How could text blocks improve log message formatting?**

- Use text blocks to write structured log messages with clear line separation. Each field on its own line makes logs more parseable. Combine with `formatted()` for runtime values.
- **Interview follow-up:** Would you ever use text blocks for log messages in production code vs. keeping them as single-line strings?

**Q: A teammate writes a text block with `\t` for indentation instead of spaces, but the output is misaligned. Why?**

- Text block whitespace stripping uses the column position of the closing `"""`. Mixed tabs and spaces cause incorrect calculation of common indentation because a tab may not align predictably. Use spaces consistently within text blocks.
- **Interview follow-up:** How does the text block algorithm handle tabs in content lines?

**Q: You need to generate an XML document as a string. How would text blocks simplify this compared to traditional string concatenation?**

- Write the entire XML document as a text block with proper indentation. No escaping of quotes is needed. Use `formatted()` for dynamic attribute values or content. The result is directly copy-pasteable to an XML validator.
- **Interview follow-up:** How would you generate XML with dynamically repeating elements (e.g., multiple `<item>` tags) using text blocks?

**Q: You want to test that a method produces the correct multi-line output. How would you use text blocks for the expected value in your assertion?**

- Use a text block for the expected string in the assertion. The test becomes self-documenting because the expected output is visually identical to the actual output. Align the closing `"""` to the same indentation as the test method content.

  ```java
  String expected = """
      Line 1
      Line 2
      Line 3
      """;
  assertEquals(expected, actual);
  ```

- **Interview follow-up:** How do you handle trailing newlines in text block comparison for assertions?

**Q: You are writing a code generator that produces Java source code. How would you use text blocks to generate method bodies?**

- Use text blocks to template method bodies with `%s` placeholders for dynamically generated names. The text block preserves the generated code's indentation structure. Escape sequences still work inside text blocks for generating strings.
- **Interview follow-up:** How would you handle the case where the generated code itself contains text blocks?

**Q: A developer uses `+` concatenation to append a text block with another string. What is the readability concern?**

- Concatenating a text block with `+` breaks the visual alignment advantage of the text block. The concatenation operator splits the logical string into parts, making it harder to copy-paste. Use `formatted()` or `String.format()` instead.
- **Interview follow-up:** Is there a performance difference between concatenation and formatted() with text blocks?

**Q: You have a text block that contains a literal sequence of three double quotes. How do you escape this without breaking the text block delimiter?**

- Use `\"""` to escape the triple quote sequence inside a text block. The backslash tells the compiler that these quotes are literal content, not the closing delimiter.
- **Interview follow-up:** Can text blocks contain binary data or only text?

## Interview Questions

- **What determines how much leading whitespace is stripped from a text block?**
  - The position of the closing `"""`. The compiler measures the column offset of the closing `"""` and subtracts that amount of leading whitespace from every non-blank content line. Additional leading whitespace may also be stripped to the minimum common indentation across all lines.

- **What is the purpose of the \s and \<linebreak> escape sequences in text blocks?**
  - `\s` forces a trailing space at the end of a line that would otherwise have its trailing whitespace stripped. `\<linebreak>` (a backslash followed by a newline) suppresses the line break for line continuation, allowing you to split a long line across multiple source lines without adding a newline to the string.

- **Can you use text blocks to represent a regular expression?**
  - Yes, but you must be careful with backslashes. Inside a text block, `\\` represents a single backslash, just like in a regular string literal. Text blocks make multi-line regex patterns more readable.

- **Are text blocks compile-time constants?**
  - Yes. A text block is a constant expression of type `String`. It is computed at compile time and stored in the constant pool, just like a regular string literal.

- **Can text blocks be used in annotations?**
  - Yes, text blocks can be used as values for annotation elements of type `String`, because they are compile-time constants.

- **How do text blocks handle carriage return (`\r`) characters?**
  - Text blocks normalize line endings. `\r\n` and `\r` are converted to `\n` (LF) during compilation, ensuring consistent behavior across Windows, Linux, and macOS.

- **What is the maximum length of a text block?**
  - There is no explicit maximum. Text blocks are limited only by available memory and the JVM's maximum string length (typically `Integer.MAX_VALUE` characters).

- **Can you nest text blocks inside each other?**
  - You cannot nest text block delimiters in Java source. The first `"""` opens the text block and the next `"""` closes it. To include literal `"""` inside a text block, escape it with `\"""`.

- **How does the indent algorithm determine which whitespace is incidental?**
  - The algorithm computes the minimum leading whitespace across all non-blank content lines. It then strips that many leading whitespace characters from each content line. The position of the closing `"""` sets this baseline.

- **What is the difference between `formatted()` and `String.format()` with text blocks?**
  - `textBlock.formatted(args)` is an instance method that calls `String.format(this, args)`. They are functionally equivalent, but `formatted()` reads more naturally as applying the template to the text block.

- **Can text blocks be used as the argument to `String.lines()` and similar methods?**
  - Yes. Since text blocks are `String` instances, all `String` methods work on them. `textBlock.lines()` is commonly used to process multi-line content stream.

- **Do text blocks support escape sequences like `\n`, `\t`, `\\`?**
  - Yes. All standard Java escape sequences work inside text blocks. `\n` adds an explicit newline, `\t` adds a tab, and `\\` adds a backslash.

- **What happens if you use a text block with only whitespace lines?**
  - The text block's indent algorithm still applies. If the closing `"""` is at position 0, all leading whitespace is stripped, potentially resulting in an empty string.

- **Can you use text blocks with `switch` expressions?**
  - Yes. Text blocks are `String` literals and can be used anywhere a `String` is expected, including as values in switch expressions.

- **How do text blocks interact with Java's `indent()` method?**
  - `textBlock.indent(n)` adds or removes leading whitespace based on the value of `n`. Positive values add indentation; negative values remove up to `n` leading whitespace characters.

- **What is the purpose of the `stripIndent()` method?**
  - `String.stripIndent()` (introduced with text blocks as a public API) programmatically applies the same indent stripping algorithm that the compiler uses for text blocks, removing common leading whitespace.

- **Can text blocks contain Unicode escape sequences?**
  - Yes. Unicode escape sequences like `\u00e9` work inside text blocks, just as in regular string literals, and are processed before any other escaping.

- **How do you write a text block that represents an empty string?**
  - Write `""" """` (opening quotes, space, closing quotes on the same line) or use two consecutive text block delimiters. However, the empty text block `""""""` does not compile because the quotes are ambiguous.

- **What is the difference between a text block and a raw string literal in other languages?**
  - Unlike raw string literals in some languages, Java text blocks still process escape sequences (`\n`, `\t`, etc.). They are not completely "raw" — they only remove the need to escape quote characters within the content.

- **Can a text block be used as a constant in a `switch` case label?**
  - Yes, because text blocks are compile-time constants. You can use a text block in a `case` label: `case """hello""" -> ...`. However, this is unusual and may harm readability.

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
