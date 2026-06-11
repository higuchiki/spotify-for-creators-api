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
    - Python から叩く場合は `query` フィールドに実際の GraphQL クエリ文字列を含める
    - **確認済み（2026-06-11）**：サーバーは完全なクエリ文字列を受け付け、イントロスペクションも有効
      （下記「アナリティクス Query（実測確定）」参照）

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

## アナリティクス Query（実測確定）

> 調査日：2026-06-11。スキーマイントロスペクション + 本番ショーへの実クエリで確認。

### アーキテクチャ：アナリティクスはネストフィールド（トップレベル操作ではない）

ブラウザ通信で観測されるオペレーション名（`getShowAudienceAllPlatformsStats` 等）は
`/v2/graph-pq` の**トップレベル Query フィールドには存在しない**（ad-hoc クエリとして
送ると `ValidationError`）。実際のアクセス経路は以下のネスト構造：

- ショーレベル：`showByShowUri` → `Show.analytics(getShowAnalyticsRequest: …)`
- エピソードレベル：`episodeByUri` → `Episode.analytics(getEpisodePlayCountRequest: …)`

!!! tip "イントロスペクション対応 — Persisted Query は必須ではない"
    `/v2/graph-pq` は通常の GraphQL クエリ文字列を受け付け、スキーマイントロスペクション
    も有効。下記の metric enum 一覧も標準のイントロスペクションクエリで取得できる。

### ショーレベル統計

```graphql
query {
  showByShowUri(getShowByShowUriRequest: { showUri: "spotify:show:YOUR_SHOW_ID" }) {
    analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_STREAMS_AND_DOWNLOADS,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: WINDOW_ALL_TIME
    }) {
      type
      startDate
      endDate
      followerCount
      analyticsValue {
        analyticsValue {
          __typename
          ... on SingleValueLong { value }
          ... on AudienceValue { totalAudienceSize foregroundAudienceSize foregroundAudiencePercent }
          ... on TimeSeriesValue {
            points {
              date
              value {
                ... on CountValueLong { value }
                ... on RatioValueFloat { value }
              }
            }
          }
        }
      }
    }
  }
}
```

`showUri` は `ShowUri` スカラー型。文字列 `"spotify:show:YOUR_SHOW_ID"` を渡す。

#### `ShowAnalyticsMetricType`（主要値）

| ENUM 値 | 実際に測っているもの |
|---------|---------------------|
| `SHOW_PLAYS` | **Spotify 単体**の再生数（podcasters API の Spotify 単体 starts と一致） |
| `SHOW_STREAMS` | **Spotify 単体**の streams（60秒以上再生） |
| `SHOW_STREAMS_AND_DOWNLOADS` | **全プラットフォーム** streams + downloads — ダッシュボード「すべてのプラットフォーム」と一致 |
| `SHOW_PLAYS_AND_DOWNLOADS` | 全プラットフォーム plays + downloads（streams+downloads より大きい） |
| `SHOW_ALL_PLATFORMS_LISTENERS` | 全プラットフォームのユニークリスナー |
| `SHOW_DOWNLOAD_LISTENERS` | ダウンロードリスナー |
| `SHOW_OFF_PLATFORM_DOWNLOADS` | プラットフォーム外ダウンロード（RSSアプリ等） |
| `SHOW_LISTENERS` | ユニークリスナー（Spotify スコープ） |
| `SHOW_FOLLOWERS` | フォロワー数 |
| `AUDIENCE_SIZE` | オーディエンスサイズ（`AudienceValue` を返す） |

!!! warning "metric 名は誤解を招く"
    名前に反して、`SHOW_PLAYS` は実測で **Spotify 単体**の starts 合計と一致した。
    全プラットフォーム値ではない。真の全プラットフォーム値は `*_AND_DOWNLOADS` 系。
    テストショーでは全プラットフォーム合計（`SHOW_STREAMS_AND_DOWNLOADS`）が
    Spotify 単体の約 **1.6倍** だった。

!!! tip "1コールでエピソードループを置き換えられる"
    全エピソードの `EPISODE_STREAMS_AND_DOWNLOADS` 合算と `SHOW_STREAMS_AND_DOWNLOADS`
    が一致することを実測確認（誤差 <0.01%）。ショー全体の全プラットフォーム累計が
    欲しい場合、ショーレベル1コールで足りる。

#### `AggregationType`

| 値 | 説明 |
|----|------|
| `AGGREGATION_TYPE_TOTAL` | 期間合計 |
| `AGGREGATION_TYPE_DAILY` | 日次時系列 |
| `AGGREGATION_TYPE_WEEKLY` | 週次時系列 |
| `AGGREGATION_TYPE_MONTHLY` | 月次時系列 |

#### `AnalyticsWindow`

