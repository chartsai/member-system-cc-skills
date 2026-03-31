---
name: springboot-welcome
description: >
  TRIGGER this skill when the user says "start", "get started", "welcome", "run springboot-welcome",
  or when they appear to be using this skill collection for the first time.
  This is the VERY FIRST skill to run — before springboot-setup or anything else.
  It selects the preferred language, sets up the development environment, and guides
  the user to run springboot-setup next.
---

# springboot-welcome

The **first skill to ever run** for a new project. It sets the preferred language, installs
any missing development tools, and hands off to `springboot-setup`.

---

## What this skill does

1. Asks the user's preferred language — shown in all supported languages so anyone can answer
2. Switches to that language for all further interaction
3. Checks and installs missing development tools (Homebrew/winget, Node.js, Java 21, Docker, gcloud)
4. Writes a minimal `.spring-config.json` recording the language and that welcome is done
5. Tells the user to run `/springboot-setup` next

---

## Step 1 — Ask for preferred language

Show this prompt exactly as written (all languages at once — do NOT translate or alter it):

```
👋 Welcome! / 歡迎！/ ようこそ！

What language would you like to use?
您希望使用哪種語言？
使用する言語を選んでください。

  1. English
  2. 繁體中文（Traditional Chinese）
  3. 日本語
  4. Other — please type your language / 其他，請輸入語言名稱 / その他（入力してください）
```

Wait for the user's reply. **From this point on, conduct all interaction in the chosen language.**

> **Traditional vs Simplified Chinese**: If the user selects 繁體中文, always use Traditional
> Chinese characters — NEVER Simplified (简体中文). They are distinct writing systems.
> Key Traditional terms: 設定, 資料, 確認, 語言, 會員, 管理員, 模組, 安裝.

---

## Step 2 — Welcome message

In the chosen language, show a brief welcome:

```
Great! I'll continue in <language>.

Welcome to the Spring Boot Membership System skill collection.
These skills will guide you — step by step — through building a full-featured
membership web app and deploying it to Google Cloud.

Let me first check that your development environment is ready.
```

---

## Step 3 — Detect OS

Run:

```bash
uname -s
```

- `Darwin` → macOS path
- Anything else or command unavailable → Windows path

---

## Step 4 — Check installed tools

Run the following and record results:

```bash
brew --version 2>/dev/null || echo "not found"
node --version 2>/dev/null || echo "not found"
java -version 2>&1 | head -1 || echo "not found"
docker --version 2>/dev/null || echo "not found"
gcloud --version 2>/dev/null | head -1 || echo "not found"
```

Show a checklist in the chosen language, for example:

```
Checking your environment...

  ✅ Homebrew     3.x.x
  ❌ Node.js      not found
  ✅ Java         21.0.x
  ✅ Docker       24.x.x
  ❌ gcloud       not found
```

If all tools are already installed, skip to Step 6.

---

## Step 5 — Install missing tools

Work through each missing tool in order. Verify each one after installing before moving on.
If an install fails, show the error and ask whether to retry or skip.

### macOS

#### Homebrew (install first if missing — required by all other brew installs)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

Verify: `brew --version`

#### Node.js

```bash
brew install node
```

Verify: `node --version && npm --version`

#### Java 21

```bash
brew install --cask temurin@21
```

Verify: `java -version`

> If the version still shows a different JDK, advise the user to set JAVA_HOME:
> ```bash
> export JAVA_HOME=$(/usr/libexec/java_home -v 21)
> ```
> Add to `~/.zshrc` to make it permanent.

#### Docker Desktop

```bash
brew install --cask docker
```

After installing, tell the user to **open Docker Desktop from Applications and wait for it to finish starting**, then verify: `docker --version`

#### gcloud CLI

Ask the user: "Do you plan to deploy to Google Cloud? (yes / no)"

If yes:
```bash
brew install --cask google-cloud-sdk
```
Verify: `gcloud --version`

If no: skip.

---

### Windows

Claude Code cannot run `winget` directly. Present the missing tools as a clear checklist of
commands for the user to run themselves in **PowerShell**. After each one, ask them to confirm
it worked (paste the version output) before continuing.

#### Node.js

```
winget install OpenJS.NodeJS
```
Restart PowerShell after this step so `npm` is on your PATH.

#### Java 21

```
winget install EclipseAdoptium.Temurin.21.JDK
```

#### Docker Desktop

```
winget install Docker.DockerDesktop
```
Open Docker Desktop and wait for it to finish starting.

#### gcloud CLI

Ask the user: "Do you plan to deploy to Google Cloud? (yes / no)"

If yes:
```
winget install Google.CloudSDK
```

---

## Step 6 — Write minimal `.spring-config.json`

Write this file to the **current working directory**:

```json
{
  "language": "<chosen_language>",
  "installed_modules": ["welcome"]
}
```

This records the language preference so `springboot-setup` can skip asking for it, and marks
that the welcome skill has been completed so `springboot-menu` can track it.

---

## Step 7 — Hand off to springboot-setup

Show this message in the chosen language:

```
✅ Your environment is ready!

Next step: run /springboot-setup

This will ask you a few questions about your project (app name, database, GCP project, etc.)
and create the full .spring-config.json that all other skills depend on.
```

> Do NOT ask the user to create a project folder or launch `claude` — they are already
> inside the project folder and running Claude Code by the time this skill executes.
