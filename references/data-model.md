# Data model

Two logs that share one fingerprint. Keep them as two tables: they are keyed
differently, read differently, and deleted differently.

| | `page_visits` | `visitor_events` |
|---|---|---|
| One row per | real page view of a resource | lifecycle moment of a subject |
| Keyed by | `resource_id` (required) | `subject` (required) |
| Read as | a resource's history, grouped into sittings | a subject's timeline; the origin event |
| Written | every page view, bots and staff included | at sign-in request, first sign-in, later sign-ins, first resource |
| When the resource is deleted | rows go with it (`ON DELETE CASCADE`) | rows stay, `resource_id` set to NULL |
| Announces | first open, new visitor, return after the window | never, by itself; the host's own events do |

Store every page view. Sittings are computed when read, so the grouping window
can change without a migration and without losing history.

## Neutral entities

`PageVisit`, `VisitSession`, `VisitSummary`, `VisitorEvent` and
`VisitorEventList` are defined in plain TypeScript in `lib/visits/types.ts`
([fingerprint.md](fingerprint.md)). Timestamps cross the store boundary as ISO
8601 strings, absent values as `null`, and ids as strings. Neither
`timestamptz` nor a Firestore `Timestamp` leaks above the store.

## Postgres / Supabase migration

Rename `public.resources` to the host's visited table before running it (the
rename is in [adaptation.md](adaptation.md)). Verified on PostgreSQL 18 with
Supabase's `anon`, `authenticated` and `service_role` roles and its default
privileges recreated: it applies cleanly twice; `anon` and `authenticated` are
refused on read, on write and on the purge function; the `via` and `kind`
checks reject unknown values; the purge clears only rows past the retention
period and returns 0 on a second run; deleting a resource removes its visits
and detaches its events.

