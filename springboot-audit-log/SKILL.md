---
name: springboot-audit-log
description: >
  TRIGGER this skill when the user says "add audit log", "add audit trail", "track changes",
  "run springboot-audit-log", "log who changed what", "add change history", "add admin audit",
  "track entity changes", "add action logging", or anything about recording which user made
  what change to which entity and when. Requires setup, scaffold, db.
---

# springboot-audit-log

Adds an audit trail: records who did what to which entity, when. Any service method can log
audit entries via `AuditLogService`. Admins can browse logs filtered by entity, actor, or date.

---

## Dependencies

### Hard prerequisites
- `setup`, `scaffold`, `db` must be installed

### Soft prerequisites
- `membership` → enables actor name/email resolution from the members table
- `admin-portal` → audit logs appear in the admin panel naturally

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
base_package      → Java package root
test_mode         → controls build verification
language          → respond in this language for ALL output (questions, explanations,
                    status messages, code comments, and user-facing copy in generated files)
                    IMPORTANT: if "zh-TW", "traditional-chinese", or "繁體中文":
                      use 繁體中文 (Traditional Chinese) throughout
                      NEVER use Simplified Chinese (简体中文) — they are different writing systems
                      Key differences: 體/体, 語/语, 資料/数据, 設定/设定, 確認/确认, 請/请
translate_terms   → whether to translate technical terms
beginner_friendly → if true, explain technical terms and decisions as you work
installed_modules → checked for membership
```

> **Language activation**: After reading `language` above, switch ALL your responses to that
> language immediately — including every question, status message, explanation, and all
> human-readable text in any generated files (HTML, Thymeleaf templates, SQL seed labels).
> If `language` is not set, ask: "What language should I use? (English / 繁體中文 / other)"



---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`, `"scaffold"`, `"db"`.

---

## Beginner-Friendly Mode

If `beginner_friendly` is `true` in `.spring-config.json`, explain key concepts as you work. Examples:
- When implementing `@EntityListeners`: "JPA entity listeners are methods that Spring calls automatically before/after database operations — before an `UPDATE`, we capture the old and new values and write an audit log entry."
- When using `SecurityContextHolder`: "Spring Security stores the currently logged-in user in `SecurityContextHolder`. We read it in the audit listener to know who made the change — even though the listener has no direct reference to the HTTP request."
- When building the admin log view: "The audit log query is just `SELECT * FROM audit_log WHERE entity_type = ? AND entity_id = ?` ordered by time. We add filters so admins can drill into 'all changes to member #42' or 'all actions by admin@example.com'."

## Step 2 — Flyway migration

Create `V_audit__create_audit_log.sql` (next available version):

```sql
CREATE TABLE audit_log (
    id           BIGSERIAL PRIMARY KEY,
    actor_id     BIGINT,                    -- nullable: null if system-initiated
    actor_email  VARCHAR(255),
    action       VARCHAR(100) NOT NULL,     -- e.g., "MEMBER_STATUS_CHANGED", "APPLICATION_APPROVED"
    entity_type  VARCHAR(100) NOT NULL,     -- e.g., "Member", "Application", "Submission"
    entity_id    VARCHAR(255),              -- the entity's ID (as string to support any type)
    old_value    JSONB,                     -- previous state (can be null)
    new_value    JSONB,                     -- new state (can be null)
    ip_address   VARCHAR(50),
    user_agent   VARCHAR(500),
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_audit_log_actor_id     ON audit_log (actor_id);
CREATE INDEX idx_audit_log_entity       ON audit_log (entity_type, entity_id);
CREATE INDEX idx_audit_log_action       ON audit_log (action);
CREATE INDEX idx_audit_log_created_at   ON audit_log (created_at DESC);
```

---

## Step 3 — `AuditLogEntry` entity

```java
package {{base_package}}.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;
import java.time.OffsetDateTime;
import java.util.Map;

@Entity
@Table(name = "audit_log")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class AuditLogEntry {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "actor_id")
    private Long actorId;

    @Column(name = "actor_email")
    private String actorEmail;

    @Column(nullable = false)
    private String action;

    @Column(name = "entity_type", nullable = false)
    private String entityType;

    @Column(name = "entity_id")
    private String entityId;

    @Column(name = "old_value", columnDefinition = "jsonb")
    @JdbcTypeCode(SqlTypes.JSON)
    private Map<String, Object> oldValue;

    @Column(name = "new_value", columnDefinition = "jsonb")
    @JdbcTypeCode(SqlTypes.JSON)
    private Map<String, Object> newValue;

    @Column(name = "ip_address")
    private String ipAddress;

    @Column(name = "user_agent")
    private String userAgent;

    @Column(name = "created_at", nullable = false, updatable = false)
    private OffsetDateTime createdAt;

    @PrePersist
    void onCreate() { createdAt = OffsetDateTime.now(); }
}
```

