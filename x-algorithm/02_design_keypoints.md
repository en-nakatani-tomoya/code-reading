# 設計上のキーポイント

`xai-org/x-algorithm` を読み解くと、システムには一貫した設計思想がある。本ドキュメントは「なぜそうなっているのか」「何を捨てて何を取ったのか」を体系化する。

前提となる全体像は [`00_overview.md`](./00_overview.md)、オーケストレーションの流れは [`01_orchestration_flow.md`](./01_orchestration_flow.md) を参照。

---

## 0. 設計思想を3行で

1. **手作業を全部捨てて transformer に任せる** — 特徴量設計もヒューリスティクスも消滅。学習可能な巨大モデル1本で勝負。
2. **ML と システム設計の co-design** — candidate isolation のように「ML 側の制約が、システム側の最適化（キャッシュ・並列化）を可能にする」設計が随所にある。
3. **trait 1本ですべてを記述する** — sources / hydrators / filters / scorers… すべて trait 化。trait 単位で観測・並列化・Feature switch ゲートが無料で付く。

---

## 1. ML モデルの設計

### 1.1 No hand-engineered features (手作業の特徴量を全廃)

**何**: ユーザーの「いいね/返信/リポスト…」のエンゲージメント履歴のみが入力。SimClusters, TwHIN, RealGraph 等の特徴量パイプラインは存在しない。

**なぜ**:
- 旧 Twitter の MaskNet (48M params, 大量の hand-crafted features) は「特徴量パイプラインが大きすぎてどこから何が来ているか追えない」状態だった
- transformer に直接 raw sequence を食わせると、特徴量設計が「学習目的」と一致する保証がある
- データパイプライン・サービングインフラの複雑性が劇的に減る (README "significantly reduces the complexity")

**コスト**:
- モデルが超巨大に (Phoenix は production で多層・広幅)
- 推論レイテンシ・GPU コスト増
- 解釈可能性ほぼゼロ → デバッグは「スコアダンプ + A/B」頼り

**参照**: [`00_overview.md`](./00_overview.md) §5, README "Key Design Decisions"

### 1.2 Multi-action prediction — 19アクション同時予測

**何**: 単一の「relevance score」ではなく、19種類のアクション確率を同時に出す:

```
positive: favorite, reply, retweet, photo_expand, click, profile_click,
          vqv, share, share_via_dm, share_via_copy_link, dwell,
          quote, quoted_click, quoted_vqv, follow_author
negative: not_interested, block_author, mute_author, report, not_dwelled
```

最終スコア = `Σ wᵢ · Pᵢ` (`home-mixer/scorers/ranking_scorer.rs:125-173`)

**なぜ**:
- 「いいねされやすい」と「dwell される (長く読まれる)」は別物。1スコアに潰すと「煽り投稿が like 多い → 上位」のような失敗パターンに陥る
- ネガティブアクション (`block, mute, report` → 負の重み) を**明示的にスコアに組み込める**
- 重みは Feature switch で動的調整可能 (`ScoringWeights::from_params`) → 実験・国別・新規ユーザー向けに変えられる

**コスト**: マルチタスク学習の難しさ (ヘッドごとに loss balance が必要)。重み調整がプロダクト判断になり「何を最大化するかの意思決定」が政治的になる。

**参照**: `home-mixer/scorers/ranking_scorer.rs:12-115` の `ScoringWeights` 構造体

### 1.3 Candidate isolation — attention で候補同士を見せない

**何**: ランキング transformer の attention mask で「候補は他の候補に attend できない、自分と user/history だけ」を強制 (`phoenix/grok.py:44`, `phoenix/test_recsys_model.py:80`)。

```
       User  History  Candidates
User    ✓     ✓        ✗
Hist    ✓     ✓        ✗
Cand    ✓     ✓        diagonal only (自身のみ)
```

**なぜ — 重要設計**:
1. **スコアの一貫性**: 同じ post を異なるバッチに入れても同じスコアが出る。バッチ内の他候補に依存しない。
2. **キャッシュ可能**: post スコアをユーザー context だけの関数として **Redis にキャッシュできる** (`RedisPostCandidateCacheSideEffect`)。
3. **並列化容易**: バッチ分割が安全。ABテストでも比較が破壊されない。

