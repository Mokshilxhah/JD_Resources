# 🔐 6. Spring Security — Basic Level Required

> **Mantra:** Almost every real-world project uses JWT. You don't need to be a security expert, but you MUST understand Auth + JWT cold.

---

## 🤔 Authentication vs Authorization — The Crucial Difference

```
Authentication = WHO are you?
  → Are you logged in? Is your username/password correct?
  → "Proving your identity"
  → Login process

Authorization = WHAT can you do?
  → Are you allowed to access THIS resource?
  → "Checking your permissions"
  → After login, checking roles/permissions

Example:
  You enter a company building with your ID card        → Authentication ✅
  You can enter the office floor but not the server room → Authorization ✅
```

---

## 🏗️ How Spring Security Works (The Big Picture)

```
HTTP Request
     ↓
Security Filter Chain (runs before your Controller)
     ↓
UsernamePasswordAuthenticationFilter  → checks credentials
     ↓
JwtAuthenticationFilter (custom)      → validates JWT token
     ↓
SecurityContextHolder                 → stores who is logged in
     ↓
Your Controller runs
```

Every request passes through a **chain of filters**. You configure which URLs need auth and which don't.

---

## 🔑 JWT Authentication — Token-Based Login

### What is JWT?

```
JWT = JSON Web Token

Structure: header.payload.signature
  header:    {"alg":"HS256","typ":"JWT"}          → base64 encoded
  payload:   {"sub":"alice","role":"USER","exp":...} → base64 encoded
  signature: HMAC-SHA256(header+payload, secret)  → verifies it's not tampered

Full token looks like:
  eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJhbGljZSJ9.abc123xyz

Why JWT?
  ✅ Stateless — server doesn't store sessions
  ✅ Scalable — any server can validate (just needs the secret)
  ✅ Self-contained — carries user info inside
  ❌ Can't invalidate before expiry (unless using blacklist)
```

### JWT Flow

```
1. User sends: POST /auth/login  {"username":"alice","password":"pass123"}
2. Server validates credentials against DB
3. Server creates JWT token (signed with secret key)
4. Server returns: {"token": "eyJhbGc..."}
5. Client stores token (localStorage or cookie)

For every subsequent request:
6. Client sends: Authorization: Bearer eyJhbGc...
7. Server validates signature + checks expiry
8. Server extracts username from token
9. Request proceeds to Controller
```

---

## 📦 Dependencies

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- JWT library -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.12.3</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.12.3</version>
    <scope>runtime</scope>
</dependency>
```

---

## 🔧 Implementation — Step by Step

### Step 1: User Entity with Roles

```java
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String username;

    @Column(nullable = false)
    private String password; // stored as BCrypt hash, NEVER plain text

    private String role; // "ROLE_USER", "ROLE_ADMIN"

    // getters and setters
}
```

### Step 2: UserDetailsService — How Spring Loads User

```java
@Service
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    public CustomUserDetailsService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));

        // Convert your User entity to Spring's UserDetails
        return org.springframework.security.core.userdetails.User
            .withUsername(user.getUsername())
            .password(user.getPassword())  // already BCrypt encoded
            .roles(user.getRole().replace("ROLE_", "")) // "USER" or "ADMIN"
            .build();
    }
}
```

### Step 3: JWT Utility — Create & Validate Tokens

```java
@Component
public class JwtUtil {

    @Value("${app.jwt.secret}")  // from application.properties
    private String secretKey;

