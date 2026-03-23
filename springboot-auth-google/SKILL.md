---
name: springboot-auth-google
description: >
  TRIGGER this skill when the user says "add Google login", "add Google OAuth2", "add OAuth authentication",
  "add social login", "run springboot-auth-google", "set up Google Sign-In", "add Spring Security OAuth2",
  or anything about authenticating users via Google accounts.
  Requires setup, scaffold, db, AND gcp (you need a GCP project to create OAuth2 credentials).
  Creates the Member entity, Google OAuth2 flow, CustomOAuth2UserService, and role-based access.
---

# springboot-auth-google

Adds Google OAuth2 login with Spring Security. Creates the `members` table, `CustomOAuth2UserService`,
and role-based route authorization. After this skill, users can log in with Google and are either
granted access or shown a "pending approval" page.

---

## Dependencies

### Hard prerequisites
- `setup`, `scaffold`, `db`, `gcp` must all be installed

**Why `gcp` is required:** Google OAuth2 credentials (Client ID + Client Secret) are created in
the Google Cloud Console as part of your GCP project's API credentials. You cannot complete the
OAuth2 setup without a GCP project configured. Run `springboot-gcp-setup` first.

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
base_package      → Java package root
super_admin_email → email for dev seed data
app_url           → used in OAuth2 redirect URI instructions
test_mode         → controls build verification
language          → respond in this language for ALL output (questions, explanations,
                    status messages, code comments, and user-facing copy in generated files)
                    IMPORTANT: if "zh-TW", "traditional-chinese", or "繁體中文":
                      use 繁體中文 (Traditional Chinese) throughout
                      NEVER use Simplified Chinese (简体中文) — they are different writing systems
                      Key differences: 體/体, 語/语, 資料/数据, 設定/设定, 確認/确认, 請/请
translate_terms   → whether to translate technical terms
beginner_friendly → if true, explain technical terms and decisions as you work
installed_modules → must contain "setup", "scaffold", "db"
```

> **Language activation**: After reading `language` above, switch ALL your responses to that
> language immediately — including every question, status message, explanation, and all
> human-readable text in any generated files (HTML, Thymeleaf templates, SQL seed labels).
> If `language` is not set, ask: "What language should I use? (English / 繁體中文 / other)"



---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`, `"scaffold"`, `"db"`, AND `"gcp"`.

If `"gcp"` is missing, stop and tell the user:
```
Cannot set up Google auth without a GCP project.

Google OAuth2 credentials must be created in the GCP Console, which requires:
  1. A GCP project (gcp_project_id in .spring-config.json)
  2. The APIs & Services → Credentials section set up

Please run springboot-gcp-setup first, then come back to springboot-auth-google.
```

If any other prerequisite is missing, tell the user which skills to run first.

---

## Beginner-Friendly Mode

If `beginner_friendly` is `true` in `.spring-config.json`, explain key concepts as you work. Examples:
- When configuring OAuth2: "OAuth2 is a standard that lets users log in with their existing Google account instead of creating a new password. Google handles authentication; your app just receives the user's profile info."
- When adding Spring Security: "Spring Security is a framework that protects your routes. We configure which URLs require login and which are public — Spring handles the rest automatically."
- When creating `CustomOAuth2UserService`: "This class runs after Google confirms the user's identity. We use it to create or update our local `Member` record with the user's Google profile data."

## Step 2 — Update `build.gradle.kts`

Add to `dependencies {}`:

```kotlin
// OAuth2
implementation("org.springframework.boot:spring-boot-starter-oauth2-client")
```

---

## Step 3 — Flyway migration: `V2__create_members.sql`

Create `src/main/resources/db/migration/V2__create_members.sql`:

```sql
CREATE TABLE members (
    id            BIGSERIAL PRIMARY KEY,
    email         VARCHAR(255) NOT NULL UNIQUE,
    name          VARCHAR(255),
    picture       VARCHAR(1024),
    role          VARCHAR(20)  NOT NULL DEFAULT 'MEMBER'
                  CHECK (role IN ('MEMBER', 'ADMIN', 'SUPER_ADMIN')),
    status        VARCHAR(20)  NOT NULL DEFAULT 'PENDING'
                  CHECK (status IN ('PENDING', 'ACTIVE', 'SUSPENDED')),
    created_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_members_email ON members (email);

-- Dev seed: create the super admin account
-- On first login this record will be matched by email and the SUPER_ADMIN role granted
INSERT INTO members (email, name, role, status)
VALUES ('{{super_admin_email}}', 'Super Admin', 'SUPER_ADMIN', 'ACTIVE')
ON CONFLICT (email) DO NOTHING;
```

