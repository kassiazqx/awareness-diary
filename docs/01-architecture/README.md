# Architecture Rules

这是架构铁律总入口。任何 AI 或开发者开始开发前必须先读本文件。

## 最高优先级规则

1. UI 不直接访问数据库、SQLite、文件系统、localStorage 或云服务。
2. 页面只负责组织交互和展示，不拥有复杂业务流程。
3. 业务流程统一放在 `src/lib/services/`。
4. 数据访问统一放在 `src/lib/data/`。
5. 外部依赖必须经过 adapter / client 封装，不能散落在页面里。
6. 核心数据使用稳定 ID，优先 UUID；必须有 `created_at`、`updated_at`。
7. 为未来同步预留软删除字段或删除日志，不默认物理删除即结束。
8. 每个字段只能有一个真源；派生字段必须由统一 service 生成。
9. `content_json` 是富文本真源；`content_plain` 是检索和 AI 用派生文本。
10. schema 里任何未启用能力必须登记状态：已启用 / 停放中 / 已退役。
11. 删除操作必须先确认所有关联关系、级联策略、导入导出影响和同步影响。
12. 同一业务对象新增第二个入口前，必须先对照旧入口，收口到同一个 service。
13. 默认数据初始化只能有一个 owner，不能页面、service、DB trigger 多处同时补写。
14. 过渡兼容逻辑必须写清退出条件、owner、复查时间。
15. 小改动写清 commit；用户可感知变化写入根目录 `CHANGELOG.md`。

## 按需细读

- 分层边界：`layering-rules.md`
- 数据真源：`data-source-rules.md`
- 删除流程：`deletion-rules.md`
- 同步预留：`sync-readiness.md`
- 旧项目教训：`lessons-learned.md`
