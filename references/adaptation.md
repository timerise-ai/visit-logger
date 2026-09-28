# Adaptation

How the visit logger lands in a host app. The module touches the host in more
places than its size suggests, because it sits at the point where a request
becomes a person. Each touch point is a seam, and each seam is listed below.

## The seam contract

| Seam | The skill ships | The host supplies | The reference host used |
|---|---|---|---|
| **Domain entities** | `resource`, `subject`, `via`, `PageVisit`, `VisitorEvent`, and the rename table below | its own vocabulary | brief, email, demo visit |
| **Resource scope** | `resourceId`, only ever from the server's own authorisation result | the id of the row being visited | the brief id behind a demo folder |
| **Viewer authorisation** | the `{ subject, via }` shape the log needs | its auth, token or share-link check | Supabase session, emailed token, admin email list |
| **Edge / geolocation** | `EdgeHeaders` adapters: `vercelEdge`, `cloudflareEdge`, `noEdge` | the adapter for its real edge | Vercel |
| **UA parsing** | `userAgent()` from `next/server`, widened by `looksAutomated` | nothing: Next ships it | same |
| **Data access** | the `VisitStore` interface, with Supabase, Firestore and in-memory implementations | its client or ORM | supabase-js, service role |
| **Background work** | `after()` from `next/server` | nothing on Next 15+ | same |
| **Notifier** | `announce(assessment, fingerprint)` plus `originLines` | Slack, email, CRM | a Slack webhook |
| **Lifecycle events** | a union of kinds and `isFirstSignIn` for Supabase Auth | its sign-in routes | Supabase magic link |
| **UI primitives, styling** | structure-only panels | card, badge, list, time formatting | Tailwind admin console |
| **Strings** | `VisitHistoryStrings`, `VisitorActivityStrings` | its i18n lookup | English literals |
| **Retention** | `purge_visit_network_data()` | the period, the schedule, the privacy text | none |
| **Tests** | three suites and a memory store | its runner | none |

A seam you cannot fill is a seam that will be hardcoded. Fill the right-hand
column for the host before copying a file.

## Host probe

```bash
grep -E '"(next|react|@supabase/[a-z-]+|firebase-admin|drizzle-orm|@prisma/client|zod|vitest|jest)"' package.json
ls proxy.ts middleware.ts 2>/dev/null        # Next 16 names it proxy.ts
ls supabase/migrations drizzle prisma 2>/dev/null | head
cat CLAUDE.md AGENTS.md 2>/dev/null | head -80
grep -rn "x-forwarded-for\|x-vercel-ip\|cf-connecting-ip\|userAgent(" --include=*.ts app lib src 2>/dev/null
grep -rn "from \"next/server\"" --include=*.ts -l app src 2>/dev/null | xargs grep -l "after(" 2>/dev/null
```

The fourth grep matters most: a host that already reads the client IP somewhere
(a rate limiter, typically) should end up with **one** IP reader. Point the
existing code at the chosen `EdgeHeaders.ip` rather than keeping two parsers
with two fallbacks.

Where is it deployed? Check `vercel.json` / `vercel.ts`, a `wrangler.toml`, a
`Dockerfile`, and the DNS: a Cloudflare-proxied domain in front of Vercel
changes the answer (see below).

## The rename

Do it once, before generating anything, and apply it to types, tables, columns,
routes, components, comments and strings in one pass.

| Canonical (this skill) | Reference host | Typical hosts |
|---|---|---|
| `resource` / `resourceId` | brief (`brief_id`) | proposal, quote, listing, report, deal room |
| `resourceKey` | demo folder | slug, title, version label |
| `subject` | email | email, user id, contact id |
| `via: "link"` | `token` | share link, magic link, invite |
| `via: "session"` | `session` | signed-in owner |
| `via: "internal"` | `admin` | staff, team member, QA |
| `page_visits` / `PageVisit` | `demo_visits` / `DemoVisit` | `proposal_views`, `listing_visits` |
| `visitor_events` / `VisitorEvent` | `visitor_events` | `account_events`, `signin_log` |
| kind `link_requested` | `otp_requested` | `magic_link_sent`, `code_requested` |
| kind `resource_created` | `brief_created` | `proposal_created` |

Leave platform terms alone: `ip`, `userAgent`, `referrer`, `timezone`,
`isBot`, `deviceType`.

**Confirm the rename with the user.** It is the one column the probe cannot
infer, and a half-applied rename teaches the next reader that both names are
live. Working unattended, with nobody to ask, either keep the canonical names
or apply the rename whole, and list the choice as an assumption.

