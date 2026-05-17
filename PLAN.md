# IELTS Ninja — Implementation Plan

## Context

Greenfield project at `/Users/sudarshaana/Desktop/Plans/ielts_ninja` (empty dir). User building AI-assisted IELTS **Academic** prep web app. MVP ships **Writing module only**; Listening, Reading, Speaking come later but architecture must extend cleanly. Goal: personalized practice with AI band scoring, weakness detection, and progress tracking. Free during MVP (no payment yet).

## Confirmed Requirements

| Aspect | Choice |
|---|---|
| Test type | IELTS Academic only |
| Stack | Next.js 16+ (App Router, TS) + Postgres 16 + Prisma ORM + Anthropic Claude (primary) |
| MVP scope | Writing only (Task 1 + Task 2) |
| Auth | Better-Auth — Google OAuth + email magic link (Resend). DB sessions. No payment yet. |
| Personalization | Target band + per-criteria weakness detection |
| Progress UI | Band trend chart + criteria radar + attempt history with diffs |
| Writing features | Prompts library, AI band scoring + per-criteria feedback, rewrite + model answers, timed mode + word counter, paste-disable |

## Tech Stack (pinned)

Next.js 16+ · TypeScript 5.6 · Node 20 LTS · Postgres 16 · **Prisma 5.22** · Tailwind 3.4 · shadcn/ui · **Better-Auth 1.x** · Anthropic SDK 0.32 (model `claude-sonnet-4-6`) · Tiptap 2.10 · Recharts 2.13 · Zod 3.23 · pnpm.

**Notes on stack choices (per user preference):**
- **Prisma** over Drizzle: richer migrations workflow (`prisma migrate dev/deploy`), Prisma Studio for DB inspection, mature client API. Slight runtime cost vs Drizzle accepted.
- **Better-Auth** over Auth.js: framework-agnostic, first-class plugin system (organizations, 2FA, magic link, OAuth all as plugins), Prisma adapter, type-safe client. Cleaner DX than NextAuth in our experience.
- **Next.js 16+**: latest App Router, Turbopack stable, React 19, async request APIs (`cookies()`, `headers()`, `params`). All server-component patterns below assume Next 16 conventions.

## File Structure

```
ielts_ninja/
  app/
    (auth)/login, signup
    api/auth/[...nextauth]/route.ts
    api/ai/score/route.ts                # streaming, maxDuration=300
    dashboard/{page,history,settings}/page.tsx
    writing/page.tsx                     # prompt library
    writing/[promptId]/page.tsx          # start attempt
    writing/attempt/[attemptId]/page.tsx           # timed editor
    writing/attempt/[attemptId]/feedback/page.tsx  # scored result
  components/
    ui/                                  # shadcn
    writing/{Editor,Timer,AnnotationOverlay,RewriteDiff}.tsx
    progress/{BandTrendChart,CriteriaRadar,AttemptHistoryTable}.tsx
  lib/
    ai/
      provider.ts                        # AIProvider interface (swap seam)
      anthropic.ts                       # Claude impl
      openai.ts                          # fallback impl (off)
      prompts/{writing-rubric.md,writing-fewshot.json,writing-score.ts,writing-rewrite.ts}
      schemas.ts                         # Zod for tool-use outputs
      cache.ts                           # cache_control helpers
      budget.ts                          # token caps + rate limits
    auth/
      auth.ts                            # Better-Auth server instance (betterAuth({...}))
      client.ts                          # Better-Auth React client (createAuthClient)
      session.ts                         # requireUser() helper
    db/
      prisma.ts                          # PrismaClient singleton
      queries/{attempts,progress,prompts}.ts
    scoring/{weakness,target}.ts         # pure functions, testable
    validation/writing.ts
    utils/{diff,wordcount}.ts
  prisma/
    schema.prisma                        # full data model
    migrations/                          # prisma migrate output
    seed.ts                              # ts-node seed for prompts
  middleware.ts                          # auth gate (Better-Auth session check)
  env.ts                                 # t3-env style zod-validated
```

**Server Actions vs Route Handlers**: actions for fast CRUD (create attempt, autosave, set target). Route handler only for `/api/ai/score` (streaming + long runtime) and Auth.js.

## Data Model (Prisma / Postgres)

**Decision**: per-section tables, not generic polymorphic `sections/questions`. Reason: question shapes diverge wildly across sections (writing = free text, listening/reading = 14 distinct Q types, speaking = audio). Shared concepts (`User`, `Attempt`, `Score`, `Annotation`) stay normalized.