**コスト**: 候補間の interaction (例: 「似た投稿が並ばないように」) を attention でモデル化できない → **post-processing で diversity を入れる必要** (§2.3 の AuthorDiversityScorer)。

**参照**: `phoenix/README.md:121-161` の attention mask 可視化、`phoenix/grok.py:572`

### 1.4 Hash-based embeddings — 語彙サイズの解放

**何**: post_id / user_id をそのまま embedding lookup せず、複数のハッシュ関数で hash → 加算する (multi-hash embeddings)。

**なぜ**:
- X の規模 (1日5億投稿) で「全 post_id ごとに embedding を持つ」のは現実的でない (`O(N)` メモリ)
- multi-hash trick で **固定サイズの embedding table** で十分な表現力が出せる (hash collision を複数 hash で緩和)
- 新しい post_id (cold start) でも即座に embedding が引ける

**コスト**: hash collision で「全然違う2つの post が似た embedding を持つ」ことがある (確率的)。

**参照**: README "Hash-Based Embeddings", `phoenix/recsys_model.py`

### 1.5 Two-tower retrieval — 出力ベクトルの dot product だけで検索

**何**:
- User Tower: ユーザー履歴 → user embedding `[B, D]`
- Candidate Tower: 全投稿 → post embedding `[N, D]`
- 検索 = 内積で top-K (ANN)

**なぜ**:
- N=数千万〜億の候補を transformer で1個ずつスコアリングするのは無理
- Candidate Tower の出力は **オフラインで事前計算可能**。ユーザーリクエスト時は user tower の forward + ANN だけ
- 業界標準パターン (YouTube/Pinterest/Instagram 全て同じ)

**コスト**: dot product だけでは複雑な interaction (例: ユーザーの**最新の興味変化**) を捉えにくい → ランキング段で transformer に再評価させる二段構成が必要。

**参照**: `phoenix/recsys_retrieval_model.py`, README "Retrieval: Two-Tower Model"

---

## 2. スコアリング・ランキングの設計

### 2.1 Weighted sum with positive/negative split

`ranking_scorer.rs:175-183` の `offset_score` が興味深い:

```rust
fn offset_score(combined_score: f64, w: &ScoringWeights) -> f64 {
    if w.total_sum == 0.0 {
        combined_score.max(0.0)
    } else if combined_score < 0.0 {
        (combined_score + w.negative_sum) / w.total_sum * NEGATIVE_SCORES_OFFSET
    } else {
        combined_score + NEGATIVE_SCORES_OFFSET
    }
}
```

**設計の意図**:
- 通常スコア (≥0): 一律に `NEGATIVE_SCORES_OFFSET` だけ底上げ
- ネガティブスコア (<0, つまり「ブロックされそう」な投稿): 正規化して **0未満の小さい値** に押し込む
- 結果: 「ネガティブな投稿は最低限のオフセットすら超えない」がスコア比較で保証される

**なぜ**:
- ネガティブ投稿を**完全に削除**するのではなく**低スコアで残す**ことで、後段の diversity 調整やデバッグで観察可能にする
- スコア空間が「正常域」と「危険域」に物理的に分離される

### 2.2 Author Diversity — 同一著者のスコアを後ろほど減衰

`author_diversity_scorer.rs:29-32` + `ranking_scorer.rs:186-217`:

```
multiplier(position) = (1 - floor) * decay^position + floor
```

**何**:
- スコア順にソートしてから、各候補について「これまで何回同じ著者が出てきたか」を数える
- N 回目の出現は `decay^N` 倍に減衰
- `floor` で下限を確保 (完全には 0 にしない)

**なぜ**:
- candidate isolation でモデル自体は「同じ著者ばかり」を回避できない (他候補が見えないから)
- post-processing で機械的に強制 → モデルの責務とフィードの責務を分離

**コスト**: 「本当にその著者の投稿が3つとも見たい」ユーザーには逆効果。`floor` で逃げ道は確保している。

### 2.3 OON (Out-of-Network) スコア減衰

`ranking_scorer.rs:220-239` + `oon_scorer.rs`:

```rust
final_score = match c.in_network {
    Some(false) => after_diversity * effective_oon,  // OON は減衰
    _ => after_diversity,
}
```

