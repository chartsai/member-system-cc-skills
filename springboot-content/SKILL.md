---
name: springboot-content
description: >
  TRIGGER this skill when the user wants to add a blog, articles, CMS, content management,
  rich text editor, or a posts system to their Spring Boot app.
  Creates an Article entity with Flyway migration, admin create/edit UI with Quill.js
  rich text editor, public article list and detail pages, categories, tags, slugs, and SEO meta tags.
  Prerequisites: setup, scaffold, db, membership must be installed.
---

# springboot-content

Adds a full CMS / blog system: articles with categories, tags, and slugs;
admin create/edit UI with a Quill.js rich text editor; public article list and
detail pages with SEO meta tags; optional search and notification integrations.

---

## Dependencies

### Hard prerequisites
- `setup`, `scaffold`, `db`, `membership` must be installed

### Soft prerequisites
- `i18n` → localized content support
- `search` → makes articles searchable via full-text search
- `notifications` → notifies members when a new article is published

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
app_name          → application name
base_package      → Java package root
language          → respond in this language for ALL output (questions, explanations,
                    status messages, code comments, and user-facing copy in generated files)
                    IMPORTANT: if "zh-TW", "traditional-chinese", or "繁體中文":
                      use 繁體中文 (Traditional Chinese) throughout
                      NEVER use Simplified Chinese (简体中文) — they are different writing systems
                      Key differences: 體/体, 語/语, 資料/数据, 設定/设定, 確認/确认, 請/请
beginner_friendly → if true, explain technical terms and decisions as you work
installed_modules → must include "setup", "scaffold", "db", "membership"
                    "i18n"          → enables localized content
                    "search"        → enables full-text article search
                    "notifications" → enables new-article notifications
test_mode         → controls build verification
```

> **Language activation**: After reading `language` above, switch ALL your responses to that
> language immediately — including every question, status message, explanation, and all
> human-readable text in any generated files (HTML, Thymeleaf templates, SQL seed labels).
> If `language` is not set, ask: "What language should I use? (English / 繁體中文 / other)"

---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`, `"scaffold"`, `"db"`, `"membership"`.

If any are missing → print which skills to run first, then run `springboot-menu`.

Check soft prerequisites and note which optional features will be enabled:
- `search` present → will add `search_vector` column and index articles
- `notifications` present → will call `notificationService` on publish
- `i18n` present → will note that article body may contain locale-specific content

---

## Step 2 — Ask scope questions

Ask the user (in the configured language):

1. "What will this content be used for? (blog / knowledge base / announcements / general articles)"
2. "Should articles be publicly visible (no login required) or members-only?"
3. "Do you want categories, tags, or both?"
4. "Do you want a comments section?" (suggest: no — that is a separate module and can be added later)

Save their answers as:
- `content_purpose` (blog / knowledge-base / announcements / articles)
- `public_access` (true = public, false = members-only)
- `use_categories` (boolean)
- `use_tags` (boolean)

---

## Step 3 — Flyway migration: articles tables

Use the next available Flyway version number (check existing migrations to find it).

```sql
CREATE TABLE articles (
    id                  BIGSERIAL PRIMARY KEY,
    title               VARCHAR(500) NOT NULL,
    slug                VARCHAR(500) NOT NULL UNIQUE,       -- URL-friendly version of title
    summary             TEXT,                               -- short excerpt for list view
    body                TEXT NOT NULL,                      -- HTML from rich text editor
    status              VARCHAR(20) NOT NULL DEFAULT 'DRAFT',  -- DRAFT, PUBLISHED, ARCHIVED
    author_id           BIGINT REFERENCES members(id),
    published_at        TIMESTAMP,
    featured_image_url  VARCHAR(1000),
    view_count          INT NOT NULL DEFAULT 0,
    created_at          TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE categories (
    id          BIGSERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL UNIQUE,
    slug        VARCHAR(100) NOT NULL UNIQUE,
    description TEXT,
    created_at  TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE TABLE article_categories (
    article_id  BIGINT REFERENCES articles(id) ON DELETE CASCADE,
    category_id BIGINT REFERENCES categories(id) ON DELETE CASCADE,
    PRIMARY KEY (article_id, category_id)
);

CREATE TABLE tags (
    id   BIGSERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL UNIQUE,
    slug VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE article_tags (
    article_id BIGINT REFERENCES articles(id) ON DELETE CASCADE,
    tag_id     BIGINT REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (article_id, tag_id)
);

CREATE INDEX idx_articles_status       ON articles(status);
CREATE INDEX idx_articles_slug         ON articles(slug);
CREATE INDEX idx_articles_published_at ON articles(published_at DESC);
```