    @Value("${app.jwt.expiration:86400000}") // 24 hours in ms
    private long expiration;

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(secretKey.getBytes(StandardCharsets.UTF_8));
    }

    // Create token from username
    public String generateToken(String username) {
        return Jwts.builder()
            .subject(username)
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expiration))
            .signWith(getSigningKey())
            .compact();
    }

    // Extract username from token
    public String extractUsername(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload()
            .getSubject();
    }

    // Is token still valid?
    public boolean isTokenValid(String token) {
        try {
            Jwts.parser()
                .verifyWith(getSigningKey())
                .build()
                .parseSignedClaims(token); // throws if expired or invalid
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
}
```

### Step 4: JWT Filter — Intercept Every Request

```java
@Component
public class JwtAuthFilter extends OncePerRequestFilter {
    // OncePerRequestFilter = runs exactly once per request

    private final JwtUtil jwtUtil;
    private final CustomUserDetailsService userDetailsService;

    public JwtAuthFilter(JwtUtil jwtUtil, CustomUserDetailsService userDetailsService) {
        this.jwtUtil = jwtUtil;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(
        HttpServletRequest request,
        HttpServletResponse response,
        FilterChain filterChain) throws ServletException, IOException {

        // 1. Get Authorization header
        String authHeader = request.getHeader("Authorization");

        // 2. Check if it's a Bearer token
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            filterChain.doFilter(request, response); // no token, skip
            return;
        }

        // 3. Extract token (remove "Bearer " prefix)
        String token = authHeader.substring(7);

        // 4. Validate and set authentication
        if (jwtUtil.isTokenValid(token)) {
            String username = jwtUtil.extractUsername(token);
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);

            // 5. Tell Spring Security "this user is authenticated"
            UsernamePasswordAuthenticationToken authToken =
                new UsernamePasswordAuthenticationToken(
                    userDetails, null, userDetails.getAuthorities());

            SecurityContextHolder.getContext().setAuthentication(authToken);
        }

        // 6. Continue to next filter / controller
        filterChain.doFilter(request, response);
    }
}
```

### Step 5: Security Configuration

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    private final JwtAuthFilter jwtAuthFilter;
    private final CustomUserDetailsService userDetailsService;

    public SecurityConfig(JwtAuthFilter jwtAuthFilter, CustomUserDetailsService userDetailsService) {
        this.jwtAuthFilter = jwtAuthFilter;
        this.userDetailsService = userDetailsService;
    }

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())  // disable CSRF for REST APIs (stateless)
            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS)) // no sessions!
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()    // login/register — open
                .requestMatchers("/api/admin/**").hasRole("ADMIN")  // admin only
                .requestMatchers(HttpMethod.GET, "/api/products/**").permitAll() // read products — open
                .anyRequest().authenticated()                   // everything else — needs login
            )
            .addFilterBefore(jwtAuthFilter,                     // add our filter
                UsernamePasswordAuthenticationFilter.class);    // before default filter

        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(); // industry standard
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config)
        throws Exception {
        return config.getAuthenticationManager();
    }
}
```

### Step 6: Auth Controller — Login & Register

```java
// DTOs
public class LoginRequest {
    private String username;
    private String password;
    // getters
}

public class AuthResponse {
    private String token;
    public AuthResponse(String token) { this.token = token; }
    // getter
}

public class RegisterRequest {
    private String username;
    private String password;
    private String email;
    // getters
}

@RestController
@RequestMapping("/api/auth")
public class AuthController {

    private final AuthenticationManager authenticationManager;
    private final JwtUtil jwtUtil;
    private final PasswordEncoder passwordEncoder;
    private final UserRepository userRepository;

    public AuthController(AuthenticationManager authenticationManager,
                          JwtUtil jwtUtil,
                          PasswordEncoder passwordEncoder,
                          UserRepository userRepository) {
        this.authenticationManager = authenticationManager;
        this.jwtUtil = jwtUtil;
        this.passwordEncoder = passwordEncoder;
        this.userRepository = userRepository;
    }

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody LoginRequest request) {
        // 1. Authenticate (throws exception if wrong credentials)
        authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.getUsername(), request.getPassword())
        );

        // 2. Generate token
        String token = jwtUtil.generateToken(request.getUsername());

        return ResponseEntity.ok(new AuthResponse(token));
    }

    @PostMapping("/register")
    @ResponseStatus(HttpStatus.CREATED)
    public void register(@RequestBody RegisterRequest request) {
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new DuplicateResourceException("Username already taken");
        }

        User user = new User();
        user.setUsername(request.getUsername());
        user.setPassword(passwordEncoder.encode(request.getPassword())); // ALWAYS encode!
        user.setRole("ROLE_USER");
        userRepository.save(user);
    }
}
```

