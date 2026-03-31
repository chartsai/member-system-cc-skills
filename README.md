# 🍃 Claude Code Skills - Membership System｜Claude Code 會員系統技能包

**27 modular Claude skills for building production-ready Spring Boot web apps — step by step.**

> Stop copy-pasting boilerplate. Just run a skill, answer a few questions, and get working code.

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)
[![Skills: 27](https://img.shields.io/badge/Skills-27-blue.svg)](#all-27-skills)
[![Spring Boot: 3.x](https://img.shields.io/badge/Spring%20Boot-3.x-brightgreen.svg)](https://spring.io/projects/spring-boot)
[![Language: Kotlin](https://img.shields.io/badge/Language-Kotlin-purple.svg)](https://kotlinlang.org)

> [繁體中文說明請點此](#-claude-code-會員系統技能包)

---

## ⚡ Quick Start (TL;DR)

**1. Install Claude Code CLI:**

```bash
npm install -g @anthropic-ai/claude-code
```

**2. Clone this repo and install the skills:**

```bash
git clone https://github.com/chartsai/member-system-cc-skills.git
mkdir -p ~/.claude/commands && cp -r member-system-cc-skills/springboot-* ~/.claude/commands/
```

**3. Run the environment setup skill — it installs Java, Docker, and everything else automatically:**

```bash
claude
```

```
/springboot-env-setup
```

> Don't have npm or git yet? → [Manual install guide](#prerequisites)

---

## What is this?

A collection of **Claude AI skills** that guide you through building a full-featured membership web application — one module at a time. Each skill knows your project config, checks prerequisites, generates production-ready code, writes unit tests, and commits to git. When everything is ready, it deploys to **[Google Cloud Server - Google Cloud](https://cloud.google.com/run)** with a single command.

**You don't need to be an expert.** Skills explain what they're doing as they go. Think of it as a **senior developer sitting next to you**, implementing features and teaching you along the way.

The web app is built on [Spring Boot](https://spring.io/projects/spring-boot). Spring Boot is a popular Java/Kotlin framework for building web applications. It handles routing, database access, authentication, and background jobs — so you can focus on your product. It's widely used in both startups and enterprise backends.

---

## What you'll build

A full-stack membership web application with:

- 🔐 **Authentication** — Google OAuth2 login or passwordless email magic links
- 👥 **Membership system** — roles, approval workflow, admin portal
- 📧 **Email** — templated emails, scheduled delivery, MailHog for local dev
- 📁 **File uploads** — local dev + Google Cloud Storage in production
- 📣 **Announcements** — admin broadcasts to members
- 📊 **Stats & exports** — activity summaries, CSV/Excel/Sheets export
- 🔍 **Search** — PostgreSQL full-text search, no Elasticsearch needed
- 🔔 **Notifications** — in-app notification bell with unread count
- 📝 **Content / Blog** — articles, categories, tags, Quill.js rich text editor
- 🌐 **i18n** — multi-language UI (English, 繁體中文, Japanese, and more)
- 📋 **Audit log** — who changed what, when
- ☁️ **GCP deployment** — Dockerfile + Cloud Run + Secret Manager

---

## Prerequisites

### macOS

**Step 1 — Install [Homebrew](https://brew.sh)** (package manager):

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

**Step 2 — Install [Node.js](https://nodejs.org)** (`npm` is bundled with it — macOS does not include npm by default):

```bash
brew install node
```

**Step 3 — Install the rest:**

| Tool | Purpose | Command |
|---|---|---|
| [Java 21+](https://adoptium.net) | Spring Boot runtime | `brew install --cask temurin@21` |
| [Docker Desktop](https://www.docker.com/products/docker-desktop) | Local PostgreSQL + MailHog | `brew install --cask docker` |
| [IntelliJ IDEA](https://www.jetbrains.com/idea/) | IDE (Community is free) | `brew install --cask intellij-idea-ce` |
| [gcloud CLI](https://cloud.google.com/sdk/docs/install) | GCP deployment (optional) | `brew install --cask google-cloud-sdk` |

---

### Windows

**`winget`** is built into Windows 10 (1809+) and Windows 11. Open **PowerShell** or **Command Prompt** and run:

**Step 1 — Install [Node.js](https://nodejs.org)** (`npm` is bundled with it — Windows does not include npm by default):

```
winget install OpenJS.NodeJS
```

> **Restart your terminal** after this step so `npm` is available on your PATH.

**Step 2 — Install the rest:**

| Tool | Purpose | Command |
|---|---|---|
| [Java 21+](https://adoptium.net) | Spring Boot runtime | `winget install EclipseAdoptium.Temurin.21.JDK` |
| [Docker Desktop](https://www.docker.com/products/docker-desktop) | Local PostgreSQL + MailHog | `winget install Docker.DockerDesktop` |
| [IntelliJ IDEA](https://www.jetbrains.com/idea/) | IDE (Community is free) | `winget install JetBrains.IntelliJIDEA.Community` |
| [gcloud CLI](https://cloud.google.com/sdk/docs/install) | GCP deployment (optional) | `winget install Google.CloudSDK` |

---

## Quick Start

**1. Clone this repo and install the skills into Claude**

```bash
git clone https://github.com/chartsai/member-system-cc-skills.git
```

Copy the skill folders into Claude Code's user commands directory:

```bash
mkdir -p ~/.claude/commands
cp -r member-system-cc-skills/springboot-* ~/.claude/commands/
```

> Skills are now available globally across all your projects.

**2. Launch Claude Code and set up your environment**

```bash
claude
```

Then run the environment setup skill — it detects your OS and installs any missing tools (Homebrew, Node.js, Java 21, Docker, gcloud):

```
/springboot-env-setup
```

**3. Create a new project folder**

```bash
mkdir my-new-app && cd my-new-app
```

**4. Launch Claude Code and run the project setup skill**

```bash
claude
```

```
/springboot-setup
```

Claude will ask a few questions and create `.spring-config.json`. Then follow the install order below, one skill at a time.

> **Lost?** Run `/springboot-menu` at any point to see your project status, what's installed, what's next, and get active guidance.

---

## Recommended Install Order

```
0.  springboot-env-setup        ← RUN FIRST — installs Homebrew, Node.js, Java 21, Docker, gcloud
1.  springboot-prototype-ui     ← optional: design your UI first (HTML mockups, no backend)
2.  springboot-setup            ← creates .spring-config.json
3.  springboot-ide-setup        ← optional: IntelliJ IDEA setup guide
4.  springboot-scaffold         ← generates the Spring Boot project
5.  springboot-db-setup         ← PostgreSQL + Flyway + Docker Compose
6.  springboot-app-config       ← DB-backed settings store
7.  springboot-gcp-setup        ← Google Cloud setup (for deployment)
8.  springboot-mail             ← SMTP email system
9.  springboot-auth-google      ← Google OAuth2 login
    springboot-auth-magic-link  ← or passwordless magic link (pick one or both)
10. springboot-membership       ← member entity, roles, admin list
11. springboot-membership-apply ← public signup + approval workflow
12. springboot-admin-portal     ← admin "preview as member" feature
13. springboot-file-upload      ← file uploads (local + GCS)
14. springboot-item-submit      ← member submission system
15. springboot-announcements    ← admin broadcast announcements
16. springboot-stats-summary    ← activity statistics
17. springboot-data-export      ← CSV / Excel / Google Sheets export
18. springboot-audit-log        ← audit trail
19. springboot-integration-tests← integration test strategy
20. springboot-deploy           ← Cloud Run deployment
--- optional, add any time after the foundation ---
21. springboot-i18n             ← multi-language UI
22. springboot-search           ← full-text search
23. springboot-notifications    ← in-app notification bell
24. springboot-content          ← blog / CMS with rich text editor
```

---

## All 27 Skills

### 🛠️ Environment

| Skill | Description |
|---|---|
| `springboot-env-setup` | **Run before anything else.** Detects your OS, checks what is already installed, and installs any missing tools — Homebrew, Node.js, Java 21, Docker Desktop, gcloud CLI. Skips tools that are already present. |

### 🗺️ Navigation

| Skill | Description |
|---|---|
| `springboot-menu` | **Start here when lost.** Shows project status (✅ installed / ⬜ ready / 🔒 blocked), recommends next step, and acts as an active mentor — diagnoses problems, maps goals to modules, guides next steps. Works even before setup. |

### 🏗️ Foundation

| Skill | Description |
|---|---|
| `springboot-setup` | Config wizard — ~13 questions → writes `.spring-config.json` |
| `springboot-scaffold` | Spring Boot project: Gradle (Kotlin DSL), Thymeleaf, Tailwind CSS, Spring Security |
| `springboot-db-setup` | PostgreSQL + Flyway migrations + Docker Compose for local dev |
| `springboot-app-config` | DB-backed key-value settings with admin UI |
| `springboot-mail` | Email system: Thymeleaf templates, rules engine, scheduled sending, MailHog for local dev |

### ☁️ GCP & Deployment

| Skill | Description |
|---|---|
| `springboot-gcp-setup` | gcloud CLI setup: project, APIs, Artifact Registry, Service Account, Cloud SQL. Includes a pre-flight checklist of which steps need browser action (billing, `gcloud init`) and direct Console links. |
| `springboot-deploy` | Dockerfile + Cloud Build CI + `deploy.sh` for Cloud Run — direct Console links for Cloud Run, Logs, Artifact Registry |

### 🔐 Authentication

| Skill | Description |
|---|---|
| `springboot-auth-google` | Google OAuth2 login with Spring Security. Includes step-by-step OAuth Consent Screen setup with **direct GCP Console deep links** — no hunting through menus. |
| `springboot-auth-magic-link` | Passwordless email magic link login — works with any email address, no Google account needed |

### 👥 Membership

| Skill | Description |
|---|---|
| `springboot-membership` | Member entity, roles (MEMBER / ADMIN / SUPER_ADMIN), admin member list |
| `springboot-membership-apply` | Public `/apply` form + admin approve/reject workflow with email notifications |
| `springboot-admin-portal` | Admin "preview as member" with session-based state switching |

### 🧩 Features

| Skill | Description |
|---|---|
| `springboot-file-upload` | File storage — local filesystem for dev, Google Cloud Storage for production |
| `springboot-item-submit` | Member submission system with admin review queue |
| `springboot-announcements` | Admin broadcast announcements shown on member dashboard |
| `springboot-stats-summary` | Period-based activity aggregation, thresholds, progress tracking |
| `springboot-data-export` | Export to CSV, Excel, or Google Sheets with configurable field mapping |
| `springboot-i18n` | Multi-language UI — Spring MessageSource, locale switcher, EN / 繁中 / 日 |
| `springboot-notifications` | In-app notification bell with unread badge, REST polling, mark-as-read |
| `springboot-search` | PostgreSQL full-text search (tsvector) — search bar, results page, CJK trigram support |
| `springboot-content` | CMS / blog — articles, categories, tags, Quill.js rich text editor, SEO Open Graph meta |

### 🔍 Quality & Observability

| Skill | Description |
|---|---|
| `springboot-audit-log` | Audit trail: who changed what, when — filterable admin view |
| `springboot-integration-tests` | Auto-generates integration tests based on your installed modules |

### 🎨 Optional

| Skill | Description |
|---|---|
| `springboot-prototype-ui` | Design-first: generates two self-contained HTML mockups (member portal + admin portal) — run before writing any backend code |
| `springboot-ide-setup` | IntelliJ IDEA setup walkthrough |

---

## How Skills Work

Each skill is a `SKILL.md` file with instructions that Claude follows. When you run a skill:

1. Claude reads `.spring-config.json` (created by `springboot-setup`)
2. Checks prerequisites — tells you exactly what's missing if something isn't installed yet
3. Asks a few clarifying questions
4. Generates all the code, SQL migrations, config files, and Thymeleaf templates
5. Writes unit tests
6. Commits to git

**Language-aware.** Set `"language": "繁體中文"` and every skill responds in Traditional Chinese — code comments, UI copy, and explanations. Traditional Chinese (繁體中文) and Simplified Chinese (简体中文) are treated as distinct writing systems and are never mixed.

**Beginner-friendly mode.** Set `"beginner_friendly": true` and skills explain what they're doing in plain language as they work.

**Prerequisite tracking.** `installed_modules` in `.spring-config.json` tracks what's been applied. Skills fail gracefully if a dependency is missing and tell you exactly what to install first.

---

## `.spring-config.json`

All 26 skills share this config file, created automatically by `springboot-setup`:

```json
{
  "app_name": "MyApp",
  "brand_name": "My Brand",
  "base_package": "com.example.myapp",
  "app_url": "https://myapp.com",
  "super_admin_email": "admin@example.com",
  "db_name": "myapp_db",
  "gcp_project_id": "my-gcp-project",
  "gcp_region": "asia-east1",
  "language": "繁體中文",
  "translate_terms": false,
  "test_mode": "build-and-test",
  "soft_delete": true,
  "beginner_friendly": false,
  "installed_modules": ["setup", "scaffold", "db"]
}
```

| Field | Description |
|---|---|
| `language` | All skill output language. `"繁體中文"` for Traditional Chinese, `"English"`, `"日本語"`, etc. |
| `test_mode` | `"build-and-test"` runs tests after generating, `"build-only"` generates only, `"token-save"` skips test code |
| `soft_delete` | All deletes set `deleted_at` timestamp instead of hard-deleting rows (default: `true`) |
| `beginner_friendly` | Skills explain technical terms inline as they work |
| `installed_modules` | Array of installed module keys — used for prerequisite checks across all skills |

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Spring Boot 3.x |
| Language | Kotlin |
| Build | Gradle (Kotlin DSL) |
| Templates | Thymeleaf |
| CSS | Tailwind CSS (CDN) |
| Database | PostgreSQL |
| Migrations | Flyway |
| Auth | Spring Security + OAuth2 / Magic Links |
| Email | Spring Mail + MailHog (local) |
| Storage | Local filesystem / Google Cloud Storage |
| Deployment | Docker + Google Cloud Run |
| Testing | JUnit 5 + Testcontainers |
| Rich Text | Quill.js (CDN) |

---

## Contributing

PRs welcome. To add a new skill:

1. Create `springboot-yourfeature/SKILL.md` following the existing pattern
2. Include: config reading, prerequisites check, language activation, beginner-friendly mode, unit tests, git commit step, `installed_modules` update
3. Update `README.md` module table
4. Open a PR

---

## License

MIT — use freely, build great things.

---
---

# 🍃 Claude Code 會員系統技能包

**27 個模組化 Claude Code 技能，逐步引導你建立專業的會員管理網站。**

> 不用再複製貼上樣板程式碼。執行技能、回答幾個問題，就能得到可運行的程式碼，並上傳至伺服器(Google Cloud)運行。

[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## ⚡ 懶人包

**1. 安裝 Claude Code CLI：**

```bash
npm install -g @anthropic-ai/claude-code
```

**2. Clone 並安裝技能：**

```bash
git clone https://github.com/chartsai/member-system-cc-skills.git
mkdir -p ~/.claude/commands && cp -r member-system-cc-skills/springboot-* ~/.claude/commands/
```

**3. 執行環境設定技能 — 自動安裝 Java、Docker 等所有開發工具：**

```bash
claude
```

```
/springboot-env-setup
```

> 還沒有 npm 或 Claude Code？→ [手動安裝說明](#環境需求)

---

## 這是什麼？

一套 **Claude AI 技能**，引導你一步一步建立完整的會員系統網站——每次安裝一個模組。每個技能都了解你的專案設定、檢查前置需求、生成商用級程式碼、撰寫單元測試，並提交到 git 版本控制。網站完成後，只需一個指令即可部署到 **[Google 雲端詞服器 - Google Cloud](https://cloud.google.com/run)**。

**不需要是專家。** 技能會邊做邊解釋它在做什麼。把它想成是一個**資深工程師坐在你旁邊**，一邊實作功能、一邊教你。

網站是基於[Spring Boot](https://spring.io/projects/spring-boot) 製作。Spring Boot是廣泛使用的 Java/Kotlin 網站框架，負責處理路由、資料庫存取、身分驗證、背景排程等底層工作，讓你專注在產品本身。它被大量用於新創公司與企業後端。

---

## 你將建立什麼

一個完整的會員制網站，包含：

- 🔐 **身分驗證** — Google OAuth2 登入或無密碼 Email 魔法連結
- 👥 **會員系統** — 角色、審核流程、管理後台
- 📧 **Email** — 樣板化 Email、排程發送、本機 MailHog 測試
- 📁 **檔案上傳** — 本機開發 + 正式環境 Google Cloud Storage
- 📣 **公告系統** — 管理員廣播給會員
- 📊 **統計與匯出** — 活動摘要、CSV/Excel/Google Sheets 匯出
- 🔍 **搜尋** — PostgreSQL 全文搜尋，不需要 Elasticsearch
- 🔔 **站內通知** — 通知鈴鐺與未讀計數
- 📝 **內容 / 部落格** — 文章、分類、標籤、Quill.js 富文字編輯器
- 🌐 **多語言** — 繁體中文 / English / 日本語 UI
- 📋 **稽核記錄** — 誰在何時改了什麼
- ☁️ **GCP 部署** — Dockerfile + Cloud Run + Secret Manager

---

## 環境需求

### macOS

**第一步 — 安裝 [Homebrew](https://brew.sh)**（套件管理器）：

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

**第二步 — 安裝 [Node.js](https://nodejs.org)**（為了執行`npm`指令）：

```bash
brew install node
```

**第三步 — 安裝其餘工具：**

| 工具 | 用途 | 指令 |
|---|---|---|
| [Java 21+](https://adoptium.net) | Spring Boot 執行環境 | `brew install --cask temurin@21` |
| [Docker Desktop](https://www.docker.com/products/docker-desktop) | 開發期間的測試用資料庫 | `brew install --cask docker` |
| [IntelliJ IDEA](https://www.jetbrains.com/idea/) | IDE（工程師選用）  | `brew install --cask intellij-idea-ce` |
| [gcloud CLI](https://cloud.google.com/sdk/docs/install) | 上傳至Google雲端並啟用會員系統 | `brew install --cask google-cloud-sdk` |

---

### Windows

**`winget`** 已內建於 Windows 10（1809 以上）和 Windows 11。在開始還單中搜尋 **PowerShell** 並執行：

**第一步 — 安裝 [Node.js](https://nodejs.org)**（為了執行`npm`指令）：

```
winget install OpenJS.NodeJS
```

> 安裝後請 **重新開啟PowerShell** ，確保 `npm` 可以執行。

**第二步 — 安裝其餘工具：**

| 工具 | 用途 | 指令 |
|---|---|---|
| [Java 21+](https://adoptium.net) | Spring Boot 執行環境 | `winget install EclipseAdoptium.Temurin.21.JDK` |
| [Docker Desktop](https://www.docker.com/products/docker-desktop) | 開發期間的測試用資料庫 | `winget install Docker.DockerDesktop` |
| [IntelliJ IDEA](https://www.jetbrains.com/idea/) | IDE（工程師選用） | `winget install JetBrains.IntelliJIDEA.Community` |
| [gcloud CLI](https://cloud.google.com/sdk/docs/install) | 上傳至Google雲端並啟用會員系統 | `winget install Google.CloudSDK` |

---

## 快速開始

**1. Clone 並安裝技能**

```bash
git clone https://github.com/chartsai/member-system-cc-skills.git
```

將技能資料夾複製到 Claude Code 的使用者指令目錄：

```bash
mkdir -p ~/.claude/commands
cp -r member-system-cc-skills/springboot-* ~/.claude/commands/
```

> 技能安裝後可在本機所有專案中使用。

**2. 建立新的專案資料夾**

```bash
mkdir my-new-app && cd my-new-app
```

**3. 在專案資料夾中執行Claude Code**

```
claude
```

**4. 在 Claude Code 中執行第一個技能**

```
/springboot-setup
```

Claude 會問數個問題並建立 `.spring-config.json`，接著按照建議順序逐步安裝模組。

> **迷失方向？** 隨時說 `執行 springboot-menu`，查看專案狀態、已安裝模組、下一步建議，並獲得主動引導。

---

## 建議安裝順序

```
0.  springboot-env-setup        ← 最先執行 — 安裝 Homebrew、Node.js、Java 21、Docker、gcloud
1.  springboot-prototype-ui     ← 選用：先設計 UI（HTML 原型，不需後端）
2.  springboot-setup            ← 建立 .spring-config.json
3.  springboot-ide-setup        ← 選用：IntelliJ IDEA 設定指南
4.  springboot-scaffold         ← 產生 Spring Boot 專案骨架
5.  springboot-db-setup         ← PostgreSQL + Flyway + Docker Compose
6.  springboot-app-config       ← 資料庫設定儲存系統
7.  springboot-gcp-setup        ← Google Cloud 設定（部署時需要）
8.  springboot-mail             ← SMTP Email 系統
9.  springboot-auth-google      ← Google OAuth2 登入
    springboot-auth-magic-link  ← 或無密碼魔法連結登入（擇一或兩者都裝）
10. springboot-membership       ← 會員實體、角色、管理員清單
11. springboot-membership-apply ← 公開申請表單 + 審核流程
12. springboot-admin-portal     ← 管理員「以會員身份預覽」功能
13. springboot-file-upload      ← 檔案上傳
14. springboot-item-submit      ← 會員提交系統
15. springboot-announcements    ← 公告廣播
16. springboot-stats-summary    ← 活動統計
17. springboot-data-export      ← CSV / Excel / Google Sheets 匯出
18. springboot-audit-log        ← 稽核記錄
19. springboot-integration-tests← 整合測試策略
20. springboot-deploy           ← Cloud Run 部署
--- 選用，基礎完成後任何時候都可安裝 ---
21. springboot-i18n             ← 多語言 UI
22. springboot-search           ← 全文搜尋
23. springboot-notifications    ← 站內通知鈴鐺
24. springboot-content          ← 部落格 / CMS
```

---

## 全部 27 個技能

### 🛠️ 環境設定

| 技能 | 功能說明 |
|---|---|
| `springboot-env-setup` | **最先執行。** 偵測作業系統，檢查已安裝的工具，並自動安裝缺少的項目 — Homebrew、Node.js、Java 21、Docker Desktop、gcloud CLI。已安裝的工具會自動跳過。 |

### 🗺️ 選單與導引

| 技能 | 功能說明 |
|---|---|
| `springboot-menu` | **迷失時從這裡開始。** 顯示專案狀態（✅ 已安裝 / ⬜ 可安裝 / 🔒 尚未解鎖）、推薦下一步，並作為主動導師 — 診斷問題、對應目標到模組、逐步引導。即使在 setup 之前也能使用。 |

### 🏗️ 基礎

| 技能 | 功能說明 |
|---|---|
| `springboot-setup` | 設定精靈 — 約 13 個問題 → 建立 `.spring-config.json` |
| `springboot-scaffold` | Spring Boot 專案骨架：Gradle（Kotlin DSL）、Thymeleaf、Tailwind CSS、Spring Security |
| `springboot-db-setup` | PostgreSQL + Flyway 資料庫版本控制 + Docker Compose 本機開發 |
| `springboot-app-config` | 資料庫 key-value 設定系統，附管理員介面 |
| `springboot-mail` | Email 系統：Thymeleaf 樣板、規則引擎、排程發送、本機 MailHog |

### ☁️ GCP 與部署

| 技能 | 功能說明 |
|---|---|
| `springboot-gcp-setup` | gcloud CLI 設定：專案、API 啟用、Artifact Registry、Service Account、Cloud SQL。包含前置檢查清單，說明哪些步驟需要瀏覽器操作（帳單、`gcloud init`）並附上 Console 直連連結。 |
| `springboot-deploy` | Dockerfile + Cloud Build CI + `deploy.sh` 部署到 Cloud Run — 附 Cloud Run、Log、Artifact Registry Console 直連連結 |

### 🔐 身分驗證

| 技能 | 功能說明 |
|---|---|
| `springboot-auth-google` | Google OAuth2 登入。包含 OAuth 同意畫面逐步設定指南，附 **GCP Console 直連連結** — 不用在選單中找。 |
| `springboot-auth-magic-link` | 無密碼 Email 魔法連結登入 — 適用任何 Email，不需要 Google 帳戶 |

### 👥 會員系統

| 技能 | 功能說明 |
|---|---|
| `springboot-membership` | 會員實體、角色（MEMBER / ADMIN / SUPER_ADMIN）、管理員清單 |
| `springboot-membership-apply` | 公開 `/apply` 申請表單 + 管理員審核/拒絕流程（含 Email 通知） |
| `springboot-admin-portal` | 管理員「以會員身份預覽」功能（Session 狀態切換） |

### 🧩 功能模組

| 技能 | 功能說明 |
|---|---|
| `springboot-file-upload` | 檔案上傳 — 本機開發用本機儲存、正式環境用 GCS |
| `springboot-item-submit` | 會員提交系統，管理員審核佇列 |
| `springboot-announcements` | 管理員廣播公告，顯示在會員儀表板 |
| `springboot-stats-summary` | 依時間週期統計活動，門檻值設定，進度追蹤 |
| `springboot-data-export` | CSV / Excel / Google Sheets 匯出，欄位映射設定 |
| `springboot-i18n` | 多語言 UI — Spring MessageSource、語言切換器、繁中/英/日訊息檔 |
| `springboot-notifications` | 站內通知鈴鐺，含未讀徽章、REST 輪詢、標記已讀 |
| `springboot-search` | PostgreSQL 全文搜尋（tsvector）— 搜尋欄、結果頁、CJK 三元組支援 |
| `springboot-content` | CMS / 部落格 — 文章、分類、標籤、Quill.js 富文字編輯器、SEO Open Graph |

### 🔍 品質與可觀測性

| 技能 | 功能說明 |
|---|---|
| `springboot-audit-log` | 稽核記錄：誰在何時改了什麼 — 可過濾的管理員介面 |
| `springboot-integration-tests` | 根據已安裝模組自動產生整合測試策略與測試程式碼 |

### 🎨 選用

| 技能 | 功能說明 |
|---|---|
| `springboot-prototype-ui` | 設計優先：產生兩個獨立 HTML 原型（會員入口 + 管理後台）— 建議在寫後端之前先執行 |
| `springboot-ide-setup` | IntelliJ IDEA 設定完整指南 |

---

## 技能運作方式

每個技能都是一個 `SKILL.md` 檔案，包含 Claude 的執行指示。執行技能時：

1. Claude 讀取 `.spring-config.json`（由 `springboot-setup` 建立）
2. 檢查前置需求 — 如果有缺少的東西，會精確告訴你缺什麼以及如何補齊
3. 詢問幾個釐清問題
4. 生成所有程式碼、SQL migration、設定檔和 Thymeleaf 樣板
5. 撰寫單元測試
6. 提交到 git

**多語言支援。** 設定 `"language": "繁體中文"`，每個技能都用繁體中文回應——包含程式碼註解、生成的 HTML UI 文字和說明。繁體中文（zh-TW）與簡體中文（zh-CN）是不同的書寫系統，技能不會混用。

**Beginner-friendly 模式。** 設定 `"beginner_friendly": true`，技能在執行時用白話文解釋每個技術概念。

**前置需求追蹤。** `.spring-config.json` 中的 `installed_modules` 記錄已安裝的模組。技能在缺少相依性時會優雅地報錯，並告訴你需要先安裝什麼。

---

## `.spring-config.json` 設定格式

全部 26 個技能共用這個設定檔，由 `springboot-setup` 自動建立：

```json
{
  "app_name": "MyApp",
  "brand_name": "My Brand",
  "base_package": "com.example.myapp",
  "app_url": "https://myapp.com",
  "super_admin_email": "admin@example.com",
  "db_name": "myapp_db",
  "gcp_project_id": "my-gcp-project",
  "gcp_region": "asia-east1",
  "language": "繁體中文",
  "translate_terms": false,
  "test_mode": "build-and-test",
  "soft_delete": true,
  "beginner_friendly": false,
  "installed_modules": ["setup", "scaffold", "db"]
}
```

| 欄位 | 說明 |
|---|---|
| `language` | 所有技能的輸出語言。`"繁體中文"`、`"English"`、`"日本語"` 等 |
| `test_mode` | `"build-and-test"` 執行測試、`"build-only"` 只產生不執行、`"token-save"` 略過測試生成 |
| `soft_delete` | 所有刪除操作設定 `deleted_at` 時間戳記（不硬刪除）（預設：`true`） |
| `beginner_friendly` | 技能在執行時內嵌解釋技術術語 |
| `installed_modules` | 已安裝模組的鍵值陣列 — 用於所有技能的前置需求檢查 |

---

## 技術棧

| 層次 | 技術 |
|---|---|
| 框架 | Spring Boot 3.x |
| 語言 | Kotlin |
| 建構工具 | Gradle（Kotlin DSL） |
| 樣板引擎 | Thymeleaf |
| CSS | Tailwind CSS（CDN） |
| 資料庫 | PostgreSQL |
| 資料庫版本控制 | Flyway |
| 身分驗證 | Spring Security + OAuth2 / Magic Links |
| Email | Spring Mail + MailHog（本機） |
| 儲存 | 本機檔案系統 / Google Cloud Storage |
| 部署 | Docker + Google Cloud Run |
| 測試 | JUnit 5 + Testcontainers |
| 富文字編輯器 | Quill.js（CDN） |

---

## 貢獻

歡迎 Pull Request！新增技能：

1. 建立 `springboot-yourfeature/SKILL.md`，遵循現有模式
2. 必須包含：讀取設定、檢查前置需求、語言啟用、beginner-friendly 模式、單元測試、git commit 步驟、更新 `installed_modules`
3. 更新 `README.md` 模組表格
4. 開 PR

---

## 授權

MIT — 自由使用，建立美好的東西。
