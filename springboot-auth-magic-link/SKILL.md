---
name: springboot-auth-magic-link
description: >
  TRIGGER this skill when the user says "add magic link login", "add passwordless login",
  "add email authentication", "run springboot-auth-magic-link", "add magic link auth",
  "set up email-based login without password", or anything about logging users in via a
  link sent to their email. Can be used alongside OR instead of Google OAuth2 auth.
  Requires setup, scaffold, db, AND mail (SMTP must be configured to send the login link).
---

# springboot-auth-magic-link

Adds passwordless email authentication: users enter their email, receive a time-limited link,
click it, and are logged in. Can coexist with Google OAuth2 auth or be used standalone.

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
base_package      → Java package root
super_admin_email → used for dev seed
app_url           → used in magic link URL construction
test_mode         → controls build verification
language          → respond in this language for ALL output (questions, explanations,
                    status messages, code comments, and user-facing copy in generated files)
                    IMPORTANT: if "zh-TW", "traditional-chinese", or "繁體中文":
                      use 繁體中文 (Traditional Chinese) throughout
                      NEVER use Simplified Chinese (简体中文) — they are different writing systems
                      Key differences: 體/体, 語/语, 資料/数据, 設定/设定, 確認/确认, 請/请
translate_terms   → whether to translate technical terms
beginner_friendly → if true, explain technical terms and decisions as you work
installed_modules → must contain "setup", "scaffold", "db", "mail"
```

> **Language activation**: After reading `language` above, switch ALL your responses to that
> language immediately — including every question, status message, explanation, and all
> human-readable text in any generated files (HTML, Thymeleaf templates, SQL seed labels).
> If `language` is not set, ask: "What language should I use? (English / 繁體中文 / other)"



---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`, `"scaffold"`, and `"db"`.

**Hard check — `mail` is required:**
If `"mail"` is NOT in `installed_modules`, stop immediately and output:
```
❌ Cannot proceed: springboot-mail is not installed.

Magic links are sent via email — SMTP must be configured first.
Please run springboot-mail to set up email sending, then return here.
```

Run `springboot-menu` to see your full project status and get guided next steps.

Do not continue until the user confirms mail is installed.

## Beginner-Friendly Mode

If `beginner_friendly` is `true` in `.spring-config.json`, explain key concepts as you work. Examples:
- When creating a magic link token: "A magic link works by generating a random, single-use token stored in the database. When the user clicks the link, we look up the token, verify it hasn't expired, and log them in — no password needed."
- When adding `spring-boot-starter-mail`: "This dependency gives Spring Boot the ability to send emails using the `JavaMailSender` interface. We'll configure it to use your SMTP server."
- When setting token expiry: "The token has a 15-minute expiry for security — if someone intercepts the email and tries to use the link later, it won't work."

Check if `"auth-google"` is also in `installed_modules`. If it is, note:
```
Google auth is already installed. Magic link auth will be added alongside it —
users will be able to log in with either Google or a magic link.
```

If neither auth module is installed yet, the `Member` entity needs to be created too
(use the same schema from `springboot-auth-google` Step 3). Otherwise, it already exists.

---

## Step 2 — Update `build.gradle.kts`

Add if not already present:

```kotlin
// Mail (for sending magic links)
implementation("org.springframework.boot:spring-boot-starter-mail")
```

---

## Step 3 — Flyway migration: create `members` table if not present

If `"auth-google"` is NOT in `installed_modules` (meaning no member table exists yet):

Create `V2__create_members.sql` (same as in `springboot-auth-google`):

