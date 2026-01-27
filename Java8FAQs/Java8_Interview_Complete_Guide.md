# Java 8 Interview Complete Guide for SDE-2

## Table of Contents
1. [Introduction & Why Java 8](#1-introduction--why-java-8)
2. [Functional Interfaces](#2-functional-interfaces)
3. [Lambda Expressions](#3-lambda-expressions)
4. [Method References](#4-method-references)
5. [Streams API](#5-streams-api)
6. [Optional Class](#6-optional-class)
7. [Default & Static Methods in Interfaces](#7-default--static-methods-in-interfaces)
8. [New Date/Time API](#8-new-datetime-api)
9. [Practical Coding Problems](#9-practical-coding-problems)
10. [Tricky Follow-up Questions](#10-tricky-follow-up-questions)
11. [Quick Revision Cheat Sheet](#11-quick-revision-cheat-sheet)

---

# 1. Introduction & Why Java 8

## Q1: Which Java version do you use? Why Java 8 specifically?

**Answer:**
"I primarily work with Java 8 (and above). Java 8 is significant because it introduced a paradigm shift in Java programming by bringing functional programming concepts to an object-oriented language."

### Key Benefits of Java 8:

```
┌─────────────────────────────────────────────────────────────────┐
│                    JAVA 8 KEY FEATURES                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────────┐    ┌─────────────────┐                    │
│  │    Lambda       │    │   Functional    │                    │
│  │  Expressions    │    │   Interfaces    │                    │
│  └────────┬────────┘    └────────┬────────┘                    │
│           │                      │                              │
│           └──────────┬───────────┘                              │
│                      ▼                                          │
│           ┌─────────────────────┐                               │
│           │    Streams API      │◄── Process collections       │
│           └─────────────────────┘    declaratively             │
│                      │                                          │
│           ┌──────────┴──────────┐                               │
│           ▼                     ▼                               │
│  ┌─────────────────┐   ┌─────────────────┐                     │
│  │   Optional      │   │   Date/Time     │                     │
│  │   Class         │   │   API           │                     │
│  └─────────────────┘   └─────────────────┘                     │
│                                                                 │
│  ┌─────────────────┐   ┌─────────────────┐                     │
│  │ Default Methods │   │ Method          │                     │
│  │ in Interfaces   │   │ References      │                     │
│  └─────────────────┘   └─────────────────┘                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Simple Comparison - Before and After Java 8:

```java
// BEFORE Java 8 - Sorting a list of names
Collections.sort(names, new Comparator<String>() {
    @Override
    public int compare(String a, String b) {
        return a.compareTo(b);
    }
});

// AFTER Java 8 - Same thing, one line!
names.sort((a, b) -> a.compareTo(b));
// OR even simpler:
names.sort(String::compareTo);
```

## Follow-up Q: What problems did Java 8 solve?

| Problem Before Java 8 | Solution in Java 8 |
|-----------------------|-------------------|
| Verbose anonymous classes | Lambda expressions |
| No way to pass behavior as parameter easily | Functional interfaces + Lambdas |
| Imperative collection processing (loops everywhere) | Streams API (declarative) |
| NullPointerException everywhere | Optional class |
| Poor Date/Time API (java.util.Date issues) | java.time package |
| Cannot add methods to interfaces without breaking implementations | Default methods |

---

# 2. Functional Interfaces

## Q2: What is a Functional Interface?

**Answer:**
A Functional Interface is an interface that has **exactly ONE abstract method**. It can have any number of default or static methods.

```
┌──────────────────────────────────────────────────────────────┐
│                  FUNCTIONAL INTERFACE                         │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   ✅ Exactly ONE abstract method (the contract)              │
│   ✅ Can have multiple default methods                       │
│   ✅ Can have multiple static methods                        │
│   ✅ Can have methods from Object class (equals, hashCode)   │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### Example:

```java
// This is a valid Functional Interface
@FunctionalInterface  // Optional but recommended annotation
public interface Calculator {
    int calculate(int a, int b);  // ONE abstract method

    // These are allowed:
    default void printInfo() {
        System.out.println("Calculator interface");
    }

    static void about() {
        System.out.println("Basic calculator");
    }
}

// Usage with Lambda
Calculator add = (a, b) -> a + b;
Calculator multiply = (a, b) -> a * b;

System.out.println(add.calculate(5, 3));      // Output: 8
System.out.println(multiply.calculate(5, 3)); // Output: 15
```

## Q3: What is @FunctionalInterface annotation? Is it mandatory?

**Answer:**
- The `@FunctionalInterface` annotation is **optional but recommended**
- It's a **compile-time check** - compiler will give error if interface doesn't qualify
- It documents the developer's intent

```java
@FunctionalInterface
interface MyInterface {
    void doSomething();
    void doSomethingElse(); // ❌ COMPILE ERROR!
                            // Two abstract methods not allowed
}
```

## Q4: What are the built-in Functional Interfaces in Java 8?

**Answer:**
Java 8 provides many ready-to-use functional interfaces in `java.util.function` package:

### The Big Four:

```
┌────────────────────────────────────────────────────────────────────────┐
│                    CORE FUNCTIONAL INTERFACES                          │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │ 1. PREDICATE<T>                                                  │  │
│  │    Method: boolean test(T t)                                     │  │
│  │    Purpose: Takes input, returns true/false                      │  │
│  │    Use: Filtering, Validation                                    │  │
│  │    Example: str -> str.length() > 5                              │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                              │                                         │
│                              ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │ 2. FUNCTION<T, R>                                                │  │
│  │    Method: R apply(T t)                                          │  │
│  │    Purpose: Takes input T, returns output R                      │  │
│  │    Use: Transformation, Mapping                                  │  │
│  │    Example: str -> str.toUpperCase()                             │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                              │                                         │
│                              ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │ 3. CONSUMER<T>                                                   │  │
│  │    Method: void accept(T t)                                      │  │
│  │    Purpose: Takes input, returns nothing                         │  │
│  │    Use: Side effects (print, save, log)                          │  │
│  │    Example: str -> System.out.println(str)                       │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                              │                                         │
│                              ▼                                         │
│  ┌─────────────────────────────────────────────────────────────────┐  │
│  │ 4. SUPPLIER<T>                                                   │  │
│  │    Method: T get()                                               │  │
│  │    Purpose: Takes nothing, returns output                        │  │
│  │    Use: Factory, Lazy evaluation                                 │  │
│  │    Example: () -> new ArrayList<>()                              │  │
│  └─────────────────────────────────────────────────────────────────┘  │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Complete Example with All Four:

```java
import java.util.function.*;

public class FunctionalInterfaceDemo {
    public static void main(String[] args) {

        // 1. PREDICATE - Checks condition, returns boolean
        Predicate<String> isLongWord = word -> word.length() > 5;
        System.out.println(isLongWord.test("Hello"));     // false
        System.out.println(isLongWord.test("Interview")); // true

        // 2. FUNCTION - Transforms input to output
        Function<String, Integer> getLength = str -> str.length();
        System.out.println(getLength.apply("Java")); // 4

        // 3. CONSUMER - Consumes input, no return
        Consumer<String> printer = msg -> System.out.println("Message: " + msg);
        printer.accept("Hello World"); // Message: Hello World

        // 4. SUPPLIER - Supplies/produces values
        Supplier<Double> randomSupplier = () -> Math.random();
        System.out.println(randomSupplier.get()); // 0.7234... (random)
    }
}
```

### Extended Variants:

| Interface | Description | Method |
|-----------|-------------|--------|
| `BiPredicate<T, U>` | Two inputs, boolean output | `boolean test(T t, U u)` |
| `BiFunction<T, U, R>` | Two inputs, one output | `R apply(T t, U u)` |
| `BiConsumer<T, U>` | Two inputs, no output | `void accept(T t, U u)` |
| `UnaryOperator<T>` | Same type input and output | `T apply(T t)` |
| `BinaryOperator<T>` | Two same type inputs, same type output | `T apply(T t1, T t2)` |

### Primitive Specializations (For Performance):

```java
// Avoid boxing/unboxing overhead
IntPredicate isPositive = num -> num > 0;
LongFunction<String> longToString = l -> String.valueOf(l);
DoubleSupplier randomDouble = () -> Math.random();
IntConsumer printInt = i -> System.out.println(i);
```

## Follow-up Q: Why do we need primitive specializations?

**Answer:**
To avoid **autoboxing/unboxing overhead**. Using `Predicate<Integer>` requires boxing int to Integer and unboxing back. `IntPredicate` works directly with primitives, which is faster.

```java
// Slower - boxing overhead
Predicate<Integer> isEven = n -> n % 2 == 0;

// Faster - no boxing
IntPredicate isEvenPrimitive = n -> n % 2 == 0;
```

---

# 3. Lambda Expressions

## Q5: What is a Lambda Expression?

**Answer:**
A Lambda Expression is a **concise way to represent an anonymous function** (a function without a name). It provides a clear and compact syntax to implement functional interfaces.

### Syntax:

```
┌───────────────────────────────────────────────────────────────┐
│                   LAMBDA EXPRESSION SYNTAX                     │
├───────────────────────────────────────────────────────────────┤
│                                                               │
│     (parameters) -> expression                                │
│                OR                                             │
│     (parameters) -> { statements; }                           │
│                                                               │
│  ┌─────────────┐    ┌────────┐    ┌─────────────────────┐    │
│  │ Parameters  │ -> │ Arrow  │ -> │ Body (expression    │    │
│  │ (input)     │    │Operator│    │  or statement block)│    │
│  └─────────────┘    └────────┘    └─────────────────────┘    │
│                                                               │
└───────────────────────────────────────────────────────────────┘
```

### Lambda Syntax Variations:

```java
// No parameters
Runnable r = () -> System.out.println("Hello");

// One parameter (parentheses optional)
Consumer<String> c1 = s -> System.out.println(s);
Consumer<String> c2 = (s) -> System.out.println(s);  // Same thing

// One parameter with explicit type
Consumer<String> c3 = (String s) -> System.out.println(s);

// Two parameters
BiFunction<Integer, Integer, Integer> add = (a, b) -> a + b;

// Multiple statements (need curly braces and return)
BiFunction<Integer, Integer, Integer> addWithLog = (a, b) -> {
    System.out.println("Adding " + a + " and " + b);
    return a + b;
};
```

## Q6: What are the rules for Lambda Expression?

**Answer:**

### Rules Summary:

```
┌─────────────────────────────────────────────────────────────────┐
│                    LAMBDA EXPRESSION RULES                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. PARAMETER RULES:                                            │
│     • Zero, one, or more parameters                             │
│     • Type can be explicit or inferred                          │
│     • Parentheses optional for single parameter (no type)       │
│     • Parentheses required for zero/multiple parameters         │
│                                                                 │
│  2. BODY RULES:                                                 │
│     • Single expression: no braces, no return keyword           │
│     • Multiple statements: braces required                      │
│     • Return keyword required if braces used (non-void)         │
│                                                                 │
│  3. VARIABLE ACCESS RULES:                                      │
│     • Can access final or "effectively final" local variables   │
│     • Can access instance variables freely                      │
│     • Cannot modify local variables (must be effectively final) │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Effectively Final - What Does It Mean?

```java
public void demo() {
    int count = 10;  // This is "effectively final" - never modified

    // ✅ This works - count is effectively final
    Consumer<String> c1 = s -> System.out.println(s + count);

    count = 20; // Now count is modified!

    // ❌ COMPILE ERROR - count is no longer effectively final
    Consumer<String> c2 = s -> System.out.println(s + count);
}
```

## Q7: Can you explain "this" reference in Lambda vs Anonymous class?

**Answer:**
This is a **very common interview question**!

```java
public class ThisDemo {
    private String name = "ThisDemo";

    public void demonstrate() {

        // Anonymous class - 'this' refers to the anonymous class instance
        Runnable anonymous = new Runnable() {
            private String name = "Anonymous";
            @Override
            public void run() {
                System.out.println(this.name);  // Prints: Anonymous
            }
        };

        // Lambda - 'this' refers to the enclosing class (ThisDemo)
        Runnable lambda = () -> {
            System.out.println(this.name);  // Prints: ThisDemo
        };

        anonymous.run();  // Anonymous
        lambda.run();     // ThisDemo
    }
}
```

```
┌─────────────────────────────────────────────────────────────────┐
│                'this' REFERENCE COMPARISON                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   Anonymous Class:                                              │
│   ┌─────────────────────────────┐                               │
│   │ this → Anonymous Class      │                               │
│   │        Instance             │                               │
│   └─────────────────────────────┘                               │
│                                                                 │
│   Lambda Expression:                                            │
│   ┌─────────────────────────────┐                               │
│   │ this → Enclosing Class      │                               │
│   │        Instance             │                               │
│   └─────────────────────────────┘                               │
│                                                                 │
│   Key Point: Lambda doesn't create new scope for 'this'         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Q8: Lambda vs Anonymous Class - Key Differences?

| Aspect | Anonymous Class | Lambda Expression |
|--------|----------------|-------------------|
| Type | Creates a new class | Doesn't create new class |
| Compilation | Generates .class file | Uses invokedynamic |
| `this` keyword | Refers to anonymous class | Refers to enclosing class |
| State | Can have instance variables | Cannot have own state |
| Interface type | Can implement any interface | Only functional interface |
| Verbosity | More verbose | Concise |

---

# 4. Method References

## Q9: What are Method References?

**Answer:**
Method References are a **shorthand notation for lambdas** that only call an existing method. They make code even more readable.

### Syntax: `ClassName::methodName` or `object::methodName`

### Four Types of Method References:

```
┌────────────────────────────────────────────────────────────────────────┐
│                    TYPES OF METHOD REFERENCES                          │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  1. STATIC METHOD REFERENCE                                            │
│     ClassName::staticMethod                                            │
│     Example: Math::max                                                 │
│     Lambda:  (a, b) -> Math.max(a, b)                                  │
│                                                                        │
│  2. INSTANCE METHOD OF PARTICULAR OBJECT                               │
│     object::instanceMethod                                             │
│     Example: System.out::println                                       │
│     Lambda:  x -> System.out.println(x)                                │
│                                                                        │
│  3. INSTANCE METHOD OF ARBITRARY OBJECT                                │
│     ClassName::instanceMethod                                          │
│     Example: String::toUpperCase                                       │
│     Lambda:  s -> s.toUpperCase()                                      │
│                                                                        │
│  4. CONSTRUCTOR REFERENCE                                              │
│     ClassName::new                                                     │
│     Example: ArrayList::new                                            │
│     Lambda:  () -> new ArrayList<>()                                   │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

### Complete Examples:

```java
import java.util.*;
import java.util.function.*;

public class MethodReferenceDemo {
    public static void main(String[] args) {
        List<String> names = Arrays.asList("Alice", "Bob", "Charlie");

        // TYPE 1: Static method reference
        // Lambda:         (a, b) -> Math.max(a, b)
        // Method Ref:     Math::max
        BiFunction<Integer, Integer, Integer> max = Math::max;
        System.out.println(max.apply(5, 10)); // 10

        // TYPE 2: Instance method of particular object
        // Lambda:         s -> System.out.println(s)
        // Method Ref:     System.out::println
        names.forEach(System.out::println);

        // TYPE 3: Instance method of arbitrary object
        // Lambda:         s -> s.toUpperCase()
        // Method Ref:     String::toUpperCase
        names.stream()
             .map(String::toUpperCase)
             .forEach(System.out::println);

        // TYPE 4: Constructor reference
        // Lambda:         () -> new ArrayList<>()
        // Method Ref:     ArrayList::new
        Supplier<List<String>> listSupplier = ArrayList::new;
        List<String> newList = listSupplier.get();
    }
}
```

## Follow-up Q: When can't you use Method References?

**Answer:**
When the lambda does more than just call a single method:

```java
// ❌ Cannot use method reference here
names.forEach(name -> System.out.println("Name: " + name));

// ❌ Cannot use method reference here (multiple operations)
names.stream()
     .map(name -> name.toUpperCase().trim())
     .collect(Collectors.toList());

// ✅ CAN use method reference (single method call)
names.forEach(System.out::println);
names.stream().map(String::toUpperCase).collect(Collectors.toList());
```

---

# 5. Streams API

## Q10: What is the Streams API?

**Answer:**
Streams API provides a **declarative way to process collections of data**. It allows you to express complex data processing queries in a functional style.

```
┌─────────────────────────────────────────────────────────────────┐
│                    STREAM PIPELINE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Source          Intermediate Ops       Terminal Op             │
│  ┌─────┐        ┌─────┐ ┌─────┐        ┌─────────┐             │
│  │List │  →     │filter│→│ map │  →    │ collect │  →  Result  │
│  │Set  │        └─────┘ └─────┘        │ forEach │             │
│  │Array│                               │ reduce  │             │
│  └─────┘                               └─────────┘             │
│                                                                 │
│  Key Points:                                                    │
│  • Stream doesn't store data - it's a pipeline                  │
│  • Intermediate ops are LAZY (don't execute until terminal op)  │
│  • Terminal op triggers the whole pipeline                      │
│  • A stream can only be consumed ONCE                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Basic Example:

```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David", "Eve");

// Find names starting with 'A' or 'B', convert to uppercase
List<String> result = names.stream()          // Source
    .filter(name -> name.startsWith("A")
                 || name.startsWith("B"))     // Intermediate
    .map(String::toUpperCase)                 // Intermediate
    .collect(Collectors.toList());            // Terminal

System.out.println(result); // [ALICE, BOB]
```

## Q11: What are the types of Stream operations?

**Answer:**

### Intermediate Operations (Lazy - Return Stream):

```java
// FILTER - Keep elements matching condition
stream.filter(x -> x > 5)

// MAP - Transform each element
stream.map(x -> x * 2)

// FLATMAP - Transform and flatten
stream.flatMap(list -> list.stream())

// DISTINCT - Remove duplicates
stream.distinct()

// SORTED - Sort elements
stream.sorted()
stream.sorted(Comparator.reverseOrder())

// PEEK - Perform action without changing stream (debugging)
stream.peek(System.out::println)

// LIMIT - Take first n elements
stream.limit(5)

// SKIP - Skip first n elements
stream.skip(3)
```

### Terminal Operations (Trigger Pipeline - Return Result):

```java
// FOREACH - Perform action on each element
stream.forEach(System.out::println)

// COLLECT - Gather into collection
stream.collect(Collectors.toList())
stream.collect(Collectors.toSet())
stream.collect(Collectors.toMap(x -> x.getId(), x -> x))

// REDUCE - Combine all elements into one
stream.reduce(0, (a, b) -> a + b)

// COUNT - Count elements
stream.count()

// FINDFIRST / FINDANY - Get one element
stream.findFirst()
stream.findAny()

// ANYMATCH / ALLMATCH / NONEMATCH - Check conditions
stream.anyMatch(x -> x > 10)
stream.allMatch(x -> x > 0)
stream.noneMatch(x -> x < 0)

// MIN / MAX - Find extreme values
stream.min(Comparator.naturalOrder())
stream.max(Comparator.naturalOrder())

// TOARRAY - Convert to array
stream.toArray()
```

## Q12: What is the difference between map() and flatMap()?

**Answer:**
This is a **very frequently asked question**!

```
┌─────────────────────────────────────────────────────────────────┐
│                    map() vs flatMap()                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  map(): ONE-TO-ONE transformation                               │
│  ┌────┐     ┌────┐                                              │
│  │ A  │ →   │ A' │                                              │
│  │ B  │ →   │ B' │                                              │
│  │ C  │ →   │ C' │                                              │
│  └────┘     └────┘                                              │
│                                                                 │
│  flatMap(): ONE-TO-MANY + FLATTEN                               │
│  ┌────┐     ┌─────────┐     ┌────┐                              │
│  │ A  │ →   │ A1,A2   │ →   │ A1 │                              │
│  │ B  │ →   │ B1,B2,B3│ →   │ A2 │                              │
│  │ C  │ →   │ C1      │ →   │ B1 │                              │
│  └────┘     └─────────┘     │ B2 │                              │
│                              │ B3 │                              │
│                              │ C1 │                              │
│                              └────┘                              │
└─────────────────────────────────────────────────────────────────┘
```

### Example:

```java
// SCENARIO: You have list of sentences, want list of all words

List<String> sentences = Arrays.asList(
    "Hello World",
    "Java is awesome",
    "Streams are powerful"
);

// Using map() - WRONG approach (gives List<String[]>)
List<String[]> wrongResult = sentences.stream()
    .map(sentence -> sentence.split(" "))
    .collect(Collectors.toList());
// Result: [["Hello", "World"], ["Java", "is", "awesome"], ...]

// Using flatMap() - CORRECT approach (gives List<String>)
List<String> correctResult = sentences.stream()
    .flatMap(sentence -> Arrays.stream(sentence.split(" ")))
    .collect(Collectors.toList());
// Result: ["Hello", "World", "Java", "is", "awesome", ...]
```

### Another Common Example - Nested Collections:

```java
// List of lists → single flattened list
List<List<Integer>> nested = Arrays.asList(
    Arrays.asList(1, 2, 3),
    Arrays.asList(4, 5),
    Arrays.asList(6, 7, 8, 9)
);

List<Integer> flat = nested.stream()
    .flatMap(List::stream)  // Flatten each inner list
    .collect(Collectors.toList());
// Result: [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

## Q13: What is the difference between findFirst() and findAny()?

**Answer:**

| Aspect | findFirst() | findAny() |
|--------|-------------|-----------|
| Returns | First element in encounter order | Any element (non-deterministic) |
| Sequential Stream | Returns first element | Returns first element (usually) |
| Parallel Stream | Returns first element (slower) | Returns any element (faster) |
| Use When | Order matters | Order doesn't matter, need performance |

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

// Sequential - both typically return 1
Optional<Integer> first = numbers.stream()
    .filter(n -> n > 0)
    .findFirst();  // Always returns 1

Optional<Integer> any = numbers.stream()
    .filter(n -> n > 0)
    .findAny();    // Returns 1 (but not guaranteed)

// Parallel - findAny() is faster
Optional<Integer> parallelAny = numbers.parallelStream()
    .filter(n -> n > 0)
    .findAny();    // Could return 1, 2, 3, 4, or 5 - faster
```

## Q14: Explain reduce() operation with examples

**Answer:**
`reduce()` combines all elements of a stream into a single result.

```
┌─────────────────────────────────────────────────────────────────┐
│                    reduce() OPERATION                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Stream: [1, 2, 3, 4, 5]                                        │
│                                                                 │
│  reduce(0, (a, b) -> a + b)                                     │
│                                                                 │
│  Step 1:  0 + 1 = 1    (identity + first element)               │
│  Step 2:  1 + 2 = 3                                             │
│  Step 3:  3 + 3 = 6                                             │
│  Step 4:  6 + 4 = 10                                            │
│  Step 5: 10 + 5 = 15   (final result)                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Different Forms of reduce():

```java
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);

// Form 1: reduce(BinaryOperator) - Returns Optional
Optional<Integer> sum1 = numbers.stream()
    .reduce((a, b) -> a + b);
System.out.println(sum1.get()); // 15

// Form 2: reduce(identity, BinaryOperator) - Returns value
Integer sum2 = numbers.stream()
    .reduce(0, (a, b) -> a + b);
System.out.println(sum2); // 15

// Form 3: reduce(identity, BiFunction, BinaryOperator) - For parallel
Integer sum3 = numbers.parallelStream()
    .reduce(0,
            (partial, element) -> partial + element,  // accumulator
            (partial1, partial2) -> partial1 + partial2);  // combiner
System.out.println(sum3); // 15
```

### Practical Examples:

```java
// Find maximum
Optional<Integer> max = numbers.stream()
    .reduce(Integer::max);

// Concatenate strings
List<String> words = Arrays.asList("Hello", "World", "Java");
String sentence = words.stream()
    .reduce("", (a, b) -> a + " " + b).trim();
// "Hello World Java"

// Calculate product
Integer product = numbers.stream()
    .reduce(1, (a, b) -> a * b);
// 120
```

## Q15: What are Collectors and common Collector methods?

**Answer:**
`Collectors` is a utility class providing implementations of the `Collector` interface for common operations.

### Most Used Collectors:

```java
import java.util.stream.Collectors;

List<Employee> employees = getEmployees();

// 1. COLLECT TO LIST
List<String> names = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.toList());

// 2. COLLECT TO SET
Set<String> departments = employees.stream()
    .map(Employee::getDepartment)
    .collect(Collectors.toSet());

// 3. COLLECT TO MAP
Map<Integer, String> idToName = employees.stream()
    .collect(Collectors.toMap(
        Employee::getId,
        Employee::getName
    ));

// 4. JOINING STRINGS
String allNames = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.joining(", "));
// "Alice, Bob, Charlie"

// 5. GROUPING BY
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));

// 6. PARTITIONING BY (Two groups: true/false)
Map<Boolean, List<Employee>> partitioned = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 50000));

// 7. COUNTING
Long count = employees.stream()
    .collect(Collectors.counting());

// 8. SUMMING
Integer totalSalary = employees.stream()
    .collect(Collectors.summingInt(Employee::getSalary));

// 9. AVERAGING
Double avgSalary = employees.stream()
    .collect(Collectors.averagingDouble(Employee::getSalary));

// 10. STATISTICS
IntSummaryStatistics stats = employees.stream()
    .collect(Collectors.summarizingInt(Employee::getSalary));
// stats.getMax(), stats.getMin(), stats.getAverage(), stats.getSum()
```

## Q16: Explain groupingBy() with downstream collectors

**Answer:**
`groupingBy()` can take a second argument - a **downstream collector** - to process grouped elements further.

```java
// Basic groupingBy
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));

// With downstream: Count employees per department
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.counting()
    ));

// With downstream: Average salary per department
Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));

// With downstream: Get just names per department
Map<String, List<String>> namesByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.mapping(Employee::getName, Collectors.toList())
    ));

// Nested groupingBy: Group by department, then by city
Map<String, Map<String, List<Employee>>> byDeptAndCity = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.groupingBy(Employee::getCity)
    ));
```

## Q17: What is a Parallel Stream? When should you use it?

**Answer:**

```
┌─────────────────────────────────────────────────────────────────┐
│              SEQUENTIAL vs PARALLEL STREAMS                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  SEQUENTIAL STREAM:                                             │
│  ┌─────────────────────────────────────────┐                    │
│  │ [1] → [2] → [3] → [4] → [5] → Result    │ (Single thread)   │
│  └─────────────────────────────────────────┘                    │
│                                                                 │
│  PARALLEL STREAM:                                               │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐                          │
│  │[1] → [2]│  │[3] → [4]│  │   [5]   │   (Multiple threads)    │
│  └────┬────┘  └────┬────┘  └────┬────┘                          │
│       └────────────┼────────────┘                               │
│                    ▼                                            │
│               Combined Result                                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Creating Parallel Streams:

```java
// Method 1: parallelStream() on collection
List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5);
numbers.parallelStream()
       .forEach(System.out::println);

// Method 2: parallel() on existing stream
numbers.stream()
       .parallel()
       .forEach(System.out::println);

// Check if stream is parallel
boolean isParallel = numbers.parallelStream().isParallel(); // true
```

### When to Use Parallel Streams:

| Use Parallel | Avoid Parallel |
|--------------|----------------|
| Large data sets (10,000+ elements) | Small data sets |
| CPU-intensive operations | I/O operations |
| Stateless operations | Stateful operations |
| Independent operations | Order-dependent operations |
| ArrayList source | LinkedList source |

### Common Pitfalls:

```java
// ❌ WRONG - Shared mutable state (race condition)
List<Integer> results = new ArrayList<>();
numbers.parallelStream()
       .forEach(n -> results.add(n * 2)); // NOT THREAD-SAFE!

// ✅ CORRECT - Use collect
List<Integer> results = numbers.parallelStream()
       .map(n -> n * 2)
       .collect(Collectors.toList());

// ❌ WRONG - forEachOrdered defeats purpose
numbers.parallelStream()
       .forEachOrdered(System.out::println); // Slower than sequential!
```

## Q18: What is Stream Lazy Evaluation?

**Answer:**
Intermediate operations don't execute until a terminal operation is called. This allows for **optimization** like short-circuiting.

```java
List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David");

// This does NOTHING yet (no terminal operation)
Stream<String> stream = names.stream()
    .filter(name -> {
        System.out.println("Filtering: " + name);
        return name.startsWith("A");
    })
    .map(name -> {
        System.out.println("Mapping: " + name);
        return name.toUpperCase();
    });

System.out.println("Stream created, no processing yet!");

// NOW it executes (terminal operation added)
Optional<String> result = stream.findFirst();

// Output:
// Stream created, no processing yet!
// Filtering: Alice
// Mapping: Alice
// (Note: Bob, Charlie, David are never even processed - short-circuit!)
```

---

# 6. Optional Class

## Q19: What is Optional and why was it introduced?

**Answer:**
`Optional` is a container class that may or may not contain a non-null value. It was introduced to **handle null values gracefully and avoid NullPointerException**.

```
┌─────────────────────────────────────────────────────────────────┐
│                    OPTIONAL - NULL HANDLER                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Before Java 8:                                                 │
│  ┌────────────────────────────────────────────┐                 │
│  │ Employee e = findEmployee(id);             │                 │
│  │ if (e != null) {                           │                 │
│  │     Department d = e.getDepartment();      │                 │
│  │     if (d != null) {                       │                 │
│  │         String name = d.getName();         │  ← Null checks  │
│  │     }                                      │    everywhere!  │
│  │ }                                          │                 │
│  └────────────────────────────────────────────┘                 │
│                                                                 │
│  With Optional:                                                 │
│  ┌────────────────────────────────────────────┐                 │
│  │ String name = findEmployee(id)             │                 │
│  │     .map(Employee::getDepartment)          │  ← Clean,       │
│  │     .map(Department::getName)              │    declarative! │
│  │     .orElse("Unknown");                    │                 │
│  └────────────────────────────────────────────┘                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

## Q20: How to create Optional objects?

```java
// Create empty Optional
Optional<String> empty = Optional.empty();

// Create Optional with non-null value (throws NPE if null)
Optional<String> present = Optional.of("Hello");

// Create Optional that may be null
Optional<String> nullable = Optional.ofNullable(possiblyNullValue);
```

## Q21: What are the main Optional methods?

```java
Optional<String> optional = Optional.of("Hello");

// 1. isPresent() / isEmpty() - Check if value exists
if (optional.isPresent()) {
    System.out.println(optional.get());
}

// 2. ifPresent() - Execute action if present
optional.ifPresent(System.out::println);

// 3. orElse() - Get value or default
String result = optional.orElse("Default");

// 4. orElseGet() - Get value or compute default (lazy)
String result = optional.orElseGet(() -> computeDefault());

// 5. orElseThrow() - Get value or throw exception
String result = optional.orElseThrow(() -> new RuntimeException("Not found"));

// 6. map() - Transform value if present
Optional<Integer> length = optional.map(String::length);

// 7. flatMap() - Transform to Optional (avoids Optional<Optional<T>>)
Optional<String> result = optional.flatMap(s -> findRelated(s));

// 8. filter() - Filter by predicate
Optional<String> filtered = optional.filter(s -> s.length() > 3);
```

## Q22: orElse() vs orElseGet() - What's the difference?

**Answer:**
This is a **common interview trick question**!

```java
// orElse() - ALWAYS evaluates the default value
String result = optional.orElse(computeExpensiveDefault());
// computeExpensiveDefault() is called EVEN IF optional has value!

// orElseGet() - LAZY evaluation
String result = optional.orElseGet(() -> computeExpensiveDefault());
// computeExpensiveDefault() is called ONLY IF optional is empty!
```

### Demo:

```java
public String getDefault() {
    System.out.println("Computing default...");
    return "Default";
}

Optional<String> present = Optional.of("Value");

// This STILL prints "Computing default..." even though we don't use it
String r1 = present.orElse(getDefault());

// This does NOT print anything - lazy evaluation
String r2 = present.orElseGet(() -> getDefault());
```

**Rule:** Use `orElseGet()` when the default value is expensive to compute.

## Q23: Optional Best Practices

```java
// ✅ DO - Return Optional from methods that may not have a result
public Optional<Employee> findById(int id) {
    // ...
}

// ❌ DON'T - Use Optional as field type
class Employee {
    private Optional<String> nickname; // BAD!
}

// ❌ DON'T - Use Optional as method parameter
public void process(Optional<String> input) { } // BAD!

// ❌ DON'T - Use Optional.get() without checking
optional.get(); // Can throw NoSuchElementException!

// ✅ DO - Chain operations
employee.map(Employee::getDepartment)
        .map(Department::getManager)
        .map(Manager::getName)
        .orElse("No manager");

// ❌ DON'T - Use Optional with collections (return empty collection instead)
public Optional<List<Employee>> findAll() { } // BAD!
public List<Employee> findAll() { return Collections.emptyList(); } // GOOD!
```

---

# 7. Default & Static Methods in Interfaces

## Q24: What are Default Methods in interfaces?

**Answer:**
Default methods allow adding **methods with implementation** to interfaces without breaking existing implementations.

```java
public interface Vehicle {
    void start();  // Abstract method

    // Default method - has implementation
    default void honk() {
        System.out.println("Beep beep!");
    }
}

class Car implements Vehicle {
    @Override
    public void start() {
        System.out.println("Car starting...");
    }

    // honk() is inherited automatically!
}

// Usage
Car car = new Car();
car.honk();  // "Beep beep!" - uses default implementation
```

## Q25: What happens with multiple inheritance conflict?

**Answer:**
When a class implements two interfaces with same default method, **compile error** occurs. You must override the method.

```java
interface A {
    default void hello() { System.out.println("Hello from A"); }
}

interface B {
    default void hello() { System.out.println("Hello from B"); }
}

// ❌ COMPILE ERROR - conflict!
class C implements A, B { }

// ✅ FIX - Override and resolve conflict
class C implements A, B {
    @Override
    public void hello() {
        A.super.hello();  // Choose A's implementation
        // OR
        B.super.hello();  // Choose B's implementation
        // OR
        System.out.println("Hello from C");  // Own implementation
    }
}
```

## Q26: What are Static Methods in interfaces?

**Answer:**
Static methods in interfaces are utility methods that belong to the interface itself.

```java
public interface MathUtils {

    static int add(int a, int b) {
        return a + b;
    }

    static int multiply(int a, int b) {
        return a * b;
    }
}

// Usage - called on interface, not instances
int sum = MathUtils.add(5, 3);
int product = MathUtils.multiply(5, 3);

// Note: Cannot be overridden by implementing classes
// Note: Cannot be inherited by implementing classes
```

---

# 8. New Date/Time API

## Q27: What was wrong with old Date API?

**Answer:**

| Problem with java.util.Date | Solution in java.time |
|----------------------------|----------------------|
| Mutable (not thread-safe) | Immutable (thread-safe) |
| Poor API design | Clean, fluent API |
| Months are 0-indexed (Jan=0) | Months are 1-indexed (Jan=1) |
| Year is offset from 1900 | Actual year value |
| No timezone support | Built-in timezone support |
| Date also has time | Clear separation |

## Q28: Main classes in java.time package?

```java
import java.time.*;

// LOCAL (no timezone)
LocalDate date = LocalDate.now();           // 2024-01-15
LocalTime time = LocalTime.now();           // 10:30:45.123
LocalDateTime dateTime = LocalDateTime.now(); // 2024-01-15T10:30:45.123

// WITH TIMEZONE
ZonedDateTime zonedDateTime = ZonedDateTime.now();
// 2024-01-15T10:30:45.123+05:30[Asia/Kolkata]

// SPECIFIC INSTANT IN TIME
Instant instant = Instant.now();  // UTC timestamp

// DURATIONS & PERIODS
Duration duration = Duration.ofHours(2);     // 2 hours
Period period = Period.ofDays(10);           // 10 days
```

## Q29: Common Date/Time Operations?

```java
// Creating dates
LocalDate date1 = LocalDate.of(2024, 1, 15);
LocalDate date2 = LocalDate.parse("2024-01-15");

// Manipulation (returns new instance - immutable!)
LocalDate tomorrow = date1.plusDays(1);
LocalDate lastMonth = date1.minusMonths(1);
LocalDate nextYear = date1.plusYears(1);

// Comparison
boolean isBefore = date1.isBefore(date2);
boolean isAfter = date1.isAfter(date2);

// Getting info
int year = date1.getYear();
Month month = date1.getMonth();
int day = date1.getDayOfMonth();
DayOfWeek dow = date1.getDayOfWeek();

// Formatting
DateTimeFormatter formatter = DateTimeFormatter.ofPattern("dd-MM-yyyy");
String formatted = date1.format(formatter);  // "15-01-2024"

// Parsing
LocalDate parsed = LocalDate.parse("15-01-2024", formatter);

// Period between dates
Period period = Period.between(date1, date2);
long daysBetween = ChronoUnit.DAYS.between(date1, date2);
```

---

# 9. Practical Coding Problems

These are the **most commonly asked** coding problems in interviews!

## Employee Class (Used in all examples):

```java
@Data  // Lombok - generates getters, setters, etc.
public class Employee {
    private int id;
    private String name;
    private String department;
    private double salary;
    private int age;
    private String city;

    public Employee(int id, String name, String department,
                    double salary, int age, String city) {
        this.id = id;
        this.name = name;
        this.department = department;
        this.salary = salary;
        this.age = age;
        this.city = city;
    }
}

// Sample Data
List<Employee> employees = Arrays.asList(
    new Employee(1, "Alice", "IT", 80000, 28, "Mumbai"),
    new Employee(2, "Bob", "HR", 60000, 35, "Delhi"),
    new Employee(3, "Charlie", "IT", 90000, 32, "Mumbai"),
    new Employee(4, "David", "Finance", 75000, 40, "Bangalore"),
    new Employee(5, "Eve", "IT", 85000, 25, "Delhi"),
    new Employee(6, "Frank", "HR", 55000, 30, "Mumbai"),
    new Employee(7, "Grace", "Finance", 95000, 45, "Delhi"),
    new Employee(8, "Henry", "IT", 70000, 27, "Bangalore")
);
```

## Problem 1: Get names of all employees in IT department

```java
List<String> itEmployeeNames = employees.stream()
    .filter(e -> e.getDepartment().equals("IT"))
    .map(Employee::getName)
    .collect(Collectors.toList());
// [Alice, Charlie, Eve, Henry]
```

## Problem 2: Find highest paid employee

```java
Optional<Employee> highestPaid = employees.stream()
    .max(Comparator.comparingDouble(Employee::getSalary));

// Or using reduce
Optional<Employee> highestPaid2 = employees.stream()
    .reduce((e1, e2) -> e1.getSalary() > e2.getSalary() ? e1 : e2);
```

## Problem 3: Find highest paid employee in each department

```java
Map<String, Optional<Employee>> highestPaidByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.maxBy(Comparator.comparingDouble(Employee::getSalary))
    ));

// OR - Get just the employee (not wrapped in Optional)
Map<String, Employee> highestPaidByDept2 = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.collectingAndThen(
            Collectors.maxBy(Comparator.comparingDouble(Employee::getSalary)),
            opt -> opt.orElse(null)
        )
    ));
```

## Problem 4: Find average salary by department

```java
Map<String, Double> avgSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.averagingDouble(Employee::getSalary)
    ));
// {HR=57500.0, Finance=85000.0, IT=81250.0}
```

## Problem 5: Find total salary expense by department

```java
Map<String, Double> totalSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.summingDouble(Employee::getSalary)
    ));
```

## Problem 6: Count employees by department

```java
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.counting()
    ));
// {HR=2, Finance=2, IT=4}
```

## Problem 7: Find second highest salary

```java
Optional<Employee> secondHighest = employees.stream()
    .sorted(Comparator.comparingDouble(Employee::getSalary).reversed())
    .skip(1)
    .findFirst();
```

## Problem 8: Find second highest salary in each department

```java
Map<String, Optional<Employee>> secondHighestByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.collectingAndThen(
            Collectors.toList(),
            list -> list.stream()
                .sorted(Comparator.comparingDouble(Employee::getSalary).reversed())
                .skip(1)
                .findFirst()
        )
    ));
```

## Problem 9: Get employees sorted by salary then by name

```java
List<Employee> sorted = employees.stream()
    .sorted(Comparator
        .comparingDouble(Employee::getSalary).reversed()
        .thenComparing(Employee::getName))
    .collect(Collectors.toList());
```

## Problem 10: Partition employees into high earners (>70000) and others

```java
Map<Boolean, List<Employee>> partitioned = employees.stream()
    .collect(Collectors.partitioningBy(e -> e.getSalary() > 70000));

List<Employee> highEarners = partitioned.get(true);
List<Employee> others = partitioned.get(false);
```

## Problem 11: Get comma-separated names of all employees

```java
String allNames = employees.stream()
    .map(Employee::getName)
    .collect(Collectors.joining(", "));
// "Alice, Bob, Charlie, David, Eve, Frank, Grace, Henry"
```

## Problem 12: Find all unique departments

```java
Set<String> departments = employees.stream()
    .map(Employee::getDepartment)
    .collect(Collectors.toSet());

// OR
List<String> departments2 = employees.stream()
    .map(Employee::getDepartment)
    .distinct()
    .collect(Collectors.toList());
```

## Problem 13: Find employees aged between 25 and 35

```java
List<Employee> filtered = employees.stream()
    .filter(e -> e.getAge() >= 25 && e.getAge() <= 35)
    .collect(Collectors.toList());
```

## Problem 14: Get employee with max salary in each city

```java
Map<String, Optional<Employee>> maxSalaryByCity = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getCity,
        Collectors.maxBy(Comparator.comparingDouble(Employee::getSalary))
    ));
```

## Problem 15: Convert list to map (id -> employee)

```java
Map<Integer, Employee> employeeMap = employees.stream()
    .collect(Collectors.toMap(
        Employee::getId,
        Function.identity()
    ));

// Handling duplicates (if ids were not unique)
Map<Integer, Employee> employeeMap2 = employees.stream()
    .collect(Collectors.toMap(
        Employee::getId,
        Function.identity(),
        (existing, replacement) -> existing  // Keep existing on conflict
    ));
```

## Problem 16: Find if any employee earns more than 90000

```java
boolean anyHighEarner = employees.stream()
    .anyMatch(e -> e.getSalary() > 90000);

// Check if ALL employees earn more than 50000
boolean allAbove50K = employees.stream()
    .allMatch(e -> e.getSalary() > 50000);

// Check if NO employee earns less than 50000
boolean noneBelow50K = employees.stream()
    .noneMatch(e -> e.getSalary() < 50000);
```

## Problem 17: Group employees by department and then by city

```java
Map<String, Map<String, List<Employee>>> byDeptAndCity = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::getDepartment,
        Collectors.groupingBy(Employee::getCity)
    ));
```

## Problem 18: Find department with maximum employees

```java
Optional<Map.Entry<String, Long>> maxDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment, Collectors.counting()))
    .entrySet()
    .stream()
    .max(Map.Entry.comparingByValue());
```

## Problem 19: Find nth highest salary

```java
public Optional<Double> findNthHighestSalary(List<Employee> employees, int n) {
    return employees.stream()
        .map(Employee::getSalary)
        .distinct()
        .sorted(Comparator.reverseOrder())
        .skip(n - 1)
        .findFirst();
}
```

## Problem 20: Calculate total, average, min, max salary in one go

```java
DoubleSummaryStatistics stats = employees.stream()
    .collect(Collectors.summarizingDouble(Employee::getSalary));

System.out.println("Count: " + stats.getCount());
System.out.println("Sum: " + stats.getSum());
System.out.println("Min: " + stats.getMin());
System.out.println("Max: " + stats.getMax());
System.out.println("Avg: " + stats.getAverage());
```

---

# 10. Tricky Follow-up Questions

## Q30: Can we reuse a Stream?

**Answer:** **No!** A stream can only be consumed once. Attempting to reuse throws `IllegalStateException`.

```java
Stream<String> stream = names.stream();
stream.forEach(System.out::println);  // OK
stream.forEach(System.out::println);  // ❌ IllegalStateException!

// Solution: Create new stream each time
Supplier<Stream<String>> streamSupplier = () -> names.stream();
streamSupplier.get().forEach(System.out::println);  // OK
streamSupplier.get().forEach(System.out::println);  // OK
```

## Q31: What happens if you call get() on empty Optional?

**Answer:** Throws `NoSuchElementException`. Always use `orElse()`, `orElseGet()`, or check with `isPresent()`.

## Q32: Can a functional interface extend another functional interface?

**Answer:** Yes, but only if the child doesn't add new abstract methods.

```java
@FunctionalInterface
interface A {
    void methodA();
}

@FunctionalInterface
interface B extends A { } // ✅ Valid - inherits single abstract method

@FunctionalInterface
interface C extends A {
    void methodC(); // ❌ Invalid - now has 2 abstract methods
}
```

## Q33: What's the difference between Collection.stream() and Arrays.stream()?

**Answer:**
- `Collection.stream()` works on collections (List, Set, etc.)
- `Arrays.stream()` works on arrays
- For primitive arrays, `Arrays.stream()` returns primitive streams (`IntStream`, etc.)

```java
List<Integer> list = Arrays.asList(1, 2, 3);
Stream<Integer> s1 = list.stream();

Integer[] array = {1, 2, 3};
Stream<Integer> s2 = Arrays.stream(array);

int[] primitiveArray = {1, 2, 3};
IntStream s3 = Arrays.stream(primitiveArray); // Returns IntStream, not Stream<Integer>
```

## Q34: What is the order of execution in a stream pipeline?

**Answer:** Elements are processed **one at a time through the whole pipeline** (not operation by operation).

```java
List.of(1, 2, 3, 4).stream()
    .peek(n -> System.out.println("Filter input: " + n))
    .filter(n -> n % 2 == 0)
    .peek(n -> System.out.println("Map input: " + n))
    .map(n -> n * 10)
    .forEach(n -> System.out.println("Final: " + n));

// Output:
// Filter input: 1
// Filter input: 2
// Map input: 2
// Final: 20
// Filter input: 3
// Filter input: 4
// Map input: 4
// Final: 40
```

## Q35: How does parallel stream handle shared mutable state?

**Answer:** It doesn't handle it safely! You must ensure thread safety yourself or avoid shared state.

```java
// ❌ DANGEROUS - Race condition
List<Integer> results = new ArrayList<>();
IntStream.range(0, 1000)
    .parallel()
    .forEach(i -> results.add(i)); // ArrayList is not thread-safe!

// ✅ SAFE - Using collect
List<Integer> results = IntStream.range(0, 1000)
    .parallel()
    .boxed()
    .collect(Collectors.toList());

// ✅ SAFE - Using thread-safe collection
List<Integer> results = Collections.synchronizedList(new ArrayList<>());
IntStream.range(0, 1000)
    .parallel()
    .forEach(i -> results.add(i));
```

## Q36: What is the difference between peek() and forEach()?

| peek() | forEach() |
|--------|-----------|
| Intermediate operation | Terminal operation |
| Returns a Stream | Returns void |
| Lazy - needs terminal op | Executes immediately |
| Use for debugging | Use for final consumption |

## Q37: Can Optional contain null?

**Answer:** `Optional.of(null)` throws NPE. Use `Optional.ofNullable(null)` which returns empty Optional.

## Q38: What's the difference between Comparator.comparing() and Comparator.comparingInt()?

**Answer:** `comparingInt()` avoids boxing/unboxing overhead for primitive comparisons.

```java
// Uses boxing/unboxing
Comparator<Employee> c1 = Comparator.comparing(Employee::getAge);

// More efficient - no boxing
Comparator<Employee> c2 = Comparator.comparingInt(Employee::getAge);
```

---

# 11. Quick Revision Cheat Sheet

## Functional Interfaces Quick Reference

| Interface | Input | Output | Method | Use Case |
|-----------|-------|--------|--------|----------|
| `Predicate<T>` | T | boolean | test(T) | Filtering |
| `Function<T,R>` | T | R | apply(T) | Transformation |
| `Consumer<T>` | T | void | accept(T) | Side effects |
| `Supplier<T>` | - | T | get() | Factory/Lazy |
| `UnaryOperator<T>` | T | T | apply(T) | Same type transform |
| `BinaryOperator<T>` | T, T | T | apply(T,T) | Combining |

## Stream Operations Quick Reference

### Intermediate (Lazy → Return Stream)
`filter`, `map`, `flatMap`, `distinct`, `sorted`, `peek`, `limit`, `skip`

### Terminal (Execute → Return Result)
`forEach`, `collect`, `reduce`, `count`, `min`, `max`, `findFirst`, `findAny`, `anyMatch`, `allMatch`, `noneMatch`, `toArray`

## Common Collectors

```java
toList()
toSet()
toMap(keyMapper, valueMapper)
joining(", ")
groupingBy(classifier)
partitioningBy(predicate)
counting()
summingInt(mapper)
averagingDouble(mapper)
summarizingInt(mapper)
maxBy(comparator)
minBy(comparator)
```

## Optional Methods

```java
Optional.of(value)
Optional.ofNullable(value)
Optional.empty()

.isPresent()
.isEmpty()
.ifPresent(consumer)
.get()
.orElse(default)
.orElseGet(supplier)
.orElseThrow(exceptionSupplier)
.map(function)
.flatMap(function)
.filter(predicate)
```

## Method Reference Types

| Type | Syntax | Lambda Equivalent |
|------|--------|-------------------|
| Static | `ClassName::staticMethod` | `x -> ClassName.staticMethod(x)` |
| Instance of particular object | `object::method` | `x -> object.method(x)` |
| Instance of arbitrary object | `ClassName::method` | `(obj, x) -> obj.method(x)` |
| Constructor | `ClassName::new` | `x -> new ClassName(x)` |

---

## Final Tips for Interview

1. **Practice typing these examples** - Don't just read, actually code them
2. **Understand the WHY** - Know why Java 8 features exist, not just how to use them
3. **Be ready for follow-ups** - Interviewers love asking "what if" scenarios
4. **Know the gotchas** - Parallel stream issues, Optional pitfalls, Stream reuse error
5. **Compare old vs new** - Be able to show before/after Java 8 code
6. **Performance awareness** - Know when to use parallel streams, primitive specializations

---

*Good luck with your interview! Remember: Java 8 is about writing cleaner, more readable, and more maintainable code. Show that you understand this philosophy, not just the syntax.*
