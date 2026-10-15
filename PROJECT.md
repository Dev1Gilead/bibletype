# PROJECT.md — Bible Typing Site

Decisions, architecture, data shapes, and roadmap. `CLAUDE.md` holds the working rules.
When a decision changes, update this file in the same commit.

## What this is

A desktop-only, Monkeytype-style website where users type out the Bible
(KJV and WEB) verse by verse, with per-verse progress, WPM, and accuracy tracking.

- Progress is stored in the browser by default. No account is required.
- Optional "Sign in with Google" syncs progress across devices.
- Personal, long-term project (20-year horizon). Priorities, in order:
  1. Stay free or as cheap as possible.
  2. Low maintenance: few moving parts, few vendors.
  3. Portability: nothing that is hard to move away from.

## Stack (decided 2026-10)

| Area | Choice | Notes |
|---|---|---|
| Framework | TanStack Start (React) | Not Solid. See "Typing performance" in CLAUDE.md. |
| Styling / UI | Tailwind + shadcn/ui | shadcn for menus/settings only. The typing view is hand-built. |
| Charts | Recharts | Deferred until the stats phase. |
| Hosting | Cloudflare Workers | Via `@cloudflare/vite-plugin` + Wrangler. |
| Database | Cloudflare D1 (SQLite) | Only for signed-in users' synced progress and auth tables. |
| DB access | Kysely + `kysely-d1` | All queries go through Kysely so a move to Postgres stays possible. |
| Auth | Better Auth, Google OAuth only | Stored in D1 via its Kysely adapter. Not Firebase. |
| Validation | Zod | At trust boundaries: the sync endpoint and localStorage parsing. |
| Tests | Vitest | The engine and merge logic must be fully covered. |
| Lint / format | ESLint + Prettier | Add OXLint later when the codebase grows. |
| Package manager | pnpm | |

### Rejected (and why)

- **Neon / Postgres:** billed by compute hours while awake. D1 bills per row, with no idle cost.
- **DynamoDB:** not supported by Kysely, awkward for relational queries, heavy AWS lock-in.
- **Supabase:** the free tier pauses after inactivity, and Pro is $25/month.
- **Firebase Auth:** viable and free, but it means two systems (Google plus Cloudflare),
  manual token verification on Workers, and deletes that must happen in both places.
- **Solid:** better default typing performance, but a smaller ecosystem (no shadcn or
  Recharts), less AI-tool fluency, and Solid 2.0 is mid-transition.
- **Express + Mongo + Redis (Monkeytype's approach):** needs an always-on server to pay for and maintain.

### Costs

- Target: **$0/month** on the Cloudflare Workers Free plan.
  - D1 free: 5M rows read/day, 100k rows written/day, 5 GB storage.
  - Static assets (including all Bible JSON) are free and unlimited.
- First step up: **Workers Paid, $5/month.** It includes 25B reads/month, 50M writes/month, and 5 GB storage.
- Only plausible extra cost: a domain name (register it at cost through Cloudflare Registrar).

## Where data lives

| Data | Location |
|---|---|
| Bible text | Static JSON in `public/bible/`, deployed with the site. Never in the DB. |
| Progress (everyone) | Browser localStorage, key `progress:v1` |
| Progress (signed in) | D1 table `chapter_progress` (synced copy) |
| User accounts | D1, Better Auth tables (`user`, `session`, `account`, `verification`) |
| Backups | Weekly `wrangler d1 export` via GitHub Actions, stored outside Cloudflare |
| Code | GitHub repo |

Better Auth's `account` table stores the Google account ID (`providerId = "google"`,
`accountId` = Google subject ID), and `user` stores the email. This keeps us free to
switch auth providers later.

## Bible data

