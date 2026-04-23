# 🗄️ 5. Spring Data JPA — Database Mastery

> **Mantra:** Without this, your project is incomplete. JPA is how Spring talks to databases — and it's way more powerful than raw JDBC.

---

## 🤔 What is JPA and Why?

```
Raw JDBC (painful):
  - Write SQL manually
  - Map ResultSet rows to objects manually
  - Handle connections manually
  - 50 lines to do a simple SELECT

JPA (Java Persistence API):
  - Map Java class ↔ Database table automatically
  - Write Java method calls instead of SQL
  - Spring Data JPA = JPA + Spring magic
  - 1 line to do the same SELECT

Hibernate:
  - The actual implementation of JPA (the engine under the hood)
  - Spring Data JPA uses Hibernate internally
```

🧠 **The Stack:**
```
Your Code
    ↓
Spring Data JPA  ← (the layer you use)
    ↓
JPA (specification)
    ↓
Hibernate  ← (actual implementation, generates SQL)
    ↓
JDBC
    ↓
Database
```

---

## 🏷️ JPA Basics — Mapping Classes to Tables

### @Entity — "This class is a database table"

```java
import jakarta.persistence.*;

@Entity                        // maps this class to a DB table
@Table(name = "users")         // table name (optional — defaults to class name)
public class User {

    @Id                                                    // primary key
    @GeneratedValue(strategy = GenerationType.IDENTITY)   // auto-increment
    private Long id;

    @Column(name = "full_name", nullable = false, length = 100)
    private String name;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    // Transient = NOT stored in DB
    @Transient
    private String tempToken;

    // JPA requires a no-arg constructor
    protected User() {}

    public User(String name, String email) {
        this.name = name;
        this.email = email;
        this.createdAt = LocalDateTime.now();
    }

    // getters and setters...
}
```

### @Column Attributes — Quick Reference

```java
@Column(
    name = "email",         // column name in DB (default = field name)
    nullable = false,       // NOT NULL constraint
    unique = true,          // UNIQUE constraint
    length = 255,           // VARCHAR length (for String)
    precision = 10,         // total digits (for BigDecimal)
    scale = 2,              // decimal places (for BigDecimal)
    updatable = false,      // never update this column after insert
    insertable = true       // include in INSERT
)
private String email;
```

### GenerationType Strategies

```java
@GeneratedValue(strategy = GenerationType.IDENTITY)  // DB auto-increment (MySQL default ✅)
@GeneratedValue(strategy = GenerationType.SEQUENCE)  // DB sequence (PostgreSQL default)
@GeneratedValue(strategy = GenerationType.AUTO)      // JPA picks for you
@GeneratedValue(strategy = GenerationType.UUID)      // Random UUID (Java 17+ / Spring Boot 3+)
```

---

## 🔗 Relationships — Connecting Tables

### OneToOne — One user has one profile

```java
@Entity
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToOne(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private UserProfile profile;
}

@Entity
public class UserProfile {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String bio;
    private String avatarUrl;

    @OneToOne
    @JoinColumn(name = "user_id") // foreign key column in this table
    private User user;            // this side "owns" the relationship
}
```

### OneToMany / ManyToOne — One department has many employees

```java
@Entity
public class Department {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    // ONE Department → MANY Employees
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<Employee> employees = new ArrayList<>();
}

@Entity
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    // MANY Employees → ONE Department
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id") // FK column in employees table
    private Department department;
}
```

### ManyToMany — Students and Courses

```java
@Entity
public class Student {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @ManyToMany
    @JoinTable(
        name = "student_courses",           // join table name
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id")
    )
    private Set<Course> courses = new HashSet<>();
}

@Entity
public class Course {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String title;

    @ManyToMany(mappedBy = "courses") // mappedBy = other side owns it
    private Set<Student> students = new HashSet<>();
}
```

### Cascade Types — "What happens to children when parent changes"

