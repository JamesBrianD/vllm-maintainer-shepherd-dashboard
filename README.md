# Pingboard

One board for the pull requests you are shepherding through CI. Pin the PRs you care about and each one tells you, in plain words, where it stands and what it needs next — instead of a browser tab per PR. The usual maintainer actions (update branch, run or retry CI, labels, approve, merge) are buttons on the board.

<img width="1668" height="1027" alt="Screenshot 2026-10-08 at 12 57 06 PM" src="https://github.com/user-attachments/assets/458d4cbf-d81b-4962-937f-36c99a792ed9" />

It is a static page that talks to the GitHub API from your browser: no server, no build step, no dependencies. It is tuned for [vllm-project/vllm](https://github.com/vllm-project/vllm) and works for any repository.

## What this fork adds

This is a fork of [mgoin/pingboard](https://github.com/mgoin/pingboard), hosted at **https://shen-shanshan.github.io/vllm-maintainer-shepherd-dashboard/**. The rest of this README describes features shared with upstream; this section lists what the fork adds on top.

### Theme picker

Upstream only follows the operating system's light/dark setting. This fork puts a theme picker in the top-right corner (next to **Refresh**) with the themes of [GitHub's appearance settings](https://github.com/settings/appearance):

| Group | Themes |
| --- | --- |
| — | **Sync with system** (default, same as upstream) |
| Light | Light default · Light high contrast · Light Protanopia & Deuteranopia · Light Tritanopia |
| Dark | Dark default · **Dark dimmed (soft dark)** · Dark high contrast · Dark Protanopia & Deuteranopia · Dark Tritanopia |

- *Light default* and *Dark default* keep upstream's palette; the other themes take their colors from GitHub's matching themes. The colorblind themes move success to blue and failure to orange (or keep red for tritanopia), so CI states stay distinguishable.
- The choice is stored per browser under `pingboard.theme` in `localStorage` — not per GitHub login, so the sign-in page uses it too — and is synced across open tabs.
- A small inline script in `index.html` applies the theme before the first paint, so a reload never flashes the light theme.
- The saved board format is unchanged; boards from upstream load as they are.

### AI review reports

Each PR's detail pane gets an **AI Review** panel showing a review report written outside Pingboard. Pingboard is a static page and cannot run an agent, so the workflow is:

1. Run a review skill locally — by default [`vllm-rocm-pr-review`](https://github.com/shen-shanshan/vllm-dev-skills/tree/master/skills/vllm-rocm-pr-review), a Claude Code skill that writes `outputs/pr-<number>-review.md`.
2. Push the report to its GitHub repository.
3. Pingboard lists that directory with your token (once every 10 minutes, REST quota) and matches file names to the PRs on your board.

The panel shows the verdict (🔴 Block / ⚠️ Needs work / ✅ LGTM), when the review was written, the summary section, and whether **the PR has new commits since the review**. That comes from front matter at the top of the report (`pr`, `repo`, `head_sha`, `verdict`, `reviewed_at`). Reports without front matter still show, with the verdict read from their 结论 (Verdict) section and the reviewed commit marked unknown. PRs without a report say *No AI review yet*.

**View full report** (or `i`) opens the whole report in a large dialog. GitHub's `/markdown` API renders it, so tables, code and `#123` links look as they do on github.com. Mermaid diagrams are drawn by [mermaid](https://mermaid.js.org/), loaded from a CDN only when a report contains one; if it cannot load, the diagrams stay as source.

Sources are configurable in **Settings → AI review reports**, one per line:

```
Label | owner/repo | path/with/{number}/in/the/file-name.md | repo of the PRs it reviews
ROCm review | shen-shanshan/vllm-dev-skills | skills/vllm-rocm-pr-review/outputs/pr-{number}-review.md | vllm-project/vllm
```

The report repository must be readable by your token. A private repository works too, but then only people whose token can read it see the reports.

### Chinese documentation

- [`docs/guide_zh.md`](docs/guide_zh.md) — a guide in Chinese: what Pingboard is, how to use it, how it works (PR fetching, CI classification, related/unrelated failure detection, refresh and rate limits), deploying a fork to GitHub Pages, and where the board data lives.
- [`docs/CHANGELOG_zh.md`](docs/CHANGELOG_zh.md) — every change made in this fork since upstream commit `8d08666`, with dates.

## The board

Each row is one pull request with a status and, where there is an obvious next step, a button for it.

| Status | Meaning | Button |
| --- | --- | --- |
| **CI not started** | no build on the latest commit | Run CI, or Update branch & run CI when the branch is behind |
| **CI running** | jobs in progress, with any failures so far | — |
| **N failures to look at** | failing jobs not seen failing anywhere else | — |
| **N failures, likely unrelated** | every failing job also fails on the base branch or on other pinned PRs | Retry failed jobs |
| **CI passed · needs review** | green, waiting on a review | — |
| **Ready to merge** | green and approved | Merge… |
| **Merge conflicts** | the author has to resolve them | — |

A blue dot marks a PR that changed since you last selected it. Drag rows to reorder them or move them between groups, and drag the divider to resize the list.

Selecting a row opens the detail pane: the status with a one-line explanation, the failing and running jobs (each links to its Buildkite job), branch freshness, reviews, labels, and the latest comments — which is where the CI bot answers `/ci` commands.

**+ Add PRs** lists open pull requests in your default repository (yours, review requested, reviewed, involved, `ready`, all recent) and narrows as you type. Click to pin; pasting PR numbers or URLs works too. Type `@name` to see one author's PRs. To keep an eye on particular people, **Follow** them from a PR (or list them in Settings): their names are highlighted everywhere and Add PRs gets a **Following** list.

## Related or unrelated failures

Pingboard does not read logs — Buildkite logs are not reachable from a static page. It sorts failing jobs using what GitHub already knows:

- **N/M on base** — the same job failed on N of the last M base-branch commits that ran CI.
- **also #123 #456** — the same job is failing on other PRs on your board.

A job with either kind of evidence is listed as "likely unrelated". Everything else is "to look at".

## Actions

Everything is done as you, with your token.

- **Comments** (`/ci run`, `/ci retry`, `/ci cancel`) and **branch updates** wait three seconds behind an Undo bar before they are sent.
- **Update branch & run CI** merges the base branch in, waits for the new commit, then posts `/ci run`.
- **Approve** and **Merge** ask for confirmation. **Labels** apply immediately.

Keyboard shortcuts exist for all of it; press `?` for the list.

## Token

Pingboard takes a **classic** personal access token with the `repo` scope (`public_repo` is enough for public repositories). It is sent only from your browser to `api.github.com`, and is kept in `sessionStorage` — or `localStorage` if you tick "Keep token after closing this tab". The board itself (pins, groups, preferences) lives in `localStorage` per GitHub login, so it belongs to one browser and one address. Settings has **Export to file** and **Import from file** to back it up or move it.

## Refresh and rate limits

Open PRs refresh every 60 seconds by default (configurable), background tabs five times slower, at one GraphQL request per PR. The base-branch baseline covers the last 30 commits and is re-read every 10 minutes. Remaining API quota is shown in the top bar, and polling pauses if it runs low.

## Other repositories

`owner/repo#123` or a URL pins a PR from anywhere. Repository-specific workflow — which checks count as CI, the comment commands, the quick labels — lives in `REPO_PROFILES` at the top of `app.js`. Repositories without a profile treat every check as CI and have no CI commands.

## Run it

Open `index.html`, or serve the folder:

```bash
python3 -m http.server 8080
```

The included workflow publishes the folder to GitHub Pages on every push to `main`.

## Tests

```bash
./tests/run
```

Runs the checks every pull request has to pass. It needs [Bun](https://bun.sh) and installs everything else itself.
