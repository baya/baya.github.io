# Test Plan — 验证计划

## 构建验证
1. `bundle exec jekyll build` 成功，无 Sass 编译错误。
2. `_site/css/main.css` 中 `pre` 含 `white-space: pre-wrap; overflow-wrap: break-word; word-break: break-word;`；`blockquote` 含 `overflow-wrap: break-word;`。
3. `_site/2017/05/11/7daystalk.html` 的 `<pre>`/`<blockquote>` HTML 结构不变（CSS-only）。

## 浏览器实测（桌面 + 窄屏）
4. **代码块无横向滚动条**：`~~~c`/`~~~bash`/`~~~text` 中 431 字符 hex（第 2680 行）、scriptSig 等长行折行显示，`pre` 内部不再出现横向滚动条。
5. **页面无横向滚动**：`document.documentElement.scrollWidth <= window.innerWidth`。
6. **短代码不回归**：普通 C/bash/js 短行、缩进、空行、endian 示意图（短行 ASCII）渲染不变。
7. **视觉不回归**：代码块暗底 `#24201a`、圆角、阴影、语法高亮颜色正常。
8. **引用块**：第 2908 行 hex 及 BCC/SHA256 长段落折行正常。

## 回归抽查
- 另开一篇含代码块的旧文章确认无异常。
