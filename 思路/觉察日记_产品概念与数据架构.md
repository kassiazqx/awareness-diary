# 觉察日记 App — 产品概念与数据架构文档 v1.0

> 本文档面向 AI 协作者、开发者、产品讨论。阅读本文档即可完整理解产品设计意图、功能边界与数据模型。

---

## 一、产品核心理念

觉察日记不是普通日记，也不是 Todo 工具。它是一个帮助用户**持续看见自己内心模式**的结构化觉察系统。

核心隐喻：**主线是骨架，记录是血肉，AI 是镜子。**

用户通过反复记录日常的情绪、事件、行动、念头，AI 帮助归纳出跨越时间的"主线"（人生中正在发生的某个主题），让用户得以跳出当下，看见自己更长时间维度的模式与变化。

---

## 二、五大核心概念

### 1. Thread（主线）
用户生活中某个持续关注的主题，如"工作压力"、"与妈妈的关系"、"身体健康"。

- 一条主线可关联多条日记（entries）
- 一条主线有独立的 AI 分析历史，可看到年初/3月/5月各是什么状态
- 主线由 AI 建议、用户决定是否采纳（user_decision：adopted/rejected/pending）
- 可手动创建，也可从 AI 分析中派生

### 2. Entry（日记记录）
用户某次觉察的完整记录。

- 每条 Entry 有一个主要关联 Thread（primary_thread_id），可额外关联多个次要 Thread
- Entry 由若干 Module（模块）组成，模块可自由拖拽排序
- Entry 有 entry_date（支持补记历史日期）
- 所有文字内容使用富文本，存两层：content_json（TipTap JSON，层1）+ content_plain（纯文字，层2）
- 前端从 content_json 实时渲染 HTML（层3），层3 不入库

### 3. Module（模块）
Entry 的最小内容单元。系统预定义固定的模块类型目录（module_types），用户自由组合：

| 模块 key | 说明 | 数据类型 |
|---|---|---|
| event | 发生了什么事 | richtext |
| person | 涉及的人 | richtext |
| weather | 天气 | structured（temp/condition） |
| emotion | 情绪状态 | structured（score + tags） |
| action | 我做了什么 | richtext |
| cognition | 念头/想法 | richtext |
| acceptance | 接纳/复盘 | richtext |
| todo_ref | 引用一个 Todo | structured（todo_id） |
| thread_ref | 引用一条次要主线 | structured（thread_id） |
| free_text | 自由漫谈，无任何框架 | richtext |
| gratitude_fact | 客观发生的事（感恩模版专用） | richtext |
| gratitude_thought | 我的感恩想法 | richtext |
| learning_content | 学习内容 | richtext |
| learning_importance | 为什么重要 | richtext |
| practice_plan | 实践计划 | richtext |
| lucky_received | 幸运走向我的事 | richtext |
| lucky_pursued | 我主动追逐幸运 | richtext |

> 特殊场景（感恩/学习/幸运模版）通过不同的模块 key + custom_label 实现，不需要单独新建模版类型。

**模块排列顺序：**
- 固定头部（非模块）：entry_date + primary_thread，始终展示在最顶部，不可拖动
- 可拖动区域：entry_modules，按 sort_order 渲染
- 固定底部：created_at 时间戳

### 4. Template（模版）
模块的有序组合，方便快速开始记录。

| 内置系统模版 | 模块组成 |
|---|---|
| 情绪觉察（长） | event, person, weather, todo_ref, thread_ref, emotion, action, cognition, acceptance |
| 情绪觉察（短） | event, emotion, action, cognition, acceptance |
| Todo 觉察 | todo_ref, event, emotion, action, cognition, acceptance |
| 感恩 | gratitude_fact, gratitude_thought |
| 学习 | learning_content, learning_importance, practice_plan |
| 幸运 | lucky_received, lucky_pursued |
| 自由笔记 | free_text（仅一个空白模块） |

