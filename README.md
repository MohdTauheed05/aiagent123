# AI Office HQ

A living virtual office where AI agents work as departments to complete a
task you give them. Give the office a brief — "create a children's story
video about a brave rabbit" — and watch Orion (the manager) break it down
and hand pieces to Luna, Pixel, Motion, Echo, Frame, and Nova in turn.

This MVP runs entirely in **Demo Mode** by default: no API keys, no
Firebase project, no auth wall. Clone it, `npm install`, `npm run dev`, and
the whole office comes alive with mock output. Real providers slot in later
without touching any UI or orchestration code.

## Quick start

```bash
npm install
npm run dev
```

Open http://localhost:3000. Type a brief in the task panel and press
**Start AI team**.

## How it's put together

```
app/                     Next.js App Router pages
components/
  office/                 The office floor, department rooms, avatars, side panel
  task-panel/             The brief/priority/agent-mode input
  activity-panel/         Live activity feed
  dashboard/               Top nav, final output view
lib/
  agents/                 One file per specialist agent + the manager's planner
    manager.ts             Turns a brief into an ordered list of subtasks
    story.ts, image.ts,
    video.ts, voice.ts,
    editor.ts, publisher.ts  Each agent's execution logic
    roster.ts               Static agent definitions (name, color, capabilities)
    base.ts                 Shared helpers every agent uses
  orchestrator/
    orchestrator.ts         Drives subtasks through agents, retries failures,
                            updates the store and activity log in real time
  ai/
    provider.ts             AI provider abstraction — Demo, OpenAI, Anthropic,
                            Gemini. Add real implementations here.
  firebase/
    config.ts               Optional Firebase init — no-ops if unconfigured
  store.ts                  Zustand store: single source of truth for the UI
  theme.ts                  Status color constants
types/index.ts             Shared TypeScript types for the whole app
```

### The pipeline

For a full video brief, the Manager Agent (Orion) plans a linear pipeline:

```
Orion (plan) → Luna (story) → Pixel (images) → Motion (video)
             → Echo (voice) → Frame (edit) → Nova (publish)
```

`lib/agents/manager.ts` builds this plan from the brief. It's currently a
small heuristic (looks for "just write the story" vs. a full video ask); the
comment there shows exactly where to swap in a real planning LLM call.

`lib/orchestrator/orchestrator.ts` walks the plan: for each subtask it moves
the agent through `receiving_task → thinking → working → completed`,
updating progress, writing to the activity log, and retrying (up to 2 times)
on failure before marking the whole project failed. Demo Mode injects an
occasional simulated failure so the retry/error UI has something real to
show — remove `simulateMaybeFail()` once you're on a real provider.

### Demo Mode vs. real providers

Every agent calls out through `resolveProvider()` in `lib/ai/provider.ts`,
never a vendor SDK directly. Demo Mode (the toggle in the top nav, or
`NEXT_PUBLIC_DEMO_MODE` in `.env.local`) returns instant mock text/image/
video/audio so the whole system — planning, delegation, retries, the
office animations — is testable before you have a single API key.

To connect a real provider — no code changes needed, just keys:

1. Add your key(s) to `.env.local` (see `.env.example`).
2. Set `NEXT_PUBLIC_AI_PROVIDER` to `openai`, `anthropic`, or `gemini`.
3. Turn off Demo Mode (toggle in the nav, or `NEXT_PUBLIC_DEMO_MODE=false`).

What's already wired up for real, server-side, in `app/api/ai/*/route.ts`:

| Capability | OpenAI | Anthropic | Gemini |
|---|---|---|---|
| Text (planning, scripts, prompts) | ✅ `gpt-4o-mini` | ✅ `claude-sonnet-4-6` | ✅ `gemini-1.5-flash` |
| Images (Pixel's department) | ✅ DALL·E 3 | ❌ needs a paired provider | ❌ needs Vertex/Imagen setup |
| Audio / narration (Echo's department) | ✅ TTS (`gpt-4o-mini-tts`) | ❌ needs a paired provider | ❌ not wired |
| Video (Motion's department) | ❌ not offered by OpenAI | ❌ | ❌ |

Because no single vendor covers every department, you can mix providers —
e.g. Anthropic for `generateText` (better planning/story quality) plus
`NEXT_PUBLIC_AI_PROVIDER=openai` left in place for Pixel/Echo — by editing
the provider map at the bottom of `lib/ai/provider.ts`. Video has no real
implementation anywhere yet: add a dedicated video-generation service (e.g.
Runway, Pika, Luma) behind a new `app/api/ai/video/route.ts`, following the
exact same pattern as `/api/ai/audio`.

### Connecting Firebase (optional)

The app runs entirely on the local Zustand store with Firebase unset. To
turn on real persistence, real-time sync across users, and login:

1. Create a Firebase project → Firestore, Storage, and Authentication
   (enable Google + Email/Password sign-in methods).
2. Copy your web app config into `.env.local` (see `.env.example` for the
   six `NEXT_PUBLIC_FIREBASE_*` vars).
3. `isFirebaseConfigured()` in `lib/firebase/config.ts` will start returning
   `true` automatically — that's the flag to gate any Firestore/Storage/Auth
   code you add against, so local dev without a project keeps working.
4. Suggested next step: mirror `useOfficeStore`'s project/subtask writes into
   a `projects` Firestore collection with `onSnapshot` listeners, so state
   survives a refresh and syncs across tabs/users.

Suggested collections, matching the brief: `users`, `agents`, `tasks`,
`projects`, `taskActivities`, `agentActivities`, `outputs`, `settings`.

### Adding a new agent / department

1. Add its definition to `AGENT_ROSTER` in `lib/agents/roster.ts` (id, name,
   role, department, color, capabilities, desk description).
2. Add an `AgentId` union member and any new `Department` value in
   `types/index.ts`.
3. Write its execution module in `lib/agents/`, following the shape of
   `story.ts` (a function that takes a `SubTask` + brief and returns
   `{ summary, outputs }`).
4. Register it in the `EXECUTORS` map in `lib/orchestrator/orchestrator.ts`.
5. Add it to a plan in `lib/agents/manager.ts` (or a new plan variant).

The office UI picks up new department rooms automatically from
`AGENT_ROSTER` — no changes needed in `components/office/`.

### Restoring the custom typefaces

This build ships with system-font stacks (`app/globals.css` → the
`--font-*` variables) because the sandbox this was built in couldn't reach
Google Fonts. The design was planned around **Sora** (display), **Inter**
(body), and **IBM Plex Mono** (timestamps/data labels). To restore them
once you have normal internet access:

```tsx
// app/layout.tsx
import { Sora, Inter, IBM_Plex_Mono } from "next/font/google";

const sora = Sora({ variable: "--font-sora", subsets: ["latin"], weight: ["500","600","700"] });
const inter = Inter({ variable: "--font-inter", subsets: ["latin"] });
const plexMono = IBM_Plex_Mono({ variable: "--font-plex-mono", subsets: ["latin"], weight: ["400","500"] });
```

and point the `--font-sans` / `--font-display` / `--font-mono` variables in
`globals.css` at `var(--font-inter)`, `var(--font-sora)`, and
`var(--font-plex-mono)` respectively (they currently point at system stacks
directly — the commented-out originals are visible in git history / the
component comments).

## Design notes

- Colors are functional, not decorative: each department has one accent
  color used consistently for its avatar ring, badge, progress bar, and
  connector line — like a building directory, not a gradient wash.
- The one deliberate animated moment is the task token traveling along the
  connector line from Orion to whichever department is currently active;
  everything else (hover states, panel opens) is quiet on purpose.
- `prefers-reduced-motion` is respected globally via Framer Motion's
  `<MotionConfig reducedMotion="user">` in `app/layout.tsx` — this is what
  actually covers the walking robots, status pulses, and connector-line
  token, none of which are plain CSS animations. `globals.css` has a
  matching rule for anything that *is* plain CSS.

## Scripts

```bash
npm run dev      # start the dev server
npm run build    # production build
npm run start    # run the production build
npm run lint     # eslint
```

## Known limitations (MVP)

- The pipeline is linear (each subtask depends only on the one before it).
  `SubTask.dependsOn` is an array specifically so a future planner can
  express fan-out/fan-in (e.g. image + voice running in parallel).
- Demo Mode's image/video/audio "outputs" are placeholder files in
  `public/demo-assets/`, not real generated media. With Demo Mode off and a
  real provider configured, images and audio are real (rendered inline in
  the final output card); video has no real provider wired up anywhere yet
  (see the provider table above) and will fail with a clear error until you
  add one.
- Real audio is returned as a base64 data URI, not a Storage URL — fine for
  an MVP, but for real traffic you'd want `app/api/ai/audio/route.ts` to
  upload the bytes to Firebase Storage/S3 and return a short-lived URL
  instead (keeps API responses small and lets outputs survive a refresh via
  the same Firestore project sync that already persists everything else).
- There's no auth wall in dev, by design (per the brief). Add a real guard
  in `app/layout.tsx` once Firebase Auth is wired up, if you want one.
