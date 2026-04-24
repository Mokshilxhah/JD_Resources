# 🔍 8. Logging & Debugging — Separates Beginners from Developers

> **Mantra:** `System.out.println` is for tutorials. Real developers use loggers, read stack traces without panic, and debug systematically. This skill makes you 10x faster at fixing bugs.

---

## 🤔 Why Not System.out.println?

```java
// ❌ System.out.println — the amateur way
System.out.println("User saved: " + user);

// Problems:
// - No timestamp
// - No log level (is this debug? error? warning?)
// - Can't turn off in production without changing code
// - No context (which class, which thread?)
// - Pollutes production logs with debug noise

// ✅ Logger — the professional way
log.info("User saved: {}", user.getId());

// Gives you:
// 2024-01-15 10:30:45.123 INFO  [main] c.e.service.UserService - User saved: 42
//  ↑ timestamp             ↑ level ↑ thread  ↑ class                  ↑ message
```

---

## 📚 SLF4J — The Logging Standard

### What is SLF4J?

```
SLF4J = Simple Logging Facade for Java

It's an INTERFACE (facade) — not an implementation.
Just like JPA (interface) + Hibernate (implementation),
SLF4J (interface) + Logback (implementation, default in Spring Boot)

You code against SLF4J — Spring Boot uses Logback under the hood.
You never change your code if you switch from Logback → Log4j2.
```

### Setting Up Logger

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Service
public class UserService {

    // One logger per class — standard pattern
    private static final Logger log = LoggerFactory.getLogger(UserService.class);

    public User createUser(UserCreateRequest request) {
        log.debug("Creating user with username: {}", request.getUsername());

        User user = mapToEntity(request);
        User saved = userRepository.save(user);

        log.info("User created successfully. ID: {}, Username: {}",
                 saved.getId(), saved.getUsername());

        return new UserResponse(saved);
    }
}
```

### With Lombok (Cleaner — No Boilerplate)

```java
import lombok.extern.slf4j.Slf4j;

@Slf4j  // Lombok creates: private static final Logger log = LoggerFactory.getLogger(...)
@Service
public class UserService {

    public User createUser(UserCreateRequest request) {
        log.info("Creating user: {}", request.getUsername()); // 'log' just works ✨
        // ... rest of code
    }
}
```

---

## 🎚️ Log Levels — Use the Right Level

```
TRACE  → Most detailed. Every tiny step. (almost never used)
DEBUG  → Detailed flow. "Entering method", variable values.
INFO   → Important events. "User created", "Payment processed".
WARN   → Something unexpected but app still works. "Retry attempt 2"
ERROR  → Something failed. "DB connection failed", exceptions.
FATAL  → App is about to crash (rarely used with SLF4J)

Level hierarchy:
TRACE < DEBUG < INFO < WARN < ERROR

If you set level to INFO → you see INFO + WARN + ERROR (not TRACE/DEBUG)
If you set level to DEBUG → you see DEBUG + INFO + WARN + ERROR (not TRACE)
```

### When to Use Which Level

```java
@Service
public class PaymentService {

    @Slf4j  // assuming this annotation
    // ... (Lombok handles logger creation)

    public PaymentResult processPayment(PaymentRequest request) {

        // DEBUG — detailed flow, turned OFF in production
        log.debug("Processing payment. Amount: {}, User: {}",
                  request.getAmount(), request.getUserId());

        try {
            // INFO — important business events, always on
            log.info("Payment initiated. OrderId: {}, Amount: {}",
                     request.getOrderId(), request.getAmount());

            PaymentResult result = paymentGateway.charge(request);

            // INFO — successful completion
            log.info("Payment successful. TransactionId: {}", result.getTransactionId());

            return result;

        } catch (PaymentGatewayTimeoutException e) {
            // WARN — something went wrong but we'll retry
            log.warn("Payment gateway timeout for order: {}. Retrying...",
                     request.getOrderId());
            return retryPayment(request);

        } catch (InsufficientFundsException e) {
            // WARN — expected business scenario (not a bug)
            log.warn("Insufficient funds for user: {}, Amount: {}",
                     request.getUserId(), request.getAmount());
            throw e;

        } catch (Exception e) {
            // ERROR — unexpected failure — always log the full exception!
            log.error("Payment processing failed for order: {}. Error: {}",
                      request.getOrderId(), e.getMessage(), e);
            //                                               ↑ passing 'e' adds full stack trace
            throw new PaymentProcessingException("Payment failed", e);
        }
    }
}
```

🧠 **Golden Rule:**
```
DEBUG  → "I want to see this while developing"
INFO   → "This should always be visible in logs"
WARN   → "This is unusual but not broken"
ERROR  → "This is broken, someone needs to look at this"
```

---

## ⚙️ Configuring Log Levels

### In application.properties

```properties
# Root level (everything not specifically configured)
logging.level.root=INFO