**なぜ**:
- Out-of-network 候補 (フォローしていない人の投稿) は、ML が「面白い」と判断しても**ユーザー期待値より上位に来ると違和感がある**
- 重み係数で慎重に持ち上げる、というプロダクト判断

**面白いディテール — 新規ユーザーへの特別扱い** (`ranking_scorer.rs:227-238`):

```rust
let is_eligible_new_user = duration_since_creation_opt(user_id)
    .map(|age| age < new_user_age_threshold).unwrap_or(false)
    && query.user_features.followed_user_ids.len() >= NEW_USER_MIN_FOLLOWING;

if is_eligible_new_user {
    NEW_USER_OON_WEIGHT_FACTOR    // 新規は OON を大きく持ち上げる
} else {
    oon_weight_factor
}
```

**なぜ**: 新規ユーザーはフォローが少ない → in-network だけだとフィードが空 → cold start を OON で埋める。古典的だが必須の設計。

### 2.4 Topic surface への切替

`topic_ids` が指定されたリクエストでは OON 重み係数を別パラメータ (`TopicOonWeightFactor`) に切り替える (`ranking_scorer.rs:221-223`)。同じパイプラインで「For You」と「Topic feed」が並列に動く。

---

## 3. パイプライン・アーキテクチャの設計

### 3.1 Trait-based composable pipeline

**何**: `candidate-pipeline/` に 7つの trait (Source / QueryHydrator / Hydrator / Filter / Scorer / Selector / SideEffect) を定義し、すべてを `CandidatePipeline::execute()` が制御する。

**なぜ**:
- 新しい候補ソースの追加 = trait 実装1つ + 配列に追加。配線変更ゼロ。
- 観測 (`#[tracing::instrument]`, latency 記録, count 記録) が trait 経由で**全 component に自動適用**
- 並列化 (`join_all`) も trait 単位で実装、各 component は async fn を書くだけ

**コスト**:
- 抽象が中央に来るので「具象を読むには trait → impl の往復」が必要 → 読み手の認知負荷
- 8段の固定ステージから外れたフローは書きづらい (例: filter の途中でループしたい等)

**参照**: `candidate-pipeline/candidate_pipeline.rs:67-137`

### 3.2 Two-pipeline separation (内側/外側の責務分離)

**何**: `PhoenixCandidatePipeline` (純粋なレコメンド) と `ForYouCandidatePipeline` (フィード組み立て) を分離。外側の Source の1つが内側を呼ぶ。

詳細は [`01_orchestration_flow.md`](./01_orchestration_flow.md) §1 を参照。

**設計判断のコア**:
- 同じ `CandidatePipeline` trait を**異なる候補型 (`PostCandidate` vs `FeedItem`) で2回**使う
- 内側を独立 gRPC サービス (`ScoredPostsService`) として公開 → 他サーフェスから再利用

### 3.3 Cheap-then-expensive ordering

パイプラインのステージ順序に「**安いチェックを先、高コスト処理を後**」が一貫して現れる:

```
filters (14個・順次・安いものから):
  DropDuplicates           ← O(N) で hash 比較だけ
  CoreDataHydration        ← hydration 失敗の即除外
  Age                      ← 単純な時刻比較
  SelfTweet                ← user_id 比較だけ
  ...
  Video / TopicIds / NewUserTopic    ← 後段でより文脈依存

scorers (順次):
  PhoenixScorer            ← gRPC 推論 (重い、ここで一気にスコア化)
  RankingScorer            ← Σ wᵢ Pᵢ + diversity + OON (CPU のみ)
  VMRanker                 ← 別のリランカで微調整

post-selection:
  TopKScoreSelector        ← まず上位 K に絞ってから
  VFCandidateHydrator      ← VF gRPC 呼び出し (重い)
  MutualFollowJaccard      ← Strato 経由 (重い)
  AdsBrandSafetyHydrator   ← Safety label 取得
```

**なぜ**:
- 上位 K (たいてい数百〜千) に絞ってから重い処理 → **N 倍の節約**になる
- 旧 Twitter the-algorithm でも同じパターン (heavy ranker は cheap ranker の後)

### 3.4 並列 / 順次の境界

`CandidatePipeline::execute()` のステージ別実行方式 (`candidate_pipeline.rs:202-428`):