**Better-Auth-owned tables** (managed by Better-Auth schema generator; do not hand-edit beyond extending `User`):
- `User` — Better-Auth core fields (id, email, name, image, emailVerified, createdAt, updatedAt). **Extended** with `targetBand Decimal? @db.Decimal(2,1)` and `examDate DateTime?` via Better-Auth's `user.additionalFields` config.
- `Session` — Better-Auth managed (token, expiresAt, ipAddress, userAgent, userId)
- `Account` — Better-Auth managed (provider, providerAccountId, accessToken, refreshToken, userId)
- `Verification` — Better-Auth managed (magic link tokens)

**App tables** (in `prisma/schema.prisma`):
- `WritingPrompt` — id, task `Task` enum (TASK1|TASK2), title, body, imageUrl?, topic, difficulty `Difficulty` enum, wordTarget Int, source?, createdAt
- `Attempt` — id, userId (relation), **section `Section` enum** (WRITING|LISTENING|READING|SPEAKING), promptId?, status `AttemptStatus` enum (IN_PROGRESS|SUBMITTED|SCORED|FAILED), startedAt, submittedAt?, durationSec?, meta Json
- `WritingAttempt` — attemptId @id (1:1 Attempt), promptId (relation WritingPrompt), responseText, wordCount, pasteDetected Boolean @default(false)
- `Score` — id, attemptId, criterion String ('task_achievement'|'coherence'|'lexical'|'grammar'|'overall'), band Decimal @db.Decimal(2,1), rationale String, modelVersion String, usage Json?, createdAt. `@@unique([attemptId, criterion])`
- `Annotation` — id, attemptId, startOffset Int, endOffset Int, category String, severity `Severity` enum (INFO|SUGGESTION|ERROR), comment String, suggestion String?
- `Rewrite` — id, attemptId, kind `RewriteKind` enum (IMPROVED|MODEL_ANSWER), text String, diff Json
- Future stubs (design now, build later): `ListeningTest/Question/Answer`, `ReadingPassage/Question/Answer`, `SpeakingPrompt/Attempt`

**Prisma indexes**:
```prisma
@@index([userId, section, submittedAt(sort: Desc)]) // on Attempt
@@index([attemptId])                                 // on Score
@@index([task, difficulty])                          // on WritingPrompt
```

**Better-Auth + Prisma wiring**: use Better-Auth's `prismaAdapter(prisma)` in `lib/auth/auth.ts`. Run `npx @better-auth/cli generate` to emit auth-managed Prisma models, then `prisma migrate dev` to apply. App models are hand-authored in the same `schema.prisma`.

## AI Integration

**Primary**: Claude Sonnet 4.6 (`claude-sonnet-4-6`). Reasons: strong rubric adherence, native **prompt caching** (huge cost lever — rubric + few-shot are static ~10k tokens), tool use gives strict JSON. OpenAI wired as fallback behind `AIProvider` interface.

**Scoring call** (single request, tool use `submit_band_scores`):
- System: examiner role + full rubric (cached) + 6-10 graded few-shot exemplars covering bands 5.0-8.5 (cached, `cache_control: ephemeral`)
- User: task type, prompt, candidate response, word count
- Output schema (Zod-validated): per-criterion `{band, rationale}` for 4 criteria + `overall_band` + `annotations[{start_offset, end_offset, category, severity, comment, suggestion?}]` + `summary`
- Annotations returned in same call; validate offsets in range, drop invalid

**Rewrite call** (lazy — on first feedback view):
- Two tool calls one request: `improved_version` (minimal edits raising ~1.0 band), `model_answer` (band 8.5+)
- Server computes diff with `diff-match-patch`, stores `diffJson`

**Cost guardrails** (`lib/ai/budget.ts`):
- Hard cap input: 600 words (Task 2 target 250)
- Per-user daily limit: 10 scored attempts (configurable)
- `max_tokens: 2000` on scoring response
- Log `usage` to `scores.meta.usage` per attempt
- Env kill-switch: `AI_DISABLE=true` returns demo result

## Writing Module UX Flow

1. `/writing` — prompt library, filters (task, topic, difficulty), server component
2. `/writing/[promptId]` — detail + "Start timed attempt" → server action creates `attempts` row
3. `/writing/attempt/[attemptId]` — **Tiptap editor** (client). Timer from `attempts.startedAt` (reload-resistant). Word counter via `@tiptap/extension-character-count`. Paste blocked via `editorProps.handlePaste`. Autosave every 15s via server action. Auto-submits at deadline (Task1 20min, Task2 40min).
4. `/writing/attempt/[attemptId]/feedback` — band scores, radar, annotated text overlay (Tiptap `Mark` extension on char offsets), tabs: Your text / Improved / Model answer (lazy-load rewrite on first visit)
5. `/dashboard/history` — paginated table, sparkline column

**Editor: Tiptap, not textarea**. Reason: annotation overlay needs char-offset highlights (Tiptap Marks/decorations API), future-proof for Reading passage rendering. Lock formatting marks for IELTS realism.