```sql
CREATE TABLE members (
    id         BIGSERIAL PRIMARY KEY,
    email      VARCHAR(255) NOT NULL UNIQUE,
    name       VARCHAR(255),
    picture    VARCHAR(1024),
    role       VARCHAR(20) NOT NULL DEFAULT 'MEMBER'
               CHECK (role IN ('MEMBER', 'ADMIN', 'SUPER_ADMIN')),
    status     VARCHAR(20) NOT NULL DEFAULT 'PENDING'
               CHECK (status IN ('PENDING', 'ACTIVE', 'SUSPENDED')),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
CREATE INDEX idx_members_email ON members (email);

INSERT INTO members (email, name, role, status)
VALUES ('{{super_admin_email}}', 'Super Admin', 'SUPER_ADMIN', 'ACTIVE')
ON CONFLICT (email) DO NOTHING;
```

Use the next available migration number (V3 if V2 is taken by Google auth).

---

## Step 4 — Flyway migration: `magic_links` table

Create `V3__create_magic_links.sql` (or next available number):

```sql
CREATE TABLE magic_links (
    id         BIGSERIAL PRIMARY KEY,
    email      VARCHAR(255) NOT NULL,
    token      VARCHAR(64)  NOT NULL UNIQUE,
    expires_at TIMESTAMPTZ  NOT NULL,
    used       BOOLEAN      NOT NULL DEFAULT FALSE,
    created_at TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_magic_links_token ON magic_links (token);
CREATE INDEX idx_magic_links_email ON magic_links (email);
```

---

## Step 5 — `MagicLink` entity

```java
package {{base_package}}.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.OffsetDateTime;

@Entity
@Table(name = "magic_links")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class MagicLink {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String email;

    @Column(nullable = false, unique = true)
    private String token;

    @Column(name = "expires_at", nullable = false)
    private OffsetDateTime expiresAt;

    @Column(nullable = false)
    private boolean used = false;

    @Column(name = "created_at", nullable = false, updatable = false)
    private OffsetDateTime createdAt;

    @PrePersist
    void onCreate() {
        createdAt = OffsetDateTime.now();
    }

    public boolean isExpired() {
        return OffsetDateTime.now().isAfter(expiresAt);
    }

    public boolean isValid() {
        return !used && !isExpired();
    }
}
```

---

## Step 6 — `MagicLinkRepository`

```java
package {{base_package}}.repository;

import {{base_package}}.entity.MagicLink;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import java.time.OffsetDateTime;
import java.util.Optional;

public interface MagicLinkRepository extends JpaRepository<MagicLink, Long> {
    Optional<MagicLink> findByToken(String token);

    @Modifying
    @Query("DELETE FROM MagicLink ml WHERE ml.expiresAt < :cutoff OR ml.used = true")
    void deleteExpiredAndUsed(OffsetDateTime cutoff);
}
```

---

## Step 7 — `MagicLinkService`

