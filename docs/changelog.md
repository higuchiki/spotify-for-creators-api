# Changelog

## 2026-06-17 — Fix publishOn integer requirement; document S3 audio upload path

**Source:** Live EP upload testing against ぼくらの戦略論 show (52.5 MB mp3, S3 path).

### rest-api.md / recipes.md

- **`publishOn` corrected to Unix integer** — sending an ISO 8601 string silently
  corrupts the value (API parses leading digits `"2026"` as 2026 seconds from epoch
  → `1970-01-01T00:33:46.000Z`). `wizardDraftedToPublishOn` is the opposite: it
  requires ISO 8601 and returns HTTP 400 if given an integer.
- **Audio upload routes to AWS S3, not GCS** — signed URL response for audio returns
  `{ requestUuid, signedUrl, fileKey }` instead of `{ uploadId, url, headers }`.
  process_upload and poll endpoints also use `requestUuid` for audio.
- **Poll endpoint returns 404 transiently for audio** — up to ~60 s after
  `process_upload` completes before the record is available. Retry with backoff.
- Fixed `userId` (not `stationId`) in video process_upload request body.

---

## 2026-06-12 — Engagement Tab and New Metrics Documented (Jun 2026 Research)

**Research method:** JavaScript bundle analysis — extracted and reconstructed
GraphQL ASTs from the S4C analytics microfrontend bundle
(`microfrontend-analytics-cdn.spotifycdn.com`). All operation names and
query structures are taken directly from the production bundle.

### New operations documented

**Engagement Tab (newly confirmed):**
- **`getEngagementStats`** — daily time series for consumption time
  (`SHOW_CONSUMPTION` → `ConsumptionValue.totalConsumptionHours`) and
  average consumption time
  (`SHOW_AVERAGE_CONSUMPTION_TIME` → `AverageConsumptionTimeValue.averageConsumptionSeconds`)
- **`getEngagementStatsNRT`** — period totals with
  `periodOverPeriodPercentageDiff` (source of the `+36.6%` / `+7.4%`
  dashboard badges)
- **`getShowRetention`** — weekly retention rate using
  `SHOW_RETENTION` / `AGGREGATION_TYPE_WEEKLY` →
  `TimeSeriesValue<RatioValueFloat>` (e.g. `0.537` = 53.7%)
- **`getEpisodeCompletionRates`** — per-episode completion rate for latest
  10 episodes using `EPISODE_AVERAGE_COMPLETION_RATE` /
  `WINDOW_FIRST_SEVEN_DAYS` → `PercentageValueFloat`
- **`getEpisodeTimeSeriesByMetric`** — single-episode time series for any
  `EpisodeAnalyticsMetricType`; also exposes
  `ConsumptionValue.foregroundConsumptionPercent`

**Audience Growth (three-tier core fan model):**
- **`getAudienceGrowthTimeSeries`** — daily breakdown into core fans /
  developing audience / new audience via `coreFanTimeSeries` field
  (uses string dates, not `AnalyticsWindow` enum)
- **`getAudienceGrowthMetricSummary`** — summary with `gainedCoreFans` and
  lookback period comparison via `coreFanMetricSummary`
- **`getAudienceGrowthInsights`** — behavioral comparison between core fans
  and total audience (retention %, consumption hours) via `coreFanInsight`

**Benchmark:**
- **`getBenchmarkTotal`** and **`getBenchmarkTimeSeries`** — compare show
  metrics against percentile bands (20/40/50/60/80th) using
  `SHOW_BENCHMARK_EPISODE` with `benchmarkParams`
- Documented `BenchmarkMetricType` enum (13 values) and
  `BenchmarkEpisodePool` enum (`ALL_EPISODES`, `LAST_10_EPISODES`)

**Overview Tab:**
- **`getShowStreamsAndDownloadsDaily`** — all-platforms time series with
  variable `$aggregationType` (daily / weekly / monthly)
- **`getShowTopEpisodesByMetric`** — top episodes for any
  `EpisodeAnalyticsMetricType` via `analyticsBatch` on episodes
- **`getShowTopEpisodesByImpressions`** — top episodes by impressions with
  per-source breakdown; uses new `TOP_EPISODES_BY_IMPRESSIONS_FACETED` metric
