# visit-logger

[![Agent Skills](https://img.shields.io/badge/Agent_Skills-open_format-059669)](https://agentskills.io)
[![skills.sh](https://img.shields.io/badge/skills.sh-npx_skills_add-059669)](https://www.skills.sh)
[![Claude Code](https://img.shields.io/badge/Claude_Code-compatible-059669)](https://docs.claude.com/en/docs/claude-code/skills)
[![Codex CLI](https://img.shields.io/badge/Codex_CLI-compatible-059669)](https://developers.openai.com/codex/skills)
[![Gemini CLI](https://img.shields.io/badge/Gemini_CLI-compatible-059669)](https://github.com/google-gemini/gemini-cli/blob/main/docs/cli/skills.md)

An [Agent Skill](https://agentskills.io) that teaches an agent to log, server-side, **who opened what, from
where and on what** in a **Next.js App Router** app: an IP, edge-geolocation and parsed-browser fingerprint
for every real page view of a shared resource, the same fingerprint at the lifecycle moments before a
resource exists (sign-in link requested, account created, signed in), storage in Postgres/Supabase or
Firestore, and the output a salesperson reads: sittings, a first-open announcement, admin history panels.

**A request is not a visit.** Prefetches, assets, mail scanners, staff previews and reloads all arrive as
requests, and every rule in the module exists to keep them from becoming "the customer opened it" in someone's
Slack. That is also what separates this from analytics: the log is attributed to one resource and one person,
it is read one customer at a time, and the cost of a wrong row is a phone call made on a fact that was never
true.

This skill was written by the engineer who has shipped this module. The earlier implementation it was audited
against was a visit log on a sales site, behind the shared links a team sends customers and on its sign-in
routes. The templates hold the properties such a log has to hold: every stored page view is a document
navigation or an App Router client navigation, never a prefetch, an asset or a Server Action; an automated or
internal client is labelled, never announced and never counted as a prior visit; the address and the
geography come from the edge the request really crossed; a visitor with no IP or no user agent still matches
itself; a history that could not be read says so, and a truncated one counts as a lower bound; a first
sign-in is the first confirmed one. The three suites state each one, and
[`references/provenance.md`](references/provenance.md) has the record.

## Install

One command, via the [skills.sh](https://www.skills.sh) CLI, which installs the skill into every
skills-compatible agent it detects, including Claude Code, Codex CLI and Gemini CLI:

```bash
npx skills add timerise-ai/visit-logger
```

Name the agents instead with `-a`, for example
`npx skills add timerise-ai/visit-logger -a claude-code -a codex`.

### Manual install

Nothing here is Claude-specific: the skill is a plain [Agent Skills](https://agentskills.io) folder,
`SKILL.md` plus markdown references with no file that calls a model, so cloning it into an agent's skills
directory is all an install is. For Claude Code:

```bash
git clone https://github.com/timerise-ai/visit-logger.git ~/.claude/skills/visit-logger
```

To scope it to a single project instead, clone it into that project's `.claude/skills/` directory. For another
agent, clone into that agent's skills directory, or symlink the Claude Code copy so one `git pull` updates
every agent:

```bash
mkdir -p ~/.agents/skills
ln -s ~/.claude/skills/visit-logger ~/.agents/skills/visit-logger
```

Update the skill with `git pull` in its directory. The current release is **0.1.1**. See
[CHANGELOG.md](CHANGELOG.md). The [skills index](https://github.com/timerise-ai/skills) lists the other
Timerise Skills and how to install them all at once.

## Activation

The skill activates automatically when a task matches its description: logging visits to a shared link, demo,
proposal, quote or report; showing an owner whether and when a customer opened what was sent; recording where
a sign-up came from; reading geolocation headers or parsing user agents for storage; a "customer opened it"
notification; or auditing an existing visit log for false pings, a wrong first-opened date or visitors counted
twice. Invoke it explicitly with `/visit-logger` in Claude Code, `$visit-logger` in Codex CLI, or from
`/skills` in Gemini CLI.

Each host matches a task against the description its own way, so invoke the skill explicitly on a first run
rather than assuming it fired. Only `SKILL.md` is read up front; the `references/` files load on demand, so
the skill stays cheap in context until a topic is actually needed.

## What's inside

| File | Contents |
|---|---|
| `SKILL.md` | Entry point: architecture diagram, critical facts, hard rules, quick start, and the reference directory |
| `README.md` | This front door |
| `CHANGELOG.md` | Keep a Changelog, one section per release, newest first |
| `CLAUDE.md` | What this repository is and the conventions for editing the skill itself |
| `LICENSE` | MIT |
| `references/adaptation.md` | The seam contract with the host app: the probe, the rename table, choosing the edge adapter and `via`, strings, styling |
| `references/fingerprint.md` | Types and formatters, `describeVisitor`, the edge adapters, bot detection, the page-view contract, the header facts |
| `references/data-model.md` | The two tables, the Supabase migration, the access posture, the Firestore shape and indexes |
| `references/rules.md` | The pure decisions: sittings, the announcement matrix, the origin event, truncation-honest summaries |
| `references/recording.md` | The failure contract, the write and read paths, the `after()` entry points, announcing |
| `references/stores.md` | `VisitStore` for Supabase, Firestore and memory |
| `references/capture.md` | Where to call it: a gated route handler, an App Router page, Supabase Auth sign-in |
| `references/admin-ui.md` | The history and activity panels, the origin line, the admin page |
| `references/operations.md` | Privacy, retention, erasure, health checks, the limits no code removes |
| `references/testing.md` | The three suites, 54 tests, and how to run them under vitest or bun |
| `references/provenance.md` | The engineering ledger: what the audit changed and how the templates verify it, what was kept on purpose, and what is new in the skill |
| `evals/` | The prompts an operator types after installing (`prompts.md`) and one file per agent eval: the skill installed into an empty Next.js app, one prompt, no help, then type-checked, built and tested |

The seam is the `VisitStore` interface in `references/stores.md` and the table at the top of
`references/adaptation.md`. It bounds four things: the store, so Supabase, Firestore or memory are the same
module; the `EdgeHeaders` adapter, so the platform in front of the app is one object and not a parser spread
through the code; the strings and the styling, which stay the host's; and the domain vocabulary, which the
rename table maps from `resource_id` and `subject` to whatever the host already calls them. The host's auth
decides who may see the resource at all, and the log is written after that decision.

## The six non-negotiables

These travel with the module and are never optional. Each is stated as a hard rule in `SKILL.md` and covered
by the suites in `references/testing.md`:

1. **Tracking never touches the response.** The fingerprint is read synchronously from the request, and the
   store, the write and the announcement are deferred with `after()`. A tracking call that awaited a database
   write would put that write's latency and its failures in front of the customer, so the write path never
   throws and its result is a flag, not an exception. The write-failure and read-failure paths are in the
   suite.
2. **A bot or an internal visit is never announced and never counts as a prior visit.** An automated client
   that follows an emailed link, or a staff member previewing what the customer will see, arrives before the
   customer does; if either counts, the customer's first open is announced to nobody. Both are stored and
   flagged, and a test holds that neither makes the customer's own first visit look like a return.
3. **The IP comes from the edge the request really crossed.** `x-forwarded-for` is only trustworthy where the
   platform overwrites it, and behind another proxy it holds that proxy's address. The adapter is chosen
   explicitly, once, and `EdgeHeaders.ip` is the only parser in the app.
4. **The tables are the server's alone.** Visit rows are a log about people who never asked to see it, and
   Supabase's default privileges grant every new table to `anon`, so the migration revokes explicitly and the
   access posture is checked by applying it against the client roles.
5. **A failed read is never rendered as an empty history.** "Could not load" and "Not opened yet" are
   different facts to the person deciding whether to call, and only one of them is a reason to call. Both
   result types carry `failed`, and the panels render it as an alert.
6. **An IP is stored only with a purpose and a retention period.** The address and the geography are personal
   data whatever the log is for, so the purge ships with the table and the erasure path is written before the
   first row is stored.

Everything else is the host app's: authorisation, tenancy, strings, locale, palette, the notification channel,
and the store behind the seam.

## Requirements

Next.js App Router with `after()` from `next/server`, which is where everything but the fingerprint runs. A
store: Postgres reached through supabase-js, Firestore through firebase-admin, or the in-memory store for
tests. An edge whose geolocation headers the app can trust, or `noEdge` and a log without geography. Nothing
else is added: the templates import from `next/server`, `next/headers` and the store's own client.

## Security

The module writes a log about identifiable people, so its posture is stated rather than implied. The tables
are service-role only, with RLS on and explicit `REVOKE` from `anon` and `authenticated`, so a leaked
publishable key reads nothing. Free-text fields are capped, and the fingerprint is never echoed back to the
visitor. Retention is a scheduled purge of the network fields, and erasure is a query by subject with an index
to support it; both are in `references/operations.md`, with the wording a privacy policy needs.

## Not this

| Not this | Use instead |
|---|---|
| Anonymous traffic, funnels, dashboards | Vercel Web Analytics, PostHog, Plausible. This log is attributed to one resource and one person |
| Clicks, scroll depth, time on page | A client analytics SDK. This is server-side and sees requests only |
| Cookie consent or an ePrivacy banner | The host's consent manager. This sets no cookies, but an IP is still personal data |
| Blocking bots or fraud | Vercel BotID, a WAF, Cloudflare Bot Management. `isBot` here labels, it never blocks |
| Hiding a whole site behind one PIN | [`site-pin-gate`](https://github.com/timerise-ai/site-pin-gate). Gating one resource is the host's auth or token layer |

## Contributing

Issues and pull requests are welcome here. Pure markdown, with no build step, but the code blocks are checked:
every block names its destination on the first line, and every TypeScript block is written to compile as one
project under `strict` and `noUncheckedIndexedAccess` and to run under vitest and `bun test`, 54 tests. The
SQL is applied twice against PostgreSQL with Supabase's roles and default privileges recreated. Claims in this
skill are meant to be verifiable: if you change a factual claim, say how you verified it, whether against
Next's `userAgent()`, the platform's header documentation, Node's HTTP parser, PostgreSQL, the Supabase Auth
server, or a reproduction.

Adding, removing or renaming a file in `references/` means updating the quick start and the reference
directory table in `SKILL.md`, the file table above, and any relative cross-links. Every odd-looking part of
the templates is there for a reason, and `references/provenance.md` is the ledger that must stay truthful:
read it before simplifying anything, and add an entry for anything you change. Commits follow Conventional
Commits and releases follow [STANDARD.md](https://github.com/timerise-ai/skills/blob/main/STANDARD.md) in the
index; `CLAUDE.md` carries the full editing conventions.

## Part of the Timerise Skills

This is one of the [Timerise Skills](https://github.com/timerise-ai/skills): modules for **Next.js App
Router** apps written by our own senior engineers from the modules they have shipped, not synthetic, each
published as its own repository and indexed there. They share one layout, so an agent that has read one knows
how to read the next: a `SKILL.md` entry point, `references/` loaded on demand, and a seam contract carrying
the module's non-negotiables.

## Author

Built and maintained by [Timerise](https://timerise.ai).

## License

MIT. See [LICENSE](LICENSE).
