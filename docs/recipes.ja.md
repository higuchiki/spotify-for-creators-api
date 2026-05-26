# レシピ・実装例

anchor.fm REST API と GraphQL API を使った Python 実装の実践的なフロー。

---

## エピソード更新（タイトル・概要欄・公開日時）

最も基本的なユースケース。既存エピソードを更新する標準フロー。

```python
import requests
from spotifyconnector import SpotifyConnector

# 事前に .env から sp_dc, sp_key, show_id を読み込む想定
# （auth.md の .env テンプレートを参照）

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
    f"https://api-v5.anchor.fm/v3/episodes/spotify:episode:{spotify_ep_id}/episodeId?isMumsCompatible=true",
    headers=headers_get,
)
anchor_id = r.json()["episodeId"]

# 3. エピソード現状取得（scheduled/published/draft の判定に必要）
r = requests.get(
    f"https://api-v5.anchor.fm/v3/episodes/{anchor_id}/overview?isMumsCompatible=true&returnWebIds=true",
    headers=headers_get,
)
overview = r.json()

# 4. エピソード更新
payload = {
    "userId": overview["userId"],
    "title": "新しいタイトル",
    "description": "<p>新しい概要欄</p>",
    "episodeType": overview.get("podcastEpisodeType", "full"),
    "isPublished": overview.get("isPublished", False),
    "podcastEpisodeIsExplicit": overview.get("podcastEpisodeIsExplicit", False),
    "publishOn": "2026-06-01T21:00:00.000Z",          # UTC で指定
    "wizardDraftedToPublishOn": "2026-06-01T21:00:00.000Z",
}
r = requests.post(
    f"https://api-v5.anchor.fm/v3/episodes/{anchor_id}/update?isMumsCompatible=true",
    json=payload,
    headers=headers_post,
)
print("成功" if r.status_code == 200 else f"失敗: {r.status_code}")

# 5. 予約状態の確認
r = requests.get(
    f"https://api-v5.anchor.fm/v3/episodes/{anchor_id}/overview?isMumsCompatible=true",
    headers=headers_get,
)
ep = r.json()
if not ep["isPublished"] and ep.get("publishOn"):
    print(f"予約済み: {ep['publishOn']}")
```

---

## JST → UTC 変換ユーティリティ

`publishOn` は UTC で指定する必要がある。JST で考えて UTC に変換する場合。

```python
from datetime import datetime, timezone, timedelta

JST = timezone(timedelta(hours=9))

def jst_to_utc_str(year, month, day, hour, minute=0, second=0):
    """JST の日時を UTC ISO8601 文字列に変換する"""
    publish_jst = datetime(year, month, day, hour, minute, second, tzinfo=JST)
    return publish_jst.astimezone(timezone.utc).strftime("%Y-%m-%dT%H:%M:%S.000Z")

# JST 06:00 → UTC 前日 21:00
utc_str = jst_to_utc_str(2026, 6, 1, 6, 0, 0)
# → "2026-05-31T21:00:00.000Z"
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

## 音声ファイルアップロード（完全フロー）

```python
import requests
import os

# 事前準備：bearer, anchor_id, station_id が必要

filename = "episode.mp3"
mime_type = "audio/mpeg"
file_path = "/path/to/episode.mp3"

# Step 1: 署名付きURL取得
r = requests.get(
    f"https://api-v5.anchor.fm/v3/episodes/{anchor_id}/upload/signedUrl"
    f"?filename={filename}&type={mime_type}&isMumsCompatible=true",
    headers={"Authorization": f"Bearer {bearer}", "Accept": "application/json"},
)
signed = r.json()
upload_id = signed["uploadId"]
signed_url = signed["url"]

# Step 2: GCS への直接 PUT
with open(file_path, "rb") as f:
    audio_data = f.read()

r = requests.put(
    signed_url,
    data=audio_data,
    headers={"Content-Type": mime_type},
)
r.raise_for_status()
etag = r.headers["ETag"].strip('"')  # ダブルクォートを除去

# Step 3: process_upload
payload = {
    "userId": user_id,
    "uploadType": "default",         # 音声ファイルは "default"（"audio" は誤り）
    "origin": "episode-media:upload",
    "caption": filename,
    "isExtractedFromVideo": False,
    "isMultipartUpload": True,
    "parts": [{"partNumber": 1, "etag": etag}],
    "uploadId": upload_id,
    "episodeId": anchor_id,
    "stationId": station_id,
}
r = requests.post(
    f"https://api-v5.anchor.fm/v3/upload/{upload_id}/process_upload?isMumsCompatible=true",
    json=payload,
    headers={
        "Authorization": f"Bearer {bearer}",
        "Content-Type": "application/json",
        "Origin": "https://creators.spotify.com",
        "Referer": "https://creators.spotify.com/",
    },
)
r.raise_for_status()

# Step 4: 処理完了待ち（ポーリング）
import time
for _ in range(30):
    r = requests.get(
        f"https://api-v5.anchor.fm/v3/upload/media/{upload_id}"
        f"?includeMediaValidation=true&isMumsCompatible=true",
        headers={"Authorization": f"Bearer {bearer}"},
    )
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
