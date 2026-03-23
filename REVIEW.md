# Skill Review Report
*Generated: 2026-03-23 — Full review of all 21 springboot-* SKILL.md files*

---

## Summary

| Priority | Count |
|----------|-------|
| 🔴 Critical | 10 |
| 🟡 Important | 12 |
| 🟢 Nice-to-have | 8 |

---

## 🔴 Critical Issues

### C1 — Invalid Flyway migration version format
**Affects:** `springboot-mail`, `springboot-file-upload`, `springboot-item-submit`, `springboot-announcements`, `springboot-app-config`, `springboot-audit-log`, `springboot-stats-summary`, `springboot-soft-delete`

Skills from `mail` onward use filenames like `V_mail__create_email_system.sql`, `V_upload__create_uploaded_files.sql`, `V_submissions__create.sql`, etc. These are **not valid Flyway versioned migration filenames**. Flyway requires `V{version}__{description}.sql` where `{version}` is numeric (`1`, `2`, `1.1`, etc.). `V_mail__` would be interpreted as version `.mail` (invalid) and **Flyway would refuse to run it**, crashing the app on startup.

**Fix:** Change all `V_<text>__` templates to instruct Claude to find the next available migration number and use `V{N}__create_email_system.sql`, etc. The parenthetical note "(next available version)" is there but the filename template overrides it.

---

### C2 — `MemberService` is missing `countByStatus()` method
**Affects:** `springboot-membership`

`AdminMemberController` (Step 6) calls:
```java
model.addAttribute("activeCount",  memberService.countByStatus(Member.Status.ACTIVE));
model.addAttribute("pendingCount", memberService.countByStatus(Member.Status.PENDING));
```
But `MemberService` (Step 4) has no `countByStatus()` method — only `MemberRepository` has it. This is a **compile error** that would surface immediately.

**Fix:** Add `public long countByStatus(Member.Status status) { return memberRepository.countByStatus(status); }` to `MemberService`.

---

### C3 — `ExportConfigLoader` is undefined
**Affects:** `springboot-data-export`

`DataExportService` (Step 6) declares:
```java
private final ExportConfigLoader configLoader;  // loads from export-config.yml
```
and calls `configLoader.getConfig(tableName)`. But `ExportConfigLoader` is **never created** anywhere in the skill. Only `ExportFieldConfig` and `ExportTableConfig` are defined. This is a compile error.

**Fix:** Add `ExportConfigLoader` bean that reads and parses `export-config.yml` using `@ConfigurationProperties` or Spring's `YamlPropertiesFactoryBean`.

---

### C4 — `DataExportService.getRowsForSheets()` referenced but not defined
**Affects:** `springboot-data-export`

`GoogleSheetsExportService` (Step 8) calls:
```java
List<List<Object>> values = dataExportService.getRowsForSheets(tableName);
```
But `DataExportService` (Step 6) has no such method. Compile error.

**Fix:** Add `getRowsForSheets(String tableName)` to `DataExportService` that returns data formatted as `List<List<Object>>` for the Sheets API.

---

### C5 — Compile error: `AnnouncementService` imports `EmailService` without hard dependency
**Affects:** `springboot-announcements`

`AnnouncementService` declares:
```java
@Autowired(required = false)
private EmailService emailService;
```
`@Autowired(required = false)` prevents Spring from failing, but the **Java import** of `{{base_package}}.service.EmailService` will cause a **compile error** if `springboot-mail` is not installed (the class literally does not exist). The skill installs without requiring `mail` in `installed_modules`.

**Fix:** Either (a) require `mail` as a hard prerequisite, or (b) instruct Claude to conditionally add the import and field only when `"mail"` is in `installed_modules`.

---

### C6 — Compile error: `StatsSummaryService` imports `SubmissionRepository` without hard dependency
**Affects:** `springboot-stats-summary`