```java
@OneToMany(cascade = CascadeType.ALL)     // save/delete/merge all — most common
@OneToMany(cascade = CascadeType.PERSIST) // only save cascades
@OneToMany(cascade = CascadeType.REMOVE)  // only delete cascades
@OneToMany(cascade = CascadeType.MERGE)   // only merge cascades

// Example: When you save a Department, its employees are also saved
department.getEmployees().add(new Employee("Alice"));
departmentRepository.save(department); // Alice is also saved! (CascadeType.PERSIST/ALL)
```

### mappedBy — Who Owns the Relationship

```
The side with @JoinColumn = "OWNER" (has the FK column in DB)
The side with mappedBy    = "INVERSE" (no FK column, just a reference)

Employee has @JoinColumn(name="dept_id")  → Employee owns the relationship
Department has mappedBy = "department"    → Department is the inverse side

Rule: Always set from OWNER side for changes to persist!
```

---

## 📁 Repositories — Your Database Operations

### JpaRepository — Everything You Need

```java
// That's it! Just extend JpaRepository<EntityType, IDType>
// Spring auto-generates all CRUD operations ✨

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    // You get these for FREE:
    // save(User) — insert or update
    // findById(Long) — find by PK → returns Optional<User>
    // findAll() — all records
    // findAll(Pageable) — paginated
    // deleteById(Long) — delete by PK
    // existsById(Long) — check existence
    // count() — count all records
}
```

### Query Methods — Magic Method Names

Spring reads the method name and generates the SQL automatically!

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    // findBy + FieldName
    List<User> findByName(String name);
    // → SELECT * FROM users WHERE name = ?

    Optional<User> findByEmail(String email);
    // → SELECT * FROM users WHERE email = ?

    // And / Or conditions
    List<User> findByNameAndEmail(String name, String email);
    // → SELECT * FROM users WHERE name = ? AND email = ?

    List<User> findByNameOrEmail(String name, String email);
    // → SELECT * FROM users WHERE name = ? OR email = ?

    // Comparison operators
    List<User> findByAgeGreaterThan(int age);
    // → SELECT * FROM users WHERE age > ?

    List<User> findByAgeBetween(int min, int max);
    // → SELECT * FROM users WHERE age BETWEEN ? AND ?

    // Like / Contains
    List<User> findByNameContaining(String keyword);
    // → SELECT * FROM users WHERE name LIKE '%keyword%'

    List<User> findByNameStartingWith(String prefix);
    // → SELECT * FROM users WHERE name LIKE 'prefix%'

    // Null checks
    List<User> findByEmailIsNull();
    List<User> findByEmailIsNotNull();

    // Boolean
    List<User> findByActiveTrue();
    List<User> findByActiveFalse();

    // Order by
    List<User> findByDepartmentOrderByNameAsc(Department dept);
    // → SELECT * FROM users WHERE department = ? ORDER BY name ASC

    // Count / Exists
    long countByDepartment(Department dept);
    boolean existsByEmail(String email);

    // Delete
    void deleteByEmail(String email);
}
```

### @Query — Write Your Own JPQL or SQL

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {

    // JPQL (uses class names, not table names)
    @Query("SELECT u FROM User u WHERE u.email = :email")
    Optional<User> findByEmailJpql(@Param("email") String email);

    // JPQL with join
    @Query("SELECT u FROM User u JOIN u.department d WHERE d.name = :deptName")
    List<User> findByDepartmentName(@Param("deptName") String deptName);

    // Native SQL (when JPQL can't do it)
    @Query(value = "SELECT * FROM users WHERE YEAR(created_at) = :year",
           nativeQuery = true)
    List<User> findByCreatedYear(@Param("year") int year);

    // Modifying query (UPDATE/DELETE) — needs @Modifying + @Transactional
    @Modifying
    @Transactional
    @Query("UPDATE User u SET u.active = false WHERE u.lastLogin < :date")
    int deactivateOldUsers(@Param("date") LocalDateTime date);
}
```

### Pagination and Sorting

