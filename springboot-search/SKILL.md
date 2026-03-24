---
name: springboot-search
description: >
  TRIGGER this skill when the user wants to add search functionality, full-text search,
  or a search bar to their Spring Boot app. Uses PostgreSQL tsvector/tsquery for
  fast full-text search across members, content, and other entities — no Elasticsearch needed.
  Prerequisites: setup, scaffold, db must be installed.
---

# springboot-search

Adds full-text search powered by PostgreSQL's native `tsvector`/`tsquery` — no Elasticsearch
required. Search results page with relevance ranking, a header search bar, and an optional
JSON API for typeahead autocomplete.

---

## Dependencies

### Hard prerequisites
- `setup`, `scaffold`, `db` must be installed

### Soft prerequisites
- `membership` → enables member search (names, emails)
- `content` → enables article/post search
- `announcements` → enables announcement search

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
app_name          → used in page titles
base_package      → Java package root
language          → respond in this language for ALL output (questions, explanations,
                    status messages, code comments, and user-facing copy in generated files)
                    IMPORTANT: if "zh-TW", "traditional-chinese", or "繁體中文":
                      use 繁體中文 (Traditional Chinese) throughout
                      NEVER use Simplified Chinese (简体中文) — they are different writing systems
                      Key differences: 體/体, 語/语, 資料/数据, 設定/设定, 確認/确认, 請/请
beginner_friendly → if true, explain technical terms and decisions as you work
installed_modules → checked for membership, content, announcements
test_mode         → controls build verification
```

> **Language activation**: After reading `language` above, switch ALL your responses to that
> language immediately — including every question, status message, explanation, and all
> human-readable text in any generated files (HTML, Thymeleaf templates, SQL seed labels).
> If `language` is not set, ask: "What language should I use? (English / 繁體中文 / other)"

---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`, `"scaffold"`, `"db"`.

If missing → stop with an error and tell the user to run `springboot-menu` to see what to install first.

Check soft prerequisites and note which entity types will be available:
- `membership` in `installed_modules` → members search available
- `content` in `installed_modules` → article/content search available
- `announcements` in `installed_modules` → announcement search available

If none of the soft prerequisites are installed, warn the user that search will only support
custom entities, and ask them to confirm they want to continue.

---

## Step 2 — Ask what to make searchable

Ask the user which entities they want to include in search. Present only the options that are
available based on `installed_modules`:

- **Members** (names, emails) — available if `membership` installed; restricted to admins
- **Articles / Content** — available if `content` installed; visible to all
- **Announcements** — available if `announcements` installed; visible to all
- **Custom entity** — ask for table name, searchable columns, and access level

Default: select all available options (user can deselect).

---

## Step 3 — Add tsvector columns and indexes

For each selected entity, create a Flyway migration (next available version) that adds a
`search_vector` column, GIN index, and auto-update trigger.

**Example for `members` table:**

```sql
-- V{N}__add_search_to_members.sql

-- Add tsvector column
ALTER TABLE members ADD COLUMN IF NOT EXISTS search_vector tsvector;

-- Populate existing rows
UPDATE members
SET search_vector = to_tsvector('english',
    coalesce(name, '') || ' ' || coalesce(email, '')
);

-- Create GIN index (optimized for tsvector lookups)
CREATE INDEX IF NOT EXISTS idx_members_search_vector
    ON members USING GIN(search_vector);

-- Trigger function: auto-update search_vector on insert/update
CREATE OR REPLACE FUNCTION members_search_vector_update() RETURNS trigger AS $$
BEGIN
    NEW.search_vector := to_tsvector('english',
        coalesce(NEW.name, '') || ' ' || coalesce(NEW.email, '')
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS members_search_vector_trigger ON members;
CREATE TRIGGER members_search_vector_trigger
    BEFORE INSERT OR UPDATE ON members
    FOR EACH ROW EXECUTE FUNCTION members_search_vector_update();
```

**Example for a `content` / `articles` table:**

