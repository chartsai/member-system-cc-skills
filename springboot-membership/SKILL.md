---
name: springboot-membership
description: >
  TRIGGER this skill when the user says "add member management", "add membership system",
  "run springboot-membership", "add member profiles", "add admin member list", "add role management",
  "add member status management", "build the member portal", or anything about managing member accounts,
  roles (MEMBER/ADMIN/SUPER_ADMIN), or statuses (PENDING/ACTIVE/SUSPENDED).
  Requires setup, scaffold, db, and at least one auth module (auth-google or auth-magic-link).
---

# springboot-membership

Extends the Member entity with profile pages, role-based access, admin member list,
admin role/status controls, and member status management.

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
base_package      → Java package root
super_admin_email → for dev seed
test_mode         → controls build verification
language          → respond in this language for ALL output (questions, explanations,
                    status messages, code comments, and user-facing copy in generated files)
                    IMPORTANT: if "zh-TW", "traditional-chinese", or "繁體中文":
                      use 繁體中文 (Traditional Chinese) throughout
                      NEVER use Simplified Chinese (简体中文) — they are different writing systems
                      Key differences: 體/体, 語/语, 資料/数据, 設定/设定, 確認/确认, 請/请
translate_terms   → whether to translate technical terms
beginner_friendly → if true, explain technical terms and decisions as you work
installed_modules → must contain "setup", "scaffold", "db", and one of "auth-google" or "auth-magic-link"
```

> **Language activation**: After reading `language` above, switch ALL your responses to that
> language immediately — including every question, status message, explanation, and all
> human-readable text in any generated files (HTML, Thymeleaf templates, SQL seed labels).
> If `language` is not set, ask: "What language should I use? (English / 繁體中文 / other)"



---

## Step 1 — Prerequisites check

Verify `installed_modules` contains:
- `"setup"`, `"scaffold"`, `"db"`
- At least one of: `"auth-google"`, `"auth-magic-link"`

If missing, tell the user which skills to run first.

---

## Beginner-Friendly Mode

If `beginner_friendly` is `true` in `.spring-config.json`, explain key concepts as you work. Examples:
- When creating the Member entity: "A JPA entity is a Java class that maps to a database table. Each field becomes a column, and Spring Data JPA handles the SQL queries for you."
- When adding roles: "Roles (MEMBER, ADMIN, SUPER_ADMIN) control what each user can access. Spring Security checks roles before allowing access to protected pages."
- When adding member status: "Status (PENDING, ACTIVE, SUSPENDED) tracks a member's lifecycle. We use it to block suspended members from logging in without deleting their data."

## Step 2 — Flyway migration: `V4__extend_members.sql`

Use the next available migration version number. Check existing migrations to find the right number.

```sql
-- V4: Extend members table with additional profile fields
-- (Only adds columns if they don't already exist — safe to run idempotently)

-- Add profile fields if not present
ALTER TABLE members
    ADD COLUMN IF NOT EXISTS display_name VARCHAR(255),
    ADD COLUMN IF NOT EXISTS bio TEXT,
    ADD COLUMN IF NOT EXISTS phone VARCHAR(50),
    ADD COLUMN IF NOT EXISTS timezone VARCHAR(50) DEFAULT 'UTC';

-- Ensure role and status columns have correct constraints
-- (They should already be there from auth module, this is a safety check)
DO $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM information_schema.columns
        WHERE table_name = 'members' AND column_name = 'role'
    ) THEN
        ALTER TABLE members ADD COLUMN role VARCHAR(20) NOT NULL DEFAULT 'MEMBER'
            CHECK (role IN ('MEMBER', 'ADMIN', 'SUPER_ADMIN'));
    END IF;

    IF NOT EXISTS (
        SELECT 1 FROM information_schema.columns
        WHERE table_name = 'members' AND column_name = 'status'
    ) THEN
        ALTER TABLE members ADD COLUMN status VARCHAR(20) NOT NULL DEFAULT 'PENDING'
            CHECK (status IN ('PENDING', 'ACTIVE', 'SUSPENDED'));
    END IF;
