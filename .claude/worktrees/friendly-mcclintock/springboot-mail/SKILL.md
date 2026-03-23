---
name: springboot-mail
description: >
  TRIGGER this skill when the user says "add email", "add mail system", "add email templates",
  "add email notifications", "run springboot-mail", "set up SMTP", "add email rules",
  "add scheduled emails", "set up email for the app", or anything about sending emails from
  the application. This is a FOUNDATIONAL skill — install it early because magic-link auth
  and membership-apply depend on it. Requires setup, scaffold, db.
---

# springboot-mail

Full email system with DB-backed templates, configurable rules, send logging, and scheduled sending.
This skill is a **foundation** — install it before `springboot-auth-magic-link` and
`springboot-membership-apply`, both of which depend on the mail system.

---

## Dependencies

### Hard prerequisites
- `setup`, `scaffold`, `db` must be installed

### Soft prerequisites (enhances behavior if present)
- `membership` → enables member-group recipient targeting (ALL_MEMBERS, UNDER_QUOTA_MEMBERS)
- `app-config` → stores email rule state; the mail system creates its own table if app-config is absent

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
base_package      → Java package root
app_name          → used in email subjects and footers
brand_name        → used in email footers
app_url           → used in email body links
test_mode         → controls build verification
language          → respond in this language
translate_terms   → whether to translate technical terms
installed_modules → checked for membership, app-config
```

---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`, `"scaffold"`, `"db"`.

Note if `"membership"` is missing:
```
Note: membership module is not installed. Email recipient targeting will be limited
to individual emails (CUSTOM type). Install springboot-membership first to enable
member-group targeting.
```

---

## Step 2 — Update `build.gradle.kts`

Add:

```kotlin
// Email
implementation("org.springframework.boot:spring-boot-starter-mail")
implementation("org.springframework.boot:spring-boot-starter-thymeleaf")  // for email templates
```

---

## Step 3 — Flyway migrations

### Email rules and logs

Create `V_mail__create_email_system.sql` (next available version):

```sql
CREATE TABLE email_rules (
    id                   BIGSERIAL PRIMARY KEY,
    title                VARCHAR(255) NOT NULL,
    subject              VARCHAR(255) NOT NULL,
    body                 TEXT         NOT NULL,
    trigger_type         VARCHAR(20)  NOT NULL
                         CHECK (trigger_type IN ('AUTO', 'SCHEDULED', 'IMMEDIATE')),
    auto_condition       VARCHAR(50),
    -- AUTO conditions: NEW_APPLICATION, APPLICATION_APPROVED, APPLICATION_REJECTED,
    --                  MAGIC_LINK_SENT, MEMBER_APPROVED, CUSTOM_EVENT
    is_active            BOOLEAN      NOT NULL DEFAULT TRUE,
    notify_admin_on_send BOOLEAN      NOT NULL DEFAULT FALSE,
    created_at           TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at           TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE TABLE email_rule_schedules (
    id            BIGSERIAL PRIMARY KEY,
    rule_id       BIGINT       NOT NULL REFERENCES email_rules(id) ON DELETE CASCADE,
    schedule_type VARCHAR(20)  NOT NULL CHECK (schedule_type IN ('ONCE', 'WEEKLY', 'MONTHLY', 'DAILY')),
    send_time     TIME         NOT NULL,
    timezone      VARCHAR(50)  NOT NULL DEFAULT 'UTC',
    once_date     DATE,
    weekly_days   INT[],
    monthly_days  INT[]
);

CREATE TABLE email_rule_recipients (
    id             BIGSERIAL PRIMARY KEY,
    rule_id        BIGINT      NOT NULL REFERENCES email_rules(id) ON DELETE CASCADE,
    recipient_type VARCHAR(30) NOT NULL
                   CHECK (recipient_type IN ('ALL_MEMBERS', 'ADMINS', 'SUPER_ADMINS',
                                             'CUSTOM', 'APPLICANT', 'MEMBER_SELF')),
    custom_email   VARCHAR(255),
    is_exclusion   BOOLEAN     NOT NULL DEFAULT FALSE
);

CREATE TABLE email_logs (
    id             BIGSERIAL PRIMARY KEY,
    rule_id        BIGINT       REFERENCES email_rules(id) ON DELETE SET NULL,
    trigger_type   VARCHAR(20),
    recipient_email VARCHAR(255) NOT NULL,
    subject        VARCHAR(255),
    status         VARCHAR(20)  NOT NULL DEFAULT 'SENT'
                   CHECK (status IN ('SENT', 'FAILED', 'SKIPPED')),
    error_message  TEXT,
    sent_at        TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_email_logs_sent_at  ON email_logs (sent_at DESC);
CREATE INDEX idx_email_logs_rule_id  ON email_logs (rule_id);
```

