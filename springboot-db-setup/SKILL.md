---
name: springboot-db-setup
description: >
  TRIGGER this skill when the user says "set up the database", "add PostgreSQL", "add Flyway",
  "add the database", "set up DB", "run springboot-db-setup", "add Docker Compose for the database",
  or anything about connecting a Spring Boot project to a PostgreSQL database with Flyway migrations.
  Requires springboot-setup and springboot-scaffold to be completed first.
---

# springboot-db-setup

Adds PostgreSQL + Flyway + Docker Compose to the scaffolded project. After this skill runs,
the app can connect to a local Dockerized PostgreSQL database and run Flyway migrations on startup.

---

## Step 0 — Read config

Read `.spring-config.json` from the current directory. Extract:

```
base_package      → Java package root
db_name           → PostgreSQL database name
app_name          → used in docker-compose service labels
test_mode         → controls build verification
language          → respond in this language
translate_terms   → whether to translate technical terms
installed_modules → must contain "setup" and "scaffold"
```

---

## Step 1 — Prerequisites check

Verify `installed_modules` contains both `"setup"` and `"scaffold"`.
If missing either, tell the user which skill to run first.

---

## Step 2 — Update `build.gradle.kts`

Add these dependencies to the existing `dependencies {}` block:

```kotlin
// Database
implementation("org.springframework.boot:spring-boot-starter-data-jpa")
runtimeOnly("org.postgresql:postgresql")
implementation("org.flywaydb:flyway-core")
implementation("org.flywaydb:flyway-database-postgresql")

// Test
testImplementation("org.springframework.boot:spring-boot-starter-test")
```

Also add to the top-level `plugins {}` block if not present — no additional plugins needed for JPA.

---

## Step 3 — `docker-compose.yml`

Create `docker-compose.yml` in the project root:

```yaml
version: "3.9"

services:
  db:
    image: postgres:16-alpine
    container_name: {{app_name_snake}}_db
    environment:
      POSTGRES_DB: {{db_name}}
      POSTGRES_USER: app
      POSTGRES_PASSWORD: localpassword
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d {{db_name}}"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  postgres_data:
```

---

## Step 4 — `application-local.yml` — datasource config

Merge these values into `application-local.yml` (preserve any existing content):

```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/{{db_name}}
    username: app
    password: localpassword
    driver-class-name: org.postgresql.Driver
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        dialect: org.hibernate.dialect.PostgreSQLDialect
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: false
    validate-on-migrate: true
```

---

## Step 5 — `application-prod.yml` — datasource config

Merge these values into `application-prod.yml`:

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 5
      connection-timeout: 30000
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
  flyway:
    enabled: true
    locations: classpath:db/migration
    validate-on-migrate: true
```

Production DB credentials come from environment variables / GCP Secret Manager.

---

## Step 6 — First Flyway migration

Create `src/main/resources/db/migration/V1__init.sql`:

```sql
-- V1: Initial schema
-- This migration verifies that Flyway is wired up correctly.
-- Future migrations will add tables here.

-- Example: create a minimal app_config table for key-value settings
CREATE TABLE app_config (
    key   VARCHAR(255) PRIMARY KEY,
    value TEXT
);

COMMENT ON TABLE app_config IS 'Application-level key-value configuration store';
```

---

## Step 7 — Base JPA configuration class

Create `src/main/java/{{BASE_PACKAGE_PATH}}/config/JpaConfig.java`:

```java
package {{base_package}}.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
import org.springframework.transaction.annotation.EnableTransactionManagement;

@Configuration
@EnableJpaRepositories(basePackages = "{{base_package}}.repository")
@EnableTransactionManagement
public class JpaConfig {
    // Spring Boot auto-configures most JPA settings.
    // Add custom beans here as needed (e.g., AuditorAware for created_by fields).
}
```

---

## Step 8 — AppConfig entity + repository (optional but useful)

Create `src/main/java/{{BASE_PACKAGE_PATH}}/entity/AppConfigEntity.java`:

```java
package {{base_package}}.entity;

import jakarta.persistence.*;
import lombok.Getter;
import lombok.NoArgsConstructor;
import lombok.Setter;

@Entity
@Table(name = "app_config")
@Getter @Setter @NoArgsConstructor
public class AppConfigEntity {

    @Id
    @Column(name = "key")
    private String key;

    @Column(name = "value", columnDefinition = "TEXT")
    private String value;

    public AppConfigEntity(String key, String value) {
        this.key = key;
        this.value = value;
    }
}
```

Create `src/main/java/{{BASE_PACKAGE_PATH}}/repository/AppConfigRepository.java`:

```java
package {{base_package}}.repository;

import {{base_package}}.entity.AppConfigEntity;
import org.springframework.data.jpa.repository.JpaRepository;

public interface AppConfigRepository extends JpaRepository<AppConfigEntity, String> {
    // find by key is just findById(key)
}
```

---

## Step 9 — Instructions for the user

Tell the user:

```
Database setup is ready. To start the local database:

  docker-compose up -d

Then start the app:

  ./gradlew bootRun --args='--spring.profiles.active=local'

Flyway will run V1__init.sql on startup and create the app_config table.
Check the logs for: "Successfully applied 1 migration to schema..."
```

---

## Step 10 — Verify the connection

Apply `test_mode`:

- `"token-save"` → Skip automated verification. Instruct the user to run docker-compose and bootRun manually.
- `"build-only"` → Run `./gradlew build -x test`. Fix any compilation errors.
- `"build-and-test"` → Run `./gradlew build`. Fix any errors.

If the user reports that Flyway fails (connection refused, auth error, etc.), troubleshoot:
1. Check if Docker is running: `docker ps`
2. Check container logs: `docker-compose logs db`
3. Verify port 5432 is not already in use: `lsof -i :5432`

---

## Step 11 — Git commit

```bash
git add -A
git commit -m "feat: add PostgreSQL + Flyway + Docker Compose dev DB"
```

---

## Step 12 — Update `installed_modules`

Read `.spring-config.json`, add `"db"` to the `installed_modules` array, write it back.

Result example: `"installed_modules": ["setup", "scaffold", "db"]`

Tell the user:
```
Database setup complete. Next steps:
- springboot-auth-google → Google OAuth2 authentication
- springboot-auth-magic-link → email magic link authentication
- springboot-gcp-setup → Google Cloud Platform setup (needed for deployment)
```
