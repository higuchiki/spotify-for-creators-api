# Changelog

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

## Endpoint Count Summary

| Category | Count |
|----------|-------|
| anchor.fm REST endpoints (1-A to 1-Y) | 25 (including sub-endpoints in 1-V, 1-W, 6-A to 6-D) |
| GraphQL queries | ~35 |
| GraphQL mutations | ~12 |
