# Questions: Date-Time API

### What are the core classes of the modern Java Date/Time API, and what are their specific use cases?

> **Direct Answer:**
> - **Local Classes (Human Time / No Zone):** `LocalDate` (date only, e.g. birthdays/holidays), `LocalTime` (time only, e.g. store opening hours), and `LocalDateTime` (combined date and time for uniform-server internal scheduling).
> - **Contextual Classes (Global Time):** `ZonedDateTime` (complete contextual moment with offset, region, and daylight saving rules for international travel/meetings) and `Instant` (UTC epoch timestamp).
> - Every core class in `java.time` is strictly immutable and thread-safe.
>
> **Key Nuance / Trade-off:**
> - It is an industry best practice to convert all incoming timestamps to an `Instant` (UTC) before persisting to databases to guarantee uniform temporal sorting and avoid timezone corruption.
>
> **Reference Concept:** `Core Classes & Contextual Classifications`

---

### What is the difference between Period and Duration in the Java Date/Time API, and when should you use each?

> **Direct Answer:**
> - `Period` measures time using human calendar units (Years, Months, Days) and is used with date-based types like `LocalDate` to calculate age or subscription terms.
> - `Duration` measures time using exact machine stopwatch units (Seconds, Nanoseconds) and is used with time-based types like `Instant` or `LocalTime` to measure elapsed execution times or network timeouts.
> - `Period` ignores daylight saving transitions, whereas `Duration` measures exact elapsed physical seconds.
>
> **Key Nuance / Trade-off:**
> - Use `Period` when the unit of measurement requires a calendar, and use `Duration` when the measurement requires a stopwatch.
>
> **Reference Concept:** `Period vs Duration (Human Calendar vs Machine Stopwatch)`

---

### How do you handle formatting and parsing in the modern Java Date/Time API, and what is its primary advantage?

> **Direct Answer:**
> - Formatting and parsing are handled using `DateTimeFormatter`, parsing strings via `LocalDate.parse(text, formatter)` and formatting dates via `date.format(formatter)`.
> - Its primary architectural advantage is that it is completely **immutable and thread-safe**, serving as a safe drop-in replacement for the thread-unsafe legacy `SimpleDateFormat`.
>
> **Key Nuance / Trade-off:**
> - Because `DateTimeFormatter` is thread-safe, instances can be safely declared as `public static final` constants across application threads without causing race conditions.
>
> **Reference Concept:** `Thread-Safe Formatting & Parsing with DateTimeFormatter`

---

### Why should developers always use the modern java.time API over the legacy java.util.Date class?

> **Direct Answer:**
> - **Immutability & Thread Safety:** Legacy `java.util.Date` is mutable (`.setTime()`), causing severe race conditions in concurrent environments; `java.time` classes are strictly immutable.
> - **Nanosecond Precision:** Modern classes track time to nanosecond resolution ($10^{-9}$ s) compared to millisecond limits in legacy classes.
> - **Domain Clarity:** The modern API cleanly separates human calendar representations (`LocalDate`, `ZonedDateTime`) from machine timeline timestamps (`Instant`).
>
> **Key Nuance / Trade-off:**
> - Legacy `Date` mixed timezone-dependent calendar concepts with epoch offsets, creating confusing 0-indexed months and 1900-offset years; `java.time` standardizes clean, predictable domain modeling.
>
> **Reference Concept:** `java.time vs Legacy java.util.Date`

---

### What is the TemporalAdjusters class in the Java Date/Time API, and what are its common pre-defined operations?

> **Direct Answer:**
> - `TemporalAdjusters` is a strategy-pattern utility class that provides pre-defined algorithms for complex relative date manipulations without manual arithmetic.
> - Common boundary adjusters include `firstDayOfMonth()`, `lastDayOfMonth()`, `firstDayOfNextMonth()`, and `lastDayOfYear()`.
> - Day-of-week adjusters include `next(DayOfWeek)`, `previous(DayOfWeek)`, `firstInMonth(DayOfWeek)`, and `lastInMonth(DayOfWeek)`.
>
> **Key Nuance / Trade-off:**
> - It eliminates fragile "magic math" (e.g., adding days based on month length) with clean, self-documenting code while preserving class immutability.
>
> **Reference Concept:** `Pre-Defined TemporalAdjusters Operations`

---

### How would you implement custom business calendar rules using TemporalAdjusters, and how do you decide when to use a custom vs. built-in approach?

> **Direct Answer:**
> - Because `TemporalAdjuster` is a functional interface, custom rules can be implemented as lambdas that inspect the temporal object and calculate adjustments (e.g., advancing past weekends to find the next working day).
> - Use standard arithmetic methods (`plusDays`) for simple fixed shifts.
> - Use built-in `TemporalAdjusters` for standard relative calendar events ("next Monday", "last day of month").
> - Implement a custom `TemporalAdjuster` lambda for domain-specific business rules (such as banking settlement dates or fiscal quarter calculations).
>
> **Key Nuance / Trade-off:**
> - Custom adjusters encapsulate complex domain rules into reusable, decoupled components that can be safely plugged into any `.with()` invocation pipeline.
>
> **Reference Concept:** `Custom Business Rules via TemporalAdjuster Lambdas`

---
