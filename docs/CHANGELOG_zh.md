# Fork 改动记录

本仓库 fork 自 [mgoin/pingboard](https://github.com/mgoin/pingboard)。这里只记录 fork 之后新增的功能和改动；在这之前的历史以上游为准。

- **上游基线**：`8d08666 Follow authors: highlight their PRs and list them in Add PRs (#3)`
- **记录约定**：按日期倒序，最新的写在最上面。每条写明类型（新功能 / 文档 / 修复 / 其他）、一句话说明和涉及的文件。以后每次改动都在最上面追加一条。

---

## 2026-10-24

- **新功能**：PR 详情页新增 **AI Review** 面板和完整报告弹窗。dashboard 用用户的 Token 读取 GitHub 仓库里由 skill 生成的 review 报告（默认是 `shen-shanshan/vllm-dev-skills` 中 `vllm-rocm-pr-review` 的 `outputs/pr-{number}-review.md`），按文件名匹配 PR。
  - 面板显示结论徽章、review 时间、第 1 节总结，以及“PR 在 review 之后是否有新提交”的提示；没有报告时显示 “No AI review yet”。
  - 完整报告由 GitHub `/markdown` 接口渲染，插入前再做一次过滤；Mermaid 图只在需要时从 CDN 加载 mermaid 绘制，失败时显示源码。快捷键 `i`。
  - 报告来源可在 Settings → AI review reports 中配置多个，保存在按账号隔离的偏好设置里（新增 `reviewSources` 字段，看板数据格式不变）。
  - 兼容没有 front matter 的旧报告：结论从“结论 (Verdict)”一节识别。
  - 涉及文件：`app.js`、`styles.css`、`README.md`、`AGENTS.md`、`docs/guide_zh.md`
- **其他**：`vllm-dev-skills` 仓库的 `vllm-rocm-pr-review` skill 改为在报告开头输出 front matter（`pr`、`repo`、`head_sha`、`verdict`、`reviewed_at`、`skill`），并在报告写完后提示推送。这个改动在另一个仓库，这里只做记录。
  - 涉及文件：`vllm-dev-skills/skills/vllm-rocm-pr-review/SKILL.md`

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
