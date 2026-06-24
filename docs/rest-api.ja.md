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

## ID システム

anchor.fm API は **anchor numeric ID**（整数）を使う。S4C UI や GraphQL で見える Spotify 文字列 ID とは異なる。ほとんどのエンドポイントを呼ぶ前に変換が必要。

| ID の種類 | 例 | 使用箇所 |
|-----------|----|---------|
| Spotify Episode ID | `"1xouj0WrH2klavXKzDWZbq"` | S4C UI、GraphQL |
| Anchor Episode ID（数値） | `123456789` | anchor.fm REST エンドポイント |
| Spotify Show ID | `"YOUR_SHOW_ID"` | S4C UI、GraphQL |
| Station ID（数値） | `YOUR_STATION_ID` | anchor.fm REST エンドポイント |

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

=== "Python"
    ```python
    resp = requests.get(
        f"https://api-v5.anchor.fm/v3/episodes/spotify:episode:{spotify_ep_id}/episodeId?isMumsCompatible=true",
        headers={"Authorization": f"Bearer {bearer}", "Accept": "application/json"},
    )
    anchor_id = resp.json()["episodeId"]
    ```

=== "cURL"
    ```bash
    curl -s \
      -H "Authorization: Bearer $BEARER" \
      "https://api-v5.anchor.fm/v3/episodes/spotify:episode:${SPOTIFY_EP_ID}/episodeId?isMumsCompatible=true"
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

!!! warning "下書きエピソードは 403 が返る"
    `isPublished: false` かつ `publishOn` なし（draft 状態）のエピソードに対してこのエンドポイントを呼ぶと **HTTP 403 Forbidden** が返る（ライブブラウザセッションでも同様）。S4C UI は下書きのデータを GraphQL で取得している。下書きエピソードのメタデータを更新する必要がある場合は、更新エンドポイントを呼ぶ前に元の値をメモリに保持しておくこと。

=== "Python"
    ```python
    r = requests.get(
        f"https://api-v5.anchor.fm/v3/episodes/{anchor_id}/overview"
        "?isMumsCompatible=true&returnWebIds=true",
        headers=headers_get,
    )
    overview = r.json()
    ```

=== "cURL"
    ```bash
    curl -s \
      -H "Authorization: Bearer $BEARER" \
      "https://api-v5.anchor.fm/v3/episodes/${ANCHOR_ID}/overview?isMumsCompatible=true&returnWebIds=true"
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
  "publishOn": 1748732400,
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
| `publishOn` | — | **Unix タイムスタンプ（整数秒）**。scheduled の場合は必須。ISO 8601 文字列を送ると値がサイレントに破損する |
| `wizardDraftedToPublishOn` | — | **UTC ISO 8601 文字列**。`publishOn` と同一の壁時計時刻を指定する。整数を送ると HTTP 400 |

!!! warning "`publishOn` は Unix 整数、`wizardDraftedToPublishOn` は ISO 文字列"
    `publishOn` に ISO 8601 文字列（例: `"2026-06-01T21:00:00.000Z"`）を送ると値がサイレントに破損する。
    API は先頭の数字だけを整数として解釈するため、`"2026"` が 2026 秒として格納され
    `1970-01-01T00:33:46.000Z` になる。
    
    `wizardDraftedToPublishOn` は逆で ISO 8601 文字列が必須（整数を送ると HTTP 400）。

    ```python
    from datetime import datetime, timezone, timedelta
    JST = timezone(timedelta(hours=9))
    publish_jst = datetime(2026, 6, 1, 6, 0, 0, tzinfo=JST)
    pub_utc = publish_jst.astimezone(timezone.utc)
    publish_on_unix = int(pub_utc.timestamp())                    # → 1748732400
    publish_on_iso  = pub_utc.strftime("%Y-%m-%dT%H:%M:%S.000Z") # → "2026-05-31T21:00:00.000Z"
    ```

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

