# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [0.1.3] - 2026-09-28

Fix release, from scoring the prompt-1 agent eval runs against 0.1.2.

### Fixed

- The App Router page in `capture.md` said it counted client-side navigations.
  On Next 16.3.6 `headers()` in a Server Component hides the flight headers
  (`rsc` and both prefetch headers), so a client navigation arrives as
  `Sec-Fetch-Dest: empty` and `isPageView` drops it. Reproduced against
  `next start`. The page comment, the page-view contract in `fingerprint.md`
  and the provenance now say the page counts document requests only, and a new
  `isPageView` row pins it: 55 tests. First-open announcements are unaffected,
  since a shared link's first open is a document request. Apps built from
  earlier versions need no code change; classifying in `proxy.ts` to count
  client navigations is described as a design, not shipped.

### Changed

- The quick start in `SKILL.md` says every template is copied as written, that
  the package registry is not an external service, to install `vitest` and run
  the suites unchanged under `npm test`, that the subject never comes from the
  query string, and to finish with a handover.
- `adaptation.md` says where the host's own code goes, what to do unattended
  with the rename, and that a host with no share-link check gets a server-side
  token rather than a `?email=` parameter.
- `testing.md` forbids converting the suites to another runner.
- `recording.md` names the store's env variables for `.env.example` and puts
  the host's sender in a file of its own, narrowing to the first open inside
  the `announce` callback.
- `operations.md` gains a handover section: the migration, the purge schedule,
  the privacy-policy line, the edge assumed, the env variables and the limit.

## [0.1.2] - 2026-09-28

Documentation release. The skill content is unchanged from 0.1.1.

### Changed

- The README file table and `CLAUDE.md` list `evals/`, the prompts an operator
  types after installing and one file per agent eval run, as the skills standard
  now asks. `CLAUDE.md` adds that evals are committed as `chore(evals)` and never
  bump the version, and describes the agent eval workflow caller.

## [0.1.1] - 2026-09-21

Wording release. The skill content is unchanged from 0.1.0.

### Added

- `SKILL.md` closes with a line linking the
  [Timerise Skills](https://github.com/timerise-ai/skills) index, so an agent that
  has the skill loaded can find the sibling skills for neighbouring modules without
  leaving the entry point.

### Changed

- `CLAUDE.md` records the closing line in the `SKILL.md` layout, and the line budget
  it states holds that line aside.

## [0.1.0] - 2026-09-21

Initial release of the `visit-logger` skill: server-side logging of who opened a
shared resource, from where and on what, plus lifecycle events at sign-in, for a
Next.js App Router app.

### Added
- `SKILL.md` entry point: architecture, six critical facts, six hard rules, the
  quick-start order and the reference directory.
- `references/adaptation.md`: the seam contract, host probe, rename table, edge
  adapter and `via` decisions, strings and styling.
- `references/fingerprint.md`: types and formatters, `describeVisitor`, the
  Vercel, Cloudflare and no-edge adapters, `looksAutomated`, `isPageView`.
- `references/data-model.md`: the Supabase migration with retention function,
  access posture, and the Firestore shape and indexes.
- `references/rules.md`: sittings, the announcement matrix, the origin event,
  summaries.
- `references/recording.md`: the failure contract, write and read paths, the
  `after()` entry points, announcing.
- `references/stores.md`: Supabase, Firestore and in-memory `VisitStore`s.
- `references/capture.md`: gated route handler, App Router page, and Supabase
  Auth sign-in capture, with `isFirstSignIn`.
- `references/admin-ui.md`: history and activity panels, the admin page.
- `references/operations.md`: privacy, retention, erasure, health checks, known
  limits.
- `references/testing.md`: three suites, 54 tests, vitest or bun.
- `references/provenance.md`: the audit ledger of the earlier implementation.