END $$;

-- Note for SUPER_ADMIN seed: already done in V2 (auth module)
```

---

## Step 3 — Update `Member` entity

Add the new profile fields to the `Member` entity (if they are not already there):

```java
@Column(name = "display_name")
private String displayName;

@Column(columnDefinition = "TEXT")
private String bio;

private String phone;

private String timezone;
```

---

## Step 4 — `MemberService`

```java
package {{base_package}}.service;

import {{base_package}}.entity.Member;
import {{base_package}}.repository.MemberRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Service
@RequiredArgsConstructor
public class MemberService {

    private final MemberRepository memberRepository;

    public List<Member> findAll() {
        return memberRepository.findAll();
    }

    public Page<Member> findAll(Pageable pageable) {
        return memberRepository.findAll(pageable);
    }

    public Member findById(Long id) {
        return memberRepository.findById(id)
            .orElseThrow(() -> new IllegalArgumentException("Member not found: " + id));
    }

    public Member findByEmail(String email) {
        return memberRepository.findByEmail(email)
            .orElseThrow(() -> new IllegalArgumentException("Member not found: " + email));
    }

    @Transactional
    public Member updateProfile(Long id, String displayName, String bio, String phone) {
        Member member = findById(id);
        member.setDisplayName(displayName);
        member.setBio(bio);
        member.setPhone(phone);
        return memberRepository.save(member);
    }

    @Transactional
    public Member updateRole(Long id, Member.Role newRole, String currentUserEmail) {
        Member member = findById(id);
        Member currentUser = findByEmail(currentUserEmail);

        // Only SUPER_ADMIN can change roles; cannot change own role
        if (currentUser.getRole() != Member.Role.SUPER_ADMIN) {
            throw new SecurityException("Only SUPER_ADMIN can change member roles");
        }
        if (member.getEmail().equalsIgnoreCase(currentUserEmail)) {
            throw new IllegalArgumentException("You cannot change your own role");
        }

        member.setRole(newRole);
        return memberRepository.save(member);
    }

    @Transactional
    public Member updateStatus(Long id, Member.Status newStatus, String currentUserEmail) {
        Member member = findById(id);

        if (member.getEmail().equalsIgnoreCase(currentUserEmail)) {
            throw new IllegalArgumentException("You cannot change your own status");
        }

        member.setStatus(newStatus);
        return memberRepository.save(member);
    }
}
```

---

## Step 5 — Update `MemberRepository`

Add search and filter methods:

```java
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import java.util.List;

// Add to existing MemberRepository:

Page<Member> findByStatus(Member.Status status, Pageable pageable);

@Query("SELECT m FROM Member m WHERE " +
       "(:search IS NULL OR LOWER(m.email) LIKE LOWER(CONCAT('%', :search, '%')) OR " +
       "LOWER(m.name) LIKE LOWER(CONCAT('%', :search, '%'))) AND " +
       "(:status IS NULL OR m.status = :status)")
Page<Member> search(@Param("search") String search,
                    @Param("status") Member.Status status,
                    Pageable pageable);

long countByStatus(Member.Status status);
```

---

## Step 6 — Admin Member List: `AdminMemberController`

```java
package {{base_package}}.controller.admin;

import {{base_package}}.entity.Member;
import {{base_package}}.service.MemberService;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.PageRequest;
import org.springframework.data.domain.Sort;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.core.user.OAuth2User;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Controller
@RequestMapping("/admin/members")
@RequiredArgsConstructor
public class AdminMemberController {

    private final MemberService memberService;

    @GetMapping
    public String list(@RequestParam(required = false) String search,
                       @RequestParam(required = false) String status,
                       @RequestParam(defaultValue = "0") int page,
                       Model model) {
        Member.Status statusFilter = null;
        if (status != null && !status.isBlank()) {
            try { statusFilter = Member.Status.valueOf(status); } catch (IllegalArgumentException ignored) {}
        }

        Page<Member> members = memberService.search(
            search, statusFilter,
            PageRequest.of(page, 20, Sort.by("createdAt").descending())
        );

        model.addAttribute("members", members);
        model.addAttribute("search", search);
        model.addAttribute("status", status);
        model.addAttribute("statuses", Member.Status.values());
        model.addAttribute("roles", Member.Role.values());
        model.addAttribute("totalCount", memberService.findAll().size());
        model.addAttribute("activeCount", memberService.countByStatus(Member.Status.ACTIVE));
        model.addAttribute("pendingCount", memberService.countByStatus(Member.Status.PENDING));
        return "admin/members/list";
    }

