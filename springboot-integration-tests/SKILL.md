---
name: springboot-integration-tests
description: >
  TRIGGER this skill when the user says "add integration tests", "generate integration tests",
  "run springboot-integration-tests", "add @SpringBootTest tests", "add Testcontainers tests",
  "test the whole system end to end", "add security integration tests", or anything about generating
  comprehensive integration tests based on which modules are installed. This skill analyzes
  installed_modules and generates appropriate tests. Requires setup, scaffold, db, and at least
  2 other modules installed.
---

# springboot-integration-tests

A consulting/generator skill that analyzes installed modules and generates appropriate
`@SpringBootTest` integration tests using Testcontainers with a real PostgreSQL container.
Does NOT mock the database.

---

## Dependencies

### Hard prerequisites
- `setup`, `scaffold`, `db` must be installed
- At least 2 additional modules beyond setup/scaffold/db

### Soft prerequisites (enables more test scenarios)
- `auth-google` or `auth-magic-link` → security integration tests
- `membership` → role-based access tests
- `membership-apply` → application flow tests
- `item-submit` → submission workflow tests
- `admin-portal` → preview mode tests

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
base_package      → Java package root
super_admin_email → for test seed data
test_mode         → if "token-save", output a test plan document instead of code
language          → respond in this language for ALL output (questions, explanations,
                    status messages, code comments, and user-facing copy in generated files)
                    IMPORTANT: if "zh-TW", "traditional-chinese", or "繁體中文":
                      use 繁體中文 (Traditional Chinese) throughout
                      NEVER use Simplified Chinese (简体中文) — they are different writing systems
                      Key differences: 體/体, 語/语, 資料/数据, 設定/设定, 確認/确认, 請/请
translate_terms   → whether to translate technical terms
beginner_friendly → if true, explain technical terms and decisions as you work
installed_modules → drives which test scenarios to generate
```

> **Language activation**: After reading `language` above, switch ALL your responses to that
> language immediately — including every question, status message, explanation, and all
> human-readable text in any generated files (HTML, Thymeleaf templates, SQL seed labels).
> If `language` is not set, ask: "What language should I use? (English / 繁體中文 / other)"



---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`, `"scaffold"`, `"db"`.

Count modules beyond the foundation (setup, scaffold, db, gcp, ide-setup, app-config, soft-delete, audit-log).
If fewer than 2 feature modules are installed, tell the user:
```
Integration tests work best after at least 2 feature modules are installed.
Currently installed: [list installed_modules]
Recommend installing springboot-membership and one auth module before running this skill.
```

If `test_mode` is `"token-save"`:
```
test_mode is "token-save" — generating a TEST PLAN DOCUMENT instead of code.
To generate actual test code, change test_mode to "build-and-test" in .spring-config.json.
```
Then generate `TEST_PLAN.md` (see Step 11) and stop.

---

## Beginner-Friendly Mode

If `beginner_friendly` is `true` in `.spring-config.json`, explain key concepts as you work. Examples:
- When using Testcontainers: "Testcontainers spins up a real PostgreSQL database in Docker just for your tests, then tears it down afterward. This means your tests run against a real database — no mocks that might behave differently from production."
- When using `@SpringBootTest`: "This annotation starts your entire Spring application for the test, including all beans, security, and database connections. It's slower than unit tests but tests the full stack."
- When testing security: "We test that protected routes return 401/403 without credentials, and that login redirects work. This catches misconfigured security rules before they reach production."

## Step 2 — Update `build.gradle.kts`

Add test dependencies:

```kotlin
// Integration testing
testImplementation("org.springframework.boot:spring-boot-starter-test")
testImplementation("org.springframework.security:spring-security-test")
testImplementation("org.testcontainers:postgresql:1.20.3")
testImplementation("org.testcontainers:junit-jupiter:1.20.3")
testImplementation("org.flywaydb:flyway-core")  // to run migrations in test container
```

---

## Step 3 — Test base class

Create `src/test/java/{{BASE_PACKAGE_PATH}}/integration/BaseIntegrationTest.java`:

```java
package {{base_package}}.integration;

import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.DynamicPropertyRegistry;
import org.springframework.test.context.DynamicPropertySource;
import org.testcontainers.containers.PostgreSQLContainer;
import org.testcontainers.junit.jupiter.Container;
import org.testcontainers.junit.jupiter.Testcontainers;

/**
 * Base class for all integration tests.
 * Starts a real PostgreSQL container via Testcontainers.
 * Flyway migrations run automatically on startup.
 *
 * Do NOT mock the database — all tests run against real PostgreSQL.
 */
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@AutoConfigureMockMvc
@ActiveProfiles("test")
@Testcontainers
public abstract class BaseIntegrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");

    @DynamicPropertySource
    static void configureProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url",      postgres::getJdbcUrl);
        registry.add("spring.datasource.username",  postgres::getUsername);
        registry.add("spring.datasource.password",  postgres::getPassword);
        registry.add("spring.flyway.enabled",       () -> "true");
        registry.add("spring.jpa.hibernate.ddl-auto", () -> "validate");
    }
}
```

