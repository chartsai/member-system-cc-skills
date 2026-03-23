---
name: springboot-announcements
description: >
  TRIGGER this skill when the user says "add announcements", "add broadcast messages",
  "add admin announcements", "run springboot-announcements", "let admins post announcements to members",
  "add a news feed", "add a notice board", "add pinned announcements", or anything about
  admins being able to send broadcast messages to members. Requires setup, scaffold, db, membership.
  Works without mail (members see announcements on dashboard), enhanced with mail for email notifications.
---

# springboot-announcements

Adds a broadcast announcement system. Admins create announcements, members see active ones
on their dashboard. Optionally sends email notifications if the `mail` module is installed.

---

## Dependencies

### Hard prerequisites
- `setup`, `scaffold`, `db`, `membership` must be installed

### Soft prerequisites
- `mail` → enables sending announcement notifications via email (gracefully skips if absent)

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
base_package      → Java package root
test_mode         → controls build verification
language          → respond in this language
translate_terms   → whether to translate technical terms
installed_modules → must include "setup", "scaffold", "db", "membership"
                    "mail" → enables email notifications (check before using EmailService)
```

---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`, `"scaffold"`, `"db"`, `"membership"`.

Check if `"mail"` is present. If not:
```
Note: mail module is not installed. Announcements will appear on the member dashboard only.
Install springboot-mail first to also send announcement notifications via email.
```

---

## Step 2 — Flyway migration

Create `V_announcements__create.sql` (next available version):

```sql
CREATE TABLE announcements (
    id           BIGSERIAL PRIMARY KEY,
    title        VARCHAR(255) NOT NULL,
    body         TEXT         NOT NULL,
    author_id    BIGINT       REFERENCES members(id) ON DELETE SET NULL,
    target_role  VARCHAR(20)  NOT NULL DEFAULT 'ALL'
                 CHECK (target_role IN ('ALL', 'MEMBER', 'ADMIN')),
    is_pinned    BOOLEAN      NOT NULL DEFAULT FALSE,
    is_published BOOLEAN      NOT NULL DEFAULT FALSE,
    published_at TIMESTAMPTZ,
    expires_at   TIMESTAMPTZ,
    created_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at   TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_announcements_published ON announcements (is_published, published_at DESC);
CREATE INDEX idx_announcements_expires   ON announcements (expires_at);
```

---

## Step 3 — `Announcement` entity

```java
package {{base_package}}.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.OffsetDateTime;

@Entity
@Table(name = "announcements")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Announcement {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String title;

    @Column(nullable = false, columnDefinition = "TEXT")
    private String body;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    private Member author;

    @Column(name = "target_role", nullable = false)
    @Enumerated(EnumType.STRING)
    private TargetRole targetRole = TargetRole.ALL;

    @Column(name = "is_pinned",    nullable = false) private boolean pinned    = false;
    @Column(name = "is_published", nullable = false) private boolean published = false;

    @Column(name = "published_at") private OffsetDateTime publishedAt;
    @Column(name = "expires_at")   private OffsetDateTime expiresAt;

    @Column(name = "created_at", nullable = false, updatable = false) private OffsetDateTime createdAt;
    @Column(name = "updated_at", nullable = false)                    private OffsetDateTime updatedAt;

    @PrePersist  void onCreate() { createdAt = updatedAt = OffsetDateTime.now(); }
    @PreUpdate   void onUpdate() { updatedAt = OffsetDateTime.now(); }

    public boolean isExpired() {
        return expiresAt != null && OffsetDateTime.now().isAfter(expiresAt);
    }

    public boolean isVisible() {
        return published && !isExpired();
    }

    public enum TargetRole { ALL, MEMBER, ADMIN }
}
```

---

## Step 4 — `AnnouncementRepository`

```java
package {{base_package}}.repository;

import {{base_package}}.entity.Announcement;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import java.time.OffsetDateTime;
import java.util.List;

public interface AnnouncementRepository extends JpaRepository<Announcement, Long> {

    /** Active announcements visible to members: published + not expired + targeting ALL or MEMBER */
    @Query("""
        SELECT a FROM Announcement a
        WHERE a.published = true
          AND (a.expiresAt IS NULL OR a.expiresAt > :now)
          AND (a.targetRole = 'ALL' OR a.targetRole = 'MEMBER')
        ORDER BY a.pinned DESC, a.publishedAt DESC
        """)
    List<Announcement> findVisibleForMembers(@Param("now") OffsetDateTime now);

    /** Active announcements visible to admins: includes ADMIN-targeted ones */
    @Query("""
        SELECT a FROM Announcement a
        WHERE a.published = true
          AND (a.expiresAt IS NULL OR a.expiresAt > :now)
        ORDER BY a.pinned DESC, a.publishedAt DESC
        """)
    List<Announcement> findVisibleForAdmins(@Param("now") OffsetDateTime now);

    Page<Announcement> findAllByOrderByCreatedAtDesc(Pageable pageable);
}
```