Same pattern: `StatsSummaryService` uses `@Autowired(required = false)` for `SubmissionRepository`, but the import will fail at compile time if `springboot-item-submit` is not installed.

**Fix:** Conditionally include the `SubmissionRepository` field and import only when `"item-submit"` is in `installed_modules`. Use runtime reflection or an interface if needed, or require `item-submit` explicitly.

---

### C7 — `DashboardController` breaks when magic-link auth is used
**Affects:** `springboot-auth-google`, `springboot-auth-magic-link`

`DashboardController.dashboard()` is created by `springboot-auth-google`:
```java
public String dashboard(@AuthenticationPrincipal OAuth2User user, ...) {
    if (user == null) { return "redirect:/"; }
    ...
```
When a magic-link user (authenticated via `UsernamePasswordAuthenticationToken`) hits `/dashboard`, `user` is **null** (they are not an `OAuth2User`), so they are redirected to home and **cannot access their dashboard at all**.

**Fix:** `springboot-auth-magic-link` should update `DashboardController` to handle both auth types, similar to the pattern already used in `MemberProfileController.resolveEmail()`.

---

### C8 — `/suspended` route missing
**Affects:** `springboot-auth-google`

`DashboardController` redirects suspended users to `/suspended`:
```java
if ("SUSPENDED".equals(status)) {
    return "redirect:/suspended";
}
```
But no controller endpoint for `/suspended` is ever created, and no `templates/auth/suspended.html` template is created. Suspended users get a **Whitelabel Error Page** (404/500).

**Fix:** Add a `@GetMapping("/suspended")` endpoint to `DashboardController` and a corresponding Thymeleaf template.

---

### C9 — `springboot-soft-delete` SKILL.md still exists as a standalone skill
**Affects:** `springboot-soft-delete`

The commit `"refactor: fold soft-delete into db-setup convention; add beginner_friendly + soft_delete config_fields"` indicates that soft-delete was folded into `db-setup`. The `soft_delete` config field now controls behavior. But `springboot-soft-delete/SKILL.md` still exists as a runnable skill. Users can still trigger it, creating confusion and potential double implementation.

**Fix:** Either (a) delete `springboot-soft-delete/SKILL.md` and ensure entity skills handle `soft_delete=true` natively, or (b) update it to say it is now a configuration option in `springboot-setup` and is no longer a separate installable module.

---

### C10 — `springboot-gcp-setup` has no git commit step
**Affects:** `springboot-gcp-setup`

`gcp-setup` creates `gcp-credentials-local.json`, modifies `.gitignore`, updates `.spring-config.json` (adding `gcp_credentials_path`, `artifact_registry_url`), yet has **no git commit step**. Changes to `.gitignore` and `.spring-config.json` are left uncommitted, which can confuse subsequent skills that check git status.

**Fix:** Add a Step 11 (before `installed_modules` update) with:
```bash
git add .gitignore .spring-config.json
git commit -m "feat: add GCP setup config and credentials gitignore"
```

---

## 🟡 Important Issues

### I1 — `springboot-auth-magic-link` missing `mail` as prerequisite
**Affects:** `springboot-auth-magic-link`

The skill self-contains mail config (adds `spring-boot-starter-mail`, sets up MailHog, etc.) rather than requiring `springboot-mail`. This creates two problems:
1. If `springboot-mail` is later installed, **duplicate/conflicting** `spring.mail.*` config blocks appear in `application-local.yml`.
2. The `springboot-mail` description says "install it before magic-link auth" but `auth-magic-link` doesn't require it.

**Fix:** Add a check: if `"mail"` is NOT in `installed_modules`, emit a note: "Adding standalone mail config. For a full email system, install `springboot-mail` first." If `"mail"` IS installed, skip Step 10 (mail config) and wire `MagicLinkService` to use the existing `EmailService` instead of `SimpleMailMessage`.

---

### I2 — `springboot-membership-apply` doesn't send welcome email on approval
**Affects:** `springboot-membership-apply`

