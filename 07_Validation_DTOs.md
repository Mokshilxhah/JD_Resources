# ⚙️ 7. Validation & DTOs — Clean Code Practice

> **Mantra:** Never expose your database entities directly to the outside world. DTOs + Validation = professional, secure, maintainable APIs.

---

## 🤔 The Two Problems This Solves

### Problem 1 — Exposing Entity Directly (Bad)

```java
// Your DB Entity
@Entity
public class User {
    private Long id;
    private String username;
    private String password;      // 🚨 EXPOSED TO CLIENT!
    private String internalNotes; // 🚨 EXPOSED TO CLIENT!
    private LocalDateTime createdAt;
    private String role;
}

// Controller — directly returning entity
@GetMapping("/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id).get();
    // JSON: {"id":1,"username":"alice","password":"$2a$10$...","internalNotes":"VIP user","role":"ADMIN"}
    // ❌ Password hash sent to client
    // ❌ Internal fields leaked
    // ❌ Client sees DB structure
}
```

### Problem 2 — No Validation (Bad)

```java
@PostMapping
public User createUser(@RequestBody User user) {
    return userRepository.save(user);
    // What if name is blank? Email is invalid? Age is -5?
    // All goes straight to DB with no checks ❌
}
```

### The Solution — DTOs + Validation

```
Client → sends RequestDTO (validated) → Service → works with Entity → returns ResponseDTO → Client
         ↑ only what we need              ↑ DB logic                  ↑ only what client should see
```

---

## 📦 DTO Pattern — Data Transfer Objects

### What is a DTO?

A DTO is a **plain Java class** that carries data between layers. It has **no JPA annotations**, **no business logic** — just fields + getters/setters.

```
Entity  = DB representation  (has @Entity, @Id, relationships)
DTO     = API representation (has only what client sends/receives)

Rule: Entities NEVER leave your service layer.
      DTOs are what controllers receive and return.
```

### Request DTO vs Response DTO

```java
// REQUEST DTO — what client sends to create a user
public class UserCreateRequest {
    private String username;  // only what's needed to CREATE
    private String password;
    private String email;
    // No id (auto-generated), no role (server decides), no createdAt
}

// RESPONSE DTO — what client receives back
public class UserResponse {
    private Long id;
    private String username;
    private String email;
    private LocalDateTime createdAt;
    // No password (never send this!)
    // No internalNotes (internal only)
}
```

### Full DTO Example

```java
// 1. Request DTO — create user
public class UserCreateRequest {
    private String username;
    private String password;
    private String email;
    private int age;

    // getters and setters (or use Lombok @Data)
    public String getUsername() { return username; }
    public void setUsername(String username) { this.username = username; }
    // ... etc
}

// 2. Update DTO — update user (different fields, partial update)
public class UserUpdateRequest {
    private String email;    // only email and age can be updated
    private int age;
    // username and password NOT included — not updatable via this endpoint
}

// 3. Response DTO — what API returns
public class UserResponse {
    private Long id;
    private String username;
    private String email;
    private int age;
    private String role;
    private LocalDateTime createdAt;

    // Constructor for easy conversion
    public UserResponse(User user) {
        this.id = user.getId();
        this.username = user.getUsername();
        this.email = user.getEmail();
        this.age = user.getAge();
        this.role = user.getRole();
        this.createdAt = user.getCreatedAt();
    }
}

// 4. Entity — DB representation
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String username;
    private String password;  // BCrypt encoded
    private String email;
    private int age;
    private String role;
    private String internalNotes; // never sent to client
    private LocalDateTime createdAt;
    // getters and setters
}
```

### Service — Converts Between Entity ↔ DTO

```java
@Service
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    public UserService(UserRepository userRepository, PasswordEncoder passwordEncoder) {
        this.userRepository = userRepository;
        this.passwordEncoder = passwordEncoder;
    }

    // Create: RequestDTO → Entity → save → ResponseDTO
    public UserResponse createUser(UserCreateRequest request) {
        // Check duplicate
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new DuplicateResourceException("Username already taken");
        }

        // Map DTO → Entity
        User user = new User();
        user.setUsername(request.getUsername());
        user.setPassword(passwordEncoder.encode(request.getPassword())); // encode!
        user.setEmail(request.getEmail());
        user.setAge(request.getAge());
        user.setRole("ROLE_USER");
        user.setCreatedAt(LocalDateTime.now());

        User saved = userRepository.save(user);

        // Map Entity → ResponseDTO (NEVER return entity directly)
        return new UserResponse(saved);
    }

    // Read: Entity → ResponseDTO
    public UserResponse getUserById(Long id) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User not found: " + id));
        return new UserResponse(user); // map to DTO before returning
    }

    // Update: RequestDTO + Entity → save → ResponseDTO
    public UserResponse updateUser(Long id, UserUpdateRequest request) {
        User user = userRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("User not found: " + id));

        // Only update what's in the DTO
        user.setEmail(request.getEmail());
        user.setAge(request.getAge());

        return new UserResponse(userRepository.save(user));
    }
}
```

