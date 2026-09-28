# AGENTS.md

Next.js 15 (App Router) + React 19 voice-agent frontend for LiveKit Agents, built on shadcn + Agents UI. **pnpm only** — the lockfile is `pnpm-lock.yaml` and pnpm 9.15.9 is pinned in `packageManager`.

## Commands

```bash
pnpm install          # pnpm install
pnpm dev              # next dev --turbopack -> http://localhost:3000
pnpm lint             # next lint
pnpm format:check     # prettier --check .
pnpm build            # next build — ALSO runs ESLint internally; takes ~5 min
pnpm format           # prettier --write .  (always run before committing)
pnpm shadcn:install   # re-pull @agents-ui/all, then pnpm format
```

CI (`.github/workflows/build-and-test.yaml`) runs exactly `lint -> format:check -> build` on every push and PR to `main`. Match that order locally.

**There is no test suite** — no test script, no runner, no test files, no test deps. Don't add a runner or claim tests passed. Verification here is lint + build.

## Critical: CRLF breaks all three checks on Windows

This checkout has `core.autocrlf=true` and there is **no `.gitattributes`**, so every file arrives on disk as CRLF. Prettier defaults to `endOfLine: lf`, so `pnpm lint`, `pnpm format:check`, and `pnpm build` all fail on ~60 files with errors like ``Delete `␍` `` (prettier/prettier). `git status` stays clean (autocrlf normalizes on commit), so the breakage is invisible until you run a check.

- Don't mass-reformat and don't commit the churn. One-liner: `pnpm format` rewrites to LF and leaves no git diff.
- To verify a change while the checkout is still CRLF: `pnpm exec next build --no-lint`.
- Permanent fix, not yet applied: add `.gitattributes` with `* text=auto eol=lf`.

## Stale docs — trust the source, not the README

`README.md`'s project-structure table lists `session-view.tsx`, `chat-transcript.tsx`, and `tile-layout.tsx`. **None of them exist.** The real wiring:

| File                                                 | Role                                                                                 |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------ |
| `app/page.tsx`                                       | Server component. Resolves token source + feature flags, renders `App`.              |
| `components/app/app.tsx`                             | `'use client'`. Builds `TokenSource`, `useSession`, wraps in `AgentSessionProvider`. |
| `components/app/view-controller.tsx`                 | Welcome <-> session transitions via `AnimatePresence`.                               |
| `components/agents-ui/blocks/agent-session-view-01/` | The real session view.                                                               |

`audioVisualizerType` is a prop on `AgentSessionView_01` / `agent-session-block.tsx` — **not** in `view-controller.tsx`, which is what the README claims.

`.eslintrc.json` is **dead**. ESLint 9 uses the flat `eslint.config.mjs`; edit that. The two also differ — `.eslintrc.json` omits `plugin:import/recommended` and `plugin:prettier/recommended`.

`components.json` points `tailwind.css` at `app/globals.css`, which does not exist. The real stylesheet is `styles/globals.css`, imported in `app/layout.tsx`.

## Env and auth behavior

`app/page.tsx` picks a token source by precedence:

1. `LIVEKIT_TOKEN_SERVER_ID` -> LiveKit Cloud development token server
2. `LIVEKIT_URL` + `LIVEKIT_API_KEY` + `LIVEKIT_API_SECRET` -> local `app/api/token`
3. **neither set** -> connects to the public LiveKit homepage agent on `livekit.com`

Consequence: `pnpm dev` with an empty `.env.local` still works, silently pointed at LiveKit's hosted agent. Copy `.env.example` -> `.env.local` to change that. Blank `AGENT_NAME` means automatic dispatch.

`app/api/token/route.ts` is dev-only by construction — it throws unless `NODE_ENV === 'development'` or `IS_VERCEL_PREVIEW === 'true'`. It has **no authentication** and hardcodes `participantName = 'user'`. Don't ship it unmodified.

`.gitignore` excludes `.env*` but re-includes `.env.example`, so `.env.local` is untracked by design. Keep secrets out of tracked files.

## Toolchain quirks

- **Tailwind v4, CSS-first.** No `tailwind.config.*` exists. Tokens live in `styles/globals.css`: `:root` / `.dark` variables, then `@theme inline`. Dark mode is a custom variant, `@custom-variant dark (&:is(.dark *))`, driven by `next-themes` writing `class` onto `<html>`. `--primary` is the only non-oklch token (`#002cf2` light, `#1fd5f9` dark).
- `styles/globals.css` contains `@source "../node_modules/streamdown/dist/index.js"` so Tailwind scans a dependency's class names. Any new dep shipping classes inside `node_modules` needs its own `@source` entry.
- **`cn` lives at `@/lib/shadcn/utils`**, not shadcn's default `@/lib/utils`. See `components.json` aliases.
- `@/*` maps to the **repo root**, not `./src`. Imports look like `@/components/app/app`.
- `next.config.ts` is empty — no rewrites, no custom server. Dev is Turbopack only.
- Two icon libraries coexist: `@phosphor-icons/react` in app/header code, `lucide-react` in primitives. Match whichever the file you're editing already uses.
- `app/layout.tsx` loads `Public Sans` via `next/font/google` **and** local `CommitMono` `.otf` files from `fonts/`. Both are required; a build without `fonts/` fails.
- `/` ships 488 kB (606 kB first load). Re-check build output after adding a dependency.
- `engines.node` is `24.x` while CI runs Node **22**. There's no `engine-strict`, so pnpm only warns — prefer Node 24 locally.

## Conventions

- Prettier overrides the defaults: `printWidth: 100`, `singleQuote`. Imports are auto-sorted (react, next, third-party, `@/`, relative) and Tailwind classes are sorted by plugin. Run `pnpm format` rather than hand-ordering.
- `components/agents-ui/` and `components/ui/` are **shadcn-managed** and `pnpm shadcn:install` overwrites them. `hooks/agents-ui/` is generated alongside them. Prefer passing props/classes over editing; if you do edit, expect the CLI to ask about overwrites.
- `renovate.json` is active and most recent commits are automated bumps. Don't hand-bump dependencies.
- `hooks/useDebug.ts` exposes the live room on `window.__lk_room` whenever `NODE_ENV !== 'production'` — use it to inspect session state from the console.
