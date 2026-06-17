# Recipes

End-to-end Python implementation examples for common workflows.

---

## Setup

Install the required packages:

```bash
pip install requests spotifyconnector python-dotenv
```

Create a `.env` file:

```ini
SP_DC=your_sp_dc_value_here
SP_KEY=your_sp_key_value_here
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
sp_key  = os.environ["SP_KEY"]
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
    sp_key=sp_key,
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
sp_key  = os.environ["SP_KEY"]
show_id = os.environ["SHOW_ID"]

BASE = "https://api-v5.anchor.fm"

connector = SpotifyConnector(
    base_url="https://generic.wg.spotify.com/podcasters/v0",
    client_id="05a1371ee5194c27860b3ff3ff3979d2",
    podcast_id=show_id,
    sp_dc=sp_dc,
    sp_key=sp_key,
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
sp_key  = os.environ["SP_KEY"]
show_id = os.environ["SHOW_ID"]

SPOTIFY_EP_ID = "1xouj0WrH2klavXKzDWZbq"
GRAPHQL_URL   = "https://creators-graph.spotify.com/v2/graph-pq"

connector = SpotifyConnector(
    base_url="https://generic.wg.spotify.com/podcasters/v0",
    client_id="05a1371ee5194c27860b3ff3ff3979d2",
    podcast_id=show_id,
    sp_dc=sp_dc,
    sp_key=sp_key,
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
sp_key  = os.environ["SP_KEY"]
show_id = os.environ["SHOW_ID"]

COMMENT_URI   = "spotify:comment:7cXkaBSy1gocZ0hAtS9WsJ"
REPLY_TEXT    = "Thank you for your comment!"
GRAPHQL_URL   = "https://creators-graph.spotify.com/v2/graph-pq"

connector = SpotifyConnector(
    base_url="https://generic.wg.spotify.com/podcasters/v0",
    client_id="05a1371ee5194c27860b3ff3ff3979d2",
    podcast_id=show_id,
    sp_dc=sp_dc,
    sp_key=sp_key,
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

Upload an audio or video file to an existing episode.

```python
import os
import time
import requests
from dotenv import load_dotenv
from spotifyconnector import SpotifyConnector

load_dotenv()
sp_dc   = os.environ["SP_DC"]
sp_key  = os.environ["SP_KEY"]
show_id = os.environ["SHOW_ID"]

FILE_PATH     = "/path/to/episode.mp3"
SPOTIFY_EP_ID = "1xouj0WrH2klavXKzDWZbq"
BASE          = "https://api-v5.anchor.fm"

connector = SpotifyConnector(
    base_url="https://generic.wg.spotify.com/podcasters/v0",
    client_id="05a1371ee5194c27860b3ff3ff3979d2",
    podcast_id=show_id,
    sp_dc=sp_dc,
    sp_key=sp_key,
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