    @GetMapping("/{id}")
    public String detail(@PathVariable Long id, Model model) {
        model.addAttribute("member", memberService.findById(id));
        model.addAttribute("roles", Member.Role.values());
        model.addAttribute("statuses", Member.Status.values());
        return "admin/members/detail";
    }

    @PostMapping("/{id}/role")
    public String updateRole(@PathVariable Long id,
                             @RequestParam String role,
                             @AuthenticationPrincipal OAuth2User currentUser,
                             RedirectAttributes redirectAttributes) {
        try {
            String email = currentUser != null
                ? (String) currentUser.getAttribute("email")
                : (String) org.springframework.security.core.context.SecurityContextHolder
                    .getContext().getAuthentication().getPrincipal();
            memberService.updateRole(id, Member.Role.valueOf(role), email);
            redirectAttributes.addFlashAttribute("success", "Role updated successfully.");
        } catch (Exception e) {
            redirectAttributes.addFlashAttribute("error", e.getMessage());
        }
        return "redirect:/admin/members/" + id;
    }

    @PostMapping("/{id}/status")
    public String updateStatus(@PathVariable Long id,
                               @RequestParam String status,
                               @AuthenticationPrincipal OAuth2User currentUser,
                               RedirectAttributes redirectAttributes) {
        try {
            String email = currentUser != null
                ? (String) currentUser.getAttribute("email")
                : (String) org.springframework.security.core.context.SecurityContextHolder
                    .getContext().getAuthentication().getPrincipal();
            memberService.updateStatus(id, Member.Status.valueOf(status), email);
            redirectAttributes.addFlashAttribute("success", "Status updated successfully.");
        } catch (Exception e) {
            redirectAttributes.addFlashAttribute("error", e.getMessage());
        }
        return "redirect:/admin/members/" + id;
    }
}
```

---

## Step 7 — Member Profile: `MemberProfileController`

```java
package {{base_package}}.controller.member;

import {{base_package}}.entity.Member;
import {{base_package}}.service.MemberService;
import lombok.RequiredArgsConstructor;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.core.user.OAuth2User;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Controller
@RequestMapping("/profile")
@RequiredArgsConstructor
public class MemberProfileController {

    private final MemberService memberService;

    @GetMapping
    public String profile(@AuthenticationPrincipal OAuth2User oauthUser,
                          jakarta.servlet.http.HttpSession session,
                          Model model) {
        String email = resolveEmail(oauthUser, session);
        model.addAttribute("member", memberService.findByEmail(email));
        return "member/profile";
    }

    @PostMapping
    public String updateProfile(@RequestParam String displayName,
                                @RequestParam String bio,
                                @RequestParam String phone,
                                @AuthenticationPrincipal OAuth2User oauthUser,
                                jakarta.servlet.http.HttpSession session,
                                RedirectAttributes redirectAttributes) {
        try {
            String email = resolveEmail(oauthUser, session);
            Member member = memberService.findByEmail(email);
            memberService.updateProfile(member.getId(), displayName, bio, phone);
            redirectAttributes.addFlashAttribute("success", "Profile updated.");
        } catch (Exception e) {
            redirectAttributes.addFlashAttribute("error", e.getMessage());
        }
        return "redirect:/profile";
    }

