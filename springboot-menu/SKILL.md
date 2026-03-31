---
name: springboot-menu
description: >
  TRIGGER this skill when the user says "show me the menu", "what modules are available",
  "project status", "where do I start", "what should I install next", "what do I have installed",
  "help", "list skills", "run springboot-menu", or seems confused about which skill to run next.
  Also trigger whenever ANY other skill fails a prerequisites check — this skill explains the
  full project status and guides the user on what to do next.
  Works even without a .spring-config.json file (handles fresh-start users too).
---

## springboot-menu — Project Status Dashboard & Guided Workflow

---

### Step 0 — Detect project state

Try to read `.spring-config.json` from the current directory.

**If the file does NOT exist**, detect language from the user's prior messages if possible; if they wrote in Traditional Chinese (繁體中文), respond in Traditional Chinese — otherwise default to English. Show this message:

```
👋 It looks like this is a brand-new project.

Start here → run /springboot-welcome

This will:
  1. Ask your preferred language
  2. Check and install any missing tools (Java, Docker, etc.)
  3. Guide you to run /springboot-setup next
```

Then stop — do not show the full module status yet.

**If the file exists but `"welcome"` is NOT in `installed_modules`**, show a gentle reminder before continuing:

```
⚠️  Heads up: it looks like you haven't run /springboot-welcome yet.
    Consider running it first — it sets your language preference and
    checks that your development environment is ready.
    (You can skip this if your environment is already set up.)
```

Then continue to Step 1 regardless.

**If the file exists and `"welcome"` IS in `installed_modules`**, continue to Step 1 directly.

---

### Step 1 — Read config and extract state

Read `.spring-config.json`. Extract the following fields:
- `app_name`
- `language`
- `beginner_friendly`
- `installed_modules` (array — may be empty `[]`)
- `test_mode`
- `soft_delete`

> **Language activation**: After reading `language` above, switch ALL your responses to that
> language immediately — including every dashboard label, module description, status indicator,
> recommendation, unblock tip, mentor mode question, diagnostic message, and quick-start guide.
> If `language` is not set, ask: "What language should I use? (English / 繁體中文 / other)"

> **Traditional vs Simplified Chinese**: If `language` is `"zh-TW"`, `"traditional-chinese"`,
> or `"繁體中文"`, use Traditional Chinese (繁體中文) throughout — NEVER Simplified Chinese (简体中文).
> They are different writing systems. Key Traditional terms: 設定 (not 设定), 資料 (not 数据),
> 確認 (not 确认), 請 (not 请), 語言 (not 语言), 會員 (not 会员), 管理員 (not 管理员),
> 模組 (not 模块), 安裝 (not 安装), 設置 (not 设置).

---

### Step 2 — Build the status dashboard

Print the header:
```
📦 {{app_name}} — Spring Boot Project Status
```

Then show a checklist of ALL 22 modules grouped by layer. For each module, display one of:
- ✅ if the module key is present in `installed_modules`
- ⬜ if NOT installed but all prerequisites are met (ready to install now)
- 🔒 if NOT installed AND one or more prerequisites are missing — show which prereqs are missing in parentheses

**Module key names** (these are the string values stored in `installed_modules`):
- `welcome`
- `setup`, `scaffold`, `db`, `app-config`, `mail`
- `gcp`, `deploy`
- `auth-google`, `auth-magic-link`
- `membership`, `membership-apply`, `admin-portal`
- `file-upload`, `item-submit`, `announcements`, `stats-summary`, `data-export`
- `audit-log`, `integration-tests`
- `prototype-ui`, `ide-setup`
- `menu` ← this skill itself (include it as ✅ if present, otherwise ⬜ — it has no prereqs)

**Module groups and dependency rules:**