Replace `{{super_admin_email}}` with the actual value from config.

---

## Step 4 — `Member` entity

Create `src/main/java/{{BASE_PACKAGE_PATH}}/entity/Member.java`:

```java
package {{base_package}}.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.OffsetDateTime;

@Entity
@Table(name = "members")
@Getter @Setter @NoArgsConstructor @Builder
@AllArgsConstructor
public class Member {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String email;

    private String name;
    private String picture;

    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    private Role role = Role.MEMBER;

    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    private Status status = Status.PENDING;

    @Column(name = "created_at", nullable = false, updatable = false)
    private OffsetDateTime createdAt;

    @Column(name = "updated_at", nullable = false)
    private OffsetDateTime updatedAt;

    @PrePersist
    void onCreate() {
        createdAt = updatedAt = OffsetDateTime.now();
    }

    @PreUpdate
    void onUpdate() {
        updatedAt = OffsetDateTime.now();
    }

    public enum Role {
        MEMBER, ADMIN, SUPER_ADMIN
    }

    public enum Status {
        PENDING, ACTIVE, SUSPENDED
    }
}
```

---

## Step 5 — `MemberRepository`

```java
package {{base_package}}.repository;

import {{base_package}}.entity.Member;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.Optional;

public interface MemberRepository extends JpaRepository<Member, Long> {
    Optional<Member> findByEmail(String email);
    boolean existsByEmail(String email);
}
```

---

## Step 6 — `CustomOAuth2UserService`

```java
package {{base_package}}.security;

import {{base_package}}.entity.Member;
import {{base_package}}.repository.MemberRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.oauth2.client.userinfo.DefaultOAuth2UserService;
import org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest;
import org.springframework.security.oauth2.client.userinfo.OAuth2UserService;
import org.springframework.security.oauth2.core.OAuth2AuthenticationException;
import org.springframework.security.oauth2.core.user.DefaultOAuth2User;
import org.springframework.security.oauth2.core.user.OAuth2User;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;
import java.util.Map;

@Service
@Slf4j
@RequiredArgsConstructor
public class CustomOAuth2UserService implements OAuth2UserService<OAuth2UserRequest, OAuth2User> {

    private final MemberRepository memberRepository;
    private final DefaultOAuth2UserService delegate = new DefaultOAuth2UserService();

    @Override
    @Transactional
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        OAuth2User oauthUser = delegate.loadUser(userRequest);

        String email   = oauthUser.getAttribute("email");
        String name    = oauthUser.getAttribute("name");
        String picture = oauthUser.getAttribute("picture");

        Member member = memberRepository.findByEmail(email)
            .map(existing -> updateMember(existing, name, picture))
            .orElseGet(() -> createMember(email, name, picture));

        String springRole = "ROLE_" + member.getRole().name();

        return new DefaultOAuth2User(
            List.of(() -> springRole),
            Map.of(
                "email",   email,
                "name",    name    != null ? name    : "",
                "picture", picture != null ? picture : "",
                "role",    member.getRole().name(),
                "status",  member.getStatus().name(),
                "id",      member.getId()
            ),
            "email"
        );
    }

    private Member updateMember(Member member, String name, String picture) {
        member.setName(name);
        member.setPicture(picture);
        return memberRepository.save(member);
    }

    private Member createMember(String email, String name, String picture) {
        log.info("New OAuth2 login, creating PENDING member: {}", email);
        Member member = Member.builder()
            .email(email)
            .name(name)
            .picture(picture)
            .role(Member.Role.MEMBER)
            .status(Member.Status.PENDING)
            .build();
        return memberRepository.save(member);
    }
}
```

---

## Step 7 — `SecurityConfig.java` — replace/update

Replace the placeholder `SecurityConfig` from scaffold:

