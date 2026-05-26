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
    `query` field. Whether the server accepts this as a fallback is not
    confirmed for all operations.

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

### Analytics — Audience Tab

| Operation | Type | Description |
|-----------|------|-------------|
| `getShowAudienceDemographicsStats` | query | Age / gender breakdown |
| `getShowAudienceAllPlatformsGeoStats` | query | Geographic breakdown across platforms |
| `getShowAudienceAllPlatformsStats` | query | Cross-platform stats |

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

## Known Limitations

| Feature | Status |
|---------|--------|
| Transcript editing | Could not be captured. The test account showed "transcript feature unavailable" — likely requires a paid plan or specific eligibility |
| Chapter create / update / delete mutations | Could not be captured. The test show had too few or too short episodes to unlock chapters |
