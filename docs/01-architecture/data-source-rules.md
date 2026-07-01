# Data Source Rules

## 单一真源

每个字段必须说明谁是真源。派生字段只能由统一 service 维护。

## 富文本

- `content_json`：唯一富文本真源。
- `content_plain`：从 `content_json` 生成，用于搜索和 AI 输入。
- 渲染 HTML：运行时从 `content_json` 生成，不入库。

禁止页面分别手写 `content_json` 和 `content_plain`。

## 设置

同一类设置只能有一个真源。阶段 1 可以存在本地设置；进入同步阶段前必须设计迁移路径，不能 DB 和本地存储都可写。
