# Phase 1 prompt

Paste everything below the line into Claude Code after finishing SETUP.md.

---

Read CLAUDE.md and PROJECT.md. We are doing **Phase 1: the typing engine** only.
No backend, no auth, no database, no charts. Follow every rule in CLAUDE.md.

Do these in order, and stop for my review after each numbered step:

1. **Data-prep script** (`scripts/build-bible.ts`, run with `pnpm build-bible`):
   - Read the source files from `data/source/kjv/` and `data/source/web/`.
     First, look at what's in those folders and tell me the format before writing the parser.
   - Apply every normalization rule in PROJECT.md ("Bible data"), including skipping
     any book outside the 66-book canon.
   - Output `public/bible/{translation}/{BOOK}/{chapter}.json` and `public/bible/{translation}/index.json`.
   - For now, generate only GEN 1, PSA 23, and JHN 1 for both translations.
   - Add a small test checking that the normalized output contains only typeable ASCII characters.

2. **Pure engine** in `src/engine/` (no React, no DOM):
   - Implement every rule under "Engine rules" in CLAUDE.md.
   - Inject time (`now: () => number`).
   - Emit `VerseResult` on completion (shape in PROJECT.md).

3. **Engine tests** (Vitest): cover every engine rule plus edge cases (punctuation,
   double spaces, backspacing across words, extra-character cap, Ctrl+Backspace,
   skipping with space, timer start, WPM/raw WPM/accuracy math with controlled time).

4. **Progress store** in `src/progress/`:
   - `ProgressStore` (shape in PROJECT.md), a localStorage adapter following the
     "localStorage rules", and the pure `merge` function.
   - Tests for merge (order-independent, idempotent) and for corrupt or missing storage.

5. **Minimal typing view** (React, following "Typing performance"):
   - One route that loads a chapter JSON and lets me type it verse by verse.
   - Show live WPM and accuracy, a smooth caret, and the verse reference.
   - Save each `VerseResult` into the progress store.
   - A simple dropdown to pick between the 3 sample chapters and the 2 translations.

When finished, list any decisions you made that aren't covered in PROJECT.md.
