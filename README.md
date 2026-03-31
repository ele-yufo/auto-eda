# Auto EDA

> Subagent-Driven 数据分析团队——让 AI Agent 像老练的数据科学家一样工作。

Auto EDA 是一个 **Claude Code 插件**，通过 subagent 架构 + 模块化知识库 + 强制性行为规范，将 Claude 转化为一个具备数据直觉、统计严谨性和业务表达力的分析团队。

## 核心架构

```
┌─────────────────────────────────────────┐
│          主 Agent (CLAUDE.md)            │
│  访谈 → 分析 → 叙事 → Dashboard → 交付   │
│  身份：三面孔 + 五习惯 (SKILL.md)         │
└─────────┬───────────────┬───────────────┘
          │               │
    ┌─────▼─────┐   ┌─────▼─────┐
    │Data Scout │   │ Reviewer  │
    │ (sonnet)  │   │ (sonnet)  │
    │快速画像    │   │质量审查    │
    │PII扫描    │   │偏见检查    │
    │多文件检测  │   │交付审查    │
    └───────────┘   └───────────┘
```

**为什么只有 2 个 subagent？** 数据分析高度依赖上下文——分析和叙事由主 Agent 直接执行以保持完整推理链。只有任务边界清晰、输入输出结构化的工作（数据侦察、质量审查）才委托给独立 subagent。

## 工作流

```
用户上传数据
  → [Scout/sonnet] 快速画像 + PII 扫描 + 多文件检测 (≈30秒)
  → [主 Agent] 展示发现 → 访谈对齐 → 深度分析（每步检查偏见）
  → [Reviewer/sonnet] 分析质量审查
  → [主 Agent] 生成 Dashboard + 行动建议
  → [Reviewer/sonnet] 交付质量审查（PII + 一致性）
  → [主 Agent] Phase 3 交付导览 → 收尾
```

## 团队角色

| 角色 | 模型 | 职责 |
|------|------|------|
| **主 Agent** | opus | 访谈、深度分析、叙事、Dashboard 生成、交付 |
| **Data Scout** | sonnet | 数据画像、编码检测（含回退）、PII 扫描、多文件 JOIN 检测、质量预警 |
| **Reviewer** | sonnet | 偏见检查完整性、信心等级合理性、结论过度声称、PII 泄露、交付一致性 |

## 核心行为规范 (SKILL.md)

**三个面孔**：数据科学家（选对方法）× 业务顾问（讲业务语言）× 数据怀疑者（每步检查偏见）

**五个不变的习惯**：
1. 先看数据，后谈需求（用发现来开场）
2. 每分析一步，都检查偏见（速查清单，不是可选项）
3. 用故事讲结论，不用统计量（SCQA / 三幕式 / AIDA）
4. 代码就是分析（实时编写，不用预制脚本）
5. 交付不只是图表（Dashboard + 行动建议 + 偏见声明）

## 项目结构

```
auto-eda/
├── CLAUDE.md                          # 总指挥：工作流编排 + subagent 调度协议
├── SKILL.md                           # 核心行为规范：三面孔 + 五习惯 + 信心等级
├── agents/
│   ├── data-scout.md                  # 数据侦察兵 (sonnet)
│   └── reviewer.md                    # 质量审查员 (sonnet)
├── references/                        # 模块化知识库（按需加载）
│   ├── analysis-playbook.md           #   分析路径决策树 + 5个标准场景
│   ├── bias-and-pitfalls.md           #   偏见百科 + 速查清单 + 数据伦理
│   ├── business-frameworks.md         #   RFM / AARRR / CLV / 归因 / 价格弹性
│   ├── data-ingestion.md              #   数据接入陷阱
│   ├── data-preparation.md            #   数据清洗 / 特征工程 / 多表关联
│   ├── data-storytelling.md           #   SCQA / 三幕式 / AIDA / 无发现沟通
│   ├── interview-protocol.md          #   三阶段访谈：开场→分析→交付
│   ├── statistical-methods.md         #   假设检验 / 因果推断 / SHAP / Bootstrap / 贝叶斯 / 生存分析
│   ├── timeseries-methods.md          #   时序分解 / Prophet / 异常检测
│   └── visualization-guide.md         #   图表选择 / 配色 / 样式配置
├── templates/
│   └── dashboard.html                 #   Dashboard 结构骨架
└── examples/
    └── example-dashboard.html         #   完整渲染的示例（防止幻觉的具体参考）
```

## 差异化优势

| 特性 | Auto EDA | 传统 EDA 工具 (ydata-profiling, Sweetviz) | AI 分析平台 (PandasAI, DATAGEN) |
|------|----------|------|------|
| 偏见检查系统 | 速查清单 + 每步强制 | 无 | 基本无 |
| 业务语言翻译 | 强制翻译，禁止统计术语 | 纯统计输出 | 部分 |
| 对话式分析 | 访谈三阶段 + 中途对齐 | 一键报告 | 简单 Q&A |
| 质量审查 | 独立 Reviewer subagent | 无 | 基本无 |
| PII 检测 | Scout 自动扫描 + Reviewer 复查 | 无 | 部分 |
| 多文件关联 | 自动 JOIN 键检测 | 不支持 | 部分 |
| 跨会话继承 | _eda_state.md 状态持久化 | 不支持 | 部分 |
| 成本 | Claude Code 内置 | 免费 | 需 API 费用 |

## 灵感来源

- [superpowers](https://github.com/obra/superpowers) — Subagent-driven development 方法论
- [ydata-profiling](https://github.com/Data-Centric-AI-Community/ydata-profiling) — 数据质量预警框架
- [ai-analyst](https://github.com/ai-analyst-lab/ai-analyst) — Claude Code 原生分析 Agent 架构
- [Sweetviz](https://github.com/fbdesignpro/sweetviz) — 数据集对比分析

## 使用方法

将此仓库克隆到本地，在 Claude Code 中作为 Skill 目录配置。Agent 会在用户上传数据或请求分析时自动激活。

## 许可

MIT
