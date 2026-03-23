---
name: springboot-item-submit
description: >
  TRIGGER this skill when the user says "add submission system", "add member submissions",
  "add credit submissions", "add work log", "add activity tracking", "run springboot-item-submit",
  "let members submit records for review", "add a submit and review workflow", or anything about
  members creating records that admins then approve or reject. The submission type is configurable —
  it can represent credits, hours, activities, or any trackable record.
  Requires setup, scaffold, db, and membership.
---

# springboot-item-submit

Adds a generic member submission system. Members create submissions, admins review (approve/reject).
Submission types are configurable via the `app_config` table.

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
base_package      → Java package root
app_name          → used in page titles
test_mode         → controls build verification
language          → respond in this language
translate_terms   → whether to translate technical terms
beginner_friendly → if true, explain technical terms and decisions as you work
installed_modules → must contain "setup", "scaffold", "db", "membership"
```

---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`, `"scaffold"`, `"db"`, `"membership"`.

---

## Beginner-Friendly Mode

If `beginner_friendly` is `true` in `.spring-config.json`, explain key concepts as you work. Examples:
- When creating submission status: "The status field (`PENDING`, `APPROVED`, `REJECTED`) creates a simple state machine. A submission moves through states as admins review it — we never delete records, just update the status."
- When building the admin review queue: "The review queue is just a filtered query — `WHERE status = 'PENDING'` — displayed as a list. Approving or rejecting just updates the status and adds a reviewer note."
- When adding duplicate prevention: "We check for existing submissions with the same member + type + period before inserting, so members can't accidentally submit the same thing twice."

## Step 2 — Flyway migration

Create `V_submissions__create.sql` (next available version number):

```sql
CREATE TABLE submission_types (
    id          BIGSERIAL PRIMARY KEY,
    code        VARCHAR(50)  NOT NULL UNIQUE,
    label       VARCHAR(100) NOT NULL,
    description TEXT,
    is_active   BOOLEAN      NOT NULL DEFAULT TRUE,
    sort_order  INT          NOT NULL DEFAULT 0
);

CREATE TABLE submissions (
    id            BIGSERIAL PRIMARY KEY,
    member_id     BIGINT       NOT NULL REFERENCES members(id) ON DELETE CASCADE,
    type_code     VARCHAR(50)  NOT NULL,
    title         VARCHAR(255) NOT NULL,
    description   TEXT,
    status        VARCHAR(20)  NOT NULL DEFAULT 'PENDING'
                  CHECK (status IN ('PENDING', 'APPROVED', 'REJECTED', 'DRAFT')),
    reviewer_note TEXT,
    reviewed_by   VARCHAR(255),
    reviewed_at   TIMESTAMPTZ,
    submitted_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    updated_at    TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    extra_data    JSONB
);

CREATE INDEX idx_submissions_member_id ON submissions (member_id);
CREATE INDEX idx_submissions_status    ON submissions (status);
CREATE INDEX idx_submissions_type_code ON submissions (type_code);

-- Seed default submission types (customize these per project)
INSERT INTO submission_types (code, label, description, sort_order) VALUES
    ('general',  'General Submission', 'A general record submission', 1),
    ('activity', 'Activity Report',    'Report a completed activity', 2),
    ('document', 'Document Upload',    'Submit a document for review', 3)
ON CONFLICT (code) DO NOTHING;
```

---

## Step 3 — Entities

### `SubmissionType` entity

```java
package {{base_package}}.entity;

import jakarta.persistence.*;
import lombok.*;

@Entity
@Table(name = "submission_types")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class SubmissionType {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true)
    private String code;

    @Column(nullable = false)
    private String label;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(name = "is_active", nullable = false)
    private boolean active = true;

    @Column(name = "sort_order", nullable = false)
    private int sortOrder = 0;
}
```

### `Submission` entity

```java
package {{base_package}}.entity;

import jakarta.persistence.*;
import lombok.*;
import org.hibernate.annotations.JdbcTypeCode;
import org.hibernate.type.SqlTypes;
import java.time.OffsetDateTime;
import java.util.Map;

@Entity
@Table(name = "submissions")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class Submission {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id", nullable = false)
    private Member member;

    @Column(name = "type_code", nullable = false)
    private String typeCode;

    @Column(nullable = false)
    private String title;

    @Column(columnDefinition = "TEXT")
    private String description;

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

    @Column(name = "updated_at", nullable = false)
    private OffsetDateTime updatedAt;

    @Column(name = "extra_data", columnDefinition = "jsonb")
    @JdbcTypeCode(SqlTypes.JSON)
    private Map<String, Object> extraData;

    @PrePersist
    void onCreate() { submittedAt = updatedAt = OffsetDateTime.now(); }

    @PreUpdate
    void onUpdate() { updatedAt = OffsetDateTime.now(); }

    public enum Status { DRAFT, PENDING, APPROVED, REJECTED }
}
```