```java
package {{base_package}}.service;

import {{base_package}}.entity.MagicLink;
import {{base_package}}.entity.Member;
import {{base_package}}.repository.MagicLinkRepository;
import {{base_package}}.repository.MemberRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.security.SecureRandom;
import java.time.OffsetDateTime;
import java.util.Base64;

@Service
@Slf4j
@RequiredArgsConstructor
public class MagicLinkService {

    private final MagicLinkRepository magicLinkRepository;
    private final MemberRepository memberRepository;
    private final JavaMailSender mailSender;

    @Value("${app.url}")
    private String appUrl;

    @Value("${app.magic-link.expiry-minutes:15}")
    private int expiryMinutes;

    @Value("${spring.mail.username:noreply@example.com}")
    private String fromEmail;

    private static final SecureRandom RANDOM = new SecureRandom();

    /**
     * Generate a magic link token and send it to the given email.
     * Creates a PENDING member record if the email is not yet registered.
     */
    @Transactional
    public void sendMagicLink(String email) {
        // Ensure member exists (or create PENDING)
        memberRepository.findByEmail(email)
            .orElseGet(() -> memberRepository.save(
                Member.builder()
                    .email(email)
                    .role(Member.Role.MEMBER)
                    .status(Member.Status.PENDING)
                    .build()
            ));

        String token = generateToken();
        MagicLink magicLink = MagicLink.builder()
            .email(email)
            .token(token)
            .expiresAt(OffsetDateTime.now().plusMinutes(expiryMinutes))
            .build();
        magicLinkRepository.save(magicLink);

        String link = appUrl + "/auth/verify?token=" + token;
        sendEmail(email, link);

        log.info("Magic link sent to {}", email);
    }

    /**
     * Validate a magic link token. Returns the Member if valid, throws otherwise.
     */
    @Transactional
    public Member validateToken(String token) {
        MagicLink magicLink = magicLinkRepository.findByToken(token)
            .orElseThrow(() -> new IllegalArgumentException("Invalid or expired magic link"));

        if (!magicLink.isValid()) {
            throw new IllegalArgumentException(
                magicLink.isExpired() ? "Magic link has expired" : "Magic link has already been used"
            );
        }

        magicLink.setUsed(true);
        magicLinkRepository.save(magicLink);

        return memberRepository.findByEmail(magicLink.getEmail())
            .orElseThrow(() -> new IllegalStateException("Member not found for email: " + magicLink.getEmail()));
    }

    private String generateToken() {
        byte[] bytes = new byte[48];
        RANDOM.nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }

    private void sendEmail(String to, String link) {
        try {
            SimpleMailMessage message = new SimpleMailMessage();
            message.setFrom(fromEmail);
            message.setTo(to);
            message.setSubject("Your login link for {{app_name}}");
            message.setText(
                "Click the link below to log in. This link expires in " + expiryMinutes + " minutes.\n\n" +
                link + "\n\n" +
                "If you did not request this link, you can ignore this email."
            );
            mailSender.send(message);
        } catch (Exception e) {
            log.error("Failed to send magic link email to {}: {}", to, e.getMessage());
            // In local/dev mode, log the link so developers can test without real email
            log.info("DEV: Magic link for {} → {}", to, link);
            throw new RuntimeException("Failed to send magic link email. Check mail configuration.", e);
        }
    }

    @Scheduled(cron = "0 0 3 * * *")  // Daily at 3am
    @Transactional
    public void cleanupExpiredTokens() {
        magicLinkRepository.deleteExpiredAndUsed(OffsetDateTime.now());
        log.info("Cleaned up expired magic link tokens");
    }
}
```

---

## Step 8 — `MagicLinkAuthController`

```java
package {{base_package}}.controller;

import {{base_package}}.entity.Member;
import {{base_package}}.service.MagicLinkService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;
import jakarta.servlet.http.HttpSession;
import java.util.List;

@Controller
@RequestMapping("/auth")
@Slf4j
@RequiredArgsConstructor
public class MagicLinkAuthController {

    private final MagicLinkService magicLinkService;

    /** GET /auth/magic-link — show the email entry form */
    @GetMapping("/magic-link")
    public String showForm() {
        return "auth/magic-link-form";
    }

    /** POST /auth/magic-link — send the magic link */
    @PostMapping("/magic-link")
    public String sendLink(@RequestParam String email,
                           RedirectAttributes redirectAttributes) {
        if (email == null || email.isBlank() || !email.contains("@")) {
            redirectAttributes.addFlashAttribute("error", "Please enter a valid email address.");
            return "redirect:/auth/magic-link";
        }
        try {
            magicLinkService.sendMagicLink(email.trim().toLowerCase());
            return "redirect:/auth/magic-link/sent?email=" + java.net.URLEncoder.encode(email, "UTF-8");
        } catch (Exception e) {
            log.error("Error sending magic link to {}: {}", email, e.getMessage());
            redirectAttributes.addFlashAttribute("error", "Failed to send link. Please try again.");
            return "redirect:/auth/magic-link";
        }
    }

    /** GET /auth/magic-link/sent — confirmation page */
    @GetMapping("/magic-link/sent")
    public String sent(@RequestParam String email, Model model) {
        model.addAttribute("email", email);
        return "auth/magic-link-sent";
    }

    /** GET /auth/verify?token=... — validate and log in */
    @GetMapping("/verify")
    public String verify(@RequestParam String token,
                         HttpSession session,
                         RedirectAttributes redirectAttributes) {
        try {
            Member member = magicLinkService.validateToken(token);

            // Build Spring Security authentication manually
            String springRole = "ROLE_" + member.getRole().name();
            var auth = new UsernamePasswordAuthenticationToken(
                member.getEmail(),
                null,
                List.of(new SimpleGrantedAuthority(springRole))
            );
            SecurityContextHolder.getContext().setAuthentication(auth);
            session.setAttribute("memberId", member.getId());
            session.setAttribute("memberEmail", member.getEmail());
            session.setAttribute("memberRole", member.getRole().name());

            if (member.getStatus() == Member.Status.PENDING) {
                return "redirect:/pending";
            }
            return "redirect:/dashboard";

        } catch (IllegalArgumentException e) {
            redirectAttributes.addFlashAttribute("error", e.getMessage());
            return "redirect:/auth/magic-link";
        }
    }
}
```

