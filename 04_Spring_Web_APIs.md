# 🌐 4. Spring Web — Building REST APIs

> **Mantra:** This is where projects actually begin. Everything before this was setup — NOW you build things people actually use.

---

## 🤔 What is a REST API?

Think of it like a **waiter** in a restaurant:
- You (client/browser/mobile app) place an **order** (HTTP request)
- Waiter (Spring Controller) takes it to the kitchen (Service)
- Kitchen prepares food (business logic + DB)
- Waiter brings back the food (HTTP response with JSON)

```
Client                    Spring Boot App
──────                    ───────────────
GET /api/users    →       Controller → Service → Repository → DB
                  ←       JSON Response [{"id":1,"name":"Alice"},...]
```

### HTTP Methods — The Vocabulary

| Method | Purpose | Body? | Example |
|--------|---------|-------|---------|
| GET | Read data | ❌ | GET /users — get all users |
| POST | Create data | ✅ | POST /users — create user |
| PUT | Update (full replace) | ✅ | PUT /users/1 — replace user 1 |
| PATCH | Update (partial) | ✅ | PATCH /users/1 — update name only |
| DELETE | Delete data | ❌ | DELETE /users/1 — delete user 1 |

---

## 🎯 @RestController & @RequestMapping

```java
// @Controller → returns VIEW NAME (for HTML pages)
// @RestController → returns DATA (JSON/XML) directly
// @RestController = @Controller + @ResponseBody on every method

@RestController                    // This class handles HTTP requests + returns JSON
@RequestMapping("/api/users")      // All endpoints in this class start with /api/users
public class UserController {

    private final UserService userService;

    // Constructor injection — always preferred
    public UserController(UserService userService) {
        this.userService = userService;
    }

    // GET /api/users
    @GetMapping
    public List<User> getAllUsers() {
        return userService.findAll();
    }

    // GET /api/users/5
    @GetMapping("/{id}")
    public User getUserById(@PathVariable Long id) {
        return userService.findById(id);
    }

    // POST /api/users
    @PostMapping
    public User createUser(@RequestBody User user) {
        return userService.save(user);
    }

    // PUT /api/users/5
    @PutMapping("/{id}")
    public User updateUser(@PathVariable Long id, @RequestBody User user) {
        return userService.update(id, user);
    }

    // DELETE /api/users/5
    @DeleteMapping("/{id}")
    public void deleteUser(@PathVariable Long id) {
        userService.delete(id);
    }
}
```

### All HTTP Mapping Annotations

```java
@GetMapping("/path")       // GET — read
@PostMapping("/path")      // POST — create
@PutMapping("/path")       // PUT — full update
@PatchMapping("/path")     // PATCH — partial update
@DeleteMapping("/path")    // DELETE — remove

// These are shortcuts for:
@RequestMapping(value = "/path", method = RequestMethod.GET)
// @GetMapping is just cleaner ✅
```

---

## 📥 Request Handling — Getting Data FROM the Client

### @PathVariable — Part of the URL path

```java
// URL: GET /api/users/42
@GetMapping("/{id}")
public User getUser(@PathVariable Long id) {
    // id = 42
    return userService.findById(id);
}

// Multiple path variables
// URL: GET /api/departments/IT/employees/5
@GetMapping("/departments/{dept}/employees/{empId}")
public Employee getEmployee(
    @PathVariable String dept,
    @PathVariable Long empId) {
    return employeeService.find(dept, empId);
}

// Custom name (when param name differs from variable)
@GetMapping("/users/{userId}")
public User getUser(@PathVariable("userId") Long id) { ... }
```

### @RequestParam — Query parameters (?key=value)

```java
// URL: GET /api/users?page=0&size=10&name=Alice
@GetMapping
public List<User> getUsers(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size,
    @RequestParam(required = false) String name) {  // optional param
    return userService.findAll(page, size, name);
}

// URL: GET /api/search?q=spring&sort=name
@GetMapping("/search")
public List<Product> search(
    @RequestParam String q,
    @RequestParam(defaultValue = "name") String sort) {
    return productService.search(q, sort);
}
```

### @RequestBody — JSON body → Java Object