---

## Step 4 — `AuditLogRepository`

```java
package {{base_package}}.repository;

import {{base_package}}.entity.AuditLogEntry;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import java.time.OffsetDateTime;
import java.util.List;

public interface AuditLogRepository extends JpaRepository<AuditLogEntry, Long> {

    Page<AuditLogEntry> findByEntityTypeAndEntityIdOrderByCreatedAtDesc(
        String entityType, String entityId, Pageable pageable);

    Page<AuditLogEntry> findByActorEmailOrderByCreatedAtDesc(
        String actorEmail, Pageable pageable);

    @Query("""
        SELECT a FROM AuditLogEntry a
        WHERE (:entityType IS NULL OR a.entityType = :entityType)
          AND (:action IS NULL OR a.action = :action)
          AND (:actorEmail IS NULL OR a.actorEmail = :actorEmail)
          AND (:since IS NULL OR a.createdAt >= :since)
        ORDER BY a.createdAt DESC
        """)
    Page<AuditLogEntry> search(
        @Param("entityType") String entityType,
        @Param("action")     String action,
        @Param("actorEmail") String actorEmail,
        @Param("since")      OffsetDateTime since,
        Pageable pageable);
}
```

---

## Step 5 — `AuditLogService`

```java
package {{base_package}}.service;

import {{base_package}}.entity.AuditLogEntry;
import {{base_package}}.repository.AuditLogRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Async;
import org.springframework.stereotype.Service;
import java.util.Map;

/**
 * Service for recording audit log entries.
 *
 * Usage pattern in other services:
 *   auditLogService.log("MEMBER_STATUS_CHANGED", "Member", memberId.toString(),
 *       actorEmail, Map.of("status", "ACTIVE"), Map.of("status", "SUSPENDED"));
 *
 * The log is written asynchronously to avoid slowing down the main operation.
 */
@Service
@Slf4j
@RequiredArgsConstructor
public class AuditLogService {

    private final AuditLogRepository auditLogRepository;

    /**
     * Log an action with before/after state.
     */
    @Async
    public void log(String action,
                    String entityType,
                    String entityId,
                    String actorEmail,
                    Map<String, Object> oldValue,
                    Map<String, Object> newValue) {
        try {
            AuditLogEntry entry = AuditLogEntry.builder()
                .action(action)
                .entityType(entityType)
                .entityId(entityId)
                .actorEmail(actorEmail)
                .oldValue(oldValue)
                .newValue(newValue)
                .build();
            auditLogRepository.save(entry);
        } catch (Exception e) {
            // Audit log failure must NEVER break the main operation
            log.error("Failed to write audit log entry: action={}, entity={}/{}, error={}",
                action, entityType, entityId, e.getMessage());
        }
    }

    /**
     * Log a simple action with no before/after state.
     */
    @Async
    public void log(String action, String entityType, String entityId, String actorEmail) {
        log(action, entityType, entityId, actorEmail, null, null);
    }

    /**
     * Log a system-initiated action (no actor).
     */
    @Async
    public void logSystem(String action, String entityType, String entityId) {
        log(action, entityType, entityId, "SYSTEM", null, null);
    }
}
```

---

## Step 6 — `@Audited` annotation for AOP-based logging

Provide an annotation + AOP aspect as an alternative for simple logging:

```java
// @Audited annotation
package {{base_package}}.audit;

import java.lang.annotation.*;

/**
 * Annotate service methods to automatically log audit entries.
 *
 * Example:
 *   @Audited(action = "MEMBER_STATUS_CHANGED", entityType = "Member")
 *   public Member updateStatus(Long id, Status status, String actorEmail) { ... }
 */
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface Audited {
    String action();
    String entityType();
}
```

```java
// AuditAspect.java
package {{base_package}}.audit;

import {{base_package}}.service.AuditLogService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.*;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;

/**
 * AOP aspect that intercepts methods annotated with @Audited.
 * Automatically captures the actor from the security context.
 *
 * Note: For fine-grained before/after state capture, use AuditLogService directly
 * in the service method — the AOP approach only logs method invocation and outcome.
 */
@Aspect
@Component
@Slf4j
@RequiredArgsConstructor
public class AuditAspect {

    private final AuditLogService auditLogService;

    @Around("@annotation(audited)")
    public Object auditMethod(ProceedingJoinPoint pjp, Audited audited) throws Throwable {
        String actorEmail = resolveActorEmail();
        Object[] args = pjp.getArgs();

        // Try to extract entity ID from the first Long argument
        String entityId = null;
        for (Object arg : args) {
            if (arg instanceof Long id) {
                entityId = id.toString();
                break;
            }
        }

        try {
            Object result = pjp.proceed();
            auditLogService.log(audited.action(), audited.entityType(), entityId, actorEmail);
            return result;
        } catch (Throwable t) {
            auditLogService.log(audited.action() + "_FAILED", audited.entityType(), entityId, actorEmail);
            throw t;
        }
    }

    private String resolveActorEmail() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null || !auth.isAuthenticated()) return "ANONYMOUS";
        return auth.getName();
    }
}
```

