# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

The repository has only `prd.md`, the approved MVP spec, and no code yet. **`prd.md` is the source of truth**: read the relevant section before you implement anything, and follow its §11 build order. When the code exists, update this file so it describes the real code and not only the plan.

The product is a PT-BR True/False web quiz about Claude Code. A round has 15 questions in a fixed order: 5 easy, then 5 medium, then 5 hard. The player gets feedback right after each answer. The app also has a public ranking and an admin area with no authentication.

## Commands (planned, per PRD §5.6)

```bash
npm install && npm run dev   # must work with no external services (RNF-07)
npm run build / npm start
npm run lint
npm test                     # Vitest unit tests
npx vitest run tests/unit/scoring.test.ts   # single unit test file
npx vitest run -t "name"     # single test by name
npm run test:e2e             # Playwright, full-round flow
npm run db:migrate           # prisma migrate dev
npm run db:seed              # prisma db seed (30 questions)
npm run db:reset             # prisma migrate reset
```

Env: copy `.env.example` → `.env` with `DATABASE_URL="file:./dev.db"`. The SQLite file is `prisma/dev.db`.

## Stack

- Next.js 15 (App Router) with TypeScript in strict mode
- Tailwind
- Prisma + SQLite
- Zod
- Vitest and Playwright
- ESLint and Prettier

## Architecture and invariants

These rules span several files and are easy to break:

- **The server owns grading.** Clients must never receive `correctAnswer` before they answer. `GET /api/attempts/:id/current` leaves it out on purpose. `POST /api/attempts/:id/answer` is the only place that grades, awards points, and on the 15th answer sets `finishedAt`/`durationMs`.
- **Answers must come in order.** The server accepts an answer only for the lowest unanswered `position` of the attempt. A repeated or out-of-order answer returns `409` and changes nothing.
- **Pure logic lives in `src/lib`.** `draw.ts` exports `drawQuestions(pool, perLevel = 5)`, which runs a Fisher–Yates shuffle within each level, returns questions ordered EASY→MEDIUM→HARD, and throws if any level is short. `scoring.ts` exports `pointsFor(level, isCorrect)` (10/20/30 or 0, max 300). Both must stay free of DB access so they're unit-testable.
- **Zod schemas are shared.** `lib/schemas.ts` validates both the admin form on the client and the route handlers on the server.
- **Data model:** `Question` ←→ `AttemptQuestion` (positions 1..15, both `(attemptId, position)` and `(attemptId, questionId)` unique) ←→ `Attempt`. If Prisma's SQLite provider rejects `enum Level`, store it as `String` and validate it with Zod.
- **Ranking rows:** each completed, non-hidden `Attempt` is one ranking row, with no per-person aggregation. Sort by score desc, then `durationMs` asc, then `finishedAt` asc. The page shows the top 20, filtered by `all` or `7d`.
- **Admin deletes:** a question that has appeared in any attempt can't be deleted (`409`). Deactivating it is the intended path. Hiding an attempt with `PATCH { hidden }` can be undone.
- **Errors** have the shape `{ error: { code, message } }`, with messages in PT-BR. Status codes are 400, 404, 409 and 422 (422 means the question bank is too small).
- **Nicknames:** trim them, collapse whitespace, and keep them to 2–20 chars. Never use `dangerouslySetInnerHTML`.

## Conventions

- All UI text is PT-BR. Routes use Portuguese names (`/quiz/[attemptId]`, `/resultado/[attemptId]`, `/ranking`, `/admin/perguntas/...`).
- Accessibility is part of the spec. The game must be fully playable from the keyboard (`V`/`F`, `←`/`→`, `Enter`). Feedback goes in an `aria-live` region, and right/wrong must use an icon and text, not color alone. Contrast must meet AA. Animations must respect `prefers-reduced-motion`. Layout must work at 360px.
- Theme colors are CSS variable tokens with light and dark modes. The primary color is orange/terracotta. The admin UI is plain, with no gamification.
- Seed content (§6) has 30 questions, 10 per level, and 40–60% of them must be False. Check every statement against the current official docs, and don't hardcode prices, limits or plan names.
- `/admin` has no auth by design. The README must warn that it must not be exposed publicly.
