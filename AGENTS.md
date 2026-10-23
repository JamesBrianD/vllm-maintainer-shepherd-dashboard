# AGENTS.md

Guidance for coding agents working in this repository.

This is a fork of [mgoin/pingboard](https://github.com/mgoin/pingboard). Upstream history ends at commit `8d08666`; everything after that is this fork's own work.

## Every new feature or user-visible change must update the docs

When you add a feature or change behaviour a user can see, update these in the same change — do not leave them for later:

1. **`README.md` → "What this fork adds"** (English)
   - Add a `###` subsection for a new feature, or extend the existing one.
   - Say what it does, how it differs from upstream, and where to find it in the UI.
   - Keep the rest of the README describing behaviour shared with upstream; fork-only features belong in this section.
2. **`docs/CHANGELOG_zh.md`** (Chinese)
   - Add an entry at the **top**, under a `## YYYY-MM-DD` heading for today's date (reuse the heading if it already exists).
   - Each entry: type (新功能 / 修复 / 文档 / 其他), a one-sentence description, and the files it touched (`涉及文件：…`).
3. **`docs/guide_zh.md`** (Chinese) — update it too when the change affects how the board is used, deployed, or where data is stored.

Pure refactors with no visible effect only need a changelog entry.

## Other conventions

- No build step and no dependencies: plain `index.html`, `app.js`, `styles.css`. Keep it that way.
- Match the surrounding code: `app.js` is organised in numbered sections (see the header comment); put new code in the matching section.
- Colours go through the CSS variables at the top of `styles.css`; every theme (`:root[data-theme=…]`) must keep working.
- Don't break saved boards. Board and prefs live in `localStorage` under `pingboard.v3.<login>.*`. If the format must change, migrate the old format on load and add a fixture under `tests/fixtures/`.
- Run `./tests/run` (needs [Bun](https://bun.sh)) before pushing; it checks that existing saved boards still load.
- Pushing to `main` deploys to GitHub Pages automatically.
