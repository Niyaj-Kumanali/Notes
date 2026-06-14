# Java Date and Time API (JSR 310)

---

## Overview

- **Definition**
  - The Java 8 Date and Time API (JSR 310, led by Stephen Colebourne, author of Joda-Time) is a comprehensive, immutable, thread-safe replacement for the legacy `java.util.Date`, `java.util.Calendar`, and `java.text.SimpleDateFormat` classes.
  - It is organized around clear, domain-specific types — date-only (`LocalDate`), time-only (`LocalTime`), date+time without zone (`LocalDateTime`), instant on timeline (`Instant`), and zoned date+time (`ZonedDateTime`) — instead of the legacy approach where `Date` represented multiple concepts ambiguously.
  - The API is ISO-8601 compliant by default, uses immutable value objects for all core types, and separates human-readable time from machine time.

- **Why It Exists**
  - The legacy `java.util.Date` had fundamental design flaws: it was mutable (every setter changed the object in place), had confusing year/month offsets (year 1900-based, month 0-indexed), and combined date and time in a single class with no timezone concept until `Calendar` was introduced.
  - `SimpleDateFormat` was not thread-safe — sharing a formatter between threads caused corrupt output or `ArrayIndexOutOfBoundsException` because the internal `Calendar` instance was mutated during formatting.
  - The calendar system was hardcoded to Gregorian — extending to other calendar systems (Hijri, Minguo, Japanese) required subclassing `Calendar`, which was error-prone and rarely done correctly.
  - The API was verbose: creating a simple date required multiple lines of boilerplate with `Calendar.getInstance()`, setters for each field, and manual conversion.
  - JSR 310 solves all of these: immutability (every operation returns a new instance), thread-safety (no shared mutable state), ISO-8601 compliance, and full support for alternate calendar systems via `java.time.chrono`.

- **Key Concepts**
  - **Immutability:** All core classes (`LocalDate`, `LocalTime`, `LocalDateTime`, `ZonedDateTime`, `Instant`, `Duration`, `Period`) are immutable and thread-safe. Methods like `plusDays()`, `withMonth()`, `atTime()` return new instances — the original is never modified.
  - **Separation of Concerns:** Date-only (`LocalDate`), time-only (`LocalTime`), date+time without zone (`LocalDateTime`), machine timestamp (`Instant`), date+time with zone (`ZonedDateTime`, `OffsetDateTime`, `OffsetTime`).
  - **Fluent API:** Method chaining is idiomatic — `date.plusDays(5).minusMonths(1).withDayOfMonth(1)`.
  - **ISO-8601 Default:** The `toString()` and `parse()` methods use ISO-8601 format by default: `2025-06-13`, `14:30:00`, `2025-06-13T14:30:00+05:30`.
  - **Null-hostility:** The API rejects null parameters with `NullPointerException` — there is no `setLenient()` equivalent.

---

## Core Concepts

### LocalDate

- Represents a date without time or timezone (year, month, day). Used for birthdays, holidays, due dates — any scenario where time and zone are irrelevant.
- Range: `-999999999-01-01` to `+999999999-12-31`.
- Key methods: `now()`, `of(int year, Month month, int dayOfMonth)`, `parse(CharSequence)`, `plusDays(long)`, `minusMonths(long)`, `with(TemporalAdjuster)`, `getDayOfWeek()`, `getMonth()`, `isLeapYear()`, `lengthOfMonth()`, `isBefore(LocalDate)`, `isAfter(LocalDate)`.

```java
// Creation
LocalDate today = LocalDate.now();
LocalDate independenceDay = LocalDate.of(1947, Month.AUGUST, 15);
LocalDate parsed = LocalDate.parse("2025-06-13"); // ISO-8601

// Manipulation — returns new instances
LocalDate nextWeek = today.plusDays(7);
LocalDate firstOfMonth = today.withDayOfMonth(1);
LocalDate sameDayNextMonth = today.plusMonths(1);

// Queries
DayOfWeek dayOfWeek = today.getDayOfWeek(); // FRIDAY
Month month = today.getMonth();             // JUNE
int dayOfYear = today.getDayOfYear();
boolean leap = today.isLeapYear();

// Comparison
boolean isBefore = independenceDay.isBefore(today); // true
long daysBetween = ChronoUnit.DAYS.between(independenceDay, today);
```

### LocalTime

- Represents a time without date or timezone (hour, minute, second, nanosecond). Used for opening hours, train schedules, daily recurring events.
- Precision: nanosecond (9 decimal places).
- Key methods: `now()`, `of(int hour, int minute)`, `of(int hour, int minute, int second)`, `parse(CharSequence)`, `plusHours(long)`, `minusMinutes(long)`, `withHour(int)`, `getHour()`, `isBefore(LocalTime)`, `isAfter(LocalTime)`.

```java
LocalTime now = LocalTime.now();
LocalTime noon = LocalTime.of(12, 0);
LocalTime parsed = LocalTime.parse("14:30:00");

// Business hours check
LocalTime open = LocalTime.of(9, 0);
LocalTime close = LocalTime.of(17, 0);
boolean isOpen = !now.isBefore(open) && !now.isAfter(close);

// Truncation
LocalTime toHour = now.truncatedTo(ChronoUnit.HOURS);

// Max/min values
LocalTime midnight = LocalTime.MIDNIGHT;      // 00:00
LocalTime noonConst = LocalTime.NOON;          // 12:00
LocalTime max = LocalTime.MAX;                 // 23:59:59.999999999
LocalTime min = LocalTime.MIN;                 // 00:00
```

### LocalDateTime

- Combines `LocalDate` and `LocalTime` but carries no timezone information. This is the most frequently misused type — developers use it when they need `ZonedDateTime` (e.g., store a global event time).
- Not an instant on the timeline — `2025-06-13T14:30:00` is ambiguous without a zone (it could be 14:30 in New York or 14:30 in Tokyo, which are 13 hours apart).
- Use `LocalDateTime` when the timezone is determined by the context (e.g., "the store closes at 22:00" meaning local time wherever the store is).

```java
// Creation
LocalDateTime now = LocalDateTime.now();
LocalDateTime specific = LocalDateTime.of(2025, Month.JUNE, 13, 14, 30);
LocalDateTime parsed = LocalDateTime.parse("2025-06-13T14:30:00");

// Combine date and time
LocalDate date = LocalDate.of(2025, 6, 13);
LocalTime time = LocalTime.of(14, 30);
LocalDateTime combined = LocalDateTime.of(date, time);

// Convert to zoned (zone is critical!)
ZonedDateTime zoned = combined.atZone(ZoneId.of("Asia/Kolkata"));
Instant instant = zoned.toInstant(); // This is the unambiguous timeline point
```

### ZonedDateTime

- A full date-time with timezone, handling DST transitions and zone offsets. This is the type to use for storing or transmitting global timestamps.
- Uses `ZoneId` to identify the timezone rules (e.g., `Europe/London`, `America/New_York`, `Asia/Tokyo`) and `ZoneOffset` for the UTC offset.
- Handles DST gaps (spring-forward: 1:59 AM → 3:00 AM) and overlaps (fall-back: 2:00 AM occurs twice).

