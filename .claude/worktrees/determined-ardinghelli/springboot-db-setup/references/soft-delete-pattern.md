---
name: springboot-soft-delete
description: >
  TRIGGER this skill when the user says "add soft delete", "don't permanently delete records",
  "run springboot-soft-delete", "add deleted_at pattern", "mark as deleted instead of deleting",
  "add restore functionality", "add delete recovery", "show deleted records in admin",
  or anything about implementing a soft-delete pattern across entities.
  Requires setup, scaffold, db. BEST INSTALLED EARLY — before entity-creating modules.
---

# springboot-soft-delete

Standardizes the soft-delete pattern across entities. Instead of permanent deletion, records get
a `deleted_at` timestamp and are automatically filtered from all queries.

---

## ⚠️ Install Early Warning

**This module is most effective when installed BEFORE other entity-creating modules.**
If you already have entities (`Member`, `Submission`, etc.), this skill will explain how to
retrofit them — but it requires a Flyway migration to add `deleted_at` to each existing table
and some refactoring of existing code.

If you're starting fresh, install `springboot-soft-delete` right after `db` and `app-config`.

---

## Dependencies

### Hard prerequisites
- `setup`, `scaffold`, `db` must be installed

### Best installed before
- `springboot-membership` (apply soft-delete to Member entity)
- `springboot-item-submit` (apply to Submission entity)
- `springboot-membership-apply` (apply to Application entity)
- Any other entity-creating module

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
base_package      → Java package root
test_mode         → controls build verification
language          → respond in this language
translate_terms   → whether to translate technical terms
installed_modules → determines which entities need retrofitting
```

---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`, `"scaffold"`, `"db"`.

If entity-creating modules are already installed (`membership`, `item-submit`, etc.), tell the user:
```
⚠️  The following modules have already created entities that need to be retrofitted:
  [list modules from installed_modules]

This skill will:
1. Create the soft-delete infrastructure (interface, base repository)
2. Add Flyway migrations to add deleted_at columns to existing tables
3. Show you how to apply @SQLRestriction to existing entities
4. Create admin "restore" endpoints for each entity

This requires more steps than early installation — please review each change carefully.
```

---

## Step 2 — `SoftDeletable` interface

```java
package {{base_package}}.softdelete;

import java.time.OffsetDateTime;

/**
 * Marker interface for entities that support soft delete.
 *
 * Add to any entity that should use soft delete:
 *   @Entity
 *   @SQLRestriction("deleted_at IS NULL")  // Spring Boot 3.x Hibernate 6 annotation
 *   public class MyEntity implements SoftDeletable { ... }
 */
public interface SoftDeletable {
    OffsetDateTime getDeletedAt();
    void setDeletedAt(OffsetDateTime deletedAt);

    default boolean isDeleted() {
        return getDeletedAt() != null;
    }
}
```

---

## Step 3 — `SoftDeleteRepository` base class

```java
package {{base_package}}.softdelete;

import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Modifying;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.NoRepositoryBean;
import org.springframework.data.repository.query.Param;
import org.springframework.transaction.annotation.Transactional;
import java.time.OffsetDateTime;
import java.util.List;
import java.util.Optional;

/**
 * Base repository for entities with soft delete.
 *
 * Extends standard JpaRepository but adds:
 *   - softDelete(id) → sets deleted_at instead of DELETE
 *   - restore(id) → clears deleted_at
 *   - findDeleted() → finds only soft-deleted records (admin use)
 *
 * NOTE: The @SQLRestriction("deleted_at IS NULL") on the entity class handles
 * filtering for all standard JpaRepository query methods automatically.
 * You do NOT need to add "AND deleted_at IS NULL" to custom queries —
 * Hibernate applies it as a SQL filter at the entity level.
 */
@NoRepositoryBean
public interface SoftDeleteRepository<T extends SoftDeletable, ID>
        extends JpaRepository<T, ID> {

    @Modifying
    @Transactional
    @Query("UPDATE #{#entityName} e SET e.deletedAt = :now WHERE e.id = :id")
    void softDeleteById(@Param("id") ID id, @Param("now") OffsetDateTime now);

    @Modifying
    @Transactional
    @Query("UPDATE #{#entityName} e SET e.deletedAt = NULL WHERE e.id = :id")
    void restoreById(@Param("id") ID id);

    /**
     * Find all deleted records (bypasses the @SQLRestriction filter).
     * Implemented using native SQL to ignore the filter.
     */
    @Query(value = "SELECT * FROM #{#entityName} WHERE deleted_at IS NOT NULL ORDER BY deleted_at DESC",
           nativeQuery = true)
    List<T> findAllDeleted();
}
```

---

## Step 4 — Applying to entities

### Pattern for NEW entities (recommended when installing early)

```java
@Entity
@Table(name = "my_entity")
@org.hibernate.annotations.SQLRestriction("deleted_at IS NULL")  // ← key annotation
@Getter @Setter @NoArgsConstructor @Builder @AllArgsConstructor
public class MyEntity implements SoftDeletable {

    // ... other fields ...

    @Column(name = "deleted_at")
    private OffsetDateTime deletedAt;  // null = active, non-null = soft-deleted
}
```

### Pattern for EXISTING entities (retrofitting)

If `membership` is installed, add to `Member`:

```java
// Add to Member entity:
@Column(name = "deleted_at")
private OffsetDateTime deletedAt;

// Add @SQLRestriction to Member class:
@org.hibernate.annotations.SQLRestriction("deleted_at IS NULL")
```

If `item-submit` is installed, add the same to `Submission`.
If `membership-apply` is installed, add to `Application`.