---

## Step 4 — `application-test.yml`

Create `src/test/resources/application-test.yml`:

```yaml
spring:
  mail:
    host: localhost
    port: 3025   # Use Greenmail or mock in tests
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: test-client-id
            client-secret: test-client-secret

app:
  url: http://localhost:8080
  magic-link:
    expiry-minutes: 15

logging:
  level:
    root: WARN
    "{{base_package}}": INFO
    org.testcontainers: WARN
    com.zaxxer.hikari: WARN
```

---

## Step 5 — Security integration tests

Generate if `auth-google` or `auth-magic-link` is in `installed_modules`:

```java
package {{base_package}}.integration;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;

import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

/**
 * Integration tests for Spring Security route rules.
 *
 * Tests the actual security configuration — not mocked.
 * Each scenario uses @WithMockUser to simulate an authenticated user with a specific role.
 */
class SecurityIntegrationTest extends BaseIntegrationTest {

    @Autowired MockMvc mockMvc;

    // --- Unauthenticated access ---

    @Test
    void unauthenticated_adminRoute_redirectsToLogin() throws Exception {
        mockMvc.perform(get("/admin/members"))
            .andExpect(status().is3xxRedirection());
    }

    @Test
    void unauthenticated_publicRoute_returns200() throws Exception {
        mockMvc.perform(get("/"))
            .andExpect(status().isOk());
    }

    @Test
    void unauthenticated_applyRoute_returns200() throws Exception {
        mockMvc.perform(get("/apply"))
            .andExpect(status().isOk());
    }

    // --- MEMBER role ---

    @Test
    @WithMockUser(roles = "MEMBER")
    void member_adminRoute_returns403() throws Exception {
        mockMvc.perform(get("/admin/members"))
            .andExpect(status().isForbidden());
    }

    @Test
    @WithMockUser(roles = "MEMBER")
    void member_dashboardRoute_returns200() throws Exception {
        mockMvc.perform(get("/dashboard"))
            .andExpect(status().isOk());
    }

    // --- ADMIN role ---

    @Test
    @WithMockUser(roles = "ADMIN")
    void admin_membersRoute_returns200() throws Exception {
        mockMvc.perform(get("/admin/members"))
            .andExpect(status().isOk());
    }

    @Test
    @WithMockUser(roles = "ADMIN")
    void admin_superAdminOnlyRoute_returns403() throws Exception {
        // Admin management is SUPER_ADMIN only
        mockMvc.perform(get("/admin/members/1/role"))  // adjust to a SUPER_ADMIN-only route
            .andExpect(status().is4xxClientError());
    }

    // --- SUPER_ADMIN role ---

    @Test
    @WithMockUser(roles = "SUPER_ADMIN")
    void superAdmin_allAdminRoutes_return200() throws Exception {
        mockMvc.perform(get("/admin/members"))
            .andExpect(status().isOk());
    }
}
```

---

## Step 6 — Membership flow integration tests

Generate if `membership` and `membership-apply` are both in `installed_modules`:

```java
package {{base_package}}.integration;

import {{base_package}}.entity.*;
import {{base_package}}.repository.*;
import {{base_package}}.service.ApplicationService;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.transaction.annotation.Transactional;

import static org.assertj.core.api.Assertions.*;

/**
 * Integration tests for the application → approval → member creation flow.
 *
 * Uses real DB (Testcontainers PostgreSQL).
 * EmailService is mocked to prevent actual email sending.
 */
@Transactional
class MembershipFlowIntegrationTest extends BaseIntegrationTest {

    @Autowired ApplicationService applicationService;
    @Autowired ApplicationRepository applicationRepository;
    @Autowired MemberRepository memberRepository;

    // Prevent actual email sending in tests
    @MockBean JavaMailSender mailSender;

    @BeforeEach
    void cleanup() {
        applicationRepository.deleteAll();
        memberRepository.deleteAll();
    }

    @Test
    void submitApplication_thenApprove_createsMember() {
        // Submit application
        Application app = applicationService.submit(
            "newuser@example.com", "New User", "I want to join", "127.0.0.1"
        );
        assertThat(app.getStatus()).isEqualTo(Application.Status.PENDING);

        // Admin approves
        Member member = applicationService.approve(app.getId(), "admin@example.com");
        assertThat(member.getStatus()).isEqualTo(Member.Status.ACTIVE);
        assertThat(member.getEmail()).isEqualTo("newuser@example.com");

        // Application is now approved
        Application updated = applicationRepository.findById(app.getId()).orElseThrow();
        assertThat(updated.getStatus()).isEqualTo(Application.Status.APPROVED);
    }

    @Test
    void submitDuplicateApplication_throws() {
        applicationService.submit("dup@example.com", "User A", "reason", "127.0.0.1");

        assertThatThrownBy(() ->
            applicationService.submit("dup@example.com", "User B", "reason 2", "127.0.0.1")
        ).isInstanceOf(IllegalStateException.class)
         .hasMessageContaining("already pending");
    }

    @Test
    void submitApplication_existingActiveMember_throws() {
        memberRepository.save(Member.builder()
            .email("existing@example.com")
            .role(Member.Role.MEMBER)
            .status(Member.Status.ACTIVE)
            .build());

        assertThatThrownBy(() ->
            applicationService.submit("existing@example.com", "Existing", "reason", "127.0.0.1")
        ).isInstanceOf(IllegalStateException.class)
         .hasMessageContaining("already associated");
    }
}
```