```java
// Creation
ZonedDateTime now = ZonedDateTime.now(); // System default zone
ZonedDateTime inKolkata = ZonedDateTime.now(ZoneId.of("Asia/Kolkata"));
ZonedDateTime parsed = ZonedDateTime.parse("2025-06-13T14:30:00+05:30[Asia/Kolkata]");

// DST-safe operations
ZonedDateTime meeting = ZonedDateTime.of(
    LocalDate.of(2025, Month.MARCH, 9),
    LocalTime.of(10, 0),
    ZoneId.of("America/New_York")
);
// March 9, 2025 is spring-forward day in US — 2:00 AM jumps to 3:00 AM
// ZonedDateTime handles this: 10:00 AM exists normally

// Conversion between zones
ZonedDateTime inLondon = meeting.withZoneSameInstant(ZoneId.of("Europe/London"));

// DST overlap resolution
// On fall-back day, 1:30 AM occurs twice. Use withEarlierOffsetAtOverlap() / withLaterOffsetAtOverlap()
ZonedDateTime ambiguous = ZonedDateTime.of(
    LocalDateTime.of(2025, Month.NOVEMBER, 2, 1, 30),
    ZoneId.of("America/New_York")
);
// By default, uses the earlier offset (EDT, -04:00)
ZonedDateTime laterOffset = ambiguous.withLaterOffsetAtOverlap();
```

### Instant

- A machine-readable timestamp: the number of seconds and nanoseconds since the Unix epoch (1970-01-01T00:00:00Z).
- Represents a single point on the timeline, independent of human timezone concepts.
- The fundamental building block for inter-system communication, logging, database timestamps (UTC).

```java
// Current instant in UTC
Instant now = Instant.now();

// From epoch
Instant epoch = Instant.EPOCH;                     // 1970-01-01T00:00:00Z
Instant fromEpochSecond = Instant.ofEpochSecond(1_700_000_000L); // ~2023-11-14

// Conversion to/from other types
ZonedDateTime zdt = now.atZone(ZoneId.of("UTC"));
LocalDateTime ldt = LocalDateTime.ofInstant(now, ZoneId.systemDefault());

// Comparison
boolean isAfter = now.isAfter(fromEpochSecond);

// Truncation
Instant toMillis = now.truncatedTo(ChronoUnit.MILLIS);

// Arithmetic
Instant later = now.plusSeconds(3600);
Instant earlier = now.minus(1, ChronoUnit.DAYS);
```

### Duration & Period

- `Duration` measures time-based amounts (hours, minutes, seconds, nanoseconds) — used for machine time.
- `Period` measures date-based amounts (years, months, days) — used for human time.
- Both are immutable, thread-safe, and support arithmetic.

```java
// Duration: machine time
Duration fiveHours = Duration.ofHours(5);
Duration twoMinutes = Duration.ofMinutes(2);
Duration fromSeconds = Duration.ofSeconds(90, 500_000_000); // 90.5 seconds

Instant start = Instant.now();
Thread.sleep(100); // simulate work
Instant end = Instant.now();
Duration elapsed = Duration.between(start, end);
long millis = elapsed.toMillis();

// Period: human time
Period oneYear = Period.ofYears(1);
Period threeMonths = Period.ofMonths(3);
Period twoWeeks = Period.ofWeeks(2);

LocalDate today = LocalDate.now();
LocalDate future = today.plus(oneYear).plus(threeMonths);
Period between = Period.between(today, future);
int years = between.getYears();   // 1
int months = between.getMonths(); // 3
int days = between.getDays();     // 0

// Duration vs Period — critical distinction
// Duration is based on seconds (24-hours is always 86400 seconds)
Duration dayDuration = Duration.ofDays(1); // 86400 seconds — ignores DST
// Period is based on calendar days
Period dayPeriod = Period.ofDays(1);        // 1 calendar day — respects DST transitions
```

### DateTimeFormatter

- The immutable, thread-safe replacement for `SimpleDateFormat`. Built-in ISO formatters, localized formatters, and custom patterns.
- Predefined formatters: `ISO_LOCAL_DATE`, `ISO_LOCAL_TIME`, `ISO_LOCAL_DATE_TIME`, `ISO_ZONED_DATE_TIME`, `ISO_INSTANT`, `ISO_OFFSET_DATE_TIME`.
- Formatting and parsing are thread-safe — a single formatter instance can be shared across threads.

```java
// Built-in ISO formatters
String dateStr = LocalDate.now().format(DateTimeFormatter.ISO_LOCAL_DATE);
String zdtStr = ZonedDateTime.now().format(DateTimeFormatter.ISO_ZONED_DATE_TIME);

// Custom patterns
DateTimeFormatter custom = DateTimeFormatter.ofPattern("dd/MM/yyyy HH:mm:ss");
String formatted = LocalDateTime.now().format(custom);

// Parsing
LocalDate parsed = LocalDate.parse("13/06/2025", DateTimeFormatter.ofPattern("dd/MM/yyyy"));

// Locale-specific
DateTimeFormatter italian = DateTimeFormatter.ofPattern("d MMMM yyyy", Locale.ITALIAN);
String italianDate = LocalDate.now().format(italian); // "13 giugno 2025"

// Thread-safe static constant
public static final DateTimeFormatter ORDER_DATE_FORMAT =
    DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSSZ");

// Optional section in pattern (brackets)
DateTimeFormatter flexible = DateTimeFormatter.ofPattern("yyyy-MM-dd['T'HH:mm:ss]");
LocalDateTime parsedWithOrWithoutTime = LocalDateTime.parse("2025-06-13T14:30:00", flexible);
LocalDate parsedDateOnly = LocalDate.parse("2025-06-13", flexible);
```

### TemporalAdjusters

- Predefined adjusters for common date manipulations: first day of month, next Monday, last day of year, etc.
- `with(TemporalAdjuster)` applies the adjuster to a temporal object and returns a new one.

```java
LocalDate today = LocalDate.now();

// Predefined adjusters
LocalDate firstOfMonth = today.with(TemporalAdjusters.firstDayOfMonth());
LocalDate lastOfMonth = today.with(TemporalAdjusters.lastDayOfMonth());
LocalDate nextMonday = today.with(TemporalAdjusters.next(DayOfWeek.MONDAY));
LocalDate previousFriday = today.with(TemporalAdjusters.previous(DayOfWeek.FRIDAY));
LocalDate firstOfYear = today.with(TemporalAdjusters.firstDayOfYear());
LocalDate lastOfYear = today.with(TemporalAdjusters.lastDayOfYear());

// Custom adjuster — find next payday (15th or last day of month)
TemporalAdjuster nextPayday = temporal -> {
    LocalDate date = LocalDate.from(temporal);
    LocalDate fifteenth = date.withDayOfMonth(15);
    if (date.isBefore(fifteenth) || date.equals(fifteenth)) {
        return fifteenth;
    }
    return date.with(TemporalAdjusters.lastDayOfMonth());
};

LocalDate nextPay = today.with(nextPayday);

// Custom adjuster as lambda
TemporalAdjuster nextWorkingDay = TemporalAdjusters.ofDateAdjuster(d -> {
    LocalDate next = d.plusDays(1);
    while (next.getDayOfWeek() == DayOfWeek.SATURDAY || next.getDayOfWeek() == DayOfWeek.SUNDAY) {
        next = next.plusDays(1);
    }
    return next;
});
```

### ZoneId & ZoneOffset

- `ZoneId` identifies a timezone with DST rules (e.g., `America/Chicago`, `Europe/London`, `Asia/Kolkata`).
- `ZoneOffset` is a fixed offset from UTC (e.g., `+05:30`, `-08:00`, `Z` for UTC).
- `ZoneId` is used with `ZonedDateTime` and `OffsetDateTime` for full timezone-aware operations.
- The IANA Time Zone Database (tzdata) is bundled with the JDK and updated in each release.

