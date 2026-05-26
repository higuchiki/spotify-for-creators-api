# REST API リファレンス

**ベースURL**: `https://api-v5.anchor.fm`

認証の詳細は [認証](auth.md) ページを参照。

---

## 共通ヘッダー

```
Authorization: Bearer {bearer_token}
Accept: application/json
```

POST 時はさらに追加：

```
Content-Type: application/json
Origin: https://creators.spotify.com
Referer: https://creators.spotify.com/
```

`Origin` と `Referer` がないと POST が拒否される場合がある。

## 共通クエリパラメータ

| パラメータ | 説明 |
|-----------|------|
| `isMumsCompatible=true` | 全エンドポイントで必須。省略しないこと |
| `returnWebIds=true` | `/overview` で推奨。省略すると一部フィールドが欠ける |

---

## 1-A. ID変換：Spotify Episode ID → Anchor Numeric ID

anchor.fm API のエンドポイントはすべて **anchor numeric ID**（整数）を使う。
S4C 上で見える Spotify Episode ID（文字列）からの変換が必要。

```
GET /v3/episodes/spotify:episode:{SPOTIFY_EP_ID}/episodeId?isMumsCompatible=true
```

**レスポンス**

```json
{ "episodeId": 123456789 }
```

```python
resp = requests.get(
    f"https://api-v5.anchor.fm/v3/episodes/spotify:episode:{spotify_ep_id}/episodeId?isMumsCompatible=true",
    headers={"Authorization": f"Bearer {bearer}", "Accept": "application/json"},
)
anchor_id = resp.json()["episodeId"]
```

---

## 1-B. エピソード詳細取得

```
GET /v3/episodes/{ANCHOR_ID}/overview?isMumsCompatible=true&returnWebIds=true
```

**レスポンスフィールド（主要）**

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `title` | string | タイトル |
| `description` | string | 概要欄（**HTML形式**） |
| `isPublished` | boolean | 公開済みかどうか |
| `publishOn` | string \| null | 公開日時（UTC ISO8601）。scheduled の場合のみ存在 |
| `userId` | integer | アカウントのユーザーID |
| `podcastEpisodeType` | string | `"full"` / `"trailer"` / `"bonus"` |
| `podcastEpisodeIsExplicit` | boolean | 露骨なコンテンツフラグ |

**エピソード状態の判定**

```
isPublished: false  かつ  publishOn が存在  →  scheduled（予約済み）
isPublished: true                          →  published（公開済み）
isPublished: false  かつ  publishOn なし   →  draft（下書き）
```

**レスポンス例**

```json
{
  "title": "ep.61 〇〇という話",
  "description": "<p>ep.61 〇〇という話</p>",
  "isPublished": false,
  "publishOn": "2026-05-31T21:00:00.000Z",
  "userId": YOUR_USER_ID,
  "podcastEpisodeType": "full",
  "podcastEpisodeIsExplicit": false
}
```

---

## 1-C. エピソード更新（タイトル・概要欄・公開日時）

```
POST /v3/episodes/{ANCHOR_ID}/update?isMumsCompatible=true
```

**リクエストボディ**

```json
{
  "userId": YOUR_USER_ID,
  "title": "エピソードタイトル",
  "description": "<p>概要欄の HTML</p>",
  "episodeType": "full",
  "isPublished": false,
  "podcastEpisodeIsExplicit": false,
  "publishOn": "2026-05-31T21:00:00.000Z",
  "wizardDraftedToPublishOn": "2026-05-31T21:00:00.000Z"
}
```

| フィールド | 必須 | 説明 |
|-----------|------|------|
| `userId` | ✅ | `/overview` から取得した値をそのまま使う |
| `title` | ✅ | 省略するとサーバー側で空になる恐れがある |
| `description` | ✅ | HTML 形式 |
| `episodeType` | ✅ | `"full"` / `"trailer"` / `"bonus"` |
| `isPublished` | ✅ | 現状を維持する場合は `/overview` の値をそのまま渡す |
| `podcastEpisodeIsExplicit` | ✅ | 同上 |
| `publishOn` | — | UTC ISO8601。scheduled の場合は必須 |
| `wizardDraftedToPublishOn` | — | `publishOn` と同じ値を入れる |

