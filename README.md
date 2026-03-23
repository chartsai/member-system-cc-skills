# Spring Boot Skills

A modular Claude skill system for scaffolding production-ready Spring Boot applications.
Each skill is a self-contained directory with a `SKILL.md` file that Claude Code reads and executes.

---

## How it works

1. Copy skill directories to your Claude skills folder
2. Claude detects which skill to run based on your message
3. Skills read `.spring-config.json` from your project root for all configuration
4. Each skill marks itself in `installed_modules` when complete
5. Skills check prerequisites before running — they fail fast with clear error messages

---

## Quick start

```bash
# 1. Copy skills to Claude's skills folder
cp -r ~/code/springboot-skills/* ~/.claude/skills/   # adjust path as needed

# 2. Create your project directory
mkdir my-new-project && cd my-new-project

# 3. Open Claude Code
claude

# 4. Start with setup
> Run springboot-setup
```

---

## `.spring-config.json` format

Written by `springboot-setup`, read by every other skill:

```json
{
  "app_name": "MyApp",
  "brand_name": "My Brand",
  "base_package": "com.example.myapp",
  "app_url": "https://myapp.com",
  "super_admin_email": "admin@example.com",
  "db_name": "myapp_db",
  "gcp_project_id": "my-gcp-project",
  "gcp_region": "us-central1",
  "language": "en",
  "translate_terms": false,
  "test_mode": "build-and-test",
  "soft_delete": true,
  "beginner_friendly": false,
  "installed_modules": []
}
```

### Config fields

| Field | Type | Description |
|---|---|---|
| `app_name` | string | Application name (used in titles, branding) |
| `brand_name` | string | Company/brand name (used in footers, emails) |
| `base_package` | string | Java package root, e.g. `com.example.myapp` |
| `app_url` | string | Production URL (no trailing slash) |
| `super_admin_email` | string | First admin account email |
| `db_name` | string | PostgreSQL database name |
| `gcp_project_id` | string | GCP project ID, or `"TBD"` |
| `gcp_region` | string | GCP region, e.g. `"us-central1"` |
| `language` | string | Claude response language (`"English"`, `"Traditional Chinese"`, etc.) |
| `translate_terms` | bool | Translate technical terms to response language? |
| `test_mode` | string | `"token-save"` / `"build-only"` / `"build-and-test"` |
| `soft_delete` | bool | Use `deleted_at` pattern (default: `true`) |
| `beginner_friendly` | bool | Explain technical terms inline (default: `false`) |
| `installed_modules` | array | Tracked by skills — do not edit manually |

### `test_mode` options

| Value | Behavior |
|---|---|
| `"token-save"` | Skip all test execution and test file creation |
| `"build-only"` | Run `./gradlew build -x test` — verify compilation only |
| `"build-and-test"` | Run `./gradlew build` — full build including all tests |

---

## All skills (20 total)

| Skill | What it does |
|---|---|
| `springboot-setup` | **Start here.** Collects config, writes `.spring-config.json` |
| `springboot-ide-setup` | Optional: Configure IntelliJ IDEA for the project |
| `springboot-scaffold` | Create Spring Boot project (Gradle, Thymeleaf, Tailwind, Security) |
| `springboot-db-setup` | Add PostgreSQL, Flyway, Docker Compose dev DB |
| `springboot-app-config` | DB-backed key-value settings with admin UI |
| `springboot-gcp-setup` | GCP project setup (APIs, Artifact Registry, Service Account) |
| `springboot-auth-google` | Google OAuth2 login with Spring Security |
| `springboot-auth-magic-link` | Passwordless email magic link login |
| `springboot-membership` | Member profiles, roles (MEMBER/ADMIN/SUPER_ADMIN), admin list |
| `springboot-membership-apply` | Public application form + admin approve/reject workflow |
| `springboot-admin-portal` | Admin "preview as member" with session-based state |
| `springboot-mail` | Email system: DB templates, rules engine, scheduled sending |
| `springboot-file-upload` | File upload with local (dev) and GCS (prod) storage |
| `springboot-item-submit` | Member submission system with admin review queue |
| `springboot-announcements` | Admin broadcast announcements on member dashboard |
| `springboot-stats-summary` | Period-based activity aggregation, progress tracking, thresholds |
| `springboot-data-export` | CSV/Excel/Google Sheets export with field mapping |
| `springboot-audit-log` | Audit trail: who changed what, when — filterable admin view |
| `springboot-integration-tests` | Auto-generates integration tests based on installed modules |
| `springboot-deploy` | Dockerfile, Cloud Build CI, `deploy.sh` for Cloud Run |

