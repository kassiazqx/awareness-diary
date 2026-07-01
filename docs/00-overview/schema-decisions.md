# Schema Decisions 中文版

> 读者：后续参与本项目的 AI 编码代理和维护者。  
> 目的：记录当前已经确认的数据库设计口径，以及仍需继续讨论的风险点。  
> 状态：设计合约，不是最终 SQL migration。后续写 SQL 前必须再转成正式 schema 文档和 migration。

## 1. 文档语言规则

本项目的权威设计文档可以使用中文。AI 执行准确度主要取决于定义是否精确，而不是英文或中文。

要求：

- 概念必须有明确含义，不能只用个人化比喻。
- 每张表必须说明：用途、字段、关系、禁止事项。
- 每个“暂不做”的能力必须说明：是不采用，还是未来可能新增。
- 如果某个设计仍有争议，必须标为“待讨论”，不能写成已确认。

## 2. 全局数据库规则

### 规则 1：只有独立对象才拆成独立表

定义：如果某类数据需要被单独创建、修改、删除、查询、统计、同步，或有自己的历史记录，它就是独立对象，应该有独立表。

正确例子：

- `todos` 存 Todo 定义，例如“八段锦”。
- `todo_completions` 存每次完成记录，例如“2026-06-26 完成 1 次”。
- `entry_modules` 存一条 Entry 内的每个内容块，因为模块需要排序、删除、编辑。

错误例子：

- 把所有 Todo 完成日期塞进 `todos.completions_json`。
- 把一条 Entry 的所有模块都塞进 `entries.modules_json`，同时又希望模块能单独排序、删除、筛选。

原因：

- 独立对象如果塞进 JSON，后续查询、删除、导出、同步、冲突处理都会变复杂。

### 规则 2：正式关系必须放在关系表或外键中，不能藏进 JSON

定义：如果 A 对象和 B 对象之间的关系会被查询、删除、同步、筛选或展示，这个关系必须用外键或关系表表达。

正确例子：

```text
entry_threads(entry_id, thread_id)
entry_tags(entry_id, tag_id)
todo_tags(todo_id, tag_id)
```

未来如果 Todo 需要直接关联 Thread，应新增：

```text
todo_threads(todo_id, thread_id)
```

错误例子：

```json
{"thread_ids":["thread_a","thread_b"]}
```

放在 `todos.metadata_json` 或 `entry_modules.structured_data` 里。

原因：

- JSON 里的关系不容易建索引。
- 删除 Thread 时不容易找到所有引用。
- 多端同步时不容易判断关系变更。
- 导入导出时容易漏。

### 规则 3：内部 ID 和用户显示名称分离

定义：内部 ID 是系统身份，不给用户看；用户显示名称是内容，可以是中文，可以修改。

正确例子：

```text
threads.id = 8f3a0c2e-xxxx
threads.title = 我和妈妈的关系
```

错误例子：

```text
threads.id = 我和妈妈的关系
```

原因：

- 用户会改名。
- 改名不应该影响历史记录和关系表。
- UUID 不需要 AI 或拼音转换，避免歧义。

### 规则 4：同一个业务含义只能有一个真源

定义：同一件事只能有一个权威存储位置。其他位置可以是派生、缓存或展示，但不能同时当真源。

正确例子：

- Entry 和 Thread 的关系只由 `entry_threads` 负责。
- Todo 完成统计只由 `todo_completions` 负责。
- Todo 在日记里被提到，由 `todo_ref` 模块表达，但不等于完成。

错误例子：

- `entries.primary_thread_id` 和 `entry_threads` 同时都表示 Entry 属于哪个 Thread。
- `todo_completions` 记录一次完成，`entry_modules.structured_data.status = done` 又被统计成一次完成。

原因：

- 两个真源迟早不同步。
- 旧项目已经踩过“多路径导致刷新、删除、统计不一致”的坑。

### 规则 5：派生字段必须由统一 service 生成

定义：如果一个字段是从另一个字段计算出来的，它不能由页面手写，必须由统一 service 生成。

正确例子：

- 用户编辑 `content_json`。
- 保存时由 `entryModuleService` 或同类 service 生成 `content_plain`。

错误例子：

- 页面组件同时手动写 `content_json` 和 `content_plain`。

原因：

- 双写字段如果没有唯一 owner，内容会不一致。
- AI 输入和搜索依赖 `content_plain`，不能让它漂移。

### 规则 6：阶段一可以少实现，但不能选择阻塞未来的结构

定义：MVP 可以只做 Entry 和 Thread，但现在的 schema 方向不能让未来 Todo、Tag、Media 同步、回顾信、AI 分析难以加入。

正确例子：

- 暂不做 `todo_threads`，但不把 Todo-Thread 关系塞进 JSON。
- Media 保存 App 自己管理的 `local_path`，未来可增加 `remote_path`。

错误例子：

- 保存手机相册原路径，用户删相册后 App 图片丢失。
- 为了省事把未来同步状态写死在 UI 逻辑里。

原因：

