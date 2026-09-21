# Recording

The write path and the read path, over whichever store the host uses, and the
request-side entry points that keep both off the response.

## Failure contract

Every function here is safe to call from a request. None of them throws; each
failure has a defined meaning:

| Failure | Result | Why |
|---|---|---|
| Store factory throws (missing env) | logged inside `after()`; page unaffected | a tracking misconfiguration must not take down the resource |
| Prior-visit read fails | row still written; `shouldAnnounce: true`, novelty unknown | an extra ping is cheaper than a missed first open |
| Insert fails | `QUIET`: no announcement | unrecorded means unannounced; the next visit gets another chance |
| Announcement fails | logged | the row is already written |
| History read fails | `failed: true`, empty lists | the panel says "could not load", never "not opened yet" |
| Event read fails | `failed: true` / `null` origin | the same |

The store implementations throw on any database error; these functions decide
what a failure means. That split is what makes them testable against the memory
store in [stores.md](stores.md).

## Write and read paths

```ts
// file: lib/visits/record.ts
import {
  ANNOUNCE_UNKNOWN,
  assessVisit,
  EMPTY_SUMMARY,
  pickOriginEvent,
  type PriorVisits,
  QUIET,
  SESSION_WINDOW_MS,
  summarizeVisits,
  type VisitAssessment,
} from "./core";
import type {
  PageVisit,
  VisitFingerprint,
  VisitorEvent,
  VisitorEventList,
  VisitSummary,
} from "./types";

/**
 * The write and read paths, over whichever backend the host uses.
 *
 * Every function here is safe to call from a request: writes never throw and
 * reads never throw. A tracking failure must not turn a working page or a
 * sign-in into an error, and a panel that says "could not load" beats a
 * crashed admin page.
 */

export type NewPageVisit = Omit<PageVisit, "id" | "createdAt">;
export type NewVisitorEvent<K extends string = string> = Omit<VisitorEvent<K>, "id" | "createdAt">;

/**
 * The queries the module needs, and nothing more. Implementations throw on
 * failure; the functions below decide what a failure means.
 */
export interface VisitStore {
  insertPageVisit(visit: NewPageVisit): Promise<void>;
  /** Both facts exclude bots and `internal` visits. */
  readPriorVisits(
    resourceId: string,
    visitor: Pick<VisitFingerprint, "ip" | "userAgent">,
  ): Promise<PriorVisits>;
  /** Newest first, `internal` excluded in the query so it cannot eat the limit. */
  listPageVisits(resourceId: string, limit: number): Promise<PageVisit[]>;
  /** Earliest human, non-internal page view. */
  firstHumanVisitAt(resourceId: string): Promise<string | null>;
  insertVisitorEvent(event: NewVisitorEvent): Promise<void>;
  listVisitorEvents(
    subject: string,
    options: { order: "asc" | "desc"; limit: number },
  ): Promise<VisitorEvent[]>;
}

/**
 * Subjects are compared exactly, so they are normalised in one place. The
 * source keyed everything by email; if yours is a case-sensitive user id,
 * make this return `subject` unchanged.
 */
export function normalizeSubject(subject: string): string {
  return subject.trim().toLowerCase();
}

function logFailure(what: string, error: unknown): void {
  console.error(`[visits] ${what}:`, error instanceof Error ? error.message : error);
}

/**
 * Writes one page view and says whether it deserves an announcement.
 *
 * Bots and `internal` visits are stored for auditability but never assessed,
 * so they can never announce. The assessment reads *before* the insert, so the
 * new row cannot answer its own question.
 */
export async function recordPageVisit(
  store: VisitStore,
  visit: NewPageVisit,
  options: { now?: number; windowMs?: number } = {},
): Promise<VisitAssessment> {
  const entry: NewPageVisit = {
    ...visit,
    subject: visit.subject === null ? null : normalizeSubject(visit.subject),
  };

  let assessment = QUIET;
  if (!entry.isBot && entry.via !== "internal") {
    try {
      const prior = await store.readPriorVisits(entry.resourceId, entry);
      assessment = assessVisit(prior, options.now ?? Date.now(), options.windowMs ?? SESSION_WINDOW_MS);
    } catch (error) {
      logFailure("Failed to read prior visits", error);
      assessment = ANNOUNCE_UNKNOWN;
    }
  }

  try {
    await store.insertPageVisit(entry);
  } catch (error) {
    // Unrecorded means unannounced: the next visit gets another chance.
    logFailure("Failed to record visit", error);
    return QUIET;
  }

  return assessment;
}

export async function recordVisitorEvent<K extends string>(
  store: VisitStore,
  event: NewVisitorEvent<K>,
): Promise<void> {
  try {
    await store.insertVisitorEvent({ ...event, subject: normalizeSubject(event.subject) });
  } catch (error) {
    logFailure(`Failed to record ${event.kind}`, error);
  }
}

/**
 * Visit history for one resource. Reads `limit + 1` rows to learn whether
 * there are more than `limit`, and only then pays for the first-visit query.
 */
export async function readVisitSummary(
  store: VisitStore,
  resourceId: string,
  limit = 200,
): Promise<VisitSummary> {
  try {
    const rows = await store.listPageVisits(resourceId, limit + 1);
    const truncated = rows.length > limit;
    const firstAt = truncated ? await store.firstHumanVisitAt(resourceId) : undefined;
    return summarizeVisits(truncated ? rows.slice(0, limit) : rows, { truncated, firstAt });
  } catch (error) {
    logFailure("Failed to list visits", error);
    return { ...EMPTY_SUMMARY, failed: true };
  }
}

/** A subject's lifecycle events, newest first. */
export async function readVisitorEvents<K extends string = string>(
  store: VisitStore,
  subject: string,
  limit = 20,
): Promise<VisitorEventList<K>> {
  try {
    const rows = await store.listVisitorEvents(normalizeSubject(subject), {
      order: "desc",
      limit: limit + 1,
    });
    return {
      events: rows.slice(0, limit) as VisitorEvent<K>[],
      truncated: rows.length > limit,
      failed: false,
    };
  } catch (error) {
    logFailure("Failed to read events", error);
    return { events: [], truncated: false, failed: true };
  }
}

/**
 * Where someone came from, by `pickOriginEvent`. Reads the oldest 50: the
 * origin is at the start of a history, never the end.
 */
export async function findOriginEvent<K extends string>(
  store: VisitStore,
  subject: string,
  preference: readonly K[],
): Promise<VisitorEvent<K> | null> {
  try {
    const rows = await store.listVisitorEvents(normalizeSubject(subject), {
      order: "asc",
      limit: 50,
    });
    return pickOriginEvent(rows as VisitorEvent<K>[], preference);
  } catch (error) {
    logFailure("Failed to read events", error);
    return null;
  }
}
```