### Controller — Uses DTOs Only

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public UserResponse create(@RequestBody @Valid UserCreateRequest request) {
        return userService.createUser(request); // DTO in, DTO out ✅
    }

    @GetMapping("/{id}")
    public UserResponse getById(@PathVariable Long id) {
        return userService.getUserById(id); // DTO out, never entity ✅
    }

    @PutMapping("/{id}")
    public UserResponse update(@PathVariable Long id,
                               @RequestBody @Valid UserUpdateRequest request) {
        return userService.updateUser(id, request);
    }
}
```

---

## ✅ Validation — @Valid + Constraint Annotations

### Setup

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-validation</artifactId>
</dependency>
```

### All Validation Annotations

```java
public class UserCreateRequest {

    // Null / Empty checks
    @NotNull                    // field must not be null
    @NotEmpty                   // not null AND not empty string
    @NotBlank                   // not null, not empty, not just whitespace ✅ (best for strings)
    private String username;

    // String length
    @Size(min = 3, max = 20, message = "Username must be 3-20 characters")
    private String username;

    // Email format
    @Email(message = "Invalid email format")
    private String email;

    // Password
    @NotBlank
    @Size(min = 8, message = "Password must be at least 8 characters")
    @Pattern(regexp = "^(?=.*[A-Z])(?=.*[0-9]).+$",
             message = "Password must have at least 1 uppercase and 1 digit")
    private String password;

    // Numbers
    @Min(value = 18, message = "Age must be at least 18")
    @Max(value = 120, message = "Age must be at most 120")
    private int age;

    @Positive(message = "Price must be positive")          // > 0
    @PositiveOrZero(message = "Quantity must be >= 0")     // >= 0
    @Negative(message = "Discount must be negative")       // < 0
    private double price;

    // Decimal range
    @DecimalMin("0.0")
    @DecimalMax("100.0")
    private BigDecimal discountPercent;

    // Dates
    @Future(message = "Date must be in the future")
    private LocalDate eventDate;

    @Past(message = "Birth date must be in the past")
    private LocalDate birthDate;

    @FutureOrPresent
    private LocalDateTime scheduledAt;

    // Boolean
    @AssertTrue(message = "Terms must be accepted")
    private boolean termsAccepted;

    // Collections/Arrays
    @NotEmpty(message = "At least one tag required")
    @Size(max = 5, message = "Max 5 tags")
    private List<String> tags;
}
```

### Enabling Validation — @Valid in Controller

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public UserResponse create(@RequestBody @Valid UserCreateRequest request) {
        //                                  ↑
        //               @Valid triggers validation before method runs
        //               If validation fails → MethodArgumentNotValidException thrown
        //               @ControllerAdvice handles it (see Spring Web notes)
        return userService.createUser(request);
    }

    @PutMapping("/{id}")
    public UserResponse update(@PathVariable Long id,
                               @RequestBody @Valid UserUpdateRequest request) {
        return userService.updateUser(id, request);
    }
}
```

### Custom Error Messages

```java
// In DTO
@NotBlank(message = "Username is required")
@Size(min = 3, max = 20, message = "Username must be between {min} and {max} characters")
private String username;

// Validation failure response (handled by @ControllerAdvice):
// {
//   "status": 400,
//   "errors": {
//     "username": "Username is required",
//     "email": "Invalid email format",
//     "age": "Age must be at least 18"
//   }
// }
```

### Better Validation Error Handler

```java
@ControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, Object>> handleValidation(
            MethodArgumentNotValidException ex) {

        Map<String, String> fieldErrors = new HashMap<>();

        ex.getBindingResult().getFieldErrors().forEach(error -> {
            fieldErrors.put(error.getField(), error.getDefaultMessage());
        });

        Map<String, Object> response = new HashMap<>();
        response.put("status", 400);
        response.put("message", "Validation failed");
        response.put("errors", fieldErrors);
        response.put("timestamp", LocalDateTime.now());

        return ResponseEntity.badRequest().body(response);
    }
}

