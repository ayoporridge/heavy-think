---
name: heavy-think
description: >-
  多 Agent 并行深度分析。对同一问题从多个独立视角并行推理，deliberation 合成出超越任何单个 agent 的结论，输出 Markdown + HTML 报告。
  适用于：技术决策、架构选型、复杂分析、有深度的判断类问题。
  Use when user says 'heavy-think', '/heavy-think', '多角度分析', '深度分析', '多agent分析'.
user_invocable: true
allowed-tools: Agent, Write, Read, Bash
---

# heavy-think: 多 Agent 并行深度分析

基于 arXiv:2605.02396：多个隔离 agent 独立推理，deliberation 合成——HP@K 常常超过所有单个 trace，因为 **deliberation 是生成性的**，合成结论包含任何单个视角都未能触达的正确答案。

---

## 核心原则

- **真正隔离**：K 个 subagent 在完全独立的 context 里运行，不是"被要求忽略之前的上下文"，而是天然不共享状态
- **不投多数票**：synthesis 按推理质量评估，而非按哪个结论出现频率高
- **Shuffle 防偏**：deliberation 前打乱 trace 顺序，消除位置偏差
- **最多 2 轮**：第 2 轮后质量退化，超过即停止
- **信源限制**：subagent 分析只引用可靠来源——学术（arXiv/Nature/ACM/IEEE）、权威媒体（Reuters/Bloomberg/FT/Economist/MIT Tech Review/Wired）、官方来源、国内可信媒体（财新/第一财经/36氪/人民网）；不引用个人博客、未经核实的社媒内容、无出处的统计数据

---

## 执行流程

### Step 0：理解问题，判断模式

分析用户的问题，判断属于哪种模式：

**Verification 模式**：问题有更正确的答案（代码 bug、技术方案对比、逻辑推理、数学推导）

**Deliberation 模式**：问题有多个合理视角，没有唯一正确答案（产品决策、战略选择、架构权衡、价值判断）

向用户用 1-2 句话确认问题理解和分析模式，然后直接进入执行，不等待批准。

生成一个简短的 `slug`（英文小写，用连字符，3-5 个词，描述问题核心），用于文件命名。

---

### Step 1：确定 K 值与视角分配

**K 值选择：**
- K=3：标准复杂度问题
- K=4：涉及多方利益或多个技术维度
- K=5：高风险决策、架构级问题、需要充分覆盖对立面

**Verification 模式**的推理方式（每个 agent 各取一种，K=3 时取前三）：
- `direct-reasoning`：直接逐步拆解，严格遵循逻辑链
- `first-principles`：从最基本的假设出发，不依赖已有结论
- `adversarial`：主动构造反例，寻找漏洞和边界失效点
- `edge-case`：极端条件和边界情况下的行为分析
- `analogy`：类比历史上解决过的相似问题
- `constraint-propagation`：从约束条件反推，限制空间里的可行解

**Deliberation 模式**的视角角色（每个 agent 各取一种，K=3 时取 Builder/Skeptic/Architect）：
- `Builder`：执行可行性，如何做到，路径规划
- `Skeptic`：质疑核心假设，识别风险和盲点
- `Architect`：系统设计，长期结构，技术债务
- `User`：使用者/受影响方的真实体验和需求
- `Economist`：成本收益，资源约束，激励机制
- `Contrarian`：反向论证，挑战主流直觉，提出非共识观点

---

### Step 2：创建输出目录

```
~/.heavy-think/[YYYY-MM-DD]-[slug]/
  traces/
    trace-01-[role].md
    trace-02-[role].md
    ...
  deliberation.md
  report.md
  report.html
```

用 Bash 创建这个目录结构：
```bash
mkdir -p ~/.heavy-think/[date]-[slug]/traces
```

---

### Step 3：并行启动 K 个 isolated subagent

**关键：所有 K 个 Agent 调用必须在同一条消息里发出，确保真正并行。**

每个 subagent 的 prompt：

```
你是一个独立分析师。你的推理视角/方式是：[ROLE]

问题：
[QUESTION]

分析要求：
1. 完全以 [ROLE] 的视角/推理方式工作，不要模拟其他视角
2. 只跟随你自己的推理，不要预判"其他分析师会怎么说"
3. 完整写出推理过程，不只给结论
4. 如果引用数据或案例，只使用以下可靠来源：学术论文（arXiv/Nature/ACM/IEEE）、权威媒体（Reuters/Bloomberg/FT/Economist/MIT Tech Review/Wired）、官方文档、国内可信媒体（财新/第一财经/36氪/人民网）
5. 明确说出你的局限性：[ROLE] 这个视角/方式天然看不到什么？

输出结构（严格按此格式）：

## 核心论点
[你的主要结论，2-4 句话，直接表态]

## 推理过程
[完整的分析步骤，展示你的思考链路]

## 关键洞察
[这个视角下最重要、最容易被忽视的发现，具体且有支撑]

## 引用来源
[如有引用，列出来源；如无需引用则写"本次分析基于逻辑推理，无需外部来源"]

## 局限性
[诚实说明：[ROLE] 这个视角/方式的盲点，以及哪些问题你没有回答]
```