If user chose not to use categories, omit the `categories` and `article_categories` tables.
If user chose not to use tags, omit the `tags` and `article_tags` tables.

---

## Step 4 — Entities and Repositories

### `Article.kt`

```kotlin
package {{base_package}}.entity

import jakarta.persistence.*
import java.time.LocalDateTime

@Entity
@Table(name = "articles")
class Article(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(nullable = false, length = 500)
    var title: String,

    @Column(nullable = false, length = 500, unique = true)
    var slug: String,

    @Column(columnDefinition = "TEXT")
    var summary: String? = null,

    @Column(nullable = false, columnDefinition = "TEXT")
    var body: String,

    @Column(nullable = false)
    @Enumerated(EnumType.STRING)
    var status: ArticleStatus = ArticleStatus.DRAFT,

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "author_id")
    var author: Member? = null,

    @Column(name = "published_at")
    var publishedAt: LocalDateTime? = null,

    @Column(name = "featured_image_url", length = 1000)
    var featuredImageUrl: String? = null,

    @Column(name = "view_count", nullable = false)
    var viewCount: Int = 0,

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "article_categories",
        joinColumns = [JoinColumn(name = "article_id")],
        inverseJoinColumns = [JoinColumn(name = "category_id")]
    )
    var categories: MutableSet<Category> = mutableSetOf(),

    @ManyToMany(fetch = FetchType.LAZY)
    @JoinTable(
        name = "article_tags",
        joinColumns = [JoinColumn(name = "article_id")],
        inverseJoinColumns = [JoinColumn(name = "tag_id")]
    )
    var tags: MutableSet<Tag> = mutableSetOf(),

    @Column(name = "created_at", nullable = false, updatable = false)
    val createdAt: LocalDateTime = LocalDateTime.now(),

    @Column(name = "updated_at", nullable = false)
    var updatedAt: LocalDateTime = LocalDateTime.now()
) {
    @PreUpdate
    fun onUpdate() { updatedAt = LocalDateTime.now() }

    fun readTimeMinutes(): Int {
        val words = body.split(Regex("\\s+")).size
        return maxOf(1, words / 200)
    }

    enum class ArticleStatus { DRAFT, PUBLISHED, ARCHIVED }
}
```

### `Category.kt`

```kotlin
package {{base_package}}.entity

import jakarta.persistence.*
import java.time.LocalDateTime

@Entity
@Table(name = "categories")
class Category(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(nullable = false, unique = true, length = 100)
    var name: String,

    @Column(nullable = false, unique = true, length = 100)
    var slug: String,

    @Column(columnDefinition = "TEXT")
    var description: String? = null,

    @Column(name = "created_at", nullable = false, updatable = false)
    val createdAt: LocalDateTime = LocalDateTime.now()
)
```

### `Tag.kt`

```kotlin
package {{base_package}}.entity

import jakarta.persistence.*

@Entity
@Table(name = "tags")
class Tag(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(nullable = false, unique = true, length = 100)
    var name: String,

    @Column(nullable = false, unique = true, length = 100)
    var slug: String
)
```

### `ArticleRepository.kt`

