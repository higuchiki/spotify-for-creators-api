# GraphQL API Reference

**Endpoint:** `POST https://creators-graph.spotify.com/v2/graph-pq`

The GraphQL API powers analytics, comments, transcripts, chapters, and most
new Spotify for Creators features. It uses the same Bearer token as the
anchor.fm REST API.

---

## Request Format

```json
{
  "operationName": "getShowData",
  "variables": { "showUri": "spotify:show:YOUR_SHOW_ID" },
  "query": "query getShowData($showUri: String!) { ... }"
}
```

!!! note "Persisted Queries"
    The S4C web app uses **Persisted Queries (PQ)** — the `/v2/graph-pq`
    path is the indicator. In browser traffic, the `query` field is an empty
    string because the server looks up the query body by `operationName`
    from a server-side registry.

    When calling from Python, include the full GraphQL query string in the
    `query` field. **Confirmed (2026-06-11):** the server accepts full query
    strings, and schema introspection is enabled — see
    [Analytics Queries](#analytics-queries-verified) below.

---

## Common Headers

```python
headers = {
    "Authorization": f"Bearer {bearer}",
    "Content-Type": "application/json",
    "Accept": "application/json",
}
```

---

## Operations Reference

### Show-Level Queries

Called when the show analytics page loads.

| Operation | Type | Description |
|-----------|------|-------------|
| `getShowData` | query | Show basic info |
| `getShowTopEpisodes` | query | Top episode rankings |
| `getShowTopClips` | query | Top clip rankings |
| `getShowAudienceAllPlatformsStats` | query | All-platforms listening stats |
| `getShowAudienceAllPlatformsGeoStats` | query | Geographic listening stats |
| `getShowAudienceDemographicsStats` | query | Audience demographics |
| `getShowAudienceDiscoveryStats` | query | Discovery source stats |
| `getShowImpressionsSourcesStats` | query | Impressions by source |
| `getShowImpressionsTrendStats` | query | Impressions trend over time |
| `getShowSmartlinksStats` | query | Smart link stats |
| `getShowChaptersOptIn` | query | Chapter feature opt-in state |
| `getShowPreviewsOptIn` | query | Preview feature opt-in state |
| `getShowTranscriptsOptIn` | query | Transcript feature opt-in state |

### Analytics — Show Overview Tab

| Operation | Type | Description |
|-----------|------|-------------|
| `getShowOnSpotifyStats` | query | Spotify stats for the show (fires on date range change) |
| `getShowOnSpotifyStatsNRT` | query | Near real-time Spotify stats |
| `getShowOverviewStatsNRT` | query | Near real-time overall show stats |
| `getPublishedEpisodeCount` | query | Count of published episodes |
| `getShowMetadata` | query | Show metadata + all-time plays total (added Jun 2026) |
| `getShowAllPlatformsStats` | query | All-platforms streams+downloads average (added Jun 2026) |
| `getShowAllPlatformsStatsNRT` | query | All-time streams+downloads total (added Jun 2026) |
| `getShowStreams` | query | Daily Spotify streams time series (added Jun 2026) |
| `getShowStreamsNRT` | query | Spotify streams total for period (added Jun 2026) |
| `getShowStreamsAndDownloadsDaily` | query | All-platforms streams+downloads daily/weekly/monthly (added Jun 2026) |
| `getPerformanceStats` | query | Daily plays + audience + follower time series (added Jun 2026) |
| `getPerformanceStatsNRT` | query | Total plays + audience + follower counts with period comparison (added Jun 2026) |
| `getShowTopEpisodesByImpressions` | query | Top episodes ranked by impressions with source breakdown (added Jun 2026) |
| `getShowTopEpisodesByMetric` | query | Top episodes for any EpisodeAnalyticsMetricType (added Jun 2026) |
| `getShowEpisodesByMetric` | query | Paginated episode list with per-episode metric (added Jun 2026) |
| `hasVideoEpisodes` | query | Whether the show has video episodes (added Jun 2026) |
| `getPermissionsForShow` | query | Creator permissions for the show (added Jun 2026) |

### Analytics — Engagement Tab (New: Jun 2026)

| Operation | Type | Description |
|-----------|------|-------------|
| `getEngagementStats` | query | Consumption time, avg consumption time, comments, follower growth (daily time series) |
| `getEngagementStatsNRT` | query | Consumption time + avg consumption time totals with period comparison |
| `getEpisodeCompletionRates` | query | Per-episode completion rate for the 10 latest episodes (7-day window) |
| `getShowRetention` | query | Week-over-week retention rate as a weekly time series |
| `getEpisodeTimeSeriesByMetric` | query | Single-episode time series for any EpisodeAnalyticsMetricType |

### Analytics — Audience Tab

| Operation | Type | Description |
|-----------|------|-------------|
| `getShowAudienceDemographicsStats` | query | Age / gender breakdown |
| `getShowAudienceAllPlatformsGeoStats` | query | Geographic breakdown across platforms |
| `getShowAudienceAllPlatformsStats` | query | Cross-platform stats |
| `getShowSegmentedAudienceTotal` | query | New vs returning listener totals (May 2026) |
| `getShowSegmentedAudienceTimeSeries` | query | New vs returning listener daily time series (May 2026) |
| `getAudienceGrowthTimeSeries` | query | Core fan / developing / new audience daily time series (Jun 2026) |
| `getAudienceGrowthMetricSummary` | query | Core fan metric summary with period comparison (Jun 2026) |
| `getAudienceGrowthInsights` | query | Core fan insights: retention and consumption vs total audience (Jun 2026) |

### Analytics — Discovery Tab (New: Jun 2026)

| Operation | Type | Description |
|-----------|------|-------------|
| `getShowDiscoveryFunnelStats` | query | Impressions → Plays → Avg Completion funnel with conversion rates (May 2026) |

### Analytics — Benchmark (New: Jun 2026)

| Operation | Type | Description |
|-----------|------|-------------|
| `getBenchmarkTotal` | query | Show vs benchmark percentiles (total, any BenchmarkMetricType) |
| `getBenchmarkTimeSeries` | query | Show vs benchmark percentiles over time |

### Analytics — AI Insights (New: Jun 2026)

| Operation | Type | Description |
|-----------|------|-------------|
| `generateAnalyticsInsight` | mutation | Trigger AI insight generation |
| `getAnalyticsInsight` | query | Fetch a generated AI insight by ID |
| `submitAnalyticsInsightFeedback` | mutation | Submit thumbs-up/down feedback on an insight |

### Episode-Level Queries

Called from the episode detail and analytics pages.

| Operation | Type | Description |
|-----------|------|-------------|
| `getEpisodeThumbnail` | query | Episode thumbnail URL |
| `getEpisodeChapters` | query | Chapter data |
| `getEpisodeChaptersOptIn` | query | Chapter feature opt-in state |
| `getEpisodePreviews` | query | Preview clip data |
| `getEpisodeTranscript` | query | Transcript text |
| `getEpisodeTranscriptAvailability` | query | Whether a transcript is available |
| `getEpisodeTranscriptsOptIn` | query | Transcript feature opt-in state |
| `getGuestsByEpisodeUri` | query | Guest information |
| `getTopicsByEpisodeUri` | query | Topic tags |
| `getCrossContentLinksByEpisodeUri` | query | Cross-content links |
| `getPollForEpisode` | query | Poll data |
| `getCommentsForEpisode` | query | Episode comments |
| `getCommentsOnForEpisode` | query | Whether comments are enabled |
| `getBlockedUsersForShow` | query | List of blocked users |
| `getEpisodePlaysTotal` | query | Total play count for an episode |
| `getFeatureEligibility` | query | Feature eligibility check (from episode detail page) |

### Episode List Page

| Operation | Type | Description |
|-----------|------|-------------|
| `WebGetIndexedEpisodeList` | query | Indexed episode list |
| `getEpisodesForShow` | query | Episode list for a show (used on the poll page) |

**`getEpisodesForShow` variables example:**
```json
{
  "showUri": "spotify:show:YOUR_SHOW_ID",
  "pageSize": 15,
  "pageToken": ""
}
```

### Episode Create / Settings Page

| Operation | Type | Description |
|-----------|------|-------------|
| `getChaptersEligibility` | query | Whether chapters can be used |
| `createEpisodeSmartlink` | mutation | Create an episode smart link |
| `getEverboardingModal` | query | Everboarding modal state |
| `getLatestCommentsForShow` | query | Latest comments for the show |

### Comment Page Queries

| Operation | Type | Description |
|-----------|------|-------------|
| `getCommentsAcrossEpisodes` | query | All comments across all episodes |
| `getEpisodesWithSearch` | query | Episode search (used for comment filters) |

---

## Analytics Queries (Verified)

> Verified 2026-06-11 by schema introspection and live queries against a
> production show.

### Architecture: analytics are nested fields, not top-level operations

The operation names captured from browser traffic (e.g.
`getShowAudienceAllPlatformsStats`) are **not top-level Query fields** on
`/v2/graph-pq` — sending them as ad-hoc queries returns a `ValidationError`.
Analytics data is accessed through a nested structure instead:

- Show level: `showByShowUri` → `Show.analytics(getShowAnalyticsRequest: …)`
- Episode level: `episodeByUri` → `Episode.analytics(getEpisodePlayCountRequest: …)`

!!! tip "Introspection is enabled — Persisted Queries are not required"
    `/v2/graph-pq` accepts full GraphQL query strings, and standard schema
    introspection works. You can discover the complete schema (including all
    metric enums below) with a normal introspection query.

### Show-Level Analytics

```graphql
query {
  showByShowUri(getShowByShowUriRequest: { showUri: "spotify:show:YOUR_SHOW_ID" }) {
    analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_STREAMS_AND_DOWNLOADS,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: WINDOW_ALL_TIME
    }) {
      type
      startDate
      endDate
      followerCount
      analyticsValue {
        analyticsValue {
          __typename
          ... on SingleValueLong { value }
          ... on AudienceValue { totalAudienceSize foregroundAudienceSize foregroundAudiencePercent }
          ... on TimeSeriesValue {
            points {
              date
              value {
                ... on CountValueLong { value }
                ... on RatioValueFloat { value }
              }
            }
          }
        }
      }
    }
  }
}
```

`showUri` is a `ShowUri` scalar — pass the string `"spotify:show:YOUR_SHOW_ID"`.

#### `ShowAnalyticsMetricType` (selected values)

| Enum value | What it actually measures |
|-----------|---------------------------|
| `SHOW_PLAYS` | **Spotify-only** plays (matches the Spotify-only starts total from the podcasters API) |
| `SHOW_STREAMS` | **Spotify-only** streams (60s+ plays) |
| `SHOW_STREAMS_AND_DOWNLOADS` | **All platforms** streams + downloads — matches the dashboard "All platforms" figure |
| `SHOW_PLAYS_AND_DOWNLOADS` | All platforms plays + downloads (larger than streams + downloads) |
| `SHOW_ALL_PLATFORMS_LISTENERS` | All-platforms unique listeners |
| `SHOW_DOWNLOAD_LISTENERS` | Download listeners |
| `SHOW_OFF_PLATFORM_DOWNLOADS` | Off-platform downloads (RSS apps etc.) |
| `SHOW_LISTENERS` | Unique listeners (Spotify-scoped) |
| `SHOW_FOLLOWERS` | Follower count |
| `AUDIENCE_SIZE` | Audience size (returns `AudienceValue`) |

!!! warning "Metric names are misleading"
    Despite the generic name, `SHOW_PLAYS` matched the **Spotify-only**
    starts total from the podcasters API in our tests — it is *not* an
    all-platforms number. The true all-platforms figures are the
    `*_AND_DOWNLOADS` metrics. For our test show, the all-platforms total
    (`SHOW_STREAMS_AND_DOWNLOADS`) was roughly **1.6×** the Spotify-only
    total.

!!! info "Plays redefined on 2026-06-11 (30s / AMP standard)"
    The tests above predate Spotify's 2026-06-11 platform change, which
    redefined a play as **30+ seconds listened** (AMP industry standard) and
    switched the dashboard headline metric to **Plays + Downloads**
    (Streams + Downloads is now "reference only"). Numerical relationships
    described above (e.g. `SHOW_PLAYS` ≒ starts total) may no longer hold
    under the new definition — see the [Changelog](changelog.md).

!!! tip "One call replaces a per-episode loop"
    The sum of `EPISODE_STREAMS_AND_DOWNLOADS` across all episodes equals
    `SHOW_STREAMS_AND_DOWNLOADS` (matched within <0.01% in our tests). If
    you need the show-level all-platforms total, one show-level call is
    enough — no need to loop over episodes.

#### `AggregationType`

| Value | Description |
|-------|-------------|
| `AGGREGATION_TYPE_TOTAL` | Total over the window |
| `AGGREGATION_TYPE_DAILY` | Daily time series |
| `AGGREGATION_TYPE_WEEKLY` | Weekly time series |
| `AGGREGATION_TYPE_MONTHLY` | Monthly time series |

#### `AnalyticsWindow`

| Value | Description |
|-------|-------------|
| `WINDOW_ALL_TIME` | All time |
| `WINDOW_LAST_SEVEN_DAYS` | Last 7 days |
| `WINDOW_LAST_THIRTY_DAYS` | Last 30 days |
| `WINDOW_LAST_NINETY_DAYS` | Last 90 days |
| `WINDOW_YEAR_TO_DATE` | Year to date |
| `WINDOW_CUSTOM` | Custom range (pass `customDateRange: { startDate, endDate }`) |

### Episode-Level Analytics

```graphql
query {
  episodeByUri(getEpisodeRequest: { episodeUri: "spotify:episode:YOUR_EPISODE_ID" }) {
    title
    analytics(getEpisodePlayCountRequest: {
      episodeAnalyticsMetricType: EPISODE_STREAMS_AND_DOWNLOADS,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: WINDOW_ALL_TIME
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on SingleValueLong { value }
        }
      }
    }
    analyticsStarts {
      startsCount  # Spotify-only starts (no arguments)
    }
  }
}
```

`episodeUri` is an `EpisodeUri` scalar — pass `"spotify:episode:YOUR_EPISODE_ID"`.

#### `EpisodeAnalyticsMetricType` (selected values)

| Enum value | What it actually measures |
|-----------|---------------------------|
| `EPISODE_PLAYS` | Plays (Spotify-scoped in our tests) |
| `EPISODE_STREAMS` | Streams, 60s+ (Spotify-only) |
| `EPISODE_LISTENERS` | Unique listeners |
| `EPISODE_ALL_PLATFORMS_LISTENERS` | All-platforms unique listeners |
| `EPISODE_STREAMS_AND_DOWNLOADS` | **All platforms** streams + downloads |
| `EPISODE_PLAYS_AND_DOWNLOADS` | All platforms plays + downloads |
| `EPISODE_OFF_PLATFORM_DOWNLOADS` | Off-platform downloads |
| `EPISODE_DOWNLOAD_LISTENERS` | Download listeners |
| `EPISODE_PERFORMANCE` | Audience retention |

For a recent episode of our test show, `EPISODE_STREAMS_AND_DOWNLOADS` was
roughly **2.5×** the Spotify-only starts — RSS-distributed platforms (Apple
Podcasts etc.) can account for the majority of early plays.

---

## Comment Queries

### Get Comments for an Episode

```graphql
query getCommentsForEpisode {
  getCommentsForEpisode(
    episodeUri: String!
    primaryFilters: [String!]!
    commentTypesFilters: [String!]!
    secondaryFilters: [String!]!
    repliesFilter: [String!]!
    pageSize: Int!
  )
}
```

**`primaryFilters` values:**

| Value | Meaning |
|-------|---------|
| `LIST_COMMENT_PRIMARY_FILTER_PUBLISHED` | Published comments |
| `LIST_COMMENT_PRIMARY_FILTER_NEEDS_REVIEW` | Pending review comments |

**`commentTypesFilters` values:**

| Value | Meaning |
|-------|---------|
| `LIST_COMMENT_TYPE_FILTER_ROOT` | Root (top-level) comments |
| `LIST_COMMENT_TYPE_FILTER_REPLY` | Reply comments |

**Variables example (fetch all comments):**
```json
{
  "episodeUri": "spotify:episode:1xouj0WrH2klavXKzDWZbq",
  "primaryFilters": [
    "LIST_COMMENT_PRIMARY_FILTER_PUBLISHED",
    "LIST_COMMENT_PRIMARY_FILTER_NEEDS_REVIEW"
  ],
  "commentTypesFilters": [
    "LIST_COMMENT_TYPE_FILTER_ROOT",
    "LIST_COMMENT_TYPE_FILTER_REPLY"
  ],
  "secondaryFilters": [],
  "repliesFilter": [
    "LIST_COMMENT_PRIMARY_FILTER_PUBLISHED",
    "LIST_COMMENT_PRIMARY_FILTER_NEEDS_REVIEW"
  ],
  "pageSize": 15
}
```

### Get Comments for a Show (Cross-episode) — Verified 2026-06-23

> **Important:** The operation name `getLatestCommentsForShow` appears in browser
> network traffic, but it is a **Persisted Query alias** — it does **not** exist as a
> top-level field in the `/v2/graph-pq` schema. The actual path is
> `showByShowUri` → `Show.showComments(…)`, the same entry point used for analytics.

```graphql
query getShowComments {
  showByShowUri(getShowByShowUriRequest: { showUri: "spotify:show:YOUR_SHOW_ID" }) {
    showComments(
      primaryFilters: [String!]!
      commentTypesFilters: [String!]!
      secondaryFilters: [String!]!
      repliesFilter: [String!]!
      pageSize: Int!
    ) {
      comments {
        commentUri
        textContent
        episodeUri
        episodeTitle
        createdAt
        status
      }
      pageToken
    }
  }
}
```

> **Warning:** Including `LIST_COMMENT_PRIMARY_FILTER_NEEDS_REVIEW` in
> `primaryFilters` causes a `DataFetchingException`. Use
> `LIST_COMMENT_PRIMARY_FILTER_PUBLISHED` only.

**Variables example (published root comments, up to 20):**
```json
{
  "primaryFilters": ["LIST_COMMENT_PRIMARY_FILTER_PUBLISHED"],
  "commentTypesFilters": ["LIST_COMMENT_TYPE_FILTER_ROOT"],
  "secondaryFilters": [],
  "repliesFilter": ["LIST_COMMENT_PRIMARY_FILTER_PUBLISHED"],
  "pageSize": 20
}
```

This query fetches comments across **all episodes of the show** in a single
request — useful for building a comment digest before each recording session.

---

### Get Comment Replies

```graphql
query GetCommentReplies {
  GetCommentReplies(
    episodeUri: String!
    primaryFilters: [String!]!
    parentCommentUri: String!
    pageSize: Int!
  )
}
```

**Variables example:**
```json
{
  "episodeUri": "spotify:episode:1xouj0WrH2klavXKzDWZbq",
  "primaryFilters": [
    "LIST_COMMENT_PRIMARY_FILTER_NEEDS_REVIEW",
    "LIST_COMMENT_PRIMARY_FILTER_PUBLISHED"
  ],
  "parentCommentUri": "spotify:comment:7cXkaBSy1gocZ0hAtS9WsJ",
  "pageSize": 15
}
```

### Get Pinned Comment

```graphql
query getPinnedCommentForEpisode {
  getPinnedCommentForEpisode(
    episodeUri: String!
    primaryFilters: [String!]!
    commentTypesFilters: [String!]!
    pageSize: Int!
    pageToken: String!
  )
}
```

**Variables example:**
```json
{
  "episodeUri": "spotify:episode:0QeJ1berqX4UL20FYKGsZT",
  "primaryFilters": ["LIST_COMMENT_PRIMARY_FILTER_PUBLISHED"],
  "commentTypesFilters": ["LIST_COMMENT_TYPE_FILTER_ROOT"],
  "pageSize": 3,
  "pageToken": ""
}
```

---

## Comment Settings Mutations

### Enable / Disable Comments

```graphql
mutation updateIsCommentsEnabledForShowByShowUri {
  updateIsCommentsEnabledForShowByShowUri(
    isEnabled: Boolean!
    showUri: String!
  )
}
```

```json
{
  "isEnabled": true,
  "showUri": "spotify:show:YOUR_SHOW_ID"
}
```

### Set Moderation Policy

```graphql
mutation updateModerationPolicy {
  updateModerationPolicy(
    showUri: String!
    moderationSignal: String!
    moderationPolicy: String!
  )
}
```

| `moderationSignal` | Always use `"SENSITIVITY_OVERALL"` |
|--------------------|-------------------------------------|

**`moderationPolicy` values:**

| Value | UI label | Meaning |
|-------|----------|---------|
| `NO_FILTERING` | Low | No comments are held for review |
| `SOME_FILTERING` | Standard | Potentially inappropriate comments are held |
| `MAXIMUM_FILTERING` | High | All comments are held for review |

---

## Comment Management Mutations

All comment management mutations were confirmed from live browser traffic
captured on **2026-05-26** using a test show.

### Approve a Comment

```graphql
mutation publishCommentByCommentUri {
  publishCommentByCommentUri(commentUri: String!)
}
```

```json
{ "commentUri": "spotify:comment:7cXkaBSy1gocZ0hAtS9WsJ" }
```

!!! note
    Approving a comment also triggers `markCommentAsRead` automatically
    (two requests are sent in sequence).

### Delete a Comment

```graphql
mutation deleteCommentByCommentUri {
  deleteCommentByCommentUri(commentUri: String!, filter: String!)
}
```

**`filter` values:**

| Value | Use |
|-------|-----|
| `DELETE_COMMENT_FILTER_PUBLISHED` | Delete a published comment |
| `DELETE_COMMENT_FILTER_NEEDS_REVIEW` | Delete a pending-review comment (inferred, not confirmed) |

```json
{
  "commentUri": "spotify:comment:7cXkaBSy1gocZ0hAtS9WsJ",
  "filter": "DELETE_COMMENT_FILTER_PUBLISHED"
}
```

### Reply to a Comment (as Creator)

```graphql
mutation createCommentReplyByCommentUri {
  createCommentReplyByCommentUri(
    parentCommentUri: String!
    replyStr: String!
  )
}
```

```json
{
  "parentCommentUri": "spotify:comment:7cXkaBSy1gocZ0hAtS9WsJ",
  "replyStr": "Thank you for listening!"
}
```

!!! note
    Replies are sent from the thread view (`?thread={COMMENT_ID}` URL
    parameter). After a reply, `GetCommentReplies` fires automatically to
    refresh the thread.

### Mark a Comment as Read

```graphql
mutation markCommentAsRead {
  markCommentAsRead(commentUri: String!)
}
```

```json
{ "commentUri": "spotify:comment:7cXkaBSy1gocZ0hAtS9WsJ" }
```

Fires automatically alongside approve, reply, and delete operations.
You can also call it independently.

### Pin a Comment

```graphql
mutation pinComment {
  pinComment(commentUri: String!)
}
```

```json
{ "commentUri": "spotify:comment:0tHI8bJTfued8QR8E2Rv6B" }
```

!!! note
    - Only applicable to **published** comments
    - Only one comment can be pinned per episode (UI constraint; not verified at the API level)
    - After pinning, `getPinnedCommentForEpisode` fires to refresh the UI

### Unpin a Comment

```graphql
mutation unPinComment {
  unPinComment(commentUri: String!)
}
```

```json
{ "commentUri": "spotify:comment:0tHI8bJTfued8QR8E2Rv6B" }
```

### Block a User

```graphql
mutation blockUserForShow {
  blockUserForShow(username: String!, showUri: String!)
}
```

```json
{
  "username": "21yafw4q377x3jpbbcohdh4py",
  "showUri": "spotify:show:YOUR_SHOW_ID"
}
```

!!! note
    - `username` is the Spotify internal user ID (format: `21xxxxx`). Obtain
      it from the `author.uri` or `author.username` field in the
      `getCommentsForEpisode` response.
    - Blocking applies to the **entire show** (all episodes).
    - The UI presents an option to simultaneously delete the comment.
      If selected, `deleteCommentByCommentUri` fires in the same request batch.
    - `unblockUserForShow` is expected to exist symmetrically (not confirmed).

### Update Thread Creator Last Seen

```graphql
mutation updateThreadCreatorLastSeen {
  updateThreadCreatorLastSeen(
    parentCommentUri: String!
    primaryFilters: [String!]!
  )
}
```

```json
{
  "parentCommentUri": "spotify:comment:0tHI8bJTfued8QR8E2Rv6B",
  "primaryFilters": [
    "LIST_COMMENT_PRIMARY_FILTER_NEEDS_REVIEW",
    "LIST_COMMENT_PRIMARY_FILTER_PUBLISHED"
  ]
}
```

Fires automatically when a creator opens a reply thread. Records the
timestamp of the creator's last view to manage unread badges.

---

## Comment URI Format

All comment operations identify comments by their URI:

```
spotify:comment:{COMMENT_ID}
```

Example: `spotify:comment:7cXkaBSy1gocZ0hAtS9WsJ`

You can retrieve comment IDs from the `getCommentsForEpisode` query response.

---

## Comment Operations Summary

| Operation | Who | Automatable via API |
|-----------|-----|---------------------|
| Post comment (listener) | Listener | No (no creator-side post UI) |
| Post comment (creator) | Creator | Unknown (mutation name not captured) |
| Fetch comments | Creator | Yes — `getCommentsForEpisode` |
| Approve comment | Creator | Yes — `publishCommentByCommentUri` |
| Delete comment | Creator | Yes — `deleteCommentByCommentUri` |
| Reply (creator) | Creator | Yes — `createCommentReplyByCommentUri` |
| Enable / disable comments | Creator | Yes — `updateIsCommentsEnabledForShowByShowUri` |
| Set moderation level | Creator | Yes — `updateModerationPolicy` |
| Pin comment | Creator | Yes — `pinComment` |
| Unpin comment | Creator | Yes — `unPinComment` |
| Block user | Creator | Yes — `blockUserForShow` |
| Un-approve (revert to review) | Creator | **Not available — no UI or API for this** |

---

## Poll Mutations

### Create or Update a Poll (Upsert)

```graphql
mutation createPollForEpisode {
  createPollForEpisode(
    episodeUri: String!
    question: String!
    options: [PollOption!]!
    type: String!
    openingDate: Int!
    closingDate: Int!
  )
}
```

!!! note
    Creating and updating a poll use the **same mutation** (upsert behaviour).
    There is no separate `updatePollForEpisode` mutation.

**Variables example:**
```json
{
  "episodeUri": "spotify:episode:1xouj0WrH2klavXKzDWZbq",
  "question": "What is your favourite podcast genre?",
  "options": [
    { "description": "Talk shows" },
    { "description": "News / information" }
  ],
  "type": "SINGLE_CHOICE",
  "openingDate": 1779721200,
  "closingDate": 1780412399
}
```

| Parameter | Constraints |
|-----------|-------------|
| `question` | Max 140 characters |
| `options` | Min 2, max 7. Each `description` max 50 characters |
| `type` | `"SINGLE_CHOICE"` or `"MULTIPLE_CHOICE"` |
| `openingDate` / `closingDate` | Unix timestamp in seconds |

### Delete a Poll

```graphql
mutation deletePollForEpisode {
  deletePollForEpisode(episodeUri: String!)
}
```

```json
{ "episodeUri": "spotify:episode:1xouj0WrH2klavXKzDWZbq" }
```

!!! warning
    Deleting a poll also deletes all votes already cast.

---

### `SHOW_SEGMENTED_AUDIENCE` — Audience Segmentation (New: May 2026)

> Verified 2026-06-12. Added in the May 2026 dashboard refresh (believed to be
> tied to the Investor Day announcement).

Corresponds to the "Audience Segment" view in the S4C dashboard. Returns
the split between new and returning listeners.

#### Total (aggregate) query

```graphql
query {
  showByShowUri(getShowByShowUriRequest: { showUri: "spotify:show:YOUR_SHOW_ID" }) {
    analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_SEGMENTED_AUDIENCE,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: WINDOW_LAST_NINETY_DAYS
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on SegmentedAudienceValue {
            totalAudience
            newAudience
            returningAudience
          }
        }
      }
    }
  }
}
```

**Response type:** `SegmentedAudienceValue`

| Field | Type | Description |
|-------|------|-------------|
| `totalAudience` | `Long` | Unique listeners for the period (Spotify-scoped) |
| `newAudience` | `Long` | Listeners who heard the show for the first time in this period |
| `returningAudience` | `Long` | Listeners who already knew the show before this period |

#### Daily time-series query

```graphql
query {
  showByShowUri(getShowByShowUriRequest: { showUri: "spotify:show:YOUR_SHOW_ID" }) {
    analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_SEGMENTED_AUDIENCE,
      aggregationType: AGGREGATION_TYPE_DAILY,
      window: WINDOW_LAST_NINETY_DAYS
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on TimeSeriesValue {
            points {
              date
              value {
                ... on SegmentedAudienceValue {
                  totalAudience
                  newAudience
                  returningAudience
                }
              }
            }
          }
        }
      }
    }
  }
}
```

**Note:** When `aggregationType` is `AGGREGATION_TYPE_DAILY`, the return type is
`TimeSeriesValue` where each `points[].value` resolves to `SegmentedAudienceValue`
— an inline fragment is required as shown above.

#### Metric semantics — verified asymmetry (important)

`newAudience` and `returningAudience` have fundamentally different counting
semantics:

| Metric | Counting method | Sum of daily values vs. period total |
|--------|----------------|--------------------------------------|
| `newAudience` | Period-unique — a listener is counted **once** no matter how many days they appear | Daily sum **equals** the period total |
| `returningAudience` | Daily count — a returning listener is counted **every day** they listen | Daily sum is **much larger** than the period total (e.g. ~9× in a 90-day window) |
| `totalAudience` | Period-unique listeners | Daily sum is **larger** than the period total (because a single listener can appear on multiple days) |

**Verification method (confirmed 2026-06-12):** `newAudience + returningAudience == totalAudience`
holds exactly for every single day in a 90-day series, confirming the internal
consistency of the data.

!!! warning "Do not sum daily `returningAudience` to get a period total"
    The daily `returningAudience` value counts each returning listener once per
    day they listen. Summing it over 90 days produces a cumulative listen-day
    count, not a unique-listener count. Use `AGGREGATION_TYPE_TOTAL` for the
    period-unique figure.

---

### `SHOW_DISCOVERY_FUNNEL` — Impressions → Plays Funnel (New: May 2026)

> Observed 2026-06-12 alongside `SHOW_SEGMENTED_AUDIENCE` work.

Returns an Impressions → Plays → Average Completion Rate funnel with
period-over-period comparison values.

```graphql
query {
  showByShowUri(getShowByShowUriRequest: { showUri: "spotify:show:YOUR_SHOW_ID" }) {
    analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_DISCOVERY_FUNNEL,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: WINDOW_LAST_NINETY_DAYS
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on DiscoveryFunnelValue {
            steps {
              stepName
              displayName
              value
              conversionRateToNext
              periodOverPeriodPercentageDiff
            }
          }
        }
      }
    }
  }
}
```

**`DiscoveryFunnelStep` fields:**

| Field | Type | Description |
|-------|------|-------------|
| `stepName` | `String` | Internal name (`"impressions"`, `"plays"`, `"average_completion_rate"`) |
| `displayName` | `String` | Human-readable label (`"Impressions"`, `"Plays"`, `"Average Completion Rate"`) |
| `value` | `Long` | Raw count (absent for `average_completion_rate`) |
| `conversionRateToNext` | `Float` | Step-to-step conversion rate (e.g. `0.15` = 15% impression-to-play rate) |
| `periodOverPeriodPercentageDiff` | `Float` | Change vs. previous equivalent period (e.g. `0.42` = +42%) |

---

### `SHOW_WINDOWED_AVERAGE_COMPLETION_RATE` — Period Average Completion Rate (New: May 2026)

> Observed 2026-06-12.

Returns the average completion rate across all episodes within the window,
as a `PercentageValueFloat` (e.g. `0.60` = 60%).

```graphql
query {
  showByShowUri(getShowByShowUriRequest: { showUri: "spotify:show:YOUR_SHOW_ID" }) {
    analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_WINDOWED_AVERAGE_COMPLETION_RATE,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: WINDOW_LAST_NINETY_DAYS
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on PercentageValueFloat {
            value
          }
        }
      }
    }
  }
}
```

This is distinct from the episode-level `EPISODE_PERFORMANCE` (audience
retention curve). It is also distinct from `completion_rate` derived from
streams/starts — this is an aggregate average across the full window.

---

### New `Show` fields — existence confirmed, not yet tested (May 2026)

The following fields were added to the `Show` type in the May 2026 update.
They appear in the schema via introspection but have **not been tested with
live queries**. Use with caution.

**Sponsorship:**

| Field | Return type | Notes |
|-------|-------------|-------|
| `sponsorshipAnalytics` | `ShowSponsorshipAnalytics` | Sponsorship analytics |
| `listSponsorships` | `ListSponsorshipsResponse` | List of sponsorships |

---

## Engagement Tab Queries (New: Jun 2026)

> Captured 2026-06-12 by extracting and reconstructing GraphQL ASTs from
> the S4C analytics JavaScript bundle
> (`microfrontend-analytics-cdn.spotifycdn.com`). All queries are confirmed
> to exist in the production bundle; live execution has not been independently
> verified for every query.

### `getEngagementStats` — Daily Engagement Time Series

Fires when the Engagement tab loads. Returns daily time-series data for
consumption time, average consumption time, comments, and follower growth.

```graphql
query getEngagementStats(
  $showUri: ShowUri!,
  $dateRangeWindow: AnalyticsWindow!,
  $customDateRange: CustomDateRangeInput,
  $publishedAfter: TimestampInput,
  $publishedBefore: TimestampInput,
  $hasComments: Boolean!
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    uri
    consumptionTimeDaily: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_CONSUMPTION,
      aggregationType: AGGREGATION_TYPE_DAILY,
      window: $dateRangeWindow,
      customDateRange: $customDateRange
    }) {
      startDate
      endDate
      analyticsValue {
        analyticsValue {
          __typename
          ... on TimeSeriesValue {
            points {
              date
              endDate
              value {
                __typename
                ... on ConsumptionValue {
                  totalConsumptionHours
                }
              }
            }
          }
        }
      }
    }
    avgConsumptionTimeDaily: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_AVERAGE_CONSUMPTION_TIME,
      aggregationType: AGGREGATION_TYPE_DAILY,
      window: $dateRangeWindow,
      customDateRange: $customDateRange
    }) {
      startDate
      endDate
      analyticsValue {
        analyticsValue {
          __typename
          ... on TimeSeriesValue {
            points {
              date
              endDate
              value {
                __typename
                ... on AverageConsumptionTimeValue {
                  averageConsumptionSeconds
                }
              }
            }
          }
        }
      }
    }
  }
}
```

**New response types:**

| Type | Fields | Description |
|------|--------|-------------|
| `ConsumptionValue` | `totalConsumptionHours` | Total listening time in hours for the period/day |
| `AverageConsumptionTimeValue` | `averageConsumptionSeconds` | Per-listener average consumption in seconds |

**Corresponding dashboard metrics:**

| Dashboard label | MetricType | Response field |
|-----------------|-----------|----------------|
| 視聴時間 (Total consumption time) | `SHOW_CONSUMPTION` | `ConsumptionValue.totalConsumptionHours` |
| 平均視聴時間 (Avg consumption time / listener) | `SHOW_AVERAGE_CONSUMPTION_TIME` | `AverageConsumptionTimeValue.averageConsumptionSeconds` |

### `getEngagementStatsNRT` — Engagement Totals with Period Comparison

Returns aggregate totals for the period (not time-series), with
`periodOverPeriodPercentageDiff` for each metric. This is the near-real-time
(NRT) counterpart to `getEngagementStats`.

```graphql
query getEngagementStatsNRT(
  $showUri: ShowUri!,
  $dateRangeWindow: AnalyticsWindow!,
  $customDateRange: CustomDateRangeInput,
  $includePeriodComparison: Boolean!,
  $hasComments: Boolean!
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    uri
    consumptionTimeTotal: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_CONSUMPTION,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: $dateRangeWindow,
      customDateRange: $customDateRange,
      includePeriodComparison: $includePeriodComparison
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on ConsumptionValue {
            totalConsumptionHours
          }
        }
        periodOverPeriodPercentageDiff
      }
    }
    avgConsumptionTimeTotal: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_AVERAGE_CONSUMPTION_TIME,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: $dateRangeWindow,
      customDateRange: $customDateRange,
      includePeriodComparison: $includePeriodComparison
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on AverageConsumptionTimeValue {
            averageConsumptionSeconds
          }
        }
        periodOverPeriodPercentageDiff
      }
    }
  }
}
```

**Key field:** `analyticsValue.periodOverPeriodPercentageDiff` — the
`+36.6%` / `+7.4%` figures shown in the dashboard are sourced from this
field (e.g. `0.366` = +36.6%).

### `getShowRetention` — Week-over-Week Retention Rate

Returns the weekly retention rate: the fraction of this week's listeners
who also listened the previous week.

```graphql
query getShowRetention(
  $showUri: ShowUri!,
  $dateRangeWindow: AnalyticsWindow!,
  $publishedAfter: TimestampInput,
  $publishedBefore: TimestampInput
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    uri
    retentionWeekly: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_RETENTION,
      aggregationType: AGGREGATION_TYPE_WEEKLY,
      window: $dateRangeWindow
    }) {
      startDate
      endDate
      analyticsValue {
        analyticsValue {
          __typename
          ... on TimeSeriesValue {
            points {
              date
              endDate
              value {
                __typename
                ... on RatioValueFloat {
                  value
                }
              }
            }
          }
        }
      }
    }
  }
}
```

**Response:** `RatioValueFloat.value` — a float between 0 and 1
(e.g. `0.537` = 53.7% retention rate shown in the dashboard).
Use `WINDOW_LAST_NINETY_DAYS` with `AGGREGATION_TYPE_WEEKLY` to reproduce
the 90-day retention chart.

### `getEpisodeCompletionRates` — Per-Episode Completion Rate (Latest 10)

Returns the average completion rate for each of the 10 most recently
published episodes, measured over the first 7 days after publication.

```graphql
query getEpisodeCompletionRates($showUri: ShowUri!) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    uri
    latestEpisodes: episodesV2(listEpisodesV2Request: {
      sort: {sortBy: PUBLISHED_ON, sortOrder: DESC},
      filter: PUBLISHED_EPISODES,
      pagination: {pageSize: 10},
      episodeMetadataFieldMask: {
        paths: [
          "hosted_episode.title",
          "hosted_episode.published_on",
          "hosted_episode.uri",
          "non_hosted_episode.title",
          "non_hosted_episode.published_on",
          "non_hosted_episode.uri"
        ]
      }
    }) {
      items {
        uri
        title
        publishedOn { seconds }
        completionRate: analyticsBatch(batchGetEpisodeAnalyticsItem: {
          episodeAnalyticsMetricType: EPISODE_AVERAGE_COMPLETION_RATE,
          aggregationType: AGGREGATION_TYPE_TOTAL,
          window: WINDOW_FIRST_SEVEN_DAYS
        }) {
          analyticsValue {
            analyticsValue {
              __typename
              ... on PercentageValueFloat {
                value
              }
            }
          }
        }
      }
    }
  }
}
```

**Response:** `PercentageValueFloat.value` — a float between 0 and 1
(e.g. `0.56` = 56% completion rate).

**New window value:** `WINDOW_FIRST_SEVEN_DAYS` — the 7-day post-publication
window used for the "エピソードの完全再生率" chart.

### `getEpisodeTimeSeriesByMetric` — Episode Metric Time Series

Returns a time series for a single episode and any `EpisodeAnalyticsMetricType`.

```graphql
query getEpisodeTimeSeriesByMetric(
  $episodeUri: EpisodeUri!,
  $metricType: EpisodeAnalyticsMetricType!,
  $window: AnalyticsWindow!,
  $aggregationType: AggregationType!
) {
  episodeByUri(getEpisodeRequest: {episodeUri: $episodeUri}) {
    analyticsBatch(batchGetEpisodeAnalyticsItem: {
      episodeAnalyticsMetricType: $metricType,
      aggregationType: $aggregationType,
      window: $window
    }) {
      startDate
      endDate
      analyticsValue {
        analyticsValue {
          __typename
          ... on TimeSeriesValue {
            points {
              date
              value {
                __typename
                ... on CountValueLong { longValue: value }
                ... on ConsumptionValue {
                  totalConsumptionHours
                  foregroundConsumptionPercent
                }
                ... on RatioValueFloat { floatValue: value }
              }
            }
          }
        }
      }
    }
  }
}
```

**Notable:** `ConsumptionValue` also exposes `foregroundConsumptionPercent`
at the episode level.

---

## Audience Growth Queries (New: Jun 2026)

These queries use the `coreFanTimeSeries` / `coreFanMetricSummary` /
`coreFanInsight` fields on the `Show` type. Audience is segmented into three
tiers: **core fans** (highly engaged), **developing audience**, and
**new audience**.

### `getAudienceGrowthTimeSeries` — Core Fan Tier Daily Time Series

```graphql
query getAudienceGrowthTimeSeries(
  $showUri: ShowUri!,
  $startDate: String!,
  $endDate: String!
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    coreFanTimeSeries(getCoreFanTimeSeriesRequest: {
      startDate: $startDate,
      endDate: $endDate
    }) {
      timeSeries {
        date
        coreFans
        coreFansPercent
        developingAudience
        developingAudiencePercent
        newAudience
        newAudiencePercent
        totalAudience
        totalAudiencePercent
      }
    }
  }
}
```

**Variables:** `startDate` / `endDate` are `"YYYY-MM-DD"` strings (not
`AnalyticsWindow` enum values — unlike most analytics queries).

**Audience tiers:**

| Field | Description |
|-------|-------------|
| `coreFans` | Highly engaged listeners (full definition not published) |
| `developingAudience` | Listeners who are becoming more engaged |
| `newAudience` | First-time listeners in the period |
| `totalAudience` | All unique listeners |
| `*Percent` fields | Each tier as a percentage of `totalAudience` |

### `getAudienceGrowthMetricSummary` — Core Fan Summary with Period Comparison

```graphql
query getAudienceGrowthMetricSummary(
  $showUri: ShowUri!,
  $startDate: String!,
  $endDate: String!,
  $isAllTime: Boolean!
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    coreFanMetricSummary(getCoreFanMetricSummaryRequest: {
      startDate: $startDate,
      endDate: $endDate,
      isAllTime: $isAllTime
    }) {
      summary {
        newAudience
        newAudiencePercent
        developingAudience
        developingAudiencePercent
        coreFans
        coreFansPercent
        totalAudience
        totalAudiencePercent
        gainedCoreFans
        startDate
        endDate
        lookbackStartDate
        lookbackEndDate
      }
    }
  }
}
```

**`gainedCoreFans`** — net new core fans gained in the period.
`lookbackStartDate` / `lookbackEndDate` describe the prior period used for
comparison.

### `getAudienceGrowthInsights` — Core Fan Behavioral Insights

```graphql
query getAudienceGrowthInsights(
  $showUri: ShowUri!,
  $startDate: String!,
  $endDate: String!
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    coreFanInsight(getCoreFanInsightRequest: {
      startDate: $startDate,
      endDate: $endDate
    }) {
      insight {
        averageEpisodeRetentionCoreFansPercent
        averageEpisodeRetentionTotalAudiencePercent
        averageEpisodeRetentionDiffPercent
        averageConsumptionHoursCoreFans
        averageConsumptionHoursTotalAudience
        averageConsumptionHoursDiffPercent
      }
    }
  }
}
```

**Purpose:** Shows how core fans behave vs. total audience —
`averageEpisodeRetentionCoreFansPercent` and
`averageConsumptionHoursCoreFans` quantify the difference.

---

## Benchmark Queries (New: Jun 2026)

Benchmark queries compare your show's metrics against percentile bands
from comparable shows.

### `getBenchmarkTotal` — Benchmark Totals

```graphql
query getBenchmarkTotal(
  $showUri: ShowUri!,
  $window: AnalyticsWindow!,
  $benchmarkMetricType: BenchmarkMetricType!,
  $benchmarkEpisodePool: BenchmarkEpisodePool!
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    benchmarkTotal: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_BENCHMARK_EPISODE,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: $window,
      benchmarkParams: {
        metricType: $benchmarkMetricType,
        episodePool: $benchmarkEpisodePool,
        percentiles: [20, 40, 50, 60, 80]
      }
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on BenchmarkTimeSeriesValue {
            points {
              label
              percentiles {
                percentile
                value
              }
            }
          }
        }
      }
    }
  }
}
```

### `getBenchmarkTimeSeries` — Benchmark Over Time

Same structure as `getBenchmarkTotal` but with
`aggregationType: AGGREGATION_TYPE_DAILY`.

#### `BenchmarkMetricType` enum values

| Value | Description |
|-------|-------------|
| `BENCHMARK_ALL_PLATFORM_AUDIENCE` | All-platform unique audience |
| `BENCHMARK_AVG_CONSUMPTION_TIME` | Average consumption time |
| `BENCHMARK_COMMENTS` | Comment count |
| `BENCHMARK_CONSUMPTION_TIME` | Total consumption time |
| `BENCHMARK_EPISODE_COMPLETION_PERCENT` | Episode completion rate |
| `BENCHMARK_IMPRESSIONS` | Impressions |
| `BENCHMARK_OFF_PLATFORM_AUDIENCE` | Off-platform audience |
| `BENCHMARK_OFF_PLATFORM_DOWNLOADS` | Off-platform downloads |
| `BENCHMARK_PLAYS` | Plays |
| `BENCHMARK_PLAYS_AND_DOWNLOADS` | Plays + downloads |
| `BENCHMARK_RETENTION` | Week-over-week retention |
| `BENCHMARK_SPOTIFY_AUDIENCE` | Spotify-only audience |
| `BENCHMARK_WATCH_TIME_PERCENT` | Watch time percentage (video) |

#### `BenchmarkEpisodePool` enum values

| Value | Description |
|-------|-------------|
| `ALL_EPISODES` | All episodes in the benchmark comparison pool |
| `LAST_10_EPISODES` | Most recent 10 episodes only |

---

## Discovery Funnel Query (Updated: Jun 2026)

### `getShowDiscoveryFunnelStats` (Named Operation)

The browser uses a named operation `getShowDiscoveryFunnelStats` to load the
discovery funnel. The underlying metric is `SHOW_DISCOVERY_FUNNEL` (already
documented). This query reveals the exact field structure used by the
production app.

```graphql
query getShowDiscoveryFunnelStats(
  $showUri: ShowUri!,
  $dateRangeWindow: AnalyticsWindow!,
  $customDateRange: CustomDateRangeInput
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    uri
    discoveryFunnel: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_DISCOVERY_FUNNEL,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: $dateRangeWindow,
      customDateRange: $customDateRange
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on DiscoveryFunnelValue {
            steps {
              stepName
              displayName
              startDate
              endDate
              conversionRateToNext
              periodOverPeriodPercentageDiff
              stepValue {
                __typename
                ... on CountValueLong { countValue: value }
                ... on RatioValueFloat { ratioValue: value }
              }
            }
          }
        }
      }
    }
  }
}
```

**Updated `DiscoveryFunnelStep` structure (corrected from earlier docs):**

| Field | Type | Description |
|-------|------|-------------|
| `stepName` | `String` | `"impressions"`, `"plays"`, `"average_completion_rate"` |
| `displayName` | `String` | Human-readable label |
| `startDate` / `endDate` | `String` | Date range for this step |
| `conversionRateToNext` | `Float` | Step-to-step conversion rate |
| `periodOverPeriodPercentageDiff` | `Float` | Change vs. previous period |
| `stepValue` | union | `CountValueLong` (impressions, plays) or `RatioValueFloat` (completion rate) |

**Note:** `stepValue` uses a union type with field aliasing (`countValue`,
`ratioValue`). The earlier documentation listed a `value` field directly on
`DiscoveryFunnelStep` — that was incorrect.

---

## Segmented Audience Queries (Updated: Jun 2026)

The browser uses named operations `getShowSegmentedAudienceTotal` and
`getShowSegmentedAudienceTimeSeries` (rather than anonymous queries).

### `getShowSegmentedAudienceTotal`

```graphql
query getShowSegmentedAudienceTotal(
  $showUri: ShowUri!,
  $dateRangeWindow: AnalyticsWindow!,
  $customDateRange: CustomDateRangeInput
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    uri
    segmentedAudienceTotal: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_SEGMENTED_AUDIENCE,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: $dateRangeWindow,
      customDateRange: $customDateRange
    }) {
      startDate
      endDate
      analyticsValue {
        analyticsValue {
          __typename
          ... on SegmentedAudienceValue {
            returningAudience
            newAudience
            totalAudience
          }
        }
      }
    }
  }
}
```

### `getShowSegmentedAudienceTimeSeries`

```graphql
query getShowSegmentedAudienceTimeSeries(
  $showUri: ShowUri!,
  $dateRangeWindow: AnalyticsWindow!,
  $customDateRange: CustomDateRangeInput,
  $publishedAfter: TimestampInput,
  $publishedBefore: TimestampInput
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    uri
    segmentedAudienceTimeSeries: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_SEGMENTED_AUDIENCE,
      aggregationType: AGGREGATION_TYPE_DAILY,
      window: $dateRangeWindow,
      customDateRange: $customDateRange
    }) {
      startDate
      endDate
      analyticsValue {
        analyticsValue {
          __typename
          ... on TimeSeriesValue {
            points {
              date
              value {
                __typename
                ... on SegmentedAudienceValue {
                  returningAudience
                  newAudience
                  totalAudience
                }
              }
            }
          }
        }
      }
    }
  }
}
```

---

## Overview Tab Queries (New: Jun 2026)

### `getShowStreamsAndDownloadsDaily` — All-Platforms Streams+Downloads

```graphql
query getShowStreamsAndDownloadsDaily(
  $showUri: ShowUri!,
  $aggregationType: AggregationType!,
  $dateRangeWindow: AnalyticsWindow!,
  $customDateRange: CustomDateRangeInput,
  $publishedAfter: TimestampInput,
  $publishedBefore: TimestampInput
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    uri
    streamsAndDownloadsDaily: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_STREAMS_AND_DOWNLOADS,
      aggregationType: $aggregationType,
      window: $dateRangeWindow,
      customDateRange: $customDateRange
    }) {
      startDate
      endDate
      analyticsValue {
        analyticsValue {
          __typename
          ... on TimeSeriesValue {
            points {
              date
              endDate
              value {
                __typename
                ... on CountValueLong { value }
              }
            }
          }
        }
      }
    }
  }
}
```

**Note:** `$aggregationType` is a variable here (unlike most queries where it is
hardcoded). Pass `AGGREGATION_TYPE_DAILY`, `AGGREGATION_TYPE_WEEKLY`, or
`AGGREGATION_TYPE_MONTHLY`.

### `getShowTopEpisodesByMetric` — Top Episodes for Any Metric

```graphql
query getShowTopEpisodesByMetric(
  $showUri: ShowUri!,
  $metricType: EpisodeAnalyticsMetricType!,
  $dateRangeWindow: AnalyticsWindow!,
  $customDateRange: CustomDateRangeInput
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    uri
    episodesV2(listEpisodesV2Request: {
      sort: {sortBy: PUBLISHED_ON, sortOrder: DESC},
      filter: PUBLISHED_EPISODES,
      pagination: {pageSize: 250},
      episodeMetadataFieldMask: {
        paths: [
          "hosted_episode.title", "hosted_episode.published_on", "hosted_episode.uri",
          "non_hosted_episode.title", "non_hosted_episode.published_on", "non_hosted_episode.uri"
        ]
      }
    }) {
      items {
        uri
        title
        publishedOn { seconds }
        metric: analyticsBatch(batchGetEpisodeAnalyticsItem: {
          episodeAnalyticsMetricType: $metricType,
          aggregationType: AGGREGATION_TYPE_TOTAL,
          window: $dateRangeWindow,
          customDateRange: $customDateRange
        }) {
          analyticsValue {
            analyticsValue {
              __typename
              ... on SingleValueLong { value }
              ... on PercentageValueFloat { value }
            }
          }
        }
      }
    }
  }
}
```

### `getShowTopEpisodesByImpressions` — Top Episodes by Impression Source

Returns top episodes ranked by impressions, with a breakdown by impression source.

```graphql
query getShowTopEpisodesByImpressions(
  $showUri: ShowUri!,
  $dateRangeWindow: AnalyticsWindow!,
  $customDateRange: CustomDateRangeInput
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    uri
    topEpisodesByImpressions: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: TOP_EPISODES_BY_IMPRESSIONS_FACETED,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: $dateRangeWindow,
      customDateRange: $customDateRange
    }) {
      startDate
      endDate
      analyticsValue {
        analyticsValue {
          __typename
          ... on TopEpisodesFacetedValue {
            episodes {
              episodeUri
              episode {
                title
                publishedOn { seconds }
              }
              facetedValue {
                totalValue
                impressionSourceBreakdown {
                  total
                  counts {
                    displayName
                    source
                    count
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

**New `ShowAnalyticsMetricType` value:** `TOP_EPISODES_BY_IMPRESSIONS_FACETED`

---

## AI Analytics Insights (New: Jun 2026)

S4C now generates AI-powered insights about analytics trends.

### `generateAnalyticsInsight` — Trigger AI Insight Generation

```graphql
mutation generateAnalyticsInsight($generateInsightRequest: GenerateInsightRequestInput!) {
  generateAnalyticsInsight(generateInsightRequest: $generateInsightRequest) {
    insightId
    state
    payload
    generatedAt
  }
}
```

**Response fields:**

| Field | Type | Description |
|-------|------|-------------|
| `insightId` | `String` | Unique ID — use with `getAnalyticsInsight` to poll |
| `state` | `String` | Generation state (e.g. `"PENDING"`, `"COMPLETE"`) |
| `payload` | `String` | Generated insight text (populated when complete) |
| `generatedAt` | `String` | ISO timestamp |

### `getAnalyticsInsight` — Fetch a Generated Insight

```graphql
query getAnalyticsInsight($showUri: ShowUri!, $insightId: String!) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    analyticsInsight(getInsightRequest: {insightId: $insightId}) {
      insightId
      state
      payload
      generatedAt
    }
  }
}
```

### `submitAnalyticsInsightFeedback` — Thumbs Up/Down Feedback

```graphql
mutation submitAnalyticsInsightFeedback(
  $submitAnalyticsInsightFeedbackRequest: SubmitAnalyticsInsightFeedbackRequestInput!
) {
  submitAnalyticsInsightFeedback(
    submitAnalyticsInsightFeedbackRequest: $submitAnalyticsInsightFeedbackRequest
  ) {
    feedbackId
  }
}
```

---

## Updated Enum Reference (Jun 2026)

### `ShowAnalyticsMetricType` — New Values

| Enum value | Description |
|-----------|-------------|
| `SHOW_CONSUMPTION` | Total listening time — returns `ConsumptionValue` |
| `SHOW_AVERAGE_CONSUMPTION_TIME` | Per-listener average consumption — returns `AverageConsumptionTimeValue` |
| `SHOW_RETENTION` | Week-over-week retention rate — returns `TimeSeriesValue<RatioValueFloat>` |
| `SHOW_BENCHMARK_EPISODE` | Benchmark comparison — requires `benchmarkParams` argument |
| `SHOW_STREAMS_AND_DOWNLOADS_AVERAGE` | Rolling average of streams+downloads |
| `SHOW_STREAMS_AND_DOWNLOADS_BY_APP` | Platform distribution (Apple, Spotify, Amazon, etc.) |
| `SHOW_STREAMS_AND_DOWNLOADS_BY_DEVICE` | Device type distribution |
| `SHOW_STREAMS_AND_DOWNLOADS_BY_GEO_ALL_PLATFORMS` | Geographic distribution across all platforms |
| `TOP_EPISODES_BY_IMPRESSIONS_FACETED` | Top episodes by impressions with source breakdown |
| `SHOW_IMPRESSIONS_FACETED` | Impressions with source facets |
| `SHOW_STREAMS_FACETED` | Streams with facets |
| `SHOW_FOLLOWER_GROWTH` | Follower growth time series |

### `EpisodeAnalyticsMetricType` — New Values

| Enum value | Description |
|-----------|-------------|
| `EPISODE_AVERAGE_COMPLETION_RATE` | Average completion rate (0–1 float) — use `WINDOW_FIRST_SEVEN_DAYS` |
| `EPISODE_AVERAGE_CONSUMPTION_TIME` | Per-listener average consumption time |
| `EPISODE_CONSUMPTION` | Total listening time |
| `EPISODE_AUDIENCE_SIZE` | Audience size |
| `EPISODE_IMPRESSIONS` | Impressions |
| `EPISODE_IMPRESSIONS_FACETED` | Impressions with facets |
| `EPISODE_IMPRESSIONS_FUNNEL` | Impressions funnel |
| `EPISODE_IMPRESSIONS_TO_PLAYS_RATE` | Impression-to-play conversion rate |
| `EPISODE_DISCOVERY_FUNNEL` | Episode-level discovery funnel |
| `EPISODE_PLAYS_FACETED` | Plays with facets |
| `EPISODE_STREAMS_FACETED` | Streams with facets |
| `EPISODE_RETENTION` | Episode-level retention |
| `EPISODE_SPOTIFY_PLAYS_BY_COUNTRY` | Spotify plays by country |

### `AnalyticsWindow` — New Values

| Value | Description |
|-------|-------------|
| `WINDOW_FIRST_SEVEN_DAYS` | First 7 days after episode publication |
| `WINDOW_FIRST_THIRTY_DAYS` | First 30 days after episode publication |
| `WINDOW_FIRST_SIXTY_DAYS` | First 60 days after episode publication |
| `WINDOW_LAST_SIXTY_DAYS` | Last 60 days |
| `WINDOW_SINCE_PUBLISHED` | Since episode publication date |
| `WINDOW_UNSPECIFIED` | Unspecified window (used for all-time averages) |

---

## Known Limitations

| Feature | Status |
|---------|--------|
| Transcript editing | Could not be captured. The test account showed "transcript feature unavailable" — likely requires a paid plan or specific eligibility |
| Chapter create / update / delete mutations | Could not be captured. The test show had too few or too short episodes to unlock chapters |

---

## showsForUser — List All Shows for Authenticated User

**Verified: 2026-06-21**

Returns all shows managed by the currently authenticated user. Useful for
resolving show IDs without navigating the S4C dashboard.

```graphql
query {
  showsForUser {
    shows {
      name
      uri
    }
    pagination {
      # additional pagination fields available
    }
  }
}
```

### Response

```json
{
  "data": {
    "showsForUser": {
      "shows": [
        {
          "name": "ぼくらの戦略論",
          "uri": "spotify:show:5vXOMdLUviA9dGrWXy79cF"
        }
      ]
    }
  }
}
```

The show ID (for use in `--show-id` CLI argument or other API calls) is
the last segment of the `uri` field: `spotify:show:<SHOW_ID>`.

### Python example

```python
import requests

headers = {
    "Authorization": f"Bearer {bearer}",
    "Content-Type": "application/json",
}
res = requests.post(
    "https://creators-graph.spotify.com/v2/graph-pq",
    headers=headers,
    json={"query": "{ showsForUser { shows { name uri } } }"},
)
shows = res.json()["data"]["showsForUser"]["shows"]
for show in shows:
    show_id = show["uri"].split(":")[-1]
    print(show_id, show["name"])
```

### Notes

- Returns up to 65+ shows in a single call (no pagination needed in practice).
- The `Show` type exposes many fields beyond `name` and `uri` — see schema
  introspection via `__type(name: "Show")` for the full field list.
- The `anchor.fm REST` endpoint `GET /v3/stations/status` covers the same
  use-case but returns 403 with the SpotifyConnector Bearer token. Use this
  GraphQL query instead.