```java
// System default
ZoneId systemDefault = ZoneId.systemDefault();

// Specific zone
ZoneId ist = ZoneId.of("Asia/Kolkata");
ZoneId ny = ZoneId.of("America/New_York");
ZoneId london = ZoneId.of("Europe/London");

// ZoneOffset — fixed offset
ZoneOffset utc = ZoneOffset.UTC;                  // Z
ZoneOffset plus530 = ZoneOffset.of("+05:30");     // IST offset (no DST)
ZoneOffset minus8 = ZoneOffset.ofHours(-8);       // -08:00

// Offset vs ZoneId — critical difference
// ZoneId accounts for DST rules; ZoneOffset is fixed
ZoneId losAngeles = ZoneId.of("America/Los_Angeles");
// On Jan 1: offset is -08:00; on Jul 1: offset is -07:00 (PDT)
// ZoneOffset.of("-08:00") is always -08:00 regardless of DST

// Available zones
Set<String> allZones = ZoneId.getAvailableZoneIds(); // ~600 zones

// Zone rules
ZoneRules rules = losAngeles.getRules();
boolean isDst = rules.isDaylightSavings(Instant.now());
ZoneOffset currentOffset = rules.getOffset(Instant.now());
```

---

## Common Mistakes

- **Using `LocalDateTime` When `ZonedDateTime` Is Needed**
  - `LocalDateTime` has no timezone. Storing a `LocalDateTime` for a global event (e.g., "webinar starts at 2025-06-13T14:30:00") is ambiguous — it could mean 14:30 in any timezone.
  - **Why it looks correct:** The code compiles, the tests pass with the local timezone, and the string representation looks complete. The bug only surfaces when users in different timezones see the wrong time.
  - Always use `ZonedDateTime` or `OffsetDateTime` for events that must occur at a specific instant, and use `LocalDateTime` only when the timezone is determined by context (e.g., store opening hours stored per-store).

```java
// WRONG — ambiguous without timezone
LocalDateTime webinar = LocalDateTime.of(2025, 6, 13, 14, 30);
// A user in London sees 14:30 BST, a user in Tokyo sees 14:30 JST — same clock, different instants

// RIGHT — unambiguous
ZonedDateTime webinar = ZonedDateTime.of(
    LocalDateTime.of(2025, 6, 13, 14, 30),
    ZoneId.of("America/New_York")
);
// User in London sees 19:30 BST (converted from 14:30 EDT)
// User in Tokyo sees 03:30 JST (next day, converted from 14:30 EDT)
```

- **Assuming `Duration.ofDays(1)` Is Always 24 Hours**
  - `Duration.ofDays(1)` is exactly 86,400 seconds (24 × 60 × 60) — it does not account for DST changes.
  - **Why it looks correct:** On most days, adding `Duration.ofDays(1)` is the same as adding one calendar day. But on DST transition days, the results differ: spring-forward day has only 23 hours, fall-back day has 25 hours.
  - Use `Period.ofDays(1)` for calendar-based arithmetic (adds 1 day regardless of clock changes) and `Duration.ofDays(1)` for machine-time arithmetic (fixed 24 hours).

```java
// DST spring-forward: March 9, 2025 in US
ZonedDateTime beforeSpringForward = ZonedDateTime.of(
    LocalDateTime.of(2025, 3, 8, 23, 0), ZoneId.of("America/New_York"));

// Using Duration — adds fixed 24 hours
ZonedDateTime withDuration = beforeSpringForward.plus(Duration.ofDays(1));
// Result: 2025-03-09T23:00:00-04:00 (still 23:00, EDT)

// Using Period — adds 1 calendar day
ZonedDateTime withPeriod = beforeSpringForward.plus(Period.ofDays(1));
// Result: 2025-03-09T23:00:00-04:00 (same here because 23:00 is after the DST jump at 2:00)
// But if starting at 01:00:
ZonedDateTime beforeDstJump = ZonedDateTime.of(
    LocalDateTime.of(2025, 3, 9, 1, 30), ZoneId.of("America/New_York"));
// This is actually 2025-03-09T01:30-05:00 (EST, before jump)
// plus(Period.ofDays(1)) → 2025-03-10T01:30-04:00 (EDT)
// plus(Duration.ofDays(1)) → 2025-03-10T02:30-04:00 (EDT) — different!
```

- **Sharing `DateTimeFormatter` as a Static Field Without Recognizing It's Thread-Safe**
  - Some developers defensively create a new `DateTimeFormatter` per operation, thinking `SimpleDateFormat` thread-safety issues apply to the new API.
  - **Why it looks correct:** Legacy experience teaches that formatters are not thread-safe. The defensive pattern of creating a new formatter per call works correctly but is wasteful — each creation allocates a `DateTimeFormatter` and its internal `DecimalStyle`, `ResolverStyle`, and `Locale` objects.
  - `DateTimeFormatter` is immutable and thread-safe. Create it once as a `static final` constant and reuse it across threads.

```java
// BAD — creates a new formatter per call (wasteful)
public String formatDate(LocalDate date) {
    DateTimeFormatter fmt = DateTimeFormatter.ofPattern("dd/MM/yyyy");
    return date.format(fmt);
}

// GOOD — shared constant, thread-safe
private static final DateTimeFormatter DATE_FORMATTER =
    DateTimeFormatter.ofPattern("dd/MM/yyyy");

public String formatDate(LocalDate date) {
    return date.format(DATE_FORMATTER);
}
```

- **Parsing Without Try-Catch or Using Invalid Patterns**
  - `DateTimeFormatter.parse()` and the `LocalDate.parse()` methods throw `DateTimeParseException` (a subclass of `DateTimeException`, which is a subclass of `RuntimeException`) for invalid input.
  - **Why it looks correct:** The parsing works for valid inputs in tests. The developer assumes inputs are always valid (they come from a trusted source), but production data often includes malformed dates from external integrations.
  - Always wrap date parsing in try-catch or validate with a `DateTimeFormatter` that uses `ResolverStyle.LENIENT` for flexible parsing.

```java
// BAD — crashes on invalid input
LocalDate date = LocalDate.parse(userInput);

// BETTER — handle parse failure
try {
    LocalDate date = LocalDate.parse(userInput);
    return date;
} catch (DateTimeParseException e) {
    log.warn("Invalid date input: {}", userInput, e);
    return null; // or throw domain exception
}

// LENIENT parsing for flexible input
DateTimeFormatter lenient = new DateTimeFormatterBuilder()
    .append(DateTimeFormatter.ISO_LOCAL_DATE)
    .parseLenient()
    .toFormatter()
    .withResolverStyle(ResolverStyle.LENIENT);

// LENIENT allows 2025-02-30 → becomes 2025-03-02
// SMART (default) throws for 2025-02-30
// STRICT rejects anything not in the specified format exactly
```

- **Forgetting Daylight Saving Time in Recurring Events**
  - A daily job scheduled at `02:00` does not exist on spring-forward day in most US timezones (the clock jumps from 01:59 to 03:00).
  - **Why it looks correct:** The schedule works 364 days a year. The missing day is discovered during production incident review when a batch job silently fails or runs at the wrong time.
  - Use `ZonedDateTime` with `ZoneId` and handle the gap: define the triggering logic to run at the next valid time after the gap, or schedule in UTC and convert to local time for display.

```java
// DST gap — 02:00 does not exist on spring-forward day
ZoneId ny = ZoneId.of("America/New_York");
try {
    ZonedDateTime twoAm = ZonedDateTime.of(
        LocalDateTime.of(2025, 3, 9, 2, 0), ny);
} catch (DateTimeException e) {
    // 2:00 AM does not exist — spring forward jumps from 1:59 to 3:00
}

// Correct approach: schedule in UTC
ZonedDateTime scheduledUtc = ZonedDateTime.of(
    LocalDateTime.of(2025, 3, 9, 2, 0), ZoneOffset.UTC);
// 02:00 UTC always exists — no DST in UTC

// Or handle the gap explicitly
ZonedDateTime adjusted = ZonedDateTime.of(
    LocalDateTime.of(2025, 3, 9, 2, 0), ny)
    .withEarlierOffsetAtOverlap(); // Only works for overlaps, not gaps
```

---

## Real-World Scenarios

### Scenario 1: Cross-Timezone Booking System

