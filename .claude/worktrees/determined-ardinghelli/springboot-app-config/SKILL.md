---
name: springboot-app-config
description: >
  TRIGGER this skill when the user says "add app config", "add application settings", "add DB-backed config",
  "run springboot-app-config", "add key-value settings", "add admin settings page",
  "add configurable settings to the database", or anything about storing application configuration
  in the database rather than in files. This is a FOUNDATIONAL skill used by mail rules,
  data-export mappings, feature flags, and other modules. Install early. Requires setup, scaffold, db.
---

# springboot-app-config

Adds a DB-backed key-value configuration store. Other modules use this to store configurable settings
(email rule state, export mappings, feature flags, etc.) without requiring file changes or redeployment.

**Install this early** — other modules check for it and use it if available.

---

## Dependencies

### Hard prerequisites
- `setup`, `scaffold`, `db` must be installed

### Best installed before
- `springboot-mail` (stores email rule config)
- `springboot-data-export` (stores field mappings)
- Any module that needs runtime-configurable settings

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
base_package      → Java package root
test_mode         → controls build verification
language          → respond in this language
translate_terms   → whether to translate technical terms
installed_modules → must contain "setup", "scaffold", "db"
```

---

## Step 1 — Prerequisites check

Note: The `springboot-db-setup` skill already creates an `app_config` table with a minimal schema
(`key VARCHAR PK, value TEXT`). If that migration already ran, this skill will **extend** it rather
than recreate it.

Check if `V1__init.sql` already created the `app_config` table by looking at the migration files.

---

## Step 2 — Flyway migration

Check if `app_config` table already exists (from `V1__init.sql`). If so, create an additive migration:

Create `V_app_config__enhance.sql` (use next available version):

```sql
-- Enhance app_config table if it exists from V1
-- Add metadata columns for the admin UI
ALTER TABLE app_config
    ADD COLUMN IF NOT EXISTS description TEXT,
    ADD COLUMN IF NOT EXISTS category     VARCHAR(50) DEFAULT 'general',
    ADD COLUMN IF NOT EXISTS is_secret    BOOLEAN     NOT NULL DEFAULT FALSE,
    ADD COLUMN IF NOT EXISTS updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW();

-- Seed essential default config values
INSERT INTO app_config (key, value, description, category) VALUES
    ('app.features.registration_open',  'true',   'Whether new member registration is open', 'features'),
    ('app.features.announcements',      'true',   'Enable announcements module', 'features'),
    ('app.limits.max_file_size_mb',     '10',     'Maximum upload file size in MB', 'storage'),
    ('app.ui.primary_color',            '#3b82f6','Primary brand color (hex)', 'ui'),
    ('app.ui.footer_text',              '',       'Footer text shown on all pages', 'ui')