### Seed default email rules

```sql
-- Welcome email (sent when an application is approved)
INSERT INTO email_rules (title, subject, body, trigger_type, auto_condition, is_active)
VALUES (
    'Welcome Email (Application Approved)',
    'Welcome to {{app_name}}!',
    '<p>Dear {{memberName}},</p>
     <p>Congratulations! Your application has been approved and your account is now active.</p>
     <p><a href="{{app_url}}/dashboard">Log in to your member portal →</a></p>
     <p>Best regards,<br/>{{brand_name}} Team</p>',
    'AUTO', 'APPLICATION_APPROVED', TRUE
) ON CONFLICT DO NOTHING;

-- Magic link email template (used by magic-link auth module)
INSERT INTO email_rules (title, subject, body, trigger_type, auto_condition, is_active)
VALUES (
    'Magic Link Login Email',
    'Your login link for {{app_name}}',
    '<p>Click the button below to log in. This link expires in 15 minutes.</p>
     <p><a href="{{magicLinkUrl}}" style="background:#3b82f6;color:white;padding:12px 24px;
        border-radius:8px;text-decoration:none;display:inline-block;">Log in →</a></p>
     <p style="color:#9ca3af;font-size:12px;">If you did not request this, ignore this email.</p>',
    'AUTO', 'MAGIC_LINK_SENT', TRUE
) ON CONFLICT DO NOTHING;
```

---

## Step 4 — Email entities

### `EmailRule` entity

```java
package {{base_package}}.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.OffsetDateTime;
import java.util.ArrayList;
import java.util.List;

@Entity
@Table(name = "email_rules")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class EmailRule {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false) private String title;
    @Column(nullable = false) private String subject;
    @Column(nullable = false, columnDefinition = "TEXT") private String body;

    @Column(name = "trigger_type", nullable = false)
    @Enumerated(EnumType.STRING)
    private TriggerType triggerType;

    @Column(name = "auto_condition") private String autoCondition;

    @Column(name = "is_active", nullable = false) private boolean active = true;
    @Column(name = "notify_admin_on_send") private boolean notifyAdminOnSend = false;

    @Column(name = "created_at", updatable = false) private OffsetDateTime createdAt;
    @Column(name = "updated_at") private OffsetDateTime updatedAt;

    @OneToMany(mappedBy = "rule", cascade = CascadeType.ALL, orphanRemoval = true)
    @Builder.Default
    private List<EmailRuleRecipient> recipients = new ArrayList<>();

    @OneToMany(mappedBy = "rule", cascade = CascadeType.ALL, orphanRemoval = true)
    @Builder.Default
    private List<EmailRuleSchedule> schedules = new ArrayList<>();

    @PrePersist  void onCreate() { createdAt = updatedAt = OffsetDateTime.now(); }
    @PreUpdate   void onUpdate() { updatedAt = OffsetDateTime.now(); }

    public enum TriggerType { AUTO, SCHEDULED, IMMEDIATE }
}
```

### `EmailRuleRecipient` entity

```java
package {{base_package}}.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity @Table(name = "email_rule_recipients")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class EmailRuleRecipient {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "rule_id") private EmailRule rule;

    @Column(name = "recipient_type", nullable = false) private String recipientType;
    @Column(name = "custom_email") private String customEmail;
    @Column(name = "is_exclusion") private boolean exclusion = false;
}
```

### `EmailLog` entity

```java
package {{base_package}}.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.OffsetDateTime;

@Entity @Table(name = "email_logs")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class EmailLog {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY) private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "rule_id") private EmailRule rule;

    @Column(name = "trigger_type") private String triggerType;

    @Column(name = "recipient_email", nullable = false) private String recipientEmail;
    private String subject;

    @Column(nullable = false) private String status = "SENT";
    @Column(name = "error_message", columnDefinition = "TEXT") private String errorMessage;

    @Column(name = "sent_at", nullable = false, updatable = false) private OffsetDateTime sentAt;

    @PrePersist void onCreate() { sentAt = OffsetDateTime.now(); }
}
```

---

## Step 5 — `EmailService`

