# REST API Reference

**Base URL:** `https://api-v5.anchor.fm`

The anchor.fm internal REST API handles episode creation, metadata
read/write, show settings, and media upload. It predates and coexists with
the GraphQL API — Anchor's acquisition by Spotify left this backend intact.

---

## Common Headers

All requests require:

```
Authorization: Bearer {bearer_token}
Accept: application/json
```

POST / PUT requests additionally require:

```
Content-Type: application/json
Origin: https://creators.spotify.com
Referer: https://creators.spotify.com/
```

## Common Query Parameters

| Parameter | Description |
|-----------|-------------|
| `isMumsCompatible=true` | Required on every endpoint. Never omit. |
| `returnWebIds=true` | Recommended on `/overview`. Omitting it may drop some fields. |

---

## ID System

The anchor.fm API uses **anchor numeric IDs** (integers), not the Spotify
string IDs visible in the S4C UI. You need to convert between them before
calling most endpoints.

| ID type | Example | Used in |
|---------|---------|---------|
| Spotify Episode ID | `"1xouj0WrH2klavXKzDWZbq"` | S4C UI, GraphQL |
| Anchor Episode ID | `123456789` | anchor.fm REST endpoints |
| Spotify Show ID | `"YOUR_SHOW_ID"` | S4C UI, GraphQL |
| Station ID (numeric) | `YOUR_STATION_ID` | anchor.fm REST endpoints |

---

## 1-A. Spotify Episode ID → Anchor Numeric ID

```
GET /v3/episodes/spotify:episode:{SPOTIFY_EP_ID}/episodeId?isMumsCompatible=true
```

Translates a Spotify Episode ID into the numeric ID required by all other
episode endpoints.

### Response

```json
{ "episodeId": 123456789 }
```

=== "Python"
    ```python
    resp = requests.get(
        f"https://api-v5.anchor.fm/v3/episodes/spotify:episode:{spotify_ep_id}/episodeId"
        "?isMumsCompatible=true",
        headers=headers_get,
    )
    anchor_id = resp.json()["episodeId"]
    ```

=== "Node.js"
    ```javascript
    const BASE = "https://api-v5.anchor.fm";
    const r = await fetch(
      `${BASE}/v3/episodes/spotify:episode:${spotifyEpId}/episodeId?isMumsCompatible=true`,
      { headers: headersGet },
    );
    const { episodeId: anchorId } = await r.json();
    ```

=== "cURL"
    ```bash
    curl -s \
      -H "Authorization: Bearer $BEARER" \
      "https://api-v5.anchor.fm/v3/episodes/spotify:episode:${SPOTIFY_EP_ID}/episodeId?isMumsCompatible=true"
    ```

---

## 1-B. Get Episode Overview

```
GET /v3/episodes/{ANCHOR_ID}/overview?isMumsCompatible=true&returnWebIds=true
```

Returns the current metadata for an episode.

### Response fields

| Field | Type | Description |
|-------|------|-------------|
| `title` | string | Episode title |
| `description` | string | Description HTML |
| `isPublished` | boolean | Whether the episode is published |
| `publishOn` | string \| null | Scheduled publish time (UTC ISO 8601). Present only for scheduled episodes |
| `userId` | integer | Account user ID — required for update calls |
| `podcastEpisodeType` | string | `"full"` / `"trailer"` / `"bonus"` |
| `podcastEpisodeIsExplicit` | boolean | Explicit content flag |

### Episode state matrix

| `isPublished` | `publishOn` | State |
|--------------|-------------|-------|
| `false` | present | Scheduled |
| `true` | — | Published |
| `false` | absent | Draft |

!!! warning "Draft episodes may return 403"
    Calling this endpoint on a draft episode (`isPublished: false`, no
    `publishOn`) was observed to return **HTTP 403 Forbidden** (2026-05) —
    even in a live browser session. If you need to update a draft episode's
    metadata, keep the original values in memory before calling the update
    endpoint.

    **Observation 2026-11-21:** `/overview?isMumsCompatible=true&returnWebIds=true`
    returned **HTTP 200** on freshly created drafts (both from the S4C web app
    and from a script, 28 drafts), including `spotifyUri`. The 403 may depend
    on the episode's state (e.g. no media yet) or may no longer apply —
    handle both.

!!! note "Numbering fields are read under different names"
    `/overview` returns the season and episode number as
    `podcastSeasonNumber` / `podcastEpisodeNumber`. The update endpoint (1-C)
    **writes** them as `seasonNumber` / `episodeNumber`.

=== "Python"
    ```python
    r = requests.get(
        f"https://api-v5.anchor.fm/v3/episodes/{anchor_id}/overview"
        "?isMumsCompatible=true&returnWebIds=true",
        headers=headers_get,
    )
    overview = r.json()
    ```

=== "Node.js"
    ```javascript
    const r = await fetch(
      `${BASE}/v3/episodes/${anchorId}/overview?isMumsCompatible=true&returnWebIds=true`,
      { headers: headersGet },
    );
    const overview = await r.json();
    ```

