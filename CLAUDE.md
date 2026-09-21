# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown only. There is no `package.json` here and nothing
in this repository executes. It teaches an agent to log, server-side, who opened a shared resource in a
**Next.js App Router** app, from where and on what. It also records the same fingerprint at sign-in moments.
Storage is Postgres/Supabase or Firestore, and the output is sittings, first-open announcements and admin
panels.

The commands and code in `references/` describe the app the agent will generate, not this repository. The
host probe in `adaptation.md`, the SQL in `data-model.md` and `operations.md`, and the test invocations in
`testing.md` all run in that app. The one thing checked here is that the templates compile and their tests
pass, in a scratch project; the recipe is below.

The skill was extracted from a production Next.js 16 site on Vercel and Supabase. `references/provenance.md`
is the ledger of the audit: nine fixed defects, what was kept deliberately, and what was designed here and has
never run in production. Read it before "simplifying" anything.

## Structure

- `SKILL.md`: entry point, loaded whole on every activation, so it stays near 150 lines. The frontmatter
  `description` is the trigger surface.
- `README.md`: the human-facing front door.
- `references/*.md`: one topic per file, loaded on demand. `adaptation.md` and `fingerprint.md` are the
  design entry points; `rules.md` holds the pure decisions; `recording.md`, `stores.md` and `capture.md` the
  wiring; `admin-ui.md` the panels; `operations.md` running it; `testing.md` the suites; `provenance.md` the
  audit.

## Editing conventions

- **Code blocks name their destination on the first line** as a comment: `// file: lib/visits/core.ts`, or
  `-- file: supabase/migrations/<timestamp>_visit_log.sql` for SQL. That line makes a block extractable.
- **The code blocks are compiled and run.** Every `// file:` block forms one project. Extract each to its named
  path in a scratch directory, then

  ```bash
  npm i -D typescript@6 next@16 react@19 react-dom@19 @types/react @types/node \
    @supabase/supabase-js firebase-admin vitest
  npx tsc --noEmit    # strict, noUncheckedIndexedAccess, skipLibCheck, jsx react-jsx, paths {"@/*": ["./*"]}
  npx vitest run lib/visits && bun test lib/visits    # 54 tests each
  ```

  Do not set `baseUrl`, which TypeScript 6 removed. The `declare function` lines in the route and page
  examples are host seams; they type-check and are meant to fail if copied unreplaced.
- **The SQL is verified separately**, on PostgreSQL with Supabase's `anon`, `authenticated` and `service_role`
  roles and its default privileges recreated. Apply it twice (it must be idempotent), then check that the
  client roles are refused and that the purge and the cascade behave as `data-model.md` states.
- **Identifiers are shared across files.** `VisitFingerprint`, `PageVisit`, `VisitSession`, `VisitSummary`,
  `VisitorEvent`, `VisitorEventList`, `VisitVia`, `EdgeHeaders`, `vercelEdge`, `cloudflareEdge`, `noEdge`,
  `describeVisitor`, `looksAutomated`, `isPageView`, `SESSION_WINDOW_MS`, `assessVisit`, `pickOriginEvent`,
  `summarizeVisits`, `VisitStore`, `recordPageVisit`, `recordVisitorEvent`, `readVisitSummary`,
  `readVisitorEvents`, `findOriginEvent`, `normalizeSubject`, `trackPageVisit`, `trackVisitorEvent`,
  `getVisitStore`, `isFirstSignIn`, the tables `page_visits` / `visitor_events` and the collections
  `pageVisits` / `visitorEvents`. Rename in all files or none.
- **Keep the three tables in sync** with `references/`: the reference directory and quick start in `SKILL.md`,
  and the file table in `README.md`.
- **Do not remove the odd-looking parts.** The synchronous fingerprint before `after()`, the store factory, the
  `is(null)` branches, `in` instead of `!=` on Firestore, `\bbot\b` instead of `bot`, `limit + 1`, the
  `failed` flags, the `REVOKE`s, the two-minute `isFirstSignIn` window. Each is a ledger entry or a documented
  judgement. Check `provenance.md` before touching one.
- **Mark additions as additions.** Anything designed here and never run in the source belongs in the "Added"
  section of `provenance.md`, or under a "not shipped" heading as a design.
