# Recipes

End-to-end implementation examples for common workflows — Python, and
Node.js where noted.

---

## Setup

Install the required packages:

=== "Python"
    ```bash
    pip install requests spotifyconnector python-dotenv
    ```

=== "Node.js"
    Node 20 or newer — no packages needed (built-in `fetch`, `crypto`,
    `fs.openAsBlob`). For authentication see
    [Authentication — Step 3 (Node.js)](auth.md#step-3-obtain-a-bearer-token).
    Pass the variables on the command line
    (`SP_DC=… SHOW_ID=… node script.mjs`) or use `node --env-file=.env`.

Create a `.env` file:

```ini
SP_DC=your_sp_dc_value_here
SHOW_ID=your_show_id_here
```

---

## Recipe 1 — Update Episode Metadata

Update the title, description, and scheduled publish time of an existing
episode identified by its Spotify Episode ID.

```python
import os
import requests
from dotenv import load_dotenv
from spotifyconnector import SpotifyConnector
from datetime import datetime, timezone, timedelta

load_dotenv()
sp_dc   = os.environ["SP_DC"]
show_id = os.environ["SHOW_ID"]

# --- Configuration ---
SPOTIFY_EP_ID = "1xouj0WrH2klavXKzDWZbq"   # target episode
NEW_TITLE     = "Ep. 42 — How Podcasts Work"
NEW_DESC      = "<p>Episode description in HTML.</p>"
# Publish at 06:00 JST → 21:00 UTC previous day
JST = timezone(timedelta(hours=9))
publish_jst  = datetime(2026, 6, 1, 6, 0, 0, tzinfo=JST)
pub_utc      = publish_jst.astimezone(timezone.utc)
# publishOn expects a Unix integer; wizardDraftedToPublishOn expects ISO 8601
PUBLISH_ON_UNIX = int(pub_utc.timestamp())
PUBLISH_ON_ISO  = pub_utc.strftime("%Y-%m-%dT%H:%M:%S.000Z")

BASE = "https://api-v5.anchor.fm"

# 1. Obtain Bearer token
connector = SpotifyConnector(
    base_url="https://generic.wg.spotify.com/podcasters/v0",
    client_id="05a1371ee5194c27860b3ff3ff3979d2",
    podcast_id=show_id,
    sp_dc=sp_dc,
    sp_key="",  # required arg, but ignored — sp_dc alone authenticates
)
connector._authenticate()
bearer = connector._bearer

headers_get  = {"Authorization": f"Bearer {bearer}", "Accept": "application/json"}
headers_post = {
    **headers_get,
    "Content-Type": "application/json",
    "Origin": "https://creators.spotify.com",
    "Referer": "https://creators.spotify.com/",
}

# 2. Convert Spotify Episode ID → Anchor numeric ID
r = requests.get(
    f"{BASE}/v3/episodes/spotify:episode:{SPOTIFY_EP_ID}/episodeId?isMumsCompatible=true",
    headers=headers_get,
)
r.raise_for_status()
anchor_id = r.json()["episodeId"]
print(f"Anchor ID: {anchor_id}")

# 3. Fetch current episode state (required to preserve existing fields)
r = requests.get(
    f"{BASE}/v3/episodes/{anchor_id}/overview?isMumsCompatible=true&returnWebIds=true",
    headers=headers_get,
)
r.raise_for_status()
overview = r.json()

# 4. Update the episode
payload = {
    "userId":                   overview["userId"],
    "title":                    NEW_TITLE,
    "description":              NEW_DESC,
    "episodeType":              overview.get("podcastEpisodeType", "full"),
    "isPublished":              overview.get("isPublished", False),
    "podcastEpisodeIsExplicit": overview.get("podcastEpisodeIsExplicit", False),
    "publishOn":                PUBLISH_ON_UNIX,   # Unix integer (ISO string silently corrupts)
    "wizardDraftedToPublishOn": PUBLISH_ON_ISO,    # ISO 8601 string (integer returns HTTP 400)
}
r = requests.post(
    f"{BASE}/v3/episodes/{anchor_id}/update?isMumsCompatible=true",
    json=payload,
    headers=headers_post,
)
r.raise_for_status()
print("Update successful:", r.json().get("title"))

# 5. Verify scheduled state
r = requests.get(
    f"{BASE}/v3/episodes/{anchor_id}/overview?isMumsCompatible=true",
    headers=headers_get,
)
ep = r.json()
if not ep["isPublished"] and ep.get("publishOn"):
    print(f"Scheduled for: {ep['publishOn']}")
```

---

## Recipe 2 — Fetch Show IDs

Convert a Spotify Show ID into the anchor.fm numeric IDs required by
most endpoints.

```python
import os
import requests
from dotenv import load_dotenv
from spotifyconnector import SpotifyConnector

load_dotenv()
sp_dc   = os.environ["SP_DC"]
show_id = os.environ["SHOW_ID"]

BASE = "https://api-v5.anchor.fm"

connector = SpotifyConnector(
    base_url="https://generic.wg.spotify.com/podcasters/v0",
    client_id="05a1371ee5194c27860b3ff3ff3979d2",
    podcast_id=show_id,
    sp_dc=sp_dc,
    sp_key="",  # required arg, but ignored — sp_dc alone authenticates
)
connector._authenticate()
bearer = connector._bearer
headers_get = {"Authorization": f"Bearer {bearer}", "Accept": "application/json"}

r = requests.get(
    f"{BASE}/v3/shows/{show_id}/legacyIds?isMumsCompatible=true",
    headers=headers_get,
)
r.raise_for_status()
ids = r.json()
print(f"User ID:    {ids['userId']}")
print(f"Station ID: {ids['stationId']}")
print(f"Web ID:     {ids['webStationId']}")
```

---

## Recipe 3 — Fetch All Comments for an Episode

Retrieve all published and pending-review comments for a given episode.

```python
import os
import requests
from dotenv import load_dotenv
from spotifyconnector import SpotifyConnector

load_dotenv()
sp_dc   = os.environ["SP_DC"]
show_id = os.environ["SHOW_ID"]

SPOTIFY_EP_ID = "1xouj0WrH2klavXKzDWZbq"
GRAPHQL_URL   = "https://creators-graph.spotify.com/v2/graph-pq"

connector = SpotifyConnector(
    base_url="https://generic.wg.spotify.com/podcasters/v0",
    client_id="05a1371ee5194c27860b3ff3ff3979d2",
    podcast_id=show_id,
    sp_dc=sp_dc,
    sp_key="",  # required arg, but ignored — sp_dc alone authenticates
)
connector._authenticate()
bearer = connector._bearer

headers = {
    "Authorization": f"Bearer {bearer}",
    "Content-Type": "application/json",
    "Accept": "application/json",
}

payload = {
    "operationName": "getCommentsForEpisode",
    "variables": {
        "episodeUri": f"spotify:episode:{SPOTIFY_EP_ID}",
        "primaryFilters": [
            "LIST_COMMENT_PRIMARY_FILTER_PUBLISHED",
            "LIST_COMMENT_PRIMARY_FILTER_NEEDS_REVIEW",
        ],
        "commentTypesFilters": [
            "LIST_COMMENT_TYPE_FILTER_ROOT",
            "LIST_COMMENT_TYPE_FILTER_REPLY",
        ],
        "secondaryFilters": [],
        "repliesFilter": [
            "LIST_COMMENT_PRIMARY_FILTER_PUBLISHED",
            "LIST_COMMENT_PRIMARY_FILTER_NEEDS_REVIEW",
        ],
        "pageSize": 15,
    },
    "query": "",   # server uses persisted query; include the query body if needed
}

r = requests.post(GRAPHQL_URL, json=payload, headers=headers)
r.raise_for_status()
data = r.json()
print(data)
```

---

## Recipe 4 — Approve and Reply to a Comment

Approve a pending comment and post a creator reply.

```python
import os
import requests
from dotenv import load_dotenv
from spotifyconnector import SpotifyConnector

load_dotenv()
sp_dc   = os.environ["SP_DC"]
show_id = os.environ["SHOW_ID"]

COMMENT_URI   = "spotify:comment:7cXkaBSy1gocZ0hAtS9WsJ"
REPLY_TEXT    = "Thank you for your comment!"
GRAPHQL_URL   = "https://creators-graph.spotify.com/v2/graph-pq"

connector = SpotifyConnector(
    base_url="https://generic.wg.spotify.com/podcasters/v0",
    client_id="05a1371ee5194c27860b3ff3ff3979d2",
    podcast_id=show_id,
    sp_dc=sp_dc,
    sp_key="",  # required arg, but ignored — sp_dc alone authenticates
)
connector._authenticate()
bearer = connector._bearer

headers = {
    "Authorization": f"Bearer {bearer}",
    "Content-Type": "application/json",
    "Accept": "application/json",
}

def graphql(operation, variables):
    r = requests.post(
        GRAPHQL_URL,
        json={"operationName": operation, "variables": variables, "query": ""},
        headers=headers,
    )
    r.raise_for_status()
    return r.json()

# Approve
graphql("publishCommentByCommentUri", {"commentUri": COMMENT_URI})
print("Comment approved")

# Reply
graphql(
    "createCommentReplyByCommentUri",
    {"parentCommentUri": COMMENT_URI, "replyStr": REPLY_TEXT},
)
print("Reply posted")
```

---

## Media Upload

Upload an audio file to an existing episode.

!!! warning "Video files"
    The video branch below (single URL, `uploadId`/`url`) reflects 2026-05
    research. Since at least 2026-11 video uploads must be **multipart** —
    see [Video upload with multipart and thumbnails](#video-upload-with-multipart-and-thumbnails)
    and [REST 6-A](rest-api.md#6-a-get-a-signed-upload-url).

```python
import os
import time
import requests
from dotenv import load_dotenv
from spotifyconnector import SpotifyConnector

load_dotenv()
sp_dc   = os.environ["SP_DC"]
show_id = os.environ["SHOW_ID"]

FILE_PATH     = "/path/to/episode.mp3"
SPOTIFY_EP_ID = "1xouj0WrH2klavXKzDWZbq"
BASE          = "https://api-v5.anchor.fm"

connector = SpotifyConnector(
    base_url="https://generic.wg.spotify.com/podcasters/v0",
    client_id="05a1371ee5194c27860b3ff3ff3979d2",
    podcast_id=show_id,
    sp_dc=sp_dc,
    sp_key="",  # required arg, but ignored — sp_dc alone authenticates
)
connector._authenticate()
bearer = connector._bearer

headers_get  = {"Authorization": f"Bearer {bearer}", "Accept": "application/json"}
headers_post = {
    **headers_get,
    "Content-Type": "application/json",
    "Origin": "https://creators.spotify.com",
    "Referer": "https://creators.spotify.com/",
}

# Resolve IDs
r = requests.get(
    f"{BASE}/v3/episodes/spotify:episode:{SPOTIFY_EP_ID}/episodeId?isMumsCompatible=true",
    headers=headers_get,
)
anchor_id = r.json()["episodeId"]

r = requests.get(
    f"{BASE}/v3/shows/{show_id}/legacyIds?isMumsCompatible=true",
    headers=headers_get,
)
ids = r.json()
station_id = ids["stationId"]
user_id    = ids["userId"]

# Step 1: Get signed URL
# Audio → S3 (returns requestUuid/signedUrl/fileKey)
# Video → GCS (returns uploadId/url/headers)
filename  = os.path.basename(FILE_PATH)
mime_type = "audio/mp3"   # adjust for video: "video/mp4"
r = requests.get(
    f"{BASE}/v3/episodes/{anchor_id}/upload/signedUrl"
    f"?filename={filename}&type={mime_type}&isMumsCompatible=true",
    headers=headers_get,
)
signed = r.json()

# Detect storage backend by response shape
if "signedUrl" in signed:          # S3 (audio)
    request_uuid = signed["requestUuid"]
    signed_url   = signed["signedUrl"]
else:                               # GCS (video)
    request_uuid = signed["uploadId"]
    signed_url   = signed["url"]

# Step 2: PUT to storage
with open(FILE_PATH, "rb") as f:
    file_data = f.read()

r = requests.put(signed_url, data=file_data, headers={"Content-Type": mime_type})
etag = r.headers["ETag"].strip('"')

# Step 3: Notify completion
payload = {
    "userId":              user_id,
    "uploadType":          "default",        # "default" for audio; "video" for video
    "origin":              "episode-media:upload",
    "caption":             filename,
    "isExtractedFromVideo": False,            # True for mp4/mov
    "isMultipartUpload":   True,
    "parts":               [{"partNumber": 1, "etag": etag}],
    "uploadId":            request_uuid,      # requestUuid for S3, uploadId for GCS
    "episodeId":           anchor_id,
    "stationId":           station_id,
}
requests.post(
    f"{BASE}/v3/upload/{request_uuid}/process_upload?isMumsCompatible=true",
    json=payload,
    headers=headers_post,
).raise_for_status()

# Step 4: Poll until complete (audio may return 404 for ~60s after process_upload)
for _ in range(60):
    r = requests.get(
        f"{BASE}/v3/upload/media/{request_uuid}?includeMediaValidation=true&isMumsCompatible=true",
        headers=headers_get,
    )
    if r.status_code == 404:
        time.sleep(5)
        continue
    status = r.json().get("status")
    if status == "completed":
        print("Upload complete")
        break
    elif status == "failed":
        raise RuntimeError("Upload failed")
    time.sleep(5)
```


---

## Video upload with multipart and thumbnails

Create a new **video** episode from scratch and leave it as a draft (or
publish it): create the episode → upload the MP4 in parts → set the square
episode image → set the horizontal video thumbnail → save title,
description, season and episode number.

**Verified 2026-11-21** — 28 video episodes (60–180 MB each) uploaded,
then published, with the Node.js version. The Python version is a
line-by-line port of it, syntax-checked but not run against the live API.

Endpoints used: [1-L](rest-api.md#1-l-spotify-show-id-anchor-numeric-ids),
[1-S](rest-api.md#1-s-create-a-new-episode),
[6-A … 6-D](rest-api.md#6-a-get-a-signed-upload-url),
[1-V](rest-api.md#1-v-image-upload-flow-cover-art-episode-image),
[GraphQL video thumbnail](graphql.md#episode-video-thumbnail-upload),
[1-C](rest-api.md#1-c-update-episode-title-description-publish-date).

!!! tip "Re-runs"
    Each step is independent once the episode exists. In a real tool, store
    `anchorId` (and the completed steps) after every stage so a failed run
    can resume without creating a second empty episode.

=== "Python"
    ```python
    """Create a draft video episode end to end.

    Steps: create episode → multipart video upload → episode image → video thumbnail
           → title/description/season/episode number as a draft → (optional) publish
    """
    import math
    import os
    import time
    from urllib.parse import quote

    import requests
    from dotenv import load_dotenv
    from spotifyconnector import SpotifyConnector

    load_dotenv()
    sp_dc   = os.environ["SP_DC"]
    show_id = os.environ["SHOW_ID"]

    VIDEO           = "episode.mp4"
    EPISODE_IMAGE   = "cover.jpg"       # square episode image (REST 1-V)
    VIDEO_THUMBNAIL = "thumbnail.png"   # horizontal video thumbnail (GraphQL)
    META = {
        "title": "Episode title",
        "description": "<p>Episode description</p>",
        "seasonNumber": 1,
        "episodeNumber": 1,
    }
    PUBLISH = False

    BASE  = "https://api-v5.anchor.fm"
    GRAPH = "https://creators-graph.spotify.com/v2/graph-pq"
    PQ = {  # persisted-query hashes captured 2026-11-21 — they change with web app deploys
        "getThumbnailSignedUrl":   "3621c97566c4cfdffc7ce3fbba1d886441498ac96ca27b4b594546a83d9bf876",
        "uploadThumbnailComplete": "c43a47136fa24af04869aad730b412789edbfeced6c9aef949a94cb2ca383082",
        "getEpisodeThumbnail":     "bf061fbd2cb87cdbb448e5509efa5e091cd6e2b50ffec8a374a37cee521a6be9",
    }

    connector = SpotifyConnector(
        base_url="https://generic.wg.spotify.com/podcasters/v0",
        client_id="05a1371ee5194c27860b3ff3ff3979d2",
        podcast_id=show_id,
        sp_dc=sp_dc,
        sp_key="",
    )
    connector._authenticate()
    bearer = connector._bearer

    headers_get  = {"Authorization": f"Bearer {bearer}", "Accept": "application/json"}
    headers_post = {
        **headers_get,
        "Content-Type": "application/json",
        "Origin": "https://creators.spotify.com",
        "Referer": "https://creators.spotify.com/",
    }
    graph_headers = {**headers_post, "x-creator-client": "public-website"}  # required for uploadThumbnailComplete


    def api(method, path, body=None, allow_404=False):
        url = BASE + path + ("&" if "?" in path else "?") + "isMumsCompatible=true"
        r = requests.request(method, url, json=body,
                             headers=headers_get if method == "GET" else headers_post, timeout=90)
        if r.status_code == 404 and allow_404:
            return None
        if not r.ok:
            raise RuntimeError(f"{method} {path} -> {r.status_code} {r.text[:300]}")
        return r.json() if r.text else {}


    def graph(op, variables):
        r = requests.post(GRAPH, headers=graph_headers, json={
            "operationName": op,
            "variables": variables,
            "extensions": {"persistedQuery": {"version": 1, "sha256Hash": PQ[op]}},
        })
        j = r.json()
        if not r.ok or j.get("errors"):
            raise RuntimeError(f"{op}: {r.status_code} {j.get('errors') or j}")
        return j["data"]


    # 0. IDs
    ids = api("GET", f"/v3/shows/{show_id}/legacyIds")
    station_id, user_id = ids["stationId"], ids["userId"]

    # 1. Create episode (1-S)
    hour_offset = round(-time.localtime().tm_gmtoff / 3600)   # JST → -9
    created = api("POST", f"/v3/stations/{station_id}/episodes?returnWebIds=true", {"hourOffset": hour_offset})
    anchor_id = created.get("episodeId") or api("GET", f"/v3/episodes/{created['episodeUri']}/episodeId")["episodeId"]
    print("episode", anchor_id)

    # 2. Video, multipart (6-A … 6-D)
    size = os.path.getsize(VIDEO)
    num_parts = max(1, math.ceil(size / (50 * 1024 * 1024)))
    part_size = math.ceil(size / num_parts)
    filename = os.path.basename(VIDEO).replace("_", "-")
    signed = api("GET",
                 f"/v3/episodes/{anchor_id}/upload/signedUrl?filename={quote(filename, safe='')}"
                 f"&type={quote('video/mp4', safe='')}&isMultipartUpload=true&numParts={num_parts}&uploadType=video")
    parts = sorted(signed.get("signedUrlParts", []), key=lambda p: p["partNumber"])
    if len(parts) != num_parts:
        raise RuntimeError(f"expected multipart URLs, got keys: {list(signed)}")
    request_uuid = signed["requestUuid"]

    etags = []
    with open(VIDEO, "rb") as f:
        for p in parts:
            f.seek((p["partNumber"] - 1) * part_size)
            r = requests.put(p["url"], data=f.read(part_size), timeout=600)
            r.raise_for_status()
            etags.append({"partNumber": p["partNumber"], "etag": r.headers["ETag"].strip('"')})
            print(f"part {p['partNumber']}/{num_parts} uploaded")

    api("POST", f"/v3/upload/{request_uuid}/process_upload", {
        "userId": user_id, "uploadType": "video", "origin": "episode-media:upload",
        "caption": os.path.basename(VIDEO), "isExtractedFromVideo": True, "isMultipartUpload": True,
        "parts": etags, "uploadId": request_uuid, "episodeId": anchor_id, "stationId": station_id,
    })

    for i in range(121):
        st = api("GET", f"/v3/upload/media/{request_uuid}?includeMediaValidation=true", allow_404=True) or {}
        req = st.get("request") or {}
        if req.get("state") == "processed":
            print("video processed,", (st.get("data") or {}).get("duration"), "ms")
            break
        if req.get("failureReason") or (st.get("mediaValidation") or {}).get("status") == "failed":
            raise RuntimeError(st)
        time.sleep(5)
    else:
        raise RuntimeError("video processing timed out")

    # 3. Episode image, square (1-V)
    img = api("GET", f"/v3/episodes/{anchor_id}/images/upload/signed_url"
                     f"?contentType=image%2Fjpeg&stationId={station_id}")
    img_url = img.get("url") or img["signedUrl"]
    img_id  = img.get("uploadId") or img["requestUuid"]
    with open(EPISODE_IMAGE, "rb") as f:
        requests.put(img_url, data=f.read(), headers={"Content-Type": "image/jpeg"}).raise_for_status()
    api("POST", f"/v3/episodes/{anchor_id}/images/upload/{img_id}/process_upload",
        {"episodeId": anchor_id, "uploadType": "episode_image"})
    for _ in range(30):
        st = api("GET", f"/v3/episodes/{anchor_id}/images/upload/{img_id}/", allow_404=True)
        if st and st.get("status") in ("processed", "completed"):
            break
        time.sleep(2)
    print("episode image done")

    # 4. Video thumbnail, horizontal (GraphQL)
    episode_uri = api("GET", f"/v3/episodes/{anchor_id}/overview?returnWebIds=true")["spotifyUri"]
    with open(VIDEO_THUMBNAIL, "rb") as f:
        thumb = f.read()                       # read first: the signed URL expires in 10 s
    t = graph("getThumbnailSignedUrl", {
        "contentType": "image/png", "episodeUriV2": episode_uri, "fileName": VIDEO_THUMBNAIL,
        "platformType": "S4C_HOSTED", "thumbnailType": "USER_UPLOADED_THUMBNAIL",
    })["createThumbnailSignedUrl"]
    requests.put(t["signedUrl"], data=thumb, headers={"Content-Type": "image/png"}).raise_for_status()
    graph("uploadThumbnailComplete", {"episodeUri": episode_uri, "thumbnailUuid": t["thumbnailUuid"]})
    for _ in range(30):
        th = graph("getEpisodeThumbnail", {"episodeUri": episode_uri})["episodeByUri"]["thumbnail"]
        if th and th.get("ingestionStatus") == "INGESTION_COMPLETE":
            break
        if th and th.get("error"):
            raise RuntimeError(th)
        time.sleep(3)
    print("video thumbnail done")


    # 5. Metadata as draft (1-C) — seasonNumber / episodeNumber, NOT podcast*Number
    def payload(publish):
        p = {"userId": user_id, **META, "episodeType": "full", "isPublished": publish,
             "podcastEpisodeIsExplicit": False, "isVideoEighteenPlus": False}
        if not publish:
            p["publishOn"] = None
        return p


    api("POST", f"/v3/episodes/{anchor_id}/update", payload(False))
    o = api("GET", f"/v3/episodes/{anchor_id}/overview?returnWebIds=true")
    print("draft saved:", o["title"], "| season", o.get("podcastSeasonNumber"), "| episode", o.get("podcastEpisodeNumber"))

    # 6. Publish (optional)
    if PUBLISH:
        api("POST", f"/v3/episodes/{anchor_id}/update", payload(True))
        print("published:", api("GET", f"/v3/episodes/{anchor_id}/overview?returnWebIds=true")["isPublished"])
    ```

=== "Node.js"
    ```javascript
    // video_upload.mjs — create a draft video episode end to end (Node 20+, no dependencies)
    //   SP_DC=... SHOW_ID=... node video_upload.mjs
    // Steps: create episode → multipart video upload → episode image → video thumbnail
    //        → title/description/season/episode number as a draft → (optional) publish
    import crypto from "node:crypto";
    import fs from "node:fs";

    const SP_DC = process.env.SP_DC;
    const SHOW_ID = process.env.SHOW_ID;            // Spotify show ID (from the S4C URL)
    const VIDEO = "episode.mp4";
    const EPISODE_IMAGE = "cover.jpg";              // square episode image (REST 1-V)
    const VIDEO_THUMBNAIL = "thumbnail.png";        // horizontal video thumbnail (GraphQL)
    const META = {
      title: "Episode title",
      description: "<p>Episode description</p>",
      seasonNumber: 1,
      episodeNumber: 1,
    };
    const PUBLISH = false;                          // true = publish after the draft is saved

    const BASE = "https://api-v5.anchor.fm";
    const GRAPH = "https://creators-graph.spotify.com/v2/graph-pq";
    const CLIENT_ID = "05a1371ee5194c27860b3ff3ff3979d2";
    const REDIRECT_URI = "https://podcasters.spotify.com";
    const PQ = { // persisted-query hashes captured 2026-11-21 — they change with web app deploys
      getThumbnailSignedUrl: "3621c97566c4cfdffc7ce3fbba1d886441498ac96ca27b4b594546a83d9bf876",
      uploadThumbnailComplete: "c43a47136fa24af04869aad730b412789edbfeced6c9aef949a94cb2ca383082",
      getEpisodeThumbnail: "bf061fbd2cb87cdbb448e5509efa5e091cd6e2b50ffec8a374a37cee521a6be9",
    };
    const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

    // ---------- auth: sp_dc → Bearer (PKCE, prompt=none) ----------
    async function bearer(spDc) {
      const rnd = (n) => crypto.randomBytes(n).toString("base64url").slice(0, n);
      const state = rnd(32), verifier = rnd(64);
      const challenge = crypto.createHash("sha256").update(verifier).digest("base64url");
      const auth = new URL("https://accounts.spotify.com/oauth2/v2/auth");
      for (const [k, v] of Object.entries({
        response_type: "code", client_id: CLIENT_ID,
        scope: "streaming ugc-image-upload user-read-email user-read-private",
        redirect_uri: REDIRECT_URI, code_challenge: challenge, code_challenge_method: "S256",
        state, response_mode: "web_message", prompt: "none",
      })) auth.searchParams.set(k, v);
      const html = await (await fetch(auth, { headers: { Cookie: `sp_dc=${spDc}; sp_key=` } })).text();
      if (html.includes("login_required")) throw new Error("sp_dc expired or invalid");
      const block = html.match(/authorizationResponse\s*=\s*([\s\S]*?);/)?.[1] ?? "";
      const code = block.match(/code["']?\s*:\s*["']([^"']+)/)?.[1];
      if (!code) throw new Error("authorization code not found");
      const tok = await (await fetch("https://accounts.spotify.com/api/token", {
        method: "POST",
        headers: { "Content-Type": "application/x-www-form-urlencoded" },
        body: new URLSearchParams({
          grant_type: "authorization_code", client_id: CLIENT_ID, code,
          redirect_uri: REDIRECT_URI, code_verifier: verifier,
        }),
      })).json();
      if (!tok.access_token) throw new Error("token: " + JSON.stringify(tok));
      return tok.access_token;
    }
    const token = await bearer(SP_DC);

    // ---------- helpers ----------
    async function api(method, path, body, { allow404 = false } = {}) {
      const headers = { Authorization: `Bearer ${token}`, Accept: "application/json" };
      if (method !== "GET") Object.assign(headers, {
        "Content-Type": "application/json",
        Origin: "https://creators.spotify.com",
        Referer: "https://creators.spotify.com/",
      });
      const url = BASE + path + (path.includes("?") ? "&" : "?") + "isMumsCompatible=true";
      const r = await fetch(url, {
        method, headers,
        body: body === undefined ? undefined : JSON.stringify(body),
        signal: AbortSignal.timeout(90_000),
      });
      if (r.status === 404 && allow404) return null;
      const text = await r.text();
      if (!r.ok) throw new Error(`${method} ${path} -> ${r.status} ${text.slice(0, 300)}`);
      return text ? JSON.parse(text) : {};
    }

    async function graph(op, variables) {
      const r = await fetch(GRAPH, {
        method: "POST",
        headers: {
          Authorization: `Bearer ${token}`, "Content-Type": "application/json", Accept: "application/json",
          "x-creator-client": "public-website", // required for uploadThumbnailComplete
          Origin: "https://creators.spotify.com", Referer: "https://creators.spotify.com/",
        },
        body: JSON.stringify({ operationName: op, variables,
          extensions: { persistedQuery: { version: 1, sha256Hash: PQ[op] } } }),
      });
      const j = await r.json().catch(() => ({}));
      if (!r.ok || j.errors) throw new Error(`${op}: ${r.status} ${JSON.stringify(j.errors ?? j)}`);
      return j.data;
    }

    // ---------- 0. IDs ----------
    const { stationId, userId } = await api("GET", `/v3/shows/${SHOW_ID}/legacyIds`);

    // ---------- 1. create episode (1-S) ----------
    const hourOffset = new Date().getTimezoneOffset() / 60;
    const created = await api("POST", `/v3/stations/${stationId}/episodes?returnWebIds=true`, { hourOffset });
    const anchorId = created.episodeId
      ?? (await api("GET", `/v3/episodes/${created.episodeUri}/episodeId`)).episodeId;
    console.log("episode", anchorId);

    // ---------- 2. video, multipart (6-A … 6-D) ----------
    const size = fs.statSync(VIDEO).size;
    const numParts = Math.max(1, Math.ceil(size / (50 * 1024 * 1024)));
    const partSize = Math.ceil(size / numParts);
    const filename = VIDEO.replace(/_/g, "-");      // the web app does the same
    const signed = await api("GET",
      `/v3/episodes/${anchorId}/upload/signedUrl?filename=${encodeURIComponent(filename)}` +
      `&type=${encodeURIComponent("video/mp4")}&isMultipartUpload=true&numParts=${numParts}&uploadType=video`);
    const parts = (signed.signedUrlParts ?? []).sort((a, b) => a.partNumber - b.partNumber);
    if (parts.length !== numParts) throw new Error("expected multipart URLs, got: " + Object.keys(signed));
    const requestUuid = signed.requestUuid;

    const file = await fs.openAsBlob(VIDEO, { type: "video/mp4" });
    const etags = [];
    for (const p of parts) {
      const start = (p.partNumber - 1) * partSize;
      const r = await fetch(p.url, { method: "PUT", body: file.slice(start, Math.min(size, start + partSize)) });
      if (!r.ok) throw new Error(`PUT part ${p.partNumber} -> ${r.status}`);
      etags.push({ partNumber: p.partNumber, etag: (r.headers.get("etag") ?? "").replace(/"/g, "") });
      console.log(`part ${p.partNumber}/${numParts} uploaded`);
    }

    await api("POST", `/v3/upload/${requestUuid}/process_upload`, {
      userId, uploadType: "video", origin: "episode-media:upload", caption: VIDEO,
      isExtractedFromVideo: true, isMultipartUpload: true, parts: etags,
      uploadId: requestUuid, episodeId: anchorId, stationId,
    });

    for (let i = 0; ; i++) {
      const st = await api("GET", `/v3/upload/media/${requestUuid}?includeMediaValidation=true`, undefined, { allow404: true });
      const state = st?.request?.state;
      if (state === "processed") { console.log("video processed,", st.data?.duration, "ms"); break; }
      if (st?.request?.failureReason || st?.mediaValidation?.status === "failed") throw new Error(JSON.stringify(st));
      if (i > 120) throw new Error("video processing timed out");
      await sleep(5000);
    }

    // ---------- 3. episode image, square (1-V) ----------
    const img = await api("GET",
      `/v3/episodes/${anchorId}/images/upload/signed_url?contentType=image%2Fjpeg&stationId=${stationId}`);
    const imgUrl = img.url ?? img.signedUrl, imgId = img.uploadId ?? img.requestUuid;
    const putImg = await fetch(imgUrl, { method: "PUT", headers: { "Content-Type": "image/jpeg" }, body: fs.readFileSync(EPISODE_IMAGE) });
    if (!putImg.ok) throw new Error(`PUT episode image -> ${putImg.status}`);
    await api("POST", `/v3/episodes/${anchorId}/images/upload/${imgId}/process_upload`,
      { episodeId: anchorId, uploadType: "episode_image" });
    for (let i = 0; i < 30; i++) {
      const st = await api("GET", `/v3/episodes/${anchorId}/images/upload/${imgId}/`, undefined, { allow404: true });
      if (["processed", "completed"].includes(st?.status)) break;
      await sleep(2000);
    }
    console.log("episode image done");

    // ---------- 4. video thumbnail, horizontal (GraphQL) ----------
    const { spotifyUri: episodeUri } = await api("GET", `/v3/episodes/${anchorId}/overview?returnWebIds=true`);
    const thumb = fs.readFileSync(VIDEO_THUMBNAIL);   // read first: the signed URL expires in 10 s
    const { createThumbnailSignedUrl: t } = await graph("getThumbnailSignedUrl", {
      contentType: "image/png", episodeUriV2: episodeUri, fileName: VIDEO_THUMBNAIL,
      platformType: "S4C_HOSTED", thumbnailType: "USER_UPLOADED_THUMBNAIL",
    });
    const putThumb = await fetch(t.signedUrl, { method: "PUT", headers: { "Content-Type": "image/png" }, body: thumb });
    if (!putThumb.ok) throw new Error(`PUT thumbnail -> ${putThumb.status}`);
    await graph("uploadThumbnailComplete", { episodeUri, thumbnailUuid: t.thumbnailUuid });
    for (let i = 0; i < 30; i++) {
      const th = (await graph("getEpisodeThumbnail", { episodeUri })).episodeByUri?.thumbnail;
      if (th?.ingestionStatus === "INGESTION_COMPLETE") break;
      if (th?.error) throw new Error(JSON.stringify(th));
      await sleep(3000);
    }
    console.log("video thumbnail done");

    // ---------- 5. metadata as draft (1-C) ----------
    const payload = (publish) => ({
      userId, ...META,                 // seasonNumber / episodeNumber — NOT podcast*Number
      episodeType: "full", isPublished: publish, podcastEpisodeIsExplicit: false,
      isVideoEighteenPlus: false, ...(publish ? {} : { publishOn: null }),
    });
    await api("POST", `/v3/episodes/${anchorId}/update`, payload(false));

    const o = await api("GET", `/v3/episodes/${anchorId}/overview?returnWebIds=true`);
    console.log("draft saved:", o.title, "| season", o.podcastSeasonNumber, "| episode", o.podcastEpisodeNumber);

    // ---------- 6. publish (optional) ----------
    if (PUBLISH) {
      await api("POST", `/v3/episodes/${anchorId}/update`, payload(true));
      console.log("published:", (await api("GET", `/v3/episodes/${anchorId}/overview?returnWebIds=true`)).isPublished);
    }
    ```