An airline booking system stores flight departure times from multiple timezones, displays them in the user's local timezone, and calculates durations. The system must handle DST changes, different timezone rules at origin and destination, and provide an unambiguous audit trail.

```java
public class FlightBookingService {
    private static final DateTimeFormatter DISPLAY_FORMAT =
        DateTimeFormatter.ofPattern("EEE, d MMM yyyy HH:mm z");

    public FlightInfo createFlight(
            String flightNo,
            LocalDateTime departureLocal,
            ZoneId departureZone,
            Duration flightDuration) {

        ZonedDateTime departure = ZonedDateTime.of(departureLocal, departureZone);
        ZonedDateTime arrival = departure.plus(flightDuration);

        return new FlightInfo(flightNo, departure, arrival, flightDuration);
    }

    public String displayDepartureInUserTimezone(FlightInfo flight, ZoneId userZone) {
        ZonedDateTime inUserTz = flight.departure().withZoneSameInstant(userZone);
        return inUserTz.format(DISPLAY_FORMAT);
    }

    public long minutesUntilDeparture(FlightInfo flight) {
        return Duration.between(Instant.now(), flight.departure().toInstant()).toMinutes();
    }

    public boolean isConnectingFlightFeasible(
            FlightInfo first, FlightInfo second, Duration minimumLayover) {

        Instant firstArrival = first.arrival().toInstant();
        Instant secondDeparture = second.departure().toInstant();
        Duration layover = Duration.between(firstArrival, secondDeparture);
        return layover.compareTo(minimumLayover) >= 0;
    }

    // Audit log uses Instant — unambiguous timeline point
    public void logFlightCreation(FlightInfo flight) {
        auditLog.info("Flight {} created, departure: {}, arrival: {}, recorded at: {}",
            flight.flightNo(),
            flight.departure().toString(),   // Full ISO string with zone
            flight.arrival().toString(),     // Full ISO string with zone
            Instant.now().toString());       // UTC instant
    }
}

record FlightInfo(
    String flightNo,
    ZonedDateTime departure,
    ZonedDateTime arrival,
    Duration duration
) {}
```

- Every flight has a `ZonedDateTime` departure and arrival — these are unambiguous timeline points that include DST-aware zone information.
- The `displayDepartureInUserTimezone()` method converts to the user's zone using `withZoneSameInstant()`, preserving the exact instant while changing the displayed time.
- `Instant` is used for audit logging and layover calculation — the system records exactly when a flight was created, regardless of the server's timezone.
- The `Duration` between arrival and next departure is compared to minimum layover — this is machine-time arithmetic on `Instant` values, which is DST-agnostic and correct.

### Scenario 2: Subscription Billing with Monthly Periods

A SaaS billing system calculates subscription expiry dates, prorates charges for partial months, and generates invoices. The system must handle months of different lengths, leap years, and DST transitions without introducing off-by-one errors.

```java
public class BillingService {
    public LocalDate calculateNextBillingDate(
            LocalDate currentPeriodStart,
            BillingCycle cycle) {

        return switch (cycle) {
            case MONTHLY -> currentPeriodStart.plusMonths(1);
            case QUARTERLY -> currentPeriodStart.plusMonths(3);
            case ANNUAL -> currentPeriodStart.plusYears(1);
        };
    }

    public BigDecimal prorateCharge(
            LocalDate subscriptionStart,
            LocalDate billingDate,
            BigDecimal monthlyRate) {

        // Period arithmetic for partial month
        Period periodUntilBilling = Period.between(subscriptionStart, billingDate);
        int months = periodUntilBilling.getMonths() + periodUntilBilling.getYears() * 12;

        // Days in the partial month (proportion of the starting month's length)
        int daysInStartMonth = subscriptionStart.lengthOfMonth();
        int daysUsed = periodUntilBilling.getDays();

        BigDecimal fullMonthsCharge = monthlyRate.multiply(BigDecimal.valueOf(months));
        BigDecimal partialCharge = monthlyRate
            .multiply(BigDecimal.valueOf(daysUsed))
            .divide(BigDecimal.valueOf(daysInStartMonth), 2, RoundingMode.HALF_UP);

        return fullMonthsCharge.add(partialCharge);
    }

    public boolean isSubscriptionExpired(LocalDate expiryDate) {
        return LocalDate.now(ZoneId.of("UTC")).isAfter(expiryDate);
    }

    // Generate invoices for all active subscriptions on billing date
    public List<Invoice> generateDailyInvoices(
            List<Subscription> activeSubscriptions) {

        LocalDate today = LocalDate.now(ZoneId.of("UTC"));
        return activeSubscriptions.stream()
            .filter(sub -> sub.nextBillingDate().equals(today))
            .map(this::generateInvoice)
            .collect(Collectors.toList());
    }

    // Handle month-end correctly
    public LocalDate addMonthEndSafe(LocalDate date, int months) {
        // plusMonths handles month-end correctly:
        // Jan 31 + 1 month = Feb 28 (or 29 in leap year)
        return date.plusMonths(months);
    }
}

enum BillingCycle { MONTHLY, QUARTERLY, ANNUAL }
record Subscription(LocalDate nextBillingDate, BigDecimal monthlyRate) {}
record Invoice(LocalDate date, BigDecimal amount) {}
```

- `LocalDate.plusMonths()` correctly handles month-end: January 31 plus 1 month returns February 28 (or 29 in leap years), not March 3. This is the correct behavior for billing dates.
- `Period.between()` calculates calendar-based differences: the period between January 15 and March 10 is 1 month and 26 days (or 0 months and 54 days depending on context) — the `BillingService` uses this for proration.
- `LocalDate.lengthOfMonth()` returns the correct number of days (28, 29, 30, or 31) for proration calculations.
- Billing uses `LocalDate` (not `ZonedDateTime`) because subscription dates are date-based, not instant-based — the timezone is only relevant for determining "today" at billing generation time.

### Scenario 3: Distributed System Event Logging with Timestamps

A microservice platform processes events from multiple geographic regions. Each event carries a timestamp, and the system must order events globally, maintain an audit trail, and handle clock skew across services.

```java
public class EventLogger {
    private static final DateTimeFormatter LOG_FORMAT =
        DateTimeFormatter.ofPattern("yyyy-MM-dd'T'HH:mm:ss.SSSXXX");

    // All services log in UTC; convert for display
    public String formatEventForLog(Event event) {
        Instant eventTime = event.timestamp();
        ZonedDateTime displayTime = eventTime.atZone(ZoneId.of("UTC"));
        return displayTime.format(LOG_FORMAT);
    }

    public boolean isEventWithinWindow(
            Event event,
            Duration window,
            Instant referenceTime) {

        Duration age = Duration.between(event.timestamp(), referenceTime);
        return !age.isNegative() && age.compareTo(window) <= 0;
    }

    // Detect clock skew between services
    public Duration estimateClockSkew(
            Event event,
            Instant serviceReceiveTime) {

        Duration networkLatency = Duration.ofMillis(50); // estimated
        Instant estimatedOriginTime = event.timestamp().minus(networkLatency);
        return Duration.between(estimatedOriginTime, serviceReceiveTime);
    }

    // Group events into 5-minute windows for analytics
    public Map<Instant, List<Event>> groupIntoWindows(
            List<Event> events,
            Duration windowSize) {

        return events.stream()
            .collect(Collectors.groupingBy(
                e -> truncateToWindow(e.timestamp(), windowSize),
                TreeMap::new,
                Collectors.toList()
            ));
    }

    private Instant truncateToWindow(Instant timestamp, Duration window) {
        long windowSeconds = window.getSeconds();
        long epochSeconds = timestamp.getEpochSecond();
        long truncated = (epochSeconds / windowSeconds) * windowSeconds;
        return Instant.ofEpochSecond(truncated);
    }
}

record Event(String id, Instant timestamp, String source, String payload) {}
```

