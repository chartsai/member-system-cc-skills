---
name: springboot-stats-summary
description: >
  TRIGGER this skill when the user says "add statistics", "add activity summary", "add member stats",
  "run springboot-stats-summary", "show member activity by year", "add period-based reporting",
  "add annual summary", "add credit tracking", "add activity tracking by period", "add progress tracking",
  or anything about aggregating member activity (submissions, items completed) by time period
  with optional thresholds and visual indicators. Requires setup, scaffold, db, membership.
---

# springboot-stats-summary

Adds period-based activity aggregation and reporting. Summarizes member submissions per period
(academic year, calendar year, quarter, etc.) with configurable boundaries and optional thresholds.

---

## Dependencies

### Hard prerequisites
- `setup`, `scaffold`, `db`, `membership` must be installed

### Soft prerequisites
- `item-submit` → aggregates submissions from the submissions table (if installed)
- `app-config` → period boundaries configurable from admin UI (falls back to application.yml if absent)
- `data-export` → summary data can be included in exports (if installed)

---

## Step 0 — Read config

Read `.spring-config.json`. Extract:

```
base_package      → Java package root
app_name          → used in page titles
test_mode         → controls build verification
language          → respond in this language for ALL output (questions, explanations,
                    status messages, code comments, and user-facing copy in generated files)
                    IMPORTANT: if "zh-TW", "traditional-chinese", or "繁體中文":
                      use 繁體中文 (Traditional Chinese) throughout
                      NEVER use Simplified Chinese (简体中文) — they are different writing systems
                      Key differences: 體/体, 語/语, 資料/数据, 設定/设定, 確認/确认, 請/请
translate_terms   → whether to translate technical terms
beginner_friendly → if true, explain technical terms and decisions as you work
installed_modules → checked for item-submit, app-config, data-export
```

> **Language activation**: After reading `language` above, switch ALL your responses to that
> language immediately — including every question, status message, explanation, and all
> human-readable text in any generated files (HTML, Thymeleaf templates, SQL seed labels).
> If `language` is not set, ask: "What language should I use? (English / 繁體中文 / other)"



---

## Step 1 — Prerequisites check

Verify `installed_modules` contains `"setup"`, `"scaffold"`, `"db"`, `"membership"`.

Note which soft prerequisites are present:
- If `"item-submit"` is missing: stats will use placeholder data or be empty
- If `"app-config"` is missing: period config will come from application.yml only

---

## Beginner-Friendly Mode

If `beginner_friendly` is `true` in `.spring-config.json`, explain key concepts as you work. Examples:
- When calculating period boundaries: "We store period start/end dates in `app_config` so they can be changed without code edits. An 'academic year' might start July 1; a 'fiscal quarter' starts every 3 months."
- When writing aggregation queries: "SQL's `COUNT()`, `SUM()`, and `GROUP BY` let us summarize many rows into totals per member. We run these queries when the summary page loads, rather than pre-computing and storing results."
- When showing progress indicators: "If an annual limit is configured, we compare a member's total against it to show '4 of 5 completed'. We read the limit from `app_config` so it can vary by year."

## Step 2 — Flyway migration

Create `V_stats__create_summary_tables.sql` (next available version):

```sql
-- Period definitions for activity tracking
CREATE TABLE stat_periods (
    id          BIGSERIAL PRIMARY KEY,
    code        VARCHAR(50)  NOT NULL UNIQUE,
    label       VARCHAR(100) NOT NULL,
    start_date  DATE         NOT NULL,
    end_date    DATE         NOT NULL,
    is_current  BOOLEAN      NOT NULL DEFAULT FALSE,
    created_at  TIMESTAMPTZ  NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_stat_periods_dates ON stat_periods (start_date, end_date);

-- Pre-aggregated summary cache (computed periodically or on demand)
CREATE TABLE member_period_stats (
    id              BIGSERIAL PRIMARY KEY,
    member_id       BIGINT   NOT NULL REFERENCES members(id) ON DELETE CASCADE,
    period_code     VARCHAR(50) NOT NULL,
    total_count     INT      NOT NULL DEFAULT 0,
    approved_count  INT      NOT NULL DEFAULT 0,
    pending_count   INT      NOT NULL DEFAULT 0,
    rejected_count  INT      NOT NULL DEFAULT 0,
    threshold_met   BOOLEAN  NOT NULL DEFAULT FALSE,
    threshold_value INT,           -- Optional: the required minimum (e.g., 5 submissions)
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE (member_id, period_code)
);

CREATE INDEX idx_member_period_stats_member_id ON member_period_stats (member_id);
CREATE INDEX idx_member_period_stats_period    ON member_period_stats (period_code);
```