---

## Step 5 — Flyway migrations for existing tables

Create `V_soft_delete__add_columns.sql` (next available version):

```sql
-- Add soft delete support to existing tables
-- Only add if the module is installed (check installed_modules)

-- For Member table (if membership or auth modules installed):
ALTER TABLE members ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMPTZ;
CREATE INDEX IF NOT EXISTS idx_members_deleted_at ON members (deleted_at) WHERE deleted_at IS NULL;

-- For Submission table (if item-submit installed):
ALTER TABLE submissions ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMPTZ;
CREATE INDEX IF NOT EXISTS idx_submissions_deleted_at ON submissions (deleted_at) WHERE deleted_at IS NULL;

-- For Application table (if membership-apply installed):
ALTER TABLE applications ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMPTZ;
CREATE INDEX IF NOT EXISTS idx_applications_deleted_at ON applications (deleted_at) WHERE deleted_at IS NULL;

-- For Announcement table (if announcements installed):
ALTER TABLE announcements ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMPTZ;
```

Only include ALTER TABLE statements for tables that actually exist.

---

## Step 6 — `SoftDeleteService` helper

```java
package {{base_package}}.softdelete;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.OffsetDateTime;

/**
 * Generic helper for soft-delete operations.
 * Individual services should call this rather than implementing their own soft-delete logic.
 */
@Service
@Slf4j
@RequiredArgsConstructor
public class SoftDeleteService {

    /**
     * Soft-delete an entity using its repository.
     * The caller is responsible for authorization checks before calling this.
     */
    @Transactional
    public <T extends SoftDeletable, ID> void softDelete(
            SoftDeleteRepository<T, ID> repository, ID id) {
        repository.softDeleteById(id, OffsetDateTime.now());
        log.info("Soft-deleted entity with id: {}", id);
    }

    /**
     * Restore a soft-deleted entity.
     */
    @Transactional
    public <T extends SoftDeletable, ID> void restore(
            SoftDeleteRepository<T, ID> repository, ID id) {
        repository.restoreById(id);
        log.info("Restored soft-deleted entity with id: {}", id);
    }
}
```

---

## Step 7 — Admin "restore" endpoint pattern

For each soft-deletable entity, add a restore endpoint. Example for `Member`:

```java
// In AdminMemberController:

@GetMapping("/deleted")
public String deletedMembers(Model model) {
    model.addAttribute("members", memberRepository.findAllDeleted());
    return "admin/members/deleted";
}

@PostMapping("/{id}/soft-delete")
public String softDelete(@PathVariable Long id, RedirectAttributes redirectAttributes) {
    softDeleteService.softDelete(memberRepository, id);
    redirectAttributes.addFlashAttribute("success", "Member soft-deleted.");
    return "redirect:/admin/members";
}

@PostMapping("/{id}/restore")
public String restore(@PathVariable Long id, RedirectAttributes redirectAttributes) {
    softDeleteService.restore(memberRepository, id);
    redirectAttributes.addFlashAttribute("success", "Member restored.");
    return "redirect:/admin/members";
}
```

---

## Step 8 — "Show deleted" toggle in admin views

Add a toggle to existing admin list pages:

```html
<!-- Toggle button in admin/members/list.html -->
<div class="flex items-center gap-2 mb-4">
    <a th:href="@{/admin/members}"
       th:class="${!showDeleted} ? 'bg-blue-600 text-white px-4 py-2 rounded-lg text-sm'
                                 : 'bg-white border border-gray-300 text-gray-600 px-4 py-2 rounded-lg text-sm'">
        Active
    </a>
    <a th:href="@{/admin/members/deleted}"
       th:class="${showDeleted} ? 'bg-red-600 text-white px-4 py-2 rounded-lg text-sm'
                                : 'bg-white border border-gray-300 text-gray-600 px-4 py-2 rounded-lg text-sm'">
        Deleted (<span th:text="${deletedCount}">0</span>)
    </a>
</div>
```

---

## Step 9 — Unit tests for filter logic + restore

If `test_mode` is NOT `"token-save"`:

```java
// SoftDeleteRepositoryTest.java (integration test with @DataJpaTest + Testcontainers)
// Test: save entity → findAll returns it (deleted_at IS NULL filter active)
// Test: softDelete → findAll does NOT return it
// Test: softDelete → findAllDeleted DOES return it
// Test: restore → findAll returns it again
// Test: direct JPA findById still works after soft-delete (bypasses filter)

// SoftDeleteServiceTest.java
// Test: softDelete calls repository.softDeleteById with current timestamp
// Test: restore calls repository.restoreById
```

---

## Step 10 — Verify

Apply `test_mode`.

---

## Step 11 — Git commits

```bash
git add -A
git commit -m "feat: add soft-delete pattern with SoftDeletable interface, base repository, and admin restore"
```

---

## Step 12 — Update `installed_modules`

Add `"soft-delete"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
Soft-delete infrastructure is ready.

To apply to any entity:
1. Add implements SoftDeletable to the entity class
2. Add @SQLRestriction("deleted_at IS NULL") to the entity class
3. Add OffsetDateTime deletedAt field
4. Add Flyway migration: ALTER TABLE your_table ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMPTZ
5. Update the repository to extend SoftDeleteRepository<Entity, Long>
6. Add /restore and /deleted endpoints to the admin controller

Warning: After adding @SQLRestriction, ALL queries via JPA will automatically exclude deleted records.
         Raw JDBC/native SQL queries are not affected — add WHERE deleted_at IS NULL manually.
```
