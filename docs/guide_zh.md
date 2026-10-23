# Pingboard 中文使用指南

本文介绍 Pingboard（本仓库）的用途、使用方法和运行原理，以及如何部署到自己的 GitHub Pages、部署时数据会不会丢失。

- [一、项目简介](#一项目简介)
- [二、使用方法](#二使用方法)
- [三、运行原理](#三运行原理)
- [四、测试](#四测试)
- [五、部署到自己的 GitHub Pages](#五部署到自己的-github-pages)
- [六、重新部署会丢失数据吗](#六重新部署会丢失数据吗)

---

## 一、项目简介

Pingboard 是给 vLLM maintainer 用的 PR“看护”看板。

维护者通常要同时盯着很多 PR 等 CI 跑完，平时只能每个 PR 开一个浏览器标签页。Pingboard 把这些 PR 放进同一个看板，每个 PR 占一行，直接写明它现在的状态和下一步该做什么。常见的维护操作也做成了按钮，可以在看板上直接点。

| 状态 | 含义 | 可用按钮 |
|---|---|---|
| CI not started | 最新 commit 还没跑 CI | Run CI；分支落后时是 Update branch & run CI |
| CI running | CI 正在跑，已经出现的失败也会列出来 | — |
| N failures to look at | 有失败的 job，而且在别处没见过同样的失败 | — |
| N failures, likely unrelated | 失败的 job 在 base 分支或其他 PR 上也失败过 | Retry failed jobs |
| CI passed · needs review | CI 全绿，等 review | — |
| Ready to merge | CI 全绿且已 approve | Merge… |
| Merge conflicts | 有冲突，需要作者自己解决 | — |

点开某个 PR 会出现详情面板，里面有失败或运行中的 job（可以跳到 Buildkite）、分支落后了多少、review 情况、label，以及最近的评论。CI bot 对 `/ci` 命令的回复就显示在评论里。

---

## 二、使用方法

1. **启动**：它是纯静态页面，不需要构建，也没有依赖。可以直接打开 `index.html`，或者在目录下启动一个本地服务：

   ```bash
   cd /home/shashen/vllm/vllm-maintainer-shepherd-dashboard
   python3 -m http.server 8080   # 然后浏览器访问 http://localhost:8080
   ```

   也可以部署到 GitHub Pages，直接在线使用，见[第五节](#五部署到自己的-github-pages)。

2. **填 Token**：需要一个 classic 类型的 GitHub Personal Access Token，权限选 `repo`（只看公开仓库的话 `public_repo` 就够了）。Token 只会从你的浏览器发往 `api.github.com`，默认存在 `sessionStorage` 里，关掉标签页就没了。勾选 “Keep token after closing this tab” 后会改存到 `localStorage`。

3. **添加 PR**：点 **+ Add PRs** 会列出默认仓库（vllm-project/vllm）里的 open PR，有这几类：你自己的、请求你 review 的、你 review 过的、和你相关的、带 `ready` 标签的、最近的。也可以：
   - 直接粘贴 PR 号或 URL；
   - 输入 `@用户名` 看某个作者的 PR；
   - 输入 `owner/repo#123` 添加其他仓库的 PR。

4. **整理看板**：可以拖动行来排序或分组，也可以拖动分隔条调整列表宽度。PR 自你上次查看后有变化时会显示蓝点。**Follow** 某个作者后，他们的名字会高亮显示，Add PRs 里也会多出一个 Following 列表。

   **切换主题**：右上角 Refresh 按钮右边有一个主题下拉框，选项参照 GitHub 的 Appearance 设置：
   - Sync with system：默认值，跟随系统深浅色；
   - Light 组：Light default、Light high contrast、Light Protanopia & Deuteranopia、Light Tritanopia；
   - Dark 组：Dark default、**Dark dimmed（soft dark）**、Dark high contrast、Dark Protanopia & Deuteranopia、Dark Tritanopia。

   选择结果保存在当前浏览器的 `localStorage`（key 为 `pingboard.theme`），不区分 GitHub 账号，登录页也会使用；多个标签页之间会自动同步。

5. **执行操作**：
   - 评论命令（`/ci run`、`/ci retry`、`/ci cancel`）和更新分支会先等 3 秒，期间可以点 Undo 撤销；
   - Approve 和 Merge 需要二次确认；
   - Label 点了立即生效；
   - 所有操作都用你的 Token、以你的身份执行；
   - 按 `?` 可以查看全部快捷键。

6. **备份**：看板数据（钉住的 PR、分组、偏好设置）按 GitHub 登录名存在 `localStorage` 里。Settings 里有导出和导入文件的功能，可以备份或者换浏览器迁移。

---

## 三、运行原理

整个应用就是 `index.html`、`app.js`（约 100KB，原生 JS）和 `styles.css` 三个文件，没有后端，浏览器直接调用 GitHub API。

```
浏览器 ──(PAT)──> api.github.com (GraphQL + REST)
   │
   ├─ 刷新调度：每 15s 检查一次，哪个 PR 到期了就排队刷新，最多 4 个并发
   ├─ fetchPR：每个 PR 一次 GraphQL 请求
   ├─ refreshBaseline：扫描 base 分支最近 30 个 commit 的 CI 结果
   └─ localStorage / sessionStorage：保存看板状态和 Token
```

### 1. 拉取 PR 数据（`app.js` 中的 `fetchPR`）

- 一次 GraphQL 请求取回 PR 的状态、mergeable、review 决定、label、评论、最新 commit 的 check 汇总，以及相对 base 分支落后多少个 commit。
- check 很多需要分页时，会先算一个指纹（commit oid 加上各状态的计数）。指纹没变就复用上次的 job 列表，以节省 API 配额。

### 2. 判断哪些是 CI（`app.js` 开头的 `REPO_PROFILES`）

- 对 vLLM 来说，只有名字以 `buildkite/` 开头的 check 算 CI。其中 `buildkite/ci/…` 这条主 pipeline 决定能否合并，其他 pipeline 只作参考。
- `/ci` 系列命令、快捷 label（`ready`、`verified`）和合并方式（squash）也在这里配置。
- 没有配置 profile 的仓库，所有 check 都算 CI，也没有 CI 命令。

### 3. 判断失败是否与本 PR 相关（`failureIndex` / `splitFailures` / `refreshBaseline`）

这是它的核心逻辑。浏览器读不到 Buildkite 日志，所以它只用 GitHub 上已有的信息来判断：

- **N/M on base**：扫描 base 分支最近 30 个跑过 CI 的 commit，统计同一个 job 失败了几次。这份基线每 10 分钟刷新一次；已经跑完的 commit 结果会缓存起来。
- **also #123 #456**：同一个 job 在看板上其他 PR 里也在失败。
- 两条证据有任意一条，就归为 “likely unrelated”，并给出 Retry 按钮；否则归为 “to look at”，需要你去看。

### 4. 刷新和限流

- open 的 PR 默认每 60 秒刷新一次（可以改）；已关闭或已合并的 PR 每 15 分钟刷新一次；页面在后台标签页时刷新频率降为五分之一。
- 剩余 API 配额显示在顶栏，低于 150 点时暂停后台轮询。

### 5. 变化提示

每个 PR 会记一个签名，由 state、head commit、CI 状态、失败数和 review 决定拼成。签名和你上次查看时不一样，就亮蓝点。

---

## 四、测试

```bash
./tests/run   # 需要先安装 Bun，Playwright/Chromium 会自动装
```

测试主要检查旧版本保存的看板数据在新版本里还能正常加载（`tests/state-compat.mjs`，配合 `tests/fixtures/`），防止升级后用户的看板丢失。每个 PR 都会在 CI 里跑一遍。

**总结**：这是一个零依赖的纯前端看板，用来同时盯多个 vLLM PR 的 CI 和 review 状态，并直接触发 `/ci`、更新分支、打 label、approve、merge 等操作。它最有用的地方是自动区分失败是 PR 自己造成的，还是主干本来就有的 flaky 失败。

---

## 五、部署到自己的 GitHub Pages

本仓库 fork 自 [mgoin/pingboard](https://github.com/mgoin/pingboard)，现在的远程仓库是 `shen-shanshan/vllm-maintainer-shepherd-dashboard`。代码不用改：仓库里已经有部署用的 workflow（`.github/workflows/pages.yml`），页面里引用的资源也都是相对路径（`./app.js`、`./styles.css`），放在子路径下也能正常加载。你只需要在 GitHub 网页上做几步设置。

### 部署步骤

1. **开启 Actions**：fork 出来的仓库默认禁用 Actions。打开 `https://github.com/shen-shanshan/vllm-maintainer-shepherd-dashboard/actions`，如果看到 “Workflows aren't being run on this forked repository”，点 **I understand my workflows, go ahead and enable them**。

2. **把 Pages 的来源设为 GitHub Actions**：进入仓库的 **Settings → Pages**，在 **Build and deployment → Source** 里选 **GitHub Actions**，不要选 “Deploy from a branch”。

3. **触发部署**（两种方式，选一种）：
   - **网页上手动触发**：到 Actions 页面，左侧选 **Deploy GitHub Pages**，点 **Run workflow**，分支选 `main`。这个 workflow 支持 `workflow_dispatch`，所以可以手动跑。
   - **推一个提交**：往 `main` 推任意一个 commit 都会自动部署，例如：

     ```bash
     git commit --allow-empty -m "Trigger Pages deploy" && git push origin main
     ```

4. **打开网站**：workflow 跑完（约 1 分钟）后访问：

   ```
   https://shen-shanshan.github.io/vllm-maintainer-shepherd-dashboard/
   ```

### 第一次使用

1. **生成 Token**：打开 GitHub → Settings → Developer settings → Personal access tokens → **Tokens (classic)** → Generate new token，勾选 `repo` 权限。必须是 classic token，fine-grained 的不行。
   - 只看、只操作公开仓库（比如 vllm）的话，勾 `public_repo` 就够了。
   - Approve、Merge、打 label 这些操作都以你的身份执行，所以你在 vllm 仓库里需要有对应权限。没有 write 权限也能查看 PR、发 `/ci` 评论，但合并、打 label 会失败。
2. **登录**：在页面上粘贴 Token。如果想关掉标签页后不用重新输入，勾选 “Keep token after closing this tab”。
3. **添加 PR**：点 **+ Add PRs** 选择要盯的 PR，或者直接粘贴 PR 号或链接。

### 注意事项

- **数据只存在当前浏览器里**：看板（钉住的 PR、分组、设置）保存在这个浏览器、这个域名的 `localStorage` 中。如果你以前在 mgoin 的站点或 `localhost` 上用过，数据不会自动带过来。可以在旧地址用 Settings → **Export to file** 导出，再到新站点 **Import from file** 导入。
- **Token 不会泄露给站点**：GitHub Pages 只托管静态文件，Token 只从你的浏览器直接发往 `api.github.com`，不经过任何服务器。
- **之后改代码**：每次 push 到 `main` 都会自动重新部署。
- **同步上游更新**：在仓库首页点 **Sync fork**，或者在本地执行：

  ```bash
  git remote add upstream https://github.com/mgoin/pingboard.git
  git pull upstream main && git push origin main
  ```

---

## 六、重新部署会丢失数据吗

不会丢。重新部署只会替换服务器上的 `index.html`、`app.js`、`styles.css` 这些静态文件，你的看板数据根本不在服务器上。

### 数据存在哪里

| 数据 | 存储位置 | 重新部署后 |
|---|---|---|
| 钉住的 PR、分组、排序、已读标记 | 浏览器 `localStorage`，key 为 `pingboard.v3.<你的登录名>.board` | 保留 |
| 偏好设置（刷新间隔、Follow 列表等） | `localStorage`，key 为 `pingboard.v3.<登录名>.prefs` | 保留 |
| Token | `sessionStorage`，勾选 “Keep token” 时改存 `localStorage` | 保留 |
| PR 的 CI 状态、评论等 | 不保存，每次打开页面从 GitHub API 实时拉取 | 不受影响 |

`localStorage` 按 **origin（域名）** 隔离。你的站点 origin 一直是 `https://shen-shanshan.github.io`，部署多少次都不会变，所以数据会一直在。

### 什么情况下数据会“看不见”或丢失

1. **你改代码时改了存储格式或 key 名**：比如把 `scopedKey()` 里的 `pingboard.v3` 改成 `v4`，或者改了 `cleanBoard()` 的校验逻辑。新代码读不到或读不了旧数据，就会显示空看板。这个仓库的 `tests/run`（`tests/state-compat.mjs`）就是专门防这种情况的：它会检查旧版本存下的数据，新版本还能原样加载。改了存储相关代码，推送前跑一遍 `./tests/run` 就行。
2. **换了域名**：比如绑定了自定义域名，origin 就变了，旧数据还留在 `shen-shanshan.github.io` 下面，新域名下看不到。只是改仓库名不影响，因为改的是 URL 路径，origin 不变。
3. **换浏览器、换电脑、用无痕模式，或者清理了浏览器数据**：这些是浏览器端的问题，和部署无关。

### 建议

时不时在 Settings 里点一下 **Export to file** 备份看板。万一出了上面这些情况，用 **Import from file** 就能恢复。
