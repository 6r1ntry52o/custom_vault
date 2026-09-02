%% tasks の結果で期日・優先度セルをクリック編集できるようにする。実装と設計メモは下の dataviewjs 本体に同梱。
   2026-08-23: meta/script/tasks-inline-edit.js を dv.view() で読む形をやめ、この md に直書きした。
   理由＝Obsidian Sync は既定で md しか同期せず（.js は「その他のファイルタイプ」＝初期OFF）、
   モバイルでは .js が届かず「見た目は正常なのにセルを押しても無反応」になっていたため。
   このブロックは tasks ブロックより先に実行される必要があるので、必ず HOME.md の最上部に置くこと。 %%
```dataviewjs
/* tasks-inline-edit
 * meta/HOME.md 最上部の dataviewjs ブロック本体（2026-08-23 に .js 直読みから直書きへ移行）。
 * 注: このコメント内にバッククォート3つを書くとコードフェンスがそこで閉じる。書かないこと。
 * tasks コードブロックの検索結果に、その場での編集を足す。
 *
 *   期日セル（＋）をクリック  → Tasks 内蔵の flatpickr カレンダー
 *   優先度セル（絵文字 / -）  → 優先度メニュー（Obsidian の Menu）
 *   本文をクリック            → そのタスクが書かれている md をその行で開く
 *                              （リンク・タグの上と、文字を選択した直後は素通り）
 *   📝 をクリック             → 従来どおり編集モーダル
 *   ファイル列                → "ファイル名 > 見出し" から見出しとフォルダを落とす
 *
 * 見た目の列組みは .obsidian/snippets/tasks-table.css が担当。対で動く。
 *
 * ---- なぜ JS が要るか ----
 * Tasks の QueryResultsRenderer.taskToHtml() は
 *     let l = serializer.componentToString(task, shortMode, component);
 *     if (l) { ...span を作り click でカレンダーを開く... }
 * という実装で、値が無いと componentToString() が空文字を返して if(l) に弾かれ、
 * span 自体が生成されない。期日なし・優先度 normal("3") がこれに当たる。
 * CSS は要素を作れずハンドラも付けられないので、空セルに直接 UI は生やせない。
 * なお Tasks が click/contextmenu を張るのは日付フィールドとチェックボックスだけで、
 * 優先度には何のハンドラも無い。
 *
 * ---- どうやって Task オブジェクトを手に入れるか ----
 * li から元ファイルの行を逆引きする手段は無い（li の data-line はファイル行番号では
 * なく addTask(task, taskIndex) の taskIndex、backlink の <a> に href も無い）。
 * 代わりにプラグイン自身の配線を借りる。addEditButton() は
 *     i.addEventListener("click", s =>
 *       this.htmlQueryRendererParameters.editTaskPencilClickHandler(s, r, ...))
 * と書かれており、r が Task オブジェクトそのもの。この htmlQueryRendererParameters は
 * QueryRenderChild が持つオブジェクトを参照で共有しているので、そこを 1 枚ラップすれば
 * 📝 クリック時に Task が引数で降ってくる。セルのクリックはフラグを立てて 📝 を click()
 * するだけでよく、DOM からの同定推測が一切要らなくなる。
 *
 * ---- 注意 ----
 * Tasks の内部実装（plugin.queryRenderer / htmlQueryRendererParameters /
 * window.flatpickr）に依存している。検証済みは Tasks v8.3.0。
 * このブロックを書き換えたら下の VERSION を必ず上げること。上げないと同じ
 * セッションでは旧版が居座り、CSS だけ新しくなって「見た目は変わったのに
 * 動かない」という原因の見えない壊れ方をする。
 * HOME を開く前に別ノートの tasks ブロックを描画していた場合、そのブロック
 * だけは再描画までフォールバック（CSS のみ・編集モーダル経由）のままになる。
 */

(() => {
  const KEY = "__tasksInlineEdit__";
  /* ★このブロックを書き換えたら必ずこの値を上げること。
     旧実装は `if (window[KEY]) return;` で「1 セッション 1 回」しか入らず、
     md を直しても Obsidian を再起動するまで旧版が居座った。CSS だけは
     ホットリロードされるので「見た目は新しいのに動かない」という、原因の
     見えない壊れ方をする（2026-09-01 に実際に踏んだ）。
     版が違えば下で前のパッチを外して入れ直すので、ノートを開き直すだけで
     更新が効く。 */
  const VERSION = "2026-09-01.5";

  const plugin = app.plugins.plugins["obsidian-tasks-plugin"];
  const qr = plugin && plugin.queryRenderer;
  if (!qr || typeof qr.addQueryRenderChild !== "function") {
    console.warn("[tasks-inline-edit] Tasks プラグインが見つからないか構造が変わっています");
    return;
  }

  const installed = window[KEY];
  if (installed && installed.version === VERSION) return;   // 同じ版＝何もしない
  if (installed && typeof installed.uninstall === "function") {
    // 前の版のパッチを外してから入れ直す（多重ラップを防ぐ）
    try { installed.uninstall(); } catch (e) { console.error("[tasks-inline-edit] uninstall", e); }
  }

  let Notice = null, Menu = null;
  try { const ob = require("obsidian"); Notice = ob.Notice; Menu = ob.Menu; } catch (e) { /* noop */ }
  const notify = (m) => { if (Notice) new Notice(m, 6000); else console.warn("[tasks-inline-edit]", m); };

  /* Task.priority は文字列。"3" = Normal（絵文字なし）。
     ラベルの記号は tasks-table.css の表示と揃えてある。md に保存される字は
     Tasks 側の記法（🔺⏫🔼）のままで、ここと CSS は「見せ方」だけを差し替えて
     いる。片方だけ直すと、一覧の表示とメニューの表示がズレる。 */
  const PRIORITIES = [
    { value: "0", label: "🔴 Highest" },
    { value: "1", label: "🟠 High" },
    { value: "2", label: "🟡 Medium" },
    { value: "3", label: "- Normal" },
    { value: "4", label: "🔽 Low" },
    { value: "5", label: "⏬ Lowest" },
  ];

  // セル click → 📝 click の間だけ立つ依頼。{ kind: "due"|"priority", anchor }
  let pending = null;

  /* Task を書き換えて元ファイルへ書き戻す。
     行の組み立てはプラグイン自身の Task クラスと toFileLineString() に任せる。
     Task のコンストラクタは resolveDate(dueDate, e._dueDate) のように
     「公開名が undefined なら _ 付きの内部フィールドで補う」設計になっている
     ので、Object.assign({}, task) をそのまま渡してよい（プラグイン内部の
     SetTaskDate / postpone も同じことをしている）。 */
  async function applyTaskChange(task, patch) {
    const Task = task.constructor;
    const updated = new Task(Object.assign({}, task, patch));
    const newLine = updated.toFileLineString();
    if (newLine === task.originalMarkdown) return;

    const file = app.vault.getFileByPath(task.path);
    if (!file) { notify(`tasks-inline-edit: ファイルが見つかりません (${task.path})`); return; }

    let stale = false;
    await app.vault.process(file, (data) => {
      const eol = data.indexOf("\r\n") !== -1 ? "\r\n" : "\n";
      const lines = data.split(/\r?\n/);
      // 描画時のスナップショットと実ファイルがズレていたら何もしない
      if (lines[task.lineNumber] !== task.originalMarkdown) { stale = true; return data; }
      lines[task.lineNumber] = newLine;
      return lines.join(eol);
    });
    if (stale) notify("tasks-inline-edit: ファイルが変更されているため書き込みを中止しました。再描画してからやり直してください。");
  }

  // --- 期日: Tasks 同梱の flatpickr（window.flatpickr で公開）をそのまま使う ---
  function openDuePicker(anchor, task) {
    const fp = window.flatpickr;
    if (!fp) { notify("tasks-inline-edit: flatpickr が見つかりません"); return; }

    /* flatpickr は clickOpens:true で anchor 自身にも click を張るため、
       開いている間にもう一度セルを押すと二重にインスタンスが生まれる。
       1 アンカーにつき 1 つだけ生かす。 */
    if (anchor.dataset.picking === "1") return;
    anchor.dataset.picking = "1";
    const release = (inst) => { delete anchor.dataset.picking; inst.destroy(); };

    let firstDay = 1;
    try {
      const wi = new Intl.Locale(navigator.language).weekInfo;
      if (wi && typeof wi.firstDay === "number") firstDay = wi.firstDay;
    } catch (e) { /* noop */ }

    fp(anchor, {
      defaultDate: new Date(),
      disableMobile: true,
      enableTime: false,
      dateFormat: "Y-m-d",
      locale: { firstDayOfWeek: firstDay },
      onClose: async (dates, _str, inst) => {
        try { if (dates.length > 0) await applyTaskChange(task, { dueDate: window.moment(dates[0]) }); }
        finally { release(inst); }
      },
      onReady: (_d, _s, inst) => {
        // 期日ありの行のピッカーと同じ見た目になるよう、同じクラス名を使う
        const bar = document.createElement("div");
        bar.classList.add("tasks-date-picker-buttons");
        const btn = document.createElement("button");
        btn.type = "button";
        btn.textContent = "Today";
        btn.classList.add("flatpickr-button");
        btn.addEventListener("click", async () => {
          try { await applyTaskChange(task, { dueDate: window.moment(new Date()) }); }
          finally { release(inst); }
        });
        bar.appendChild(btn);
        inst.calendarContainer.appendChild(bar);
      },
    }).open();
  }

  /* --- 優先度: プラグインに優先度メニューは無い（日付とステータスにしか
         右クリックメニューが用意されていない）ので Obsidian の Menu で作る。
         Task.priority は素のフィールドなので Object.assign で差し替えられる。 */
  function openPriorityMenu(anchor, task) {
    if (!Menu) { notify("tasks-inline-edit: Obsidian の Menu を読み込めません"); return; }
    const menu = new Menu();
    for (const p of PRIORITIES) {
      menu.addItem((item) =>
        item
          .setTitle(p.label)
          .setChecked(task.priority === p.value)
          .onClick(() => applyTaskChange(task, { priority: p.value })),
      );
    }
    const r = anchor.getBoundingClientRect();
    menu.showAtPosition({ x: r.left, y: r.bottom });
  }

  /* --- 本文クリックで、そのタスクが書かれている md を開く ---
     ファイル列のバックリンクと同じ行き先だが、そちらは末尾の細い列で狙いにくい。
     Task を持っているので lineNumber まで渡せる＝ノートを開いた直後にその行へ飛ぶ
     （バックリンクは見出し単位）。newLeaf は Ctrl/Cmd+クリックで新しいタブに開く。 */
  async function openTaskFile(task, newLeaf) {
    const file = app.vault.getFileByPath(task.path);
    if (!file) { notify(`tasks-inline-edit: ファイルが見つかりません (${task.path})`); return; }
    try {
      const leaf = app.workspace.getLeaf(newLeaf ? "tab" : false);
      await leaf.openFile(file, { active: true, eState: { line: task.lineNumber } });
    } catch (e) {
      // 握り潰すと「押しても何も起きない」になり原因が追えない
      console.error("[tasks-inline-edit] openTaskFile", e);
      notify("tasks-inline-edit: ノートを開けませんでした（コンソールを確認）");
    }
  }

  /* --- ファイル列を「ファイル名だけ」にする ---
     Tasks の getLinkText() は "ファイル名 > 見出し" を返す（同名ファイルが他にも
     ある時は、ファイル名の代わりにパスが入る）。一覧では「どのノートのタスクか」
     さえ分かればよいので、見出しとフォルダを落として表示する。
     元の文字列は title に残すので、ホバーすれば場所まで分かる。
     ※ CSS では削れない。"(" と ")" と同じくテキストノードの一部であり、
        セレクタで掴めるのは <a> 全体だけだから。 */
  function shortenBacklink(a) {
    /* ⚠ Tasks は <a> を先に DOM へ挿し、そのあとで text を入れる（addBacklinks:
       Ge("a", span) → 属性設定 → o.text = getLinkText(...)）。decorate はその
       途中でも走りうるので、空のうちに「済み」の印を立てると二度と直らない。
       印は実際に詰められた時だけ立て、空なら何もせず次の変化を待つ。 */
    const full = a.dataset.tieFull || a.textContent;
    if (!full) return;
    /* getLinkText() が返す形は 2 通り＋見出し:
         ファイル名が vault 内で一意 → "ファイル名"（拡張子なし）
         同名ファイルが他にもある     → "/フォルダ/ファイル名.md"
       いずれも見出しがあれば " > 見出し" が付く。
       末尾の .md はパス形式の時だけ付くので、落としてから使う。 */
    const short = full.split(" > ")[0].split("/").pop().replace(/\.md$/i, "");
    if (!short) return;
    if (a.dataset.tieFull && a.textContent === short) return;  // 既に詰め済み
    a.dataset.tieFull = full;                 // 元の文字列（再描画時の再計算用）
    a.setAttribute("title", full);            // ホバーすれば見出しまで分かる
    a.textContent = short;
  }

  /* セルに click を張る。Task オブジェクトは自前で同定せず、フラグを立てて
     その行の 📝 を click() し、ラップした editTaskPencilClickHandler の
     引数として受け取る（下の hookChild を参照）。 */
  function wire(el, li, kind, title) {
    if (el.dataset.tieWired === "1") return;
    const edit = li.querySelector(":scope > .task-extras > .tasks-edit");
    if (!edit) return; // hide edit button のクエリでは何もしない
    el.dataset.tieWired = "1";
    el.setAttribute("role", "button");
    el.setAttribute("title", title);
    el.addEventListener("click", (ev) => {
      ev.preventDefault();
      ev.stopPropagation();
      pending = { kind, anchor: el };
      try { edit.click(); } finally { pending = null; }
    });
  }

  /* 本文セルに click を張る。wire() と違い「素通りさせる条件」がある。
       ・<a> の上（🔗 / [[ ]] / #タグ）→ 本来の遷移を優先する。ここが
         「リンクが無い領域だけ効く」の実装。closest() なので <a> の子の
         <strong> や <code> を押しても正しく素通りする。
       ・文字を選択した直後 → ドラッグで選択して指を離すと click が出るため、
         これを弾かないと引用しようとするたびにノートへ飛んでしまう。
     role="button" は付けない（本文は読み物であって操作子ではない）。 */
  function wireOpen(el, li) {
    if (el.dataset.tieOpen === "1") return;
    const edit = li.querySelector(":scope > .task-extras > .tasks-edit");
    if (!edit) return; // hide edit button のクエリでは何もしない
    el.dataset.tieOpen = "1";
    el.setAttribute("title", "Click to open the note");
    el.addEventListener("click", (ev) => {
      if (ev.target.closest("a")) return;
      const sel = window.getSelection();
      if (sel && !sel.isCollapsed) return;
      ev.preventDefault();
      ev.stopPropagation();
      pending = { kind: "open", anchor: el, newLeaf: ev.ctrlKey || ev.metaKey };
      try { edit.click(); } finally { pending = null; }
    });
  }

  function placeholder(text, cls) {
    const el = document.createElement("span");
    el.className = cls;
    return el;
  }

  function decorate(root) {
    // この ul は JS が面倒を見ている、という印。CSS 側の Option A を止める。
    root.querySelectorAll("ul.plugin-tasks-query-result").forEach((ul) => {
      ul.dataset.duePicker = "on";
    });

    root.querySelectorAll("li.plugin-tasks-list-item").forEach((li) => {
      const text = li.querySelector(":scope > .tasks-list-text");
      if (!text) return;

      // 期日: 未設定なら span 自体が生成されないのでプレースホルダを挿す
      if (!li.hasAttribute("data-task-due")) {
        let due = text.querySelector(":scope > .tasks-due-placeholder");
        if (!due) {
          due = placeholder("", "tasks-due-placeholder");
          text.insertBefore(due, text.firstChild);
        }
        wire(due, li, "due", "Click to set due date");
      }

      // 優先度: normal では span が生成されないので同様にプレースホルダ
      let pri = text.querySelector(":scope > .task-priority");
      if (!pri) {
        pri = text.querySelector(":scope > .tasks-priority-placeholder");
        if (!pri) {
          pri = placeholder("", "tasks-priority-placeholder");
          text.insertBefore(pri, text.firstChild);
        }
      }
      wire(pri, li, "priority", "Click to set priority");

      // 本文: クリックでそのタスクが書かれている md を開く
      const desc = text.querySelector(":scope > .task-description");
      if (desc) wireOpen(desc, li);

      // ファイル列: "ファイル名 > 見出し" から見出しとフォルダを落とす
      const back = li.querySelector(":scope > .task-extras > .tasks-backlink > a");
      if (back) shortenBacklink(back);
    });
  }

  function hookChild(child) {
    const params = child && child.queryResultsRenderer && child.queryResultsRenderer.htmlQueryRendererParameters;
    if (params && params.__tieWrapped !== VERSION) {
      const orig = params.editTaskPencilClickHandler;
      params.editTaskPencilClickHandler = function (event, task, allTasks) {
        const req = pending;
        pending = null;
        if (req) {                          // セル経由 → その場で編集 UI
          event.preventDefault();
          event.stopPropagation();
          if (req.kind === "due") openDuePicker(req.anchor, task);
          else if (req.kind === "priority") openPriorityMenu(req.anchor, task);
          else if (req.kind === "open") openTaskFile(task, req.newLeaf);
          return;
        }
        return orig.call(this, event, task, allTasks); // 素の 📝 → 編集モーダル
      };
      params.__tieWrapped = VERSION;
    }

    const root = child && child.containerEl;
    if (!root) return;
    // Tasks はキャッシュ更新のたび debounce(300ms) で DOM ごと作り直すので、
    // 一度挿すだけでは足りない。
    const obs = new MutationObserver(() => decorate(root));
    obs.observe(root, { childList: true, subtree: true });
    if (typeof child.register === "function") child.register(() => obs.disconnect());
    decorate(root);
  }

  // tasks ブロックごとに作られる QueryRenderChild を捕まえる。
  // _addQueryRenderChild は生成したインスタンスを返さず ctx.addChild(o) に
  // 渡すだけなので、その 1 回の呼び出しだけ ctx.addChild をラップする。
  const origAdd = qr.addQueryRenderChild;
  qr.addQueryRenderChild = async function (source, el, ctx) {
    const hadOwn = Object.prototype.hasOwnProperty.call(ctx, "addChild");
    const origAddChild = ctx.addChild;
    const restore = () => { if (hadOwn) ctx.addChild = origAddChild; else delete ctx.addChild; };
    ctx.addChild = function (child) {
      restore();
      try { hookChild(child); } catch (e) { console.error("[tasks-inline-edit]", e); }
      return origAddChild.call(this, child);
    };
    try { return await origAdd(source, el, ctx); }
    finally { restore(); }
  };

  window[KEY] = {
    version: VERSION,
    uninstall: () => { qr.addQueryRenderChild = origAdd; },
  };
  console.log("[tasks-inline-edit] installed", VERSION);
})();

/* ここまで到達したらブロック自体を隠す。途中で throw した場合は隠さないので、
   Dataview が出すエラー表示がそのまま見える。 */
dv.container.style.display = "none";
const _wrap = dv.container.closest(".el-pre, .el-div");
if (_wrap) _wrap.style.display = "none";
```

