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

初回調査：2026-05-26（Claude in Chrome MCP による JS インターセプター計測）／
追加調査：2026-06-11〜12（スキーマイントロスペクション・JS バンドル AST 解析。
「added Jun 2026」「2026年6月追加」と記載のあるオペレーションが対象）

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

### アナリティクス — エンゲージメントタブ（2026年6月追加）

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `getEngagementStats` | query | 視聴時間・平均視聴時間・コメント・フォロワー増減の日次時系列 |
| `getEngagementStatsNRT` | query | 視聴時間・平均視聴時間の期間合計（前期比付き） |
| `getEpisodeCompletionRates` | query | 最新10エピソードの完全再生率（初回7日間） |
| `getShowRetention` | query | 前週比リテンション率の週次時系列 |
| `getEpisodeTimeSeriesByMetric` | query | 単一エピソードの任意メトリクス時系列 |

### アナリティクス — ディスカバリータブ（2026年6月追加）

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `getShowDiscoveryFunnelStats` | query | インプレッション → 再生 → 平均完了率ファネル（転換率付き） |

### アナリティクス — ベンチマーク（2026年6月追加）

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `getBenchmarkTotal` | query | 類似ショーとのパーセンタイル比較（合計） |
| `getBenchmarkTimeSeries` | query | 類似ショーとのパーセンタイル比較（時系列） |

### アナリティクス — AI インサイト（2026年6月追加）

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `generateAnalyticsInsight` | mutation | AI インサイト生成を依頼 |
| `getAnalyticsInsight` | query | 生成済みインサイトを ID で取得 |
| `submitAnalyticsInsightFeedback` | mutation | インサイトへの評価（thumbs up/down）を送信 |

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

### アナリティクス — 概要タブ

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `getShowOnSpotifyStats` | query | Spotifyでのショー統計（期間変更時に発火） |
| `getShowOnSpotifyStatsNRT` | query | Spotifyでのショー統計 NRT（Near Real Time） |
| `getShowOverviewStatsNRT` | query | ショー全体概要統計 NRT |
| `getPublishedEpisodeCount` | query | 公開済みエピソード数取得 |
| `getShowMetadata` | query | ショーメタデータ + 全期間再生数合計（2026年6月追加） |
| `getShowAllPlatformsStats` | query | 全プラットフォームのストリーム+ダウンロード平均（2026年6月追加） |
| `getShowAllPlatformsStatsNRT` | query | 全期間全プラットフォームのストリーム+ダウンロード合計（2026年6月追加） |
| `getShowStreams` | query | Spotify 日次ストリーム時系列（2026年6月追加） |
| `getShowStreamsNRT` | query | 期間内 Spotify ストリーム合計（2026年6月追加） |
| `getShowStreamsAndDownloadsDaily` | query | 全プラットフォームの日次/週次/月次ストリーム+ダウンロード（2026年6月追加） |
| `getPerformanceStats` | query | 日次再生数・オーディエンス・フォロワー時系列（2026年6月追加） |
| `getPerformanceStatsNRT` | query | 再生数・オーディエンス・フォロワー合計（前期比付き、2026年6月追加） |
| `getShowTopEpisodesByImpressions` | query | インプレッション別トップエピソード（ソース別内訳付き、2026年6月追加） |
| `getShowTopEpisodesByMetric` | query | 任意 `EpisodeAnalyticsMetricType` でのトップエピソード（2026年6月追加） |
| `getShowEpisodesByMetric` | query | メトリクス付きページネーションエピソード一覧（2026年6月追加） |
| `hasVideoEpisodes` | query | ショーに動画エピソードがあるか（2026年6月追加） |
| `getPermissionsForShow` | query | 制作者のショー権限（2026年6月追加） |

### アナリティクス — オーディエンスタブ

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `getShowAudienceDemographicsStats` | query | 年齢・性別の内訳 |
| `getShowAudienceAllPlatformsGeoStats` | query | 全プラットフォームの地域別内訳 |
| `getShowAudienceAllPlatformsStats` | query | 全プラットフォーム統計 |
| `getShowSegmentedAudienceTotal` | query | 新規・リピーターのリスナー合計（2026年5月追加） |
| `getShowSegmentedAudienceTimeSeries` | query | 新規・リピーターの日次時系列（2026年5月追加） |
| `getAudienceGrowthTimeSeries` | query | コアファン / 発展途上 / 新規オーディエンスの日次時系列（2026年6月追加） |
| `getAudienceGrowthMetricSummary` | query | コアファンのサマリー（前期比付き、2026年6月追加） |
| `getAudienceGrowthInsights` | query | リテンション率・視聴時間のコアファン vs 全体比較（2026年6月追加） |

