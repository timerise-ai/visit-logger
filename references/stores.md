# Stores

`VisitStore` is the six queries the module needs, and nothing more: two
inserts, the two facts behind the announcement rule, a bounded newest-first
list, and the first human visit. It is not an abstraction over databases. Each
implementation below is idiomatic for its backend, and each throws on failure;
[recording.md](recording.md) decides what a failure means.

| Implementation | Status | Use it when |
|---|---|---|
| `createSupabaseVisitStore` | the earlier implementation's queries, renamed; type-checked | Supabase, or any Postgres reached through supabase-js |
| `createFirestoreVisitStore` | designed here; type-checked against firebase-admin 14, not run | Firebase / Firestore hosts |
| `createMemoryVisitStore` | the test double; runs in every suite | tests, and local prototyping before the table exists |

If the host talks to Postgres through Drizzle or Prisma, port the Supabase
implementation to it: same six queries, same NULL handling. Do not add
supabase-js to a host that does not use it. Mixed data-access styles in one
codebase are the thing a reviewer rejects.

## Supabase

```ts
// file: lib/visits/supabase-store.ts
import type { SupabaseClient } from "@supabase/supabase-js";

import type { PriorVisits } from "./core";
import type { NewPageVisit, NewVisitorEvent, VisitStore } from "./record";
import type { PageVisit, VisitFingerprint, VisitorEvent, VisitVia } from "./types";

/**
 * `VisitStore` over Postgres through supabase-js.
 *
 * Pass a service-role client, server-side only, and only after the request has
 * been authorised: the tables grant nothing to `anon` or `authenticated`, so a
 * visitor can neither read nor forge their own tracking rows. With generated
 * types, type the client as `SupabaseClient<Database>` and drop the row casts.
 */

interface FingerprintColumns {
  ip: string | null;
  country: string | null;
  region: string | null;
  city: string | null;
  timezone: string | null;
  browser: string | null;
  browser_version: string | null;
  os: string | null;
  device_type: string | null;
  user_agent: string | null;
  referrer: string | null;
  is_bot: boolean;
}

interface PageVisitRow extends FingerprintColumns {
  id: string;
  resource_id: string;
  resource_key: string | null;
  subject: string | null;
  via: VisitVia;
  path: string | null;
  created_at: string;
}

interface VisitorEventRow extends FingerprintColumns {
  id: string;
  subject: string;
  kind: string;
  resource_id: string | null;
  created_at: string;
}

/** One mapping per direction, shared by both tables. */
function toColumns(fingerprint: VisitFingerprint): FingerprintColumns {
  return {
    ip: fingerprint.ip,
    country: fingerprint.country,
    region: fingerprint.region,
    city: fingerprint.city,
    timezone: fingerprint.timezone,
    browser: fingerprint.browser,
    browser_version: fingerprint.browserVersion,
    os: fingerprint.os,
    device_type: fingerprint.deviceType,
    user_agent: fingerprint.userAgent,
    referrer: fingerprint.referrer,
    is_bot: fingerprint.isBot,
  };
}

function fromColumns(row: FingerprintColumns): VisitFingerprint {
  return {
    ip: row.ip,
    country: row.country,
    region: row.region,
    city: row.city,
    timezone: row.timezone,
    browser: row.browser,
    browserVersion: row.browser_version,
    os: row.os,
    deviceType: row.device_type,
    userAgent: row.user_agent,
    referrer: row.referrer,
    isBot: row.is_bot,
  };
}

function toPageVisit(row: PageVisitRow): PageVisit {
  return {
    ...fromColumns(row),
    id: row.id,
    resourceId: row.resource_id,
    resourceKey: row.resource_key,
    subject: row.subject,
    via: row.via,
    path: row.path,
    createdAt: row.created_at,
  };
}

function toVisitorEvent(row: VisitorEventRow): VisitorEvent {
  return {
    ...fromColumns(row),
    id: row.id,
    subject: row.subject,
    kind: row.kind,
    resourceId: row.resource_id,
    createdAt: row.created_at,
  };
}

function fail(error: { message: string }): never {
  throw new Error(error.message);
}

export function createSupabaseVisitStore(supabase: SupabaseClient): VisitStore {
  /** Human, non-internal page views of one resource, newest first. */
  const humanVisits = (resourceId: string) =>
    supabase
      .from("page_visits")
      .select("created_at")
      .eq("resource_id", resourceId)
      .eq("is_bot", false)
      .neq("via", "internal");

  return {
    async insertPageVisit(visit: NewPageVisit) {
      const { error } = await supabase.from("page_visits").insert({
        ...toColumns(visit),
        resource_id: visit.resourceId,
        resource_key: visit.resourceKey,
        subject: visit.subject,
        via: visit.via,
        path: visit.path,
      });
      if (error) fail(error);
    },

    async readPriorVisits(resourceId, visitor): Promise<PriorVisits> {
      // `eq` never matches NULL in Postgres, so a missing IP or user agent has
      // to be matched with `is` or the visitor is new on every page view.
      const byIp = visitor.ip
        ? humanVisits(resourceId).eq("ip", visitor.ip)
        : humanVisits(resourceId).is("ip", null);
      const sameVisitor = visitor.userAgent
        ? byIp.eq("user_agent", visitor.userAgent)
        : byIp.is("user_agent", null);

      const [anyone, same] = await Promise.all([
        humanVisits(resourceId).order("created_at", { ascending: false }).limit(1).maybeSingle(),
        sameVisitor.order("created_at", { ascending: false }).limit(1).maybeSingle(),
      ]);
      if (anyone.error) fail(anyone.error);
      if (same.error) fail(same.error);

      const at = (data: unknown) => (data as { created_at: string } | null)?.created_at ?? null;
      return { lastHumanAt: at(anyone.data), lastSameVisitorAt: at(same.data) };
    },

    async listPageVisits(resourceId, limit) {
      const { data, error } = await supabase
        .from("page_visits")
        .select("*")
        .eq("resource_id", resourceId)
        .neq("via", "internal")
        .order("created_at", { ascending: false })
        .limit(limit);
      if (error) fail(error);
      return ((data ?? []) as PageVisitRow[]).map(toPageVisit);
    },

    async firstHumanVisitAt(resourceId) {
      const { data, error } = await humanVisits(resourceId)
        .order("created_at", { ascending: true })
        .limit(1)
        .maybeSingle();
      if (error) fail(error);
      return (data as { created_at: string } | null)?.created_at ?? null;
    },

    async insertVisitorEvent(event: NewVisitorEvent) {
      const { error } = await supabase.from("visitor_events").insert({
        ...toColumns(event),
        subject: event.subject,
        kind: event.kind,
        resource_id: event.resourceId,
      });
      if (error) fail(error);
    },

    async listVisitorEvents(subject, { order, limit }) {
      const { data, error } = await supabase
        .from("visitor_events")
        .select("*")
        .eq("subject", subject)
        .order("created_at", { ascending: order === "asc" })
        .limit(limit);
      if (error) fail(error);
      return ((data ?? []) as VisitorEventRow[]).map(toVisitorEvent);
    },
  };
}
```