=== "Python"
    ```python
    pub_utc = publish_jst.astimezone(timezone.utc)
    payload = {
        "userId": overview["userId"],
        "title": "新しいエピソードタイトル",
        "description": "<p>新しい説明</p>",
        "episodeType": overview.get("podcastEpisodeType", "full"),
        "isPublished": overview.get("isPublished", False),
        "podcastEpisodeIsExplicit": overview.get("podcastEpisodeIsExplicit", False),
        "publishOn": int(pub_utc.timestamp()),                    # Unix 整数
        "wizardDraftedToPublishOn": pub_utc.strftime("%Y-%m-%dT%H:%M:%S.000Z"),  # ISO 文字列
    }
    r = requests.post(
        f"https://api-v5.anchor.fm/v3/episodes/{anchor_id}/update?isMumsCompatible=true",
        json=payload,
        headers=headers_post,
    )
    ```

=== "cURL"
    ```bash
    curl -s -X POST \
      -H "Authorization: Bearer $BEARER" \
      -H "Content-Type: application/json" \
      -H "Origin: https://creators.spotify.com" \
      -H "Referer: https://creators.spotify.com/" \
      -d '{"userId":YOUR_USER_ID,"title":"新タイトル","description":"<p>説明</p>","episodeType":"full","isPublished":false,"podcastEpisodeIsExplicit":false,"publishOn":1748732400,"wizardDraftedToPublishOn":"2026-06-01T21:00:00.000Z"}' \
      "https://api-v5.anchor.fm/v3/episodes/${ANCHOR_ID}/update?isMumsCompatible=true"
    ```

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

| フィールド | 説明 |
|-----------|------|
| `stationId` | エピソード一覧・メタデータ等の他エンドポイントで使う numeric ID |
| `userId` | アカウントの numeric ユーザー ID |
| `webStationId` | URL スラッグ（バニティ ID） |

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

=== "cURL"
    ```bash
    curl -s \
      -H "Authorization: Bearer $BEARER" \
      "https://api-v5.anchor.fm/v3/shows/${SPOTIFY_SHOW_ID}/legacyIds?isMumsCompatible=true"
    ```

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

## 1-Z. エピソード削除

```
POST /v3/episodes/{ANCHOR_ID}/delete?isMumsCompatible=true
```

エピソードを完全に削除する。下書き・予約公開（未公開）・公開済みのいずれの
状態でも動作する。

### リクエストボディ

```json
{}
```

空のJSONボディでよい。エピソードはパスの `{ANCHOR_ID}`（Spotifyの
エピソードURIではなく **Anchorの数値ID**）で特定される。Spotify IDしか
持っていない場合は 1-A で先に数値IDへ変換する。

### レスポンス

```json
{}
```

成功時 `200` を返す。

!!! warning "取り消し不可"
    ゴミ箱・復元の仕組みはなく、成功した時点で即座にエピソードが消える。
    呼ぶ前に `{ANCHOR_ID}` が意図したエピソードを指しているか
    （`/overview`（1-B）等で）確認すること。

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

## 音声/動画アップロード（6-A 〜 6-D）

**対応フォーマット**：mp3, m4a, aifc, aiff, ogg, wav, flac, mp4, mov

**ファイルサイズ上限**

| 種別 | 上限 |
|------|------|
| 動画 | 4 GB |
| 音声 | 500 MB |

### 6-A. アップロード署名付きURL取得

```
GET /v3/episodes/{ANCHOR_ID}/upload/signedUrl?filename={FILENAME}&type={MIME_TYPE}&isMumsCompatible=true
```

| パラメータ | 例 | 説明 |
|-----------|-----|------|
| `filename` | `episode.mp3` | アップロードするファイル名 |
| `type` | `audio/mp3` | MIMEタイプ（動画なら `video/mp4`） |

**レスポンス — ストレージバックエンドによって2種類：**

音声ファイルは **AWS S3** にルーティングされ、以下を返す：

```json
{
  "requestUuid": "b8c96d9c-1d37-45a5-7ba3-2dcfb17e1fdc",
  "fileKey": "b8c96d9c-1d37-45a5-7ba3-2dcfb17e1fdc/episode.mp3",
  "signedUrl": "https://audio-uploaded-default-production.s3.us-west-1.amazonaws.com/..."
}
```

動画ファイルは **Google Cloud Storage** にルーティングされ、以下を返す：

```json
{
  "uploadId": "9e1fdfce-4485-6bde-e582-3b7c84978d55",
  "url": "https://storage.googleapis.com/anchor_exclusive_media_upload_production/...",
  "headers": {}
}
```

音声は `signedUrl` / `requestUuid` を使う。動画は `url` / `uploadId` を使う。

### 6-B. ストレージへの直接 PUT