```java
// Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Page<User> findByDepartment(Department dept, Pageable pageable);
}

// Service
public Page<User> getUsers(int page, int size, String sortBy) {
    Pageable pageable = PageRequest.of(
        page,               // page number (0-based)
        size,               // page size
        Sort.by(sortBy)     // sort column
    );
    return userRepository.findAll(pageable);
}

// Controller
@GetMapping
public Page<User> getUsers(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size,
    @RequestParam(defaultValue = "id") String sortBy) {
    return userService.getUsers(page, size, sortBy);
}

// Response includes:
// content: [array of users]
// totalElements: 100
// totalPages: 10
// number: 0 (current page)
```

---

## ⚡ Hibernate Concepts — The Engine Details

### Lazy vs Eager Loading — VERY IMPORTANT

```java
@Entity
public class Department {
    @Id
    private Long id;
    private String name;

    // LAZY (default for @OneToMany, @ManyToMany)
    // Employees are NOT loaded until you call getEmployees()
    @OneToMany(mappedBy = "department", fetch = FetchType.LAZY)
    private List<Employee> employees;

    // EAGER (default for @ManyToOne, @OneToOne)
    // Manager IS loaded immediately when Department is loaded
    @ManyToOne(fetch = FetchType.EAGER)
    private Employee manager;
}
```

```java
// LAZY in action:
Department dept = departmentRepository.findById(1L).get();
// SQL: SELECT * FROM departments WHERE id = 1
// employees NOT fetched yet

dept.getEmployees(); // ← SQL fires HERE (N+1 problem!)
// SQL: SELECT * FROM employees WHERE dept_id = 1
```

### The N+1 Problem — Must Know!

```java
// PROBLEM: You load 10 departments
List<Department> depts = departmentRepository.findAll();
// SQL: SELECT * FROM departments  (1 query)

for (Department d : depts) {
    System.out.println(d.getEmployees()); // 10 separate queries!
    // SQL: SELECT * FROM employees WHERE dept_id = 1
    // SQL: SELECT * FROM employees WHERE dept_id = 2
    // ... 10 more queries = N+1 = 11 queries total ❌
}

// SOLUTION: Use JOIN FETCH in JPQL
@Query("SELECT d FROM Department d LEFT JOIN FETCH d.employees")
List<Department> findAllWithEmployees();
// SQL: SELECT * FROM departments LEFT JOIN employees ON ... (1 query ✅)
```

### First-Level Cache (Session Cache)

```java
// Within same transaction, Hibernate caches — same ID = same object
@Transactional
public void example() {
    User u1 = userRepository.findById(1L).get(); // SQL fires
    User u2 = userRepository.findById(1L).get(); // NO SQL — from cache!
    System.out.println(u1 == u2); // true — same object!
}
```

---

## 🔁 @Transactional — Database Transactions

### What is a Transaction?

```
A transaction = a group of DB operations that succeed or fail TOGETHER.

Example: Bank transfer
  1. Deduct ₹1000 from Account A
  2. Add ₹1000 to Account B

If step 1 succeeds but step 2 fails:
  Without @Transactional → ₹1000 vanishes! 💸
  With @Transactional    → Both rolled back → no money lost ✅
```

### Basic @Transactional Usage

```java
@Service
public class BankService {

    private final AccountRepository accountRepository;

    public BankService(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    @Transactional // if ANYTHING fails, ALL DB changes are rolled back
    public void transfer(Long fromId, Long toId, BigDecimal amount) {
        Account from = accountRepository.findById(fromId)
            .orElseThrow(() -> new ResourceNotFoundException("Source account not found"));

        Account to = accountRepository.findById(toId)
            .orElseThrow(() -> new ResourceNotFoundException("Target account not found"));

        if (from.getBalance().compareTo(amount) < 0) {
            throw new InsufficientFundsException("Not enough balance");
        }

        from.setBalance(from.getBalance().subtract(amount));
        to.setBalance(to.getBalance().add(amount));

        accountRepository.save(from); // both must succeed
        accountRepository.save(to);   // or both roll back
    }
}
```

### @Transactional Attributes