- 本项目目标是本地优先，后续可同步。第一阶段不能把第三阶段堵死。

## 3. 表类型总览

| 类型 | 表 | 定义 |
|---|---|---|
| 核心对象表 | `entries`, `threads`, `todos` | 用户创建和长期维护的主要对象 |
| 子明细表 | `entry_modules`, `template_modules`, `todo_completions`, `media` | 依附于某个核心对象，但自身有独立行 |
| 字典/目录表 | `module_types`, `templates`, `tags` | 可复用定义或配置 |
| 关系表 | `entry_threads`, `entry_tags`, `thread_tags`, `todo_tags`, 未来可能的 `todo_threads` | 多对多关系 |
| AI/历史表 | `thread_analyses`, `review_letters`, `review_letter_threads` | AI 生成内容和历史快照 |
| 设置/安全存储 | settings service, secure storage | 普通偏好和敏感凭证 |

## 4. 当前已确认的表设计方向

### 4.1 `entries`

用途：一条日记记录的外壳。

建议字段：

| 字段 | 格式 | 备注 |
|---|---|---|
| `id` | UUID | 内部身份 |
| `entry_date` | datetime | 用户选择的日记日期，支持补记 |
| `template_origin_id` | UUID nullable | 从哪个模板开始写 |
| `created_at` | datetime | 创建时间 |
| `updated_at` | datetime | 更新时间 |
| `deleted_at` | datetime nullable | 为回收站/软删除预留 |

禁止：

- 如果 `entry_threads` 是 Entry-Thread 真源，就不要再加 `primary_thread_id` 当另一个真源。
- 不把所有模块内容直接塞进 `entries`。

### 4.2 `entry_modules`

用途：一条 Entry 里的一个内容块。

建议字段：

| 字段 | 格式 | 备注 |
|---|---|---|
| `id` | UUID | 内部身份 |
| `entry_id` | UUID FK | 所属 Entry |
| `module_type_key` | text FK | 模块类型，例如 `event`, `weather`, `emotion`, `todo_ref` |
| `sort_order` | integer | 模块在 Entry 内的顺序 |
| `content_json` | text/json nullable | 富文本真源 |
| `content_plain` | text nullable | 搜索和 AI 用派生文本 |
| `structured_data` | json nullable | 当前仍有争议，见第 5 节 |
| `created_at` | datetime | 创建时间 |
| `updated_at` | datetime | 更新时间 |

### 4.3 `module_types`

用途：定义系统支持哪些模块类型。

建议字段：

| 字段 | 格式 | 备注 |
|---|---|---|
| `key` | text PK | 例如 `event`, `weather`, `emotion`, `todo_ref` |
| `display_name` | text | 用户可见名称 |
| `data_kind` | text | `richtext`, `structured`, `mixed` |
| `default_hint` | text nullable | 默认提示语 |
| `schema_json` | json nullable | 如果使用 `structured_data`，用于定义结构和校验 |
| `is_system` | boolean | 是否系统内置 |

## 5. 重要待讨论：是否使用 `structured_data`

当前状态：**未最终确认，需要专门讨论。**

### 方案 A：使用 `structured_data`

例子：

```text
module_type_key = weather
structured_data = {"condition":"rainy","temp":25}
```

```text
module_type_key = emotion
structured_data = {"score":4,"tags":["焦虑","烦躁"]}
```

优点：

- 新增模块时不一定要改 `entry_modules` 表结构。
- 适合模块种类多、每类字段差异大的情况。
- 未来支持开发者新增模块更灵活。

风险：

- 如果需要频繁按 JSON 内字段筛选，例如情绪分数、天气、Todo 状态，查询会更复杂。
- 如果 JSON 里放了关系 ID，会违反“关系不能藏 JSON”的规则。
- 需要严格的 `module_types.schema_json` 和 service 校验，否则容易变成杂物箱。

### 方案 B：给常用结构化模块建专门字段或专门表

例子：

```text
entry_weather(entry_module_id, condition, temp)
entry_emotion(entry_module_id, score)
entry_todo_refs(entry_module_id, todo_id, status)
```

优点：

- 查询和筛选更清晰。
- 外键关系更明确。
- 删除和同步更容易验证。

缺点：

- 表更多。
- 新增模块类型可能要新增表或 migration。
- 自定义模块更难。

### 当前判断

`structured_data` 不是必然错误，但它是高风险设计点。后续应单独开 schema 讨论，决定：

1. 哪些模块可以用 JSON。
2. 哪些模块必须拆成专门表。
3. JSON 内是否允许存外键 ID。
4. 如何为 JSON 建校验和索引。

在最终确认前，AI 不得把 `structured_data` 当作已完全定稿的设计。

## 6. Templates

已确认：

- 系统提供默认模板。
- 用户可以创建、编辑自己的模板。
- 模板只用于写日记前拉取默认结构。
- 写某条 Entry 时调整模块，不影响模板。
- 修改模板，不影响历史 Entry。

表：

```text
templates
template_modules
```

关系：