---

## Dependency graph

```
FOUNDATION (install in this order):
════════════════════════════════════════════════════════════════
springboot-setup          (no deps — always first)
  └── springboot-ide-setup    (optional, no deps)
  └── springboot-scaffold     (needs: setup)
        └── springboot-db-setup         (needs: setup, scaffold)
              └── springboot-app-config (needs: setup, scaffold, db)
              └── springboot-audit-log  (needs: setup, scaffold, db)
              └── springboot-mail       (needs: setup, scaffold, db)

GCP (can be done in parallel with db-setup):
════════════════════════════════════════════════════════════════
springboot-gcp-setup      (needs: setup only)

AUTH (pick at least one):
════════════════════════════════════════════════════════════════
springboot-auth-google      (needs: setup, scaffold, db, gcp ← HARD DEP)
springboot-auth-magic-link  (needs: setup, scaffold, db, mail ← HARD DEP)

MEMBERSHIP (needs at least one auth module):
════════════════════════════════════════════════════════════════
springboot-membership        (needs: setup, scaffold, db, auth-google OR auth-magic-link)
springboot-membership-apply  (needs: setup, scaffold, db, mail, membership)
springboot-admin-portal      (needs: setup, scaffold, db, membership)

FEATURES (most need membership):
════════════════════════════════════════════════════════════════
springboot-file-upload       (needs: setup, scaffold, db)
                             (soft: gcp → enables GCS storage)
springboot-item-submit       (needs: setup, scaffold, db, membership)
springboot-announcements     (needs: setup, scaffold, db, membership)
                             (soft: mail → email notifications)
springboot-stats-summary     (needs: setup, scaffold, db, membership)
                             (soft: item-submit, app-config, data-export)
springboot-data-export       (needs: setup, scaffold, db, membership)
                             (soft: gcp → Google Sheets export)

DEVOPS:
════════════════════════════════════════════════════════════════
springboot-integration-tests (needs: setup, scaffold, db + 2+ feature modules)
springboot-deploy            (needs: setup, scaffold, db, gcp)
```

**Hard prerequisite** = skill will fail with an error if missing
**Soft prerequisite** = skill degrades gracefully (feature disabled, with a note)

---

## Recommended install order for a full app

```
1.  springboot-setup
2.  springboot-ide-setup        (optional — if new to the toolchain)
3.  springboot-scaffold
4.  springboot-db-setup
5.  springboot-app-config       (install early — other modules use it)
6.  springboot-gcp-setup        (can do in parallel with steps 4-5)
7.  springboot-mail             (install before auth-magic-link and membership-apply)
8.  springboot-auth-google      (or auth-magic-link, or both)
9.  springboot-membership
10. springboot-membership-apply
11. springboot-admin-portal
12. springboot-file-upload
13. springboot-item-submit
14. springboot-announcements
15. springboot-stats-summary
16. springboot-data-export
17. springboot-audit-log
18. springboot-integration-tests
19. springboot-deploy
```

---

## Skill directory structure

```
skill-name/
├── SKILL.md          ← required — YAML frontmatter + instructions
└── references/       ← optional reference files (SQL, config templates, etc.)
```

`SKILL.md` frontmatter format:

```yaml
---
name: skill-name
description: >
  When to trigger and what it does.
  Be specific and "pushy" about triggering — list exact phrases.
---
```

---

## Reference material

These skills were built from real production patterns collected in `~/code/springboot-prompts/`.
Tech stack: Java 21, Spring Boot 3.x, Thymeleaf SSR, Spring Security, JPA + PostgreSQL,
Flyway, Gradle (Kotlin DSL), Google Cloud Run.

---

---

# 繁體中文說明

## Spring Boot Skills 模組化技能系統

這是一套供 Claude Code 使用的模組化 Spring Boot 專案建置技能系統。
每個技能是一個獨立的資料夾，包含 `SKILL.md` 文件，Claude 會讀取並執行其中的指令。

---

## 運作方式

1. 將技能資料夾複製到 Claude 的 skills 資料夾
2. Claude 根據你的訊息自動判斷要執行哪個技能
3. 每個技能從專案根目錄的 `.spring-config.json` 讀取設定
4. 技能完成後，會將自己記錄在 `installed_modules` 陣列中
5. 技能會在執行前檢查前置條件，若不滿足會立即給出明確的錯誤訊息

---

## 快速開始

