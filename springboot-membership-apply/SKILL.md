---
name: springboot-membership-apply
description: >
  TRIGGER this skill when the user says "add application form", "add member application",
  "add public apply form", "add registration form", "run springboot-membership-apply",
  "let people apply for membership", "build the /apply page", "add onboarding form",
  or anything about a public-facing form where non-logged-in users can apply to join.
  No auth required to view or submit the form. Requires setup, scaffold, db, AND mail
  (sends welcome email when application is approved).
---

# springboot-membership-apply

Adds a public membership application form at `/apply`. No login required. Submissions are stored
in an `applications` table. Admins review at `/admin/applications` and can approve or reject.
On approval, a Member account is created and a welcome email is sent.

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
base_package      → Java package root
app_name          → shown in form heading and email content
brand_name        → shown in footer and success page
app_url           → for success page link back to main site
super_admin_email → for dev data only
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

Verify `installed_modules` contains `"setup"`, `"scaffold"`, `"db"`.

**Hard check — `mail` is required:**
If `"mail"` is NOT in `installed_modules`, stop immediately and output:
```
❌ Cannot proceed: springboot-mail is not installed.

When an application is approved, a welcome email is sent to the applicant.
Please run springboot-mail to configure email sending, then return here.
```

Do not continue until the user confirms mail is installed.

## Beginner-Friendly Mode

If `beginner_friendly` is `true` in `.spring-config.json`, explain key concepts as you work. Examples:
- When creating the applications table: "This Flyway migration creates the `applications` table — a database table that stores each person's application. Flyway runs this script automatically when the app starts."
- When adding status enum: "We use a status field (`PENDING`, `APPROVED`, `REJECTED`) to track where each application is in the review process, rather than storing raw strings."
- When sending welcome email: "On approval, we trigger the mail skill's email service to send a welcome message. This is why mail must be installed first."

---

## Step 2 — Flyway migration: `V5__create_applications.sql`

Use the next available migration version number.

```sql
CREATE TABLE applications (
    id            BIGSERIAL PRIMARY KEY,
    email         VARCHAR(255) NOT NULL,
    name          VARCHAR(255) NOT NULL,
    motivation    TEXT,
    status        VARCHAR(20)  NOT NULL DEFAULT 'PENDING'
                  CHECK (status IN ('PENDING', 'APPROVED', 'REJECTED')),
    reviewer_note TEXT,
    reviewed_by   VARCHAR(255),
    reviewed_at   TIMESTAMPTZ,
    submitted_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    ip_address    VARCHAR(50),
    extra_data    JSONB
);

CREATE INDEX idx_applications_email  ON applications (email);
CREATE INDEX idx_applications_status ON applications (status);
```

Note: `extra_data` JSONB column allows future fields without migrations.
`ip_address` is stored for duplicate/spam detection only.

---

## Step 3 — `Application` entity

```java
package {{base_package}}.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;
import java.time.OffsetDateTime;
import java.util.Map;

@Entity
@Table(name = "applications")
@Getter @Setter @NoArgsConstructor @Builder @AllArgsConstructor
public class Application {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String email;

    @Column(nullable = false)
    private String name;

    @Column(columnDefinition = "TEXT")
    private String motivation;

    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    private Status status = Status.PENDING;

    @Column(name = "reviewer_note", columnDefinition = "TEXT")
    private String reviewerNote;

    @Column(name = "reviewed_by")
    private String reviewedBy;

    @Column(name = "reviewed_at")
    private OffsetDateTime reviewedAt;

    @Column(name = "submitted_at", nullable = false, updatable = false)
    private OffsetDateTime submittedAt;

    @Column(name = "ip_address")
    private String ipAddress;

    @Column(name = "extra_data", columnDefinition = "jsonb")
    @JdbcTypeCode(SqlTypes.JSON)
    private Map<String, Object> extraData;

    @PrePersist
    void onCreate() {
        submittedAt = OffsetDateTime.now();
    }

    public enum Status {
        PENDING, APPROVED, REJECTED
    }
}
```

---

## Step 4 — `ApplicationRepository`

```java
package {{base_package}}.repository;

import {{base_package}}.entity.Application;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface ApplicationRepository extends JpaRepository<Application, Long> {
    List<Application> findByEmail(String email);
    Page<Application> findByStatus(Application.Status status, Pageable pageable);
    boolean existsByEmailAndStatus(String email, Application.Status status);
    long countByStatus(Application.Status status);
}
```

---

## Step 5 — `ApplicationService`

