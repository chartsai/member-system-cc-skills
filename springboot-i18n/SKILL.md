---
name: springboot-i18n
description: >
  TRIGGER this skill when the user wants to add multi-language support, internationalization,
  i18n, locale switching, or translate their Spring Boot app UI into multiple languages.
  Adds Spring MessageSource, message property files, a locale switcher component,
  and Thymeleaf #{key} integration.
  Prerequisites: setup, scaffold must be installed.
---

# springboot-i18n

Adds full internationalization (i18n) support to a Spring Boot + Thymeleaf app:
Spring `MessageSource` configuration, per-locale message property files, a
`CookieLocaleResolver`-based locale switcher, and Thymeleaf `#{key}` integration.

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
app_name          → used in page titles
base_package      → Java package root
language          → respond in this language for ALL output (questions, explanations,
                    status messages, code comments, and user-facing copy in generated files)
                    IMPORTANT: if "zh-TW", "traditional-chinese", or "繁體中文":
                      use 繁體中文 (Traditional Chinese) throughout
                      NEVER use Simplified Chinese (简体中文) — they are different writing systems
                      Key differences: 體/体, 語/语, 資料/数据, 設定/设定, 確認/确认, 請/请
translate_terms   → whether to translate technical terms
beginner_friendly → if true, explain technical terms and decisions as you work
test_mode         → controls build verification
installed_modules → checked for hard and soft prerequisites
```

> **Language activation**: After reading `language` above, switch ALL your responses to that
> language immediately — including every question, status message, explanation, and all
> human-readable text in any generated files (HTML, Thymeleaf templates, property file comments).
> If `language` is not set, ask: "What language should I use? (English / 繁體中文 / other)"

---

## Step 1 — Prerequisites check

### Hard prerequisites

Verify `installed_modules` contains `"setup"` and `"scaffold"`.

If either is missing, stop and tell the user:
```
Missing prerequisites: {{missing_module}} must be installed first.
Run springboot-menu to see the guided install order.
```

### Soft prerequisites

Check if `"membership"` is in `installed_modules`.

- **If present**: the locale preference can be persisted per member (Step 7 will add this).
- **If absent**: locale is stored in a cookie only; note this as a future enhancement in Step 7.

---

## Beginner-Friendly Mode

If `beginner_friendly` is `true` in `.spring-config.json`, explain key concepts as you work. Examples:

- When configuring `MessageSource`: "Instead of hardcoding text like 'Sign In' directly in your HTML, you put it in a `.properties` file. Spring swaps in the right translation based on the user's language."
- When explaining `LocaleResolver`: "This decides which language to show each user — based on a cookie, a URL parameter, or their browser settings. We use cookies so the choice is remembered across page loads."
- When explaining fallback: "If a key is missing in the Chinese file, Spring automatically falls back to the English file rather than showing a broken page. That's why English is always the default."
- When showing `#{key}` in Thymeleaf: "The `#{}` syntax tells Thymeleaf to look up a message key. The text inside the tag is just a fallback that shows in your IDE preview — Spring replaces it at runtime."

---

## Step 2 — Ask which languages to support

Ask the user which languages they want to add. Suggest:

- **English (en)** — always recommended as the fallback
- **繁體中文 (zh-TW)** — Traditional Chinese
- **日本語 (ja)** — Japanese
- **Other** — ask for BCP 47 locale code (e.g. `ko`, `fr`, `de`)

> Default if not specified: English + 繁體中文.

Record the selected locales — you will need them for Steps 3–5.

---

## Step 3 — Add Spring MessageSource config

### `application.yml` (or `application-local.yml`)

Add the following under the `spring:` block:

```yaml
spring:
  messages:
    basename: i18n/messages
    encoding: UTF-8
    fallback-to-system-locale: false
```

If `beginner_friendly` is true, explain:
> "We set `fallback-to-system-locale: false` so Spring always falls back to the English
> `messages.properties` file rather than the JVM's system locale, which is unpredictable
> in cloud environments."

### `WebMvcConfig.kt`

Check if a `WebMvcConfigurer` already exists (e.g. from scaffold). If it does, add the beans and interceptor to it. If not, create:

```kotlin
package {{base_package}}.config

import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import org.springframework.web.servlet.LocaleResolver
import org.springframework.web.servlet.config.annotation.InterceptorRegistry
import org.springframework.web.servlet.config.annotation.WebMvcConfigurer
import org.springframework.web.servlet.i18n.CookieLocaleResolver
import org.springframework.web.servlet.i18n.LocaleChangeInterceptor
import java.util.Locale

@Configuration
class WebMvcConfig : WebMvcConfigurer {

    @Bean
    fun localeResolver(): LocaleResolver =
        CookieLocaleResolver("locale").apply {
            setDefaultLocale(Locale.ENGLISH)
            setCookieMaxAge(60 * 60 * 24 * 365) // 1 year
        }

    @Bean
    fun localeChangeInterceptor(): LocaleChangeInterceptor =
        LocaleChangeInterceptor().apply {
            paramName = "lang"
        }

    override fun addInterceptors(registry: InterceptorRegistry) {
        registry.addInterceptor(localeChangeInterceptor())
    }
}
```

---

## Step 4 — Create message property files

### `src/main/resources/i18n/messages.properties` (English — default fallback)

```properties
# Navigation
nav.home=Home
nav.dashboard=Dashboard
nav.profile=Profile
nav.logout=Sign Out
nav.login=Sign In
nav.admin=Admin

# Common actions
action.save=Save
action.cancel=Cancel
action.delete=Delete
action.edit=Edit
action.back=Back
action.submit=Submit
action.confirm=Confirm
action.search=Search

# Common labels
label.email=Email
label.name=Name
label.status=Status
label.createdAt=Created
label.actions=Actions

# Auth
auth.login.title=Sign In
auth.login.subtitle=Welcome back
auth.magic-link.prompt=Enter your email to receive a sign-in link
auth.magic-link.sent=Check your email for a sign-in link

# Membership
membership.status.pending=Pending Approval
membership.status.active=Active
membership.status.suspended=Suspended
membership.role.member=Member
membership.role.admin=Admin
membership.role.superAdmin=Super Admin

# Errors
error.notFound=Page not found
error.forbidden=Access denied
error.generic=Something went wrong. Please try again.

# Validation
validation.required=This field is required
validation.email=Please enter a valid email address
```

### `src/main/resources/i18n/messages_zh_TW.properties` (Traditional Chinese)

```properties
# Navigation
nav.home=首頁
nav.dashboard=主控台
nav.profile=個人資料
nav.logout=登出
nav.login=登入
nav.admin=管理後台

# Common actions
action.save=儲存
action.cancel=取消
action.delete=刪除
action.edit=編輯
action.back=返回
action.submit=送出
action.confirm=確認
action.search=搜尋

# Common labels
label.email=電子郵件
label.name=姓名
label.status=狀態
label.createdAt=建立時間
label.actions=操作

# Auth
auth.login.title=登入
auth.login.subtitle=歡迎回來
auth.magic-link.prompt=輸入您的電子郵件以接收登入連結
auth.magic-link.sent=請查看您的電子郵件以取得登入連結

# Membership
membership.status.pending=待審核
membership.status.active=已啟用
membership.status.suspended=已停用
membership.role.member=會員
membership.role.admin=管理員
membership.role.superAdmin=超級管理員

# Errors
error.notFound=找不到頁面
error.forbidden=存取被拒絕
error.generic=發生錯誤，請再試一次

# Validation
validation.required=此欄位為必填
validation.email=請輸入有效的電子郵件地址
```

### `src/main/resources/i18n/messages_ja.properties` (Japanese — only if user selected Japanese)

```properties
# Navigation
nav.home=ホーム
nav.dashboard=ダッシュボード
nav.profile=プロフィール
nav.logout=ログアウト
nav.login=ログイン
nav.admin=管理画面

# Common actions
action.save=保存
action.cancel=キャンセル
action.delete=削除
action.edit=編集
action.back=戻る
action.submit=送信
action.confirm=確認
action.search=検索

# Common labels
label.email=メールアドレス
label.name=名前
label.status=ステータス
label.createdAt=作成日時
label.actions=操作

# Auth
auth.login.title=ログイン
auth.login.subtitle=おかえりなさい
auth.magic-link.prompt=メールアドレスを入力してログインリンクを受け取ってください
auth.magic-link.sent=ログインリンクをメールで送信しました

# Membership
membership.status.pending=承認待ち
membership.status.active=有効
membership.status.suspended=停止中
membership.role.member=メンバー
membership.role.admin=管理者
membership.role.superAdmin=スーパー管理者

# Errors
error.notFound=ページが見つかりません
error.forbidden=アクセスが拒否されました
error.generic=エラーが発生しました。もう一度お試しください。

# Validation
validation.required=この項目は必須です
validation.email=有効なメールアドレスを入力してください
```

For any other language the user selected, create `messages_{{locale}}.properties` with
the same keys translated into that language.

---

## Step 5 — Add language switcher UI component

Create `src/main/resources/templates/fragments/locale-switcher.html`:

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" lang="en">
<body>