- All events carry `Instant` — the unambiguous machine timestamp. Systems in different timezones can compare, sort, and window events without conversion.
- `Duration.between()` measures the exact time between two instants, used for age checks and clock skew estimation.
- The windowing function truncates the epoch second to the nearest window boundary using simple integer arithmetic — no timezone or calendar complexity because `Instant` is just a linear count of seconds.
- The clock skew estimation acknowledges that `event.timestamp()` was set on the producing service's clock, which may differ from the consuming service's clock — a fundamental challenge in distributed systems that `Instant` cannot solve, only expose.

---

## Use Cases

Reach for the Java 8 Date and Time API whenever you need to handle dates, times, or timezones in a safe, immutable, and unambiguous way.

- **User-local date/time display** — Use `ZonedDateTime` or `OffsetDateTime` to store both the instant and the zone, converting to the user's timezone only at presentation time.
  - Prevents the classic "stored LocalDateTime and displayed wrong time" bug.
  - **Avoid when:** timezone is irrelevant (e.g., birthdate) — use `LocalDate` for simplicity.

- **Machine timestamps and scheduling** — Use `Instant` for a linear, timezone-free point on the timeline. Combine with `Duration` for elapsed time and `Period` for calendar-based intervals.
  - `Instant` is the natural choice for logging, timestamps, and distributed systems.
  - **Avoid when:** you need human-readable relative times ("2 hours ago") — format separately with a library.

- **JSON serialization in REST APIs** — Use `Instant` or `OffsetDateTime` serialized to ISO-8601 strings (e.g., `2025-06-13T14:30:00Z`) for unambiguous, timezone-aware API contracts.
  - Jackson supports `java.time` types natively with `jackson-datatype-jsr310`.
  - **Avoid when:** clients only need a date (YYYY-MM-DD) — use `LocalDate` to avoid timezone confusion.

- **Database column mapping** — Map `Instant` to `TIMESTAMPTZ` (timezone-aware) and `LocalDateTime` to `TIMESTAMP` (timezone-naive) based on whether the application or the database owns timezone semantics.
  - **Avoid when:** you are unsure which side owns timezone logic — prefer `TIMESTAMPTZ` / `Instant` as the safer default.

- **Date arithmetic for business logic** — `LocalDate.plusDays()`, `ChronoUnit.between()`, and `TemporalAdjusters` handle calendar arithmetic correctly across month/year/month boundaries.
  - Much safer than manually adding milliseconds to `java.util.Date`.
  - **Avoid when:** you need exact second precision — use `Instant` and `Duration` for machine-time arithmetic.

---

## Scenario-Based Questions

**Q: A team stores `LocalDateTime` in a database for a global event schedule. Users in different timezones see wrong event times. What is the root cause and how do you fix it?**

  - The root cause is using `LocalDateTime`, which has no timezone. The stored value "2025-06-13T14:30:00" is interpreted as local time by each user's application, showing 14:30 in every timezone — which represents different instants.
  - The fix: store all event times as `ZonedDateTime` with the event's timezone, or store as `Instant` (UTC) with a separate timezone field for display purposes.
  - Migration approach: add a `zone_id` column to the database, then convert stored `LocalDateTime` values to `ZonedDateTime` using the event's designated zone before displaying to users.
  - For new development, always use `Instant` for storage and `ZonedDateTime` for display.

  > **Interview follow-up:** The candidate suggested adding a zone_id column. How would you handle existing data where the intended timezone is unknown — assume UTC, infer from the user's timezone at creation time, or require manual data remediation?

**Q: A scheduler runs a job every day at `02:00` local time using `LocalTime.now().getHour() == 2`. On DST spring-forward day, the job never runs. On fall-back day, it runs twice. How do you fix the scheduling logic?**

  - Using `LocalTime` for scheduling is fundamentally flawed because DST transitions create gaps (spring-forward, where 02:00 does not exist) and overlaps (fall-back, where 02:00 occurs twice).
  - The fix: schedule jobs in UTC, which does not observe DST. Convert the desired local time to UTC at schedule creation time, or use a scheduler library that handles DST natively (e.g., Quartz with timezone support).
  - For the spring-forward gap: if 02:00 does not exist, decide whether to run at 03:00 (next valid time) or skip the day.
  - For the fall-back overlap: decide whether to run at the first 02:00 (EDT) or the second 02:00 (EST), or run once at a configurable offset.

  > **Interview follow-up:** The candidate suggested scheduling in UTC. If the business requirement is to always run at "02:00 local time" regardless of DST, how would you handle the spring-forward gap — skip, run at 03:00, or adjust to 01:00 the day before — and what are the domain implications of each choice?

**Q: A developer writes `Duration.between(start.toInstant(), end.toInstant())` where `start` and `end` are `LocalDateTime`. The result is sometimes wrong by one hour. Why?**

  - `LocalDateTime.toInstant()` requires a `ZoneOffset` parameter — calling `toInstant()` without arguments does not compile. The developer must be using `start.atZone(zone).toInstant()` or has hidden the zone offset.
  - If `start` and `end` are created from different `ZonedDateTime` values and then stripped to `LocalDateTime`, the conversion back assumes a single zone offset — if the time interval spans a DST transition, the offset at start differs from the offset at end, and using a single offset produces a wrong `Duration`.
  - The fix: use `ZonedDateTime` throughout, never strip to `LocalDateTime` when instants are needed. Calculate `Duration.between(start.toInstant(), end.toInstant())` with `start` and `end` as `ZonedDateTime` — the conversion uses each value's own offset.

  > **Interview follow-up:** The candidate identified the DST transition as the cause. If the system must work with `LocalDateTime` inputs (e.g., from a legacy API), how would you determine the correct timezone offset for each value without knowing the user's intent — use the zone at the start instant for both, or interpolate?

**Q: A team stores `Instant` in a `TIMESTAMP` column in PostgreSQL. Queries filter by date range using `Instant.now().minus(7, ChronoUnit.DAYS)` and `Instant.now()`. The results are missing records that were created on the boundary. What could be wrong?**

  - `Instant.now()` returns the current UTC instant with nanosecond precision. The query filter uses `between` or `>=`, but the stored timestamps may have been truncated to millisecond or microsecond precision by the database column type.
  - If the application stores `Instant` with nanoseconds but the database column is `TIMESTAMP(3)` (milliseconds), the stored value is truncated. A query using the original `Instant` value may exclude the last record if the stored value is slightly less than the original.
  - The fix: ensure consistent precision between application and database — use `TIMESTAMP(6)` or `TIMESTAMP(9)` to match nanosecond precision, or truncate both query parameters and stored values to the same precision using `truncatedTo(ChronoUnit.MILLIS)`.
  - Another possibility: the application stored `LocalDateTime` instead of `Instant` in the database, and the query compares `Instant` to `LocalDateTime` — these are incomparable types. Use `atZone()` and `toInstant()` to convert consistently.

  > **Interview follow-up:** The candidate mentioned precision mismatch. If the database column is `TIMESTAMP WITH TIME ZONE` and the application stores `Instant`, how does the JDBC driver handle the conversion — does it truncate nanoseconds, and which JDBC type constant corresponds to nanosecond precision?

**Q: A developer uses `Period.between(date1, date2).getDays()` to calculate the number of days between two dates. The result is 0 for dates that are 30 days apart. Why?**

  - `Period.between()` returns a `Period` of years, months, and days components. Two dates that are 30 days apart may span a month boundary: January 15 to February 14 results in `Period` of 0 years, 0 months, 30 days — `getDays()` returns 30 correctly.
  - But January 31 to March 2 results in `Period` of 0 years, 1 month, 2 days (January 31 + 1 month = February 28, plus 2 days = March 2) — `getMonths()` returns 1 and `getDays()` returns 2, not 30.
  - To get the total number of days between two dates, use `ChronoUnit.DAYS.between(date1, date2)` instead of `Period.between().getDays()`.
  - `Period` is for calendar-based human reading ("1 month and 2 days"), not for total duration calculation.

  > **Interview follow-up:** The candidate correctly explained the difference between `Period` and `ChronoUnit`. If you need total months between two dates for a billing proration, does `ChronoUnit.MONTHS.between()` give fractional months or whole months, and what precision does the casting to `long` lose?