**レスポンス**: 成功時 HTTP 200

**レスポンスボディ（主要フィールド）**

```json
{
  "description": "<p>概要欄 HTML</p>",
  "title": "タイトル",
  "didChangePublishState": true,
  "isAnchorCreate": false,
  "shareUrl": "https://creators.spotify.com/pod/show/..."
}
```

`didChangePublishState: true` は `isPublished` が変わったことを示す。

---

## 1-D. リージョン取得

```
GET /v3/region?isMumsCompatible=true
```

ユーザーアカウントのリージョン情報を返す。

**レスポンス（推定）**: `{ "region": "JP" }`

---

## 1-E. スポンサードコンテンツ状態取得

```
GET /v3/episodes/{ANCHOR_ID}/sponsoredContentStatus?isMumsCompatible=true
```

エピソードのスポンサー（広告）設定状態を返す。

---

## 1-F. カバーアート配色取得

```
POST /MultiGetColor?isMumsCompatible=true
```

画像 URL からカラーパレットを返す。

**リクエストボディ**

```json
{
  "color_queries": [
    { "url": "https://anchor-generated-image-bank.s3.amazonaws.com/...jpg" }
  ]
}
```

---

## 1-G. エピソードフォームデータ取得

```
GET /v3/episodes/pcdFormData?episodeUri[]=spotify:episode:{SPOTIFY_EP_ID}&isMumsCompatible=true
```

`episodeUri[]` を複数指定して一括取得可能。

---

## 1-H. ショーメタデータ取得

```
GET /v3/stations/{STATION_ID}/metadata?isMumsCompatible=true
```

`STATION_ID` は `1-L` の `legacyIds` エンドポイントで取得できる。

**レスポンスフィールド（全フィールド、2026-05-26 実測）**

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `rssFeedUrl` | string | RSS フィード URL |
| `podcastName` | string | 番組タイトル |
| `podcastDescription` | string | 番組説明 |
| `podcastAuthorName` | string | 著者名 |
| `podcastAuthorEmail` | string | 著者メールアドレス |
| `podcastImage` | string | カバーアート URL（オリジナルサイズ） |
| `podcastImage400` | string | カバーアート URL（400px） |
| `podcastCategory` | string | カテゴリ（`"Arts\|Design"` 形式） |
| `podcastLanguage` | string | 言語コード（例: `"ja"`） |
| `language` | string | 言語コード（`podcastLanguage` と同値） |
| `isDistributedOnSpotify` | boolean | Spotify 配信中か |
| `isRssFeedDisabled` | boolean | RSS フィードが無効か |
| `hasPublishedEpisode` | boolean | 公開済みエピソードがあるか |
| `isExternalSyndicationSetup` | boolean | 外部シンジケーション設定済みか |
| `isSaiAndDaiFullyActivated` | boolean | SAI/DAI が有効か |
| `isAdCategoryExclusionsEnabled` | boolean | 広告カテゴリ除外が有効か |
| `distributionEligibilityForPaywalls` | string | ペイウォール配信資格（`"eligible"` 等） |
| `hasAnchorBrandingRemoved` | boolean | Anchor ブランディング除去済みか |
| `isStationOver30DaysOld` | boolean | 30 日以上経過したショーか |
| `isStationMarkedAsSpam` | boolean | スパム判定されているか |
| `isPublicCallinShownInRSS` | boolean | コールイン機能が RSS に含まれるか |
| `isPublicCallinHiddenFromWeb` | boolean | コールイン機能が Web 非表示か |

---

## 1-I. パートナーID取得

```
POST /v3/{USER_ID}/partnerIds?isMumsCompatible=true
```

収益化（広告・Fan Support 等）に関連するパートナー ID を取得。

---

## 1-J. ユーザー認証状態確認

```
GET /v3/users/verification?isMumsCompatible=true
```

---

## 1-K. オンボーディング状態確認

```
GET /v3/onboarding/redirectStatus/{USER_ID}/?isMumsCompatible=true
GET /v3/onboarding/steps/{USER_ID}/?userId={USER_ID}&isMumsCompatible=true
```

新規ユーザーのオンボーディング進捗確認。既存ユーザーには通常影響しない。

