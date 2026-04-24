# 📦 9. Build Tools — Maven & Gradle

> **Mantra:** Build tools are what turn your source code into a runnable application. You can't work on any real project without understanding this.

---

## 🤔 What Does a Build Tool Actually Do?

```
Without a build tool, you'd manually:
  1. Download every library JAR from the internet
  2. Put them in the right folders
  3. Compile every .java file in the right order
  4. Run tests
  5. Package everything into a JAR/WAR
  6. Manage 50 library versions that must be compatible

Build tool does ALL of this automatically.
```

```
Source code (.java)
      ↓
Build Tool (Maven / Gradle)
  → Downloads dependencies
  → Compiles .java → .class
  → Runs tests
  → Packages → .jar or .war
      ↓
Deployable artifact (myapp.jar)
```

---

## 🏗️ Maven — The Standard

### Project Structure (Maven Standard)

```
my-project/
├── pom.xml                   ← Maven config (Project Object Model)
├── src/
│   ├── main/
│   │   ├── java/             ← Your source code
│   │   │   └── com/example/
│   │   │       └── App.java
│   │   └── resources/        ← Config files
│   │       └── application.properties
│   └── test/
│       ├── java/             ← Test code
│       └── resources/        ← Test config
└── target/                   ← Build output (auto-generated, don't commit!)
    └── myapp-1.0.jar
```

### pom.xml — The Heart of Maven

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0">
    <modelVersion>4.0.0</modelVersion>

    <!-- Your project identity -->
    <groupId>com.example</groupId>       <!-- your company/org name -->
    <artifactId>my-spring-app</artifactId> <!-- your project name -->
    <version>1.0.0</version>             <!-- your version -->
    <packaging>jar</packaging>           <!-- jar (default) or war -->

    <!-- Spring Boot parent — manages all dependency versions for you -->
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
    </parent>

    <!-- Java version -->
    <properties>
        <java.version>17</java.version>
    </properties>

    <!-- Dependencies -->
    <dependencies>

        <!-- Spring Web (no version needed — parent manages it) -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>

        <!-- Spring Data JPA -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>

        <!-- Spring Security -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>

        <!-- MySQL Driver — only needed at runtime, not compile time -->
        <dependency>
            <groupId>com.mysql</groupId>
            <artifactId>mysql-connector-j</artifactId>
            <scope>runtime</scope>
        </dependency>

        <!-- H2 — in-memory DB for testing only -->
        <dependency>
            <groupId>com.h2database</groupId>
            <artifactId>h2</artifactId>
            <scope>test</scope>  <!-- only available during testing -->
        </dependency>

        <!-- Lombok — compile-time only, not in final jar -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <scope>provided</scope>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>

    </dependencies>

    <!-- Build plugins -->
    <build>
        <plugins>
            <!-- Spring Boot plugin — lets you run with mvn spring-boot:run -->
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <!-- Excludes Lombok from final JAR -->
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>

</project>
```

### Dependency Scopes — Critical to Understand

```xml
<!-- compile (default) — available everywhere: main code, tests, final jar -->
<dependency>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- no scope = compile -->
</dependency>

<!-- test — ONLY during testing, not in final jar -->
<dependency>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>

<!-- runtime — not needed for compilation, but needed at runtime -->
<dependency>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
    <!-- You code against JDBC API, driver only needed when running -->
</dependency>

<!-- provided — needed for compile, but provided by the container (Tomcat, etc.) -->
<dependency>
    <artifactId>jakarta.servlet-api</artifactId>
    <scope>provided</scope>
    <!-- Tomcat provides this at runtime, don't bundle it -->
</dependency>
```

### Maven Build Lifecycle

```
Maven has 3 built-in lifecycles:
  1. default (main build)
  2. clean (cleanup)
  3. site (documentation)

Default lifecycle phases (in order):
  validate  → check project is correct
  compile   → compile source code
  test      → run unit tests
  package   → create JAR/WAR
  verify    → run integration tests
  install   → install JAR into local ~/.m2 repository
  deploy    → upload to remote repository (CI/CD)

Key: Running a phase runs ALL previous phases too!
  mvn package → runs validate + compile + test + package
```

### Essential Maven Commands

```bash
# Clean build output
mvn clean

# Compile source code
mvn compile

# Run tests
mvn test

# Create JAR (skips nothing)
mvn package

# Skip tests (faster, use in CI if tests run elsewhere)
mvn package -DskipTests

# Install to local repo (other local projects can use it)
mvn install

# Run the Spring Boot app
mvn spring-boot:run

# Clean + Package (most common combo)
mvn clean package

# Clean + Skip Tests + Package (fastest build)
mvn clean package -DskipTests

# See dependency tree (debug version conflicts)
mvn dependency:tree

# Check for outdated dependencies
mvn versions:display-dependency-updates
```

### Dependency Management — Versions Conflict

```xml
<!-- Force a specific version when parent's version isn't what you want -->
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
            <version>2.15.2</version>  <!-- override parent's version -->
        </dependency>
    </dependencies>
</dependencyManagement>
```

```bash
# Debug version conflicts
mvn dependency:tree | grep jackson

# Output shows:
# [INFO] +- org.springframework.boot:spring-boot-starter-web:jar:3.2.0:compile
# [INFO] |  \- com.fasterxml.jackson.core:jackson-databind:jar:2.15.2:compile
```

---

## ⚡ Gradle — Faster, Modern

### build.gradle (Groovy DSL)

```groovy
plugins {
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
    id 'java'
}

group = 'com.example'
version = '1.0.0'
sourceCompatibility = '17'

