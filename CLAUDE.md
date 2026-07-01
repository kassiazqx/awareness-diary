# CLAUDE.md

本文件是 Claude Code 在本项目的协作入口。若用户当轮明确指令与本文冲突，以用户当轮指令为准。

## 启动读取顺序

开始任何任务前，先读：

1. `docs/00-overview/ai-start-here.md`
2. `docs/01-architecture/README.md`

按任务类型继续读取：

- 写代码 / 重构 / 调试：读 `docs/04-guides/workflow.md`、`docs/04-guides/testing.md`
- 改 schema / 新增字段 / 新增枚举 / 新增外键：读 `docs/03-schema/schema-register.md`、`docs/01-architecture/sync-readiness.md`
- 涉及删除：读 `docs/01-architecture/deletion-rules.md`
- 涉及本地存储 / 数据真源 / 同步：读 `docs/01-architecture/data-source-rules.md`
- 涉及页面、service、data 边界：读 `docs/01-architecture/layering-rules.md`
- 涉及旧项目经验复用：读 `docs/01-architecture/lessons-learned.md`

## 文档写入规则

- 新功能设计：写入 `docs/02-design/YYYY-MM-DD-feature-name/spec.md`
- 新功能计划：写入 `docs/02-design/YYYY-MM-DD-feature-name/plan.md`
- 关键设计决策：写入 `docs/02-design/YYYY-MM-DD-feature-name/decisions.md`
- schema 变更：同步更新 `docs/03-schema/schema-register.md` 和 `docs/03-schema/changelog.md`
- 用户可感知的小变更：记录到根目录 `CHANGELOG.md`

## 架构底线

- UI 不直接访问数据库或本地存储实现。
- 页面只组织交互，不拥有业务流程细节。
- 业务流程收口到 `src/lib/services/`。
- 数据访问收口到 `src/lib/data/`。
- 所有核心数据必须有稳定 ID、created_at、updated_at；为未来同步保留软删除和冲突处理空间。
- `content_json` 是富文本真源；`content_plain` 只能是派生字段，由统一 service 生成和维护。
- 未来能力先登记，后进入 schema；没有 owner 和当前入口的能力不得伪装成已启用。