---

## 1-L. ショーID変換（Spotify Show ID → Anchor numeric IDs）

```
GET /v3/shows/{SPOTIFY_SHOW_ID}/legacyIds?isMumsCompatible=true
```

Spotify Show ID から anchor.fm 内部の numeric ID を一括変換して返す。

**レスポンス例**

```json
{
  "userId": YOUR_STATION_ID,
  "stationId": YOUR_STATION_ID,
  "webStationId": "112fef78c"
}
```

`stationId` がエピソード一覧・メタデータ等の他エンドポイントで使う numeric ID。

---

## 1-M. エピソード一覧インデックス更新

```
POST /v3/stations/{STATION_ID}/episode-list-index?isMumsCompatible=true
```

**レスポンス**: `{ "success": true }`

---

## 1-N. ショー形式データ取得

```
GET /v3/stations/{STATION_ID}/pcdForm?isMumsCompatible=true
```

**レスポンス**: `{ "formUrl": "" }`

---

## 1-O. 配信設定取得

```
GET /v3/stations/{STATION_ID}/distribution?isMumsCompatible=true
```

**レスポンスフィールド（主要）**

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `canEnableRss` | boolean | RSS 配信可否 |
| `externalUrls` | object | `{"spotify": "https://open.spotify.com/show/..."}` |
| `spotifyDistributionStatus` | string | Spotify 配信状態 |
| `spotifyShowUri` | string | Spotify Show URI |
| `platforms` | array | 配信プラットフォーム一覧 |
| `vanitySlug` | string | URL スラッグ |
| `isEnterprisePodcast` | boolean | エンタープライズ契約か |

---

## 1-P. ユーザー設定取得

```
GET /v3/users/{USER_ID}/settings/termsAndContactability?isMumsCompatible=true
```

**レスポンス例**

```json
{
  "termsOfServiceVersion": 1,
  "agreedToMinimumRequiredTermsOfService": true,
  "contactabilityStatus": "optedOut",
  "countryRule": "needsResponseOptInSelected"
}
```

---

## 1-Q. スポンサードコンテンツ状態更新

```
PUT /v3/episodes/{ANCHOR_ID}/sponsoredContentStatus?isMumsCompatible=true
```

**リクエストボディ**

```json
{
  "containsSponsoredContent": false,
  "publishOn": 1748732022
}
```

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `containsSponsoredContent` | boolean | スポンサードコンテンツを含むか |
| `publishOn` | number \| null | 公開日時の **Unix タイムスタンプ（秒）**。null は不可 |

> ⚠️ `publishOn` は ISO8601 文字列ではなく **Unix タイムスタンプ（整数秒）** で指定する。

---

## 1-R. マネタイズライフサイクル同期

```
POST /v5/monetization-lifecycle/{STATION_ID}/sync-and-fetch?isMumsCompatible=true
```

**レスポンスフィールド（主要）**

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `products.SPOTIFY_PARTNER_PROGRAM.lifecycleState` | string | パートナープログラムの状態 |
| `products.SPOTIFY_PARTNER_PROGRAM.pendingRequirements` | array | 完了待ちの要件リスト |

---

## 1-S. エピソード新規作成

```
POST /v3/stations/{STATION_ID}/episodes?isMumsCompatible=true
```

エピソードの **器（空レコード）** を作るだけ。タイトル・音声・説明は後続の `1-C`（update）で設定する。

**リクエストボディ**

```json
{ "hourOffset": -9 }
```

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `hourOffset` | integer | ユーザーのタイムゾーンオフセット（JST = -9） |

**レスポンス例**

```json
{
  "episodeId": 120529038,
  "stationId": YOUR_STATION_ID
}
```

> エピソード作成の完全フローは **3ステップ**：
> 1. `POST /v3/stations/{STATION_ID}/episodes`（空レコード作成 = 本エンドポイント）
> 2. アップロードフロー（`6-A`〜`6-D`）で音声/動画をアップロード
> 3. `POST /v3/episodes/{ANCHOR_ID}/update`（`1-C`）でタイトル・説明・公開設定

---

## 1-T. ショーメタデータ更新

```
POST /v3/stations/{STATION_ID}/metadata?isMumsCompatible=true
```