Notes:

- **Service role, server-side, after authorisation.** The tables grant nothing
  to `anon` or `authenticated` ([data-model.md](data-model.md)). A request
  reaches this store only after the host's own check has decided who the
  viewer is.
- **`is("ip", null)`, not `eq("ip", null)`.** `eq` compiles to `ip = NULL`,
  which is never true. Verified on Postgres 18: `WHERE ip = NULL` matches 0 of
  1 NULL rows, `WHERE ip IS NULL` matches 1. Without the branch, a visitor
  with no IP is "new" on every page view and the throttle never holds.
- **Generated types.** Typed as `SupabaseClient<Database>`, the row interfaces
  and the `as PageVisitRow[]` casts can go; keep `toColumns` and `fromColumns`,
  which are the single mapping between the fingerprint and its twelve columns.
  The earlier implementation wrote that mapping out four times, once per
  insert and once per reader.

## Firestore

```ts
// file: lib/visits/firestore-store.ts
import {
  FieldValue,
  type Firestore,
  type Query,
  type QueryDocumentSnapshot,
  Timestamp,
} from "firebase-admin/firestore";

import type { PriorVisits } from "./core";
import type { NewPageVisit, NewVisitorEvent, VisitStore } from "./record";
import type { PageVisit, VisitorEvent, VisitVia } from "./types";

/**
 * `VisitStore` over Firestore through the Admin SDK.
 *
 * The Admin SDK bypasses security rules, so the posture is: no rule grants
 * either collection to clients, and every read and write goes through the
 * server. That absence is the policy; keep it written down here.
 *
 * Fingerprint fields are stored camelCase, exactly as `VisitFingerprint`, and
 * every absent value is written as an explicit `null`: Firestore rejects
 * `undefined`, and an equality filter on `null` only matches documents where
 * the field exists.
 */

const PAGE_VISITS = "pageVisits";
const VISITOR_EVENTS = "visitorEvents";

/** `in` instead of `!= "internal"`, so no inequality constrains the `createdAt` ordering. */
const COUNTED_VIAS: VisitVia[] = ["link", "session"];

function iso(value: unknown): string {
  return value instanceof Timestamp ? value.toDate().toISOString() : new Date(0).toISOString();
}

function toPageVisit(doc: QueryDocumentSnapshot): PageVisit {
  const data = doc.data() as Omit<PageVisit, "id" | "createdAt"> & { createdAt: unknown };
  return { ...data, id: doc.id, createdAt: iso(data.createdAt) };
}

function toVisitorEvent(doc: QueryDocumentSnapshot): VisitorEvent {
  const data = doc.data() as Omit<VisitorEvent, "id" | "createdAt"> & { createdAt: unknown };
  return { ...data, id: doc.id, createdAt: iso(data.createdAt) };
}

export function createFirestoreVisitStore(db: Firestore): VisitStore {
  const humanVisits = (resourceId: string) =>
    db
      .collection(PAGE_VISITS)
      .where("resourceId", "==", resourceId)
      .where("isBot", "==", false)
      .where("via", "in", COUNTED_VIAS);

  const latest = async (query: Query): Promise<string | null> => {
    const snapshot = await query.orderBy("createdAt", "desc").limit(1).get();
    const doc = snapshot.docs[0];
    return doc ? iso(doc.get("createdAt")) : null;
  };

  return {
    async insertPageVisit(visit: NewPageVisit) {
      await db.collection(PAGE_VISITS).add({ ...visit, createdAt: FieldValue.serverTimestamp() });
    },

    async readPriorVisits(resourceId, visitor): Promise<PriorVisits> {
      const sameVisitor = humanVisits(resourceId)
        .where("ip", "==", visitor.ip)
        .where("userAgent", "==", visitor.userAgent);

      const [lastHumanAt, lastSameVisitorAt] = await Promise.all([
        latest(humanVisits(resourceId)),
        latest(sameVisitor),
      ]);
      return { lastHumanAt, lastSameVisitorAt };
    },

    async listPageVisits(resourceId, limit) {
      const snapshot = await db
        .collection(PAGE_VISITS)
        .where("resourceId", "==", resourceId)
        .where("via", "in", COUNTED_VIAS)
        .orderBy("createdAt", "desc")
        .limit(limit)
        .get();
      return snapshot.docs.map(toPageVisit);
    },

    async firstHumanVisitAt(resourceId) {
      const snapshot = await humanVisits(resourceId).orderBy("createdAt", "asc").limit(1).get();
      const doc = snapshot.docs[0];
      return doc ? iso(doc.get("createdAt")) : null;
    },

    async insertVisitorEvent(event: NewVisitorEvent) {
      await db.collection(VISITOR_EVENTS).add({ ...event, createdAt: FieldValue.serverTimestamp() });
    },

    async listVisitorEvents(subject, { order, limit }) {
      const snapshot = await db
        .collection(VISITOR_EVENTS)
        .where("subject", "==", subject)
        .orderBy("createdAt", order)
        .limit(limit)
        .get();
      return snapshot.docs.map(toVisitorEvent);
    },
  };
}
```