=== "cURL"
    ```bash
    curl -s \
      -H "Authorization: Bearer $BEARER" \
      "https://api-v5.anchor.fm/v3/episodes/${ANCHOR_ID}/overview?isMumsCompatible=true&returnWebIds=true"
    ```

---

## 1-C. Update Episode (Title / Description / Publish Date)

```
POST /v3/episodes/{ANCHOR_ID}/update?isMumsCompatible=true
```

Updates episode metadata. You can change the title, description, episode
type, explicit flag, and scheduled publish time in a single call.

### Request body

```json
{
  "userId": YOUR_USER_ID,
  "title": "Episode title",
  "description": "<p>Episode description HTML</p>",
  "seasonNumber": 1,
  "episodeNumber": 3,
  "episodeType": "full",
  "isPublished": false,
  "podcastEpisodeIsExplicit": false,
  "publishOn": 1748732400,
  "wizardDraftedToPublishOn": "2026-06-01T21:00:00.000Z"
}
```

The S4C web app itself (episode wizard, 2026-11) sends
`{userId, title, description, seasonNumber, episodeNumber, episodeType,
isPublished, podcastEpisodeIsExplicit, isVideoEighteenPlus, publishOn: null}`
when saving a draft.

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `userId` | integer | Yes | Copy directly from the `/overview` response |
| `title` | string | Yes | Omitting it risks the server setting a blank title |
| `description` | string | Yes | HTML format |
| `episodeType` | string | Yes | `"full"` / `"trailer"` / `"bonus"` |
| `isPublished` | boolean | Yes | Pass the current value from `/overview` to preserve state |
| `podcastEpisodeIsExplicit` | boolean | Yes | Same as above |
| `seasonNumber` | integer | No | Season number. **Not** `podcastSeasonNumber` — see warning below |
| `episodeNumber` | integer | No | Episode number. **Not** `podcastEpisodeNumber` |
| `isVideoEighteenPlus` | boolean | No | 18+ flag for video episodes (sent by the web app) |
| `publishOn` | integer | No | **Unix timestamp (seconds).** Required when scheduling. See warning below |
| `wizardDraftedToPublishOn` | string | No | UTC ISO 8601. Must match `publishOn` in wall-clock time |

### Response

HTTP 200 on success:

```json
{
  "description": "<p>Episode description HTML</p>",
  "title": "Episode title",
  "didChangePublishState": true,
  "shareUrl": "https://creators.spotify.com/pod/show/..."
}
```

!!! warning "Scheduling behaviour"
    Sending `isPublished: false` together with a `publishOn` value causes
    the server to automatically treat it as `isPublished: true` (scheduled
    for future publication). This is the correct way to schedule an episode.

!!! warning "Published episodes lock publishOn"
    Once an episode is published (`isPublished: true`), you cannot change
    `publishOn` via this endpoint. Use the S4C UI or Playwright to modify
    the publish date of a live episode.

!!! warning "Season / episode number: write names differ from read names"
    Send `seasonNumber` and `episodeNumber`. The names returned by
    `/overview` (`podcastSeasonNumber`, `podcastEpisodeNumber`) are
    **silently ignored** by this endpoint — it answers HTTP 200 and nothing is
    saved. Read them back from `/overview` under the `podcast*` names to
    verify (verified 2026-11-21).

!!! note "Unpublishing"
    Sending `isPublished: false` (without `publishOn`) for a published episode
    moved it back to drafts (verified 2026-11-21 on three video episodes).
    Be careful: in the web app, unpublishing an episode deletes its clips.

!!! warning "`publishOn` must be a Unix timestamp integer"
    `publishOn` expects a **Unix timestamp in integer seconds**, not an ISO 8601
    string. Sending an ISO string (e.g. `"2026-06-01T21:00:00.000Z"`) causes
    the server to parse only the leading digits as an integer — `"2026"` becomes
    `2026` seconds, storing `1970-01-01T00:33:46.000Z`.

    `wizardDraftedToPublishOn` is the opposite: it requires an **ISO 8601
    string**. Sending an integer returns HTTP 400.

    ```python
    from datetime import datetime, timezone, timedelta
    JST = timezone(timedelta(hours=9))
    publish_jst = datetime(2026, 6, 1, 6, 0, 0, tzinfo=JST)
    pub_utc = publish_jst.astimezone(timezone.utc)
    publish_on_unix = int(pub_utc.timestamp())                    # → 1748732400
    publish_on_iso  = pub_utc.strftime("%Y-%m-%dT%H:%M:%S.000Z") # → "2026-05-31T21:00:00.000Z"
    ```

=== "Python"
    ```python
    pub_utc = publish_jst.astimezone(timezone.utc)
    payload = {
        "userId": overview["userId"],
        "title": "New episode title",
        "description": "<p>New description</p>",
        "episodeType": overview.get("podcastEpisodeType", "full"),
        "isPublished": overview.get("isPublished", False),
        "podcastEpisodeIsExplicit": overview.get("podcastEpisodeIsExplicit", False),
        "publishOn": int(pub_utc.timestamp()),                    # Unix integer
        "wizardDraftedToPublishOn": pub_utc.strftime("%Y-%m-%dT%H:%M:%S.000Z"),  # ISO string
    }
    r = requests.post(
        f"https://api-v5.anchor.fm/v3/episodes/{anchor_id}/update?isMumsCompatible=true",
        json=payload,
        headers=headers_post,
    )
    ```

