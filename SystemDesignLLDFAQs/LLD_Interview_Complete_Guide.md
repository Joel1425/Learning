# Low-Level Design (LLD) Interview Complete Guide for SDE-2

> **Target**: Backend SDE-2 interviews | **Language**: Java | **Duration**: 90 minutes

---

## Table of Contents

1. [The LLD Interview Framework](#1-the-lld-interview-framework)
2. [How to Approach Any LLD Problem](#2-how-to-approach-any-lld-problem)
3. [Entity Identification & Class Design](#3-entity-identification--class-design)
4. [SOLID Principles - Practical Application](#4-solid-principles---practical-application)
5. [Essential Design Patterns](#5-essential-design-patterns)
6. [Concurrency & Thread Safety](#6-concurrency--thread-safety)
7. [Worked Example 1: Parking Lot System](#7-worked-example-1-parking-lot-system)
8. [Worked Example 2: Splitwise (Expense Sharing)](#8-worked-example-2-splitwise-expense-sharing)
9. [Worked Example 3: Snake and Ladder](#9-worked-example-3-snake-and-ladder)
10. [Common Follow-up Questions & Answers](#10-common-follow-up-questions--answers)
11. [Quick Reference Cheatsheet](#11-quick-reference-cheatsheet)

---

## 1. The LLD Interview Framework

### What Interviewers Evaluate

```
┌─────────────────────────────────────────────────────────────────┐
│                    LLD Interview Evaluation                      │
├─────────────────────────────────────────────────────────────────┤
│  1. Requirement Gathering (5-10 min)                             │
│     → Do you ask the RIGHT questions?                            │
│     → Can you identify core vs nice-to-have features?            │
│                                                                   │
│  2. Entity Identification (5-10 min)                             │
│     → Can you identify nouns → classes?                          │
│     → Can you identify verbs → methods?                          │
│                                                                   │
│  3. Class Design & Relationships (15-20 min)                     │
│     → Are your classes cohesive?                                 │
│     → Are relationships (HAS-A, IS-A) correct?                   │
│                                                                   │
│  4. Code Implementation (40-50 min)                              │
│     → Is code clean and readable?                                │
│     → Does it follow SOLID principles?                           │
│     → Is it extensible for future requirements?                  │
│                                                                   │
│  5. Handling Edge Cases & Follow-ups (10-15 min)                 │
│     → Concurrency handling                                        │
│     → Error handling                                              │
│     → Extensibility                                               │
└─────────────────────────────────────────────────────────────────┘
```

### Time Management (90 minutes)

| Phase | Time | What to Do |
|-------|------|------------|
| Clarify Requirements | 5-10 min | Ask questions, define scope |
| Identify Entities | 5-10 min | List classes, attributes, methods |
| High-Level Design | 5 min | Quick relationships overview |
| Core Implementation | 40-50 min | Write actual code |
| Handle Follow-ups | 15-20 min | Edge cases, concurrency, extensions |

---

## 2. How to Approach Any LLD Problem

### Step-by-Step Framework

```
┌──────────────────────────────────────────────────────────────────┐
│                    THE CREDI FRAMEWORK                            │
├──────────────────────────────────────────────────────────────────┤
│                                                                    │
│  C - Clarify Requirements                                          │
│      "What are the core features we need to support?"              │
│      "What are the constraints?"                                   │
│      "Who are the actors in the system?"                           │
│                                                                    │
│  R - Recognize Entities                                            │
│      Nouns → Classes                                               │
│      Verbs → Methods                                               │
│      Adjectives → Attributes/Enums                                 │
│                                                                    │
│  E - Establish Relationships                                       │
│      IS-A (Inheritance)                                            │
│      HAS-A (Composition/Aggregation)                               │
│      USES (Dependency)                                             │
│                                                                    │
│  D - Design Interfaces & Abstract Classes                          │
│      What behaviors are common?                                    │
│      What needs to be extended?                                    │
│                                                                    │
│  I - Implement Core Logic                                          │
│      Start with the main flow                                      │
│      Handle edge cases progressively                               │
│                                                                    │
└──────────────────────────────────────────────────────────────────┘
```

### Requirement Gathering Questions (Memorize These)

**Always ask these types of questions:**

```java
// ACTORS
"Who are the users of this system?"
"Are there different types of users with different permissions?"

// SCALE (for context, not HLD)
"How many concurrent users should we expect?"
"Is this a single-machine or distributed system?"

// CORE FEATURES
"What are the must-have features for MVP?"
"What operations are most frequent?"

// CONSTRAINTS
"Are there any time/space constraints?"
"Should operations be thread-safe?"

// EDGE CASES
"What happens if [X] fails?"
"How do we handle [edge case]?"
```

### The "Tell, Don't Guess" Rule

**WRONG Approach:**
```
Interviewer: Design a parking lot
Candidate: *Starts coding immediately*
```

**RIGHT Approach:**
```
Interviewer: Design a parking lot

Candidate: "Before I start, let me clarify a few things:
1. What types of vehicles do we need to support?
2. Do we need multiple floors or single floor?
3. Is pricing involved? Hourly/flat rate?
4. Do we need different parking spot sizes?
5. Should the system be thread-safe for concurrent entry/exit?"

Interviewer: "Good questions. Let's say..."
```

---

## 3. Entity Identification & Class Design

### The Noun-Verb Technique

```
Problem Statement Analysis:
───────────────────────────────────────��─────────────────
"Design a PARKING LOT where VEHICLES can PARK and UNPARK.
The LOT has multiple FLOORS with different types of SPOTS.
Each VEHICLE is given a TICKET when it ENTERS. PAYMENT is
calculated based on HOURS parked."
─────────────────────────────────────────────────────────

Nouns (→ Classes/Entities):
• Parking Lot
• Vehicle
• Floor
• Spot
• Ticket
• Payment

Verbs (→ Methods):
• park()
• unpark()
• enter()
• calculatePayment()

Adjectives (→ Enums/Attributes):
• Types of spots → SpotType enum
• Types of vehicles → VehicleType enum
```

### Standard Class Template

```java
/**
 * Entity: [EntityName]
 * Responsibility: [Single responsibility description]
 */
public class EntityName {

    // ============ ATTRIBUTES ============
    private final String id;              // Immutable identifier
    private EntityType type;              // Enum for types
    private Status status;                // Current state
    private LocalDateTime createdAt;      // Audit field

    // ============ RELATIONSHIPS ============
    private OtherEntity relatedEntity;    // HAS-A
    private List<ChildEntity> children;   // HAS-MANY

    // ============ CONSTRUCTOR ============
    public EntityName(String id, EntityType type) {
        this.id = id;
        this.type = type;
        this.status = Status.ACTIVE;
        this.createdAt = LocalDateTime.now();
        this.children = new ArrayList<>();
    }

    // ============ CORE BEHAVIORS ============
    public void performAction() {
        // Core business logic
    }

    // ============ GETTERS (No Setters for Immutables) ============
    public String getId() { return id; }
    public EntityType getType() { return type; }
}
```

### Enum Design Pattern

```java
// Always use enums for fixed categories
public enum VehicleType {
    MOTORCYCLE(1),
    CAR(2),
    BUS(4);

    private final int spotsRequired;

    VehicleType(int spotsRequired) {
        this.spotsRequired = spotsRequired;
    }

    public int getSpotsRequired() {
        return spotsRequired;
    }
}

public enum SpotStatus {
    AVAILABLE,
    OCCUPIED,
    RESERVED,
    OUT_OF_SERVICE
}
```

---

## 4. SOLID Principles - Practical Application

### Quick Reference

```
┌────────────────────────────────────────────────────────────────────┐
│                      SOLID PRINCIPLES                               │
├────────┬───────────────────────────────────────────────────────────┤
│   S    │ Single Responsibility: One class = One reason to change   │
├────────┼───────────────────────────────────────────────────────────┤
│   O    │ Open/Closed: Open for extension, closed for modification  │
├────────┼───────────────────────────────────────────────────────────┤
│   L    │ Liskov Substitution: Subtypes must be substitutable       │
├────────┼───────────────────────────────────────────────────────────┤
│   I    │ Interface Segregation: Many specific > one general        │
├────────┼───────────────────────────────────────────────────────────┤
│   D    │ Dependency Inversion: Depend on abstractions              │
└────────┴───────────────────────────────────────────────────────────┘
```

### S - Single Responsibility Principle

```java
// ❌ WRONG: Class does too many things
public class User {
    public void saveToDatabase() { ... }
    public void sendEmail() { ... }
    public void generateReport() { ... }
}

// ✅ RIGHT: Each class has one responsibility
public class User {
    private String id;
    private String name;
    private String email;
    // Only user data and behavior
}

public class UserRepository {
    public void save(User user) { ... }
    public User findById(String id) { ... }
}

public class EmailService {
    public void sendEmail(User user, String message) { ... }
}

public class ReportGenerator {
    public Report generateReport(User user) { ... }
}
```

### O - Open/Closed Principle

```java
// ❌ WRONG: Must modify class to add new payment type
public class PaymentProcessor {
    public void process(Payment payment) {
        if (payment.type == "CREDIT_CARD") {
            // process credit card
        } else if (payment.type == "UPI") {
            // process UPI
        } else if (payment.type == "WALLET") {
            // process wallet - HAD TO MODIFY!
        }
    }
}

// ✅ RIGHT: Open for extension, closed for modification
public interface PaymentStrategy {
    void process(Payment payment);
}

public class CreditCardPayment implements PaymentStrategy {
    @Override
    public void process(Payment payment) {
        // Credit card logic
    }
}

public class UPIPayment implements PaymentStrategy {
    @Override
    public void process(Payment payment) {
        // UPI logic
    }
}

// Adding new payment - just create new class, no modification!
public class WalletPayment implements PaymentStrategy {
    @Override
    public void process(Payment payment) {
        // Wallet logic
    }
}

public class PaymentProcessor {
    public void process(Payment payment, PaymentStrategy strategy) {
        strategy.process(payment);
    }
}
```

### L - Liskov Substitution Principle

```java
// ❌ WRONG: Square violates LSP when used as Rectangle
public class Rectangle {
    protected int width;
    protected int height;

    public void setWidth(int w) { this.width = w; }
    public void setHeight(int h) { this.height = h; }
    public int getArea() { return width * height; }
}

public class Square extends Rectangle {
    @Override
    public void setWidth(int w) {
        this.width = w;
        this.height = w; // Violates expected behavior!
    }
}

// ✅ RIGHT: Use composition or separate hierarchy
public interface Shape {
    int getArea();
}

public class Rectangle implements Shape {
    private final int width;
    private final int height;

    public Rectangle(int width, int height) {
        this.width = width;
        this.height = height;
    }

    @Override
    public int getArea() { return width * height; }
}

public class Square implements Shape {
    private final int side;

    public Square(int side) {
        this.side = side;
    }

    @Override
    public int getArea() { return side * side; }
}
```

### I - Interface Segregation Principle

```java
// ❌ WRONG: Fat interface forces unnecessary implementations
public interface Worker {
    void work();
    void eat();
    void sleep();
}

public class Robot implements Worker {
    public void work() { /* OK */ }
    public void eat() { /* Robots don't eat! */ }
    public void sleep() { /* Robots don't sleep! */ }
}

// ✅ RIGHT: Segregated interfaces
public interface Workable {
    void work();
}

public interface Feedable {
    void eat();
}

public interface Restable {
    void sleep();
}

public class Human implements Workable, Feedable, Restable {
    public void work() { ... }
    public void eat() { ... }
    public void sleep() { ... }
}

public class Robot implements Workable {
    public void work() { ... }
    // Only implements what it needs
}
```

### D - Dependency Inversion Principle

```java
// ❌ WRONG: High-level module depends on low-level module
public class NotificationService {
    private EmailSender emailSender = new EmailSender(); // Direct dependency!

    public void notify(User user, String message) {
        emailSender.send(user.getEmail(), message);
    }
}

// ✅ RIGHT: Both depend on abstraction
public interface MessageSender {
    void send(String destination, String message);
}

public class EmailSender implements MessageSender {
    @Override
    public void send(String email, String message) {
        // Send email
    }
}

public class SMSSender implements MessageSender {
    @Override
    public void send(String phone, String message) {
        // Send SMS
    }
}

public class NotificationService {
    private final MessageSender messageSender; // Depends on abstraction

    // Dependency injected
    public NotificationService(MessageSender messageSender) {
        this.messageSender = messageSender;
    }

    public void notify(String destination, String message) {
        messageSender.send(destination, message);
    }
}
```

---

## 5. Essential Design Patterns

### Pattern Selection Guide

```
┌─────────────────────────────────────────────────────────────────────┐
│                    WHEN TO USE WHICH PATTERN                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  Use STRATEGY when:                                                   │
│  → Multiple algorithms/behaviors can be swapped                       │
│  → e.g., Payment methods, Pricing strategies, Sorting algorithms     │
│                                                                       │
│  Use FACTORY when:                                                    │
│  → Object creation logic is complex                                   │
│  → You need to create family of related objects                       │
│  → e.g., Vehicle creation, Document generation                        │
│                                                                       │
│  Use SINGLETON when:                                                  │
│  → Only ONE instance should exist                                     │
│  → e.g., Configuration, Connection pool, Cache                        │
│                                                                       │
│  Use OBSERVER when:                                                   │
│  → One-to-many notification needed                                    │
│  → e.g., Event systems, Stock price updates, Notifications           │
│                                                                       │
│  Use STATE when:                                                      │
│  → Object behavior changes based on internal state                    │
│  → e.g., Order status, Vending machine, ATM                           │
│                                                                       │
│  Use DECORATOR when:                                                  │
│  → Add responsibilities dynamically                                   │
│  → e.g., Pizza toppings, Coffee additions, I/O streams               │
│                                                                       │
│  Use CHAIN OF RESPONSIBILITY when:                                    │
│  → Multiple handlers can process a request                            │
│  → e.g., Logger levels, Request filters, Approval workflows          │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### 1. Strategy Pattern

**Use Case**: Splitwise (different split strategies), Payment processing

```java
// Strategy Interface
public interface SplitStrategy {
    Map<User, Double> split(double amount, List<User> users, Map<User, Object> params);
}

// Concrete Strategies
public class EqualSplitStrategy implements SplitStrategy {
    @Override
    public Map<User, Double> split(double amount, List<User> users, Map<User, Object> params) {
        Map<User, Double> result = new HashMap<>();
        double perHead = amount / users.size();
        users.forEach(user -> result.put(user, perHead));
        return result;
    }
}

public class PercentageSplitStrategy implements SplitStrategy {
    @Override
    public Map<User, Double> split(double amount, List<User> users, Map<User, Object> params) {
        Map<User, Double> result = new HashMap<>();
        for (User user : users) {
            Double percentage = (Double) params.get(user);
            result.put(user, amount * percentage / 100);
        }
        return result;
    }
}

public class ExactSplitStrategy implements SplitStrategy {
    @Override
    public Map<User, Double> split(double amount, List<User> users, Map<User, Object> params) {
        Map<User, Double> result = new HashMap<>();
        for (User user : users) {
            result.put(user, (Double) params.get(user));
        }
        return result;
    }
}

// Context that uses strategy
public class Expense {
    private final String id;
    private final double amount;
    private final User paidBy;
    private final SplitStrategy splitStrategy;

    public Expense(String id, double amount, User paidBy, SplitStrategy strategy) {
        this.id = id;
        this.amount = amount;
        this.paidBy = paidBy;
        this.splitStrategy = strategy;
    }

    public Map<User, Double> calculateSplit(List<User> users, Map<User, Object> params) {
        return splitStrategy.split(amount, users, params);
    }
}
```

### 2. Factory Pattern

**Use Case**: Vehicle creation in Parking Lot, Document generation

```java
// Simple Factory
public class VehicleFactory {

    public static Vehicle createVehicle(VehicleType type, String licensePlate) {
        return switch (type) {
            case MOTORCYCLE -> new Motorcycle(licensePlate);
            case CAR -> new Car(licensePlate);
            case BUS -> new Bus(licensePlate);
            default -> throw new IllegalArgumentException("Unknown vehicle type: " + type);
        };
    }
}

// Abstract Factory (for families of objects)
public interface ParkingSpotFactory {
    ParkingSpot createSmallSpot();
    ParkingSpot createMediumSpot();
    ParkingSpot createLargeSpot();
}

public class StandardParkingSpotFactory implements ParkingSpotFactory {
    @Override
    public ParkingSpot createSmallSpot() {
        return new CompactSpot();
    }

    @Override
    public ParkingSpot createMediumSpot() {
        return new RegularSpot();
    }

    @Override
    public ParkingSpot createLargeSpot() {
        return new LargeSpot();
    }
}

public class HandicappedParkingSpotFactory implements ParkingSpotFactory {
    @Override
    public ParkingSpot createSmallSpot() {
        return new HandicappedCompactSpot();
    }
    // ... etc
}
```

### 3. Singleton Pattern (Thread-Safe)

**Use Case**: Configuration, Database connection pool, Cache

```java
// ✅ BEST: Bill Pugh Singleton (Lazy + Thread-safe)
public class ConfigurationManager {

    private Map<String, String> config;

    private ConfigurationManager() {
        config = new HashMap<>();
        loadConfiguration();
    }

    // Static inner class - loaded only when getInstance() is called
    private static class Holder {
        private static final ConfigurationManager INSTANCE = new ConfigurationManager();
    }

    public static ConfigurationManager getInstance() {
        return Holder.INSTANCE;
    }

    public String get(String key) {
        return config.get(key);
    }

    private void loadConfiguration() {
        // Load from file/env
    }
}

// Alternative: Enum Singleton (Simplest, handles serialization)
public enum CacheManager {
    INSTANCE;

    private final Map<String, Object> cache = new ConcurrentHashMap<>();

    public void put(String key, Object value) {
        cache.put(key, value);
    }

    public Object get(String key) {
        return cache.get(key);
    }
}
```

### 4. Observer Pattern

**Use Case**: Notification systems, Stock price updates, Event handling

```java
// Observer Interface
public interface Observer {
    void update(String event, Object data);
}

// Subject Interface
public interface Subject {
    void addObserver(Observer observer);
    void removeObserver(Observer observer);
    void notifyObservers(String event, Object data);
}

// Concrete Subject
public class StockMarket implements Subject {
    private final List<Observer> observers = new ArrayList<>();
    private final Map<String, Double> stockPrices = new HashMap<>();

    @Override
    public void addObserver(Observer observer) {
        observers.add(observer);
    }

    @Override
    public void removeObserver(Observer observer) {
        observers.remove(observer);
    }

    @Override
    public void notifyObservers(String event, Object data) {
        for (Observer observer : observers) {
            observer.update(event, data);
        }
    }

    public void updatePrice(String symbol, double price) {
        stockPrices.put(symbol, price);
        notifyObservers("PRICE_UPDATE", Map.of("symbol", symbol, "price", price));
    }
}

// Concrete Observers
public class EmailNotifier implements Observer {
    @Override
    public void update(String event, Object data) {
        if ("PRICE_UPDATE".equals(event)) {
            Map<String, Object> priceData = (Map<String, Object>) data;
            System.out.println("Email: Stock " + priceData.get("symbol") +
                             " is now " + priceData.get("price"));
        }
    }
}

public class SMSNotifier implements Observer {
    @Override
    public void update(String event, Object data) {
        // Send SMS notification
    }
}
```

### 5. State Pattern

**Use Case**: Vending Machine, Order Status, ATM

```java
// State Interface
public interface VendingMachineState {
    void insertMoney(VendingMachine machine, double amount);
    void selectProduct(VendingMachine machine, String productId);
    void dispense(VendingMachine machine);
    void cancel(VendingMachine machine);
}

// Concrete States
public class IdleState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine machine, double amount) {
        machine.setBalance(amount);
        machine.setState(new HasMoneyState());
        System.out.println("Money inserted: " + amount);
    }

    @Override
    public void selectProduct(VendingMachine machine, String productId) {
        System.out.println("Please insert money first");
    }

    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Please insert money and select product");
    }

    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Nothing to cancel");
    }
}

public class HasMoneyState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine machine, double amount) {
        machine.setBalance(machine.getBalance() + amount);
    }

    @Override
    public void selectProduct(VendingMachine machine, String productId) {
        Product product = machine.getProduct(productId);
        if (product != null && machine.getBalance() >= product.getPrice()) {
            machine.setSelectedProduct(product);
            machine.setState(new DispensingState());
            machine.dispense();
        } else {
            System.out.println("Insufficient balance or invalid product");
        }
    }

    @Override
    public void dispense(VendingMachine machine) {
        System.out.println("Please select a product first");
    }

    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Returning money: " + machine.getBalance());
        machine.setBalance(0);
        machine.setState(new IdleState());
    }
}

public class DispensingState implements VendingMachineState {
    @Override
    public void insertMoney(VendingMachine machine, double amount) {
        System.out.println("Please wait, dispensing in progress");
    }

    @Override
    public void selectProduct(VendingMachine machine, String productId) {
        System.out.println("Please wait, dispensing in progress");
    }

    @Override
    public void dispense(VendingMachine machine) {
        Product product = machine.getSelectedProduct();
        System.out.println("Dispensing: " + product.getName());
        double change = machine.getBalance() - product.getPrice();
        if (change > 0) {
            System.out.println("Returning change: " + change);
        }
        machine.setBalance(0);
        machine.setSelectedProduct(null);
        machine.setState(new IdleState());
    }

    @Override
    public void cancel(VendingMachine machine) {
        System.out.println("Cannot cancel while dispensing");
    }
}

// Context
public class VendingMachine {
    private VendingMachineState state;
    private double balance;
    private Product selectedProduct;
    private Map<String, Product> products;

    public VendingMachine() {
        this.state = new IdleState();
        this.products = new HashMap<>();
    }

    public void insertMoney(double amount) {
        state.insertMoney(this, amount);
    }

    public void selectProduct(String productId) {
        state.selectProduct(this, productId);
    }

    public void dispense() {
        state.dispense(this);
    }

    public void cancel() {
        state.cancel(this);
    }

    // Getters and setters
    public void setState(VendingMachineState state) { this.state = state; }
    public double getBalance() { return balance; }
    public void setBalance(double balance) { this.balance = balance; }
    public Product getSelectedProduct() { return selectedProduct; }
    public void setSelectedProduct(Product product) { this.selectedProduct = product; }
    public Product getProduct(String id) { return products.get(id); }
}
```

### 6. Chain of Responsibility

**Use Case**: Logger, Request handling, Approval workflows

```java
// Handler Interface
public abstract class Logger {
    protected LogLevel level;
    protected Logger nextLogger;

    public void setNext(Logger nextLogger) {
        this.nextLogger = nextLogger;
    }

    public void log(LogLevel level, String message) {
        if (this.level.ordinal() <= level.ordinal()) {
            write(message);
        }
        if (nextLogger != null) {
            nextLogger.log(level, message);
        }
    }

    protected abstract void write(String message);
}

public enum LogLevel {
    DEBUG, INFO, WARN, ERROR
}

// Concrete Handlers
public class ConsoleLogger extends Logger {
    public ConsoleLogger(LogLevel level) {
        this.level = level;
    }

    @Override
    protected void write(String message) {
        System.out.println("Console: " + message);
    }
}

public class FileLogger extends Logger {
    public FileLogger(LogLevel level) {
        this.level = level;
    }

    @Override
    protected void write(String message) {
        // Write to file
        System.out.println("File: " + message);
    }
}

public class ErrorLogger extends Logger {
    public ErrorLogger(LogLevel level) {
        this.level = level;
    }

    @Override
    protected void write(String message) {
        System.err.println("ERROR: " + message);
        // Could also send alert
    }
}

// Usage
public class LoggerChain {
    public static Logger getLogger() {
        Logger errorLogger = new ErrorLogger(LogLevel.ERROR);
        Logger fileLogger = new FileLogger(LogLevel.INFO);
        Logger consoleLogger = new ConsoleLogger(LogLevel.DEBUG);

        consoleLogger.setNext(fileLogger);
        fileLogger.setNext(errorLogger);

        return consoleLogger;
    }
}
```

---

## 6. Concurrency & Thread Safety

### Common Interview Questions on Concurrency

```
┌─────────────────────────────────────────────────────────────────────┐
│              CONCURRENCY INTERVIEW QUESTIONS                         │
├─────────────────────────────────────────────────────────────────────┤
│                                                                       │
│  1. "How do you handle concurrent access to parking spots?"          │
│  2. "What if two users try to book the same seat?"                   │
│  3. "How do you ensure thread safety in your design?"                │
│  4. "What locking mechanism would you use?"                          │
│  5. "How do you prevent deadlocks?"                                  │
│  6. "What about race conditions?"                                     │
│                                                                       │
└─────────────────────────────────────────────────────────────────────┘
```

### Synchronization Options

```
┌─────────────────────────────────────────────────────────────────────┐
│                   SYNCHRONIZATION OPTIONS                            │
├──────────────────────┬──────────────────────────────────────────────┤
│  synchronized        │ Simple, blocks entire method/block           │
│  ReentrantLock       │ More control, try-lock, fairness option      │
│  ReadWriteLock       │ Multiple readers, single writer              │
│  AtomicVariables     │ Lock-free, for single variables              │
│  ConcurrentHashMap   │ Thread-safe map with better concurrency      │
│  Semaphore           │ Control access to limited resources          │
│  CountDownLatch      │ Wait for multiple threads to complete        │
└──────────────────────┴──────────────────────────────────────────────┘
```

### 1. Using synchronized

```java
public class ParkingSpot {
    private final String id;
    private volatile boolean isOccupied;
    private Vehicle currentVehicle;

    // Method-level synchronization
    public synchronized boolean park(Vehicle vehicle) {
        if (isOccupied) {
            return false;
        }
        this.currentVehicle = vehicle;
        this.isOccupied = true;
        return true;
    }

    public synchronized Vehicle unpark() {
        if (!isOccupied) {
            return null;
        }
        Vehicle vehicle = this.currentVehicle;
        this.currentVehicle = null;
        this.isOccupied = false;
        return vehicle;
    }
}
```

### 2. Using ReentrantLock (Preferred for Complex Cases)

```java
public class ParkingSpot {
    private final String id;
    private boolean isOccupied;
    private Vehicle currentVehicle;
    private final ReentrantLock lock = new ReentrantLock();

    public boolean park(Vehicle vehicle) {
        lock.lock();
        try {
            if (isOccupied) {
                return false;
            }
            this.currentVehicle = vehicle;
            this.isOccupied = true;
            return true;
        } finally {
            lock.unlock(); // ALWAYS unlock in finally!
        }
    }

    // Try-lock: non-blocking attempt
    public boolean tryPark(Vehicle vehicle, long timeout, TimeUnit unit)
            throws InterruptedException {
        if (lock.tryLock(timeout, unit)) {
            try {
                if (isOccupied) {
                    return false;
                }
                this.currentVehicle = vehicle;
                this.isOccupied = true;
                return true;
            } finally {
                lock.unlock();
            }
        }
        return false; // Couldn't acquire lock in time
    }
}
```

### 3. Using ReadWriteLock (Multiple Readers, Single Writer)

```java
public class AvailabilityCache {
    private final Map<String, Integer> spotCounts = new HashMap<>();
    private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
    private final Lock readLock = rwLock.readLock();
    private final Lock writeLock = rwLock.writeLock();

    // Multiple threads can read simultaneously
    public int getAvailableSpots(String floorId) {
        readLock.lock();
        try {
            return spotCounts.getOrDefault(floorId, 0);
        } finally {
            readLock.unlock();
        }
    }

    // Only one thread can write at a time
    public void updateAvailability(String floorId, int delta) {
        writeLock.lock();
        try {
            int current = spotCounts.getOrDefault(floorId, 0);
            spotCounts.put(floorId, current + delta);
        } finally {
            writeLock.unlock();
        }
    }
}
```

### 4. Atomic Variables

```java
public class TicketCounter {
    private final AtomicLong counter = new AtomicLong(0);

    public long getNextTicketNumber() {
        return counter.incrementAndGet();
    }
}

public class ParkingSpot {
    private final AtomicBoolean isOccupied = new AtomicBoolean(false);
    private volatile Vehicle currentVehicle;

    public boolean park(Vehicle vehicle) {
        // compareAndSet is atomic - only one thread can succeed
        if (isOccupied.compareAndSet(false, true)) {
            this.currentVehicle = vehicle;
            return true;
        }
        return false;
    }

    public Vehicle unpark() {
        Vehicle vehicle = this.currentVehicle;
        this.currentVehicle = null;
        isOccupied.set(false);
        return vehicle;
    }
}
```

### 5. ConcurrentHashMap

```java
public class SpotManager {
    // Thread-safe map with better concurrency than synchronized HashMap
    private final ConcurrentHashMap<String, ParkingSpot> spots = new ConcurrentHashMap<>();

    public void addSpot(ParkingSpot spot) {
        spots.put(spot.getId(), spot);
    }

    public ParkingSpot getSpot(String id) {
        return spots.get(id);
    }

    // Atomic compute operations
    public boolean reserveSpot(String spotId, Vehicle vehicle) {
        return spots.compute(spotId, (id, spot) -> {
            if (spot != null && !spot.isOccupied()) {
                spot.park(vehicle);
            }
            return spot;
        }) != null;
    }
}
```

### 6. Semaphore (Limited Resource Access)

```java
public class ParkingFloor {
    private final Semaphore availableSpots;
    private final List<ParkingSpot> spots;

    public ParkingFloor(int capacity) {
        this.availableSpots = new Semaphore(capacity);
        this.spots = new ArrayList<>();
    }

    public boolean parkVehicle(Vehicle vehicle) throws InterruptedException {
        // Try to acquire a permit (blocks if none available)
        if (availableSpots.tryAcquire(5, TimeUnit.SECONDS)) {
            try {
                // Find and occupy an actual spot
                for (ParkingSpot spot : spots) {
                    if (spot.park(vehicle)) {
                        return true;
                    }
                }
                // If no spot found, release permit
                availableSpots.release();
                return false;
            } catch (Exception e) {
                availableSpots.release();
                throw e;
            }
        }
        return false;
    }

    public void unparkVehicle(String spotId) {
        // Release permit when vehicle leaves
        spots.stream()
            .filter(s -> s.getId().equals(spotId))
            .findFirst()
            .ifPresent(spot -> {
                spot.unpark();
                availableSpots.release();
            });
    }
}
```

### Common Concurrency Patterns

```java
// 1. Double-Checked Locking (for lazy initialization)
public class LazyResource {
    private volatile Resource resource;

    public Resource getResource() {
        if (resource == null) {                 // First check (no lock)
            synchronized (this) {
                if (resource == null) {         // Second check (with lock)
                    resource = new Resource();
                }
            }
        }
        return resource;
    }
}

// 2. Producer-Consumer with BlockingQueue
public class TaskQueue {
    private final BlockingQueue<Task> queue = new LinkedBlockingQueue<>(100);

    public void submit(Task task) throws InterruptedException {
        queue.put(task);  // Blocks if queue is full
    }

    public Task getNextTask() throws InterruptedException {
        return queue.take();  // Blocks if queue is empty
    }
}

// 3. Copy-on-Write for mostly-read scenarios
public class ConfigStore {
    private volatile List<String> configs = new ArrayList<>();

    public List<String> getConfigs() {
        return configs;  // Safe to read
    }

    public synchronized void updateConfigs(List<String> newConfigs) {
        configs = new ArrayList<>(newConfigs);  // Copy on write
    }
}
```

### Deadlock Prevention

```java
// ❌ POTENTIAL DEADLOCK
public void transfer(Account from, Account to, double amount) {
    synchronized (from) {
        synchronized (to) {
            from.withdraw(amount);
            to.deposit(amount);
        }
    }
}

// ✅ DEADLOCK PREVENTION: Always acquire locks in consistent order
public void transfer(Account from, Account to, double amount) {
    Account first = from.getId().compareTo(to.getId()) < 0 ? from : to;
    Account second = from.getId().compareTo(to.getId()) < 0 ? to : from;

    synchronized (first) {
        synchronized (second) {
            from.withdraw(amount);
            to.deposit(amount);
        }
    }
}

// ✅ ALTERNATIVE: Use tryLock with timeout
public boolean transfer(Account from, Account to, double amount) {
    while (true) {
        if (from.getLock().tryLock()) {
            try {
                if (to.getLock().tryLock()) {
                    try {
                        from.withdraw(amount);
                        to.deposit(amount);
                        return true;
                    } finally {
                        to.getLock().unlock();
                    }
                }
            } finally {
                from.getLock().unlock();
            }
        }
        // Back off before retrying
        Thread.sleep(random.nextInt(100));
    }
}
```

---

## 7. Worked Example 1: Parking Lot System

### Requirements Gathering

```
Interviewer: Design a Parking Lot System

You: "Let me clarify the requirements:
1. What types of vehicles? (Motorcycle, Car, Bus)
2. Multiple floors or single floor?
3. Do we need payment system?
4. Different spot sizes for different vehicles?
5. Should support concurrent parking?
6. Do we need entry/exit gates?"

Interviewer: "Yes to all. Let's focus on core parking logic first,
then we can extend."
```

### Entity Identification

```
Entities (Nouns):
• ParkingLot - The main system
• ParkingFloor - Each floor in the lot
• ParkingSpot - Individual parking spots
• Vehicle - Cars, motorcycles, buses
• Ticket - Issued when vehicle enters
• Payment - For calculating fees

Behaviors (Verbs):
• parkVehicle()
• unparkVehicle()
• findAvailableSpot()
• calculateFee()
• issueTicket()
```

### Class Design

```java
// ============ ENUMS ============
public enum VehicleType {
    MOTORCYCLE, CAR, BUS
}

public enum SpotType {
    COMPACT(VehicleType.MOTORCYCLE),
    REGULAR(VehicleType.CAR),
    LARGE(VehicleType.BUS);

    private final VehicleType suitableFor;

    SpotType(VehicleType suitableFor) {
        this.suitableFor = suitableFor;
    }

    public boolean canFit(VehicleType vehicleType) {
        return this.ordinal() >= vehicleType.ordinal();
    }
}

public enum SpotStatus {
    AVAILABLE, OCCUPIED, RESERVED, OUT_OF_SERVICE
}

// ============ VEHICLE (Abstract) ============
public abstract class Vehicle {
    private final String licensePlate;
    private final VehicleType type;

    protected Vehicle(String licensePlate, VehicleType type) {
        this.licensePlate = licensePlate;
        this.type = type;
    }

    public String getLicensePlate() { return licensePlate; }
    public VehicleType getType() { return type; }
}

public class Car extends Vehicle {
    public Car(String licensePlate) {
        super(licensePlate, VehicleType.CAR);
    }
}

public class Motorcycle extends Vehicle {
    public Motorcycle(String licensePlate) {
        super(licensePlate, VehicleType.MOTORCYCLE);
    }
}

public class Bus extends Vehicle {
    public Bus(String licensePlate) {
        super(licensePlate, VehicleType.BUS);
    }
}

// ============ PARKING SPOT ============
public class ParkingSpot {
    private final String id;
    private final SpotType type;
    private final int floorNumber;
    private SpotStatus status;
    private Vehicle parkedVehicle;
    private final ReentrantLock lock = new ReentrantLock();

    public ParkingSpot(String id, SpotType type, int floorNumber) {
        this.id = id;
        this.type = type;
        this.floorNumber = floorNumber;
        this.status = SpotStatus.AVAILABLE;
    }

    public boolean canFitVehicle(Vehicle vehicle) {
        return type.canFit(vehicle.getType());
    }

    public boolean park(Vehicle vehicle) {
        lock.lock();
        try {
            if (status != SpotStatus.AVAILABLE || !canFitVehicle(vehicle)) {
                return false;
            }
            this.parkedVehicle = vehicle;
            this.status = SpotStatus.OCCUPIED;
            return true;
        } finally {
            lock.unlock();
        }
    }

    public Vehicle unpark() {
        lock.lock();
        try {
            if (status != SpotStatus.OCCUPIED) {
                return null;
            }
            Vehicle vehicle = this.parkedVehicle;
            this.parkedVehicle = null;
            this.status = SpotStatus.AVAILABLE;
            return vehicle;
        } finally {
            lock.unlock();
        }
    }

    public boolean isAvailable() {
        return status == SpotStatus.AVAILABLE;
    }

    // Getters
    public String getId() { return id; }
    public SpotType getType() { return type; }
    public int getFloorNumber() { return floorNumber; }
    public SpotStatus getStatus() { return status; }
}

// ============ PARKING FLOOR ============
public class ParkingFloor {
    private final int floorNumber;
    private final List<ParkingSpot> spots;
    private final Map<SpotType, List<ParkingSpot>> spotsByType;

    public ParkingFloor(int floorNumber, int compactSpots, int regularSpots, int largeSpots) {
        this.floorNumber = floorNumber;
        this.spots = new ArrayList<>();
        this.spotsByType = new EnumMap<>(SpotType.class);

        initializeSpots(compactSpots, regularSpots, largeSpots);
    }

    private void initializeSpots(int compact, int regular, int large) {
        spotsByType.put(SpotType.COMPACT, new ArrayList<>());
        spotsByType.put(SpotType.REGULAR, new ArrayList<>());
        spotsByType.put(SpotType.LARGE, new ArrayList<>());

        for (int i = 0; i < compact; i++) {
            ParkingSpot spot = new ParkingSpot(
                floorNumber + "-C-" + i, SpotType.COMPACT, floorNumber
            );
            spots.add(spot);
            spotsByType.get(SpotType.COMPACT).add(spot);
        }

        for (int i = 0; i < regular; i++) {
            ParkingSpot spot = new ParkingSpot(
                floorNumber + "-R-" + i, SpotType.REGULAR, floorNumber
            );
            spots.add(spot);
            spotsByType.get(SpotType.REGULAR).add(spot);
        }

        for (int i = 0; i < large; i++) {
            ParkingSpot spot = new ParkingSpot(
                floorNumber + "-L-" + i, SpotType.LARGE, floorNumber
            );
            spots.add(spot);
            spotsByType.get(SpotType.LARGE).add(spot);
        }
    }

    public Optional<ParkingSpot> findAvailableSpot(VehicleType vehicleType) {
        // Find the smallest suitable spot type
        for (SpotType spotType : SpotType.values()) {
            if (spotType.canFit(vehicleType)) {
                Optional<ParkingSpot> spot = spotsByType.get(spotType).stream()
                    .filter(ParkingSpot::isAvailable)
                    .findFirst();
                if (spot.isPresent()) {
                    return spot;
                }
            }
        }
        return Optional.empty();
    }

    public int getFloorNumber() { return floorNumber; }
    public List<ParkingSpot> getSpots() { return Collections.unmodifiableList(spots); }
}

// ============ TICKET ============
public class ParkingTicket {
    private final String ticketId;
    private final Vehicle vehicle;
    private final ParkingSpot spot;
    private final LocalDateTime entryTime;
    private LocalDateTime exitTime;
    private double fee;
    private boolean isPaid;

    public ParkingTicket(String ticketId, Vehicle vehicle, ParkingSpot spot) {
        this.ticketId = ticketId;
        this.vehicle = vehicle;
        this.spot = spot;
        this.entryTime = LocalDateTime.now();
        this.isPaid = false;
    }

    public void markExit() {
        this.exitTime = LocalDateTime.now();
    }

    public long getParkedHours() {
        LocalDateTime end = exitTime != null ? exitTime : LocalDateTime.now();
        return ChronoUnit.HOURS.between(entryTime, end) + 1; // Minimum 1 hour
    }

    // Getters and setters
    public String getTicketId() { return ticketId; }
    public Vehicle getVehicle() { return vehicle; }
    public ParkingSpot getSpot() { return spot; }
    public LocalDateTime getEntryTime() { return entryTime; }
    public void setFee(double fee) { this.fee = fee; }
    public double getFee() { return fee; }
    public void setPaid(boolean paid) { this.isPaid = paid; }
    public boolean isPaid() { return isPaid; }
}

// ============ PRICING STRATEGY ============
public interface PricingStrategy {
    double calculateFee(ParkingTicket ticket);
}

public class HourlyPricingStrategy implements PricingStrategy {
    private final Map<VehicleType, Double> hourlyRates;

    public HourlyPricingStrategy() {
        hourlyRates = new EnumMap<>(VehicleType.class);
        hourlyRates.put(VehicleType.MOTORCYCLE, 10.0);
        hourlyRates.put(VehicleType.CAR, 20.0);
        hourlyRates.put(VehicleType.BUS, 30.0);
    }

    @Override
    public double calculateFee(ParkingTicket ticket) {
        double hourlyRate = hourlyRates.get(ticket.getVehicle().getType());
        return hourlyRate * ticket.getParkedHours();
    }
}

// ============ PARKING LOT (Main Class) ============
public class ParkingLot {
    private static ParkingLot instance;

    private final String name;
    private final List<ParkingFloor> floors;
    private final Map<String, ParkingTicket> activeTickets;
    private final PricingStrategy pricingStrategy;
    private final AtomicLong ticketCounter;
    private final ReentrantReadWriteLock rwLock;

    private ParkingLot(String name, int numFloors, int spotsPerFloor) {
        this.name = name;
        this.floors = new ArrayList<>();
        this.activeTickets = new ConcurrentHashMap<>();
        this.pricingStrategy = new HourlyPricingStrategy();
        this.ticketCounter = new AtomicLong(0);
        this.rwLock = new ReentrantReadWriteLock();

        initializeFloors(numFloors, spotsPerFloor);
    }

    // Singleton pattern
    public static synchronized ParkingLot getInstance(String name, int floors, int spots) {
        if (instance == null) {
            instance = new ParkingLot(name, floors, spots);
        }
        return instance;
    }

    private void initializeFloors(int numFloors, int spotsPerFloor) {
        for (int i = 0; i < numFloors; i++) {
            // 20% compact, 60% regular, 20% large
            int compact = (int) (spotsPerFloor * 0.2);
            int regular = (int) (spotsPerFloor * 0.6);
            int large = spotsPerFloor - compact - regular;
            floors.add(new ParkingFloor(i, compact, regular, large));
        }
    }

    public Optional<ParkingTicket> parkVehicle(Vehicle vehicle) {
        rwLock.readLock().lock();
        try {
            for (ParkingFloor floor : floors) {
                Optional<ParkingSpot> spotOpt = floor.findAvailableSpot(vehicle.getType());
                if (spotOpt.isPresent()) {
                    ParkingSpot spot = spotOpt.get();
                    if (spot.park(vehicle)) {
                        ParkingTicket ticket = new ParkingTicket(
                            generateTicketId(),
                            vehicle,
                            spot
                        );
                        activeTickets.put(ticket.getTicketId(), ticket);
                        return Optional.of(ticket);
                    }
                }
            }
        } finally {
            rwLock.readLock().unlock();
        }
        return Optional.empty();
    }

    public Optional<ParkingTicket> unparkVehicle(String ticketId) {
        ParkingTicket ticket = activeTickets.get(ticketId);
        if (ticket == null) {
            return Optional.empty();
        }

        ticket.markExit();
        double fee = pricingStrategy.calculateFee(ticket);
        ticket.setFee(fee);

        // Process payment (simplified)
        ticket.setPaid(true);

        // Release the spot
        ticket.getSpot().unpark();
        activeTickets.remove(ticketId);

        return Optional.of(ticket);
    }

    private String generateTicketId() {
        return "TKT-" + ticketCounter.incrementAndGet();
    }

    public int getAvailableSpotsCount() {
        return floors.stream()
            .flatMap(floor -> floor.getSpots().stream())
            .filter(ParkingSpot::isAvailable)
            .mapToInt(s -> 1)
            .sum();
    }
}

// ============ MAIN/DEMO ============
public class ParkingLotDemo {
    public static void main(String[] args) {
        ParkingLot lot = ParkingLot.getInstance("Downtown Parking", 3, 100);

        Vehicle car = new Car("ABC-123");
        Vehicle bike = new Motorcycle("XYZ-789");

        // Park vehicles
        Optional<ParkingTicket> carTicket = lot.parkVehicle(car);
        Optional<ParkingTicket> bikeTicket = lot.parkVehicle(bike);

        carTicket.ifPresent(ticket ->
            System.out.println("Car parked at spot: " + ticket.getSpot().getId())
        );

        // Later... unpark
        carTicket.ifPresent(ticket -> {
            Optional<ParkingTicket> exitTicket = lot.unparkVehicle(ticket.getTicketId());
            exitTicket.ifPresent(t ->
                System.out.println("Total fee: $" + t.getFee())
            );
        });
    }
}
```

---

## 8. Worked Example 2: Splitwise (Expense Sharing)

### Requirements Gathering

```
Interviewer: Design Splitwise

You: "Let me understand the requirements:
1. Users can create groups?
2. Users can add expenses to groups?
3. What types of splits? (Equal, Exact, Percentage)
4. How do we settle balances?
5. Do we need transaction history?
6. Any concurrent access concerns?"
```

### Class Design

```java
// ============ ENUMS ============
public enum SplitType {
    EQUAL, EXACT, PERCENTAGE
}

// ============ USER ============
public class User {
    private final String id;
    private final String name;
    private final String email;

    public User(String id, String name, String email) {
        this.id = id;
        this.name = name;
        this.email = email;
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public String getEmail() { return email; }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        User user = (User) o;
        return id.equals(user.id);
    }

    @Override
    public int hashCode() {
        return Objects.hash(id);
    }
}

// ============ SPLIT (Abstract) ============
public abstract class Split {
    protected final User user;
    protected double amount;

    protected Split(User user) {
        this.user = user;
    }

    public User getUser() { return user; }
    public double getAmount() { return amount; }
    public void setAmount(double amount) { this.amount = amount; }

    public abstract boolean validate();
}

public class EqualSplit extends Split {
    public EqualSplit(User user) {
        super(user);
    }

    @Override
    public boolean validate() {
        return true; // Amount will be calculated
    }
}

public class ExactSplit extends Split {
    public ExactSplit(User user, double amount) {
        super(user);
        this.amount = amount;
    }

    @Override
    public boolean validate() {
        return amount > 0;
    }
}

public class PercentageSplit extends Split {
    private final double percentage;

    public PercentageSplit(User user, double percentage) {
        super(user);
        this.percentage = percentage;
    }

    public double getPercentage() { return percentage; }

    @Override
    public boolean validate() {
        return percentage > 0 && percentage <= 100;
    }
}

// ============ EXPENSE ============
public class Expense {
    private final String id;
    private final String description;
    private final double totalAmount;
    private final User paidBy;
    private final List<Split> splits;
    private final LocalDateTime createdAt;

    public Expense(String id, String description, double totalAmount,
                   User paidBy, List<Split> splits) {
        this.id = id;
        this.description = description;
        this.totalAmount = totalAmount;
        this.paidBy = paidBy;
        this.splits = new ArrayList<>(splits);
        this.createdAt = LocalDateTime.now();

        calculateSplitAmounts();
    }

    private void calculateSplitAmounts() {
        // For equal splits
        if (splits.get(0) instanceof EqualSplit) {
            double perPerson = totalAmount / splits.size();
            splits.forEach(split -> split.setAmount(perPerson));
        }
        // For percentage splits
        else if (splits.get(0) instanceof PercentageSplit) {
            for (Split split : splits) {
                PercentageSplit pSplit = (PercentageSplit) split;
                split.setAmount(totalAmount * pSplit.getPercentage() / 100);
            }
        }
        // Exact splits already have amounts set
    }

    public boolean validate() {
        // Validate splits
        for (Split split : splits) {
            if (!split.validate()) return false;
        }

        // Validate total
        double totalSplit = splits.stream().mapToDouble(Split::getAmount).sum();
        return Math.abs(totalSplit - totalAmount) < 0.01; // Allow small floating point error
    }

    public String getId() { return id; }
    public double getTotalAmount() { return totalAmount; }
    public User getPaidBy() { return paidBy; }
    public List<Split> getSplits() { return Collections.unmodifiableList(splits); }
}

// ============ GROUP ============
public class Group {
    private final String id;
    private final String name;
    private final List<User> members;
    private final List<Expense> expenses;

    public Group(String id, String name) {
        this.id = id;
        this.name = name;
        this.members = new ArrayList<>();
        this.expenses = new ArrayList<>();
    }

    public void addMember(User user) {
        if (!members.contains(user)) {
            members.add(user);
        }
    }

    public void addExpense(Expense expense) {
        expenses.add(expense);
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public List<User> getMembers() { return Collections.unmodifiableList(members); }
    public List<Expense> getExpenses() { return Collections.unmodifiableList(expenses); }
}

// ============ BALANCE SHEET ============
public class BalanceSheet {
    // Map: FromUserId -> ToUserId -> Amount (FromUser owes ToUser this amount)
    private final Map<String, Map<String, Double>> balances;
    private final ReentrantReadWriteLock lock = new ReentrantReadWriteLock();

    public BalanceSheet() {
        this.balances = new ConcurrentHashMap<>();
    }

    public void addExpense(Expense expense) {
        lock.writeLock().lock();
        try {
            User paidBy = expense.getPaidBy();

            for (Split split : expense.getSplits()) {
                User owes = split.getUser();
                double amount = split.getAmount();

                // Skip if payer owes themselves
                if (owes.equals(paidBy)) continue;

                // owes -> paidBy : amount
                updateBalance(owes.getId(), paidBy.getId(), amount);
            }
        } finally {
            lock.writeLock().unlock();
        }
    }

    private void updateBalance(String fromId, String toId, double amount) {
        balances.computeIfAbsent(fromId, k -> new ConcurrentHashMap<>());
        balances.get(fromId).merge(toId, amount, Double::sum);

        // Simplify: if A owes B 50 and B owes A 30, net is A owes B 20
        simplifyBalances(fromId, toId);
    }

    private void simplifyBalances(String user1, String user2) {
        double user1OwesUser2 = getBalance(user1, user2);
        double user2OwesUser1 = getBalance(user2, user1);

        if (user1OwesUser2 > 0 && user2OwesUser1 > 0) {
            if (user1OwesUser2 > user2OwesUser1) {
                setBalance(user1, user2, user1OwesUser2 - user2OwesUser1);
                setBalance(user2, user1, 0);
            } else {
                setBalance(user2, user1, user2OwesUser1 - user1OwesUser2);
                setBalance(user1, user2, 0);
            }
        }
    }

    private double getBalance(String from, String to) {
        return balances.getOrDefault(from, Collections.emptyMap())
                       .getOrDefault(to, 0.0);
    }

    private void setBalance(String from, String to, double amount) {
        if (amount == 0) {
            if (balances.containsKey(from)) {
                balances.get(from).remove(to);
            }
        } else {
            balances.computeIfAbsent(from, k -> new ConcurrentHashMap<>())
                    .put(to, amount);
        }
    }

    public Map<String, Double> getBalancesForUser(String userId) {
        lock.readLock().lock();
        try {
            Map<String, Double> result = new HashMap<>();

            // What this user owes others
            if (balances.containsKey(userId)) {
                balances.get(userId).forEach((toId, amount) ->
                    result.merge(toId, -amount, Double::sum)  // Negative = user owes
                );
            }

            // What others owe this user
            for (Map.Entry<String, Map<String, Double>> entry : balances.entrySet()) {
                String fromId = entry.getKey();
                Double owedAmount = entry.getValue().get(userId);
                if (owedAmount != null && owedAmount > 0) {
                    result.merge(fromId, owedAmount, Double::sum);  // Positive = user is owed
                }
            }

            return result;
        } finally {
            lock.readLock().unlock();
        }
    }
}

// ============ EXPENSE MANAGER (Main Service) ============
public class ExpenseManager {
    private static ExpenseManager instance;

    private final Map<String, User> users;
    private final Map<String, Group> groups;
    private final BalanceSheet balanceSheet;
    private final AtomicLong idCounter;

    private ExpenseManager() {
        this.users = new ConcurrentHashMap<>();
        this.groups = new ConcurrentHashMap<>();
        this.balanceSheet = new BalanceSheet();
        this.idCounter = new AtomicLong(0);
    }

    public static synchronized ExpenseManager getInstance() {
        if (instance == null) {
            instance = new ExpenseManager();
        }
        return instance;
    }

    public User addUser(String name, String email) {
        String id = "U" + idCounter.incrementAndGet();
        User user = new User(id, name, email);
        users.put(id, user);
        return user;
    }

    public Group createGroup(String name, List<User> members) {
        String id = "G" + idCounter.incrementAndGet();
        Group group = new Group(id, name);
        members.forEach(group::addMember);
        groups.put(id, group);
        return group;
    }

    public Expense addExpense(String description, double amount, User paidBy,
                              List<Split> splits, String groupId) {
        String id = "E" + idCounter.incrementAndGet();
        Expense expense = new Expense(id, description, amount, paidBy, splits);

        if (!expense.validate()) {
            throw new IllegalArgumentException("Invalid expense split");
        }

        // Add to group if specified
        if (groupId != null && groups.containsKey(groupId)) {
            groups.get(groupId).addExpense(expense);
        }

        // Update balance sheet
        balanceSheet.addExpense(expense);

        return expense;
    }

    public Map<String, Double> getBalances(String userId) {
        return balanceSheet.getBalancesForUser(userId);
    }

    public void printBalances(String userId) {
        User user = users.get(userId);
        Map<String, Double> balances = getBalances(userId);

        System.out.println("Balances for " + user.getName() + ":");
        balances.forEach((otherId, amount) -> {
            User other = users.get(otherId);
            if (amount > 0) {
                System.out.printf("  %s owes you: $%.2f%n", other.getName(), amount);
            } else {
                System.out.printf("  You owe %s: $%.2f%n", other.getName(), -amount);
            }
        });
    }
}

// ============ DEMO ============
public class SplitwiseDemo {
    public static void main(String[] args) {
        ExpenseManager manager = ExpenseManager.getInstance();

        // Create users
        User alice = manager.addUser("Alice", "alice@email.com");
        User bob = manager.addUser("Bob", "bob@email.com");
        User charlie = manager.addUser("Charlie", "charlie@email.com");

        // Create group
        Group trip = manager.createGroup("Goa Trip", List.of(alice, bob, charlie));

        // Add expense with equal split
        // Alice paid $300 for dinner, split equally
        manager.addExpense(
            "Dinner",
            300.0,
            alice,
            List.of(
                new EqualSplit(alice),
                new EqualSplit(bob),
                new EqualSplit(charlie)
            ),
            trip.getId()
        );

        // Add expense with exact split
        // Bob paid $200 for cab, Alice owes 80, Charlie owes 120
        manager.addExpense(
            "Cab",
            200.0,
            bob,
            List.of(
                new ExactSplit(alice, 80.0),
                new ExactSplit(charlie, 120.0)
            ),
            trip.getId()
        );

        // Print balances
        manager.printBalances(alice.getId());
        manager.printBalances(bob.getId());
        manager.printBalances(charlie.getId());
    }
}
```

---

## 9. Worked Example 3: Snake and Ladder

### Requirements Gathering

```
Interviewer: Design Snake and Ladder game

You: "Let me clarify:
1. How many players?
2. Standard 100-cell board?
3. Single dice (1-6)?
4. Rules: Exact 100 to win, or can exceed?
5. What happens with snake/ladder at destination?
6. Any multiplayer concurrency concerns?"
```

### Class Design

```java
// ============ PLAYER ============
public class Player {
    private final String id;
    private final String name;
    private int position;

    public Player(String id, String name) {
        this.id = id;
        this.name = name;
        this.position = 0; // Start before the board
    }

    public String getId() { return id; }
    public String getName() { return name; }
    public int getPosition() { return position; }
    public void setPosition(int position) { this.position = position; }
}

// ============ DICE ============
public class Dice {
    private final int numberOfDice;
    private final int sides;
    private final Random random;

    public Dice(int numberOfDice, int sides) {
        this.numberOfDice = numberOfDice;
        this.sides = sides;
        this.random = new Random();
    }

    public int roll() {
        int total = 0;
        for (int i = 0; i < numberOfDice; i++) {
            total += random.nextInt(sides) + 1;
        }
        return total;
    }
}

// ============ BOARD ENTITY (Snake/Ladder) ============
public abstract class BoardEntity {
    protected final int start;
    protected final int end;

    protected BoardEntity(int start, int end) {
        this.start = start;
        this.end = end;
    }

    public int getStart() { return start; }
    public int getEnd() { return end; }
    public abstract String getType();
}

public class Snake extends BoardEntity {
    public Snake(int head, int tail) {
        super(head, tail);
        if (head <= tail) {
            throw new IllegalArgumentException("Snake head must be above tail");
        }
    }

    @Override
    public String getType() { return "Snake"; }
}

public class Ladder extends BoardEntity {
    public Ladder(int bottom, int top) {
        super(bottom, top);
        if (bottom >= top) {
            throw new IllegalArgumentException("Ladder bottom must be below top");
        }
    }

    @Override
    public String getType() { return "Ladder"; }
}

// ============ BOARD ============
public class Board {
    private final int size;
    private final Map<Integer, BoardEntity> entities;

    public Board(int size) {
        this.size = size;
        this.entities = new HashMap<>();
    }

    public void addSnake(int head, int tail) {
        validatePosition(head);
        validatePosition(tail);
        entities.put(head, new Snake(head, tail));
    }

    public void addLadder(int bottom, int top) {
        validatePosition(bottom);
        validatePosition(top);
        entities.put(bottom, new Ladder(bottom, top));
    }

    private void validatePosition(int position) {
        if (position < 1 || position > size) {
            throw new IllegalArgumentException("Position must be between 1 and " + size);
        }
    }

    public int getNextPosition(int currentPosition) {
        // Check if there's a snake or ladder at this position
        if (entities.containsKey(currentPosition)) {
            BoardEntity entity = entities.get(currentPosition);
            System.out.println("  " + entity.getType() + " at " + currentPosition +
                             " -> " + entity.getEnd());
            return entity.getEnd();
        }
        return currentPosition;
    }

    public int getSize() { return size; }
}

// ============ GAME ============
public class Game {
    private final Board board;
    private final Dice dice;
    private final Queue<Player> players;
    private Player winner;
    private GameStatus status;

    public Game(int boardSize, int diceCount, int diceSides) {
        this.board = new Board(boardSize);
        this.dice = new Dice(diceCount, diceSides);
        this.players = new LinkedList<>();
        this.status = GameStatus.NOT_STARTED;
    }

    public void addPlayer(Player player) {
        if (status != GameStatus.NOT_STARTED) {
            throw new IllegalStateException("Cannot add player after game started");
        }
        players.add(player);
    }

    public void addSnake(int head, int tail) {
        board.addSnake(head, tail);
    }

    public void addLadder(int bottom, int top) {
        board.addLadder(bottom, top);
    }

    public void start() {
        if (players.size() < 2) {
            throw new IllegalStateException("Need at least 2 players");
        }
        status = GameStatus.IN_PROGRESS;
        play();
    }

    private void play() {
        while (status == GameStatus.IN_PROGRESS) {
            Player currentPlayer = players.poll();
            playTurn(currentPlayer);

            if (status == GameStatus.IN_PROGRESS) {
                players.add(currentPlayer);
            }
        }
    }

    private void playTurn(Player player) {
        int diceValue = dice.roll();
        int currentPosition = player.getPosition();
        int newPosition = currentPosition + diceValue;

        System.out.println(player.getName() + " rolled " + diceValue +
                          " (at " + currentPosition + ")");

        // Check if exceeds board size
        if (newPosition > board.getSize()) {
            System.out.println("  Stays at " + currentPosition + " (exceeded board)");
            return;
        }

        // Check for snake or ladder
        newPosition = board.getNextPosition(newPosition);
        player.setPosition(newPosition);
        System.out.println("  Moves to " + newPosition);

        // Check for win
        if (newPosition == board.getSize()) {
            winner = player;
            status = GameStatus.FINISHED;
            System.out.println(player.getName() + " WINS!");
        }
    }

    public Player getWinner() { return winner; }
    public GameStatus getStatus() { return status; }
}

public enum GameStatus {
    NOT_STARTED, IN_PROGRESS, FINISHED
}

// ============ DEMO ============
public class SnakeAndLadderDemo {
    public static void main(String[] args) {
        Game game = new Game(100, 1, 6);

        // Add players
        game.addPlayer(new Player("P1", "Alice"));
        game.addPlayer(new Player("P2", "Bob"));

        // Add snakes
        game.addSnake(99, 10);
        game.addSnake(70, 30);
        game.addSnake(55, 15);

        // Add ladders
        game.addLadder(5, 25);
        game.addLadder(40, 75);
        game.addLadder(60, 95);

        // Start game
        game.start();
    }
}
```

---

## 10. Common Follow-up Questions & Answers

### Category 1: Concurrency

| Question | Answer |
|----------|--------|
| "What if two users try to book the same spot?" | Use optimistic locking (version field) or pessimistic locking (synchronized/ReentrantLock). Prefer CAS operations for high concurrency. |
| "How do you ensure thread safety?" | Use immutable objects where possible, synchronized blocks for critical sections, ConcurrentHashMap for shared state, atomic variables for counters. |
| "How to prevent deadlocks?" | Always acquire locks in consistent order, use tryLock with timeout, minimize lock scope. |
| "What locking mechanism would you choose?" | ReentrantLock for complex scenarios (tryLock, conditions), synchronized for simple cases, ReadWriteLock when reads >> writes. |

### Category 2: Extensibility

| Question | Answer |
|----------|--------|
| "How to add a new vehicle type?" | Add new enum value + create new class extending Vehicle. Open/Closed principle - no existing code changes. |
| "How to add new split type in Splitwise?" | Create new class implementing Split interface. Strategy pattern handles this elegantly. |
| "How to support multiple payment methods?" | Strategy pattern - each payment method implements PaymentStrategy interface. |
| "How to add new game rules?" | Use Command pattern for rules, or State pattern if behavior changes based on game state. |

### Category 3: Scale

| Question | Answer |
|----------|--------|
| "What if we have 1 million users?" | Replace in-memory storage with database. Use caching (Redis). Add indexing for frequent queries. |
| "How to handle high-frequency reads?" | ReadWriteLock, Read replicas, Caching layer, Eventually consistent reads where acceptable. |
| "How to distribute across servers?" | Consistent hashing for partitioning, Message queues for async processing. (Note: This ventures into HLD) |

### Category 4: Error Handling

| Question | Answer |
|----------|--------|
| "What if payment fails after spot reserved?" | Use compensation/rollback pattern. Release reservation. Consider Saga pattern for distributed transactions. |
| "What if system crashes mid-transaction?" | Use write-ahead logging, Store state before operations, Implement idempotency. |
| "How to handle network failures?" | Retry with exponential backoff, Circuit breaker pattern, Graceful degradation. |

### Category 5: Design Decisions

| Question | Answer |
|----------|--------|
| "Why use Strategy vs Factory?" | Strategy: runtime behavior change. Factory: object creation logic. Use Strategy when algorithms vary, Factory when object types vary. |
| "Why not use inheritance for vehicles?" | Composition over inheritance. Inheritance creates tight coupling. Use interfaces + composition for flexibility. |
| "When to use Singleton?" | Only when exactly one instance is needed system-wide (Config, Connection Pool). Avoid for testability concerns. |
| "Why abstract class vs interface?" | Abstract class: shared code + partial implementation. Interface: contract only. Java 8+: interfaces can have default methods. |

### Category 6: Data Modeling

| Question | Answer |
|----------|--------|
| "How to store historical data?" | Separate historical tables, Event sourcing for audit trail, Soft deletes for recovery. |
| "How to optimize balance calculations?" | Precompute and cache balances, Update on write (denormalization), Eventual consistency. |
| "How to handle currency/decimals?" | Use BigDecimal for money, not double. Store in smallest units (cents). |

---

## 11. Quick Reference Cheatsheet

### Design Pattern Selection

```
┌──────────────────────────────────────────────────────────────────────────┐
│                    PATTERN QUICK REFERENCE                                │
├────────────────────────┬─────────────────────────────────────────────────┤
│ Multiple algorithms    │ → Strategy                                       │
│ Object creation        │ → Factory                                        │
│ Single instance        │ → Singleton (Bill Pugh)                          │
│ Event notifications    │ → Observer                                       │
│ State-based behavior   │ → State                                          │
│ Add responsibilities   │ → Decorator                                      │
│ Request chain          │ → Chain of Responsibility                        │
│ Undo operations        │ → Command                                        │
│ Object variations      │ → Builder                                        │
└────────────────────────┴─────────────────────────────────────────────────┘
```

### Thread Safety Quick Reference

```java
// Single variable, high concurrency
AtomicInteger counter = new AtomicInteger(0);
counter.incrementAndGet();

// Simple synchronization
synchronized (lock) { /* critical section */ }

// Advanced locking
ReentrantLock lock = new ReentrantLock();
lock.lock();
try { /* ... */ } finally { lock.unlock(); }

// Read-heavy workloads
ReadWriteLock rwLock = new ReentrantReadWriteLock();
rwLock.readLock().lock();  // Multiple readers OK
rwLock.writeLock().lock(); // Single writer

// Thread-safe collections
ConcurrentHashMap<K, V> map = new ConcurrentHashMap<>();
CopyOnWriteArrayList<T> list = new CopyOnWriteArrayList<>();
BlockingQueue<T> queue = new LinkedBlockingQueue<>();
```

### Class Template

```java
public class Entity {
    // 1. Immutable fields
    private final String id;

    // 2. Mutable state (with thread safety if needed)
    private volatile Status status;

    // 3. Relationships
    private final List<Child> children = new ArrayList<>();

    // 4. Thread safety (if concurrent)
    private final ReentrantLock lock = new ReentrantLock();

    // 5. Constructor (validation + initialization)
    public Entity(String id) {
        this.id = Objects.requireNonNull(id);
        this.status = Status.ACTIVE;
    }

    // 6. Core business methods
    public void performAction() { /* ... */ }

    // 7. Getters (no setters for immutables)
    public String getId() { return id; }
}
```

### Interview Checklist

```
Before Implementation:
□ Clarified requirements (5 min)
□ Identified actors and use cases
□ Listed main entities and relationships
□ Asked about concurrency requirements

During Implementation:
□ Started with enums for types/statuses
□ Created interfaces before implementations
□ Used final for immutables
□ Added thread safety where needed
□ Used descriptive variable names

After Implementation:
□ Validated design handles edge cases
□ Ensured extensibility (SOLID)
□ Can explain tradeoffs
□ Ready for follow-up questions
```

---

## Final Tips

1. **Start Simple**: Don't over-engineer. Start with working code, then extend.

2. **Think Out Loud**: Explain your thought process. Interviewers evaluate how you think.

3. **Ask Questions**: It shows you think about edge cases and requirements.

4. **Use Patterns Wisely**: Don't force patterns. Use them when they solve a real problem.

5. **Handle Errors**: Show you think about failure cases.

6. **Name Things Well**: `findAvailableSpot()` > `find()` > `f()`

7. **Test Mentally**: Walk through your code with a simple example.

8. **Stay Calm**: If stuck, verbalize what you're thinking. Interviewers often help.

---

*Good luck with your interviews!*