```dataviewjs
const day = moment().locale("en"); // 曜日を英語表記（Mon/Tue...）にするため
const path = `journal/daily/${day.format("YYYY-MM-DD")}.md`;
const exists = app.vault.getAbstractFileByPath(path) !== null;
dv.paragraph(`${exists ? "📄" : "➕"} [[${path}|${day.format("YYYY-MM-DD (ddd)")}]]`);
```

![[MEMO]]

##### [[DropZone]]

## Issues

![[tickets.base]]

## Tasks

```tasks
# 要: 設定 → Tasks → Searches → "Enable custom searches" を ON（端末ごと）
not done
filter by function (task.file.path.includes('db/') && task.file.property('by') === 'me') || task.file.path.includes('journal/')
# 並びは「① 期限 → ② 優先度」。以下は Tasks の仕様なので把握しておくとよい:
#   ・優先度は第2キー＝同じ期限どうしの中でしか効かない。期限がバラバラな間は
#     並びが実質「期限順そのもの」になり、優先度が高くても上には来ない。
#   ・期日なしの行は必ず最後にまとまる（無効な日付だけは先頭）。
#   ・両方を1つのスコアで混ぜたいなら sort by urgency（期限と優先度の合成）に置き換える。
sort by due
sort by priority
# group by filename
# 優先度・📅期限日・📝編集ボタンを列として揃えるため、他の要素は非表示にする
hide recurrence rule
hide created date
hide start date
hide scheduled date
hide done date
hide cancelled date
hide id
hide depends on
hide on completion
```
- [[done_tasks]]

## Documents

![[documents.base]]

%%以下自由編集%%

