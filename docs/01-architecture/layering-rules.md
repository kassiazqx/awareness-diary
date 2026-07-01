# Layering Rules

## 依赖方向

UI / pages -> services -> data -> infrastructure

反向依赖禁止。

## 目录职责

- `src/pages/`：页面入口，组织展示和交互。
- `src/components/`：复用 UI 组件，不直接读写数据层。
- `src/lib/services/`：业务流程，例如保存 entry、生成分析、完成 todo。
- `src/lib/data/`：SQLite / 文件 / 未来云同步的数据访问适配。
- `src/lib/utils/`：纯函数工具。

## 禁止

- 页面直接写 SQL。
- 组件直接访问本地存储或云 API。
- service 返回 UI 专属结构。
- data 层知道页面或组件存在。
