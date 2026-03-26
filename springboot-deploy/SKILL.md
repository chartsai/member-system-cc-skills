---
name: springboot-deploy
description: >
  TRIGGER this skill when the user says "deploy to Cloud Run", "set up deployment", "add Dockerfile",
  "add Cloud Build", "run springboot-deploy", "configure GCP deployment", "set up CI/CD for GCP",
  "create the deploy script", "deploy the app to production", or anything about packaging the
  Spring Boot app as a Docker image and deploying to Google Cloud Run.
  Requires setup, scaffold, db, and gcp.
---

# springboot-deploy

Sets up full Cloud Run deployment: Dockerfile (multi-stage), Cloud Build config (`cloudbuild.yaml`),
Artifact Registry, a `deploy.sh` script, and production configuration documentation.

---

## Dependencies

### Hard prerequisites
- `setup`, `scaffold`, `db`, `gcp` must all be installed

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
app_name              → used in Cloud Run service name and image tag
base_package          → Java package root
db_name               → production DB name
gcp_project_id        → GCP project ID
gcp_region            → deployment region
artifact_registry_url → set by springboot-gcp-setup (or construct from gcp_project_id + gcp_region)
test_mode             → controls build verification
language              → respond in this language
translate_terms       → whether to translate technical terms
```

> **Language activation**: After reading `language` above, switch ALL your responses to that
> language immediately — including every question, status message, explanation, and all
> human-readable text in any generated files (HTML, Thymeleaf templates, SQL seed labels).
> If `language` is not set, ask: "What language should I use? (English / 繁體中文 / other)"



Construct the artifact registry URL if not already in config:
```
{{gcp_region}}-docker.pkg.dev/{{gcp_project_id}}/{{app_name_kebab}}-repo
```

---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`, `"scaffold"`, `"db"`, `"gcp"`.

If `"gcp"` is missing:
```
Please run springboot-gcp-setup first. The deploy skill needs:
- A configured GCP project
- Artifact Registry repository
- Service Account credentials

Run `springboot-menu` to see your full project status and get guided next steps.
```

---

## Beginner-Friendly Mode

If `beginner_friendly` is `true` in `.spring-config.json`, explain key concepts as you work. Examples:
- When writing the Dockerfile: "A Dockerfile is a recipe for building a container image — a self-contained package with your app and everything it needs to run. The multi-stage build compiles your code first, then copies only the output jar into a smaller runtime image."
- When setting up Artifact Registry: "Artifact Registry is GCP's container image storage. We push your Docker image there, and Cloud Run pulls it from there when deploying. Think of it like GitHub but for Docker images."
- When creating `deploy.sh`: "This script wraps the three deployment steps — build image, push to Artifact Registry, deploy to Cloud Run — into one command so you don't have to remember them separately."

## Step 2 — `Dockerfile` (multi-stage build)

```dockerfile
# Stage 1: Build
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /workspace

# Cache Gradle dependencies
COPY gradle/ gradle/
COPY gradlew .
COPY build.gradle.kts .
COPY settings.gradle.kts .
RUN ./gradlew dependencies --no-daemon 2>/dev/null || true

# Build the application
COPY src/ src/
RUN ./gradlew bootJar --no-daemon -x test

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine AS runtime
WORKDIR /app

# Create non-root user for security
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Copy the built jar
COPY --from=build /workspace/build/libs/*.jar app.jar

EXPOSE 8080

ENTRYPOINT ["java", \
  "-XX:+UseContainerSupport", \
  "-XX:MaxRAMPercentage=75.0", \
  "-Djava.security.egd=file:/dev/./urandom", \
  "-jar", "app.jar"]
```

---

## Step 3 — `cloudbuild.yaml`

```yaml
steps:
  # Step 1: Run tests (skip for faster deploys with substitution variable)
  - name: 'gradle:8-jdk21'
    id: 'test'
    entrypoint: 'gradle'
    args: ['build', '-x', 'test']  # Change to 'build' to run tests

  # Step 2: Build Docker image
  - name: 'gcr.io/cloud-builders/docker'
    id: 'build-image'
    args:
      - 'build'
      - '-t'
      - '${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO_NAME}/${_SERVICE_NAME}:${SHORT_SHA}'
      - '-t'
      - '${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO_NAME}/${_SERVICE_NAME}:latest'
      - '.'

  # Step 3: Push to Artifact Registry
  - name: 'gcr.io/cloud-builders/docker'
    id: 'push-image'
    args:
      - 'push'
      - '--all-tags'
      - '${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO_NAME}/${_SERVICE_NAME}'

  # Step 4: Deploy to Cloud Run
  - name: 'gcr.io/google.com/cloudsdktool/cloud-sdk'
    id: 'deploy'
    entrypoint: 'gcloud'
    args:
      - 'run'
      - 'deploy'
      - '${_SERVICE_NAME}'
      - '--image=${_REGION}-docker.pkg.dev/${PROJECT_ID}/${_REPO_NAME}/${_SERVICE_NAME}:${SHORT_SHA}'
      - '--region=${_REGION}'
      - '--platform=managed'
      - '--allow-unauthenticated'
      - '--service-account=${_SERVICE_ACCOUNT}'
      - '--set-env-vars=SPRING_PROFILES_ACTIVE=prod'
      - '--set-secrets=DB_URL=DB_URL:latest,DB_USERNAME=DB_USERNAME:latest,DB_PASSWORD=DB_PASSWORD:latest'

substitutions:
  _REGION: '{{gcp_region}}'
  _REPO_NAME: '{{app_name_kebab}}-repo'
  _SERVICE_NAME: '{{app_name_kebab}}'
  _SERVICE_ACCOUNT: '{{app_name_kebab}}-sa@{{gcp_project_id}}.iam.gserviceaccount.com'

options:
  logging: CLOUD_LOGGING_ONLY
```