**Paste defense is best-effort** (a11y — keyboard paste must work for some users). Flag `pasteDetected`, don't hard-block.

## Personalization Logic (`lib/scoring/`)

**Weakness (`weakness.ts`)**: pull last 10 scored attempts → mean band per criterion → weakest = lowest mean AND ≥0.3 below user's cross-criteria mean (filters noise). Tie-break by widest gap from target.

**Target band (`target.ts`)**: rolling band = mean of last 5 `overall_band`. Delta = target − current. If `examDate` set, suggest attempts/week to close gap (1 attempt per 0.25 band gap/week, capped).

Pure functions, unit-testable, no DB inside.

## Progress Tracking

**Library**: Recharts 2.13. RSC-friendly, lightweight.

Components: `BandTrendChart` (line, target dotted), `CriteriaRadar` (4 axes, last attempt vs 10-attempt avg), `AttemptHistoryTable` (keyset paginated + sparkline).

Queries (`lib/db/queries/progress.ts`): `getBandTrend`, `getCriteriaAverages`, `getAttemptHistory` — **all section-scoped** so Listening/Reading reuse same UI later.

## Auth (Better-Auth)

**Better-Auth 1.x** with Prisma adapter. Database sessions (Better-Auth default). Plugins:
- `magicLink` plugin → email login via Resend
- Social provider: `google` (Google OAuth)
- `nextCookies` plugin → server-action cookie integration for Next 16

**`lib/auth/auth.ts`** (sketch):
```ts
import { betterAuth } from "better-auth"
import { prismaAdapter } from "better-auth/adapters/prisma"
import { magicLink, nextCookies } from "better-auth/plugins"
import { prisma } from "@/lib/db/prisma"

export const auth = betterAuth({
  database: prismaAdapter(prisma, { provider: "postgresql" }),
  socialProviders: {
    google: { clientId: process.env.GOOGLE_CLIENT_ID!, clientSecret: process.env.GOOGLE_CLIENT_SECRET! },
  },
  user: {
    additionalFields: {
      targetBand: { type: "number", required: false },
      examDate: { type: "date", required: false },
    },
  },
  plugins: [
    magicLink({ sendMagicLink: async ({ email, url }) => { /* Resend send */ } }),
    nextCookies(), // must be LAST plugin in array
  ],
})
```

**Route mount**: `app/api/auth/[...all]/route.ts` exports Better-Auth's `toNextJsHandler(auth)`.
**Client**: `lib/auth/client.ts` exports `createAuthClient({ baseURL })` for use in client components (`signIn.email()`, `signIn.social({ provider: "google" })`).
**Server helper**: `lib/auth/session.ts` exposes `requireUser()` that calls `auth.api.getSession({ headers: await headers() })` and redirects if no session.
**`middleware.ts`** gates `/dashboard/*`, `/writing/attempt/*`, `/api/ai/*` by checking session cookie presence (cheap check; full validation in server actions).

