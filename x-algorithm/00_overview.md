# X For You Feed Algorithm — 全体像

リポジトリ: [xai-org/x-algorithm](https://github.com/xai-org/x-algorithm)
公開: 2026年1月 / 最新アップデート: 2026年5月15日
参考記事: [OpenTweet — X Open-Sourced Its Algorithm on GitHub](https://opentweet.io/blog/x-algorithm-open-source-github-2026)

---

## 0. 30秒で掴む

**For You フィードがやっていること = 「あなたが反応しそうな投稿を集めて、並べる」** これだけ。
そのために 2つのソースから候補を集め、1つのMLモデルで全部スコアリングして、上から並べる。

```mermaid
flowchart LR
    subgraph IN["① 候補を集める (Retrieve)"]
      direction TB
      A["フォロー中の投稿<br/>(in-network)"]
      B["全世界の投稿から<br/>ML が掘り出す<br/>(out-of-network)"]
    end

    IN --> ML["② ランク付け (Rank)<br/><br/>Grok-based transformer が<br/>「いいね/返信/リポスト/通報…」<br/>19種類の確率を予測<br/>→ 重み付き和で1スコアに"]

    ML --> OUT["③ 整える (Serve)<br/><br/>フィルタ → 上位K件 → 多様性調整<br/>→ あなたのフィードへ"]

    style IN fill:#e8f4ff,stroke:#36a
    style ML fill:#fff4e0,stroke:#c80
    style OUT fill:#e8ffe8,stroke:#393
```

**この3ステップを実現するための役者:**

| 段階 | 主な実装 | 言語 |
|---|---|---|
| ① 候補集め | `thunder/` (in-network), `phoenix/` retrieval (out-of-network) | Rust / Python |
| ② ランク付け | `phoenix/` ranking (Grok transformer) | Python |
| ③ 整える・オーケストレーション | `home-mixer/` | Rust |
| 横串の基盤 | `candidate-pipeline/` (trait 群), `grox/` (コンテンツ理解) | Rust / Python |

**設計の一行サマリ:** 手作業の特徴量を全部捨てて、ユーザーの行動履歴を transformer に食わせ、19種類のアクション確率を出させる。それを重み付き和で並べるだけ。

---

## 1. ハイレベル・データフロー

ユーザーが「For You」を開いてから、ランク済みフィードが返るまでの流れ。

```mermaid
flowchart TB
    U([User: For You request]) --> HM

    subgraph HM["home-mixer (Rust, orchestration)"]
      QH["Query Hydration<br/>followed topics / starter packs /<br/>impression bloom / mutual follow"]
      CS["Candidate Sources"]
      HY["Candidate Hydration<br/>(engagement counts, brand safety,<br/>lang, media, quote, mutual follow…)"]
      F1["Pre-Scoring Filters"]
      SC["Scorers"]
      SEL["Selector<br/>(top-K by final score)"]
      F2["Post-Selection Filters<br/>(VF / dedup conversation)"]
      SE["Side Effects<br/>(cache / served history)"]
      QH --> CS --> HY --> F1 --> SC --> SEL --> F2 --> SE
    end

    subgraph SRC["Candidate Sources"]
      direction LR
      TH["Thunder<br/>(in-network)"]
      PHR["Phoenix Retrieval<br/>(out-of-network, 2-tower)"]
      MOE["Phoenix MoE / Topics"]
      ADS["Ads source"]
      WTF["Who To Follow"]
      PRM["Promptable Feeds"]
    end

    CS --- SRC

    subgraph SCORERS["Scoring stack"]
      direction TB
      PHX["Phoenix Scorer<br/>(Grok-based transformer,<br/>19 engagement probs)"]
      WS["Weighted Scorer<br/>Σ wᵢ · P(actionᵢ)"]
      ADV["Author Diversity Scorer"]
      OON["OON Scorer"]
      PHX --> WS --> ADV --> OON
    end

    SC --- SCORERS

    SE --> R([Ranked Feed Response])

    classDef ext fill:#eef,stroke:#447
    class TH,PHR,MOE,ADS,WTF,PRM ext
```

---

## 2. サブシステム俯瞰

各サブシステムの「役割 / 言語 / I/O」を一目で。

```mermaid
flowchart LR
    subgraph IN["Inputs"]
      KAFKA[(Kafka<br/>post create/delete)]
      USER([User context])
    end

    subgraph THUNDER["thunder/ (Rust)"]
      TKAFKA["kafka_utils / deserializer"]
      TSTORE["posts/<br/>in-memory per-user store<br/>original / replies-reposts / video"]
      TSVC["thunder_service.rs (gRPC)"]
      TKAFKA --> TSTORE --> TSVC
    end
    KAFKA --> TKAFKA

    subgraph PIPE["candidate-pipeline/ (Rust framework)"]
      direction TB
      TR["traits:<br/>Source / Hydrator / Filter /<br/>Scorer / Selector / SideEffect /<br/>QueryHydrator"]
    end

    subgraph PHOENIX["phoenix/ (Python, JAX/Grok-1 port)"]
      direction TB
      RET["recsys_retrieval_model.py<br/>(two-tower)"]
      RANK["recsys_model.py<br/>(transformer + candidate isolation)"]
      RUN["run_pipeline.py<br/>retrieval → ranking"]
      ART["artifacts/<br/>(~3 GB Git LFS, mini model)"]
      RET --> RUN
      RANK --> RUN
      ART --> RUN
    end

    subgraph GROX["grox/ (Python, content understanding)"]
      direction TB
      GC["classifiers/<br/>(spam, PTOS, category)"]
      GE["embedder/"]
      GG["generators / summarizer"]
      GT["tasks / plans / schedules"]
      GENG["engine.py + dispatcher.py"]
      GC & GE & GG & GT --> GENG
    end

    subgraph HMX["home-mixer/ (Rust, orchestration)"]
      direction TB
      HSRV["for_you_server.rs<br/>scored_posts_server.rs (gRPC)"]
      HCP["candidate_pipeline/<br/>(uses pipeline traits)"]
      HQ["query_hydrators/"]
      HS["sources/"]
      HH["candidate_hydrators/"]
      HF["filters/"]
      HSC["scorers/"]
      HSE["selectors/"]
      HSDE["side_effects/"]
      HADS["ads/ (injection + brand safety)"]
      HSRV --> HCP
      HCP --> HQ & HS & HH & HF & HSC & HSE & HSDE
      HS --- HADS
    end

    USER --> HQ
    TSVC --> HS
    PHOENIX --> HS
    PHOENIX --> HSC
    GROX --> HH
    PIPE -. depended on .-> HCP

    HSRV --> OUT([Ranked feed])

    classDef rust fill:#fde,stroke:#a33
    classDef py fill:#def,stroke:#36a
    class THUNDER,PIPE,HMX rust
    class PHOENIX,GROX py
```

---

## 3. リポジトリ・ディレクトリ対応

| ディレクトリ | 言語 | 役割 | 主要ファイル |
|---|---|---|---|
| `home-mixer/` | Rust | フィード組み立てのオーケストレーション層・gRPC エンドポイント | `for_you_server.rs`, `scored_posts_server.rs`, `candidate_pipeline/` |
| `home-mixer/ads/` | Rust | 広告挿入・ポジショニング・ブランドセーフティ | — |
| `thunder/` | Rust | In-network 投稿のインメモリストア + Kafka 取り込み | `thunder_service.rs`, `posts/`, `kafka/` |
| `phoenix/` | Python | Two-tower retrieval + Grok-based ランキング | `recsys_retrieval_model.py`, `recsys_model.py`, `run_pipeline.py`, `grok.py` |
| `phoenix/artifacts/` | (LFS) | Pre-trained mini Phoenix (256-dim, 4 heads, 2 layers, 約3GB) | — |
| `grox/` | Python | コンテンツ理解 (分類・埋め込み・スパム/PTOS) | `engine.py`, `dispatcher.py`, `classifiers/`, `embedder/` |
| `candidate-pipeline/` | Rust | パイプライン基盤 (trait 集) | `source.rs`, `hydrator.rs`, `filter.rs`, `scorer.rs`, `selector.rs`, `side_effect.rs` |

---

## 4. ランキングの核 — Phoenix のスコアリング

Phoenix の transformer は1投稿あたり **19のエンゲージメント確率** を出し、重み付き和で最終スコアにする。

```mermaid
flowchart LR
    UC["User context<br/>(engagement sequence)"] --> TX
    CAND["Candidate post"] --> TX["Phoenix Transformer<br/>(candidate isolation:<br/>candidates cannot attend to each other)"]
    TX --> P1[P_favorite]
    TX --> P2[P_reply]
    TX --> P3[P_repost]
    TX --> P4[P_quote]
    TX --> P5[P_click]
    TX --> P6[P_profile_click]
    TX --> P7[P_video_view]
    TX --> P8[P_photo_expand]
    TX --> P9[P_share]
    TX --> P10[P_dwell]
    TX --> P11[P_follow_author]
    TX --> N1[P_not_interested]
    TX --> N2[P_block_author]
    TX --> N3[P_mute_author]
    TX --> N4[P_report]
    P1 & P2 & P3 & P4 & P5 & P6 & P7 & P8 & P9 & P10 & P11 --> WS["Σ wᵢ · Pᵢ<br/>(positive weights)"]
    N1 & N2 & N3 & N4 --> WS2["Σ wⱼ · Pⱼ<br/>(negative weights)"]
    WS --> FS([Final Score])
    WS2 --> FS
```

**Candidate isolation** が重要設計: バッチ内の他候補に attend させないことで、スコアが「同じバッチに何が来たか」に依存せず、キャッシュ可能・一貫性ありになる。

---

## 5. 設計上のキーポイント

1. **No hand-engineered features** — 関連性に関する手作業の特徴量は全廃。transformer がエンゲージメント履歴から学習。
2. **Candidate isolation** — attention で候補同士を見えなくする。スコアの再現性とキャッシュ性。
3. **Hash-based embeddings** — retrieval / ranking 両方で複数ハッシュ関数による埋め込み参照。
4. **Multi-action prediction** — 単一の「関連性スコア」ではなく19アクションの確率を出す。ネガティブアクションは負の重み。
5. **Composable pipeline** — `candidate-pipeline` crate が trait を提供し、source/hydrator/filter/scorer の追加を容易に。並列実行と graceful error handling。
6. **Time decay** — 約6時間ごとに可視性が約50%減衰 (OpenTweet 記事より)。
7. **TweepCred** — 0〜100のレピュテーション、毎日再計算 (OpenTweet 記事より)。
8. **End-to-end runnable** — 2026-05-15 リリースで `phoenix/run_pipeline.py` により retrieval → ranking が単一エントリで動く。pre-trained mini モデル同梱。

---

## 6. 次に読むなら

- **オーケストレーションの流れ全体**を追う → [`01_orchestration_flow.md`](./01_orchestration_flow.md) (`home-mixer/` の二段パイプラインを関数呼び出しレベルで追う)
- **設計上のキーポイント**を体系的に → [`02_design_keypoints.md`](./02_design_keypoints.md) (なぜそうなっているか / 何を捨てて何を取ったか)
- **考察 (アーキテクチャから読み取れる暗黙の仮説)** → [`considerations/`](./considerations/)
  - [`01_user_representation_hypothesis.md`](./considerations/01_user_representation_hypothesis.md) — `user_hashes` は何を encode しているか、なぜプロフィール特徴量がモデル入力にないのか
- **パイプラインの抽象** → `candidate-pipeline/lib.rs` と各 trait ファイル
- **ML 本体** → `phoenix/recsys_model.py`（ランキング）, `phoenix/recsys_retrieval_model.py`（2-tower）
- **動かす** → `phoenix/run_pipeline.py` + `phoenix/artifacts/` の mini モデル
- **In-network 経路** → `thunder/thunder_service.rs` → `thunder/posts/`
