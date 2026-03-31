# Auto EDA — Subagent-Driven 数据分析团队

> 本插件提供数据分析能力。**仅在用户需要数据分析时激活**，不影响其他工作流。

## 激活条件

当以下条件**同时满足**时，读取 `skills/auto-eda/SKILL.md` 并按其工作流执行：

- 用户上传了数据文件（CSV/Excel/PDF）或提供了数据源
- 上下文表明用户想要**分析数据**（而非仅读取/转换/迁移）

**不激活**（遇到以下场景时忽略本插件，使用其他工作流）：
- 纯代码开发任务
- 数据工程 / ETL 管道搭建
- 数据库管理和运维
- 纯可视化请求（无分析意图）
- ML 模型部署/运维
- 邮件、文档或其他非数据分析任务

## 激活后的行为

1. 读取 `skills/auto-eda/SKILL.md`（核心行为规范 + 工作流编排 + 参考文档索引）
2. 按 SKILL.md 定义的工作流执行（侦察 → 访谈 → 分析 → 审查 → 交付）
3. 调度 subagent 时参考 `skills/auto-eda/agents/` 下的角色定义

## 团队架构

```
主 Agent (opus) — 访谈、分析、叙事、Dashboard、交付
  ├── Data Scout (sonnet) — 快速画像、PII 扫描、多文件检测
  └── Reviewer (sonnet) — 偏见检查、质量审查、交付审查
```

详细的工作流编排、参考文档索引和行为规范均在 `skills/auto-eda/SKILL.md` 中。
