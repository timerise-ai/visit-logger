# Admin UI

Two panels and one line on the resource's admin page. They ship as structure
and states only: bare semantic elements, no classes, strings passed in. Put the
host's card, badge and list primitives on them, and its i18n behind the strings.

## Panel contract

| Panel | Shows | States |
|---|---|---|
| Visit history | summary line (visits · page views · first · last), then one row per sitting: via badge, bot badge, start time, location, client · timezone · page views · duration | failed, empty, list; "at least" when truncated |
| Visitor activity | one row per lifecycle event, newest first: kind label, bot badge, time, location, client · timezone | failed, empty, list; "more" note when truncated |
| Origin line | "Signed up from Kraków, 12, PL · Chrome 131 · Mac OS · desktop · Europe/Warsaw · <date>" under the page title | absent when there is no origin event |

Behaviour that must survive restyling:

- **Bot rows stay listed, de-emphasised and badged.** They are excluded from the
  counts, but an "open" three seconds after the email was sent needs its
  explanation on screen.
- **`internal` never appears.** The store filters it out before the summary is
  built. The `internal` label exists only so the strings type is total.
- **Failed is not empty.** `failed` renders an alert, not "Not opened yet".
- **Truncated is a lower bound.** The counts say "at least", and "first" is the
  real first visit, supplied by `readVisitSummary`.
- **Durations under a minute are omitted.** A one-page sitting has no length.

## Visit history

```tsx
// file: components/visits/VisitHistory.tsx
import {
  formatVisitClient,
  formatVisitLocation,
  type VisitSession,
  type VisitSummary,
  type VisitVia,
} from "@/lib/visits/types";

/**
 * Structure and states only. Swap the bare elements for the host's card,
 * badge and list primitives, and `strings` for its i18n lookup.
 */

export interface VisitHistoryStrings {
  title: string;
  empty: string;
  failed: string;
  unknownLocation: string;
  bot: string;
  via: Record<VisitVia, string>;
  visits: (count: number) => string;
  pageViews: (count: number) => string;
  /** Shown when `truncated`: the counts are a lower bound. */
  atLeast: (text: string) => string;
  first: (when: string) => string;
  last: (when: string) => string;
  duration: (minutes: number) => string;
}

export const VISIT_HISTORY_STRINGS_EN: VisitHistoryStrings = {
  title: "Visits",
  empty: "Not opened yet.",
  failed: "Could not load visits.",
  unknownLocation: "Unknown location",
  bot: "bot",
  via: { link: "link", session: "signed in", internal: "internal" },
  visits: (n) => `${n} ${n === 1 ? "visit" : "visits"}`,
  pageViews: (n) => `${n} ${n === 1 ? "page view" : "page views"}`,
  atLeast: (text) => `at least ${text}`,
  first: (when) => `first ${when}`,
  last: (when) => `last ${when}`,
  duration: (m) =>
    m < 60 ? `${m} min` : `${Math.floor(m / 60)} h${m % 60 ? ` ${m % 60} min` : ""}`,
};

/** Whole minutes; null for a sitting under a minute, which has no useful length. */
export function sessionMinutes(session: Pick<VisitSession, "startedAt" | "endedAt">): number | null {
  const ms = new Date(session.endedAt).getTime() - new Date(session.startedAt).getTime();
  const minutes = Math.round(ms / 60_000);
  return minutes >= 1 ? minutes : null;
}

export function VisitHistory({
  summary,
  strings = VISIT_HISTORY_STRINGS_EN,
  formatTimestamp,
}: {
  summary: VisitSummary;
  strings?: VisitHistoryStrings;
  formatTimestamp: (iso: string) => string;
}) {
  const { sessions, sessionCount, pageViewCount, firstAt, lastAt, truncated, failed } = summary;
  const bound = (text: string) => (truncated ? strings.atLeast(text) : text);

  return (
    <section aria-labelledby="visit-history-title">
      <h2 id="visit-history-title">{strings.title}</h2>

      {failed ? (
        <p role="alert">{strings.failed}</p>
      ) : sessions.length === 0 ? (
        <p>{strings.empty}</p>
      ) : (
        <>
          <p>
            {bound(strings.visits(sessionCount))} · {bound(strings.pageViews(pageViewCount))}
            {firstAt && <> · {strings.first(formatTimestamp(firstAt))}</>}
            {lastAt && lastAt !== firstAt && <> · {strings.last(formatTimestamp(lastAt))}</>}
          </p>

          <ul>
            {sessions.map((session) => {
              const minutes = sessionMinutes(session);
              return (
                // Bot sittings stay listed, de-emphasised, so an early "open" can be explained.
                <li key={session.id} data-bot={session.isBot || undefined}>
                  <span>{strings.via[session.via]}</span>
                  {session.isBot && <span>{strings.bot}</span>}
                  <time dateTime={session.startedAt}>{formatTimestamp(session.startedAt)}</time>
                  <span>{formatVisitLocation(session) ?? strings.unknownLocation}</span>
                  <small>
                    {[
                      formatVisitClient(session),
                      session.timezone,
                      strings.pageViews(session.pageViews),
                      minutes === null ? null : strings.duration(minutes),
                    ]
                      .filter(Boolean)
                      .join(" · ")}
                  </small>
                </li>
              );
            })}
          </ul>
        </>
      )}
    </section>
  );
}
```

