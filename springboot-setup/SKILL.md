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

1. Asks the user 10–12 configuration questions
2. Writes `.spring-config.json` to the project root
3. Marks `"setup"` in `installed_modules`

---

## Step 1 — Run the interview

Ask the user the following questions. You can ask them all at once in a numbered list, or group them into
two rounds if that feels less overwhelming. Accept "TBD" or blank for optional fields.

```
I need to collect some configuration details before scaffolding your Spring Boot project.
Please answer the following (you can write "TBD" for anything you don't know yet):

1.  App name            — The name of your application (e.g. "MemberHub", "CertTrack")
2.  Brand name          — Your company or brand name (e.g. "Acme Corp") — can be same as app name
3.  Base Java package   — e.g. "com.example.myapp" or "io.myorg.appname"
4.  App URL             — Production URL, e.g. "https://app.example.com" (or TBD)
5.  Super admin email   — The first admin account email, e.g. "you@example.com"
6.  Database name       — PostgreSQL DB name, e.g. "myapp_db"
7.  GCP project ID      — Your Google Cloud project ID (or TBD if not set up yet)
8.  GCP region          — e.g. "us-central1", "asia-east1" (or TBD)
9.  Preferred language  — Language for Claude's responses: "English", "Traditional Chinese", etc.
10. Translate terms?    — Should technical terms (Controller, Service, Entity, etc.) be translated,
                          or kept in English even when responding in another language? (yes/no)
11. Test mode           — How should tests be handled?
                            "token-save"     → skip all tests (fastest, fewest tokens)
                            "build-only"     → run ./gradlew build -x test (verify compilation only)
                            "build-and-test" → run ./gradlew build (full build + tests)
```

---

## Step 2 — Confirm and write config

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

## Step 3 — Write `.spring-config.json`

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
  "installed_modules": ["setup"]
}
```

---

## Step 4 — Confirm success

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

## Notes on `language` and `translate_terms`

- All subsequent skills should respond in the user's preferred `language`.
- If `translate_terms` is `false`, keep technical terms (Controller, Service, Repository, Entity, Bean,
  Flyway, etc.) in English even when writing in another language.
- If `translate_terms` is `true`, translate everything including technical terms.
