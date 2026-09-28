# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repository is

An [Agent Skill](https://agentskills.io) package: markdown only. There is no `package.json` here and nothing
in this repository executes. It teaches an agent to log, server-side, who opened a shared resource in a
**Next.js App Router** app, from where and on what, and to record the same fingerprint at sign-in moments.
Storage is Postgres/Supabase or Firestore, and the output is sittings, first-open announcements and admin
panels.

The commands and code in `references/` describe the app the agent will generate, not this repository. The
host probe in `adaptation.md`, the SQL in `data-model.md` and `operations.md`, and the test invocations in
`testing.md` all run in that app. The one thing checked here is that the templates compile and their tests
pass, in a scratch project; the recipe is below.

`references/provenance.md` is the rationale layer and the ledger of the audit against the earlier
implementation: what the audit changed and how the templates verify it, what was kept deliberately with the
reason it is safe, and what was designed here and has never run in production. Read it before "simplifying"
anything.

## Structure

- `SKILL.md`: entry point, loaded whole on every activation, so it stays between 130 and 160 lines, the
  closing index line aside. The frontmatter `description` is the trigger surface, and the body closes with a
  line linking the skills index.
- `README.md`: the human-facing front door, in the section order every Timerise skill shares.
- `CHANGELOG.md`: Keep a Changelog, newest release first. The version lives here, in the README's
  current-release line, and in the git tag, and the three agree.
- `references/*.md`: one topic per file, loaded on demand. `adaptation.md` and `fingerprint.md` are the
  design entry points; `rules.md` holds the pure decisions; `recording.md`, `stores.md` and `capture.md` the
  wiring; `admin-ui.md` the panels; `operations.md` running it; `testing.md` the suites; `provenance.md` the
  ledger.
- `evals/`: `prompts.md` holds what an operator types after installing, in their words; the first prompt
  is the agent eval run before every release. Every other file there is one eval run: measured frontmatter
  that is never edited, then the notes of the person who ran it. Add a prompt rather than rewording one that
  has results. The procedure is section 10 of the index's STANDARD.md.
- `.github/workflows/agent-eval.yml`: the caller of the index's reusable eval workflow, run on every
  published release and on a maintainer's dispatch. It is the same in every skill and was set up by a
  maintainer; do not edit it, and never add a trigger on `push` or `pull_request`.

## Editing conventions

- **Code blocks name their destination on the first line** as a comment: `// file: lib/visits/core.ts`, or
  `-- file: supabase/migrations/<timestamp>_visit_log.sql` for SQL. That line makes a block extractable. A
  block that continues a file already introduced omits it.
- **The code blocks are compiled and run.** Every `// file:` block forms one project. Write each to its named
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
  `describeVisitor`, `looksAutomated`, `isPageView`, `FRAGMENT_SEPARATOR`, `SESSION_WINDOW_MS`,
  `assessVisit`, `pickOriginEvent`,
  `summarizeVisits`, `VisitStore`, `recordPageVisit`, `recordVisitorEvent`, `readVisitSummary`,
  `readVisitorEvents`, `findOriginEvent`, `normalizeSubject`, `trackPageVisit`, `trackVisitorEvent`,
  `getVisitStore`, `isFirstSignIn`, the tables `page_visits` / `visitor_events` and the collections
  `pageVisits` / `visitorEvents`. Rename in all files or none.
- **Two kinds of name, and they are not the same kind.** The domain vocabulary the host renames is the
  rename table in `adaptation.md`: `resource_id`, `resource_key`, `subject`, `link`, `internal` and the
  event kinds. Everything in the list above is the authoring contract of this repository, which the host may
  rename in its own app but which must stay consistent here.
- **Keep the three tables in sync** with `references/`: the reference directory and quick start in `SKILL.md`,
  and the file table in `README.md`.
- **The non-negotiables are never presented as optional.** The six hard rules in `SKILL.md` and the six
  non-negotiables in `README.md` are one list, in one order, and each is covered by a suite in `testing.md`.
  Changing one is a MAJOR release and says what broke.
- **Measured numbers are load-bearing.** The 30-minute session window, the 512-character cap, the `limit + 1`
  read, the two-minute `isFirstSignIn` window and the test counts are design parameters or facts this
  repository verifies. Do not restate one loosely and do not invent new ones.
- **Do not remove the odd-looking parts.** The synchronous fingerprint before `after()`, the store factory,
  the `is(null)` branches, `in` instead of `!=` on Firestore, `\bbot\b` instead of `bot`, `limit + 1`, the
  `failed` flags, the `REVOKE`s, the two-minute `isFirstSignIn` window. Each is a ledger entry or a
  documented judgement. Check `provenance.md` before touching one.
- **Mark additions as additions.** Anything designed here and never run in the earlier implementation belongs
  in the "Added" section of `provenance.md`, or under a "not shipped" heading as a design.
- **Plain punctuation.** No em-dashes, en-dashes, arrows, middle dots or smart quotes anywhere in this
  repository's markdown, code blocks included. The only non-ASCII characters are those in proper names, such
  as the city fixtures and the Latin-1 example in `fingerprint.md`. Prose wraps at 110 columns; table rows
  and commands stay on one line.
- **Claims are verifiable.** A changed factual claim says how it was verified: against Next's `userAgent()`,
  the platform's header documentation, Node's HTTP parser, PostgreSQL, the Supabase Auth server, or a
  reproduction. Never from memory.
- **Evals are not skill content.** A new prompt or an eval result is committed as `chore(evals): ...`,
  never causes a version bump and never rides in a release commit. The frontmatter of a result file is what
  was measured and is not edited; a failing run stays committed, and the fix is the next release.
- **Commits follow Conventional Commits**, and no file or commit message names a tool or a model as author.