收到所有 K 个 agent 的结果后，分别写入：
`~/.heavy-think/[date]-[slug]/traces/trace-0N-[role].md`

---

### Step 4：Deliberation 合成

将所有 trace **打乱顺序**后（防止位置偏差），启动一个专门的 synthesis subagent。

Synthesis subagent 的 prompt：

```
你是一个 deliberation 合成者。以下是 [K] 个独立分析师对同一问题的完整推理 trace，顺序已随机打乱。

问题：
[QUESTION]

[打乱顺序后的所有 trace，用 === Trace N === 分隔]

你的任务——严格执行以下四步协议：

**Step 1 — 分类**
这是 Verification 问题（存在更正确的答案）还是 Deliberation 问题（多视角共存，无唯一正解）？一句话说明判断依据。

**Step 2 — 逐一评估每个 trace**
对每个 trace，评估：
- 推理链路是否严密？有无逻辑跳跃？
- 哪些洞察有独特价值？
- 哪些结论有明显盲点或需要质疑？
不要因为某个 trace 的措辞更好而给它更高权重。

**Step 3 — 重新独立推导（最关键的步骤）**
不要直接汇总。基于你在所有 trace 里看到的推理材料，**重新独立推导**结论。
- 如果某个 trace 开启了一条新推理路径，顺着走下去，不要只是引用它
- 如果各 trace 存在真实分歧，分析分歧的根源（假设不同？还是推理不同？）
- 目标：产出一个单个 trace 无法独立推导出来的、更深的结论

**Step 4 — 合成输出**
按以下结构输出最终合成（这是你的原始输出，后续不会被改写）：

### 问题性质
[Verification / Deliberation，一句话说明]

### 核心结论
[直接回答问题，1-2 段，明确表态]

### 关键论点
[3-5 条，每条带推理依据，不是简单列点]

### 各视角共识
[哪些结论在多个独立 trace 里都出现了，说明这些结论的鲁棒性高]

### 视角分歧
[哪些地方 trace 之间有真实争议，分析分歧根源，说明为什么某个方向更可信]

### 反直觉洞察
[deliberation 过程中发现的、单个 trace 都没有的新发现，如果有的话]

### 行动建议或决策框架
[如果问题是决策类，给出具体可执行的建议；如果是分析类，给出思考框架]

### 置信度评估
[对这次合成的置信度打分：高/中/低，说明原因，以及还需要什么信息才能提高置信度]
```

将 synthesis 结果原文写入：
`~/.heavy-think/[date]-[slug]/deliberation.md`

---

### Step 5：生成报告文件

**5a. 生成 report.md**

将 deliberation 结果整理为最终报告，写入 `report.md`：

```markdown
# [问题标题]

**分析模式**：[Verification / Deliberation]
**Agent 数量**：K=[N]
**视角组合**：[role_1] · [role_2] · [role_3] ...
**生成时间**：[YYYY-MM-DD HH:MM]

---

[deliberation.md 的原文，不做改写]

---

*本报告由 [N] 个独立 agent 并行分析后经 deliberation 合成。*
*方法论参考：arXiv:2605.02396*
```

**5b. 生成 report.html**

写入 `report.html`，Medium 风格排版：

