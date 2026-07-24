# C# Strings and StringBuilder - Complete Interview Guide

---

## String Immutability

### Why Strings Are Immutable in C#

- In C#, every `string` object is **immutable** — once created, it cannot be changed
- This is a deliberate **design decision** by Microsoft, not a limitation
- The immutability provides several critical guarantees:

  - **Security**: Strings are used extensively for file paths, registry keys, SQL queries, URLs, and authentication tokens. If strings were mutable, a malicious method could modify a string after it had been validated but before it was used
  - **Thread safety**: Since the contents never change, multiple threads can read the same string concurrently without locks or synchronization
  - **Hash code caching**: The hash code of a string can be computed once and cached permanently. This makes dictionaries and hash sets extremely efficient with string keys
  - **Predictability**: Any method that receives a string reference can trust that the value will not change underneath it

```csharp
string original = "hello";
string modified = original.ToUpper();

// original is still "hello" — a NEW string was created
Console.WriteLine(original);   // "hello"
Console.WriteLine(modified);   // "HELLO"
Console.WriteLine(ReferenceEquals(original, modified)); // False
```

### Every String Operation Creates a New String

- Methods like `ToUpper()`, `ToLower()`, `Trim()`, `Replace()`, `Substring()` all return **new** string instances
- The original string remains untouched in memory
- This applies to operators too — the `+` operator on strings creates a brand new string

```csharp
string a = "hello";
string b = a.Trim();       // new string
string c = a + " world";  // new string
string d = a.Replace('l', 'L'); // new string

// All different objects on the heap
Console.WriteLine(a); // "hello"
Console.WriteLine(b); // "hello"  (same content, different object)
Console.WriteLine(c); // "hello world"
Console.WriteLine(d); // "heLLo"
```

### String Pooling and Interning

- The CLR maintains a special memory area called the **intern pool** (or string pool)
- When the same string literal appears multiple times in your code, the compiler and runtime ensure only **one copy** exists in memory
- All variables pointing to that literal share the **same reference**

```csharp
string x = "hello";
string y = "hello";

Console.WriteLine(ReferenceEquals(x, y)); // True — same object
Console.WriteLine(object.ReferenceEquals(x, y)); // True

// But dynamically constructed strings are NOT interned by default
string z = new string(new char[] { 'h', 'e', 'l', 'l', 'o' });
Console.WriteLine(ReferenceEquals(x, z)); // False
```

- You can manually intern a string using `string.Intern()`:

```csharp
string dynamicStr = new string(new char[] { 'h', 'e', 'l', 'l', 'o' });
string interned = string.Intern(dynamicStr);

Console.WriteLine(ReferenceEquals(x, interned)); // True — now it is in the pool
```

### Why String Concatenation in Loops Is Bad (O(n²))

- Each concatenation creates a **new** string, copies all characters from the old string plus the new characters into it, then the old string becomes garbage
- In a loop with N iterations, you create approximately N strings, each time copying up to N characters
- Total character copies: `1 + 2 + 3 + ... + N = N*(N+1)/2` — which is **O(n²)**
- For a loop of 10,000 iterations, that is ~50 million character copies

```csharp
// BAD — O(n²) time complexity
string result = "";
for (int i = 0; i < 10000; i++)
{
    result += i.ToString(); // each iteration allocates a new, larger string
}

// GOOD — O(n) amortized with StringBuilder
var sb = new System.Text.StringBuilder();
for (int i = 0; i < 10000; i++)
{
    sb.Append(i.ToString());
}
string result = sb.ToString();
```

### Memory Impact of String Immutability

- Every modification allocates a new object on the **managed heap**
- The garbage collector must eventually reclaim all those intermediate strings
- This creates **GC pressure** — more frequent garbage collection cycles
- In tight loops, this can cause significant pauses and throughput loss
- Interned strings live in the large object heap (LOH) if >= 85,000 bytes and are collected less frequently
- Small interned strings may live for the lifetime of the application domain

```csharp
// Memory snapshot during this code:
string s = "a";           // 1 string on heap
s += "b";                 // "a" becomes garbage, "ab" is new
s += "c";                 // "ab" becomes garbage, "abc" is new
// Two dead strings + one live string — all from 3 assignments
```

---

## StringBuilder

### What Is StringBuilder

- `System.Text.StringBuilder` is a **mutable** class for building strings efficiently
- It maintains an internal **array of characters** that you can modify in place
- It avoids creating new string objects for every operation
- Located in the `System.Text` namespace

```csharp
using System.Text;

var sb = new StringBuilder();
sb.Append("Hello");
sb.Append(" ");
sb.Append("World");

string result = sb.ToString(); // "Hello World"
// Only one string allocation when ToString() is called
```

### How StringBuilder Works Internally

- StringBuilder wraps an internal `char[]` buffer
- When you call `Append()`, characters are copied directly into this buffer
- No new string objects are created until you call `ToString()`
- The internal buffer starts with a default capacity and grows as needed

```csharp
// Simplified internal structure (conceptual):
internal class StringBuilder
{
    internal char[] m_ChunkChars;  // the current buffer
    internal int m_ChunkLength;    // how many chars are used
    internal int m_MaxCapacity;    // maximum allowed size
    internal StringBuilder m_ChunkPrevious; // linked list for large strings
}
```

### Default Capacity and Growth Strategy

- **Default capacity**: 16 characters
- **Maximum capacity**: `int.MaxValue` (2,147,483,647) by default
- When the buffer fills up, StringBuilder **doubles** its capacity (approximately)
- If the needed size is more than double, it grows to exactly what is needed
- Growth triggers a new `char[]` allocation and copies existing characters into it
- Very large strings (> ~85,000 chars) may end up on the Large Object Heap (LOH)