<div th:fragment="localeSwitcher" class="flex items-center gap-1 text-sm">
  <!-- Render one link per supported locale. Adapt this list to match the user's selected locales. -->

  <!-- English -->
  <a href="?lang=en"
     th:href="@{''(lang='en')}"
     class="px-2 py-1 rounded transition-colors hover:bg-gray-100"
     th:classappend="${#locale.language == 'en'} ? ' font-semibold bg-gray-100' : ' text-gray-500'">
    EN
  </a>

  <!-- Traditional Chinese (zh-TW) — include if user selected zh-TW -->
  <span class="text-gray-300 select-none">|</span>
  <a href="?lang=zh_TW"
     th:href="@{''(lang='zh_TW')}"
     class="px-2 py-1 rounded transition-colors hover:bg-gray-100"
     th:classappend="${#locale.language == 'zh'} ? ' font-semibold bg-gray-100' : ' text-gray-500'">
    繁中
  </a>

  <!-- Japanese (ja) — include if user selected ja -->
  <span class="text-gray-300 select-none">|</span>
  <a href="?lang=ja"
     th:href="@{''(lang='ja')}"
     class="px-2 py-1 rounded transition-colors hover:bg-gray-100"
     th:classappend="${#locale.language == 'ja'} ? ' font-semibold bg-gray-100' : ' text-gray-500'">
    日本語
  </a>

</div>

</body>
</html>
```

Only render `<span>` separators and locale links for the languages the user actually selected.

**How to include this fragment in the base layout:**

In `templates/layouts/base.html` (or equivalent), add inside the `<nav>` or header area:

```html
<div th:replace="~{fragments/locale-switcher :: localeSwitcher}"></div>
```

---

## Step 6 — Update Thymeleaf templates to use `#{key}`

Show the user how to update existing templates. The pattern:

- The fallback text inside the tag shows in IDE preview.
- `th:text="#{key}"` overrides it at runtime with the translated value.

**Examples:**

```html
<!-- Before -->
<span>Sign In</span>
<!-- After -->
<span th:text="#{auth.login.title}">Sign In</span>

<!-- Before -->
<a href="/dashboard">Dashboard</a>
<!-- After -->
<a href="/dashboard" th:text="#{nav.dashboard}">Dashboard</a>

<!-- Before -->
<h1>Welcome back</h1>
<!-- After -->
<h1 th:text="#{auth.login.subtitle}">Welcome back</h1>

<!-- Before -->
<button type="submit">Save</button>
<!-- After -->
<button type="submit" th:text="#{action.save}">Save</button>

<!-- For input placeholders -->
<!-- Before -->
<input type="email" placeholder="Email">
<!-- After -->
<input type="email" th:placeholder="#{label.email}">
```

Scan the project's existing templates and show 3–4 concrete substitutions from the actual
files (e.g. from `templates/auth/login.html`, `templates/layouts/nav.html`, or similar).

---

## Step 7 — Per-member language preference (membership conditional)

### If `membership` IS in `installed_modules`:

Add a Flyway migration (use the next available version number):

```sql
-- Add preferred_locale column to members table
ALTER TABLE members ADD COLUMN IF NOT EXISTS preferred_locale VARCHAR(10) DEFAULT 'en';
```

Create `src/main/kotlin/{{base_package}}/i18n/LocalePreferenceFilter.kt`:

```kotlin
package {{base_package}}.i18n

import {{base_package}}.repository.MemberRepository
import jakarta.servlet.FilterChain
import jakarta.servlet.http.HttpServletRequest
import jakarta.servlet.http.HttpServletResponse
import org.springframework.security.core.context.SecurityContextHolder
import org.springframework.stereotype.Component
import org.springframework.web.filter.OncePerRequestFilter
import org.springframework.web.servlet.LocaleResolver
import java.util.Locale

@Component
class LocalePreferenceFilter(
    private val memberRepository: MemberRepository,
    private val localeResolver: LocaleResolver,
) : OncePerRequestFilter() {

    override fun doFilterInternal(
        request: HttpServletRequest,
        response: HttpServletResponse,
        filterChain: FilterChain,
    ) {
        // Only apply if the user is authenticated and no explicit lang= param was given
        if (request.getParameter("lang") == null) {
            val auth = SecurityContextHolder.getContext().authentication
            if (auth != null && auth.isAuthenticated && auth.name != "anonymousUser") {
                memberRepository.findByEmail(auth.name).ifPresent { member ->
                    val preferred = member.preferredLocale
                    if (!preferred.isNullOrBlank()) {
                        localeResolver.setLocale(request, response, Locale.forLanguageTag(preferred))
                    }
                }
            }
        }
        filterChain.doFilter(request, response)
    }
}
```