```bash
# 1. 將技能複製到 Claude 的 skills 資料夾
cp -r ~/code/springboot-skills/* ~/.claude/skills/

# 2. 建立你的專案資料夾
mkdir my-new-project && cd my-new-project

# 3. 開啟 Claude Code
claude

# 4. 從 setup 開始
> 執行 springboot-setup
```

---

## `.spring-config.json` 設定格式

由 `springboot-setup` 寫入，其他所有技能都會讀取：

```json
{
  "app_name": "MyApp",
  "brand_name": "我的品牌",
  "base_package": "com.example.myapp",
  "app_url": "https://myapp.com",
  "super_admin_email": "admin@example.com",
  "db_name": "myapp_db",
  "gcp_project_id": "my-gcp-project",
  "gcp_region": "asia-east1",
  "language": "Traditional Chinese",
  "translate_terms": false,
  "test_mode": "build-and-test",
  "soft_delete": true,
  "beginner_friendly": false,
  "installed_modules": []
}
```

### 設定欄位說明

| 欄位 | 類型 | 說明 |
|---|---|---|
| `app_name` | string | 應用程式名稱（用於頁面標題、品牌） |
| `brand_name` | string | 公司/品牌名稱（用於頁尾、Email） |
| `base_package` | string | Java 套件根目錄，例如 `com.example.myapp` |
| `app_url` | string | 正式環境 URL（結尾不含斜線） |
| `super_admin_email` | string | 第一個管理員帳號的 Email |
| `db_name` | string | PostgreSQL 資料庫名稱 |
| `gcp_project_id` | string | GCP 專案 ID，或填 `"TBD"` |
| `gcp_region` | string | GCP 區域，例如 `"asia-east1"` |
| `language` | string | Claude 回覆語言（`"English"`、`"Traditional Chinese"` 等） |
| `translate_terms` | bool | 是否翻譯技術術語（Controller、Service 等）？ |
| `test_mode` | string | `"token-save"` / `"build-only"` / `"build-and-test"` |
| `soft_delete` | bool | 使用 `deleted_at` 軟刪除模式（預設：`true`） |
| `beginner_friendly` | bool | 是否在行文中說明技術術語（預設：`false`） |
| `installed_modules` | array | 由技能自動管理，請勿手動修改 |

### `test_mode` 選項

| 值 | 行為 |
|---|---|
| `"token-save"` | 跳過所有測試執行與測試檔案建立（最節省 token） |
| `"build-only"` | 執行 `./gradlew build -x test`，只驗證是否能編譯成功 |
| `"build-and-test"` | 執行 `./gradlew build`，完整建置包含所有測試 |

### `beginner_friendly` 說明

設為 `true` 時，每個技能會在行文中主動說明技術術語，例如：
> "我們正在加入 Flyway migration — 這是一個有版本號的 SQL 檔案，每次應用程式啟動時會自動執行，確保資料庫結構與程式碼保持同步。"

適合剛接觸 Spring Boot / Java 的開發者。

### `soft_delete` 說明

設為 `true`（預設）時，所有刪除操作會在記錄上設定 `deleted_at` 時間戳記，而非真正從資料庫刪除。
透過 Hibernate 的 `@SQLRestriction("deleted_at IS NULL")` 自動過濾已刪除的資料。
管理員介面會提供「復原」功能，可還原被軟刪除的資料。

---

## 全部模組（共 20 個）

| 技能 | 功能說明 |
|---|---|
| `springboot-setup` | **從這裡開始。** 收集設定、寫入 `.spring-config.json` |
| `springboot-ide-setup` | 選用：設定 IntelliJ IDEA 開發環境 |
| `springboot-scaffold` | 建立 Spring Boot 專案（Gradle、Thymeleaf、Tailwind、Security） |
| `springboot-db-setup` | 加入 PostgreSQL、Flyway、Docker Compose 本機資料庫 |
| `springboot-app-config` | 資料庫儲存的 key-value 設定系統，附管理員介面 |
| `springboot-gcp-setup` | GCP 專案設定（API 啟用、Artifact Registry、Service Account） |
| `springboot-auth-google` | Google OAuth2 登入（Spring Security） |
| `springboot-auth-magic-link` | 無密碼 Email 魔法連結登入 |
| `springboot-membership` | 會員資料、角色（MEMBER/ADMIN/SUPER_ADMIN）、管理員清單 |
| `springboot-membership-apply` | 公開申請表單 + 管理員審核流程 |
| `springboot-admin-portal` | 管理員「以會員身份預覽」功能（Session 狀態管理） |
| `springboot-mail` | Email 系統：資料庫樣板、規則引擎、排程發送 |
| `springboot-file-upload` | 檔案上傳（本機開發 / GCS 正式環境雙後端） |
| `springboot-item-submit` | 會員提交系統，管理員審核佇列 |
| `springboot-announcements` | 管理員廣播公告，顯示在會員儀表板 |
| `springboot-stats-summary` | 依時間週期統計活動，進度追蹤，可設定門檻值 |
| `springboot-data-export` | CSV / Excel / Google Sheets 匯出，欄位映射設定 |
| `springboot-audit-log` | 操作稽核記錄：誰改了什麼、何時改的 |
| `springboot-integration-tests` | 根據已安裝模組自動產生整合測試 |
| `springboot-deploy` | Dockerfile、Cloud Build CI、`deploy.sh` 部署到 Cloud Run |