Seed current period (adjust for your use case — academic year example):

```sql
-- Seed current academic year (adjust dates as needed)
INSERT INTO stat_periods (code, label, start_date, end_date, is_current)
VALUES (
    'AY_' || EXTRACT(YEAR FROM CURRENT_DATE)::TEXT,
    'Academic Year ' || EXTRACT(YEAR FROM CURRENT_DATE)::TEXT || '-' || (EXTRACT(YEAR FROM CURRENT_DATE) + 1)::TEXT,
    (EXTRACT(YEAR FROM CURRENT_DATE)::TEXT || '-07-01')::DATE,
    ((EXTRACT(YEAR FROM CURRENT_DATE) + 1)::TEXT || '-06-30')::DATE,
    TRUE
) ON CONFLICT (code) DO NOTHING;
```

---

## Step 3 — `StatPeriod` and `MemberPeriodStats` entities

```java
// StatPeriod.java
package {{base_package}}.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.LocalDate;
import java.time.OffsetDateTime;

@Entity
@Table(name = "stat_periods")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class StatPeriod {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true) private String code;
    @Column(nullable = false) private String label;
    @Column(name = "start_date", nullable = false) private LocalDate startDate;
    @Column(name = "end_date", nullable = false) private LocalDate endDate;
    @Column(name = "is_current", nullable = false) private boolean current = false;
    @Column(name = "created_at", updatable = false) private OffsetDateTime createdAt;

    @PrePersist void onCreate() { createdAt = OffsetDateTime.now(); }

    public boolean contains(LocalDate date) {
        return !date.isBefore(startDate) && !date.isAfter(endDate);
    }
}
```

```java
// MemberPeriodStats.java
package {{base_package}}.entity;

import jakarta.persistence.*;
import lombok.*;
import java.time.OffsetDateTime;

@Entity
@Table(name = "member_period_stats")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class MemberPeriodStats {

    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "member_id", nullable = false)
    private Member member;

    @Column(name = "period_code", nullable = false) private String periodCode;
    @Column(name = "total_count") private int totalCount = 0;
    @Column(name = "approved_count") private int approvedCount = 0;
    @Column(name = "pending_count") private int pendingCount = 0;
    @Column(name = "rejected_count") private int rejectedCount = 0;
    @Column(name = "threshold_met") private boolean thresholdMet = false;
    @Column(name = "threshold_value") private Integer thresholdValue;
    @Column(name = "computed_at") private OffsetDateTime computedAt;
}
```

---

## Step 4 — `PeriodCalculator` — utility for period boundary logic

```java
package {{base_package}}.stats;

import {{base_package}}.entity.StatPeriod;
import {{base_package}}.repository.StatPeriodRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;
import java.time.LocalDate;
import java.util.Optional;

/**
 * Determines which period a given date falls into.
 * Supports: calendar year, academic year (configurable start month), quarterly.
 *
 * Period boundaries are stored in stat_periods table.
 * The "current" period is flagged with is_current=true.
 */
@Component
@RequiredArgsConstructor
public class PeriodCalculator {

    private final StatPeriodRepository statPeriodRepository;

    public Optional<StatPeriod> findCurrentPeriod() {
        return statPeriodRepository.findByCurrentTrue();
    }

    public Optional<StatPeriod> findPeriodForDate(LocalDate date) {
        return statPeriodRepository.findAll().stream()
            .filter(p -> p.contains(date))
            .findFirst();
    }

    /**
     * Get the period code for today.
     */
    public String getCurrentPeriodCode() {
        return findCurrentPeriod()
            .map(StatPeriod::getCode)
            .orElse("UNKNOWN");
    }

    /**
     * Determine academic year period code for a given date.
     * Academic year starts in July (month 7) by default.
     * Override startMonth via app_config: "stats.period.start_month"
     */
    public String academicYearCode(LocalDate date, int startMonth) {
        int year = date.getMonthValue() >= startMonth
            ? date.getYear()
            : date.getYear() - 1;
        return "AY_" + year;
    }
}
```

---

## Step 5 — `StatsSummaryService`

