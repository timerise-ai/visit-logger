# Provenance

This is the engineering ledger for the person editing the skill, not a story
for the reader of the README. It separates three things: what the audit of the
earlier implementation changed and how the templates verify it, what was kept
deliberately and why it is safe, and what was designed here and has never run
in production.

The earlier implementation was a sales site on Next.js with Supabase, where two
logs shared one fingerprint helper:

- **Resource visits.** Gated builds served by a route handler, one row per HTML
  page view, grouped into sittings in an internal console, with a Slack
  "customer opened it" message.
- **Account events.** Sign-in link requested, account created, signed in and
  first resource, with a "signed up from" line on the customer's console page.

A fingerprint helper, two stores, two migrations, two console panels and the
Slack origin block. No test covered any of it.

The architecture here is that one's. The templates are not a transcription, and
this ledger says why. Every entry was verified by reading the code it describes.
Entries marked **reproduced** were also run: user-agent claims through Next
16.2.12's own `userAgent()`, header decoding through Node 22's HTTP parser, SQL
on PostgreSQL 18 with Supabase's roles and default privileges recreated.
Supabase Auth behaviour was checked against the auth server's own code.

## Fixed in the templates

### 1. Automated clients counted as the customer (reproduced)

The bot flag was `userAgent().isBot` alone, a crawler list. HeadlessChrome,
`curl`, `python-requests` and a request with no user agent all came back
`false`. Mail-security scanners follow emailed links in headless browsers, so
each such open was stored as a human page view, counted in the console, and
eligible for the "opened the demo" Slack message.

**Shipped:** `looksAutomated`, with a test row per client and real browsers
(CUBOT phones included) that must not match. See [fingerprint.md](fingerprint.md).

### 2. New accounts filed as returning sign-ins

The auth callback called a user new when `created_at` was under five minutes
old. Supabase creates the user when the link is *requested*, so anyone who
opened the email more than five minutes later was recorded `signed_in`, and
the "workspace created" Slack message, gated on the same test, never fired for
them. Nothing on screen showed it: those customers simply had no
"account created" row.

**Shipped:** `isFirstSignIn` on `email_confirmed_at`. See [capture.md](capture.md).

### 3. A "first opened" date that was the oldest row that fitted

The history read the newest 200 page views and took the earliest of them as
"first". Past 200, the console printed a wrong first-opened date and counts
with no sign of truncation.

**Shipped:** a `limit + 1` read, `truncated`, a separate first-visit query,
and "at least" in the panel. See [rules.md](rules.md) and
[recording.md](recording.md).

### 4. A failed read shown as "Not opened yet"

Every read returned an empty list on error. The panel then said the customer
had never looked, which is the one wrong answer a salesperson acts on.

**Shipped:** `failed` on both result types, rendered as an alert. See
[admin-ui.md](admin-ui.md).

### 5. Browser version in full, device guessed for scripts (reproduced)

Chrome showed as "131.0.0.0". The parser leaves the device type undefined for
desktops, and it was defaulted to `"desktop"` for everything, including
`curl` and a request with no user agent; `python-requests` parsed as
`"wearable"`.

**Shipped:** the major version in the client line; `deviceType` null for
automated clients.

### 6. The rules lived inside the queries, untested

The announcement decision was interleaved with its two queries, and the
sign-up origin selection sat inside a database read. Only the grouping was a
pure function, and nothing tested it.

**Shipped:** `core.ts` (pure), the `VisitStore` seam, a memory store, and 54
tests. See [testing.md](testing.md).

### 7. The fingerprint mapped four times

Two inserts and two row mappers each spelled out the twelve fingerprint
columns. A field added to one would silently miss the others.

**Shipped:** `toColumns` / `fromColumns`, once. See [stores.md](stores.md).

### 8. Comments that promised formats the code did not produce

"Kraków, Małopolskie, PL" (the region header is a code) and "Chrome 131 on
macOS" (the parser says "Mac OS", version "131.0.0.0").

**Shipped:** comments that match tested output.

### 9. Two client-IP parsers

The sign-in route parsed `x-forwarded-for` itself, with an `"unknown"`
fallback, beside the shared helper. They would drift the day the edge changed.

**Shipped:** one `EdgeHeaders.ip`; the probe in
[adaptation.md](adaptation.md) looks for the second parser.

## Kept deliberately

- **Geolocation from edge headers**, no IP-lookup service: no third-party call
  on the request path, no extra processor to disclose.
- **Every page view stored; sittings computed on read.** The window can change
  without a migration.
- **Bots and staff stored, flagged, never counted.** An early "open" stays
  explainable, and an audit sees everything.
- **Visitor = IP + user agent**, with the limits listed in [rules.md](rules.md).
  No cookie.
- **30-minute window, per visitor rather than per resource.**
- **A failed prior-visit read announces; a failed insert does not.**
- **Check-then-insert.** Two interleaved page views can both announce. One
  duplicate ping was accepted; the fix is described in
  [operations.md](operations.md).
- **Staff wins over the link.** An admin holding the customer's token cookie
  is recorded as `internal`.
- **The sign-in form submission is the preferred origin**, ahead of the link
  click a scanner may make.
- **Fingerprint read synchronously; everything else in `after()`.**
- **512-character cap** on free-text fields.
- **Service-role-only tables**, with explicit `REVOKE`.
- **Raw IP stored.** Minimising it (truncating, hashing) breaks the visitor
  key; that is the host's legal call, now with a retention function to make it.

## Changed without a defect

- Renamed to neutral vocabulary: `demo_visits` to `page_visits`, `brief_id` to
  `resource_id`, `demo_folder` to `resource_key`, `email` to `subject`, `token`
  to `link`, `admin` to `internal`, `otp_requested` to `link_requested`,
  `brief_created` to `resource_created`.
- The "service role has full access" RLS policy dropped: `service_role`
  bypasses RLS, so it granted nothing. RLS on and the `REVOKE` kept.
- Formatters return `null` and language-neutral fragments. The earlier
  implementation's "Unknown location" and "on" are now host strings.
- A sitting carries its entry `referrer`.
- The store is a factory resolved inside `after()`.

## Added (designed here, not run in production)

- `isPageView` counts App Router client navigations (`rsc: 1`) and drops `<Link>`
  and segment prefetches and Server Actions. The header names were checked
  against Next 16.2.12's router constants; the behaviour has not been
  exercised in a running app.
- The Server Component capture point, the admin page example, and the Supabase
  Auth sign-in routes as templates.
- `cloudflareEdge` and `utf8FromLatin1` (the Latin-1 decoding was reproduced;
  the header names come from Cloudflare's documentation), and `noEdge`.
- The Firestore store and its indexes.
- `purge_visit_network_data()` (verified on PostgreSQL 18) and the subject
  index for erasure.
- The memory store, the three suites, and the structure-only panels with
  strings objects.

## If you are upgrading an existing log

Most damaging first:

1. **First sign-in** (entry 2). New customers are misfiled and their creation
   ping is lost. The query in [operations.md](operations.md) counts them.
2. **Bot detection** (entry 1). False "opened" pings reach sales.
3. **Truncation and failure signals** (entries 3 and 4).
4. **Tests and the single mapping** (entries 6 and 7), before anything else is
   changed.
5. The rest.
