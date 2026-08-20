# Decision Review — 决策评审

## 决策
代码块 `pre` 由"横向滚动"改为"折行"（`white-space: pre-wrap` + `overflow-wrap: break-word`），引用块显式加固折行。

## 评审结论
- **正确性**：根因是超长不可断行串（最长 431 字符 hex）导致内部横向滚动，`pre-wrap + break-word` 对症且是标准做法。
- **风险**：`pre-wrap` 使极个别超宽 ASCII 图折行——本文未发现此类图，且符合"看全内容"诉求；`word-break: break-word` 为弃用别名，仅作兼容兜底，主行为由 `overflow-wrap: break-word` 保证。
- **排除项复核**：
  - 方案 B（维持滚动）未满足目标，正确排除。
  - 方案 C（改 Markdown）对 431 字符 hex 不可行，正确排除。
- **范围**：主题级，附带修复全站同类问题。

## 结论
方案合理、风险可控，批准执行。
