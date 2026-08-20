# Tech Design — 代码块/引用块折行（工程视角）

## 问题
`pre` 代码块内超长不可断行串（最长 431 字符 hex）在 `white-space: pre` + `overflow-x: auto` 下产生内部横向滚动条；用户需左右滚动才能看全内容。

## 修复设计（`_sass/_base.scss`）

### 1. 代码块折行
```scss
pre {
    /* 现有：mono 字体、14px、padding、暗底、圆角、阴影 */
    overflow-x: auto;           /* 保留兜底 */
    white-space: pre-wrap;      /* 超宽行折行，保留缩进与空行 */
    overflow-wrap: break-word;  /* 打断 431 字符 hex 等长 token */
    word-break: break-word;     /* 旧浏览器兼容 */
}
```
- `pre-wrap` 与 `pre` 差异仅在"超宽行是否折行"，短行/缩进/ASCII 图不受影响。
- `> code` 的 reset 保持不动。

### 2. 引用块显式加固
```scss
blockquote { overflow-wrap: break-word; }
```
（已从 `.post-content` 继承，此处显式声明更稳。）

## 不动
- `_syntax-highlighting.scss`、`pre > code` reset、`img`、表格规则。
- 不引入 `overflow-x: hidden`。

## 影响面
主题级、全站生效；仅超宽行折行，短内容为 no-op。