`normalizeSubject` runs on the way in *and* on the way out. The source
lower-cased on both sides too; a store that normalises only on write finds
nothing when an admin page passes the address as the account has it.

## Request-side entry points

```ts
// file: lib/visits/track.ts
import { after } from "next/server";

import type { VisitAssessment } from "./core";
import { describeVisitor, type EdgeHeaders, vercelEdge } from "./fingerprint";
import {
  type NewPageVisit,
  type NewVisitorEvent,
  recordPageVisit,
  recordVisitorEvent,
  type VisitStore,
} from "./record";
import type { VisitFingerprint } from "./types";

/**
 * Request-side entry points. Both read the fingerprint *now*, synchronously,
 * because the request is not guaranteed to outlive the response, and defer the
 * store, the write and the announcement with `after()`. The store is a factory
 * for the same reason: a missing env var must fail the tracking, not the page.
 */

export type VisitTarget = Omit<NewPageVisit, keyof VisitFingerprint>;

export type Announce = (
  assessment: VisitAssessment,
  fingerprint: VisitFingerprint,
) => Promise<void>;

export function trackPageVisit(options: {
  getStore: () => VisitStore;
  /** A `Request`, or `await headers()` in a Server Component. */
  request: Request | Headers;
  visit: VisitTarget;
  edge?: EdgeHeaders;
  /** Called after the write, only when it deserves it. Send the message here. */
  announce?: Announce;
}): void {
  const fingerprint = describeVisitor(options.request, options.edge ?? vercelEdge);

  after(async () => {
    try {
      const store = options.getStore();
      const assessment = await recordPageVisit(store, { ...fingerprint, ...options.visit });
      if (assessment.shouldAnnounce && options.announce) {
        await options.announce(assessment, fingerprint);
      }
    } catch (error) {
      console.error("[visits] Visit tracking failed:", error);
    }
  });
}

export function trackVisitorEvent<K extends string>(options: {
  getStore: () => VisitStore;
  request: Request | Headers;
  event: Omit<NewVisitorEvent<K>, keyof VisitFingerprint>;
  edge?: EdgeHeaders;
}): VisitFingerprint {
  const fingerprint = describeVisitor(options.request, options.edge ?? vercelEdge);

  after(async () => {
    try {
      await recordVisitorEvent(options.getStore(), { ...fingerprint, ...options.event });
    } catch (error) {
      console.error("[visits] Event tracking failed:", error);
    }
  });

  // Returned so the caller can put the same origin in its own notification.
  return fingerprint;
}
```