```kotlin
package {{base_package}}.repository

import {{base_package}}.entity.Article
import org.springframework.data.domain.Page
import org.springframework.data.domain.Pageable
import org.springframework.data.jpa.repository.JpaRepository
import org.springframework.data.jpa.repository.Modifying
import org.springframework.data.jpa.repository.Query
import org.springframework.data.repository.query.Param
import java.util.Optional

interface ArticleRepository : JpaRepository<Article, Long> {
    fun findBySlug(slug: String): Optional<Article>
    fun findByStatusOrderByPublishedAtDesc(status: Article.ArticleStatus, pageable: Pageable): Page<Article>
    fun findByAuthorIdOrderByCreatedAtDesc(authorId: Long, pageable: Pageable): Page<Article>
    fun existsBySlug(slug: String): Boolean

    @Query("SELECT a FROM Article a JOIN a.categories c WHERE c.slug = :categorySlug AND a.status = 'PUBLISHED' ORDER BY a.publishedAt DESC")
    fun findPublishedByCategorySlug(@Param("categorySlug") categorySlug: String, pageable: Pageable): Page<Article>

    @Query("SELECT a FROM Article a JOIN a.tags t WHERE t.slug = :tagSlug AND a.status = 'PUBLISHED' ORDER BY a.publishedAt DESC")
    fun findPublishedByTagSlug(@Param("tagSlug") tagSlug: String, pageable: Pageable): Page<Article>

    @Modifying
    @Query("UPDATE Article a SET a.viewCount = a.viewCount + 1 WHERE a.id = :id")
    fun incrementViewCount(@Param("id") id: Long)
}
```

### `CategoryRepository.kt`

```kotlin
package {{base_package}}.repository

import {{base_package}}.entity.Category
import org.springframework.data.jpa.repository.JpaRepository
import java.util.Optional

interface CategoryRepository : JpaRepository<Category, Long> {
    fun findBySlug(slug: String): Optional<Category>
}
```

### `TagRepository.kt`

```kotlin
package {{base_package}}.repository

import {{base_package}}.entity.Tag
import org.springframework.data.jpa.repository.JpaRepository
import java.util.Optional

interface TagRepository : JpaRepository<Tag, Long> {
    fun findBySlug(slug: String): Optional<Tag>
    fun findBySlugIn(slugs: Collection<String>): List<Tag>
    fun findByName(name: String): Optional<Tag>
}
```

---

## Step 5 — `ArticleService`

```kotlin
package {{base_package}}.service

import {{base_package}}.entity.*
import {{base_package}}.repository.*
import org.springframework.data.domain.Page
import org.springframework.data.domain.Pageable
import org.springframework.stereotype.Service
import org.springframework.transaction.annotation.Transactional
import java.text.Normalizer
import java.time.LocalDateTime

@Service
class ArticleService(
    private val articleRepository: ArticleRepository,
    private val categoryRepository: CategoryRepository,
    private val tagRepository: TagRepository,
    private val memberRepository: MemberRepository
) {

    @Transactional
    fun createDraft(
        title: String,
        body: String,
        summary: String? = null,
        authorId: Long,
        categoryIds: List<Long> = emptyList(),
        tagNames: List<String> = emptyList()
    ): Article {
        val author = memberRepository.findById(authorId)
            .orElseThrow { IllegalArgumentException("Author not found: $authorId") }

        val slug = generateSlug(title)
        val categories = categoryRepository.findAllById(categoryIds).toMutableSet()
        val tags = tagNames.map { findOrCreateTag(it) }.toMutableSet()

        return articleRepository.save(
            Article(
                title = title,
                slug = slug,
                summary = summary,
                body = body,
                author = author,
                categories = categories,
                tags = tags
            )
        )
    }

    @Transactional
    fun publish(articleId: Long, authorId: Long): Article {
        val article = findById(articleId)
        article.status = Article.ArticleStatus.PUBLISHED
        article.publishedAt = LocalDateTime.now()
        return articleRepository.save(article)
    }

    @Transactional
    fun unpublish(articleId: Long): Article {
        val article = findById(articleId)
        article.status = Article.ArticleStatus.DRAFT
        article.publishedAt = null
        return articleRepository.save(article)
    }

    @Transactional
    fun update(
        articleId: Long,
        title: String,
        body: String,
        summary: String? = null,
        featuredImageUrl: String? = null,
        categoryIds: List<Long> = emptyList(),
        tagNames: List<String> = emptyList()
    ): Article {
        val article = findById(articleId)
        article.title = title
        article.body = body
        article.summary = summary
        article.featuredImageUrl = featuredImageUrl
        article.categories = categoryRepository.findAllById(categoryIds).toMutableSet()
        article.tags = tagNames.map { findOrCreateTag(it) }.toMutableSet()
        return articleRepository.save(article)
    }

    @Transactional
    fun delete(articleId: Long) {
        val article = findById(articleId)
        article.status = Article.ArticleStatus.ARCHIVED
        articleRepository.save(article)
    }

    fun findPublished(pageable: Pageable): Page<Article> =
        articleRepository.findByStatusOrderByPublishedAtDesc(Article.ArticleStatus.PUBLISHED, pageable)

    fun findAll(pageable: Pageable): Page<Article> =
        articleRepository.findAll(pageable)

    fun findBySlug(slug: String): Article? =
        articleRepository.findBySlug(slug).orElse(null)

    fun findById(id: Long): Article =
        articleRepository.findById(id)
            .orElseThrow { IllegalArgumentException("Article not found: $id") }

    @Transactional
    fun incrementViewCount(articleId: Long) =
        articleRepository.incrementViewCount(articleId)

    fun findPublishedByCategory(categorySlug: String, pageable: Pageable): Page<Article> =
        articleRepository.findPublishedByCategorySlug(categorySlug, pageable)

    fun findPublishedByTag(tagSlug: String, pageable: Pageable): Page<Article> =
        articleRepository.findPublishedByTagSlug(tagSlug, pageable)

    fun generateSlug(title: String): String {
        val base = Normalizer.normalize(title, Normalizer.Form.NFD)
            .replace(Regex("[^\\p{ASCII}]"), "")
            .lowercase()
            .replace(Regex("[^a-z0-9\\s-]"), "")
            .trim()
            .replace(Regex("\\s+"), "-")
            .replace(Regex("-+"), "-")
            .take(490)
            .ifEmpty { "article" }

        if (!articleRepository.existsBySlug(base)) return base

        var suffix = 2
        while (articleRepository.existsBySlug("$base-$suffix")) suffix++
        return "$base-$suffix"
    }

    private fun findOrCreateTag(name: String): Tag {
        val slug = name.lowercase().trim().replace(Regex("\\s+"), "-")
        return tagRepository.findByName(name).orElseGet {
            tagRepository.save(Tag(name = name, slug = slug))
        }
    }
}
```