### application.properties for JWT

```properties
# Strong secret — at least 32 characters!
app.jwt.secret=myVerySecretKeyForJWTSigningThatIsLongEnough123
app.jwt.expiration=86400000
# 86400000 ms = 24 hours
```

---

## 🔒 Password Encoding — BCrypt

### Why BCrypt?

```
Plain text: "password123"
MD5:        "482c811da5d5b4bc6d497ffa98491e38"  (fast = hackable in seconds)
BCrypt:     "$2a$10$N9qo8u..."                   (slow by design = hard to brute-force)

BCrypt features:
  ✅ Adds random "salt" — same password = different hash every time
  ✅ Work factor (rounds) — you control how slow it is
  ✅ One-way — can't reverse to get original password
  ✅ Industry standard for passwords
```

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(); // default strength = 10 rounds
    // return new BCryptPasswordEncoder(12); // stronger, slower
}

// Usage:
String rawPassword = "myPassword123";
String encoded = passwordEncoder.encode(rawPassword);
// "$2a$10$randomsalthere..."

// Verify (never decode — just match)
boolean matches = passwordEncoder.matches(rawPassword, encoded); // true ✅

// NEVER store plain text passwords!
// NEVER compare passwords directly!
// Always use .matches()
```

---

## 🛡️ Protecting Endpoints — Method Level Security

```java
// Enable in config
@Configuration
@EnableMethodSecurity
public class SecurityConfig { ... }

// Use in Service or Controller
@Service
public class UserService {

    @PreAuthorize("hasRole('ADMIN')")
    public void deleteUser(Long id) {
        userRepository.deleteById(id);
    }

    @PreAuthorize("hasRole('ADMIN') or #username == authentication.name")
    public User getUserProfile(String username) {
        // Admin can access any profile
        // Regular user can only access their own
        return userRepository.findByUsername(username).orElseThrow();
    }

    @PostAuthorize("returnObject.username == authentication.name")
    public User getMyProfile() {
        // Validates AFTER method runs — returned object must belong to caller
        return userRepository.findByUsername(getCurrentUsername()).orElseThrow();
    }
}
```

### Get Current User Anywhere

```java
// Get currently logged-in username from Security Context
public String getCurrentUsername() {
    return SecurityContextHolder.getContext()
        .getAuthentication()
        .getName();
}

// Or in controller via @AuthenticationPrincipal
@GetMapping("/me")
public User getMe(@AuthenticationPrincipal UserDetails userDetails) {
    return userService.findByUsername(userDetails.getUsername());
}
```

---

## 🎯 Quick Revision Flashcards

| Concept | One-Line Summary |
|---------|----------------|
| Authentication | Prove who you are (login) |
| Authorization | Check what you can do (roles/permissions) |
| JWT | Signed token carrying user info — stateless auth |
| JWT Structure | header.payload.signature — all base64 encoded |
| `UserDetailsService` | Tells Spring how to load a user from DB |
| `JwtAuthFilter` | Intercepts every request, validates JWT |
| `SecurityFilterChain` | Configure which URLs need auth |
| `permitAll()` | Anyone can access |
| `authenticated()` | Must be logged in |
| `hasRole("ADMIN")` | Must have ADMIN role |
| `SessionCreationPolicy.STATELESS` | No server-side sessions (JWT apps) |
| BCrypt | Password hashing — one-way, salted, slow |
| `passwordEncoder.encode()` | Hash a password for storage |
| `passwordEncoder.matches()` | Verify entered password against hash |
| `SecurityContextHolder` | Stores currently authenticated user |
| `@PreAuthorize` | Method-level security check before execution |

---

> ✅ **You've mastered Spring Security basics when:** You can implement JWT login, protect endpoints by role, hash passwords with BCrypt, and explain why JWT is stateless — and what that means for scalability.
