# Module 7: Date-Time API

## 1. Modern Date/Time API Architecture (java.time)

### Core Classes & Contextual Classifications
Introduced in Java 8 (JSR 310), the `java.time` package completely replaces the legacy `java.util.Date` and `java.util.Calendar` classes. The modern architecture categorizes classes based on temporal context and timezone requirements:

```
java.time Architecture
 ├── Human Time (Local / Wall-Clock, No Timezone)
 │    ├── LocalDate (Year, Month, Day)
 │    ├── LocalTime (Hour, Minute, Second, Nanosecond)
 │    └── LocalDateTime (Combined Date and Time)
 └── Global Time (Contextual / Timeline)
      ├── ZonedDateTime (Date, Time, ZoneId, ZoneOffset, DST Rules)
      ├── OffsetDateTime (Date, Time, Fixed UTC ZoneOffset)
      └── Instant (UTC Epoch Timestamp, Machine Time)
```

#### Detailed Class Specifications & Use Cases
| Class | Format Example | Internal Representation | Enterprise Use Case |
| :--- | :--- | :--- | :--- |
| **`LocalDate`** | `2026-03-10` | Year, Month, Day | Birthdays, anniversary tracking, public holidays, invoicing dates. |
| **`LocalTime`** | `20:45:30.123456789` | Hour, Minute, Second, Nano | Opening hours, store closing times, recurring daily alarms. |
| **`LocalDateTime`** | `2026-03-10T20:45:30` | Combined date and time without timezone | Internal event timestamps where all application nodes share an implicit server timezone. |
| **`ZonedDateTime`** | `2026-03-10T20:45:30+05:30[Asia/Kolkata]` | Full context: Date, Time, ZoneOffset, ZoneId | International flight departures, multinational meeting scheduling, Daylight Saving Time (DST) shifts. |
| **`Instant`** | `2026-03-10T15:15:30.123456789Z` | Nanoseconds from Unix Epoch (`1970-01-01T00:00:00Z`) | Database transaction logs, audit trails, event timestamps (standard persistence best practice). |

---

### java.time vs Legacy java.util.Date
The legacy `java.util.Date` and `java.util.Calendar` APIs were fundamentally flawed and have been deprecated in modern enterprise systems.

#### Core Architectural Comparisons
1. **Immutability and Thread Safety:**
   - **Legacy `Date`:** Inherent **mutability**. If a `Date` instance is passed across layers or threads, any caller can invoke `.setTime(long)`, silently altering state for all consumers and causing severe concurrency bugs.
   - **Modern `java.time`:** Strictly **immutable** and **thread-safe**. Any modifying operation (`plusDays()`, `withMonth()`) returns a brand new instance, guaranteeing thread safety without locks.
2. **Precision:**
   - **Legacy `Date`:** Limited to millisecond precision ($10^{-3}$ seconds).
   - **Modern `java.time`:** Full **nanosecond precision** ($10^{-9}$ seconds), essential for high-frequency trading (HFT), distributed transaction tracing, and micro-benchmarking.
3. **Domain Separation (Human vs Machine Time):**
   - **Legacy `Date`:** Conflated machine timestamps with calendar days (months were 0-indexed, years started from 1900).
   - **Modern `java.time`:** Clearly separates human wall-clock time (`LocalDate`, `ZonedDateTime`) from machine timeline points (`Instant`).

---

## 2. Temporal Spans & Formatting

### Period vs Duration (Human Calendar vs Machine Stopwatch)
Both `Period` and `Duration` implement `TemporalAmount` to measure the temporal distance between two points in time, but they operate on distinct units.

| Feature | `java.time.Period` | `java.time.Duration` |
| :--- | :--- | :--- |
| **Conceptual Model** | **Human Calendar** | **Machine Stopwatch** |
| **Internal Measurement** | Years, Months, Days | Seconds, Nanoseconds (converts to hours/minutes/ms) |
| **Compatible Classes** | `LocalDate`, `LocalDateTime` | `Instant`, `LocalTime`, `LocalDateTime` |
| **Daylight Saving Time** | Ignores DST shifts (a day is always 1 calendar day) | Tracks exact physical seconds elapsed across DST transitions |
| **Primary Use Cases** | Calculating age, contract expiration, subscription tenure | Measuring method execution time, socket timeouts, rate limits |

