---
name: springboot-ide-setup
description: >
  TRIGGER this skill when the user says "set up IntelliJ", "configure my IDE", "set up the development environment",
  "run springboot-ide-setup", "help me set up IntelliJ IDEA for this project", "configure IntelliJ for Spring Boot",
  "set up my editor", or anything about configuring an IDE to work with this Spring Boot project.
  OPTIONAL skill — for developers new to the Java/Spring toolchain. No code changes required.
  Can be run at any time, no module dependencies.
---

# springboot-ide-setup

**Optional guided walkthrough** for setting up IntelliJ IDEA for this Spring Boot project.
Recommended for developers who are new to the Java/Spring toolchain.

No code is written by this skill — it is purely instructional.

---

## Note to Claude

This skill does NOT write or modify any project files. Respond in the user's preferred `language`
(from `.spring-config.json`). Read the config to personalize instructions with the actual app name,
DB name, and package names.

---

## Step 0 — Read config

Read `.spring-config.json` if it exists. Extract:

```
app_name     → used to personalize instructions
db_name      → for DB connection config
base_package → for package structure explanation
language     → respond in this language
```

If `.spring-config.json` doesn't exist, use generic placeholders.

---

## Step 1 — Check IntelliJ IDEA edition

Ask the user:
```
Which edition of IntelliJ IDEA do you have?
1. Community Edition (free)
2. Ultimate Edition (paid)

Note: Ultimate Edition has built-in support for Spring Boot, Thymeleaf, and Endpoints.
      Community Edition works fine but requires plugins for some features.
```

---

## Beginner-Friendly Mode

If `beginner_friendly` is `true` in `.spring-config.json`, explain key concepts as you work. Examples:
- When opening the project: "When IntelliJ says 'Import Gradle project', it means it will read your `build.gradle.kts` file to understand the project structure, download dependencies, and configure code completion."
- When setting up JDK: "JDK (Java Development Kit) is the toolkit needed to compile and run Java/Kotlin code. JDK 21 is the current LTS (Long-Term Support) version — use it for new projects."
- When creating a Run Configuration: "A Run Configuration tells IntelliJ how to start your app — which main class to run, which JVM flags to use, and which Spring profile to activate. The `local` profile loads your local database settings."

## Step 2 — Installing IntelliJ IDEA

If the user doesn't have IntelliJ IDEA installed:

```
Download IntelliJ IDEA:
  https://www.jetbrains.com/idea/download/

Editions:
  Community Edition — Free. Works for Java + Spring Boot. Thymeleaf support requires a plugin.
  Ultimate Edition  — Paid (free for students). Has first-class Spring Boot, Thymeleaf,
                      JPA, REST Endpoints tool windows. Worth it for serious development.

Tip: You can start a 30-day Ultimate trial to evaluate it.

macOS users (Homebrew):
  brew install --cask intellij-idea            # Ultimate
  brew install --cask intellij-idea-ce         # Community
```

---

## Step 3 — Opening the project

```
Opening {{app_name}} in IntelliJ IDEA:

1. Launch IntelliJ IDEA
2. Click "Open" (NOT "New Project")
3. Navigate to your project directory and click OK
4. IntelliJ will ask: "Open as Project" or "Open as Gradle Project"
   → Choose "Open as Gradle Project" (or it may detect automatically)
5. Wait for Gradle sync to complete (bottom status bar shows progress)

Troubleshooting:
  If you see "Cannot find build file" → make sure you opened the root folder
  (the one containing build.gradle.kts), not a subfolder.
```

---

## Step 4 — JDK setup

```
Setting up JDK 21:

1. Go to: File → Project Structure → SDKs
2. If JDK 21 is not listed → click "+" → "Add JDK..."
   - On macOS with SDKMAN: ~/.sdkman/candidates/java/21.x.x-tem/
   - Or download from: https://adoptium.net/ (Eclipse Temurin 21 LTS recommended)
3. Set Project SDK to JDK 21
4. Go to: File → Project Structure → Project
   - Language level: 21 (or "SDK default")

Via Homebrew (macOS):
  brew install --cask temurin@21

Verify in terminal:
  java -version   # should show 21.x.x
```

---

## Step 5 — Recommended plugins

Tell the user to install these plugins via `Settings → Plugins → Marketplace`:

```
Essential plugins:
  ✅ Spring Boot Assistant (freeware)
     → Auto-completion for application.yml, @Value annotations, Spring beans
     → Install: Plugins → Marketplace → search "Spring Boot Assistant"

  ✅ .env files support
     → Syntax highlighting and auto-completion for .env files
     → Install: Plugins → Marketplace → search ".env files support"

  ✅ Docker (bundled with Ultimate, available for Community)
     → Manage docker-compose services from within IntelliJ
     → Install: Plugins → Marketplace → search "Docker"

Ultimate Edition (already included):
  ✅ Spring (built-in) → Spring bean wiring, endpoint navigation
  ✅ Thymeleaf (built-in) → Attribute completion, expression validation
  ✅ JPA Buddy → Entity generation, migration helpers

Community Edition alternatives:
  ☑ JPA Buddy (free tier) → Helps generate entities and Flyway migrations
  ☑ Thymeleaf Support → partial Thymeleaf support for Community
```

---

## Step 6 — Run Configuration for bootRun with local profile