**Q: A financial system stores transaction timestamps as `LocalDateTime` at the branch's local time. The branch moves to a different timezone (e.g., from `America/Chicago` to `America/Denver`). Historical transaction order is now inconsistent. How do you recover?**

  - The original design stored local clock time without zone information — when the branch changes timezone, new transactions have timestamps that cannot be compared to historical ones (the clock was shifted).
  - Recovery: for all historical transactions, determine the timezone at the time of transaction (based on branch location history) and reconstruct the `ZonedDateTime`. Store all new transactions as `Instant` (UTC) with the branch timezone as metadata.
  - A migration script reads each historical `LocalDateTime`, applies the appropriate `ZoneId` based on the transaction date, converts to `Instant`, and updates the database.
  - Going forward, the system stores `Instant` (the unambiguous timeline point) and the branch's `ZoneId` as a separate field for display purposes.

  > **Interview follow-up:** The candidate proposed reconstructing historical timezone mappings. If the branch's timezone change coincided with a DST transition, how would you disambiguate which offset applies to a historical transaction — use the zone rules at the transaction's local date/time, or look up the actual offset from historical tzdata?

**Q: A developer uses `Instant.now()` for logging timestamps and `LocalDateTime.now()` for display in the UI. The logs and UI show times that differ by several hours. The developer verifies both calls happen within the same millisecond. What is wrong?**

  - `Instant.now()` always returns the current UTC instant. `LocalDateTime.now()` uses the JVM's default timezone, which is typically the server's configured timezone. If the server's timezone is not UTC, the two values represent different points on the timeline.
  - The fix: use `Instant.now()` for both logging and as the source of truth for the UI. Convert to `ZonedDateTime` with the user's timezone for display: `Instant.now().atZone(userZone).toLocalDateTime()`.
  - The root cause is that `LocalDateTime.now()` silently uses the default timezone, which may differ from UTC. The developer assumed both calls returned the same instant — but `LocalDateTime.now()` returns the local clock time, not the UTC instant.

  > **Interview follow-up:** The candidate identified the default timezone assumption. If the server's JVM timezone is changed from UTC to America/New_York to fix a display bug in one region, what cascading effects occur for other systems that rely on the server's UTC timestamps?

**Q: A batch process reads `Period.ofDays(30)` intervals from a configuration file and adds them to `LocalDate.now()` for scheduling. Some months the schedule jumps to the 28th instead of the 30th. What is happening?**

  - `Period.ofDays(30)` adds exactly 30 calendar days. When the current date is near month-end, adding 30 days may cross a month boundary: January 31 + 30 days = March 2 (because February has 28 days, so January 31 + 30 days = March 2).
  - The developer expected "30 days" to mean "approximately one month," but `Period.ofDays(30)` does not preserve the day-of-month like `plusMonths(1)` does. `plusMonths(1)` from January 31 returns February 28 (the last day of February), preserving month semantics.
  - The fix: use `plusMonths(1)` for monthly scheduling (preserves the day-of-month within month-end constraints) or use `ChronoUnit.DAYS.addTo(date, 30)` for exact 30-day intervals.

  > **Interview follow-up:** The candidate distinguished `Period.ofDays(30)` from `plusMonths(1)`. For a monthly billing cycle that should always bill on the same day-of-month (e.g., the 15th), what happens with `plusMonths(1)` if the 15th falls on a weekend — do you bill on the 15th regardless, or adjust to the prior Friday?

**Q: A REST API accepts date strings in the format `"yyyy-MM-dd"` and parses them with `LocalDate.parse()`. A client sends `"2025-13-01"` (month 13). The API returns a 500 error. What happened, and how do you make the API more robust?**

  - `LocalDate.parse()` uses `ResolverStyle.SMART` by default, which throws `DateTimeParseException` for month 13. The exception propagates as a 500 error because there is no handler for `DateTimeParseException`.
  - The fix: add a validation layer that catches `DateTimeParseException` and returns a 400 Bad Request with a descriptive error message. Alternatively, use `ResolverStyle.LENIENT` for flexible parsing (month 13 becomes January of next year).
  - For REST APIs, always validate date inputs before parsing and return structured error responses (e.g., `{"field": "date", "error": "Invalid date format. Expected yyyy-MM-dd."}`).
  - The broader principle: date parsing at system boundaries should be validated and produce domain-specific errors, not low-level parsing exceptions.

  > **Interview follow-up:** The candidate suggested returning 400 with a structured error message. If the API uses `ResolverStyle.LENIENT` and the client sends February 30, the date is silently adjusted to March 2. Is this acceptable, or should the API reject ambiguous dates — and how would you configure the formatter to reject February 30 while accepting other lenient inputs?

**Q: A developer writes `ZonedDateTime.now().toInstant()` and compares it to `Instant.now()`. The two values differ by the DST offset on fall-back day. What is the explanation?**

  - `ZonedDateTime.now()` creates a `ZonedDateTime` with the current instant and the system default timezone. Calling `toInstant()` strips the timezone and returns the UTC instant — which should be identical to `Instant.now()`.
  - If the two values differ, it is not because of DST. It is because `ZonedDateTime.now()` and `Instant.now()` were called at different times (even microseconds apart). The DST offset is irrelevant because `toInstant()` normalizes to UTC.
  - The actual issue is likely that the developer compared the `ZonedDateTime`'s local time to `Instant.now()` without converting — comparing `ZonedDateTime.toString()` (which shows local time) to `Instant.toString()` (which shows UTC). The "DST offset" difference is just the difference between local time and UTC.

  > **Interview follow-up:** The candidate correctly identified that `toInstant()` normalizes to UTC. If the developer instead uses `OffsetDateTime.now()` and compares it to `Instant.now()`, what relationship do the two UTC instants have, and under what condition would they differ?

---

## Interview Questions

**What are the core classes in the Java 8 Date and Time API and when do you use each?**

  - `Instant` — a machine timestamp (nanoseconds since Unix epoch). Use for logging, inter-system communication, and any scenario requiring an unambiguous timeline point.
  - `LocalDate` — a date without time or timezone. Use for birthdays, holidays, due dates.
  - `LocalTime` — a time without date or timezone. Use for opening hours, recurring daily schedules.
  - `LocalDateTime` — date and time without timezone. Use when the timezone is determined by context (e.g., store-local time).
  - `ZonedDateTime` — date and time with full timezone rules. Use for global events, flight schedules, any scenario where the same instant must be displayed in different timezones.
  - `OffsetDateTime` — date and time with fixed UTC offset (no DST rules). Use for wire protocols and ISO-8601 representations.
  - `Duration` — time-based amount (seconds, nanoseconds). Use for machine time arithmetic.
  - `Period` — date-based amount (years, months, days). Use for human calendar arithmetic.

**Why was the legacy `java.util.Date` and `Calendar` replaced?**

  - Mutability: every setter mutated the object in place, causing thread-safety issues and subtle bugs when the same `Date` was passed to multiple methods.
  - Ambiguity: `Date` represented both a date and a time, with confusing 1900-based years and 0-indexed months.
  - Non-thread-safe formatting: `SimpleDateFormat` shared an internal mutable `Calendar` between threads, causing corrupt output.
  - No ISO-8601 compliance: parsing and formatting required explicit pattern strings with non-standard symbols.
  - Poor API design: verbose, inconsistent, and difficult to extend with alternate calendar systems.