- **`getShowStreams`**, **`getShowStreamsNRT`** — Spotify-only streams daily /
  total
- **`getShowAllPlatformsStats`**, **`getShowAllPlatformsStatsNRT`** — all-time
  all-platforms totals using `SHOW_STREAMS_AND_DOWNLOADS_AVERAGE` /
  `SHOW_STREAMS_AND_DOWNLOADS`
- **`getPerformanceStats`**, **`getPerformanceStatsNRT`** — daily/total plays,
  audience, follower growth
- **`getShowMetadata`** — show metadata including all-time plays total
- **`getShowEpisodesByMetric`** — paginated episode list with per-episode metric

**Discovery Funnel (corrected):**
- Named operation **`getShowDiscoveryFunnelStats`** confirmed; corrected
  `DiscoveryFunnelStep.stepValue` field structure (union:
  `CountValueLong` for impressions/plays, `RatioValueFloat` for completion rate)

**Segmented Audience (named operations confirmed):**
- Named operations **`getShowSegmentedAudienceTotal`** and
  **`getShowSegmentedAudienceTimeSeries`** confirmed (previously documented
  as anonymous queries)

**AI Insights:**
- **`generateAnalyticsInsight`** (mutation), **`getAnalyticsInsight`** (query),
  **`submitAnalyticsInsightFeedback`** (mutation) — AI-powered analytics
  insight generation and feedback loop

### New enum values documented
- `ShowAnalyticsMetricType`: `SHOW_CONSUMPTION`, `SHOW_AVERAGE_CONSUMPTION_TIME`,
  `SHOW_RETENTION`, `SHOW_BENCHMARK_EPISODE`, `SHOW_STREAMS_AND_DOWNLOADS_AVERAGE`,
  `SHOW_STREAMS_AND_DOWNLOADS_BY_APP`, `SHOW_STREAMS_AND_DOWNLOADS_BY_DEVICE`,
  `SHOW_STREAMS_AND_DOWNLOADS_BY_GEO_ALL_PLATFORMS`,
  `TOP_EPISODES_BY_IMPRESSIONS_FACETED`, `SHOW_IMPRESSIONS_FACETED`,
  `SHOW_STREAMS_FACETED`, `SHOW_FOLLOWER_GROWTH`
- `EpisodeAnalyticsMetricType`: `EPISODE_AVERAGE_COMPLETION_RATE`,
  `EPISODE_AVERAGE_CONSUMPTION_TIME`, `EPISODE_CONSUMPTION`,
  `EPISODE_AUDIENCE_SIZE`, `EPISODE_IMPRESSIONS`, `EPISODE_IMPRESSIONS_FACETED`,
  `EPISODE_IMPRESSIONS_FUNNEL`, `EPISODE_IMPRESSIONS_TO_PLAYS_RATE`,
  `EPISODE_DISCOVERY_FUNNEL`, `EPISODE_PLAYS_FACETED`, `EPISODE_STREAMS_FACETED`,
  `EPISODE_RETENTION`, `EPISODE_SPOTIFY_PLAYS_BY_COUNTRY`
- `AnalyticsWindow`: `WINDOW_FIRST_SEVEN_DAYS`, `WINDOW_FIRST_THIRTY_DAYS`,
  `WINDOW_FIRST_SIXTY_DAYS`, `WINDOW_LAST_SIXTY_DAYS`, `WINDOW_SINCE_PUBLISHED`,
  `WINDOW_UNSPECIFIED`

### Endpoint count update
| Category | Count |
|----------|-------|
| anchor.fm REST endpoints | 25 |
| GraphQL queries | ~50 (+15 new) |
| GraphQL mutations | ~15 (+3 new AI insight mutations) |

---

## 2026-06-12 — Episode Delete Endpoint Documented

**Research method:** browser traffic capture during a live wizard
upload + delete cycle on a production show, then direct API verification.

- **`POST /v3/episodes/{ANCHOR_ID}/delete`** — documented the episode
  deletion endpoint (REST API → 1-Z). Empty `{}` body, returns `200`.
  Verified on disposable test episodes (works on drafts, scheduled, and
  published episodes). Irreversible — no trash/restore step.

## 2026-06-12 — May 2026 New Metrics Documented