`ApplicationService.approve()` creates the `Member` account but **never calls any mail service**. The `springboot-mail` description advertises "springboot-membership-apply will use it for welcome emails" and the review criteria lists this as a known dependency. But the current code simply creates the member silently.

**Fix:** In `ApplicationService.approve()`, after creating the member, conditionally call `emailService.fireAutoRules("APPLICATION_APPROVED", email, vars)` if `emailService` is available. Also update the prerequisites check to note that `mail` enables welcome emails.

---

### I3 — `beginner_friendly` is not used by most skills
**Affects:** All skills except `springboot-db-setup`

`springboot-db-setup` is the only skill with explicit beginner_friendly behavior (Step 0 shows the Flyway explanation block). All other skills extract `beginner_friendly` from config in Step 0 but **never use it** — there are no instructions telling Claude how to behave differently.

**Fix:** Each skill should include a note like:
> If `beginner_friendly` is `true`, explain each new concept (e.g., "A `@Service` is a Spring-managed component that holds business logic...") as it is introduced. Follow the db-setup pattern.

---

### I4 — Session-based auth used in controllers created before auth is known
**Affects:** `springboot-file-upload`, `springboot-item-submit`

`FileUploadController` and `MemberSubmissionController` resolve the current member via:
```java
Long memberId = (Long) session.getAttribute("memberId");
```
This only works for **magic-link auth** (where memberId is stored in session). For **Google OAuth2** users, `memberId` is never stored in session — it's in the `OAuth2User` attributes. These controllers silently redirect Google OAuth2 users to home.

**Fix:** Both controllers should use a resolver pattern like `MemberProfileController.resolveEmail()` and then look up the memberId from the member email.

---

### I5 — Entity-creating skills don't implement `soft_delete` when `soft_delete=true`
**Affects:** `springboot-auth-google`, `springboot-membership`, `springboot-membership-apply`, `springboot-item-submit`, `springboot-announcements`

The `soft_delete` config field exists and `springboot-db-setup` documents the pattern, but **no entity-creating skill actually checks `soft_delete` and implements the pattern**. When `soft_delete=true`, the Flyway migrations for `members`, `applications`, `submissions`, `announcements` should add `deleted_at TIMESTAMPTZ` and entities should get `@SQLRestriction("deleted_at IS NULL")`. This never happens automatically.

**Fix:** Each entity-creating skill's Flyway migration step should conditionally include `deleted_at TIMESTAMPTZ` when `soft_delete=true`. Entity class should get `@SQLRestriction` and the field. Admin controllers should get soft-delete/restore endpoints. Add a note: "Uses soft delete by default (`deleted_at` timestamp). Tell Claude explicitly if you want permanent delete."

---

### I6 — `AdminMemberController` and `AdminAnnouncementController` assume OAuth2User principal
**Affects:** `springboot-membership`, `springboot-announcements`

Both controllers have:
```java
@AuthenticationPrincipal OAuth2User currentUser
```
For magic-link users, `currentUser` is null and they fall back to `SecurityContextHolder.getContext().getAuthentication().getPrincipal()`, which returns an email string. While this is handled with a ternary, it's fragile and inconsistent.

**Fix:** Introduce a shared `AuthHelper` utility that resolves the current user's email regardless of auth method, and use it in all admin controllers.

---

### I7 — `springboot-gcp-setup` missing `test_mode` and `beginner_friendly`
**Affects:** `springboot-gcp-setup`

This is the only non-trivial skill that doesn't extract or reference `test_mode` or `beginner_friendly` from config. While it runs CLI commands rather than generating code, the skill should still acknowledge these fields (e.g., beginner_friendly → explain what each API does, what a service account is).

**Fix:** Add Step 0 extract for `beginner_friendly`, add explanations for GCP concepts when it's true.

---

### I8 — `springboot-scaffold` doesn't reference `beginner_friendly`
**Affects:** `springboot-scaffold`

