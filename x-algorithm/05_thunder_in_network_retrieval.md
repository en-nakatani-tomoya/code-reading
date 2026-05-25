# Thunder: In-network Retrieval を理解する

対象コード:

- `external_repo/x-algorithm/thunder/`
- `external_repo/x-algorithm/home-mixer/sources/thunder_source.rs`

---

## 0. ひとことで

Thunder は **フォロー中ユーザーの直近投稿を高速に返す in-network retrieval service**。

投稿の良し悪し、ユーザーごとの深いおすすめ度、engagement 予測を判断するサービスではない。基本的には、投稿イベントを Kafka から受け取り、author ごとの軽量な投稿 index をメモリに持ち、リクエスト時に「このユーザーがフォローしている人たちの最近の投稿」を集めて返す。

```text
Thunder = in-network の新鮮な候補投稿を作る軽量 retrieval
Phoenix / RankingScorer = 候補のおすすめ度・最終ランキングを決める
```

---

## 1. Thunder がやっていること

### 1.1 Kafka の投稿イベントを読む

Thunder は tweet create / delete event を Kafka から読み、配信用に必要な最小限の情報だけを `LightPost` に変換する。

`LightPost` に入る主な情報:

| Field | 意味 |
|---|---|
| `post_id` | 投稿 ID |
| `author_id` | 投稿者 ID |
| `created_at` | 作成時刻 |
| `in_reply_to_post_id` / `in_reply_to_user_id` | reply 情報 |
| `is_retweet` / `is_reply` | retweet / reply 判定 |
| `source_post_id` / `source_user_id` | retweet 元 |
| `has_video` | video 候補か |
| `conversation_id` | 会話 root |

旧形式の tweet event から `LightPost` を作る処理は `kafka/tweet_events_listener.rs`、新形式の `InNetworkEvent` から作る処理は `kafka/tweet_events_listener_v2.rs` にある。

### 1.2 author ごとのメモリ index を作る

中心は `posts/post_store.rs` の `PostStore`。

`PostStore` は次の index を持つ。

| Index | 役割 |
|---|---|
| `posts` | `post_id -> LightPost` の本体 map |
| `original_posts_by_user` | author ごとの通常投稿 |
| `secondary_posts_by_user` | author ごとの reply / retweet |
| `video_posts_by_user` | author ごとの video 候補 |
| `deleted_posts` | delete event 反映用 |

insert 時には、未来投稿・retention window 外の古い投稿・削除済み投稿を除外する。さらに original / secondary / video の各 index に `TinyPost { post_id, created_at }` を積む。

### 1.3 古い投稿・削除投稿を掃除する

Thunder は serving 開始時に Kafka catch-up を待ち、`PostStore::finalize_init()` で index を整える。その後、定期的に `start_auto_trim()` で retention window 外の投稿を削除する。

delete event は `deleted_posts` に記録され、`posts` からも除外される。create / delete の順序が前後しても、初期化時に削除済み投稿を再除去する。

---

## 2. Thunder が返すもの

Thunder の gRPC service は `InNetworkPostsService`。

主な endpoint:

```text
get_in_network_posts(GetInNetworkPostsRequest)
  -> GetInNetworkPostsResponse { posts: Vec<LightPost> }
```

リクエストで受け取る主な入力:

| Input | 意味 |
|---|---|
| `user_id` | request user |
| `following_user_ids` | フォロー中ユーザー一覧 |
| `exclude_tweet_ids` | 既読など除外したい投稿 |
| `max_results` | 返す最大件数 |
| `is_video_request` | video 専用取得か |

`following_user_ids` が空で debug request の場合だけ、Thunder 側が Strato から following list を取りに行く。通常の home-mixer 経由では、home-mixer 側の hydrated query から following list が渡される。

---

## 3. retrieval の中身

Thunder の候補取得は、次のような軽量処理。

