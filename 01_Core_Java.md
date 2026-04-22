# 🧠 1. Core Java — Non-Negotiable Foundation

> **Mantra:** If this is weak, Spring will feel like "magic" — and magic you don't understand will always bite you in production.

---

## 📦 OOP — The 4 Pillars

### 1. Encapsulation — "Hide your data, expose behavior"

Keep fields private. Control access through getters/setters.

```java
public class BankAccount {
    private double balance; // hidden from outside

    public double getBalance() { return balance; } // controlled access

    public void deposit(double amount) {
        if (amount > 0) balance += amount; // validation logic here
    }
}
```

🧠 **Remember:** Like a capsule pill — medicine inside is protected by the outer shell.

---

### 2. Inheritance — "Child gets parent's stuff"

```java
public class Animal {
    public void eat() {
        System.out.println("Animal eats");
    }
}

public class Dog extends Animal {
    public void bark() {
        System.out.println("Dog barks");
    }
}

// Dog gets eat() for free + has its own bark()
Dog d = new Dog();
d.eat();  // ✅ inherited
d.bark(); // ✅ own method
```

🧠 **Remember:** `extends` = "is a" relationship. Dog IS an Animal.

---

### 3. Polymorphism — "One interface, many forms"

**Compile-time (Overloading):**
```java
public class Calculator {
    public int add(int a, int b) { return a + b; }
    public double add(double a, double b) { return a + b; } // same name, diff params
}
```

**Runtime (Overriding):**
```java
public class Animal {
    public void sound() { System.out.println("Generic sound"); }
}

public class Cat extends Animal {
    @Override
    public void sound() { System.out.println("Meow"); } // overrides parent
}

Animal a = new Cat(); // parent ref, child object
a.sound(); // prints "Meow" — runtime decides which method ✨
```

🧠 **Remember:** Parent reference + child object = runtime polymorphism. The actual object type decides which method runs.

---

### 4. Abstraction — "Show what, hide how"

```java
// Abstract class - can have both abstract and concrete methods
public abstract class Shape {
    public abstract double area(); // must be implemented by child

    public void describe() {  // concrete method, shared
        System.out.println("I am a shape with area: " + area());
    }
}

public class Circle extends Shape {
    double r;
    Circle(double r) { this.r = r; }

    @Override
    public double area() { return Math.PI * r * r; }
}

// Interface - pure abstraction (contract)
public interface Flyable {
    void fly(); // abstract by default
    default void land() { System.out.println("Landing..."); } // Java 8+
}
```

🧠 **Remember:**
- Abstract class = partial abstraction (some implementation allowed)
- Interface = full contract (what to do, not how)

---

## 📚 Collections — Your Data Toolbox

### Quick Cheat Sheet

| Type | When to use | Allows Null? | Ordered? | Duplicate? |
|------|------------|-------------|---------|-----------|
| `ArrayList` | Default list | ✅ | ✅ index-ordered | ✅ |
| `LinkedList` | Frequent insert/delete | ✅ | ✅ | ✅ |
| `HashSet` | Unique, fast lookup | 1 null | ❌ | ❌ |
| `LinkedHashSet` | Unique + insertion order | 1 null | ✅ insertion | ❌ |
| `TreeSet` | Sorted unique elements | ❌ | ✅ sorted | ❌ |
| `HashMap` | Key-value, fast | 1 null key | ❌ | ❌ keys |
| `LinkedHashMap` | Key-value + insertion order | ✅ | ✅ insertion | ❌ keys |
| `TreeMap` | Sorted key-value | ❌ | ✅ sorted | ❌ keys |

### List
```java
List<String> names = new ArrayList<>();
names.add("Alice");
names.add("Bob");
names.add("Alice"); // duplicates allowed

names.get(0);      // "Alice"
names.size();      // 3
names.contains("Bob"); // true
names.remove("Bob");

// Iterate
for (String name : names) {
    System.out.println(name);
}

// Streams (Java 8+)
names.stream()
     .filter(n -> n.startsWith("A"))
     .forEach(System.out::println);
```

