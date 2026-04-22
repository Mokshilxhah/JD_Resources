# 🌐 3. Spring Boot Basics — Entry to Real Development

> **Mantra:** Spring Boot removes Spring complexity. It's Spring with smart defaults so you stop configuring and start building.

---

## 🤔 What is Spring Boot?

### The Problem Spring Boot Solves

**Plain Spring App Setup (painful):**
```xml
<!-- pom.xml — you pick and match compatible versions manually -->
<dependency>spring-core 5.3.x</dependency>
<dependency>spring-web 5.3.x</dependency>
<dependency>spring-webmvc 5.3.x</dependency>
<dependency>jackson-databind 2.x.x</dependency>
<dependency>tomcat 9.x.x</dependency>
<!-- + 20 more... and hope versions are compatible -->
```

```java
// Manual configuration — you write this XML or Java config
@Bean
public DispatcherServlet dispatcherServlet() { ... }

@Bean
public ViewResolver viewResolver() { ... }

@Bean
public Jackson2ObjectMapperBuilder jacksonBuilder() { ... }
// ... 50+ lines of boilerplate
```

**With Spring Boot:**
```xml
<!-- ONE dependency replaces all of the above -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- No version needed! Spring Boot manages compatible versions -->
</dependency>
```

```java
@SpringBootApplication // ONE annotation replaces all config
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
        // That's it. App is running on port 8080 ✅
    }
}
```

### Spring Boot's 3 Superpowers

```
1. Auto-configuration   → Detects what you added and configures automatically
2. Starter POMs         → Pre-packaged compatible dependency bundles
3. Embedded Server      → Tomcat bundled inside — no separate server setup
```

---

## ⚙️ Auto-Configuration — The Magic Explained

Auto-config is NOT magic. Here's what actually happens:

```
You add: spring-boot-starter-web to pom.xml
          ↓
Spring Boot sees: spring-webmvc.jar on classpath
          ↓
Thinks: "They want a web app. Let me set up DispatcherServlet, ViewResolver, etc."
          ↓
Creates beans automatically — UNLESS you define your own
```

```java
// @SpringBootApplication is actually 3 annotations combined:
@SpringBootApplication
// = @Configuration        (this class has config)
// + @EnableAutoConfiguration (scan classpath, auto-configure)  
// + @ComponentScan        (scan this package + sub-packages for @Component)

public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

### Overriding Auto-Config

Auto-config steps aside if YOU define a bean:
```java
// Spring Boot auto-creates a DataSource from application.properties
// But if YOU define one, Spring Boot backs off

@Configuration
public class MyConfig {
    @Bean
    public DataSource dataSource() {
        // Your custom DataSource wins over auto-config
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://localhost/mydb");
        return ds;
    }
}
```

---

## 📦 Starter Dependencies — Pre-packaged Bundles

A **starter** = a curated set of compatible dependencies for a feature.

| Starter | What you get |
|---------|-------------|
| `spring-boot-starter-web` | Spring MVC + Tomcat + Jackson |
| `spring-boot-starter-data-jpa` | Spring Data JPA + Hibernate + JDBC |
| `spring-boot-starter-security` | Spring Security (auth/authz) |
| `spring-boot-starter-test` | JUnit + Mockito + AssertJ |
| `spring-boot-starter-validation` | Bean Validation (Hibernate Validator) |
| `spring-boot-starter-actuator` | Health checks, metrics endpoints |
| `spring-boot-starter-mail` | JavaMail for sending emails |

```xml
<!-- pom.xml example -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.2.0</version>
    <!-- This parent manages ALL dependency versions for you -->
</parent>

<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
        <!-- NO version tag — parent handles it -->
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <dependency>
        <groupId>com.mysql</groupId>
        <artifactId>mysql-connector-j</artifactId>
        <scope>runtime</scope>
    </dependency>

    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## 🛠️ Project Setup

### Spring Initializr — Your Project Generator

Go to **https://start.spring.io** and select:

```
Project:   Maven (standard) or Gradle (faster builds)
Language:  Java
Spring Boot: 3.x.x (latest stable)
Packaging: Jar (embedded Tomcat) or War (deploy to external server)
Java:      17 or 21 (LTS versions)

Dependencies:
  + Spring Web
  + Spring Data JPA
  + MySQL Driver
  + Lombok (optional, reduces boilerplate)
```

Download → Extract → Open in IntelliJ IDEA → Done.

### Project Structure