| ステージ | 実行 | 理由 |
|---|---|---|
| query_hydrators | 並列 (`join_all`) | 各 hydrator は独立な外部呼び出し |
| sources | 並列 | 各 source は独立コーパス |
| hydrators | 並列 | 各 hydrator は独立な enrichment |
| filters | **順次** | 前 filter の結果に次が依存 |
| scorers | **順次** | scorer 同士でスコアを書き換える (RankingScorer は PhoenixScorer の結果を使う) |
| post_selection_hydrators | 並列 | top-K に対する独立 enrichment |
| post_selection_filters | 順次 | 同上 |
| side_effects | 並列 + spawn | fire-and-forget |

**境界の判断基準**: 「次のステップが前のステップの出力に依存するか」だけ。これだけで決まる。

---

## 4. 運用・本番運用上の設計

### 4.1 Feature switch gating (`enable()`)

すべての component に `enable(&query) -> bool` がある。Feature switch でユーザー/国/トラフィック比率で条件評価される。

**運用パターン**:
1. 新機能を `enable() -> false` (デフォルト off) で merge
2. FS で社内ユーザーだけ on (内部 dogfooding)
3. 1% → 10% → 50% → 100% でロールアウト
4. 問題があれば FS で即 off (rollback ではなく config 変更だけ)

**観測**: 無効化された component の名前は span に `disabled=foo,bar` として記録 (`candidate_pipeline.rs:451`) → どこで何が動いていないか即座にわかる。

**参照**: `candidate_pipeline.rs:435-454` の `record_enabled_components`

### 4.2 Fire-and-forget side effects

`candidate_pipeline.rs:419-428`:

```rust
fn run_side_effects(&self, input: Arc<SideEffectInput<Q, C>>) {
    let side_effects = self.side_effects();
    tokio::spawn(async move {
        let futures = side_effects.iter()
            .filter(|se| se.enable(input.query.clone()))
            .map(|se| se.run(input.clone()));
        let _ = join_all(futures).await;
    });
}
```

**何**: side effects (Kafka publish, Redis 書き込み, 統計記録) は `tokio::spawn` でメインフローと独立に走らせる。

**なぜ**:
- レスポンスを返す前に side effect 完了を待つと**ユーザー体験のレイテンシが伸びる**
- side effect が失敗してもユーザーに 500 を返さない
- 実行成功は best-effort で十分 (Kafka なら at-least-once でリトライ機構が別にある)

**コスト**: 「ログが書かれていない」「Kafka に流れていない」が発生し得る → デバッグ時に観察ポイントが減る。tracing で side effect 内部もカバーすることで補う。

### 4.3 Debug endpoint with FS overrides

`scored_posts_server.rs:236-267` の `get_debug_scored_posts`:

```rust
let debug_query = request.into_inner();
let fs_overrides = debug_query.feature_switch_overrides;  // HashMap<String, String>
// → QueryBuilder.build() で FS を上書きしてパイプラインを実行
```

**何**: 同じパイプラインを通すが、Feature switch を任意に上書きできる + 各ステージの中間結果 (`retrieved_candidates`, `filtered_candidates`, `selected_candidates`) を JSON で返す。

**なぜ**:
- 本番と**完全に同じコードパス**で「もし FS_X を true にしたら何が変わるか」を観察できる
- 「なぜこの post が落ちたか」を build_debug_json (`scored_posts_server.rs:115-132`) で追える
- A/B テストの "before they ship" 検証ツール

### 4.4 Mock-complete testability

`PhoenixCandidatePipeline::mock()` と `ForYouCandidatePipeline::mock()` が**全クライアントを mock で組んだ完全なパイプライン**を返す。

```rust
let phoenix_client = Arc::new(MockPredictClient);
let phoenix_retrieval_client = Arc::new(MockRetrievalClient);
let thunder_client = Arc::new(ThunderClient::mock());
// ... 全部 mock ...
let pipeline = PhoenixCandidatePipeline::build_with_clients(...).await;
```

**なぜ**:
- E2E テストが gRPC 依存なしで書ける
- 新 component を追加するときの統合テストが簡単
- ローカル開発が `gcloud auth` 等の手間なしに回る

**コスト**: mock 実装の保守。本物との挙動乖離があると false negative テストになる。各 trait に mock も用意するため面倒。