---

## Step 4 — Repositories

```java
// SubmissionTypeRepository.java
package {{base_package}}.repository;

import {{base_package}}.entity.SubmissionType;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface SubmissionTypeRepository extends JpaRepository<SubmissionType, Long> {
    List<SubmissionType> findByActiveTrueOrderBySortOrderAsc();
}
```

```java
// SubmissionRepository.java
package {{base_package}}.repository;

import {{base_package}}.entity.Submission;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface SubmissionRepository extends JpaRepository<Submission, Long> {
    List<Submission>  findByMemberIdOrderBySubmittedAtDesc(Long memberId);
    Page<Submission>  findByStatus(Submission.Status status, Pageable pageable);
    Page<Submission>  findByMemberIdAndStatus(Long memberId, Submission.Status status, Pageable pageable);
    long              countByStatus(Submission.Status status);
    long              countByMemberId(Long memberId);
}
```

---

## Step 5 — `SubmissionService`

```java
package {{base_package}}.service;

import {{base_package}}.entity.*;
import {{base_package}}.repository.*;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.OffsetDateTime;
import java.util.List;

@Service
@RequiredArgsConstructor
public class SubmissionService {

    private final SubmissionRepository submissionRepository;
    private final SubmissionTypeRepository submissionTypeRepository;
    private final MemberRepository memberRepository;

    public List<SubmissionType> getActiveTypes() {
        return submissionTypeRepository.findByActiveTrueOrderBySortOrderAsc();
    }

    @Transactional
    public Submission create(Long memberId, String typeCode, String title, String description) {
        Member member = memberRepository.findById(memberId)
            .orElseThrow(() -> new IllegalArgumentException("Member not found: " + memberId));

        Submission submission = Submission.builder()
            .member(member)
            .typeCode(typeCode)
            .title(title)
            .description(description)
            .status(Submission.Status.PENDING)
            .build();

        return submissionRepository.save(submission);
    }

    @Transactional
    public Submission approve(Long submissionId, String reviewerEmail, String note) {
        Submission submission = findById(submissionId);
        submission.setStatus(Submission.Status.APPROVED);
        submission.setReviewedBy(reviewerEmail);
        submission.setReviewedAt(OffsetDateTime.now());
        submission.setReviewerNote(note);
        return submissionRepository.save(submission);
    }

    @Transactional
    public Submission reject(Long submissionId, String reviewerEmail, String note) {
        Submission submission = findById(submissionId);
        submission.setStatus(Submission.Status.REJECTED);
        submission.setReviewedBy(reviewerEmail);
        submission.setReviewedAt(OffsetDateTime.now());
        submission.setReviewerNote(note);
        return submissionRepository.save(submission);
    }

    public Submission findById(Long id) {
        return submissionRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Submission not found: " + id));
    }

    public List<Submission> findByMember(Long memberId) {
        return submissionRepository.findByMemberIdOrderBySubmittedAtDesc(memberId);
    }

    /** Ensure a member can only view/edit their own submissions */
    public void assertOwnership(Long submissionId, Long memberId) {
        Submission submission = findById(submissionId);
        if (!submission.getMember().getId().equals(memberId)) {
            throw new SecurityException("Access denied: this submission belongs to another member");
        }
    }
}
```

---

## Step 6 — `MemberSubmissionController`

```java
package {{base_package}}.controller.member;

import {{base_package}}.service.SubmissionService;
import jakarta.servlet.http.HttpSession;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Controller
@RequestMapping("/submissions")
@RequiredArgsConstructor
public class MemberSubmissionController {

    private final SubmissionService submissionService;

    @GetMapping
    public String list(HttpSession session, Model model) {
        Long memberId = (Long) session.getAttribute("memberId");
        if (memberId == null) return "redirect:/";
        model.addAttribute("submissions", submissionService.findByMember(memberId));
        return "member/submissions/list";
    }

    @GetMapping("/new")
    public String newForm(HttpSession session, Model model) {
        Long memberId = (Long) session.getAttribute("memberId");
        if (memberId == null) return "redirect:/";
        model.addAttribute("types", submissionService.getActiveTypes());
        return "member/submissions/new";
    }

    @PostMapping
    public String create(@RequestParam String typeCode,
                         @RequestParam String title,
                         @RequestParam(required = false) String description,
                         HttpSession session,
                         RedirectAttributes redirectAttributes) {
        Long memberId = (Long) session.getAttribute("memberId");
        if (memberId == null) return "redirect:/";
        try {
            submissionService.create(memberId, typeCode, title, description);
            redirectAttributes.addFlashAttribute("success", "Submission created successfully.");
        } catch (Exception e) {
            redirectAttributes.addFlashAttribute("error", e.getMessage());
        }
        return "redirect:/submissions";
    }
}
```

