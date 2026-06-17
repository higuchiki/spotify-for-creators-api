# レシピ・実装例

anchor.fm REST API と GraphQL API を使った Python 実装の実践的なフロー。

---

## エピソード更新（タイトル・概要欄・公開日時）

最も基本的なユースケース。既存エピソードを更新する標準フロー。

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

# --- 設定 ---
SPOTIFY_EP_ID = "1xouj0WrH2klavXKzDWZbq"   # 対象エピソード
NEW_TITLE     = "Ep. 42 — How Podcasts Work"
NEW_DESC      = "<p>Episode description in HTML.</p>"
# JST 06:00 → UTC 前日 21:00
JST = timezone(timedelta(hours=9))
publish_jst     = datetime(2026, 6, 1, 6, 0, 0, tzinfo=JST)
pub_utc         = publish_jst.astimezone(timezone.utc)
# publishOn は Unix 整数。wizardDraftedToPublishOn は ISO 8601 文字列
PUBLISH_ON_UNIX = int(pub_utc.timestamp())
PUBLISH_ON_ISO  = pub_utc.strftime("%Y-%m-%dT%H:%M:%S.000Z")

BASE = "https://api-v5.anchor.fm"

# 1. Bearer トークン取得
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
headers_post = {
    **headers_get,
    "Content-Type": "application/json",
    "Origin": "https://creators.spotify.com",
    "Referer": "https://creators.spotify.com/",
}

# 2. Spotify Episode ID → Anchor Numeric ID
r = requests.get(
    f"{BASE}/v3/episodes/spotify:episode:{SPOTIFY_EP_ID}/episodeId?isMumsCompatible=true",
    headers=headers_get,
)
r.raise_for_status()
anchor_id = r.json()["episodeId"]
print(f"Anchor ID: {anchor_id}")

# 3. エピソード現状取得（既存フィールドを保持するために必要）
r = requests.get(
    f"{BASE}/v3/episodes/{anchor_id}/overview?isMumsCompatible=true&returnWebIds=true",
    headers=headers_get,
)
r.raise_for_status()
overview = r.json()

# 4. エピソード更新
payload = {
    "userId":                   overview["userId"],
    "title":                    NEW_TITLE,
    "description":              NEW_DESC,
    "episodeType":              overview.get("podcastEpisodeType", "full"),
    "isPublished":              overview.get("isPublished", False),
    "podcastEpisodeIsExplicit": overview.get("podcastEpisodeIsExplicit", False),
    "publishOn":                PUBLISH_ON_UNIX,   # Unix 整数（ISO 文字列を送ると値が破損）
    "wizardDraftedToPublishOn": PUBLISH_ON_ISO,    # ISO 8601 文字列（整数を送ると HTTP 400）
}
r = requests.post(
    f"{BASE}/v3/episodes/{anchor_id}/update?isMumsCompatible=true",
    json=payload,
    headers=headers_post,
)
r.raise_for_status()
print("更新成功:", r.json().get("title"))

# 5. 予約状態の確認
r = requests.get(
    f"{BASE}/v3/episodes/{anchor_id}/overview?isMumsCompatible=true",
    headers=headers_get,
)
ep = r.json()
if not ep["isPublished"] and ep.get("publishOn"):
    print(f"予約済み: {ep['publishOn']}")
```

---

## JST → UTC 変換ユーティリティ

`publishOn` には Unix 整数秒を指定する。JST で考えて変換する場合。

```python
from datetime import datetime, timezone, timedelta

JST = timezone(timedelta(hours=9))

def jst_to_publish_values(year, month, day, hour, minute=0, second=0):
    """JST の日時を publishOn (Unix整数) と wizardDraftedToPublishOn (ISO文字列) に変換する"""
    publish_jst = datetime(year, month, day, hour, minute, second, tzinfo=JST)
    pub_utc = publish_jst.astimezone(timezone.utc)
    return int(pub_utc.timestamp()), pub_utc.strftime("%Y-%m-%dT%H:%M:%S.000Z")

# JST 06:00 → Unix 整数 & ISO 文字列
publish_on_unix, publish_on_iso = jst_to_publish_values(2026, 6, 1, 6, 0, 0)
# publish_on_unix → 1748732400
# publish_on_iso  → "2026-05-31T21:00:00.000Z"
```

---

## エピソード一覧の取得

`spotifyconnector` のネイティブ機能を使う。

```python
connector = SpotifyConnector(
    base_url="https://generic.wg.spotify.com/podcasters/v0",
    client_id="05a1371ee5194c27860b3ff3ff3979d2",
    podcast_id=show_id,
    sp_dc=sp_dc,
    sp_key=sp_key,
)
connector._authenticate()

episodes = connector.episodes()  # ページネーション対応
for ep in episodes:
    print(ep["id"], ep["title"])  # id が Spotify Episode ID