| 値 | 説明 |
|----|------|
| `WINDOW_ALL_TIME` | 全期間 |
| `WINDOW_LAST_SEVEN_DAYS` | 直近7日 |
| `WINDOW_LAST_THIRTY_DAYS` | 直近30日 |
| `WINDOW_LAST_NINETY_DAYS` | 直近90日 |
| `WINDOW_YEAR_TO_DATE` | 年初来 |
| `WINDOW_CUSTOM` | カスタム期間（`customDateRange: { startDate, endDate }` を渡す） |

### エピソードレベル統計

```graphql
query {
  episodeByUri(getEpisodeRequest: { episodeUri: "spotify:episode:YOUR_EPISODE_ID" }) {
    title
    analytics(getEpisodePlayCountRequest: {
      episodeAnalyticsMetricType: EPISODE_STREAMS_AND_DOWNLOADS,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: WINDOW_ALL_TIME
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on SingleValueLong { value }
        }
      }
    }
    analyticsStarts {
      startsCount  # Spotify 単体の starts（引数なし）
    }
  }
}
```

`episodeUri` は `EpisodeUri` スカラー型。文字列 `"spotify:episode:YOUR_EPISODE_ID"` を渡す。

#### `EpisodeAnalyticsMetricType`（主要値）

| ENUM 値 | 実際に測っているもの |
|---------|---------------------|
| `EPISODE_PLAYS` | 再生数（実測では Spotify スコープ） |
| `EPISODE_STREAMS` | streams・60秒以上（Spotify 単体） |
| `EPISODE_LISTENERS` | ユニークリスナー |
| `EPISODE_ALL_PLATFORMS_LISTENERS` | 全プラットフォームのユニークリスナー |
| `EPISODE_STREAMS_AND_DOWNLOADS` | **全プラットフォーム** streams + downloads |
| `EPISODE_PLAYS_AND_DOWNLOADS` | 全プラットフォーム plays + downloads |
| `EPISODE_OFF_PLATFORM_DOWNLOADS` | プラットフォーム外ダウンロード |
| `EPISODE_DOWNLOAD_LISTENERS` | ダウンロードリスナー |
| `EPISODE_PERFORMANCE` | 視聴維持率（Audience Retention） |

テストショーの直近エピソードでは `EPISODE_STREAMS_AND_DOWNLOADS` が Spotify 単体
starts の約 **2.5倍** だった。公開直後の再生は RSS 配信先（Apple Podcasts 等）が
過半を占めることがある。

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
  "showUri": "spotify:show:YOUR_SHOW_ID"
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

## 新メトリクス（2026年5月ダッシュボード刷新で追加）

> 2026-06-12 動作確認済み。2026年5月下旬のダッシュボード刷新（Investor Day
> 発表に連動とみられる）で追加されたメトリクス。

### `SHOW_SEGMENTED_AUDIENCE` — オーディエンスセグメント

S4C ダッシュボードの「オーディエンスセグメント」画面に対応する。
新規リスナーとリピーターの内訳を取得できる。

#### 期間合計クエリ

```graphql
query {
  showByShowUri(getShowByShowUriRequest: { showUri: "spotify:show:YOUR_SHOW_ID" }) {
    analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_SEGMENTED_AUDIENCE,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: WINDOW_LAST_NINETY_DAYS
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on SegmentedAudienceValue {
            totalAudience
            newAudience
            returningAudience
          }
        }
      }
    }
  }
}
```

**レスポンス型：** `SegmentedAudienceValue`

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `totalAudience` | `Long` | 期間内のユニークリスナー数（Spotify スコープ） |
| `newAudience` | `Long` | 期間内に番組を初めて聴いたリスナー数 |
| `returningAudience` | `Long` | 期間前から番組を知っていたリスナー数 |

#### 日次時系列クエリ

```graphql
query {
  showByShowUri(getShowByShowUriRequest: { showUri: "spotify:show:YOUR_SHOW_ID" }) {
    analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_SEGMENTED_AUDIENCE,
      aggregationType: AGGREGATION_TYPE_DAILY,
      window: WINDOW_LAST_NINETY_DAYS
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on TimeSeriesValue {
            points {
              date
              value {
                ... on SegmentedAudienceValue {
                  totalAudience
                  newAudience
                  returningAudience
                }
              }
            }
          }
        }
      }
    }
  }
}
```

**注意：** `aggregationType` が `AGGREGATION_TYPE_DAILY` のとき、返り値は
`TimeSeriesValue` になり、各 `points[].value` が `SegmentedAudienceValue` に解決される。
上記のようにインラインフラグメントが必要。

#### 指標のセマンティクス — 重要な非対称性（2026-06-12 検証済み）

`newAudience` と `returningAudience` はカウント方法が根本的に異なる。