### Map
```java
Map<String, Integer> scores = new HashMap<>();
scores.put("Alice", 95);
scores.put("Bob", 87);
scores.put("Alice", 99); // overwrites previous value

scores.get("Alice");           // 99
scores.getOrDefault("X", 0);   // 0 (safe get)
scores.containsKey("Bob");     // true

// Iterate entries
for (Map.Entry<String, Integer> entry : scores.entrySet()) {
    System.out.println(entry.getKey() + " = " + entry.getValue());
}

// Java 8 forEach
scores.forEach((k, v) -> System.out.println(k + ": " + v));
```

### Set
```java
Set<String> cities = new HashSet<>();
cities.add("Delhi");
cities.add("Mumbai");
cities.add("Delhi"); // ignored - already exists

cities.size();          // 2
cities.contains("Delhi"); // true

// Set operations
Set<String> a = new HashSet<>(Arrays.asList("1","2","3"));
Set<String> b = new HashSet<>(Arrays.asList("2","3","4"));

a.retainAll(b); // intersection → {2,3}
a.addAll(b);    // union
a.removeAll(b); // difference
```

---

## ⚡ Exception Handling

### The Hierarchy
```
Throwable
├── Error (OutOfMemoryError — don't catch these)
└── Exception
    ├── Checked (IOException, SQLException — must handle)
    └── RuntimeException (NullPointerException, ArrayIndexOutOfBounds — optional)
```

### try-catch-finally
```java
public String readFile(String path) {
    BufferedReader reader = null;
    try {
        reader = new BufferedReader(new FileReader(path));
        return reader.readLine();
    } catch (FileNotFoundException e) {
        System.out.println("File not found: " + e.getMessage());
        return null;
    } catch (IOException e) {
        System.out.println("Read error: " + e.getMessage());
        return null;
    } finally {
        // ALWAYS runs — cleanup here
        if (reader != null) {
            try { reader.close(); } catch (IOException ignored) {}
        }
    }
}
```

### try-with-resources (Java 7+) — Cleaner way
```java
public String readFile(String path) throws IOException {
    try (BufferedReader reader = new BufferedReader(new FileReader(path))) {
        return reader.readLine(); // reader auto-closed ✨
    }
}
```

### Custom Exception
```java
public class InsufficientFundsException extends RuntimeException {
    private double amount;

    public InsufficientFundsException(double amount) {
        super("Insufficient funds. Need: " + amount);
        this.amount = amount;
    }

    public double getAmount() { return amount; }
}

// Usage
public void withdraw(double amount) {
    if (amount > balance) {
        throw new InsufficientFundsException(amount - balance);
    }
    balance -= amount;
}
```

🧠 **Remember:**
- **Checked** = compiler forces you to handle (use for recoverable situations)
- **Unchecked** = runtime, optional handling (use for programmer errors)
- **finally** always runs (even with return statements)

---

## 🔄 Multithreading Basics

### What is a Thread?
A thread = a single path of execution. Your program by default has 1 thread (main). Multithreading = multiple tasks running concurrently.

### Creating Threads
```java
// Way 1: Extend Thread
public class MyThread extends Thread {
    @Override
    public void run() {
        System.out.println("Thread running: " + getName());
    }
}
MyThread t = new MyThread();
t.start(); // DON'T call run() — that runs in same thread!

// Way 2: Implement Runnable (PREFERRED — allows extending other classes)
public class MyTask implements Runnable {
    @Override
    public void run() {
        System.out.println("Task in: " + Thread.currentThread().getName());
    }
}
Thread t = new Thread(new MyTask());
t.start();

// Way 3: Lambda (Java 8+, cleanest)
Thread t = new Thread(() -> System.out.println("Lambda thread!"));
t.start();
```