```java
package {{base_package}}.service;

import {{base_package}}.entity.Application;
import {{base_package}}.entity.Member;
import {{base_package}}.repository.ApplicationRepository;
import {{base_package}}.repository.MemberRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.OffsetDateTime;

@Service
@Slf4j
@RequiredArgsConstructor
public class ApplicationService {

    private final ApplicationRepository applicationRepository;
    private final MemberRepository memberRepository;

    /**
     * Submit a new application. Prevents duplicate active/pending applications.
     */
    @Transactional
    public Application submit(String email, String name, String motivation, String ipAddress) {
        email = email.trim().toLowerCase();

        // Prevent duplicate pending applications
        if (applicationRepository.existsByEmailAndStatus(email, Application.Status.PENDING)) {
            throw new IllegalStateException(
                "An application is already pending for this email address."
            );
        }

        // Prevent re-application if already an active member
        if (memberRepository.findByEmail(email)
                .filter(m -> m.getStatus() == Member.Status.ACTIVE).isPresent()) {
            throw new IllegalStateException(
                "This email is already associated with an active member account."
            );
        }

        Application application = Application.builder()
            .email(email)
            .name(name)
            .motivation(motivation)
            .status(Application.Status.PENDING)
            .ipAddress(ipAddress)
            .build();

        return applicationRepository.save(application);
    }

    /**
     * Approve an application and create a Member account.
     */
    @Transactional
    public Member approve(Long applicationId, String reviewerEmail) {
        Application application = applicationRepository.findById(applicationId)
            .orElseThrow(() -> new IllegalArgumentException("Application not found: " + applicationId));

        application.setStatus(Application.Status.APPROVED);
        application.setReviewedBy(reviewerEmail);
        application.setReviewedAt(OffsetDateTime.now());
        applicationRepository.save(application);

        // Create member account if not already exists
        return memberRepository.findByEmail(application.getEmail())
            .map(existing -> {
                existing.setStatus(Member.Status.ACTIVE);
                existing.setName(application.getName());
                return memberRepository.save(existing);
            })
            .orElseGet(() -> memberRepository.save(
                Member.builder()
                    .email(application.getEmail())
                    .name(application.getName())
                    .role(Member.Role.MEMBER)
                    .status(Member.Status.ACTIVE)
                    .build()
            ));
    }

    /**
     * Reject an application with an optional note.
     */
    @Transactional
    public Application reject(Long applicationId, String reviewerEmail, String note) {
        Application application = applicationRepository.findById(applicationId)
            .orElseThrow(() -> new IllegalArgumentException("Application not found: " + applicationId));

        application.setStatus(Application.Status.REJECTED);
        application.setReviewedBy(reviewerEmail);
        application.setReviewedAt(OffsetDateTime.now());
        application.setReviewerNote(note);
        return applicationRepository.save(application);
    }
}
```

---

## Step 6 — `PublicApplicationController`

```java
package {{base_package}}.controller;

import {{base_package}}.service.ApplicationService;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.validation.BindingResult;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;
import jakarta.validation.Valid;

@Controller
@RequestMapping("/apply")
@RequiredArgsConstructor
public class PublicApplicationController {

    private final ApplicationService applicationService;

    @GetMapping
    public String form(Model model) {
        model.addAttribute("form", new ApplicationForm());
        return "public/apply";
    }

    @PostMapping
    public String submit(@Valid @ModelAttribute("form") ApplicationForm form,
                         BindingResult bindingResult,
                         HttpServletRequest request,
                         RedirectAttributes redirectAttributes,
                         Model model) {
        if (bindingResult.hasErrors()) {
            return "public/apply";
        }
        try {
            String ip = request.getRemoteAddr();
            applicationService.submit(form.email(), form.name(), form.motivation(), ip);
            return "redirect:/apply/success";
        } catch (IllegalStateException e) {
            model.addAttribute("form", form);
            model.addAttribute("error", e.getMessage());
            return "public/apply";
        }
    }

    @GetMapping("/success")
    public String success() {
        return "public/apply-success";
    }

    /** Simple form record for binding */
    public record ApplicationForm(
        @Email(message = "Please enter a valid email address")
        @NotBlank(message = "Email is required")
        String email,

        @NotBlank(message = "Name is required")
        String name,

        String motivation
    ) {
        public ApplicationForm() { this(null, null, null); }
    }
}
```

---

## Step 7 — `AdminApplicationController`