=== "Node.js"
    ```javascript
    const pubUtc = new Date("2026-06-01T06:00:00+09:00"); // 06:00 JST
    const payload = {
      userId: overview.userId,
      title: "New episode title",
      description: "<p>New description</p>",
      episodeType: overview.podcastEpisodeType ?? "full",
      isPublished: overview.isPublished ?? false,
      podcastEpisodeIsExplicit: overview.podcastEpisodeIsExplicit ?? false,
      publishOn: Math.floor(pubUtc.getTime() / 1000),  // Unix integer
      wizardDraftedToPublishOn: pubUtc.toISOString(),  // ISO string
    };
    const r = await fetch(
      `${BASE}/v3/episodes/${anchorId}/update?isMumsCompatible=true`,
      { method: "POST", headers: headersPost, body: JSON.stringify(payload) },
    );
    if (!r.ok) throw new Error(`update -> ${r.status} ${await r.text()}`);
    ```

=== "cURL"
    ```bash
    curl -s -X POST \
      -H "Authorization: Bearer $BEARER" \
      -H "Content-Type: application/json" \
      -H "Origin: https://creators.spotify.com" \
      -H "Referer: https://creators.spotify.com/" \
      -d '{"userId":YOUR_USER_ID,"title":"New title","description":"<p>Desc</p>","episodeType":"full","isPublished":false,"podcastEpisodeIsExplicit":false,"publishOn":1748732400,"wizardDraftedToPublishOn":"2026-06-01T21:00:00.000Z"}' \
      "https://api-v5.anchor.fm/v3/episodes/${ANCHOR_ID}/update?isMumsCompatible=true"
    ```

---

## 1-D. Get Region

```
GET /v3/region?isMumsCompatible=true
```

Returns the user account's region. Called automatically when loading the
episode detail page.

**Example response:** `{ "region": "JP" }`

---

## 1-E. Get Sponsored Content Status

```
GET /v3/episodes/{ANCHOR_ID}/sponsoredContentStatus?isMumsCompatible=true
```

Returns whether the episode contains sponsored content.

**Response:** `{ "containsSponsoredContent": false }`

---

## 1-F. Get Cover Art Colors

```
POST /MultiGetColor?isMumsCompatible=true
```

Returns the color palette extracted from an image URL. Called when
rendering cover art in the UI.

### Request body

```json
{
  "color_queries": [
    { "url": "https://anchor-generated-image-bank.s3.amazonaws.com/...jpg" }
  ]
}
```

---

## 1-G. Get Episode Form Data (Batch)

```
GET /v3/episodes/pcdFormData?episodeUri[]=spotify:episode:{SPOTIFY_EP_ID}&isMumsCompatible=true
```

You can supply multiple `episodeUri[]` parameters to batch-fetch form data.
Called when the episode list page loads.

---

## 1-H. Get Show Metadata

```
GET /v3/stations/{STATION_ID}/metadata?isMumsCompatible=true
```

Returns show-level information: title, description, categories, cover art,
and distribution flags. Called when the show settings page loads.

The `STATION_ID` is the anchor numeric station ID (distinct from the user
ID). You can obtain it via the ID conversion endpoint (1-L).

### Response fields

| Field | Type | Description |
|-------|------|-------------|
| `rssFeedUrl` | string | RSS feed URL |
| `podcastName` | string | Show title |
| `podcastDescription` | string | Show description |
| `podcastAuthorName` | string | Author name |
| `podcastAuthorEmail` | string | Author email |
| `podcastImage` | string | Cover art URL (original size) |
| `podcastImage400` | string | Cover art URL (400 px) |
| `podcastCategory` | string | Category in `"Arts\|Design"` format |
| `podcastLanguage` | string | Language code (e.g. `"ja"`) |
| `language` | string | Same as `podcastLanguage` |
| `isDistributedOnSpotify` | boolean | Whether the show is on Spotify |
| `isRssFeedDisabled` | boolean | Whether the RSS feed is disabled |
| `hasPublishedEpisode` | boolean | Whether at least one episode is published |
| `distributionEligibilityForPaywalls` | string | Paywall eligibility (`"eligible"` etc.) |
| `isStationOver30DaysOld` | boolean | Whether the show is older than 30 days |
| `isStationMarkedAsSpam` | boolean | Whether the show is flagged as spam |

---

## 1-I. Get Partner IDs

```
POST /v3/{USER_ID}/partnerIds?isMumsCompatible=true
```

Returns partner IDs related to monetisation (ads, Fan Support, etc.).
Called when the show settings page loads.

---

## 1-J. Get User Verification Status

```
GET /v3/users/verification?isMumsCompatible=true
```

Returns the account's identity verification state.

---

## 1-K. Onboarding Status