```csharp
var sb = new StringBuilder(); // capacity = 16
sb.Append(new string('x', 20)); // triggers reallocation — new capacity is at least 32
Console.WriteLine(sb.Capacity); // 32 (doubled from 16)
```

### StringBuilder vs String Concatenation Performance

| Scenario | string + | StringBuilder |
|---|---|---|
| 2-3 concatenations | Fast (JIT may optimize) | Slight overhead |
| 10+ concatenations | Noticeably slow | Fast |
| 100+ in a loop | Very slow (O(n²)) | Fast (O(n)) |
| Known final length | O(n²) | O(n) with pre-sized capacity |

- For **few** concatenations, the `+` operator is fine and may even be slightly faster due to compiler/JIT optimizations
- For **many** concatenations (especially in loops), StringBuilder is dramatically faster
- Benchmark example:

```csharp
var sw = Stopwatch.StartNew();
string s = "";
for (int i = 0; i < 100000; i++)
    s += "a";
sw.Stop();
Console.WriteLine($"String concat: {sw.ElapsedMilliseconds} ms");

sw.Restart();
var sb = new StringBuilder();
for (int i = 0; i < 100000; i++)
    sb.Append("a");
string s2 = sb.ToString();
sw.Stop();
Console.WriteLine($"StringBuilder: {sw.ElapsedMilliseconds} ms");

// Typical result: String concat takes 10-20x longer
```

### When to Use StringBuilder vs String Interpolation

- **Use string interpolation** (`$"..."`) when:
  - You have 1-5 values to embed in a string
  - The expression is simple and readable
  - Performance is not critical for that specific operation

- **Use StringBuilder** when:
  - You are building a string in a loop
  - You have many concatenations (10+)
  - You need to conditionally append parts
  - You know the approximate final size and can pre-allocate

```csharp
// Interpolation — clean and readable
string msg = $"Hello {name}, you have {count} items";

// StringBuilder — efficient for building
var sb = new StringBuilder();
sb.AppendLine("Items:");
foreach (var item in items)
{
    sb.AppendLine($"  - {item.Name}: {item.Quantity}");
}
string report = sb.ToString();
```

### Capacity vs Length

- **`Capacity`**: The size of the internal `char[]` buffer (can be larger than needed)
- **`Length`**: The number of characters actually stored in the StringBuilder
- You can **set** `Capacity` manually to pre-allocate space:

```csharp
var sb = new StringBuilder();
Console.WriteLine(sb.Capacity);   // 16 (default)
Console.WriteLine(sb.Length);     // 0 (nothing written yet)

sb.Append("Hello");
Console.WriteLine(sb.Capacity);   // 16
Console.WriteLine(sb.Length);     // 5

// Pre-allocating for known size is more efficient
var sb2 = new StringBuilder(1000); // capacity = 1000
// No reallocations needed for up to 1000 characters
```

- Setting `Capacity` to a value less than `Length` throws `ArgumentOutOfRangeException`
- Setting `Capacity` exactly to `Length` trims the internal buffer

### ToString() Method

- `ToString()` returns a **new string** containing the current contents of the StringBuilder
- It does NOT clear the StringBuilder — you can continue appending after calling `ToString()`
- In .NET Core 3.0+ and .NET 5+, there is an optimized `ToString(int startIndex, int length)` overload

```csharp
var sb = new StringBuilder();
sb.Append("Hello World");

string full = sb.ToString();      // "Hello World"
string partial = sb.ToString(6, 5); // "World" (start index, length)

Console.WriteLine(full);  // "Hello World"
Console.WriteLine(partial); // "World"
Console.WriteLine(sb.Length); // 11 — StringBuilder is unchanged
```

---

## String Operations

### string.Equals with StringComparison

- Always prefer the **overload** that accepts a `StringComparison` parameter
- This avoids ambiguity about whether comparison is case-sensitive, culture-sensitive, or ordinal

```csharp
// DANGEROUS — ambiguous, relies on overloads
bool result = str1.Equals(str2);

// EXPLICIT — always clear what you mean
bool result = str1.Equals(str2, StringComparison.Ordinal);
bool result = str1.Equals(str2, StringComparison.OrdinalIgnoreCase);
```

### StringComparison Types Explained

- **`StringComparison.Ordinal`**:
  - Compares raw Unicode code points
  - Case-sensitive
  - Fastest option
  - Best for: identifiers, file paths, protocol strings, any ASCII data

- **`StringComparison.OrdinalIgnoreCase`**:
  - Compares raw Unicode code points but ignores case
  - Faster than culture-aware comparisons
  - Best for: case-insensitive comparisons of identifiers, URLs, email addresses

- **`StringComparison.CurrentCulture`**:
  - Uses the current thread's culture for sorting rules
  - Culture-aware (e.g., "ß" equals "ss" in German)
  - Best for: displaying sorted lists to users

- **`StringComparison.InvariantCulture`**:
  - Uses a fixed, culture-invariant set of rules
  - Consistent across all machines regardless of locale
  - Best for: persisted data, cross-system communication

- **`StringComparison.Ordinal` vs `CurrentCulture` example:**