Env required: `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `GOOGLE_CLIENT_ID`, `GOOGLE_CLIENT_SECRET`, `RESEND_API_KEY`.

## Expansion Path (architect for, don't build)

| Future need | MVP abstraction that enables it |
|---|---|
| Listening (audio + 14 Q types) | `attempts.section` enum + per-section tables. Progress charts unchanged. |
| Reading (14 Q types) | Same pattern. Tiptap already supports passage rendering + highlights. |
| Speaking (STT + AI examiner) | Add `transcribe()` + `evaluateSpeaking()` to `AIProvider` interface. Route handler streams examiner turns. R2/S3 for audio. |
| Stripe billing | Webhook stub in place. Add `subscriptions` table + middleware check. |
| Adaptive prompt selection | Weakness detector already exists; topic-tag match to recommend prompts. |

**Critical abstractions to put in MVP** so we don't rewrite:
1. `AIProvider` interface — never call SDK directly from app code
2. `section` enum on `attempts` — every progress query is section-aware
3. Tool-use Zod schemas — same validation pattern reused for listening/reading auto-grading
4. Server-action return shape `{ok: true, data} | {ok: false, error}`

**Do NOT build now**: speaking/listening/reading tables, payment. Stub routes; don't ship those migrations.

## Critical Files to Create

- `/Users/sudarshaana/Desktop/Plans/ielts_ninja/prisma/schema.prisma` — entire data model (Better-Auth + app tables in one file)
- `/Users/sudarshaana/Desktop/Plans/ielts_ninja/lib/auth/auth.ts` — Better-Auth server instance + plugins config
- `/Users/sudarshaana/Desktop/Plans/ielts_ninja/lib/db/prisma.ts` — PrismaClient singleton (avoid hot-reload connection storm)
- `/Users/sudarshaana/Desktop/Plans/ielts_ninja/lib/ai/provider.ts` — AI swap seam
- `/Users/sudarshaana/Desktop/Plans/ielts_ninja/lib/ai/prompts/writing-score.ts` — rubric + cache_control + tool schema (single biggest quality + cost driver)
- `/Users/sudarshaana/Desktop/Plans/ielts_ninja/app/api/ai/score/route.ts` — orchestrates auth → budget → AI → validate → persist
- `/Users/sudarshaana/Desktop/Plans/ielts_ninja/app/api/auth/[...all]/route.ts` — Better-Auth catch-all handler
- `/Users/sudarshaana/Desktop/Plans/ielts_ninja/components/writing/Editor.tsx` — Tiptap with paste-disable + word count + annotation overlay (hardest client component)
- `/Users/sudarshaana/Desktop/Plans/ielts_ninja/PLAN.md` — this plan, copied to project root for reference

## Verification Plan (manual E2E)

1. **Setup**: `pnpm i`. Copy `.env.example` → `.env.local` (`DATABASE_URL`, `ANTHROPIC_API_KEY`, `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `GOOGLE_CLIENT_ID/SECRET`, `RESEND_API_KEY`). Run `npx @better-auth/cli generate` → confirm Better-Auth models in `schema.prisma`. `pnpm prisma migrate dev --name init`. `pnpm prisma db seed` (loads ~20 prompts: 10 Task 1, 10 Task 2 across difficulties).
2. **Auth**: sign in with Google → `/dashboard`. Sign out. Sign in with email magic link → same user row (account-linking-by-email).
3. **Settings**: set target 7.0, exam date 60 days out. Reload, persists.
4. **Browse**: `/writing` → filter Task 2 + Medium → cards shown. Click one.
5. **Attempt**: "Start". URL `/writing/attempt/<uuid>`. Timer 40:00 counts down. Try Ctrl+V → blocked, toast, `pasteDetected=true` in DB. Type ~250-word essay. Word counter live. Reload → content restored from autosave.
6. **Submit**: streaming, ~5-15s, redirects to feedback.
7. **Feedback**: 4 band scores + overall. Radar renders. Hover annotations show comment + suggestion. Tabs "Improved" / "Model answer" trigger lazy rewrite call, show diff.
8. **History**: `/dashboard/history` lists attempt. Click → feedback.
9. **Multiple attempts** (do 3-5 on prompts of varying difficulty):
   - `BandTrendChart` plots multiple points
   - `CriteriaRadar` shows averages
   - Dashboard "Focus area" callout names weakest criterion
   - Delta-to-target updates
10. **Guardrails**: 11th attempt/day → rejected. 700-word essay → rejected at boundary before AI call. Empty essay → validation error.
11. **AI swap**: `AI_PROVIDER=openai` → repeat one submission. Same UI. `scores.modelVersion` reflects new model. Revert.
12. **Cache check**: inspect `scores.meta.usage` after 5 calls → cache read tokens >> cache write tokens (caching working).

## Build Order (recommended)

1. `pnpm create next-app@latest` (Next 16+, TS, App Router, Tailwind, no src/, alias `@/*`). Init shadcn. Add env validation (`@t3-oss/env-nextjs` + zod).
2. Prisma init: `pnpm dlx prisma init` → set `DATABASE_URL` → author `schema.prisma` app models.
3. Better-Auth install + config (`lib/auth/auth.ts`, `lib/auth/client.ts`, `app/api/auth/[...all]/route.ts`). Run `@better-auth/cli generate` → `prisma migrate dev --name init`.
4. Login/signup pages using Better-Auth client (`signIn.social`, `signIn.magicLink`). `middleware.ts` session-cookie gate. `requireUser()` helper.
5. Seed prompts (`prisma/seed.ts`). `/dashboard` + `/dashboard/settings` (target band, exam date).
6. `/writing` library + filters. `/writing/[promptId]` detail. Server action `createAttempt`.
7. Tiptap editor (`components/writing/Editor.tsx`) + timer + autosave action + paste-disable + character count.
8. `lib/ai/provider.ts` interface. `lib/ai/anthropic.ts` Claude impl. Rubric + few-shot + Zod schemas. Cache_control wiring.
9. `/api/ai/score` route handler: auth → budget → AI → validate → persist `Score`s + `Annotation`s.
10. Feedback page: band display + radar + annotated text overlay (Tiptap Marks from offsets).
11. Rewrite call (lazy on feedback view) + diff view (`diff-match-patch`).
12. Progress queries + Recharts (`BandTrendChart`, `CriteriaRadar`, `AttemptHistoryTable`).
13. Weakness + target band logic + Focus Area callout on dashboard.
14. Polish: error/empty states, mobile responsive, loading skeletons, basic analytics.
15. Copy plan to `PLAN.md` in project root for ongoing reference.