### Notifications integration (if `notifications` is in `installed_modules`)

In `publish()`, after saving, add:
```kotlin
notificationService.notifyAll(
    "NEW_ARTICLE",
    article.title,
    article.summary ?: "",
    "/articles/${article.slug}"
)
```

Inject `NotificationService` optionally:
```kotlin
@org.springframework.beans.factory.annotation.Autowired(required = false)
private val notificationService: NotificationService? = null
```

---

## Step 6 — Controllers

### Admin `ArticleController` (`/admin/articles`)

```kotlin
package {{base_package}}.controller.admin

import {{base_package}}.entity.Article
import {{base_package}}.repository.CategoryRepository
import {{base_package}}.service.ArticleService
import jakarta.servlet.http.HttpSession
import org.springframework.data.domain.PageRequest
import org.springframework.data.domain.Sort
import org.springframework.security.core.annotation.AuthenticationPrincipal
import org.springframework.security.oauth2.core.user.OAuth2User
import org.springframework.stereotype.Controller
import org.springframework.ui.Model
import org.springframework.web.bind.annotation.*
import org.springframework.web.servlet.mvc.support.RedirectAttributes

@Controller
@RequestMapping("/admin/articles")
class AdminArticleController(
    private val articleService: ArticleService,
    private val categoryRepository: CategoryRepository
) {

    @GetMapping
    fun list(@RequestParam(defaultValue = "0") page: Int, model: Model): String {
        model.addAttribute("articles",
            articleService.findAll(PageRequest.of(page, 20, Sort.by("createdAt").descending())))
        return "admin/articles/list"
    }

    @GetMapping("/new")
    fun newForm(model: Model): String {
        model.addAttribute("categories", categoryRepository.findAll())
        model.addAttribute("statuses", Article.ArticleStatus.values())
        return "admin/articles/form"
    }

    @PostMapping
    fun create(
        @RequestParam title: String,
        @RequestParam body: String,
        @RequestParam(required = false) summary: String?,
        @RequestParam(required = false) categoryIds: List<Long>?,
        @RequestParam(required = false) tagNames: String?,
        @AuthenticationPrincipal oauth: OAuth2User?,
        session: HttpSession,
        redirectAttributes: RedirectAttributes
    ): String {
        return try {
            val authorId = resolveAuthorId(oauth, session)
            val tags = tagNames?.split(",")?.map { it.trim() }?.filter { it.isNotBlank() } ?: emptyList()
            articleService.createDraft(title, body, summary, authorId, categoryIds ?: emptyList(), tags)
            redirectAttributes.addFlashAttribute("success", "Article saved as draft.")
            "redirect:/admin/articles"
        } catch (e: Exception) {
            redirectAttributes.addFlashAttribute("error", e.message)
            "redirect:/admin/articles/new"
        }
    }

    @GetMapping("/{id}/edit")
    fun editForm(@PathVariable id: Long, model: Model): String {
        model.addAttribute("article", articleService.findById(id))
        model.addAttribute("categories", categoryRepository.findAll())
        return "admin/articles/form"
    }

    @PostMapping("/{id}")
    fun update(
        @PathVariable id: Long,
        @RequestParam title: String,
        @RequestParam body: String,
        @RequestParam(required = false) summary: String?,
        @RequestParam(required = false) featuredImageUrl: String?,
        @RequestParam(required = false) categoryIds: List<Long>?,
        @RequestParam(required = false) tagNames: String?,
        redirectAttributes: RedirectAttributes
    ): String {
        return try {
            val tags = tagNames?.split(",")?.map { it.trim() }?.filter { it.isNotBlank() } ?: emptyList()
            articleService.update(id, title, body, summary, featuredImageUrl, categoryIds ?: emptyList(), tags)
            redirectAttributes.addFlashAttribute("success", "Article updated.")
            "redirect:/admin/articles"
        } catch (e: Exception) {
            redirectAttributes.addFlashAttribute("error", e.message)
            "redirect:/admin/articles/$id/edit"
        }
    }

    @PostMapping("/{id}/publish")
    fun publish(
        @PathVariable id: Long,
        @AuthenticationPrincipal oauth: OAuth2User?,
        session: HttpSession,
        redirectAttributes: RedirectAttributes
    ): String {
        return try {
            val authorId = resolveAuthorId(oauth, session)
            articleService.publish(id, authorId)
            redirectAttributes.addFlashAttribute("success", "Article published.")
            "redirect:/admin/articles"
        } catch (e: Exception) {
            redirectAttributes.addFlashAttribute("error", e.message)
            "redirect:/admin/articles"
        }
    }

    @PostMapping("/{id}/unpublish")
    fun unpublish(@PathVariable id: Long, redirectAttributes: RedirectAttributes): String {
        articleService.unpublish(id)
        redirectAttributes.addFlashAttribute("success", "Article moved back to draft.")
        return "redirect:/admin/articles"
    }

    @PostMapping("/{id}/delete")
    fun delete(@PathVariable id: Long, redirectAttributes: RedirectAttributes): String {
        articleService.delete(id)
        redirectAttributes.addFlashAttribute("success", "Article archived.")
        return "redirect:/admin/articles"
    }

    private fun resolveAuthorId(oauth: OAuth2User?, session: HttpSession): Long =
        oauth?.getAttribute<Long>("id")
            ?: session.getAttribute("memberId") as? Long
            ?: throw SecurityException("Not authenticated")
}
```