    /** Works with both OAuth2 (Google) and session-based (magic link) auth */
    private String resolveEmail(OAuth2User oauthUser, jakarta.servlet.http.HttpSession session) {
        if (oauthUser != null) return (String) oauthUser.getAttribute("email");
        Object sessionEmail = session.getAttribute("memberEmail");
        if (sessionEmail != null) return (String) sessionEmail;
        throw new SecurityException("Not authenticated");
    }
}
```

---

## Step 8 — Thymeleaf templates

### `templates/admin/members/list.html`

Create a clean admin table listing members with:
- Stats row at top: Total / Active / Pending counts
- Search input + status filter dropdown
- Table columns: Name, Email, Role (badge), Status (badge), Joined, Actions
- Action buttons: View/Edit per row
- Pagination controls at bottom
- Color-coded status badges: ACTIVE=green, PENDING=yellow, SUSPENDED=red
- Color-coded role badges: SUPER_ADMIN=purple, ADMIN=blue, MEMBER=gray

### `templates/admin/members/detail.html`

Member detail/edit page:
- Member info display (name, email, picture, joined date)
- Role change form (select + submit) — only shown if current user is SUPER_ADMIN
- Status change form (select + submit)
- Back to list button

### `templates/member/profile.html`

Member self-edit profile page:
- Display current info
- Editable fields: display name, bio, phone
- Non-editable: email, role, status, joined date
- Save button

---

## Step 9 — Update `SecurityConfig`

Ensure these routes are secured:

```java
.requestMatchers("/admin/**").hasAnyRole("ADMIN", "SUPER_ADMIN")
.requestMatchers("/profile/**").authenticated()
```

---

## Step 10 — Unit tests for role/status logic

If `test_mode` is NOT `"token-save"`:

```java
package {{base_package}}.service;

import {{base_package}}.entity.Member;
import {{base_package}}.repository.MemberRepository;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import java.util.Optional;
import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class MemberServiceTest {

    @Mock MemberRepository memberRepository;
    @InjectMocks MemberService memberService;

    @Test
    void updateRole_bySuperAdmin_succeeds() {
        Member target = Member.builder().id(1L).email("user@example.com").role(Member.Role.MEMBER).build();
        Member admin  = Member.builder().id(2L).email("sa@example.com").role(Member.Role.SUPER_ADMIN).build();
        when(memberRepository.findById(1L)).thenReturn(Optional.of(target));
        when(memberRepository.findByEmail("sa@example.com")).thenReturn(Optional.of(admin));
        when(memberRepository.save(any())).thenReturn(target);

        memberService.updateRole(1L, Member.Role.ADMIN, "sa@example.com");
        assertThat(target.getRole()).isEqualTo(Member.Role.ADMIN);
    }

    @Test
    void updateRole_byNonSuperAdmin_throws() {
        Member target = Member.builder().id(1L).email("user@example.com").role(Member.Role.MEMBER).build();
        Member admin  = Member.builder().id(2L).email("a@example.com").role(Member.Role.ADMIN).build();
        when(memberRepository.findById(1L)).thenReturn(Optional.of(target));
        when(memberRepository.findByEmail("a@example.com")).thenReturn(Optional.of(admin));

        assertThatThrownBy(() -> memberService.updateRole(1L, Member.Role.ADMIN, "a@example.com"))
            .isInstanceOf(SecurityException.class);
    }

    @Test
    void updateRole_ownRole_throws() {
        Member admin = Member.builder().id(1L).email("sa@example.com").role(Member.Role.SUPER_ADMIN).build();
        when(memberRepository.findById(1L)).thenReturn(Optional.of(admin));
        when(memberRepository.findByEmail("sa@example.com")).thenReturn(Optional.of(admin));

        assertThatThrownBy(() -> memberService.updateRole(1L, Member.Role.MEMBER, "sa@example.com"))
            .isInstanceOf(IllegalArgumentException.class);
    }
}
```

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
git commit -m "feat: add member management with admin list, role/status controls, and profile pages"
```

---

## Step 13 — Update `installed_modules`

Add `"membership"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
Member management is ready. Visit:
  http://localhost:8080/admin/members   ← Admin member list
  http://localhost:8080/profile         ← Member profile

Next steps:
- springboot-membership-apply → public application form
- springboot-admin-portal → admin preview mode
- springboot-announcements → broadcast announcements to members
```
