# Auto EDA

> Subagent-Driven 数据分析团队——让 AI Agent 像老练的数据科学家一样工作。

Auto EDA 是一个 **Claude Code 插件**，通过 subagent 架构 + 模块化知识库 + 强制性行为规范，将 Claude 转化为一个具备数据直觉、统计严谨性和业务表达力的分析团队。

## 安装

### 方式 1：Claude Code 插件安装（推荐）

```bash
# 在 Claude Code 中执行
/plugin marketplace add ele-yufo/auto-eda
/plugin install auto-eda
/exit  # 重启生效
```

### 方式 2：手动安装

```bash
# 克隆到你的项目目录
git clone https://github.com/ele-yufo/auto-eda.git .auto-eda

# 或作为 git submodule
git submodule add https://github.com/ele-yufo/auto-eda.git .auto-eda
```

安装后，当你上传数据文件或请求分析时，插件会自动激活。

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

**为什么只有 2 个 subagent？** 数据分析高度依赖上下文——分析和叙事由主 Agent 直接执行以保持完整推理链。只有任务边界清晰、输入输出结构化的工作才委托给独立 subagent。

## 工作流

```
用户上传数据
  → [Scout/sonnet] 快速画像 + PII 扫描 + 多文件检测
  → [主 Agent] 展示发现 → 访谈对齐 → 深度分析（每步检查偏见）
  → [Reviewer/sonnet] 分析质量审查
  → [主 Agent] 生成 Dashboard + 行动建议
  → [Reviewer/sonnet] 交付质量审查（PII + 一致性）
  → [主 Agent] 交付导览 → 收尾
```

## 团队角色

| 角色 | 模型 | 职责 |
|------|------|------|
| **主 Agent** | opus | 访谈、深度分析、叙事、Dashboard 生成、交付 |
| **Data Scout** | sonnet | 数据画像、编码检测、PII 扫描、多文件 JOIN 检测、质量预警 |
| **Reviewer** | sonnet | 偏见检查完整性、信心等级验证、PII 泄露检查、交付一致性 |

## 项目结构

```
auto-eda/
├── .claude-plugin/
│   └── marketplace.json               # 插件元数据
├── CLAUDE.md                           # 总指挥：工作流编排 + subagent 调度
├── hooks/
│   └── session-start                   # 会话启动时检测未完成分析
├── skills/
│   └── auto-eda/
│       ├── SKILL.md                    # 核心行为规范
│       ├── agents/
│       │   ├── data-scout.md           # 数据侦察兵
│       │   └── reviewer.md            # 质量审查员
│       ├── references/                 # 10 个模块化知识文档
│       │   ├── analysis-playbook.md
│       │   ├── bias-and-pitfalls.md
│       │   ├── business-frameworks.md
│       │   ├── data-ingestion.md
│       │   ├── data-preparation.md
│       │   ├── data-storytelling.md
│       │   ├── interview-protocol.md
│       │   ├── statistical-methods.md
│       │   ├── timeseries-methods.md
│       │   └── visualization-guide.md
│       ├── templates/
│       │   └── dashboard.html          # Dashboard 结构骨架
│       └── examples/
│           └── example-dashboard.html  # 完整渲染示例
├── package.json
├── README.md
└── LICENSE
```

## 核心行为规范

**三个面孔**：数据科学家 × 业务顾问 × 数据怀疑者

**五个不变的习惯**：
1. 先看数据，后谈需求
2. 每分析一步，都检查偏见
3. 用故事讲结论，不用统计量
4. 代码就是分析
5. 交付不只是图表

**信心等级**：
- 🟢 强证据 — p<0.05 + 效应量>小 + n>最小样本量 + 偏见检查通过
- 🟡 弱暗示 — p<0.10 或样本边缘或存在混杂
- 🔴 不确定 — p>0.10 或 n<30 或被其他分析矛盾

## 差异化优势

| 特性 | Auto EDA | 传统 EDA 工具 | AI 分析平台 |
|------|----------|-------------|-----------|
| 偏见检查系统 | 速查清单 + 每步强制 | 无 | 基本无 |
| 业务语言翻译 | 强制翻译 | 纯统计输出 | 部分 |
| 质量审查 | 独立 Reviewer subagent | 无 | 基本无 |
| PII 检测 | Scout 扫描 + Reviewer 复查 | 无 | 部分 |
| 多文件关联 | 自动 JOIN 键检测 | 不支持 | 部分 |
| 跨会话继承 | _eda_state.md 持久化 | 不支持 | 部分 |

## 灵感来源

- [superpowers](https://github.com/obra/superpowers) — Subagent-driven development 方法论与插件架构
- [ydata-profiling](https://github.com/Data-Centric-AI-Community/ydata-profiling) — 数据质量预警框架
- [ai-analyst](https://github.com/ai-analyst-lab/ai-analyst) — Claude Code 原生分析 Agent 架构

## 许可

MIT