```
GET /v3/onboarding/redirectStatus/{USER_ID}/?isMumsCompatible=true
GET /v3/onboarding/steps/{USER_ID}/?userId={USER_ID}&isMumsCompatible=true
```

Returns onboarding progress for new users. Has no practical effect on
established accounts.

---

## 1-L. Spotify Show ID → Anchor Numeric IDs

```
GET /v3/shows/{SPOTIFY_SHOW_ID}/legacyIds?isMumsCompatible=true
```

Converts a Spotify Show ID into the anchor.fm internal numeric IDs needed
by most other endpoints.

### Response

```json
{
  "userId": YOUR_STATION_ID,
  "stationId": YOUR_STATION_ID,
  "webStationId": "112fef78c"
}
```

| Field | Description |
|-------|-------------|
| `stationId` | The numeric ID used in episode list, metadata, and other endpoints |
| `userId` | The numeric user account ID |
| `webStationId` | The URL slug (vanity ID) |

=== "Python"
    ```python
    r = requests.get(
        f"https://api-v5.anchor.fm/v3/shows/{spotify_show_id}/legacyIds?isMumsCompatible=true",
        headers=headers_get,
    )
    ids = r.json()
    station_id = ids["stationId"]
    user_id    = ids["userId"]
    ```

=== "Node.js"
    ```javascript
    const r = await fetch(
      `${BASE}/v3/shows/${spotifyShowId}/legacyIds?isMumsCompatible=true`,
      { headers: headersGet },
    );
    const { stationId, userId } = await r.json();
    ```

=== "cURL"
    ```bash
    curl -s \
      -H "Authorization: Bearer $BEARER" \
      "https://api-v5.anchor.fm/v3/shows/${SPOTIFY_SHOW_ID}/legacyIds?isMumsCompatible=true"
    ```

---

## 1-M. Update Episode List Index

```
POST /v3/stations/{STATION_ID}/episode-list-index?isMumsCompatible=true
```

Synchronises the episode list index. Called automatically by the browser
when the episode list page loads.

**Response:** `{ "success": true }`

---

## 1-N. Get Show Form Data

```
GET /v3/stations/{STATION_ID}/pcdForm?isMumsCompatible=true
```

**Response:** `{ "formUrl": "" }`

---

## 1-O. Get Distribution Settings

```
GET /v3/stations/{STATION_ID}/distribution?isMumsCompatible=true
```

Returns the show's distribution and RSS status. Called when the episode
list page loads.

### Response fields

| Field | Type | Description |
|-------|------|-------------|
| `canEnableRss` | boolean | Whether RSS distribution can be enabled |
| `externalUrls` | object | E.g. `{"spotify": "https://open.spotify.com/show/..."}` |
| `spotifyDistributionStatus` | string | Spotify distribution status |
| `spotifyShowUri` | string | Spotify Show URI |
| `platforms` | array | List of active distribution platforms |
| `vanitySlug` | string | URL slug |
| `isEnterprisePodcast` | boolean | Enterprise plan flag |

---

## 1-P. Get User Settings

```
GET /v3/users/{USER_ID}/settings/termsAndContactability?isMumsCompatible=true
```

### Example response

```json
{
  "termsOfServiceVersion": 1,
  "agreedToMinimumRequiredTermsOfService": true,
  "contactabilityStatus": "optedOut",
  "countryRule": "needsResponseOptInSelected"
}
```

---

## 1-Q. Update Sponsored Content Status

```
PUT /v3/episodes/{ANCHOR_ID}/sponsoredContentStatus?isMumsCompatible=true
```

Sets whether the episode contains sponsored content. Called automatically
when saving an episode.

### Request body

```json
{
  "containsSponsoredContent": false,
  "publishOn": 1748732022
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `containsSponsoredContent` | boolean | Yes | Whether the episode includes sponsored content |
| `publishOn` | integer | Yes | Publish time as a **Unix timestamp in seconds** (not ISO 8601) |

!!! warning
    `publishOn` here is a **Unix timestamp (integer seconds)**, not an ISO 8601
    string. Passing an ISO 8601 string will silently corrupt the value.

---

## 1-R. Sync Monetisation Lifecycle

```
POST /v5/monetization-lifecycle/{STATION_ID}/sync-and-fetch?isMumsCompatible=true
```

Synchronises and fetches the monetisation state (e.g. Spotify Partner
Program). Called when the episode list page loads.

### Key response fields

| Field | Description |
|-------|-------------|
| `products.SPOTIFY_PARTNER_PROGRAM.lifecycleState` | Partner program state |
| `products.SPOTIFY_PARTNER_PROGRAM.pendingRequirements` | Outstanding requirements |

---

## 1-S. Create a New Episode

```
POST /v3/stations/{STATION_ID}/episodes?isMumsCompatible=true
```

Creates an empty episode record. This is the first step in the episode
wizard — it allocates the episode ID before audio or metadata are set.

### Request body

```json
{ "hourOffset": -9 }
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `hourOffset` | integer | User timezone offset (JST = `-9`) |

### Response

```json
{
  "episodeId": 120529038,
  "stationId": YOUR_STATION_ID
}
```