# Your app's packages — DEBUG during development
logging.level.com.example=DEBUG
logging.level.com.example.service=DEBUG
logging.level.com.example.repository=TRACE

# Third-party libraries — INFO or higher (too noisy at DEBUG)
logging.level.org.springframework=INFO
logging.level.org.hibernate=INFO

# See SQL queries Hibernate generates
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql=TRACE

# Log to a file (in addition to console)
logging.file.name=logs/application.log
logging.file.max-size=10MB
logging.file.max-history=30  # keep 30 days of logs

# Log format
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} %-5level [%thread] %logger{36} - %msg%n
```

### In application-dev.properties vs application-prod.properties

```properties
# application-dev.properties
logging.level.com.example=DEBUG
logging.level.org.hibernate.SQL=DEBUG  # see all SQL in dev

# application-prod.properties
logging.level.com.example=INFO
logging.level.org.springframework=WARN
logging.file.name=/var/log/myapp/application.log
```

---

## 🐛 Debugging — Reading Stack Traces

### Anatomy of a Stack Trace

```
java.lang.NullPointerException: Cannot invoke "String.length()" because "str" is null
  ↑ Exception type              ↑ The actual error message

    at com.example.service.UserService.processName(UserService.java:45)
    ↑ class                              ↑ method          ↑ file  ↑ line number  ← YOUR CODE (start here!)

    at com.example.controller.UserController.createUser(UserController.java:28)
    at java.base/jdk.internal.reflect.NativeMethodAccessorImpl.invoke0(Native Method)
    at org.springframework.web.servlet.FrameworkServlet.service(FrameworkServlet.java:897)
    ... (Spring framework internals — usually ignore these)
```

**How to read it:**
1. Read the **first line** — that's the exception + message
2. Find **YOUR code** in the stack (ignore `java.`, `org.springframework.`, `com.sun.`)
3. The **topmost** entry of your code = where it actually broke
4. Read **up the call chain** to understand HOW you got there

### Common Exceptions — What They Mean

```java
// NullPointerException — you called method on null object
User user = null;
user.getName(); // ❌ NPE

// Fix: null check or use Optional
if (user != null) { user.getName(); }
userRepository.findById(id).orElseThrow(() -> new ResourceNotFoundException("..."));


// ClassCastException — wrong type cast
Object obj = "hello";
Integer num = (Integer) obj; // ❌ ClassCastException

// Fix: check type first
if (obj instanceof Integer) { Integer num = (Integer) obj; }


// ArrayIndexOutOfBoundsException — array index doesn't exist
int[] arr = {1, 2, 3};
arr[5]; // ❌ index 5 doesn't exist (0-2 only)

// Fix: check length
if (index < arr.length) { arr[index]; }


// StackOverflowError — infinite recursion
public int factorial(int n) {
    return n * factorial(n); // ❌ never stops — forgot base case
}

// Fix: add base case
public int factorial(int n) {
    if (n <= 1) return 1; // base case
    return n * factorial(n - 1);
}


// LazyInitializationException — accessed lazy relation outside session
// (Spring Data JPA specific)
User user = userRepository.findById(1L).get(); // session closes after this
user.getOrders().size(); // ❌ lazy loaded orders, session already closed