| 指標 | カウント方式 | 日次合計 vs. 期間合計 |
|------|------------|----------------------|
| `newAudience` | 期間ユニーク — 何日聴いても1回だけカウント | 日次合計 **= 期間合計** |
| `returningAudience` | 日次カウント — リピーターが聴いた**日ごとに**カウント | 日次合計 **>> 期間合計**（90日窓では約9倍になる例も） |
| `totalAudience` | 期間ユニークリスナー | 日次合計 > 期間合計（同一リスナーが複数日に出現するため） |

**検証方法（2026-06-12 確認）：** 90日分の全日付で
`newAudience + returningAudience == totalAudience` が完全一致することを確認。
データの内部整合性が保証されている。

!!! warning "日次 `returningAudience` を合算して期間合計にしてはいけない"
    日次の `returningAudience` は、リピーターが聴いた日ごとにカウントされる。
    90日分を合算すると「ユニークリスナー数」ではなく「リスナー延べ日数」になる。
    期間内のユニーク数が必要な場合は `AGGREGATION_TYPE_TOTAL` を使うこと。

---

### `SHOW_DISCOVERY_FUNNEL` — Impressions → Plays ファネル（2026年5月追加）

> 2026-06-12 動作確認済み。

Impressions → Plays → 平均完了率の3ステップファネルを取得できる。
前期比較値も含まれる。

```graphql
query {
  showByShowUri(getShowByShowUriRequest: { showUri: "spotify:show:YOUR_SHOW_ID" }) {
    analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_DISCOVERY_FUNNEL,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: WINDOW_LAST_NINETY_DAYS
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on DiscoveryFunnelValue {
            steps {
              stepName
              displayName
              value
              conversionRateToNext
              periodOverPeriodPercentageDiff
            }
          }
        }
      }
    }
  }
}
```

**`DiscoveryFunnelStep` フィールド：**

| フィールド | 型 | 説明 |
|-----------|-----|------|
| `stepName` | `String` | 内部名（`"impressions"`、`"plays"`、`"average_completion_rate"`） |
| `displayName` | `String` | 表示名（`"Impressions"`、`"Plays"`、`"Average Completion Rate"`） |
| `value` | `Long` | カウント値（`average_completion_rate` ステップには存在しない） |
| `conversionRateToNext` | `Float` | 次ステップへの転換率（例：`0.15` = 15%） |
| `periodOverPeriodPercentageDiff` | `Float` | 前期比増減率（例：`0.42` = +42%） |

---

### `SHOW_WINDOWED_AVERAGE_COMPLETION_RATE` — 期間平均完了率（2026年5月追加）

> 2026-06-12 動作確認済み。

期間内の全エピソードを横断した平均完了率を `PercentageValueFloat` で返す
（例：`0.60` = 60%）。

```graphql
query {
  showByShowUri(getShowByShowUriRequest: { showUri: "spotify:show:YOUR_SHOW_ID" }) {
    analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_WINDOWED_AVERAGE_COMPLETION_RATE,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: WINDOW_LAST_NINETY_DAYS
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on PercentageValueFloat {
            value
          }
        }
      }
    }
  }
}
```

エピソードレベルの `EPISODE_PERFORMANCE`（視聴維持率カーブ）とは別指標。
streams/starts から導出する完了率とも異なり、期間全体の集計平均値。

---

### `Show` 型の新フィールド — スキーマ確認済み・未テスト（2026年5月追加）

以下のフィールドが 2026年5月アップデートで `Show` 型に追加された。
イントロスペクションでスキーマ上の存在は確認済みだが、**実クエリによる動作検証は未実施**。

**コアファン指標：**

| フィールド | 返り型 | 備考 |
|-----------|--------|------|
| `coreFanTimeSeries` | `GetCoreFanTimeSeriesResponse` | 熱心なリスナーの時系列 |
| `coreFanMetricSummary` | `GetCoreFanMetricSummaryResponse` | コアファン指標サマリー |
| `coreFanInsight` | `GetCoreFanInsightResponse` | コアファンに関するインサイト |

**スポンサーシップ：**

| フィールド | 返り型 | 備考 |
|-----------|--------|------|
| `sponsorshipAnalytics` | `ShowSponsorshipAnalytics` | スポンサーシップ分析 |
| `listSponsorships` | `ListSponsorshipsResponse` | スポンサーシップ一覧 |

**AI インサイト：**

| フィールド | 返り型 | 備考 |
|-----------|--------|------|
| `analyticsInsight` | `InsightResponse` | アナリティクスインサイト（AI生成の可能性） |
| `getEpisodeInsights` | `GetEpisodeInsightsByShowResponse` | エピソードごとのインサイト |

---

## 未取得・取得不可のオペレーション

| オペレーション名 | 取得不可の理由 |
|----------------|----------------|
| トランスクリプト編集 mutation | テストアカウントでは「文字起こし機能が利用できません」と表示され無効化。有料プランまたは特定条件でのみ有効化される機能と推定 |
| チャプター作成・更新・削除 mutation | テストショーはエピソード数が少なく資格なし。`getChaptersEligibility` query で条件を確認してから実装を試みること |
