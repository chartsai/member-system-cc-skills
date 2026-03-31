---
name: springboot-setup
description: >
  TRIGGER this skill immediately when the user says "set up a new Spring Boot project", "start a Spring Boot
  project", "initialize Spring Boot", "configure my project settings", "run springboot-setup", or asks
  to create any kind of Spring Boot application from scratch. This is always the FIRST skill to run before
  any other springboot-* skill. It interviews the user and writes `.spring-config.json` to the project root.
---

# springboot-setup

This is the **first skill to run** for any new Spring Boot project. It collects project configuration through
a short interview and writes `.spring-config.json` to the current directory. Every other `springboot-*` skill
depends on this file.

---

## What this skill does

1. Asks for the user's preferred language first, then switches to that language immediately
2. Asks the remaining 11 configuration questions in the chosen language
3. Writes `.spring-config.json` to the project root
4. Marks `"setup"` in `installed_modules`

---

## Step 1 — Ask for preferred language first

Before anything else, ask this single question in **all supported languages** so the user can answer in their language:

```
What is your preferred language for this setup?
您希望使用哪種語言進行設定？
セットアップに使用する言語を選択してください。

  1. English
  2. 繁體中文 (Traditional Chinese)
  3. 日本語
  4. Other — please specify / 其他，請說明 / その他（入力してください）
```

Once the user replies, **switch all subsequent output to that language immediately** — including every question, explanation, and confirmation in the steps below.

Also ask at this point:

```
Should technical terms (Controller, Service, Entity, Repository, etc.) be translated,
or kept in English even when responding in your language? (yes = translate / no = keep in English)
```

---

## Step 2 — Run the rest of the interview

Now ask the remaining questions in the chosen language. Accept "TBD" or blank for optional fields.

```
Great! I'll continue in <language>. Please answer the following
(you can write "TBD" for anything you don't know yet):

1.  App name            — The name of your application (e.g. "MemberHub", "CertTrack")
2.  Brand name          — Your company or brand name (e.g. "Acme Corp") — can be same as app name
3.  Base Java package   — e.g. "com.example.myapp" or "io.myorg.appname"
4.  App URL             — Production URL, e.g. "https://app.example.com" (or TBD)
5.  Super admin email   — The first admin account email, e.g. "you@example.com"
6.  Database name       — PostgreSQL DB name, e.g. "myapp_db"
7.  GCP project ID      — Your Google Cloud project ID (or TBD if not set up yet)
8.  GCP region          — e.g. "us-central1", "asia-east1" (or TBD)
9.  Test mode           — How should tests be handled?
                            "token-save"     → skip all tests (fastest, fewest tokens)
                            "build-only"     → run ./gradlew build -x test (verify compilation only)
                            "build-and-test" → run ./gradlew build (full build + tests)
10. Soft delete?        — Should deleted records be permanently removed, or kept with a
                          deleted_at timestamp (recommended — allows data recovery)?
                          Default: yes (soft delete) — type "no" only for permanent delete.
11. Beginner-friendly?  — Would you like Claude to explain technical terms and decisions
                          as it works? (e.g., "A Flyway migration is a versioned SQL script...")
                          Default: no — type "yes" if you are new to Spring Boot / Java.
```

Translate the question text above into the chosen language before asking.

---

## Step 3 — Confirm and write config

After collecting answers, show the user a summary:

```
Here's your project configuration:

  app_name:          <value>
  brand_name:        <value>
  base_package:      <value>
  app_url:           <value>
  super_admin_email: <value>
  db_name:           <value>
  gcp_project_id:    <value>
  gcp_region:        <value>
  language:          <value>
  translate_terms:   <true|false>
  test_mode:         <value>

Shall I write this to .spring-config.json and proceed?
```

Wait for confirmation. If the user wants to change anything, update the values and show the summary again.

---

## Step 4 — Write `.spring-config.json`

Write this file to the **current working directory** (the project root):

```json
{
  "app_name": "<app_name>",
  "brand_name": "<brand_name>",
  "base_package": "<base_package>",
  "app_url": "<app_url>",
  "super_admin_email": "<super_admin_email>",
  "db_name": "<db_name>",
  "gcp_project_id": "<gcp_project_id>",
  "gcp_region": "<gcp_region>",
  "language": "<language>",
  "translate_terms": <true|false>,
  "test_mode": "<token-save|build-only|build-and-test>",
  "soft_delete": <true|false>,
  "beginner_friendly": <true|false>,
  "installed_modules": ["setup"]
}
```

---

## Step 5 — Confirm success

Tell the user:

```
.spring-config.json has been written. Setup complete.

Next step: run springboot-scaffold to create your Spring Boot project structure.

Installed modules: [setup]
```

---

## Notes on `test_mode`

Every subsequent skill reads `test_mode` from the config and applies it:

| Value | Behavior |
|---|---|
| `"token-save"` | Skip all test execution and test file creation |
| `"build-only"` | Run `./gradlew build -x test` to verify compilation |
| `"build-and-test"` | Run `./gradlew build` (full build including tests) |

The user can change `test_mode` at any time by editing `.spring-config.json` directly.

---

## Notes on `soft_delete`

When `soft_delete` is `true` (the default), every skill that implements delete functionality will:
- Add a `deleted_at TIMESTAMPTZ` column to entities instead of deleting rows
- Use Hibernate's `@SQLRestriction("deleted_at IS NULL")` to auto-filter queries
- Add admin "restore" endpoints for soft-deleted records
- Add a "Show deleted" toggle to admin list views

When `soft_delete` is `false`, skills will use hard (permanent) DELETE SQL.

**Each skill that implements delete will note:** "This module uses soft delete by default.
If you configured `soft_delete: false`, tell Claude to use hard delete instead."

---

## Notes on `beginner_friendly`

When `beginner_friendly` is `true`:
- Every skill will explain technical terms inline as it introduces them
- Example: "We're adding a Flyway migration — this is a versioned SQL file that runs automatically
  each time the app starts, ensuring your database schema stays in sync with your code."
- Skills will explain WHY, not just WHAT
- Code patterns will have extra comments explaining the reasoning

When `beginner_friendly` is `false` (the default), skills assume familiarity with Spring Boot,
Java, SQL, and standard web patterns.

---

## Notes on `language` and `translate_terms`

- All subsequent skills should respond in the user's preferred `language` for ALL output — questions, explanations, status messages, code comments, and user-facing copy in generated files.
- If `translate_terms` is `false`, keep technical terms (Controller, Service, Repository, Entity, Bean, Flyway, etc.) in English even when writing in another language.
- If `translate_terms` is `true`, translate everything including technical terms.

> **Traditional vs Simplified Chinese**: These are different writing systems — never use one in place of the other.
> If the user specifies Traditional Chinese (繁體中文), Taiwan, or Hong Kong context, always use Traditional characters.
> Common Traditional terms: 設定 (not 设定), 資料 (not 数据), 確認 (not 确认), 請 (not 请),
> 語言 (not 语言), 會員 (not 会员), 管理員 (not 管理员), 功能 (not 功能 — same here but check others).
> When asking for language preference in Step 1, explicitly offer "繁體中文" as an option, not just "Chinese".

> **Language activation for all skills**: Once `language` is set in `.spring-config.json`, every subsequent skill
> reads it in Step 0 and switches its entire interaction to that language immediately — including every question,
> option, and all human-readable text in generated files.