Three things about `after()` that decide whether this works:

- **Read the request first.** Only the fingerprint crosses into the callback.
  In a Server Component, `headers()` cannot be called inside `after()` at all;
  in a route handler it can, but the request object is not guaranteed to
  outlive the response. `describeVisitor` runs before `after()` in both entry
  points for that reason.
- **It needs a dynamic render.** On a statically rendered page, `after()` runs
  at build time or on revalidation. Reading `headers()` makes the page dynamic,
  which a page that authorises its viewer is anyway.
- **It is supported in route handlers, Server Components, Server Functions and
  `proxy.ts`**. On Vercel the function stays alive until the callback
  settles. A Node server or Docker container runs it too; static export does
  not, and other adapters must provide `waitUntil`.

## Choosing the store

```ts
// file: lib/visits/store.ts
import { createClient } from "@supabase/supabase-js";

import type { VisitStore } from "./record";
import { createSupabaseVisitStore } from "./supabase-store";

/**
 * The one place the backend is chosen. Server-only: it holds the service-role
 * key. Replace the body with the host's own service-role client factory, or
 * with `createFirestoreVisitStore(getFirestore())`. Throws when unconfigured;
 * `trackPageVisit` calls it inside `after()`, so that failure is logged and
 * the page still renders.
 */
export function getVisitStore(): VisitStore {
  const url = process.env.NEXT_PUBLIC_SUPABASE_URL;
  const key = process.env.SUPABASE_SERVICE_ROLE_KEY;
  if (!url || !key) throw new Error("Visit store: Supabase service-role env is not set");

  return createSupabaseVisitStore(
    createClient(url, key, { auth: { persistSession: false, autoRefreshToken: false } }),
  );
}
```

This is the one file that names a backend. The store is created lazily, inside
`after()`, so a missing key is a log line rather than a broken page.

## Announcing

The module decides *whether* to announce; the host decides *how*. `announce`
receives the assessment and the fingerprint, and the helpers below turn them
into the lines every notification carries. The labels are the host's language;
the facts are the module's.

```ts
// file: lib/visits/announce.ts
import type { VisitAssessment } from "./core";
import { describeOrigin, type VisitFingerprint } from "./types";

/**
 * The "where from" block every notification carries, as Slack mrkdwn lines.
 * Swap the labels for the host's language; the facts stay the same.
 */
export function originLines(fingerprint: VisitFingerprint): string[] {
  const origin = describeOrigin(fingerprint);
  const lines = [
    `*Location:* ${origin.location ?? "unknown"}`,
    `*Browser:* ${origin.client ?? "unknown"}`,
  ];
  if (origin.timezone) lines.push(`*Timezone:* ${origin.timezone}`);
  if (origin.ip) lines.push(`*IP:* ${origin.ip}`);
  return lines;
}

/** "opened for the first time" / "opened again, from a new device" / "came back". */
export function openedHeadline(subject: string, assessment: VisitAssessment): string {
  if (assessment.isFirstVisit) return `${subject} opened it for the first time`;
  if (assessment.isNewVisitor) return `${subject} opened it from a new device or network`;
  return `${subject} came back to it`;
}
```

Never announce from anywhere else. The assessment already excludes bots and
staff, applies the per-visitor window, and fails towards announcing. A second
code path that posts on "any visit" undoes all three.

What to include in the message, in order of usefulness to the person reading
it: who, what, first/new/back, location and timezone (when to call), client
(phone or laptop), referrer (where the link was clicked, when it survived).

## Recording checklist

- [ ] `describeVisitor` called before `after()`, never inside it
- [ ] Store obtained through a factory inside `after()`
- [ ] Announcements only from `trackPageVisit`'s `announce`, only when `shouldAnnounce`
- [ ] `readVisitSummary` used by the admin page, so `truncated` and `failed` reach the UI
- [ ] Subjects normalised on write and on read