repositories {
    mavenCentral()  // where to download dependencies from
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'org.springframework.boot:spring-boot-starter-security'

    runtimeOnly 'com.mysql:mysql-connector-j'       // runtime only
    testRuntimeOnly 'com.h2database:h2'              // test only, runtime
    compileOnly 'org.projectlombok:lombok'           // compile only
    annotationProcessor 'org.projectlombok:lombok'   // Lombok annotation processor

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

test {
    useJUnitPlatform()
}
```

### Gradle Dependency Configurations

```groovy
// Gradle equivalent of Maven scopes:
implementation      // compile + runtime + in final jar (= Maven compile)
runtimeOnly         // runtime only, not compile (= Maven runtime)
compileOnly         // compile only, not in jar (= Maven provided)
testImplementation  // test compile + runtime (= Maven test)
testRuntimeOnly     // test runtime only
annotationProcessor // annotation processors (Lombok, MapStruct)
```

### Essential Gradle Commands

```bash
# Build everything
./gradlew build

# Run tests only
./gradlew test

# Run the app
./gradlew bootRun

# Create JAR
./gradlew bootJar

# Clean build output
./gradlew clean

# Clean + build
./gradlew clean build

# Skip tests
./gradlew build -x test

# See dependencies
./gradlew dependencies

# See specific config dependencies
./gradlew dependencies --configuration runtimeClasspath
```

---

## 🔌 Plugins — Extending Build Tools

### Maven Plugins

```xml
<build>
    <plugins>

        <!-- Spring Boot plugin — run and package -->
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>

        <!-- Surefire — controls how tests run -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-surefire-plugin</artifactId>
            <configuration>
                <includes>
                    <include>**/*Test.java</include>
                </includes>
            </configuration>
        </plugin>

        <!-- Checkstyle — code style enforcement -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-checkstyle-plugin</artifactId>
            <version>3.3.0</version>
        </plugin>

        <!-- JaCoCo — code coverage reports -->
        <plugin>
            <groupId>org.jacoco</groupId>
            <artifactId>jacoco-maven-plugin</artifactId>
            <version>0.8.10</version>
        </plugin>

    </plugins>
</build>
```

### Gradle Plugins

```groovy
plugins {
    id 'org.springframework.boot' version '3.2.0'         // Spring Boot
    id 'io.spring.dependency-management' version '1.1.4'  // dependency versions
    id 'java'                                              // Java compilation
    id 'jacoco'                                           // code coverage
    id 'checkstyle'                                       // code style
}

jacoco {
    toolVersion = "0.8.10"
}

jacocoTestReport {
    reports {
        xml.required = true
        html.required = true
    }
}
```

---

## 📦 The Local Repository — ~/.m2

```
When Maven downloads a dependency, it caches it in:
  ~/.m2/repository/   (Linux/Mac)
  C:\Users\You\.m2\repository\  (Windows)

Structure:
  ~/.m2/repository/
    org/springframework/boot/
      spring-boot-starter-web/
        3.2.0/
          spring-boot-starter-web-3.2.0.jar  ← cached here
          spring-boot-starter-web-3.2.0.pom

Next time you build, Maven uses cache — no re-download!
Delete ~/.m2 to force re-download everything (fix corruption issues)
```

---

## 🔄 Multi-Module Projects (Advanced)

```xml
<!-- Parent pom.xml -->
<project>
    <groupId>com.example</groupId>
    <artifactId>my-microservices</artifactId>
    <version>1.0.0</version>
    <packaging>pom</packaging>  <!-- pom = parent project -->

    <modules>
        <module>user-service</module>
        <module>order-service</module>
        <module>common-lib</module>
    </modules>
</project>

<!-- user-service/pom.xml -->
<project>
    <parent>
        <groupId>com.example</groupId>
        <artifactId>my-microservices</artifactId>
        <version>1.0.0</version>
    </parent>

    <artifactId>user-service</artifactId>
    <!-- inherits dependencies and plugins from parent -->
</project>
```

---

## 🆚 Maven vs Gradle — Quick Comparison

| Feature | Maven | Gradle |
|---------|-------|--------|
| Config file | pom.xml (XML) | build.gradle (Groovy/Kotlin) |
| Readability | Verbose XML | Concise code |
| Speed | Slower | Faster (incremental builds, caching) |
| Learning curve | Lower | Slightly higher |
| Flexibility | Less flexible | Highly customizable |
| IDE support | Excellent | Excellent |
| Spring Boot default | Both supported | Both supported |
| Enterprise adoption | More common | Growing fast |

**Recommendation for beginners: Start with Maven** — more examples online, simpler mental model.

---

## 🎯 Quick Revision Flashcards

| Concept | One-Line Summary |
|---------|----------------|
| Build tool | Automates: download deps + compile + test + package |
| pom.xml | Maven's config file — defines project, dependencies, build |
| build.gradle | Gradle's config file — same purpose, code-based |
| `groupId` | Your company/organization identifier |
| `artifactId` | Your project name |
| `version` | Your project version |
| `compile` scope | Available everywhere (Maven default) |
| `test` scope | Only for tests |
| `runtime` scope | Not needed for compile, needed to run |
| `mvn clean package` | Most common build command |
| `mvn spring-boot:run` | Run app with Maven |
| `./gradlew bootRun` | Run app with Gradle |
| `-DskipTests` | Skip tests in Maven |
| `-x test` | Skip tests in Gradle |
| `~/.m2` | Maven local cache — downloaded JARs live here |
| `dependency:tree` | Debug dependency version conflicts |
| Build lifecycle | validate→compile→test→package→install→deploy |

---

> ✅ **You've mastered Build Tools when:** You can create a Spring Boot project from scratch with the right dependencies, understand what scope each dep should have, know the build lifecycle phases, and can diagnose a version conflict using dependency:tree.
