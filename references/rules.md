# Rules

The decisions the module makes, as pure functions: what one sitting is, when a
visit deserves an announcement, and which event says where someone came from.
No database and no clock of their own, so every rule here is covered by
`core.test.ts` ([testing.md](testing.md)).

## Who is "the same visitor"

Same IP **and** same user agent. It is a heuristic, and its failure modes are
worth knowing before someone asks why the history looks the way it does:

| Situation | What the log shows | Acceptable because |
|---|---|---|
| One person, phone on mobile data, IP changes | a new visitor mid-sitting | one extra row; "new device or network" is literally true |
| Two colleagues, same office NAT, same browser build | one visitor | rare in practice, and still attributed to the right company |
| A browser update | a new visitor | the update is visible in the client line |
| No IP (local, `noEdge`) and no user agent | all such rows are one visitor | only scripts send no user agent, and they are bots anyway |

Do not add a cookie to sharpen it. A first-party visitor cookie turns a
server-side log with no client storage into one that needs consent in most of
the EU. Adding one is a legal decision, not a technical one.

## The session window

`SESSION_WINDOW_MS` is 30 minutes and is measured **from the last page view,
not the first**. Page views at 0, 25 and 50 minutes are one sitting; so is
someone reading a long proposal for two hours with a click every twenty
minutes.

The same window drives both the grouping in the history panel and the
announcement throttle, so what the panel calls one visit is what notified
once.

## When to announce

`assessVisit` answers from two facts read *before* the new row is written:

| Anyone human before? | This visitor before? | Last seen by this visitor | `isFirstVisit` | `isNewVisitor` | `shouldAnnounce` |
|---|---|---|---|---|---|
| no | no | n/a | true | true | **yes**: first open ever |
| yes | no | n/a | false | true | **yes**: someone new, a forward or a colleague |
| yes | yes | within the window | false | false | no: same sitting |
| yes | yes | beyond the window | false | false | **yes**: they came back |
| read failed | | | false | false | **yes**: losing the throttle beats losing the ping |

The window is **per visitor, not per resource**. Two different people opening
the same proposal within five minutes are two announcements, and the second
one is usually the more interesting (the champion forwarded it).

Bots and `internal` visits are excluded from both facts by the store, and are
never assessed themselves. That exclusion is what stops a staff member's
preview five minutes before the customer from turning the customer's first
open into "came back".

## The origin event

`pickOriginEvent` answers "where did this person sign up from" from their
lifecycle events. Pass the kinds in order of trust:

1. **The sign-in form submission** (`link_requested`). The one request that was
   certainly made by the person's own browser. It is unauthenticated, so anyone
   can submit someone else's address; for an internal "signed up from" line
   that risk is accepted, and it is the reason a bot flag is not enough.
2. **The first verified sign-in** (`account_created`). A magic-link click, which
   a mail scanner may have made first; the scanner's click is skipped when it
   is flagged `isBot`.
3. Otherwise the earliest human event, then the earliest event of any kind, so
   that an old record still gets a line.

## The code

