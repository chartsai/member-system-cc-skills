---
name: springboot-scaffold
description: >
  TRIGGER this skill when the user says "scaffold the project", "create the Spring Boot project",
  "generate the project structure", "run springboot-scaffold", "set up Gradle and Spring Boot",
  or anything about creating a new Spring Boot project from scratch with Thymeleaf and Tailwind.
  Always runs AFTER springboot-setup. Creates the full foundational project structure.
---

# springboot-scaffold

Creates a complete Spring Boot 3.x project from scratch with Gradle Kotlin DSL, Thymeleaf,
Tailwind CSS (CDN), Spring Security (basic), and a working home page.

---

## Step 0 — Read config

Read `.spring-config.json` from the current directory. Extract:

```
app_name          → used in titles, README, app properties
brand_name        → used in page footer / nav
base_package      → Java package root (e.g. com.example.myapp)
app_url           → used in application.yml
super_admin_email → used in dev seed data
test_mode         → controls build verification at the end
language          → respond in this language for ALL output (questions, explanations,
                    status messages, code comments, and user-facing copy in generated files)
                    IMPORTANT: if "zh-TW", "traditional-chinese", or "繁體中文":
                      use 繁體中文 (Traditional Chinese) throughout
                      NEVER use Simplified Chinese (简体中文) — they are different writing systems
                      Key differences: 體/体, 語/语, 資料/数据, 設定/设定, 確認/确认, 請/请 throughout
translate_terms   → whether to translate technical terms
beginner_friendly → if true, explain technical terms and decisions as you work
installed_modules → must contain "setup" before proceeding
```

> **Language activation**: After reading `language` above, switch ALL your responses to that
> language immediately — including every question, status message, explanation, and all
> human-readable text in any generated files (HTML, Thymeleaf templates, SQL seed labels).
> If `language` is not set, ask: "What language should I use? (English / 繁體中文 / other)"


> **Traditional vs Simplified Chinese**: If the user is in Taiwan/Hong Kong or specifies
> 繁體中文, always use Traditional Chinese characters. Never substitute Simplified Chinese.
> Common Traditional terms: 設定 (not 设定), 資料 (not 数据), 確認 (not 确认),
> 請 (not 请), 語言 (not 语言), 會員 (not 会员), 管理員 (not 管理员).



---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`. If not, tell the user to run `springboot-setup` first. Run `springboot-menu` to see your full project status and get guided next steps.

---

## Beginner-Friendly Mode

If `beginner_friendly` is `true` in `.spring-config.json`, explain key concepts as you work. Examples:
- When creating `build.gradle.kts`: "Gradle is the build tool — it downloads your dependencies, compiles your code, and runs tests. The `.kts` extension means we're writing the config in Kotlin instead of Groovy."
- When adding Spring Boot dependencies: "Each `implementation(...)` line tells Gradle to download a library. `spring-boot-starter-web` gives us a web server; `spring-boot-starter-thymeleaf` lets us render HTML templates."
- When creating `application.yml`: "This file configures your app — things like the server port, database URL, and feature flags. Spring Boot reads it automatically on startup."

## Step 2 — Create project directory structure

Create the following layout (replace `{{BASE_PACKAGE_PATH}}` with `base_package` using `/` separators,
e.g. `com/example/myapp`):

```
src/
├── main/
│   ├── java/{{BASE_PACKAGE_PATH}}/
│   │   ├── {{AppNameCamelCase}}Application.java
│   │   ├── config/
│   │   │   └── SecurityConfig.java
│   │   └── controller/
│   │       └── HomeController.java
│   └── resources/
│       ├── application.yml
│       ├── application-local.yml
│       ├── application-prod.yml
│       ├── logback-spring.xml
│       ├── static/
│       │   └── css/
│       │       └── app.css
│       └── templates/
│           ├── layout/
│           │   └── base.html
│           ├── index.html
│           └── error/
│               ├── 403.html
│               └── 404.html
├── test/
│   └── java/{{BASE_PACKAGE_PATH}}/
│       └── {{AppNameCamelCase}}ApplicationTests.java
build.gradle.kts
settings.gradle.kts
gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
gradlew
gradlew.bat
.gitignore
README.md
```

---

## Step 3 — `settings.gradle.kts`

```kotlin
rootProject.name = "{{app_name_kebab}}"
```

---

## Step 4 — `build.gradle.kts`

```kotlin
plugins {
    id("org.springframework.boot") version "3.3.5"
    id("io.spring.dependency-management") version "1.1.6"
    kotlin("jvm") version "1.9.25"                   // only if user wants Kotlin; default to Java
    java
}

group = "{{base_package}}"
version = "0.0.1-SNAPSHOT"

