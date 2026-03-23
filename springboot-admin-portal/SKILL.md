---
name: springboot-admin-portal
description: >
  TRIGGER this skill when the user says "add admin preview mode", "let admins view as members",
  "add preview as member", "run springboot-admin-portal", "add impersonation", "add member impersonation",
  "admin should be able to see what members see", or anything about giving admins the ability to browse
  the member portal as if they were a specific member. Requires setup, scaffold, db, and membership.
---

# springboot-admin-portal

Adds "preview as member" functionality. Admins can select any member and view the portal exactly
as that member would see it. A sticky banner shows when preview mode is active.

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
installed_modules → must contain "setup", "scaffold", "db", "membership"
```

> **Language activation**: After reading `language` above, switch ALL your responses to that
> language immediately — including every question, status message, explanation, and all
> human-readable text in any generated files (HTML, Thymeleaf templates, SQL seed labels).
> If `language` is not set, ask: "What language should I use? (English / 繁體中文 / other)"



---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`, `"scaffold"`, `"db"`, `"membership"`.

---

## Beginner-Friendly Mode

If `beginner_friendly` is `true` in `.spring-config.json`, explain key concepts as you work. Examples:
- When implementing preview mode: "Preview mode stores the target member's ID in the HTTP session — a server-side temporary storage tied to the current browser session. When the admin navigates, we check the session to serve data as that member."
- When adding the preview banner: "Thymeleaf's `th:if` lets us conditionally show HTML based on server-side data — here, we show the preview banner only when a `previewMemberId` exists in the session."
- When implementing exit preview: "Removing the `previewMemberId` from the session instantly restores the admin's normal view — no database changes needed."

## Step 2 — `PreviewSessionHelper` — session-based preview state

```java
package {{base_package}}.admin;

import {{base_package}}.entity.Member;
import {{base_package}}.repository.MemberRepository;
import jakarta.servlet.http.HttpSession;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import java.util.Optional;

/**
 * Manages the admin "preview as member" session state.
 *
 * When an admin enters preview mode, the previewMemberId is stored in the HTTP session.
 * All subsequent requests check this value to determine whether to render the portal
 * with the preview member's data instead of the admin's own data.
 */
@Component
@RequiredArgsConstructor
public class PreviewSessionHelper {

    private static final String SESSION_KEY = "previewMemberId";

    private final MemberRepository memberRepository;

    /** Enter preview mode for a specific member. */
    public void enterPreview(HttpSession session, Long memberId) {
        session.setAttribute(SESSION_KEY, memberId);
    }

    /** Exit preview mode. */
    public void exitPreview(HttpSession session) {
        session.removeAttribute(SESSION_KEY);
    }

    /** Is preview mode active? */
    public boolean isInPreview(HttpSession session) {
        return session.getAttribute(SESSION_KEY) != null;
    }

    /** Get the member being previewed, or empty if not in preview mode. */
    public Optional<Member> getPreviewMember(HttpSession session) {
        Object id = session.getAttribute(SESSION_KEY);
        if (id == null) return Optional.empty();
        return memberRepository.findById((Long) id);
    }

    /** Get the preview member ID from the session. */
    public Optional<Long> getPreviewMemberId(HttpSession session) {
        Object id = session.getAttribute(SESSION_KEY);
        return id == null ? Optional.empty() : Optional.of((Long) id);
    }
}
```

---

## Step 3 — `AdminPreviewController`

