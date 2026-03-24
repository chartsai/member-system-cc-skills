---
name: springboot-notifications
description: >
  TRIGGER this skill when the user wants to add in-app notifications, a notification bell,
  unread notification badges, or a notification center to their Spring Boot app.
  Trigger phrases: "add notifications", "add a notification bell", "add unread badge",
  "add notification center", "run springboot-notifications", "notify members in-app",
  "add bell icon with unread count", "add in-app alerts".
  Creates a notifications table, NotificationService, REST endpoints for polling,
  and a Thymeleaf bell icon component with live unread count.
  Prerequisites: setup, scaffold, db, membership must be installed.
---

# springboot-notifications

Adds an in-app notification system: a bell icon in the header shows an unread count badge,
clicking it opens a dropdown with recent notifications. Members can mark individual or all
notifications as read. A `/notifications` page shows the full paginated history.
Other modules (announcements, membership-apply, item-submit) can hook in to send notifications
automatically when key events occur.

---

## Dependencies

### Hard prerequisites
- `setup`, `scaffold`, `db`, `membership` must be installed

### Soft prerequisites
- `announcements` → auto-notify all members when an announcement is published
- `membership-apply` → notify member when application is approved or rejected
- `item-submit` → notify submitter when item is approved or rejected

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
app_name          → application name (used in UI copy)
base_package      → Java package root
language          → respond in this language for ALL output (questions, explanations,
                    status messages, code comments, and user-facing copy in generated files)
                    IMPORTANT: if "zh-TW", "traditional-chinese", or "繁體中文":
                      use 繁體中文 (Traditional Chinese) throughout
                      NEVER use Simplified Chinese (简体中文) — they are different writing systems
                      Key differences: 體/体, 語/语, 資料/数据, 設定/设定, 確認/确认, 請/请
translate_terms   → whether to translate technical terms
beginner_friendly → if true, explain technical terms and decisions as you work
installed_modules → must include "setup", "scaffold", "db", "membership"
                    check for "announcements", "membership-apply", "item-submit"
test_mode         → controls build verification
```

> **Language activation**: After reading `language` above, switch ALL your responses to that
> language immediately — including every question, status message, explanation, and all
> human-readable text in any generated files (HTML, Thymeleaf templates, SQL seed labels).
> If `language` is not set, ask: "What language should I use? (English / 繁體中文 / other)"

---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`, `"scaffold"`, `"db"`, `"membership"`.

If any are missing, stop and show:
```
Missing prerequisites: [list missing modules]
Run springboot-menu to see what needs to be installed first.
```

Note which soft prerequisites are present — used in Step 8.

---

## Step 2 — Flyway migration: notifications table

Scan `src/main/resources/db/migration/` for existing files to find the next available version number.

Create `src/main/resources/db/migration/V__create_notifications.sql` (use next version):

```sql
CREATE TABLE notifications (
    id         BIGSERIAL PRIMARY KEY,
    member_id  BIGINT       NOT NULL REFERENCES members(id) ON DELETE CASCADE,
    type       VARCHAR(50)  NOT NULL,        -- e.g. 'ANNOUNCEMENT', 'APPROVAL', 'SYSTEM'
    title      VARCHAR(255) NOT NULL,
    message    TEXT,
    action_url VARCHAR(500),                 -- optional link when notification is clicked
    read_at    TIMESTAMP,                    -- NULL = unread; non-NULL = read (records when)
    created_at TIMESTAMP    NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_notifications_member_id ON notifications(member_id);
CREATE INDEX idx_notifications_unread    ON notifications(member_id, read_at) WHERE read_at IS NULL;
```

---

## Step 3 — `Notification` entity and repository

### `Notification.kt`

```kotlin
package {{base_package}}.entity

import jakarta.persistence.*
import java.time.LocalDateTime

@Entity
@Table(name = "notifications")
class Notification(

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(name = "member_id", nullable = false)
    val memberId: Long,

    @Column(nullable = false, length = 50)
    val type: String,                        // e.g. "ANNOUNCEMENT", "APPROVAL", "SYSTEM"

    @Column(nullable = false)
    val title: String,

    @Column(columnDefinition = "TEXT")
    val message: String? = null,

    @Column(name = "action_url", length = 500)
    val actionUrl: String? = null,

    @Column(name = "read_at")
    var readAt: LocalDateTime? = null,       // NULL = unread

    @Column(name = "created_at", nullable = false, updatable = false)
    val createdAt: LocalDateTime = LocalDateTime.now()
) {
    val isRead: Boolean get() = readAt != null
    val isUnread: Boolean get() = readAt == null
}
```