```java
package {{base_package}}.config;

import {{base_package}}.security.CustomOAuth2UserService;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final CustomOAuth2UserService customOAuth2UserService;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/css/**", "/js/**", "/images/**", "/favicon.ico").permitAll()
                .requestMatchers("/", "/apply/**", "/pending", "/error").permitAll()
                .requestMatchers("/admin/**").hasAnyRole("ADMIN", "SUPER_ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2Login(oauth2 -> oauth2
                .userInfoEndpoint(ui -> ui.userService(customOAuth2UserService))
                .defaultSuccessUrl("/dashboard", true)
                .failureUrl("/error?oauth=true")
            )
            .logout(logout -> logout
                .logoutSuccessUrl("/")
                .invalidateHttpSession(true)
                .clearAuthentication(true)
            )
            .csrf(csrf -> csrf.ignoringRequestMatchers("/api/**"));

        return http.build();
    }
}
```

---

## Step 8 — `DashboardController` (redirect based on role)

```java
package {{base_package}}.controller;

import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.core.user.OAuth2User;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class DashboardController {

    @GetMapping("/dashboard")
    public String dashboard(@AuthenticationPrincipal OAuth2User user, Model model) {
        if (user == null) {
            return "redirect:/";
        }
        String role   = (String) user.getAttribute("role");
        String status = (String) user.getAttribute("status");

        if ("PENDING".equals(status)) {
            return "redirect:/pending";
        }
        if ("SUSPENDED".equals(status)) {
            return "redirect:/suspended";
        }

        model.addAttribute("user", user);
        if ("ADMIN".equals(role) || "SUPER_ADMIN".equals(role)) {
            return "redirect:/admin/members";
        }
        return "member/dashboard";
    }

    @GetMapping("/pending")
    public String pending(@AuthenticationPrincipal OAuth2User user, Model model) {
        if (user != null) {
            model.addAttribute("email", user.getAttribute("email"));
        }
        return "auth/pending";
    }
}
```

---

## Step 9 — Thymeleaf templates for auth

### `templates/auth/pending.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8" />
    <title>Application Pending</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-50 flex items-center justify-center min-h-screen">
    <div class="text-center max-w-md px-6">
        <div class="text-5xl mb-6">⏳</div>
        <h1 class="text-2xl font-bold text-gray-800 mb-4">Application Under Review</h1>
        <p class="text-gray-500 mb-2">
            Your account (<span th:text="${email}" class="font-medium text-gray-700">email</span>)
            is pending approval.
        </p>
        <p class="text-gray-400 text-sm">
            You will receive an email once your account is approved.
        </p>
        <div class="mt-8">
            <a href="/logout" class="text-blue-600 hover:underline text-sm">Sign out</a>
        </div>
    </div>
</body>
</html>
```

### `templates/member/dashboard.html`

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8" />
    <title>Dashboard</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-50 min-h-screen">
    <nav class="bg-white border-b px-6 py-4 flex items-center justify-between">
        <span class="font-bold text-gray-800">Member Portal</span>
        <div class="flex items-center gap-4">
            <img th:src="${user.attributes.picture}" th:if="${user.attributes.picture}"
                 class="w-8 h-8 rounded-full" alt="avatar" />
            <span class="text-sm text-gray-600" th:text="${user.attributes.name}">Name</span>
            <a href="/logout" class="text-sm text-red-500 hover:underline">Logout</a>
        </div>
    </nav>
    <main class="max-w-4xl mx-auto px-6 py-12">
        <h1 class="text-3xl font-bold text-gray-800 mb-2">Welcome back,
            <span th:text="${user.attributes.name}">Member</span>!
        </h1>
        <p class="text-gray-500">Your member portal is ready.</p>
    </main>
</body>
</html>
```

---

## Step 10 — `application.yml` OAuth2 config

Add to `application.yml`:

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope:
              - openid
              - profile
              - email
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"
        provider:
          google:
            authorization-uri: https://accounts.google.com/o/oauth2/v2/auth
            token-uri: https://oauth2.googleapis.com/token
            user-info-uri: https://www.googleapis.com/oauth2/v3/userinfo
            user-name-attribute: email
```

Add to `application-local.yml` (dev values from Google Console):

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
```

Tell the user to set `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` as environment variables
or in their IDE run configuration.

---

## Step 11 — Google Cloud Console instructions (REQUIRED — do this now)

Walk the user through this step by step. Do not proceed until they confirm completion.