---

## Step 4 — `deploy.sh` — one-command deploy script

```bash
#!/bin/bash
set -euo pipefail

# ====================================================================
# deploy.sh — Build, push, and deploy {{app_name}} to Cloud Run
# Usage: ./deploy.sh [--skip-tests]
# ====================================================================

PROJECT_ID="{{gcp_project_id}}"
REGION="{{gcp_region}}"
SERVICE_NAME="{{app_name_kebab}}"
REPO_NAME="{{app_name_kebab}}-repo"
REGISTRY="${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPO_NAME}"
IMAGE="${REGISTRY}/${SERVICE_NAME}"
SERVICE_ACCOUNT="${SERVICE_NAME}-sa@${PROJECT_ID}.iam.gserviceaccount.com"

# Git short SHA for tagging
SHA=$(git rev-parse --short HEAD 2>/dev/null || echo "local")
TAG="${IMAGE}:${SHA}"
LATEST="${IMAGE}:latest"

echo "========================================"
echo "  Deploying {{app_name}} to Cloud Run"
echo "  Commit: ${SHA}"
echo "  Image:  ${TAG}"
echo "========================================"

# Authenticate Docker with Artifact Registry
echo "→ Authenticating with Artifact Registry..."
gcloud auth configure-docker "${REGION}-docker.pkg.dev" --quiet

# Build
echo "→ Building Docker image..."
docker build -t "${TAG}" -t "${LATEST}" .

# Push
echo "→ Pushing image to Artifact Registry..."
docker push "${TAG}"
docker push "${LATEST}"

# Deploy
echo "→ Deploying to Cloud Run..."
gcloud run deploy "${SERVICE_NAME}" \
  --image="${TAG}" \
  --region="${REGION}" \
  --platform=managed \
  --allow-unauthenticated \
  --service-account="${SERVICE_ACCOUNT}" \
  --set-env-vars="SPRING_PROFILES_ACTIVE=prod" \
  --set-secrets="DB_URL=DB_URL:latest,DB_USERNAME=DB_USERNAME:latest,DB_PASSWORD=DB_PASSWORD:latest" \
  --project="${PROJECT_ID}"

echo ""
echo "✅ Deployment complete!"
echo "   Service URL: $(gcloud run services describe ${SERVICE_NAME} --region=${REGION} --format='value(status.url)')"
```

Make it executable: `chmod +x deploy.sh`

---

## Step 5 — `application-prod.yml` — complete production config

```yaml
spring:
  profiles:
    active: prod
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
    driver-class-name: org.postgresql.Driver
    hikari:
      maximum-pool-size: 5
      minimum-idle: 2
      connection-timeout: 30000
      idle-timeout: 600000
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
  flyway:
    enabled: true
    validate-on-migrate: true
  thymeleaf:
    cache: true
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
  mail:
    host: ${SMTP_HOST}
    port: ${SMTP_PORT:587}
    username: ${SMTP_USERNAME}
    password: ${SMTP_PASSWORD}
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true

server:
  port: 8080
  compression:
    enabled: true
    mime-types: text/html,text/css,application/javascript,application/json

logging:
  level:
    root: WARN
    "{{base_package}}": INFO
```

---

## Step 6 — Secret Manager setup

Document which secrets must be created in GCP Secret Manager:

```
Required secrets (create in GCP Console → Secret Manager):

1. DB_URL         → jdbc:postgresql:///{{db_name}}?cloudSqlInstance={{gcp_project_id}}:{{gcp_region}}:{{app_name_kebab}}-db&socketFactory=com.google.cloud.sql.postgres.SocketFactory
2. DB_USERNAME    → app
3. DB_PASSWORD    → [your Cloud SQL password]
4. GOOGLE_CLIENT_ID     → [from OAuth2 credentials]
5. GOOGLE_CLIENT_SECRET → [from OAuth2 credentials]
6. SMTP_HOST      → smtp.sendgrid.net (or your provider)
7. SMTP_USERNAME  → apikey (SendGrid) or your username
8. SMTP_PASSWORD  → [SMTP API key]
```