```java
package {{base_package}}.service;

import {{base_package}}.entity.*;
import {{base_package}}.repository.*;
import {{base_package}}.stats.PeriodCalculator;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.time.OffsetDateTime;
import java.util.*;

@Service
@Slf4j
@RequiredArgsConstructor
public class StatsSummaryService {

    private final MemberRepository memberRepository;
    private final StatPeriodRepository statPeriodRepository;
    private final MemberPeriodStatsRepository memberPeriodStatsRepository;
    private final PeriodCalculator periodCalculator;

    // SubmissionRepository is optional — only present if item-submit module is installed
    @org.springframework.beans.factory.annotation.Autowired(required = false)
    private SubmissionRepository submissionRepository;

    @Value("${app.stats.threshold:0}")
    private int defaultThreshold;

    /**
     * Compute stats for all members for the current period.
     * Called on demand or via scheduled job.
     */
    @Transactional
    public void computeCurrentPeriodStats() {
        Optional<StatPeriod> currentPeriod = periodCalculator.findCurrentPeriod();
        if (currentPeriod.isEmpty()) {
            log.warn("No current period defined. Stats not computed.");
            return;
        }

        StatPeriod period = currentPeriod.get();
        List<Member> activeMembers = memberRepository.findAll().stream()
            .filter(m -> m.getStatus() == Member.Status.ACTIVE)
            .toList();

        for (Member member : activeMembers) {
            computeMemberStats(member, period);
        }
        log.info("Computed stats for {} members in period: {}", activeMembers.size(), period.getCode());
    }

    private void computeMemberStats(Member member, StatPeriod period) {
        int total = 0, approved = 0, pending = 0, rejected = 0;

        if (submissionRepository != null) {
            List<Submission> submissions = submissionRepository.findByMemberIdOrderBySubmittedAtDesc(member.getId());
            List<Submission> inPeriod = submissions.stream()
                .filter(s -> {
                    if (s.getSubmittedAt() == null) return false;
                    return period.contains(s.getSubmittedAt().toLocalDate());
                }).toList();

            total    = inPeriod.size();
            approved = (int) inPeriod.stream().filter(s -> s.getStatus() == Submission.Status.APPROVED).count();
            pending  = (int) inPeriod.stream().filter(s -> s.getStatus() == Submission.Status.PENDING).count();
            rejected = (int) inPeriod.stream().filter(s -> s.getStatus() == Submission.Status.REJECTED).count();
        }

        MemberPeriodStats stats = memberPeriodStatsRepository
            .findByMemberIdAndPeriodCode(member.getId(), period.getCode())
            .orElseGet(() -> MemberPeriodStats.builder()
                .member(member)
                .periodCode(period.getCode())
                .build());

        stats.setTotalCount(total);
        stats.setApprovedCount(approved);
        stats.setPendingCount(pending);
        stats.setRejectedCount(rejected);
        stats.setThresholdValue(defaultThreshold > 0 ? defaultThreshold : null);
        stats.setThresholdMet(defaultThreshold <= 0 || approved >= defaultThreshold);
        stats.setComputedAt(OffsetDateTime.now());
        memberPeriodStatsRepository.save(stats);
    }

    public Optional<MemberPeriodStats> getMemberCurrentStats(Long memberId) {
        String periodCode = periodCalculator.getCurrentPeriodCode();
        return memberPeriodStatsRepository.findByMemberIdAndPeriodCode(memberId, periodCode);
    }

    public List<MemberPeriodStats> getAllCurrentStats() {
        String periodCode = periodCalculator.getCurrentPeriodCode();
        return memberPeriodStatsRepository.findByPeriodCodeOrderByApprovedCountDesc(periodCode);
    }

    public List<StatPeriod> getAllPeriods() {
        return statPeriodRepository.findAllByOrderByStartDateDesc();
    }

    /** Recompute stats hourly */
    @Scheduled(cron = "0 0 * * * *")
    public void scheduledRecompute() {
        computeCurrentPeriodStats();
    }
}
```

---

## Step 6 — Repositories

```java
// StatPeriodRepository.java
public interface StatPeriodRepository extends JpaRepository<StatPeriod, Long> {
    Optional<StatPeriod> findByCurrentTrue();
    List<StatPeriod> findAllByOrderByStartDateDesc();
}

// MemberPeriodStatsRepository.java
public interface MemberPeriodStatsRepository extends JpaRepository<MemberPeriodStats, Long> {
    Optional<MemberPeriodStats> findByMemberIdAndPeriodCode(Long memberId, String periodCode);
    List<MemberPeriodStats> findByPeriodCodeOrderByApprovedCountDesc(String periodCode);
    List<MemberPeriodStats> findByMemberId(Long memberId);
}
```

---

## Step 7 — Admin stats dashboard: `AdminStatsController`