Step 0 extracts `test_mode`, `language`, `translate_terms` but NOT `beginner_friendly`. It is the second skill a user runs after setup, so first-time users hit it early.

**Fix:** Add `beginner_friendly` to Step 0 extraction and add a note about explaining the scaffold structure when it's true (e.g., "A Spring Boot project follows this structure: `controller/` handles HTTP requests, `service/` holds business logic...").

---

### I9 — `springboot-mail` and `springboot-auth-magic-link` are architecturally disconnected
**Affects:** `springboot-mail`, `springboot-auth-magic-link`

`springboot-mail` creates a sophisticated `EmailService` with DB-backed templates. `springboot-auth-magic-link` creates its own `MagicLinkService` that uses raw `JavaMailSender.send()`. The `springboot-mail` description says "magic-link auth will use it to send login links" but there is **no integration path** — `MagicLinkService` never calls `EmailService`. If both are installed, magic link emails bypass the template/logging system entirely.

**Fix:** When `"mail"` is in `installed_modules`, `MagicLinkService` should use `emailService.fireAutoRules("MAGIC_LINK_SENT", email, vars)` instead of its own `sendEmail()` method. The mail skill already seeds a `MAGIC_LINK_SENT` template for this purpose.

---

### I10 — `springboot-integration-tests` references removed module in its exclusion list
**Affects:** `springboot-integration-tests`

Step 1 counts modules beyond the foundation:
> `(setup, scaffold, db, gcp, ide-setup, app-config, **soft-delete**, audit-log)`

If `soft-delete` is a removed module, it should not be listed here. Stale reference.

**Fix:** Remove `soft-delete` from the exclusion list in Step 1.

---

### I11 — `springboot-auth-google` Step 0 doesn't extract `"gcp"` from installed_modules
**Affects:** `springboot-auth-google`

Step 0 lists what to extract but omits `installed_modules → must contain "setup", "scaffold", "db", "gcp"`. It only says `installed_modules → must contain "setup", "scaffold", "db"` in the extract block. The full check for `"gcp"` is in Step 1 but not visible in the Step 0 extract list, which is the canonical reference for what a skill reads.

**Fix:** Add `gcp` requirement to the Step 0 extract block.

---

### I12 — `springboot-membership-apply` has no mail prerequisite despite needing it
**Affects:** `springboot-membership-apply`

Per the review criteria: `springboot-membership-apply` must have `mail` as a HARD prerequisite because it sends welcome email on approval. Currently `installed_modules` only requires `"setup"`, `"scaffold"`, `"db"`. Combined with issue I2 (no actual email call), this creates a silent failure mode.

**Fix:** Either require `"mail"` as a prerequisite (hard dep), or (preferred) make mail a soft dep with a clear note: "Install `springboot-mail` before this skill to enable welcome emails on approval."

---

## 🟢 Nice-to-Have Improvements

### N1 — Add "uses soft delete by default" notes on delete operations
**Affects:** `springboot-announcements`, `springboot-membership`, `springboot-item-submit`, `springboot-membership-apply`

Skills that implement delete operations (`announcementService.delete()`, etc.) should include a note per the established convention:
> "This module uses soft delete by default (`deleted_at` timestamp). Tell Claude explicitly if you want permanent delete (i.e., `soft_delete: false` in config)."

---

### N2 — `springboot-gcp-setup` Step 8 should be more actionable
**Affects:** `springboot-gcp-setup`

The manual steps in Step 8 (Cloud SQL, Secret Manager, OAuth2) are listed as a prose block but don't pause for user confirmation before proceeding. The skill should ask the user to confirm each sub-task before marking the checklist item as done.

---

### N3 — `springboot-deploy` `application-prod.yml` includes oauth2 and mail config unconditionally
**Affects:** `springboot-deploy`

