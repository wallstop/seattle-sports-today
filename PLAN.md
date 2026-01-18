# Seattle Sports Today - Technical Action Plan

## Executive Summary

This document presents a comprehensive, prioritized action plan derived from an in-depth analysis of the Seattle Sports Today codebase. The analysis evaluated the project across five dimensions: **Correctness**, **Performance**, **Ease-of-Use**, **Ease-of-Understanding**, and **Maintainability**.

**Key Statistics:**
- **4 Critical Bugs** - Production crash risks requiring immediate attention
- **10 High-Priority Issues** - Significant quality, performance, and reliability concerns
- **12 Medium-Priority Issues** - Code quality and maintainability improvements
- **8 Low-Priority Issues** - Polish and documentation enhancements

---

## Table of Contents

1. [Critical Issues (Phase 1)](#phase-1-critical-issues)
2. [High-Priority Issues (Phase 2)](#phase-2-high-priority-issues)
3. [Medium-Priority Issues (Phase 3)](#phase-3-medium-priority-issues)
4. [Low-Priority Issues (Phase 4)](#phase-4-low-priority-issues)
5. [Implementation Roadmap](#implementation-roadmap)
6. [Appendix: Detailed Findings](#appendix-detailed-findings)

---

## Phase 1: Critical Issues

> **Timeline:** Immediate - Deploy within 1-2 days
> **Estimated Effort:** 4-8 hours
> **Impact:** Production crash prevention

### 1.1 Date Format Bug in main.go

| Attribute | Value |
|-----------|-------|
| **Location** | `main.go:46, 113, 115` |
| **Severity** | CRITICAL |
| **Type** | Correctness |

**Problem:** The Go time layout format `"2006-04-02"` is incorrect. In Go's reference time (`Mon Jan 2 15:04:05 MST 2006`), `04` represents minutes, not month.

**Current Code:**
```go
// Line 46 - INCORRECT
seattleToday, err = time.ParseInLocation("2006-04-02", event.Today, events.SeattleTimeZone)

// Lines 113, 115 - INCORRECT
seattleToday.Format("2006-04-02")
seattleTomorrow.Format("2006-04-02")
```

**Fix:**
```go
// Use "2006-01-02" where 01 = month, 02 = day
seattleToday, err = time.ParseInLocation("2006-01-02", event.Today, events.SeattleTimeZone)
seattleToday.Format("2006-01-02")
```

**Impact:** Without this fix, date parsing will fail or produce incorrect results, causing the application to malfunction.

---

### 1.2 Missing Error Check in Notifier

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/notifier/notifier.go:43-45` |
| **Severity** | CRITICAL |
| **Type** | Correctness |

**Problem:** The error from `secrets.GetSecretString()` is never checked. The `err` variable is immediately overwritten on the next line.

**Current Code:**
```go
notifierKey, err := secrets.GetSecretString(ctx, secretARN)  // Error not checked!

req, err := http.NewRequestWithContext(ctx, http.MethodPost,
    fmt.Sprintf("https://ntfy.sh/%s", notifierKey),  // Uses potentially empty value
    strings.NewReader(text))
```

**Fix:**
```go
notifierKey, err := secrets.GetSecretString(ctx, secretARN)
if err != nil {
    log.Error().Err(err).Msg("could not retrieve notifier secret")
    return fmt.Errorf("notifier: Notify: could not retrieve secret: %w", err)
}

req, err := http.NewRequestWithContext(ctx, http.MethodPost, ...)
```

**Impact:** Silent notification failures go undetected; notifications may be sent to invalid URLs.

---

### 1.3 Unchecked Array Bounds in UW Huskies Fetcher

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/events/uw_huskies.go:128-133` |
| **Severity** | CRITICAL |
| **Type** | Correctness |

**Problem:** Direct array access without bounds checking will cause panic if ESPN API returns unexpected data.

**Current Code:**
```go
for _, curr := range payload.Team.NextEvent {
    competition := curr.Competitions[0]      // Panics if empty
    homeTeam := competition.Competitors[0]   // Panics if empty
    awayTeam := competition.Competitors[1]   // Panics if < 2 items
    // ...
}
```

**Fix:**
```go
for _, curr := range payload.Team.NextEvent {
    if len(curr.Competitions) == 0 {
        log.Warn().Str("team", teamName).Msg("no competitions found in event")
        continue
    }
    competition := curr.Competitions[0]

    if len(competition.Competitors) < 2 {
        log.Warn().Str("team", teamName).Int("count", len(competition.Competitors)).Msg("insufficient competitors")
        continue
    }
    homeTeam := competition.Competitors[0]
    awayTeam := competition.Competitors[1]
    // ...
}
```

**Impact:** Without this fix, malformed ESPN API responses will crash the Lambda function.

---

### 1.4 Nil Pointer Dereference in Secrets Package

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/secrets/secrets.go:34` |
| **Severity** | CRITICAL |
| **Type** | Correctness |

**Problem:** Dereferencing `res.SecretString` without checking if it's nil.

**Current Code:**
```go
return *res.SecretString, nil  // Panics if SecretString is nil
```

**Fix:**
```go
if res.SecretString == nil {
    return "", fmt.Errorf("secrets: GetSecretString: secret value is nil for %s", secretName)
}
return *res.SecretString, nil
```

**Impact:** AWS Secrets Manager could theoretically return a nil SecretString, causing a panic.

---

## Phase 2: High-Priority Issues

> **Timeline:** Next sprint (1-2 weeks)
> **Estimated Effort:** 24-32 hours
> **Impact:** Performance, reliability, and developer experience

### 2.1 Sequential API Calls in Ticketmaster Fetcher

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/events/ticketmaster_api.go:250-268` |
| **Severity** | HIGH |
| **Type** | Performance |

**Problem:** The Ticketmaster fetcher queries 4 venues sequentially, with rate limiting after each call.

**Current Behavior:**
- 4 venues × ~1-2 seconds each = 4-8 seconds
- Rate limiter adds additional delays

**Recommendation:** Parallelize venue queries using errgroup with bounded concurrency.

**Implementation Example:**
```go
func (tm *ticketmasterFetcher) GetEvents(ctx context.Context, seattleToday, seattleTomorrow time.Time) ([]*Event, []*Event, error) {
    var mu sync.Mutex
    var todayEvents, tomorrowEvents []*Event

    eg, egCtx := errgroup.WithContext(ctx)

    // Limit concurrency to 2 parallel requests to respect rate limits
    sem := make(chan struct{}, 2)

    for venueName, venueID := range tm.venues {
        venueName, venueID := venueName, venueID // capture loop vars
        eg.Go(func() error {
            sem <- struct{}{} // acquire
            defer func() { <-sem }() // release

            if err := tm.limiter.Wait(egCtx); err != nil {
                return err
            }

            today, tomorrow, err := tm.getEventsForVenueID(egCtx, venueID, venueName, seattleToday, seattleTomorrow)
            if err != nil {
                return err
            }

            mu.Lock()
            todayEvents = append(todayEvents, today...)
            tomorrowEvents = append(tomorrowEvents, tomorrow...)
            mu.Unlock()
            return nil
        })
    }

    if err := eg.Wait(); err != nil {
        return nil, nil, err
    }
    return todayEvents, tomorrowEvents, nil
}
```

**Expected improvement:** 40-60% reduction in API fetch time (from ~8s to ~3-4s)

---

### 2.2 Sequential ESPN API Calls

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/events/uw_huskies.go:170-189` |
| **Severity** | HIGH |
| **Type** | Performance |

**Problem:** Three ESPN API calls are made sequentially when they could run in parallel.

**Implementation Example:**
```go
func GetUWGames(ctx context.Context, seattleToday, seattleTomorrow time.Time) ([]*Event, []*Event, error) {
    var mu sync.Mutex
    var today, tomorrow []*Event

    teams := []struct {
        url   string
        name  string
        venue string
    }{
        {huskiesFootballURL, huskiesFootballName, huskyStadium},
        {huskiesMensBasketballURL, huskiesMensBasketballName, alaskaAirlinesArena},
        {huskiesWomensBasketballURL, huskiesWomensBasketballName, alaskaAirlinesArena},
    }

    eg, egCtx := errgroup.WithContext(ctx)
    for _, team := range teams {
        team := team // capture
        eg.Go(func() error {
            t, tm, err := queryESPN(egCtx, team.url, team.name, team.venue, seattleToday, seattleTomorrow)
            if err != nil {
                return err
            }
            mu.Lock()
            today = append(today, t...)
            tomorrow = append(tomorrow, tm...)
            mu.Unlock()
            return nil
        })
    }

    if err := eg.Wait(); err != nil {
        return nil, nil, err
    }
    return today, tomorrow, nil
}
```

**Expected improvement:** 60-70% reduction in ESPN fetch time (from ~3s to ~1s)

---

### 2.3 Multiple AWS Config Loads

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/uploader/uploader.go:29-39`, `internal/events/special_events.go:21-29` |
| **Severity** | HIGH |
| **Type** | Performance (Cold Start) |

**Problem:** Multiple packages load AWS config independently in `init()` functions, adding 100-400ms to Lambda cold starts.

**Prioritization Rationale:** This is HIGH rather than CRITICAL because:
1. Cold starts affect latency but don't cause crashes or data corruption
2. Lambda is triggered once daily at 3:14 AM, so cold starts are infrequent
3. The 100-400ms overhead is acceptable for a non-real-time service
4. If this were a user-facing API with frequent cold starts, it would be CRITICAL

**Recommendation:**
- Load AWS config once in `main.go`
- Pass shared config to all packages that need it
- Expected improvement: 100-200ms cold start reduction

**Implementation:** Create a shared AWS config package:
```go
// internal/awsconfig/config.go
package awsconfig

var SharedConfig aws.Config

func Init(ctx context.Context) error {
    cfg, err := config.LoadDefaultConfig(ctx)
    if err != nil {
        return err
    }
    SharedConfig = cfg
    return nil
}
```

---

### 2.4 No API Response Caching

| Attribute | Value |
|-----------|-------|
| **Location** | Multiple files |
| **Severity** | HIGH |
| **Type** | Performance |

**Problem:** Every Lambda invocation makes fresh API calls with no caching.

**Recommendation:** Implement DynamoDB-backed caching for API responses.

**Why DynamoDB over in-memory?**
- Lambda instances can be recycled between invocations
- DynamoDB provides persistence across cold starts
- On-demand billing aligns with once-daily invocation pattern
- Enables cache reuse if Lambda is manually re-triggered

**Implementation Approach:**
```go
// internal/cache/cache.go
type CacheEntry struct {
    Key       string `dynamodbav:"pk"`
    Data      []byte `dynamodbav:"data"`
    ExpiresAt int64  `dynamodbav:"expires_at"`
}

func Get(ctx context.Context, key string) ([]byte, bool) {
    // Query DynamoDB, check expires_at > time.Now().Unix()
}

func Set(ctx context.Context, key string, data []byte, ttl time.Duration) error {
    // Put to DynamoDB with expires_at = time.Now().Add(ttl).Unix()
}

// Usage in Ticketmaster fetcher:
cacheKey := fmt.Sprintf("ticketmaster:%s:%s", venueID, date)
if cached, ok := cache.Get(ctx, cacheKey); ok {
    return parseCachedEvents(cached)
}
// ... fetch from API, then cache result
cache.Set(ctx, cacheKey, responseBytes, 30*time.Minute)
```

**Expected improvement:** 5-10s saved per cached invocation; resilience against API failures

---

### 2.5 Missing HTTP Timeouts

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/events/ticketmaster_api.go:155`, `internal/events/uw_huskies.go:91` |
| **Severity** | HIGH |
| **Type** | Reliability |

**Problem:** HTTP requests use `http.DefaultClient` with no explicit timeout. If upstream APIs hang, Lambda times out wastefully.

**Recommendation:**
```go
ctx, cancel := context.WithTimeout(ctx, 10*time.Second)
defer cancel()
```

---

### 2.6 Insufficient Test Coverage

| Attribute | Value |
|-----------|-------|
| **Location** | Entire codebase |
| **Severity** | HIGH |
| **Type** | Maintainability |

**Problem:** Only ~15% of code is tested. Critical paths like UW Huskies fetcher have no tests.

**Current Coverage:**
| Package | Coverage |
|---------|----------|
| `events` | ~30% (only Ticketmaster) |
| `renderhtml` | 0% |
| `renderjson` | 0% |
| `uploader` | 0% |
| `secrets` | 0% |
| `notifier` | 0% |
| `main` | 0% |

**Recommendation:**
- Add unit tests for all packages
- Add integration tests for event pipeline
- Add error path and edge case tests
- Target: 80%+ coverage

---

### 2.7 Hardcoded Team and Venue Configuration

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/events/ticketmaster_api.go:28-43`, `internal/events/uw_huskies.go:15-25` |
| **Severity** | HIGH |
| **Type** | Maintainability |

**Problem:** Adding a new team or venue requires code changes and redeployment.

**Hardcoded Values:**
- 4 venue IDs in `seattleVenueMap`
- 7 team attraction IDs in `seattleTeamAttractionIDs`
- 3 ESPN URLs with hardcoded team identifiers

**Recommendation:**
- Move configuration to external source (JSON file, DynamoDB, or environment variables)
- Create configuration validation at startup
- Allow runtime team enable/disable

---

### 2.8 Missing Documentation (README and Deployment)

| Attribute | Value |
|-----------|-------|
| **Location** | `README.md`, `cdk/README.md` |
| **Severity** | HIGH |
| **Type** | Ease-of-Use |

**Problem:**
- No list of required environment variables
- CDK README is generic template
- Missing deployment prerequisites and runbook
- "Exercise left to the reader" for environment setup

**Recommendation:**
- Create `.env.example` with all required variables
- Create `DEPLOYMENT.md` with step-by-step instructions
- Document AWS Secrets Manager secret creation
- Add troubleshooting guide

---

### 2.9 Missing TBA Event Return Statement

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/events/ticketmaster_api.go:68-70` |
| **Severity** | HIGH |
| **Type** | Correctness |

**Problem:** Events with TBA times are logged but not actually ignored.

**Current Code:**
```go
if e.Dates.Start.DateTBD || e.Dates.Start.DateTBA {
    log.Info().Str("name", e.Name).Msg("time is TBA")
}  // Missing: return true
```

**Fix:** Add `return true` after the log statement.

**Prioritization Rationale:** This is HIGH rather than CRITICAL because:
1. TBA events are relatively rare in the Seattle sports schedule
2. The impact is displaying events with unknown times, not a crash
3. The function continues to validate the event in other ways
4. However, it could cause user confusion (events shown without valid times)

---

### 2.10 Inconsistent Error Handling Patterns

| Attribute | Value |
|-----------|-------|
| **Location** | Multiple files |
| **Severity** | HIGH |
| **Type** | Maintainability |

**Problem:**
- Some errors logged but not returned (`ticketmaster_api.go:265`)
- Some errors silently ignored (`notifier.go:43`)
- Inconsistent error wrapping patterns

**Proposed Error Handling Strategy:**

| Error Category | Policy | Example |
|----------------|--------|---------|
| **Critical** (API failures, parsing errors) | Return immediately with context | `return fmt.Errorf("events: fetch: %w", err)` |
| **Recoverable** (single event malformed) | Log and continue processing | `log.Warn().Err(err).Msg("skipping event")` |
| **Best-effort** (notifications) | Log if failed, don't return | `if err != nil { log.Warn().Err(err).Msg("notify failed") }` |
| **Rate limiting** | Return to prevent API abuse | `return fmt.Errorf("rate limit: %w", err)` |

**Error Wrapping Pattern:**
```go
// Standard format: "package: function: action: %w"
return fmt.Errorf("events: getEventsForVenueID: query API: %w", err)

// For user-facing errors, include context:
return fmt.Errorf("events: getEventsForVenueID: failed to fetch venue %s: %w", venueName, err)
```

**Implementation Checklist:**
- [ ] Fix `notifier.go:43` - check error before continuing
- [ ] Fix `ticketmaster_api.go:265` - return rate limiter errors
- [ ] Audit all `_ = ` assignments for silent discards
- [ ] Document error handling policy in `CONTRIBUTING.md`

---

## Phase 3: Medium-Priority Issues

> **Timeline:** Future sprints (2-4 weeks)
> **Estimated Effort:** 16-24 hours
> **Impact:** Code quality and developer productivity

### 3.1 Global Singleton Clients

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/events/event.go:25`, `internal/notifier/notifier.go:38`, etc. |
| **Severity** | MEDIUM |
| **Type** | Maintainability/Testability |

**Problem:** AWS clients and HTTP clients are global singletons initialized in `init()` functions, making testing difficult.

**Affected Clients:**
| Package | Client | Line |
|---------|--------|------|
| `events` | `httpClient` | `event.go:25` |
| `events` | `dynamoClient` | `special_events.go:19` |
| `notifier` | `httpClient` | `notifier.go:38` |
| `uploader` | `s3Client`, `cfClient` | `uploader.go:25-26` |
| `secrets` | `secretsManagerClient` | `secrets.go:12` |

**Recommendation:** Implement dependency injection for all clients.

**Implementation Scope:**
1. Create interface types for each client dependency
2. Modify structs to accept clients via constructor
3. Update `main.go` to create and inject clients
4. Create mock implementations for testing

**Example Refactoring:**
```go
// Before (global singleton)
var httpClient = xray.Client(http.DefaultClient)

func GetEvents(ctx context.Context, ...) ([]*Event, error) {
    resp, err := httpClient.Do(req)
    // ...
}

// After (dependency injection)
type Fetcher struct {
    client *http.Client
}

func NewFetcher(client *http.Client) *Fetcher {
    return &Fetcher{client: xray.Client(client)}
}

func (f *Fetcher) GetEvents(ctx context.Context, ...) ([]*Event, error) {
    resp, err := f.client.Do(req)
    // ...
}

// In tests
mockClient := &http.Client{Transport: mockTransport}
fetcher := NewFetcher(mockClient)
```

**Migration Strategy:**
1. Start with `notifier` package (simplest, 1 client)
2. Then `secrets` package
3. Then `uploader` package
4. Finally `events` package (most complex, multiple clients)

---

### 3.2 Code Duplication in HTTP Error Handling

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/events/uw_huskies.go:105-113`, `internal/events/ticketmaster_api.go:180-188` |
| **Severity** | MEDIUM |
| **Type** | Maintainability |

**Problem:** Same HTTP error handling pattern repeated in multiple places.

**Recommendation:** Extract to shared utility function.

---

### 3.3 Complex Function: queryESPNAndAdd

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/events/uw_huskies.go:89-168` |
| **Severity** | MEDIUM |
| **Type** | Ease-of-Understanding |

**Problem:** 80-line function handling HTTP, parsing, validation, and mutation.

**Recommendation:** Break into smaller single-purpose functions.

---

### 3.4 Pointer-to-Slice Parameters

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/events/uw_huskies.go:89` |
| **Severity** | MEDIUM |
| **Type** | Ease-of-Understanding |

**Problem:** Functions modify slices via pointer parameters instead of returning values.

**Recommendation:** Return events instead of modifying passed slices.

---

### 3.5 Missing Package Documentation

| Attribute | Value |
|-----------|-------|
| **Location** | All packages |
| **Severity** | MEDIUM |
| **Type** | Ease-of-Understanding |

**Problem:** No package-level documentation explaining purpose and responsibilities.

**Recommendation:** Add doc comments to each package.

---

### 3.6 Unexplained Magic Constants

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/events/ticketmaster_api.go:23-43` |
| **Severity** | MEDIUM |
| **Type** | Ease-of-Understanding |

**Problem:** Ticketmaster IDs like `"KovZ917Ahkk"` have no explanation of origin or purpose.

**Recommendation:** Add comments explaining where IDs come from and how to find them.

---

### 3.7 Scattered Date Format Strings

| Attribute | Value |
|-----------|-------|
| **Location** | Multiple files |
| **Severity** | MEDIUM |
| **Type** | Maintainability |

**Problem:** Date formats like `"2006-01-02"`, `"3:04 PM"` are hardcoded in multiple places.

**Recommendation:** Centralize in a constants package.

---

### 3.8 No Configuration Validation

| Attribute | Value |
|-----------|-------|
| **Location** | `main.go` |
| **Severity** | MEDIUM |
| **Type** | Ease-of-Use |

**Problem:** Missing environment variables cause cryptic runtime errors instead of clear startup failures.

**Recommendation:** Validate all required configuration at startup.

---

### 3.9 Confusing Opponent Team Lookup Logic

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/events/ticketmaster_api.go:130-143` |
| **Severity** | MEDIUM |
| **Type** | Ease-of-Understanding |

**Problem:** Logic uses inverted condition (`team == ""`) which is confusing.

**Recommendation:** Refactor to use clearer variable names and positive conditions.

---

### 3.10 No DynamoDB Pagination

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/events/special_events.go:31-76` |
| **Severity** | MEDIUM |
| **Type** | Correctness |

**Problem:** DynamoDB query doesn't handle pagination, could fail with large result sets.

**Recommendation:** Add `Limit` and handle pagination if more items exist.

---

### 3.11 Large TicketmasterEvent Type Without Documentation

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/events/ticketmaster_types.go` |
| **Severity** | MEDIUM |
| **Type** | Ease-of-Understanding |

**Problem:** 274 lines of nested structs with no overview documentation.

**Recommendation:** Add file-level documentation explaining the structure.

---

### 3.12 Environment Variable Names Inconsistently Defined

| Attribute | Value |
|-----------|-------|
| **Location** | Multiple files |
| **Severity** | MEDIUM |
| **Type** | Maintainability |

**Problem:** Environment variable names defined as constants in scattered locations with inconsistent naming.

**Recommendation:** Create centralized `internal/config/vars.go`.

---

## Phase 4: Low-Priority Issues

> **Timeline:** As time permits
> **Estimated Effort:** 4-8 hours
> **Impact:** Code polish and minor improvements

### 4.1 Typos in Error Messages

| Location | Issue |
|----------|-------|
| `internal/events/ticketmaster_api.go:187` | "retireve" → "retrieve" |
| `internal/events/uw_huskies.go:112` | "retireve" → "retrieve" |

---

### 4.2 Hardcoded S3 Object Keys

| Attribute | Value |
|-----------|-------|
| **Location** | `internal/uploader/uploader.go:62, 65` |

**Problem:** `"index.html"` and `"todays_events.json"` hardcoded in multiple places.

---

### 4.3 Commented-Out CDK Tests

| Attribute | Value |
|-----------|-------|
| **Location** | `cdk/cdk_test.go` |

**Problem:** Entire test file is commented out.

**Recommendation:** Either enable tests or remove file.

---

### 4.4 TODO Comments Without Tracking

| Location | Comment |
|----------|---------|
| `internal/events/ticketmaster_api.go:82` | "TODO: can this be exactly 2 attractions?" |
| `internal/events/ticketmaster_api.go:228` | "TODO: clean this up" |

**Recommendation:** Create issues or address directly.

---

### 4.5 Verbose Logging

| Attribute | Value |
|-----------|-------|
| **Location** | Multiple files |

**Problem:** 51 logging statements, many at INFO level that could be DEBUG.

**Recommendation:** Audit and reduce logging verbosity.

---

### 4.6 Missing Help/Usage for CLI

| Attribute | Value |
|-----------|-------|
| **Location** | `main.go` |

**Problem:** No `--help` flag or usage documentation for CLI mode.

---

### 4.7 Undocumented Environment Variables

| Variable | Location | Purpose |
|----------|----------|---------|
| `TEST_DATE` | `main.go:138` | Test with specific date |
| `UPLOAD_ANYWAY` | `main.go:93` | Force upload in local mode |
| `INVALIDATE_ALL` | `uploader.go:77` | Full CloudFront invalidation |

---

### 4.8 Silent Notification Discards in Error Paths

| Attribute | Value |
|-----------|-------|
| **Location** | `main.go:30, 57, 80, 86, 99` |

**Problem:** Notification errors are silently discarded with `_ = notifier.Notify(...)` in 5 locations:

| Line | Context | Severity |
|------|---------|----------|
| 30 | Panic recovery handler | Low (best-effort in panic context) |
| 57 | Failed to get games | Medium (error already returned) |
| 80 | Failed to render page | Medium (error already returned) |
| 86 | Failed to render JSON | Medium (error already returned) |
| 99 | Failed to upload | Medium (error already returned) |

**Note:** Line 117-119 correctly handles notification errors (logs warning). The discards in error paths (57, 80, 86, 99) are semi-acceptable since the primary error is already being returned, but ideally should log the notification failure.

**Recommendation:** For error path notifications, wrap with logging:
```go
if notifyErr := notifier.Notify(ctx, msg, priority, emoji); notifyErr != nil {
    log.Warn().Err(notifyErr).Msg("failed to send error notification")
}
```

---

## Implementation Roadmap

```
Week 1: Phase 1 (Critical)
├── Day 1-2: Fix all 4 critical bugs
├── Day 3: Code review and testing
└── Day 4-5: Deploy and monitor

Week 2-3: Phase 2 (High Priority)
├── Add comprehensive test coverage
├── Fix performance issues (parallel API calls)
├── Create documentation (.env.example, DEPLOYMENT.md)
├── Implement error handling improvements
└── Add configuration validation

Week 4-6: Phase 3 (Medium Priority)
├── Refactor global clients to dependency injection
├── Extract code duplication
├── Simplify complex functions
├── Add package documentation
└── Centralize configuration

Ongoing: Phase 4 (Low Priority)
├── Fix typos and polish
├── Reduce logging verbosity
└── Address TODO comments
```

---

## Success Metrics

| Metric | Current | Target | Measurement Method |
|--------|---------|--------|-------------------|
| Critical Bugs | 4 | 0 | Static analysis + code review |
| Test Coverage | ~15% | 80%+ | `go test -cover ./...` |
| Cold Start Time | ~500ms | <300ms | CloudWatch Lambda INIT duration metric |
| API Fetch Time | ~15s | <8s | CloudWatch Lambda Duration metric (warm invocations) |
| Documentation | Incomplete | Comprehensive | Checklist of required docs (see below) |

### Metric Definitions

**Critical Bugs (Current: 4)**
- Counted by static analysis and code review
- Includes only bugs that can cause production crashes or data corruption
- Issues 1.1, 1.2, 1.3, 1.4 from Phase 1

**Test Coverage (Current: ~15%)**
- Measured using `go test -cover ./...`
- Current state: Only `ticketmaster_api_test.go` has meaningful tests
- Target: All packages should have ≥80% coverage

**Cold Start Time (Current: ~500ms)**
- Measured via CloudWatch Logs INIT Duration segment
- Baseline established from Lambda cold start metrics
- Expected reduction: ~200ms from consolidating AWS config loads

**API Fetch Time (Current: ~15s)**
- Measured as Lambda Duration minus INIT time on warm invocations
- Breakdown: Ticketmaster (~8s) + ESPN (~3s) + DynamoDB (~1s) + rendering (~1s)
- Target reduction via parallelization (2.1, 2.2) and caching (2.4)

**Documentation Completeness Checklist:**
- [ ] `.env.example` with all environment variables
- [ ] `DEPLOYMENT.md` with step-by-step deployment guide
- [ ] Updated `README.md` with quick start guide
- [ ] Package-level godoc comments for all packages
- [ ] Inline comments for complex logic (Ticketmaster IDs, date handling)

---

## Appendix: Detailed Findings

### A. Files Modified by Phase

**Phase 1:**
- `main.go` (lines 46, 113, 115)
- `internal/notifier/notifier.go` (lines 43-45)
- `internal/events/uw_huskies.go` (lines 128-133)
- `internal/secrets/secrets.go` (line 34)

**Phase 2:**
- `internal/events/ticketmaster_api.go` (lines 68-70, 250-268)
- `internal/events/uw_huskies.go` (lines 170-189)
- `internal/events/event.go`
- `README.md`
- New: `DEPLOYMENT.md`, `.env.example`
- New: Test files for all packages

**Phase 3:**
- All `internal/` packages (dependency injection)
- `internal/events/ticketmaster_types.go` (documentation)
- New: `internal/config/` package

### B. External Dependencies

| Dependency | Version | Purpose |
|------------|---------|---------|
| `aws-lambda-go` | v1.47.0 | Lambda handler |
| `aws-sdk-go-v2` | v1.36.3 | AWS services |
| `zerolog` | v1.34.0 | Structured logging |
| `testify` | v1.10.0 | Testing utilities |

### C. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Critical bug causes production crash | High | High | Fix in Phase 1 |
| API rate limiting issues | Medium | Medium | Add caching, respect limits |
| Adding new team breaks existing | Low | Medium | Add configuration system |
| Cold start timeout | Low | Low | Optimize AWS config loading |

---

*Generated: 2026-01-18*
*Analysis Methodology: Multi-dimensional code review covering Correctness, Performance, Ease-of-Use, Ease-of-Understanding, and Maintainability*
