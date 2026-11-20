# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Carrot Wall: a live Q&A/feedback wall for a 5-day Claude Code masterclass. Attendees post from
their phones, the instructor answers, pins and hides posts from `/admin`, and a projector shows
`/tv`. The UI is in Portuguese. **`spec.md` is the source of truth** (features F1–F8, acceptance
criteria, design tokens). `feature-ideas.md` is the challenge backlog. `setup-docs/` holds the
course guides.

F1–F8 are built: Panache entities, REST resources and their tests in `apps/api`, and the
Angular routes `/`, `/post`, `/admin`, `/tv` and `/materials` (course decks and guides, served
from `apps/web/public/course/`).

## Commands

```bash
cd apps/api && ./mvnw quarkus:dev                         # API on :8080
cd apps/web && npm install && npm start                   # web on :4200, proxies /api to :8080
cd apps/api && ./mvnw test                                # JUnit + RestAssured
cd apps/api && ./mvnw test -Dtest=ClassName#methodName    # single test
cd apps/web && npm test                                   # vitest + jsdom (ng test)
cd apps/web && npx ng test --include src/app/wall/wall.spec.ts   # single file
cd apps/web && npm run e2e                                # Playwright, chromium only
cd apps/web && npx prettier --check .                     # no eslint / lint script
```

- `npm run e2e` starts both servers itself through `webServer` in `playwright.config.ts`, or
  reuses servers that already run.
- Env vars: `ADMIN_PIN` (default `0000`) and `WALL_DB_PATH` (default `./data/carrot-wall`).
  Requires Java 21 and Node 20+.

## Architecture

- **`apps/api`** (package `pt.ecrop.wall`, flat): Quarkus 3.39 on Java 21. `WallResource`
  (public reads), `PostsResource` (create, upvote), `AdminResource` (login, answer, pin, hide,
  prompts). Admin endpoints carry `@AdminOnly`, and `AdminAuthFilter` checks the session cookie
  against the in-memory `AdminSessionStore`. `RateLimiter` limits posting.
- **Database**: H2 **file** DB in PostgreSQL mode. **Flyway owns the schema**
  (`schema-management.strategy=none`). The `%test` profile uses in-memory H2 with
  `wall.page-size=3`, so pagination tests need only a few rows.
- **`apps/web`**: Angular 22 standalone components, Angular Material, TS 6, one folder per
  route under `src/app/`. Clients poll every 5 s (no websockets). They fetch changes with
  `changedSince` / `hiddenSince` deltas on `updated_at`, which is why `updated_at` must be
  correct (see below).
- **Production is one container** (root `Dockerfile`): the Angular build goes into the Quarkus
  jar's `META-INF/resources`, and both are served on :8080. `SpaRoutes.java` reroutes the SPA
  paths to `index.html`. **Add every new Angular route to `SPA_PATHS` there**, or a refresh
  returns 404.

## Conventions

- **Never edit an existing Flyway migration** (`apps/api/src/main/resources/db/migration/`,
  currently V1–V6). Add a new versioned file. A PreToolUse hook in `.claude/settings.json`
  blocks such edits. Keep the SQL portable across H2 PG mode and real Postgres: no `JSONB`, no
  arrays.
- **Never change a `Post` with a bulk `update(...)` query.** It skips `@PreUpdate`, so
  `updated_at` stays stale and polling clients never see the change. Load the entity and call
  `pin()/unpin()/hide()/unhide()/upvote()/answer(text)`.
- **Post text renders as text, never as HTML.** Use interpolation only. Never use
  `[innerHTML]` (acceptance criterion 12 is an XSS check; `tv/qr.ts` builds the QR without it
  for this reason).
- Hidden posts are a soft delete: leave them out of every public response, never delete them
  from the DB.
- Work flow: `docs/intents/NN-slug.md` → spec `docs/specs/NN-slug.md` → plan
  `docs/specs/NN-slug-plan.md`. Tests sit beside the code: JUnit in `apps/api/src/test`,
  `*.spec.ts` beside each component, Playwright in `apps/web/e2e/`.
- A PostToolUse hook runs Prettier on edited `.ts/.html/.scss/.json` files.
- Design: follow `DESIGN.md`, with spec §4 for the tokens. Note that they disagree on coral
  (`spec.md` `#D97757` vs `DESIGN.md` primary `#cc785c`). Ask before you choose one.
