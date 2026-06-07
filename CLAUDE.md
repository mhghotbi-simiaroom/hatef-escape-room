# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm install        # install dependencies
npm run dev        # start dev server at http://localhost:3000
npm run build      # production build
npm run lint       # ESLint via Next.js
npm run start      # serve production build
```

No test suite is configured. Type-check manually with `npx tsc --noEmit`.

## Environment

Copy `.env.example` to `.env.local`. Layer 5 (prompt engineering) needs an AI backend — three modes in priority order:

1. **OpenRouter** (`OPENROUTER_API_KEY`) — default model `google/gemma-4-31b-it:free`, auto-falls through a chain of free models on 429s
2. **Anthropic** (`ANTHROPIC_API_KEY`) — uses `claude-opus-4-8` if OpenRouter key absent
3. **Offline** — no keys required; a local heuristic evaluates the prompt instead

The game is fully playable in offline mode.

## Architecture

**Stack:** Next.js 15 (App Router) · TypeScript · Tailwind CSS · Framer Motion · Tone.js · Zustand · `@anthropic-ai/sdk`

### Route layout

| Path | Purpose |
|------|---------|
| `app/page.tsx` | Landing / intro screen |
| `app/game/page.tsx` | Main game orchestrator (mounts puzzle per `currentLayer`) |
| `app/api/oracle/route.ts` | Server-side route for Layer 5 — calls OpenRouter or Claude, validates acrostic |

### State (`lib/store.ts`)

Single Zustand store (`useGame`) persisted to `localStorage` under key `hatef-escape-room`. Tracks: `phase` (`intro | playing | win | lose`), `currentLayer` (1–5), `solvedLayers`, `startTimestamp`, `penaltySeconds`, `hintsUsed*`, `soundOn`, `finalRemaining`.

Timer is computed on-the-fly (`remainingSeconds()`) from `startTimestamp + penaltySeconds` — not stored as a countdown. Each hint costs 90 s (`HINT_PENALTY_SECONDS`). Total time is 10 min (`TOTAL_TIME_SECONDS`).

### Puzzle data (`lib/puzzles.ts`)

Single source of truth for all game content (Persian text, puzzle data, answers, hints). Exports per-layer types and constants:

- **Layer 1 — Tokenization:** `layer1Tokens` (Token array with `isKey`/`keyDigit`), answer `layer1LockCode = "7391"`
- **Layer 2 — Embedding:** `layer2Clusters` (Cluster array with `intruder`/`revealLetter`), answer `layer2Password = "نگار"`
- **Layer 3 — Attention:** `layer3Sentence` (AttentionWord array with `role`), answer `layer3Answer` (pronoun→entity id map)
- **Layer 4 — Generation:** `layer4Steps` (GenStep array with probabilistic word choices), final sentence exported separately
- **Layer 5 — Prompt Engineering:** `layer5Goal` (topic, acrostic target `"رها"`, line count)

### Component structure

```
components/
  puzzles/          # One component per layer (Layer1–5)
  screens/          # Intro, Win, Lose
  LayerShell.tsx    # Wrapper: shows Hatef narration → reveals puzzle → shows outro on solve
  Hud.tsx           # Timer + score bar
  HatefSpeech.tsx   # Animated dialogue bubbles
  HatefOrb.tsx      # Animated AI avatar
  SceneBackground.tsx # Background image with gradient fallback
  ProgressLocks.tsx # Five lock icons showing solved state
```

Each puzzle component receives an `onSolved` callback from `LayerShell`; calling it triggers `useGame.solveLayer(id)`.

### Layer 5 oracle flow (`app/api/oracle/route.ts`)

`POST /api/oracle` → tries OpenRouter models in sequence → if all fail, tries Anthropic → if no keys, uses offline evaluator. Response shape: `{ success, output, message, mode: "live"|"offline" }`. Acrostic validation logic lives in `lib/acrostic.ts` (shared between server route and offline fallback).

### Audio (`lib/audio.ts`)

Generates ambient music procedurally with Tone.js. If `public/audio/ambient.mp3` exists it takes precedence. All assets in `public/images/` are optional — missing images fall back to CSS gradients.

## Key conventions

- All game text is Persian (RTL). UI uses `dir="rtl"` globally.
- Puzzle answers/hints are hardcoded in `lib/puzzles.ts` — edit there to change game content.
- `LayerId` is a union type `1 | 2 | 3 | 4 | 5`, not a plain number.
- The oracle API route uses `export const runtime = "nodejs"` (not Edge) because `@anthropic-ai/sdk` requires Node.