### `NotificationRepository.kt`

```kotlin
package {{base_package}}.repository

import {{base_package}}.entity.Notification
import org.springframework.data.domain.Page
import org.springframework.data.domain.Pageable
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Modifying
import org.springframework.data.jpa.repository.Query

interface NotificationRepository : JpaRepository<Notification, Long> {

    fun findByMemberIdOrderByCreatedAtDesc(memberId: Long): List<Notification>

    fun countByMemberIdAndReadAtIsNull(memberId: Long): Long

    fun findByMemberIdAndReadAtIsNullOrderByCreatedAtDesc(memberId: Long): List<Notification>

    fun findByMemberIdOrderByCreatedAtDesc(memberId: Long, pageable: Pageable): Page<Notification>

    @Modifying
    @Query("UPDATE Notification n SET n.readAt = CURRENT_TIMESTAMP WHERE n.memberId = :memberId AND n.readAt IS NULL")
    fun markAllReadByMemberId(memberId: Long): Int
}
```

---

## Step 4 — `NotificationService`

```kotlin
package {{base_package}}.service

import {{base_package}}.entity.Notification
import {{base_package}}.repository.MemberRepository
import {{base_package}}.repository.NotificationRepository
import org.slf4j.LoggerFactory
import org.springframework.data.domain.PageRequest
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.time.LocalDateTime

@Service
class NotificationService(
    private val notificationRepository: NotificationRepository,
    private val memberRepository: MemberRepository
) {
    private val log = LoggerFactory.getLogger(javaClass)

    /** Create a notification for a single member. */
    @Transactional
    fun createNotification(
        memberId: Long,
        type: String,
        title: String,
        message: String? = null,
        actionUrl: String? = null
    ): Notification {
        val notification = Notification(
            memberId = memberId,
            type = type,
            title = title,
            message = message,
            actionUrl = actionUrl
        )
        return notificationRepository.save(notification)
    }

    /** Create notifications for ALL active members (processed in batches of 100). */
    @Transactional
    fun notifyAll(type: String, title: String, message: String? = null, actionUrl: String? = null) {
        val activeMembers = memberRepository.findAll()
            .filter { it.status?.name == "ACTIVE" }  // adjust to your Member.Status field

        activeMembers.chunked(100).forEach { batch ->
            val notifications = batch.map { member ->
                Notification(
                    memberId = member.id,
                    type = type,
                    title = title,
                    message = message,
                    actionUrl = actionUrl
                )
            }
            notificationRepository.saveAll(notifications)
        }
        log.info("Created '{}' notifications for {} active members", type, activeMembers.size)
    }

    /** Create notifications for all active members with a given role. */
    @Transactional
    fun notifyRole(role: String, type: String, title: String, message: String? = null) {
        val members = memberRepository.findAll()
            .filter { it.status?.name == "ACTIVE" && it.role?.name == role }

        val notifications = members.map { member ->
            Notification(memberId = member.id, type = type, title = title, message = message)
        }
        notificationRepository.saveAll(notifications)
        log.info("Created '{}' notifications for {} members with role {}", type, members.size, role)
    }

    /** Mark one notification as read. Verifies ownership to prevent spoofing. */
    @Transactional
    fun markRead(notificationId: Long, memberId: Long) {
        val notification = notificationRepository.findById(notificationId)
            .orElseThrow { IllegalArgumentException("Notification not found: $notificationId") }
        require(notification.memberId == memberId) { "Notification does not belong to this member" }
        if (notification.isUnread) {
            notification.readAt = LocalDateTime.now()
            notificationRepository.save(notification)
        }
    }

    /** Mark all notifications as read for a member. */
    @Transactional
    fun markAllRead(memberId: Long): Int {
        return notificationRepository.markAllReadByMemberId(memberId)
    }

    /** Returns the number of unread notifications for a member. */
    fun getUnreadCount(memberId: Long): Long {
        return notificationRepository.countByMemberIdAndReadAtIsNull(memberId)
    }

    /** Returns the most recent notifications for a member (default: last 10). */
    fun getRecent(memberId: Long, limit: Int = 10): List<Notification> {
        return notificationRepository.findByMemberIdOrderByCreatedAtDesc(
            memberId, PageRequest.of(0, limit)
        ).content
    }

    /** Returns ALL notifications for a member, paginated. */
    fun getAll(memberId: Long, page: Int = 0, size: Int = 20) =
        notificationRepository.findByMemberIdOrderByCreatedAtDesc(
            memberId, PageRequest.of(page, size)
        )
}
```