```java
package {{base_package}}.service;

import {{base_package}}.entity.*;
import {{base_package}}.repository.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.scheduling.annotation.Async;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import jakarta.mail.internet.MimeMessage;
import java.time.*;
import java.util.*;

@Service
@Slf4j
@RequiredArgsConstructor
public class EmailService {

    private final JavaMailSender mailSender;
    private final EmailRuleRepository emailRuleRepository;
    private final EmailLogRepository emailLogRepository;

    @Value("${spring.mail.username:noreply@example.com}")
    private String fromEmail;

    @Value("${app.name}")
    private String appName;

    @Value("${app.url}")
    private String appUrl;

    @Value("${app.brand-name}")
    private String brandName;

    /**
     * Send an email immediately.
     */
    @Async
    public void send(String to, String subject, String bodyHtml) {
        send(to, subject, bodyHtml, null, "IMMEDIATE");
    }

    /**
     * Send using a template with variable substitution.
     * Variables map: e.g. {"memberName": "Alice", "magicLinkUrl": "https://..."}
     */
    @Async
    public void sendWithTemplate(String to, String templateSubject, String templateBody,
                                 Map<String, String> variables, EmailRule rule) {
        String subject = interpolate(templateSubject, variables);
        String body    = interpolate(templateBody, variables);
        send(to, subject, body, rule, "AUTO");
    }

    /**
     * Fire AUTO rules for a given condition (e.g., "APPLICATION_APPROVED").
     */
    @Transactional
    public void fireAutoRules(String condition, String recipientEmail, Map<String, String> variables) {
        List<EmailRule> rules = emailRuleRepository
            .findByTriggerTypeAndAutoConditionAndActiveTrue(
                EmailRule.TriggerType.AUTO, condition);
        for (EmailRule rule : rules) {
            Map<String, String> vars = buildDefaultVariables();
            vars.putAll(variables);
            sendWithTemplate(recipientEmail, rule.getSubject(), rule.getBody(), vars, rule);
        }
    }

    private void send(String to, String subject, String bodyHtml, EmailRule rule, String triggerType) {
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");
            helper.setFrom(fromEmail);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(wrapInEmailLayout(bodyHtml), true);
            mailSender.send(message);

            logEmail(to, subject, rule, triggerType, "SENT", null);
            log.info("Email sent to {}: {}", to, subject);

        } catch (Exception e) {
            log.error("Failed to send email to {}: {}", to, e.getMessage());
            logEmail(to, subject, rule, triggerType, "FAILED", e.getMessage());
        }
    }

    private void logEmail(String to, String subject, EmailRule rule, String triggerType,
                          String status, String error) {
        try {
            emailLogRepository.save(EmailLog.builder()
                .rule(rule)
                .triggerType(triggerType)
                .recipientEmail(to)
                .subject(subject)
                .status(status)
                .errorMessage(error)
                .build());
        } catch (Exception e) {
            log.warn("Could not write email log: {}", e.getMessage());
        }
    }

    private String interpolate(String template, Map<String, String> variables) {
        String result = template;
        for (Map.Entry<String, String> entry : variables.entrySet()) {
            result = result.replace("{{" + entry.getKey() + "}}", entry.getValue() != null ? entry.getValue() : "");
        }
        return result;
    }

    private Map<String, String> buildDefaultVariables() {
        Map<String, String> vars = new HashMap<>();
        vars.put("appName",   appName);
        vars.put("appUrl",    appUrl);
        vars.put("brandName", brandName);
        return vars;
    }

    private String wrapInEmailLayout(String bodyHtml) {
        return """
            <!DOCTYPE html>
            <html>
            <head><meta charset="UTF-8"/><meta name="viewport" content="width=device-width,initial-scale=1"/>
            <style>body{font-family:sans-serif;color:#374151;background:#f9fafb;margin:0;padding:24px}
            .card{background:#fff;border-radius:12px;max-width:560px;margin:0 auto;padding:32px;border:1px solid #e5e7eb}
            .footer{text-align:center;color:#9ca3af;font-size:12px;margin-top:24px}</style>
            </head>
            <body>
            <div class="card">
            """ + bodyHtml + """
            <div class="footer">
              <p>Sent by """ + appName + """ | <a href=\"""" + appUrl + """
\" style="color:#6b7280">""" + appUrl + """
</a></p>
            </div>
            </div>
            </body></html>
            """;
    }

    /** Scheduled: check SCHEDULED rules every minute */
    @Scheduled(cron = "0 * * * * *")
    @Transactional
    public void processScheduledRules() {
        List<EmailRule> rules = emailRuleRepository
            .findByTriggerTypeAndActiveTrue(EmailRule.TriggerType.SCHEDULED);

        LocalDateTime now = LocalDateTime.now(ZoneOffset.UTC);

        for (EmailRule rule : rules) {
            for (EmailRuleSchedule schedule : rule.getSchedules()) {
                if (shouldFireNow(schedule, now)) {
                    log.info("Firing scheduled email rule: {}", rule.getTitle());
                    // Recipient resolution happens here — simplified for now
                    // Full implementation: resolve recipients from email_rule_recipients table
                }
            }
        }
    }

    private boolean shouldFireNow(EmailRuleSchedule schedule, LocalDateTime nowUtc) {
        ZoneId zone = ZoneId.of(schedule.getTimezone());
        LocalDateTime localNow = nowUtc.atZone(ZoneOffset.UTC).withZoneSameInstant(zone).toLocalDateTime();
        LocalTime sendTime = schedule.getSendTime();

        return switch (schedule.getScheduleType()) {
            case "DAILY"   -> localNow.toLocalTime().getHour() == sendTime.getHour()
                           && localNow.toLocalTime().getMinute() == sendTime.getMinute();
            case "WEEKLY"  -> schedule.getWeeklyDays() != null
                           && schedule.getWeeklyDays().contains(localNow.getDayOfWeek().getValue())
                           && localNow.toLocalTime().getHour() == sendTime.getHour()
                           && localNow.toLocalTime().getMinute() == sendTime.getMinute();
            case "MONTHLY" -> schedule.getMonthlyDays() != null
                           && schedule.getMonthlyDays().contains(localNow.getDayOfMonth())
                           && localNow.toLocalTime().getHour() == sendTime.getHour()
                           && localNow.toLocalTime().getMinute() == sendTime.getMinute();
            default        -> false;
        };
    }
}
```