```sql
-- V{N}__add_search_to_content.sql

ALTER TABLE articles ADD COLUMN IF NOT EXISTS search_vector tsvector;

UPDATE articles
SET search_vector = to_tsvector('english',
    coalesce(title, '') || ' ' || coalesce(body, '') || ' ' || coalesce(tags, '')
);

CREATE INDEX IF NOT EXISTS idx_articles_search_vector
    ON articles USING GIN(search_vector);

CREATE OR REPLACE FUNCTION articles_search_vector_update() RETURNS trigger AS $$
BEGIN
    NEW.search_vector := to_tsvector('english',
        coalesce(NEW.title, '') || ' ' || coalesce(NEW.body, '') || ' ' || coalesce(NEW.tags, '')
    );
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS articles_search_vector_trigger ON articles;
CREATE TRIGGER articles_search_vector_trigger
    BEFORE INSERT OR UPDATE ON articles
    FOR EACH ROW EXECUTE FUNCTION articles_search_vector_update();
```

> **CJK note**: For Traditional Chinese / Japanese content, use `'simple'` dictionary instead
> of `'english'`. See Step 8 for the recommended `pg_trgm` approach.

---

## Step 4 — Update JPA entities

Add the `search_vector` field to each affected JPA entity. Since `tsvector` is a PostgreSQL-specific
type, map it as a read-only `String` — the trigger handles writes.

```kotlin
// In Member.kt (or relevant entity)
import org.hibernate.annotations.Formula

// Add as a non-insertable, non-updatable column managed by the DB trigger:
@Column(name = "search_vector", insertable = false, updatable = false)
private val searchVector: String? = null
```

Alternatively, for repositories that use native queries, no entity change is needed — the
`search_vector` column is referenced directly in SQL.

---

## Step 5 — Repository search methods

Add search methods to each affected repository. Use **native SQL** for `@@` operator and
`ts_rank` — JPQL does not support PostgreSQL-specific operators.

```kotlin
// MemberRepository.kt additions

@Query(value = """
    SELECT * FROM members
    WHERE search_vector @@ plainto_tsquery('english', :query)
    ORDER BY ts_rank(search_vector, plainto_tsquery('english', :query)) DESC
    LIMIT 20
""", nativeQuery = true)
fun searchByVector(@Param("query") query: String): List<Member>
```

```kotlin
// ArticleRepository.kt additions (if content module installed)

@Query(value = """
    SELECT * FROM articles
    WHERE search_vector @@ plainto_tsquery('english', :query)
    ORDER BY ts_rank(search_vector, plainto_tsquery('english', :query)) DESC
    LIMIT 20
""", nativeQuery = true)
fun searchByVector(@Param("query") query: String): List<Article>
```

> **`plainto_tsquery` vs `to_tsquery`**: Use `plainto_tsquery` — it safely handles arbitrary
> user input (spaces, special characters, partial words). `to_tsquery` requires strict boolean
> syntax and will throw on inputs like `"spring & boot |"`.

---

## Step 6 — `SearchService`

Create `src/main/kotlin/{{base_package}}/service/SearchService.kt`:

```kotlin
package {{base_package}}.service

import {{base_package}}.repository.MemberRepository          // if membership installed
// import {{base_package}}.repository.ArticleRepository      // if content installed
// import {{base_package}}.repository.AnnouncementRepository // if announcements installed
import lombok.RequiredArgsConstructor
import org.springframework.stereotype.Service

data class SearchResult(
    val id: Long,
    val type: String,          // "member", "article", "announcement"
    val title: String,
    val snippet: String,
    val url: String
)

data class SearchResults(
    val query: String,
    val members: List<SearchResult> = emptyList(),
    val articles: List<SearchResult> = emptyList(),
    val announcements: List<SearchResult> = emptyList()
) {
    val total: Int get() = members.size + articles.size + announcements.size
}

@Service
class SearchService(
    private val memberRepository: MemberRepository,          // if membership installed
    // private val articleRepository: ArticleRepository,     // if content installed
) {

    fun search(query: String, types: List<String> = listOf("all")): SearchResults {
        if (query.isBlank()) return SearchResults(query = query)

        val safeQuery = query.trim()
        val searchAll = types.contains("all")

        val members = if (searchAll || types.contains("member")) {
            memberRepository.searchByVector(safeQuery).map { m ->
                SearchResult(
                    id      = m.id!!,
                    type    = "member",
                    title   = m.name ?: m.email,
                    snippet = m.email,
                    url     = "/admin/members/${m.id}"
                )
            }
        } else emptyList()

        // Add similar blocks for articles, announcements when those modules are installed

        return SearchResults(
            query   = query,
            members = members
        )
    }
}
```

---

## Step 7 — `SearchController`

Create `src/main/kotlin/{{base_package}}/controller/SearchController.kt`:

```kotlin
package {{base_package}}.controller

import {{base_package}}.service.SearchService
import lombok.RequiredArgsConstructor
import org.springframework.http.ResponseEntity
import org.springframework.security.access.prepost.PreAuthorize
import org.springframework.stereotype.Controller
import org.springframework.ui.Model
import org.springframework.web.bind.annotation.*

@Controller
@RequiredArgsConstructor
class SearchController(private val searchService: SearchService) {

    /** Full search results page (Thymeleaf) */
    @GetMapping("/search")
    @PreAuthorize("isAuthenticated()")
    fun searchPage(
        @RequestParam(defaultValue = "") q: String,
        @RequestParam(defaultValue = "all") type: String,
        model: Model
    ): String {
        val results = if (q.isNotBlank()) searchService.search(q, listOf(type)) else null
        model.addAttribute("query", q)
        model.addAttribute("type", type)
        model.addAttribute("results", results)
        return "search/results"
    }

    /** JSON API for typeahead / autocomplete */
    @GetMapping("/api/search")
    @PreAuthorize("isAuthenticated()")
    @ResponseBody
    fun searchApi(
        @RequestParam(defaultValue = "") q: String,
        @RequestParam(defaultValue = "all") type: String
    ): ResponseEntity<Any> {
        if (q.length < 2) return ResponseEntity.ok(emptyMap<String, Any>())
        val results = searchService.search(q, listOf(type))
        return ResponseEntity.ok(results)
    }
}
```