**リクエストボディ（主要フィールド）**

```json
{
  "podcastName": "番組タイトル",
  "podcastDescription": "番組説明",
  "podcastAuthorName": "著者名",
  "podcastCategory": "Arts|Design",
  "podcastLanguage": "ja",
  "podcastEpisodeIsExplicit": false
}
```

| フィールド | 型 | 必須 | 説明 |
|-----------|-----|------|------|
| `podcastName` | string | ✅ | 番組タイトル（最大100文字） |
| `podcastDescription` | string | — | 番組説明（最大600文字） |
| `podcastAuthorName` | string | — | 著者名（最大80文字） |
| `podcastCategory` | string | — | カテゴリ（`"Arts|Design"` 形式） |
| `podcastLanguage` | string | — | 言語コード（例: `"ja"`） |
| `podcastEpisodeIsExplicit` | boolean | — | 露骨コンテンツフラグ |

**カテゴリオプション一覧取得**

```
GET /v3/settings/podcast-category/options?isMumsCompatible=true
```

---

## 1-U. vanity slug 更新

```
POST /v3/settings?isMumsCompatible=true
```

**リクエストボディ**

```json
{
  "webStationId": "your-vanity-slug"
}
```

> ⚠️ `webStationId` はグローバルでユニークである必要がある。すでに使用されているスラッグを指定すると 409 Conflict が返る可能性がある。

---

## 1-V. 画像アップロードフロー

S4C の画像アップロードは音声/動画とは**別の専用パイプライン**を使う。ショーカバーアート・エピソードサムネイルで共通フロー。

### 1-V-1. 署名付きURL取得

**ショーカバーアート**

```
GET /v3/stations/{STATION_ID}/images/upload/signed_url?isMumsCompatible=true
```

**エピソードサムネイル**

```
GET /v3/episodes/{ANCHOR_ID}/images/upload/signed_url?isMumsCompatible=true
```

**レスポンス例**

```json
{
  "url": "https://storage.googleapis.com/anchor-image-storage-production/{UUID}/{FILENAME}?X-Goog-Algorithm=...",
  "uploadId": "a46e21a4-1d2d-1686-1355-905e191a3bef"
}
```

### 1-V-2. Google Cloud Storage への直接 PUT

```
PUT {signed_url}
Content-Type: image/png  # または image/jpeg
```

署名付きURL に対して画像ファイルを直接 PUT する。認証ヘッダー不要（署名パラメータに埋め込み済み）。

### 1-V-3. アップロード完了通知（process_upload）

**ショーカバーアート**

```
POST /v3/stations/{STATION_ID}/images/upload/{UPLOAD_ID}/process_upload?isMumsCompatible=true
```

**エピソードサムネイル**

```
POST /v3/episodes/{ANCHOR_ID}/images/upload/{UPLOAD_ID}/process_upload?isMumsCompatible=true
```

**リクエストボディ（エピソードサムネイル）**

```json
{
  "episodeId": 120520683,
  "uploadType": "episode_image"
}
```

### 1-V-4. アップロード状態ポーリング

`status` が `"completed"` になるまでポーリング（通常数秒）。

```
GET /v3/stations/{STATION_ID}/images/upload/{UPLOAD_ID}/?isMumsCompatible=true
GET /v3/episodes/{ANCHOR_ID}/images/upload/{UPLOAD_ID}/?isMumsCompatible=true
```

---

## 1-W. 収益化関連エンドポイント

```
GET /v4/paywalls/{STATION_ID}/lifecycle-state?isMumsCompatible=true
GET /v6/wallet/{STATION_ID}/availability?isMumsCompatible=true
GET /v6/wallet/{STATION_ID}/?isMumsCompatible=true
GET /v3/paywalls/{STATION_ID}/?isMumsCompatible=true
```

---

## 1-X. 番組設定ページ固有エンドポイント

```
GET /v3/stations/{STATION_ID}/episodePage?isMumsCompatible=true
GET /v3/station/{STATION_ID}/episodes/featured?isMumsCompatible=true
GET /v3/stations/{STATION_ID}/hostRecommendations?isMumsCompatible=true
```

---

## 1-Y. 配信ページ固有エンドポイント

