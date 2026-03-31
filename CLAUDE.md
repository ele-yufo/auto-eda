# Auto EDA — Subagent-Driven 数据分析团队

> 本文件是 Auto EDA 的总指挥。它定义工作流编排、subagent 调度协议和访谈交互。
> 主 Agent 直接执行分析、叙事和交付——只有数据侦察和质量审查委托给独立 subagent。

## 触发条件

当以下条件**同时满足**时激活此 Skill：
- 用户上传了数据文件（CSV/Excel/PDF）或提供了数据源
- 上下文表明用户想要分析数据（而非仅仅读取/转换/迁移数据）

**不触发**：纯数据工程任务（ETL 管道搭建）、数据库管理、纯可视化请求（无分析意图）、ML 模型部署/运维。

---

## 团队架构

```
┌─────────────────────────────────────────┐
│            主 Agent (opus)               │
│  职责：访谈、分析、叙事、Dashboard、交付    │
│  身份：SKILL.md 定义的三面孔 + 五习惯      │
└─────────┬───────────────┬───────────────┘
          │               │
    ┌─────▼─────┐   ┌─────▼─────┐
    │Data Scout │   │ Reviewer  │
    │ (sonnet)  │   │ (sonnet)  │
    │ 数据侦察  │   │ 质量审查   │
    └───────────┘   └───────────┘
```

**为什么只有 2 个 subagent？**

数据分析是高度上下文依赖的——分析师需要理解数据的每个细节，叙事者需要理解分析的推理过程。如果把这些拆成独立 subagent，每次传递都会丢失隐式知识。

但 **Scout** 和 **Reviewer** 的任务边界清晰、输入输出结构化，适合独立执行：
- Scout：输入是原始文件 → 输出是结构化画像 JSON + PII 警告
- Reviewer：输入是分析发现 + Dashboard → 输出是审查报告（通过/需修改）

---

## 工作流编排

### Phase 0：环境准备（主 Agent 直接执行）

按 SKILL.md 的环境准备协议，静默检查 Python 和依赖。

### Phase 1：数据侦察（调度 Scout subagent）

```
主 Agent 调度 Data Scout (sonnet)：
  "对以下数据文件进行快速画像。参考 agents/data-scout.md 的完整指引。
   文件路径：{paths}
   请输出：_eda_data_profile.json + 初步发现 + PII 警告 + 多文件关联建议"
```

Scout 返回后，主 Agent：
1. 读取 `_eda_data_profile.json`
2. 如果有 PII 警告 → 立即告知用户，建议脱敏
3. 如果有多文件 → 展示 Scout 的关联建议，与用户确认 JOIN 策略
4. 用 Scout 发现的 2-3 个有趣点作为 Hook 开场

### Phase 2：访谈对齐（主 Agent 直接执行）

参考 `skills/auto-eda/references/interview-protocol.md` Phase 1：
- 展示 Scout 发现的惊喜点 → 开放式引导 → 对齐分析目标
- 用"五个为什么"深挖需求
- 确认对齐检查点（业务背景、分析类型、关键决策、成功标准、受众）

### Phase 3：深度分析（主 Agent 直接执行）

这是核心价值所在，主 Agent 保持完整上下文：

1. **参考 `skills/auto-eda/references/analysis-playbook.md`** 选择分析路径
2. **实时编写代码** 执行分析（Habit 4：代码就是分析）
3. **每步检查偏见** 参考 `skills/auto-eda/references/bias-and-pitfalls.md` 速查清单
4. **中途对齐** 每发现重要 insight 与用户同步（参考 interview-protocol Phase 2）
5. **按需参考** 其他 references（统计方法、时序分析、业务框架等）

**预算感知**：如果数据量大或分析复杂度高，主动评估分析范围：
- 大数据集（> 10 万行）→ 先采样分析，确认方向后再全量
- 列数多（> 30 列）→ 先做相关性筛选，优先分析高价值维度
- 分析进行中如果发现范围膨胀 → 主动提议收尾或聚焦

### Phase 4：质量审查（调度 Reviewer subagent）

在生成 Dashboard 之前，调度 Reviewer 审查分析质量：

```
主 Agent 调度 Reviewer (sonnet)：
  "审查以下分析发现的质量。参考 agents/reviewer.md 的完整指引。
   发现列表：{findings_summary}
   数据画像：{data_profile_summary}
   请输出：审查报告（每个发现的偏见检查完整性、信心等级合理性、结论是否过度声称）"
```

Reviewer 返回后：
- 如果有阻塞性问题 → 修正后重新提交审查
- 如果有建议性修改 → 酌情采纳
- 审查通过 → 进入 Dashboard 生成

### Phase 5：Dashboard 生成（主 Agent 直接执行）

主 Agent 根据分析结果生成 Dashboard：

1. 参考 SKILL.md 前端美学指南确定设计方向
2. 参考 `skills/auto-eda/templates/dashboard.html` 的结构骨架
3. 参考 `skills/auto-eda/examples/example-dashboard.html` 的具体渲染效果
4. 生成 `report.html` + 更新 `_eda_findings.md`

**图表策略**：
- 默认用 matplotlib/seaborn 生成静态 PNG → base64 内联
- 如果用户需要交互式探索 → 使用 Plotly 生成交互式 HTML 图表内联嵌入
- 控制图表数量（≤ 10 张），超出时优先展示最高信心等级的发现

### Phase 6：交付审查（调度 Reviewer subagent）

Dashboard 生成后，调度 Reviewer 做交付审查：

```
主 Agent 调度 Reviewer (sonnet)：
  "审查以下 Dashboard 的交付质量。参考 agents/reviewer.md 的完整指引。
   Dashboard 文件：report.html
   请检查：PII 泄露、结论准确性、信心等级标注、偏见声明完整性"
```

### Phase 7：交付与收尾（主 Agent 直接执行）

参考 `skills/auto-eda/references/interview-protocol.md` Phase 3：
- Dashboard 导览（核心数字 + 关键发现 + 局限 + Quick Win）
- 收集反馈、处理修改请求
- 更新 `_eda_state.md`
- 后续分析方向建议

---

## Subagent 调度协议

### 调度原则

1. **只调度 Scout 和 Reviewer**，其他工作主 Agent 直接做
2. **Scout 在分析开始前调度一次**，除非用户上传新数据
3. **Reviewer 在 Dashboard 生成前后各调度一次**（分析审查 + 交付审查）
4. **subagent 模型统一用 sonnet**——数据画像和质量审查的准确性优先于速度

### 调度方式

使用 Agent 工具调度 subagent，在 prompt 中包含：
- 指向 `skills/auto-eda/agents/data-scout.md` 或 `skills/auto-eda/agents/reviewer.md` 的完整指引
- 需要处理的数据/内容
- 期望的输出格式

### 失败处理

- subagent 执行失败 → 主 Agent 直接执行该任务（降级但不中断）
- subagent 返回不完整结果 → 主 Agent 补充完成

---

## 核心参考文档

主 Agent 的行为规范、三个面孔、五个习惯、信心等级定义 → 参见 `skills/auto-eda/SKILL.md`。

参考文档按需加载 → 参见 SKILL.md 底部的参考文档索引。

---

## 跨会话继承

复杂分析跨越多个会话时，遵循 SKILL.md 的分析遗产协议：
1. 启动时检查 `_eda_state.md` 是否存在
2. 存在 → 读取状态 + 发现，从"下一步建议"继续
3. 不存在 → 全新分析，调度 Scout 开始
