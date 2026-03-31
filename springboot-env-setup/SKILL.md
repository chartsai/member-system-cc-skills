---
name: springboot-env-setup
description: >
  TRIGGER this skill when the user says "set up my environment", "install the tools",
  "run springboot-env-setup", or asks to install prerequisites for a Spring Boot project.
  Run this BEFORE springboot-setup. It detects the OS, checks what is already installed,
  and installs any missing tools (Homebrew, Node.js, Claude Code CLI, Java 21, Docker Desktop,
  gcloud CLI).
---

# springboot-env-setup

Installs all tools needed to run the springboot-* skills. Detects what is already installed
and skips those — only installs what is missing.

---

## What this skill does

1. Detects the operating system
2. Checks each required tool
3. Installs missing tools one by one, verifying each after install
4. Prints a final checklist showing what is ready

---

## Step 1 — Detect OS

Run:

```bash
uname -s
```

- If output is `Darwin` → macOS path
- If output contains `MINGW`, `MSYS`, or `CYGWIN`, or if `uname` is unavailable → Windows path
- Inform the user which path will be followed

---

## Step 2 — Check installed tools

Run the following checks and record which tools are missing:

```bash
which brew        # macOS only
node --version
npm --version
java -version
docker --version
gcloud --version
```

Show the user a checklist of results, for example:

```
Checking your environment...

  ✅ Homebrew     3.x.x
  ❌ Node.js      not found
  ❌ npm          not found
  ✅ Java         21.0.x
  ✅ Docker       24.x.x
  ❌ gcloud       not found

Installing 3 missing tools...
```

---

## Step 3 — Install missing tools

Work through each missing tool in order. After each install, re-run its version check to confirm
it succeeded before moving on. If an install fails, show the error and ask the user whether to
retry or skip.

### macOS

#### Homebrew (install first — required for everything else)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

After install, verify:

```bash
brew --version
```

#### Node.js (provides npm)

```bash
brew install node
```

Verify:

```bash
node --version && npm --version
```

#### Claude Code CLI

```bash
npm install -g @anthropic-ai/claude-code
```

Verify:

```bash
claude --version
```

#### Java 21

```bash
brew install --cask temurin@21
```

Verify:

```bash
java -version
```

> If `java -version` still shows a different version, the user may need to set `JAVA_HOME`. Advise:
> ```bash
> export JAVA_HOME=$(/usr/libexec/java_home -v 21)
> ```
> And add it to `~/.zshrc` or `~/.bash_profile` to make it permanent.

#### Docker Desktop

```bash
brew install --cask docker
```

After install, tell the user: **Open Docker Desktop from Applications and wait for it to finish starting** before proceeding. Verify with:

```bash
docker --version
```

#### gcloud CLI (optional — needed only for GCP deployment)

Ask the user: "Do you plan to deploy to Google Cloud? (yes/no)"

If yes:

```bash
brew install --cask google-cloud-sdk
```

Verify:

```bash
gcloud --version
```

---

### Windows

On Windows, Claude Code runs inside a shell that may not have access to `winget`. Provide each command for the user to **copy and run themselves in PowerShell**.

Present all missing tools as a clear checklist of commands to run in order:

#### Node.js (provides npm)

```
winget install OpenJS.NodeJS
```

After installing, **restart PowerShell** so `npm` is on your PATH.

#### Claude Code CLI

```
npm install -g @anthropic-ai/claude-code
```

#### Java 21

```
winget install EclipseAdoptium.Temurin.21.JDK
```

#### Docker Desktop

```
winget install Docker.DockerDesktop
```

After installing, open Docker Desktop and wait for it to finish starting.

#### gcloud CLI (optional)

Ask the user: "Do you plan to deploy to Google Cloud? (yes/no)"

If yes:

```
winget install Google.CloudSDK
```

After each step, ask the user to confirm it worked (paste the version output), then continue to the next.

---

## Step 4 — Final summary

Re-run all version checks and print a final summary:

```
Environment setup complete!

  ✅ Node.js      x.x.x
  ✅ npm          x.x.x
  ✅ Java         21.x.x
  ✅ Docker       xx.x.x
  ✅ gcloud       xxx.x.x   (or ⏭️ skipped)

You're ready to start. Next step:

  1. Create a new project folder:   mkdir my-app && cd my-app
  2. Launch Claude Code:            claude
  3. Run the setup skill:           /springboot-setup
```

If any tool failed to install, list it clearly with the manual install link:

| Tool | Manual install |
|---|---|
| Homebrew | https://brew.sh |
| Node.js | https://nodejs.org |
| Java 21 | https://adoptium.net |
| Docker Desktop | https://www.docker.com/products/docker-desktop |
| gcloud CLI | https://cloud.google.com/sdk/docs/install |