java {
    toolchain {
        languageVersion = JavaLanguageVersion.of(21)
    }
}

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-thymeleaf")
    implementation("org.springframework.boot:spring-boot-starter-security")
    implementation("org.thymeleaf.extras:thymeleaf-extras-springsecurity6")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    compileOnly("org.projectlombok:lombok")
    annotationProcessor("org.projectlombok:lombok")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
    testImplementation("org.springframework.security:spring-security-test")
}

tasks.withType<Test> {
    useJUnitPlatform()
}
```

Note: Use Java 21, not Kotlin, unless the user specifically requests Kotlin.

---

## Step 5 — `application.yml`

```yaml
spring:
  application:
    name: {{app_name}}
  thymeleaf:
    cache: false
  security:
    user:
      name: dev
      password: dev

app:
  name: "{{app_name}}"
  brand-name: "{{brand_name}}"
  url: "{{app_url}}"

server:
  port: 8080

logging:
  level:
    root: INFO
    "{{base_package}}": DEBUG
```

---

## Step 6 — `application-local.yml`

```yaml
spring:
  thymeleaf:
    cache: false

logging:
  level:
    root: DEBUG
    "{{base_package}}": DEBUG
    org.springframework.security: DEBUG
```

---

## Step 7 — `application-prod.yml`

```yaml
spring:
  thymeleaf:
    cache: true

logging:
  level:
    root: WARN
    "{{base_package}}": INFO

server:
  port: 8080
```

---

## Step 8 — `logback-spring.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <springProfile name="local">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="DEBUG">
            <appender-ref ref="CONSOLE" />
        </root>
    </springProfile>

    <springProfile name="prod">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{yyyy-MM-dd HH:mm:ss} %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="WARN">
            <appender-ref ref="CONSOLE" />
        </root>
    </springProfile>

    <!-- Default (no profile) -->
    <springProfile name="!local &amp; !prod">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss} %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="INFO">
            <appender-ref ref="CONSOLE" />
        </root>
    </springProfile>
</configuration>
```

---

## Step 9 — `SecurityConfig.java`

```java
package {{base_package}}.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/css/**", "/js/**", "/images/**", "/favicon.ico").permitAll()
                .requestMatchers("/", "/apply/**").permitAll()
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .permitAll()
                .disable()   // disabled for now — will be replaced by OAuth2 or magic-link auth
            )
            .csrf(csrf -> csrf.ignoringRequestMatchers("/api/**"));
        return http.build();
    }
}
```

Note: Login is disabled for now. Auth will be added by `springboot-auth-google` or
`springboot-auth-magic-link`.

---

## Step 10 — `HomeController.java`

```java
package {{base_package}}.controller;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class HomeController {

    @Value("${app.name}")
    private String appName;

    @Value("${app.brand-name}")
    private String brandName;

    @GetMapping("/")
    public String home(Model model) {
        model.addAttribute("appName", appName);
        model.addAttribute("brandName", brandName);
        return "index";
    }
}
```

---

## Step 11 — Thymeleaf templates

### `templates/layout/base.html`

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://www.thymeleaf.org/extras/spring-security">
<head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title th:text="${pageTitle} ?: ${appName}">App</title>
    <script src="https://cdn.tailwindcss.com"></script>
    <link rel="preconnect" href="https://fonts.googleapis.com" />
    <link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap"
          rel="stylesheet" />
    <style>
        body { font-family: 'Inter', sans-serif; }
    </style>
</head>
<body class="bg-gray-50 min-h-screen">

    <!-- Navigation -->
    <nav class="bg-white border-b border-gray-200 px-6 py-4">
        <div class="max-w-6xl mx-auto flex items-center justify-between">
            <a th:href="@{/}" class="text-xl font-bold text-gray-800"
               th:text="${appName}">App Name</a>
            <div class="flex items-center gap-4">
                <sec:authorize access="isAuthenticated()">
                    <span class="text-sm text-gray-500"
                          sec:authentication="name">user@example.com</span>
                    <a th:href="@{/logout}"
                       class="text-sm text-red-600 hover:text-red-800">Logout</a>
                </sec:authorize>
                <sec:authorize access="!isAuthenticated()">
                    <a th:href="@{/apply}"
                       class="text-sm bg-blue-600 text-white px-4 py-2 rounded-lg hover:bg-blue-700">
                        Apply
                    </a>
                </sec:authorize>
            </div>
        </div>
    </nav>

    <!-- Main content -->
    <main th:replace="${content}">
        <!-- content injected here -->
    </main>

    <!-- Footer -->
    <footer class="mt-16 py-8 border-t border-gray-200 text-center text-sm text-gray-400">
        <p th:text="${brandName}">Brand Name</p>
    </footer>

</body>
</html>
```

### `templates/index.html`

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org"
      th:replace="~{layout/base :: layout(content=~{::main})}">
<head><title>Home</title></head>
<body>
<main th:fragment="content">
    <div class="max-w-4xl mx-auto px-6 py-20 text-center">
        <h1 class="text-5xl font-bold text-gray-900 mb-4"
            th:text="${appName}">App Name</h1>
        <p class="text-xl text-gray-500 mb-10">Welcome to the member portal.</p>
        <div class="flex gap-4 justify-center">
            <a th:href="@{/apply}"
               class="bg-blue-600 text-white px-8 py-3 rounded-lg text-lg font-medium hover:bg-blue-700 transition">
                Apply for Membership
            </a>
            <a th:href="@{/dashboard}"
               class="bg-white border border-gray-300 text-gray-700 px-8 py-3 rounded-lg text-lg font-medium hover:bg-gray-50 transition">
                Member Login
            </a>
        </div>
    </div>
</main>
</body>
</html>
```

### `templates/error/403.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8" />
    <title>403 Forbidden</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-50 flex items-center justify-center min-h-screen">
    <div class="text-center">
        <h1 class="text-6xl font-bold text-gray-300 mb-4">403</h1>
        <p class="text-xl text-gray-600 mb-8">You don't have permission to access this page.</p>
        <a href="/" class="text-blue-600 hover:underline">← Go home</a>
    </div>
</body>
</html>
```

### `templates/error/404.html`

Similar to 403 but with "404" and "Page not found."

---

## Step 12 — `.gitignore`

```gitignore
HELP.md
.gradle/
build/
!gradle/wrapper/gradle-wrapper.jar
!**/src/main/**/build/
!**/src/test/**/build/
.spring-config.json
application-local.yml
*.class
*.jar
*.war
.DS_Store
.idea/
*.iml
*.ipr
*.iws
out/
uploads/
*.log
target/
```

Note: Add `.spring-config.json` to `.gitignore` since it may contain sensitive values.

---

## Step 13 — `README.md`

Write a basic README with:
- Project name and description
- Tech stack (Spring Boot 3.x, Java 21, Thymeleaf, Tailwind, PostgreSQL, Flyway, Gradle)
- How to run locally: `./gradlew bootRun --args='--spring.profiles.active=local'`
- How to build: `./gradlew build`

---

## Step 14 — Application main class

```java
package {{base_package}};

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