```
🛠️  ENVIRONMENT
  ✅/⬜  springboot-welcome         — Language + environment setup (run before all)    [needs: nothing]

🗺️  NAVIGATION
  ✅/⬜  springboot-menu            — Project status dashboard (this skill)           [needs: nothing]

🏗️  FOUNDATION
  ✅/⬜  springboot-setup           — Project config wizard                           [needs: welcome]
  ✅/⬜  springboot-scaffold        — Spring Boot project, Thymeleaf, Tailwind         [needs: setup]
  ✅/⬜  springboot-db-setup        — PostgreSQL + Docker Compose + Flyway             [needs: setup, scaffold]
  ✅/⬜  springboot-app-config      — DB-backed app settings store                    [needs: setup, scaffold, db]
  ✅/⬜  springboot-mail            — SMTP email + templates + scheduling              [needs: setup, scaffold, db]

☁️  GCP & DEPLOYMENT
  ✅/⬜  springboot-gcp-setup       — gcloud CLI + GCP project + APIs                 [needs: setup]
  ✅/⬜  springboot-deploy          — Dockerfile + Cloud Run + deploy.sh              [needs: setup, scaffold, db, gcp]

🔐  AUTHENTICATION (pick at least one)
  ✅/⬜  springboot-auth-google     — Google OAuth2 login                             [needs: setup, scaffold, db, gcp]
  ✅/⬜  springboot-auth-magic-link — Passwordless email login                        [needs: setup, scaffold, db, mail]

👥  MEMBERSHIP
  ✅/⬜  springboot-membership      — Member entity, roles, CRUD                      [needs: scaffold, db, auth-google OR auth-magic-link]
  ✅/⬜  springboot-membership-apply — Public /apply form                             [needs: scaffold, db, mail, membership]
  ✅/⬜  springboot-admin-portal    — Admin UI + preview-as-member                    [needs: scaffold, db, membership]

🧩  FEATURES
  ✅/⬜  springboot-file-upload     — File storage (local/GCS)                        [needs: setup, scaffold, db]
  ✅/⬜  springboot-item-submit     — Member submission/record system                 [needs: scaffold, db, membership]
  ✅/⬜  springboot-announcements   — Broadcast notifications                         [needs: scaffold, db, membership]
  ✅/⬜  springboot-stats-summary   — Activity summaries by period                    [needs: scaffold, db, membership]
  ✅/⬜  springboot-data-export     — Export to CSV / Excel / Google Sheets           [needs: scaffold, db, membership]

🔍  QUALITY & OBSERVABILITY
  ✅/⬜  springboot-audit-log       — Track who changed what, when                    [needs: setup, scaffold, db]
  ✅/⬜  springboot-integration-tests — Cross-module test strategy                    [needs: scaffold, db + at least 2 non-foundation modules]

🎨  OPTIONAL
  ✅/⬜  springboot-prototype-ui    — Static HTML UI mockup (no backend needed)       [needs: nothing]
  ✅/⬜  springboot-ide-setup       — IntelliJ IDEA setup guide                       [needs: nothing]
```

**Logic for ⬜ vs 🔒:**

- ⬜ (ready): All listed prerequisites are present in `installed_modules`
- 🔒 (blocked): One or more prerequisites are missing — show which ones in parentheses, e.g. `🔒 springboot-auth-google (missing: gcp)`
- Special case — `springboot-membership`: requires `auth-google` OR `auth-magic-link` — if EITHER is in `installed_modules`, this prereq is satisfied
- Special case — `springboot-integration-tests`: requires `scaffold` and `db` in `installed_modules`, PLUS at least 2 modules from the non-foundation list (`auth-google`, `auth-magic-link`, `membership`, `membership-apply`, `admin-portal`, `file-upload`, `item-submit`, `announcements`, `stats-summary`, `data-export`, `audit-log`) — if fewer than 2, show `🔒 (need 2+ feature modules installed)`

---

### Step 3 — Recommended next step

After the checklist, print:
```
🎯 Recommended next step:
```

Then apply this decision logic (first matching rule wins):

1. If `setup` not in `installed_modules` → recommend `springboot-setup`
   - Show: "Run **springboot-setup** to configure your project. This is always the first step — it creates `.spring-config.json` which every other skill depends on."

2. Else if `scaffold` not in `installed_modules` → recommend `springboot-scaffold`
   - Show: "Run **springboot-scaffold** to generate your Spring Boot project structure with Gradle, Thymeleaf, and Tailwind."

3. Else if `db` not in `installed_modules` → recommend `springboot-db-setup`
   - Show: "Run **springboot-db-setup** to add PostgreSQL, Flyway migrations, and a Docker Compose dev database."

4. Else if NEITHER `auth-google` NOR `auth-magic-link` is in `installed_modules` → explain the auth choice:
   - Show: "You need at least one authentication method before adding members. Choose:
     - **springboot-auth-google** — Fastest if your users have Google accounts. Requires GCP setup first (run `springboot-gcp-setup` if not done).
     - **springboot-auth-magic-link** — Works with any email address, no Google account needed. Requires mail setup first (run `springboot-mail` if not done).
     - You can install both."