---

## Step 7 — Admin preview mode tests

Generate if `admin-portal` is in `installed_modules`:

```java
package {{base_package}}.integration;

import {{base_package}}.admin.PreviewSessionHelper;
import {{base_package}}.entity.Member;
import {{base_package}}.repository.MemberRepository;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.mock.web.MockHttpSession;
import org.springframework.security.test.context.support.WithMockUser;
import org.springframework.test.web.servlet.MockMvc;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.mail.javamail.JavaMailSender;

import static org.assertj.core.api.Assertions.*;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.post;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

class AdminPreviewIntegrationTest extends BaseIntegrationTest {

    @Autowired MockMvc mockMvc;
    @Autowired MemberRepository memberRepository;
    @Autowired PreviewSessionHelper previewSessionHelper;
    @MockBean JavaMailSender mailSender;

    @Test
    @WithMockUser(roles = "SUPER_ADMIN")
    void superAdmin_canEnterPreviewMode() throws Exception {
        Member target = memberRepository.save(Member.builder()
            .email("member@example.com")
            .role(Member.Role.MEMBER)
            .status(Member.Status.ACTIVE)
            .build());

        MockHttpSession session = new MockHttpSession();
        mockMvc.perform(post("/admin/preview/" + target.getId())
            .session(session)
            .with(org.springframework.security.test.web.servlet.request
                  .SecurityMockMvcRequestPostProcessors.csrf()))
            .andExpect(status().is3xxRedirection());

        assertThat(previewSessionHelper.isInPreview(session)).isTrue();
    }

    @Test
    @WithMockUser(roles = "ADMIN")  // ADMIN cannot preview as member
    void admin_cannotEnterPreviewAsSpecificMember() throws Exception {
        mockMvc.perform(post("/admin/preview/1")
            .with(org.springframework.security.test.web.servlet.request
                  .SecurityMockMvcRequestPostProcessors.csrf()))
            .andExpect(status().isForbidden());
    }
}
```

---

## Step 8 — Item submission integration tests

Generate if `item-submit` is in `installed_modules`:

```java
package {{base_package}}.integration;

// SubmissionFlowIntegrationTest
// Test: member creates submission → status is PENDING
// Test: admin approves → status is APPROVED with reviewer info
// Test: admin rejects → status is REJECTED with note
// Test: member cannot access other member's submission (assertOwnership)
// Test: only ADMIN/SUPER_ADMIN can access /admin/submissions
```

---

## Step 9 — Run the generated tests

Apply `test_mode`:
- `"build-and-test"` → Run `./gradlew test`. Fix any failures before marking complete.
- `"build-only"` → Run `./gradlew build -x test`.
- `"token-save"` → This path was handled in Step 1 (generates TEST_PLAN.md instead).

---

## Step 10 — Git commit

```bash
git add -A
git commit -m "test: add integration tests for installed modules (Testcontainers + real PostgreSQL)"
```

---

## Step 11 — TEST_PLAN.md (token-save mode only)

If `test_mode` is `"token-save"`, write `TEST_PLAN.md` to the project root instead of generating code:

```markdown
# Integration Test Plan

Generated based on installed modules: [list from installed_modules]

## Test Scenarios

### Security Tests
- [ ] Unauthenticated → admin routes redirect to login
- [ ] MEMBER role → admin routes return 403
- [ ] ADMIN role → admin routes return 200
- [ ] SUPER_ADMIN role → all routes accessible
- [ ] Public routes (/, /apply) → accessible without auth

### Membership Flow Tests (if membership-apply installed)
- [ ] Submit application → creates PENDING record
- [ ] Duplicate email application → rejected
- [ ] Approve application → creates ACTIVE member
- [ ] Reject application → sets REJECTED status

### Admin Preview Tests (if admin-portal installed)
- [ ] SUPER_ADMIN can enter preview mode
- [ ] ADMIN cannot preview as specific member

[Add more scenarios based on installed modules...]

## To generate actual test code:
Change test_mode to "build-and-test" in .spring-config.json and re-run springboot-integration-tests.
```

---

## Step 12 — Update `installed_modules`

Add `"integration-tests"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
Integration tests generated. Run with:
  ./gradlew test

Tests use Testcontainers — Docker must be running.
```