```java
package {{base_package}}.controller.admin;

import {{base_package}}.admin.PreviewSessionHelper;
import {{base_package}}.entity.Member;
import {{base_package}}.service.MemberService;
import jakarta.servlet.http.HttpSession;
import lombok.RequiredArgsConstructor;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;

@Controller
@RequestMapping("/admin/preview")
@RequiredArgsConstructor
public class AdminPreviewController {

    private final PreviewSessionHelper previewSessionHelper;
    private final MemberService memberService;

    /**
     * Enter preview mode as a specific member.
     * Only SUPER_ADMIN can preview as a real member.
     */
    @PostMapping("/{memberId}")
    @PreAuthorize("hasRole('SUPER_ADMIN')")
    public String enterPreview(@PathVariable Long memberId,
                               HttpSession session,
                               RedirectAttributes redirectAttributes) {
        try {
            Member target = memberService.findById(memberId);
            previewSessionHelper.enterPreview(session, memberId);
            redirectAttributes.addFlashAttribute("previewName", target.getName());
            return "redirect:/dashboard";
        } catch (Exception e) {
            redirectAttributes.addFlashAttribute("error", "Could not enter preview mode: " + e.getMessage());
            return "redirect:/admin/members";
        }
    }

    /**
     * Exit preview mode and return to the admin panel.
     */
    @PostMapping("/exit")
    public String exitPreview(HttpSession session) {
        previewSessionHelper.exitPreview(session);
        return "redirect:/admin/members";
    }

    /**
     * GET version of exit (for banner links).
     */
    @GetMapping("/exit")
    public String exitPreviewGet(HttpSession session) {
        previewSessionHelper.exitPreview(session);
        return "redirect:/admin/members";
    }
}
```

---

## Step 4 — Preview banner Thymeleaf fragment

Create `templates/fragments/preview-banner.html`:

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<body>
<!--
  Preview mode banner fragment.
  Include in portal layout template with:
    <div th:replace="~{fragments/preview-banner :: previewBanner}"></div>

  The banner is automatically hidden when not in preview mode.
-->
<div th:fragment="previewBanner"
     th:if="${previewMember != null}"
     class="w-full bg-amber-400 text-amber-900 px-6 py-3 flex items-center justify-between
            sticky top-0 z-50 shadow-md">
    <div class="flex items-center gap-3">
        <span class="font-bold text-sm">🔍 ADMIN PREVIEW MODE</span>
        <span class="text-sm" th:if="${previewMember != null}">
            Viewing as:
            <strong th:text="${previewMember.name}">Member Name</strong>
            (<span th:text="${previewMember.email}">email</span>)
        </span>
    </div>
    <div class="flex items-center gap-4">
        <span class="text-xs opacity-75">Changes made here affect the real account</span>
        <a th:href="@{/admin/preview/exit}"
           class="bg-amber-900 text-amber-100 px-4 py-1.5 rounded-lg text-sm font-medium
                  hover:bg-amber-800 transition">
            Exit Preview
        </a>
    </div>
</div>
</body>
</html>
```

---

## Step 5 — `PreviewContextInterceptor` — inject preview state into every request

```java
package {{base_package}}.admin;

import {{base_package}}.entity.Member;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.springframework.lang.NonNull;
import org.springframework.stereotype.Component;
import org.springframework.web.servlet.HandlerInterceptor;
import org.springframework.web.servlet.ModelAndView;
import java.util.Optional;

/**
 * Injects preview context into the model for every request.
 * This makes preview state available in all Thymeleaf templates via ${previewMember}.
 */
@Component
@RequiredArgsConstructor
public class PreviewContextInterceptor implements HandlerInterceptor {

    private final PreviewSessionHelper previewSessionHelper;
    private final {{base_package}}.repository.MemberRepository memberRepository;

    @Override
    public void postHandle(@NonNull HttpServletRequest request,
                           @NonNull HttpServletResponse response,
                           @NonNull Object handler,
                           ModelAndView modelAndView) {
        if (modelAndView == null) return;
        if (request.getSession(false) == null) return;

        Optional<Member> previewMember = previewSessionHelper.getPreviewMember(request.getSession());
        modelAndView.addObject("previewMember", previewMember.orElse(null));
        modelAndView.addObject("isInPreview", previewMember.isPresent());
    }
}
```

Register the interceptor in a `WebMvcConfig`:

```java
package {{base_package}}.config;

import {{base_package}}.admin.PreviewContextInterceptor;
import lombok.RequiredArgsConstructor;
import org.springframework.context.annotation.Configuration;
import org.springframework.web.servlet.config.annotation.InterceptorRegistry;
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer;

@Configuration
@RequiredArgsConstructor
public class WebMvcConfig implements WebMvcConfigurer {

    private final PreviewContextInterceptor previewContextInterceptor;

    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(previewContextInterceptor)
            .addPathPatterns("/**")
            .excludePathPatterns("/css/**", "/js/**", "/images/**", "/favicon.ico");
    }
}
```

---

## Step 6 — Update portal templates to respect preview mode

In any portal controller that loads "the current member's data", add preview support:

```java
// In member portal controllers, replace:
//   String email = getCurrentUserEmail();
//   Member member = memberService.findByEmail(email);
// with:

private Member resolveCurrentMember(HttpSession session,
                                    org.springframework.security.core.Authentication auth) {
    // If in preview mode, load the preview member instead
    Optional<Member> previewMember = previewSessionHelper.getPreviewMember(session);
    if (previewMember.isPresent()) return previewMember.get();

    // Otherwise load the authenticated user's own record
    String email = auth.getName();
    return memberService.findByEmail(email);
}
```

Apply this pattern to all member-facing controllers (dashboard, profile, etc.).

---

## Step 7 — Add "Preview as Member" button to admin member list

In `templates/admin/members/list.html` and `templates/admin/members/detail.html`,
add a "Preview as this member" button (SUPER_ADMIN only):

```html
<!-- Add next to other action buttons, only for SUPER_ADMIN -->
<form th:action="@{/admin/preview/{id}(id=${member.id})}" method="post"
      sec:authorize="hasRole('SUPER_ADMIN')">
    <input type="hidden" th:name="${_csrf.parameterName}" th:value="${_csrf.token}" />
    <button type="submit"
            class="text-xs bg-amber-100 text-amber-700 border border-amber-300
                   px-3 py-1.5 rounded-lg hover:bg-amber-200 transition">
        Preview as Member
    </button>
</form>
```

---

## Step 8 — Add preview banner to portal layout

In `templates/layout/base.html` (or the member portal layout), add:

```html
<!-- At the very top of <body>, before the nav -->
<div th:replace="~{fragments/preview-banner :: previewBanner}"></div>
```

---

## Step 9 — Unit tests for preview session logic

If `test_mode` is NOT `"token-save"`:

```java
package {{base_package}}.admin;

import {{base_package}}.entity.Member;
import {{base_package}}.repository.MemberRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.mock.web.MockHttpSession;
import java.util.Optional;

import static org.assertj.core.api.Assertions.*;
import static org.mockito.Mockito.*;

@ExtendWith(MockitoExtension.class)
class PreviewSessionHelperTest {

    @Mock MemberRepository memberRepository;
    PreviewSessionHelper helper;

    @BeforeEach
    void setup() {
        helper = new PreviewSessionHelper(memberRepository);
    }

    @Test
    void notInPreview_byDefault() {
        MockHttpSession session = new MockHttpSession();
        assertThat(helper.isInPreview(session)).isFalse();
        assertThat(helper.getPreviewMember(session)).isEmpty();
    }

    @Test
    void enterPreview_setsSession() {
        MockHttpSession session = new MockHttpSession();
        Member member = Member.builder().id(42L).email("m@test.com").build();
        when(memberRepository.findById(42L)).thenReturn(Optional.of(member));

        helper.enterPreview(session, 42L);

        assertThat(helper.isInPreview(session)).isTrue();
        assertThat(helper.getPreviewMember(session)).isPresent();
        assertThat(helper.getPreviewMember(session).get().getId()).isEqualTo(42L);
    }

    @Test
    void exitPreview_clearsSession() {
        MockHttpSession session = new MockHttpSession();
        helper.enterPreview(session, 42L);
        helper.exitPreview(session);

        assertThat(helper.isInPreview(session)).isFalse();
    }
}
```

---

## Step 10 — Verify

Apply `test_mode`:
- `"token-save"` → Skip.
- `"build-only"` → `./gradlew build -x test`.
- `"build-and-test"` → `./gradlew build`.

---

## Step 11 — Git commits

```bash
git add -A
git commit -m "feat: add admin preview mode — view portal as any member with session-based state"
```

---

## Step 12 — Update `installed_modules`

Add `"admin-portal"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
Admin preview mode is ready.

To test it:
1. Log in as a SUPER_ADMIN
2. Go to /admin/members
3. Click "Preview as Member" on any active member
4. The portal will load with that member's data + a yellow banner at top
5. Click "Exit Preview" in the banner to return to admin

Note: Only SUPER_ADMIN can preview as a real member.
      ADMIN accounts see preview without member-specific data (future enhancement).
```
