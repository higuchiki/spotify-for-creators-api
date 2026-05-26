# GraphQL API

**エンドポイント**: `POST https://creators-graph.spotify.com/v2/graph-pq`

アナリティクス・コメント・トランスクリプト・章立て・ゲスト管理など、S4C の新しい機能のほとんどがここを使っている。anchor.fm REST API とは役割が異なる。

---

## 認証

anchor.fm と同じ Bearer トークンを使用する（同じ `spotifyconnector` で取得したもの）。

## リクエスト形式

```json
{
  "operationName": "getShowData",
  "variables": { "showId": "..." },
  "query": "query getShowData($showId: String!) { ... }"
}
```

!!! note "Persisted Query について"
    S4C の GraphQL は **Persisted Query（PQ）** を使用している。
    - ブラウザはリクエスト URL `/v2/graph-pq` に `query` フィールドを含まない（空文字列で送信）
    - Python から叩く場合は `query` フィールドに実際の GraphQL クエリ文字列を含める必要がある
      （サーバー側がフォールバックで受け付けるかは未確認）

---

## 既知のオペレーション一覧

調査日：2026-05-26（Claude in Chrome MCP による JS インターセプター計測）

### ショーレベル（show analytics ページ）

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `getShowData` | query | ショー基本情報取得 |
| `getShowTopEpisodes` | query | 人気エピソードランキング |
| `getShowTopClips` | query | 人気クリップランキング |
| `getShowAudienceAllPlatformsStats` | query | 全プラットフォーム視聴統計 |
| `getShowAudienceAllPlatformsGeoStats` | query | 地域別視聴統計 |
| `getShowAudienceDemographicsStats` | query | オーディエンス属性統計 |
| `getShowAudienceDiscoveryStats` | query | 発見経路統計 |
| `getShowImpressionsSourcesStats` | query | インプレッションソース統計 |
| `getShowImpressionsTrendStats` | query | インプレッション推移統計 |
| `getShowSmartlinksStats` | query | スマートリンク統計 |
| `getShowChaptersOptIn` | query | 章立て機能のオプトイン状態 |
| `getShowPreviewsOptIn` | query | プレビュー機能のオプトイン状態 |
| `getShowTranscriptsOptIn` | query | トランスクリプト機能のオプトイン状態 |

### エピソードレベル（episode 詳細・analytics ページ）

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `getEpisodeThumbnail` | query | エピソードサムネイル取得 |
| `getEpisodeChapters` | query | 章立てデータ取得 |
| `getEpisodeChaptersOptIn` | query | 章立て機能のオプトイン状態 |
| `getEpisodePreviews` | query | プレビューデータ取得 |
| `getEpisodeTranscript` | query | トランスクリプト取得 |
| `getEpisodeTranscriptAvailability` | query | トランスクリプトの利用可否 |
| `getEpisodeTranscriptsOptIn` | query | トランスクリプト機能のオプトイン状態 |
| `getGuestsByEpisodeUri` | query | ゲスト情報取得 |
| `getTopicsByEpisodeUri` | query | トピックタグ取得 |
| `getCrossContentLinksByEpisodeUri` | query | クロスコンテンツリンク取得 |
| `getPollForEpisode` | query | 投票機能データ取得 |
| `getCommentsForEpisode` | query | エピソードのコメント取得 |
| `getCommentsOnForEpisode` | query | コメント機能のオン/オフ状態 |
| `getBlockedUsersForShow` | query | ブロックユーザー一覧 |

### コメントページ

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `getCommentsAcrossEpisodes` | query | ショー全体のコメント一覧 |
| `getEpisodesWithSearch` | query | エピソード検索（コメントフィルター用） |

### コメント・投票設定（mutation）

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `updateIsCommentsEnabledForShowByShowUri` | mutation | コメント機能のON/OFF |
| `updateModerationPolicy` | mutation | コメントのモデレーションレベル変更 |
| `createPollForEpisode` | mutation | 投票の作成・更新（upsert動作） |
| `deletePollForEpisode` | mutation | 投票の削除 |

