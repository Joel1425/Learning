# Java Interview Complete Guide for SDE-2 Backend

> A comprehensive guide covering Core Java concepts, OOP, SOLID, Memory Management, and common interview questions with code examples and explanations.

---

## Table of Contents

1. [OOP Concepts](#1-oop-concepts)
2. [SOLID Principles](#2-solid-principles)
3. [Dependency Injection](#3-dependency-injection)
4. [Inheritance & Diamond Problem](#4-inheritance--diamond-problem)
5. [Memory Management & JVM](#5-memory-management--jvm)
6. [Exception Handling](#6-exception-handling)
7. [Collections Framework](#7-collections-framework)
8. [String & Immutability](#8-string--immutability)
9. [Program Output Questions](#9-program-output-questions)
10. [Design Patterns in Java](#10-design-patterns-in-java)
11. [Generics & Type Erasure](#11-generics--type-erasure)
12. [Serialization](#12-serialization)
13. [Miscellaneous Tricky Questions](#13-miscellaneous-tricky-questions)

---

## 1. OOP Concepts

### 1.1 Four Pillars of OOP

```
┌─────────────────────────────────────────────────────────────┐
│                    OOP PILLARS                              │
├─────────────┬─────────────┬──────────────┬─────────────────┤
│ Encapsulation│ Abstraction │ Inheritance  │ Polymorphism    │
│             │             │              │                 │
│ Data hiding │ Hide complex│ Code reuse   │ Many forms      │
│ using access│ implementation│ IS-A relation│ Same interface │
│ modifiers   │ details     │              │ diff behavior   │
└─────────────┴─────────────┴──────────────┴─────────────────┘
```

---

### 1.2 Encapsulation

**Definition**: Bundling data (variables) and methods that operate on data within a single unit (class), restricting direct access to some components.

```java
// BAD - No encapsulation
public class Employee {
    public String name;
    public double salary;  // Anyone can set negative salary!
}

// GOOD - Proper encapsulation
public class Employee {
    private String name;
    private double salary;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        if (name != null && !name.isEmpty()) {
            this.name = name;
        }
    }

    public double getSalary() {
        return salary;
    }

    public void setSalary(double salary) {
        if (salary >= 0) {  // Validation logic
            this.salary = salary;
        }
    }
}
```

**Follow-up Q: Why use getters/setters instead of public fields?**
- **Validation**: Can add checks before setting values
- **Computed Properties**: Getter can return calculated values
- **Future Flexibility**: Can change internal implementation without breaking API
- **Debugging**: Can add breakpoints in setters
- **Read-only/Write-only**: Can provide only getter or only setter

---

### 1.3 Abstraction

**Definition**: Hiding complex implementation details and showing only necessary features.

```java
// Abstraction using Abstract Class
abstract class Payment {
    protected double amount;

    public Payment(double amount) {
        this.amount = amount;
    }

    // Abstract method - WHAT to do (no implementation)
    public abstract boolean processPayment();

    // Concrete method - shared behavior
    public void logTransaction() {
        System.out.println("Transaction of $" + amount + " logged");
    }
}

class CreditCardPayment extends Payment {
    private String cardNumber;

    public CreditCardPayment(double amount, String cardNumber) {
        super(amount);
        this.cardNumber = cardNumber;
    }

    @Override
    public boolean processPayment() {
        // Complex credit card processing hidden from user
        System.out.println("Processing credit card: " + maskCard());
        return true;
    }

    private String maskCard() {
        return "****" + cardNumber.substring(cardNumber.length() - 4);
    }
}

class UPIPayment extends Payment {
    private String upiId;

    public UPIPayment(double amount, String upiId) {
        super(amount);
        this.upiId = upiId;
    }

    @Override
    public boolean processPayment() {
        System.out.println("Processing UPI payment to: " + upiId);
        return true;
    }
}
```

```java
// Abstraction using Interface
interface Notifiable {
    void sendNotification(String message);
}

class EmailNotification implements Notifiable {
    @Override
    public void sendNotification(String message) {
        // Complex SMTP logic hidden
        System.out.println("Email sent: " + message);
    }
}

class SMSNotification implements Notifiable {
    @Override
    public void sendNotification(String message) {
        // Complex SMS gateway logic hidden
        System.out.println("SMS sent: " + message);
    }
}
```

**Follow-up Q: Abstract Class vs Interface - When to use which?**

| Aspect | Abstract Class | Interface |
|--------|---------------|-----------|
| **Multiple Inheritance** | Single class only | Multiple interfaces |
| **State (Fields)** | Can have instance variables | Only static final (constants) |
| **Constructors** | Yes | No |
| **Access Modifiers** | Any | public (default for methods) |
| **Use When** | IS-A relationship, shared code | CAN-DO capability, contract |

---

### 1.4 Inheritance

**Definition**: Mechanism where a new class inherits properties and behaviors from an existing class.

```java
// Base class
class Vehicle {
    protected String brand;
    protected int speed;

    public Vehicle(String brand) {
        this.brand = brand;
        this.speed = 0;
    }

    public void accelerate(int increment) {
        speed += increment;
        System.out.println(brand + " speed: " + speed + " km/h");
    }

    public void brake() {
        speed = Math.max(0, speed - 10);
    }
}

// Derived class
class Car extends Vehicle {
    private int numDoors;

    public Car(String brand, int numDoors) {
        super(brand);  // Call parent constructor
        this.numDoors = numDoors;
    }

    // Additional method specific to Car
    public void openTrunk() {
        System.out.println("Trunk opened");
    }

    // Override parent method
    @Override
    public void accelerate(int increment) {
        if (increment > 50) {
            System.out.println("Safety limit: max 50 km/h acceleration");
            increment = 50;
        }
        super.accelerate(increment);  // Call parent implementation
    }
}

// Usage
Car myCar = new Car("Toyota", 4);
myCar.accelerate(30);  // Inherited + overridden behavior
myCar.openTrunk();     // Car-specific behavior
```

**Types of Inheritance in Java:**

```
┌────────────────────────────────────────────────────────────────┐
│                    TYPES OF INHERITANCE                        │
├──────────────────┬────────────────────┬───────────────────────┤
│   Single         │   Multilevel       │   Hierarchical        │
│                  │                    │                       │
│   A              │   A                │        A              │
│   │              │   │                │      / │ \            │
│   B              │   B                │     B  C  D           │
│                  │   │                │                       │
│ class B          │   C                │                       │
│ extends A        │                    │                       │
├──────────────────┴────────────────────┴───────────────────────┤
│                                                                │
│   Multiple Inheritance (NOT supported with classes)            │
│                                                                │
│      A       B        ← ALLOWED with interfaces only!          │
│       \     /                                                  │
│         C                                                      │
│                                                                │
│   interface A { }                                              │
│   interface B { }                                              │
│   class C implements A, B { }                                  │
│                                                                │
└────────────────────────────────────────────────────────────────┘
```

---

### 1.5 Polymorphism

**Definition**: Ability of an object to take many forms. Same interface, different implementations.

#### Compile-time Polymorphism (Method Overloading)

```java
class Calculator {
    // Same method name, different parameters
    public int add(int a, int b) {
        return a + b;
    }

    public double add(double a, double b) {
        return a + b;
    }

    public int add(int a, int b, int c) {
        return a + b + c;
    }

    public String add(String a, String b) {
        return a + b;
    }
}

// Resolved at COMPILE time based on method signature
Calculator calc = new Calculator();
calc.add(1, 2);         // Calls int version
calc.add(1.5, 2.5);     // Calls double version
calc.add(1, 2, 3);      // Calls 3-param version
calc.add("Hello", " World");  // Calls String version
```

**Overloading Rules:**
- ✅ Different number of parameters
- ✅ Different types of parameters
- ✅ Different order of parameter types
- ❌ Only different return type (won't compile)
- ❌ Only different access modifier (won't compile)

#### Runtime Polymorphism (Method Overriding)

```java
class Animal {
    public void makeSound() {
        System.out.println("Some generic sound");
    }
}

class Dog extends Animal {
    @Override
    public void makeSound() {
        System.out.println("Woof! Woof!");
    }
}

class Cat extends Animal {
    @Override
    public void makeSound() {
        System.out.println("Meow!");
    }
}

// Runtime polymorphism in action
Animal myAnimal;

myAnimal = new Dog();
myAnimal.makeSound();  // Output: Woof! Woof! (Resolved at RUNTIME)

myAnimal = new Cat();
myAnimal.makeSound();  // Output: Meow! (Resolved at RUNTIME)

// Polymorphic array
Animal[] animals = {new Dog(), new Cat(), new Dog()};
for (Animal animal : animals) {
    animal.makeSound();  // Each calls its own implementation
}
```

**Overriding Rules:**
- ✅ Same method signature (name + parameters)
- ✅ Same or covariant return type
- ✅ Same or less restrictive access modifier
- ❌ Cannot override `static`, `final`, or `private` methods
- ❌ Cannot throw broader checked exception

---

### 1.6 Association, Aggregation, Composition

```
┌──────────────────────────────────────────────────────────────────┐
│                    RELATIONSHIP TYPES                            │
├────────────────┬────────────────┬────────────────────────────────┤
│  Association   │  Aggregation   │  Composition                   │
│                │    (HAS-A)     │    (PART-OF)                   │
│                │                │                                │
│  Teacher ───   │  Department ◇──│  House ◆────── Room            │
│         │      │             │  │                                │
│  Student ───   │      Teacher   │  If House is destroyed,        │
│                │                │  Room is also destroyed        │
│ Loose coupling │ Weak ownership │ Strong ownership               │
│ Both exist     │ Part can exist │ Part cannot exist              │
│ independently  │ independently  │ independently                  │
└────────────────┴────────────────┴────────────────────────────────┘
```

```java
// ASSOCIATION - Teacher and Student (loose relationship)
class Student {
    private String name;
    // Student exists independently
}

class Teacher {
    private String name;
    private List<Student> students;  // Association
    // Teacher exists independently
}

// AGGREGATION - Department and Teacher (weak HAS-A)
class Department {
    private String name;
    private List<Teacher> teachers;  // Aggregation

    public Department(String name) {
        this.name = name;
        this.teachers = new ArrayList<>();
    }

    public void addTeacher(Teacher teacher) {
        // Teacher created OUTSIDE, passed in
        // Teacher can exist without Department
        teachers.add(teacher);
    }
}

// Usage - Teacher exists independently
Teacher t1 = new Teacher("John");
Department dept = new Department("CS");
dept.addTeacher(t1);
// If dept is deleted, t1 still exists

// COMPOSITION - House and Room (strong PART-OF)
class Room {
    private String name;
    private int area;

    public Room(String name, int area) {
        this.name = name;
        this.area = area;
    }
}

class House {
    private String address;
    private List<Room> rooms;  // Composition

    public House(String address) {
        this.address = address;
        // Rooms created INSIDE House
        // Rooms cannot exist without House
        this.rooms = new ArrayList<>();
        this.rooms.add(new Room("Living Room", 200));
        this.rooms.add(new Room("Bedroom", 150));
    }
}

// Usage - Room lifecycle tied to House
House house = new House("123 Main St");
// If house is garbage collected, rooms are also garbage collected
```

**Follow-up Q: How is Composition preferred over Inheritance?**

```java
// PROBLEM with Inheritance - Tight coupling
class ArrayList<E> {
    public boolean add(E e) { /* ... */ }
    public boolean addAll(Collection<E> c) { /* adds each element */ }
}

class CountingList<E> extends ArrayList<E> {
    private int addCount = 0;

    @Override
    public boolean add(E e) {
        addCount++;
        return super.add(e);
    }

    @Override
    public boolean addAll(Collection<E> c) {
        addCount += c.size();
        return super.addAll(c);  // BUG! addAll internally calls add()
        // addCount gets incremented TWICE for each element!
    }
}

// SOLUTION with Composition - Loose coupling
class CountingList<E> {
    private final List<E> list = new ArrayList<>();  // Composition
    private int addCount = 0;

    public boolean add(E e) {
        addCount++;
        return list.add(e);
    }

    public boolean addAll(Collection<E> c) {
        addCount += c.size();
        return list.addAll(c);  // No double counting!
    }

    public int getAddCount() {
        return addCount;
    }
}
```

---

## 2. SOLID Principles

```
┌────────────────────────────────────────────────────────────────┐
│                      SOLID PRINCIPLES                          │
├───────┬────────────────────────────────────────────────────────┤
│   S   │  Single Responsibility - One class, one reason to change│
│   O   │  Open/Closed - Open for extension, closed for modification│
│   L   │  Liskov Substitution - Subtypes must be substitutable  │
│   I   │  Interface Segregation - Many specific interfaces      │
│   D   │  Dependency Inversion - Depend on abstractions         │
└───────┴────────────────────────────────────────────────────────┘
```

### 2.1 Single Responsibility Principle (SRP)

> A class should have only one reason to change.

```java
// BAD - Multiple responsibilities
class Employee {
    private String name;
    private double salary;

    public void calculatePay() { /* payroll logic */ }
    public void saveToDatabase() { /* database logic */ }
    public void generateReport() { /* reporting logic */ }
}
// If payroll rules change, we modify Employee
// If database changes, we modify Employee
// If report format changes, we modify Employee

// GOOD - Single responsibility each
class Employee {
    private String name;
    private double salary;
    // Only employee data and behavior
    public String getName() { return name; }
    public double getSalary() { return salary; }
}

class PayrollCalculator {
    public double calculatePay(Employee emp) {
        // Only payroll calculation logic
        return emp.getSalary() * 1.1;  // with bonus
    }
}

class EmployeeRepository {
    public void save(Employee emp) {
        // Only database logic
    }
}

class EmployeeReportGenerator {
    public String generateReport(Employee emp) {
        // Only reporting logic
        return "Report for: " + emp.getName();
    }
}
```

---

### 2.2 Open/Closed Principle (OCP)

> Software entities should be open for extension, but closed for modification.

```java
// BAD - Need to modify class for each new shape
class AreaCalculator {
    public double calculateArea(Object shape) {
        if (shape instanceof Rectangle) {
            Rectangle r = (Rectangle) shape;
            return r.width * r.height;
        } else if (shape instanceof Circle) {
            Circle c = (Circle) shape;
            return Math.PI * c.radius * c.radius;
        }
        // Adding Triangle? Modify this class! VIOLATES OCP
        return 0;
    }
}

// GOOD - Open for extension, closed for modification
interface Shape {
    double calculateArea();
}

class Rectangle implements Shape {
    private double width, height;

    public Rectangle(double width, double height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public double calculateArea() {
        return width * height;
    }
}

class Circle implements Shape {
    private double radius;

    public Circle(double radius) {
        this.radius = radius;
    }

    @Override
    public double calculateArea() {
        return Math.PI * radius * radius;
    }
}

// Adding Triangle? Just create new class, no modification needed!
class Triangle implements Shape {
    private double base, height;

    public Triangle(double base, double height) {
        this.base = base;
        this.height = height;
    }

    @Override
    public double calculateArea() {
        return 0.5 * base * height;
    }
}

class AreaCalculator {
    public double calculateArea(Shape shape) {
        return shape.calculateArea();  // Works for ANY shape!
    }
}
```

---

### 2.3 Liskov Substitution Principle (LSP)

> Objects of a superclass should be replaceable with objects of subclasses without breaking the application.

```java
// BAD - Violates LSP
class Rectangle {
    protected int width;
    protected int height;

    public void setWidth(int width) { this.width = width; }
    public void setHeight(int height) { this.height = height; }
    public int getArea() { return width * height; }
}

class Square extends Rectangle {
    @Override
    public void setWidth(int width) {
        this.width = width;
        this.height = width;  // Unexpected behavior!
    }

    @Override
    public void setHeight(int height) {
        this.height = height;
        this.width = height;  // Unexpected behavior!
    }
}

// This breaks!
void testRectangle(Rectangle r) {
    r.setWidth(5);
    r.setHeight(4);
    assert r.getArea() == 20;  // FAILS for Square! Returns 16
}

// GOOD - Proper abstraction
interface Shape {
    int getArea();
}

class Rectangle implements Shape {
    private final int width;
    private final int height;

    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public int getArea() { return width * height; }
}

class Square implements Shape {
    private final int side;

    public Square(int side) {
        this.side = side;
    }

    @Override
    public int getArea() { return side * side; }
}
```

**LSP Violation Signs:**
- Subclass throws exception for parent method
- Subclass returns null when parent doesn't
- Subclass has stricter preconditions
- Subclass has weaker postconditions

---

### 2.4 Interface Segregation Principle (ISP)

> Clients should not be forced to depend on interfaces they don't use.

```java
// BAD - Fat interface
interface Worker {
    void work();
    void eat();
    void sleep();
}

class HumanWorker implements Worker {
    public void work() { System.out.println("Working"); }
    public void eat() { System.out.println("Eating"); }
    public void sleep() { System.out.println("Sleeping"); }
}

class RobotWorker implements Worker {
    public void work() { System.out.println("Working"); }
    public void eat() { /* Robots don't eat! Empty or throw? */ }
    public void sleep() { /* Robots don't sleep! */ }
}

// GOOD - Segregated interfaces
interface Workable {
    void work();
}

interface Eatable {
    void eat();
}

interface Sleepable {
    void sleep();
}

class HumanWorker implements Workable, Eatable, Sleepable {
    public void work() { System.out.println("Working"); }
    public void eat() { System.out.println("Eating"); }
    public void sleep() { System.out.println("Sleeping"); }
}

class RobotWorker implements Workable {
    public void work() { System.out.println("Working 24/7"); }
    // No need to implement eat() or sleep()!
}
```

---

### 2.5 Dependency Inversion Principle (DIP)

> High-level modules should not depend on low-level modules. Both should depend on abstractions.

```java
// BAD - High-level depends on low-level
class MySQLDatabase {
    public void save(String data) {
        System.out.println("Saving to MySQL: " + data);
    }
}

class UserService {
    private MySQLDatabase database;  // Direct dependency on concrete class

    public UserService() {
        this.database = new MySQLDatabase();  // Tight coupling!
    }

    public void saveUser(String user) {
        database.save(user);
    }
}
// What if we want to switch to PostgreSQL? Must modify UserService!

// GOOD - Both depend on abstraction
interface Database {
    void save(String data);
}

class MySQLDatabase implements Database {
    @Override
    public void save(String data) {
        System.out.println("Saving to MySQL: " + data);
    }
}

class PostgreSQLDatabase implements Database {
    @Override
    public void save(String data) {
        System.out.println("Saving to PostgreSQL: " + data);
    }
}

class UserService {
    private Database database;  // Depends on abstraction

    // Dependency Injection via constructor
    public UserService(Database database) {
        this.database = database;
    }

    public void saveUser(String user) {
        database.save(user);
    }
}

// Usage - Easy to swap implementations
Database mysql = new MySQLDatabase();
Database postgres = new PostgreSQLDatabase();

UserService service1 = new UserService(mysql);
UserService service2 = new UserService(postgres);
```

```
┌─────────────────────────────────────────────────────────────┐
│                 DEPENDENCY DIRECTION                        │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   BAD:                     GOOD:                            │
│                                                             │
│   UserService              UserService                      │
│       │                        │                            │
│       ▼                        ▼                            │
│   MySQLDatabase            <<interface>>                    │
│                              Database                       │
│                                △                            │
│                           ┌────┴────┐                       │
│                           │         │                       │
│                       MySQL    PostgreSQL                   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## 3. Dependency Injection

### 3.1 What is Dependency Injection?

> A design pattern where dependencies are provided to an object rather than the object creating them itself.

```
┌─────────────────────────────────────────────────────────────┐
│              WITHOUT vs WITH DEPENDENCY INJECTION            │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│   WITHOUT DI:                  WITH DI:                     │
│                                                             │
│   class Service {              class Service {              │
│     Repository repo;             Repository repo;           │
│                                                             │
│     Service() {                  Service(Repository repo) { │
│       repo = new MySQLRepo();      this.repo = repo;       │
│     }                            }                          │
│   }                            }                            │
│                                                             │
│   • Service controls creation  • External control           │
│   • Tight coupling             • Loose coupling             │
│   • Hard to test               • Easy to mock/test          │
│   • Hard to change             • Easy to swap               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Types of Dependency Injection

```java
interface MessageService {
    void sendMessage(String message);
}

class EmailService implements MessageService {
    @Override
    public void sendMessage(String message) {
        System.out.println("Email: " + message);
    }
}

class SMSService implements MessageService {
    @Override
    public void sendMessage(String message) {
        System.out.println("SMS: " + message);
    }
}
```

#### 1. Constructor Injection (Recommended)

```java
class NotificationManager {
    private final MessageService messageService;  // Can be final!

    // Dependency injected via constructor
    public NotificationManager(MessageService messageService) {
        this.messageService = messageService;
    }

    public void notify(String message) {
        messageService.sendMessage(message);
    }
}

// Usage
MessageService emailService = new EmailService();
NotificationManager manager = new NotificationManager(emailService);
manager.notify("Hello!");
```

**Advantages:**
- Dependencies are required (no null issues)
- Can use `final` fields (immutability)
- Easy to see all dependencies in constructor
- Object is fully initialized after construction

#### 2. Setter Injection

```java
class NotificationManager {
    private MessageService messageService;

    // Dependency injected via setter
    public void setMessageService(MessageService messageService) {
        this.messageService = messageService;
    }

    public void notify(String message) {
        if (messageService == null) {
            throw new IllegalStateException("MessageService not set!");
        }
        messageService.sendMessage(message);
    }
}

// Usage
NotificationManager manager = new NotificationManager();
manager.setMessageService(new SMSService());
manager.notify("Hello!");
```

**Use When:**
- Optional dependencies
- Dependencies can change at runtime
- Circular dependencies (use with caution)

#### 3. Interface Injection

```java
interface MessageServiceAware {
    void setMessageService(MessageService messageService);
}

class NotificationManager implements MessageServiceAware {
    private MessageService messageService;

    @Override
    public void setMessageService(MessageService messageService) {
        this.messageService = messageService;
    }

    public void notify(String message) {
        messageService.sendMessage(message);
    }
}
```

### 3.3 DI Container (Spring Example)

```java
// Spring handles DI automatically with annotations
@Service
public class NotificationManager {
    private final MessageService messageService;

    @Autowired  // Spring injects the dependency
    public NotificationManager(MessageService messageService) {
        this.messageService = messageService;
    }
}

@Component
public class EmailService implements MessageService {
    @Override
    public void sendMessage(String message) {
        System.out.println("Email: " + message);
    }
}
```

### 3.4 Benefits of DI

| Benefit | Explanation |
|---------|-------------|
| **Loose Coupling** | Classes don't create dependencies, just receive them |
| **Testability** | Easy to inject mocks for unit testing |
| **Flexibility** | Swap implementations without changing client code |
| **Single Responsibility** | Object creation separated from usage |
| **Maintainability** | Changes in one module don't ripple through |

```java
// Easy testing with DI
class MockMessageService implements MessageService {
    private String lastMessage;

    @Override
    public void sendMessage(String message) {
        this.lastMessage = message;  // Just capture, don't send
    }

    public String getLastMessage() { return lastMessage; }
}

@Test
void testNotification() {
    MockMessageService mockService = new MockMessageService();
    NotificationManager manager = new NotificationManager(mockService);

    manager.notify("Test message");

    assertEquals("Test message", mockService.getLastMessage());
}
```

---

## 4. Inheritance & Diamond Problem

### 4.1 The Diamond Problem

```
┌──────────────────────────────────────────────────────────────┐
│                    THE DIAMOND PROBLEM                       │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│                      interface A                             │
│                     default void foo()                       │
│                           /\                                 │
│                          /  \                                │
│                         /    \                               │
│               interface B    interface C                     │
│                         \    /                               │
│                          \  /                                │
│                           \/                                 │
│                      class D implements B, C                 │
│                                                              │
│   PROBLEM: When D calls foo(), which version?                │
│            B's foo() or C's foo()?                           │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

### 4.2 Why Java Doesn't Allow Multiple Class Inheritance

```java
// HYPOTHETICAL - NOT ALLOWED IN JAVA
class A {
    void display() { System.out.println("A"); }
}

class B extends A {
    void display() { System.out.println("B"); }
}

class C extends A {
    void display() { System.out.println("C"); }
}

// NOT ALLOWED!
class D extends B, C {  // Compilation Error!
    // Which display() should D inherit?
}
```

### 4.3 Diamond Problem with Interfaces (Java 8+)

```java
interface Vehicle {
    default void start() {
        System.out.println("Vehicle starting...");
    }
}

interface Car extends Vehicle {
    @Override
    default void start() {
        System.out.println("Car starting with key...");
    }
}

interface ElectricVehicle extends Vehicle {
    @Override
    default void start() {
        System.out.println("Electric vehicle starting silently...");
    }
}

// DIAMOND! TeslaCar implements both Car and ElectricVehicle
class TeslaCar implements Car, ElectricVehicle {
    // COMPILATION ERROR if we don't override start()!
    // "class TeslaCar inherits unrelated defaults for start()"

    @Override
    public void start() {
        // MUST explicitly choose or provide own implementation

        // Option 1: Use specific parent's implementation
        Car.super.start();

        // Option 2: Use other parent's implementation
        // ElectricVehicle.super.start();

        // Option 3: Combine both
        // Car.super.start();
        // ElectricVehicle.super.start();

        // Option 4: Completely new implementation
        // System.out.println("Tesla starting...");
    }
}
```

### 4.4 Resolution Rules

```
┌──────────────────────────────────────────────────────────────┐
│            JAVA DEFAULT METHOD RESOLUTION RULES              │
├──────────────────────────────────────────────────────────────┤
│                                                              │
│   1. CLASSES ALWAYS WIN                                      │
│      Class method beats interface default method             │
│                                                              │
│   2. MORE SPECIFIC INTERFACE WINS                            │
│      Sub-interface default beats parent interface default    │
│                                                              │
│   3. EXPLICIT OVERRIDE REQUIRED                              │
│      If conflict can't be resolved, must override explicitly │
│                                                              │
└──────────────────────────────────────────────────────────────┘
```

```java
// Rule 1: Class wins over interface
interface A {
    default void hello() { System.out.println("A"); }
}

class B {
    public void hello() { System.out.println("B"); }
}

class C extends B implements A {
    // No need to override - B's hello() is used
}

new C().hello();  // Output: "B"

// Rule 2: More specific interface wins
interface Parent {
    default void greet() { System.out.println("Parent"); }
}

interface Child extends Parent {
    @Override
    default void greet() { System.out.println("Child"); }
}

class MyClass implements Parent, Child {
    // No override needed - Child is more specific
}

new MyClass().greet();  // Output: "Child"
```

### 4.5 Practical Example: Multiple Interface Conflict

```java
interface Printable {
    default void print() {
        System.out.println("Printing document...");
    }
}

interface Showable {
    default void print() {  // Same signature!
        System.out.println("Showing on screen...");
    }
}

class Document implements Printable, Showable {
    // MUST override to resolve conflict
    @Override
    public void print() {
        System.out.println("Document output:");
        Printable.super.print();  // Call Printable's version
        Showable.super.print();   // Call Showable's version
    }
}

// Usage
Document doc = new Document();
doc.print();
// Output:
// Document output:
// Printing document...
// Showing on screen...
```

### 4.6 Interview Follow-ups

**Q: Why did Java 8 add default methods if they create diamond problem?**

A: Benefits outweigh the risks:
- **Backward Compatibility**: Add methods to interfaces without breaking existing implementations
- **Code Reuse**: Share common code in interfaces
- **Example**: `Collection.forEach()` was added without modifying all implementations

**Q: What's the difference between abstract class and interface with default methods?**

| Feature | Abstract Class | Interface |
|---------|---------------|-----------|
| **Multiple Inheritance** | No | Yes |
| **State (fields)** | Instance variables | Only static final |
| **Constructors** | Yes | No |
| **Access Modifiers** | Any | public only |

---

## 5. Memory Management & JVM

### 5.1 JVM Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        JVM ARCHITECTURE                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   .java ──► javac ──► .class (bytecode)                        │
│                            │                                    │
│                            ▼                                    │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                    CLASS LOADER                          │  │
│   │  Bootstrap ──► Extension ──► Application                 │  │
│   └─────────────────────────────────────────────────────────┘  │
│                            │                                    │
│                            ▼                                    │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                    RUNTIME DATA AREAS                    │  │
│   │                                                          │  │
│   │   ┌─────────┐  ┌─────────┐  ┌─────────────────────────┐ │  │
│   │   │  HEAP   │  │  STACK  │  │  METHOD AREA (Metaspace)│ │  │
│   │   │         │  │         │  │                         │ │  │
│   │   │ Objects │  │ Frames  │  │ Class metadata          │ │  │
│   │   │ Arrays  │  │ Local   │  │ Static variables        │ │  │
│   │   │         │  │ vars    │  │ Method bytecode         │ │  │
│   │   └─────────┘  └─────────┘  └─────────────────────────┘ │  │
│   │                                                          │  │
│   │   ┌─────────────────┐  ┌───────────────────────────┐    │  │
│   │   │ PC REGISTERS    │  │ NATIVE METHOD STACKS      │    │  │
│   │   │ (per thread)    │  │ (for native code)         │    │  │
│   │   └─────────────────┘  └───────────────────────────┘    │  │
│   └─────────────────────────────────────────────────────────┘  │
│                            │                                    │
│                            ▼                                    │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                  EXECUTION ENGINE                        │  │
│   │  Interpreter ◄──► JIT Compiler ◄──► Garbage Collector   │  │
│   └─────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Memory Areas in Detail

#### Stack Memory

```java
public class StackExample {
    public static void main(String[] args) {
        int x = 10;           // x stored in stack
        int y = 20;           // y stored in stack
        int result = add(x, y);  // New frame pushed to stack
        System.out.println(result);
    }  // Main frame popped when method ends

    public static int add(int a, int b) {  // New stack frame
        int sum = a + b;     // sum is local variable in this frame
        return sum;
    }  // Frame popped, sum is gone
}
```

```
┌─────────────────────────────────────────────────────────────┐
│                    STACK (per thread)                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌──────────────────────────────────────┐                  │
│   │ add() Frame                          │  ◄── TOP         │
│   │   a = 10                             │                  │
│   │   b = 20                             │                  │
│   │   sum = 30                           │                  │
│   └──────────────────────────────────────┘                  │
│   ┌──────────────────────────────────────┐                  │
│   │ main() Frame                         │                  │
│   │   args = ref to String[]             │                  │
│   │   x = 10                             │                  │
│   │   y = 20                             │                  │
│   │   result = (waiting)                 │                  │
│   └──────────────────────────────────────┘                  │
│                                                              │
│   • LIFO structure (Last In First Out)                      │
│   • Each thread has its own stack                           │
│   • Stores: primitives, references, method calls            │
│   • Fixed size (StackOverflowError if exceeded)             │
│   • Automatically managed (no GC needed)                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### Heap Memory

```java
public class HeapExample {
    public static void main(String[] args) {
        // Objects created in HEAP
        Person p1 = new Person("Alice");  // Object in heap, p1 (ref) in stack
        Person p2 = new Person("Bob");    // Another object in heap

        int[] numbers = {1, 2, 3};        // Array object in heap

        p1 = null;  // Object becomes eligible for GC
    }
}

class Person {
    String name;  // Reference stored in object (heap)
    int age;      // Primitive stored in object (heap)

    Person(String name) {
        this.name = name;
    }
}
```

```
┌─────────────────────────────────────────────────────────────┐
│                        HEAP MEMORY                           │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌───────────────────────────────────────────────────────┐ │
│   │              YOUNG GENERATION                          │ │
│   │  ┌─────────────┐  ┌──────────┐  ┌──────────┐         │ │
│   │  │    EDEN     │  │    S0    │  │    S1    │         │ │
│   │  │             │  │(Survivor)│  │(Survivor)│         │ │
│   │  │ New objects │  │          │  │          │         │ │
│   │  │ created here│  │          │  │          │         │ │
│   │  └─────────────┘  └──────────┘  └──────────┘         │ │
│   │                                                        │ │
│   │  Minor GC collects here (frequent, fast)              │ │
│   └───────────────────────────────────────────────────────┘ │
│                                                              │
│   ┌───────────────────────────────────────────────────────┐ │
│   │              OLD GENERATION (Tenured)                  │ │
│   │                                                        │ │
│   │  Long-lived objects that survived multiple GC cycles  │ │
│   │                                                        │ │
│   │  Major GC (Full GC) collects here (less frequent)     │ │
│   └───────────────────────────────────────────────────────┘ │
│                                                              │
│   • Shared across all threads                               │
│   • OutOfMemoryError if full                                │
│   • Managed by Garbage Collector                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.3 Garbage Collection

```
┌─────────────────────────────────────────────────────────────┐
│                    OBJECT LIFECYCLE                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   1. Creation      Object obj = new Object();               │
│         │                                                    │
│         ▼                                                    │
│   2. In Use        obj.doSomething();                       │
│         │                                                    │
│         ▼                                                    │
│   3. Unreachable   obj = null;  // No references            │
│         │                                                    │
│         ▼                                                    │
│   4. Collected     GC reclaims memory                       │
│         │                                                    │
│         ▼                                                    │
│   5. Finalized     finalize() called (deprecated)           │
│         │                                                    │
│         ▼                                                    │
│   6. Deallocated   Memory returned to heap                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

#### GC Algorithms

```java
// JVM GC Options (command line)
// -XX:+UseSerialGC          - Single threaded, simple
// -XX:+UseParallelGC        - Multiple threads, throughput focus
// -XX:+UseConcMarkSweepGC   - Low pause (deprecated in Java 9+)
// -XX:+UseG1GC              - Default in Java 9+, balanced
// -XX:+UseZGC               - Ultra-low latency (Java 11+)
// -XX:+UseShenandoahGC      - Low pause (Java 12+)
```

**G1 Garbage Collector (Default):**

```
┌─────────────────────────────────────────────────────────────┐
│                   G1 GC HEAP LAYOUT                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐        │
│   │ E │ E │ S │ O │ O │ E │ O │ O │ O │ H │ E │ S │        │
│   └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘        │
│                                                              │
│   E = Eden (new objects)                                    │
│   S = Survivor                                              │
│   O = Old                                                   │
│   H = Humongous (large objects spanning regions)            │
│                                                              │
│   • Heap divided into equal-sized regions                   │
│   • Each region can be any type                             │
│   • GC collects regions with most garbage first             │
│   • Predictable pause times                                 │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 5.4 Memory Leaks in Java

```java
// Common causes of memory leaks

// 1. Static Collections
class LeakyClass {
    private static List<Object> cache = new ArrayList<>();

    public void addToCache(Object obj) {
        cache.add(obj);  // Never removed, grows forever!
    }
}

// 2. Unclosed Resources
void leakyMethod() {
    Connection conn = DriverManager.getConnection(url);
    // If exception occurs, connection never closed!
    conn.executeQuery();
}

// FIXED with try-with-resources
void safeMethod() {
    try (Connection conn = DriverManager.getConnection(url)) {
        conn.executeQuery();
    }  // Auto-closed, even if exception
}

// 3. Inner Class Holding Reference
class Outer {
    private byte[] data = new byte[1024 * 1024];  // 1MB

    class Inner {
        // Inner class holds implicit reference to Outer
        // Even if Inner is needed, Outer can't be GC'd
    }
}

// FIXED with static inner class
class Outer {
    private byte[] data = new byte[1024 * 1024];

    static class Inner {
        // No implicit reference to Outer
    }
}

// 4. Listeners not removed
class Publisher {
    private List<Listener> listeners = new ArrayList<>();

    public void addListener(Listener l) {
        listeners.add(l);
    }

    // Missing removeListener method!
    // Listeners are never released
}
```

### 5.5 Stack vs Heap Comparison

| Feature | Stack | Heap |
|---------|-------|------|
| **Storage** | Primitives, references | Objects, arrays |
| **Thread Safety** | Thread-local (each thread has own) | Shared across threads |
| **Access Speed** | Faster | Slower |
| **Memory Management** | Automatic (LIFO) | Garbage Collected |
| **Size** | Limited (usually 1MB) | Large (configurable) |
| **Error** | StackOverflowError | OutOfMemoryError |

### 5.6 Common Interview Questions

**Q: When is an object eligible for garbage collection?**

```java
// Object is eligible when no reachable references exist

public void example() {
    Object obj1 = new Object();  // Object 1 created
    Object obj2 = new Object();  // Object 2 created

    obj1 = obj2;    // Object 1 now unreachable - ELIGIBLE
    obj2 = null;    // obj1 still references Object 2 - NOT eligible
    obj1 = null;    // Now Object 2 is unreachable - ELIGIBLE
}

// Island of Isolation
class Node {
    Node next;
}

void islandExample() {
    Node a = new Node();
    Node b = new Node();
    a.next = b;
    b.next = a;  // Circular reference

    a = null;
    b = null;
    // Both objects reference each other but unreachable from stack
    // BOTH are eligible for GC (JVM detects islands)
}
```

**Q: Can we force garbage collection?**

```java
System.gc();           // SUGGESTS GC, doesn't guarantee
Runtime.getRuntime().gc();  // Same as above

// JVM may or may not run GC
// Never rely on this in production code!
```

---

## 6. Exception Handling

### 6.1 Exception Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                    EXCEPTION HIERARCHY                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                        Throwable                             │
│                        /       \                             │
│                       /         \                            │
│                    Error      Exception                      │
│                      │           │                           │
│              (Unchecked)    ┌────┴────┐                     │
│                             │         │                      │
│              • OutOfMemory  │    RuntimeException            │
│              • StackOverflow│    (Unchecked)                │
│              • VirtualMachine│                               │
│                             │   • NullPointer               │
│                      Checked │   • ArrayIndexOutOfBounds    │
│                      Exceptions  • ClassCast                │
│                             │   • ArithmeticException       │
│                      • IOException                          │
│                      • SQLException                         │
│                      • FileNotFoundException                │
│                      • ClassNotFoundException               │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Checked vs Unchecked Exceptions

```java
// CHECKED EXCEPTION - Must handle or declare
public void readFile(String path) throws IOException {  // Must declare
    FileReader file = new FileReader(path);  // FileNotFoundException is checked
}

// Must be caught or thrown
public void caller() {
    try {
        readFile("test.txt");
    } catch (IOException e) {
        e.printStackTrace();
    }
}

// UNCHECKED EXCEPTION - Optional handling
public int divide(int a, int b) {
    return a / b;  // ArithmeticException if b is 0
    // No need to declare or catch
}
```

### 6.3 try-catch-finally Flow

```java
public class ExceptionFlow {
    public static void main(String[] args) {
        System.out.println(testMethod());  // What's the output?
    }

    public static int testMethod() {
        try {
            System.out.println("try block");
            return 1;
        } catch (Exception e) {
            System.out.println("catch block");
            return 2;
        } finally {
            System.out.println("finally block");
            // return 3;  // BAD PRACTICE - overrides try/catch return!
        }
    }
}

// Output:
// try block
// finally block
// Returns: 1
```

### 6.4 try-with-resources (Java 7+)

```java
// Before Java 7 - verbose and error-prone
BufferedReader reader = null;
try {
    reader = new BufferedReader(new FileReader("file.txt"));
    return reader.readLine();
} catch (IOException e) {
    e.printStackTrace();
} finally {
    if (reader != null) {
        try {
            reader.close();
        } catch (IOException e) {
            e.printStackTrace();
        }
    }
}

// Java 7+ with try-with-resources
try (BufferedReader reader = new BufferedReader(new FileReader("file.txt"))) {
    return reader.readLine();
} catch (IOException e) {
    e.printStackTrace();
}
// reader.close() called automatically!

// Multiple resources (closed in reverse order)
try (FileInputStream fis = new FileInputStream("input.txt");
     FileOutputStream fos = new FileOutputStream("output.txt")) {
    // Use both streams
}  // fos closed first, then fis
```

### 6.5 Custom Exceptions

```java
// Checked Custom Exception
public class InsufficientFundsException extends Exception {
    private double amount;

    public InsufficientFundsException(double amount) {
        super("Insufficient funds: need " + amount + " more");
        this.amount = amount;
    }

    public double getAmount() {
        return amount;
    }
}

// Unchecked Custom Exception
public class InvalidUserException extends RuntimeException {
    public InvalidUserException(String message) {
        super(message);
    }

    public InvalidUserException(String message, Throwable cause) {
        super(message, cause);  // Chain the original exception
    }
}

// Usage
public class BankAccount {
    private double balance;

    public void withdraw(double amount) throws InsufficientFundsException {
        if (amount > balance) {
            throw new InsufficientFundsException(amount - balance);
        }
        balance -= amount;
    }
}
```

### 6.6 Exception Best Practices

```java
// 1. Catch specific exceptions, not generic
// BAD
try {
    // code
} catch (Exception e) {  // Too broad!
    e.printStackTrace();
}

// GOOD
try {
    // code
} catch (FileNotFoundException e) {
    // Handle missing file
} catch (IOException e) {
    // Handle other IO issues
}

// 2. Don't swallow exceptions
// BAD
try {
    riskyOperation();
} catch (Exception e) {
    // Empty - exception lost!
}

// GOOD
try {
    riskyOperation();
} catch (Exception e) {
    logger.error("Operation failed", e);
    throw new ServiceException("Failed to process", e);
}

// 3. Use exception chaining
try {
    lowLevelOperation();
} catch (SQLException e) {
    throw new DataAccessException("Database error", e);  // Preserve cause
}

// 4. Clean up resources in finally or use try-with-resources
// Already covered above

// 5. Throw early, catch late
public void processFile(String filename) {
    if (filename == null) {
        throw new IllegalArgumentException("Filename cannot be null");  // Throw early
    }
    // Rest of processing
}
```

### 6.7 Tricky Interview Questions

```java
// Q: What happens here?
public static int puzzle() {
    try {
        return 1;
    } finally {
        return 2;  // WARNING: Finally return overrides try return!
    }
}
System.out.println(puzzle());  // Output: 2

// Q: What if finally throws exception?
public static void puzzle2() throws Exception {
    try {
        throw new RuntimeException("Try exception");
    } finally {
        throw new Exception("Finally exception");
    }
}
// RuntimeException is LOST! Finally exception is thrown.

// Q: Can finally block NOT execute?
public static void main(String[] args) {
    try {
        System.exit(0);  // JVM terminates
    } finally {
        System.out.println("This won't print!");
    }
}
// Also won't execute if:
// - Infinite loop in try block
// - JVM crash
// - Thread is killed
```

---

## 7. Collections Framework

### 7.1 Collections Hierarchy

```
┌─────────────────────────────────────────────────────────────┐
│                 COLLECTIONS FRAMEWORK                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│                       Iterable<E>                            │
│                           │                                  │
│                     Collection<E>                            │
│                    /      │       \                          │
│                   /       │        \                         │
│              List<E>   Set<E>    Queue<E>                   │
│                │         │          │                        │
│         ┌──────┼──────┐  │    ┌─────┴─────┐                │
│         │      │      │  │    │           │                 │
│     ArrayList  │  Vector │  Deque<E>  PriorityQueue        │
│              │      │    │      │                           │
│         LinkedList  │    │  ArrayDeque                      │
│                     │    │                                   │
│                 Stack    ├── HashSet                         │
│                          │                                   │
│                          ├── LinkedHashSet                   │
│                          │                                   │
│                          └── TreeSet (SortedSet)             │
│                                                              │
│                         Map<K,V>                             │
│                    /       │       \                         │
│              HashMap  LinkedHashMap  TreeMap                │
│                 │                  (SortedMap)               │
│            Hashtable                                         │
│                 │                                            │
│            Properties                                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 List Implementations

```java
// ArrayList - Dynamic array, fast random access
ArrayList<String> arrayList = new ArrayList<>();
arrayList.add("A");        // O(1) amortized
arrayList.get(0);          // O(1) - direct index access
arrayList.remove(0);       // O(n) - shifts elements
arrayList.contains("A");   // O(n) - linear search

// LinkedList - Doubly linked list, fast insert/delete
LinkedList<String> linkedList = new LinkedList<>();
linkedList.add("A");       // O(1) at end
linkedList.addFirst("B");  // O(1)
linkedList.get(5);         // O(n) - must traverse
linkedList.remove(0);      // O(1)
```

**ArrayList vs LinkedList:**

| Operation | ArrayList | LinkedList |
|-----------|-----------|------------|
| get(index) | O(1) | O(n) |
| add(end) | O(1)* | O(1) |
| add(index) | O(n) | O(n)** |
| remove(index) | O(n) | O(n)** |
| Memory | Less | More (node overhead) |

*Amortized, **O(1) if you have iterator at position

### 7.3 Set Implementations

```java
// HashSet - No order, O(1) operations
Set<String> hashSet = new HashSet<>();
hashSet.add("banana");
hashSet.add("apple");
hashSet.add("cherry");
// Iteration order: unpredictable!

// LinkedHashSet - Insertion order maintained
Set<String> linkedHashSet = new LinkedHashSet<>();
linkedHashSet.add("banana");
linkedHashSet.add("apple");
linkedHashSet.add("cherry");
// Iteration order: banana, apple, cherry

// TreeSet - Sorted order
Set<String> treeSet = new TreeSet<>();
treeSet.add("banana");
treeSet.add("apple");
treeSet.add("cherry");
// Iteration order: apple, banana, cherry (natural order)
```

**When to use which Set:**

| Use Case | Implementation |
|----------|---------------|
| Fast lookup, no order needed | HashSet |
| Maintain insertion order | LinkedHashSet |
| Sorted elements | TreeSet |
| Thread-safe | ConcurrentSkipListSet |

### 7.4 Map Implementations

```java
// HashMap - O(1) operations, no order
Map<String, Integer> hashMap = new HashMap<>();
hashMap.put("Alice", 25);
hashMap.put("Bob", 30);
hashMap.get("Alice");      // O(1) average
hashMap.containsKey("Bob"); // O(1) average

// Internal structure (Java 8+)
// Array of Node/TreeNode
// Bucket uses LinkedList for <=8 items, TreeMap for >8 items

// LinkedHashMap - Maintains insertion order or access order
Map<String, Integer> linkedHashMap = new LinkedHashMap<>();
linkedHashMap.put("Z", 1);
linkedHashMap.put("A", 2);
linkedHashMap.put("M", 3);
// Iteration: Z->A->M (insertion order)

// Access-order LinkedHashMap (for LRU cache)
Map<String, Integer> accessOrderMap = new LinkedHashMap<>(16, 0.75f, true);

// TreeMap - Sorted by keys
Map<String, Integer> treeMap = new TreeMap<>();
treeMap.put("Z", 1);
treeMap.put("A", 2);
treeMap.put("M", 3);
// Iteration: A->M->Z (sorted order)

// ConcurrentHashMap - Thread-safe
Map<String, Integer> concurrentMap = new ConcurrentHashMap<>();
```

### 7.5 HashMap Internals

```
┌─────────────────────────────────────────────────────────────┐
│                    HASHMAP INTERNALS                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Key: "Alice"                                               │
│   hashCode() = 63374653                                      │
│   index = hash & (n-1) = 63374653 & 15 = 13                 │
│                                                              │
│   Array (buckets):                                          │
│   ┌───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┬───┐│
│   │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │ 7 │ 8 │ 9 │10 │11 │12 │13 ││
│   └───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┴───┘│
│                                                          │   │
│                                                          ▼   │
│                                             ┌─────────────┐  │
│                                             │key: "Alice" │  │
│                                             │value: 25    │  │
│                                             │next: ────────┼──┼─► (collision chain)
│                                             └─────────────┘  │
│                                                              │
│   When bucket has >8 items: converts to TreeMap (Red-Black) │
│   When bucket drops <=6: converts back to LinkedList        │
│                                                              │
│   Load Factor: 0.75 (default)                               │
│   Rehashing: when size > capacity * loadFactor              │
│   New capacity: 2 * old capacity                            │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 7.6 equals() and hashCode() Contract

```java
public class Employee {
    private int id;
    private String name;

    // RULE: If equals() returns true, hashCode() MUST be equal
    // RULE: If hashCode() is equal, equals() MAY or may not be true

    @Override
    public boolean equals(Object obj) {
        if (this == obj) return true;
        if (obj == null || getClass() != obj.getClass()) return false;
        Employee other = (Employee) obj;
        return id == other.id && Objects.equals(name, other.name);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id, name);
    }
}

// WRONG - equals without hashCode
class BadEmployee {
    private int id;

    @Override
    public boolean equals(Object obj) {
        if (obj instanceof BadEmployee) {
            return this.id == ((BadEmployee) obj).id;
        }
        return false;
    }
    // No hashCode override!
}

BadEmployee e1 = new BadEmployee(1);
BadEmployee e2 = new BadEmployee(1);

Set<BadEmployee> set = new HashSet<>();
set.add(e1);
set.add(e2);
System.out.println(set.size());  // 2! (should be 1)
// Different hashCodes put them in different buckets
```

### 7.7 Fail-Fast vs Fail-Safe Iterators

```java
// FAIL-FAST: ArrayList, HashMap, HashSet
List<String> list = new ArrayList<>(Arrays.asList("A", "B", "C"));
Iterator<String> iterator = list.iterator();

list.add("D");  // Structural modification

while (iterator.hasNext()) {
    System.out.println(iterator.next());  // ConcurrentModificationException!
}

// Safe removal during iteration
Iterator<String> it = list.iterator();
while (it.hasNext()) {
    if (it.next().equals("B")) {
        it.remove();  // Use iterator's remove, not list.remove()
    }
}

// FAIL-SAFE: CopyOnWriteArrayList, ConcurrentHashMap
List<String> cowList = new CopyOnWriteArrayList<>(Arrays.asList("A", "B", "C"));
Iterator<String> cowIterator = cowList.iterator();

cowList.add("D");  // Works! Iterator uses snapshot

while (cowIterator.hasNext()) {
    System.out.println(cowIterator.next());  // Prints A, B, C (not D)
}
```

### 7.8 Comparable vs Comparator

```java
// Comparable - Natural ordering (class defines its own order)
class Employee implements Comparable<Employee> {
    private int id;
    private String name;

    @Override
    public int compareTo(Employee other) {
        return Integer.compare(this.id, other.id);  // Natural: by ID
    }
}

List<Employee> employees = new ArrayList<>();
Collections.sort(employees);  // Uses natural ordering

// Comparator - External ordering (multiple orderings possible)
Comparator<Employee> byName = (e1, e2) -> e1.getName().compareTo(e2.getName());
Comparator<Employee> bySalary = Comparator.comparingDouble(Employee::getSalary);
Comparator<Employee> byNameThenId = Comparator
    .comparing(Employee::getName)
    .thenComparing(Employee::getId);

Collections.sort(employees, byName);
employees.sort(bySalary);
employees.sort(byNameThenId.reversed());
```

---

## 8. String & Immutability

### 8.1 String Pool (Interning)

```
┌─────────────────────────────────────────────────────────────┐
│                    STRING POOL (HEAP)                        │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Stack                          Heap                        │
│   ┌─────────┐                    ┌─────────────────────────┐│
│   │ s1 ─────┼────────────────────┼──► "Hello"              ││
│   │         │                    │         △                ││
│   │ s2 ─────┼────────────────────┼─────────┘ (same object) ││
│   │         │                    │                          ││
│   │ s3 ─────┼────────────────────┼──► "Hello" (new Object) ││
│   │         │                    │      (outside pool)      ││
│   │         │                    │                          ││
│   │ s4 ─────┼────────────────────┼──► "World"              ││
│   └─────────┘                    │         △                ││
│                                  │ s5 ─────┘ (after intern)││
│                                  └─────────────────────────┘│
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

```java
String s1 = "Hello";           // Goes to String Pool
String s2 = "Hello";           // Reuses from Pool
String s3 = new String("Hello"); // New object on heap (outside pool)
String s4 = "World";           // Goes to String Pool
String s5 = new String("World").intern(); // Puts in pool if not exists, returns pool reference

System.out.println(s1 == s2);       // true (same pool reference)
System.out.println(s1 == s3);       // false (different objects)
System.out.println(s1.equals(s3));  // true (same content)
System.out.println(s4 == s5);       // true (s5 interned to pool)
```

### 8.2 Why is String Immutable?

```java
// String is final class with final char[] (or byte[] in Java 9+)
public final class String {
    private final byte[] value;  // Cannot be changed after creation

    // No setter methods that modify value
    // All "modification" methods return NEW String
}

// Example showing immutability
String s = "Hello";
s.concat(" World");  // Returns NEW string, doesn't modify s
System.out.println(s);  // Still "Hello"

s = s.concat(" World");  // Reassignment, s now points to new object
System.out.println(s);  // "Hello World"
```

**Reasons for String Immutability:**

1. **String Pool**: Enables safe sharing of strings
2. **Security**: Credentials, URLs, file paths can't be modified
3. **Thread Safety**: Immutable objects are inherently thread-safe
4. **Caching hashCode**: Hash computed once and cached
5. **Class Loading**: Class names are strings, must be reliable

### 8.3 String vs StringBuilder vs StringBuffer

```java
// String - Immutable, creates new object for each modification
String s = "Hello";
for (int i = 0; i < 1000; i++) {
    s = s + i;  // Creates 1000 new String objects! O(n²)
}

// StringBuilder - Mutable, NOT thread-safe, FAST
StringBuilder sb = new StringBuilder("Hello");
for (int i = 0; i < 1000; i++) {
    sb.append(i);  // Modifies same object. O(n)
}
String result = sb.toString();

// StringBuffer - Mutable, thread-safe (synchronized), slower
StringBuffer sbf = new StringBuffer("Hello");
sbf.append(" World");  // Synchronized method
```

| Feature | String | StringBuilder | StringBuffer |
|---------|--------|---------------|--------------|
| Mutable | No | Yes | Yes |
| Thread-safe | Yes (immutable) | No | Yes |
| Performance | Slow for concatenation | Fast | Medium |
| Use case | Few modifications | Many modifications, single thread | Many modifications, multi-thread |

### 8.4 Common String Methods

```java
String s = "Hello World";

// Basic operations
s.length();                    // 11
s.charAt(0);                   // 'H'
s.substring(0, 5);            // "Hello"
s.indexOf("World");           // 6
s.lastIndexOf("o");           // 7
s.contains("World");          // true

// Comparison
s.equals("hello world");      // false (case-sensitive)
s.equalsIgnoreCase("HELLO WORLD");  // true
s.compareTo("Hello Xorld");   // negative (W < X)
s.startsWith("Hello");        // true
s.endsWith("World");          // true

// Transformation (all return NEW String)
s.toLowerCase();              // "hello world"
s.toUpperCase();              // "HELLO WORLD"
s.trim();                     // Removes leading/trailing whitespace
s.strip();                    // Java 11+, Unicode-aware trim
s.replace("World", "Java");   // "Hello Java"
s.replaceAll("\\s+", "-");    // "Hello-World" (regex)
s.split(" ");                 // ["Hello", "World"]

// Java 11+ String methods
"  ".isBlank();               // true (vs isEmpty())
"abc".repeat(3);              // "abcabcabc"
"  hello  ".strip();          // "hello"
"hello\nworld".lines();       // Stream of lines

// Java 15+ Text Blocks
String json = """
    {
        "name": "John",
        "age": 30
    }
    """;
```

### 8.5 String Interview Questions

```java
// Q: How many objects created?
String s1 = "Hello";                // 1 (pool)
String s2 = "Hello";                // 0 (reuses pool)
String s3 = new String("Hello");    // 1 (heap, but "Hello" might already be in pool)
String s4 = new String("World");    // 2 (1 pool + 1 heap)

// Q: What's the output?
String s = "Hello";
s.toUpperCase();
System.out.println(s);  // "Hello" (immutable!)

// Q: Will this print true or false?
String str1 = "Hello";
String str2 = "Hel" + "lo";  // Compile-time constant
System.out.println(str1 == str2);  // true (compiler optimizes)

String str3 = "Hel";
String str4 = str3 + "lo";  // Runtime concatenation
System.out.println(str1 == str4);  // false (new object created)

// Q: Making String mutable (reflection trick - DON'T DO THIS!)
String s = "Hello";
Field valueField = String.class.getDeclaredField("value");
valueField.setAccessible(true);
valueField.set(s, "World".getBytes());  // May work pre-Java 9
// Violates immutability contract, causes undefined behavior
```

---

## 9. Program Output Questions

### 9.1 Static and Instance Blocks

```java
class Parent {
    static { System.out.println("Parent static block"); }
    { System.out.println("Parent instance block"); }

    Parent() {
        System.out.println("Parent constructor");
    }
}

class Child extends Parent {
    static { System.out.println("Child static block"); }
    { System.out.println("Child instance block"); }

    Child() {
        System.out.println("Child constructor");
    }
}

public class Main {
    public static void main(String[] args) {
        new Child();
        System.out.println("---");
        new Child();
    }
}

// Output:
// Parent static block      (1. Static blocks run once when class loads)
// Child static block       (2. Child static block)
// Parent instance block    (3. Parent instance block before constructor)
// Parent constructor       (4. Parent constructor)
// Child instance block     (5. Child instance block)
// Child constructor        (6. Child constructor)
// ---
// Parent instance block    (Static blocks don't run again)
// Parent constructor
// Child instance block
// Child constructor
```

### 9.2 Method Overriding Puzzle

```java
class Animal {
    public void eat() {
        System.out.println("Animal eating");
    }

    public void drink() {
        System.out.println("Animal drinking");
        eat();  // Which eat() is called?
    }
}

class Dog extends Animal {
    @Override
    public void eat() {
        System.out.println("Dog eating");
    }
}

public class Main {
    public static void main(String[] args) {
        Animal a = new Dog();
        a.drink();
    }
}

// Output:
// Animal drinking
// Dog eating  <-- Runtime polymorphism! Dog's eat() is called
```

### 9.3 Exception in Static Block

```java
class Problem {
    static int value;

    static {
        value = 10 / 0;  // ArithmeticException
    }
}

public class Main {
    public static void main(String[] args) {
        try {
            System.out.println(Problem.value);
        } catch (Throwable t) {
            System.out.println(t.getClass().getSimpleName());
        }
    }
}

// Output: ExceptionInInitializerError
// (wraps the ArithmeticException)
```

### 9.4 finally with return

```java
public class Finally {
    public static int getValue() {
        int x = 1;
        try {
            return x;
        } finally {
            x = 2;
        }
    }

    public static void main(String[] args) {
        System.out.println(getValue());
    }
}

// Output: 1
// return value (1) is saved BEFORE finally runs
// finally changes x to 2, but return value already captured

// But with reference types:
public static StringBuilder getBuilder() {
    StringBuilder sb = new StringBuilder("Hello");
    try {
        return sb;
    } finally {
        sb.append(" World");  // Modifies the SAME object
    }
}
// Output: "Hello World"
```

### 9.5 String Pool Puzzle

```java
public class StringPuzzle {
    public static void main(String[] args) {
        String s1 = "Java";
        String s2 = "Java";
        String s3 = new String("Java");
        String s4 = s3.intern();

        System.out.println(s1 == s2);       // ?
        System.out.println(s1 == s3);       // ?
        System.out.println(s1 == s4);       // ?
        System.out.println(s3 == s4);       // ?
    }
}

// Output:
// true   (both reference pool)
// false  (s3 is new object on heap)
// true   (intern() returns pool reference)
// false  (s3 is heap, s4 is pool)
```

### 9.6 Method Hiding (Static Methods)

```java
class Parent {
    static void display() {
        System.out.println("Parent static");
    }

    void show() {
        System.out.println("Parent instance");
    }
}

class Child extends Parent {
    static void display() {  // HIDING, not overriding!
        System.out.println("Child static");
    }

    @Override
    void show() {  // OVERRIDING
        System.out.println("Child instance");
    }
}

public class Main {
    public static void main(String[] args) {
        Parent p = new Child();
        p.display();  // ?
        p.show();     // ?

        Child c = new Child();
        c.display();  // ?
        c.show();     // ?
    }
}

// Output:
// Parent static   (static methods resolved at compile-time based on reference type)
// Child instance  (instance methods resolved at runtime based on object type)
// Child static
// Child instance
```

### 9.7 Autoboxing Puzzle

```java
public class AutoboxingPuzzle {
    public static void main(String[] args) {
        Integer a = 100;
        Integer b = 100;
        Integer c = 200;
        Integer d = 200;

        System.out.println(a == b);  // ?
        System.out.println(c == d);  // ?
    }
}

// Output:
// true   (-128 to 127 are cached by IntegerCache)
// false  (200 is outside cache range, different objects)

// Integer cache range can be extended with -XX:AutoBoxCacheMax=<size>
```

### 9.8 Varargs Puzzle

```java
public class VarargsPuzzle {
    static void method(int... a) {
        System.out.println("int varargs");
    }

    static void method(Integer... a) {
        System.out.println("Integer varargs");
    }

    public static void main(String[] args) {
        method(1, 2);  // ?
    }
}

// Output: Compilation Error!
// Both methods are applicable and neither is more specific
// Ambiguous method call!
```

### 9.9 Covariant Return Type

```java
class Animal {
    Animal create() {
        System.out.println("Creating Animal");
        return new Animal();
    }
}

class Dog extends Animal {
    @Override
    Dog create() {  // Covariant return - more specific is OK!
        System.out.println("Creating Dog");
        return new Dog();
    }
}

public class Main {
    public static void main(String[] args) {
        Animal a = new Dog();
        Animal created = a.create();  // ?
        System.out.println(created.getClass().getSimpleName());
    }
}

// Output:
// Creating Dog
// Dog
```

### 9.10 Order of Initialization

```java
class Demo {
    static int a = initA();
    int b = initB();

    static int initA() {
        System.out.println("Static variable a initialized");
        return 1;
    }

    int initB() {
        System.out.println("Instance variable b initialized");
        return 2;
    }

    static {
        System.out.println("Static block executed");
    }

    {
        System.out.println("Instance block executed");
    }

    Demo() {
        System.out.println("Constructor executed");
    }
}

public class Main {
    public static void main(String[] args) {
        System.out.println("Creating first object");
        Demo d1 = new Demo();
        System.out.println("Creating second object");
        Demo d2 = new Demo();
    }
}

// Output:
// Static variable a initialized
// Static block executed
// Creating first object
// Instance variable b initialized
// Instance block executed
// Constructor executed
// Creating second object
// Instance variable b initialized
// Instance block executed
// Constructor executed
```

---

## 10. Design Patterns in Java

### 10.1 Singleton Pattern

```java
// 1. Eager Initialization (Thread-safe, simple)
public class EagerSingleton {
    private static final EagerSingleton INSTANCE = new EagerSingleton();

    private EagerSingleton() {}

    public static EagerSingleton getInstance() {
        return INSTANCE;
    }
}

// 2. Lazy Initialization with Double-Checked Locking (Thread-safe)
public class LazySingleton {
    private static volatile LazySingleton instance;

    private LazySingleton() {}

    public static LazySingleton getInstance() {
        if (instance == null) {                   // First check (no locking)
            synchronized (LazySingleton.class) {
                if (instance == null) {           // Second check (with lock)
                    instance = new LazySingleton();
                }
            }
        }
        return instance;
    }
}

// 3. Bill Pugh Singleton (Recommended - Lazy + Thread-safe)
public class BillPughSingleton {
    private BillPughSingleton() {}

    // Inner class loaded only when getInstance() called
    private static class SingletonHolder {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }

    public static BillPughSingleton getInstance() {
        return SingletonHolder.INSTANCE;
    }
}

// 4. Enum Singleton (Best - handles serialization & reflection)
public enum EnumSingleton {
    INSTANCE;

    public void doSomething() {
        System.out.println("Singleton action");
    }
}
// Usage: EnumSingleton.INSTANCE.doSomething();
```

**Breaking Singleton:**
```java
// 1. Reflection - can access private constructor
Constructor<Singleton> constructor = Singleton.class.getDeclaredConstructor();
constructor.setAccessible(true);
Singleton newInstance = constructor.newInstance();  // New instance!

// Prevention: Throw exception in constructor if instance exists
private Singleton() {
    if (instance != null) {
        throw new RuntimeException("Use getInstance() instead");
    }
}

// 2. Serialization - creates new instance on deserialization
// Prevention: Add readResolve()
protected Object readResolve() {
    return getInstance();
}

// Enum Singleton handles both automatically!
```

### 10.2 Factory Pattern

```java
// Product interface
interface Notification {
    void notifyUser(String message);
}

// Concrete products
class EmailNotification implements Notification {
    @Override
    public void notifyUser(String message) {
        System.out.println("Email: " + message);
    }
}

class SMSNotification implements Notification {
    @Override
    public void notifyUser(String message) {
        System.out.println("SMS: " + message);
    }
}

class PushNotification implements Notification {
    @Override
    public void notifyUser(String message) {
        System.out.println("Push: " + message);
    }
}

// Simple Factory
class NotificationFactory {
    public static Notification createNotification(String type) {
        return switch (type.toLowerCase()) {
            case "email" -> new EmailNotification();
            case "sms" -> new SMSNotification();
            case "push" -> new PushNotification();
            default -> throw new IllegalArgumentException("Unknown type: " + type);
        };
    }
}

// Usage
Notification notification = NotificationFactory.createNotification("email");
notification.notifyUser("Hello!");

// Factory Method Pattern (uses inheritance)
abstract class NotificationCreator {
    // Factory method - subclasses decide what to create
    protected abstract Notification createNotification();

    public void sendNotification(String message) {
        Notification notification = createNotification();
        notification.notifyUser(message);
    }
}

class EmailNotificationCreator extends NotificationCreator {
    @Override
    protected Notification createNotification() {
        return new EmailNotification();
    }
}
```

### 10.3 Builder Pattern

```java
// Useful for objects with many optional parameters
public class User {
    private final String firstName;    // Required
    private final String lastName;     // Required
    private final int age;             // Optional
    private final String email;        // Optional
    private final String phone;        // Optional
    private final String address;      // Optional

    private User(Builder builder) {
        this.firstName = builder.firstName;
        this.lastName = builder.lastName;
        this.age = builder.age;
        this.email = builder.email;
        this.phone = builder.phone;
        this.address = builder.address;
    }

    public static class Builder {
        // Required parameters
        private final String firstName;
        private final String lastName;

        // Optional parameters with defaults
        private int age = 0;
        private String email = "";
        private String phone = "";
        private String address = "";

        public Builder(String firstName, String lastName) {
            this.firstName = firstName;
            this.lastName = lastName;
        }

        public Builder age(int age) {
            this.age = age;
            return this;  // For method chaining
        }

        public Builder email(String email) {
            this.email = email;
            return this;
        }

        public Builder phone(String phone) {
            this.phone = phone;
            return this;
        }

        public Builder address(String address) {
            this.address = address;
            return this;
        }

        public User build() {
            return new User(this);
        }
    }

    @Override
    public String toString() {
        return "User{firstName='" + firstName + "', lastName='" + lastName +
               "', age=" + age + ", email='" + email + "'}";
    }
}

// Usage - clean and readable
User user = new User.Builder("John", "Doe")
    .age(30)
    .email("john@example.com")
    .phone("123-456-7890")
    .build();
```

### 10.4 Strategy Pattern

```java
// Strategy interface
interface PaymentStrategy {
    void pay(double amount);
}

// Concrete strategies
class CreditCardPayment implements PaymentStrategy {
    private String cardNumber;

    public CreditCardPayment(String cardNumber) {
        this.cardNumber = cardNumber;
    }

    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " using Credit Card: " +
            maskCard(cardNumber));
    }

    private String maskCard(String card) {
        return "****" + card.substring(card.length() - 4);
    }
}

class UPIPayment implements PaymentStrategy {
    private String upiId;

    public UPIPayment(String upiId) {
        this.upiId = upiId;
    }

    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " using UPI: " + upiId);
    }
}

class CryptoPayment implements PaymentStrategy {
    private String walletAddress;

    public CryptoPayment(String walletAddress) {
        this.walletAddress = walletAddress;
    }

    @Override
    public void pay(double amount) {
        System.out.println("Paid $" + amount + " using Crypto wallet");
    }
}

// Context
class ShoppingCart {
    private PaymentStrategy paymentStrategy;
    private double total;

    public void addItem(double price) {
        total += price;
    }

    public void setPaymentStrategy(PaymentStrategy strategy) {
        this.paymentStrategy = strategy;
    }

    public void checkout() {
        if (paymentStrategy == null) {
            throw new IllegalStateException("Payment method not set!");
        }
        paymentStrategy.pay(total);
    }
}

// Usage
ShoppingCart cart = new ShoppingCart();
cart.addItem(100);
cart.addItem(50);

// Pay with credit card
cart.setPaymentStrategy(new CreditCardPayment("1234567890123456"));
cart.checkout();

// Or pay with UPI
cart.setPaymentStrategy(new UPIPayment("user@upi"));
cart.checkout();
```

### 10.5 Observer Pattern

```java
import java.util.ArrayList;
import java.util.List;

// Observer interface
interface Observer {
    void update(String event);
}

// Subject interface
interface Subject {
    void subscribe(Observer observer);
    void unsubscribe(Observer observer);
    void notifyObservers(String event);
}

// Concrete Subject
class NewsAgency implements Subject {
    private List<Observer> observers = new ArrayList<>();
    private String latestNews;

    @Override
    public void subscribe(Observer observer) {
        observers.add(observer);
    }

    @Override
    public void unsubscribe(Observer observer) {
        observers.remove(observer);
    }

    @Override
    public void notifyObservers(String event) {
        for (Observer observer : observers) {
            observer.update(event);
        }
    }

    public void publishNews(String news) {
        this.latestNews = news;
        notifyObservers(news);
    }
}

// Concrete Observers
class EmailSubscriber implements Observer {
    private String email;

    public EmailSubscriber(String email) {
        this.email = email;
    }

    @Override
    public void update(String news) {
        System.out.println("Email to " + email + ": " + news);
    }
}

class MobileSubscriber implements Observer {
    private String phoneNumber;

    public MobileSubscriber(String phoneNumber) {
        this.phoneNumber = phoneNumber;
    }

    @Override
    public void update(String news) {
        System.out.println("SMS to " + phoneNumber + ": " + news);
    }
}

// Usage
NewsAgency agency = new NewsAgency();

Observer email1 = new EmailSubscriber("user1@example.com");
Observer mobile1 = new MobileSubscriber("123-456-7890");

agency.subscribe(email1);
agency.subscribe(mobile1);

agency.publishNews("Breaking: Java 21 released!");
// Output:
// Email to user1@example.com: Breaking: Java 21 released!
// SMS to 123-456-7890: Breaking: Java 21 released!

agency.unsubscribe(email1);
agency.publishNews("Update: New features announced");
// Output:
// SMS to 123-456-7890: Update: New features announced
```

---

## 11. Generics & Type Erasure

### 11.1 Generics Basics

```java
// Without Generics (pre-Java 5)
List list = new ArrayList();
list.add("Hello");
list.add(123);  // No compile error!
String s = (String) list.get(1);  // ClassCastException at runtime!

// With Generics
List<String> list = new ArrayList<>();
list.add("Hello");
list.add(123);  // Compile error! Type safety
String s = list.get(0);  // No casting needed

// Generic Class
public class Box<T> {
    private T content;

    public void set(T content) {
        this.content = content;
    }

    public T get() {
        return content;
    }
}

Box<String> stringBox = new Box<>();
stringBox.set("Hello");
String value = stringBox.get();

Box<Integer> intBox = new Box<>();
intBox.set(123);
Integer num = intBox.get();

// Generic Method
public <T> void printArray(T[] array) {
    for (T element : array) {
        System.out.println(element);
    }
}

// Bounded Type Parameters
public <T extends Number> double sum(List<T> numbers) {
    double total = 0;
    for (T num : numbers) {
        total += num.doubleValue();
    }
    return total;
}
```

### 11.2 Wildcards

```java
// Upper Bounded Wildcard (Producer - read only)
public void printNumbers(List<? extends Number> list) {
    for (Number n : list) {
        System.out.println(n);
    }
    // list.add(1);  // COMPILE ERROR! Cannot add
}
// Can pass List<Integer>, List<Double>, List<Number>

// Lower Bounded Wildcard (Consumer - write only)
public void addNumbers(List<? super Integer> list) {
    list.add(1);
    list.add(2);
    // Integer n = list.get(0);  // COMPILE ERROR! Only Object guaranteed
}
// Can pass List<Integer>, List<Number>, List<Object>

// Unbounded Wildcard
public void printList(List<?> list) {
    for (Object obj : list) {
        System.out.println(obj);
    }
    // list.add("test");  // COMPILE ERROR! Cannot add (except null)
}
```

**PECS Principle**: Producer Extends, Consumer Super

```
┌─────────────────────────────────────────────────────────────┐
│                    PECS PRINCIPLE                            │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   PRODUCER EXTENDS (read from)                               │
│   List<? extends Number>                                     │
│   ┌─────────────────────┐                                   │
│   │ Can read as Number  │ ──► Number n = list.get(0); ✓    │
│   │ Cannot add          │ ──► list.add(1); ✗               │
│   └─────────────────────┘                                   │
│                                                              │
│   CONSUMER SUPER (write to)                                  │
│   List<? super Integer>                                      │
│   ┌─────────────────────┐                                   │
│   │ Can add Integer     │ ──► list.add(1); ✓               │
│   │ Can only read Object│ ──► Object o = list.get(0);      │
│   └─────────────────────┘                                   │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 11.3 Type Erasure

```java
// At compile time
public class Box<T> {
    private T value;
    public T getValue() { return value; }
    public void setValue(T value) { this.value = value; }
}

// After type erasure (bytecode) - T becomes Object
public class Box {
    private Object value;
    public Object getValue() { return value; }
    public void setValue(Object value) { this.value = value; }
}

// With bounded type - T becomes the bound
public class NumberBox<T extends Number> {
    private T value;
}

// After erasure - T becomes Number
public class NumberBox {
    private Number value;
}
```

**Consequences of Type Erasure:**

```java
// 1. Cannot create generic arrays
T[] array = new T[10];  // COMPILE ERROR!
// Workaround:
T[] array = (T[]) new Object[10];  // Warning, but works

// 2. Cannot use instanceof with generics
if (list instanceof ArrayList<String>) { }  // COMPILE ERROR!
// Workaround:
if (list instanceof ArrayList<?>) { }  // OK

// 3. Cannot create instance of type parameter
T obj = new T();  // COMPILE ERROR!
// Workaround: use Class<T>
public <T> T create(Class<T> clazz) throws Exception {
    return clazz.getDeclaredConstructor().newInstance();
}

// 4. Generic types with same erasure clash
void process(List<String> list) { }
void process(List<Integer> list) { }  // COMPILE ERROR! Same erasure

// 5. Static fields cannot use type parameters
class Box<T> {
    static T value;  // COMPILE ERROR!
}
```

### 11.4 Generic Interview Questions

```java
// Q: What's wrong with this code?
public static void swap(List<?> list, int i, int j) {
    list.set(i, list.get(j));  // COMPILE ERROR!
}
// Cannot put anything in List<?>

// Fix with helper method:
public static void swap(List<?> list, int i, int j) {
    swapHelper(list, i, j);
}

private static <T> void swapHelper(List<T> list, int i, int j) {
    T temp = list.get(i);
    list.set(i, list.get(j));
    list.set(j, temp);
}

// Q: Will this compile?
List<Object> objects = new ArrayList<String>();  // NO!
// List<String> is NOT a subtype of List<Object>

// This works:
List<? extends Object> objects = new ArrayList<String>();  // OK

// Q: What's the difference?
List list1 = new ArrayList();      // Raw type - no type checking
List<?> list2 = new ArrayList<>(); // Wildcard - type safe, read-only
List<Object> list3 = new ArrayList<>();  // Object type - can add Objects
```

---

## 12. Serialization

### 12.1 Serialization Basics

```java
import java.io.*;

// Mark class as serializable
public class Employee implements Serializable {
    private static final long serialVersionUID = 1L;  // Version control

    private String name;
    private int age;
    private transient String password;  // Won't be serialized
    private static String company = "ABC Corp";  // Static - not serialized

    public Employee(String name, int age, String password) {
        this.name = name;
        this.age = age;
        this.password = password;
    }

    @Override
    public String toString() {
        return "Employee{name='" + name + "', age=" + age +
               ", password='" + password + "', company='" + company + "'}";
    }
}

// Serialization
Employee emp = new Employee("John", 30, "secret123");

try (ObjectOutputStream oos = new ObjectOutputStream(
        new FileOutputStream("employee.ser"))) {
    oos.writeObject(emp);
}

// Deserialization
try (ObjectInputStream ois = new ObjectInputStream(
        new FileInputStream("employee.ser"))) {
    Employee deserializedEmp = (Employee) ois.readObject();
    System.out.println(deserializedEmp);
    // Output: Employee{name='John', age=30, password='null', company='ABC Corp'}
    // password is null (transient), company has current static value
}
```

### 12.2 serialVersionUID

```java
// If not provided, JVM generates one based on class structure
// Changing class (adding fields) changes generated UID
// Causes InvalidClassException during deserialization

// Best practice: Always declare explicit serialVersionUID
private static final long serialVersionUID = 1L;

// If you change class structure but want old serialized data to work:
// 1. Keep same serialVersionUID
// 2. Handle missing fields in readObject()
```

### 12.3 Custom Serialization

```java
public class Account implements Serializable {
    private static final long serialVersionUID = 1L;

    private String accountNumber;
    private transient String encryptedPassword;  // Custom handling

    // Custom serialization
    private void writeObject(ObjectOutputStream oos) throws IOException {
        oos.defaultWriteObject();  // Serialize non-transient fields
        // Custom: encrypt and write password
        String encrypted = encrypt(encryptedPassword);
        oos.writeObject(encrypted);
    }

    // Custom deserialization
    private void readObject(ObjectInputStream ois)
            throws IOException, ClassNotFoundException {
        ois.defaultReadObject();  // Deserialize non-transient fields
        // Custom: read and decrypt password
        String encrypted = (String) ois.readObject();
        this.encryptedPassword = decrypt(encrypted);
    }

    private String encrypt(String data) {
        return "ENC:" + data;  // Simplified
    }

    private String decrypt(String data) {
        return data.substring(4);  // Simplified
    }
}
```

### 12.4 Externalizable

```java
// Full control over serialization
public class CustomData implements Externalizable {
    private int id;
    private String data;

    // REQUIRED: No-arg constructor for Externalizable
    public CustomData() {}

    public CustomData(int id, String data) {
        this.id = id;
        this.data = data;
    }

    @Override
    public void writeExternal(ObjectOutput out) throws IOException {
        out.writeInt(id);
        out.writeUTF(data);
    }

    @Override
    public void readExternal(ObjectInput in) throws IOException {
        this.id = in.readInt();
        this.data = in.readUTF();
    }
}
```

### 12.5 Serialization with Inheritance

```java
// Parent NOT serializable
class Parent {
    int parentValue;

    Parent() {
        this.parentValue = 10;
        System.out.println("Parent constructor called");
    }
}

// Child IS serializable
class Child extends Parent implements Serializable {
    int childValue;

    Child(int childValue) {
        this.childValue = childValue;
    }
}

// When deserializing Child:
// 1. Parent constructor IS called (Parent not serializable)
// 2. Parent fields get default/constructor values
// 3. Child fields are restored from stream

Child c = new Child(100);
c.parentValue = 50;

// Serialize and deserialize...

// After deserialization:
// parentValue = 10 (from constructor, not 50!)
// childValue = 100 (restored from stream)
```

---

## 13. Miscellaneous Tricky Questions

### 13.1 final, finally, finalize

```java
// final - keyword for constants, preventing inheritance/override
final int MAX = 100;                    // Constant variable
final class ImmutableClass { }          // Cannot be extended
final void criticalMethod() { }         // Cannot be overridden

// finally - block that always executes after try-catch
try {
    riskyOperation();
} catch (Exception e) {
    handleError(e);
} finally {
    cleanup();  // Always runs (except System.exit, infinite loop, etc.)
}

// finalize - deprecated method called before GC (DON'T USE!)
@Override
protected void finalize() throws Throwable {
    // Cleanup code - unreliable, deprecated since Java 9
    super.finalize();
}
// Use try-with-resources or explicit cleanup instead
```

### 13.2 == vs equals() vs hashCode()

```java
String s1 = new String("hello");
String s2 = new String("hello");
String s3 = "hello";
String s4 = "hello";

// == compares references (memory addresses)
System.out.println(s1 == s2);   // false (different objects)
System.out.println(s3 == s4);   // true (same pool reference)

// equals() compares content (for String)
System.out.println(s1.equals(s2));  // true (same content)

// hashCode() returns integer hash
System.out.println(s1.hashCode() == s2.hashCode());  // true

// Contract:
// 1. If equals() returns true, hashCode() MUST be equal
// 2. If hashCode() is equal, equals() MAY be true or false
// 3. hashCode() must be consistent during execution
```

### 13.3 Pass by Value in Java

```java
public class PassByValue {
    public static void main(String[] args) {
        // Primitives - value is copied
        int x = 10;
        modifyPrimitive(x);
        System.out.println(x);  // Still 10

        // Objects - reference is copied (not the object)
        StringBuilder sb = new StringBuilder("Hello");
        modifyObject(sb);
        System.out.println(sb);  // "Hello World"

        // Reassigning reference doesn't affect original
        StringBuilder sb2 = new StringBuilder("Test");
        reassignReference(sb2);
        System.out.println(sb2);  // Still "Test"
    }

    static void modifyPrimitive(int num) {
        num = 20;  // Only local copy changed
    }

    static void modifyObject(StringBuilder sb) {
        sb.append(" World");  // Modifies the same object
    }

    static void reassignReference(StringBuilder sb) {
        sb = new StringBuilder("New");  // Only local reference changed
    }
}
```

```
┌─────────────────────────────────────────────────────────────┐
│                 JAVA IS ALWAYS PASS BY VALUE                 │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│   Primitives: Value is copied                                │
│                                                              │
│   main()          modifyPrimitive()                          │
│   ┌─────┐         ┌─────┐                                   │
│   │ x=10│  ────►  │num=10│ ──► │num=20│                     │
│   └─────┘         └─────┘      └──────┘                     │
│   x unchanged                                                │
│                                                              │
│   Objects: Reference value is copied (NOT the object)        │
│                                                              │
│   main()          modifyObject()                             │
│   ┌─────┐         ┌─────┐                                   │
│   │ sb ─┼────┬───►│ sb ─┼───────┐                           │
│   └─────┘    │    └─────┘       │                           │
│              │                   ▼                           │
│              │         ┌─────────────────┐                  │
│              └────────►│ "Hello World"   │ ◄── Same object! │
│                        └─────────────────┘                  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 13.4 Marker Interface

```java
// Marker interface - empty interface used for type information
public interface Serializable { }  // JDK marker
public interface Cloneable { }     // JDK marker
public interface RandomAccess { }  // JDK marker

// How they work
public class ObjectOutputStream {
    public void writeObject(Object obj) {
        if (!(obj instanceof Serializable)) {
            throw new NotSerializableException();
        }
        // Serialize...
    }
}

// Custom marker interface
public interface Auditable { }

public class AuditService {
    public void audit(Object entity) {
        if (entity instanceof Auditable) {
            // Perform auditing
            logChange(entity);
        }
    }
}

// Modern alternative: Annotations
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Auditable { }

@Auditable
public class User { }
```

### 13.5 Cloning in Java

```java
// Shallow Copy - references point to same objects
class Address {
    String city;
    Address(String city) { this.city = city; }
}

class Person implements Cloneable {
    String name;
    Address address;

    Person(String name, Address address) {
        this.name = name;
        this.address = address;
    }

    // Shallow clone
    @Override
    protected Object clone() throws CloneNotSupportedException {
        return super.clone();
    }
}

Person p1 = new Person("John", new Address("NYC"));
Person p2 = (Person) p1.clone();

p2.address.city = "LA";
System.out.println(p1.address.city);  // "LA"! Same Address object!

// Deep Copy - all nested objects also cloned
class Person implements Cloneable {
    String name;
    Address address;

    @Override
    protected Object clone() throws CloneNotSupportedException {
        Person cloned = (Person) super.clone();
        cloned.address = new Address(this.address.city);  // Clone nested
        return cloned;
    }
}

// Better: Copy constructor
class Person {
    String name;
    Address address;

    // Copy constructor
    public Person(Person other) {
        this.name = other.name;
        this.address = new Address(other.address.city);
    }
}
```

### 13.6 Immutable Class

```java
// Requirements for immutability:
// 1. Class is final
// 2. All fields are private and final
// 3. No setters
// 4. Deep copy in constructor and getters for mutable fields

public final class ImmutablePerson {
    private final String name;
    private final Date birthDate;  // Date is mutable!
    private final List<String> hobbies;

    public ImmutablePerson(String name, Date birthDate, List<String> hobbies) {
        this.name = name;
        this.birthDate = new Date(birthDate.getTime());  // Defensive copy
        this.hobbies = new ArrayList<>(hobbies);  // Defensive copy
    }

    public String getName() {
        return name;  // String is immutable, safe to return
    }

    public Date getBirthDate() {
        return new Date(birthDate.getTime());  // Return copy!
    }

    public List<String> getHobbies() {
        return Collections.unmodifiableList(hobbies);  // Unmodifiable view
        // OR: return new ArrayList<>(hobbies);  // Return copy
    }
}

// Attempting to modify:
ImmutablePerson person = new ImmutablePerson("John", new Date(),
    Arrays.asList("Reading", "Gaming"));

person.getBirthDate().setTime(0);  // Doesn't affect internal state
person.getHobbies().add("Cooking");  // UnsupportedOperationException!
```

### 13.7 Java Class Loading

```java
// Class loading order:
// 1. Bootstrap ClassLoader - loads core Java classes (rt.jar)
// 2. Extension ClassLoader - loads from jre/lib/ext
// 3. Application ClassLoader - loads from classpath

// Classes are loaded lazily (when first referenced)
public class LoadingDemo {
    public static void main(String[] args) {
        System.out.println("Main started");
        System.out.println(MyClass.STATIC_FIELD);  // MyClass loaded here
    }
}

class MyClass {
    static String STATIC_FIELD = "Hello";

    static {
        System.out.println("MyClass loaded");
    }
}

// Output:
// Main started
// MyClass loaded
// Hello

// Forcing class loading
Class.forName("com.example.MyClass");  // Loads and initializes
ClassLoader.loadClass("com.example.MyClass");  // Loads but doesn't initialize
```

### 13.8 Access Modifiers

```
┌─────────────────────────────────────────────────────────────┐
│                    ACCESS MODIFIERS                          │
├──────────────┬────────┬─────────┬───────────┬───────────────┤
│              │ Class  │ Package │ Subclass  │ World         │
├──────────────┼────────┼─────────┼───────────┼───────────────┤
│ public       │   ✓    │    ✓    │     ✓     │      ✓        │
│ protected    │   ✓    │    ✓    │     ✓     │      ✗        │
│ default      │   ✓    │    ✓    │     ✗     │      ✗        │
│ private      │   ✓    │    ✗    │     ✗     │      ✗        │
└──────────────┴────────┴─────────┴───────────┴───────────────┘
```

```java
// default (package-private) - no modifier
class PackageClass {
    int packageField;
    void packageMethod() { }
}

// protected - tricky behavior!
class Parent {
    protected void show() { }
}

// Same package - can call via any reference
class SamePackageChild extends Parent {
    void test() {
        show();                    // OK
        new Parent().show();       // OK - same package
    }
}

// Different package - can only call via inheritance
class DiffPackageChild extends Parent {
    void test() {
        show();                    // OK - inherited
        new Parent().show();       // COMPILE ERROR! Different package
        new DiffPackageChild().show();  // OK - subclass reference
    }
}
```

### 13.9 Inner Classes

```java
public class Outer {
    private int outerField = 10;

    // 1. Non-static (Inner) Class
    class Inner {
        void display() {
            System.out.println(outerField);  // Can access outer's private
        }
    }

    // 2. Static Nested Class
    static class StaticNested {
        void display() {
            // Cannot access outerField (no Outer instance)
            System.out.println("Static nested class");
        }
    }

    void method() {
        final int localVar = 20;  // Effectively final

        // 3. Local Class
        class Local {
            void display() {
                System.out.println(outerField + localVar);
            }
        }
        new Local().display();

        // 4. Anonymous Class
        Runnable runnable = new Runnable() {
            @Override
            public void run() {
                System.out.println(outerField + localVar);
            }
        };
    }
}

// Creating instances:
Outer outer = new Outer();
Outer.Inner inner = outer.new Inner();  // Needs Outer instance
Outer.StaticNested nested = new Outer.StaticNested();  // No Outer needed
```

### 13.10 Java Memory Model Questions

```java
// Q: What's wrong here?
class Counter {
    private int count = 0;

    public void increment() {
        count++;  // NOT atomic! Read-Modify-Write
    }
}
// Race condition in multi-threaded environment

// Fix 1: synchronized
public synchronized void increment() {
    count++;
}

// Fix 2: AtomicInteger
private AtomicInteger count = new AtomicInteger(0);
public void increment() {
    count.incrementAndGet();
}

// Q: What does volatile do?
class Flag {
    private volatile boolean running = true;

    public void stop() {
        running = false;  // Immediately visible to all threads
    }

    public void run() {
        while (running) {  // Always reads from main memory
            // Do work
        }
    }
}
// volatile provides: visibility guarantee, prevents instruction reordering
// volatile does NOT provide: atomicity for compound operations
```

---

## Quick Reference Card

### Common Interview Traps

| Topic | Trap | Correct Answer |
|-------|------|----------------|
| String | `"a" == "a"` | true (pool) |
| String | `new String("a") == "a"` | false (different objects) |
| Integer | `Integer.valueOf(127) == Integer.valueOf(127)` | true (cached) |
| Integer | `Integer.valueOf(128) == Integer.valueOf(128)` | false (not cached) |
| finally | `return` in both try and finally | finally's return wins |
| Static methods | Can they be overridden? | No, only hidden |
| Private methods | Can they be overridden? | No, not inherited |
| Constructor | Can it be final/static/abstract? | No, none of these |
| Interface | Can have constructor? | No |
| Abstract class | Must have abstract method? | No |

### Key Differences Summary

| Concept A | Concept B | Key Difference |
|-----------|-----------|----------------|
| `==` | `equals()` | Reference vs Content |
| `ArrayList` | `LinkedList` | Array vs Doubly-linked |
| `HashMap` | `TreeMap` | Unordered O(1) vs Sorted O(log n) |
| `Comparable` | `Comparator` | Natural vs Custom ordering |
| `throw` | `throws` | Throw exception vs Declare exception |
| `final` | `finally` | Constant vs Cleanup block |
| Abstract class | Interface | Single vs Multiple inheritance |
| Overloading | Overriding | Compile-time vs Runtime polymorphism |
| Stack | Heap | Local vars/refs vs Objects |
| Checked | Unchecked | Must handle vs Optional handling |

---

**Good luck with your interviews!**
