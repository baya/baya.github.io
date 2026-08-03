---
layout: post
title: Alpha Skill：我做的一个 side project
---

利用业余时间捣鼓了一个 side project，叫 Alpha Skill。起因很简单：我自己一直在用各种 AI 助手做投资分析，但每次都要在聊天框里重新描述一遍需求，来回折腾提示词，效率很低，也很枯燥。于是就想做一个东西，把“和 AI 聊投资”这件事变得更好用一点。做出来之后发现除了投资这个具体场景，中间踩的几个坑和想通的几个道理，可能比结果本身更值得写下来。

先放一段完整流程的录屏，大致感受下从提需求到生成一个可用 Skill 的过程：

<figure>
  <video class="post-video" controls playsinline preload="metadata" poster="/images/alpha-skill/full-flow-poster.jpg">
    <source src="/videos/alpha-skill/full-flow.mp4" type="video/mp4">
  </video>
  <figcaption>从一句需求到一个可交互 Skill 的完整构建过程</figcaption>
</figure>

## 1. Chat 输入框太单调了，给 Skill 装一个 UI

用惯了各种 AI Chat 产品会发现一个通病：不管你要做什么事，交互的起点永远是同一个空荡荡的输入框。想分析一只股票的图表型态，得先打字告诉 AI 我要哪只股票、看多长周期、关注哪类形态——这些参数其实是结构化的，用表单选一下比敲字快得多，AI 也不用再费一轮猜参数。

所以 Alpha Skill 里每个 Skill 除了一段 Prompt，还会自动生成一个对应的操作界面（Params2UI）：

![Alpha Skill Params2UI 表单界面](/images/alpha-skill/params2ui.png)

股票代码、分析周期、分析重点直接选，选完点“开始分析”，比在 Chat 里一句一句描述省心太多。更有意思的是这个 UI 本身也是可以对话着改的，比如我在 Chat 里说了句“把分析重点改成 select”，界面就直接从卡片选择变成了下拉框——UI 不再是写死的模板，而是能被自然语言持续调整的东西：

<figure>
  <video class="post-video" controls playsinline preload="metadata" poster="/images/alpha-skill/chat1-poster.jpg">
    <source src="/videos/alpha-skill/chat1.mp4" type="video/mp4">
  </video>
  <figcaption>对话着改 UI：一句“把分析重点改成 select”，卡片选择就变成了下拉框</figcaption>
</figure>

分析结果的呈现同样不满足于一段文字。比如“图表型态分析”这个 Skill，我让它在结果页里同时用 Infographic 示意图讲清楚“上升三角形”这种形态的标准特征，又用 ECharts 画出真实股票的 K 线图做对照：

![Infographic 示意图 + ECharts K 线图混排](/images/alpha-skill/a2ui-mix.png)

一边是教科书式的形态讲解，一边是这只股票当下的真实走势，两者放在一起看，比单纯读一段 AI 生成的文字判断力强多了。做完之后这个 Skill 会变成一个独立可分享的小应用，直接把 URL 发给别人就能用：

![生成的独立分析应用](/images/alpha-skill/final-chat.png)

实际跑起来是这样的，打开链接直接就是那个专属的分析界面，不用再和 AI 解释一遍自己想干什么：

<figure>
  <video class="post-video" controls playsinline preload="metadata" poster="/images/alpha-skill/ai-chat-poster.jpg">
    <source src="/videos/alpha-skill/ai-chat.mp4" type="video/mp4">
  </video>
  <figcaption>生成后的独立 Skill 应用，打开即用</figcaption>
</figure>

## 2. 让系统把“解决问题的过程”自己变成“能力”

这是这个项目里我最想验证的一个想法。现在大部分 AI 应用的模式是：每次对话都从零开始推理，同样的问题被反复以自然语言的方式“重新想一遍”，浪费的不仅是时间，还有大量 token。但人类专家不是这么工作的——一个分析师做过几次“图表型态分析”之后，会沉淀出一套稳定的分析框架，下次遇到同类问题，变的只是股票代码和参数，思路是复用的。

Alpha Skill 想做的就是这件事：把一次性的问题解决过程，转化成系统自身可复用的能力。具体的做法是把每次对话拆成两部分——

- **重复的部分 prompt 化**：分析框架、判断逻辑、输出结构这些跨请求不变的东西，沉淀成一份 Skill 的 Prompt。
- **差异的部分参数化**：股票代码、时间周期、关注的形态类型这些每次都不一样的东西，抽成结构化参数，通过 UI 表单收集，而不是每次都用自然语言重新描述一遍。

![Skill 生成的 prompt.md 与结构化参数文件](/images/alpha-skill/prompt-code.png)

打开生成出来的文件目录能更直观地看到这个“可复用资产”长什么样：一份 `prompt.md` 承载分析框架，`params2ui`、`a2ui` 两个目录各自是独立的 UI 组件，外加一份 `schema.json` 定义参数结构：

![Skill 生成的完整文件结构：prompt.md、params2ui、a2ui、schema.json](/images/alpha-skill/file-structure.png)

效果是：第一次做“图表型态分析”可能要走完整的推理和构建过程（20 多步），但这个过程本身就是在生产一个可复用资产——一份 prompt.md，一套 UI 组件，一份参数 schema。第二次同样的需求，不再需要 LLM 重新“想”一遍分析框架，只需要把新的参数灌进去执行既有的 Skill 就行，响应更快，token 消耗也大幅下降。系统用得越多，沉淀下来的 Skill 库越厚，往后要做的“新鲜推理”占比就越低——这本质上是一种把 LLM 的即时推理能力，逐步转化为结构化、可复用能力的架构，某种意义上是让系统具备了“经验积累”的能力。

## 3. 落到投资这个具体场景：发现、验证、执行

前两点是我在做这个项目时琢磨出的通用架构思路，但项目本身还是要解决一个具体问题——帮我自己（以及愿意用它的人）在投资上少走弯路。我给它定的目标是覆盖投资决策的一整条链路：

- **发现**能持续产生超额收益的投资能力，而不是碰运气式的单次判断；
- **验证**这种能力是否真的站得住脚，而不是自己骗自己；
- **执行**下来，把想法落到实际的交易动作上。

这条链路上每一段都对应着一批可以沉淀的 Skill：形态识别、事件驱动分析、仓位管理、复盘总结……每个 Skill 既是一个独立的分析工具，又在被使用的过程中反过来验证自己是不是真的“有效”。这也是为什么我觉得第 2 点的架构思路特别适合投资这个场景——投资本身就是一件需要不断试错、沉淀经验、然后把经验固化下来复用的事情，和“把问题解决过程转化为能力”这个思路天然契合。

目前项目还在很早期，很多地方粗糙得很，但这几个方向——用 UI 替代单调的 Chat 输入、让系统自己积累能力、服务于投资决策的完整链路——我觉得是值得继续投入的。后面做出更多东西再来更新。