```java
// readOnly = true → optimization for read-only methods (no dirty check)
@Transactional(readOnly = true)
public List<User> getAllUsers() {
    return userRepository.findAll();
}

// rollbackFor → also rollback for checked exceptions
@Transactional(rollbackFor = Exception.class)
public void riskyOperation() throws Exception { ... }

// propagation → how transaction behaves when called from another transaction
@Transactional(propagation = Propagation.REQUIRED)      // default: join existing or create
@Transactional(propagation = Propagation.REQUIRES_NEW)  // always new transaction
@Transactional(propagation = Propagation.SUPPORTS)      // use if exists, else no transaction

// timeout → rollback if takes too long
@Transactional(timeout = 30) // 30 seconds
```

### Where to Put @Transactional

```java
// ✅ On Service methods — the right layer
@Service
public class UserService {
    @Transactional
    public User createUser(User user) { ... }
}

// ❌ On Controller — wrong layer, controllers shouldn't deal with transactions
// ❌ On Repository — Spring Data already wraps each operation in a transaction
// ❌ On private methods — @Transactional doesn't work on private methods!
```

---

## 🏗️ Complete JPA Example

```java
// Entity
@Entity
@Table(name = "orders")
public class Order {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String status;

    private BigDecimal total;

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id")
    private User user;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items = new ArrayList<>();

    @PrePersist  // runs before INSERT
    protected void onCreate() {
        createdAt = LocalDateTime.now();
        status = "PENDING";
    }
}

// Repository
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findByUserAndStatus(User user, String status);
    List<Order> findByCreatedAtBetween(LocalDateTime start, LocalDateTime end);

    @Query("SELECT o FROM Order o WHERE o.total > :amount ORDER BY o.total DESC")
    List<Order> findExpensiveOrders(@Param("amount") BigDecimal amount);
}

// Service
@Service
public class OrderService {
    private final OrderRepository orderRepository;
    private final UserRepository userRepository;

    public OrderService(OrderRepository orderRepository, UserRepository userRepository) {
        this.orderRepository = orderRepository;
        this.userRepository = userRepository;
    }

    @Transactional
    public Order createOrder(Long userId, OrderRequest request) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new ResourceNotFoundException("User not found"));

        Order order = new Order();
        order.setUser(user);
        order.setTotal(request.calculateTotal());
        return orderRepository.save(order);
    }

    @Transactional(readOnly = true)
    public List<Order> getUserOrders(Long userId) {
        User user = userRepository.findById(userId)
            .orElseThrow(() -> new ResourceNotFoundException("User not found"));
        return orderRepository.findByUserAndStatus(user, "ACTIVE");
    }
}
```

---

## 🎯 Quick Revision Flashcards

| Concept | One-Line Summary |
|---------|----------------|
| `@Entity` | Map Java class to DB table |
| `@Id` | Primary key field |
| `@GeneratedValue(IDENTITY)` | Auto-increment PK |
| `@Column` | Customize column mapping |
| `@OneToMany` | One parent → many children |
| `@ManyToOne` | Many children → one parent |
| `@ManyToMany` | Junction table relationship |
| `mappedBy` | Non-owning side (no FK) |
| `@JoinColumn` | Owning side, has FK column |
| `CascadeType.ALL` | Save/delete parent → cascades to children |
| `FetchType.LAZY` | Load related data only when accessed (default for collections) |
| `FetchType.EAGER` | Load related data immediately (default for single entities) |
| N+1 Problem | 1 query loads N records → N more queries for related data |
| JOIN FETCH | Fix N+1 — load all in single query |
| `JpaRepository` | Extends to get free CRUD operations |
| Query methods | `findByEmail` — Spring generates SQL from method name |
| `@Query` | Custom JPQL or native SQL |
| `@Transactional` | All-or-nothing DB operations |
| `@Transactional(readOnly=true)` | Performance optimization for reads |
| `@PrePersist` | Run code before entity is inserted |
| `@PreUpdate` | Run code before entity is updated |

---

> ✅ **You've mastered Spring Data JPA when:** You can define entities with relationships, write a repository with query methods, handle transactions correctly, and explain why N+1 is a problem and how to fix it.