```java
@Controller
@RequestMapping("/admin/stats")
@RequiredArgsConstructor
public class AdminStatsController {

    private final StatsSummaryService statsSummaryService;
    private final StatPeriodRepository statPeriodRepository;

    @GetMapping
    public String dashboard(Model model) {
        List<MemberPeriodStats> stats = statsSummaryService.getAllCurrentStats();
        long metThreshold  = stats.stream().filter(MemberPeriodStats::isThresholdMet).count();
        long total         = stats.size();

        model.addAttribute("stats", stats);
        model.addAttribute("periods", statsSummaryService.getAllPeriods());
        model.addAttribute("metCount", metThreshold);
        model.addAttribute("totalCount", total);
        model.addAttribute("pendingCount", total - metThreshold);
        return "admin/stats/dashboard";
    }

    @PostMapping("/recompute")
    public String recompute(RedirectAttributes redirectAttributes) {
        statsSummaryService.computeCurrentPeriodStats();
        redirectAttributes.addFlashAttribute("success", "Stats recomputed successfully.");
        return "redirect:/admin/stats";
    }
}
```

---

## Step 8 — Member-facing stats view

In the member dashboard controller, inject current stats:

```java
Optional<MemberPeriodStats> stats = statsSummaryService.getMemberCurrentStats(memberId);
model.addAttribute("periodStats", stats.orElse(null));
```

In dashboard template, show progress:

```html
<div th:if="${periodStats != null}" class="bg-white rounded-xl border border-gray-200 p-6 mb-6">
    <div class="flex items-center justify-between mb-4">
        <h2 class="font-semibold text-gray-700">Activity This Period</h2>
        <span th:class="${periodStats.thresholdMet} ? 'bg-green-100 text-green-700 text-xs px-3 py-1 rounded-full'
                                                    : 'bg-yellow-100 text-yellow-700 text-xs px-3 py-1 rounded-full'"
              th:text="${periodStats.thresholdMet} ? '✓ Requirement Met' : 'In Progress'">Status</span>
    </div>
    <div class="grid grid-cols-3 gap-4 text-center">
        <div>
            <div class="text-3xl font-bold text-blue-600" th:text="${periodStats.approvedCount}">0</div>
            <div class="text-sm text-gray-400">Approved</div>
        </div>
        <div>
            <div class="text-3xl font-bold text-yellow-500" th:text="${periodStats.pendingCount}">0</div>
            <div class="text-sm text-gray-400">Pending</div>
        </div>
        <div>
            <div class="text-3xl font-bold text-gray-300" th:text="${periodStats.totalCount}">0</div>
            <div class="text-sm text-gray-400">Total</div>
        </div>
    </div>
    <!-- Progress bar toward threshold -->
    <div th:if="${periodStats.thresholdValue != null}" class="mt-4">
        <div class="flex justify-between text-xs text-gray-400 mb-1">
            <span>Progress to requirement</span>
            <span th:text="${periodStats.approvedCount} + '/' + ${periodStats.thresholdValue}">0/5</span>
        </div>
        <div class="w-full bg-gray-100 rounded-full h-2">
            <div class="bg-blue-500 h-2 rounded-full transition-all"
                 th:style="'width: ' + ${T(Math).min(100, periodStats.approvedCount * 100 / periodStats.thresholdValue)} + '%'">
            </div>
        </div>
    </div>
</div>
```

---

## Step 9 — `application.yml` config

```yaml
app:
  stats:
    threshold: 0          # 0 = no threshold (optional). Set to 5 for "5 submissions required"
    period:
      start-month: 7      # Academic year starts in July. Set to 1 for calendar year.
```

---

## Step 10 — Unit tests

If `test_mode` is NOT `"token-save"`:

```java
// PeriodCalculatorTest.java
// Test: academicYearCode for July onwards → current year
// Test: academicYearCode for January → previous year (for July-start academic year)
// Test: StatPeriod.contains(date within range) → true
// Test: StatPeriod.contains(date outside range) → false

// StatsSummaryServiceTest.java
// Test: computeMemberStats with 3 approved + 1 pending → thresholdMet depends on threshold value
// Test: thresholdMet = true when threshold = 0
// Test: thresholdMet = false when approvedCount < thresholdValue
```

---

## Step 11 — Verify

Apply `test_mode`.

---

## Step 12 — Git commits

```bash
git add -A
git commit -m "feat: add period-based stats summary with member progress tracking and admin dashboard"
```

---

## Step 13 — Update `installed_modules`

Add `"stats-summary"` to `installed_modules` in `.spring-config.json`.

Tell the user:
```
Stats summary system ready.

Admin dashboard: http://localhost:8080/admin/stats
Member dashboard: stats appear automatically on the member dashboard

Configure thresholds and period start month in application.yml:
  app.stats.threshold: 5        ← require 5 approved submissions
  app.stats.period.start-month: 7  ← academic year starts in July
```