---

## Step 5 — `NotificationController` (REST API)

These are lightweight JSON endpoints used by the frontend for polling and actions.

```kotlin
package {{base_package}}.controller

import {{base_package}}.service.MemberService
import {{base_package}}.service.NotificationService
import jakarta.servlet.http.HttpSession
import org.springframework.http.ResponseEntity
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.security.oauth2.core.user.OAuth2User
import org.springframework.web.bind.annotation.*

@RestController
@RequestMapping("/api/notifications")
class NotificationController(
    private val notificationService: NotificationService,
    private val memberService: MemberService
) {

    /** GET /api/notifications?limit=10 — recent notifications for current user */
    @GetMapping
    fun getRecent(
        @RequestParam(defaultValue = "10") limit: Int,
        @AuthenticationPrincipal oauth: OAuth2User?,
        session: HttpSession
    ): ResponseEntity<*> {
        val memberId = resolveMemberId(oauth, session) ?: return ResponseEntity.status(401).build<Any>()
        val notifications = notificationService.getRecent(memberId, limit)
        return ResponseEntity.ok(notifications.map { n ->
            mapOf(
                "id"        to n.id,
                "type"      to n.type,
                "title"     to n.title,
                "message"   to n.message,
                "actionUrl" to n.actionUrl,
                "isRead"    to n.isRead,
                "createdAt" to n.createdAt.toString()
            )
        })
    }

    /** GET /api/notifications/unread-count — used for polling every 30 seconds */
    @GetMapping("/unread-count")
    fun getUnreadCount(
        @AuthenticationPrincipal oauth: OAuth2User?,
        session: HttpSession
    ): ResponseEntity<*> {
        val memberId = resolveMemberId(oauth, session) ?: return ResponseEntity.status(401).build<Any>()
        return ResponseEntity.ok(mapOf("count" to notificationService.getUnreadCount(memberId)))
    }

    /** POST /api/notifications/{id}/read — mark one notification as read */
    @PostMapping("/{id}/read")
    fun markRead(
        @PathVariable id: Long,
        @AuthenticationPrincipal oauth: OAuth2User?,
        session: HttpSession
    ): ResponseEntity<*> {
        val memberId = resolveMemberId(oauth, session) ?: return ResponseEntity.status(401).build<Any>()
        return try {
            notificationService.markRead(id, memberId)
            ResponseEntity.ok(mapOf("success" to true))
        } catch (e: IllegalArgumentException) {
            ResponseEntity.badRequest().body(mapOf("error" to e.message))
        }
    }

    /** POST /api/notifications/read-all — mark all notifications as read */
    @PostMapping("/read-all")
    fun markAllRead(
        @AuthenticationPrincipal oauth: OAuth2User?,
        session: HttpSession
    ): ResponseEntity<*> {
        val memberId = resolveMemberId(oauth, session) ?: return ResponseEntity.status(401).build<Any>()
        val count = notificationService.markAllRead(memberId)
        return ResponseEntity.ok(mapOf("marked" to count))
    }

    private fun resolveMemberId(oauth: OAuth2User?, session: HttpSession): Long? {
        return (oauth?.getAttribute("id") as? Long)
            ?: (session.getAttribute("memberId") as? Long)
    }
}
```

---

## Step 6 — Notification bell Thymeleaf fragment

Create `src/main/resources/templates/fragments/notification-bell.html`:

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" lang="en">
<body>