**How do you handle timezone conversions in the Java 8 Date and Time API?**

  - Use `ZonedDateTime` with `ZoneId` for full timezone-aware values.
  - Convert between zones with `withZoneSameInstant()` — preserves the instant, changes the local time.
  - Convert from `LocalDateTime` to `ZonedDateTime` with `atZone(ZoneId)` — this assigns a timezone to a local time.
  - Convert from `ZonedDateTime` to `Instant` with `toInstant()` — strips timezone, leaves the unambiguous timestamp.
  - Convert from `Instant` to `ZonedDateTime` with `atZone(ZoneId)` — assigns a timezone for display.
  - Always store and transmit in UTC (`Instant` or `ZonedDateTime` with `ZoneOffset.UTC`), convert to local timezone only for display.

**What is the difference between `Duration` and `Period`?**

  - `Duration` measures time in seconds and nanoseconds — it is machine-oriented, fixed-length (24 hours = 86,400 seconds regardless of DST).
  - `Period` measures time in years, months, and days — it is human-oriented, variable-length (a day may be 23, 24, or 25 hours depending on DST; a month has 28-31 days).
  - Use `Duration` for timing, timeout calculations, and machine-time intervals. Use `Period` for calendar arithmetic (adding months, calculating age).

**How does the API handle daylight saving time transitions?**

  - `ZonedDateTime` automatically handles DST gaps and overlaps using timezone rules from the IANA Time Zone Database.
  - During a spring-forward gap (e.g., 02:00 becoming 03:00), creating a `ZonedDateTime` for the missing time throws a `DateTimeException` (unless using `ResolverStyle.LENIENT`, which adjusts forward).
  - During a fall-back overlap (e.g., 01:30 occurring twice), `ZonedDateTime` defaults to the earlier offset (summer time). Use `withEarlierOffsetAtOverlap()` or `withLaterOffsetAtOverlap()` to choose.
  - For addition, `plusDays(1)` on a `ZonedDateTime` adds 24 calendar hours (not 24 clock hours), correctly handling the 23-hour and 25-hour DST days.

**What is `DateTimeFormatter` and how is it different from `SimpleDateFormat`?**

  - `DateTimeFormatter` is immutable and thread-safe — a single instance can be shared across threads without synchronization.
  - `SimpleDateFormat` is mutable and not thread-safe — sharing an instance between threads produces corrupt output or exceptions.
  - `DateTimeFormatter` provides predefined ISO-8601 formatters, locale-specific formatters via `ofLocalizedDate()`, and custom patterns similar to `SimpleDateFormat` but with improved symbols (e.g., `yyyy` vs `YYYY` for week-based year).
  - `DateTimeFormatter` throws `DateTimeParseException` (a runtime exception) for parse failures; `SimpleDateFormat` returns `ParseException` (a checked exception).

**What are `TemporalAdjusters` and how do you create a custom one?**

  - `TemporalAdjusters` provides predefined date adjustments: `firstDayOfMonth()`, `next(DayOfWeek.MONDAY)`, `lastDayOfYear()`, etc.
  - A custom `TemporalAdjuster` implements the `adjustInto(Temporal)` method or uses `TemporalAdjusters.ofDateAdjuster(UnaryOperator<LocalDate>)`.
  - The adjuster is applied via `date.with(adjuster)`, which returns a new adjusted date.
  - Common custom adjusters: next working day, first business day of month, next quarterly date.

**How do you parse a date string with an unknown format?**

  - Use `DateTimeFormatterBuilder` with optional patterns and `parseDefaulting()` for missing fields.
  - Combine multiple formatters in a chain: try each one and catch `DateTimeParseException`, returning the first successful parse.
  - For truly unknown formats, use `java.time.format.ResolverStyle.LENIENT` to allow relaxed parsing (e.g., month 13 → January of next year).
  - For user-facing input, use a library like Apache Commons Validator or provide format examples to guide input.

**How does the API handle leap years?**

  - `LocalDate` automatically handles leap years in all date arithmetic and queries.
  - `Year.isLeap()` and `LocalDate.isLeapYear()` check for leap years.
  - February 28 plus 1 day is February 29 in leap years, March 1 in non-leap years — handled correctly by `plusDays()`.
  - `LocalDate.of(2024, 2, 29)` succeeds (2024 is leap); `LocalDate.of(2025, 2, 29)` throws `DateTimeException`.
  - `Period.ofYears(1).plusDays(1)` from a leap year date lands on the correct date.

**How do you calculate the difference between two dates in days, months, or years?**

  - `ChronoUnit.DAYS.between(date1, date2)` — total number of days (as a `long`).
  - `ChronoUnit.MONTHS.between(date1, date2)` — total number of months (truncated to whole months).
  - `ChronoUnit.YEARS.between(date1, date2)` — total number of years.
  - `Period.between(date1, date2)` — a `Period` of years, months, and days components (for human-readable differences like "1 year, 3 months, 2 days").
  - `Duration.between(instant1, instant2)` — time-based duration for `Instant` and `LocalTime` values.

**How do you convert between `LocalDateTime` and `Instant`?**
  - `LocalDateTime.toInstant(ZoneOffset)` requires an explicit offset — there is no no-argument version because `LocalDateTime` has no timezone.
  - `Instant.atZone(ZoneId)` returns a `ZonedDateTime`, and `toLocalDateTime()` strips the zone.
  - The conversion always requires a `ZoneId` or `ZoneOffset` because `LocalDateTime` is an ambiguous local clock reading that must be pinned to a specific zone before it can be mapped to a timeline instant.

**What is `OffsetDateTime` and when would you use it over `ZonedDateTime`?**
  - `OffsetDateTime` stores date, time, and a fixed UTC offset (e.g., `+05:30`) but no DST rules. It cannot tell you whether the offset changes on March 9 — it only knows the offset at the instant it was created.
  - `ZonedDateTime` stores date, time, a `ZoneId`, and the applicable offset from the zone rules. It knows when DST transitions occur and adjusts offsets automatically.
  - Use `OffsetDateTime` for wire protocols (ISO-8601 specifies offset-based formats) and for values from systems that only provide a fixed offset. Use `ZonedDateTime` for human-facing scheduling where DST matters.

**How do you handle dates before the Unix epoch (before 1970)?**
  - `LocalDate`, `LocalDateTime`, and `ZonedDateTime` support dates from year -999,999,999 to +999,999,999. They are not limited to the Unix epoch range.
  - `Instant` stores seconds and nanoseconds from 1970-01-01T00:00:00Z, supporting values up to ±292 years (the range of a `long` in seconds). For dates outside this range, use `LocalDateTime` with a `ZoneOffset`.
  - `ChronoUnit.between()` works with `Instant` values but may overflow for very distant dates — use `Duration.between()` for the widest range.

**What is the purpose of `Clock` in the Date and Time API?**
  - `Clock` provides the current instant, date, and time using a timezone. It is the injectable time source that makes date/time code testable.
  - `Clock.fixed(Instant, ZoneId)` returns a clock that always returns the same instant — useful for testing time-dependent logic.
  - `Clock.offset(Clock, Duration)` returns a clock shifted by a duration — useful for testing future or past dates.
  - Best practice: accept a `Clock` parameter in methods that need the current time, defaulting to `Clock.systemDefaultZone()` in production and injecting a fixed clock in tests.

**What is `ResolverStyle` and how does it affect date parsing?**
  - `ResolverStyle` controls how the date/time parser handles out-of-range values: `STRICT` rejects any value outside the exact field range, `SMART` adjusts within range (Feb 29 in non-leap year → Feb 28), `LENIENT` adjusts rolling (month 13 → January of next year).
  - Default is `SMART` for `DateTimeFormatter.ofPattern()`. The ISO formatters use `STRICT`.
  - Use `STRICT` for system-to-system communication where format compliance is critical. Use `LENIENT` for user-facing input where flexibility is valued over precision. Use `SMART` for most internal processing.