### Public `ArticleController` (`/articles`)

```kotlin
package {{base_package}}.controller

import {{base_package}}.service.ArticleService
import {{base_package}}.repository.CategoryRepository
import {{base_package}}.repository.TagRepository
import org.springframework.data.domain.PageRequest
import org.springframework.data.domain.Sort
import org.springframework.stereotype.Controller
import org.springframework.ui.Model
import org.springframework.web.bind.annotation.*

@Controller
@RequestMapping("/articles")
class ArticleController(
    private val articleService: ArticleService,
    private val categoryRepository: CategoryRepository,
    private val tagRepository: TagRepository
) {

    @GetMapping
    fun list(@RequestParam(defaultValue = "0") page: Int, model: Model): String {
        model.addAttribute("articles",
            articleService.findPublished(PageRequest.of(page, 10, Sort.by("publishedAt").descending())))
        model.addAttribute("categories", categoryRepository.findAll())
        return "articles/list"
    }

    @GetMapping("/{slug}")
    fun detail(@PathVariable slug: String, model: Model): String {
        val article = articleService.findBySlug(slug)
            ?: return "redirect:/articles"
        articleService.incrementViewCount(article.id)
        model.addAttribute("article", article)
        model.addAttribute("relatedArticles",
            articleService.findPublished(PageRequest.of(0, 3)).content
                .filter { it.id != article.id }
                .take(3))
        return "articles/detail"
    }

    @GetMapping("/category/{slug}")
    fun byCategory(@PathVariable slug: String, @RequestParam(defaultValue = "0") page: Int, model: Model): String {
        model.addAttribute("articles",
            articleService.findPublishedByCategory(slug, PageRequest.of(page, 10)))
        model.addAttribute("currentCategory", slug)
        model.addAttribute("categories", categoryRepository.findAll())
        return "articles/list"
    }

    @GetMapping("/tag/{slug}")
    fun byTag(@PathVariable slug: String, @RequestParam(defaultValue = "0") page: Int, model: Model): String {
        model.addAttribute("articles",
            articleService.findPublishedByTag(slug, PageRequest.of(page, 10)))
        model.addAttribute("currentTag", slug)
        return "articles/list"
    }
}
```

