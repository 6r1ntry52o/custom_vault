---
by: me
created: 2026-08-23
tags:
  - meta
  - setup
---
# 設定値一覧

`meta/` を動かすのに必要な設定を、Obsidian の設定画面の並び順どおりに書く。
**上から順に入れれば動く。** 理由・仕組み・トラブルシュートは [[SETUP]]。

> ★ … vault を同期しても引き継がれない端末ローカル設定。**端末ごとに手で入れる**。

## 0. 事前作業

### Obsidian
- バージョン: v1.9 以上
	- コアプラグイン Bases が入ったバージョン。`.base` が表示されないならまずここ

### フォルダを作る
- `db/`
- `db/aigc/`
- `db/ext/`
- `journal/daily/`

### ファイルを置く
- `.obsidian/snippets/tasks-table.css`
	- **必須。** tasks の 7 列表示と、期日・優先度セルのクリック領域をこれが作る
	- スニペットは git 管理下なので、リポジトリを clone したなら何もしなくてよい
	- `meta/` だけを持ってきた場合は、リポジトリの `.obsidian/snippets/` からこの 1 ファイルをコピーする
- `meta/MEMO.md`
	- 走り書き用の空ノート。作らないなら `HOME.md` の `![[MEMO]]` の行を消す

## 1. Option

### 外観
- アクセントカラー:
	- R: 0
	- G: 0
	- B: 120
- CSS スニペット:
	- `tasks-table`: ON

### エディタ
- 読みやすい行の長さ: OFF
	- Bases の表と tasks のテーブルを幅いっぱいに出すため

### ファイルとリンク
- 新規ノートの作成場所: 指定したフォルダ配下
- 新規ノートの保存先: `db`
- 内部リンクの更新: 自動更新

## 2. コアプラグイン

### Bases
- 有効化: ON

### プロパティ
- 有効化: ON
- 型（「すべてのプロパティ」ペインから設定）:
	- by: テキスト
	- db: テキスト
	- status: テキスト
	- created: 日付
	- due: 日付
	- done: チェックボックス
	- halu: チェックボックス
	- ai: テキスト
	- model: テキスト
	- tags: タグ

### バックリンク
- 有効化: ON

### アウトゴーイングリンク
- 有効化: ON

### デイリーノート
- 有効化: ON
- 新規ファイルの場所: `journal/daily`
- テンプレートファイルの場所: (空欄)
	- テンプレート適用は Templater 側で行う。両方入れると二重適用になる

### テンプレート
- 有効化: ON（未使用）
- テンプレートフォルダの場所: (空欄)
	- `meta/template/` は Templater 構文なのでコア機能では展開できない。二重に設定しない

## 3. コミュニティプラグイン

### Templater
- Template folder location: `meta/template/templater`
- Trigger Templater on new file creation: ON ★
	- OFF だと Ctrl+N で作ったノートが空ファイルになる
- Template matching mode: Folder templates
- Automatic jump to cursor: OFF
- Timeout: 5
- User Scripts folder: (空欄)
- Shell path: (空欄)
- Enable Folder Templates: ON
- Folder Templates:
	- `/`: `meta/template/default.md`
	- `db`: `meta/template/default.md`
	- `journal/daily`: `meta/template/daily.md`
	- `meta/template`: (空欄)
		- 空欄で登録するのが要点。無いとテンプレート自体を作ったときに再帰適用されて壊れる

### Tasks
- バージョン: v8.3.0 で検証
	- 期日・優先度のインライン編集がプラグイン内部実装に依存している（[[SETUP#インライン編集（期日・優先度）]]）
- Task format: Tasks emoji format
- Searches → Enable custom searches: ON ★
	- OFF だと `filter by function` が動かず tasks ブロックがエラーになる
- Set done date on every completed task: ON
- Set cancelled date on every cancelled task: ON
- Set created date on every added task: OFF
- Task Statuses（カスタムステータスを追加）:
	- `/`: In Progress / 次のステータス `x` / 種別 IN_PROGRESS
	- `-`: Cancelled / 次のステータス `(空白)` / 種別 CANCELLED

### Dataview
- Codeblocks → Enable JavaScript Queries: ON ★
	- OFF だと当日の日付リンクが出ず、期日・優先度セルのクリックも無反応になる
- Codeblocks → Enable Inline JavaScript Queries: (任意・未使用)

### Webpage HTML Export
- 任意。`meta/` の動作には無関係

### Claudian
- 任意。`meta/` の動作には無関係

## 4. 確認

1. `meta/HOME.md` を開く → Issues の表が出る
	- NG → Bases（§0 の Obsidian バージョン）
2. 同じノートの Tasks ブロックにエラーが出ない
	- NG → Tasks の Enable custom searches（§3）
3. 冒頭に当日の日付リンク（`📄` / `➕`）が出る
	- NG → Dataview の Enable JavaScript Queries（§3）
4. Tasks の結果が 7 列に整列している
	- NG → `tasks-table.css`（§0・§1 の CSS スニペット）
5. 期日セルの `＋` をクリック → カレンダーが開く／優先度セルをクリック → メニューが開く
	- NG → Dataview の JavaScript Queries（§3）か Tasks のバージョン（§3）
	- `HOME.md` 以外のノートだけ NG → §5 運用
6. Ctrl+N で新規ノート → `db/` に作られ frontmatter が入る
	- 空ファイル → Templater の Trigger on new file creation（§3）
	- 中身が違う → Folder Templates（§3）
7. `journal/daily/` に新規ノート → `## Tasks` / `## Log` / `## Note` が入る
8. `meta/template/` に新規ノート → 何も展開されない（空欄登録が効いている）
9. `DropZone.md` を開く → by_ai / by_ext / by_me の 3 表が出る
10. `db/` のノートに `done: true` → DropZone の表から消える
	- 消えない → `done` の型がチェックボックスになっていない（§2）

## 5. 運用

- Obsidian を起動したら、**まず `meta/HOME.md` を開く**
	- 期日・優先度のインライン編集はセッション 1 回だけ入り、以降に描画される全 tasks ブロックに効く
	- `HOME.md` より先に描画した tasks ブロックは、そのセッション中は効かない
	- 効いていない兆候: 優先度セルが無反応／期日セルがカレンダーでなく編集モーダルを開く
	- 直し方: そのノートを開き直す（再描画）か、`HOME.md` を開いてから戻る