```ts
// file: lib/visits/core.ts
import type {
  PageVisit,
  VisitFingerprint,
  VisitorEvent,
  VisitSession,
  VisitSummary,
} from "./types";

/**
 * The rules of the module, as pure functions: what one sitting is, when a
 * visit is worth announcing, and which lifecycle event says where someone
 * came from. No database, no clock of their own, so all of it is testable.
 */

/**
 * Page views by the same visitor closer together than this are one sitting:
 * one row in the history panel, and at most one announcement. Every page view
 * is still stored.
 */
export const SESSION_WINDOW_MS = 30 * 60 * 1000;

const time = (iso: string): number => new Date(iso).getTime();

/** Same IP and same user agent is "the same visitor". Nulls match nulls. */
export function visitorKey(visit: Pick<VisitFingerprint, "ip" | "userAgent">): string {
  return `${visit.ip ?? ""}|${visit.userAgent ?? ""}`;
}

/**
 * Collapses page views into sittings, newest first: same visitor key, and no
 * gap longer than `gapMs` between one page view and the next. Does not mutate
 * its input and does not trust its order.
 */
export function groupVisitsIntoSessions(
  visits: readonly PageVisit[],
  gapMs: number = SESSION_WINDOW_MS,
): VisitSession[] {
  const ascending = [...visits].sort((a, b) => time(a.createdAt) - time(b.createdAt));
  const sessions: VisitSession[] = [];
  const open = new Map<string, VisitSession>();

  for (const visit of ascending) {
    const key = visitorKey(visit);
    const current = open.get(key);

    if (current && time(visit.createdAt) - time(current.endedAt) <= gapMs) {
      current.endedAt = visit.createdAt;
      current.pageViews += 1;
      // One bot hit taints the sitting: it was never a person after all.
      current.isBot = current.isBot || visit.isBot;
      continue;
    }

    const { resourceId: _r, resourceKey: _k, path: _p, createdAt, ...rest } = visit;
    const session: VisitSession = { ...rest, startedAt: createdAt, endedAt: createdAt, pageViews: 1 };
    sessions.push(session);
    open.set(key, session);
  }

  return sessions.sort((a, b) => time(b.startedAt) - time(a.startedAt));
}

export const EMPTY_SUMMARY: VisitSummary = {
  sessions: [],
  sessionCount: 0,
  pageViewCount: 0,
  firstAt: null,
  lastAt: null,
  truncated: false,
  failed: false,
};

/**
 * Turns a bounded, newest-first read into what the history panel shows.
 *
 * When the read was truncated, the earliest row in hand is not the first
 * visit, so the caller passes the real one from a separate query. Without it
 * a busy resource reports a "first opened" date that is merely the oldest row
 * that fitted under the limit.
 */
export function summarizeVisits(
  visits: readonly PageVisit[],
  options: { truncated: boolean; firstAt?: string | null },
): VisitSummary {
  const sessions = groupVisitsIntoSessions(visits.filter((visit) => visit.via !== "internal"));
  const human = sessions.filter((session) => !session.isBot);

  return {
    sessions,
    sessionCount: human.length,
    pageViewCount: human.reduce((sum, session) => sum + session.pageViews, 0),
    firstAt: options.truncated
      ? (options.firstAt ?? null)
      : (human.at(-1)?.startedAt ?? null),
    lastAt: human[0]?.endedAt ?? null,
    truncated: options.truncated,
    failed: false,
  };
}

/** The two facts the announcement rule needs, read before the new row is written. */
export interface PriorVisits {
  /** Latest human, non-internal page view of this resource by anyone. */
  lastHumanAt: string | null;
  /** Latest human, non-internal page view of this resource by this visitor key. */
  lastSameVisitorAt: string | null;
}

export interface VisitAssessment {
  /** Nobody had opened this resource before now. */
  isFirstVisit: boolean;
  /** This IP + browser had never opened it before. */
  isNewVisitor: boolean;
  /** First ever, a new visitor, or a returning one after a break. */
  shouldAnnounce: boolean;
}

export const QUIET: VisitAssessment = {
  isFirstVisit: false,
  isNewVisitor: false,
  shouldAnnounce: false,
};

/**
 * When the prior-visit read fails. Losing the throttle beats losing the
 * notification, so announce, and claim nothing about novelty.
 */
export const ANNOUNCE_UNKNOWN: VisitAssessment = {
  isFirstVisit: false,
  isNewVisitor: false,
  shouldAnnounce: true,
};

/**
 * The quiet window is per visitor, not per resource: two different people
 * opening the same thing within half an hour are two announcements.
 */
export function assessVisit(
  prior: PriorVisits,
  now: number,
  windowMs: number = SESSION_WINDOW_MS,
): VisitAssessment {
  const isFirstVisit = prior.lastHumanAt === null;
  const isNewVisitor = prior.lastSameVisitorAt === null;
  const elapsed =
    prior.lastSameVisitorAt === null
      ? Number.POSITIVE_INFINITY
      : now - time(prior.lastSameVisitorAt);

  return {
    isFirstVisit,
    isNewVisitor,
    shouldAnnounce: isFirstVisit || isNewVisitor || elapsed > windowMs,
  };
}

/**
 * Which event says where someone came from.
 *
 * Walks `preference` in order and returns the earliest human event of the
 * first kind that has one; then the earliest human event of any kind; then
 * the earliest event at all, so an old record still gets a line. Put the kind
 * that is certainly the person's own browser first (the sign-in form
 * submission), ahead of a link click a mail scanner may have made.
 */
export function pickOriginEvent<K extends string>(
  events: readonly VisitorEvent<K>[],
  preference: readonly K[],
): VisitorEvent<K> | null {
  const ascending = [...events].sort((a, b) => time(a.createdAt) - time(b.createdAt));
  const human = ascending.filter((event) => !event.isBot);

  for (const kind of preference) {
    const match = human.find((event) => event.kind === kind);
    if (match) return match;
  }
  return human[0] ?? ascending[0] ?? null;
}
```

## Summaries

`summarizeVisits` counts people: bot sittings are listed but not counted, and
`internal` visits are neither listed nor counted. It takes the result of a
bounded, newest-first read, so it needs to be told two things:

- `truncated`: more rows exist than were read. The panel then says "at least".
- `firstAt`: when truncated, the real first human visit, from its own query.
  Without it, a resource with more page views than the limit reports a "first
  opened" date that is merely the oldest row that fitted. The source did exactly
  that, with no signal on screen.

`readVisitSummary` in [recording.md](recording.md) does both: it reads
`limit + 1` rows to learn whether there are more, and pays for the first-visit
query only when there are.

## Rules checklist

- [ ] The same window used for grouping and for the throttle
- [ ] The throttle keyed per visitor, not per resource
- [ ] Bots and `internal` excluded from prior-visit facts, not just from announcements
- [ ] Origin preference lists the form submission first
- [ ] A truncated summary carries the real first visit and says "at least"