```

---

## Spotify Show ID → Anchor numeric IDs 変換

ショーレベルの操作に必要な `stationId` を取得する。

```python
r = requests.get(
    f"https://api-v5.anchor.fm/v3/shows/{spotify_show_id}/legacyIds?isMumsCompatible=true",
    headers=headers_get,
)
ids = r.json()
station_id = ids["stationId"]   # エピソード一覧・メタデータ等で使う
user_id = ids["userId"]         # エピソード作成等で使う
```

---

## 音声/動画ファイルアップロード（完全フロー）

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

# ID 解決
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

# Step 1: 署名付きURL取得
# 音声 → S3 (requestUuid/signedUrl/fileKey が返る)
# 動画 → GCS (uploadId/url/headers が返る)
filename  = os.path.basename(FILE_PATH)
mime_type = "audio/mp3"   # 動画の場合は "video/mp4"
r = requests.get(
    f"{BASE}/v3/episodes/{anchor_id}/upload/signedUrl"
    f"?filename={filename}&type={mime_type}&isMumsCompatible=true",
    headers=headers_get,
)
signed = r.json()

# レスポンスの形でストレージバックエンドを判定
if "signedUrl" in signed:          # S3（音声）
    request_uuid = signed["requestUuid"]
    signed_url   = signed["signedUrl"]
else:                               # GCS（動画）
    request_uuid = signed["uploadId"]
    signed_url   = signed["url"]

# Step 2: ストレージへの直接 PUT
with open(FILE_PATH, "rb") as f:
    file_data = f.read()

r = requests.put(signed_url, data=file_data, headers={"Content-Type": mime_type})
etag = r.headers["ETag"].strip('"')  # ダブルクォートを除去

# Step 3: process_upload
payload = {
    "userId":              user_id,
    "uploadType":          "default",        # 音声は "default"（"audio" は誤り）。動画は "video"
    "origin":              "episode-media:upload",
    "caption":             filename,
    "isExtractedFromVideo": False,            # mp4/mov は True
    "isMultipartUpload":   True,
    "parts":               [{"partNumber": 1, "etag": etag}],
    "uploadId":            request_uuid,      # S3 の場合は requestUuid、GCS の場合は uploadId
    "episodeId":           anchor_id,
    "stationId":           station_id,
}
requests.post(
    f"{BASE}/v3/upload/{request_uuid}/process_upload?isMumsCompatible=true",
    json=payload,
    headers=headers_post,
).raise_for_status()

# Step 4: 処理完了待ち（ポーリング）
# 音声（S3）は process_upload 完了後 30〜60 秒程度 404 を返すことがある
for _ in range(60):
    r = requests.get(
        f"{BASE}/v3/upload/media/{request_uuid}"
        f"?includeMediaValidation=true&isMumsCompatible=true",
        headers=headers_get,
    )
    if r.status_code == 404:
        time.sleep(5)
        continue
    status = r.json().get("status")
    print(f"status: {status}")
    if status == "completed":
        print("アップロード完了")
        break
    elif status == "failed":
        raise RuntimeError("アップロード失敗")
    time.sleep(5)
```

---

## コメント一覧取得と返信（GraphQL）

```python
import requests

graphql_url = "https://creators-graph.spotify.com/v2/graph-pq"
headers = {
    "Authorization": f"Bearer {bearer}",
    "Content-Type": "application/json",
}
episode_uri = f"spotify:episode:{spotify_ep_id}"

# コメント一覧取得
query_payload = {
    "operationName": "getCommentsForEpisode",
    "variables": {
        "episodeUri": episode_uri,
        "primaryFilters": [
            "LIST_COMMENT_PRIMARY_FILTER_PUBLISHED",
            "LIST_COMMENT_PRIMARY_FILTER_NEEDS_REVIEW",
        ],
        "commentTypesFilters": [
            "LIST_COMMENT_TYPE_FILTER_ROOT",
        ],
        "secondaryFilters": [],
        "repliesFilter": [
            "LIST_COMMENT_PRIMARY_FILTER_PUBLISHED",
        ],
        "pageSize": 15,
    },
    "query": """
        query getCommentsForEpisode(
          $episodeUri: String!
          $primaryFilters: [String!]!
          $commentTypesFilters: [String!]!
          $secondaryFilters: [String!]!
          $repliesFilter: [String!]!
          $pageSize: Int!
        ) {
          getCommentsForEpisode(
            episodeUri: $episodeUri
            primaryFilters: $primaryFilters
            commentTypesFilters: $commentTypesFilters
            secondaryFilters: $secondaryFilters
            repliesFilter: $repliesFilter
            pageSize: $pageSize
          ) {
            items {
              uri
              text
              status
              author {
                username
              }
            }
          }
        }
    """,
}
r = requests.post(graphql_url, json=query_payload, headers=headers)
comments = r.json()["data"]["getCommentsForEpisode"]["items"]

# 最初の審査中コメントを承認して返信する
for comment in comments:
    if comment["status"] == "NEEDS_REVIEW":
        comment_uri = comment["uri"]

        # 承認
        approve_payload = {
            "operationName": "publishCommentByCommentUri",
            "variables": {"commentUri": comment_uri},
            "query": """
                mutation publishCommentByCommentUri($commentUri: String!) {
                  publishCommentByCommentUri(commentUri: $commentUri)
                }
            """,
        }
        requests.post(graphql_url, json=approve_payload, headers=headers)

        # 返信
        reply_payload = {
            "operationName": "createCommentReplyByCommentUri",
            "variables": {
                "parentCommentUri": comment_uri,
                "replyStr": "ご視聴ありがとうございます！",
            },
            "query": """
                mutation createCommentReplyByCommentUri(
                  $parentCommentUri: String!
                  $replyStr: String!
                ) {
                  createCommentReplyByCommentUri(
                    parentCommentUri: $parentCommentUri
                    replyStr: $replyStr
                  )
                }
            """,
        }
        requests.post(graphql_url, json=reply_payload, headers=headers)
        print(f"承認・返信完了: {comment_uri}")
        break
```