Step 5 always includes `spring.security.oauth2` and `spring.mail` sections in `application-prod.yml`, even if those modules are not installed. This causes startup errors if Google auth or mail secrets are not set.

**Fix:** Wrap each section with a note: "Include only if `auth-google` / `mail` is in `installed_modules`."

---

### N4 — Flyway migration numbers for foundation skills may collide with flexible-number skills
**Affects:** `springboot-auth-magic-link` (Step 3/4 uses V3 but it may be V4 if auth-google is installed)

The instruction "Use the next available migration version number" is correct, but the skills show specific numbers as examples (V2, V3, V4, V5) that may be wrong depending on installation order. This could confuse Claude into using the wrong number.

**Fix:** Make the instruction more explicit: "Check existing migration files in `src/main/resources/db/migration/` to find the highest version number, then use N+1."

---

### N5 — `springboot-admin-portal` only allows SUPER_ADMIN to preview
**Affects:** `springboot-admin-portal`

The end-of-skill message says "ADMIN accounts see preview without member-specific data (future enhancement)" but this is never addressed. A regular ADMIN wanting to debug a member's view has no path.

---

### N6 — `springboot-stats-summary` seeds academic year dates that may be stale
**Affects:** `springboot-stats-summary`

The seed SQL uses `EXTRACT(YEAR FROM CURRENT_DATE)` which runs at migration time. If the migration was run in June 2024, the period is June 2024. If the app is deployed in September 2024, it's already in a new academic year but the period is wrong.

**Fix:** Add a note to prompt the admin to add new periods via the UI or a follow-up migration each year.

---

### N7 — Missing `@EnableScheduling` instruction in `springboot-auth-magic-link`
**Affects:** `springboot-auth-magic-link`

Step 12 says "Add `@EnableScheduling` to the main application class or a config class for the cleanup cron" but doesn't show where exactly to add it. The `@Scheduled(cron = "0 0 3 * * *")` in `MagicLinkService` will silently not run without it.

**Fix:** Explicitly show the annotation addition to the main `@SpringBootApplication` class.

---

### N8 — Description for `springboot-stats-summary` is generic
**Affects:** `springboot-stats-summary`

The description includes "add credit tracking", "add annual summary" etc., which are reasonable. But it could more clearly say it's for **member activity aggregation** (not general statistics) to avoid triggering on unrelated stats requests.

---

## Config Completeness Check

The `springboot-setup` wizard collects all fields needed by downstream skills. No fields are missing. Fields set by later skills (`artifact_registry_url`, `gcp_credentials_path`) are correctly populated by `springboot-gcp-setup`.

---

## Brand Name Leakage Check

✅ No occurrences of `babysleepzzz`, `好眠師`, `好眠寶寶`, `sleep-beauty`, `mybabyzzz`, or other domain-specific hardcoded values found in any SKILL.md. All project-specific values use `{{app_name}}`, `{{brand_name}}`, `{{base_package}}` placeholders.

---

## Soft-Delete Reference Check

- ✅ `springboot-db-setup/references/soft-delete-pattern.md` exists and is referenced by `db-setup`
- ❌ `springboot-soft-delete/SKILL.md` still exists (see C9)
- ❌ No entity-creating skill actually implements soft delete when `soft_delete=true` (see I5)
- ❌ No skill says "uses soft delete by default" on delete operations (see N1)

---

## Dependency Graph Accuracy

| Dependency | Expected | Actual | Status |
|-----------|----------|--------|--------|
| `auth-google` requires `gcp` | HARD | ✅ Enforced in Step 1 | ✅ |
| `auth-magic-link` requires `mail` | HARD (per criteria) | ❌ Not required | 🔴 |
| `membership-apply` requires `mail` | HARD (per criteria) | ❌ Not required | 🔴 |
| `membership` requires auth module | HARD | ✅ Requires one of auth-google / auth-magic-link | ✅ |
| `auth-google` referenced in `membership` | auth-google OR magic-link | ✅ Either accepted | ✅ |

