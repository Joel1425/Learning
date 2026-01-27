# Java Spring Boot Interview Complete Guide
## SDE-2 Backend Interview Preparation

---

## Table of Contents

1. [Why Spring Boot?](#1-why-spring-boot)
2. [Core Concepts - The Foundation](#2-core-concepts---the-foundation)
   - [Inversion of Control (IoC)](#21-inversion-of-control-ioc)
   - [Dependency Injection (DI)](#22-dependency-injection-di)
   - [Spring Beans](#23-spring-beans)
   - [ApplicationContext](#24-applicationcontext)
3. [Annotations - The Building Blocks](#3-annotations---the-building-blocks)
4. [Spring Boot Auto-Configuration](#4-spring-boot-auto-configuration)
5. [REST API Development](#5-rest-api-development)
6. [Exception Handling](#6-exception-handling)
7. [Validation](#7-validation)
8. [Spring Data JPA](#8-spring-data-jpa)
9. [Spring Security Basics](#9-spring-security-basics)
10. [Testing in Spring Boot](#10-testing-in-spring-boot)
11. [Configuration Management](#11-configuration-management)
12. [Common Interview Q&A with Follow-ups](#12-common-interview-qa-with-follow-ups)

---

## 1. Why Spring Boot?

### The Problem Before Spring Boot

Before Spring Boot, setting up a Spring application was painful:

```
Traditional Spring Setup:
┌─────────────────────────────────────────────────────────────────┐
│  1. Create web.xml (Deployment Descriptor)                      │
│  2. Configure DispatcherServlet                                 │
│  3. Create applicationContext.xml                               │
│  4. Configure ViewResolver, DataSource, TransactionManager      │
│  5. Add all dependencies manually with correct versions         │
│  6. Configure logging, security, etc.                           │
│  7. Package as WAR and deploy to external server                │
└─────────────────────────────────────────────────────────────────┘
         ↓
    50+ XML configuration lines just to say "Hello World"!
```

### Spring Boot Solution

```
Spring Boot Setup:
┌─────────────────────────────────────────────────────────────────┐
│  1. Add spring-boot-starter-web dependency                      │
│  2. Write @SpringBootApplication class                          │
│  3. Run!                                                        │
└─────────────────────────────────────────────────────────────────┘
         ↓
    3 steps to production-ready application!
```

### Key Benefits of Spring Boot

| Feature | What It Means | Example |
|---------|---------------|---------|
| **Auto-Configuration** | Spring Boot automatically configures your application based on dependencies | Add `spring-boot-starter-data-jpa` → DataSource, EntityManager auto-configured |
| **Starter Dependencies** | Pre-packaged dependency bundles | `spring-boot-starter-web` includes Tomcat, Jackson, Spring MVC |
| **Embedded Server** | No need for external server | Built-in Tomcat/Jetty/Undertow |
| **Production Ready** | Health checks, metrics out of box | Actuator endpoints |
| **Opinionated Defaults** | Sensible defaults, less decisions | Convention over configuration |

### Interview Answer Template

> **Q: Why use Spring Boot over traditional Spring?**
>
> **A:** "Spring Boot eliminates boilerplate configuration through auto-configuration and starter dependencies. It provides an embedded server, production-ready features like health checks via Actuator, and follows convention over configuration. This lets developers focus on business logic rather than infrastructure setup. For example, adding `spring-boot-starter-web` auto-configures an embedded Tomcat server, JSON serialization with Jackson, and Spring MVC - all without a single XML file."

---

## 2. Core Concepts - The Foundation

### 2.1 Inversion of Control (IoC)

#### What is IoC?

**Simple Analogy:** Think of IoC like ordering food at a restaurant vs cooking at home.

```
WITHOUT IoC (Cooking at Home):
┌─────────────────────────────────────────┐
│  You (Code) control everything:         │
│  - Buy ingredients                      │
│  - Prepare food                         │
│  - Cook food                            │
│  - Serve food                           │
│  - Clean up                             │
│                                         │
│  YOU are in control of the flow         │
└─────────────────────────────────────────┘

WITH IoC (Restaurant):
┌─────────────────────────────────────────┐
│  Restaurant (Framework) controls:        │
│  - Manages ingredients                  │
│  - Prepares food                        │
│  - Serves you                           │
│                                         │
│  You just ORDER and RECEIVE             │
│  CONTROL is INVERTED to the framework   │
└─────────────────────────────────────────┘
```

#### Code Example - Without IoC

```java
// WITHOUT IoC - You create and manage dependencies yourself
public class OrderService {

    // You are CREATING the dependency
    private EmailService emailService = new EmailService();
    private PaymentService paymentService = new PaymentService();

    public void placeOrder(Order order) {
        paymentService.processPayment(order);
        emailService.sendConfirmation(order);
    }
}

// Problems:
// 1. OrderService is tightly coupled to EmailService and PaymentService
// 2. Hard to test (can't mock EmailService)
// 3. Hard to change (what if you want SMSService instead?)
```

#### Code Example - With IoC

```java
// WITH IoC - Framework creates and manages dependencies
@Service
public class OrderService {

    // Dependencies are INJECTED by Spring (you don't create them)
    private final EmailService emailService;
    private final PaymentService paymentService;

    // Spring provides the dependencies through constructor
    public OrderService(EmailService emailService, PaymentService paymentService) {
        this.emailService = emailService;
        this.paymentService = paymentService;
    }

    public void placeOrder(Order order) {
        paymentService.processPayment(order);
        emailService.sendConfirmation(order);
    }
}

// Benefits:
// 1. Loose coupling - OrderService doesn't know HOW to create EmailService
// 2. Easy to test - can inject mock services
// 3. Easy to change - just configure different implementation
```

#### IoC Container Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    SPRING IoC CONTAINER                          │
│                                                                   │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │ Configuration │    │    Bean      │    │    Bean      │       │
│  │  Metadata     │───▶│  Definitions │───▶│  Instances   │       │
│  │ (Annotations/ │    │              │    │              │       │
│  │   XML/Java)   │    │              │    │              │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
│         │                   │                   │                │
│         ▼                   ▼                   ▼                │
│  ┌──────────────────────────────────────────────────────┐       │
│  │              FULLY CONFIGURED SYSTEM                  │       │
│  │                  (Ready to Use)                       │       │
│  └──────────────────────────────────────────────────────┘       │
└─────────────────────────────────────────────────────────────────┘
```

---

### 2.2 Dependency Injection (DI)

#### What is DI?

**DI is HOW IoC is implemented.** IoC is the principle, DI is the pattern.

```
IoC = "Don't call us, we'll call you" (Hollywood Principle)
DI  = "Here are your dependencies, use them"
```

#### Types of Dependency Injection

```
┌─────────────────────────────────────────────────────────────────┐
│                 DEPENDENCY INJECTION TYPES                       │
├─────────────────┬───────────────────┬───────────────────────────┤
│   CONSTRUCTOR   │     SETTER        │      FIELD                │
│   INJECTION     │   INJECTION       │    INJECTION              │
│   (Recommended) │                   │   (Not Recommended)       │
├─────────────────┼───────────────────┼───────────────────────────┤
│  Dependencies   │  Dependencies     │  Dependencies injected    │
│  via constructor│  via setter       │  directly into field      │
│  parameters     │  methods          │  using reflection         │
└─────────────────┴───────────────────┴───────────────────────────┘
```

#### 1. Constructor Injection (RECOMMENDED)

```java
@Service
public class UserService {

    private final UserRepository userRepository;  // final = immutable
    private final EmailService emailService;

    // Constructor Injection - Spring calls this constructor
    // @Autowired is optional for single constructor (Spring 4.3+)
    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }

    public User createUser(User user) {
        User savedUser = userRepository.save(user);
        emailService.sendWelcomeEmail(savedUser);
        return savedUser;
    }
}
```

**Why Constructor Injection is Best:**
- Dependencies are **required** (no null values)
- Makes dependencies **immutable** (final fields)
- Easy to **test** (just pass mocks in constructor)
- **Clear contract** - you see all dependencies immediately

#### 2. Setter Injection

```java
@Service
public class NotificationService {

    private EmailService emailService;
    private SMSService smsService;

    // Setter Injection
    @Autowired
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }

    @Autowired(required = false)  // Optional dependency
    public void setSmsService(SMSService smsService) {
        this.smsService = smsService;
    }
}
```

**When to use:** For **optional** dependencies only.

#### 3. Field Injection (AVOID)

```java
@Service
public class ProductService {

    @Autowired  // Injected directly - no constructor or setter needed
    private ProductRepository productRepository;

    @Autowired
    private PriceCalculator priceCalculator;
}
```

**Why to Avoid:**
- Can't make fields `final`
- Hard to test without Spring context
- Hides dependencies (not visible in constructor)
- Possible to create objects with null dependencies

---

### 2.3 Spring Beans

#### What is a Bean?

**A Bean is simply an object that Spring creates, configures, and manages for you.**

```
┌─────────────────────────────────────────────────────────────────┐
│                        WHAT IS A BEAN?                           │
│                                                                   │
│   Regular Java Object  +  Managed by Spring  =  Spring Bean     │
│                                                                   │
│   ┌─────────────┐           ┌─────────────┐                      │
│   │   POJO      │  ──────▶  │   BEAN      │                      │
│   │  (Plain Old │           │  (Spring    │                      │
│   │ Java Object)│           │   Managed)  │                      │
│   └─────────────┘           └─────────────┘                      │
│                                                                   │
│   You create it             Spring creates, configures,          │
│                             injects dependencies, manages        │
│                             lifecycle, and destroys it           │
└─────────────────────────────────────────────────────────────────┘
```

#### How to Create Beans

**Method 1: Stereotype Annotations (Most Common)**

```java
@Component      // Generic bean
public class MyComponent { }

@Service        // Business logic layer
public class UserService { }

@Repository     // Data access layer (adds exception translation)
public class UserRepository { }

@Controller     // Web layer (MVC controller)
public class UserController { }

@RestController // Web layer (REST API) = @Controller + @ResponseBody
public class UserRestController { }
```

**Method 2: @Bean in Configuration Class**

```java
@Configuration
public class AppConfig {

    @Bean  // This method returns a bean managed by Spring
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }

    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
        return mapper;
    }
}
```

**When to use @Bean:**
- For third-party library classes (you can't add @Component to them)
- When you need custom configuration during bean creation

#### Bean Scopes

```
┌─────────────────────────────────────────────────────────────────┐
│                        BEAN SCOPES                               │
├──────────────┬──────────────────────────────────────────────────┤
│   SCOPE      │   DESCRIPTION                                    │
├──────────────┼──────────────────────────────────────────────────┤
│  singleton   │  (DEFAULT) One instance per Spring container     │
│              │  Same object returned every time                 │
├──────────────┼──────────────────────────────────────────────────┤
│  prototype   │  New instance created every time bean is         │
│              │  requested                                       │
├──────────────┼──────────────────────────────────────────────────┤
│  request     │  One instance per HTTP request (Web only)        │
├──────────────┼──────────────────────────────────────────────────┤
│  session     │  One instance per HTTP session (Web only)        │
├──────────────┼──────────────────────────────────────────────────┤
│  application │  One instance per ServletContext (Web only)      │
└──────────────┴──────────────────────────────────────────────────┘
```

**Code Example:**

```java
@Component
@Scope("singleton")  // Default - can be omitted
public class SingletonBean {
    // One instance shared across the application
}

@Component
@Scope("prototype")
public class PrototypeBean {
    // New instance created each time it's injected
}

// Usage
@Service
public class MyService {
    @Autowired private SingletonBean singleton1;
    @Autowired private SingletonBean singleton2;
    // singleton1 == singleton2 (same object)

    @Autowired private PrototypeBean prototype1;
    @Autowired private PrototypeBean prototype2;
    // prototype1 != prototype2 (different objects)
}
```

#### Bean Lifecycle

```
┌─────────────────────────────────────────────────────────────────┐
│                     BEAN LIFECYCLE                               │
│                                                                   │
│  ┌──────────────┐                                                │
│  │  Instantiate │  ← Spring creates the object                  │
│  └──────┬───────┘                                                │
│         ▼                                                        │
│  ┌──────────────┐                                                │
│  │   Populate   │  ← Inject dependencies                        │
│  │  Properties  │                                                │
│  └──────┬───────┘                                                │
│         ▼                                                        │
│  ┌──────────────┐                                                │
│  │ BeanNameAware│  ← Aware interfaces called                    │
│  │  setBeanName │                                                │
│  └──────┬───────┘                                                │
│         ▼                                                        │
│  ┌──────────────┐                                                │
│  │ @PostConstruct│ ← Initialization method called               │
│  │  init-method │                                                │
│  └──────┬───────┘                                                │
│         ▼                                                        │
│  ┌──────────────┐                                                │
│  │  Bean Ready  │  ← Bean is ready to use                       │
│  │   for Use    │                                                │
│  └──────┬───────┘                                                │
│         ▼                                                        │
│  ┌──────────────┐                                                │
│  │ @PreDestroy  │  ← Cleanup before destruction                 │
│  │destroy-method│                                                │
│  └──────┬───────┘                                                │
│         ▼                                                        │
│  ┌──────────────┐                                                │
│  │  Bean Gone   │                                                │
│  └──────────────┘                                                │
└─────────────────────────────────────────────────────────────────┘
```

**Lifecycle Hooks Example:**

```java
@Component
public class DatabaseConnectionPool {

    private Connection connection;

    // Called after all dependencies are injected
    @PostConstruct
    public void init() {
        System.out.println("Initializing database connection pool...");
        this.connection = createConnection();
    }

    // Called before bean is destroyed
    @PreDestroy
    public void cleanup() {
        System.out.println("Closing database connections...");
        this.connection.close();
    }

    private Connection createConnection() {
        // Create connection logic
        return null;
    }
}
```

---

### 2.4 ApplicationContext

#### What is ApplicationContext?

**ApplicationContext is the Spring IoC container** - it's the central interface that manages all your beans.

```
┌─────────────────────────────────────────────────────────────────┐
│                    APPLICATION CONTEXT                           │
│              (The Heart of Spring Application)                   │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                    What it does:                            │ │
│  │                                                             │ │
│  │  1. Creates Beans         - Instantiates objects           │ │
│  │  2. Configures Beans      - Sets properties                │ │
│  │  3. Assembles Beans       - Injects dependencies           │ │
│  │  4. Manages Lifecycle     - Init and destroy               │ │
│  │  5. Provides Beans        - Returns beans when requested   │ │
│  │  6. Publishes Events      - Application events             │ │
│  │  7. Resolves Messages     - i18n support                   │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

#### BeanFactory vs ApplicationContext

```
┌─────────────────────────────────────────────────────────────────┐
│        BeanFactory              vs        ApplicationContext     │
├─────────────────────────────────┬───────────────────────────────┤
│  - Basic IoC container          │  - Advanced IoC container     │
│  - Lazy loading (creates beans  │  - Eager loading (creates     │
│    when requested)              │    singleton beans at startup)│
│  - No event publishing          │  - Event publishing           │
│  - No internationalization      │  - i18n support               │
│  - Lightweight                  │  - More features, heavier     │
│                                 │                               │
│  Use when: Memory constrained   │  Use when: Normal apps        │
│           (rare)                │           (always)            │
└─────────────────────────────────┴───────────────────────────────┘
```

**ApplicationContext is a superset of BeanFactory. Always use ApplicationContext.**

---

## 3. Annotations - The Building Blocks

### Annotation Hierarchy

```
┌─────────────────────────────────────────────────────────────────┐
│                    SPRING ANNOTATIONS HIERARCHY                  │
│                                                                   │
│                        @Component                                │
│                            │                                     │
│            ┌───────────────┼───────────────┐                     │
│            │               │               │                     │
│            ▼               ▼               ▼                     │
│       @Service       @Repository     @Controller                 │
│    (Business Layer) (Data Layer)   (Web Layer)                   │
│                                          │                       │
│                                          ▼                       │
│                                   @RestController                │
│                              (@Controller + @ResponseBody)       │
└─────────────────────────────────────────────────────────────────┘
```

### Core Annotations Explained

#### @SpringBootApplication

```java
@SpringBootApplication  // This is a combo annotation!
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
    }
}

// @SpringBootApplication =
//     @Configuration           → This class can contain @Bean methods
//   + @EnableAutoConfiguration → Enable Spring Boot auto-config
//   + @ComponentScan           → Scan this package and sub-packages for components
```

#### @Component vs @Service vs @Repository vs @Controller

```java
// @Component - Generic stereotype for any Spring-managed component
@Component
public class EmailValidator {
    public boolean isValid(String email) {
        return email.contains("@");
    }
}

// @Service - Business logic layer (semantic clarity, no extra behavior)
@Service
public class UserService {
    public User createUser(UserDTO dto) {
        // Business logic here
    }
}

// @Repository - Data access layer (adds exception translation)
@Repository
public class UserRepository {
    // JPA/JDBC code here
    // SQLException → Spring's DataAccessException (unchecked)
}

// @Controller - Web MVC controller (returns view names)
@Controller
public class HomeController {
    @GetMapping("/")
    public String home(Model model) {
        model.addAttribute("message", "Hello");
        return "home";  // Returns view name
    }
}

// @RestController - REST API controller (returns data directly)
@RestController  // = @Controller + @ResponseBody
public class UserApiController {
    @GetMapping("/api/users")
    public List<User> getUsers() {
        return userService.getAllUsers();  // Returns JSON automatically
    }
}
```

#### @Autowired

```java
@Service
public class OrderService {

    // Field injection (not recommended but common)
    @Autowired
    private ProductService productService;

    // Constructor injection (recommended - @Autowired optional since Spring 4.3)
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }

    // Setter injection
    private NotificationService notificationService;

    @Autowired
    public void setNotificationService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }

    // Optional dependency
    @Autowired(required = false)
    private AnalyticsService analyticsService;  // May be null
}
```

#### @Qualifier and @Primary

**Problem:** What if you have multiple beans of the same type?

```java
// Two implementations of the same interface
public interface NotificationService {
    void notify(String message);
}

@Service
public class EmailNotificationService implements NotificationService {
    public void notify(String message) {
        // Send email
    }
}

@Service
public class SMSNotificationService implements NotificationService {
    public void notify(String message) {
        // Send SMS
    }
}

// PROBLEM: Which one to inject?
@Service
public class OrderService {
    @Autowired
    private NotificationService notificationService;  // ERROR: Two beans found!
}
```

**Solution 1: @Primary**

```java
@Service
@Primary  // This will be the default choice
public class EmailNotificationService implements NotificationService {
    // ...
}

@Service
public class SMSNotificationService implements NotificationService {
    // ...
}

@Service
public class OrderService {
    @Autowired
    private NotificationService notificationService;  // Gets EmailNotificationService
}
```

**Solution 2: @Qualifier**

```java
@Service("emailNotification")  // Give beans names
public class EmailNotificationService implements NotificationService {
    // ...
}

@Service("smsNotification")
public class SMSNotificationService implements NotificationService {
    // ...
}

@Service
public class OrderService {
    @Autowired
    @Qualifier("smsNotification")  // Specify which one you want
    private NotificationService notificationService;  // Gets SMSNotificationService
}
```

#### @Configuration and @Bean

```java
@Configuration  // Indicates this class contains bean definitions
public class AppConfig {

    @Bean  // Method return value is registered as a bean
    public RestTemplate restTemplate() {
        RestTemplate restTemplate = new RestTemplate();
        // Custom configuration
        restTemplate.setRequestFactory(new HttpComponentsClientHttpRequestFactory());
        return restTemplate;
    }

    @Bean
    @Scope("prototype")  // New instance each time
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }

    @Bean
    @Profile("production")  // Only create this bean in production profile
    public DataSource productionDataSource() {
        // Production database config
    }

    @Bean
    @Profile("development")
    public DataSource developmentDataSource() {
        // Development database config (H2 in-memory)
    }
}
```

#### @Value

```java
@Service
public class ExternalApiService {

    // Inject from application.properties
    @Value("${api.base-url}")
    private String apiBaseUrl;

    @Value("${api.timeout:5000}")  // Default value if not found
    private int timeout;

    @Value("${api.enabled:true}")
    private boolean enabled;

    // SpEL (Spring Expression Language)
    @Value("#{systemProperties['user.home']}")
    private String userHome;

    @Value("${api.allowed-origins}")  // Comma-separated to List
    private List<String> allowedOrigins;
}
```

**application.properties:**
```properties
api.base-url=https://api.example.com
api.timeout=10000
api.enabled=true
api.allowed-origins=http://localhost:3000,https://myapp.com
```

---

## 4. Spring Boot Auto-Configuration

### How Auto-Configuration Works

```
┌─────────────────────────────────────────────────────────────────┐
│                  AUTO-CONFIGURATION FLOW                         │
│                                                                   │
│  1. Spring Boot starts                                           │
│         ↓                                                        │
│  2. Scans classpath for dependencies                             │
│         ↓                                                        │
│  3. Reads META-INF/spring/org.springframework.boot.             │
│     autoconfigure.AutoConfiguration.imports                      │
│         ↓                                                        │
│  4. Evaluates @Conditional annotations                           │
│         ↓                                                        │
│  5. Creates beans if conditions are met                          │
│                                                                   │
│  Example:                                                        │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │ spring-boot-starter-data-jpa in classpath?                  ││
│  │         ↓ YES                                               ││
│  │ DataSource bean already defined?                            ││
│  │         ↓ NO                                                ││
│  │ H2/MySQL/PostgreSQL driver present?                         ││
│  │         ↓ YES                                               ││
│  │ → Auto-configure DataSource, EntityManagerFactory,          ││
│  │   TransactionManager, etc.                                  ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
```

### Key @Conditional Annotations

```java
// Bean created only if specific class is on classpath
@ConditionalOnClass(DataSource.class)

// Bean created only if specific class is NOT on classpath
@ConditionalOnMissingClass("com.example.SomeClass")

// Bean created only if another bean of this type doesn't exist
@ConditionalOnMissingBean(DataSource.class)

// Bean created only if another bean of this type exists
@ConditionalOnBean(DataSource.class)

// Bean created only if property has specific value
@ConditionalOnProperty(name = "feature.enabled", havingValue = "true")

// Bean created only in web application context
@ConditionalOnWebApplication
```

### Overriding Auto-Configuration

```java
@Configuration
public class CustomDataSourceConfig {

    // This bean takes precedence over auto-configured DataSource
    @Bean
    @Primary
    public DataSource customDataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl("jdbc:mysql://localhost:3306/mydb");
        dataSource.setUsername("root");
        dataSource.setPassword("password");
        dataSource.setMaximumPoolSize(20);  // Custom pool size
        return dataSource;
    }
}
```

### Starter Dependencies

```
┌─────────────────────────────────────────────────────────────────┐
│                    STARTER DEPENDENCIES                          │
├─────────────────────────────┬───────────────────────────────────┤
│  STARTER                    │  WHAT IT INCLUDES                 │
├─────────────────────────────┼───────────────────────────────────┤
│ spring-boot-starter-web     │ Spring MVC, Tomcat, Jackson,     │
│                             │ Validation                        │
├─────────────────────────────┼───────────────────────────────────┤
│ spring-boot-starter-data-   │ Spring Data JPA, Hibernate,      │
│ jpa                         │ HikariCP, Spring TX               │
├─────────────────────────────┼───────────────────────────────────┤
│ spring-boot-starter-        │ Spring Security, security         │
│ security                    │ auto-configuration                │
├─────────────────────────────┼───────────────────────────────────┤
│ spring-boot-starter-test    │ JUnit 5, Mockito, AssertJ,       │
│                             │ Spring Test                       │
├─────────────────────────────┼───────────────────────────────────┤
│ spring-boot-starter-        │ Health checks, metrics, info     │
│ actuator                    │ endpoints                         │
└─────────────────────────────┴───────────────────────────────────┘
```

---

## 5. REST API Development

### REST Controller Basics

```java
@RestController
@RequestMapping("/api/users")  // Base path for all endpoints
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    // GET /api/users
    @GetMapping
    public List<UserDTO> getAllUsers() {
        return userService.getAllUsers();
    }

    // GET /api/users/123
    @GetMapping("/{id}")
    public UserDTO getUserById(@PathVariable Long id) {
        return userService.getUserById(id);
    }

    // GET /api/users?email=john@example.com
    @GetMapping("/search")
    public UserDTO getUserByEmail(@RequestParam String email) {
        return userService.getUserByEmail(email);
    }

    // POST /api/users (with JSON body)
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)  // Return 201 instead of 200
    public UserDTO createUser(@RequestBody @Valid CreateUserRequest request) {
        return userService.createUser(request);
    }

    // PUT /api/users/123
    @PutMapping("/{id}")
    public UserDTO updateUser(@PathVariable Long id,
                              @RequestBody @Valid UpdateUserRequest request) {
        return userService.updateUser(id, request);
    }

    // DELETE /api/users/123
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)  // Return 204
    public void deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
    }
}
```

### Request/Response Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                    HTTP REQUEST FLOW                             │
│                                                                   │
│  Client                                                          │
│    │                                                             │
│    │  POST /api/users                                            │
│    │  Content-Type: application/json                             │
│    │  {"name": "John", "email": "john@example.com"}              │
│    ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                   DispatcherServlet                          ││
│  │              (Front Controller Pattern)                      ││
│  └─────────────────────────────────────────────────────────────┘│
│    │                                                             │
│    ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │               Handler Mapping                                ││
│  │    (Maps URL to Controller method)                           ││
│  └─────────────────────────────────────────────────────────────┘│
│    │                                                             │
│    ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │           @RequestBody Processing                            ││
│  │  (Jackson: JSON → CreateUserRequest object)                  ││
│  └─────────────────────────────────────────────────────────────┘│
│    │                                                             │
│    ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              @Valid Processing                               ││
│  │  (Hibernate Validator validates the request)                 ││
│  └─────────────────────────────────────────────────────────────┘│
│    │                                                             │
│    ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              Controller Method                               ││
│  │    userController.createUser(request)                        ││
│  └─────────────────────────────────────────────────────────────┘│
│    │                                                             │
│    ▼                                                             │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │           @ResponseBody Processing                           ││
│  │  (Jackson: UserDTO object → JSON)                            ││
│  └─────────────────────────────────────────────────────────────┘│
│    │                                                             │
│    ▼                                                             │
│  Client receives: HTTP 201                                       │
│  {"id": 1, "name": "John", "email": "john@example.com"}         │
└─────────────────────────────────────────────────────────────────┘
```

### @PathVariable vs @RequestParam vs @RequestBody

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    // @PathVariable - Part of the URL path
    // GET /api/products/123
    @GetMapping("/{id}")
    public Product getProduct(@PathVariable Long id) {
        return productService.findById(id);
    }

    // GET /api/products/category/electronics/brand/apple
    @GetMapping("/category/{category}/brand/{brand}")
    public List<Product> getProducts(@PathVariable String category,
                                     @PathVariable String brand) {
        return productService.findByCategoryAndBrand(category, brand);
    }

    // @RequestParam - Query parameters (after ?)
    // GET /api/products?category=electronics&minPrice=100&maxPrice=500
    @GetMapping
    public List<Product> searchProducts(
            @RequestParam(required = false) String category,
            @RequestParam(defaultValue = "0") Double minPrice,
            @RequestParam(defaultValue = "999999") Double maxPrice,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        return productService.search(category, minPrice, maxPrice, page, size);
    }

    // @RequestBody - JSON body of the request
    // POST /api/products
    // Body: {"name": "iPhone", "price": 999.99, "category": "electronics"}
    @PostMapping
    public Product createProduct(@RequestBody ProductDTO product) {
        return productService.create(product);
    }
}
```

### ResponseEntity for Full Control

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public ResponseEntity<UserDTO> getUserById(@PathVariable Long id) {
        return userService.findById(id)
            .map(user -> ResponseEntity.ok(user))  // 200 OK with body
            .orElse(ResponseEntity.notFound().build());  // 404 Not Found
    }

    @PostMapping
    public ResponseEntity<UserDTO> createUser(@RequestBody @Valid CreateUserRequest request) {
        UserDTO createdUser = userService.createUser(request);

        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(createdUser.getId())
            .toUri();

        return ResponseEntity
            .created(location)  // 201 Created
            .header("X-Custom-Header", "value")  // Custom header
            .body(createdUser);
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        if (!userService.exists(id)) {
            return ResponseEntity.notFound().build();  // 404
        }
        userService.delete(id);
        return ResponseEntity.noContent().build();  // 204 No Content
    }
}
```

---

## 6. Exception Handling

### Global Exception Handler

```java
@RestControllerAdvice  // = @ControllerAdvice + @ResponseBody
public class GlobalExceptionHandler {

    // Handle specific exception
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleResourceNotFound(ResourceNotFoundException ex) {
        return new ErrorResponse(
            HttpStatus.NOT_FOUND.value(),
            ex.getMessage(),
            LocalDateTime.now()
        );
    }

    // Handle validation errors
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidationErrors(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(error -> error.getField() + ": " + error.getDefaultMessage())
            .collect(Collectors.toList());

        return new ErrorResponse(
            HttpStatus.BAD_REQUEST.value(),
            "Validation failed",
            errors,
            LocalDateTime.now()
        );
    }

    // Handle all other exceptions
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleAllExceptions(Exception ex) {
        return new ErrorResponse(
            HttpStatus.INTERNAL_SERVER_ERROR.value(),
            "An unexpected error occurred",
            LocalDateTime.now()
        );
    }
}

// Custom exception
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }

    public ResourceNotFoundException(String resourceName, Long id) {
        super(String.format("%s not found with id: %d", resourceName, id));
    }
}

// Error response DTO
public class ErrorResponse {
    private int status;
    private String message;
    private List<String> errors;
    private LocalDateTime timestamp;

    // Constructors, getters, setters
}
```

### Exception Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                   EXCEPTION HANDLING FLOW                        │
│                                                                   │
│  Controller throws exception                                     │
│         ↓                                                        │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │              @RestControllerAdvice                           ││
│  │                                                              ││
│  │  Is there @ExceptionHandler for this exact exception type?  ││
│  │         ↓ YES                     ↓ NO                      ││
│  │    Handle it                Check parent exception types     ││
│  │                                   ↓                          ││
│  │                            Found handler?                    ││
│  │                          YES ↓         ↓ NO                  ││
│  │                       Handle it    Default error handling    ││
│  └─────────────────────────────────────────────────────────────┘│
│         ↓                                                        │
│  Return ErrorResponse as JSON                                    │
└─────────────────────────────────────────────────────────────────┘
```

---

## 7. Validation

### Bean Validation Annotations

```java
public class CreateUserRequest {

    @NotNull(message = "Name is required")
    @Size(min = 2, max = 100, message = "Name must be between 2 and 100 characters")
    private String name;

    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;

    @NotNull(message = "Age is required")
    @Min(value = 18, message = "Age must be at least 18")
    @Max(value = 120, message = "Age must be at most 120")
    private Integer age;

    @Pattern(regexp = "^\\+?[1-9]\\d{1,14}$", message = "Invalid phone number")
    private String phone;

    @NotEmpty(message = "At least one role is required")
    private List<String> roles;

    @Valid  // Validate nested object
    @NotNull(message = "Address is required")
    private AddressDTO address;

    // Getters and setters
}

public class AddressDTO {
    @NotBlank(message = "Street is required")
    private String street;

    @NotBlank(message = "City is required")
    private String city;

    @Pattern(regexp = "^\\d{5}(-\\d{4})?$", message = "Invalid ZIP code")
    private String zipCode;
}
```

### Using Validation in Controller

```java
@RestController
@RequestMapping("/api/users")
@Validated  // Enable validation for path variables and request params
public class UserController {

    // @Valid triggers validation on request body
    @PostMapping
    public UserDTO createUser(@RequestBody @Valid CreateUserRequest request) {
        return userService.createUser(request);
    }

    // Validate path variable
    @GetMapping("/{id}")
    public UserDTO getUser(@PathVariable @Min(1) Long id) {
        return userService.findById(id);
    }

    // Validate request param
    @GetMapping("/search")
    public List<UserDTO> searchUsers(
            @RequestParam @Size(min = 2) String query) {
        return userService.search(query);
    }
}
```

### Custom Validator

```java
// Custom annotation
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already exists";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Validator implementation
@Component
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {

    private final UserRepository userRepository;

    public UniqueEmailValidator(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public boolean isValid(String email, ConstraintValidatorContext context) {
        if (email == null) {
            return true;  // Let @NotNull handle null validation
        }
        return !userRepository.existsByEmail(email);
    }
}

// Usage
public class CreateUserRequest {
    @UniqueEmail
    @Email
    private String email;
}
```

---

## 8. Spring Data JPA

### Repository Pattern

```
┌─────────────────────────────────────────────────────────────────┐
│                  SPRING DATA JPA HIERARCHY                       │
│                                                                   │
│                       Repository<T, ID>                          │
│                            │                                     │
│                            ▼                                     │
│                    CrudRepository<T, ID>                         │
│                 (save, findById, findAll, delete)                │
│                            │                                     │
│                            ▼                                     │
│              PagingAndSortingRepository<T, ID>                   │
│                 (findAll with Pageable/Sort)                     │
│                            │                                     │
│                            ▼                                     │
│                  JpaRepository<T, ID>                            │
│           (flush, saveAndFlush, deleteInBatch)                   │
└─────────────────────────────────────────────────────────────────┘
```

### Entity Definition

```java
@Entity
@Table(name = "users")
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(unique = true, nullable = false)
    private String email;

    @Enumerated(EnumType.STRING)
    private UserStatus status;

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;

    @LastModifiedDate
    private LocalDateTime updatedAt;

    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Order> orders = new ArrayList<>();

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")
    private Department department;

    // Getters, setters, constructors
}
```

### Repository with Query Methods

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    // Spring Data creates implementation automatically based on method name!

    // SELECT * FROM users WHERE email = ?
    Optional<User> findByEmail(String email);

    // SELECT * FROM users WHERE name LIKE ?
    List<User> findByNameContaining(String name);

    // SELECT * FROM users WHERE status = ? ORDER BY created_at DESC
    List<User> findByStatusOrderByCreatedAtDesc(UserStatus status);

    // SELECT * FROM users WHERE age > ? AND status = ?
    List<User> findByAgeGreaterThanAndStatus(int age, UserStatus status);

    // SELECT * FROM users WHERE name IN (?)
    List<User> findByNameIn(List<String> names);

    // EXISTS query
    boolean existsByEmail(String email);

    // COUNT query
    long countByStatus(UserStatus status);

    // DELETE query
    void deleteByStatus(UserStatus status);

    // Custom JPQL query
    @Query("SELECT u FROM User u WHERE u.email LIKE %:domain")
    List<User> findByEmailDomain(@Param("domain") String domain);

    // Native SQL query
    @Query(value = "SELECT * FROM users WHERE created_at > :date",
           nativeQuery = true)
    List<User> findRecentUsers(@Param("date") LocalDateTime date);

    // Modifying query (UPDATE/DELETE)
    @Modifying
    @Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
    int updateStatus(@Param("id") Long id, @Param("status") UserStatus status);
}
```

### Service Layer with Transactions

```java
@Service
@Transactional(readOnly = true)  // Default for all methods
public class UserService {

    private final UserRepository userRepository;
    private final EmailService emailService;

    public UserService(UserRepository userRepository, EmailService emailService) {
        this.userRepository = userRepository;
        this.emailService = emailService;
    }

    public List<User> getAllUsers() {
        return userRepository.findAll();  // Read-only transaction
    }

    public Optional<User> findById(Long id) {
        return userRepository.findById(id);
    }

    @Transactional  // Override to read-write transaction
    public User createUser(CreateUserRequest request) {
        User user = new User();
        user.setName(request.getName());
        user.setEmail(request.getEmail());
        user.setStatus(UserStatus.ACTIVE);

        User savedUser = userRepository.save(user);
        emailService.sendWelcomeEmail(savedUser);  // Part of same transaction

        return savedUser;
    }

    @Transactional
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }
}
```

---

## 9. Spring Security Basics

### Security Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                 SPRING SECURITY FILTER CHAIN                     │
│                                                                   │
│  HTTP Request                                                    │
│       ↓                                                          │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  SecurityContextPersistenceFilter                           ││
│  │  (Load SecurityContext from session)                         ││
│  └─────────────────────────────────────────────────────────────┘│
│       ↓                                                          │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  UsernamePasswordAuthenticationFilter                        ││
│  │  (Process login form submission)                             ││
│  └─────────────────────────────────────────────────────────────┘│
│       ↓                                                          │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  BasicAuthenticationFilter                                   ││
│  │  (Process HTTP Basic authentication)                         ││
│  └────────────────────────────────────��────────────────────────┘│
│       ↓                                                          │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  BearerTokenAuthenticationFilter                             ││
│  │  (Process JWT/OAuth2 tokens)                                 ││
│  └─────────────────────────────────────────────────────────────┘│
│       ↓                                                          │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  ExceptionTranslationFilter                                  ││
│  │  (Handle security exceptions)                                ││
│  └─────────────────────────────────────────────────────────────┘│
│       ↓                                                          │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │  FilterSecurityInterceptor                                   ││
│  │  (Authorization check - does user have access?)              ││
│  └─────────────────────────────────────────────────────────────┘│
│       ↓                                                          │
│  Controller                                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Basic Security Configuration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            // Disable CSRF for REST API (use JWT instead)
            .csrf(csrf -> csrf.disable())

            // Session management
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            // Authorization rules
            .authorizeHttpRequests(auth -> auth
                // Public endpoints
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()

                // Role-based access
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/users/**").hasAnyRole("USER", "ADMIN")

                // All other requests need authentication
                .anyRequest().authenticated()
            )

            // Add JWT filter
            .addFilterBefore(jwtAuthenticationFilter(),
                           UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

### Method-Level Security

```java
@Service
public class UserService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long id) {
        // Only ADMIN can delete
    }

    @PreAuthorize("hasRole('ADMIN') or #userId == authentication.principal.id")
    public User getUser(Long userId) {
        // ADMIN or the user themselves can access
    }

    @PostAuthorize("returnObject.email == authentication.principal.email")
    public User getCurrentUser() {
        // Check access after method execution
    }

    @PreAuthorize("@securityService.canAccess(#projectId)")
    public Project getProject(Long projectId) {
        // Delegate to custom security service
    }
}
```

---

## 10. Testing in Spring Boot

### Test Types

```
┌─────────────────────────────────────────────────────────────────┐
│                   TESTING PYRAMID                                │
│                                                                   │
│                        /\                                        │
│                       /  \                                       │
│                      / E2E\                                      │
│                     /------\                                     │
│                    /        \                                    │
│                   / Integra- \                                   │
│                  /   tion     \                                  │
│                 /--------------\                                 │
│                /                \                                │
│               /     Unit Tests   \                               │
│              /____________________\                              │
│                                                                   │
│  More tests at the bottom, fewer at the top                      │
│  Unit tests are fast and isolated                                │
│  Integration tests verify components work together               │
│  E2E tests verify entire user flows                              │
└─────────────────────────────────────────────────────────────────┘
```

### Unit Test (No Spring Context)

```java
class UserServiceTest {

    @Mock
    private UserRepository userRepository;

    @Mock
    private EmailService emailService;

    @InjectMocks
    private UserService userService;

    @BeforeEach
    void setUp() {
        MockitoAnnotations.openMocks(this);
    }

    @Test
    void createUser_ShouldSaveAndSendEmail() {
        // Given
        CreateUserRequest request = new CreateUserRequest("John", "john@example.com");
        User savedUser = new User(1L, "John", "john@example.com");

        when(userRepository.save(any(User.class))).thenReturn(savedUser);

        // When
        User result = userService.createUser(request);

        // Then
        assertThat(result.getId()).isEqualTo(1L);
        assertThat(result.getName()).isEqualTo("John");

        verify(userRepository).save(any(User.class));
        verify(emailService).sendWelcomeEmail(savedUser);
    }

    @Test
    void findById_WhenUserNotFound_ShouldThrowException() {
        // Given
        when(userRepository.findById(1L)).thenReturn(Optional.empty());

        // When/Then
        assertThrows(ResourceNotFoundException.class,
                    () -> userService.findById(1L));
    }
}
```

### Integration Test (@SpringBootTest)

```java
@SpringBootTest
@AutoConfigureMockMvc
@Transactional  // Rollback after each test
class UserControllerIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private ObjectMapper objectMapper;

    @Autowired
    private UserRepository userRepository;

    @Test
    void createUser_ShouldReturnCreatedUser() throws Exception {
        // Given
        CreateUserRequest request = new CreateUserRequest("John", "john@example.com");

        // When/Then
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(request)))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.name").value("John"))
            .andExpect(jsonPath("$.email").value("john@example.com"))
            .andExpect(jsonPath("$.id").isNumber());

        // Verify in database
        assertThat(userRepository.findByEmail("john@example.com")).isPresent();
    }

    @Test
    void getUser_WhenNotFound_ShouldReturn404() throws Exception {
        mockMvc.perform(get("/api/users/999"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.message").value("User not found with id: 999"));
    }
}
```

### Controller Slice Test (@WebMvcTest)

```java
@WebMvcTest(UserController.class)  // Only loads web layer
class UserControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @MockBean  // Mock the service
    private UserService userService;

    @Test
    void getAllUsers_ShouldReturnUserList() throws Exception {
        // Given
        List<UserDTO> users = Arrays.asList(
            new UserDTO(1L, "John", "john@example.com"),
            new UserDTO(2L, "Jane", "jane@example.com")
        );
        when(userService.getAllUsers()).thenReturn(users);

        // When/Then
        mockMvc.perform(get("/api/users"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.length()").value(2))
            .andExpect(jsonPath("$[0].name").value("John"));
    }
}
```

### Repository Test (@DataJpaTest)

```java
@DataJpaTest  // Only loads JPA components
class UserRepositoryTest {

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private TestEntityManager entityManager;

    @Test
    void findByEmail_ShouldReturnUser() {
        // Given
        User user = new User();
        user.setName("John");
        user.setEmail("john@example.com");
        entityManager.persistAndFlush(user);

        // When
        Optional<User> found = userRepository.findByEmail("john@example.com");

        // Then
        assertThat(found).isPresent();
        assertThat(found.get().getName()).isEqualTo("John");
    }

    @Test
    void findByEmail_WhenNotExists_ShouldReturnEmpty() {
        Optional<User> found = userRepository.findByEmail("notfound@example.com");
        assertThat(found).isEmpty();
    }
}
```

---

## 11. Configuration Management

### application.properties vs application.yml

```
┌─────────────────────────────────────────────────────────────────┐
│           application.properties                                 │
├─────────────────────────────────────────────────────────────────┤
│  server.port=8080                                                │
│  spring.datasource.url=jdbc:mysql://localhost:3306/mydb         │
│  spring.datasource.username=root                                 │
│  spring.datasource.password=secret                               │
│  spring.jpa.hibernate.ddl-auto=update                            │
│  spring.jpa.show-sql=true                                        │
│  logging.level.org.springframework=INFO                          │
│  logging.level.com.myapp=DEBUG                                   │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│           application.yml (Same config, YAML format)            │
├─────────────────────────────────────────────────────────────────┤
│  server:                                                         │
│    port: 8080                                                    │
│                                                                  │
│  spring:                                                         │
│    datasource:                                                   │
│      url: jdbc:mysql://localhost:3306/mydb                      │
│      username: root                                              │
│      password: secret                                            │
│    jpa:                                                          │
│      hibernate:                                                  │
│        ddl-auto: update                                          │
│      show-sql: true                                              │
│                                                                  │
│  logging:                                                        │
│    level:                                                        │
│      org.springframework: INFO                                   │
│      com.myapp: DEBUG                                            │
└─────────────────────────────────────────────────────────────────┘
```

### Profiles

```
┌─────────────────────────────────────────────────────────────────┐
│                   SPRING PROFILES                                │
│                                                                   │
│  application.yml          (Common config - always loaded)        │
│  application-dev.yml      (Development profile)                  │
│  application-prod.yml     (Production profile)                   │
│  application-test.yml     (Testing profile)                      │
│                                                                   │
│  Active profile: spring.profiles.active=dev                      │
│  Or: java -jar app.jar --spring.profiles.active=prod             │
│  Or: SPRING_PROFILES_ACTIVE=prod                                 │
└─────────────────────────────────────────────────────────────────┘
```

**application.yml (common):**
```yaml
spring:
  application:
    name: my-app

app:
  name: MyApplication
```

**application-dev.yml:**
```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:h2:mem:devdb
    driver-class-name: org.h2.Driver
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: create-drop

logging:
  level:
    root: DEBUG
```

**application-prod.yml:**
```yaml
server:
  port: 80

spring:
  datasource:
    url: jdbc:mysql://prod-server:3306/proddb
    username: ${DB_USERNAME}  # From environment variable
    password: ${DB_PASSWORD}
  jpa:
    show-sql: false
    hibernate:
      ddl-auto: validate  # Never auto-modify in production!

logging:
  level:
    root: WARN
```

### @ConfigurationProperties (Type-Safe Configuration)

```java
// Configuration properties class
@ConfigurationProperties(prefix = "app.api")
@Validated
public class ApiProperties {

    @NotBlank
    private String baseUrl;

    @Min(1000)
    private int timeout = 5000;

    private RetryConfig retry = new RetryConfig();

    // Nested configuration
    public static class RetryConfig {
        private int maxAttempts = 3;
        private long delay = 1000;

        // Getters and setters
    }

    // Getters and setters
}

// Enable it in main class or config
@SpringBootApplication
@EnableConfigurationProperties(ApiProperties.class)
public class MyApplication { }

// Usage
@Service
public class ExternalApiService {

    private final ApiProperties apiProperties;

    public ExternalApiService(ApiProperties apiProperties) {
        this.apiProperties = apiProperties;
    }

    public void callApi() {
        String url = apiProperties.getBaseUrl();
        int timeout = apiProperties.getTimeout();
        // Use configuration...
    }
}
```

**application.yml:**
```yaml
app:
  api:
    base-url: https://api.example.com
    timeout: 10000
    retry:
      max-attempts: 5
      delay: 2000
```

---

## 12. Common Interview Q&A with Follow-ups

### Q1: What is Spring Boot and why use it?

**Answer:**
> "Spring Boot is an opinionated framework built on top of Spring that simplifies application development. It provides auto-configuration, starter dependencies, embedded servers, and production-ready features. This eliminates boilerplate XML configuration and lets developers focus on business logic."

**Follow-up Q: How does auto-configuration work?**
> "Spring Boot scans the classpath for libraries and automatically configures beans based on what's present. It uses @Conditional annotations to create beans only when certain conditions are met. For example, if H2 is on the classpath and no DataSource is defined, it auto-configures an H2 DataSource."

**Follow-up Q: Can you override auto-configuration?**
> "Yes, you can override by defining your own beans. Spring Boot's auto-configuration uses @ConditionalOnMissingBean, so your beans take precedence. You can also exclude specific auto-configurations using @SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})."

---

### Q2: Explain IoC and DI

**Answer:**
> "IoC (Inversion of Control) is a design principle where the control of object creation and lifecycle is transferred from the application code to a framework. DI (Dependency Injection) is the implementation pattern - the framework injects dependencies into objects rather than objects creating their own dependencies."

**Follow-up Q: What are the types of DI?**
> "There are three types: Constructor injection (recommended - dependencies are required and immutable), Setter injection (for optional dependencies), and Field injection (not recommended - hides dependencies and harder to test)."

**Follow-up Q: Why is constructor injection preferred?**
> "Constructor injection ensures dependencies are required (no nulls), allows fields to be final (immutable), makes testing easier (just pass mocks), and makes dependencies visible in the constructor signature."

---

### Q3: What are Spring Beans?

**Answer:**
> "A Bean is an object that is instantiated, assembled, and managed by the Spring IoC container. Beans are defined using annotations like @Component, @Service, @Repository, or through @Bean methods in @Configuration classes."

**Follow-up Q: What's the default scope of a bean?**
> "Singleton - one instance per Spring container. Other scopes include prototype (new instance each time), request, session, and application (for web apps)."

**Follow-up Q: What happens if you inject a prototype bean into a singleton?**
> "The prototype bean is created once when the singleton is initialized, defeating the purpose of prototype scope. To fix this, you can use @Lookup annotation, ObjectFactory<T>, Provider<T>, or make the singleton scope prototype as well."

---

### Q4: Explain @Autowired

**Answer:**
> "@Autowired tells Spring to inject a dependency automatically. Spring looks for a bean of the matching type in the container and injects it. Since Spring 4.3, @Autowired is optional for single-constructor classes."

**Follow-up Q: What if there are multiple beans of the same type?**
> "Spring will throw NoUniqueBeanDefinitionException. Solutions: use @Primary on one bean to make it default, use @Qualifier to specify which bean to inject, or use constructor injection with the bean name matching the parameter name."

**Follow-up Q: What's the difference between @Autowired and @Inject?**
> "@Autowired is Spring-specific and has a 'required' attribute. @Inject is from JSR-330 (javax.inject) and is vendor-neutral. Functionally they're the same, but @Autowired is more commonly used in Spring projects."

---

### Q5: What is @SpringBootApplication?

**Answer:**
> "@SpringBootApplication is a convenience annotation that combines three annotations:
> - @Configuration: Indicates the class can contain @Bean definitions
> - @EnableAutoConfiguration: Enables Spring Boot's auto-configuration
> - @ComponentScan: Enables scanning for components in the current package and sub-packages"

**Follow-up Q: What if my components are in a different package?**
> "Use @ComponentScan with explicit base packages: @ComponentScan(basePackages = {\"com.myapp\", \"com.external\"}) or move @SpringBootApplication to a common parent package."

---

### Q6: Difference between @Controller and @RestController

**Answer:**
> "@Controller is used for MVC controllers that return view names, typically for server-side rendering. @RestController is a convenience annotation that combines @Controller and @ResponseBody, meaning every method returns data directly (JSON/XML) instead of a view name."

**Follow-up Q: Can you return JSON from @Controller?**
> "Yes, by adding @ResponseBody to individual methods or to the class. @RestController just saves you from adding @ResponseBody to every method."

---

### Q7: How do you handle exceptions in Spring Boot?

**Answer:**
> "Use @RestControllerAdvice (or @ControllerAdvice) with @ExceptionHandler methods. This creates a global exception handler that catches exceptions across all controllers and returns appropriate error responses."

**Follow-up Q: How do you handle validation errors specifically?**
> "Validation errors throw MethodArgumentNotValidException. You can catch this with @ExceptionHandler and extract field errors from BindingResult to return a detailed error response with all validation failures."

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
    List<String> errors = ex.getBindingResult()
        .getFieldErrors()
        .stream()
        .map(e -> e.getField() + ": " + e.getDefaultMessage())
        .toList();
    return ResponseEntity.badRequest().body(new ErrorResponse(errors));
}
```

---

### Q8: Explain @Transactional

**Answer:**
> "@Transactional defines the scope of a single database transaction. When a method is marked @Transactional, Spring creates a proxy that opens a transaction before the method executes and commits or rolls back based on the outcome."

**Follow-up Q: What's the default rollback behavior?**
> "By default, transactions roll back for unchecked exceptions (RuntimeException) and errors, but commit for checked exceptions. You can customize with rollbackFor and noRollbackFor attributes."

**Follow-up Q: What's @Transactional(readOnly = true)?**
> "It's a hint to the underlying database that no writes will occur. This enables optimizations like skipping dirty checks in Hibernate and potentially routing to read replicas. Use it for read-only methods."

**Follow-up Q: Why might @Transactional not work?**
> "Common issues: calling a @Transactional method from within the same class (proxy not invoked), the method is private (proxies can't intercept), or the method is not in a Spring-managed bean. Transactions work through proxies, so internal calls bypass the proxy."

---

### Q9: How does Spring Security work?

**Answer:**
> "Spring Security uses a filter chain that intercepts HTTP requests before they reach controllers. Key filters include authentication filters (verify identity) and authorization filters (verify permissions). Security configuration is done through SecurityFilterChain bean."

**Follow-up Q: How do you implement JWT authentication?**
> "Create a custom filter that extracts the JWT from the Authorization header, validates it, and sets the Authentication object in SecurityContext. Add this filter before UsernamePasswordAuthenticationFilter in the chain."

---

### Q10: @PathVariable vs @RequestParam vs @RequestBody

**Answer:**
> "- @PathVariable: Extracts values from the URL path (`/users/{id}`)
> - @RequestParam: Extracts query parameters (`/users?name=John`)
> - @RequestBody: Deserializes the request body (typically JSON) to an object"

**Follow-up Q: Can @RequestParam have default values?**
> "Yes, use defaultValue attribute: @RequestParam(defaultValue = \"10\") int size. You can also make it optional with required = false."

---

### Q11: Difference between JpaRepository and CrudRepository

**Answer:**
> "CrudRepository provides basic CRUD operations (save, findById, findAll, delete). JpaRepository extends it and adds JPA-specific features like flush(), saveAndFlush(), deleteInBatch(), and returns List instead of Iterable for findAll()."

**Follow-up Q: How do you write custom queries?**
> "Three ways:
> 1. Query methods by naming convention: findByNameAndStatus()
> 2. @Query annotation with JPQL: @Query(\"SELECT u FROM User u WHERE...\")
> 3. @Query with native SQL: @Query(value = \"SELECT * FROM users...\", nativeQuery = true)"

---

### Q12: What is Spring Actuator?

**Answer:**
> "Spring Actuator provides production-ready features like health checks, metrics, info about the app, and environment details through HTTP endpoints or JMX. Common endpoints include /actuator/health, /actuator/info, /actuator/metrics."

**Follow-up Q: How do you secure Actuator endpoints?**
> "Configure Spring Security to require authentication for sensitive endpoints. By default, only /health and /info are exposed. Use management.endpoints.web.exposure.include to control which endpoints are available."

---

### Q13: Difference between PUT and PATCH

**Answer:**
> "PUT replaces the entire resource - you must send all fields. PATCH partially updates the resource - you send only the fields to change. In Spring, use @PutMapping for full updates and @PatchMapping for partial updates."

**Follow-up Q: How do you handle PATCH requests?**
> "Options: Accept a Map<String, Object>, use a DTO with all nullable fields, use JSON Patch (RFC 6902), or use a library like MapStruct to merge non-null fields."

---

### Q14: What are Spring Profiles?

**Answer:**
> "Profiles allow you to define different configurations for different environments (dev, test, prod). Profile-specific properties go in application-{profile}.yml. Activate with spring.profiles.active property or SPRING_PROFILES_ACTIVE environment variable."

**Follow-up Q: Can you have multiple active profiles?**
> "Yes, comma-separated: spring.profiles.active=dev,local. Properties from later profiles override earlier ones. You can also use @Profile on beans to conditionally create them."

---

### Q15: How do you test a Spring Boot application?

**Answer:**
> "Spring Boot provides:
> - @SpringBootTest: Full integration test with complete context
> - @WebMvcTest: Only web layer, mocks services
> - @DataJpaTest: Only JPA layer with embedded database
> - @MockBean: Creates mock of a bean in context
> - MockMvc: Tests controllers without starting server"

**Follow-up Q: How do you test a private method?**
> "You shouldn't test private methods directly. Test them through public methods that use them. If a private method is complex enough to need its own test, consider extracting it to a separate class with a public method."

---

## Quick Reference: Annotation Cheat Sheet

```
┌─────────────────────────────────────────────────────────────────┐
│                  ANNOTATION CHEAT SHEET                          │
├──────────────────────┬──────────────────────────────────────────┤
│  CORE                │                                          │
│  @SpringBootApp      │  Main entry point                        │
│  @Component          │  Generic bean                            │
│  @Service            │  Business logic bean                     │
│  @Repository         │  Data access bean                        │
│  @Controller         │  MVC controller (returns view)           │
│  @RestController     │  REST controller (returns data)          │
│  @Configuration      │  Contains @Bean definitions              │
│  @Bean               │  Method returns managed bean             │
├──────────────────────┼──────────────────────────────────────────┤
│  INJECTION           │                                          │
│  @Autowired          │  Inject dependency                       │
│  @Qualifier          │  Specify which bean to inject            │
│  @Primary            │  Default bean for injection              │
│  @Value              │  Inject property value                   │
├──────────────────────┼──────────────────────────────────────────┤
│  WEB                 │                                          │
│  @RequestMapping     │  Map URL to handler                      │
│  @GetMapping         │  HTTP GET                                │
│  @PostMapping        │  HTTP POST                               │
│  @PutMapping         │  HTTP PUT                                │
│  @DeleteMapping      │  HTTP DELETE                             │
│  @PatchMapping       │  HTTP PATCH                              │
│  @PathVariable       │  Extract from URL path                   │
│  @RequestParam       │  Extract query parameter                 │
│  @RequestBody        │  Deserialize request body                │
│  @ResponseBody       │  Serialize return value                  │
│  @ResponseStatus     │  Set HTTP status code                    │
├──────────────────────┼──────────────────────────────────────────┤
│  VALIDATION          │                                          │
│  @Valid              │  Trigger validation                      │
│  @NotNull            │  Cannot be null                          │
│  @NotBlank           │  Cannot be null or blank                 │
│  @NotEmpty           │  Cannot be null or empty                 │
│  @Size               │  Length constraints                      │
│  @Email              │  Valid email format                      │
│  @Min/@Max           │  Numeric bounds                          │
│  @Pattern            │  Regex match                             │
├──────────────────────┼──────────────────────────────────────────┤
│  JPA                 │                                          │
│  @Entity             │  JPA entity                              │
│  @Table              │  Table name                              │
│  @Id                 │  Primary key                             │
│  @GeneratedValue     │  Auto-generate ID                        │
│  @Column             │  Column mapping                          │
│  @OneToMany          │  One-to-many relationship                │
│  @ManyToOne          │  Many-to-one relationship                │
│  @JoinColumn         │  Foreign key column                      │
│  @Query              │  Custom query                            │
│  @Modifying          │  For UPDATE/DELETE queries               │
├──────────────────────┼──────────────────────────────────────────┤
│  TRANSACTION         │                                          │
│  @Transactional      │  Wrap in transaction                     │
├──────────────────────┼──────────────────────────────────────────┤
│  LIFECYCLE           │                                          │
│  @PostConstruct      │  After bean initialization               │
│  @PreDestroy         │  Before bean destruction                 │
├──────────────────────┼──────────────────────────────────────────┤
│  TESTING             │                                          │
│  @SpringBootTest     │  Full integration test                   │
│  @WebMvcTest         │  Web layer only                          │
│  @DataJpaTest        │  JPA layer only                          │
│  @MockBean           │  Mock a bean                             │
└──────────────────────┴──────────────────────────────────────────┘
```

---

## Key Takeaways for Interview

1. **Always explain with examples** - Don't just define terms, show code
2. **Know the "why"** - Why constructor injection? Why @Service over @Component?
3. **Understand trade-offs** - When to use what approach
4. **Be ready for follow-ups** - Interviewers dig deeper on each topic
5. **Practice explaining** - Clear communication matters

**Good luck with your interview!**