@SpringBootApplication
public class {{AppNameCamelCase}}Application {
    public static void main(String[] args) {
        SpringApplication.run({{AppNameCamelCase}}Application.class, args);
    }
}
```

---

## Step 15 — Run the Gradle wrapper

Initialize the Gradle wrapper if not already present:

```bash
gradle wrapper --gradle-version 8.10
```

If `gradle` is not available locally, download the Gradle wrapper files manually or ask the user
to run this command themselves.

---

## Step 16 — Verify the build

Apply `test_mode`:

- `"token-save"` → Skip. Tell the user to run `./gradlew bootRun` when ready.
- `"build-only"` → Run `./gradlew build -x test`. Report success or errors.
- `"build-and-test"` → Run `./gradlew build`. Report success or errors.

If errors occur, fix them before proceeding.

Also verify the application starts:

```bash
./gradlew bootRun --args='--spring.profiles.active=local'
```

The home page at `http://localhost:8080` should load. Tell the user to visit it and confirm they
see the welcome page.

---

## Step 16b — Unit Tests

Generate a smoke test that verifies the Spring application context loads without errors:

```kotlin
// src/test/kotlin/{{base_package}}/ApplicationContextTest.kt
package {{base_package}}

import org.junit.jupiter.api.Test
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.context.ActiveProfiles

@SpringBootTest
@ActiveProfiles("test")
class ApplicationContextTest {

    @Test
    fun contextLoads() {
        // Verifies the Spring application context starts without errors
    }
}
```

Also create `src/test/resources/application-test.yml` with an H2 in-memory database so tests
don't require a running PostgreSQL instance:

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa:
    hibernate:
      ddl-auto: create-drop
  flyway:
    enabled: false
```

Add `testImplementation("com.h2database:h2")` to `build.gradle.kts` if not present.

Run the test: `./gradlew test` — it should pass before committing.

---

## Step 17 — Git commit

```bash
git init  # only if not already a git repo
git add -A
git commit -m "feat: initial Spring Boot scaffold with Thymeleaf + Tailwind"
```

---

## Step 18 — Update `installed_modules`

Read `.spring-config.json`, add `"scaffold"` to the `installed_modules` array, and write it back.

Result: `"installed_modules": ["setup", "scaffold"]`

Tell the user:
```
Scaffold complete. Next steps:
- springboot-db-setup → add PostgreSQL + Flyway
- springboot-auth-google or springboot-auth-magic-link → add authentication
```