```
GET /v3/onboarding/redirectStatus/{STATION_ID}/?isMumsCompatible=true
GET /v3/onboarding/steps/{STATION_ID}/?userId={USER_ID}&isMumsCompatible=true
GET /v3/stations/status?isMumsCompatible=true
```

---

## 音声/動画アップロード（6-A 〜 6-E）

**対応フォーマット**：mp3, m4a, aifc, aiff, ogg, wav, flac, mp4, mov

### 6-A. アップロード署名付きURL取得

```
GET /v3/episodes/{ANCHOR_ID}/upload/signedUrl?filename={FILENAME}&type={MIME_TYPE}&isMumsCompatible=true
```

| パラメータ | 例 | 説明 |
|-----------|-----|------|
| `filename` | `episode.mp4` | アップロードするファイル名 |
| `type` | `video/mp4` | MIMEタイプ（音声なら `audio/mp3` 等） |

**レスポンス例**

```json
{
  "uploadId": "9e1fdfce-4485-6bde-e582-3b7c84978d55",
  "url": "https://storage.googleapis.com/anchor_exclusive_media_upload_production/...",
  "headers": {}
}
```

### 6-B. Google Cloud Storage への直接 PUT

```
PUT {signed_url}
Content-Type: {MIME_TYPE}
```

レスポンスヘッダーの `ETag` を記録する（`process_upload` で使う）。

### 6-C. アップロード完了通知（process_upload）

```
POST /v3/upload/{UPLOAD_ID}/process_upload?isMumsCompatible=true
```

**リクエストボディ（ビデオの場合）**

```json
{
  "userId": YOUR_STATION_ID,
  "uploadType": "video",
  "origin": "episode-media:upload",
  "caption": "episode.mp4",
  "isExtractedFromVideo": true,
  "isMultipartUpload": true,
  "parts": [
    { "partNumber": 1, "etag": "4d49fffd0ed1c6a291577ac47d26b997" }
  ],
  "uploadId": "9e1fdfce-4485-6bde-e582-3b7c84978d55",
  "episodeId": 120525547,
  "stationId": YOUR_STATION_ID
}
```

!!! warning "uploadType の注意"
    音声ファイルは `uploadType: "default"` を使う。`"audio"` は誤り（2026-05-26 実測確認）。

### 6-D. メディア検証確認

`status` が `"completed"` になるまでポーリングする。

```
GET /v3/upload/media/{UPLOAD_ID}?includeMediaValidation=true&isMumsCompatible=true
```

**レスポンスフィールド（主要）**

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `status` | string | `"processing"` / `"completed"` / `"failed"` |
| `mediaType` | string | `"video"` / `"audio"` |
| `durationInMilliseconds` | integer | 動画/音声の長さ（ms） |

---

## 既知の注意事項

### description フィールドのサーバー側自動変換

| 変換前 | 変換後 |
|--------|--------|
| `<br>` | `<br />` |
| `<a href="...">` | `<a href="..." rel="ugc noopener noreferrer" target="_blank">` |

### draft エピソードの overview は 403

`isPublished: false`（draft 状態）のエピソードに対して `GET /v3/episodes/{ANCHOR_ID}/overview` を呼ぶと **HTTP 403 Forbidden** が返る。

### publishOn のタイムゾーン

`publishOn` は **UTC** で指定する。JST 06:00 に公開したい場合は前日 21:00 UTC。

```python
from datetime import datetime, timezone, timedelta

JST = timezone(timedelta(hours=9))
publish_jst = datetime(2026, 6, 1, 6, 0, 0, tzinfo=JST)
publish_utc_str = publish_jst.astimezone(timezone.utc).strftime("%Y-%m-%dT%H:%M:%S.000Z")
# → "2026-05-31T21:00:00.000Z"
```

### publishOn を isPublished: false → true に一発設定する挙動

`isPublished: false` + `publishOn: "..."` で update を呼ぶと、サーバー側が **自動的に `isPublished: true`** に変換する（予約公開として即公開状態になる）。

### publishOn は公開済みエピソードでは変更不可

`isPublished: true` のエピソードに対してこの API で `publishOn` を変更しても、サーバー側でロックされており変更されない。