```csharp
// In Turkish, uppercase of 'i' is 'İ' (not 'I')
// In English, uppercase of 'i' is 'I'

string turkish = "TITLE";

// Ordinal — raw code point comparison
Console.WriteLine("title".Equals(turkish, StringComparison.Ordinal)); // False

// CurrentCulture — depends on locale
// In Turkish culture, "title" does NOT equal "TITLE"
// In English culture, "title" equals "TITLE"
Console.WriteLine("title".Equals(turkish, StringComparison.CurrentCulture)); // depends on culture

// InvariantCulture — same result everywhere
Console.WriteLine("title".Equals(turkish, StringComparison.InvariantCulture)); // False (case-sensitive)

// Case-insensitive options
Console.WriteLine("title".Equals(turkish, StringComparison.OrdinalIgnoreCase)); // True
```

### string.Contains, StartsWith, EndsWith

- All three have overloads accepting `StringComparison` and ` StringComparison` + `StringComparison` with `ReadOnlySpan<char>` in newer .NET
- Always pass an explicit `StringComparison` for clarity and correctness

```csharp
string input = "Hello World";

// StartsWith
bool starts = input.StartsWith("Hello", StringComparison.Ordinal); // true
bool startsIgnoresCase = input.StartsWith("hello", StringComparison.OrdinalIgnoreCase); // true

// EndsWith
bool ends = input.EndsWith("World", StringComparison.Ordinal); // true

// Contains
bool contains = input.Contains("llo Wo", StringComparison.Ordinal); // true

// .NET Core 2.1+ — Span-based overloads (no allocation for char/ReadOnlySpan search)
bool containsChar = input.Contains('W', StringComparison.Ordinal); // true
```

### string.Replace, string.Split, string.Substring

```csharp
string text = "Hello World Hello";

// Replace — returns new string
string replaced = text.Replace("Hello", "Hi"); // "Hi World Hi"
string replacedOnce = text.Replace("Hello", "Hi", 1); // "Hi World Hello" (count = 1)

// Replace with StringComparison (.NET 5+)
string replacedCaseInsensitive = text.Replace("hello", "Hi", StringComparison.OrdinalIgnoreCase);

// Split
string csv = "one,two,,four";
string[] parts = csv.Split(','); // ["one", "two", "", "four"]
string[] noEmpty = csv.Split(',', StringSplitOptions.RemoveEmptyEntries); // ["one", "two", "four"]

// Substring
string sub = "Hello World".Substring(6);    // "World"
string sub2 = "Hello World".Substring(0, 5); // "Hello"

// Substring with range operator (C# 8+)
string sub3 = "Hello World"[6..];     // "World"
string sub4 = "Hello World"[..5];     // "Hello"
```

### string.Join and string.Concat

```csharp
string[] words = { "Hello", "World", "From", "CSharp" };

// Join — combines array elements with a separator
string joined = string.Join(" ", words);      // "Hello World From CSharp"
string csvJoin = string.Join(",", words);     // "Hello,World,From,CSharp"

// Join with objects — calls ToString() on each
int[] numbers = { 1, 2, 3, 4, 5 };
string numStr = string.Join(" - ", numbers); // "1 - 2 - 3 - 4 - 5"

// Concat — combines strings without a separator
string concatenated = string.Concat(words);   // "HelloWorldFromCSharp"
string concatWithSeparator = string.Concat(words.Select(w => w + " ")); // "Hello World From CSharp "

// Concat with null handling
string result = string.Concat("Hello", null, "World"); // "HelloWorld"
```

### string.Format vs String Interpolation

```csharp
// string.Format — positional placeholders
string formatted = string.Format("Name: {0}, Age: {1}", name, age);

// String interpolation — same thing, cleaner syntax
string interpolated = $"Name: {name}, Age: {age}";

// Both compile to the same IL (interpolation becomes string.Format under the hood)

// Format specifiers
string price = string.Format("Price: {0:C2}", 42.5);   // "Price: $42.50"
string priceInterp = $"Price: {42.5:C2}";               // "Price: $42.50"

// Alignment
string aligned = string.Format("{0,-20} {1,10}", "Left", "Right");
// "Left                   Right"

string alignedInterp = $"{"Left",-20} {"Right",10}";
// "Left                   Right"
```

### string.IsNullOrEmpty vs string.IsNullOrWhiteSpace

```csharp
string empty = "";
string whitespace = "   ";
string nullStr = null;
string valid = "hello";

// IsNullOrEmpty — checks for null or empty ("")
Console.WriteLine(string.IsNullOrEmpty(nullStr));     // True
Console.WriteLine(string.IsNullOrEmpty(empty));       // True
Console.WriteLine(string.IsNullOrEmpty(whitespace));  // False
Console.WriteLine(string.IsNullOrEmpty(valid));       // False

// IsNullOrWhiteSpace — checks for null, empty, or whitespace only
Console.WriteLine(string.IsNullOrWhiteSpace(nullStr));     // True
Console.WriteLine(string.IsNullOrWhiteSpace(empty));       // True
Console.WriteLine(string.IsNullOrWhiteSpace(whitespace));  // True
Console.WriteLine(string.IsNullOrWhiteSpace(valid));       // False

// Performance note: both short-circuit (null check first, no allocation)
// Use IsNullOrEmpty for strict empty checks
// Use IsNullOrWhiteSpace when spaces/tabs should be treated as "empty"
```

---

## String Interpolation

### $"" Syntax (C# 6+)

- String interpolation was introduced in **C# 6** (Visual Studio 2015)
- The `$` prefix allows embedding expressions directly inside `{}` delimiters
- Much more readable than positional `string.Format`

```csharp
string name = "Alice";
int age = 30;

// Old way
string oldWay = string.Format("My name is {0} and I am {1} years old.", name, age);

// New way (C# 6+)
string newWay = $"My name is {name} and I am {age} years old.";

// Complex expressions
string complex = $"Next year I will be {age + 1} years old.";
string methodResult = $"Uppercase: {name.ToUpper()}";

// Conditional logic inside interpolation
string status = $"Status: {(age >= 18 ? "adult" : "minor")}";
```