!!! note "Variant used by the web app: `returnWebIds=true` (2026-11)"
    The S4C web app calls

    ```
    POST /v3/stations/{STATION_ID}/episodes?isMumsCompatible=true&returnWebIds=true
    ```

    with the same `{ "hourOffset": … }` body, and receives the Spotify-side
    identifiers instead of the numeric ID:

    ```json
    { "episodeUri": "spotify:episode:…", "webEpisodeId": "…" }
    ```

    It then resolves the numeric ID with
    `GET /v3/episodes/{episodeUri}/episodeId?isMumsCompatible=true`
    (same shape as 1-A). Without `returnWebIds` the response is
    `{ "episodeId": … }` as above. Handle both.

=== "Node.js"
    ```javascript
    const hourOffset = new Date().getTimezoneOffset() / 60; // JST → -9
    const c = await (await fetch(
      `${BASE}/v3/stations/${stationId}/episodes?isMumsCompatible=true&returnWebIds=true`,
      { method: "POST", headers: headersPost, body: JSON.stringify({ hourOffset }) },
    )).json();
    const anchorId = c.episodeId ?? (await (await fetch(
      `${BASE}/v3/episodes/${c.episodeUri}/episodeId?isMumsCompatible=true`,
      { headers: headersGet },
    )).json()).episodeId;
    ```