---

## Step 7 — `AdminSubmissionController`

```java
package {{base_package}}.controller.admin;

import {{base_package}}.entity.Submission;
import {{base_package}}.repository.SubmissionRepository;
import {{base_package}}.service.SubmissionService;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Controller
@RequestMapping("/admin/submissions")
@RequiredArgsConstructor
public class AdminSubmissionController {

    private final SubmissionRepository submissionRepository;
    private final SubmissionService submissionService;

    @GetMapping
    public String list(@RequestParam(defaultValue = "PENDING") String status,
                       @RequestParam(defaultValue = "0") int page,
                       Model model) {
        Submission.Status statusFilter = Submission.Status.PENDING;
        try { statusFilter = Submission.Status.valueOf(status); } catch (Exception ignored) {}

        model.addAttribute("submissions",
            submissionRepository.findByStatus(statusFilter,
                PageRequest.of(page, 20, Sort.by("submittedAt").descending())));
        model.addAttribute("status", status);
        model.addAttribute("pendingCount",  submissionRepository.countByStatus(Submission.Status.PENDING));
        model.addAttribute("approvedCount", submissionRepository.countByStatus(Submission.Status.APPROVED));
        model.addAttribute("rejectedCount", submissionRepository.countByStatus(Submission.Status.REJECTED));
        return "admin/submissions/list";
    }

    @PostMapping("/{id}/approve")
    public String approve(@PathVariable Long id,
                          @RequestParam(required = false) String note,
                          Authentication auth,
                          RedirectAttributes redirectAttributes) {
        try {
            submissionService.approve(id, auth.getName(), note);
            redirectAttributes.addFlashAttribute("success", "Submission approved.");
        } catch (Exception e) {
            redirectAttributes.addFlashAttribute("error", e.getMessage());
        }
        return "redirect:/admin/submissions";
    }

    @PostMapping("/{id}/reject")
    public String reject(@PathVariable Long id,
                         @RequestParam(required = false) String note,
                         Authentication auth,
                         RedirectAttributes redirectAttributes) {
        try {
            submissionService.reject(id, auth.getName(), note);
            redirectAttributes.addFlashAttribute("success", "Submission rejected.");
        } catch (Exception e) {
            redirectAttributes.addFlashAttribute("error", e.getMessage());
        }
        return "redirect:/admin/submissions";
    }
}
```

---

## Step 8 — Thymeleaf templates

### `templates/member/submissions/list.html`
- Member's own submission history table: Title, Type, Status badge, Submitted At
- "New Submission" button
- Filter by status tabs: All / Pending / Approved / Rejected

### `templates/member/submissions/new.html`
- Form: Type select (from active `submission_types`), Title input, Description textarea
- Submit button

### `templates/admin/submissions/list.html`
- Admin review queue: Submission Title, Member Name/Email, Type, Submitted At, Actions
- Tabs: Pending / Approved / Rejected with counts
- Quick approve/reject buttons with optional note

---

## Step 9 — Unit tests

If `test_mode` is NOT `"token-save"`, write tests for `SubmissionService`:
- `create()` with valid member → returns PENDING submission
- `approve()` → sets status to APPROVED, sets reviewed_by and reviewed_at
- `reject()` → sets status to REJECTED with note
- `assertOwnership()` with wrong member → throws SecurityException

---

## Step 10 — Verify

Apply `test_mode`.

---

## Step 11 — Git commits

```bash
git add -A
git commit -m "feat: add member submission system with admin review queue"
```

---

## Step 12 — Update `installed_modules`

Add `"item-submit"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
Submission system is ready.

Endpoints:
  /submissions      → member view (create + history)
  /admin/submissions → admin review queue

Submission types are configurable in the submission_types table.
Add/edit rows there to change what types members can submit.
```