**Research method:** schema introspection + live GraphQL queries against a
production show. Metrics believed to have been added in the May 2026
dashboard refresh (tied to Spotify Investor Day announcement).

- **`SHOW_SEGMENTED_AUDIENCE`** — new metric returning `SegmentedAudienceValue`
  (`totalAudience` / `newAudience` / `returningAudience`). Both
  `AGGREGATION_TYPE_TOTAL` and `AGGREGATION_TYPE_DAILY` verified. Daily mode
  returns `TimeSeriesValue` with inline `SegmentedAudienceValue` per point.
- **Key semantic finding — `newAudience` vs `returningAudience` are asymmetric:**
  `newAudience` is period-unique (daily sum = period total);
  `returningAudience` is counted every day a returning listener appears
  (daily sum >> period total — roughly 9× in a 90-day window). Verified by
  confirming `new + returning == total` holds for every day in a 90-day series.
- **`SHOW_DISCOVERY_FUNNEL`** — new metric returning `DiscoveryFunnelValue`.
  Three-step funnel: Impressions → Plays → Average Completion Rate, with
  `conversionRateToNext` and `periodOverPeriodPercentageDiff` on each step.
- **`SHOW_WINDOWED_AVERAGE_COMPLETION_RATE`** — confirmed returning
  `PercentageValueFloat` for the window-average completion rate.
- **New `Show` fields (schema-confirmed, untested):** `coreFanTimeSeries`,
  `coreFanMetricSummary`, `coreFanInsight`; `sponsorshipAnalytics`,
  `listSponsorships`; `analyticsInsight`, `getEpisodeInsights`.