**How do you calculate age from a birth date?**
  - `Period.between(birthDate, now).getYears()` — `Period`'s year component gives the integer age. This is correct: if born on June 13, 2000, `Period.between` on June 13, 2025 returns exactly 25 years.
  - The calculation is calendar-based and does not use millisecond precision — a person born on February 29 (leap year) legally ages on February 28 in non-leap years, which `Period` handles correctly.
  - For precise age in months or days for legal contexts, use `ChronoUnit.YEARS.between(birthDate, now)` which returns the same integer value.

**What is the difference between `ZonedDateTime` and `OffsetDateTime` in serialization?**
  - `ZonedDateTime` serializes to `"2025-06-13T14:30:00+05:30[Asia/Kolkata]"` — includes the zone ID in brackets. Not all JSON libraries handle the zone ID correctly.
  - `OffsetDateTime` serializes to `"2025-06-13T14:30:00+05:30"` — no zone ID, just the offset. This is more portable across systems and wire protocols.
  - For REST APIs and database storage, prefer `OffsetDateTime` or `Instant` over `ZonedDateTime` because the zone ID adds complexity with minimal benefit.

**How do you handle week-based years and week-of-year in the API?**
  - `IsoFields.WEEK_BASED_YEAR` and `IsoFields.WEEK_OF_WEEK_BASED_YEAR` provide ISO-8601 week date fields.
  - In ISO-8601, week 1 of a year is the week containing the first Thursday. January 1 may belong to week 52 or 53 of the previous year.
  - Use `date.get(IsoFields.WEEK_OF_WEEK_BASED_YEAR)` and `date.get(IsoFields.WEEK_BASED_YEAR)` to get the correct week-based values, not the calendar year.

**What is the difference between `DateTimeFormatter.ISO_INSTANT` and `DateTimeFormatter.ISO_ZONED_DATE_TIME`?**
  - `ISO_INSTANT` formats `Instant` values as UTC: `"2025-06-13T14:30:00Z"`. It always outputs the `Z` suffix.
  - `ISO_ZONED_DATE_TIME` formats `ZonedDateTime` values with the zone offset and optional zone ID: `"2025-06-13T14:30:00+05:30[Asia/Kolkata]"`.
  - `ISO_INSTANT` cannot parse values with zone offsets other than `Z` (UTC). `ISO_ZONED_DATE_TIME` can parse any valid ISO-8601 zoned datetime string.

**What is the difference between `Clock.systemDefaultZone()` and `Clock.systemUTC()`?**
  - `Clock.systemDefaultZone()` returns a clock using the JVM's default timezone — determined by the `user.timezone` system property, the `TZ` environment variable, or the OS timezone configuration.
  - `Clock.systemUTC()` returns a clock that always uses UTC regardless of JVM defaults. `LocalDate.now(systemUTC)` gives the UTC date, which may differ from the local date around midnight.
  - The JVM's default timezone can be changed at runtime via `TimeZone.setDefault()`, making `systemDefaultZone()` non-deterministic in long-running servers. Production systems should use `systemUTC()` for server-side date logic.

---

## Developer Recommendations

- **Always use `Instant` for persistence and inter-service communication**
  - Storing `Instant` (UTC) eliminates timezone ambiguity from the data layer. Convert to local timezones only at the presentation layer.
  - A team inherited a database with `LocalDateTime` columns in a multi-region application. They discovered that events from the Asia-Pacific region appeared to happen "in the future" from the US perspective because the stored times were in local time without zone markers. The migration to `Instant` required reconstructing the timezone from the event's origin metadata and was a six-month project.
  - For new projects, use `TIMESTAMP WITH TIME ZONE` in PostgreSQL or `DATETIMEOFFSET` in SQL Server, mapping to `Instant` or `OffsetDateTime` in Java.

- **Use `ZonedDateTime` for user-facing scheduling**
  - When a user says "schedule at 2:00 PM every day", they mean 2:00 PM in their local timezone, including DST adjustments.
  - Store the user's timezone (`ZoneId`) alongside the schedule's local time (`LocalTime`), and compute the next fire time using `ZonedDateTime` arithmetic.
  - A production incident occurred when a US-based team scheduled a maintenance window at "2:00 AM server time" — the server was in UTC but the team thought it was in EST. The window opened at 2:00 AM UTC (9:00 PM EST the previous day) and caused 4 hours of unexpected downtime. The fix was to always specify timezone explicitly in scheduling configurations.

- **Define `DateTimeFormatter` constants and share them across the application**
  - `DateTimeFormatter` is thread-safe and immutable — one instance per format pattern is sufficient for the entire application.
  - A codebase review revealed 47 different `DateTimeFormatter` instances created in various utility methods, each with the same `"yyyy-MM-dd"` pattern. Consolidating them into a single `static final` constant reduced allocation pressure and eliminated a class of bugs where different formatters had slightly different patterns (e.g., `"yyyy/MM/dd"` vs `"yyyy-MM-dd"`).
  - Create a `DateFormats` utility class with named constants: `STANDARD_DATE`, `STANDARD_DATETIME`, `LOG_TIMESTAMP`, `API_FORMAT`.

- **Prefer `TemporalAdjusters` over manual date arithmetic**
  - Manual arithmetic like `date.withDayOfMonth(1)` works but `date.with(TemporalAdjusters.firstDayOfMonth())` is more explicit and handles edge cases (e.g., first Monday of the month requires complex logic manually).
  - `TemporalAdjusters.next(DayOfWeek.FRIDAY)` returns the next Friday, skipping the current day if it's already Friday — use `nextOrSame()` if you want to include the current day.
  - Custom adjusters should be extracted to named `static final` constants and unit tested.

- **Profile and test timezone logic with multiple zones**
  - A unit test that only runs in the developer's local timezone (e.g., `America/New_York`) may pass while failing in production for users in `Asia/Tokyo`.
  - Parameterize timezone tests: run the same test against UTC, a zone without DST (e.g., `Asia/Kolkata`), a zone with DST (e.g., `America/New_York`), and a zone in the Southern Hemisphere with opposite DST (e.g., `Australia/Sydney`).
  - A team's billing system calculated monthly subscription expiry dates correctly for all US timezones but failed for Australian customers because the DST transition occurred in October (spring) instead of March — the `plusMonths()` logic worked correctly, but the test suite never exercised the opposite DST hemisphere.

- **Handle date parsing failures gracefully at system boundaries**
  - External data (CSV uploads, API requests, database reads) may contain invalid date strings. Always wrap parsing in try-catch for `DateTimeParseException`.
  - Log the problematic input and context (source file, row number, API endpoint) to enable debugging.
  - Use `DateTimeFormatterBuilder` with `parseDefaulting()` to fill missing fields (e.g., default time to midnight) rather than assuming all fields are present.
  - A production issue occurred when an ETL pipeline encountered a date string `"2025-02-30"` — the default `ResolverStyle.SMART` threw an exception, crashing the entire pipeline. The fix was to use `ResolverStyle.LENIENT` for data ingestion with validation rules that flagged adjusted dates for manual review.

- **Be explicit about timezone in `now()` calls**
  - `LocalDate.now()` uses the JVM's default timezone, which may differ from the server's configured timezone or the user's timezone.
  - Use `LocalDate.now(ZoneId.of("UTC"))` for server-side date logic to avoid timezone-dependent test failures and production inconsistencies.
  - A deployment pipeline ran integration tests on a CI server configured with UTC, while developers ran the same tests on machines in `America/Los_Angeles`. Tests involving `LocalDate.now()` passed on CI but failed on developer machines on DST transition days because the date boundary shifted. The fix was to use fixed `Clock` instances in tests and explicit zone IDs in production code.