5. Else if `membership` not in `installed_modules` → recommend `springboot-membership`
   - Show: "Run **springboot-membership** to add the Member entity, roles (MEMBER/ADMIN/SUPER_ADMIN), and the admin member list."

6. Else → list all ⬜ (ready) modules as options, sorted by how many other modules they unblock (most first)
   - Show: "You have a solid foundation. Here are the modules ready to install (sorted by what they unlock next):"
   - List each ⬜ module with a one-line description

---

### Step 4 — Constructive guidance on blocked modules

For any 🔒 blocked module that is **only one prerequisite away** from being unblocked (i.e., exactly 1 prereq missing), show a brief tip:

```
💡 To unblock springboot-auth-magic-link:
   → Install springboot-mail first (sets up SMTP email sending)

💡 To unblock springboot-membership-apply:
   → Install springboot-mail first, then springboot-membership
```

**Only show tips for modules with exactly 1 missing prerequisite.** Do not overwhelm with tips for deeply-blocked modules (2+ missing prereqs).

If `beginner_friendly` is true, add a one-sentence plain-language explanation of what the blocking module does:
- e.g., "springboot-mail sets up your app's ability to send emails — without it, magic link login has no way to deliver login links."

---

### Step 4b — Newbie Quick-Start Guide

If `installed_modules` is empty OR contains only `["setup"]`, show this numbered guide (translated to the configured `language`):

---
🚀 **Quick-Start Order** (follow this sequence for a new project):

1. `springboot-setup` — Answer ~13 questions → creates `.spring-config.json`
2. `springboot-scaffold` — Generates the Spring Boot project (Gradle + Thymeleaf + Tailwind)
3. `springboot-db-setup` — Adds PostgreSQL database + Docker Compose + Flyway migrations
4. `springboot-mail` — Sets up SMTP email (needed for passwordless login or notifications)
5. `springboot-gcp-setup` — Connects to Google Cloud Platform (needed for Google login + deploy)
6. **Choose login method** (at least one required):
   - `springboot-auth-google` — Google OAuth2 (fastest if users have Google accounts)
   - `springboot-auth-magic-link` — Passwordless email login (works with any email)
7. `springboot-membership` — Member entity, roles (MEMBER / ADMIN / SUPER_ADMIN), member list
8. `springboot-admin-portal` — Admin dashboard with member management
9. Add features as needed: `file-upload`, `item-submit`, `announcements`, `stats-summary`, `data-export`, `audit-log`
10. `springboot-deploy` — Docker image + Cloud Run deployment script
11. `springboot-integration-tests` — Cross-module integration test strategy

💡 **Opening the project in IntelliJ IDEA** (after Step 2):
   File → Open → select the project folder → click "Import Gradle project" when prompted.
   Run `springboot-ide-setup` any time for a full IDE walkthrough.

---

If `ide-setup` is NOT in `installed_modules`, also add this tip at the end of Step 3:

> 💡 **IDE tip**: Run `springboot-ide-setup` any time to get a guided IntelliJ IDEA setup walkthrough.

---

### Step 5 — Mentor Mode (active investigation)

After the dashboard, do NOT just show a static reference screen. Act as a senior developer sitting next to the user. Ask:

```
What are you trying to do, or what problem are you running into?
```

Then based on their answer, actively help:

---

#### If they describe a goal (e.g., "I want to add email login", "I want to build a member portal"):

1. Map the goal to the right module(s)
2. Check if prerequisites are already satisfied (from `installed_modules`)
3. Give a step-by-step "here's exactly what to do" plan — not a pointer to docs, a concrete ordered list
4. Offer to kick off the first step right now: "Want me to run **springboot-mail** now?"

Example:
> User: "I want to add passwordless login"
> Mentor: "That's springboot-auth-magic-link. You need mail set up first for sending login links. You have db installed but not mail — let's install springboot-mail first (SMTP config + templates), then auth-magic-link. Want me to start springboot-mail now?"

---

#### If they describe a problem (e.g., "the app won't start", "login isn't working", "emails aren't sending"):

Run a diagnostic investigation:
1. Check `.spring-config.json` for relevant config gaps
2. Ask targeted questions to narrow down the issue:
   - "Is this happening locally or in production?"
   - "What exact error do you see? Paste the stack trace."
   - "Which profile are you running? (`local` or `prod`?)"
3. Check if the relevant module is even installed
4. Work through the common causes checklist for that module (see **Common Issues** section below)
5. Give a concrete ordered checklist of things to try, most likely first
6. Offer to dig deeper: "Want me to look at your SecurityConfig?" / "Can you run `./gradlew build` and paste the output?"

