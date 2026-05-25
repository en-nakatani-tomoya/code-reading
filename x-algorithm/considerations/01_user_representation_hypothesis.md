# 考察: user_hashes は何を encode しているのか — そして暗黙の設計仮説

## 出発点となった疑問

Phoenix モデル (retrieval / ranking 両方) の入力を `phoenix/recsys_model.py:126-144` の `RecsysBatch` で確認すると、ユーザー側の入力は次だけしかない:

- `user_hashes` — user_id をハッシュした2つの整数
- `history_*` — 直近のエンゲージメント履歴 (post / author / action / dwell / product surface)
- (オプション) `user_ip_hashes`

**bio・自己紹介・性別・年齢・興味タグなどの "プロフィール特徴量" は、Phoenix モデルには直接入力されていない**。

ここから自然と次の疑問が立ち上がる:

> 「user_id ハッシュ自体は意味的な情報を持たないはず。だとすれば、このアーキテクチャは『**プロフィール情報はユーザーの将来エンゲージメントとほぼ無関係**』という強い仮説を暗黙に置いているのではないか?」

本ドキュメントはこの問いを、コードを根拠に体系的に整理する。結論を先に書くと: **その読みは正確で、アーキテクチャはこの仮説を意図的に embedded している**。

---

## 1. メカニカルに: `user_hashes` の中身

### 1.1 構造

`phoenix/recsys_model.py:94-99` の `HashConfig`:

```python
@dataclass
class HashConfig:
    num_user_hashes: int = 2          # ユーザー1人につき 2 つの hash
    num_item_hashes: int = 2
    num_author_hashes: int = 2
    num_ip_hashes: int = 0            # IP は default 無効
```

`phoenix/runners.py:454` の参考デフォルト値:

```python
num_user_embeddings: int = 1_000_000  # 埋め込みテーブルの行数 V_user
```

つまり `user_hashes` の正体は:

```
user_id ─ hash1 ─► 整数 ∈ [1, V_user]
        └ hash2 ─► 整数 ∈ [1, V_user]
```

### 1.2 利用

`block_user_reduce` (`recsys_model.py:147-197`) で:

```python
user_embedding = user_embeddings.reshape((B, 1, num_user_hashes * D))
# proj_mat_1 [num_user_hashes * D, D] で線形射影
user_embedding = jnp.dot(user_embedding, proj_mat_1)   # → [B, 1, D]
```

2つのハッシュで埋め込みテーブルから2行引き、concat → 線形射影で 1 個の `[D]` ベクトルに圧縮される。

### 1.3 重要事実

**`user_hashes` 自体には意味的な情報は 1 ビットも含まれていない。** murmur3 のような決定論的関数の出力で、user_id (`987654321`) を別の整数に変換しているだけ。bio や性別と相関するわけでもなく、user_id 同士の "近さ" を表すわけでもない。

---

## 2. 情報を運んでいるのは「学習される埋め込みテーブル」

### 2.1 何が学習されるか

意味は**埋め込みテーブルの行が学習を通じて獲得する**:

```
hash(user_id) = 42  ─►  embedding_table[42] = [0.3, -0.1, 0.7, ...] ← 勾配で更新
```

訓練時の損失関数は「過去履歴を見せた状態で、候補投稿への 19 アクション確率を予測」。この loss を下げるため、`embedding_table[42]` の値は **「この行に対応する全ユーザー (ハッシュ衝突含む) の集約された行動傾向」を encode するように勾配が流れる**。

### 2.2 学習されうるシグナル

ハッシュテーブルが捉えうるもの:

- **個人のベースライン傾向**: 平均 dwell time, click 率, night-owl 度合い、ネガティブ反応頻度など、直近 128 件の history から見えない長期傾向
- **Collaborative filtering signal**: 同じハッシュバケットに落ちたユーザー群 (≒ 「似た行動を取るユーザーの暗黙クラスタ」) の共通傾向
- **コールドスタート時のデフォルト**: 履歴がない / 短いユーザーに対しても「ハッシュ衝突先の他ユーザーの平均」を返せる

