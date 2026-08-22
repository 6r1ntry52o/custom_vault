## custom_vault — Obsidian スターター vault テンプレート

一般の Obsidian ユーザーが clone してそのまま使い始められる、**vault 規約層のスターター**。
フォルダ構成・命名・テンプレート・Bases・タグ/frontmatter・運用ルールを一式で配る。

> **状態: 骨組みのみ（2026-08-19）。** 中身（フォルダ構成・テンプレート・Bases・運用ルール）は
> まだ入っていません。原理の書き起こし（A0）が終わってから実装します。

## 使い方（実装後）

```bash
git clone <repo> my-vault
cd my-vault
# Obsidian で「フォルダを vault として開く」→ my-vault を選ぶ
```

## 同梱していないもの（意図的）

| もの                              | 理由                                                   | 代わりに                                                       |
| ------------------------------- | ---------------------------------------------------- | ---------------------------------------------------------- |
| `.obsidian/plugins/*/data.json` | プラグイン個別の設定には秘密（APIキー・パス）が混入しうる。他人の vault で意味が通る保証もない | `docs/plugins.md` に**設定手順**として記述（実装後）                      |
| `.obsidian/workspace.json`      | 開くだけで変化するノイズの塊。端末ごとにペイン配置も違う                         | `starter/workspace.example.json` を手で `.obsidian/` に置く（実装後） |

同梱するのは `community-plugins.json`（プラグイン**一覧**）・`core-plugins.json`・`appearance.json`・
`hotkeys.json`・`templates.json` のみ。

## ライセンス

MIT（[LICENSE](LICENSE)）。
