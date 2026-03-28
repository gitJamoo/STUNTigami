# STUNTigami — Scorigami Website for NCAA STUNT

## Context

STUNT is an NCAA Emerging Sport (promoted to Championship status January 2026) where two teams compete head-to-head across 4 quarters. Each match produces a final score per team (e.g., 16–14). **STUNTigami** is a Scorigami-style visualization showing every score combination that has ever occurred in STUNT history — inspired by Jon Bois's NFL scorigami concept.

This is the `website/` package in a monorepo. Everything needs to be built from scratch inside this directory.

## STUNT Scoring

- **Q1–Q3**: 4 routines each (Partner Stunts, Pyramids/Tosses, Jumps/Tumbling) — 1 pt per routine won
- **Q4**: 3 routines (Team performance) — up to 3 pts per routine
- **Theoretical max per team**: ~21 pts (12 from Q1-Q3 + 9 from Q4)
- **Ties**: Possible at both routine and match level
- **Note**: Grid range should be data-driven (start at 0–25, expand if needed)
- **Example real score**: 16–14 (Oklahoma State vs. Alma College)

## Tech Stack

- **Next.js 16** (App Router, TypeScript, Turbopack)
- **Tailwind CSS v4** + **shadcn/ui** (zinc/dark theme, `className="dark"` on `<html>`)
- **Data**: Static `data/matches.json` → future Neon Postgres migration via `lib/data.ts` seam
- **Deployment**: Vercel

---

## Implementation Plan

### Phase 0: Bootstrap (run from repo root)

```bash
npx create-next-app@latest website --yes
# (TypeScript, Tailwind, ESLint, App Router, Turbopack, @/* alias)

cd website
pnpm dlx shadcn@latest init -t next
# Accept defaults: style=default, base color=zinc, CSS variables=yes

pnpm dlx shadcn@latest add tooltip card badge separator button
pnpm add clsx
```

---

### Phase 1: Types and Data Layer

**`/types/match.ts`**
```ts
export interface Match {
  id: string;            // "2025-04-12-usc-ucla"
  date: string;          // "2025-04-12"
  season: string;        // "2024-25"
  homeTeam: string;
  awayTeam: string;
  homeScore: number;
  awayScore: number;
  winnerScore: number;   // max(home, away) — pre-computed at seed time
  loserScore: number;    // min(home, away)
  isTie: boolean;
  source?: string;       // "collegestunt.org" | "varsitytv.com"
  sourceUrl?: string;
}
```

**`/types/scorigami.ts`**
```ts
export interface CellData {
  winnerScore: number;
  loserScore: number;
  count: number;
  matches: Match[];
  isImpossible: boolean;  // loserScore > winnerScore
  isTie: boolean;
}

export interface ScorigamiStats {
  totalGames: number;
  uniqueScores: number;
  possibleScores: number;      // computed from actual score range
  mostCommonScore: CellData | null;
  recentScorigami: Match | null;
  topOccurrences: CellData[];  // top 5
}
```

**`/data/matches.json`**
Array of seed matches from real STUNT results. Manually verified from collegestunt.org and university athletics pages.

**`/lib/data.ts`** — single data seam (swap for DB here later)
```ts
import matchData from '@/data/matches.json';
export async function getMatches(): Promise<Match[]> {
  'use cache'
  return matchData as Match[];
}
```

**`/lib/scorigami.ts`**
- `buildFrequencyMap(matches)` → `Map<"winnerScore-loserScore", Match[]>`
- `getScoreRange(matches)` → `{ min: 0, max: number }` (data-driven upper bound)
- `buildGrid(map, max)` → `CellData[][]` — (max+1)×(max+1) matrix, `isImpossible` where loserScore > winnerScore
- `buildStats(matches, grid)` → `ScorigamiStats`

**`/lib/colorScale.ts`**
```ts
export function getScoreColor(count: number, maxCount: number): string
// 0 occurrences → zinc-900 (dark/empty)
// 1–2 → muted blue (#1e3a5f)
// 3–5 → sky (#0ea5e9)
// 6–10 → amber (#f59e0b)
// 11+ → orange → red gradient
```

---

### Phase 2: Next.js Config and Layout

**`/next.config.ts`**
```ts
const nextConfig: NextConfig = { cacheComponents: true }
```

**`/app/globals.css`** — add dark mode variant after shadcn init:
```css
@custom-variant dark (&:where(.dark, .dark *));
```

**`/app/layout.tsx`** — `<html className="dark">`, dark background, Geist Sans font

---

### Phase 3: Home Page