// Response looks like:
// {
//   "status": 400,
//   "message": "Validation failed",
//   "errors": {
//     "username": "Username must be 3-20 characters",
//     "email": "Invalid email format"
//   },
//   "timestamp": "2024-01-15T10:30:00"
// }
```

### Custom Validator — When Built-ins Aren't Enough

```java
// Step 1: Create annotation
@Documented
@Constraint(validatedBy = PasswordStrengthValidator.class)
@Target({ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
public @interface StrongPassword {
    String message() default "Password must be 8+ chars with uppercase and digit";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

// Step 2: Create validator
public class PasswordStrengthValidator implements ConstraintValidator<StrongPassword, String> {

    @Override
    public boolean isValid(String password, ConstraintValidatorContext context) {
        if (password == null) return false;
        return password.length() >= 8
            && password.chars().anyMatch(Character::isUpperCase)
            && password.chars().anyMatch(Character::isDigit);
    }
}

// Step 3: Use it
public class UserCreateRequest {
    @StrongPassword
    private String password;
}
```

### Validating Path Variables and Request Params

```java
@RestController
@RequestMapping("/api/users")
@Validated  // needed for @PathVariable and @RequestParam validation
public class UserController {

    @GetMapping("/{id}")
    public UserResponse getById(@PathVariable @Positive Long id) {
        // @Positive ensures id > 0 — rejects /users/-1 or /users/0
        return userService.getUserById(id);
    }

    @GetMapping
    public List<UserResponse> getAll(
            @RequestParam(defaultValue = "0") @Min(0) int page,
            @RequestParam(defaultValue = "10") @Min(1) @Max(100) int size) {
        return userService.getAll(page, size);
    }
}
```

---

## 🗺️ ModelMapper / MapStruct — Auto Mapping (Optional but Useful)

```java
// Manual mapping gets tedious with many fields
// ModelMapper automates it

@Bean
public ModelMapper modelMapper() {
    return new ModelMapper();
}

@Service
public class UserService {
    private final ModelMapper modelMapper;

    public UserResponse createUser(UserCreateRequest request) {
        User user = modelMapper.map(request, User.class); // DTO → Entity
        user.setPassword(passwordEncoder.encode(user.getPassword()));
        User saved = userRepository.save(user);
        return modelMapper.map(saved, UserResponse.class); // Entity → DTO
    }
}
```

---

## 📐 Complete DTO + Validation Example

```java
// Full UserCreateRequest with all validations
public class UserCreateRequest {

    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 20, message = "Username must be 3-20 characters")
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "Only letters, numbers, underscores")
    private String username;

    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;

    @NotBlank(message = "Email is required")
    @Email(message = "Must be a valid email address")
    private String email;

    @NotNull(message = "Age is required")
    @Min(value = 18, message = "Must be 18 or older")
    @Max(value = 120, message = "Invalid age")
    private Integer age;

    @AssertTrue(message = "You must accept the terms and conditions")
    private boolean termsAccepted;

    // getters and setters
}

// Compact UserResponse
public class UserResponse {
    private Long id;
    private String username;
    private String email;
    private int age;
    private LocalDateTime createdAt;

    public UserResponse() {}

    public UserResponse(User user) {
        this.id = user.getId();
        this.username = user.getUsername();
        this.email = user.getEmail();
        this.age = user.getAge();
        this.createdAt = user.getCreatedAt();
    }
    // getters
}
```

---

## 🎯 Quick Revision Flashcards

| Concept | One-Line Summary |
|---------|----------------|
| DTO | Plain class to carry data — no JPA, no business logic |
| Request DTO | What client sends (create/update input) |
| Response DTO | What client receives (never expose entity) |
| Why DTOs? | Hide DB structure, control what's exposed, decouple layers |
| `@Valid` | Trigger validation on method argument |
| `@NotBlank` | Not null + not empty + not whitespace (best for Strings) |
| `@NotNull` | Field must not be null |
| `@Size` | Min/max length for strings or collections |
| `@Email` | Valid email format |
| `@Min` / `@Max` | Number range |
| `@Pattern` | Regex validation |
| `@Future` / `@Past` | Date must be future/past |
| `@Positive` | Number must be > 0 |
| `@Validated` | Needed on controller class for PathVariable/RequestParam validation |
| `MethodArgumentNotValidException` | Thrown when @Valid fails — handle in @ControllerAdvice |

---

> ✅ **You've mastered Validation & DTOs when:** Every controller method receives a validated DTO and returns a DTO — entities never leave the service layer, passwords never leave the server, and bad input never reaches the database.