```text
templates 1 -> N template_modules
module_types 1 -> N template_modules
```

规则：

- 创建 Entry 时，把 `template_modules` 复制成 `entry_modules`。
- 复制后，Entry 与模板脱钩，只保留 `template_origin_id` 用于记录来源。

## 7. Threads 与 Entries

已确认：

- 一条 Entry 可以关联多个 Threads。
- 当前不做主次关系。
- 当前不加 `role`、`source`、`is_primary`。

表：

```text
threads
entry_threads
```

关系表字段：

```text
entry_threads
- entry_id
- thread_id
- created_at
```

主键：

```text
(entry_id, thread_id)
```

未来如果确实需要主次，再新增 nullable 字段；现在不提前占位。

## 8. Media

已确认：

- MVP 阶段 `media` 只挂 `entry_id`。
- 暂时不加 `entry_module_id`。
- Todo 页面想展示相关照片时，通过 `todo_ref -> entry -> media` 查询。

建议字段：

| 字段 | 格式 | 备注 |
|---|---|---|
| `id` | UUID | 内部身份 |
| `entry_id` | UUID FK | 所属 Entry |
| `local_path` | text | App 自己复制并管理的本地路径 |
| `remote_path` | text nullable | 未来云端路径 |
| `file_type` | text | 例如 `image` |
| `mime_type` | text nullable | 例如 `image/jpeg` |
| `sync_status` | text nullable | 未来同步状态 |
| `created_at` | datetime | 创建时间 |

禁止：

- 不把手机相册原始路径作为唯一路径保存。

## 9. Todos

### 9.1 `todos`

用途：Todo 定义，例如“八段锦”。

### 9.2 `todo_completions`

用途：每次完成记录。热力图、完成次数、连续打卡都从这里统计。

规则：

- “没做八段锦”不是 completion。
- “没做的原因”应属于 Entry 内容，可能由 `todo_ref` 或后续专门结构表达。

## 10. Tags

已确认方向：

- 最终希望使用统一 tags 系统。
- Entry、Thread、Todo 后续都可以接统一 tags。
- 不为 Thread / Todo 各自发明一套 category。

典型表：

```text
tags
entry_tags
thread_tags
todo_tags
```

标签值可以是中文。Tag ID 仍使用 UUID。

## 11. Thread Analyses 与 Review Letters

已确认：

- 不需要 `tag_filter` 和 `custom` analysis type。
- 标签筛选、手动筛选只是发现或生成 Thread 的入口，不是独立 AI 分析类型。
- Thread 分析和回顾信倾向分表。

推荐表：

```text
thread_analyses
review_letters
review_letter_threads
```

### 11.1 `thread_analyses`

用途：某条 Thread 在某个时间点的 AI 阶段分析。

这是第 1.5 阶段功能，但现在要预留清晰接口。

### 11.2 `review_letters`

用途：某个时间周期内，全体 Entries 的整体回顾信。

### 11.3 `review_letter_threads`

用途：回顾信和相关 Threads 的关系。

例子：一封周回顾信关联 3 条 Thread，并显示每条 Thread 在这一周相关 Entry 数量和摘要。

## 12. AI 搜索、导出、洞察页口径

已确认：

- 全局搜索需要覆盖 Entry、Todo、Thread、回顾信、Thread 分析等内容。
- 即使回顾信和 Thread 分析分表，搜索 service 也要能统一搜索。
- 导出结构优先按真实表结构导出，避免人为合并成难维护的格式。
- 不需要一个“所有 AI 生成内容”列表页。
- 洞察页是入口聚合页，可以卡片式展示最新回顾信、重要 Threads、统计摘要等。

AI 第 1.5 阶段实现前必须明确：

- `searchService` 如何同时搜索多种表。
- `exportService` 是否按表原样导出。
- 洞察页卡片从哪些 service 获取数据。

## 13. Settings 与 API Key

设置分三类：

1. 普通本地设置：主题颜色、字号、页面展开状态、草稿缓存。
2. 普通数据库设置：分析频率、默认模板、是否自动生成回顾信、Todo 默认排序。
3. 安全存储：API key、token、敏感凭证。

API key 规则：

- 不进普通 SQLite。
- 不进普通 DB。
- 不进普通导出备份。
- 默认不跨设备同步；换设备需要重新输入。

## 14. 当前不采用的设计

以下设计当前不采用，也不登记为停放中：

- `analysis_type = tag_filter`
- `analysis_type = custom`
- `entry_threads.role`
- `entry_threads.source`
- `entry_threads.is_primary`
- `media.entry_module_id`
- 用 JSON 保存 Todo 和 Thread 的正式关系

## 15. 仍需继续讨论

- `structured_data` 是否保留，或哪些模块拆专门表。
- Tags 是否第一阶段进 schema，还是第 1.5/2 阶段再实现。
- Todo 未来是否需要直接关联 Thread；如果需要，是一对一还是多对多。
- 回收站、软删除、物理清理、多端同步确认的具体字段。
- 全局搜索 service 的具体索引方案。