Restrict routes based on user's choice from Step 2:
- Public: no security restriction on `/articles/**`
- Members-only: add `.requestMatchers("/articles/**").authenticated()` to `SecurityConfig`

Restrict admin routes: `.requestMatchers("/admin/articles/**").hasAnyRole("ADMIN", "SUPER_ADMIN")`

---

## Step 7 — Thymeleaf templates

### `templates/admin/articles/list.html`

Table of all articles:
- Columns: Title, Status (badge), Author, Published At, View Count, Actions
- Status badges: PUBLISHED=green, DRAFT=yellow, ARCHIVED=gray
- Actions per row: Edit, Publish (if DRAFT), Unpublish (if PUBLISHED), Archive
- "New Article" button at top
- Pagination controls

### `templates/admin/articles/form.html`

Create/edit form with Quill.js rich text editor:

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title th:text="${article} ? 'Edit Article' : 'New Article'">Article</title>
    <!-- Quill CSS -->
    <link href="https://cdn.quilljs.com/1.3.7/quill.snow.css" rel="stylesheet">
</head>
<body>
<form th:action="${article} ? @{/admin/articles/{id}(id=${article.id})} : @{/admin/articles}"
      method="post">

    <input type="text" name="title" th:value="${article?.title}" placeholder="Title" required>

    <textarea name="summary" th:text="${article?.summary}" placeholder="Short summary (optional)"></textarea>

    <!-- Quill editor container -->
    <div id="editor" style="height: 400px;" th:utext="${article?.body}"></div>
    <!-- Hidden field synced before submit -->
    <input type="hidden" name="body" id="bodyInput" th:value="${article?.body}">

    <!-- Categories (checkboxes) -->
    <div th:each="cat : ${categories}">
        <input type="checkbox" name="categoryIds" th:value="${cat.id}"
               th:checked="${article?.categories?.contains(cat)}">
        <label th:text="${cat.name}"></label>
    </div>

    <!-- Tags (comma-separated) -->
    <input type="text" name="tagNames"
           th:value="${article?.tags?.collect { it.name }?.join(', ')}"
           placeholder="tag1, tag2, tag3">

    <input type="text" name="featuredImageUrl" th:value="${article?.featuredImageUrl}"
           placeholder="Featured image URL (optional)">

    <button type="submit">Save Draft</button>
</form>

<!-- Quill JS -->
<script src="https://cdn.quilljs.com/1.3.7/quill.min.js"></script>
<script>
  const quill = new Quill('#editor', {
    theme: 'snow',
    modules: {
      toolbar: [
        [{ 'header': [1, 2, 3, false] }],
        ['bold', 'italic', 'underline', 'strike'],
        [{ 'list': 'ordered'}, { 'list': 'bullet' }],
        ['blockquote', 'code-block'],
        ['link', 'image'],
        ['clean']
      ]
    }
  });

  // Pre-fill existing content when editing
  const existingBody = document.getElementById('bodyInput').value;
  if (existingBody) {
    quill.clipboard.dangerouslyPasteHTML(existingBody);
  }

  // Sync to hidden input before form submit
  document.querySelector('form').addEventListener('submit', () => {
    document.getElementById('bodyInput').value = quill.root.innerHTML;
  });