### 4.5 全ステージへの自動観測

`#[tracing::instrument]` + `record_enabled_components` + `log_stage_size` がパイプラインの全ステージに付いている。出力例:

```
query_hydrators: total_count=15 enabled_count=14 disabled=ImpressionBloomFilter latency_ms=42
sources:        total_count=6  enabled_count=6  candidate_count=1843 latency_ms=87
hydrators:      total_count=10 enabled_count=10 latency_ms=64 size=1843
filters:        input_count=1843 kept_count=412 removed_count=1431 filter_rate=0.776
                removed_per_filter=[PreviouslySeen=812, Age=433, AuthorSocialgraph=186]
scorers:        total_count=3 enabled_count=3 latency_ms=234 size=412
```

**なぜ**:
- レイテンシ予算が逼迫したら**どのステージが犯人か即わかる**
- フィルタごとの除去数 → 「PreviouslySeen で 8割消える」ような状況を即発見
- Disabled component が一覧で出る → A/B 検証が容易

---

## 5. なぜこの設計に至ったか — 進化の文脈

旧 Twitter the-algorithm (2023年公開) との差分を見ると、設計判断の意図が明確になる:

| 観点 | 旧 (twitter/the-algorithm) | 新 (xai-org/x-algorithm) |
|---|---|---|
| ML モデル | MaskNet (48M params) + SimClusters + TwHIN + RealGraph | Phoenix transformer (Grok-based, 数百M+ params) |
| 特徴量 | 数百の hand-engineered features | エンゲージメント sequence のみ |
| Heavy ranker | MaskNet (CPU/GPU) | transformer (GPU 必須) |
| Candidate generation | RealGraph, SimClusters, TwHIN ベース | Two-tower (Phoenix retrieval) |
| 候補数 | ~1500 → MaskNet → 上位 | sources で集める → Phoenix transformer |
| 言語 | Scala (大部分) | Rust (パイプライン) + Python/JAX (モデル) |
| パイプライン | Twitter 内製 framework | trait-based composable pipeline |

**読み取れること**:
- **ML 賭けの強化**: 特徴量パイプラインの代わりに巨大モデル
- **インフラの簡素化**: 多数の特徴量サービスが消滅、Phoenix と Thunder の2サービスで主要部分が回る
- **Rust 採用**: Scala の JVM レイテンシ問題と複雑な依存ツリーから脱却

---

## 6. 設計上のトレードオフ・サマリ

「何を捨てて何を取ったか」を一表に:

| 取った | 捨てた |
|---|---|
| 巨大 transformer による表現力 | 解釈可能性, GPU 安価さ |
| Candidate isolation によるキャッシュ可能性 | 候補間 interaction のモデル化 |
| Multi-action prediction の柔軟性 | マルチタスク学習の安定性 |
| Two-pipeline 分離による再利用性 | コードの一貫した1経路読み心地 |
| Trait-based 抽象化の拡張性 | 具象コードへの直行性 |
| Fire-and-forget side effects の低レイテンシ | side effect 結果の保証性 |
| Feature switch gating の運用安全性 | コードの "ここで何が動いているか" の明瞭さ |
| Mock-complete のテスト容易性 | mock 保守コスト |
| 全段の tracing コスト | サンプリングしない場合の overhead |

**全体として**: 「**スケールと運用安全性を最優先**、品質は ML モデルの大きさで殴る、開発者体験は trait と mock で補う」という極めて一貫した哲学。

---

## 7. ここから何を学べるか

このコードベースから持ち帰れる設計教訓:

1. **同じ抽象 (trait) を異なる候補型で使い回す** と、新しい合成パターンが書きやすい
2. **ML の制約が逆にシステムを助ける** ことがある (candidate isolation = キャッシュ可能性)
3. **post-selection という段階を作る** だけで、cheap-then-expensive が綺麗に書ける
4. **fire-and-forget side effects** はレイテンシクリティカルなシステムの基本
5. **Feature switch を component-level に持つ** ことで rollback ではなく config で運用できる
6. **同じパイプラインを別 endpoint (debug) から FS override で呼べる** ようにすると、A/B 検証 + デバッグツールが1つで済む
7. **mock 実装をプロダクション実装と並べて trait に書く** とテストの再現性が圧倒的に上がる