Notes:

- **Explicit `null`s.** `NewPageVisit` has every fingerprint field typed
  `string | null`, and `describeVisitor` never returns `undefined`, so the
  spread writes every field. Keep it that way if you extend the fingerprint:
  a missing field is invisible to `where("ip", "==", null)`.
- **`in`, not `!=`.** Excluding `internal` with `where("via", "in", ["link",
  "session"])` keeps every filter an equality filter, so `orderBy("createdAt")`
  needs only the composite indexes listed in [data-model.md](data-model.md).
  Add a value to `COUNTED_VIAS` when adding a `via`.
- **Server timestamps.** `createdAt` is `FieldValue.serverTimestamp()`, set at
  commit, so instance clocks never reorder the log; the reader converts the
  `Timestamp` to an ISO string. A document without one (written by hand) reads
  as the epoch rather than crashing the panel.

## Memory

```ts
// file: lib/visits/memory-store.ts
import type { PriorVisits } from "./core";
import type { NewPageVisit, NewVisitorEvent, VisitStore } from "./record";
import type { PageVisit, VisitorEvent } from "./types";

/**
 * `VisitStore` in memory, for tests and local prototyping. It mirrors the
 * Postgres implementation's semantics, including NULL-matches-NULL for the
 * same-visitor lookup. `clock` stamps `createdAt`; `fail` makes the named
 * operations throw, to exercise the failure paths.
 */
export function createMemoryVisitStore(
  options: { clock?: () => number; fail?: ReadonlyArray<keyof VisitStore> } = {},
) {
  const visits: PageVisit[] = [];
  const events: VisitorEvent[] = [];
  const clock = options.clock ?? Date.now;
  let sequence = 0;

  const guard = (operation: keyof VisitStore) => {
    if (options.fail?.includes(operation)) throw new Error(`${operation} failed`);
  };
  const newest = (a: { createdAt: string }, b: { createdAt: string }) =>
    b.createdAt.localeCompare(a.createdAt);
  const human = (resourceId: string) =>
    visits.filter((v) => v.resourceId === resourceId && !v.isBot && v.via !== "internal");

  const store: VisitStore = {
    async insertPageVisit(visit: NewPageVisit) {
      guard("insertPageVisit");
      sequence += 1;
      visits.push({ ...visit, id: `v${sequence}`, createdAt: new Date(clock()).toISOString() });
    },
    async readPriorVisits(resourceId, visitor): Promise<PriorVisits> {
      guard("readPriorVisits");
      const all = human(resourceId).sort(newest);
      const same = all.filter((v) => v.ip === visitor.ip && v.userAgent === visitor.userAgent);
      return { lastHumanAt: all[0]?.createdAt ?? null, lastSameVisitorAt: same[0]?.createdAt ?? null };
    },
    async listPageVisits(resourceId, limit) {
      guard("listPageVisits");
      return visits
        .filter((v) => v.resourceId === resourceId && v.via !== "internal")
        .sort(newest)
        .slice(0, limit);
    },
    async firstHumanVisitAt(resourceId) {
      guard("firstHumanVisitAt");
      return human(resourceId).sort(newest).at(-1)?.createdAt ?? null;
    },
    async insertVisitorEvent(event: NewVisitorEvent) {
      guard("insertVisitorEvent");
      sequence += 1;
      events.push({ ...event, id: `e${sequence}`, createdAt: new Date(clock()).toISOString() });
    },
    async listVisitorEvents(subject, { order, limit }) {
      guard("listVisitorEvents");
      const sorted = events.filter((e) => e.subject === subject).sort(newest);
      return (order === "asc" ? sorted.reverse() : sorted).slice(0, limit);
    },
  };

  return { store, visits, events };
}
```

It mirrors the Postgres semantics that matter, including NULL matching NULL in
the same-visitor lookup, and takes a `fail` list so the failure paths in
[recording.md](recording.md) can be tested.

## Stores checklist

- [ ] One implementation, matching the host's existing data-access style
- [ ] Missing IP and user agent matched as NULL, not by equality
- [ ] `internal` excluded inside the queries, so it cannot eat the list limit
- [ ] The fingerprint-to-columns mapping written once
- [ ] Firestore: every field written, `null` included; indexes deployed