<!--/* Notification Bell Fragment
     Usage in base layout header:
       <div th:replace="~{fragments/notification-bell :: notificationBell}"></div>
*/ -->
<div th:fragment="notificationBell" x-data="notificationBell()" x-init="init()">

  <!-- Bell button -->
  <div class="relative">
    <button
      @click="toggle()"
      class="relative p-2 text-gray-500 hover:text-gray-700 hover:bg-gray-100 rounded-full transition-colors focus:outline-none focus:ring-2 focus:ring-blue-500"
      aria-label="Notifications"
    >
      <!-- Bell icon (SVG) -->
      <svg xmlns="http://www.w3.org/2000/svg" class="h-6 w-6" fill="none" viewBox="0 0 24 24" stroke="currentColor">
        <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
              d="M15 17h5l-1.405-1.405A2.032 2.032 0 0118 14.158V11a6.002 6.002 0 00-4-5.659V5a2 2 0 10-4 0v.341C7.67 6.165 6 8.388 6 11v3.159c0 .538-.214 1.055-.595 1.436L4 17h5m6 0v1a3 3 0 11-6 0v-1m6 0H9" />
      </svg>
      <!-- Unread count badge (hidden when zero) -->
      <span
        x-show="unreadCount > 0"
        x-text="unreadCount > 99 ? '99+' : unreadCount"
        class="absolute -top-1 -right-1 inline-flex items-center justify-center min-w-[1.25rem] h-5 px-1 text-xs font-bold text-white bg-red-500 rounded-full"
        style="display: none;"
      ></span>
    </button>

    <!-- Dropdown panel -->
    <div
      x-show="open"
      @click.away="open = false"
      x-transition:enter="transition ease-out duration-150"
      x-transition:enter-start="opacity-0 scale-95"
      x-transition:enter-end="opacity-100 scale-100"
      x-transition:leave="transition ease-in duration-100"
      x-transition:leave-start="opacity-100 scale-100"
      x-transition:leave-end="opacity-0 scale-95"
      class="absolute right-0 mt-2 w-96 max-w-sm bg-white rounded-xl shadow-lg border border-gray-200 z-50 overflow-hidden"
      style="display: none;"
    >
      <!-- Header -->
      <div class="flex items-center justify-between px-4 py-3 border-b border-gray-100 bg-gray-50">
        <h3 class="text-sm font-semibold text-gray-700">Notifications</h3>
        <button
          x-show="unreadCount > 0"
          @click="markAllRead()"
          class="text-xs text-blue-600 hover:text-blue-800 font-medium"
        >Mark all read</button>
      </div>

      <!-- Notification list -->
      <ul class="divide-y divide-gray-100 max-h-80 overflow-y-auto">
        <template x-if="notifications.length === 0">
          <li class="px-4 py-8 text-center text-sm text-gray-400">No notifications</li>
        </template>
        <template x-for="n in notifications" :key="n.id">
          <li
            :class="n.isRead ? 'bg-white' : 'bg-blue-50'"
            class="px-4 py-3 hover:bg-gray-50 transition-colors cursor-pointer"
            @click="handleClick(n)"
          >
            <div class="flex items-start gap-3">
              <div class="flex-1 min-w-0">
                <p class="text-sm font-medium text-gray-800 truncate" x-text="n.title"></p>
                <p x-show="n.message" class="text-xs text-gray-500 mt-0.5 line-clamp-2" x-text="n.message"></p>
                <p class="text-xs text-gray-400 mt-1" x-text="formatDate(n.createdAt)"></p>
              </div>
              <span x-show="!n.isRead" class="mt-1 w-2 h-2 rounded-full bg-blue-500 flex-shrink-0"></span>
            </div>
          </li>
        </template>
      </ul>

      <!-- Footer -->
      <div class="px-4 py-2 border-t border-gray-100 bg-gray-50 text-center">
        <a href="/notifications" class="text-xs text-blue-600 hover:text-blue-800 font-medium">
          View all notifications →
        </a>
      </div>
    </div>
  </div>
</div>