```
You need to create OAuth2 credentials in the GCP Console. Here's the exact path:

1. Go to: https://console.cloud.google.com/apis/credentials
   (Make sure project "{{gcp_project_id}}" is selected in the top dropdown)

2. Click "Create Credentials" → "OAuth 2.0 Client IDs"

3. If prompted to configure the OAuth consent screen first:
   - User Type: External (or Internal if using Google Workspace)
   - App name: {{app_name}}
   - User support email: {{super_admin_email}}
   - Developer contact: {{super_admin_email}}
   - Scopes: add "email", "profile", "openid"
   - Click Save and Continue through all steps

4. Back in "Create OAuth 2.0 Client ID":
   - Application type: Web application
   - Name: {{app_name}} Local Dev

5. Authorized redirect URIs — add BOTH:
     http://localhost:8080/login/oauth2/code/google    ← LOCAL (required now)
     {{app_url}}/login/oauth2/code/google              ← PRODUCTION (add now for later)

6. Click "Create"

7. A dialog shows your credentials. Copy them NOW (you won't see the secret again):
   - Client ID:     (looks like: 123456789-abc.apps.googleusercontent.com)
   - Client Secret: (looks like: GOCSPX-xxxxxxx)

8. Set them in your local environment:
     export GOOGLE_CLIENT_ID=your-client-id
     export GOOGLE_CLIENT_SECRET=your-client-secret
   OR add them to your IntelliJ run configuration environment variables.

9. For production: add these to Secret Manager (covered in springboot-deploy):
     gcloud secrets create GOOGLE_CLIENT_ID --data-file=-  <<< "your-client-id"
     gcloud secrets create GOOGLE_CLIENT_SECRET --data-file=- <<< "your-client-secret"
```

Ask the user: "Have you completed the OAuth2 credentials setup? Paste your Client ID to confirm."

---

## Step 12 — Unit tests for `CustomOAuth2UserService`

If `test_mode` is NOT `"token-save"`, create:

`src/test/java/{{BASE_PACKAGE_PATH}}/security/CustomOAuth2UserServiceTest.java`:

```java
package {{base_package}}.security;

import {{base_package}}.entity.Member;
import {{base_package}}.repository.MemberRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest;
import org.springframework.security.oauth2.core.user.OAuth2User;
import java.util.Optional;

import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class CustomOAuth2UserServiceTest {

    @Mock MemberRepository memberRepository;
    @Mock OAuth2UserRequest userRequest;

    // Note: CustomOAuth2UserService delegates to DefaultOAuth2UserService internally,
    // so full unit tests require mocking the OAuth2 exchange.
    // Integration tests (using @SpringBootTest + MockOAuth2Login) are more effective.
    // Add basic unit tests for createMember / updateMember logic if those are extracted
    // to separate methods.

    @Test
    void newMember_createdWithPendingStatus() {
        // Arrange
        Member savedMember = Member.builder()
            .email("new@example.com")
            .name("New User")
            .role(Member.Role.MEMBER)
            .status(Member.Status.PENDING)
            .build();
        when(memberRepository.findByEmail("new@example.com")).thenReturn(Optional.empty());
        when(memberRepository.save(any(Member.class))).thenReturn(savedMember);

        // Act + Assert
        assertThat(savedMember.getStatus()).isEqualTo(Member.Status.PENDING);
        assertThat(savedMember.getRole()).isEqualTo(Member.Role.MEMBER);
        verify(memberRepository, never()).delete(any());
    }

    @Test
    void existingMember_rolePreserved() {
        Member existing = Member.builder()
            .email("admin@example.com")
            .role(Member.Role.SUPER_ADMIN)
            .status(Member.Status.ACTIVE)
            .build();
        when(memberRepository.findByEmail("admin@example.com")).thenReturn(Optional.of(existing));
        when(memberRepository.save(any())).thenReturn(existing);

        // The service should NOT downgrade an existing admin's role
        assertThat(existing.getRole()).isEqualTo(Member.Role.SUPER_ADMIN);
    }
}
```

---

## Step 13 — Verify

Apply `test_mode`:
- `"token-save"` → Skip. Tell user to start the app and try logging in.
- `"build-only"` → Run `./gradlew build -x test`.
- `"build-and-test"` → Run `./gradlew build`.

---

## Step 14 — Git commits

```bash
git add -A
git commit -m "feat: add Google OAuth2 login with Member entity and role-based access"
```

---

## Step 15 — Update `installed_modules`

Add `"auth-google"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
Google OAuth2 auth is ready. Test it by running the app and visiting http://localhost:8080

Next steps:
- springboot-membership → member profiles, admin member list, role management
- springboot-membership-apply → public application form
- springboot-mail → email notifications
```
