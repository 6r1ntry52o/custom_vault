---
aliases:
db: member
created: <% tp.file.creation_date("YYYY-MM-DD") %>
---
## Tasks
```tasks
# 要: 設定 → Tasks → Searches → "Enable custom searches" を ON（端末ごと）
# not done
# このノートへのリンクが張られている task に絞る
filter by function (task.description.match(/\[\[([^\]|#^]+)/g) || []).some(l => l.slice(2).split('/').pop().trim() === '{{query.file.filenameWithoutExtension}}')
sort by due
# group by filename
# CSS列として揃えるため、他の要素は非表示
hide priority
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

## Relations
```dataview
TABLE WITHOUT ID file.link AS "filename", item.text AS "contens"
FLATTEN file.lists AS item
WHERE item.task != true AND !contains(file.tasks.line, item.line) AND file.path != this.file.path AND contains(item.text, "[[" + this.file.name)
SORT file.name ASC
```

