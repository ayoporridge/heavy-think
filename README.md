# heavy-think

> 多 Agent 并行深度分析 Claude Code Skill

本 skill 改造自：[joeseesun/qiaomu-heavyskill](https://github.com/joeseesun/qiaomu-heavyskill)

---

## 是什么

对同一问题从多个独立视角并行推理，经 deliberation 合成出超越任何单个 agent 的结论，输出 Markdown + HTML 报告。

方法论依据：[arXiv:2605.02396](https://arxiv.org/abs/2605.02396) — 多个隔离 agent 独立推理后 deliberation 合成（HP@K），常常超过所有单个 trace，因为 **deliberation 是生成性的**。

---

## 适用场景

- 技术决策、架构选型
- 复杂分析、有深度的判断类问题
- 产品决策、战略选择
- 需要充分覆盖对立面的高风险决策

**不适用**：简单事实查询、有明确唯一答案的问题。

---

## 使用方法

安装：将 `SKILL.md` 放置到 `~/.claude/skills/heavy-think/SKILL.md`

触发词：`heavy-think`、`/heavy-think`、`多角度分析`、`深度分析`、`多agent分析`

---

## 核心机制

| 步骤 | 内容 |
|------|------|
| Step 0 | 分析问题，判断 Verification / Deliberation 模式 |
| Step 1 | 确定 K 值（3-5）与视角分配 |
| Step 2 | 创建输出目录 `~/.heavy-think/[date]-[slug]/` |
| Step 3 | 并行启动 K 个 isolated subagent |
| Step 4 | Shuffle 后 deliberation 合成 |
| Step 5 | 生成 `report.md` + `report.html`（Medium 风格排版） |
| Step 6 | 对话输出结果 |
| Step 7 | 低置信度时可选第二轮（最多 2 轮）|

---

## 改造说明

在原版基础上的主要调整：

- 补充了信源可信度限制（学术/权威媒体/官方/国内可信媒体，禁止个人博客和无出处数据）
- 细化了 HTML 报告排版（Medium 风格，带 traces accordion）
- 明确了写作风格要求（短段落、直接表态、避免学术腔）
- 完善了置信度评估与低置信度迭代协议

---

## 输出结构

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

---

*改造自 [joeseesun/qiaomu-heavyskill](https://github.com/joeseesun/qiaomu-heavyskill)*