## Visitor activity

```tsx
// file: components/visits/VisitorActivity.tsx
import {
  formatVisitClient,
  formatVisitLocation,
  type VisitorEventList,
} from "@/lib/visits/types";

/**
 * A subject's lifecycle events, newest first. `K` is the host's union of
 * event kinds; `kindLabel` maps each to its translated label.
 */
export interface VisitorActivityStrings<K extends string> {
  title: string;
  empty: string;
  failed: string;
  /** Shown under the list when `truncated`. */
  more: string;
  unknownLocation: string;
  bot: string;
  kindLabel: Record<K, string>;
}

export function VisitorActivity<K extends string>({
  list,
  strings,
  formatTimestamp,
}: {
  list: VisitorEventList<K>;
  strings: VisitorActivityStrings<K>;
  formatTimestamp: (iso: string) => string;
}) {
  return (
    <section aria-labelledby="visitor-activity-title">
      <h2 id="visitor-activity-title">{strings.title}</h2>

      {list.failed ? (
        <p role="alert">{strings.failed}</p>
      ) : list.events.length === 0 ? (
        <p>{strings.empty}</p>
      ) : (
        <ul>
          {list.events.map((event) => (
            <li key={event.id} data-bot={event.isBot || undefined}>
              <span>{strings.kindLabel[event.kind]}</span>
              {event.isBot && <span>{strings.bot}</span>}
              <time dateTime={event.createdAt}>{formatTimestamp(event.createdAt)}</time>
              <span>{formatVisitLocation(event) ?? strings.unknownLocation}</span>
              <small>
                {[formatVisitClient(event), event.timezone].filter(Boolean).join(" · ")}
              </small>
            </li>
          ))}
        </ul>
      )}
      {list.truncated && <p>{strings.more}</p>}
    </section>
  );
}
```

## Wiring on the admin page

Read everything in parallel on the server. Each read fails on its own and says
so, and none blocks the others. The origin line sits under the page title.

```tsx
// file: app/admin/proposals/[id]/page.tsx
import { notFound } from "next/navigation";

import { VisitHistory } from "@/components/visits/VisitHistory";
import { VisitorActivity, type VisitorActivityStrings } from "@/components/visits/VisitorActivity";
import { findOriginEvent, readVisitorEvents, readVisitSummary } from "@/lib/visits/record";
import { getVisitStore } from "@/lib/visits/store";
import { formatVisitClient, formatVisitLocation } from "@/lib/visits/types";

type Kind = "link_requested" | "account_created" | "signed_in" | "resource_created";

// Host seams: the admin guard and resource loader, and the house date format.
declare function getProposalForAdmin(id: string): Promise<{ id: string; title: string; ownerEmail: string } | null>;
declare function formatTimestamp(iso: string): string;

const ACTIVITY_STRINGS: VisitorActivityStrings<Kind> = {
  title: "Account activity",
  empty: "Nothing recorded yet.",
  failed: "Could not load activity.",
  more: "Older events are not shown.",
  unknownLocation: "Unknown location",
  bot: "bot",
  kindLabel: {
    link_requested: "sign-in requested",
    account_created: "account created",
    signed_in: "signed in",
    resource_created: "first proposal",
  },
};

export default async function AdminProposalPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const proposal = await getProposalForAdmin(id);
  if (!proposal) notFound();

  // Inside the admin, a missing store should fail loudly, unlike on the tracked page.
  const store = getVisitStore();
  // Each read fails on its own and says so; none blocks the others.
  const [visits, activity, origin] = await Promise.all([
    readVisitSummary(store, proposal.id),
    readVisitorEvents<Kind>(store, proposal.ownerEmail),
    findOriginEvent<Kind>(store, proposal.ownerEmail, ["link_requested", "account_created"]),
  ]);

  return (
    <main>
      <h1>{proposal.title}</h1>
      {origin && (
        <p>
          Signed up from{" "}
          {[
            formatVisitLocation(origin) ?? "unknown location",
            formatVisitClient(origin),
            origin.timezone,
            formatTimestamp(origin.createdAt),
          ]
            .filter(Boolean)
            .join(" · ")}
        </p>
      )}
      <VisitHistory summary={visits} formatTimestamp={formatTimestamp} />
      <VisitorActivity list={activity} strings={ACTIVITY_STRINGS} formatTimestamp={formatTimestamp} />
    </main>
  );
}
```

## Extensions worth building (not shipped)

These are gaps an operator will hit. None is in the source:

| Gap | Design |
|---|---|
| Which pages did they read? | `path` is stored per page view. Expand a sitting to list its paths in order: a second query by `resource_id` and the sitting's time range |
| Where did they click the link? | the sitting carries the entry `referrer`; show its host when present |
| Export for a CRM | a CSV of sittings, one row each, from `readVisitSummary` with a higher limit |
| "Last seen" in the resource list | `max(created_at)` over human, non-internal visits, joined into the list query; one indexed read per page of resources |

## Admin UI checklist

- [ ] Host primitives and semantic tokens; no hardcoded colours
- [ ] Strings from the host's i18n, plurals included
- [ ] Bot rows visible and badged; internal visits absent
- [ ] Failed, empty and truncated states all reachable and distinct