Access control rules:
- `/search` and `/api/search` require authentication
- Member search results should only be shown to users with `ADMIN` or `SUPER_ADMIN` role
  (check in `SearchService.search()` using `SecurityContextHolder` or pass the current user's role)
- Content and announcement results are visible to all authenticated users

Add to `SecurityConfig`:
```kotlin
.requestMatchers("/search/**", "/api/search/**").authenticated()
```

---

## Step 8 — Search results page

Create `src/main/resources/templates/search/results.html`:

- Search bar pre-filled with current query
- "Showing N results for 'keyword'" or "No results found for 'keyword'"
- Results grouped by entity type with icons (e.g., person icon for members, document icon for articles)
- Each result card shows:
  - Title / name (bold)
  - Snippet (truncated to ~150 chars)
  - Link to detail page
- Empty state with helpful suggestion ("Try shorter keywords or check spelling")
- Use Tailwind CSS, consistent with the app's existing layout fragment

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org"
      xmlns:sec="http://www.thymeleaf.org/extras/springsecurity6"
      th:replace="~{layout/main :: page(title='Search', ~{::main})}">
<main>
  <!-- Search bar -->
  <form method="get" action="/search" class="mb-6">
    <div class="flex gap-2">
      <input type="text" name="q" th:value="${query}"
             placeholder="Search..."
             class="flex-1 border rounded-lg px-4 py-2 focus:outline-none focus:ring-2 focus:ring-blue-500" />
      <button type="submit"
              class="bg-blue-600 text-white px-6 py-2 rounded-lg hover:bg-blue-700">
        Search
      </button>
    </div>
  </form>

  <!-- Result count -->
  <p th:if="${results != null && results.total > 0}" class="text-gray-600 mb-4">
    Showing <span th:text="${results.total}">0</span>
    results for '<span th:text="${query}" class="font-medium"></span>'
  </p>

  <!-- No results -->
  <div th:if="${results != null && results.total == 0}" class="text-center py-12 text-gray-500">
    <p class="text-lg">No results found for '<span th:text="${query}"></span>'</p>
    <p class="text-sm mt-2">Try shorter keywords or check your spelling.</p>
  </div>

  <!-- Member results (admin only) -->
  <section th:if="${results != null && !results.members.isEmpty()}"
           sec:authorize="hasAnyRole('ADMIN','SUPER_ADMIN')" class="mb-8">
    <h2 class="text-lg font-semibold text-gray-700 mb-3">Members</h2>
    <div class="space-y-2">
      <a th:each="r : ${results.members}" th:href="${r.url}"
         class="block p-4 bg-white border rounded-lg hover:shadow-sm transition">
        <div class="font-medium text-gray-900" th:text="${r.title}"></div>
        <div class="text-sm text-gray-500" th:text="${r.snippet}"></div>
      </a>
    </div>
  </section>

  <!-- Article / Content results -->
  <section th:if="${results != null && !results.articles.isEmpty()}" class="mb-8">
    <h2 class="text-lg font-semibold text-gray-700 mb-3">Articles</h2>
    <div class="space-y-2">
      <a th:each="r : ${results.articles}" th:href="${r.url}"
         class="block p-4 bg-white border rounded-lg hover:shadow-sm transition">
        <div class="font-medium text-gray-900" th:text="${r.title}"></div>
        <div class="text-sm text-gray-500" th:text="${r.snippet}"></div>
      </a>
    </div>
  </section>
</main>
</html>
```

---

## Step 9 — Header search bar

Add a compact search input to the app's main navigation layout fragment
(`templates/layout/main.html` or `fragments/nav.html`):

```html
<!-- In the nav bar, add the search form -->
<form method="get" action="/search" class="flex items-center gap-1">
  <input type="text" name="q"
         placeholder="Search..."
         class="text-sm border rounded px-3 py-1.5 w-48 focus:w-64 transition-all focus:outline-none focus:ring-2 focus:ring-blue-500" />
  <button type="submit" class="text-gray-500 hover:text-blue-600 px-1">
    <!-- Search icon (Heroicons outline) -->
    <svg xmlns="http://www.w3.org/2000/svg" class="h-5 w-5" fill="none" viewBox="0 0 24 24" stroke="currentColor">
      <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
            d="M21 21l-4.35-4.35M17 11A6 6 0 1 1 5 11a6 6 0 0 1 12 0z"/>
    </svg>
  </button>
</form>
```

**Optional typeahead dropdown** (add below the form, requires Alpine.js or vanilla JS):

```html
<script>
// Typeahead with 300ms debounce
(function() {
  const input = document.querySelector('input[name="q"]');
  if (!input) return;
  let timer;
  input.addEventListener('input', function() {
    clearTimeout(timer);
    const q = this.value.trim();
    if (q.length < 2) return;
    timer = setTimeout(() => {
      fetch(`/api/search?q=${encodeURIComponent(q)}`)
        .then(r => r.json())
        .then(data => {
          // Render dropdown with data.members, data.articles, etc.
          // Implementation depends on app's JS stack
          console.log('Typeahead results:', data);
        });
    }, 300);
  });
})();
</script>
```

---

## Step 10 — CJK / Traditional Chinese support

If `language` is `zh-TW`, `traditional-chinese`, or `繁體中文`:

PostgreSQL's built-in `to_tsvector('english', ...)` does not tokenize Chinese characters —
each CJK character is treated as a single token, which produces poor results for multi-character
words and phrases.

**Recommended: `pg_trgm` (trigram similarity)**

`pg_trgm` breaks text into 3-character trigrams, which works well for CJK search and is
available by default in most PostgreSQL installations (including Cloud SQL and Supabase).

```sql
-- Enable the extension (run once, in a Flyway migration)
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Create trigram GIN indexes on text columns
CREATE INDEX IF NOT EXISTS idx_members_name_trgm
    ON members USING GIN(name gin_trgm_ops);

CREATE INDEX IF NOT EXISTS idx_members_email_trgm
    ON members USING GIN(email gin_trgm_ops);

-- For articles
CREATE INDEX IF NOT EXISTS idx_articles_title_trgm
    ON articles USING GIN(title gin_trgm_ops);
```

Query using trigram similarity or ILIKE (both use the index):

```kotlin
// Repository — native query for CJK trigram search
@Query(value = """
    SELECT * FROM members
    WHERE name ILIKE '%' || :query || '%'
       OR email ILIKE '%' || :query || '%'
    ORDER BY similarity(name, :query) DESC
    LIMIT 20
""", nativeQuery = true)
fun searchBySimilarity(@Param("query") query: String): List<Member>
```

**Three options, in order of recommendation:**

1. **`pg_trgm`** (recommended above) — works for CJK, available everywhere, no extra installation
2. **`simple` dictionary** — `to_tsvector('simple', ...)` with `ILIKE '%query%'` fallback for short queries
3. **`pg_bigm`** extension — best precision for Japanese/Chinese bigrams, but requires manual installation

Show the `pg_trgm` approach and note: "For production apps with large Chinese-language datasets,
consider `pg_bigm` for better precision."

---

## Step 11 — Unit Tests

If `test_mode` is NOT `"token-save"`:

```kotlin
// src/test/kotlin/{{base_package}}/service/SearchServiceTest.kt

package {{base_package}}.service

import {{base_package}}.repository.MemberRepository
import org.junit.jupiter.api.Test
import org.junit.jupiter.api.extension.ExtendWith
import org.mockito.InjectMocks
import org.mockito.Mock
import org.mockito.junit.jupiter.MockitoExtension
import org.assertj.core.api.Assertions.*
import org.mockito.Mockito.*

@ExtendWith(MockitoExtension::class)
class SearchServiceTest {

    @Mock lateinit var memberRepository: MemberRepository
    @InjectMocks lateinit var searchService: SearchService

    @Test
    fun `search returns matching results for valid query`() {
        val member = mock(Member::class.java).also {
            `when`(it.id).thenReturn(1L)
            `when`(it.name).thenReturn("Alice")
            `when`(it.email).thenReturn("alice@example.com")
        }
        `when`(memberRepository.searchByVector("alice")).thenReturn(listOf(member))

        val results = searchService.search("alice")

        assertThat(results.members).hasSize(1)
        assertThat(results.members[0].title).isEqualTo("Alice")
        assertThat(results.total).isEqualTo(1)
    }

    @Test
    fun `search returns empty results for no matches`() {
        `when`(memberRepository.searchByVector("zzznomatch")).thenReturn(emptyList())

        val results = searchService.search("zzznomatch")

        assertThat(results.total).isEqualTo(0)
        assertThat(results.members).isEmpty()
    }

    @Test
    fun `search with blank query returns empty results without hitting repository`() {
        val results = searchService.search("   ")

        assertThat(results.total).isEqualTo(0)
        verifyNoInteractions(memberRepository)
    }

    @Test
    fun `search query with special characters does not throw`() {
        // plainto_tsquery handles special chars safely unlike to_tsquery
        `when`(memberRepository.searchByVector("spring & boot |")).thenReturn(emptyList())

        assertThatCode { searchService.search("spring & boot |") }
            .doesNotThrowAnyException()
    }

    @Test
    fun `search respects entity type filter`() {
        val results = searchService.search("alice", listOf("article"))

        // member repository should NOT be called when type filter excludes members
        verifyNoInteractions(memberRepository)
        assertThat(results.members).isEmpty()
    }
}
```

---

## Step 12 — Beginner-Friendly Mode

If `beginner_friendly` is `true` in `.spring-config.json`, explain key concepts as you work:

- **What `tsvector` is**: "PostgreSQL pre-processes your text into a list of searchable word roots
  called 'lexemes' — e.g., 'running', 'ran', and 'runs' all become 'run'. Instead of scanning
  every row with `LIKE '%keyword%'` (slow!), it looks up a pre-built index. Much faster for
  large datasets."

- **Why GIN index**: "GIN stands for Generalized Inverted Index — it's like the index at the
  back of a book, mapping each word root to which rows contain it. Without this index, PostgreSQL
  would need to scan every row for every search query."

- **Why `plainto_tsquery` over `to_tsquery`**: "`plainto_tsquery` handles messy user input safely.
  If a user types 'spring & boot |', `to_tsquery` would throw an error because the trailing `|`
  is invalid syntax. `plainto_tsquery` cleans it up automatically — it treats everything as plain
  words joined by AND."

- **Why DB trigger**: "The trigger runs automatically every time a row is inserted or updated.
  Without it, you'd need to manually update `search_vector` in every service method that
  modifies the row — easy to forget. The trigger handles it for you."

- **Why limit to 20 results**: "Unlike Google, most in-app searches don't need 1,000 results.
  Limiting to 20 per entity type keeps the query fast and the results page useful."

---

## Step 13 — Verify

Apply `test_mode`:
- `"token-save"` → Skip build.
- `"build-only"` → `./gradlew build -x test`
- `"build-and-test"` → `./gradlew build`

---

## Step 14 — Git commit

```bash
git add -A
git commit -m "feat: add PostgreSQL full-text search with tsvector and search results page"
```

---

## Step 15 — Update `installed_modules`

Add `"search"` to `installed_modules` in `.spring-config.json`.

Tell the user:

```
Search is ready.

Search page:    http://localhost:8080/search
Typeahead API:  http://localhost:8080/api/search?q=keyword

The search_vector column is updated automatically by a DB trigger on insert/update.
No code changes needed in existing services.

Next steps:
- springboot-audit-log  → track who searched what
- springboot-membership → add more member fields to the search index
```