Add `preferredLocale` field to the `Member` entity:

```kotlin
@Column(name = "preferred_locale", length = 10)
var preferredLocale: String? = "en"
```

Add a language preference selector to the member profile edit page
(`templates/member/profile.html`):

```html
<div class="form-group">
  <label th:text="#{label.language}">Language</label>
  <select name="preferredLocale" class="form-select">
    <option value="en"    th:selected="${member.preferredLocale == 'en'}">English</option>
    <option value="zh-TW" th:selected="${member.preferredLocale == 'zh-TW'}">繁體中文</option>
    <!-- Add additional locales here if selected in Step 2 -->
  </select>
</div>
```

Add the `label.language` key to all message property files:

```properties
# messages.properties
label.language=Language

# messages_zh_TW.properties
label.language=語言

# messages_ja.properties (if applicable)
label.language=言語
```

Update `MemberProfileController` to handle `preferredLocale` in the profile update POST.

Register the filter in `SecurityConfig` (or as a `@Bean` — Spring Boot auto-detects `OncePerRequestFilter` components).

### If `membership` is NOT in `installed_modules`:

Skip this step and add a note:

```
Note: Per-member language preference requires the membership module.
Install springboot-membership to enable persistent language preferences.
For now, the selected locale is stored in a browser cookie.
```

---

## Step 8 — Unit tests

If `test_mode` is NOT `"token-save"`:

Create `src/test/kotlin/{{base_package}}/i18n/LocaleResolutionTest.kt`:

```kotlin
package {{base_package}}.i18n

import org.junit.jupiter.api.Test
import org.junit.jupiter.api.extension.ExtendWith
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.context.MessageSource
import org.springframework.test.context.ActiveProfiles
import org.springframework.test.context.junit.jupiter.SpringExtension
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.get
import java.util.Locale
import org.assertj.core.api.Assertions.assertThat

@SpringBootTest
@AutoConfigureMockMvc
@ActiveProfiles("test")
@ExtendWith(SpringExtension::class)
class LocaleResolutionTest {

    @Autowired
    lateinit var messageSource: MessageSource

    @Test
    fun `default locale resolves English messages`() {
        val msg = messageSource.getMessage("nav.home", null, Locale.ENGLISH)
        assertThat(msg).isEqualTo("Home")
    }

    @Test
    fun `zh_TW locale resolves Traditional Chinese messages`() {
        val msg = messageSource.getMessage("nav.home", null, Locale.forLanguageTag("zh-TW"))
        assertThat(msg).isEqualTo("首頁")
    }

    @Test
    fun `missing key in zh_TW falls back to English`() {
        // If a key exists only in messages.properties, zh_TW should fall back
        // This test documents the fallback behaviour — add a key only to messages.properties
        // to verify (or use an intentionally missing key with a default)
        val msg = messageSource.getMessage("nav.home", null, "FALLBACK", Locale.forLanguageTag("zh-TW"))
        assertThat(msg).isNotEqualTo("FALLBACK") // key exists, no fallback needed
    }

    @Test
    fun `all supported locales resolve nav_home without throwing`() {
        val locales = listOf(
            Locale.ENGLISH,
            Locale.forLanguageTag("zh-TW"),
            // Add Locale.JAPANESE here if user selected ja
        )
        for (locale in locales) {
            val msg = messageSource.getMessage("nav.home", null, locale)
            assertThat(msg).isNotBlank()
        }
    }
}
```

---

## Step 9 — Verify

Apply `test_mode`:

- `"token-save"` → Skip all test execution and test file creation.
- `"build-only"` → `./gradlew build -x test`
- `"build-and-test"` → `./gradlew build`

---

## Step 10 — Git commit

```bash
git add -A
git commit -m "feat: add i18n support with Spring MessageSource and locale switcher"
```

---

## Step 11 — Update `installed_modules`

Add `"i18n"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
i18n support is ready.

To switch language:
  Append ?lang=en or ?lang=zh_TW (or ?lang=ja) to any URL.
  The choice is saved in a cookie and persists across page loads.

Locale switcher fragment:
  th:replace="~{fragments/locale-switcher :: localeSwitcher}"
  Add this wherever you want the language switcher to appear (e.g. nav bar).

Message files:
  src/main/resources/i18n/messages.properties        ← English (default)
  src/main/resources/i18n/messages_zh_TW.properties  ← Traditional Chinese
  (add more keys here as you build new pages)

Next steps:
- Update your existing templates to use #{key} instead of hardcoded text
- springboot-membership → enables per-member language preference persistence
```
