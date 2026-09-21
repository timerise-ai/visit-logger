# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

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