<script>
function notificationBell() {
  return {
    open: false,
    unreadCount: 0,
    notifications: [],
    pollInterval: null,

    async init() {
      await this.loadNotifications();
      await this.refreshUnreadCount();
      // Poll for new notifications every 30 seconds
      this.pollInterval = setInterval(async () => {
        await this.refreshUnreadCount();
      }, 30000);
    },

    toggle() {
      this.open = !this.open;
      if (this.open) {
        this.loadNotifications();
      }
    },

    async loadNotifications() {
      try {
        const res = await fetch('/api/notifications?limit=10');
        if (res.ok) {
          this.notifications = await res.json();
        }
      } catch (e) {
        console.error('Failed to load notifications', e);
      }
    },

    async refreshUnreadCount() {
      try {
        const res = await fetch('/api/notifications/unread-count');
        if (res.ok) {
          const data = await res.json();
          this.unreadCount = data.count;
        }
      } catch (e) {
        console.error('Failed to fetch unread count', e);
      }
    },

    async handleClick(notification) {
      if (!notification.isRead) {
        try {
          await fetch(`/api/notifications/${notification.id}/read`, { method: 'POST' });
          notification.isRead = true;
          this.unreadCount = Math.max(0, this.unreadCount - 1);
        } catch (e) {
          console.error('Failed to mark notification as read', e);
        }
      }
      if (notification.actionUrl) {
        window.location.href = notification.actionUrl;
        this.open = false;
      }
    },

    async markAllRead() {
      try {
        await fetch('/api/notifications/read-all', { method: 'POST' });
        this.notifications.forEach(n => n.isRead = true);
        this.unreadCount = 0;
      } catch (e) {
        console.error('Failed to mark all as read', e);
      }
    },

    formatDate(isoString) {
      if (!isoString) return '';
      const date = new Date(isoString);
      const now = new Date();
      const diffMs = now - date;
      const diffMin = Math.floor(diffMs / 60000);
      if (diffMin < 1) return 'Just now';
      if (diffMin < 60) return `${diffMin}m ago`;
      const diffHr = Math.floor(diffMin / 60);
      if (diffHr < 24) return `${diffHr}h ago`;
      return date.toLocaleDateString();
    }
  };
}
</script>

</body>
</html>
```

**Alpine.js is required** for the dropdown. Add the CDN script to your base layout `<head>` if not already present:
```html
<script defer src="https://cdn.jsdelivr.net/npm/alpinejs@3.x.x/dist/cdn.min.js"></script>
```

**To add the bell to your header**, insert this in your base layout (e.g. `templates/layout/base.html`) inside the header nav:
```html
<div th:replace="~{fragments/notification-bell :: notificationBell}"></div>
```

---

## Step 7 — Notification list page (full history)

### Controller addition

Add a `@Controller` method for the full notification list page (can be added to an existing `MemberController` or a new `NotificationPageController.kt`):

```kotlin
@GetMapping("/notifications")
fun notificationsPage(
    @RequestParam(defaultValue = "0") page: Int,
    @AuthenticationPrincipal oauth: OAuth2User?,
    session: HttpSession,
    model: Model
): String {
    val memberId = resolveMemberId(oauth, session) ?: return "redirect:/login"
    model.addAttribute("notifications", notificationService.getAll(memberId, page, 20))
    model.addAttribute("unreadCount", notificationService.getUnreadCount(memberId))
    return "member/notifications"
}
```

### Template: `templates/member/notifications.html`

A full-page notification list with:
- "Mark all read" button at the top (only shown when unreadCount > 0)
- Table or card list: title, message, type badge, date, read/unread indicator
- Unread rows highlighted (e.g. `bg-blue-50`)
- Pagination controls (previous / next page)
- Click on a row → marks as read via fetch, then follows `actionUrl` if present
- Empty state: "You have no notifications yet."

---

## Step 8 — Integration with existing modules (if installed)

Show the user how to wire `notificationService` into existing services based on what's installed.

### If `announcements` is in `installed_modules`

Add to `AnnouncementService.publish()`, after the announcement is saved:

```kotlin
// In AnnouncementService.publish(), after saving:
notificationService.notifyAll(
    type = "ANNOUNCEMENT",
    title = announcement.title,
    message = announcement.body.take(100),  // truncate for notification preview
    actionUrl = "/announcements"            // or a direct link if you have one
)
```

### If `membership-apply` is in `installed_modules`

In `ApplicationService` (or wherever approval/rejection happens):

```kotlin
// On approval:
notificationService.createNotification(
    memberId = application.memberId,
    type = "APPROVAL",
    title = "Your membership application was approved!",
    message = "Welcome to ${appName}. You can now log in as a full member.",
    actionUrl = "/member/dashboard"
)

// On rejection:
notificationService.createNotification(
    memberId = application.memberId,
    type = "APPROVAL",
    title = "Your membership application was not approved",
    message = rejectionReason,
    actionUrl = null
)
```

### If `item-submit` is in `installed_modules`

In the item review service (wherever approval/rejection happens):

```kotlin
// On approval:
notificationService.createNotification(
    memberId = submission.memberId,
    type = "SYSTEM",
    title = "Your submission \"${submission.title}\" was approved",
    actionUrl = "/member/submissions/${submission.id}"
)