```
my-app/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/myapp/
│   │   │       ├── MyAppApplication.java  ← main class
│   │   │       ├── controller/
│   │   │       │   └── UserController.java
│   │   │       ├── service/
│   │   │       │   └── UserService.java
│   │   │       ├── repository/
│   │   │       │   └── UserRepository.java
│   │   │       └── model/
│   │   │           └── User.java
│   │   └── resources/
│   │       ├── application.properties  ← config here
│   │       └── static/                 ← HTML/CSS/JS
│   └── test/
│       └── java/
│           └── com/example/myapp/
│               └── UserServiceTest.java
├── pom.xml
└── README.md
```

---

## ⚙️ Maven vs Gradle

### Maven (pom.xml)
```xml
<!-- Standard, more verbose, XML-based -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>

<!-- Build -->
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

```bash
# Maven commands
mvn clean install      # clean + compile + test + package
mvn spring-boot:run    # run the app
mvn test               # run tests only
mvn package            # create JAR
```

### Gradle (build.gradle)
```groovy
// Modern, faster, Groovy/Kotlin-based
plugins {
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
    id 'java'
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

```bash
# Gradle commands
./gradlew build        # compile + test + package
./gradlew bootRun      # run the app
./gradlew test         # run tests
./gradlew bootJar      # create JAR
```

🧠 **Which to choose?**
- Maven: More common in enterprise, lots of examples online
- Gradle: Faster builds, modern projects (Android uses Gradle)
- **For interviews/learning: Maven is safer choice**

---

## 📄 Configuration — application.properties vs application.yml

### application.properties (flat key-value)

```properties
# Server
server.port=8080
server.servlet.context-path=/api

# Database
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=secret
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# JPA / Hibernate
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
spring.jpa.database-platform=org.hibernate.dialect.MySQL8Dialect

# Logging
logging.level.org.springframework=INFO
logging.level.com.example=DEBUG

# Custom properties
app.name=My Awesome App
app.version=1.0.0
app.max-users=1000
```

### application.yml (hierarchical, cleaner)

```yaml
# Same config, YAML format
server:
  port: 8080
  servlet:
    context-path: /api

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: root
    password: secret
    driver-class-name: com.mysql.cj.jdbc.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    org.springframework: INFO
    com.example: DEBUG

app:
  name: My Awesome App
  version: 1.0.0
  max-users: 1000
```

### Reading Custom Properties in Code

```java
@Component
public class AppInfo {

    @Value("${app.name}")
    private String appName;

    @Value("${app.max-users:500}") // 500 is default if not found
    private int maxUsers;

    public void print() {
        System.out.println(appName + " allows " + maxUsers + " users");
    }
}

// Better approach — bind to a class (for many related properties)
@ConfigurationProperties(prefix = "app")
@Component
public class AppProperties {
    private String name;
    private String version;
    private int maxUsers;

    // getters + setters (or use Lombok @Data)
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }
    public int getMaxUsers() { return maxUsers; }
    public void setMaxUsers(int maxUsers) { this.maxUsers = maxUsers; }
}
```

---

## 🌍 Profiles — Different Config per Environment

### The Problem
- Dev: H2 in-memory DB, debug logging
- Prod: MySQL, minimal logging, SSL
- You don't want to manually change config every deploy

### Solution: Profiles

```
application.properties          ← shared/default config
application-dev.properties      ← dev-specific (overrides default)
application-prod.properties     ← prod-specific (overrides default)
application-test.properties     ← test-specific
```

```properties
# application.properties (shared)
app.name=My App
server.port=8080

# application-dev.properties
spring.datasource.url=jdbc:h2:mem:testdb
spring.jpa.show-sql=true
logging.level.com.example=DEBUG

# application-prod.properties
spring.datasource.url=jdbc:mysql://prod-server:3306/proddb
spring.datasource.username=${DB_USER}     # from environment variable
spring.datasource.password=${DB_PASS}
spring.jpa.show-sql=false
logging.level.com.example=WARN
```

### YAML Profiles (all in one file)

```yaml
# application.yml
spring:
  application:
    name: My App

---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:testdb
  jpa:
    show-sql: true

---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:mysql://prod-server/proddb
```

### Activating Profiles

```properties
# application.properties — set active profile
spring.profiles.active=dev
```

```bash
# Command line — overrides everything
java -jar myapp.jar --spring.profiles.active=prod

# Or as environment variable
SPRING_PROFILES_ACTIVE=prod java -jar myapp.jar
```

```java
// Profile-specific beans
@Configuration
public class DataSourceConfig {

    @Bean
    @Profile("dev")
    public DataSource devDataSource() {
        return new EmbeddedDatabaseBuilder()
            .setType(EmbeddedDatabaseType.H2)
            .build();
    }

    @Bean
    @Profile("prod")
    public DataSource prodDataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:mysql://prod-server/mydb");
        return ds;
    }
}
```

---

## 🖥️ Embedded Server — Tomcat Inside Spring Boot

### Traditional vs Spring Boot

```
Traditional Spring:
  1. Install Tomcat on server
  2. Build your app as WAR file
  3. Copy WAR into Tomcat's webapps folder
  4. Start Tomcat
  → App running (but setup is painful)

Spring Boot:
  1. Build your app as JAR file
  2. java -jar myapp.jar
  → App running on embedded Tomcat ✅
```

### How It Works

```
spring-boot-starter-web
    └── includes: tomcat-embed-core.jar
                  (Tomcat packaged AS a library)

When you run SpringApplication.run():
  1. Creates ApplicationContext
  2. Detects web app (spring-webmvc on classpath)
  3. Creates embedded Tomcat instance
  4. Configures DispatcherServlet
  5. Starts Tomcat on port 8080
  6. Your app is live!
```

### Configuring the Server

```properties
# Change port
server.port=9090

# Context path (all URLs prefixed with /api)
server.servlet.context-path=/api

# HTTPS
server.ssl.key-store=classpath:keystore.p12
server.ssl.key-store-password=secret

# Timeouts
server.tomcat.connection-timeout=5000
server.tomcat.max-threads=200
```

### Switching from Tomcat to Jetty

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <!-- Remove embedded Tomcat -->
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- Add Jetty instead -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

---

## 🚀 Your First Spring Boot REST App (Full Example)

```java
// Entity
@Entity
@Table(name = "users")
public class User {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;
    // constructors, getters, setters
}

// Repository
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    List<User> findByName(String name); // Spring generates query automatically!
}

// Service
@Service
public class UserService {
    private final UserRepository userRepository;

    public UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    public List<User> getAllUsers() { return userRepository.findAll(); }
    public User getById(Long id) { return userRepository.findById(id).orElseThrow(); }
    public User save(User user) { return userRepository.save(user); }
    public void delete(Long id) { userRepository.deleteById(id); }
}

// Controller
@RestController
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping
    public List<User> getAll() { return userService.getAllUsers(); }

    @GetMapping("/{id}")
    public User getById(@PathVariable Long id) { return userService.getById(id); }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public User create(@RequestBody User user) { return userService.save(user); }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable Long id) { userService.delete(id); }
}

// Main App
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        SpringApplication.run(MyApp.class, args);
    }
}
```

```properties
# application.properties
server.port=8080
spring.datasource.url=jdbc:mysql://localhost:3306/mydb
spring.datasource.username=root
spring.datasource.password=root
spring.jpa.hibernate.ddl-auto=update
```

```bash
# Test your API
curl http://localhost:8080/api/users
curl -X POST http://localhost:8080/api/users \
     -H "Content-Type: application/json" \
     -d '{"name":"Alice","email":"alice@example.com"}'
```

---

## 🎯 Quick Revision Flashcards

| Concept | One-Line Summary |
|---------|----------------|
| Spring Boot | Spring + smart defaults + embedded server |
| Auto-configuration | Detects classpath, configures beans automatically |
| spring-boot-starter | Bundle of compatible dependencies for a feature |
| Starter Parent | Manages all dependency versions for you |
| application.properties | Flat key=value config file |
| application.yml | Hierarchical config (same purpose, cleaner) |
| Profiles | Environment-specific config (dev/prod/test) |
| @Value | Inject single property from config |
| @ConfigurationProperties | Bind group of properties to a class |
| Embedded Tomcat | Tomcat packaged inside your JAR — `java -jar` just works |
| @SpringBootApplication | = @Configuration + @EnableAutoConfiguration + @ComponentScan |
| spring.profiles.active | Switch active profile |
| ddl-auto=update | Hibernate auto-creates/updates DB tables |
| ddl-auto=validate | Just check schema, don't change (for prod) |

---

## 🔗 The 3 Files Every Spring Boot App Has

```
1. MyAppApplication.java       → Entry point, @SpringBootApplication
2. application.properties      → All configuration
3. pom.xml                     → All dependencies
```

Everything else is YOUR business logic on top.

---

> ✅ **You've mastered Spring Boot Basics when:** You can create a project from Spring Initializr, write a REST controller, connect it to a DB via application.properties, and run it with `java -jar`. That's real development. Everything else is just more endpoints.