// Fix: use @Transactional, or fetch eager, or use JOIN FETCH query


// BeanCreationException — Spring can't create a bean
// Usually means: constructor injection failed, missing dependency, circular dependency
// Read the "Caused by:" section at the bottom of the stack trace!
```

### Debugging Strategy — Step by Step

```
Step 1: Read the exception type — what kind of error?
Step 2: Read the message — what's the specific problem?
Step 3: Find YOUR code in the stack trace
Step 4: Go to that exact line
Step 5: Ask: "What could be null / wrong here?"
Step 6: Add log statements around that area
Step 7: Fix and verify
```

### IntelliJ Debugger — The Power Tool

```java
// Breakpoint debugging — click on line number in IntelliJ to add breakpoint
// Then run in Debug mode (Shift+F9)

@Service
public class OrderService {

    public Order calculateTotal(List<OrderItem> items) {
        // Put breakpoint on next line ← click line number
        double total = 0;
        for (OrderItem item : items) {  // ← or here to inspect each item
            total += item.getPrice() * item.getQuantity();
        }
        return new Order(total);
    }
}

// In Debug mode you can:
// F8 → Step Over (go to next line)
// F7 → Step Into (go inside method call)
// F9 → Resume (continue to next breakpoint)
// Evaluate Expression → run any Java expression mid-debug
// Watch → monitor variable changes in real-time
```

### Conditional Breakpoints

```java
// Right-click breakpoint → "Edit breakpoint"
// Condition: items.size() > 10
// Now it only stops when list has more than 10 items — great for loops!
```

---

## 📊 Structured Logging — Best Practices

```java
@Slf4j
@Service
public class UserService {

    // ✅ Use placeholders {} — NOT string concatenation
    log.info("User created: {}", userId);        // right
    log.info("User created: " + userId);         // wrong (always evaluates, even if DEBUG is off)

    // ✅ Log context, not just messages
    log.info("Payment processed. userId={}, orderId={}, amount={}", userId, orderId, amount);

    // ✅ Log exceptions with full stack trace (pass exception as last arg)
    try {
        // ...
    } catch (Exception e) {
        log.error("Failed to process order: {}", orderId, e);  // e at end = full stack trace
    }

    // ✅ Don't log sensitive data!
    log.info("Login attempt for user: {}", username);   // OK
    log.info("Password: {}", password);                  // ❌ NEVER LOG PASSWORDS!
    log.info("Token: {}", jwtToken);                     // ❌ NEVER LOG TOKENS!
    log.info("Card: {}", cardNumber);                    // ❌ NEVER LOG CARD NUMBERS!

    // ✅ Guard expensive debug logs
    if (log.isDebugEnabled()) {
        log.debug("Full request payload: {}", objectMapper.writeValueAsString(bigObject));
        // Only serializes to JSON if DEBUG is actually enabled
    }
}
```

---

## 🎯 Quick Revision Flashcards

| Concept | One-Line Summary |
|---------|----------------|
| SLF4J | Logging interface — Logback is the default implementation in Spring Boot |
| `@Slf4j` | Lombok annotation — creates `log` field automatically |
| TRACE | Finest detail — almost never used |
| DEBUG | Dev-time detail — off in production |
| INFO | Important events — always on |
| WARN | Unexpected but survivable |
| ERROR | Something broke — needs attention |
| `log.error("msg", e)` | Last param is exception — includes full stack trace |
| `{}` in log | Placeholder — evaluated only if level is active |
| Stack trace reading | First line = error, find YOUR code, topmost = where it broke |
| NPE | Called method on null object |
| LazyInitializationException | Accessed lazy-loaded entity outside transaction |
| BeanCreationException | Spring failed to create a bean — read "Caused by:" |
| Breakpoint | Pause execution at a line in IntelliJ debugger |
| F8 | Step over in debugger |
| F7 | Step into method in debugger |

---

> ✅ **You've mastered Logging & Debugging when:** You never use `System.out.println` in real code, you can read a 50-line stack trace and find the actual problem in 30 seconds, and your logs tell a story about what your app is doing.