### How Interpolation Compiles to string.Format

- The compiler translates `$"...{expr}..."` into `string.Format("...{0}...", expr)`
- This means there is a **boxing** cost for value types (int, double, etc.)
- The interpolation happens at runtime, not compile time
- Each `{}` expression is evaluated once and passed as an `object` argument

```csharp
// What you write:
string s = $"Hello {name}, count: {count}";

// What the compiler generates (conceptually):
string s = string.Format("Hello {0}, count: {1}", name, count);

// For value types, boxing occurs:
int x = 42;
string s2 = $"Value: {x}"; // x is boxed to object for string.Format
```

### Formatted Interpolation

- You can apply format specifiers directly inside interpolation expressions
- The format is `{expression:formatString}`

```csharp
double pi = Math.PI;

// Numeric formats
Console.WriteLine($"{pi:F2}");    // "3.14"
Console.WriteLine($"{pi:N2}");    // "3.14"
Console.WriteLine($"{pi:E2}");    // "3.14E+000"
Console.WriteLine($"{pi:P0}");    // "314%"
Console.WriteLine($"{1234567:C}"); // "$1,234,567.00" (culture-dependent)

// Date formats
DateTime now = DateTime.Now;
Console.WriteLine($"{now:d}");    // short date
Console.WriteLine($"{now:yyyy-MM-dd HH:mm:ss}"); // "2026-07-24 14:30:00"

// Alignment (width)
Console.WriteLine($"{"hello",10}");    // "     hello" (right-aligned, width 10)
Console.WriteLine($"{"hello",-10}");   // "hello     " (left-aligned, width 10)

// Combined alignment and format
Console.WriteLine($"{pi,10:F2}");      // "      3.14"
Console.WriteLine($"{"text",-10}|");   // "text      |"
```

### Interpolated Strings and IFormattable

- An interpolated string expression can produce either a `string` or an `IFormattable`
- This allows you to capture the structure of the interpolation without immediately formatting it
- Useful for logging frameworks, structured data, and localization

```csharp
// Capturing as IFormattable (useful for structured logging)
IFormattable message = $"User {userId} logged in from {ipAddress}";

// Convert to string when needed
string text = message.ToString(); // "User 123 logged in from 192.168.1.1"

// The interpolated string handler pattern
void Log(IFormattable message)
{
    // Can inspect the template and arguments before formatting
    Console.WriteLine($"[LOG] {message}");
}
```

### DefaultInterpolatedStringHandler (C# 10+)

- In **C# 10+**, interpolated strings use `DefaultInterpolatedStringHandler` by default
- This is more efficient than `string.Format` because it:
  - Avoids boxing of value types
  - Avoids creating intermediate arrays
  - Uses a `Span<char>` based approach when possible
  - Can use `stackalloc` for small strings

```csharp
// C# 10+ optimized path (simplified):
// The compiler generates:
var handler = new DefaultInterpolatedStringHandler(literalLength, formattedCount);
handler.AppendLiteral("Hello ");
handler.AppendFormatted(name);
handler.AppendLiteral(", count: ");
handler.AppendFormatted(count);
string result = handler.ToStringAndClear();

// Benefits:
// 1. No boxing for value types (count stays as int, formatted inline)
// 2. No string.Format overhead
// 3. Can benefit from Span<T> optimizations
// 4. Allocations minimized through stack buffers
```

---

## String Memory and Performance

### String Pooling

- String literals defined in code are **automatically pooled** by the CLR
- Two variables assigned the same literal share the same memory reference
- This is why `ReferenceEquals` returns `true` for identical literals

```csharp
string a = "hello";
string b = "hello";

// Same reference — pooled
Console.WriteLine(ReferenceEquals(a, b)); // True

// Dynamically constructed strings are NOT pooled
string c = new string(new char[] { 'h', 'e', 'l', 'l', 'o' });
Console.WriteLine(ReferenceEquals(a, c)); // False

// Concatenation of two literals at compile time produces one pooled literal
string d = "hel" + "lo"; // Compiler optimizes to "hello"
Console.WriteLine(ReferenceEquals(a, d)); // True

// But runtime concatenation does NOT pool
string e = "hel";
string f = e + "lo"; // Runtime concatenation — new string
Console.WriteLine(ReferenceEquals(a, f)); // False
```

### String Interning with RuntimeHelpers.Intern

- The interning pool is accessible via `string.Intern()` and `string.IsInterned()`
- Interned strings live for the lifetime of the AppDomain
- Useful when you have many duplicate strings from external sources (files, network, databases)

```csharp
// Manual interning
string fromFile = ReadFromFile(); // "frequently used value"
string interned = string.Intern(fromFile);

// Now the CLR checks the pool and returns the existing reference
// if the value already exists, or adds it to the pool

Console.WriteLine(string.IsInterned(interned)); // not null — it's in the pool

// Warning: Interned strings are NOT garbage collected
// Do NOT intern user-provided or unbounded strings
// The pool is limited and interned strings live forever
```

### StringBuilder Memory Allocation Strategy

- Default buffer: 16 chars
- When capacity is exceeded: new buffer = `max(oldCapacity * 2, oldCapacity + needed)`
- Very large appends may cause the buffer to grow to exactly the needed size
- Each reallocation copies the entire existing content to a new buffer

