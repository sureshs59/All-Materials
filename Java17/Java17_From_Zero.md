# ☕ Java 17 — From Absolute Zero
### Complete Beginner's Guide to Every Java 17 Feature

> Covers all 10 major Java 17 features with clear explanations, before/after comparisons, and real-world examples. Written for developers coming from any background.

---

## 📋 Table of Contents

1. [Text Blocks](#1-text-blocks)
2. [Records](#2-records)
3. [Sealed Classes](#3-sealed-classes)
4. [Pattern Matching for instanceof](#4-pattern-matching-for-instanceof)
5. [Switch Expressions](#5-switch-expressions)
6. [New String Methods](#6-new-string-methods)
7. [var — Local Type Inference](#7-var--local-type-inference)
8. [Helpful NullPointerException Messages](#8-helpful-nullpointerexception-messages)
9. [Stream Enhancements](#9-stream-enhancements)
10. [Immutable Collections & Other APIs](#10-immutable-collections--other-apis)
11. [Putting It All Together](#11-putting-it-all-together)

---

## 1. Text Blocks

### What is a text block?

Before Java 17, writing multi-line strings (JSON, SQL, HTML) inside Java was painful. You had to add `\n` for every new line and escape every quote with `\"`. Text blocks — introduced in Java 15 and stable in Java 17 — fix this completely.

### The old way — painful escaping

```java
// Java 11 — writing JSON was painful
String json = "{\n" +
              "  \"name\": \"Alice\",\n" +
              "  \"age\": 30,\n" +
              "  \"city\": \"NYC\"\n" +
              "}";

// SQL was just as bad
String sql = "SELECT id, name\n" +
             "FROM users\n" +
             "WHERE active = true";
```

### Java 17 way — text blocks with triple quotes

```java
// JSON — reads like real JSON!
String json = """
        {
          "name": "Alice",
          "age": 30,
          "city": "NYC"
        }
        """;

// SQL — perfectly readable
String sql = """
        SELECT id, name
        FROM   users
        WHERE  active = true
        ORDER  BY name
        """;

// HTML email template — no escaping needed
String html = """
        <html>
          <body>
            <h1>Welcome, Alice!</h1>
            <p>Your account is ready.</p>
          </body>
        </html>
        """;
```

### Inserting variables with .formatted()

```java
String name = "Bob";
int    age  = 25;

// Use .formatted() to insert values into a text block
String message = """
        Hello %s!
        You are %d years old.
        Welcome to Java 17.
        """.formatted(name, age);

System.out.println(message);
// Hello Bob!
// You are 25 years old.
// Welcome to Java 17.
```

### Rules to remember

- Start with `"""` followed immediately by a newline — the opening quotes must be the last thing on the line
- The indentation of the closing `"""` controls how much leading whitespace is stripped from each line
- No need to escape internal double quotes

---

## 2. Records

### What is a record?

A record is a special class whose entire purpose is to hold data. Java automatically generates the constructor, getters, `equals()`, `hashCode()`, and `toString()` for you. No more boilerplate!

### Before records — 30+ lines for 2 fields

```java
// Java 11 — writing a simple data class
public final class Person {
    private final String name;
    private final int    age;

    public Person(String name, int age) {
        this.name = name;
        this.age  = age;
    }

    public String getName() { return name; }
    public int    getAge()  { return age; }

    @Override
    public String toString() {
        return "Person[name=" + name + ", age=" + age + "]";
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Person)) return false;
        Person p = (Person) o;
        return age == p.age && name.equals(p.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(name, age);
    }
}
// ~30 lines for just 2 fields!
```

### Java 17 — one line!

```java
// Everything above, replaced by this:
public record Person(String name, int age) {}

// Java automatically generates:
// ✓ Constructor:  new Person(String name, int age)
// ✓ Getters:      name()  and  age()   (no "get" prefix!)
// ✓ toString():   Person[name=Alice, age=30]
// ✓ equals():     compares all fields
// ✓ hashCode():   based on all fields
```

### Using a record

```java
public record Person(String name, int age) {}

public class Main {
    public static void main(String[] args) {

        // Create using the auto-generated constructor
        Person p = new Person("Alice", 30);

        // Getters use the field name — no "get" prefix!
        System.out.println(p.name());   // Alice
        System.out.println(p.age());    // 30

        // toString() works automatically
        System.out.println(p);          // Person[name=Alice, age=30]

        // equals() works automatically
        Person p2 = new Person("Alice", 30);
        System.out.println(p.equals(p2));  // true

        // Records are immutable — no setters!
    }
}
```

### Records with validation (compact constructor)

```java
public record Product(String name, double price) {

    // Compact constructor — validate the data
    public Product {  // no () needed in compact constructor
        if (name == null || name.isBlank())
            throw new IllegalArgumentException("Name is required");
        if (price < 0)
            throw new IllegalArgumentException("Price must be >= 0");
    }

    // You can add custom methods
    public String displayLabel() {
        return "%s ($%.2f)".formatted(name, price);
    }
}

// Usage:
Product p = new Product("Coffee", 3.99);
System.out.println(p.displayLabel());  // Coffee ($3.99)

// This throws IllegalArgumentException:
// new Product("", -1.0);
```

### Key rules for records

- Records are automatically `final` — they cannot be subclassed
- All fields are automatically `final` — records are immutable
- Getters use the field name directly: `name()` not `getName()`
- Perfect for DTOs, API responses, and value objects in Spring Boot

---

## 3. Sealed Classes

### What is a sealed class?

Normally, any class can extend your class from anywhere in the codebase. Sealed classes let you declare exactly which classes are allowed to extend or implement them. Think of it as a locked, controlled list of subtypes.

### The problem without sealed

```java
// Without sealed — anyone can add unexpected subtypes!
abstract class Shape {}
class Circle    extends Shape {}
class Rectangle extends Shape {}
class Hexagon   extends Shape {}  // surprise subtype!
class StarShape extends Shape {}  // another surprise!

// When you process shapes, you can never be sure
// you've handled all possible subtypes.
```

### Java 17 — sealed interface with permits

```java
// Only Circle, Rectangle, and Triangle are allowed
sealed interface Shape
        permits Circle, Rectangle, Triangle {}

// Each permitted class must be: final, sealed, or non-sealed
final record Circle(double radius)                implements Shape {}
final record Rectangle(double width, double height) implements Shape {}
final record Triangle(double base, double height)   implements Shape {}

// This would be a compile error:
// class Hexagon implements Shape {}  // ERROR: not permitted!
```

### Power of sealed — exhaustive switch

```java
// Because Shape is sealed, the compiler knows there are
// EXACTLY 3 possible subtypes. No default case needed!
static double calculateArea(Shape shape) {
    return switch (shape) {
        case Circle c    -> Math.PI * c.radius() * c.radius();
        case Rectangle r -> r.width() * r.height();
        case Triangle t  -> 0.5 * t.base() * t.height();
        // No default needed!
        // If you add a new permitted type and forget it here,
        // the compiler gives a compile ERROR. Safe!
    };
}

public static void main(String[] args) {
    System.out.println(calculateArea(new Circle(5)));         // 78.54
    System.out.println(calculateArea(new Rectangle(4, 6)));   // 24.0
    System.out.println(calculateArea(new Triangle(3, 8)));    // 12.0
}
```

### Sealed class (not just interface)

```java
// Works with classes too, not just interfaces
public sealed class Vehicle
        permits Car, Truck, Motorcycle {}

public final class Car        extends Vehicle { int seats; }
public final class Truck      extends Vehicle { double payloadTons; }
public final class Motorcycle extends Vehicle { boolean hasSidecar; }
```

### Think of sealed as "enum but richer"

| Feature | Enum | Sealed |
|---|---|---|
| Fixed set of values | Yes | Yes |
| Each variant has same fields | Yes | No — each can differ |
| Each variant is a class | No | Yes |
| Can hold complex data | Limited | Yes |

---

## 4. Pattern Matching for instanceof

### What is pattern matching?

Before Java 17, when you used `instanceof` to check a type, you still had to cast the object to that type separately. Pattern matching merges both the check AND the cast into a single, clean step.

### Old way — three repetitive steps

```java
Object obj = "Hello, World!";

// Step 1: check the type
if (obj instanceof String) {
    // Step 2: cast (redundant — we just checked it IS a String!)
    String s = (String) obj;
    // Step 3: use it
    System.out.println(s.length());  // 13
}
// Three steps when one should be enough.
```

### Java 17 — one step!

```java
Object obj = "Hello, World!";

// Check + cast + bind in ONE expression:
if (obj instanceof String s) {
    // s is already String — no cast needed!
    System.out.println(s.length());  // 13
    System.out.println(s.toUpperCase());  // HELLO, WORLD!
}
```

### Pattern matching with conditions (guard)

```java
Object obj = "Hello, World!";

// Check type AND add a condition with &&
if (obj instanceof String s && s.length() > 5) {
    System.out.println("Long string: " + s.toUpperCase());
    // Long string: HELLO, WORLD!
}

// Can be used with else-if for multiple types:
static void describe(Object obj) {
    if (obj instanceof String s && s.length() > 5) {
        System.out.println("Long string: " + s);
    } else if (obj instanceof String s) {
        System.out.println("Short string: " + s);
    } else if (obj instanceof Integer n && n > 0) {
        System.out.println("Positive number: " + n);
    } else if (obj instanceof Double d) {
        System.out.printf("Decimal: %.2f%n", d);
    } else {
        System.out.println("Something else: " + obj);
    }
}
```

### Real-world use — processing mixed API data

```java
// Imagine receiving an Object from an external API
// and needing to handle different types:
static String formatValue(Object value) {
    if (value instanceof String s)   return "\"" + s + "\"";
    if (value instanceof Integer n)  return n.toString();
    if (value instanceof Boolean b)  return b ? "yes" : "no";
    if (value instanceof Double d)   return "%.2f".formatted(d);
    return "unknown";
}

System.out.println(formatValue("hello"));  // "hello"
System.out.println(formatValue(42));       // 42
System.out.println(formatValue(true));     // yes
System.out.println(formatValue(3.14));     // 3.14
```

> **Scope rule:** The pattern variable `s` is only available inside the `if` block where the check was true. Java won't let you use it where it might not be a `String`.

---

## 5. Switch Expressions

### The problem with old switch

The old `switch` statement had a dangerous "fall-through" behaviour — if you forgot a `break`, execution continued silently into the next case. This was a very common source of bugs.

### Old switch — fall-through trap

```java
String day = "MONDAY";
int numLetters;

switch (day) {
    case "MONDAY":
    case "FRIDAY":
    case "SUNDAY":
        numLetters = 6;
        break;    // Easy to forget this break!
    case "TUESDAY":
        numLetters = 7;
        break;
    case "THURSDAY":
    case "SATURDAY":
        numLetters = 8;
        break;
    default:
        numLetters = -1;
}
// If you forget a break, the code falls through to the next case silently.
```

### Java 17 — arrow switch (no fall-through possible)

```java
String day = "MONDAY";

// Switch IS the value — assign it directly
int numLetters = switch (day) {
    case "MONDAY", "FRIDAY", "SUNDAY"    -> 6;  // arrow syntax
    case "TUESDAY"                        -> 7;
    case "THURSDAY", "SATURDAY"          -> 8;
    default                              -> -1;
};
// No break needed! Arrow -> means no fall-through is possible.
System.out.println(numLetters);  // 6
```

### Switch with multiple statements — use yield

```java
// When you need more than one line in a case, use a block + yield
static String getDayType(String day) {
    return switch (day) {
        case "SATURDAY", "SUNDAY" -> "Weekend";

        case "MONDAY", "TUESDAY", "WEDNESDAY",
             "THURSDAY", "FRIDAY" -> {
                 String result = day + " is a weekday";
                 System.out.println("Processing: " + result);  // extra step
                 yield "Weekday";     // yield returns the value (not return!)
             }

        default -> "Unknown";
    };
}

System.out.println(getDayType("MONDAY"));    // Processing: MONDAY is a weekday → Weekday
System.out.println(getDayType("SATURDAY"));  // Weekend
```

### Switch with enum — exhaustive and safe

```java
enum Season { SPRING, SUMMER, AUTUMN, WINTER }

static String getActivity(Season season) {
    return switch (season) {
        case SPRING -> "Plant seeds";
        case SUMMER -> "Go swimming";
        case AUTUMN -> "Collect leaves";
        case WINTER -> "Build a snowman";
        // No default needed!
        // Compiler verifies all 4 enum values are handled.
        // Add a new season and forget to add a case = compile error!
    };
}

System.out.println(getActivity(Season.SUMMER));  // Go swimming
```

### Key differences to remember

| Old switch statement | New switch expression |
|---|---|
| Can fall through to next case | No fall-through possible |
| Needs `break` to stop | No `break` needed |
| Cannot return a value directly | IS a value — can assign it |
| Uses `:` colon | Uses `->` arrow |
| Multiple values need repeated `case` | Multiple values with commas |

---

## 6. New String Methods

### Java 11 additions

```java
String text = "   Hello World   ";

// isBlank() — true if empty or only whitespace
System.out.println("   ".isBlank());    // true
System.out.println(" a ".isBlank());    // false
System.out.println("".isBlank());       // true

// strip() — removes ALL Unicode whitespace (better than trim()!)
System.out.println(text.strip());           // "Hello World"
System.out.println(text.stripLeading());    // "Hello World   "
System.out.println(text.stripTrailing());   // "   Hello World"

// repeat() — repeat a string n times
System.out.println("Ha".repeat(3));     // HaHaHa
System.out.println("=".repeat(30));     // ==============================
System.out.println("-".repeat(20));     // --------------------

// lines() — split into a Stream<String> of lines
String poem = "Roses are red,\nViolets are blue,\nJava 17 rocks!";
poem.lines().forEach(System.out::println);
// Roses are red,
// Violets are blue,
// Java 17 rocks!
```

### Why strip() is better than trim()

```java
// trim() was written in Java 1.0 — only removes ASCII chars <= 32
// strip() was added in Java 11 — handles ALL Unicode whitespace

String withUnicodeSpace = "\u2003Hello\u2003";  // em space (Unicode)

System.out.println(withUnicodeSpace.trim().isEmpty());   // false — missed it!
System.out.println(withUnicodeSpace.strip().isEmpty());  // true  — got it!

// Always use strip() in new code. Never use trim().
```

### Java 17 additions

```java
// formatted() — call directly on a String (cleaner than String.format)
String name  = "Alice";
int    age   = 30;
double score = 95.5;

// Old way — static method, verbose:
String old = String.format("Name: %s, Age: %d, Score: %.1f", name, age, score);

// Java 17 way — on the string itself:
String msg = "Name: %s, Age: %d, Score: %.1f".formatted(name, age, score);
System.out.println(msg);
// Name: Alice, Age: 30, Score: 95.5

// Perfect combined with text blocks:
String report = """
        Student Report
        ==============
        Name:  %s
        Age:   %d
        Score: %.1f%%
        """.formatted(name, age, score);

System.out.println(report);
// Student Report
// ==============
// Name:  Alice
// Age:   30
// Score: 95.5%

// indent() — add leading spaces to each line
String indented = "Hello\nWorld".indent(4);
System.out.println(indented);
//     Hello
//     World
```

### Common format specifiers

| Specifier | Meaning | Example |
|---|---|---|
| `%s` | String | `"Alice"` |
| `%d` | Integer | `42` |
| `%f` | Floating-point | `3.140000` |
| `%.2f` | 2 decimal places | `3.14` |
| `%n` | Newline | `\n` |
| `%%` | Literal % | `%` |

---

## 7. var — Local Type Inference

### What is var?

`var` (added in Java 10, widely used from Java 11+) tells Java to infer the variable's type automatically from the right-hand side of the assignment. You don't have to write the type on the left when Java can figure it out.

> **Important:** `var` is NOT dynamic typing. Java is still 100% type-safe. The type is fixed at compile time — `var` just moves the declaration to the right side.

### Basic var usage

```java
// Without var — repeat the type twice (redundant):
String            name    = "Alice";
List<String>      names   = new ArrayList<String>();
Map<String, Integer> scores = new HashMap<String, Integer>();

// With var — Java infers the type from the right side:
var name   = "Alice";                    // Java sees: String
var names  = new ArrayList<String>();    // Java sees: ArrayList<String>
var scores = new HashMap<String, Integer>(); // Java sees: HashMap<String,Integer>
var count  = 42;                         // Java sees: int
var price  = 9.99;                       // Java sees: double
var active = true;                       // Java sees: boolean

// Still fully type-safe — this is a compile ERROR:
// name = 42;   ERROR: String variable cannot hold int
```

### var in for loops — much cleaner

```java
var students = List.of("Alice", "Bob", "Carol", "Dave");

// Old way:
for (String student : students) {
    System.out.println(student.toUpperCase());
}

// With var:
for (var student : students) {
    System.out.println(student.toUpperCase());  // ALICE, BOB, CAROL, DAVE
}

// Especially helpful with complex generic types:
var courses = Map.of("Math", 90, "Science", 85, "Art", 92);

// Without var: Map.Entry<String, Integer> entry — very verbose!
for (var entry : courses.entrySet()) {
    System.out.printf("%s: %d%n", entry.getKey(), entry.getValue());
}
// Art: 92
// Math: 90
// Science: 85
```

### When to use and NOT use var

```java
// GOOD — type is obvious from the right side:
var name     = "Alice";                       // clearly String
var list     = new ArrayList<String>();       // clearly ArrayList<String>
var today    = LocalDate.now();               // clearly LocalDate
var response = new HttpResponse<>();          // clearly HttpResponse

// BAD — type is not clear from the right side:
var result   = processData();           // What does this return? Unclear.
var data     = getData(id);             // What type is data? We can't tell.

// var ONLY works for local variables:
// var field;                  ERROR: cannot use for class fields
// public var getMethod() {}   ERROR: cannot use for return types
// void method(var param) {}   ERROR: cannot use for method parameters
```

---

## 8. Helpful NullPointerException Messages

### The problem before Java 17

`NullPointerException` (NPE) used to give you almost no information about *which* variable was null. You would get an exception with just a line number, and had to manually read the code to figure out what was null.

### Before Java 17 — useless message

```java
// Suppose we have:
String firstName = null;
int length = firstName.length();  // NPE!

// Exception message — gives you almost nothing:
// Exception in thread "main" java.lang.NullPointerException
//   at Main.main(Main.java:3)
// Which variable was null? We have to guess.
```

### Java 17 — precise, helpful message

```java
String firstName = null;
int length = firstName.length();  // NPE!

// Java 17 exception message — tells you exactly what was null:
// Cannot invoke "String.length()" because "firstName" is null
//                                              ↑ exact variable name!
```

### Chained calls — Java 17 pinpoints the exact null

```java
record Address(String city) {}
record Person(String name, Address address) {}

Person person = new Person("Alice", null);  // address is null

// This crashes:
String city = person.address().city();

// Java 17 message:
// Cannot invoke "Address.city()" because
// the return value of "Person.address()" is null
//   ↑ Tells you: person.address() returned null!

// Arrays — even gives the index:
String[] arr = new String[3];  // all elements are null
int len = arr[0].length();

// Java 17 message:
// Cannot invoke "String.length()" because "arr[0]" is null
//                                               ↑ even the array index!
```

### Defensive coding to avoid NPE

```java
import java.util.Objects;
import java.util.Optional;

String input = null;

// Option 1: Objects.requireNonNullElse — provide a default value
String safe1 = Objects.requireNonNullElse(input, "default");
System.out.println(safe1);  // default

// Option 2: Optional — wrap nullable values safely
Optional<String> opt = Optional.ofNullable(input);

String result = opt.orElse("no value");
System.out.println(result);  // no value

// Transform safely — only runs if value is present
String upper = opt.map(String::toUpperCase).orElse("EMPTY");
System.out.println(upper);  // EMPTY

// With a non-null value:
Optional<String> name = Optional.of("Alice");
String greeting = name.map(n -> "Hello, " + n + "!").orElse("Hello!");
System.out.println(greeting);  // Hello, Alice!
```

---

## 9. Stream Enhancements

### toList() — collect to list in one word

Java 16 added `Stream.toList()` — a much shorter way to collect stream results into a list.

```java
import java.util.List;

List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// Old way — verbose Collectors.toList():
List<Integer> evens1 = numbers.stream()
        .filter(n -> n % 2 == 0)
        .collect(java.util.stream.Collectors.toList());

// Java 17 — just .toList()!
List<Integer> evens2 = numbers.stream()
        .filter(n -> n % 2 == 0)
        .toList();

System.out.println(evens2);  // [2, 4, 6, 8, 10]

// Full pipeline example:
var names   = List.of("alice", "bob", "carol", "dave", "eve");
var result  = names.stream()
        .filter(n -> n.length() > 3)      // only names longer than 3 chars
        .map(String::toUpperCase)          // convert to uppercase
        .sorted()                          // alphabetical order
        .toList();

System.out.println(result);  // [ALICE, CAROL, DAVE]
```

> **Note:** The list returned by `.toList()` is unmodifiable. If you need to add or remove elements later, use `.collect(Collectors.toList())` instead.

### mapMulti() — one element becomes zero or many

`mapMulti()` lets one input element produce zero, one, or multiple output elements. It's more flexible than `flatMap` for certain patterns.

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5);

// For each even number, emit it twice. Skip odd numbers entirely.
List<Integer> result = numbers.stream()
        .<Integer>mapMulti((n, consumer) -> {
            if (n % 2 == 0) {
                consumer.accept(n);   // emit first time
                consumer.accept(n);   // emit second time
            }
            // odd numbers: don't call consumer = they disappear
        })
        .toList();

System.out.println(result);  // [2, 2, 4, 4]

// Practical use: extract values from Optional objects
var optionals = List.of(
        Optional.of("Alice"),
        Optional.empty(),
        Optional.of("Carol"),
        Optional.empty(),
        Optional.of("Eve")
);

List<String> present = optionals.stream()
        .<String>mapMulti(Optional::ifPresent)
        .toList();

System.out.println(present);  // [Alice, Carol, Eve]
```

### Common stream operations refresher

```java
List<String> names = List.of("Alice", "Bob", "Carol", "Dave", "Eve");

// filter — keep elements that match a condition
names.stream().filter(n -> n.startsWith("A")).toList();
// [Alice]

// map — transform every element
names.stream().map(String::toLowerCase).toList();
// [alice, bob, carol, dave, eve]

// sorted — alphabetical order
names.stream().sorted().toList();
// [Alice, Bob, Carol, Dave, Eve]

// count — how many elements
long count = names.stream().filter(n -> n.length() > 3).count();
// 3  (Alice=5, Carol=5, Dave=4)

// findFirst — get the first matching element
Optional<String> first = names.stream()
        .filter(n -> n.contains("o"))
        .findFirst();
System.out.println(first.orElse("none"));  // Bob

// reduce — combine all elements
String joined = names.stream()
        .reduce("", (a, b) -> a.isEmpty() ? b : a + ", " + b);
System.out.println(joined);  // Alice, Bob, Carol, Dave, Eve
```

---

## 10. Immutable Collections & Other APIs

### List.of(), Set.of(), Map.of() — Java 9+, widely used in Java 17

```java
import java.util.*;

// Create immutable lists, sets, and maps in one line:
var colours = List.of("red", "green", "blue");
var days    = Set.of("Mon", "Tue", "Wed", "Thu", "Fri");
var scores  = Map.of("Alice", 95, "Bob", 88, "Carol", 72);

// Access elements normally:
System.out.println(colours.get(0));        // red
System.out.println(days.contains("Mon"));  // true
System.out.println(scores.get("Alice"));   // 95

// These are IMMUTABLE — you CANNOT modify them:
// colours.add("yellow");      UnsupportedOperationException!
// days.remove("Mon");         UnsupportedOperationException!
// scores.put("Dave", 100);    UnsupportedOperationException!

// Iterating:
for (var colour : colours) {
    System.out.println(colour.toUpperCase());
}
// RED, GREEN, BLUE

// Map iteration:
for (var entry : scores.entrySet()) {
    System.out.printf("%s scored %d%n", entry.getKey(), entry.getValue());
}
```

### List.copyOf() — make a safe immutable snapshot

```java
// Start with a mutable list
var original = new ArrayList<String>();
original.add("Alpha");
original.add("Beta");

// Make a safe immutable copy
var safeCopy = List.copyOf(original);

// Modify the original:
original.add("Gamma");
original.set(0, "Changed");

// The copy is unaffected:
System.out.println(original);  // [Changed, Beta, Gamma]
System.out.println(safeCopy);  // [Alpha, Beta] — unchanged!
```

### instanceof without a variable

```java
// Sometimes you only need to CHECK type, not use the value
static boolean isString(Object obj) {
    return obj instanceof String;  // no variable binding needed
}

// When you DO need the value, bind it:
static int lengthOrZero(Object obj) {
    return (obj instanceof String s) ? s.length() : 0;
}

System.out.println(isString("hello"));      // true
System.out.println(isString(42));           // false
System.out.println(lengthOrZero("hello"));  // 5
System.out.println(lengthOrZero(42));       // 0
```

### Random enhancements (Java 17)

```java
import java.util.random.RandomGenerator;
import java.util.random.RandomGeneratorFactory;

// Java 17 — new unified Random API
RandomGenerator rng = RandomGenerator.getDefault();

int    randomInt    = rng.nextInt(1, 101);     // 1 to 100 inclusive
double randomDouble = rng.nextDouble(0.0, 1.0); // 0.0 to 1.0
boolean randomBool  = rng.nextBoolean();

System.out.println("Random int:  " + randomInt);
System.out.println("Random dbl:  " + randomDouble);
System.out.println("Random bool: " + randomBool);

// Specific algorithm:
RandomGenerator secure = RandomGeneratorFactory
        .of("L64X128MixRandom")
        .create();
System.out.println(secure.nextInt(100));
```

---

## 11. Putting It All Together

### Mini expense tracker — using 7 Java 17 features at once

This example demonstrates text blocks, records, sealed classes, switch expressions, pattern matching, Stream `.toList()`, and `var` all working together in one realistic program.

```java
import java.util.List;

// ── Sealed hierarchy for expense categories ───────────────────────
sealed interface Category permits Food, Transport, Entertainment, Other {}
record Food()          implements Category {}
record Transport()     implements Category {}
record Entertainment() implements Category {}
record Other()         implements Category {}

// ── Record for an expense ─────────────────────────────────────────
record Expense(String title, double amount, Category category) {

    // Compact constructor — validation
    public Expense {
        if (title == null || title.isBlank())
            throw new IllegalArgumentException("Title required");
        if (amount < 0)
            throw new IllegalArgumentException("Amount must be >= 0");
    }

    // Custom method on record
    public String categoryName() {
        return switch (category) {
            case Food f          -> "Food";
            case Transport t     -> "Transport";
            case Entertainment e -> "Entertainment";
            case Other o         -> "Other";
        };
    }
}

// ── Main program ──────────────────────────────────────────────────
public class ExpenseTracker {

    public static void main(String[] args) {

        // Create expense list using var + List.of
        var expenses = List.of(
            new Expense("Team Lunch",      48.50, new Food()),
            new Expense("Bus pass",         30.00, new Transport()),
            new Expense("Coffee",            3.50, new Food()),
            new Expense("Movie night",      15.00, new Entertainment()),
            new Expense("Taxi to airport",  22.00, new Transport()),
            new Expense("Groceries",        65.00, new Food()),
            new Expense("Office supplies",  12.00, new Other())
        );

        // ── Filter food expenses ─────────────────────────────────
        var foodExpenses = expenses.stream()
                .filter(e -> e.category() instanceof Food)
                .toList();  // Java 17 toList()

        // ── Calculate total per category ─────────────────────────
        double foodTotal = foodExpenses.stream()
                .mapToDouble(Expense::amount)
                .sum();

        double grandTotal = expenses.stream()
                .mapToDouble(Expense::amount)
                .sum();

        // ── Print report using text block + formatted() ──────────
        String report = """
                ╔══════════════════════════════════╗
                ║       Expense Tracker Report      ║
                ╚══════════════════════════════════╝
                
                Total expenses:  %d items
                Grand total:     $%.2f
                
                Food expenses:   %d items  ($%.2f)
                """.formatted(
                        expenses.size(),    grandTotal,
                        foodExpenses.size(), foodTotal
                );

        System.out.println(report);

        // ── List all expenses grouped by category ─────────────────
        System.out.println("All expenses:");
        System.out.println("-".repeat(40));

        for (var expense : expenses) {
            System.out.printf("  %-25s %-15s $%.2f%n",
                    expense.title(),
                    expense.categoryName(),
                    expense.amount());
        }

        System.out.println("-".repeat(40));
        System.out.printf("  %-40s $%.2f%n", "TOTAL", grandTotal);
    }
}
```

**Output:**

```
╔══════════════════════════════════╗
║       Expense Tracker Report      ║
╚══════════════════════════════════╝

Total expenses:  7 items
Grand total:     $196.00

Food expenses:   3 items  ($117.00)

All expenses:
----------------------------------------
  Team Lunch                Food            $48.50
  Bus pass                  Transport       $30.00
  Coffee                    Food            $3.50
  Movie night               Entertainment   $15.00
  Taxi to airport           Transport       $22.00
  Groceries                 Food            $65.00
  Office supplies           Other           $12.00
----------------------------------------
  TOTAL                                    $196.00
```

---

## 📋 Quick Reference — All Java 17 Features

| Feature | What it does | Added in |
|---|---|---|
| **Text blocks** `"""..."""` | Multi-line strings without escaping | Java 15 (stable) |
| **Records** `record Foo(T x){}` | Data class in one line | Java 16 (stable) |
| **Sealed classes** `sealed...permits` | Control who can extend a class | Java 17 |
| **Pattern instanceof** `obj instanceof T t` | Check + cast in one step | Java 16 (stable) |
| **Switch expression** `switch(x) { case a -> b; }` | Safe, modern switch that returns a value | Java 14 (stable) |
| **String.strip()** | Unicode-aware whitespace removal | Java 11 |
| **String.isBlank()** | True if empty or only whitespace | Java 11 |
| **String.lines()** | Split into stream of lines | Java 11 |
| **String.repeat(n)** | Repeat string n times | Java 11 |
| **String.formatted()** | Call format directly on string | Java 15 |
| **var** | Local type inference | Java 10 |
| **Better NPE messages** | Says exactly which variable was null | Java 17 |
| **Stream.toList()** | Collect to unmodifiable list | Java 16 |
| **Stream.mapMulti()** | One input → zero or many outputs | Java 16 |
| **List.of() / Map.of()** | Immutable collection factory | Java 9 |
| **RandomGenerator API** | Unified Random interface | Java 17 |

---

## 🚀 Getting Started

### Install Java 17

```bash
# macOS (Homebrew)
brew install openjdk@17
export JAVA_HOME=$(brew --prefix openjdk@17)

# Ubuntu / Debian
sudo apt install openjdk-17-jdk

# Windows — download from:
# https://adoptium.net  → Java 17 → Windows x64 .msi

# Verify installation:
java -version
# openjdk version "17.0.x"
```

### Compile and run

```bash
# Save your code to Main.java, then:
javac Main.java    # compile
java Main          # run

# Or run directly (Java 11+):
java Main.java     # compile + run in one step
```

---

*Happy coding with Java 17! Each feature was designed to make your code shorter, safer, and more readable. Start with records and text blocks — you'll use them every day. ☕*
