
```tasks
# 要: 設定 → Tasks → Searches → "Enable custom searches" を ON（端末ごと）
done
filter by function (task.file.path.includes('db/') && task.file.property('by') === 'me') || task.file.path.includes('journal/')
sort by due
sort by priority
# group by filename
# 優先度・📅期限日・📝編集ボタンを列として揃えるため、他の要素は非表示にする
hide recurrence rule
hide created date
hide start date
hide scheduled date
# hide done date
hide cancelled date
hide id
hide depends on
hide on completion
```