```csharp
var sb = new StringBuilder(); // capacity = 16
Console.WriteLine(sb.Capacity); // 16

sb.Append(new string('a', 16)); // fills the buffer
Console.WriteLine(sb.Capacity); // 16
Console.WriteLine(sb.Length);   // 16

sb.Append('b'); // triggers growth
Console.WriteLine(sb.Capacity); // 32 (doubled)
Console.WriteLine(sb.Length);   // 17

// Pre-allocation avoids repeated copying
var sb2 = new StringBuilder(50000); // one allocation, no resizing needed
```

### String vs char[] for Frequent Modifications

- If you need to modify individual characters frequently, use `char[]` or `Span<char>`
- `string` requires a full copy for any modification
- Convert to `string` only when you need to pass it to an API that requires string

```csharp
// BAD: modifying string in place is impossible
string s = "hello";
// s[0] = 'H'; // Compile error — string is read-only

// GOOD: use char array for character-level modifications
char[] chars = "hello".ToCharArray();
chars[0] = 'H';
string modified = new string(chars); // "Hello"

// BETTER (.NET Core 2.1+): use Span<char>
Span<char> span = "hello".ToCharArray();
span[0] = 'H';
string result = span.ToString(); // "Hello"
```

### String Pool Size Limits

- The intern pool has **no explicit limit** on the number of strings
- However, interned strings are **never garbage collected**
- Each interned string consumes memory permanently for the AppDomain lifetime
- The pool is stored in the managed heap and can grow into the Large Object Heap
- **Do not intern unbounded strings** (user input, file contents, random data)

```csharp
// DANGEROUS: interning unbounded data
for (int i = 0; i < 1000000; i++)
{
    string.Intern(ReadFromDatabase(i)); // Memory leak — never reclaimed
}

// SAFE: interning bounded, repeated strings
string[] commonStatuses = { "Active", "Inactive", "Pending", "Deleted" };
for (int i = 0; i < commonStatuses.Length; i++)
{
    commonStatuses[i] = string.Intern(commonStatuses[i]);
}
```

### Memory Optimization Tips

1. **Pre-size StringBuilder** when you know the approximate output length
2. **Use `string.Concat`** for joining a small, known number of strings
3. **Use `string.Join`** for arrays/collections — it calculates the total size and does one allocation
4. **Avoid string concatenation in loops** — always use StringBuilder
5. **Use `ReadOnlySpan<char>`** to avoid substring allocations
6. **Use `string.Create`** (.NET Core 2.1+) to build strings without intermediate allocations
7. **Cache frequently used computed strings** instead of recomputing
8. **Use `stackalloc`** with `DefaultInterpolatedStringHandler` for small temporary strings

```csharp
// string.Create — allocate and fill in one step
string result = string.Create(5, 'a', (span, ch) =>
{
    span.Fill(ch);
}); // "aaaaa"

// ReadOnlySpan — substring without allocation
ReadOnlySpan<char> span = "Hello World".AsSpan();
ReadOnlySpan<char> sub = span.Slice(6, 5); // "World" — no new string allocated
```

---

## Common Mistakes

### String Concatenation in Loops Without StringBuilder

```csharp
// MISTAKE
string result = "";
foreach (var item in largeCollection)
{
    result += item.ToString(); // O(n²) — creates a new string each iteration
}

// FIX
var sb = new StringBuilder();
foreach (var item in largeCollection)
{
    sb.Append(item.ToString());
}
string result = sb.ToString();
```

### Ignoring StringComparison in Comparisons

```csharp
// MISTAKE — ambiguous, locale-dependent, platform-dependent
bool eq = str1 == str2;
bool eq2 = str1.Equals(str2);

// FIX — always explicit
bool eq = str1.Equals(str2, StringComparison.Ordinal);
bool eq2 = str1.Equals(str2, StringComparison.OrdinalIgnoreCase);
bool eq3 = string.Equals(str1, str2, StringComparison.Ordinal);
```

### Using == vs Equals for Culture-Sensitive Comparison

```csharp
// == operator always uses Ordinal comparison for strings
// Equals(string) without StringComparison also uses Ordinal
// Neither is appropriate for culture-sensitive sorting or display

// MISTAKE: using == when you need culture awareness
if (name == "Müller") // might fail in cultures where "ü" != "u"

// FIX: use culture-aware comparison
if (name.Equals("Muller", StringComparison.CurrentCulture))
// or better:
if (string.Equals(name, "Müller", StringComparison.InvariantCulture))
```

### Not Pre-sizing StringBuilder Capacity

```csharp
// MISTAKE — multiple reallocations
var sb = new StringBuilder(); // starts at 16
for (int i = 0; i < 10000; i++)
{
    sb.Append("data"); // triggers ~13 reallocations
}

// FIX — pre-allocate if you know the size
var sb = new StringBuilder(40000); // 10000 items * 4 chars each
for (int i = 0; i < 10000; i++)
{
    sb.Append("data"); // no reallocations
}
```

### Forgetting String Is a Reference Type with Value Equality

```csharp
// string is a reference type (class), but == and Equals compare VALUES
string a = "hello";
string b = "hello";

Console.WriteLine(a == b);           // True (value equality)
Console.WriteLine(a.Equals(b));     // True (value equality)
Console.WriteLine(ReferenceEquals(a, b)); // True (same interned reference)

// But this is not always the case:
string c = new string(new char[] { 'h', 'e', 'l', 'l', 'o' });
Console.WriteLine(a == c);           // True (value equality)
Console.WriteLine(ReferenceEquals(a, c)); // False (different objects!)

// When implementing == for your own types, string's behavior is the model
// for value semantics on reference types
```