---

## Step 5 — `AnnouncementService`

```java
package {{base_package}}.service;

import {{base_package}}.entity.*;
import {{base_package}}.repository.*;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.OffsetDateTime;
import java.util.List;
import java.util.Map;

@Service
@Slf4j
@RequiredArgsConstructor
public class AnnouncementService {

    private final AnnouncementRepository announcementRepository;
    private final MemberRepository memberRepository;

    // EmailService is optional — only injected if mail module is present
    // Use @Autowired(required = false) or check installed_modules and conditionally wire
    @org.springframework.beans.factory.annotation.Autowired(required = false)
    private EmailService emailService;

    public List<Announcement> getVisibleForMembers() {
        return announcementRepository.findVisibleForMembers(OffsetDateTime.now());
    }

    public List<Announcement> getVisibleForAdmins() {
        return announcementRepository.findVisibleForAdmins(OffsetDateTime.now());
    }

    @Transactional
    public Announcement create(String title, String body, Long authorId,
                               Announcement.TargetRole targetRole,
                               boolean pinned, OffsetDateTime expiresAt) {
        Member author = memberRepository.findById(authorId)
            .orElseThrow(() -> new IllegalArgumentException("Author not found"));

        Announcement announcement = Announcement.builder()
            .title(title)
            .body(body)
            .author(author)
            .targetRole(targetRole)
            .pinned(pinned)
            .expiresAt(expiresAt)
            .build();

        return announcementRepository.save(announcement);
    }

    @Transactional
    public Announcement publish(Long id, boolean sendEmail) {
        Announcement announcement = findById(id);
        announcement.setPublished(true);
        announcement.setPublishedAt(OffsetDateTime.now());
        Announcement saved = announcementRepository.save(announcement);

        // Send email notification if mail module is present and requested
        if (sendEmail && emailService != null) {
            sendAnnouncementEmail(saved);
        } else if (sendEmail) {
            log.info("Mail module not installed — skipping email notification for announcement: {}", id);
        }

        return saved;
    }

    @Transactional
    public void togglePin(Long id) {
        Announcement announcement = findById(id);
        announcement.setPinned(!announcement.isPinned());
        announcementRepository.save(announcement);
    }

    @Transactional
    public void delete(Long id) {
        announcementRepository.deleteById(id);
    }

    public Announcement findById(Long id) {
        return announcementRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Announcement not found: " + id));
    }

    private void sendAnnouncementEmail(Announcement announcement) {
        // Fire an AUTO email rule for the announcement, or send directly
        // This is a simplified version — adapt based on your recipient targeting needs
        List<Member> recipients = switch (announcement.getTargetRole()) {
            case ALL    -> memberRepository.findAll().stream()
                              .filter(m -> m.getStatus() == Member.Status.ACTIVE).toList();
            case MEMBER -> memberRepository.findAll().stream()
                              .filter(m -> m.getStatus() == Member.Status.ACTIVE
                                        && m.getRole() == Member.Role.MEMBER).toList();
            case ADMIN  -> memberRepository.findAll().stream()
                              .filter(m -> m.getStatus() == Member.Status.ACTIVE
                                        && (m.getRole() == Member.Role.ADMIN
                                            || m.getRole() == Member.Role.SUPER_ADMIN)).toList();
        };

        for (Member recipient : recipients) {
            Map<String, String> vars = Map.of(
                "memberName",           recipient.getName() != null ? recipient.getName() : recipient.getEmail(),
                "announcementTitle",    announcement.getTitle(),
                "announcementBody",     announcement.getBody()
            );
            emailService.fireAutoRules("ANNOUNCEMENT_PUBLISHED", recipient.getEmail(), vars);
        }
        log.info("Sent announcement email notification to {} recipients", recipients.size());
    }
}
```

---

## Step 6 — `AdminAnnouncementController`