---

## 相依關係圖

```
基礎模組（按此順序安裝）：
════════════════════════════════════════════════════════════════
springboot-setup          （無相依 — 永遠最先）
  └── springboot-ide-setup    （選用，無相依）
  └── springboot-scaffold     （需要：setup）
        └── springboot-db-setup         （需要：setup, scaffold）
              └── springboot-app-config （需要：setup, scaffold, db）
              └── springboot-audit-log  （需要：setup, scaffold, db）
              └── springboot-mail       （需要：setup, scaffold, db）

GCP（可與 db-setup 並行進行）：
════════════════════════════════════════════════════════════════
springboot-gcp-setup      （只需要：setup）

驗證模組（至少選一）：
════════════════════════════════════════════════════════════════
springboot-auth-google      （需要：setup, scaffold, db, gcp ← 強制相依）
springboot-auth-magic-link  （需要：setup, scaffold, db, mail ← 強制相依）

會員系統（需要至少一個驗證模組）：
════════════════════════════════════════════════════════════════
springboot-membership        （需要：setup, scaffold, db, auth-google 或 auth-magic-link）
springboot-membership-apply  （需要：setup, scaffold, db, mail, membership）
springboot-admin-portal      （需要：setup, scaffold, db, membership）

功能模組（大多需要 membership）：
════════════════════════════════════════════════════════════════
springboot-file-upload       （需要：setup, scaffold, db）
                             （軟性：gcp → 啟用 GCS 儲存）
springboot-item-submit       （需要：setup, scaffold, db, membership）
springboot-announcements     （需要：setup, scaffold, db, membership）
                             （軟性：mail → 啟用 Email 通知）
springboot-stats-summary     （需要：setup, scaffold, db, membership）
                             （軟性：item-submit, app-config, data-export）
springboot-data-export       （需要：setup, scaffold, db, membership）
                             （軟性：gcp → 啟用 Google Sheets 匯出）

DevOps：
════════════════════════════════════════════════════════════════
springboot-integration-tests （需要：setup, scaffold, db + 2 個以上功能模組）
springboot-deploy            （需要：setup, scaffold, db, gcp）
```

**強制相依（Hard）** = 缺少時技能會停止並報錯
**軟性相依（Soft）** = 缺少時功能降級，並顯示說明訊息

---

## 建議安裝順序（完整應用程式）

```
1.  springboot-setup
2.  springboot-ide-setup        （選用 — 適合剛接觸工具鏈的開發者）
3.  springboot-scaffold
4.  springboot-db-setup
5.  springboot-app-config       （建議早裝 — 其他模組會使用它）
6.  springboot-gcp-setup        （可與步驟 4-5 並行）
7.  springboot-mail             （在 auth-magic-link 和 membership-apply 之前安裝）
8.  springboot-auth-google      （或 auth-magic-link，或兩者都裝）
9.  springboot-membership
10. springboot-membership-apply
11. springboot-admin-portal
12. springboot-file-upload
13. springboot-item-submit
14. springboot-announcements
15. springboot-stats-summary
16. springboot-data-export
17. springboot-audit-log
18. springboot-integration-tests
19. springboot-deploy
```

---

## 技能目錄結構

```
skill-name/
├── SKILL.md          ← 必要 — YAML frontmatter + 指令說明
└── references/       ← 選用參考檔案（SQL、設定範本等）
```

`SKILL.md` frontmatter 格式：

```yaml
---
name: skill-name
description: >
  說明觸發時機與功能。
  列出使用者可能說的具體語句，讓 Claude 能準確識別。
---
```

---

## 參考資料來源

這些技能的實作模式來自 `~/code/springboot-prompts/` 中整理的真實生產專案模式。

技術堆疊：Java 21、Spring Boot 3.x、Thymeleaf SSR、Spring Security、
JPA + PostgreSQL、Flyway、Gradle（Kotlin DSL）、Google Cloud Run。
