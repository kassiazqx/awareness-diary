# Workflow

## 日常开发流程

1. 读 `CLAUDE.md` 或 `AGENTS.md`。
2. 读 `docs/00-overview/ai-start-here.md`。
3. 读 `docs/01-architecture/README.md`。
4. 新功能先在 `docs/02-design/YYYY-MM-DD-feature-name/` 写 spec / plan / decisions。
5. 实现后更新必要文档。
6. 用户可感知变化写入根目录 `CHANGELOG.md`。

## 小改动

小 bug、文案、轻微样式调整，可以只写清 commit message。若用户能明显感知变化，再补 `CHANGELOG.md`。