</script>
</body>
</html>
```

### `templates/articles/list.html`

Public article list with:
- Article cards: title, summary, category badges, author name, date, estimated read time
- Category filter sidebar or tabs (if categories enabled)
- Pagination controls (10 per page)
- Link to `/articles/{slug}` on each card

### `templates/articles/detail.html`

Full article detail page with SEO meta tags in `<head>`:

```html
<!-- SEO / Open Graph / Twitter Card meta tags -->
<meta property="og:title"       th:content="${article.title}">
<meta property="og:description" th:content="${article.summary}">
<meta property="og:url"         th:content="@{/articles/{slug}(slug=${article.slug})}">
<meta property="og:type"        content="article">
<meta property="og:image"       th:content="${article.featuredImageUrl}"
                                th:if="${article.featuredImageUrl}">
<meta name="twitter:card"       content="summary_large_image">
<meta name="description"        th:content="${article.summary}">
```

Article body section:
```html
<!-- Author, date, categories, tags, view count, read time -->
<h1 th:text="${article.title}">Title</h1>
<p>By <span th:text="${article.author?.name}">Author</span>
   · <span th:text="${#temporals.format(article.publishedAt, 'MMM d, yyyy')}">Date</span>
   · <span th:text="${article.readTimeMinutes()} + ' min read'">Read time</span>
   · <span th:text="${article.viewCount} + ' views'">Views</span>
</p>

<!-- Category badges -->
<span th:each="cat : ${article.categories}">
    <a th:href="@{/articles/category/{slug}(slug=${cat.slug})}" th:text="${cat.name}">Category</a>
</span>

<!-- Article body — use th:utext to render HTML from Quill -->
<div class="prose" th:utext="${article.body}">Body</div>

<!-- Tag links -->
<span th:each="tag : ${article.tags}">
    <a th:href="@{/articles/tag/{slug}(slug=${tag.slug})}" th:text="'#' + ${tag.name}">tag</a>
</span>

<!-- Related articles -->
<div th:each="related : ${relatedArticles}">
    <a th:href="@{/articles/{slug}(slug=${related.slug})}" th:text="${related.title}">Related</a>
</div>
```

---

## Step 8 — Search integration (if installed)

If `search` is in `installed_modules`:

Create a separate Flyway migration to add a `search_vector` column:
```sql
ALTER TABLE articles ADD COLUMN search_vector tsvector
    GENERATED ALWAYS AS (
        to_tsvector('english', coalesce(title, '') || ' ' || coalesce(summary, '') || ' ' || coalesce(body, ''))
    ) STORED;

CREATE INDEX idx_articles_search ON articles USING GIN(search_vector);
```

Update `ArticleService` to include articles in search results by implementing the project's `Searchable` interface or registering with `SearchService`.

---

## Step 9 — Unit Tests

If `test_mode` is NOT `"token-save"`:

```kotlin
package {{base_package}}.service

import {{base_package}}.entity.Article
import {{base_package}}.repository.ArticleRepository
import {{base_package}}.repository.CategoryRepository
import {{base_package}}.repository.MemberRepository
import {{base_package}}.repository.TagRepository
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.extension.ExtendWith
import org.mockito.InjectMocks
import org.mockito.Mock
import org.mockito.Mockito.*
import org.mockito.junit.jupiter.MockitoExtension
import org.assertj.core.api.Assertions.*
import java.util.Optional

@ExtendWith(MockitoExtension::class)
class ArticleServiceTest {

    @Mock lateinit var articleRepository: ArticleRepository
    @Mock lateinit var categoryRepository: CategoryRepository
    @Mock lateinit var tagRepository: TagRepository
    @Mock lateinit var memberRepository: MemberRepository
    @InjectMocks lateinit var articleService: ArticleService