- source = system：系统预设，不可编辑，只能克隆
- source = user：用户自建或克隆后修改，cloned_from_id 记录来源
- entries.template_origin_id：记录"从哪个模版开始"，仅作起点记录和统计分析用，不约束后续编辑

### 5. Todo
一件事情的持续追踪单元，独立于 Entry 存在，可被多条 Entry 引用。

**分类：**
- 一次性 Todo：做完即完成，is_evergreen = false
- 重复 Todo：daily/weekly/monthly，按 repeat_rule JSON 设定规则
- 常青 Todo（Evergreen）：如"八段锦"，永远存在，无截止日，统计完成次数
  - 统计：COUNT(todo_completions WHERE todo_id = X)
  - 支持补记：completed_at 可填写历史时间，is_backdated = true

**关联关系：**
- 一个 Todo 可关联多个 Thread（todo_threads）
- 一个 Todo 可关联多个 Tag
- 一个 Todo 可被多个 Entry 引用（todo_entries，多少条 Entry 提到就插多少行，完全正常）

---

## 三、标签系统（Tags）

全局统一标签总表，所有实体共用：

| type | 举例 |
|---|---|
| emotion | 焦虑、平静、喜悦、委屈 |
| context | 工作、家庭、个人学习、健康、社会关系 |
| person | 妈妈、男友、同事xx、朋友xx |
| need | 安全感、被看见、被认可、控制感 |

- threads、entries、todos 各自通过 junction 表关联到同一个 tags 表
- 同一个标签只定义一次，三者共用，互不冲突
- 通过标签可以跨越所有实体做筛选和检索

---

## 四、AI 功能设计

### AI 输入
- 将 content_plain（纯净文字，无格式）拼接后送给 AI
- 按 Thread 批次、或按时间范围、或按标签筛选批次送入
- 用户主动触发 + 数据上传授权（非自动后台上传）

### AI 输出存储（analyses 表）
每次分析都是新的一行记录，永久追加，不覆盖旧数据。

| type | 用途 |
|---|---|
| thread | 主线的阶段性分析（按N条entry触发 或 定期触发） |
| period | 周报 / 月报 |
| tag_filter | 按标签（单选或多选交集）+ 日期范围的专项分析 |
| custom | 用户手动框选任意范围触发 |

每条分析记录包含：
- **stats_json**：基础统计快照（entry 数量、总字数、情绪均值、各 tag 频率、Todo 完成次数等）
- **analysis_json**：AI 分析内容（TipTap JSON 富文本，层1）
- **analysis_plain**：AI 分析内容（纯文字，层2）
- **entry_ids_snapshot**：本次分析包含的 entry ID 列表

### AI 建议主线（ai_suggestions）
AI 分析后可建议新的主线，存入 ai_suggestions 表：
- suggestion_type = new_thread / tag_adjustment / insight
- user_decision = pending → adopted（自动创建 thread）或 rejected

### Thread 分析触发规则
- 全局默认规则存 app_settings
- 每条主线可独立覆盖，存 thread_analysis_settings（trigger_every_n_entries / trigger_every_n_days）

---

## 五、数据库完整表结构

### 核心表