ON CONFLICT (key) DO NOTHING;
```

If `app_config` does NOT exist yet (the `db-setup` skill migration didn't include it):

```sql
CREATE TABLE app_config (
    key         VARCHAR(255) PRIMARY KEY,
    value       TEXT,
    description TEXT,
    category    VARCHAR(50) NOT NULL DEFAULT 'general',
    is_secret   BOOLEAN     NOT NULL DEFAULT FALSE,
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE app_config IS 'DB-backed key-value configuration store';
```

---

## Step 3 — Update `AppConfigEntity`

If the entity already exists from `springboot-db-setup`, update it with the new fields:

```java
package {{base_package}}.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.OffsetDateTime;

@Entity
@Table(name = "app_config")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class AppConfigEntity {

    @Id
    @Column(name = "key")
    private String key;

    @Column(name = "value", columnDefinition = "TEXT")
    private String value;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column
    private String category;

    @Column(name = "is_secret")
    private boolean secret = false;

    @Column(name = "updated_at")
    private OffsetDateTime updatedAt;

    @PrePersist
    @PreUpdate
    void onUpdate() {
        updatedAt = OffsetDateTime.now();
    }
}
```

---

## Step 4 — `AppConfigService`

```java
package {{base_package}}.service;

import {{base_package}}.entity.AppConfigEntity;
import {{base_package}}.repository.AppConfigRepository;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.*;

@Service
@Slf4j
@RequiredArgsConstructor
public class AppConfigService {

    private final AppConfigRepository appConfigRepository;
    private final ObjectMapper objectMapper;

    /**
     * Get a config value as a String. Returns defaultValue if key not found.
     */
    public String get(String key, String defaultValue) {
        return appConfigRepository.findById(key)
            .map(AppConfigEntity::getValue)
            .orElse(defaultValue);
    }

    /**
     * Get a config value as boolean.
     */
    public boolean getBoolean(String key, boolean defaultValue) {
        String value = get(key, null);
        if (value == null) return defaultValue;
        return Boolean.parseBoolean(value);
    }

    /**
     * Get a config value as integer.
     */
    public int getInt(String key, int defaultValue) {
        String value = get(key, null);
        if (value == null) return defaultValue;
        try { return Integer.parseInt(value); }
        catch (NumberFormatException e) { return defaultValue; }
    }

    /**
     * Get a config value deserialized from JSON.
     * Useful for complex config objects (field mappings, rule lists, etc.)
     */
    public <T> Optional<T> getJson(String key, TypeReference<T> typeRef) {
        String value = get(key, null);
        if (value == null) return Optional.empty();
        try {
            return Optional.of(objectMapper.readValue(value, typeRef));
        } catch (Exception e) {
            log.warn("Failed to parse JSON config for key '{}': {}", key, e.getMessage());
            return Optional.empty();
        }
    }

    /**
     * Set a config value. Creates the entry if it doesn't exist.
     */
    @Transactional
    public void set(String key, String value) {
        AppConfigEntity entity = appConfigRepository.findById(key)
            .orElseGet(() -> AppConfigEntity.builder().key(key).build());
        entity.setValue(value);
        appConfigRepository.save(entity);
        log.debug("Config updated: {} = {}", key, entity.isSecret() ? "****" : value);
    }

    /**
     * Set a config value from any object (serialized to JSON).
     */
    @Transactional
    public void setJson(String key, Object value) {
        try {
            set(key, objectMapper.writeValueAsString(value));
        } catch (Exception e) {
            throw new RuntimeException("Failed to serialize config value for key: " + key, e);
        }
    }

    /**
     * Set a config value with metadata.
     */
    @Transactional
    public void set(String key, String value, String description, String category, boolean secret) {
        AppConfigEntity entity = appConfigRepository.findById(key)
            .orElseGet(() -> AppConfigEntity.builder().key(key).build());
        entity.setValue(value);
        entity.setDescription(description);
        entity.setCategory(category);
        entity.setSecret(secret);
        appConfigRepository.save(entity);
    }

    public List<AppConfigEntity> findAll() {
        return appConfigRepository.findAll();
    }

    public List<AppConfigEntity> findByCategory(String category) {
        return appConfigRepository.findByCategoryOrderByKey(category);
    }

    public void delete(String key) {
        appConfigRepository.deleteById(key);
    }

    /** Convenience: check a feature flag */
    public boolean isFeatureEnabled(String featureName) {
        return getBoolean("app.features." + featureName, false);
    }
}
```

---

## Step 5 — Update `AppConfigRepository`

```java
package {{base_package}}.repository;

import {{base_package}}.entity.AppConfigEntity;
import org.springframework.data.jpa.repository.JpaRepository;
import java.util.List;

public interface AppConfigRepository extends JpaRepository<AppConfigEntity, String> {
    List<AppConfigEntity> findByCategoryOrderByKey(String category);
    List<AppConfigEntity> findAllByOrderByCategoryAscKeyAsc();
}
```

---

## Step 6 — `AdminConfigController`

```java
package {{base_package}}.controller.admin;

import {{base_package}}.service.AppConfigService;
import lombok.RequiredArgsConstructor;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.servlet.mvc.support.RedirectAttributes;
import java.util.Map;
import java.util.stream.Collectors;

@Controller
@RequestMapping("/admin/config")
@PreAuthorize("hasRole('SUPER_ADMIN')")
@RequiredArgsConstructor
public class AdminConfigController {

    private final AppConfigService appConfigService;

    @GetMapping
    public String list(Model model) {
        // Group by category for display
        Map<String, java.util.List<{{base_package}}.entity.AppConfigEntity>> grouped =
            appConfigService.findAll().stream()
                .collect(Collectors.groupingBy(
                    e -> e.getCategory() != null ? e.getCategory() : "general"
                ));
        model.addAttribute("configGroups", grouped);
        return "admin/config/list";
    }

    @PostMapping("/{key}")
    public String update(@PathVariable String key,
                         @RequestParam String value,
                         RedirectAttributes redirectAttributes) {
        try {
            appConfigService.set(key, value);
            redirectAttributes.addFlashAttribute("success", "Config updated: " + key);
        } catch (Exception e) {
            redirectAttributes.addFlashAttribute("error", e.getMessage());
        }
        return "redirect:/admin/config";
    }

    @PostMapping("/new")
    public String create(@RequestParam String key,
                         @RequestParam String value,
                         @RequestParam(required = false) String description,
                         @RequestParam(required = false) String category,
                         @RequestParam(required = false, defaultValue = "false") boolean secret,
                         RedirectAttributes redirectAttributes) {
        try {
            appConfigService.set(key, value, description,
                category != null && !category.isBlank() ? category : "general",
                secret);
            redirectAttributes.addFlashAttribute("success", "Config key created: " + key);
        } catch (Exception e) {
            redirectAttributes.addFlashAttribute("error", e.getMessage());
        }
        return "redirect:/admin/config";
    }

    @PostMapping("/{key}/delete")
    public String delete(@PathVariable String key, RedirectAttributes redirectAttributes) {
        appConfigService.delete(key);
        redirectAttributes.addFlashAttribute("success", "Config key deleted: " + key);
        return "redirect:/admin/config";
    }
}
```

---

## Step 7 — Thymeleaf template

### `templates/admin/config/list.html`

Admin config page:
- Grouped by category (features, storage, ui, general, etc.)
- Table: Key, Value (masked for secrets), Description, Edit inline
- Inline edit: click value → input field appears → save button
- "Add new config key" form at bottom (key, value, description, category, secret checkbox)
- SUPER_ADMIN only access (secured at controller level)

---

## Step 8 — Usage pattern for other modules

Document how other modules use `AppConfigService`:

```java
// In any Spring component:
@Autowired
private AppConfigService configService;

// Boolean feature flag
boolean registrationOpen = configService.getBoolean("app.features.registration_open", true);

// String value with default
String footerText = configService.get("app.ui.footer_text", "");

// Integer limit
int maxFileSizeMb = configService.getInt("app.limits.max_file_size_mb", 10);

// Complex JSON object
Optional<List<ExportFieldConfig>> mapping = configService.getJson(
    "export.members.fields",
    new TypeReference<List<ExportFieldConfig>>() {}
);
```

---

## Step 9 — Unit tests

If `test_mode` is NOT `"token-save"`:

```java
// AppConfigServiceTest.java
// Test: get with missing key → returns defaultValue
// Test: set then get → returns new value
// Test: getBoolean("true", false) → true
// Test: getBoolean("false", true) → false
// Test: getBoolean(missing, true) → true (default)
// Test: getInt("42", 0) → 42
// Test: getInt("not-a-number", 5) → 5 (default)
// Test: isFeatureEnabled with "true" value → returns true
```

---

## Step 10 — Verify

Apply `test_mode`.

---

## Step 11 — Git commits

```bash
git add -A
git commit -m "feat: add DB-backed app config system with admin UI and typed service"
```

---

## Step 12 — Update `installed_modules`

Add `"app-config"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
App config system ready.

Admin config UI: http://localhost:8080/admin/config (SUPER_ADMIN only)

Usage in other components:
  configService.getBoolean("app.features.registration_open", true)
  configService.get("app.ui.footer_text", "")

Other modules will automatically use this for their configurable settings.
```