#### Implementation Examples
```java
// 1. Period (Date-based: Human Calendar)
LocalDate birthday = LocalDate.of(1990, 5, 15);
LocalDate today = LocalDate.now();

Period age = Period.between(birthday, today);
System.out.println("Age: " + age.getYears() + " years, " + age.getMonths() + " months");

// 2. Duration (Time-based: Machine Stopwatch)
Instant startTime = Instant.now();
// ... execute computational workload ...
Instant endTime = Instant.now();

Duration executionTime = Duration.between(startTime, endTime);
System.out.println("Execution took: " + executionTime.toMillis() + " ms (" + executionTime.toNanos() + " ns)");
```

---

### Thread-Safe Formatting & Parsing with DateTimeFormatter
In legacy Java, `java.text.SimpleDateFormat` was notoriously thread-unsafe because it maintained internal state variables during parsing/formatting. Sharing an instance across threads caused corrupted date conversions.

#### The Modern Standard: `DateTimeFormatter`
- `java.time.format.DateTimeFormatter` is completely **immutable** and **thread-safe**.
- Formatters can be safely instantiated once and stored in `public static final` constants.

```java
// 1. Parsing (String -> Date)
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd-MM-yyyy");
LocalDate date = LocalDate.parse("10-03-2026", formatter);

// 2. Formatting (Date -> String)
String formattedDate = date.format(formatter); // "10-03-2026"

// 3. Predefined ISO Formatters
String isoDate = LocalDate.now().format(DateTimeFormatter.ISO_LOCAL_DATE);
```

---

## 3. Date Adjustments & Custom Business Calendars

### Pre-Defined TemporalAdjusters Operations
`java.time.temporal.TemporalAdjusters` is a utility class implementing the Strategy pattern for complex relative date calculations without manual date arithmetic ("Magic Math").

#### Pre-Defined Factory Adjusters
1. **Boundary Adjusters:**
   - `TemporalAdjusters.firstDayOfMonth()`
   - `TemporalAdjusters.lastDayOfMonth()`
   - `TemporalAdjusters.firstDayOfNextMonth()`
   - `TemporalAdjusters.firstDayOfYear()`
   - `TemporalAdjusters.lastDayOfYear()`
2. **Day-of-Week Adjusters:**
   - `TemporalAdjusters.next(DayOfWeek.MONDAY)`: Finds the next occurrence of Monday.
   - `TemporalAdjusters.previous(DayOfWeek.FRIDAY)`: Finds the previous occurrence of Friday.
   - `TemporalAdjusters.firstInMonth(DayOfWeek.WEDNESDAY)`: First Wednesday of the month.
   - `TemporalAdjusters.lastInMonth(DayOfWeek.FRIDAY)`: Last Friday of the month.
   - `TemporalAdjusters.dayOfWeekInMonth(2, DayOfWeek.WEDNESDAY)`: The 2nd Wednesday of the month.

```java
LocalDate today = LocalDate.now();
LocalDate nextMonday = today.with(TemporalAdjusters.next(DayOfWeek.MONDAY));
LocalDate monthEnd = today.with(TemporalAdjusters.lastDayOfMonth());
```

---

### Custom Business Rules via TemporalAdjuster Lambdas
Because `TemporalAdjuster` is a functional interface (`Temporal adjustInto(Temporal temporal)`), developers can construct custom enterprise date calculators (e.g., banking settlement days, fiscal quarters, or holiday exclusion).

#### Custom Business Rule: Next Working Day Adjuster
```java
LocalDate today = LocalDate.of(2026, 3, 15); // Sunday

// Functional Custom TemporalAdjuster: skips Saturday and Sunday
TemporalAdjuster nextWorkingDay = temporal -> {
    LocalDate date = LocalDate.from(temporal);
    DayOfWeek day = date.getDayOfWeek();

    if (day == DayOfWeek.FRIDAY) {
        return date.plusDays(3); // Skip Saturday & Sunday -> Monday
    } else if (day == DayOfWeek.SATURDAY) {
        return date.plusDays(2); // Skip Sunday -> Monday
    } else {
        return date.plusDays(1); // Regular working day
    }
};

LocalDate nextWork = today.with(nextWorkingDay);
System.out.println("Next working day: " + nextWork); // 2026-03-16 (Monday)
```

#### Decision Matrix for Date Manipulations
| Requirement Type | Architectural Approach | Example Code |
| :--- | :--- | :--- |
| **Simple Shifts** | Built-in arithmetic methods | `date.plusDays(5)`, `date.minusWeeks(2)` |
| **Relative Calendar Milestones** | Standard `TemporalAdjusters` | `date.with(TemporalAdjusters.lastDayOfMonth())` |
| **Domain Business Logic** | Custom `TemporalAdjuster` lambda | `date.with(nextWorkingDay)`, `date.with(fiscalQuarterStart)` |