```
tags
  id, type, value, color, icon

module_types
  key(PK), display_name, default_hint, data_type, icon

templates
  id, name, source(system/user), cloned_from_id(nullable FK→templates), created_at

template_modules
  id, template_id(FK), module_type_key(FK), sort_order,
  custom_label, custom_hint, is_required

threads
  id, title, description, color, is_active, sort_order, created_at, updated_at

thread_analysis_settings
  thread_id(PK,FK), trigger_every_n_entries, trigger_every_n_days, is_auto_enabled

app_settings
  key(PK), value  -- 全局默认分析规则等

entries
  id, primary_thread_id(FK,nullable), template_origin_id(FK,nullable),
  entry_date(支持补记), created_at, updated_at

entry_modules
  id, entry_id(FK), module_type_key(FK), sort_order,
  content_json(TEXT,富文本层1), content_plain(TEXT,富文本层2),
  structured_data(JSON,非富文本模块用)

todos
  id, title, repeat_type(none/daily/weekly/monthly/custom),
  repeat_rule(JSON), is_evergreen, is_archived, due_date(nullable),
  priority, created_at

todo_completions
  id, todo_id(FK), completed_at(支持历史时间补记), is_backdated,
  note(TEXT), entry_id(FK,nullable)

media
  id, entry_id(FK), file_path, file_type, created_at

analyses
  id, type(thread/period/tag_filter/custom),
  thread_id(FK,nullable), period_type(weekly/monthly/custom),
  period_start(date), period_end(date),
  filter_tag_ids(JSON,nullable), filter_thread_ids(JSON,nullable),
  entry_ids_snapshot(JSON), entry_count, word_count,
  stats_json(JSON,情绪均值/tag频率/Todo完成数等),
  analysis_json(TEXT,AI内容层1), analysis_plain(TEXT,AI内容层2),
  trigger_type(manual/scheduled/auto), created_at

ai_suggestions
  id, analysis_id(FK), suggestion_type(new_thread/tag/insight),
  content_json(TEXT), user_decision(pending/adopted/rejected), decided_at
```

### Junction 表（多对多关联）

```
thread_tags       thread_id, tag_id
entry_tags        entry_id, tag_id
todo_tags         todo_id, tag_id
entry_threads     entry_id, thread_id  (次要 thread 关联)
todo_threads      todo_id, thread_id
todo_entries      todo_id, entry_id
```

---

## 六、富文本三层架构

| 层 | 字段名 | 内容 | 存库 |
|---|---|---|---|
| 层1（唯一真源） | content_json | TipTap JSON，含全部格式信息 | ✅ |
| 层2（检索/AI用） | content_plain | 纯文字，剥离所有格式标记 | ✅ |
| 层3（视觉展示） | 无字段 | 由前端从 content_json 实时渲染的 HTML | ❌ 不入库 |

- 层3 永远从层1实时生成，格式与源数据永远同步，无需维护
- AI 分析、全文搜索全部使用 content_plain
- entry_modules 每行都有独立的 content_json + content_plain
- analyses 表的分析文字同理：analysis_json + analysis_plain

---

## 七、技术选型建议

| 层 | 选型 | 理由 |
|---|---|---|
| 跨平台框架 | React Native + Expo | 一套代码出 iOS + Android，Expo 封装好文件/相机权限 |
| 本地数据库 | SQLite（expo-sqlite） | 离线优先，本地存储，成熟稳定 |
| 富文本编辑器 | TipTap | 输出标准 JSON，层1/层2 转换有官方工具 |
| AI 接入 | 外部 API（用户主动触发） | 与本地存储解耦，不影响数据模型 |
| 打包发布 | EAS Build | Expo 官方打包，直出 APK / IPA |
| 云同步（后期） | JSON 导出 → iCloud / Google Drive | 最简实现，不需要自建服务器 |

---

## 八、开发优先级建议

```
阶段1（MVP）：
  ✅ entries + entry_modules 的增删查改（含富文本三层）
  ✅ 主线（threads）的创建和关联
  ✅ 固定几个系统模版
  ✅ 基础标签打标

阶段2：
  ✅ templates 自定义
  ✅ Todo + todo_completions（含常青/重复/补记）
  ✅ analyses 基础统计（stats_json，暂不接 AI）

阶段3：
  ✅ AI 接入（调外部 API，送 content_plain，存 analyses）
  ✅ AI 建议主线（ai_suggestions）
  ✅ 历史分析时间线展示

阶段4：
  ✅ 多媒体附件（先图片）
  ✅ 云备份/同步
  ✅ 统计可视化页面
```

---

*文档版本：v1.0 | 生成时间：2026-05-07*