---

## Step 7 — Cloud SQL connector dependency

Add to `build.gradle.kts` for production Cloud SQL connectivity:

```kotlin
// Cloud SQL connector (for production Cloud Run → Cloud SQL connection)
runtimeOnly("com.google.cloud.sql:postgres-socket-factory:1.19.0")
```

---

## Step 8 — `.dockerignore`

```dockerignore
.git/
.gradle/
build/
*.md
uploads/
gcp-credentials-local.json
.spring-config.json
application-local.yml
.gitignore
docker-compose.yml
deploy.sh
```

---

## Step 9 — Deployment checklist

Print this checklist for the user:

```
Cloud Run Deployment Checklist:

Infrastructure (GCP Console):
  □ Cloud SQL instance running: {{app_name_kebab}}-db
  □ Cloud SQL database created: {{db_name}}
  □ Cloud SQL user created: app
  □ Artifact Registry repo exists: {{gcp_region}}-docker.pkg.dev/{{gcp_project_id}}/{{app_name_kebab}}-repo
  □ Service Account has roles: secretmanager.secretAccessor, cloudsql.client, storage.objectAdmin

Secrets (Secret Manager):
  □ DB_URL created
  □ DB_USERNAME created
  □ DB_PASSWORD created
  □ GOOGLE_CLIENT_ID created (if using Google auth)
  □ GOOGLE_CLIENT_SECRET created (if using Google auth)
  □ SMTP_HOST, SMTP_USERNAME, SMTP_PASSWORD created (if using mail)

OAuth2 (Google Console):
  □ Production redirect URI added: {{app_url}}/login/oauth2/code/google

Deploy:
  □ Run: ./deploy.sh
  □ Check Cloud Run logs for startup errors
  □ Verify Flyway migrations ran successfully
  □ Visit the Cloud Run URL and test login
```

> 🔗 **Useful Console links** — substitute your `gcp_project_id` value from `.spring-config.json`:
>
> - Cloud Run services: `https://console.cloud.google.com/run?project={{gcp_project_id}}`
> - Artifact Registry: `https://console.cloud.google.com/artifacts?project={{gcp_project_id}}`
> - Cloud Build history: `https://console.cloud.google.com/cloud-build/builds?project={{gcp_project_id}}`
> - Cloud SQL instances: `https://console.cloud.google.com/sql/instances?project={{gcp_project_id}}`
> - Secret Manager: `https://console.cloud.google.com/security/secret-manager?project={{gcp_project_id}}`
> - Logs Explorer: `https://console.cloud.google.com/logs/query?project={{gcp_project_id}}`
>
> (When showing these links to the user, substitute the actual `gcp_project_id` value directly
> into the URLs — do not show `{{gcp_project_id}}` literally.)

---

## Step 10 — Cloud Build trigger (optional)

Tell the user how to set up automatic deployment on push to main:

```
To set up automatic deployment on push to main:
1. Go to GCP Console → Cloud Build → Triggers
2. Create trigger:
   - Event: Push to branch
   - Repository: [connect your repo]
   - Branch: ^main$
   - Configuration: cloudbuild.yaml
3. Grant Cloud Build the roles it needs:
   - Cloud Run Admin
   - Service Account User
   - Artifact Registry Writer
```

---

## Step 11 — Verify

If `test_mode` is `"build-only"` or `"build-and-test"`:
```bash
./gradlew build -x test
docker build -t {{app_name_kebab}}:test .
```

Verify the Docker image builds successfully locally.

---

## Step 11b — Validate Dockerfile (no unit tests for deploy scripts)

The deploy module creates infrastructure scripts, not application code, so no Spring unit tests
are generated. Validate the Dockerfile builds correctly with a local test build:

```bash
# Validate the Dockerfile parses and builds successfully
docker build -t {{app_name}}-deploy-test .
```

If it builds without error, the Dockerfile is valid. You can clean up with:
`docker rmi {{app_name}}-deploy-test`

> If `test_mode` is `build-and-test`, also verify `deploy.sh` is executable: `chmod +x deploy.sh`

---

## Step 12 — Git commits

```bash
git add Dockerfile cloudbuild.yaml deploy.sh .dockerignore
git commit -m "feat: add Dockerfile, Cloud Build config, deploy.sh for Cloud Run"
```

---

## Step 13 — Update `installed_modules`

Add `"deploy"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
Deployment setup complete!

To deploy:
  1. Complete the checklist above (infrastructure + secrets)
  2. Run: ./deploy.sh

For CI/CD: set up a Cloud Build trigger as documented above.
```