### Null String Operations (NullReferenceException)

```csharp
string s = null;

// These are fine — they handle null gracefully
Console.WriteLine(string.IsNullOrEmpty(s)); // True
Console.WriteLine(s?.Length);               // null (no exception)
Console.WriteLine(s ?? "default");          // "default"

// These throw NullReferenceException
try
{
    Console.WriteLine(s.Length);           // NullReferenceException
    Console.WriteLine(s.ToUpper());       // NullReferenceException
    Console.WriteLine(s.Contains("a"));   // NullReferenceException
}
catch (NullReferenceException)
{
    // Expected
}

// The null-conditional operator is your friend
int? len = s?.Length;             // null
bool? hasA = s?.Contains('a');   // null
string upper = s?.ToUpper();     // null
```

---

## Real-World Scenarios

### Building SQL Queries Dynamically

```csharp
// StringBuilder is ideal for dynamic SQL construction
public string BuildWhereClause(List<Filter> filters)
{
    var sb = new StringBuilder("WHERE 1=1");

    foreach (var filter in filters)
    {
        switch (filter.Operator)
        {
            case "equals":
                sb.Append($" AND {filter.Column} = @{filter.Column}");
                break;
            case "contains":
                sb.Append($" AND {filter.Column} LIKE @{filter.Column}");
                break;
            case "greaterThan":
                sb.Append($" AND {filter.Column} > @{filter.Column}");
                break;
            case "between":
                sb.Append($" AND {filter.Column} BETWEEN @{filter.Column}Start AND @{filter.Column}End");
                break;
        }
    }

    return sb.ToString();
}

// Pre-sizing for performance:
var sb = new StringBuilder(256); // reasonable estimate for WHERE clause
```

### Log Message Construction

```csharp
// StringBuilder for building structured log entries
public string FormatLogEntry(LogEntry entry)
{
    var sb = new StringBuilder(128);

    sb.Append('[');
    sb.Append(entry.Timestamp.ToString("yyyy-MM-dd HH:mm:ss.fff"));
    sb.Append("] [");
    sb.Append(entry.Level);
    sb.Append("] [");
    sb.Append(entry.Source);
    sb.Append("] ");

    if (entry.Exception != null)
    {
        sb.Append(entry.Exception.GetType().Name);
        sb.Append(": ");
        sb.Append(entry.Exception.Message);
        sb.Append(" | Stack: ");
        sb.Append(entry.Exception.StackTrace);
    }
    else
    {
        sb.Append(entry.Message);
    }

    return sb.ToString();
}

// For simple one-off log messages, string interpolation is fine:
_logger.LogInformation($"[{DateTime.Now:HH:mm:ss}] User {userId} performed {action}");
```

### JSON/String Serialization Concatenation

```csharp
// StringBuilder for manual JSON-like construction
public string BuildJsonObject(Dictionary<string, object> properties)
{
    var sb = new StringBuilder();
    sb.Append('{');
    bool first = true;

    foreach (var kvp in properties)
    {
        if (!first) sb.Append(',');
        first = false;

        sb.Append('"');
        sb.Append(kvp.Key);
        sb.Append("\":");

        switch (kvp.Value)
        {
            case string s:
                sb.Append('"');
                sb.Append(s);
                sb.Append('"');
                break;
            case bool b:
                sb.Append(b ? "true" : "false");
                break;
            case null:
                sb.Append("null");
                break;
            default:
                sb.Append(kvp.Value.ToString());
                break;
        }
    }

    sb.Append('}');
    return sb.ToString();
}

// Note: always prefer a proper JSON library (System.Text.Json, Newtonsoft.Json)
// over manual JSON construction
```

### CSV Generation from Large Datasets

```csharp
// StringBuilder with pre-allocation for CSV export
public string GenerateCsv(IEnumerable<DataRow> rows, string[] headers)
{
    // Estimate: ~20 chars per field * 5 fields * row count
    var estimatedSize = headers.Length * 20 * rows.Count();
    var sb = new StringBuilder(estimatedSize);

    // Header row
    sb.AppendLine(string.Join(",", headers));

    // Data rows
    foreach (var row in rows)
    {
        for (int i = 0; i < headers.Length; i++)
        {
            if (i > 0) sb.Append(',');

            string value = row[headers[i]]?.ToString() ?? "";

            // Escape CSV fields that contain commas, quotes, or newlines
            if (value.Contains(',') || value.Contains('"') || value.Contains('\n'))
            {
                sb.Append('"');
                sb.Append(value.Replace("\"", "\"\""));
                sb.Append('"');
            }
            else
            {
                sb.Append(value);
            }
        }
        sb.AppendLine();
    }

    return sb.ToString();
}
```

---

## Interview Questions

### Question 1: Why are strings immutable in C#?

**Answer:**
Strings are immutable by design for several important reasons:
- **Security**: Strings are used for file paths, SQL queries, URLs, and authentication tokens. Immutability prevents a malicious code path from altering a string after validation
- **Thread safety**: Since the content never changes, multiple threads can safely read the same string without synchronization
- **Hash code caching**: The hash code can be computed once and stored permanently, making strings highly efficient as dictionary keys
- **Reference sharing**: The CLR can safely share string references across the application since no one can modify the value
- **Predictability**: Methods that accept string parameters are guaranteed that the value will not change during execution

---

### Question 2: What happens when you concatenate strings in a loop?

**Answer:**
Each concatenation creates a **new string** on the heap by copying all characters from the original string plus the new characters. The old string becomes garbage. In a loop of N iterations, this results in approximately `N*(N+1)/2` character copies, which is **O(n²)** time complexity. This causes significant memory allocation and GC pressure. The solution is to use `StringBuilder`, which maintains a mutable character buffer and achieves **O(n)** amortized time complexity.