```java
// Client sends:
// POST /api/users
// Content-Type: application/json
// Body: {"name": "Alice", "email": "alice@example.com", "age": 25}

@PostMapping
public User createUser(@RequestBody User user) {
    // Spring auto-converts JSON → User object (Jackson does this)
    // user.getName() = "Alice"
    // user.getEmail() = "alice@example.com"
    return userService.save(user);
}
```

### All Three Together — Quick Comparison

```java
// @PathVariable → /users/42          → identifies a resource
// @RequestParam → /users?active=true → filters/options
// @RequestBody  → JSON in body       → data to create/update

@PutMapping("/{id}")          // PathVariable for WHICH user
public User update(
    @PathVariable Long id,                    // /users/42
    @RequestParam boolean notify,             // ?notify=true
    @RequestBody UserUpdateRequest request) { // JSON body with new data
    return userService.update(id, request, notify);
}
// Full URL: PUT /api/users/42?notify=true  + JSON body
```

---

## 📤 Response Handling — Sending Data TO the Client

### ResponseEntity — Full Control Over Response

```java
// Without ResponseEntity — just returns data, Spring picks 200 OK
@GetMapping("/{id}")
public User getUser(@PathVariable Long id) {
    return userService.findById(id); // always 200 OK
}

// With ResponseEntity — you control status code + headers + body
@GetMapping("/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) {
    User user = userService.findById(id);

    if (user == null) {
        return ResponseEntity.notFound().build(); // 404, no body
    }

    return ResponseEntity.ok(user); // 200 + user as JSON body
}
```

### Building ResponseEntity

```java
// 200 OK with body
return ResponseEntity.ok(user);
return ResponseEntity.status(HttpStatus.OK).body(user);

// 201 Created with body + Location header
return ResponseEntity
    .created(URI.create("/api/users/" + saved.getId()))
    .body(saved);

// 204 No Content (for DELETE)
return ResponseEntity.noContent().build();

// 400 Bad Request
return ResponseEntity.badRequest().body("Invalid input");

// 404 Not Found
return ResponseEntity.notFound().build();

// 500 Internal Server Error
return ResponseEntity.internalServerError().body("Something went wrong");

// Custom status
return ResponseEntity.status(HttpStatus.CONFLICT).body("User already exists");
```

### HTTP Status Codes — The Essentials

```
2xx — Success
  200 OK          → Standard success (GET, PUT, PATCH)
  201 Created     → Resource created (POST)
  204 No Content  → Success, no body returned (DELETE)

4xx — Client Error (your fault)
  400 Bad Request   → Invalid data sent
  401 Unauthorized  → Not logged in
  403 Forbidden     → Logged in but no permission
  404 Not Found     → Resource doesn't exist
  409 Conflict      → Duplicate resource

5xx — Server Error (our fault)
  500 Internal Server Error → Something broke in code
  503 Service Unavailable   → Server overloaded/down
```

### @ResponseStatus — Simpler status for whole method

```java
@PostMapping
@ResponseStatus(HttpStatus.CREATED)   // always returns 201
public User createUser(@RequestBody User user) {
    return userService.save(user); // no need for ResponseEntity
}

@DeleteMapping("/{id}")
@ResponseStatus(HttpStatus.NO_CONTENT) // always returns 204
public void delete(@PathVariable Long id) {
    userService.delete(id);
}
```

---

## 🚨 Exception Handling — @ControllerAdvice

### The Problem Without Global Handler

```java
// Without global handling — every controller needs try-catch
@GetMapping("/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) {
    try {
        return ResponseEntity.ok(userService.findById(id));
    } catch (UserNotFoundException e) {
        return ResponseEntity.notFound().build();
    } catch (Exception e) {
        return ResponseEntity.internalServerError().build();
    }
    // Repeated in EVERY endpoint = awful ❌
}
```

### The Solution — @ControllerAdvice (Global Handler)

**Step 1: Create a custom exception**
```java
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}

public class DuplicateResourceException extends RuntimeException {
    public DuplicateResourceException(String message) {
        super(message);
    }
}
```

**Step 2: Throw it from service**
```java
@Service
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public User findById(Long id) {
        return userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User not found with id: " + id));
    }

    public User create(User user) {
        if (userRepository.existsByEmail(user.getEmail())) {
            throw new DuplicateResourceException("Email already exists: " + user.getEmail());
        }
        return userRepository.save(user);
    }
}
```