```sql
-- file: supabase/migrations/<timestamp>_visit_log.sql
-- =====================================================
-- Visit log: page views of a resource, and lifecycle events of a subject.
-- =====================================================
-- Rename `public.resources` to the table whose rows are visited (proposals,
-- demos, listings, quotes) before running. Both tables are written and read
-- only by the server, through the service-role client.

-- =====================================================
-- TABLE: page_visits (one row per page view)
-- =====================================================
CREATE TABLE IF NOT EXISTS public.page_visits (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- What was visited. The cascade is the delete path: removing a resource
    -- removes its visit history with it.
    resource_id     UUID NOT NULL REFERENCES public.resources(id) ON DELETE CASCADE,
    -- The resource's label at visit time; it can be renamed or relinked later.
    resource_key    TEXT,

    -- Who, when known. Captured at visit time, because it can change later.
    subject         TEXT,

    -- How they got in. `internal` rows are staff previews: stored for audit,
    -- never counted, listed or announced.
    via             TEXT NOT NULL DEFAULT 'link'
        CONSTRAINT page_visits_via_check CHECK (via IN ('link', 'session', 'internal')),

    path            TEXT,
    referrer        TEXT,

    -- Network origin. `ip` is personal data under GDPR: keep it only with a
    -- stated purpose and a retention period (see purge function below).
    ip              TEXT,
    country         TEXT,
    region          TEXT,
    city            TEXT,
    timezone        TEXT,

    -- Client, parsed from the User-Agent at write time.
    browser         TEXT,
    browser_version TEXT,
    os              TEXT,
    device_type     TEXT,
    user_agent      TEXT,

    -- Crawlers, scripts, link scanners. Recorded rather than dropped, so a
    -- suspiciously early "open" can be explained, but never counted.
    is_bot          BOOLEAN NOT NULL DEFAULT FALSE,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE public.page_visits IS 'One row per page view of a resource: history panel and first-open announcements';
COMMENT ON COLUMN public.page_visits.via IS 'link = shared link, session = signed-in owner, internal = staff preview (never counted)';

-- History panel, the announcement throttle and the first-visit lookup all
-- read one resource newest-first.
CREATE INDEX IF NOT EXISTS idx_page_visits_resource
    ON public.page_visits(resource_id, created_at DESC);

-- Erasure requests and "everything this person opened".
CREATE INDEX IF NOT EXISTS idx_page_visits_subject
    ON public.page_visits(subject)
    WHERE subject IS NOT NULL;

-- =====================================================
-- TABLE: visitor_events (lifecycle moments of a subject)
-- =====================================================
CREATE TABLE IF NOT EXISTS public.visitor_events (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),

    -- Normalised subject (lower-cased email in the reference host).
    subject         TEXT NOT NULL,

    -- The host's own kinds. These are the reference set:
    -- link_requested  = sign-in form submitted (certainly the person's browser)
    -- account_created = first verified sign-in
    -- signed_in       = any later sign-in
    -- resource_created = the subject's first resource
    kind            TEXT NOT NULL
        CONSTRAINT visitor_events_kind_check
        CHECK (kind IN ('link_requested', 'account_created', 'signed_in', 'resource_created')),

    -- Set when the event is about one resource; kept if the resource goes.
    resource_id     UUID REFERENCES public.resources(id) ON DELETE SET NULL,

    ip              TEXT,
    country         TEXT,
    region          TEXT,
    city            TEXT,
    timezone        TEXT,
    browser         TEXT,
    browser_version TEXT,
    os              TEXT,
    device_type     TEXT,
    user_agent      TEXT,
    referrer        TEXT,

    -- Sign-in links are often opened by mail-security scanners before the
    -- person ever sees the email. Kept, but flagged.
    is_bot          BOOLEAN NOT NULL DEFAULT FALSE,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE public.visitor_events IS 'Fingerprint at sign-in request, account creation, sign-in and first resource';

CREATE INDEX IF NOT EXISTS idx_visitor_events_subject
    ON public.visitor_events(subject, created_at DESC);

CREATE INDEX IF NOT EXISTS idx_visitor_events_resource
    ON public.visitor_events(resource_id)
    WHERE resource_id IS NOT NULL;

-- =====================================================
-- SECURITY
-- =====================================================
-- RLS on with no policy: `anon` and `authenticated` see nothing, and the
-- service role bypasses RLS. The REVOKE matters as much as the RLS: Supabase
-- default privileges grant new public tables to `anon` and `authenticated`.
ALTER TABLE public.page_visits ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.visitor_events ENABLE ROW LEVEL SECURITY;

REVOKE ALL ON public.page_visits, public.visitor_events FROM anon, authenticated;
GRANT ALL ON public.page_visits, public.visitor_events TO service_role;

-- =====================================================
-- RETENTION (optional; schedule it, or delete this block)
-- =====================================================
-- Forgets network identity after the retention period and keeps the facts:
-- counts, dates, country, browser. Rows older than that stop grouping into
-- sittings by visitor, which is the point.
CREATE OR REPLACE FUNCTION public.purge_visit_network_data(retain INTERVAL DEFAULT INTERVAL '13 months')
RETURNS INTEGER
LANGUAGE plpgsql
SECURITY INVOKER
SET search_path = ''
AS $$
DECLARE
    visits_cleared INTEGER;
    events_cleared INTEGER;
BEGIN
    UPDATE public.page_visits
       SET ip = NULL, user_agent = NULL, referrer = NULL, city = NULL
     WHERE created_at < now() - retain
       AND (ip IS NOT NULL OR user_agent IS NOT NULL OR referrer IS NOT NULL OR city IS NOT NULL);
    GET DIAGNOSTICS visits_cleared = ROW_COUNT;

    UPDATE public.visitor_events
       SET ip = NULL, user_agent = NULL, referrer = NULL, city = NULL
     WHERE created_at < now() - retain
       AND (ip IS NOT NULL OR user_agent IS NOT NULL OR referrer IS NOT NULL OR city IS NOT NULL);
    GET DIAGNOSTICS events_cleared = ROW_COUNT;

    RETURN visits_cleared + events_cleared;
END;
$$;

-- Functions are executable by PUBLIC by default.
REVOKE EXECUTE ON FUNCTION public.purge_visit_network_data(INTERVAL) FROM PUBLIC, anon, authenticated;
GRANT EXECUTE ON FUNCTION public.purge_visit_network_data(INTERVAL) TO service_role;

-- With pg_cron enabled:
-- SELECT cron.schedule('purge-visit-network-data', '17 3 * * *',
--                      $$SELECT public.purge_visit_network_data()$$);
```

### Why each odd-looking line is there

| Line | Reason |
|---|---|
| `resource_id ... ON DELETE CASCADE` | the delete path: a deleted resource must not leave a history of visits to nothing |
| `visitor_events.resource_id ... ON DELETE SET NULL` | the sign-up origin outlives the resource it was about |
| `subject` on `page_visits`, captured at visit time | an account's email can change; the log records who it was then |
| `resource_key` | the resource's label at visit time: a demo folder, a version, a title. It can be relinked later |
| `CHECK (via IN ...)` / `CHECK (kind IN ...)` | a typo in a `via` value would silently count staff as customers |
| no RLS policy | `service_role` bypasses RLS; with RLS on and no policy, the client roles see nothing |
| `REVOKE ALL ... FROM anon, authenticated` | Supabase's default privileges grant every new `public` table to both. RLS alone is one misconfigured policy away from an open table |
| `REVOKE EXECUTE ... FROM PUBLIC` | functions are executable by `PUBLIC` by default |
| `SECURITY INVOKER`, `SET search_path = ''` | the purge runs with the caller's rights and cannot be hijacked through the search path |
| `idx_page_visits_subject` (partial) | erasure requests and "everything this person opened" |