    @Test
    fun `createDraft creates article with DRAFT status`() {
        val member = mock({{base_package}}.entity.Member::class.java)
        `when`(memberRepository.findById(1L)).thenReturn(Optional.of(member))
        `when`(categoryRepository.findAllById(emptyList())).thenReturn(emptyList())
        `when`(articleRepository.existsBySlug(any())).thenReturn(false)
        `when`(articleRepository.save(any())).thenAnswer { it.arguments[0] }

        val article = articleService.createDraft("Hello World", "<p>Body</p>", null, 1L)

        assertThat(article.status).isEqualTo(Article.ArticleStatus.DRAFT)
        assertThat(article.publishedAt).isNull()
    }

    @Test
    fun `publish sets status to PUBLISHED and sets published_at`() {
        val article = Article(id = 1L, title = "Test", slug = "test", body = "<p>Body</p>")
        `when`(articleRepository.findById(1L)).thenReturn(Optional.of(article))
        `when`(articleRepository.save(any())).thenAnswer { it.arguments[0] }

        val published = articleService.publish(1L, 99L)

        assertThat(published.status).isEqualTo(Article.ArticleStatus.PUBLISHED)
        assertThat(published.publishedAt).isNotNull()
    }

    @Test
    fun `generateSlug converts title to URL-safe string`() {
        `when`(articleRepository.existsBySlug(any())).thenReturn(false)

        assertThat(articleService.generateSlug("Hello World!")).isEqualTo("hello-world")
        assertThat(articleService.generateSlug("Spring Boot 3.0")).isEqualTo("spring-boot-30")
    }

    @Test
    fun `findPublished excludes DRAFT and ARCHIVED articles`() {
        // Verify repository is called with PUBLISHED status only
        val pageable = org.springframework.data.domain.PageRequest.of(0, 10)
        `when`(articleRepository.findByStatusOrderByPublishedAtDesc(
            Article.ArticleStatus.PUBLISHED, pageable))
            .thenReturn(org.springframework.data.domain.Page.empty())

        articleService.findPublished(pageable)

        verify(articleRepository).findByStatusOrderByPublishedAtDesc(Article.ArticleStatus.PUBLISHED, pageable)
        verify(articleRepository, never()).findAll(pageable)
    }

    @Test
    fun `slug uniqueness - duplicate title gets numeric suffix`() {
        `when`(articleRepository.existsBySlug("spring-boot")).thenReturn(true)
        `when`(articleRepository.existsBySlug("spring-boot-2")).thenReturn(false)

        val slug = articleService.generateSlug("Spring Boot")

        assertThat(slug).isEqualTo("spring-boot-2")
    }
}
```

---

## Step 10 — Beginner-Friendly Mode

If `beginner_friendly` is true, explain:

- **What a slug is**: "A slug is the URL-friendly version of your article title. 'My First Blog Post' becomes `my-first-blog-post`. This makes your URLs readable and SEO-friendly. We ensure slugs are unique by appending a number if a duplicate exists."

- **What Open Graph is**: "These meta tags tell social networks (Facebook, LINE, Twitter) what image and description to show when someone shares your article link. Without them, social networks guess and often show the wrong thing."

- **Why store HTML in body**: "Quill.js produces HTML output. We store it as-is and render it with `th:utext` (unescaped text). This is safe for trusted editors (admins). If untrusted users can submit content, add server-side HTML sanitization (e.g., jsoup's `Safelist`) to prevent XSS attacks."

- **What `@Modifying` does**: "The `@Modifying` annotation tells Spring Data JPA that this query changes data, not just reads it. Without it, `UPDATE` and `DELETE` queries will fail."

---

## Step 11 — Verify

Apply `test_mode`:
- `"token-save"` → Skip.
- `"build-only"` → `./gradlew build -x test`.
- `"build-and-test"` → `./gradlew build`.

---

## Step 12 — Git commit

```bash
git add -A
git commit -m "feat: add CMS content system with articles, categories, tags, and Quill.js editor"
```

---

## Step 13 — Update `installed_modules`

Add `"content"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
Content system is ready.

Admin:  http://localhost:8080/admin/articles     ← create, edit, publish articles
Public: http://localhost:8080/articles           ← public article list
        http://localhost:8080/articles/{slug}    ← individual article

Quill.js rich text editor is loaded from CDN (no npm required).

Next steps:
- springboot-file-upload → enable featured image uploads
- springboot-search      → make articles full-text searchable
```