```
Creating a Run Configuration for local development:

1. Open the Gradle tool window (View → Tool Windows → Gradle)
2. Navigate to: {{app_name}} → Tasks → application → bootRun
3. Right-click "bootRun" → "Modify Run Configuration"
4. In the dialog:
   - Name: "bootRun (local)"
   - Under "Run" → "Arguments": --args='--spring.profiles.active=local'
   - OR use Environment Variables field (see Step 7 for secrets)
5. Click OK

Alternative — Run/Debug Configuration:
1. Click "Add Configuration" (top right) → "+" → "Gradle"
2. Name: bootRun local
3. Gradle project: {{app_name}}
4. Tasks: bootRun
5. Arguments: --args='--spring.profiles.active=local'
6. OK

Run with: Shift+F10 (or the green play button)
```

---

## Step 7 — Environment variables for secrets

```
Setting up secrets (GOOGLE_CLIENT_ID, etc.) in IntelliJ:

Option 1: Run Configuration Environment Variables (recommended)
1. Edit your "bootRun (local)" run configuration
2. Click "Modify options" → "Environment variables"
3. Add:
   GOOGLE_CLIENT_ID=your-client-id
   GOOGLE_CLIENT_SECRET=your-client-secret
   SPRING_PROFILES_ACTIVE=local
4. Click OK

Option 2: .env.local file with the EnvFile plugin
1. Install plugin: "EnvFile" from Marketplace
2. Create .env.local in project root (add to .gitignore!)
3. Add variables:
   GOOGLE_CLIENT_ID=your-client-id
   GOOGLE_CLIENT_SECRET=your-client-secret
4. In run configuration → EnvFile tab → enable → add .env.local

Option 3: Shell export (macOS/Linux)
  In ~/.zshrc or ~/.bash_profile:
    export GOOGLE_CLIENT_ID=your-client-id
    export GOOGLE_CLIENT_SECRET=your-client-secret
  Then restart IntelliJ (it inherits shell environment on launch).
```

---

## Step 8 — Connect Database plugin to local Docker PostgreSQL

```
Connecting IntelliJ Database tool to the local Docker PostgreSQL:

Prerequisites: docker-compose up -d (from springboot-db-setup) must be running

1. Open Database tool window: View → Tool Windows → Database
2. Click "+" → "Data Source" → "PostgreSQL"
3. Fill in:
   Host:     localhost
   Port:     5432
   Database: {{db_name}}
   User:     app
   Password: localpassword
4. Click "Test Connection" → should show "Successful"
5. Click OK

Benefits once connected:
  - Browse tables and data directly in IntelliJ
  - Run SQL queries with auto-completion
  - View Flyway migration history
  - Export/import table data

If connection fails:
  docker ps               → verify the db container is running
  docker-compose logs db  → check for errors
```

---

## Step 9 — Useful IntelliJ shortcuts for this stack

```
Helpful shortcuts:

Navigation:
  ⌘+B (Ctrl+B)         → Go to declaration/definition
  ⌘+O (Ctrl+N)         → Find class by name
  ⌘+Shift+O (Ctrl+Shift+N) → Find file by name
  ⌘+Shift+A (Ctrl+Shift+A) → Find any action
  ⌘+E (Ctrl+E)         → Recent files

Spring-specific:
  ⌘+F12 (Ctrl+F12)     → File structure (methods in current class)
  Alt+F7               → Find usages of bean/class
  In Ultimate: check the "Spring" tool window (bean wiring diagram)
  In Ultimate: check the "Endpoints" tool window (all @GetMapping etc.)

Code:
  ⌘+/ (Ctrl+/)         → Toggle line comment
  ⌘+P (Ctrl+P)         → Parameter info (in method calls)
  Shift+F6             → Rename (refactor-safe)
  ⌘+Alt+L (Ctrl+Alt+L) → Reformat code
  ⌘+Alt+O (Ctrl+Alt+O) → Optimize imports

Run & Debug:
  Shift+F10 / Shift+F9 → Run / Debug current configuration
  ⌘+F2 (Ctrl+F2)       → Stop running process
  F8                   → Step over (in debugger)
  F7                   → Step into (in debugger)
  F9                   → Resume (in debugger)
```

---

## Step 10 — Verify the setup

```
Final verification steps:

1. ✅ Start Docker Compose:
     docker-compose up -d

2. ✅ Run the app in IntelliJ:
     Click the green play button next to "bootRun (local)"
     OR press Shift+F10

3. ✅ Check the console output:
     Look for: "Started {{app_name}}Application in X.XXX seconds"
     Look for: "Successfully applied N migrations" (Flyway)
     Should NOT see: "Connection refused" or "Password authentication failed"

4. ✅ Open in browser:
     http://localhost:8080
     You should see the {{app_name}} home page

5. ✅ Verify DB connection:
     In Database tool window, expand the connection
     You should see the tables created by Flyway

Common issues:
  "Port 8080 already in use" → kill the other process or change server.port in application-local.yml
  "Cannot connect to DB"     → make sure Docker is running: docker-compose up -d
  "GOOGLE_CLIENT_ID missing" → check Step 7 — set environment variables
```

---

## Step 11 — No `installed_modules` update

This skill does not modify `installed_modules` because it makes no code changes.

Tell the user:
```
IntelliJ IDEA setup guide complete.

If you have any issues, the most common fixes are:
1. JDK not set → File → Project Structure → SDKs
2. Gradle not syncing → click the elephant icon in Gradle tool window → Reload
3. Can't find Spring annotations → make sure Spring Boot Assistant plugin is installed
4. Secrets not loading → check your run configuration environment variables

You're ready to start developing! Suggested next steps based on your installed modules:
[List remaining uninstalled skills from installed_modules]
```
