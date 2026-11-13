# Life To Do — notes for Claude

Weekly checklist PWA, hosted on GitHub Pages from `main` (repo root). Data lives in Firebase (Firestore + email/password Auth, project `life-todo-334bf`). This repo is **public**: never commit personal data (names, bills, list contents, emails). Defaults in code stay generic.

## Files
- `index.html`: the whole app (CSS, classic script for UI/state, module script for Firebase).
- `sw.js`: offline cache. **Bump `CACHE` (`life-todo-vN`) on every change** or phones keep the old version, and bump `APP_VERSION` in `index.html` to the same N (it's shown at the bottom of Settings so the owner can confirm the update arrived). Only same-origin and `www.gstatic.com` are cached; Firestore traffic must bypass the worker.
- `manifest.json`, `icon-*.png`: home-screen install.

## Architecture
- Adding items: the `+ Add item` bottom sheet (`#itemDlg`) handles add and edit (long-press an item to edit). Items are `{text,status,fromTemplate,day?}`; never write `day: undefined` (Firestore rejects it). "Repeat every week" also appends the text to that section's template.
- Sorting: Bills/Nutrition/Chores render open → done → missed, and within each by day tag Mon→Sun then untagged (display only; saved order is untouched, and `idx` in `render()` is always the saved index). A tapped item keeps its pre-tap rank until taps pause ~1s (`settleHold`), then slides into place (FLIP). Exercise is never sorted.
- Today / Overdue: shown above the sections only on the current calendar week (hidden on Sundays, when the app displays next week). Items are moved, not copied: tagged today (any status) plus today's Exercise line (matched by leading day name) go in Today; tagged for an earlier day and still open go in Overdue (red/missed stay home). Held (just-tapped) items keep their frozen rank so nothing jumps mid-tap. `makeItem()` builds every row, so tap/long-press/delete behave the same everywhere; `idx` is always the saved index.
- UI talks only to `Store` (in-memory cache). Local mode → localStorage. Cloud mode → `Store.cloud.saveSettings/saveWeek` → Firestore `users/{uid}` `{settings}` and `users/{uid}/weeks/{weekId}` `{sections}`.
- Cloud mode renders only after BOTH the settings and weeks snapshots arrive, so the current week is never auto-created over unloaded data.
- Week id = Monday's date `YYYY-MM-DD`; Sunday maps to the next Monday. Only the current week auto-creates.
- `buildWeek` seeds from templates (`{n}` = rep count) and carries open non-template items, de-duplicated against template text.
- Sunday check-in: `exportWeek(id)` / `parsePlan(text)`. Format: `WEEK MM.DD.YY`, then section headers `BILLS/EXERCISE/NUTRITION/CHORES`, then `[ ]`/`[x]`/`[-]` lines. An optional trailing ` @Mon`…`@Sun` on an item sets its day tag. Applying a plan replaces the week.

## Mobile constraints (owner uses an iPhone home-screen icon)
- Zoom fully disabled (viewport meta, `touch-action: pan-x pan-y`, `gesturestart` blocked). Keep it that way.
- Text inputs must be 16px (iOS zooms on focus otherwise).
- Large tap targets; week arrows are 48×48.
- Status bar style `default`. Do not switch to `black-translucent` (causes iOS header blur).
- Dialogs are capped to screen height with sticky action buttons.
- The board is `-webkit-user-select: none` (iOS needs the prefix) so long-press-to-edit doesn't select text; keep it, and the add/edit sheet (`#itemDlg`) too, but never on inputs: `#itemText` must stay `user-select: text` or iOS won't let you type. A `selectionchange` listener also clears any stray non-input selection.

## Workflow
- Test at iPhone SE and iPhone 13 sizes with Playwright. To test without Firebase, copy `index.html` with `FIREBASE_CONFIG = null` (local mode).
- Commit, push to `main`; Pages redeploys in 1–2 minutes.
