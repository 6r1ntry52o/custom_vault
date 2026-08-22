---
by: me
created: 2026-08-22
tags:
  - meta
  - setup
---
# meta/ を機能させるためのプラグイン・設定

`meta/` 配下（`HOME.md` / `base/*.base` / `template/*.md`）は、**Obsidian 本体の設定とプラグインが揃って初めて動く**。
clone 直後は `.obsidian/plugins/` と `.obsidian/workspace.json` と `.obsidian/types.json` と `.obsidian/app.json` が
git 管理外（[[.gitignore]] 参照）なので、**プラグイン本体のインストールと個別設定は手作業**になる。

その手順をここにまとめる。

---

## 0. 前提

| 項目 | 要件 |
| --- | --- |
| Obsidian | **v1.9 以上**（コアプラグイン **Bases** が入ったバージョン。`*.base` が動かない場合はまずここを疑う） |
| vault ルート | このリポジトリを clone したフォルダを「フォルダを vault として開く」で開く |
| 同期 | `.obsidian/plugins/` は同梱していない。端末ごとにインストールが要る |

---

## 1. コアプラグイン（設定 → コアプラグイン）

`.obsidian/core-plugins.json` は git 管理下なので、clone すれば下記は自動で入る。
手で確認するなら必須は次の 4 つ。

| コアプラグイン | 何に必要か |
| --- | --- |
| **Bases** | `meta/base/*.base` の表示すべて。`![[tickets.base]]` などの埋め込みも含む |
| **プロパティ（Properties）** | `by` / `status` / `db` / `done` / `halu` などの frontmatter 編集 UI |
| **バックリンク / アウトゴーイングリンク** | `HOME.md` → `DropZone` 等の導線 |
| **デイリーノート** | `journal/daily/` の運用（下の §4 で設定が要る） |

`templates`（コアのテンプレート）は有効だが **未使用**。`meta/template/` は Templater 構文（`<% %>`）で書かれているため、
コアのテンプレート機能では展開できない。テンプレートフォルダを二重に設定しないこと。

---

## 2. コミュニティプラグイン

`.obsidian/community-plugins.json` は「一覧」だけを持つ。**本体は各端末でインストールする**。