```html
<!DOCTYPE html>
<html lang="zh">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>[问题标题] — Heavy Think Report</title>
<style>
  /* ── Reset & Base ── */
  *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
  
  body {
    font-family: -apple-system, "Noto Serif SC", Georgia, serif;
    font-size: 20px;
    line-height: 1.8;
    color: #1a1a1a;
    background: #fafaf8;
  }

  /* ── Layout ── */
  .container {
    max-width: 740px;
    margin: 0 auto;
    padding: 80px 24px 120px;
  }

  /* ── Header ── */
  .report-header {
    border-bottom: 1px solid #e8e8e4;
    padding-bottom: 40px;
    margin-bottom: 56px;
  }

  .report-meta {
    display: flex;
    gap: 24px;
    flex-wrap: wrap;
    margin-bottom: 20px;
    font-family: -apple-system, "PingFang SC", sans-serif;
    font-size: 13px;
    color: #888;
    letter-spacing: 0.02em;
  }

  .meta-badge {
    background: #f0f0eb;
    border-radius: 4px;
    padding: 3px 10px;
  }

  .meta-badge.mode-verification { background: #eef6ff; color: #2563eb; }
  .meta-badge.mode-deliberation { background: #fef3e8; color: #d97706; }

  h1.report-title {
    font-size: 36px;
    font-weight: 700;
    line-height: 1.3;
    letter-spacing: -0.02em;
    color: #111;
    margin-bottom: 16px;
  }

  .report-subtitle {
    font-size: 18px;
    color: #555;
    font-weight: 400;
    line-height: 1.5;
  }

  /* ── Section ── */
  .section {
    margin-bottom: 56px;
  }

  h2 {
    font-size: 13px;
    font-weight: 600;
    text-transform: uppercase;
    letter-spacing: 0.12em;
    color: #888;
    font-family: -apple-system, "PingFang SC", sans-serif;
    margin-bottom: 20px;
  }

  /* ── Core Conclusion — big callout ── */
  .core-conclusion {
    font-size: 24px;
    font-weight: 500;
    line-height: 1.6;
    color: #111;
    border-left: 3px solid #1a1a1a;
    padding-left: 24px;
    margin-bottom: 20px;
  }

  /* ── Key Points ── */
  .key-points {
    list-style: none;
    counter-reset: point-counter;
  }

  .key-points li {
    counter-increment: point-counter;
    display: flex;
    gap: 16px;
    margin-bottom: 28px;
    align-items: flex-start;
  }

  .key-points li::before {
    content: counter(point-counter, decimal-leading-zero);
    font-family: -apple-system, "PingFang SC", sans-serif;
    font-size: 12px;
    font-weight: 600;
    color: #bbb;
    min-width: 24px;
    padding-top: 4px;
  }

  /* ── Insight callout ── */
  .insight-box {
    background: #f5f5f0;
    border-radius: 8px;
    padding: 24px 28px;
    margin: 32px 0;
  }

  .insight-box .insight-label {
    font-family: -apple-system, "PingFang SC", sans-serif;
    font-size: 11px;
    font-weight: 700;
    text-transform: uppercase;
    letter-spacing: 0.14em;
    color: #888;
    margin-bottom: 12px;
  }

  .insight-box p {
    font-size: 18px;
    font-weight: 500;
    line-height: 1.7;
    color: #222;
  }

  /* ── Consensus / Divergence ── */
  .consensus-block {
    border-left: 2px solid #22c55e;
    padding-left: 20px;
    margin-bottom: 20px;
  }

  .divergence-block {
    border-left: 2px solid #f97316;
    padding-left: 20px;
    margin-bottom: 20px;
  }

  /* ── Action items ── */
  .action-list {
    list-style: none;
  }

  .action-list li {
    display: flex;
    align-items: flex-start;
    gap: 12px;
    margin-bottom: 16px;
    padding: 16px 20px;
    background: #fff;
    border: 1px solid #e8e8e4;
    border-radius: 8px;
  }

  .action-list li::before {
    content: "→";
    color: #888;
    font-weight: 600;
    min-width: 16px;
  }

  /* ── Confidence badge ── */
  .confidence {
    display: inline-flex;
    align-items: center;
    gap: 8px;
    font-family: -apple-system, "PingFang SC", sans-serif;
    font-size: 13px;
    padding: 6px 14px;
    border-radius: 20px;
    font-weight: 500;
  }

  .confidence.high   { background: #dcfce7; color: #16a34a; }
  .confidence.medium { background: #fef9c3; color: #ca8a04; }
  .confidence.low    { background: #fee2e2; color: #dc2626; }

  /* ── Traces accordion ── */
  .traces-section {
    margin-top: 80px;
    padding-top: 40px;
    border-top: 1px solid #e8e8e4;
  }

  details {
    margin-bottom: 12px;
    border: 1px solid #e8e8e4;
    border-radius: 8px;
    overflow: hidden;
  }

  summary {
    cursor: pointer;
    padding: 16px 20px;
    font-family: -apple-system, "PingFang SC", sans-serif;
    font-size: 14px;
    font-weight: 500;
    color: #333;
    list-style: none;
    display: flex;
    justify-content: space-between;
    align-items: center;
    background: #fff;
  }

  summary::-webkit-details-marker { display: none; }

  summary::after {
    content: "+";
    font-size: 18px;
    color: #999;
  }

  details[open] summary::after { content: "−"; }

  .trace-content {
    padding: 24px;
    font-size: 16px;
    line-height: 1.8;
    background: #fafaf8;
    border-top: 1px solid #e8e8e4;
  }

  /* ── Footer ── */
  .report-footer {
    margin-top: 80px;
    padding-top: 32px;
    border-top: 1px solid #e8e8e4;
    font-family: -apple-system, "PingFang SC", sans-serif;
    font-size: 12px;
    color: #aaa;
    display: flex;
    justify-content: space-between;
    flex-wrap: wrap;
    gap: 8px;
  }

  /* ── Prose ── */
  p { margin-bottom: 20px; }
  strong { font-weight: 600; }
  em { font-style: italic; }

  @media (max-width: 600px) {
    body { font-size: 18px; }
    h1.report-title { font-size: 28px; }
    .core-conclusion { font-size: 20px; }
  }
</style>
</head>
<body>
<div class="container">

  <header class="report-header">
    <div class="report-meta">
      <span class="meta-badge mode-[MODE_CLASS]">[MODE_ZH]</span>
      <span class="meta-badge">K=[N] Agents</span>
      <span class="meta-badge">[ROLES_LIST]</span>
      <span>[DATETIME]</span>
    </div>
    <h1 class="report-title">[QUESTION_TITLE]</h1>
    <p class="report-subtitle">[QUESTION_FULL_OR_SUMMARY]</p>
  </header>

  <section class="section">
    <h2>核心结论</h2>
    <div class="core-conclusion">
      [CORE_CONCLUSION_TEXT]
    </div>
  </section>

  <section class="section">
    <h2>关键论点</h2>
    <ul class="key-points">
      <!-- 每条论点一个 <li> -->
      [KEY_POINTS_LI]
    </ul>
  </section>

  [INSIGHT_BOX_IF_EXISTS]
  <!-- 如有反直觉洞察：
  <div class="insight-box">
    <div class="insight-label">反直觉洞察</div>
    <p>[INSIGHT_TEXT]</p>
  </div>
  -->

  <section class="section">
    <h2>视角共识</h2>
    [CONSENSUS_BLOCKS]
    <!-- <div class="consensus-block"><p>[TEXT]</p></div> -->
  </section>

  <section class="section">
    <h2>视角分歧</h2>
    [DIVERGENCE_BLOCKS]
    <!-- <div class="divergence-block"><p>[TEXT]</p></div> -->
  </section>

  [ACTION_SECTION_IF_EXISTS]
  <!-- 如有行动建议：
  <section class="section">
    <h2>行动建议</h2>
    <ul class="action-list">
      [ACTION_LI]
    </ul>
  </section>
  -->

  <section class="section">
    <h2>置信度</h2>
    <span class="confidence [CONFIDENCE_CLASS]">[CONFIDENCE_LABEL]</span>
    <p style="margin-top:16px; font-size:16px; color:#555;">[CONFIDENCE_REASON]</p>
  </section>

  <!-- 可展开的各 Agent Trace -->
  <div class="traces-section">
    <h2>各 Agent 独立推理 Trace</h2>
    [TRACE_DETAILS_BLOCKS]
    <!--
    <details>
      <summary>Trace 1 — [ROLE]</summary>
      <div class="trace-content">[TRACE_CONTENT_AS_HTML]</div>
    </details>
    -->
  </div>

  <footer class="report-footer">
    <span>Heavy Think Report · [DATE]</span>
    <span>方法论：arXiv:2605.02396 · [N] agents + deliberation synthesis</span>
  </footer>

</div>
</body>
</html>
```