### synchronized — Preventing Race Conditions
```java
public class Counter {
    private int count = 0;

    // Without synchronized: multiple threads can read-modify-write simultaneously → wrong count
    public synchronized void increment() {
        count++; // only 1 thread can execute this at a time
    }

    public synchronized int getCount() { return count; }
}
```

### Common Thread Methods
```java
Thread t = new Thread(() -> {
    try {
        Thread.sleep(1000); // pause 1 second
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
});

t.start();
t.join();   // wait for this thread to finish before continuing
t.isAlive(); // check if still running
```

🧠 **Why this matters for Spring:**
Spring Boot applications handle multiple HTTP requests simultaneously — each request runs in its own thread. If you have shared state (static variables, singleton beans), you need to think about thread safety!

---

## 🗄️ JDBC — Java Database Connectivity

### Why JDBC Matters
JDBC is the raw layer underneath Spring Data JPA. When Spring says "it connected to DB" — JDBC is doing the actual work. Understanding this makes Spring magic less magical.

### The Flow: Java App → JDBC Driver → Database

```java
import java.sql.*;

public class JDBCDemo {
    // Connection string format: jdbc:databaseType://host:port/dbName
    private static final String URL = "jdbc:mysql://localhost:3306/mydb";
    private static final String USER = "root";
    private static final String PASS = "password";

    public static void main(String[] args) {
        // 1. Get Connection
        try (Connection conn = DriverManager.getConnection(URL, USER, PASS)) {

            // 2. Create Statement (PreparedStatement prevents SQL injection!)
            String sql = "SELECT * FROM users WHERE age > ?";
            PreparedStatement ps = conn.prepareStatement(sql);
            ps.setInt(1, 18); // replace ? with 18

            // 3. Execute Query
            ResultSet rs = ps.executeQuery();

            // 4. Process Results
            while (rs.next()) {
                int id = rs.getInt("id");
                String name = rs.getString("name");
                System.out.println(id + " : " + name);
            }

        } catch (SQLException e) {
            e.printStackTrace();
        }
        // Connection auto-closed (try-with-resources)
    }

    // INSERT example
    public static void insertUser(String name, int age) throws SQLException {
        String sql = "INSERT INTO users (name, age) VALUES (?, ?)";
        try (Connection conn = DriverManager.getConnection(URL, USER, PASS);
             PreparedStatement ps = conn.prepareStatement(sql)) {
            ps.setString(1, name);
            ps.setInt(2, age);
            int rowsAffected = ps.executeUpdate(); // returns rows affected
            System.out.println("Inserted: " + rowsAffected + " row(s)");
        }
    }
}
```

### JDBC vs Spring Data JPA — The Comparison
```
Raw JDBC:
  - Write SQL manually ✅
  - Handle Connection/ResultSet manually ❌
  - Very verbose ❌
  - Full control ✅

Spring JDBC Template:
  - Write SQL manually ✅
  - Connection handling automated ✅
  - Less verbose ✅

Spring Data JPA:
  - SQL auto-generated from method names ✅
  - Everything automated ✅
  - Less control, more magic ⚠️
```

---

## 🎯 Quick Revision Flashcards

| Concept | One-Line Summary |
|---------|----------------|
| Encapsulation | Private fields + public methods |
| Inheritance | `extends` — child IS a parent |
| Polymorphism | Same method name, different behavior |
| Abstraction | Hide complexity, show interface |
| ArrayList vs LinkedList | ArrayList = fast read; LinkedList = fast insert/delete |
| HashMap | O(1) lookup by key |
| Checked Exception | Must handle at compile time |
| Unchecked Exception | Optional handling, caught at runtime |
| `synchronized` | One thread at a time in that method |
| JDBC | Raw Java-to-DB connection API |
| PreparedStatement | Prevents SQL injection, better than Statement |

---

> ✅ **You're ready for Spring Core when:** You can explain DI without knowing Spring exists. Think: "Instead of creating objects yourself, someone gives them to you." That's IoC. That's next.