## Where the host's code goes

The templates are copied as written; the rename is the only edit to them. The
host's own code goes in files of its own and calls the templates: the sender
that posts to Slack, the access check that produces `{ subject, via }`, the
admin page. A change a template seems to need is a finding to report with the
input that shows it, not an edit: the suites pin the templates' behaviour, and
an edited template no longer matches the ledger in [provenance.md](provenance.md).

Install what the templates import (`@supabase/supabase-js` or `firebase-admin`,
and `vitest` for the suites). The package registry is not an external service,
even where the app's own services are unreachable; a hand-written REST client
or a converted suite is a new module nobody has verified.

## Choosing the edge adapter

The IP and the location both come from headers the edge adds. The adapter must
match the edge the request actually crossed last, or the log records your CDN.

| Deployed as | Adapter | What goes wrong otherwise |
|---|---|---|
| Vercel, domain pointed straight at Vercel | `vercelEdge` | nothing: Vercel overwrites `x-forwarded-for` and adds `x-vercel-ip-*` |
| Cloudflare proxy (orange cloud) in front of Vercel | `cloudflareEdge` | `vercelEdge` records a Cloudflare address and Cloudflare's data-centre city for everyone |
| Cloudflare Workers / Pages | `cloudflareEdge` | n/a; enable the "Add visitor location headers" managed transform for city, region, timezone |
| Self-hosted behind your own proxy | a small adapter reading the one header *your* proxy sets and strips from clients | reading `x-forwarded-for` records whatever the client typed |
| Local development | `noEdge` | a spoofable guess ends up in the table |

A Cloudflare-fronted Vercel project has one more trap: the `*.vercel.app`
address still answers directly, and there `cf-connecting-ip` is a header anyone
can send. Either lock direct access down, or accept that a visitor who bypasses
Cloudflare can put any IP they like in the log. It is an attribution log, not a
security control, so the second is usually fine once it is written down.

Writing a custom adapter is three lines; see `EdgeHeaders` in
[fingerprint.md](fingerprint.md).

## Choosing `via`

`via` decides what counts, so the host's authorisation must produce it, and the
order of the checks matters:

1. **Staff first.** A staff member opening the customer's link is `internal`,
   even though the link would also let them in. In the earlier implementation, a super admin
   holding the customer's token cookie kept showing up as the customer until
   the admin check was made to win over the token. Check for a staff session
   whenever one is present.
2. **Then the link**, `link`: the share token, invite or magic link.
3. **Then the owner's session**, `session`.

Anything the host cannot attribute to a resource is not logged at all. An admin
opening a resource that no customer owns yet has nothing to be attributed to.

**When the host has no share-link check**, add one before logging anything: a
random token per resource and recipient, stored server-side with the
recipient's subject, and a link that carries only the token. Never read the
subject from the query string (`?email=`) or any other value the visitor
typed. A log that takes the name from the URL lets anyone make it say, and the
announcement post, that a customer opened something they never saw.

## Strings

The panels take a strings object; the formatters return language-neutral
fragments (a location such as `"Kraków, 12, PL"`, and a client line of
browser, OS and device joined by `FRAGMENT_SEPARATOR`) or `null`.

- **Host has i18n**: build `VisitHistoryStrings` and `VisitorActivityStrings`
  from its dictionaries, in every locale, including the plural functions.
- **Host has none**: use `VISIT_HISTORY_STRINGS_EN` as the one constants block.
- The notification lines in `announce.ts` are strings too. An internal Slack
  channel usually stays in the team's language; say which, and keep it in one
  place.

## Styling

The panels ship bare semantic elements. Replace them with the host's card,
badge and list primitives and its semantic colour tokens. Two presentational
decisions are part of the behaviour and must survive the restyle:

- Bot rows stay visible but de-emphasised (the `data-bot` attribute), with a
  badge. Hiding them removes the explanation for a suspiciously early "open".
- The summary line says "at least" when `truncated` is set.

## Adaptation checklist

- [ ] Seam table filled for this host; rename confirmed with the user
- [ ] One IP reader in the codebase, from the chosen `EdgeHeaders`
- [ ] Edge adapter matches the real edge, including a Cloudflare proxy in front
- [ ] `via` produced by the host's authorisation, staff check first; the subject never from the URL
- [ ] Templates copied as written; host code in its own files; dependencies installed
- [ ] Store implementation matches the host's data-access style ([stores.md](stores.md))
- [ ] Strings in the host's i18n system; panels on the host's primitives
- [ ] Retention period chosen and the privacy policy says what is kept ([operations.md](operations.md))
