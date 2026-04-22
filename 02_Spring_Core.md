# 🌱 2. Spring Core — Foundation of Everything

> **Mantra:** If you don't understand DI deeply, you'll struggle in debugging. This is THE heart of Spring.

---

## 🔄 IoC — Inversion of Control

### What Problem Does It Solve?

**Without IoC (you're in control):**
```java
public class OrderService {
    // You create the dependency yourself
    private PaymentService paymentService = new PaymentService();
    private EmailService emailService = new EmailService();

    public void placeOrder(Order order) {
        paymentService.charge(order);
        emailService.sendConfirmation(order);
    }
}
```

**Problems with this:**
- `OrderService` is tightly coupled to `PaymentService`
- Want to swap `PaymentService` with `MockPaymentService` for testing? You must edit the class.
- Want to change how `EmailService` is created? Same problem.

**With IoC (Spring is in control):**
```java
public class OrderService {
    private PaymentService paymentService; // just declare, don't create
    private EmailService emailService;

    // Spring creates AND gives you the objects
    public OrderService(PaymentService paymentService, EmailService emailService) {
        this.paymentService = paymentService;
        this.emailService = emailService;
    }
}
```

🧠 **Remember:** IoC = "Don't call us, we'll call you."
- Normal flow: Your code creates objects
- Inverted flow: Framework creates and provides objects to your code

---

## 💉 Dependency Injection (DI) — How IoC Actually Happens

DI is the **mechanism** of IoC. Spring "injects" dependencies into your class.

### 1. Constructor Injection ✅ (MOST IMPORTANT — Use This Always)

```java
@Service
public class OrderService {

    private final PaymentService paymentService; // final = immutable ✅
    private final EmailService emailService;

    // Spring sees @Service + constructor = automatically injects
    @Autowired // optional in newer Spring (if only 1 constructor)
    public OrderService(PaymentService paymentService, EmailService emailService) {
        this.paymentService = paymentService;
        this.emailService = emailService;
    }

    public void placeOrder(Order order) {
        paymentService.charge(order);
        emailService.sendConfirmation(order);
    }
}
```

**Why Constructor Injection is BEST:**
```
✅ Fields can be final (immutable, thread-safe)
✅ Easy to test — just pass mock objects to constructor
✅ Fails fast — if dependency missing, app won't start
✅ No hidden dependencies — all deps visible in constructor
✅ Works without Spring (just new OrderService(mockPayment, mockEmail))
```

### 2. Setter Injection (Optional dependencies)

```java
@Service
public class NotificationService {

    private EmailService emailService;

    @Autowired // Spring calls this setter after creating the bean
    public void setEmailService(EmailService emailService) {
        this.emailService = emailService;
    }
}
```

**When to use:** Only for optional dependencies that have defaults.

### 3. Field Injection ❌ (Avoid in production)

```java
@Service
public class BadService {
    @Autowired
    private PaymentService paymentService; // Spring injects directly into field
}
```

**Why it's bad:**
```
❌ Can't make it final
❌ Hard to test (need reflection or Spring context)
❌ Hides dependencies — not visible from constructor
❌ Can lead to NullPointerExceptions if used outside Spring
```

### Side-by-Side Comparison

```java
// ❌ Field Injection (avoid)
@Autowired
private UserRepository repo;

// ⚠️ Setter Injection (optional deps only)
@Autowired
public void setRepo(UserRepository repo) { this.repo = repo; }

// ✅ Constructor Injection (always prefer)
private final UserRepository repo;

public UserService(UserRepository repo) { this.repo = repo; }
```

---

## 🫘 Bean Lifecycle — Birth to Death

### What is a Bean?
A **Bean** = an object managed by Spring's IoC container. Spring creates it, wires dependencies, and destroys it.

### The Full Lifecycle

```
1. Spring reads @Component / @Bean / XML config
2. Creates the bean instance (calls constructor)
3. Injects dependencies (DI happens here)
4. Calls @PostConstruct method (your init code)
5. Bean is READY — serving requests 🟢
6. App shutting down → @PreDestroy method called
7. Bean destroyed
```

```java
@Component
public class DatabaseConnectionPool {

    private List<Connection> connections;

    public DatabaseConnectionPool() {
        System.out.println("1. Constructor called");
    }

    @PostConstruct // runs AFTER DI is complete
    public void init() {
        System.out.println("2. @PostConstruct — creating connection pool");
        connections = new ArrayList<>();
        for (int i = 0; i < 10; i++) {
            connections.add(createConnection());
        }
    }

    @PreDestroy // runs BEFORE bean is destroyed
    public void cleanup() {
        System.out.println("3. @PreDestroy — closing all connections");
        connections.forEach(Connection::close);
    }

    private Connection createConnection() { /* ... */ return null; }
}
```

### Bean Scopes

```java
@Component
@Scope("singleton") // default — one instance for entire app
public class AppConfig { }

@Component
@Scope("prototype") // new instance every time you request it
public class ReportGenerator { }

// In Spring MVC:
@Component
@Scope("request") // new instance per HTTP request
public class RequestContext { }

@Component
@Scope("session") // new instance per user session
public class ShoppingCart { }
```

🧠 **Remember:**
- **Singleton** = one shared instance (like a shared printer in office)
- **Prototype** = new instance every time (like a new coffee cup each time)

---

## 🏭 Application Context — The IoC Container

### BeanFactory vs ApplicationContext

```
BeanFactory                    ApplicationContext
──────────────────────         ──────────────────────────────
Basic IoC container            BeanFactory + extra features
Lazy initialization            Eager initialization (default)
No AOP support                 AOP support ✅
No Event handling              Event publishing ✅
No i18n                        Internationalization ✅
Older, low-memory use          Standard for all Spring apps ✅

→ Always use ApplicationContext
→ BeanFactory is only for extremely memory-constrained environments
```

### How to Use ApplicationContext

```java
// Spring Boot does this automatically for you
// But understanding it is key

// Manual setup (non-Boot Spring):
ApplicationContext context = new ClassPathXmlApplicationContext("beans.xml");
// or
ApplicationContext context = new AnnotationConfigApplicationContext(AppConfig.class);

// Get a bean
UserService userService = context.getBean(UserService.class);
userService.doSomething();

// In Spring Boot — it's done via @SpringBootApplication
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        ApplicationContext ctx = SpringApplication.run(MyApp.class, args);

        // You can access beans directly:
        UserService svc = ctx.getBean(UserService.class);
    }
}
```

### @Configuration + @Bean (Java Config)

```java
@Configuration // tells Spring: this class has bean definitions
public class AppConfig {

    @Bean // Spring manages this object as a bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // returns the bean
    }

    @Bean
    public UserService userService(UserRepository repo) {
        // Spring auto-injects 'repo' here
        return new UserService(repo);
    }
}
```

🧠 **When to use @Bean vs @Component:**
- `@Component` — on YOUR classes that Spring should manage
- `@Bean` — when you don't own the class (e.g., third-party libraries), define in @Configuration

---

## 📝 Annotations — The Most Important Ones

### Stereotype Annotations (All are @Component specializations)

```java
// @Component — generic Spring-managed bean
@Component
public class UtilityHelper {
    public String format(String s) { return s.trim().toUpperCase(); }
}

// @Service — business logic layer (same as @Component, more semantic meaning)
@Service
public class UserService {
    public User findById(Long id) { /* business logic */ return null; }
}

// @Repository — data access layer
// BONUS: Spring translates DB exceptions to Spring's DataAccessException
@Repository
public class UserRepository {
    public User findById(Long id) { /* DB query */ return null; }
}

// @Controller — Spring MVC web layer (handles HTTP requests)
@Controller
public class UserController {
    @GetMapping("/users")
    public String listUsers(Model model) {
        return "users"; // returns view name
    }
}

// @RestController = @Controller + @ResponseBody (returns JSON/XML directly)
@RestController
public class UserApiController {
    @GetMapping("/api/users")
    public List<User> getUsers() {
        return List.of(new User("Alice"), new User("Bob")); // auto-converted to JSON
    }
}
```

### Wiring Annotations

```java
@Service
public class OrderService {

    private final UserService userService;
    private final PaymentService paymentService;

    // @Autowired — tells Spring to inject matching beans by TYPE
    @Autowired
    public OrderService(UserService userService, PaymentService paymentService) {
        this.userService = userService;
        this.paymentService = paymentService;
    }
}
```

### @Qualifier — When Multiple Beans of Same Type Exist

```java
public interface NotificationService {
    void send(String message);
}

@Service("emailNotification") // give it a name
public class EmailNotificationService implements NotificationService {
    public void send(String message) { System.out.println("Email: " + message); }
}

@Service("smsNotification")
public class SmsNotificationService implements NotificationService {
    public void send(String message) { System.out.println("SMS: " + message); }
}

@Service
public class AlertService {
    private final NotificationService notifier;

    @Autowired
    public AlertService(@Qualifier("emailNotification") NotificationService notifier) {
        // Without @Qualifier: Spring confused — 2 beans implement NotificationService!
        // With @Qualifier: Spring picks emailNotification specifically ✅
        this.notifier = notifier;
    }
}
```

🧠 **Remember the hierarchy of resolution:**
```
1. By TYPE → if 1 match, done
2. By NAME (field/param name matches bean name) → if match, done
3. @Qualifier → explicitly pick which one
4. @Primary → mark one bean as default winner
```

### @Primary vs @Qualifier

```java
@Service
@Primary // this one wins when ambiguous
public class EmailNotificationService implements NotificationService { }

@Service
public class SmsNotificationService implements NotificationService { }

@Service
public class AlertService {
    @Autowired
    private NotificationService notifier; // gets EmailNotificationService (it's @Primary)
}
```

---

## 🔑 Complete Annotation Quick Reference

| Annotation | Layer | Purpose |
|-----------|-------|---------|
| `@Component` | Any | Generic Spring bean |
| `@Service` | Business | Business logic (semantic @Component) |
| `@Repository` | Data | DB access + exception translation |
| `@Controller` | Web | MVC controller (returns views) |
| `@RestController` | Web | REST API (returns JSON) |
| `@Autowired` | Any | Auto-inject dependency |
| `@Qualifier("name")` | Any | Pick specific bean when multiple exist |
| `@Primary` | Any | Default bean when multiple exist |
| `@Bean` | Config | Define bean in @Configuration class |
| `@Configuration` | Config | Class contains bean definitions |
| `@Scope` | Any | Define bean scope (singleton/prototype) |
| `@PostConstruct` | Any | Run after DI complete |
| `@PreDestroy` | Any | Run before bean destroyed |
| `@Value("${key}")` | Any | Inject property from application.properties |
| `@ComponentScan` | Config | Tell Spring where to look for components |

---

## 🧪 Testing DI — Why Constructor Injection Wins

```java
// With field injection — PAINFUL to test
@Service
public class OrderServiceBad {
    @Autowired
    private PaymentService paymentService; // can't mock easily
}

// Test needs full Spring context... slow and heavy

// With constructor injection — EASY to test
@Service
public class OrderService {
    private final PaymentService paymentService;

    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}

// Test is pure Java — no Spring needed!
class OrderServiceTest {
    @Test
    void testPlaceOrder() {
        PaymentService mockPayment = mock(PaymentService.class); // Mockito mock
        OrderService service = new OrderService(mockPayment);    // just new!
        // test away...
    }
}
```

---

## 🎯 Quick Revision Flashcards

| Concept | One-Line Summary |
|---------|----------------|
| IoC | Framework creates & manages objects, not you |
| DI | How IoC works — objects are "injected" in |
| Constructor Injection | Best — final fields, testable, fail-fast |
| Setter Injection | Optional dependencies only |
| Field Injection | Avoid — not testable, not recommended |
| Bean | Any object managed by Spring container |
| Singleton scope | One instance per Spring context (default) |
| Prototype scope | New instance every `getBean()` call |
| ApplicationContext | The IoC container (holds all beans) |
| @Component | "Make me a Spring bean" |
| @Autowired | "Inject matching bean here" |
| @Qualifier | "Inject THIS specific bean" |
| @PostConstruct | Init code after DI done |
| @PreDestroy | Cleanup before bean destroyed |

---

> ✅ **You've mastered Spring Core when:** You can draw the flow: Source code → @Component scanned → Bean created → Dependencies injected → @PostConstruct → Ready. That's the heart of every Spring application.