```
PUT {signedUrl}    # 音声（S3）
PUT {url}          # 動画（GCS）
Content-Type: {MIME_TYPE}
```

レスポンスヘッダーの `ETag` を記録する（`process_upload` で使う）。

### 6-C. アップロード完了通知（process_upload）

```
POST /v3/upload/{requestUuid}/process_upload?isMumsCompatible=true   # 音声
POST /v3/upload/{uploadId}/process_upload?isMumsCompatible=true      # 動画
```

**リクエストボディ（音声の場合）**

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

**リクエストボディ（動画の場合）**

```json
{
  "userId": YOUR_USER_ID,
  "uploadType": "video",
  "origin": "episode-media:upload",
  "caption": "episode.mp4",
  "isExtractedFromVideo": true,
  "isMultipartUpload": true,
  "parts": [{ "partNumber": 1, "etag": "4d49fffd0ed1c6a291577ac47d26b997" }],
  "uploadId": "9e1fdfce-4485-6bde-e582-3b7c84978d55",
  "episodeId": 120525547,
  "stationId": YOUR_STATION_ID
}
```

| パラメータ | 型 | 説明 |
|-----------|-----|------|
| `uploadType` | string | 音声は `"default"`（`"audio"` は誤り）。動画は `"video"` |
| `isExtractedFromVideo` | boolean | mp4/mov は `true`、音声のみのファイルは `false` |
| `parts` | array | ストレージ PUT レスポンスの ETag 一覧 |
| `uploadId` | string | 音声は 6-A の `requestUuid`。動画は 6-A の `uploadId` |

### 6-D. メディア検証確認（ポーリング）

```
GET /v3/upload/media/{requestUuid}?includeMediaValidation=true&isMumsCompatible=true  # 音声
GET /v3/upload/media/{uploadId}?includeMediaValidation=true&isMumsCompatible=true     # 動画
```

`status` が `"completed"` になるまでポーリングする。

**レスポンスフィールド（主要）**

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `status` | string | `"processing"` / `"completed"` / `"failed"` |
| `mediaType` | string | `"video"` / `"audio"` |
| `durationInMilliseconds` | integer | 動画/音声の長さ（ms） |

!!! note "音声ポーリングは一時的に 404 を返すことがある"
    音声（S3）アップロードでは、`process_upload` 完了後 30〜60 秒程度ポーリングエンドポイントが
    HTTP 404 を返すことがある。バックオフを入れながら最大約 300 秒までリトライすること。

---

## 既知の注意事項

### description フィールドのサーバー側自動変換

| 変換前 | 変換後 |
|--------|--------|
| `<br>` | `<br />` |
| `<a href="...">` | `<a href="..." rel="ugc noopener noreferrer" target="_blank">` |

### draft エピソードの overview は 403

`isPublished: false`（draft 状態）のエピソードに対して `GET /v3/episodes/{ANCHOR_ID}/overview` を呼ぶと **HTTP 403 Forbidden** が返る。

### publishOn は Unix 整数で指定する

`publishOn` には **Unix タイムスタンプ（整数秒）** を指定する。ISO 8601 文字列を送ると値がサイレントに破損する（API が先頭の数字だけを整数として解釈し、1970年になる）。

JST 06:00 に公開したい場合：

```python
from datetime import datetime, timezone, timedelta

JST = timezone(timedelta(hours=9))
publish_jst = datetime(2026, 6, 1, 6, 0, 0, tzinfo=JST)
pub_utc = publish_jst.astimezone(timezone.utc)
publish_on_unix = int(pub_utc.timestamp())                    # → 1748732400
publish_on_iso  = pub_utc.strftime("%Y-%m-%dT%H:%M:%S.000Z") # → "2026-05-31T21:00:00.000Z"
```

`wizardDraftedToPublishOn` は逆に ISO 8601 文字列が必須（整数を送ると HTTP 400）。詳細は 1-C の警告を参照。

### publishOn を isPublished: false → true に一発設定する挙動

`isPublished: false` + `publishOn: "..."` で update を呼ぶと、サーバー側が **自動的に `isPublished: true`** に変換する（予約公開として即公開状態になる）。

### publishOn は公開済みエピソードでは変更不可

`isPublished: true` のエピソードに対してこの API で `publishOn` を変更しても、サーバー側でロックされており変更されない。