```text
following_user_ids を受け取る
  -> 各 author の original_posts / secondary_posts / video_posts を見る
  -> exclude_tweet_ids を除外する
  -> deleted_posts を除外する
  -> author ごとの取得上限をかける
  -> reply / retweet の一部条件をチェックする
  -> created_at の新しい順に並べる
  -> max_results 件だけ返す
```

重要なのは、Thunder 内の `score_recent()` は名前こそ score だが、実体は `created_at` 降順ソートであること。

```text
score_recent(posts)
  = posts.sort_by(created_at desc)
  = newer posts first
```

つまり Thunder は「品質スコア」や「個人化 relevance score」を計算していない。新しさ、フォロー関係、除外条件、author ごとの上限で、in-network の候補集合を作る。

---

## 4. Thunder のデータを誰が消費するか

主な消費者は `home-mixer/sources/thunder_source.rs`。

`ThunderSource` は `PhoenixCandidatePipeline` の `sources` の1つとして登録される。

```text
PhoenixCandidatePipeline.sources
  - ThunderSource
  - TweetMixerSource
  - PhoenixSource
  - PhoenixTopicsSource
  - PhoenixMOESource
  - CachedPostsSource
```

`ThunderSource` は Thunder の `get_in_network_posts()` を呼び、返ってきた `LightPost` を `PostCandidate` に変換する。

```text
LightPost
  post_id
  author_id
  in_reply_to_post_id
  source_post_id
  conversation_id

-> PostCandidate
  tweet_id
  author_id
  in_reply_to_tweet_id
  retweeted_tweet_id
  ancestors
  served_type = ForYouInNetwork or RankedFollowing
```

変換後の `PostCandidate` は、Phoenix retrieval 由来の候補と同じ `PhoenixCandidatePipeline` に合流し、後段の hydrators / filters / scorers / selector を通る。

---

## 5. 最終 ranking までの流れ

Thunder が返した時点では、投稿はまだ最終 ranking 済みではない。

```text
Thunder
  Kafka tweet events
    -> LightPost
    -> PostStore in-memory index
    -> get_in_network_posts()
    -> recent-first LightPost list

home-mixer / ThunderSource
  -> LightPost を PostCandidate に変換
  -> PhoenixCandidatePipeline の candidates に合流

PhoenixCandidatePipeline
  -> hydrators
  -> filters
  -> PhoenixScorer
  -> RankingScorer
  -> VMRanker
  -> TopKScoreSelector
  -> final ranked posts
```

Thunder は **in-network 候補の供給元**。最終的なおすすめ度や表示順位は、その後の Phoenix scoring / RankingScorer / VMRanker / TopK selection で決まる。

---

## 6. reply 情報の副次利用

`ThunderSource` は、Thunder が返した `LightPost` の reply 情報から `InNetworkReply` を作り、`query.in_network_replies` に保存する。

これは後段の `FollowingRepliedUsersHydrator` が使う。

用途は、選ばれた root post に対して「フォロー中ユーザーがこの投稿に返信している」ことを facepile 的に表示するための情報付与。

```text
ThunderSource
  -> posts から reply 情報を抽出
  -> query.in_network_replies に保存

FollowingRepliedUsersHydrator
  -> root tweet ごとに reply author を集計
  -> candidate.following_replied_user_ids に入れる
```

この情報も ranking 本体ではなく、選択後候補への文脈付与に近い。

---

## 7. 読むときのチェックポイント

- Thunder は ML ranker ではなく、author timeline 型の in-network retrieval
- `LightPost` は ranking 用の豊富な特徴量ではなく、候補化に必要な軽量投稿情報
- `PostStore` は author ごとの最近投稿 index
- `score_recent()` は新しさ順ソートであり、品質評価ではない
- Thunder の直接消費者は `home-mixer` の `ThunderSource`
- Thunder 由来候補は、その後 `PhoenixCandidatePipeline` で他 source の候補と同じように scoring / Top-K される