---

## Step 9 — Thymeleaf templates

### `templates/auth/magic-link-form.html`

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8" />
    <title>Sign In</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-50 flex items-center justify-center min-h-screen">
    <div class="bg-white rounded-2xl shadow-sm border border-gray-200 p-10 w-full max-w-md">
        <h1 class="text-2xl font-bold text-gray-800 mb-2">Sign in</h1>
        <p class="text-gray-500 mb-8 text-sm">Enter your email to receive a login link.</p>

        <div th:if="${error}"
             class="bg-red-50 border border-red-200 text-red-700 rounded-lg px-4 py-3 mb-6 text-sm"
             th:text="${error}">Error</div>

        <form th:action="@{/auth/magic-link}" method="post">
            <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}" />
            <div class="mb-4">
                <label for="email" class="block text-sm font-medium text-gray-700 mb-1">Email address</label>
                <input type="email" id="email" name="email" required autofocus
                       class="w-full px-4 py-3 border border-gray-300 rounded-lg focus:ring-2
                              focus:ring-blue-500 focus:border-transparent outline-none text-sm"
                       placeholder="you@example.com" />
            </div>
            <button type="submit"
                    class="w-full bg-blue-600 text-white py-3 rounded-lg font-medium
                           hover:bg-blue-700 transition text-sm">
                Send Login Link
            </button>
        </form>
    </div>
</body>
</html>
```

### `templates/auth/magic-link-sent.html`

```html
<!DOCTYPE html>
<html lang="en" xmlns:th="http://www.thymeleaf.org">
<head>
    <meta charset="UTF-8" />
    <title>Check Your Email</title>
    <script src="https://cdn.tailwindcss.com"></script>
</head>
<body class="bg-gray-50 flex items-center justify-center min-h-screen">
    <div class="text-center max-w-md px-6">
        <div class="text-5xl mb-6">📬</div>
        <h1 class="text-2xl font-bold text-gray-800 mb-4">Check your email</h1>
        <p class="text-gray-500 mb-2">
            We sent a login link to
            <strong th:text="${email}" class="text-gray-700">email</strong>.
        </p>
        <p class="text-gray-400 text-sm">The link expires in 15 minutes.</p>
        <div class="mt-8">
            <a href="/auth/magic-link" class="text-blue-600 hover:underline text-sm">
                ← Try a different email
            </a>
        </div>
    </div>
</body>
</html>
```

---

## Step 10 — Mail config in `application-local.yml`

Add MailHog config for local development (or log-mode if MailHog is not installed):

```yaml
spring:
  mail:
    host: localhost
    port: 1025
    username: ''
    password: ''
    properties:
      mail:
        smtp:
          auth: false
          starttls:
            enable: false

app:
  magic-link:
    expiry-minutes: 15
```

Tell the user:
```
For local email testing, install MailHog:
  brew install mailhog
  mailhog

Then visit http://localhost:8025 to see sent emails.