### アナリティクス全体（その他）

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `getFeatureEligibility` | query | 機能利用可否チェック |
| `getEpisodePlaysTotal` | query | エピソード総再生数取得 |

### エピソード一覧ページ

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `WebGetIndexedEpisodeList` | query | インデックス済みエピソード一覧取得 |
| `getEpisodesForShow` | query | ショーのエピソード一覧取得（投票ページ） |

### エピソード作成・設定ページ

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `getChaptersEligibility` | query | チャプター機能の利用資格確認 |
| `createEpisodeSmartlink` | mutation | エピソードスマートリンク作成 |
| `getLatestCommentsForShow` | query | ショー最新コメント一覧取得 |

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

!!! info "2026-06-11 に Plays の定義が変更（30秒 / AMP標準）"
    上記の実測は、2026-06-11 のプラットフォーム変更（再生数 =
    **30秒以上の視聴で1再生**、AMP業界標準準拠。看板指標は
    **Plays + Downloads** に交代、Streams + Downloads は「参照用」に降格）
    より前のもの。`SHOW_PLAYS` ≒ starts 合計などの数値的関係は新定義のもとでは
    成立しなくなっている可能性がある。詳細は[更新履歴](changelog.md)を参照。

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
  publishCommentByCommentUri(commentUri: String!)
}
```

```json
{ "commentUri": "spotify:comment:7cXkaBSy1gocZ0hAtS9WsJ" }
```

!!! note
    承認と同時に `markCommentAsRead` も自動発火する（2リクエストが連続して送信される）。

---

### コメント削除

```graphql
mutation deleteCommentByCommentUri {
  deleteCommentByCommentUri(commentUri: String!, filter: String!)
}
```

**`filter` の値**

| 値 | 用途 |
|----|------|
| `DELETE_COMMENT_FILTER_PUBLISHED` | 公開済みコメントを削除する |
| `DELETE_COMMENT_FILTER_NEEDS_REVIEW` | （推定）審査中コメントを削除する（未確認） |

```json
{
  "commentUri": "spotify:comment:7cXkaBSy1gocZ0hAtS9WsJ",
  "filter": "DELETE_COMMENT_FILTER_PUBLISHED"
}
```

---

### 制作者返信

```graphql
mutation createCommentReplyByCommentUri {
  createCommentReplyByCommentUri(
    parentCommentUri: String!
    replyStr: String!
  )
}
```

```json
{
  "parentCommentUri": "spotify:comment:7cXkaBSy1gocZ0hAtS9WsJ",
  "replyStr": "ご視聴ありがとうございます！"
}
```

!!! note
    返信はスレッドビュー（`?thread={COMMENT_ID}` URL パラメータ）から行われる。
    返信後、`GetCommentReplies` が自動発火してスレッドを更新する。

---

### コメント既読マーク

```graphql
mutation markCommentAsRead {
  markCommentAsRead(commentUri: String!)
}
```

```json
{ "commentUri": "spotify:comment:7cXkaBSy1gocZ0hAtS9WsJ" }
```

承認・返信・削除と同時に自動発火する。単独での呼び出しも可能。

---

### コメントピン留め

```graphql
mutation pinComment {
  pinComment(commentUri: String!)
}
```

```json
{ "commentUri": "spotify:comment:0tHI8bJTfued8QR8E2Rv6B" }
```

!!! note
    - 公開済みコメントにのみ適用可能
    - ピン留めできるコメントは1エピソードにつき1件のみ（UIの制約。API レベルでは未確認）
    - ピン留め後、`getPinnedCommentForEpisode` が自動発火して UI を更新する

---

### コメントピン留め解除

```graphql
mutation unPinComment {
  unPinComment(commentUri: String!)
}
```

```json
{ "commentUri": "spotify:comment:0tHI8bJTfued8QR8E2Rv6B" }
```

---

### アカウントブロック

```graphql
mutation blockUserForShow {
  blockUserForShow(username: String!, showUri: String!)
}
```

```json
{
  "username": "21yafw4q377x3jpbbcohdh4py",
  "showUri": "spotify:show:YOUR_SHOW_ID"
}
```

!!! note
    - `username` は Spotify 内部ユーザーID（`21xxxxx` 形式）。`getCommentsForEpisode` レスポンスの `author.uri` または `author.username` フィールドから取得する
    - ブロックは**ショー全体**（全エピソード）に適用される
    - UI ではコメント同時削除オプションがある。選択すると同じリクエストバッチで `deleteCommentByCommentUri` が発火する
    - `unblockUserForShow` が対称的に存在すると思われる（未確認）

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

### 番組横断コメント一覧取得（showByShowUri.showComments）— 2026-06-23 動作確認済み・2026-07-27 スキーマ修正＆投稿者フィールド追加

> **重要：** ネットワークキャプチャでは `getLatestCommentsForShow` というオペレーション名が見えるが、
> これは **Persisted Query のエイリアス名** であり、`/v2/graph-pq` スキーマの
> トップレベルフィールドには**存在しない**。
> 実際のパスは `showByShowUri` → `Show.showComments(…)` で、アナリティクス取得と同じ入り口を使う。

> **スキーマ修正（2026-07-27）：** 本ページの旧版には簡略化したシグネチャ
> （`primaryFilters:` 直指定・`comments{…}` 形式）を掲載していたが、
> 実装検証済みの正しい形は `listShowCommentsRequest` 入力オブジェクト＋
> `decoratedCommentOrReply` 戻り値である（以下）。

```graphql
query {
  showByShowUri(getShowByShowUriRequest: { showUri: "spotify:show:YOUR_SHOW_ID" }) {
    showComments(listShowCommentsRequest: {
      commentFilters: {
        primaryFilters: [LIST_COMMENT_PRIMARY_FILTER_PUBLISHED]
        typeFilters: [LIST_COMMENT_TYPE_FILTER_ROOT]
        secondaryFilters: []
      }
      sortOrder: LIST_COMMENT_SORT_ORDER_MOST_RECENT
      pagingInfo: { pageSize: 20, pageToken: "" }
    }) {
      decoratedCommentOrReply {
        episodeUri
        oneOfCommentOrReply {
          ... on DecoratedComment {
            commentUri
            commentStr
            createDate          # Long・Unixミリ秒
            oneOfAuthorDisplayMetadata {
              ... on UserAuthorDisplayMetadata {
                username        # 不透明なユーザーID文字列
                userFullName    # S4C UIに表示される表示名
                userCoverImageUrl
              }
            }
          }
        }
        episode {
          title
          publishedOn { seconds }
        }
      }
    }
  }
}
```

> **注意：** `primaryFilters` に `LIST_COMMENT_PRIMARY_FILTER_NEEDS_REVIEW` を含めると
> このクエリでは `DataFetchingException` が発生する（2026-06-23 確認）。
> `LIST_COMMENT_PRIMARY_FILTER_PUBLISHED` のみを指定すること
> （`getCommentsForEpisode` は両方の値を安全に受け付けるのと対照的）。

このクエリは**番組全体（全エピソード）のコメントを1リクエストで取得**できる。
収録前のコメントダイジェスト作成などに活用できる。

#### コメント投稿者メタデータ — 2026-07-27 発見

`DecoratedComment.oneOfAuthorDisplayMetadata` は **UNION型**で、2つの型を取りうる：

| 型 | 意味 | フィールド |
|----|------|-----------|
| `UserAuthorDisplayMetadata` | リスナーのコメント | `username`・`userFullName`・`userCoverImageUrl` |
| `ShowAuthorDisplayMetadata` | 番組（制作者）側の投稿 | （使用前にイントロスペクション推奨） |

`userFullName` がSpotifyアプリに表示される表示名。`username` は不透明なID文字列。
別途 `authorMetadata.oneOfAuthorMetadata` というUNION（`UserIdString` \| `ParentEntityUriString`）にも生IDが入る。

#### `DecoratedComment` の全フィールド一覧（2026-07-27 イントロスペクション）

`commentUri`・`parentEntityUri`・`entityUri`・`authorMetadata`・`createDate`（Long）・
`commentStr`・`language`・`initialModerationState`・`approvalDate`（Long）・
`numberOfReplies`（Int）・`hasParentEntityReacted`・`hasParentEntityReplied`・
`isAnySensitivityPolicyInitiallyViolated`・`hasParentEntityPinned`・
`containsBlockedWords`・`surfaceType`・`oneOfAuthorDisplayMetadata`

`hasParentEntityReplied` / `hasParentEntityPinned` は、返信スレッドを取得せずに
「未返信コメント一覧」を作るのに便利。

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

**スポンサーシップ：**

| フィールド | 返り型 | 備考 |
|-----------|--------|------|
| `sponsorshipAnalytics` | `ShowSponsorshipAnalytics` | スポンサーシップ分析 |
| `listSponsorships` | `ListSponsorshipsResponse` | スポンサーシップ一覧 |

---

## エンゲージメントタブ クエリ（2026年6月追加）

> 2026-06-12 取得。S4C アナリティクス JavaScript バンドル
> (`microfrontend-analytics-cdn.spotifycdn.com`) から GraphQL AST を
> 解析・再構成。本番バンドルに実在することを確認済み。

### オペレーション一覧

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `getEngagementStats` | query | 視聴時間・平均視聴時間・コメント・フォロワー増減の日次時系列 |
| `getEngagementStatsNRT` | query | 視聴時間・平均視聴時間の期間合計（前期比付き） |
| `getShowRetention` | query | 前週比リテンション率の週次時系列 |
| `getEpisodeCompletionRates` | query | 最新10エピソードの完全再生率（初回7日間） |
| `getEpisodeTimeSeriesByMetric` | query | 単一エピソードの任意メトリクス時系列 |

### `getEngagementStats` — エンゲージメント日次時系列

エンゲージメントタブのロード時に発火。視聴時間（`SHOW_CONSUMPTION`）と
平均視聴時間（`SHOW_AVERAGE_CONSUMPTION_TIME`）の日次時系列を返す。

```graphql
query getEngagementStats(
  $showUri: ShowUri!,
  $dateRangeWindow: AnalyticsWindow!,
  $customDateRange: CustomDateRangeInput,
  $publishedAfter: TimestampInput,
  $publishedBefore: TimestampInput,
  $hasComments: Boolean!
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    uri
    consumptionTimeDaily: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_CONSUMPTION,
      aggregationType: AGGREGATION_TYPE_DAILY,
      window: $dateRangeWindow,
      customDateRange: $customDateRange
    }) {
      startDate
      endDate
      analyticsValue {
        analyticsValue {
          __typename
          ... on TimeSeriesValue {
            points {
              date
              endDate
              value {
                __typename
                ... on ConsumptionValue {
                  totalConsumptionHours
                }
              }
            }
          }
        }
      }
    }
    avgConsumptionTimeDaily: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_AVERAGE_CONSUMPTION_TIME,
      aggregationType: AGGREGATION_TYPE_DAILY,
      window: $dateRangeWindow,
      customDateRange: $customDateRange
    }) {
      startDate
      endDate
      analyticsValue {
        analyticsValue {
          __typename
          ... on TimeSeriesValue {
            points {
              date
              endDate
              value {
                __typename
                ... on AverageConsumptionTimeValue {
                  averageConsumptionSeconds
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

**新レスポンス型：**

| 型名 | フィールド | 説明 |
|------|-----------|------|
| `ConsumptionValue` | `totalConsumptionHours` | 期間内の総視聴時間（時間単位） |
| `AverageConsumptionTimeValue` | `averageConsumptionSeconds` | リスナー1人あたりの平均視聴秒数 |

**ダッシュボード指標との対応：**

| ダッシュボード表示 | MetricType | レスポンスフィールド |
|-----------------|-----------|-------------------|
| 視聴時間 | `SHOW_CONSUMPTION` | `ConsumptionValue.totalConsumptionHours` |
| 平均視聴時間 | `SHOW_AVERAGE_CONSUMPTION_TIME` | `AverageConsumptionTimeValue.averageConsumptionSeconds` |

### `getEngagementStatsNRT` — 期間合計（前期比付き）

期間合計（時系列ではない）を `periodOverPeriodPercentageDiff` 付きで返す。
`+36.6%` / `+7.4%` バッジの値がこのフィールドから取得できる（例：`0.366` = +36.6%）。

```graphql
query getEngagementStatsNRT(
  $showUri: ShowUri!,
  $dateRangeWindow: AnalyticsWindow!,
  $customDateRange: CustomDateRangeInput,
  $includePeriodComparison: Boolean!,
  $hasComments: Boolean!
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    uri
    consumptionTimeTotal: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_CONSUMPTION,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: $dateRangeWindow,
      customDateRange: $customDateRange,
      includePeriodComparison: $includePeriodComparison
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on ConsumptionValue {
            totalConsumptionHours
          }
        }
        periodOverPeriodPercentageDiff
      }
    }
    avgConsumptionTimeTotal: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_AVERAGE_CONSUMPTION_TIME,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: $dateRangeWindow,
      customDateRange: $customDateRange,
      includePeriodComparison: $includePeriodComparison
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on AverageConsumptionTimeValue {
            averageConsumptionSeconds
          }
        }
        periodOverPeriodPercentageDiff
      }
    }
  }
}
```

**重要フィールド：** `analyticsValue.periodOverPeriodPercentageDiff` — ダッシュボードの `+36.6%` / `+7.4%` バッジの値（例：`0.366` = +36.6%）。

### `getShowRetention` — 前週比リテンション率

今週のリスナーのうち前週も聴いていた割合を週次時系列で返す。

```graphql
query getShowRetention(
  $showUri: ShowUri!,
  $dateRangeWindow: AnalyticsWindow!,
  $publishedAfter: TimestampInput,
  $publishedBefore: TimestampInput
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    uri
    retentionWeekly: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_RETENTION,
      aggregationType: AGGREGATION_TYPE_WEEKLY,
      window: $dateRangeWindow
    }) {
      startDate
      endDate
      analyticsValue {
        analyticsValue {
          __typename
          ... on TimeSeriesValue {
            points {
              date
              endDate
              value {
                __typename
                ... on RatioValueFloat {
                  value
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

`RatioValueFloat.value` は 0〜1 の float（例：`0.537` = 53.7%）。
`WINDOW_LAST_NINETY_DAYS` + `AGGREGATION_TYPE_WEEKLY` でダッシュボードの
90日リテンショングラフを再現できる。

### `getEpisodeCompletionRates` — エピソード完全再生率

最新10エピソードについて、公開後7日間（`WINDOW_FIRST_SEVEN_DAYS`）の
平均完全再生率（`EPISODE_AVERAGE_COMPLETION_RATE`）を取得する。

**新 `AnalyticsWindow` 値：** `WINDOW_FIRST_SEVEN_DAYS` — エピソード公開後7日間

レスポンス：`PercentageValueFloat.value`（例：`0.56` = 56%）

```graphql
query getEpisodeCompletionRates($showUri: ShowUri!) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    latestEpisodes: episodesV2(listEpisodesV2Request: {
      sort: {sortBy: PUBLISHED_ON, sortOrder: DESC},
      filter: PUBLISHED_EPISODES,
      pagination: {pageSize: 10},
      episodeMetadataFieldMask: {
        paths: [
          "hosted_episode.title", "hosted_episode.published_on", "hosted_episode.uri",
          "non_hosted_episode.title", "non_hosted_episode.published_on", "non_hosted_episode.uri"
        ]
      }
    }) {
      items {
        uri
        title
        publishedOn { seconds }
        completionRate: analyticsBatch(batchGetEpisodeAnalyticsItem: {
          episodeAnalyticsMetricType: EPISODE_AVERAGE_COMPLETION_RATE,
          aggregationType: AGGREGATION_TYPE_TOTAL,
          window: WINDOW_FIRST_SEVEN_DAYS
        }) {
          analyticsValue {
            analyticsValue {
              __typename
              ... on PercentageValueFloat { value }
            }
          }
        }
      }
    }
  }
}
```

### `getEpisodeTimeSeriesByMetric` — エピソードメトリクス時系列

単一エピソードの任意 `EpisodeAnalyticsMetricType` について時系列データを取得する。

```graphql
query getEpisodeTimeSeriesByMetric(
  $episodeUri: EpisodeUri!,
  $metricType: EpisodeAnalyticsMetricType!,
  $window: AnalyticsWindow!,
  $aggregationType: AggregationType!
) {
  episodeByUri(getEpisodeRequest: {episodeUri: $episodeUri}) {
    analyticsBatch(batchGetEpisodeAnalyticsItem: {
      episodeAnalyticsMetricType: $metricType,
      aggregationType: $aggregationType,
      window: $window
    }) {
      startDate
      endDate
      analyticsValue {
        analyticsValue {
          __typename
          ... on TimeSeriesValue {
            points {
              date
              value {
                __typename
                ... on CountValueLong { longValue: value }
                ... on ConsumptionValue {
                  totalConsumptionHours
                  foregroundConsumptionPercent
                }
                ... on RatioValueFloat { floatValue: value }
              }
            }
          }
        }
      }
    }
  }
}
```

**注意：** `ConsumptionValue` はエピソードレベルで `foregroundConsumptionPercent`（フォアグラウンド再生率）も取得できる。

---

## オーディエンス成長クエリ（コアファンモデル、2026年6月確定）

これまで「スキーマ確認済み・未テスト」として記載していた
`coreFanTimeSeries` / `coreFanMetricSummary` / `coreFanInsight` フィールドの
実際のクエリ構造をバンドルから取得・確定した。

オーディエンスは3層に分類される：**コアファン**（高エンゲージメント）・
**発展途上オーディエンス**・**新規オーディエンス**。

### オペレーション一覧

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `getAudienceGrowthTimeSeries` | query | 3層オーディエンス日次時系列 |
| `getAudienceGrowthMetricSummary` | query | サマリー（前期比・獲得コアファン数付き） |
| `getAudienceGrowthInsights` | query | コアファンと全体オーディエンスの行動比較 |

### `getAudienceGrowthTimeSeries` — コアファン層別日次時系列

**重要：** `startDate` / `endDate` は `"YYYY-MM-DD"` 形式の文字列。
他のアナリティクスクエリが使う `AnalyticsWindow` enum ではない。

```graphql
query getAudienceGrowthTimeSeries(
  $showUri: ShowUri!, $startDate: String!, $endDate: String!
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    coreFanTimeSeries(getCoreFanTimeSeriesRequest: {
      startDate: $startDate, endDate: $endDate
    }) {
      timeSeries {
        date
        coreFans
        coreFansPercent
        developingAudience
        developingAudiencePercent
        newAudience
        newAudiencePercent
        totalAudience
        totalAudiencePercent
      }
    }
  }
}
```

### `getAudienceGrowthMetricSummary` — サマリーと前期比

```graphql
query getAudienceGrowthMetricSummary(
  $showUri: ShowUri!, $startDate: String!, $endDate: String!, $isAllTime: Boolean!
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    coreFanMetricSummary(getCoreFanMetricSummaryRequest: {
      startDate: $startDate, endDate: $endDate, isAllTime: $isAllTime
    }) {
      summary {
        newAudience
        newAudiencePercent
        developingAudience
        developingAudiencePercent
        coreFans
        coreFansPercent
        totalAudience
        totalAudiencePercent
        gainedCoreFans
        startDate
        endDate
        lookbackStartDate
        lookbackEndDate
      }
    }
  }
}
```

`gainedCoreFans` = 期間中に新たに獲得したコアファン数。

### `getAudienceGrowthInsights` — コアファン行動インサイト

```graphql
query getAudienceGrowthInsights(
  $showUri: ShowUri!, $startDate: String!, $endDate: String!
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    coreFanInsight(getCoreFanInsightRequest: {
      startDate: $startDate, endDate: $endDate
    }) {
      insight {
        averageEpisodeRetentionCoreFansPercent
        averageEpisodeRetentionTotalAudiencePercent
        averageEpisodeRetentionDiffPercent
        averageConsumptionHoursCoreFans
        averageConsumptionHoursTotalAudience
        averageConsumptionHoursDiffPercent
      }
    }
  }
}
```

---

## ベンチマーククエリ（2026年6月追加）

### `getBenchmarkTotal` / `getBenchmarkTimeSeries`

自分のショーを類似ショーのパーセンタイル（20/40/50/60/80）と比較する。

```graphql
query getBenchmarkTotal(
  $showUri: ShowUri!,
  $window: AnalyticsWindow!,
  $benchmarkMetricType: BenchmarkMetricType!,
  $benchmarkEpisodePool: BenchmarkEpisodePool!
) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    benchmarkTotal: analytics(getShowAnalyticsRequest: {
      showAnalyticsMetricType: SHOW_BENCHMARK_EPISODE,
      aggregationType: AGGREGATION_TYPE_TOTAL,
      window: $window,
      benchmarkParams: {
        metricType: $benchmarkMetricType,
        episodePool: $benchmarkEpisodePool,
        percentiles: [20, 40, 50, 60, 80]
      }
    }) {
      analyticsValue {
        analyticsValue {
          __typename
          ... on BenchmarkTimeSeriesValue {
            points {
              label
              percentiles { percentile value }
            }
          }
        }
      }
    }
  }
}
```

#### `BenchmarkMetricType` enum 値

| 値 | 説明 |
|----|------|
| `BENCHMARK_ALL_PLATFORM_AUDIENCE` | 全プラットフォームユニークオーディエンス |
| `BENCHMARK_AVG_CONSUMPTION_TIME` | 平均視聴時間 |
| `BENCHMARK_COMMENTS` | コメント数 |
| `BENCHMARK_CONSUMPTION_TIME` | 総視聴時間 |
| `BENCHMARK_EPISODE_COMPLETION_PERCENT` | エピソード完全再生率 |
| `BENCHMARK_IMPRESSIONS` | インプレッション数 |
| `BENCHMARK_OFF_PLATFORM_AUDIENCE` | オフプラットフォームオーディエンス |
| `BENCHMARK_OFF_PLATFORM_DOWNLOADS` | オフプラットフォームダウンロード数 |
| `BENCHMARK_PLAYS` | 再生数 |
| `BENCHMARK_PLAYS_AND_DOWNLOADS` | 再生数＋ダウンロード数 |
| `BENCHMARK_RETENTION` | 前週比リテンション率 |
| `BENCHMARK_SPOTIFY_AUDIENCE` | Spotify 専用オーディエンス |
| `BENCHMARK_WATCH_TIME_PERCENT` | 視聴時間率（動画） |

#### `BenchmarkEpisodePool` enum 値

| 値 | 説明 |
|----|------|
| `ALL_EPISODES` | すべてのエピソード |
| `LAST_10_EPISODES` | 最新10エピソード |

---

## AI アナリティクスインサイト（2026年6月追加）

| オペレーション名 | 種別 | 用途 |
|----------------|------|------|
| `generateAnalyticsInsight` | mutation | AI インサイト生成を依頼 |
| `getAnalyticsInsight` | query | 生成済みインサイトを ID で取得 |
| `submitAnalyticsInsightFeedback` | mutation | インサイトへの評価（thumbs up/down）を送信 |

```graphql
mutation generateAnalyticsInsight($generateInsightRequest: GenerateInsightRequestInput!) {
  generateAnalyticsInsight(generateInsightRequest: $generateInsightRequest) {
    insightId
    state
    payload
    generatedAt
  }
}