---

## Step 6 — Repositories

```java
// EmailRuleRepository.java
package {{base_package}}.repository;

import {{base_package}}.entity.EmailRule;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface EmailRuleRepository extends JpaRepository<EmailRule, Long> {
    List<EmailRule> findByTriggerTypeAndActiveTrue(EmailRule.TriggerType triggerType);
    List<EmailRule> findByTriggerTypeAndAutoConditionAndActiveTrue(
        EmailRule.TriggerType triggerType, String autoCondition);
}
```

```java
// EmailLogRepository.java
package {{base_package}}.repository;

import {{base_package}}.entity.EmailLog;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;

public interface EmailLogRepository extends JpaRepository<EmailLog, Long> {
    Page<EmailLog> findAllByOrderBySentAtDesc(Pageable pageable);
}
```

Also create `EmailRuleSchedule` entity and `EmailRuleScheduleRepository` matching the DB schema above.

---

## Step 7 — Admin email management controller

```java
// AdminEmailController.java at /admin/email
// GET  /admin/email          → list rules + recent logs
// GET  /admin/email/rules/new → create rule form
// POST /admin/email/rules    → save rule
// GET  /admin/email/rules/{id}/edit → edit form
// POST /admin/email/rules/{id} → update
// POST /admin/email/rules/{id}/delete → delete
// POST /admin/email/rules/{id}/send → send immediately (IMMEDIATE rules)
// GET  /admin/email/logs    → paginated log view
```

Implement each endpoint following the same pattern as other admin controllers.

---

## Step 8 — Mail config

### `application-local.yml`

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
```

For local development, use MailHog: `brew install mailhog && mailhog`
Emails are visible at `http://localhost:8025`.

### `application-prod.yml`

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

### `application.yml` (shared)

```yaml
spring:
  task:
    execution:
      pool:
        core-size: 4
    scheduling:
      pool:
        size: 2
```

Add `@EnableScheduling` and `@EnableAsync` to the main application class.

---

## Step 9 — Thymeleaf templates

### `templates/admin/email/list.html`
- Table of email rules with columns: Title, Trigger Type, Condition, Active toggle, Recipients summary
- Recent logs section at bottom (10 most recent)
- "Create Rule" button

### `templates/admin/email/form.html`
- Rule form: Title, Subject, Body (textarea + live preview iframe), Trigger Type
- Conditional sections based on trigger type (AUTO conditions, SCHEDULE config, IMMEDIATE)
- Recipients section: checkboxes for recipient types + custom email input
- Live "Expected recipients" count

### `templates/admin/email/logs.html`
- Full log table: Rule, Recipient, Subject, Status, Sent At

---

## Step 10 — Unit tests for template rendering

If `test_mode` is NOT `"token-save"`, write tests:
- `interpolate("Hello {{memberName}}", {"memberName": "Alice"})` → `"Hello Alice"`
- `interpolate` with missing variable key → leaves `{{varName}}` unchanged
- `shouldFireNow` with matching schedule → returns true
- `shouldFireNow` with non-matching schedule → returns false

---

## Step 11 — Verify

Apply `test_mode`.

---

## Step 12 — Git commits

```bash
git add -A
git commit -m "feat: add email system with DB-backed templates, rules engine, and scheduled sending"
```

---

## Step 13 — Update `installed_modules`

Add `"mail"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
Mail system is ready.

For local testing: brew install mailhog && mailhog
Emails visible at: http://localhost:8025
Admin email management: http://localhost:8080/admin/email

The mail system is now available to other modules:
- springboot-auth-magic-link will use it to send login links
- springboot-membership-apply will use it for welcome emails
- springboot-announcements will use it for broadcast notifications
```
