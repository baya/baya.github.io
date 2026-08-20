# Changelog — 变更记录

## [主题] 代码块/引用块超长内容折行，消除横向滚动条
- `_sass/_base.scss`：
  - `pre` 新增 `white-space: pre-wrap; overflow-wrap: break-word; word-break: break-word;`（保留 `overflow-x: auto` 兜底），代码块超长 hex/长行自动折行。
  - `blockquote` 新增 `overflow-wrap: break-word;` 显式加固。
- 效果：`/2017/05/11/7daystalk.html` 及全站代码块/引用块不再出现横向滚动条。
- 未改动任何 Markdown 源文件。