生成 HTML 时，将 deliberation 结论里的各部分映射到对应的 HTML 块。Markdown 转 HTML：`**text**` → `<strong>text</strong>`，段落加 `<p>` 标签，换行保留。

---

### Step 6：呈现结果

在对话里直接输出 deliberation 合成的完整内容（不做二次改写，原文呈现）。

同时告知用户报告文件位置：
```
📁 报告已保存至：~/.heavy-think/[date]-[slug]/
   ├── traces/          各 agent 独立推理 trace
   ├── deliberation.md  deliberation 原始输出
   ├── report.md        最终报告（Markdown）
   └── report.html      最终报告（HTML，可在浏览器打开）
```

---

### Step 7（可选）：低置信度迭代

如果置信度评估为**低**，主动告知用户，说明具体缺少什么信息，询问是否补充后重新运行一轮。

**最多进行 2 轮**（含第一轮）。第 2 轮之后即使置信度仍低也停止，因为研究表明更多轮次会导致质量退化。

---

## 写作风格要求

所有输出（trace、deliberation、报告）遵循以下原则：

- **短段落**：每段不超过 4 句话，段落之间留白
- **关键洞察独立成行**：重要结论加粗，前后空行
- **对话感**：避免"让我们来分析"、"综上所述"、"值得注意的是"等学术腔套话
- **直接表态**：不要用"可能"、"或许"来回避判断，有洞察就说出来
- Em dash 全文不超过 3 个

---

## 注意事项

- **真正并行**：K 个 subagent 必须在同一条消息里一次性 dispatch，不允许顺序启动
- **Shuffle 必做**：deliberation 前打乱 trace 顺序，减少位置偏差
- **不改写 deliberation**：synthesis subagent 的输出原文进报告，不做二次润色
- **语言跟随问题**：用户用中文提问就输出中文，英文同理
- **不滥用**：简单事实查询、有明确唯一答案的问题不需要用这个 skill