### 2.3 ハッシュ衝突の意味

`num_user_hashes=2` と固定サイズ `V_user` で、必然的に複数のユーザーが同じ行を共有する。これは**バグではなく設計**で、以下の利点を生む:

- **暗黙の正則化**: 衝突するユーザー間で表現を共有 → 過学習を抑制
- **メモリ効率**: `O(V_user)` で済む (`O(num_users)` ≒ 数億行なら現実的でない)
- **新規ユーザー対応**: 未見の user_id でも `hash(user_id)` は何かしらの行を引ける

→ つまり **`user_hashes` が運ぶ情報量 ≒ 「エンゲージメント履歴から推測されうる "ユーザーらしさ" の残差」**。プロフィール文や bio は一切含まれない。

---

## 3. 暗黙の設計仮説

ここからアーキテクチャに織り込まれた仮説を抽出すると:

> **「ユーザーの将来エンゲージメントを予測するのに必要な情報は、`user_id` (collaborative filtering signal) + 直近 128 件の行動履歴で十分。プロフィール (bio / 性別 / 年齢 / 興味タグ等) は、行動履歴から既に間接的に推測できる以上の情報を提供しない。」**

これは**反証可能 (falsifiable) な強い仮説**である。

### 3.1 なぜこれが (おそらく) 正しい賭けか

| 理由 | 説明 |
|---|---|
| **行動 > 自己申告** | 「basketball が好き」と bio に書くより、実際に basketball 投稿に 500 回 like するほうが信号が強く、誠実 |
| **プロフィールはスパース・古い・偽れる** | 多くのユーザーは bio が空。書いてあっても何年も前の自分の興味で固まっている。スパム/工作で意図的に偽装される |
| **業界の収束** | YouTube, TikTok, Netflix 全てが「demographic-based → behavior-based」に移行済み。Twitter (旧) の MaskNet も後期は behavioral feature が支配的 |
| **行動履歴は本質的に高次元** | 128 投稿に対するアクション/dwell の組み合わせは膨大な状態空間 → bio に書かれた数語より遥かに多くを語る |

### 3.2 この仮説が破綻する場面

設計側もこの限界を認識している痕跡が、コード内に散見される:

- **コールドスタート問題**:
  履歴ゼロのユーザーで `user_hashes` は実質「衝突する他ユーザーの平均」しか引けない。
  → コードは新規ユーザーに OON 重みを大きく上げて補う (`home-mixer/scorers/ranking_scorer.rs:227-238` の `NEW_USER_OON_WEIGHT_FACTOR`)
- **興味の急変**:
  出産・転職・趣味変更があっても、bio 更新では反映できず行動履歴に出るまで時差あり。
  → 設計上は時差を許容している
- **言語/ロケール**:
  履歴から推測可能だが、新規ユーザーには弱い。
  → `IpQueryHydrator` で IP→地域を取得しているが、これはモデル入力ではなくフィルタ/FS 評価に使われる

---

## 4. 「意図的な選択」である証拠

これがサボタージュや簡略化ではなく**意図された判断**であることが、コードに残された痕跡からわかる:

### 証拠 1: 拡張可能な構造を持ちつつ default 無効

```python
# phoenix/recsys_model.py:360
use_ip_address: bool = False        # IP を入れる経路は存在するが default 無効

# phoenix/recsys_model.py:100
num_ip_hashes: int = 0              # 構造的に追加可能だがゼロ
```

→ 「やろうと思えばできるが、効かないから入れない」が読み取れる。実験して採用しなかったか、cost/benefit が見合わなかった、と推察できる。

### 証拠 2: プロフィール系データを取得しているがモデルには流していない

`home-mixer/candidate_pipeline/phoenix_candidate_pipeline.rs:185-232` の query_hydrators:

```rust
Box::new(UserDemographicsQueryHydrator { ... }),          // 年齢層を取得
Box::new(UserInferredGenderQueryHydrator::new(...)),       // 推定性別を取得
Box::new(FollowedGrokTopicsQueryHydrator::new(...)),       // フォロー中トピック
Box::new(FollowedStarterPacksQueryHydrator::new(...)),
Box::new(InferredGrokTopicsQueryHydrator { ... }),         // 推定トピック
```

これらは Phoenix モデル入力 (`RecsysBatch`) には**入っていない**。主な用途は:

- Feature Switch の recipient 評価 (国/年齢層別に FS を切る) (`server.rs:144-160`)
- フィルタの判定材料 (年齢制限コンテンツの除外など)
- `ScoringSequenceQueryHydrator` の中で履歴生成時のメタデータとして使われる可能性

→ 「データは取れるのに、敢えてモデルに流していない」明確な選択。

### 証拠 3: 明示的ドクトリン

リポジトリ README に「No hand-engineered features」が**Key Design Decision として明示**されている (`x-algorithm/README.md`, `phoenix/README.md`)。偶然や時間不足ではなく方針。

### 推察

これらを総合すると、xAI 内部では「プロフィール特徴量を試した結果、エンゲージメント履歴を超える lift が出なかった」可能性が高い。あるいは「行動履歴に既に encode されていて、追加しても多重カウントになる」と判断したと考えられる。

---

## 5. 一段抽象化すると

このコードベースが encode している**メタ的仮説**は階層的に整理できる:

| レベル | 仮説 |
|---|---|
| **L1 (data)** | 直近 128 件の (post, author, action, dwell, surface) があれば、ユーザーの当面の興味は捉えられる |
| **L2 (model)** | `user_hashes` の埋め込みが、履歴 128 件で捉えきれない長期傾向を吸収する |
| **L3 (architecture)** | 静的属性 (bio / 性別 / 年齢) は L1+L2 が捕捉する以上の情報を持たない |
| **L4 (philosophy)** | 「ユーザー自身を観測する」より「ユーザーの行動を観測する」ほうが、推薦に必要な情報を正確に取れる |

ユーザーが「自分は何が好きか」を申告する情報より、「実際に何に時間を使ったか」を観測する情報の方を信用する、というのが哲学レベルでの選択。

---

## 6. 反証可能性 — 仮説が間違っていたら検出できる

この設計の良いところは、仮説が**測定可能で反証可能**な点。例えば:

- **仮説検証 A**: 「bio に明示的興味タグがあるユーザー」と「ないユーザー」で予測誤差を比較。前者が有意に高ければ、プロフィールが追加情報を持っている証拠。
- **仮説検証 B**: モデル入力に explicit user features を足したアブレーション実験。lift があれば現行設計は局所最適。
- **仮説検証 C**: コールドスタートユーザーのみで、プロフィール特徴量入りモデル vs なしモデルを比較。前者が勝てば「履歴がない場合はプロフィールが効く」という限定的な反証。

Phoenix が "continuously trained" と README に記されている (`x-algorithm/README.md`) ので、内部ではおそらくこういう実験が継続して回されている。

---

## 7. 持ち帰り

1. **`user_hashes` 自体は 0-information identifier** — 意味は埋め込みテーブルが学習する
2. **学習されるシグナル ≒ 行動履歴の "残差"** — 履歴で捉えきれない長期傾向の暗黙吸収
3. **プロフィール特徴量がモデル入力にない = 設計仮説の表明** — 「行動 >> 自己申告」を信じている
4. **コードベースは反証可能な実験プラットフォームでもある** — 仮説が外れたら検出できる構造

このコードベースから他システムに転用できる教訓は: **「アーキテクチャ選択は暗黙の仮説の宣言である」** ということ。何を入力にしないかは、何が大事でないかという主張に等しい。