### コメント管理（mutation）— 2026-05-26 実測確定

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `publishCommentByCommentUri` | mutation | コメントを承認して公開する |
| `deleteCommentByCommentUri` | mutation | コメントを削除する |
| `createCommentReplyByCommentUri` | mutation | 制作者として返信を投稿する |
| `markCommentAsRead` | mutation | コメントを既読にマークする（承認・返信と同時に自動発火） |
| `pinComment` | mutation | コメントをピン固定する |
| `unPinComment` | mutation | コメントのピン固定を解除する |
| `blockUserForShow` | mutation | コメント投稿者のアカウントをショー単位でブロックする |
| `updateThreadCreatorLastSeen` | mutation | 返信スレッドを制作者が最後に確認した時刻を記録する |

### コメント管理（query）— 2026-05-26 実測確定

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `getCommentsForEpisode` | query | エピソードのコメント一覧取得 |
| `GetCommentReplies` | query | 指定コメントの返信一覧取得 |
| `getEpisodeData` | query | エピソードデータ取得（コメント操作後の再取得に使用） |
| `getPinnedCommentForEpisode` | query | エピソードのピン留めコメント取得 |

### アナリティクス全体概要

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `getShowOnSpotifyStats` | query | Spotifyでのショー統計（期間変更時に発火） |
| `getShowOnSpotifyStatsNRT` | query | Spotifyでのショー統計 NRT（Near Real Time） |
| `getShowOverviewStatsNRT` | query | ショー全体概要統計 NRT |
| `getFeatureEligibility` | query | 機能利用可否チェック |
| `getEpisodePlaysTotal` | query | エピソード総再生数取得 |

### その他

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `getPublishedEpisodeCount` | query | 公開済みエピソード数取得 |
| `getChaptersEligibility` | query | チャプター機能の利用資格確認 |
| `createEpisodeSmartlink` | mutation | エピソードスマートリンク作成 |
| `getLatestCommentsForShow` | query | ショー最新コメント一覧取得 |
| `WebGetIndexedEpisodeList` | query | インデックス済みエピソード一覧取得 |
| `getEpisodesForShow` | query | ショーのエピソード一覧取得（投票ページ） |

---

## Mutation スキーマ詳細

### コメント機能 ON/OFF

```graphql
mutation updateIsCommentsEnabledForShowByShowUri {
  updateIsCommentsEnabledForShowByShowUri(
    isEnabled: Boolean!
    showUri: String!   # "spotify:show:{SHOW_ID}"
  )
}
```

**variables 例**
```json
{
  "isEnabled": true,
  "showUri": "spotify:show:033mWbN806GYwFrVjdQtol"
}
```

---

### コメントモデレーションポリシー変更

```graphql
mutation updateModerationPolicy {
  updateModerationPolicy(
    showUri: String!
    moderationSignal: String!   # 常に "SENSITIVITY_OVERALL"
    moderationPolicy: String!   # 下表参照
  )
}
```

**`moderationPolicy` の値**

| 値 | UI表示 | 意味 |
|----|--------|------|
| `NO_FILTERING` | 低 | どのコメントも保留されません |
| `SOME_FILTERING` | 標準 | 不適切な内容の可能性があるコメントは保留 |
| `MAXIMUM_FILTERING` | 高 | すべてのコメントは確認のため保留されます |

---

### 投票作成・更新

```graphql
mutation createPollForEpisode {
  createPollForEpisode(
    episodeUri: String!    # "spotify:episode:{EPISODE_ID}"
    question: String!      # 質問テキスト（最大140文字）
    options: [PollOption!]! # 選択肢（最低2つ・最大7つ）
    type: String!          # "SINGLE_CHOICE" | "MULTIPLE_CHOICE"
    openingDate: Int!      # Unix タイムスタンプ（秒）
    closingDate: Int!      # Unix タイムスタンプ（秒）
  )
}
```

> ⚠️ 作成と更新で**同じ mutation** を使う（upsert動作）。