---

### Question 3: Explain the difference between `StringComparison.Ordinal`, `CurrentCulture`, and `InvariantCulture`.

**Answer:**
- **`Ordinal`**: Compares raw Unicode code points. Fastest, most predictable. Best for identifiers, file paths, and protocol strings. Case-sensitive
- **`CurrentCulture`**: Uses the current thread's culture rules for comparison. Culture-aware (e.g., German "ß" equals "ss"). Best for displaying sorted lists to users
- **`InvariantCulture`**: Uses a fixed, culture-independent set of rules. Consistent across all machines. Best for persisted data and cross-system communication

The Turkish "I" problem is a classic example: in Turkish, uppercase of 'i' is 'İ', so `Ordinal` and `CurrentCulture` can give different results for the same comparison.

---

### Question 4: When should you use StringBuilder vs string interpolation?

**Answer:**
Use **string interpolation** when:
- You have 1-5 expressions to embed in a string
- Readability is important
- Performance is not critical for that operation

Use **StringBuilder** when:
- You are building a string in a loop
- You have 10+ concatenations
- You need to conditionally append parts
- You know the final size and can pre-allocate capacity

The rule of thumb: 2-3 concatenations with `+` or `$""` is fine. Beyond that, switch to StringBuilder.

---

### Question 5: What is string pooling and how does it work?

**Answer:**
String pooling is a CLR optimization where identical string literals are stored in a single memory location called the **intern pool**. When the compiler encounters the same literal multiple times, all variables point to the same reference. This saves memory and improves comparison performance. The pool is accessible via `string.Intern()` for manual interning. Interned strings are **never garbage collected** — they live for the lifetime of the AppDomain. This is why you should never intern unbounded or user-provided strings.

---

### Question 6: How does StringBuilder work internally?

**Answer:**
StringBuilder maintains an internal `char[]` buffer with a default capacity of 16 characters. When you call `Append()`, characters are written directly into this buffer. When the buffer is full, a new, larger buffer is allocated (typically doubling the capacity) and existing characters are copied. The final string is created by `ToString()`, which copies the buffer contents into a new `string` object. This approach avoids creating intermediate string objects and achieves O(n) amortized performance for sequential appends.

---

### Question 7: What is the difference between `string.IsNullOrEmpty` and `string.IsNullOrWhiteSpace`?

**Answer:**
- `string.IsNullOrEmpty(s)` returns `true` if `s` is `null` or `""` (empty string)
- `string.IsNullOrWhiteSpace(s)` returns `true` if `s` is `null`, `""`, or contains only whitespace characters (spaces, tabs, newlines)

Both are null-safe (they do not throw on null input) and both short-circuit their evaluation. Use `IsNullOrEmpty` for strict empty checks. Use `IsNullOrWhiteSpace` when whitespace should be treated as "empty" — which is the case for most user input validation scenarios.

---

### Question 8: What is the difference between `string.Format` and string interpolation (`$"..."`)?

**Answer:**
Functionally they are identical — the compiler translates `$"..."` into `string.Format(...)` calls. The difference is **syntax and readability**. Interpolation is cleaner and less error-prone (no need to count positional placeholders). In C# 10+, interpolation uses `DefaultInterpolatedStringHandler` which is more efficient than `string.Format` — it avoids boxing of value types and uses span-based operations for better performance.

---

### Question 9: What is the `StringComparison.Ordinal` vs `StringComparison.OrdinalIgnoreCase` performance difference?

**Answer:**
Both are very fast compared to culture-aware comparisons. `Ordinal` compares raw Unicode values and is slightly faster because it does not need to apply case folding. `OrdinalIgnoreCase` performs case-insensitive comparison by applying simple Unicode case folding rules (upper/lower case mapping). For most practical purposes, the difference is negligible — both are O(n) and run in nanoseconds for typical string lengths. The choice should be based on **semantics**, not performance.

---

### Question 10: How do you pre-allocate StringBuilder capacity?

**Answer:**
Pass the expected final size to the constructor: `new StringBuilder(estimatedSize)`. This prevents reallocations and copying during appending. A good rule of thumb: estimate the total character count of your output. For example, if you are joining 1000 items each averaging 20 characters, use `new StringBuilder(20000)`. You can also set `sb.Capacity = value` after construction. Setting capacity below current `Length` throws `ArgumentOutOfRangeException`.

---

### Question 11: Can you modify individual characters in a string?

**Answer:**
No, strings are immutable. You cannot modify individual characters in a `string` object. If you need character-level modification, use a `char[]` array, a `Span<char>`, or `Memory<char>`. Convert back to `string` when done. In C# 8+, you can use range/span syntax to work with substrings without allocations using `ReadOnlySpan<char>`.

```csharp
char[] chars = "hello".ToCharArray();
chars[0] = 'H';
string result = new string(chars); // "Hello"
```

---

### Question 12: What happens when you compare two interned strings with `ReferenceEquals`?

**Answer:**
If both strings are interned literals with the same value, `ReferenceEquals` returns `true` because they point to the same memory location in the intern pool. However, if one or both strings were dynamically constructed (e.g., via concatenation at runtime, `new string()`, or reading from a file), they will be different objects even if they have the same value, and `ReferenceEquals` returns `false`. This is why you should never use `ReferenceEquals` for string comparison — always use `Equals` with `StringComparison` or the `==` operator.

---

### Question 13: Explain the memory impact of string concatenation in a loop with 10,000 iterations.

