

## by AI
![[by_ai.base]]

## by Ext
![[by_ext.base]]

## by Me
![[by_me.base]]

```yaml
filters:
  and:
    - file.path.startsWith("db/")
    - by == "H2"
    - done != true
```