---

## Cross-Skill Consistency

| Check | Status |
|-------|--------|
| Flyway naming: V1/V2/V3 for foundation, V_text__ for feature skills | ❌ V_text__ invalid |
| Admin URL pattern: `/admin/...` | ✅ Consistent |
| Member-facing URL pattern: `/dashboard`, `/profile`, `/files`, `/submissions` | ✅ Consistent |
| Gradle dependency format: `implementation("group:artifact:version")` | ✅ Consistent |
| Entity lifecycle: `@PrePersist` + `@PreUpdate` | ✅ Consistent |
| Error flash attributes: `"success"` and `"error"` keys | ✅ Consistent |

---

## Health Scores

| Skill | Score | Key Issues |
|-------|-------|-----------|
| springboot-setup | ★★★★★ | Excellent — all fields, clear instructions |
| springboot-db-setup | ★★★★★ | Best skill — beginner_friendly, soft_delete, complete |
| springboot-scaffold | ★★★★☆ | Missing beginner_friendly reference |
| springboot-auth-google | ★★★★☆ | DashboardController magic-link gap (C7), /suspended missing (C8) |
| springboot-auth-magic-link | ★★★★☆ | Missing mail prerequisite (I1), arch disconnect (I9) |
| springboot-admin-portal | ★★★★☆ | Solid, minor enhancement gap |
| springboot-deploy | ★★★★☆ | Conditional prod config needed (N3) |
| springboot-integration-tests | ★★★★☆ | Stale soft-delete reference (I10) |
| springboot-app-config | ★★★★☆ | Well structured |
| springboot-audit-log | ★★★★☆ | Solid |
| springboot-ide-setup | ★★★★☆ | Instructional, intentionally no code |
| springboot-mail | ★★★☆☆ | Arch disconnect with magic-link (I9), compile issue if EmailService absent in announcements |
| springboot-gcp-setup | ★★★☆☆ | No git commit (C10), no test_mode, no beginner_friendly |
| springboot-membership | ★★★☆☆ | Missing countByStatus in service (C2), OAuth2-assumption (I6) |
| springboot-membership-apply | ★★★☆☆ | No welcome email (I2), no mail prerequisite (I12) |
| springboot-file-upload | ★★★☆☆ | Session-only auth (I4) |
| springboot-item-submit | ★★★☆☆ | Session-only auth (I4) |
| springboot-announcements | ★★☆☆☆ | Compile error (C5), hard delete instead of soft, OAuth2 assumption (I6) |
| springboot-data-export | ★★☆☆☆ | ExportConfigLoader undefined (C3), getRowsForSheets undefined (C4) |
| springboot-stats-summary | ★★☆☆☆ | Compile error (C6), stale period seeds (N6) |
| springboot-soft-delete | ★★☆☆☆ | Should not exist as standalone (C9) |

---

## Recommended Fix Priority

**Sprint 1 (blockers — fix before any user runs these skills):**
1. C1 — Fix Flyway naming format in 8 skills
2. C2 — Add `countByStatus` to `MemberService`
3. C3 + C4 — Add `ExportConfigLoader` and `getRowsForSheets` to data-export
4. C5 + C6 — Fix conditional imports in announcements and stats-summary
5. C7 — Update DashboardController to handle magic-link users
6. C8 — Add `/suspended` route and template

**Sprint 2 (important quality):**
7. C9 — Decide fate of `springboot-soft-delete` skill
8. C10 — Add git commit step to gcp-setup
9. I2 — Add welcome email to membership-apply approve()
10. I4 — Fix session-only auth in file-upload and item-submit
11. I5 — Implement soft_delete pattern in entity-creating skills

**Sprint 3 (polish):**
12. I3 — Add beginner_friendly instructions to all skills
13. I9 — Integrate magic-link email with mail module's EmailService
14. N3 — Conditional prod config in deploy skill
15. N4 — Improve migration number guidance