// On rejection:
notificationService.createNotification(
    memberId = submission.memberId,
    type = "SYSTEM",
    title = "Your submission \"${submission.title}\" was not approved",
    message = reviewNote,
    actionUrl = "/member/submissions/${submission.id}"
)
```

---

## Step 9 — Unit tests

If `test_mode` is NOT `"token-save"`:

```kotlin
// NotificationServiceTest.kt
@SpringBootTest
@ActiveProfiles("test")
@Transactional
class NotificationServiceTest {

    @Autowired lateinit var notificationService: NotificationService
    @Autowired lateinit var notificationRepository: NotificationRepository

    @Test
    fun `createNotification persists notification and it appears as unread`() {
        val notification = notificationService.createNotification(
            memberId = 1L,
            type = "SYSTEM",
            title = "Test notification"
        )
        assertNotNull(notification.id)
        assertTrue(notification.isUnread)
        assertNull(notification.readAt)
        assertEquals(1L, notificationService.getUnreadCount(1L))
    }

    @Test
    fun `markRead updates readAt and reduces unread count`() {
        val n = notificationService.createNotification(1L, "SYSTEM", "Test")
        assertEquals(1L, notificationService.getUnreadCount(1L))

        notificationService.markRead(n.id, 1L)

        assertEquals(0L, notificationService.getUnreadCount(1L))
        val updated = notificationRepository.findById(n.id).orElseThrow()
        assertNotNull(updated.readAt)
    }

    @Test
    fun `markAllRead clears all unread for member without affecting other members`() {
        notificationService.createNotification(1L, "SYSTEM", "Notif 1")
        notificationService.createNotification(1L, "SYSTEM", "Notif 2")
        notificationService.createNotification(2L, "SYSTEM", "Other member")

        notificationService.markAllRead(1L)

        assertEquals(0L, notificationService.getUnreadCount(1L))
        assertEquals(1L, notificationService.getUnreadCount(2L))  // unaffected
    }

    @Test
    fun `getUnreadCount returns zero for member with no notifications`() {
        assertEquals(0L, notificationService.getUnreadCount(999L))
    }
}
```

---

## Step 10 — Beginner-Friendly Mode

If `beginner_friendly` is `true`, explain:

**On the polling pattern:**
> "Every 30 seconds, the browser quietly asks the server 'are there any new notifications?' using `fetch('/api/notifications/unread-count')`. This is called *polling*. It's simpler than WebSockets (which require a persistent bidirectional connection) and works well for most apps where near-real-time is good enough. The tradeoff: a user could wait up to 30 seconds to see a new notification appear."

**On the `read_at` NULL pattern:**
> "Instead of a boolean `is_read` column, we use `read_at TIMESTAMP` — if it's NULL, the notification is unread. If it has a value, the user read it at that timestamp. This gives you two things for the price of one: the read/unread status AND the exact time the user saw it. It's a common pattern called 'nullable timestamp as state'."

**On the partial index:**
> "The `CREATE INDEX ... WHERE read_at IS NULL` is a *partial index* — it only indexes unread rows. Since most notifications get read quickly, this index stays small and fast. Querying 'how many unread notifications does member 42 have?' uses this index and avoids scanning all historical notifications."

---

## Step 11 — Git commit

```bash
git add -A
git commit -m "feat: add in-app notification system with bell, badge, and REST polling"
```

---

## Step 12 — Update `installed_modules`

Add `"notifications"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
In-app notification system is ready.

Bell icon: add to your header with:
  <div th:replace="~{fragments/notification-bell :: notificationBell}"></div>

Full history page: http://localhost:8080/notifications

REST API:
  GET  /api/notifications          → recent notifications (JSON)
  GET  /api/notifications/unread-count → {"count": N}
  POST /api/notifications/{id}/read    → mark one as read
  POST /api/notifications/read-all     → mark all as read

To send a notification from any service:
  notificationService.createNotification(memberId, "SYSTEM", "Title", "Message", "/link")
  notificationService.notifyAll("ANNOUNCEMENT", "Title", "Message")
  notificationService.notifyRole("ADMIN", "SYSTEM", "Admin-only alert")
```