Add `@EnableAspectJAutoProxy` to main application class and add AOP dependency:

```kotlin
// build.gradle.kts
implementation("org.springframework.boot:spring-boot-starter-aop")
```

---

## Step 7 — Admin audit log controller

```java
package {{base_package}}.controller.admin;

import {{base_package}}.repository.AuditLogRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;
import java.time.OffsetDateTime;

@Controller
@RequestMapping("/admin/audit-log")
@PreAuthorize("hasAnyRole('ADMIN', 'SUPER_ADMIN')")
@RequiredArgsConstructor
public class AdminAuditLogController {

    private final AuditLogRepository auditLogRepository;

    @GetMapping
    public String list(@RequestParam(required = false) String entityType,
                       @RequestParam(required = false) String action,
                       @RequestParam(required = false) String actorEmail,
                       @RequestParam(required = false) Integer days,
                       @RequestParam(defaultValue = "0") int page,
                       Model model) {
        OffsetDateTime since = days != null ? OffsetDateTime.now().minusDays(days) : null;

        model.addAttribute("entries",
            auditLogRepository.search(
                entityType, action, actorEmail, since,
                PageRequest.of(page, 50, Sort.by("createdAt").descending())
            ));
        model.addAttribute("entityType", entityType);
        model.addAttribute("action", action);
        model.addAttribute("actorEmail", actorEmail);
        model.addAttribute("days", days);
        return "admin/audit-log/list";
    }
}
```

---

## Step 8 — Integrate with existing services

Show the user how to wire audit logging into existing services. Example for `MemberService`:

```java
// In MemberService.updateStatus():
@Transactional
@Audited(action = "MEMBER_STATUS_CHANGED", entityType = "Member")
public Member updateStatus(Long id, Member.Status newStatus, String actorEmail) {
    Member member = findById(id);
    Member.Status oldStatus = member.getStatus();

    member.setStatus(newStatus);
    memberRepository.save(member);

    // Fine-grained log with before/after state
    auditLogService.log(
        "MEMBER_STATUS_CHANGED", "Member", id.toString(), actorEmail,
        Map.of("status", oldStatus.name()),
        Map.of("status", newStatus.name())
    );

    return member;
}
```

Recommended audit actions to add:
- `MEMBER_STATUS_CHANGED` → in MemberService
- `MEMBER_ROLE_CHANGED` → in MemberService
- `APPLICATION_APPROVED` / `APPLICATION_REJECTED` → in ApplicationService
- `SUBMISSION_APPROVED` / `SUBMISSION_REJECTED` → in SubmissionService
- `ANNOUNCEMENT_PUBLISHED` → in AnnouncementService
- `CONFIG_UPDATED` → in AppConfigService

---

## Step 9 — `templates/admin/audit-log/list.html`

Audit log table:
- Columns: Date/Time, Actor, Action, Entity Type, Entity ID, Old Value, New Value
- Old/New value: show as compact JSON with a "expand" link if long
- Filters: entity type dropdown, action dropdown, actor email input, date range
- Pagination

---

## Step 10 — Unit tests

If `test_mode` is NOT `"token-save"`:

```java
// AuditLogServiceTest.java
// Test: log() saves entry to repository
// Test: log() failure does NOT throw (logs error instead)
// Test: logSystem() sets actorEmail to "SYSTEM"

// AuditAspectTest.java
// Test: method with @Audited creates audit entry on success
// Test: method with @Audited creates FAILED entry on exception (and re-throws)
```

---

## Step 11 — Verify

Apply `test_mode`.

---

## Step 12 — Git commits

```bash
git add -A
git commit -m "feat: add audit log with AuditLogService, @Audited AOP annotation, and admin view"
```

---

## Step 13 — Update `installed_modules`

Add `"audit-log"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
Audit log is ready.

Admin view: http://localhost:8080/admin/audit-log

To log actions in existing services, inject AuditLogService and call:
  auditLogService.log("ACTION_NAME", "EntityType", entityId.toString(), actorEmail, oldValue, newValue)

OR annotate service methods with @Audited for simple, automatic logging.
```