**Step 3: @ControllerAdvice catches globally**
```java
// Standard error response structure
public class ErrorResponse {
    private int status;
    private String message;
    private LocalDateTime timestamp;

    public ErrorResponse(int status, String message) {
        this.status = status;
        this.message = message;
        this.timestamp = LocalDateTime.now();
    }
    // getters...
}

@ControllerAdvice // intercepts ALL exceptions from ALL controllers
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        ErrorResponse error = new ErrorResponse(404, ex.getMessage());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(DuplicateResourceException ex) {
        ErrorResponse error = new ErrorResponse(409, ex.getMessage());
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class) // @Valid failures
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult()
            .getFieldErrors()
            .stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        ErrorResponse error = new ErrorResponse(400, message);
        return ResponseEntity.badRequest().body(error);
    }

    @ExceptionHandler(Exception.class) // catch-all for unexpected errors
    public ResponseEntity<ErrorResponse> handleAll(Exception ex) {
        ErrorResponse error = new ErrorResponse(500, "Internal server error");
        return ResponseEntity.internalServerError().body(error);
    }
}
```

**Now your controllers are CLEAN:**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping("/{id}")
    public User getUser(@PathVariable Long id) {
        return userService.findById(id); // exception auto-handled globally ✅
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public User create(@RequestBody User user) {
        return userService.create(user); // same here ✅
    }
}
```

### What the Error Response Looks Like

```json
// GET /api/users/999 (not found)
{
  "status": 404,
  "message": "User not found with id: 999",
  "timestamp": "2024-01-15T10:30:00"
}

// POST /api/users (duplicate email)
{
  "status": 409,
  "message": "Email already exists: alice@example.com",
  "timestamp": "2024-01-15T10:30:01"
}
```

---

## 🏗️ Complete REST Controller Pattern

```java
@RestController
@RequestMapping("/api/products")
public class ProductController {

    private final ProductService productService;

    public ProductController(ProductService productService) {
        this.productService = productService;
    }

    // GET /api/products                     → all
    // GET /api/products?category=electronics → filtered
    @GetMapping
    public List<Product> getAll(
        @RequestParam(required = false) String category) {
        if (category != null) return productService.findByCategory(category);
        return productService.findAll();
    }

    // GET /api/products/5
    @GetMapping("/{id}")
    public ResponseEntity<Product> getById(@PathVariable Long id) {
        return ResponseEntity.ok(productService.findById(id));
        // ResourceNotFoundException auto-handled by @ControllerAdvice
    }

    // POST /api/products
    @PostMapping
    public ResponseEntity<Product> create(@RequestBody @Valid ProductRequest request) {
        Product saved = productService.create(request);
        return ResponseEntity
            .created(URI.create("/api/products/" + saved.getId()))
            .body(saved);
    }

    // PUT /api/products/5
    @PutMapping("/{id}")
    public Product update(@PathVariable Long id, @RequestBody @Valid ProductRequest request) {
        return productService.update(id, request);
    }

    // DELETE /api/products/5
    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) {
        productService.delete(id);
    }
}
```

---

## 🎯 Quick Revision Flashcards

| Concept | One-Line Summary |
|---------|----------------|
| `@RestController` | = @Controller + @ResponseBody — returns JSON |
| `@RequestMapping` | Base URL prefix for all methods in controller |
| `@GetMapping` | Handle GET requests (read) |
| `@PostMapping` | Handle POST requests (create) |
| `@PutMapping` | Handle PUT requests (full update) |
| `@DeleteMapping` | Handle DELETE requests (remove) |
| `@PathVariable` | Get value from URL path `/users/{id}` |
| `@RequestParam` | Get query param `/users?page=0` |
| `@RequestBody` | Convert JSON body → Java object |
| `ResponseEntity` | Full control: status + headers + body |
| `@ResponseStatus` | Set default HTTP status for a method |
| `@ControllerAdvice` | Global exception handler for ALL controllers |
| `@ExceptionHandler` | Handle specific exception type in advice |
| 200 OK | Success |
| 201 Created | Resource created (POST) |
| 204 No Content | Success, no body (DELETE) |
| 404 Not Found | Resource missing |
| 400 Bad Request | Invalid input |

---

> ✅ **You've mastered Spring Web when:** You can build a full CRUD API, handle path/query params, parse request bodies, return proper status codes, and all exceptions are handled in ONE place via @ControllerAdvice — without a single try-catch in controllers.