---

#### If they say "I don't know where to start":

Ask 2–3 clarifying questions:
- "Will this app have user accounts / members?"
- "Do your users have Google accounts, or do you need email-only login?"
- "Are you planning to deploy to Google Cloud, or somewhere else?"

Based on their answers, recommend a **personalized install order** — not just the generic one. Explain WHY that order fits their specific case.

Example:
> "Since you want email-only login and no GCP, skip gcp-setup for now. Your path: setup → scaffold → db → mail → auth-magic-link → membership. Want to start?"

---

#### If they ask "what does X module do?":

Give a plain-language explanation covering:
- What problem it solves (1 sentence)
- What it creates (tables, pages, services — be specific)
- How it connects to modules they already have installed
- Any gotchas or things to know upfront

---

### Common issues reference (use during diagnosis)

**springboot-db-setup:**
- Docker not running → `docker ps` should show the db container
- Wrong DB URL in `application-local.yml` → check `spring.datasource.url` matches `docker-compose.yml`
- Flyway migration failed → check `flyway_schema_history` table; look for failed migrations with `success = false`
- Port conflict → another PostgreSQL instance on port 5432?

**springboot-auth-google:**
- Redirect URI mismatch → in GCP Console, Authorized Redirect URIs must include `http://localhost:8080/login/oauth2/code/google`
- Missing `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` env vars → check `application-local.yml` or `.env`
- "Access blocked: This app's request is invalid" → OAuth consent screen not configured in GCP Console
- Member stuck in PENDING status → check `CustomOAuth2UserService` — new users are created as PENDING by default

**springboot-auth-magic-link:**
- SMTP not configured → check `spring.mail.*` properties in `application-local.yml`
- MailHog not running locally → `docker ps` should show mailhog; check `localhost:8025`
- Token expired immediately → check server clock vs token expiry (tokens are 15 minutes)
- "No such file" for email template → check `resources/templates/mail/magic-link.html` exists

**springboot-mail:**
- MailHog not running → add MailHog to `docker-compose.yml` and restart
- Missing `from` address → check `spring.mail.properties.mail.smtp.from` in config
- Production SMTP failing → verify `SMTP_HOST`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD` env vars are set in Cloud Run

**springboot-gcp-setup:**
- APIs not enabled → run `gcloud services enable ...` or check GCP Console > APIs & Services
- Wrong project ID → `gcloud config get-value project` should match `gcp_project_id` in `.spring-config.json`
- Missing service account key → check if `service-account-key.json` exists and is referenced correctly

**springboot-deploy:**
- Cloud Run service account missing roles → needs `roles/run.invoker`, `roles/cloudsql.client`, `roles/artifactregistry.writer`
- Wrong region → `gcp_region` in config must match where Cloud Run service is deployed
- Artifact Registry not enabled → `gcloud services enable artifactregistry.googleapis.com`
- Build fails in Cloud Build → check `cloudbuild.yaml` and that Gradle wrapper is executable (`chmod +x gradlew`)

---

### Quick answer offer (fallback if no problem/goal stated)

If the user just wants reference info without a specific problem, offer:

```
Ask me anything:
  • "What does [module name] do?"
  • "Why do I need [module] before [other module]?"
  • "What's the fastest path to [feature]?"
  • "I want to add [feature] — where do I start?"
```

---

### Beginner-friendly mode

If `beginner_friendly` is `true` in the config, add a brief plain-language explanation after each section header:

- After **🏗️ FOUNDATION**: "These must be installed first — they set up the project structure, database, and email that everything else depends on."
- After **☁️ GCP & DEPLOYMENT**: "GCP (Google Cloud Platform) is where your app will run in production. You only need this if you're deploying to Google Cloud."
- After **🔐 AUTHENTICATION**: "Authentication means 'how do users prove who they are when logging in'. You need at least one method before you can build any member features."
- After **👥 MEMBERSHIP**: "These modules manage your users — their profiles, roles, and how they join your app."
- After **🧩 FEATURES**: "These are the main features your members will use once they're logged in."
- After **🔍 QUALITY & OBSERVABILITY**: "These modules help you understand what's happening in your app — audit trails and automated tests."
- After **🎨 OPTIONAL**: "These can be installed any time, independently of the main stack."

---

## Note for other skill authors

When your skill fails a prerequisites check, include this line in your error message:

```
Run springboot-menu to see your full project status and get guided next steps.
```

This ensures users always have a path forward instead of a dead end.