**variables 例**
```json
{
  "episodeUri": "spotify:episode:1xouj0WrH2klavXKzDWZbq",
  "question": "あなたの好きなポッドキャストのジャンルは？",
  "options": [
    { "description": "トーク系" },
    { "description": "情報系" }
  ],
  "type": "SINGLE_CHOICE",
  "openingDate": 1779721200,
  "closingDate": 1780412399
}
```

---

### コメント承認（公開）

```graphql
mutation publishCommentByCommentUri {
  publishCommentByCommentUri(
    commentUri: String!   # "spotify:comment:{COMMENT_ID}"
  )
}
```

> 承認と同時に `markCommentAsRead` も自動発火する。

---

### コメント削除

```graphql
mutation deleteCommentByCommentUri {
  deleteCommentByCommentUri(
    commentUri: String!
    filter: String!       # 下表参照
  )
}
```

**`filter` の値**

| 値 | 用途 |
|----|------|
| `DELETE_COMMENT_FILTER_PUBLISHED` | 公開済みコメントを削除する |
| `DELETE_COMMENT_FILTER_NEEDS_REVIEW` | （推定）審査中コメントを削除する（未確認） |

---

### 制作者返信

```graphql
mutation createCommentReplyByCommentUri {
  createCommentReplyByCommentUri(
    parentCommentUri: String!   # 返信先コメントの URI
    replyStr: String!           # 返信テキスト（UI上500文字）
  )
}
```

---

### コメントピン留め

```graphql
mutation pinComment {
  pinComment(
    commentUri: String!
  )
}
```

> - 公開済みコメントにのみ適用可能
> - ピン留めできるコメントは1エピソードにつき1件のみ（UIの制約）

---

### コメントピン留め解除

```graphql
mutation unPinComment {
  unPinComment(
    commentUri: String!
  )
}
```

---

### アカウントブロック

```graphql
mutation blockUserForShow {
  blockUserForShow(
    username: String!   # Spotifyの内部ユーザーID（"21xxxxx" 形式）
    showUri: String!    # "spotify:show:{SHOW_ID}"
  )
}
```

> `username` は `getCommentsForEpisode` のレスポンスの `author.username` フィールドから取得する。

---

## コメント Query スキーマ詳細

### コメント一覧取得

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

**`primaryFilters` の値**

| 値 | 意味 |
|----|------|
| `LIST_COMMENT_PRIMARY_FILTER_PUBLISHED` | 公開済みコメント |
| `LIST_COMMENT_PRIMARY_FILTER_NEEDS_REVIEW` | 審査中コメント |

**variables 例（全コメント取得）**
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

---

## コメント機能の仕様まとめ

| 操作 | 誰が行う | API自動化可能か |
|------|---------|-----------------|
| コメント投稿（リスナー） | リスナー（Spotifyアプリから） | 不可 |
| コメント投稿（制作者） | 制作者 | 可（S4Cの「コメントを追加」から） |
| コメント取得 | 制作者 | **可**（`getCommentsForEpisode` query） |
| コメント返信 | 制作者 | **可**（`createCommentReplyByCommentUri`） |
| コメント承認 | 制作者 | **可**（`publishCommentByCommentUri`） |
| コメント削除 | 制作者 | **可**（`deleteCommentByCommentUri`） |
| コメントON/OFF | 制作者 | 可（`updateIsCommentsEnabledForShowByShowUri`） |
| モデレーションレベル | 制作者 | 可（`updateModerationPolicy`） |
| ピン留め / 解除 | 制作者 | **可**（`pinComment` / `unPinComment`） |
| アカウントブロック | 制作者 | **可**（`blockUserForShow`） |
| 非承認（審査に戻す） | 制作者 | **UIに存在しない**（2026-05-26 確認） |

---

## 未取得・取得不可のオペレーション

| オペレーション名 | 取得不可の理由 |
|----------------|----------------|
| トランスクリプト編集 mutation | テストアカウントでは「文字起こし機能が利用できません」と表示され無効化。有料プランまたは特定条件でのみ有効化される機能と推定 |
| チャプター作成・更新・削除 mutation | テストショーはエピソード数が少なく資格なし。`getChaptersEligibility` query で条件を確認してから実装を試みること |