**Answer:**
Each concatenation creates a new string that copies all previous characters plus the new ones. In 10,000 iterations:
- Approximately 10,000 string objects are created (the old ones become garbage)
- Total character copies: ~50 million (1+2+3+...+10000)
- Memory allocated: roughly `50M * 2 bytes = ~100MB` of temporary allocations
- The garbage collector must reclaim all those intermediate strings
- With StringBuilder, only ~10,000-20,000 characters are written to a buffer that grows a handful of times, using a few KB total

---

### Question 14: When would you use `string.Concat` vs `string.Join` vs `StringBuilder`?

**Answer:**
- **`string.Concat`**: Combines 2-4 strings without a separator. Efficient for small, known-count concatenations
- **`string.Join`**: Combines an array/collection of strings with a separator. Calculates total size upfront and does a single allocation. Best for arrays
- **`StringBuilder`**: Best when you are building a string incrementally (loops, conditional appends, unknown count). Allows appending one piece at a time

```csharp
string.Concat(a, b, c);              // "abc"
string.Join(", ", array);            // "a, b, c"
new StringBuilder().Append(a).Append(b).ToString(); // "ab" (for dynamic building)
```

---

### Question 15: What is `DefaultInterpolatedStringHandler` and why was it introduced in C# 10?

**Answer:**
`DefaultInterpolatedStringHandler` is the default handler for interpolated strings in C# 10+. It replaces the older approach of compiling interpolated strings to `string.Format` calls. Benefits include:
- **No boxing**: Value types like `int` and `double` are formatted directly without boxing to `object`
- **Span-based**: Uses `Span<char>` and `stackalloc` for temporary buffers, reducing heap allocations
- **Customization**: You can create custom handlers to control how interpolated strings are processed (e.g., for structured logging)
- **Performance**: Benchmarks show 2-5x faster interpolation compared to `string.Format` in tight loops

---

### Question 16: What are the common pitfalls with string comparison in C#?

**Answer:**
1. **Using `==` without understanding it**: `==` on strings uses ordinal comparison, which is not culture-aware
2. **Ignoring `StringComparison`**: `string.Equals(a, b)` uses ordinal comparison by default — this may not be what you want for user-facing text
3. **The Turkish I problem**: `OrdinalIgnoreCase` treats 'I' and 'i' as equivalent everywhere, but in Turkish culture 'I' uppercases to 'İ' not 'I'
4. **Using culture comparison for identifiers**: Protocol strings, file paths, and identifiers should always use `Ordinal` or `OrdinalIgnoreCase`
5. **Forgetting null handling**: Always check for null before comparison, or use `string.Equals` which handles nulls gracefully

---

### Question 17: How does StringBuilder handle very large strings (> 85,000 characters)?

**Answer:**
When StringBuilder's internal buffer exceeds approximately 85,000 characters, the new buffer may be allocated on the **Large Object Heap (LOH)**. LOH collections occur less frequently than small object heap collections (only during Gen 2 collections). This means very large StringBuilder buffers may remain in memory longer than necessary. After calling `ToString()`, the StringBuilder retains its internal buffer — it does not release the memory. If you are building extremely large strings, consider whether you can stream the output instead of building it entirely in memory.

---

### Question 18: Write code to demonstrate string pooling and interning.

```csharp
using System;
using System.Runtime.CompilerServices;

class Program
{
    static void Main()
    {
        // Pooling demonstration
        string a = "hello";
        string b = "hello";
        Console.WriteLine(ReferenceEquals(a, b)); // True — same pool reference

        string c = new string(new char[] { 'h', 'e', 'l', 'l', 'o' });
        Console.WriteLine(ReferenceEquals(a, c)); // False — not pooled

        // Manual interning
        string d = string.Intern(c);
        Console.WriteLine(ReferenceEquals(a, d)); // True — now in pool

        // IsInterned check
        string e = "not pooled by code";
        Console.WriteLine(string.IsInterned(e)); // Returns "not pooled by code" (it's already pooled as a literal)

        string f = new string(new char[] { 'n', 'o', 't', ' ', 'p', 'o', 'o', 'l', 'e', 'd' });
        Console.WriteLine(string.IsInterned(f)); // null — not in pool
    }
}
```

---

### Question 19: What is the difference between `string.Create` and `new string()`?

**Answer:**
- `new string(char[])` copies the array contents into a new string
- `string.Create(int length, T state, SpanAction<char, T> action)` allows you to **fill** a pre-allocated string using a span-based callback, avoiding the intermediate `char[]` allocation

```csharp
// Traditional approach
char[] buffer = new char[10];
for (int i = 0; i < 10; i++) buffer[i] = (char)('a' + i);
string result = new string(buffer);

// string.Create — no intermediate array
string result2 = string.Create(10, 0, (span, state) =>
{
    for (int i = 0; i < span.Length; i++)
        span[i] = (char)('a' + i);
});
```

`string.Create` is more efficient because it avoids allocating and then copying from a `char[]` — the string's internal buffer is filled directly.

---

### Question 20: Explain the concept of string value equality vs reference equality.

**Answer:**
In C#, `string` is a **reference type** (it inherits from `object`), but it overrides `Equals()` and `==` to perform **value equality** — comparing the character contents rather than memory addresses. This means two different string objects with the same characters will be considered equal by `==` and `.Equals()`. However, `object.ReferenceEquals()` and `System.Runtime.CompilerServices.RuntimeHelpers.GetHashCode()` will distinguish them as different objects. This behavior was a deliberate design choice to make strings behave intuitively, despite being reference types.