```java
package {{base_package}}.controller.admin;

import {{base_package}}.entity.Announcement;
import {{base_package}}.service.AnnouncementService;
import {{base_package}}.repository.AnnouncementRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.core.user.OAuth2User;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;
import java.time.OffsetDateTime;
import jakarta.servlet.http.HttpSession;

@Controller
@RequestMapping("/admin/announcements")
@RequiredArgsConstructor
public class AdminAnnouncementController {

    private final AnnouncementService announcementService;
    private final AnnouncementRepository announcementRepository;

    @GetMapping
    public String list(@RequestParam(defaultValue = "0") int page, Model model) {
        model.addAttribute("announcements",
            announcementRepository.findAllByOrderByCreatedAtDesc(
                PageRequest.of(page, 20)));
        return "admin/announcements/list";
    }

    @GetMapping("/new")
    public String newForm(Model model) {
        model.addAttribute("roles", Announcement.TargetRole.values());
        return "admin/announcements/form";
    }

    @PostMapping
    public String create(@RequestParam String title,
                         @RequestParam String body,
                         @RequestParam String targetRole,
                         @RequestParam(required = false) boolean pinned,
                         @RequestParam(required = false) String expiresAt,
                         @AuthenticationPrincipal OAuth2User oauth,
                         HttpSession session,
                         RedirectAttributes redirectAttributes) {
        try {
            Long authorId = oauth != null
                ? (Long) oauth.getAttribute("id")
                : (Long) session.getAttribute("memberId");

            OffsetDateTime expires = (expiresAt != null && !expiresAt.isBlank())
                ? OffsetDateTime.parse(expiresAt + "T00:00:00Z")
                : null;

            announcementService.create(title, body, authorId,
                Announcement.TargetRole.valueOf(targetRole), pinned, expires);
            redirectAttributes.addFlashAttribute("success", "Announcement created.");
        } catch (Exception e) {
            redirectAttributes.addFlashAttribute("error", e.getMessage());
        }
        return "redirect:/admin/announcements";
    }

    @PostMapping("/{id}/publish")
    public String publish(@PathVariable Long id,
                          @RequestParam(required = false, defaultValue = "false") boolean sendEmail,
                          RedirectAttributes redirectAttributes) {
        try {
            announcementService.publish(id, sendEmail);
            redirectAttributes.addFlashAttribute("success",
                "Announcement published." + (sendEmail ? " Email notifications sent." : ""));
        } catch (Exception e) {
            redirectAttributes.addFlashAttribute("error", e.getMessage());
        }
        return "redirect:/admin/announcements";
    }

    @PostMapping("/{id}/pin")
    public String togglePin(@PathVariable Long id, RedirectAttributes redirectAttributes) {
        announcementService.togglePin(id);
        redirectAttributes.addFlashAttribute("success", "Pin status updated.");
        return "redirect:/admin/announcements";
    }

    @PostMapping("/{id}/delete")
    public String delete(@PathVariable Long id, RedirectAttributes redirectAttributes) {
        announcementService.delete(id);
        redirectAttributes.addFlashAttribute("success", "Announcement deleted.");
        return "redirect:/admin/announcements";
    }
}
```

---

## Step 7 — Member dashboard integration

In the member dashboard controller, inject announcements:

```java
// In DashboardController or MemberDashboardController:
model.addAttribute("announcements", announcementService.getVisibleForMembers());
```

In the dashboard template, show announcements at the top:

```html
<!-- Announcements section in templates/member/dashboard.html -->
<div th:if="${!announcements.isEmpty()}" class="mb-8">
    <h2 class="text-lg font-semibold text-gray-700 mb-4">Announcements</h2>
    <div th:each="ann : ${announcements}"
         th:class="${ann.pinned} ? 'bg-amber-50 border border-amber-200 rounded-xl p-5 mb-4'
                                 : 'bg-white border border-gray-200 rounded-xl p-5 mb-4'">
        <div class="flex items-center gap-2 mb-2">
            <span th:if="${ann.pinned}" class="text-amber-600 text-xs font-bold uppercase">📌 Pinned</span>
            <h3 class="font-semibold text-gray-800" th:text="${ann.title}">Title</h3>
        </div>
        <div class="text-gray-600 text-sm" th:utext="${ann.body}">Body</div>
        <p class="text-xs text-gray-400 mt-3"
           th:text="${#temporals.format(ann.publishedAt, 'MMM d, yyyy')}">Date</p>
    </div>
</div>
```

---

## Step 8 — Templates

### `templates/admin/announcements/list.html`
- Table: Title, Target, Pinned, Published, Expires, Created At, Actions
- Actions: Publish (if not published), Pin/Unpin, Delete
- "Create New" button

### `templates/admin/announcements/form.html`
- Fields: Title, Body (textarea), Target Role (select), Pinned (checkbox), Expires At (date)
- Optional: "Send email notification on publish?" checkbox (shown only if mail module is installed)

---

## Step 9 — Unit tests

If `test_mode` is NOT `"token-save"`:
- `create()` with valid data → returns unpublished announcement
- `publish()` without mail → sets published=true, does not throw
- `isExpired()` with past expires_at → returns true
- `isVisible()` with unpublished → returns false

---

## Step 10 — Verify

Apply `test_mode`.

---

## Step 11 — Git commits

```bash
git add -A
git commit -m "feat: add announcement system with admin creation and member dashboard display"
```

---

## Step 12 — Update `installed_modules`

Add `"announcements"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
Announcement system is ready.

Admin: http://localhost:8080/admin/announcements → create and publish announcements
Members: announcements appear at the top of the dashboard when published

Email notifications: only available if springboot-mail is installed
```