- Translations: `kjv`, `web`. Both are public domain.
- Canon: the 66-book Protestant canon only. The build script skips any other book (Apocrypha, intros, glossaries).
- Source files: eBible.org USFM zips, edition IDs `eng-kjv2006` (KJV) and `engwebp` (WEB),
  unzipped into `data/source/kjv/` and `data/source/web/` (git-ignored).
  `scripts/build-bible.ts` converts them to the app format.
  USFM notes for the parser: drop `\f ... \f*` footnotes and `\x ... \x*` cross-references;
  `\w word|strong="..."\w*` keeps only `word`; `\add ...\add*` (KJV supplied words) keeps the text.
- Book IDs: 3-letter USFM codes (`GEN`, `EXO`, … `REV`), uppercase.
- Output: `public/bible/{translation}/{BOOK}/{chapter}.json`

```json
[{ "v": 1, "t": "In the beginning God created the heaven and the earth." }]
```

- Also generate `public/bible/{translation}/index.json`: a list of books with their names and chapter counts.
- Text normalization (done at build time, never at runtime):
  - Curly quotes become straight quotes (`’` → `'`, `“ ”` → `"`).
  - Em/en dashes become `-`, and the ellipsis character becomes `...`.
  - Strip footnotes, cross-references, Strong's numbers, and markup.
  - KJV italic (supplied-word) markers are stripped; keep the words.
  - Collapse whitespace and trim.
  - Result: every character can be typed on a standard US keyboard.

## Data shapes

### Engine output

```ts
type VerseResult = {
  translation: "kjv" | "web";
  book: string;        // USFM code
  chapter: number;
  verse: number;
  wpm: number;         // net WPM
  rawWpm: number;
  accuracy: number;    // 0–1
  durationMs: number;
  completedAt: number; // epoch ms
};
```

### Stored progress (localStorage `progress:v1`, and the JSON in D1)

```ts
type VerseStats = { w: number; a: number; c: number; t: number };
// w = best WPM, a = best accuracy (0–1), c = completions, t = last completed (epoch ms)

type ChapterProgress = {
  verses: Record<string, VerseStats>; // key = verse number
  updatedAt: number;
};

type ProgressStore = {
  version: 1;
  chapters: Record<string, ChapterProgress>; // key = "kjv/GEN/1"
  dirty: string[]; // chapter keys changed since last successful sync
};
```

### D1 table (Phase 4)

```sql
CREATE TABLE chapter_progress (
  user_id     TEXT    NOT NULL,
  translation TEXT    NOT NULL,
  book        TEXT    NOT NULL,
  chapter     INTEGER NOT NULL,
  data        TEXT    NOT NULL,   -- JSON: ChapterProgress["verses"]
  updated_at  INTEGER NOT NULL,
  PRIMARY KEY (user_id, translation, book, chapter)
);
```

There are no extra indexes; the primary key covers every query
("all rows for user X", "one chapter for user X").

### Merge rule (client and server use the same pure function)

For each verse: `w = max`, `a = max`, `c = max`, `t = max`.
The merge is deterministic, order-independent, and idempotent: merging the same data twice changes nothing.

## Roadmap

1. **Phase 1: Engine.** The data-prep script (sample chapters), the pure TS engine with tests,
   and a minimal typing view saving to localStorage. No backend.
2. **Phase 2: The app.** Generate all chapters, add a book/chapter picker, progress overview
   (completed verses per chapter/book), translation switch, settings, and theme.
3. **Phase 3: Deploy.** Ship to Cloudflare Workers (static only) and connect a custom domain.
4. **Phase 4: Accounts and sync.** Better Auth (Google), the D1 `chapter_progress` table, the sync
   endpoint, the merge, and the weekly backup GitHub Action.
5. **Phase 5: Stats.** Recharts pages: WPM over time, accuracy, per-book completion.
6. **Later / maybe:** OXLint, a plain-DOM typing renderer (the "middle path") if profiling
   shows React jank, and more translations (public domain only).

## Decision log

- 2026-10-12: Chose Cloudflare Workers + D1 over Neon/Supabase/AWS for $0 idle cost.
- 2026-10-14: Stayed with React over Solid. Switched npm → pnpm. Chose Better Auth over Firebase Auth.