If you don't want MailHog, magic link tokens are also logged to the console at DEBUG level.
```

---

## Step 11 — `application-prod.yml` mail config

```yaml
spring:
  mail:
    host: ${SMTP_HOST}
    port: ${SMTP_PORT:587}
    username: ${SMTP_USERNAME}
    password: ${SMTP_PASSWORD}
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true
```

---

## Step 12 — Security config: permit auth routes

Update `SecurityConfig` to allow magic link routes:

```java
.requestMatchers("/auth/magic-link", "/auth/magic-link/sent", "/auth/verify").permitAll()
```

Add `@EnableScheduling` to the main application class or a config class for the cleanup cron.

---

## Step 13 — Unit tests for `MagicLinkService`

If `test_mode` is NOT `"token-save"`:

```java
package {{base_package}}.service;

import {{base_package}}.entity.MagicLink;
import {{base_package}}.entity.Member;
import {{base_package}}.repository.MagicLinkRepository;
import {{base_package}}.repository.MemberRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import java.time.OffsetDateTime;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.ArgumentMatchers.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class MagicLinkServiceTest {

    @Mock MagicLinkRepository magicLinkRepository;
    @Mock MemberRepository memberRepository;
    @Mock org.springframework.mail.javamail.JavaMailSender mailSender;
    @InjectMocks MagicLinkService magicLinkService;

    @Test
    void validateToken_validToken_returnsMember() {
        Member member = Member.builder()
            .email("user@example.com").role(Member.Role.MEMBER).status(Member.Status.ACTIVE).build();
        MagicLink link = MagicLink.builder()
            .token("valid-token")
            .email("user@example.com")
            .expiresAt(OffsetDateTime.now().plusMinutes(10))
            .used(false)
            .build();

        when(magicLinkRepository.findByToken("valid-token")).thenReturn(Optional.of(link));
        when(memberRepository.findByEmail("user@example.com")).thenReturn(Optional.of(member));
        when(magicLinkRepository.save(any())).thenReturn(link);

        Member result = magicLinkService.validateToken("valid-token");

        assertThat(result.getEmail()).isEqualTo("user@example.com");
        assertThat(link.isUsed()).isTrue();
    }

    @Test
    void validateToken_expiredToken_throws() {
        MagicLink link = MagicLink.builder()
            .token("expired-token")
            .email("user@example.com")
            .expiresAt(OffsetDateTime.now().minusMinutes(5))
            .used(false)
            .build();
        when(magicLinkRepository.findByToken("expired-token")).thenReturn(Optional.of(link));

        assertThatThrownBy(() -> magicLinkService.validateToken("expired-token"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("expired");
    }

    @Test
    void validateToken_usedToken_throws() {
        MagicLink link = MagicLink.builder()
            .token("used-token")
            .email("user@example.com")
            .expiresAt(OffsetDateTime.now().plusMinutes(10))
            .used(true)
            .build();
        when(magicLinkRepository.findByToken("used-token")).thenReturn(Optional.of(link));

        assertThatThrownBy(() -> magicLinkService.validateToken("used-token"))
            .isInstanceOf(IllegalArgumentException.class)
            .hasMessageContaining("already been used");
    }

    @Test
    void validateToken_unknownToken_throws() {
        when(magicLinkRepository.findByToken("bad-token")).thenReturn(Optional.empty());

        assertThatThrownBy(() -> magicLinkService.validateToken("bad-token"))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

---

## Step 14 — Verify

Apply `test_mode`:
- `"token-save"` → Skip.
- `"build-only"` → Run `./gradlew build -x test`.
- `"build-and-test"` → Run `./gradlew build`.

---

## Step 15 — Git commits

```bash
git add -A
git commit -m "feat: add magic link email authentication"
```

---

## Step 16 — Update `installed_modules`

Add `"auth-magic-link"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
Magic link auth is ready. Start MailHog (brew services start mailhog) then visit:
  http://localhost:8080/auth/magic-link

Next steps:
- springboot-membership → member profiles and admin controls
- springboot-mail → full email template system
```
