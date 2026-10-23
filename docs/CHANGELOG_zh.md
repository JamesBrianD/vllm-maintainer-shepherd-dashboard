# Fork 改动记录

本仓库 fork 自 [mgoin/pingboard](https://github.com/mgoin/pingboard)。这里只记录 fork 之后新增的功能和改动；在这之前的历史以上游为准。

- **上游基线**：`8d08666 Follow authors: highlight their PRs and list them in Add PRs (#3)`
- **记录约定**：按日期倒序，最新的写在最上面。每条写明类型（新功能 / 文档 / 修复 / 其他）、一句话说明和涉及的文件。以后每次改动都在最上面追加一条。

---

## 2026-10-23

- **其他**：新增 `AGENTS.md`，约定 agent 每次新增功能或做用户可见的改动时，同步更新 `README.md` 的 “What this fork adds” 一节和本改动记录（必要时也更新 `docs/guide_zh.md`），并写明项目的其他约定。
  - 涉及文件：`AGENTS.md`
- **文档**：`README.md` 新增 “What this fork adds” 一节（英文），集中介绍本 fork 相对上游新增的功能：主题切换、中文文档和改动记录，并给出 GitHub Pages 地址。原先写在 “The board” 一节里的主题说明一并移到这里。
  - 涉及文件：`README.md`
- **新功能**：右上角新增主题切换下拉框，主题参照 GitHub Appearance 设置，共 10 个：
  - Sync with system（默认，跟随系统，和原来的行为一致）
  - Light default / Light high contrast / Light Protanopia & Deuteranopia / Light Tritanopia
  - Dark default / **Dark dimmed（soft dark）** / Dark high contrast / Dark Protanopia & Deuteranopia / Dark Tritanopia

  Light default 和 Dark default 沿用原来的配色。主题保存在 `localStorage` 的 `pingboard.theme` 里，不按账号区分，多个标签页之间同步；`index.html` 里有一小段内联脚本在首次绘制前设好主题，刷新时不会闪屏。看板数据格式没有改动。
  - 涉及文件：`styles.css`、`index.html`、`app.js`、`README.md`、`docs/guide_zh.md`
- **文档**：新增中文使用指南（提交 `79c9d36`），内容包括项目介绍、使用方法、运行原理，以及部署到 GitHub Pages 和数据持久化说明。
  - 涉及文件：`docs/guide_zh.md`
- **文档**：新增本改动记录。
  - 涉及文件：`docs/CHANGELOG_zh.md`