query getAnalyticsInsight($showUri: ShowUri!, $insightId: String!) {
  showByShowUri(getShowByShowUriRequest: {showUri: $showUri}) {
    analyticsInsight(getInsightRequest: {insightId: $insightId}) {
      insightId
      state
      payload
      generatedAt
    }
  }
}
```

---

## enum 追加値（2026年6月）

### `ShowAnalyticsMetricType` 追加値

| 値 | 説明 |
|----|------|
| `SHOW_CONSUMPTION` | 総視聴時間 → `ConsumptionValue` |
| `SHOW_AVERAGE_CONSUMPTION_TIME` | 平均視聴時間 → `AverageConsumptionTimeValue` |
| `SHOW_RETENTION` | 前週比リテンション率 → `TimeSeriesValue<RatioValueFloat>` |
| `SHOW_BENCHMARK_EPISODE` | ベンチマーク比較（`benchmarkParams` 引数必須） |
| `SHOW_STREAMS_AND_DOWNLOADS_AVERAGE` | 全プラットフォーム移動平均 |
| `SHOW_STREAMS_AND_DOWNLOADS_BY_APP` | プラットフォーム分布（Apple / Spotify / Amazon 等） |
| `SHOW_STREAMS_AND_DOWNLOADS_BY_DEVICE` | デバイス種別分布 |
| `SHOW_STREAMS_AND_DOWNLOADS_BY_GEO_ALL_PLATFORMS` | 全プラットフォーム地域分布 |
| `TOP_EPISODES_BY_IMPRESSIONS_FACETED` | インプレッション別トップエピソード（ソース別内訳付き） |
| `SHOW_IMPRESSIONS_FACETED` | ファセット付きインプレッション |
| `SHOW_STREAMS_FACETED` | ファセット付きストリーム |
| `SHOW_FOLLOWER_GROWTH` | フォロワー増減時系列 |

### `EpisodeAnalyticsMetricType` 追加値

| 値 | 説明 |
|----|------|
| `EPISODE_AVERAGE_COMPLETION_RATE` | 平均完全再生率（0〜1 float） |
| `EPISODE_AVERAGE_CONSUMPTION_TIME` | 平均視聴時間 |
| `EPISODE_CONSUMPTION` | 総視聴時間 |
| `EPISODE_AUDIENCE_SIZE` | オーディエンスサイズ |
| `EPISODE_IMPRESSIONS` | インプレッション |
| `EPISODE_IMPRESSIONS_FACETED` | ファセット付きインプレッション |
| `EPISODE_IMPRESSIONS_FUNNEL` | インプレッションファネル |
| `EPISODE_IMPRESSIONS_TO_PLAYS_RATE` | インプレッション→再生転換率 |
| `EPISODE_DISCOVERY_FUNNEL` | エピソードレベルのディスカバリーファネル |
| `EPISODE_PLAYS_FACETED` | ファセット付き再生 |
| `EPISODE_STREAMS_FACETED` | ファセット付きストリーム |
| `EPISODE_RETENTION` | エピソードレベルのリテンション |
| `EPISODE_SPOTIFY_PLAYS_BY_COUNTRY` | Spotify 再生の国別分布 |

### `AnalyticsWindow` 追加値

| 値 | 説明 |
|----|------|
| `WINDOW_FIRST_SEVEN_DAYS` | エピソード公開後7日間 |
| `WINDOW_FIRST_THIRTY_DAYS` | エピソード公開後30日間 |
| `WINDOW_FIRST_SIXTY_DAYS` | エピソード公開後60日間 |
| `WINDOW_LAST_SIXTY_DAYS` | 過去60日間 |
| `WINDOW_SINCE_PUBLISHED` | 公開日以降すべて |
| `WINDOW_UNSPECIFIED` | 未指定（全期間平均等に使用） |

---

## 未取得・取得不可のオペレーション

| オペレーション名 | 取得不可の理由 |
|----------------|----------------|
| トランスクリプト編集 mutation | テストアカウントでは「文字起こし機能が利用できません」と表示され無効化。有料プランまたは特定条件でのみ有効化される機能と推定 |
| チャプター作成・更新・削除 mutation | テストショーはエピソード数が少なく資格なし。`getChaptersEligibility` query で条件を確認してから実装を試みること |

---

## showsForUser — 認証ユーザーの番組一覧取得

**検証日: 2026-06-21**

認証ユーザーが管理するすべての番組を取得する。ダッシュボードを操作せずに
Show ID を解決するときに使用する。

```graphql
query {
  showsForUser {
    shows {
      name
      uri
    }
    pagination {
      # 追加のページネーションフィールドが利用可能
    }
  }
}
```

### レスポンス例

```json
{
  "data": {
    "showsForUser": {
      "shows": [
        {
          "name": "ぼくらの戦略論",
          "uri": "spotify:show:5vXOMdLUviA9dGrWXy79cF"
        }
      ]
    }
  }
}
```

Show ID は `uri` の末尾セグメント（`spotify:show:<SHOW_ID>` の `<SHOW_ID>` 部分）。

### Python サンプル

```python
import requests

headers = {
    "Authorization": f"Bearer {bearer}",
    "Content-Type": "application/json",
}
res = requests.post(
    "https://creators-graph.spotify.com/v2/graph-pq",
    headers=headers,
    json={"query": "{ showsForUser { shows { name uri } } }"},
)
shows = res.json()["data"]["showsForUser"]["shows"]
for show in shows:
    show_id = show["uri"].split(":")[-1]
    print(show_id, show["name"])
```

### 備考

- 65件以上でもページネーション不要で一括取得可能（実績値）。
- `Show` 型は `name` / `uri` 以外にも多数のフィールドを持つ。
  `__type(name: "Show")` でスキーマを確認できる。
- anchor.fm REST の `GET /v3/stations/status` は同等の機能を持つが、
  SpotifyConnector で取得したトークンでは 403 になる。このクエリを使うこと。