**`/app/page.tsx`** (Server Component)
- Calls `getMatches()`, `buildFrequencyMap()`, `buildGrid()`, `buildStats()`
- Computes `maxCount` (for color scaling)
- Passes serialized data to `<ScorigamiPageClient>`

**`/app/loading.tsx`** — Animated skeleton grid + stat cards

---

### Phase 4: Components

#### Scorigami Components (`/components/scorigami/`)

| File | Type | Purpose |
|------|------|---------|
| `ScorigamiPageClient.tsx` | Client | Owns selected-cell state, orchestrates layout |
| `ScoreGrid.tsx` | Client | CSS Grid rendering the (max+1)×(max+1) matrix |
| `ScoreCell.tsx` | Client | Individual cell button + shadcn Tooltip trigger |
| `CellTooltip.tsx` | Presentational | Score, count, recent games list |
| `EmptyCell.tsx` | Presentational | Impossible score placeholder |
| `AxisLabels.tsx` | Presentational | Y-axis (winner) and X-axis (loser) numeric labels |
| `StatsPanel.tsx` | Presentational | 4 stat cards (total games, unique scores, most common, recent scorigami) |
| `CellDetailPanel.tsx` | Presentational | Full game list for selected cell; "Scorigami!" banner if count === 1 |

**Grid layout logic:**
```tsx
// Y-axis: winner score (top = max, bottom = 0)
// X-axis: loser score (left = 0, right = max)
// Impossible cells (loserScore > winnerScore) → <EmptyCell />
// Ties (winnerScore === loserScore) → distinct dashed border styling
<div style={{ gridTemplateColumns: `repeat(${max+1}, minmax(0, 1fr))` }}>
```

#### Site Components (`/components/site/`)
- `SiteHeader.tsx` (client, `usePathname`) — logo, nav to `/about` + `/data`, GitHub link
- `SiteFooter.tsx` — data source attribution, last-updated, fan project disclaimer

---

### Phase 5: Additional Pages

**`/app/(info)/about/page.tsx`** — What is STUNT, what is Scorigami, data sources, how to submit corrections

**`/app/(info)/data/page.tsx`** — Sortable table of all recorded games (sort via `?sort=date&dir=desc` URL params)

**`/app/api/matches/route.ts`** — `GET` returns raw match JSON for future tooling/integrations

---

### Phase 6: Data Collection Strategy

**Immediate (seed):** Manually compile matches from:
- collegestunt.org results pages
- Varsity TV competition results
- University athletics pages (OSU, UCLA, USC, etc.)

**Future scraper** (`/scripts/scrape-collegestunt.ts`, not part of Next.js build):
- `cheerio` + `node-fetch` → parse collegestunt.org results
- Output JSON to stdout for human review before merging to `data/matches.json`

**Future DB** (`lib/data.ts` seam): Replace JSON import with Neon Postgres query; add `DATABASE_URL` via `vercel env add`

---

## File Structure

```
website/
├── app/
│   ├── (info)/about/page.tsx
│   ├── (info)/data/page.tsx
│   ├── api/matches/route.ts
│   ├── globals.css
│   ├── layout.tsx
│   ├── loading.tsx
│   ├── not-found.tsx
│   └── page.tsx
├── components/
│   ├── scorigami/
│   │   ├── AxisLabels.tsx
│   │   ├── CellDetailPanel.tsx
│   │   ├── CellTooltip.tsx
│   │   ├── EmptyCell.tsx
│   │   ├── ScoreCell.tsx
│   │   ├── ScoreGrid.tsx
│   │   ├── ScorigamiPageClient.tsx
│   │   └── StatsPanel.tsx
│   └── site/
│       ├── SiteFooter.tsx
│       └── SiteHeader.tsx
├── data/
│   └── matches.json
├── lib/
│   ├── colorScale.ts
│   ├── data.ts
│   └── scorigami.ts
├── types/
│   ├── match.ts
│   └── scorigami.ts
├── scripts/
│   └── scrape-collegestunt.ts  (future, not built by Next.js)
├── next.config.ts
├── components.json
└── PLAN.md
```

---

## Verification

1. **Dev server**: `pnpm dev` → localhost:3000 shows dark scorigami grid
2. **Grid correctness**: All cells where `loserScore > winnerScore` are empty/impossible; diagonal (ties) has distinct styling
3. **Color scale**: Empty cells dark, seeded matches show color gradient based on frequency
4. **Tooltip**: Hovering a cell shows score + count + game list (or "First occurrence would be a Scorigami!")
5. **Stats panel**: Correct counts match seed data
6. **Routes**: `/`, `/about`, `/data`, `/api/matches` all load without errors
7. **Vercel deploy**: `vercel` → preview URL works