One index serves the three hot reads on `page_visits`: the history panel, the
announcement throttle and the first-visit lookup all filter by `resource_id`
and order by `created_at`. The same-visitor lookup adds `ip` and `user_agent`
filters on top. At the volumes a sales log sees (tens to low thousands of rows
per resource), that is cheap. Add `(resource_id, ip, user_agent, created_at
DESC)` only if `EXPLAIN` says otherwise.

### Generated types

With `supabase gen types`, the store can drop its row interfaces and casts; see
[stores.md](stores.md). Regenerate after running the migration.

## Firestore

Two top-level collections, `pageVisits` and `visitorEvents`. The documents hold
the entity fields camelCase, as in `types.ts`, with `createdAt` as a server
timestamp. Every absent value is written as an explicit `null`: an equality
filter on `null` matches only documents where the field exists and is `null`,
so a missing field would make the same-visitor lookup treat a repeat visitor as
new.

Composite indexes the queries in [stores.md](stores.md) need, as
`firestore.indexes.json`:

```json
{
  "indexes": [
    {
      "collectionGroup": "pageVisits",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "resourceId", "order": "ASCENDING" },
        { "fieldPath": "isBot", "order": "ASCENDING" },
        { "fieldPath": "via", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "pageVisits",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "resourceId", "order": "ASCENDING" },
        { "fieldPath": "isBot", "order": "ASCENDING" },
        { "fieldPath": "via", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "ASCENDING" }
      ]
    },
    {
      "collectionGroup": "pageVisits",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "resourceId", "order": "ASCENDING" },
        { "fieldPath": "isBot", "order": "ASCENDING" },
        { "fieldPath": "via", "order": "ASCENDING" },
        { "fieldPath": "ip", "order": "ASCENDING" },
        { "fieldPath": "userAgent", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "pageVisits",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "resourceId", "order": "ASCENDING" },
        { "fieldPath": "via", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "visitorEvents",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "subject", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "DESCENDING" }
      ]
    },
    {
      "collectionGroup": "visitorEvents",
      "queryScope": "COLLECTION",
      "fields": [
        { "fieldPath": "subject", "order": "ASCENDING" },
        { "fieldPath": "createdAt", "order": "ASCENDING" }
      ]
    }
  ]
}
```

The index set is derived from the queries and has not been deployed. If
Firestore still asks for one, its error message carries a link that creates
exactly the missing index.

**Access posture.** The Admin SDK bypasses security rules. Write no rule that
grants `pageVisits` or `visitorEvents` to clients: the absence is the policy,
and every read and write goes through the server. Someone porting this to
Supabase who reasons "there were no rules, so no policies are needed" ships an
open table. On Postgres the equivalent is the `REVOKE` above.

**Deletes.** Firestore has no cascade. When a resource is deleted, delete its
`pageVisits` in batches of up to 500 in the same server action, and null the
`resourceId` of its `visitorEvents`.

**Retention.** No server-side function: a scheduled function (Cloud Scheduler
or a cron route) that queries `createdAt < cutoff` and overwrites `ip`,
`userAgent`, `referrer` and `city` with `null` in batches.

## Mapping

| Concern | Postgres / Supabase | Firestore |
|---|---|---|
| Resource delete | `ON DELETE CASCADE` | batch delete in the same action |
| Exclude staff | `via <> 'internal'` | `via in ['link', 'session']`: `in` is an equality filter, so no inequality constrains the ordering |
| Match a missing IP | `ip IS NULL` | `ip == null`, only because `null` is written explicitly |
| Counts | the summary counts rows read | the same; no aggregates needed |
| Access | `REVOKE`, RLS on, service role only | no client rules, Admin SDK only |
| Retention | `purge_visit_network_data()` + pg_cron | scheduled batch update |

## Data model checklist

- [ ] `public.resources` renamed to the visited table in the migration
- [ ] `via` and `kind` checks list exactly the host's values
- [ ] `REVOKE` present; `anon` cannot `SELECT` (see [operations.md](operations.md))
- [ ] Firestore: `null` written explicitly; indexes deployed; no client rules
- [ ] Delete path for the resource covers its visits
- [ ] Retention scheduled, or the purge block deleted on purpose
