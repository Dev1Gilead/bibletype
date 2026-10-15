# CLAUDE.md

Read `PROJECT.md` first. It has the decisions, data shapes, and roadmap.
This file contains the rules to follow while working.

## Working style

- Work only on the current phase (see the roadmap in PROJECT.md). Don't build ahead.
- Keep it simple. This is a solo, 20-year project: fewer dependencies and fewer moving parts win.
  Ask before adding any new dependency or service.
- If a decision isn't covered by PROJECT.md, make the simplest choice, then list it at the
  end of your response so it can be added to PROJECT.md.
- Never introduce a paid service or anything with idle/hourly billing.

## Commands

- `pnpm dev`: local dev server
- `pnpm test`: Vitest
- `pnpm lint` / `pnpm format`: ESLint / Prettier
- `pnpm build-bible`: regenerate `public/bible/` from `data/source/`
- `pnpm run deploy`: build and deploy to Cloudflare Workers (from Phase 3)

Use pnpm only. Never npm or yarn.

## Code organization

- `src/engine/`: pure TypeScript typing engine. **No React, no DOM, no browser APIs.**
  Time is injected (`now: () => number`) so tests are deterministic.
- `src/progress/`: progress store, localStorage adapter, merge function. Pure functions plus a thin storage adapter.
- `src/components/typing/`: the hand-built typing view.
- `src/components/ui/`: shadcn components (menus, dialogs, settings only).
- `src/server/`: server functions. `src/server/db/` is the **only** place that touches Kysely/D1.
- `scripts/`: build-time scripts (e.g. `build-bible.ts`).
- `public/bible/`: generated Bible JSON. Never hand-edit it.

## Typing performance (React rules)

What users feel is keystroke-to-paint latency. It must stay under one frame (~16 ms) at 150+ WPM.

- Engine state lives outside React render state where possible. React subscribes narrowly.
- Only the **current word** re-renders on a keystroke. Finished and upcoming words are memoized and never re-render.
- Move the caret with a CSS `transform` (and a CSS transition for smoothness), not by re-rendering.
- Use a single hidden `<input>` for keyboard capture. Never rely on one input per character.
- No layout-thrashing reads (e.g. `getBoundingClientRect`) inside the keystroke path except for caret positioning, done once per keystroke.
- Never use shadcn components inside the typing surface.
- If this can't hold, the fallback is a plain-DOM renderer for the typing surface (the "middle path"). Don't do this unless profiling proves it's needed.

## Engine rules

- Per-character state: `untyped | correct | incorrect | extra`.
- Extra characters are allowed past the end of a word, up to a cap (e.g. 10).
- Backspace edits within the current word. Ctrl+Backspace deletes the current word.
  Going back into a previous word is allowed only if that word has errors (Monkeytype behavior).
- Space moves to the next word. Skipped characters count as incorrect.
- The timer starts on the first keystroke of the verse.
- Metrics:
  - WPM = (correct characters in correctly completed words + spaces) / 5 / minutes
  - Raw WPM = (all typed characters) / 5 / minutes
  - Accuracy = correct keystrokes / total keystrokes (counted live, including corrected mistakes)
- A verse completes when the last character of the last word is typed correctly.
  It then emits a `VerseResult` (shape in PROJECT.md).

## Sync & cost rules (must follow)

- localStorage is the source of truth while typing. The server is only a backup and sync target.
- Never call the server per keystroke or per verse. Sync only on: session end,
  `visibilitychange` → hidden, or a 3-minute debounce, and only when signed in.
- The sync payload contains only the chapters listed in `dirty`. Clear `dirty` only after the server confirms.
- One D1 row per (user_id, translation, book, chapter), holding compact JSON. **Never one row per verse.**
- Only aggregates are stored and synced (w, a, c, t). **Never keystroke logs.**
- Merge rule: per verse, keep max of each field. The merge must be pure, deterministic, order-independent,
  idempotent, and fully tested. The client and server use the same function.
- Validate every sync request with Zod. Cap the payload size (e.g. 200 chapters per request).
- Write all rows for one sync in a single D1 batch.
- Avoid `SELECT *` and unindexed scans. Always query by primary key prefix (user_id …).
- Bible text is static assets. Never query the DB for verse text.

## localStorage rules

- Key: `progress:v1`. Parse with Zod; if invalid, back up the raw value under
  `progress:corrupt:<timestamp>` and start fresh. Never silently delete user data.
- Wrap every read and write in try/catch. Storage can be unavailable or full.
- A schema change means a new version plus a migration function from the previous version.

## Auth rules (Phase 4)

- Better Auth, Google provider only, stored in D1 via Kysely.
- Signing in is optional. Every feature except sync must work signed out.
- On first sign-in, merge local progress into the account (using the merge rule), then sync.
- Account deletion removes the user's Better Auth rows **and** all their `chapter_progress` rows in one batch.

## Testing

- The engine and merge/progress logic require thorough Vitest coverage: every rule above, plus edge cases
  (punctuation, double spaces, backspacing across words, extra characters, empty storage, corrupt storage).
- Tests never depend on real time. Inject `now`.
- Run `pnpm test` and `pnpm lint` before calling a task done.

## Style

- TypeScript strict mode. No `any` without a comment explaining why.
- Small pure functions over classes, unless state encapsulation is clearly better (the engine may use one).
- Desktop-only: don't spend effort on mobile layouts, but show a friendly
  "please use a desktop browser" message on small screens.