!!! note "Three-step episode creation"
    1. `POST /v3/stations/{STATION_ID}/episodes` — allocate the episode record (this endpoint)
    2. Upload audio/video via the media upload flow (6-A through 6-D)
    3. `POST /v3/episodes/{ANCHOR_ID}/update` (1-C) — set title, description, and publish settings

    Optionally between steps 2 and 3: episode image (1-V) and, for video
    episodes, the horizontal video thumbnail (GraphQL —
    [Episode video thumbnail upload](graphql.md#episode-video-thumbnail-upload)).

---

## 1-T. Update Show Metadata

```
POST /v3/stations/{STATION_ID}/metadata?isMumsCompatible=true
```

Updates show-level metadata: title, description, author, category, language,
and explicit flag. Called when the podcast settings page is saved.

### Request body

```json
{
  "podcastName": "Your Podcast Title",
  "podcastDescription": "Show description",
  "podcastAuthorName": "Author Name",
  "podcastCategory": "Arts|Design",
  "podcastLanguage": "ja",
  "podcastEpisodeIsExplicit": false
}
```

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `podcastName` | string | Yes | Show title (max 100 characters) |
| `podcastDescription` | string | No | Show description (max 600 characters) |
| `podcastAuthorName` | string | No | Author name (max 80 characters) |
| `podcastCategory` | string | No | Category in `"Arts\|Design"` format; fetch valid values from `/v3/settings/podcast-category/options` |
| `podcastLanguage` | string | No | Language code (e.g. `"ja"`) |
| `podcastEpisodeIsExplicit` | boolean | No | Explicit content flag |

**HTTP 200** returns the updated metadata object.

---

## 1-U. Update Vanity Slug

```
POST /v3/settings?isMumsCompatible=true
```

Updates the show's custom URL slug.

### Request body

```json
{ "webStationId": "your-vanity-slug" }
```

!!! warning
    `webStationId` must be globally unique. Using an already-taken slug may
    return **409 Conflict**.

---

## 1-V. Image Upload Flow (Cover Art / Episode Image)

Show cover art and the square **episode image** ("Episode art" in the web
app) share the same upload pipeline but use different endpoints. The
pipeline is **separate** from the media upload pipeline (6-A through 6-D).

!!! note "Episode image ≠ video thumbnail"
    For video episodes the web app has a second, horizontal image — the
    **video thumbnail**. It is uploaded through GraphQL, not through this
    flow: see [Episode video thumbnail upload](graphql.md#episode-video-thumbnail-upload).

### 1-V-1. Get a Signed Upload URL

**Show cover art:**
```
GET /v3/stations/{STATION_ID}/images/upload/signed_url?isMumsCompatible=true
```

Response:
```json
{
  "url": "https://storage.googleapis.com/anchor-image-storage-production/{UUID}/{FILENAME}?X-Goog-Algorithm=...",
  "uploadId": "a46e21a4-1d2d-1686-1355-905e191a3bef"
}
```

**Episode image** (verified 2026-11-21):
```
GET /v3/episodes/{ANCHOR_ID}/images/upload/signed_url?contentType=image%2Fjpeg&stationId={STATION_ID}&isMumsCompatible=true
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `contentType` | **Yes** | URL-encoded MIME type (`image%2Fjpeg`, `image%2Fpng`). Without it: HTTP 400 `request/query must have required property 'contentType'` |
| `stationId` | Yes (sent by the web app) | Numeric station ID (1-L) |

Response — an **S3** variant, not the GCS `{url, uploadId}` shape:
```json
{
  "requestUuid": "…",
  "signedUrl": "https://anchor-img-uploads-production.s3.us-east-1.amazonaws.com/…"
}
```

Use `requestUuid` wherever the steps below say `{UPLOAD_ID}`. Code that
must handle both shapes can read `url ?? signedUrl` and
`uploadId ?? requestUuid`.

### 1-V-2. PUT the Image to Storage

```
PUT {url | signedUrl}
Content-Type: image/jpeg   # must match contentType from 1-V-1
```

No auth headers needed — the signature is embedded in the URL.

### 1-V-3. Notify Upload Completion

**Show cover art:**
```
POST /v3/stations/{STATION_ID}/images/upload/{UPLOAD_ID}/process_upload?isMumsCompatible=true
```
Body: `{}` or empty.

**Episode image:**
```
POST /v3/episodes/{ANCHOR_ID}/images/upload/{UPLOAD_ID}/process_upload?isMumsCompatible=true
```

Body:
```json
{
  "episodeId": 120520683,
  "uploadType": "episode_image"
}
```

### 1-V-4. Poll for Completion

**Show cover art:**
```
GET /v3/stations/{STATION_ID}/images/upload/{UPLOAD_ID}/?isMumsCompatible=true
```

**Episode image:**
```
GET /v3/episodes/{ANCHOR_ID}/images/upload/{UPLOAD_ID}/?isMumsCompatible=true
```

Poll until `status` reaches its final value (typically a few seconds). For
the episode image the observed sequence is `uploaded` → **`processed`**
(2026-11-21), not `completed`; accept either. The poll may return 404
briefly right after `process_upload`.

=== "Python"
    ```python
    import time, requests

    r = requests.get(
        f"{BASE}/v3/episodes/{anchor_id}/images/upload/signed_url"
        f"?contentType=image%2Fjpeg&stationId={station_id}&isMumsCompatible=true",
        headers=headers_get,
    )
    r.raise_for_status()
    s = r.json()
    url = s.get("url") or s["signedUrl"]
    upload_id = s.get("uploadId") or s["requestUuid"]

    with open("cover.jpg", "rb") as f:
        requests.put(url, data=f.read(), headers={"Content-Type": "image/jpeg"}).raise_for_status()

    requests.post(
        f"{BASE}/v3/episodes/{anchor_id}/images/upload/{upload_id}/process_upload?isMumsCompatible=true",
        json={"episodeId": anchor_id, "uploadType": "episode_image"},
        headers=headers_post,
    ).raise_for_status()

    for _ in range(30):
        r = requests.get(
            f"{BASE}/v3/episodes/{anchor_id}/images/upload/{upload_id}/?isMumsCompatible=true",
            headers=headers_get,
        )
        if r.status_code != 404 and r.json().get("status") in ("processed", "completed"):
            break
        time.sleep(2)
    ```

=== "Node.js"
    ```javascript
    import fs from "node:fs";

    const s = await (await fetch(
      `${BASE}/v3/episodes/${anchorId}/images/upload/signed_url` +
        `?contentType=image%2Fjpeg&stationId=${stationId}&isMumsCompatible=true`,
      { headers: headersGet },
    )).json();
    const url = s.url ?? s.signedUrl;
    const uploadId = s.uploadId ?? s.requestUuid;

    const put = await fetch(url, {
      method: "PUT",
      headers: { "Content-Type": "image/jpeg" },
      body: fs.readFileSync("cover.jpg"),
    });
    if (!put.ok) throw new Error(`PUT image -> ${put.status}`);

    await fetch(
      `${BASE}/v3/episodes/${anchorId}/images/upload/${uploadId}/process_upload?isMumsCompatible=true`,
      { method: "POST", headers: headersPost,
        body: JSON.stringify({ episodeId: anchorId, uploadType: "episode_image" }) },
    );

    for (let i = 0; i < 30; i++) {
      const r = await fetch(
        `${BASE}/v3/episodes/${anchorId}/images/upload/${uploadId}/?isMumsCompatible=true`,
        { headers: headersGet },
      );
      if (r.ok && ["processed", "completed"].includes((await r.json()).status)) break;
      await new Promise((res) => setTimeout(res, 2000));
    }
    ```

---

## 1-W. Monetisation Endpoints

Called from the monetisation page (`/pod/show/{SHOW_ID}/monetize/...`):

```
GET /v4/paywalls/{STATION_ID}/lifecycle-state?isMumsCompatible=true
GET /v6/wallet/{STATION_ID}/availability?isMumsCompatible=true
GET /v6/wallet/{STATION_ID}/?isMumsCompatible=true
GET /v3/paywalls/{STATION_ID}/?isMumsCompatible=true
```

These endpoints return paywall and wallet state. Used together with 1-R.

---

## 1-X. Show Settings Page Endpoints

Called from the podcast settings page (`/pod/show/{SHOW_ID}/podcast/edit`):

```
GET /v3/stations/{STATION_ID}/episodePage?isMumsCompatible=true
GET /v3/station/{STATION_ID}/episodes/featured?isMumsCompatible=true
GET /v3/stations/{STATION_ID}/hostRecommendations?isMumsCompatible=true
```

These return "Start Here" featured episode and host recommendation settings.

---

## 1-Y. Distribution Page Endpoints

Called from the distribution page (`/pod/show/{SHOW_ID}/podcast/distribution`):

```
GET /v3/onboarding/redirectStatus/{STATION_ID}/?isMumsCompatible=true
GET /v3/onboarding/steps/{STATION_ID}/?userId={USER_ID}&isMumsCompatible=true
GET /v3/stations/status?isMumsCompatible=true
```

`/v3/stations/status` returns a list of all shows associated with the
currently authenticated user.

---

## 1-Z. Delete Episode

```
POST /v3/episodes/{ANCHOR_ID}/delete?isMumsCompatible=true
```

Permanently deletes an episode. Works on drafts, scheduled (not yet
published) episodes, and published episodes alike.

### Request body

```json
{}
```

An empty JSON body is sufficient. The episode is identified entirely by the
`{ANCHOR_ID}` path segment (the numeric Anchor ID, not the Spotify episode
URI — resolve it first via 1-A if you only have the Spotify ID).

### Response

```json
{}
```

Returns `200` on success.

!!! warning "Irreversible"
    There is no trash/restore step — a successful call removes the episode
    immediately. Confirm the `{ANCHOR_ID}` resolves to the intended episode
    (e.g. via `/overview`, 1-B) before calling.

```bash
curl -X POST \
  "https://api-v5.anchor.fm/v3/episodes/{ANCHOR_ID}/delete?isMumsCompatible=true" \
  -H "Authorization: Bearer ${BEARER}" \
  -H "Content-Type: application/json" \
  -H "anchor-client-type: web" \
  -H "anchor-api-version: 3.8.3" \
  -d '{}'
```

---

## Known Quirks

### description field — server-side transformations

When the server stores the `description` field, it automatically applies
these transformations:

| Input | Stored as |
|-------|-----------|
| `<br>` | `<br />` |
| `<a href="...">` | `<a href="..." rel="ugc noopener noreferrer" target="_blank">` |

If you compare the raw value from `/overview` against the value you
originally posted, they will not match exactly. Normalise both sides before
comparing.

### publishOn must be a Unix timestamp integer — not an ISO 8601 string

Passing an ISO 8601 string (e.g. `"2026-06-01T21:00:00.000Z"`) as `publishOn`
in the update body will silently corrupt the value. The server parses only the
leading digits as an integer — `"2026"` becomes `2026` seconds from epoch,
producing `"1970-01-01T00:33:46.000Z"`.

Always pass `publishOn` as a Unix timestamp integer:

```python
publishOn = int(datetime(2026, 6, 1, 21, 0, 0, tzinfo=timezone.utc).timestamp())
# → 1748732400
```

`wizardDraftedToPublishOn` behaves oppositely — it requires an ISO 8601 string
and returns HTTP 400 if given an integer. See section 1-C for the full example.

---

## Media Upload Flow (Audio / Video)

See [Recipes — Media upload](recipes.md#media-upload) for the full
end-to-end Python example using endpoints **6-A through 6-D**.

### Supported formats

`mp3`, `m4a`, `aifc`, `aiff`, `ogg`, `wav`, `flac`, `mp4`, `mov`

### File size limits

| Type | Limit |
|------|-------|
| Video | 4 GB |
| Audio | 500 MB |

### 6-A. Get a Signed Upload URL

**Audio:**
```
GET /v3/episodes/{ANCHOR_ID}/upload/signedUrl?filename={FILENAME}&type={MIME_TYPE}&isMumsCompatible=true
```

**Video — multipart** (verified 2026-11-21):
```
GET /v3/episodes/{ANCHOR_ID}/upload/signedUrl?filename={FILENAME}&type=video%2Fmp4&isMumsCompatible=true&isMultipartUpload=true&numParts={N}&uploadType=video
```

| Parameter | Example | Description |
|-----------|---------|-------------|
| `filename` | `episode.mp4` | File name to upload (URL-encoded). The web app replaces `_` with `-` |
| `type` | `video%2Fmp4` | MIME type, **must be URL-encoded** — see warning below |
| `isMultipartUpload` | `true` | Video only — required to get GCS part URLs |
| `numParts` | `2` | Video only — number of parts you will upload |
| `uploadType` | `video` | Video only |

!!! warning "`type` must be URL-encoded"
    `type=video/mp4` returns HTTP 400
    `Parameter 'type' must be url encoded. Its value may not contain reserved characters.`
    Send `type=video%2Fmp4` (`encodeURIComponent` / `urllib.parse.quote(…, safe="")`).

!!! warning "Video without the multipart parameters fails later"
    Without `isMultipartUpload` / `numParts` / `uploadType`, a video request
    receives the **audio (S3)** variant below, and the subsequent
    `process_upload` with `uploadType: "video"` fails with **HTTP 500**.

**Response — audio** (AWS S3):
```json
{
  "requestUuid": "b8c96d9c-1d37-45a5-7ba3-2dcfb17e1fdc",
  "fileKey": "b8c96d9c-1d37-45a5-7ba3-2dcfb17e1fdc/episode.mp3",
  "signedUrl": "https://audio-uploaded-default-production.s3.us-west-1.amazonaws.com/..."
}
```

**Response — video multipart** (Google Cloud Storage):
```json
{
  "fileKey": "…",
  "filename": "episode.mp4",
  "requestUuid": "…",
  "uploadId": "…",
  "signedUrlParts": [
    { "partNumber": 1, "url": "https://storage.googleapis.com/anchor_exclusive_media_upload_production/{UUID}/episode.mp4-1?…" },
    { "partNumber": 2, "url": "https://storage.googleapis.com/anchor_exclusive_media_upload_production/{UUID}/episode.mp4-2?…" }
  ]
}
```

!!! note "Older single-URL video variant"
    Earlier research (2026-05) recorded a video response of the form
    `{ "uploadId", "url", "headers" }`. It was not seen in 2026-11; the
    multipart shape above is what the web app uses now.

### 6-B. PUT the File to Storage

**Audio:**
```
PUT {signedUrl}
Content-Type: {MIME_TYPE}
```

**Video (multipart):** split the file into `N` equal consecutive byte
ranges (the last one may be shorter) and `PUT` each range to the `url` of
the matching `partNumber`. No auth headers — the signature is in the URL.

```
PUT {signedUrlParts[k].url}      # body = bytes of part k
```

Save the `ETag` response header of **every** part, with the surrounding
quotes stripped — they go into `parts` in 6-C.

!!! tip "Choosing `N`"
    The web app split a 77 MB file into 2 parts of `ceil(size / 2)` bytes.
    `N = ceil(size / 50 MiB)` (2 parts for 60–180 MB files) worked for 28
    uploads of 60–180 MB (2026-11-21).

### 6-C. Notify Upload Completion

```
POST /v3/upload/{requestUuid}/process_upload?isMumsCompatible=true
```

For both audio and video use `requestUuid` from 6-A in the path and as
`uploadId` in the body.

**Request body (audio):**
```json
{
  "userId": YOUR_USER_ID,
  "uploadType": "default",
  "origin": "episode-media:upload",
  "caption": "episode.mp3",
  "isExtractedFromVideo": false,
  "isMultipartUpload": true,
  "parts": [{ "partNumber": 1, "etag": "9e3290d363f9552ada7cb4fa412a9fc7" }],
  "uploadId": "b8c96d9c-1d37-45a5-7ba3-2dcfb17e1fdc",
  "episodeId": 120525547,
  "stationId": YOUR_STATION_ID
}
```

**Request body (video):**
```json
{
  "userId": YOUR_USER_ID,
  "uploadType": "video",
  "origin": "episode-media:upload",
  "caption": "episode.mp4",
  "isExtractedFromVideo": true,
  "isMultipartUpload": true,
  "parts": [
    { "partNumber": 1, "etag": "4d49fffd0ed1c6a291577ac47d26b997" },
    { "partNumber": 2, "etag": "0c2f7a3b6e8d4f1a9b5c7d3e2f1a0b9c" }
  ],
  "uploadId": "{requestUuid from 6-A}",
  "episodeId": 120525547,
  "stationId": YOUR_STATION_ID
}
```

| Parameter | Type | Description |
|-----------|------|-------------|
| `uploadType` | string | `"default"` for audio; `"video"` for video (**not** `"audio"`) |
| `isExtractedFromVideo` | boolean | `true` for mp4/mov; `false` for audio-only |
| `parts` | array | One `{partNumber, etag}` per uploaded part |
| `uploadId` | string | `requestUuid` from 6-A (audio and video) |

Response (video): `{ "requestUuid": "…" }`.

### 6-D. Poll for Media Validation

```
GET /v3/upload/media/{requestUuid}?includeMediaValidation=true&isMumsCompatible=true
```

**Audio** — response fields:

| Response field | Description |
|----------------|-------------|
| `status` | `"processing"` / `"completed"` / `"failed"` |
| `mediaType` | `"video"` / `"audio"` |
| `durationInMilliseconds` | Duration in ms |
| `validationStatus` | Validation result |

Poll until `status` is `"completed"`.

**Video (multipart)** — a different shape (verified 2026-11-21):

```json
{
  "request": {
    "requestUuid": "…",
    "state": "processed",
    "uploadType": "video",
    "storageProviderType": "GCP",
    "failureReason": null
  },
  "data": { "duration": 312480 },
  "isAnchorFormatFileReady": true,
  "mediaValidation": { "status": "…" }
}
```

`request.state` goes `requested` → `uploaded` → **`processed`**; stop at
`processed` (usually 5–15 s after 6-C). `data.duration` is in milliseconds.
Treat `request.failureReason` or `mediaValidation.status === "failed"` as
an error.

!!! note "Audio polling may return 404 transiently"
    For audio (S3) uploads, the poll endpoint sometimes returns HTTP 404
    for 30–60 seconds after `process_upload` completes before the record
    becomes available. Retry with backoff up to ~300 seconds.

See [Recipes — Video upload with multipart and thumbnails](recipes.md#video-upload-with-multipart-and-thumbnails)
for a complete Python and Node.js script.