See [GraphQL API → Analytics Queries](graphql.md#analytics-queries-verified).

## 2026-06-11 — Platform Change: "Plays" Redefined (30s / AMP Standard)

**Not an API research entry — a Spotify platform change that affects metric
semantics throughout this documentation.**

- Spotify redefined **Plays** as **30+ seconds listened = 1 play**, aligning
  with the AMP (Audience Measurement Principles) industry standard. Previously
  a play was counted on start.
- The S4C dashboard headline metric for all platforms switched from
  **Streams + Downloads** to **Plays + Downloads**. The old Streams + Downloads
  figure remains available but is demoted to "reference" status.
- New features shipped alongside: the **Engagement tab** (consumption time,
  retention, completion rates), **benchmarks** (percentile comparison against
  similar shows), and **audience segments** (core fans / new / returning).
  The corresponding GraphQL operations are documented in the
  2026-06-12 entry above.
- **Caution:** play-count semantics captured before this date (e.g. notes that
  `SHOW_PLAYS` matched the podcasters-API *starts* total) reflect the old
  definition and may no longer hold numerically.

## 2026-06-11 — Analytics Queries Verified

**Research method:** schema introspection + live GraphQL queries against a
production show.

- **Confirmed `/v2/graph-pq` accepts full GraphQL query strings** — Persisted
  Queries are not required, and schema introspection is enabled
- Discovered the real access path for analytics data: browser-captured
  operation names (`getShowAudienceAllPlatformsStats` etc.) are **not**
  top-level Query fields; data is accessed via
  `showByShowUri → Show.analytics(getShowAnalyticsRequest)` and
  `episodeByUri → Episode.analytics(getEpisodePlayCountRequest)`
- Documented `ShowAnalyticsMetricType` / `EpisodeAnalyticsMetricType` /
  `AggregationType` / `AnalyticsWindow` enums with verified semantics
- **Key finding — metric names are misleading:** `SHOW_PLAYS` is
  Spotify-only; the all-platforms figures (matching the dashboard
  "All platforms" view) are the `*_AND_DOWNLOADS` metrics
- Verified `SHOW_STREAMS_AND_DOWNLOADS` equals the sum of per-episode
  `EPISODE_STREAMS_AND_DOWNLOADS` (within <0.01%) — one call replaces a
  per-episode loop
- Documented daily/weekly/monthly time series and window options, the
  `ShowUri` / `EpisodeUri` scalars, and the no-argument
  `analyticsStarts { startsCount }` field (Spotify-only starts)

See [GraphQL API → Analytics Queries](graphql.md#analytics-queries-verified).

## 2026-05-25 to 2026-05-26 — Initial Research

**Research method:** Claude in Chrome MCP with JavaScript fetch/XHR
interceptors and console log capture.

### 2026-05-25 — Core API discovery

- Discovered the two-backend architecture: anchor.fm REST API and GraphQL API
- Documented authentication flow: `sp_dc` / `sp_key` → Bearer token via `spotifyconnector`
- Captured core episode endpoints:
    - **1-A**: Spotify Episode ID → Anchor numeric ID conversion
    - **1-B**: Episode overview (`GET /v3/episodes/{id}/overview`)
    - **1-C**: Episode update (`POST /v3/episodes/{id}/update`)
- Confirmed `publishOn` timezone handling (UTC), state matrix, and scheduling behaviour
- Documented known quirks: `description` HTML transformations, draft 403 behaviour

### 2026-05-26 — Extended API coverage

Additional pages explored on a test show (STATION_ID: YOUR_STATION_ID) using a
dedicated test account:

**New REST endpoints discovered:**

- **1-D**: Region endpoint
- **1-E / 1-Q**: Sponsored content status (GET / PUT)
- **1-F**: Cover art color extraction
- **1-G**: Episode form data batch
- **1-H**: Show metadata (`GET /v3/stations/{id}/metadata`) — full field list captured
- **1-I**: Partner IDs
- **1-J**: User verification
- **1-K**: Onboarding status
- **1-L**: Spotify Show ID → anchor numeric IDs (`legacyIds`) — **new key endpoint**
- **1-M**: Episode list index update
- **1-N**: Show form data
- **1-O**: Distribution settings — full field list captured
- **1-P**: User terms and contactability settings
- **1-R**: Monetisation lifecycle sync
- **1-S**: Episode creation — allocate empty episode record
- **1-T**: Show metadata update
- **1-U**: Vanity slug update
- **1-V**: Image upload pipeline (cover art + episode thumbnail)
- **1-W**: Monetisation endpoints (paywalls, wallet)
- **1-X**: Show settings page endpoints
- **1-Y**: Distribution page endpoints

**New GraphQL operations discovered:**

- Analytics: `getShowOnSpotifyStats`, `getShowOnSpotifyStatsNRT`, `getShowOverviewStatsNRT`, `getEpisodePlaysTotal`, `WebGetIndexedEpisodeList`
- Audience tab: `getShowAudienceDemographicsStats`, `getShowAudienceAllPlatformsGeoStats`
- Distribution: `getPublishedEpisodeCount`
- Episode create: `getChaptersEligibility`, `createEpisodeSmartlink`, `getEverboardingModal`, `getLatestCommentsForShow`

**Comment management — live-action capture:**

A listener account posted a comment on the test show. The creator account
then performed real UI operations: approve, reply, delete, pin, unpin,
block — all captured via the interceptor.

Confirmed mutations:
- `publishCommentByCommentUri` (approve)
- `deleteCommentByCommentUri` (delete, with `filter` field)
- `createCommentReplyByCommentUri` (creator reply)
- `markCommentAsRead` (auto-fires on approve / reply)
- `pinComment` / `unPinComment`
- `blockUserForShow`
- `updateThreadCreatorLastSeen`

Confirmed queries:
- `getPinnedCommentForEpisode`
- `GetCommentReplies`
- `getEpisodeData`

Confirmed absence:
- **Un-approve (revert to pending review)**: no such UI action or API call exists in the S4C interface.

**Media upload pipeline — confirmed:**

- Audio files use `uploadType: "default"` (not `"audio"`)
- Video files use `uploadType: "video"` with `isExtractedFromVideo: true`
- GCS ETag must be stripped of surrounding double-quotes before use in `parts`

**Known limitations noted:**

- Transcript editing mutation: unavailable on the test account (feature disabled)
- Chapter create/update/delete mutations: could not be reached (test show ineligible)

---

## Endpoint Count Summary (as of 2026-06-12)

| Category | Count |
|----------|-------|
| anchor.fm REST endpoints (1-A to 1-Z) | 26 (including sub-endpoints in 1-V, 1-W, 6-A to 6-D) |
| GraphQL queries | ~50 |
| GraphQL mutations | ~15 |
