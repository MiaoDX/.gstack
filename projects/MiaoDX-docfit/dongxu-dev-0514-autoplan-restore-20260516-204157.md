# /autoplan Restore Point
Captured: 2026-05-16T20:41:57+08:00 | Branch: dongxu/dev-0514 | Commit: a7e5b41

## Re-run Instructions
1. Copy "Original Plan State" below back to your plan file
2. Invoke /autoplan docs/decisions/0012-review-evaluation-architecture.md

## Original Plan State
# 0012 · Review 与 Evaluation 架构

- **Status**: Accepted
- **Date**: 2026-05-16
- **Consolidates**: [0006](0006-uncertainty-ledger.md), [0007](0007-evaluator-scopes.md), [0008](0008-review-gates-and-status.md), [0009](0009-advisor-and-promotion-boundaries.md), [0011](0011-evaluation-refactor-sequence.md)

## Context

docfit 的默认转换必须保持确定、可复现：学生 DOCX 被理解成 `ThesisModel`，再渲染进学校工程模板，产出可审阅的 Word 和报告。真实文档仍然会出现低置信度标题、未知样式、空槽位、manual-only requirement、LLM-assisted requirement 等情况。

这些情况不能被隐藏成日志，也不能直接让 AI 改 DOCX。系统需要一个清晰的 review/evaluation 架构：既能让用户拿到产物继续审阅，又能让 strict/CI/批量质检保持严格，还能让重复问题通过 Promotion 变成确定性规则。

## Decision

Review/evaluation 架构采用以下边界：

- **Uncertainty Ledger** 是一等证据产物。它汇总低置信度、`needs_review`、空槽位、manual/AI review blocker 等信号，供报告、review DOCX、status、Evaluator 和可选 Advisor 消费。第一阶段只增加 schema/report，不改变最终 DOCX 输出。
- **Evaluator Finding 必须有 scope**。Evaluator 不是单一最终文件检查器；finding 应归到 `model`、`render` 或 `requirement` scope，分别对应源文档理解、模板渲染、学校要求满足情况。
- **Artifact Status 和 Quality Status 分开**。`done` 只表示产物生成完成，不表示合规通过。默认转换中，artifact done + quality `needs_human` 允许下载、付款和继续使用，但 UI/API 必须明确标记需要人工确认。strict、CI、批量质检和学校验证中，`needs_human` 是 blocking。
- **Advisor 只读 scoped evidence**。Advisor 可以读取 selected uncertainty、相关段落片段、候选标签、Evaluator Finding 和必要上下文摘要；不读完整学生文档，不拥有转换决策，不直接改最终 DOCX。
- **Promotion 才能改变规则或自动行为**。同类 Advisor suggestion 或 Evaluator Finding 只有在真实样本中反复出现，并具备可复现样本、明确 scope、确定性实现方案和回归/golden/evaluator 覆盖后，才能提升为确定性规则、school-specific evaluator check、模板修正或 fixture。

## Implementation Order

按以下顺序迁移，不按旧 ADR 数量拆任务：

1. 稳住 **School Pack baseline**：确认 school pack 的 schema、requirements、checks、template manifest、template slots 和 coverage 能独立验证。
2. 实现 **Uncertainty Ledger schema/report**：从现有 `ReportItem`、低置信度 detection、空 slot、manual/LLM review blocker 桥接出 ledger；不改变 DOCX 输出、不改变默认 LLM 行为。
3. 从第一天开始给 ledger/finding 带 **scope**：至少区分 `model`、`render`、`requirement`。
4. 实现 **Quality Status aggregation**：在 CLI/report 层计算 artifact status 与 quality status。默认转换允许 artifact done + quality `needs_human`；strict/CI blocking。
5. 拆分 **Web/API status**：数据库、API 和 UI 不再只用 `done/failed` 表示最终质量；下载、付款、继续使用可以允许，但必须清楚展示 `needs_human`。
6. 迁移 **Advisor** 到 ledger/finding 后面：现有低置信度 hierarchy advisor fallback 应迁成 advisor suggestion / evaluator finding，或明确保留为 internal/beta legacy mode。
7. 最后统一 **Evaluator Finding**：逐步收敛 `ReportItem`、`ComplianceReviewStage` result、review annotation 和 advisor suggestion，统一包含 scope、decision、severity、evidence、location 和 promotion hint。

## Consequences

### 好处

- 默认转换仍然确定、可回归，同时不确定性变成可审阅证据。
- 用户能拿到产物，但不会把 `needs_human` 误解成合规通过。
- Web/API 可以诚实表达“生成完成”和“质量待确认”的差异。
- Advisor 有稳定输入面，不需要重新理解 pipeline 内部状态。
- Promotion 有证据链，可以持续把重复问题沉淀成确定性规则。

### 坏处

- 短期会同时存在旧 `ReportItem` / `ComplianceReviewStage` 结果和新 ledger/finding，需要桥接。
- 状态模型比单一 `done/failed` 更复杂。
- Advisor migration 需要处理旧 `--llm` hierarchy fallback，避免它继续直接改变转换决策。

### 在什么 context 下应该重新考虑

- 如果产品形态改成全人工代办，不再需要自动转换加可审阅报告。
- 如果决定短期完全不做 Web/API 产品化，只保留 CLI 研究工具。
- 如果真实样本显示 uncertainty 几乎没有价值，无法支撑 quality status 或 promotion。
