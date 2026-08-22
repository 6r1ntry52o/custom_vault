%% tasks の結果で期日・優先度セルをクリック編集できるようにする。実装と設計メモは meta/script/tasks-inline-edit.js %%
```dataviewjs
await dv.view("meta/script/tasks-inline-edit")
```

```dataviewjs
const day = moment().locale("en"); // 曜日を英語表記（Mon/Tue...）にするため
const path = `journal/daily/${day.format("YYYY-MM-DD")}.md`;
const exists = app.vault.getAbstractFileByPath(path) !== null;
dv.paragraph(`${exists ? "📄" : "➕"} [[${path}|${day.format("YYYY-MM-DD (ddd)")}]]`);
```
![[MEMO]]

## Issues

![[tickets.base]]

## Tasks

```tasks
# 要: 設定 → Tasks → Searches → "Enable custom searches" を ON（端末ごと）
not done
filter by function (task.file.path.includes('db/') && task.file.property('by') === 'me') || task.file.path.includes('journal/')
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

##### [[DropZone]]
![[documents.base]]

%%以下自由編集%%

