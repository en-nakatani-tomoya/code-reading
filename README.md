# code-reading

OSS のコードリーディング・ノート集。

## 構成

```
.
├── external_repo/        # クローン済み外部リポジトリ (gitignored)
└── <repo-name>/          # 対応するリーディングノート
```

外部リポジトリ本体は `.gitignore` で除外しているため、対応するノートディレクトリの手順に従ってローカルでクローンしてください。

## 現在のノート

### x-algorithm/

[xai-org/x-algorithm](https://github.com/xai-org/x-algorithm) — X (旧 Twitter) の "For You" フィードのレコメンドアルゴリズム。

セットアップ:

```bash
git clone https://github.com/xai-org/x-algorithm.git external_repo/x-algorithm
```

エントリポイント: [`x-algorithm-notes/00_overview.md`](./x-algorithm-notes/00_overview.md)

## License

ノート部分は MIT。`external_repo/` 配下にクローンする各 OSS はそれぞれのライセンスに従ってください。
