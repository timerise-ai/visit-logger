# visit-logger

[![Agent Skills](https://img.shields.io/badge/Agent_Skills-open_format-059669)](https://agentskills.io)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-059669)](https://docs.claude.com/en/docs/claude-code/skills)

An [Agent Skill](https://agentskills.io) that teaches an agent to log, server-side, **who opened what, from
where and on what** in a Next.js App Router app. It covers:

- an IP, edge-geolocation and parsed-browser fingerprint for every real page view of a shared resource;
- the same fingerprint at the lifecycle moments before a resource exists: sign-in requested, account created,
  signed in;
- storage in Postgres/Supabase or Firestore;
- grouping into sittings, a per-visitor first-open announcement, and admin history panels.

**A request is not a visit.** Prefetches, assets, mail scanners, staff previews and reloads all arrive as
requests, and every rule in the module exists to keep them from becoming "the customer opened it" in someone's
Slack. It is an attribution log for sales and success work, not product analytics.

Extracted from a production Next.js 16 site on Vercel and Supabase, where it records customer demo opens and
sign-up origin. It was audited before extraction, and the templates fix nine defects the source still
carries. [`references/provenance.md`](references/provenance.md) is the ledger: what changed, what was kept on
purpose, and what was designed here without running in production.

## Install

The skill is a plain folder: `SKILL.md` plus markdown references. Nothing in it executes. Copy or clone it into
an agent's skills directory; for Claude Code:

```bash
cp -R visit-logger ~/.claude/skills/visit-logger
```

To scope it to one project, put it in that project's `.claude/skills/` instead. The current release is
**0.1.0**; see [`CHANGELOG.md`](CHANGELOG.md).

## Activation

The skill activates when a task matches its description: logging visits to a shared link, demo, proposal or
report; recording where a sign-up came from; reading geolocation headers or parsing user agents for storage; a
"customer opened it" notification; or auditing an existing visit log. Invoke it explicitly with `/visit-logger`
in Claude Code on a first run rather than assuming it fired.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: architecture, critical facts, hard rules, quick start, reference directory |
| `references/adaptation.md` | The seam contract, host probe, rename table, choosing the edge adapter and `via`, strings, styling |
| `references/fingerprint.md` | Types, formatters, `describeVisitor`, edge adapters, bot detection, the page-view contract, header facts |
| `references/data-model.md` | The two tables, the Supabase migration, access posture, Firestore shape and indexes |
| `references/rules.md` | Sittings, the announcement matrix, the origin event, truncation-honest summaries |
| `references/recording.md` | The failure contract, write and read paths, `after()` entry points, announcing |
| `references/stores.md` | `VisitStore` for Supabase, Firestore and memory |
| `references/capture.md` | Where to call it: a gated route handler, an App Router page, Supabase Auth sign-in |
| `references/admin-ui.md` | The history and activity panels, the origin line, the admin page |
| `references/operations.md` | Privacy, retention, erasure, health checks, the limits no code removes |
| `references/testing.md` | Three suites, 54 tests, vitest or bun |
| `references/provenance.md` | The audit ledger |

## The six non-negotiables

1. **Tracking never touches the response.** The fingerprint is read synchronously, and everything else runs in
   `after()`. The write path never throws.
2. **Bots and staff never announce and never count as a prior visit.**
3. **The IP comes from the edge the request really crossed**, never from `x-forwarded-for` on a platform
   that does not overwrite it.
4. **The tables are the server's alone.** Explicit `REVOKE` from the client roles.
5. **A failed read is not an empty history.**
6. **An IP is kept only with a purpose and a retention period.**

## Not this

| Need | Use |
|---|---|
| Anonymous traffic analytics | Vercel Web Analytics, PostHog, Plausible |
| Client events, time on page | a client analytics SDK |
| Consent management | the host's consent manager |
| Blocking bots | Vercel BotID / WAF, Cloudflare Bot Management |
| Gating a whole site | `site-pin-gate` |

## Contributing

Read [`CLAUDE.md`](CLAUDE.md) first: the code blocks are compiled and tested, identifiers are shared across
files, and the ledger in `provenance.md` explains every odd-looking line.

## License

[MIT](LICENSE)