| プラグイン                                                                | 必須度    | 用途                                                                                                   |
| -------------------------------------------------------------------- | ------ | ---------------------------------------------------------------------------------------------------- |
| [Templater](obsidian://show-plugin?id=templater-obsidian)            | **必須** | `meta/template/*.md` の `<% tp.* %>` 展開・フォルダ別の自動テンプレート適用                                              |
| [Tasks](obsidian://show-plugin?id=obsidian-tasks-plugin)             | **必須** | `HOME.md` の ` ```tasks ` ブロック（`filter by function` 込み）                                               |
| [Dataview](obsidian://show-plugin?id=dataview)                       | **必須** | `HOME.md` 冒頭の ` ```dataviewjs `（当日のデイリーノートへのリンク）と `member.md` の ` ```dataview `。表形式の一覧は Bases が担うので、Dataview はこの 2 用途のみ |
| [Webpage HTML Export](obsidian://show-plugin?id=webpage-html-export) | 任意     | ノートの HTML 書き出し。`meta/` の動作には無関係                                                                      |
| Claudian (`realclaudian`)                                            | 任意     | vault 内で Claude を動かす。`meta/` の動作には無関係。`.claudian/` は git 管理外                                         |

### 2.1 Templater の設定

設定 → Templater：

| 設定項目 | 値 | 備考 |
| --- | --- | --- |
| Template folder location | `meta/template/templater` | 手動挿入（insert template modal）の対象。Folder Templates の参照先とは別物 |
| Trigger Templater on new file creation | **ON** | **端末ごとに ON が要る**（下記） |
| Template matching mode | Folder templates | 上を ON にすると現れる |
| Automatic jump to cursor | OFF | |
| Timeout | 5 | |

> **`Trigger Templater on new file creation` は data.json に入らない。**
> このトグルだけは端末ローカル設定（Obsidian の localStorage `templater-local-settings`）に保存される。
> つまり vault を clone しても・`data.json` を手で書いても引き継がれず、**端末ごとに UI で ON にする必要がある**。
> OFF のままだと Ctrl+N で作った新規ノートは frontmatter が入らず **空ファイル**になる（症状はこれ）。

**Folder Templates**（`Enable Folder Templates` を ON にして追加）:

| フォルダ | テンプレート |
| --- | --- |
| `/` | `meta/template/default.md` |
| `db` | `meta/template/default.md` |
| `journal/daily` | `meta/template/daily.md` |
| `meta/template` | *(空欄)* |

`db` を `default.md` にしているのは、**Ctrl+N（新規ノート＝作成先 `db/`）を `default.md` ベースにするため**。

**Folder Templates と「手動挿入できるテンプレート」は別管理**になっている点に注意。

| | 置き場所 | 中身 |
| --- | --- | --- |
| Folder Templates が参照 | `meta/template/*.md` | `default.md` / `daily.md` / `db.md` |
| 手動挿入（`Templater: Open insert template modal`）の候補 | `meta/template/templater/*.md` | `ai.md` / `me.md` / `member.md` |

`Template folder location` は `meta/template/templater` なので、**挿入モーダルに出るのは `templater/` 配下の 3 つだけ**。
`meta/template/db.md` は現状 `default.md` と同内容で、folder template からも挿入候補からも外れている（予備）。

> `meta/template` を **空欄で登録するのが要点**。これが無いと、テンプレート自体を新規作成したときに
> テンプレートが再帰的に適用されて `<% %>` が展開済みで壊れる。

使っている Templater 構文は `tp.file.creation_date("YYYY-MM-DD")` と `tp.file.title` の 2 つだけ。
`daily.md` は frontmatter も見出しも持たず、`## Tasks` / `## Log` / `## Note` の 3 見出しのみ（＝ Templater 構文を含まない）。
日付見出しは作らず、ファイル名（`YYYY-MM-DD`）をそのまま日付として扱う。
ユーザースクリプト・shell コマンドは使っていないので、`User Scripts folder` と `Shell path` は空のままでよい。

> **`data.json` を手で編集する場合の順番**: Obsidian 起動中に
> `.obsidian/plugins/templater-obsidian/data.json` を書き換えたら、**先に Obsidian（または Templater）を再読み込み**する。
> 先に設定 UI のトグルを触ると、メモリ上の旧設定で `data.json` が上書きされて編集が消える。

### 2.2 Dataview の設定

設定 → Dataview：

| 設定項目 | 値 | 理由 |
| --- | --- | --- |
| Codeblocks → **Enable JavaScript Queries** | **ON** | `HOME.md` の ` ```dataviewjs ` ブロックがこれ無しでは描画されない。**端末ごとに ON が要る**（`data.json` は git 管理外） |
| Codeblocks → Enable Inline JavaScript Queries | 任意 | `meta/` では未使用 |

`HOME.md` **冒頭**の ` ```dataviewjs ` ブロックが、`moment()` で当日を求めて `journal/daily/YYYY-MM-DD.md` への
リンクを描画する（見出しは付けていない）。ノートが未作成なら `➕`、作成済みなら `📄` が付く。
`moment().locale("en")` にしているのは、**曜日を Obsidian の表示言語に依存させず英語（Mon/Tue…）で固定するため**。
**未作成のリンクをクリックするとその場で作成され**、Templater の Folder Templates（`journal/daily` → `daily.md`）が
当たって `## Tasks` / `## Log` / `## Note` が入る
（＝ §2.1 の `Trigger Templater on new file creation` が ON である必要がある）。

`HOME.md` はこの下で `![[MEMO]]`（`meta/MEMO.md`）を埋め込んでいる。走り書き用の空ノートなので、
不要なら埋め込み行ごと消してよい。

### 2.3 Tasks の設定

設定 → Tasks：

| 設定項目 | 値 | 理由 |
| --- | --- | --- |
| Searches → **Enable custom searches** | **ON** | `HOME.md` の `filter by function ...` がこれ無しでは動かない。**端末ごとに ON が要る**（設定が同期されない） |
| Task format | Tasks emoji format | `📅 2026-08-22` などの絵文字表記 |
| Set done date on every completed task | ON | |
| Set cancelled date on every cancelled task | ON | |
| Set created date on every added task | OFF | |

**カスタムステータス**（Settings → Tasks → Task Statuses に追加）:

| 記号 | 名前 | 次のステータス | 種別 |
| --- | --- | --- | --- |
| `/` | In Progress | `x` | IN_PROGRESS |
| `-` | Cancelled | ` ` | CANCELLED |

`HOME.md` の tasks クエリは `db/` 配下で `by: me` のノート、および `journal/` 配下のタスクを拾う。
`by` プロパティが無い `db/` のノートのタスクは出てこない（`journal/` 配下は `by` に関係なく拾う）。

`meta/template/templater/member.md` も `filter by function` を使い、**そのノートへのリンクを含むタスク**だけを集める。
こちらも `Enable custom searches` が ON でないと動かない。

---

## 3. プロパティ（frontmatter）の型

`.obsidian/types.json` は **git 管理外**。clone 直後は型が未定義で、
Bases の `done` / `halu` 列がチェックボックスにならず文字列として表示される。

設定 → プロパティ で、最低限これを設定する（`meta/base/file_property.md` の規約に対応）:

| プロパティ          | 型                         |
| -------------- | ------------------------- |
| `by`           | テキスト（`me` / `ai` / `ext`）  |
| `db`           | テキスト（`doc` / `issue` / `member`） |
| `status`       | テキスト（`wip` / `done` など）   |
| `created`      | 日付                        |
| `due`          | 日付                        |
| `done`         | チェックボックス                  |
| `halu`         | チェックボックス                  |
| `ai` / `model` | テキスト                      |
| `tags`         | タグ                        |

> 型はノートに値が入れば Obsidian が推測もするが、**Bases のフィルタ（`done != true` など）は型に依存する**ので明示しておくのが安全。

---

## 4. Obsidian 本体の設定

`.obsidian/app.json` は **git 管理外**（[[.gitignore]] の `/.obsidian/*` に対して whitelist していない）。
つまり下記はすべて **端末ごとに手で設定する**。

| 設定 | 値 | 理由 |
| --- | --- | --- |
| ファイル → 新規ノートの作成場所 | 指定フォルダ → `db` | Ctrl+N が `db/` に作られる前提。Templater の Folder Templates（`db`）もこれに合わせてある |
| ファイル → リンク更新 | 自動更新 ON | ノート移動時に `![[tickets.base]]` などの埋め込みリンクを壊さない |
| エディタ → 読みやすい行の長さ | OFF | Bases の表と ` ```tasks ` テーブルを幅いっぱいに出すため |
| デイリーノート → 新規ファイルの場所 | `journal/daily` | `daily-notes.json` は管理外。既定のままだと vault 直下に作られ、Templater の `journal/daily` テンプレートも当たらない |
| デイリーノート → テンプレートファイルの場所 | *(空欄)* | テンプレート適用は Templater の Folder Templates 側で行う。両方設定すると二重適用になる |

### CSS スニペット

`.obsidian/snippets/` は git 管理下・`appearance.json` の有効化リストも管理下なので、通常は clone だけで効く。
効いていなければ 設定 → 外観 → CSS スニペット で再読み込みして ON にする。

| スニペット | 役割 |
| --- | --- |
| `tasks-table.css` | ` ```tasks ` の結果をテーブル風に整列（`HOME.md` の見た目はこれ前提） |
| `section-headings.css` | 見出し・リスト・表の可読性 |
| `code-vscode-dark.css` | コードブロックを VSCode Dark+ 風に |
| `dataview-compact.css` | ` ```dataviewjs ` / ` ```dataview ` ブロックの上下余白を 1rem に統一（読み取りビューとライブプレビューで同じ量になるよう揃える。調査結果はファイル冒頭のコメント参照） |

---

## 5. meta/ の各ファイルと依存関係

| ファイル | 依存 |
| --- | --- |
| `meta/HOME.md` | Bases（`tickets.base` / `documents.base` の埋め込み）+ Tasks（custom searches）+ Dataview（JavaScript Queries）+ `tasks-table.css` + `![[MEMO]]` |
| `meta/MEMO.md` | なし（`HOME.md` に埋め込まれる走り書き用の空ノート） |
| `meta/base/db.base` | Bases。`db/` 配下・`status` / `by` / `ai` / `model` / `halu` / `created` |
| `meta/base/db/documents.base` | Bases。`db == "doc"` |
| `meta/base/db/tickets.base` | Bases。`db == "issue"` + `done`(checkbox) + `due`(date) |
| `meta/base/db/member.base` | Bases。`db == "member"` |
| `meta/base/db/DropZone/*.base` | Bases。`by == me/ai/ext` かつ `done != true` |
| `meta/base/db/DropZone/DropZone.md` | 上記 3 つの `.base` の埋め込み |
| `meta/template/*.md` | Templater（Folder Templates が参照） |
| `meta/template/templater/*.md` | Templater（手動挿入用）。`member.md` は Tasks + Dataview にも依存 |
| `meta/base/file_property.md` | プロパティ規約の定義（読み物。プラグイン依存なし） |

---

## 6. 動作確認チェックリスト

1. `meta/HOME.md` を開く → Issues の表（Bases）が描画される → **NG なら Bases（Obsidian ≥1.9）**
2. 同ノートの Tasks ブロックにエラーが出ない → **NG なら Tasks の Enable custom searches**
2-b. 同ノート冒頭に当日の日付リンク（`📄` / `➕`）が出る → **NG なら Dataview の Enable JavaScript Queries**（§2.2）
3. **Ctrl+N** で新規ノートを作る（`db/` に作られる）→ frontmatter に `by: me` / `created: <今日>` が入る
   → 空ファイルなら **`Trigger Templater on new file creation` が OFF**（§2.1）
   → 中身が違うなら **Folder Templates のマッピング**（§2.1）
4. `journal/daily/` に新規ノートを作る → `## Tasks` / `## Log` / `## Note` が入る
5. `meta/template/`（`templater/` 配下も含む）に新規ノートを作る → **何も展開されない**（空欄登録が効いている）
6. `DropZone.md` を開く → by_ai / by_ext / by_me の 3 表が出る
7. `db/` のノートに `done: true` を入れる → DropZone の表から消える（`done` がチェックボックス型になっているか）

---

## 7. 意図的に同梱していないもの

| もの | 理由 |
| --- | --- |
| `.obsidian/plugins/*/data.json` | API キー・ローカルパスなどの秘密が混入しうる。だから**この文書に手順として書く** |
| `.obsidian/workspace.json` | 開くだけで変化する。ペイン配置は端末ごとに違う |
| `.obsidian/types.json` | 管理外（§3 を手で設定する） |
| `.obsidian/app.json` | 管理外（§4 を手で設定する） |
| `.obsidian/plugins/` 本体 | プラグインは各自コミュニティストアからインストール |