```java
package {{base_package}}.controller.admin;

import {{base_package}}.entity.Application;
import {{base_package}}.repository.ApplicationRepository;
import {{base_package}}.service.ApplicationService;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Controller
@RequestMapping("/admin/applications")
@RequiredArgsConstructor
public class AdminApplicationController {

    private final ApplicationRepository applicationRepository;
    private final ApplicationService applicationService;

    @GetMapping
    public String list(@RequestParam(required = false, defaultValue = "PENDING") String status,
                       @RequestParam(defaultValue = "0") int page,
                       Model model) {
        Application.Status statusFilter = Application.Status.PENDING;
        try { statusFilter = Application.Status.valueOf(status); } catch (Exception ignored) {}

        model.addAttribute("applications",
            applicationRepository.findByStatus(statusFilter,
                PageRequest.of(page, 20, Sort.by("submittedAt").descending())));
        model.addAttribute("status", status);
        model.addAttribute("pendingCount",  applicationRepository.countByStatus(Application.Status.PENDING));
        model.addAttribute("approvedCount", applicationRepository.countByStatus(Application.Status.APPROVED));
        model.addAttribute("rejectedCount", applicationRepository.countByStatus(Application.Status.REJECTED));
        return "admin/applications/list";
    }

    @GetMapping("/{id}")
    public String detail(@PathVariable Long id, Model model) {
        model.addAttribute("application",
            applicationRepository.findById(id)
                .orElseThrow(() -> new IllegalArgumentException("Application not found")));
        return "admin/applications/detail";
    }

    @PostMapping("/{id}/approve")
    public String approve(@PathVariable Long id,
                          Authentication authentication,
                          RedirectAttributes redirectAttributes) {
        try {
            applicationService.approve(id, authentication.getName());
            redirectAttributes.addFlashAttribute("success", "Application approved. Member account created.");
        } catch (Exception e) {
            redirectAttributes.addFlashAttribute("error", e.getMessage());
        }
        return "redirect:/admin/applications";
    }

    @PostMapping("/{id}/reject")
    public String reject(@PathVariable Long id,
                         @RequestParam(required = false) String note,
                         Authentication authentication,
                         RedirectAttributes redirectAttributes) {
        try {
            applicationService.reject(id, authentication.getName(), note);
            redirectAttributes.addFlashAttribute("success", "Application rejected.");
        } catch (Exception e) {
            redirectAttributes.addFlashAttribute("error", e.getMessage());
        }
        return "redirect:/admin/applications";
    }
}
```

---

## Step 8 — Thymeleaf templates

### `templates/public/apply.html`

Public application form page (no nav requiring login):
- Clear heading: "Apply for {{app_name}} Membership"
- Form fields: Email (with note about Google account), Full Name, Motivation (textarea, optional)
- Submit button
- Error message display
- Back to home link

### `templates/public/apply-success.html`

Success confirmation page:
- Heading: "Application Received!"
- Body: "Thank you, [name]! We've received your application and will review it soon."
- Subtext: "You'll receive an email notification when your application is reviewed."
- Link back to home

### `templates/admin/applications/list.html`

Admin review list:
- Tab bar: Pending / Approved / Rejected with counts
- Table: Name, Email, Submitted At, Status badge, Actions (View, Quick Approve, Quick Reject)
- Quick reject shows a small modal or inline form for a note

### `templates/admin/applications/detail.html`

Detail view:
- All applicant fields displayed
- Approve button
- Reject form with note textarea

---

## Step 9 — Security config

Add to `SecurityConfig` — permit the apply routes:

```java
.requestMatchers("/apply", "/apply/**").permitAll()
```

The admin applications routes (`/admin/applications/**`) are already covered by
the existing `/admin/**` rule.

---

## Step 10 — Unit tests

If `test_mode` is NOT `"token-save"`:

```java
package {{base_package}}.service;

// ApplicationServiceTest.java
// Test: submit with duplicate pending email → throws IllegalStateException
// Test: submit with existing active member email → throws IllegalStateException
// Test: approve → creates ACTIVE member record
// Test: reject → sets status to REJECTED with note
```

Write these tests following the same pattern as other service tests:
- Use `@ExtendWith(MockitoExtension.class)`
- Mock `applicationRepository` and `memberRepository`
- Assert state changes on returned objects

---

## Step 11 — Verify

Apply `test_mode`:
- `"token-save"` → Skip.
- `"build-only"` → `./gradlew build -x test`.
- `"build-and-test"` → `./gradlew build`.

---

## Step 12 — Git commits

```bash
git add -A
git commit -m "feat: add public membership application form with admin review workflow"
```

---

## Step 13 — Update `installed_modules`

Add `"membership-apply"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
Membership application form is ready. Test it:
  http://localhost:8080/apply            ← public form (no login needed)
  http://localhost:8080/admin/applications  ← admin review queue

Next steps:
- springboot-mail → send notification emails on application events
- springboot-membership → full member management (if not already installed)
```
