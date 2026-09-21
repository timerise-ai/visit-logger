# Operations

Running a visit log: what it may keep, how long, how to erase it, how to tell a
working log from a dead one, and the limits no code here removes.

## Privacy

An IP address is personal data under the GDPR, and so is a city attached to a
named person's email. The log sets no cookie and stores nothing on the device,
so it does not need an ePrivacy consent banner. It does need a legal basis and
a disclosure:

| Decision | The host's to make | The skill's default |
|---|---|---|
| Legal basis | usually legitimate interest: knowing whether a sent proposal was read | none; write it down |
| What is disclosed | the privacy policy names IP, approximate location, browser and device, and why | none; add the sentence |
| Retention of network data | a period | 13 months, then `ip`, `user_agent`, `referrer`, `city` cleared |
| Retention of the fact of a visit | a period or "for the life of the resource" | kept until the resource is deleted |
| Access | who in the company can see the panels | whoever can see the admin page |

If the host has a DPA or a privacy policy that already mentions IP and browser
data "for security purposes" only, attribution is a new purpose. Update the
text before shipping, not after.

## Retention

Schedule the purge from the migration ([data-model.md](data-model.md)). With
pg_cron on Supabase:

```sql
-- run: once, as the service role (Supabase SQL editor or psql)
SELECT cron.schedule(
  'purge-visit-network-data',
  '17 3 * * *',
  $$SELECT public.purge_visit_network_data(INTERVAL '13 months')$$
);
```

Without pg_cron, a daily cron route calling it through the service-role client
(`supabase.rpc("purge_visit_network_data")`) does the same.

The purge keeps counts, dates, country, region, browser and device. Rows it has
cleared stop grouping into sittings by visitor, because every cleared row now
has the same empty visitor key. That is the point of clearing them.

To delete old rows instead of clearing columns, swap each `UPDATE` for
`DELETE ... WHERE created_at < now() - retain`. The history panel then forgets
old visits entirely.

## Erasure

A subject's rows, on request:

```sql
-- run: on an erasure request, as the service role
DELETE FROM public.page_visits    WHERE subject = $1;
DELETE FROM public.visitor_events WHERE subject = $1;
```

Pass the normalised subject (`normalizeSubject`). `page_visits` rows with a null
subject (a visitor the host could not name) cannot be found by subject; find
them by resource instead, or let retention clear them.

## Health checks

Stored status needs a cron that can fail silently. These are derived, and
cheap:

```sql
-- run: as a health check, as the service role
-- The client roles are locked out. Both must be false.
SELECT has_table_privilege('anon', 'public.page_visits', 'SELECT') AS anon_can_read,
       has_table_privilege('authenticated', 'public.visitor_events', 'INSERT') AS user_can_forge;

-- The log is alive: the newest row of each kind.
SELECT 'page_visits' AS log, max(created_at) FROM public.page_visits
UNION ALL
SELECT 'visitor_events', max(created_at) FROM public.visitor_events;

-- Automation that slipped through: counted as human, but no browser was parsed.
SELECT user_agent, count(*) FROM public.page_visits
 WHERE NOT is_bot AND browser IS NULL
 GROUP BY user_agent ORDER BY count(*) DESC LIMIT 20;
```

The third query is how `looksAutomated` grows: every user agent it returns is a
candidate for the pattern, plus a test row in `fingerprint.test.ts`.

Watch the logs for the prefix `[visits]`. A steady stream of "Failed to record
visit" means a store misconfiguration that no page will ever show.

## Limits no code here removes

**Mail scanners with ordinary browsers.** Microsoft Defender Safe Links, Mimecast
and Proofpoint detonate links in a sandboxed browser that can identify itself
as a normal Chrome, from a cloud address, seconds after delivery. No user-agent
rule catches them. What helps, as designs not shipped: the host knows when it
sent the link, so a first open within about a minute of sending can be shown
as "possibly automated". Scanners rarely return, so a second sitting from a
different network is stronger evidence of a person than the first.

**Staff who are not signed in.** A colleague opening the customer's link on
their own phone is the customer, as far as the log can tell. Designs, not
shipped: an office IP allowlist that maps to `via: "internal"`, or asking staff
to preview through the admin page.

**Two page views in the same few milliseconds.** The announcement reads prior
visits, then inserts. Two requests that interleave can both see "no prior
visit" and announce twice. The earlier implementation accepted one duplicate ping. If it
matters, move the assessment and the insert into one Postgres function that
takes `pg_advisory_xact_lock` on the resource id first; that function is a
design, not verified here.

**Shared networks.** Same office, same browser build: one visitor
([rules.md](rules.md)).

## Upgrading an existing log

When the host already has a visit log, check it against the ledger in
[provenance.md](provenance.md) in its fix order. The first-sign-in check, on
Supabase, tells you how many accounts an age-based "new user" test misfiled:

```sql
-- run: once against the host's database, as the service role
SELECT count(*) AS misfiled_as_returning
  FROM auth.users
 WHERE email_confirmed_at - created_at > INTERVAL '5 minutes';
```

## Operations checklist

- [ ] Legal basis and privacy text written; the purpose names attribution
- [ ] Purge scheduled; period agreed
- [ ] Erasure path documented for support
- [ ] Health queries run once after deploy; `anon` is locked out
- [ ] Limits above explained to whoever reads the notifications
