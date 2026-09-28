# Testing

Three suites, 56 tests. They cover the claims the skill makes: the rules in
[rules.md](rules.md), the fingerprint and page-view contract in
[fingerprint.md](fingerprint.md), and the failure contract in
[recording.md](recording.md), against the memory store from
[stores.md](stores.md).

All three import from `vitest`. `bun test` rewrites that import to its own
runner, so they run unchanged under either:

```bash
npm i -D vitest                # the registry is not an external service
npx vitest run lib/visits      # vitest
bun test lib/visits            # bun
```

Wire one of them to `npm test` (`"test": "vitest run"`) and copy the three
files unchanged. Never convert them to `node:test` or another runner, and never
edit an assertion to make it pass: a converted suite is new code that proves
nothing about the templates. Tests of the host's own code go in files beside
them.

Verified under vitest 5 and bun 1.2 (56 passing in each), and type-checked
with TypeScript 6 under `strict` and `noUncheckedIndexedAccess` together with
every other code block in the skill. `fingerprint.test.ts` needs `next/server`,
which any Next.js app has.

| Suite | Proves |
|---|---|
| `core.test.ts` | sittings join and split on the gap from the last view; visitors interleave; input not mutated; bot taint; entry referrer; counts exclude bots and staff; truncated `firstAt`; the announcement matrix; a failed read never headlined as a return; origin preference and fallbacks |
| `fingerprint.test.ts` | Vercel IP and decoded city; malformed escape survives; major-only client line; every automated client flagged and no real browser flagged (CUBOT included); Cloudflare Latin-1 repair; `noEdge`; the full `isPageView` contract |
| `record.test.ts` | first, then quiet, then back; NULL IP matches NULL IP; bots and staff stored, never announced, never "prior"; read failure announces; write failure is quiet; truncation with the real first visit; failed reads say so; subject normalisation both ways |

## Rules

```ts
// file: lib/visits/core.test.ts
import { describe, expect, it } from "vitest";

import { openedHeadline } from "./announce";
import {
  ANNOUNCE_UNKNOWN,
  assessVisit,
  groupVisitsIntoSessions,
  pickOriginEvent,
  SESSION_WINDOW_MS,
  summarizeVisits,
} from "./core";
import type { PageVisit, VisitFingerprint, VisitorEvent } from "./types";

const T0 = Date.parse("2026-09-01T10:00:00.000Z");
const at = (minutes: number) => new Date(T0 + minutes * 60_000).toISOString();

const FINGERPRINT: VisitFingerprint = {
  ip: "203.0.113.7",
  country: "PL",
  region: "12",
  city: "Kraków",
  timezone: "Europe/Warsaw",
  browser: "Chrome",
  browserVersion: "131.0.0.0",
  os: "Mac OS",
  deviceType: "desktop",
  userAgent: "UA-A",
  referrer: null,
  isBot: false,
};

let sequence = 0;
function visit(minutes: number, overrides: Partial<PageVisit> = {}): PageVisit {
  sequence += 1;
  return {
    ...FINGERPRINT,
    id: `v${sequence}`,
    resourceId: "r1",
    resourceKey: null,
    subject: "ada@example.com",
    via: "link",
    path: "/",
    createdAt: at(minutes),
    ...overrides,
  };
}

describe("groupVisitsIntoSessions", () => {
  it("joins page views within the window and splits after a longer gap", () => {
    const sessions = groupVisitsIntoSessions([visit(0), visit(10), visit(39), visit(70)]);
    expect(sessions.map((s) => s.pageViews)).toEqual([1, 3]);
    expect(sessions[1]?.startedAt).toBe(at(0));
    expect(sessions[1]?.endedAt).toBe(at(39));
  });

  it("measures the gap from the last page view, not the first", () => {
    // 0, 25, 50: every gap is 25 min, the span is 50. One sitting.
    expect(groupVisitsIntoSessions([visit(0), visit(25), visit(50)])).toHaveLength(1);
  });

  it("keeps interleaved visitors apart", () => {
    const sessions = groupVisitsIntoSessions([
      visit(0),
      visit(1, { userAgent: "UA-B" }),
      visit(2),
      visit(3, { ip: null }),
    ]);
    expect(sessions).toHaveLength(3);
    expect(sessions.find((s) => s.userAgent === "UA-A" && s.ip)?.pageViews).toBe(2);
  });

  it("returns newest first regardless of input order, without mutating the input", () => {
    const input = [visit(100), visit(0), visit(50)];
    const before = input.map((v) => v.id);
    const sessions = groupVisitsIntoSessions(input);
    expect(sessions.map((s) => s.startedAt)).toEqual([at(100), at(50), at(0)]);
    expect(input.map((v) => v.id)).toEqual(before);
  });

  it("lets one bot hit taint the whole sitting", () => {
    const [session] = groupVisitsIntoSessions([visit(0), visit(1, { isBot: true })]);
    expect(session?.isBot).toBe(true);
  });

  it("keeps the entry referrer", () => {
    const [session] = groupVisitsIntoSessions([
      visit(0, { referrer: "https://www.linkedin.com/" }),
      visit(1, { referrer: "https://app.example.com/r1" }),
    ]);
    expect(session?.referrer).toBe("https://www.linkedin.com/");
  });
});

describe("summarizeVisits", () => {
  it("counts people, not bots or internal previews", () => {
    const summary = summarizeVisits(
      [visit(0), visit(1), visit(90, { isBot: true, ip: "198.51.100.1" }), visit(200, { via: "internal" })],
      { truncated: false },
    );
    expect(summary.sessions).toHaveLength(2);
    expect(summary.sessionCount).toBe(1);
    expect(summary.pageViewCount).toBe(2);
    expect(summary.firstAt).toBe(at(0));
    expect(summary.lastAt).toBe(at(1));
  });

  it("takes firstAt from the caller when the read was truncated", () => {
    const summary = summarizeVisits([visit(500), visit(501)], {
      truncated: true,
      firstAt: at(0),
    });
    expect(summary.firstAt).toBe(at(0));
    expect(summary.truncated).toBe(true);
  });
});

describe("assessVisit", () => {
  const now = T0 + 60 * 60_000;

  it("announces the first visit ever", () => {
    expect(assessVisit({ lastHumanAt: null, lastSameVisitorAt: null }, now)).toEqual({
      isFirstVisit: true,
      isNewVisitor: true,
      shouldAnnounce: true,
      noveltyKnown: true,
    });
  });

  it("announces a second person inside someone else's window", () => {
    const result = assessVisit({ lastHumanAt: at(59), lastSameVisitorAt: null }, now);
    expect(result).toEqual({ isFirstVisit: false, isNewVisitor: true, shouldAnnounce: true, noveltyKnown: true });
  });

  it("stays quiet for the same visitor inside the window", () => {
    const result = assessVisit({ lastHumanAt: at(59), lastSameVisitorAt: at(59) }, now);
    expect(result.shouldAnnounce).toBe(false);
  });

  it("announces the same visitor again after the window", () => {
    const lastSeen = new Date(now - SESSION_WINDOW_MS - 1).toISOString();
    const result = assessVisit({ lastHumanAt: lastSeen, lastSameVisitorAt: lastSeen }, now);
    expect(result).toEqual({ isFirstVisit: false, isNewVisitor: false, shouldAnnounce: true, noveltyKnown: true });
  });

  it("does not headline a failed read as a return", () => {
    const lastSeen = new Date(now - SESSION_WINDOW_MS - 1).toISOString();
    const back = assessVisit({ lastHumanAt: lastSeen, lastSameVisitorAt: lastSeen }, now);
    expect(openedHeadline("ada@example.com", back)).toBe("ada@example.com came back to it");
    expect(openedHeadline("ada@example.com", ANNOUNCE_UNKNOWN)).toBe(
      "ada@example.com opened it; earlier visits could not be checked",
    );
  });
});

describe("pickOriginEvent", () => {
  type Kind = "link_requested" | "account_created" | "signed_in";
  const event = (minutes: number, kind: Kind, isBot = false): VisitorEvent<Kind> => ({
    ...FINGERPRINT,
    id: `${kind}-${minutes}`,
    subject: "ada@example.com",
    kind,
    resourceId: null,
    createdAt: at(minutes),
    isBot,
  });
  const preference: Kind[] = ["link_requested", "account_created"];

  it("prefers the form submission over an earlier link click", () => {
    const picked = pickOriginEvent(
      [event(5, "account_created"), event(0, "signed_in"), event(3, "link_requested")],
      preference,
    );
    expect(picked?.id).toBe("link_requested-3");
  });

  it("skips a scanner's click and falls back down the preference list", () => {
    const picked = pickOriginEvent(
      [event(1, "account_created", true), event(2, "account_created"), event(9, "signed_in")],
      preference,
    );
    expect(picked?.id).toBe("account_created-2");
  });

  it("falls back to the earliest human event, then to anything", () => {
    expect(pickOriginEvent([event(9, "signed_in"), event(4, "signed_in")], preference)?.id).toBe(
      "signed_in-4",
    );
    expect(pickOriginEvent([event(1, "signed_in", true)], preference)?.id).toBe("signed_in-1");
    expect(pickOriginEvent([], preference)).toBeNull();
  });
});
```

## Fingerprint

```ts
// file: lib/visits/fingerprint.test.ts
import { describe, expect, it } from "vitest";

import { cloudflareEdge, describeVisitor, isPageView, noEdge, utf8FromLatin1 } from "./fingerprint";
import { formatVisitClient, formatVisitLocation, FRAGMENT_SEPARATOR } from "./types";

const CHROME_MAC =
  "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36";
const SAFARI_IPHONE =
  "Mozilla/5.0 (iPhone; CPU iPhone OS 18_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Version/18.1 Mobile/15E148 Safari/604.1";
const HEADLESS =
  "Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) HeadlessChrome/120.0.0.0 Safari/537.36";
const CUBOT =
  "Mozilla/5.0 (Linux; Android 11; CUBOT X30) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Mobile Safari/537.36";
const INSTAGRAM =
  "Mozilla/5.0 (iPhone; CPU iPhone OS 17_5 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) Mobile/15E148 Instagram 339.0.3.12.108 (iPhone13,2; iOS 17_5; en_US; en; scale=3.00; 1170x2532; 618569201)";

const headers = (init: Record<string, string>) => new Headers(init);

describe("describeVisitor on Vercel", () => {
  const fingerprint = describeVisitor(
    headers({
      "user-agent": CHROME_MAC,
      "x-forwarded-for": "203.0.113.7, 10.0.0.1",
      "x-vercel-ip-country": "PL",
      "x-vercel-ip-country-region": "12",
      "x-vercel-ip-city": "Krak%C3%B3w",
      "x-vercel-ip-timezone": "Europe/Warsaw",
      referer: "https://mail.example.com/",
    }),
  );

  it("takes the left-most forwarded address and decodes the city", () => {
    expect(fingerprint.ip).toBe("203.0.113.7");
    expect(fingerprint.city).toBe("Kraków");
    expect(formatVisitLocation(fingerprint)).toBe("Kraków, 12, PL");
  });

  it("parses the browser and formats the major version only", () => {
    expect(fingerprint).toMatchObject({ browser: "Chrome", os: "Mac OS", deviceType: "desktop", isBot: false });
    expect(formatVisitClient(fingerprint)).toBe(
      ["Chrome 131", "Mac OS", "desktop"].join(FRAGMENT_SEPARATOR),
    );
  });

  it("falls back to x-real-ip and survives a malformed escape", () => {
    const fp = describeVisitor(
      headers({ "user-agent": CHROME_MAC, "x-real-ip": "198.51.100.2", "x-vercel-ip-city": "%E0%A4%A" }),
    );
    expect(fp.ip).toBe("198.51.100.2");
    expect(fp.city).toBe("%E0%A4%A");
  });
});

describe("bot detection", () => {
  const botFor = (ua: string | null) =>
    describeVisitor(headers(ua === null ? {} : { "user-agent": ua })).isBot;

  it.each([
    ["no user agent", null],
    ["headless Chrome", HEADLESS],
    ["curl", "curl/8.4.0"],
    ["python-requests", "python-requests/2.31.0"],
    ["Googlebot", "Mozilla/5.0 (compatible; Googlebot/2.1; +http://www.google.com/bot.html)"],
    ["Slack unfurl", "Slackbot-LinkExpanding 1.0 (+https://api.slack.com/robots)"],
    ["uptime monitor", "Mozilla/5.0+(compatible; UptimeRobot/2.0; http://www.uptimerobot.com/)"],
  ])("flags %s", (_label, ua) => {
    expect(botFor(ua)).toBe(true);
  });

  it.each([
    ["Chrome on macOS", CHROME_MAC],
    ["Safari on iPhone", SAFARI_IPHONE],
    ["a CUBOT phone", CUBOT],
    ["the Instagram in-app browser", INSTAGRAM],
  ])("does not flag %s", (_label, ua) => {
    expect(botFor(ua)).toBe(false);
  });

  it("does not guess a device type for automation", () => {
    expect(describeVisitor(headers({ "user-agent": "python-requests/2.31.0" })).deviceType).toBeNull();
    expect(describeVisitor(headers({ "user-agent": SAFARI_IPHONE })).deviceType).toBe("mobile");
  });
});

describe("other edges", () => {
  it("reads Cloudflare's headers and repairs UTF-8 read as Latin-1", () => {
    const mangled = String.fromCharCode(...new TextEncoder().encode("Kraków"));
    const fp = describeVisitor(
      headers({ "user-agent": CHROME_MAC, "cf-connecting-ip": "203.0.113.9", "cf-ipcountry": "PL", "cf-ipcity": mangled }),
      cloudflareEdge,
    );
    expect(fp.ip).toBe("203.0.113.9");
    expect(fp.city).toBe("Kraków");
  });

  it("leaves real Latin-1 and real Unicode alone", () => {
    expect(utf8FromLatin1("Zürich")).toBe("Zürich");
    expect(utf8FromLatin1("Łódź")).toBe("Łódź");
  });

  it("records no address when there is no trusted edge", () => {
    const fp = describeVisitor(headers({ "user-agent": CHROME_MAC, "x-forwarded-for": "6.6.6.6" }), noEdge);
    expect(fp.ip).toBeNull();
  });
});

describe("isPageView", () => {
  it.each([
    ["a document navigation", { "sec-fetch-dest": "document" }, true],
    ["a client without Sec-Fetch headers", {}, true],
    ["an App Router client navigation", { rsc: "1", "sec-fetch-dest": "empty" }, true],
    ["an iframe", { "sec-fetch-dest": "iframe" }, false],
    ["a script", { "sec-fetch-dest": "script" }, false],
    ["a Chrome prerender", { "sec-fetch-dest": "document", "sec-purpose": "prefetch;prerender" }, false],
    ["a legacy prefetch", { purpose: "prefetch" }, false],
    ["a mail-client preview", { "x-purpose": "preview" }, false],
    ["a <Link> prefetch", { rsc: "1", "next-router-prefetch": "1" }, false],
    ["a segment prefetch", { rsc: "1", "next-router-segment-prefetch": "/_tree" }, false],
    ["a Server Action", { "next-action": "abc123", "sec-fetch-dest": "empty" }, false],
    ["a client navigation as a page's headers() shows it", { "sec-fetch-dest": "empty" }, false],
  ])("%s is a page view: %s", (_label, init, expected) => {
    expect(isPageView(headers(init))).toBe(expected);
  });
});
```

## Write and read paths

```ts
// file: lib/visits/record.test.ts
import { afterAll, beforeAll, describe, expect, it } from "vitest";

import { createMemoryVisitStore } from "./memory-store";
import {
  findOriginEvent,
  type NewPageVisit,
  readVisitorEvents,
  readVisitSummary,
  recordPageVisit,
  recordVisitorEvent,
} from "./record";
import type { VisitFingerprint } from "./types";

const FINGERPRINT: VisitFingerprint = {
  ip: "203.0.113.7",
  country: "PL",
  region: "12",
  city: "Kraków",
  timezone: "Europe/Warsaw",
  browser: "Chrome",
  browserVersion: "131.0.0.0",
  os: "Mac OS",
  deviceType: "desktop",
  userAgent: "UA-A",
  referrer: null,
  isBot: false,
};

const VISIT: NewPageVisit = {
  ...FINGERPRINT,
  resourceId: "r1",
  resourceKey: "demo-a",
  subject: "Ada@Example.com ",
  via: "link",
  path: "/",
};

/** A clock the test moves by hand. */
function manualClock(start = Date.parse("2026-09-01T10:00:00Z")) {
  let now = start;
  return { now: () => now, advance: (minutes: number) => (now += minutes * 60_000) };
}

// Failure paths log on purpose; keep the output readable.
const consoleError = console.error;
beforeAll(() => {
  console.error = () => {};
});
afterAll(() => {
  console.error = consoleError;
});

describe("recordPageVisit", () => {
  it("announces the first visit, then stays quiet inside the window", async () => {
    const clock = manualClock();
    const { store, visits } = createMemoryVisitStore({ clock: clock.now });

    const first = await recordPageVisit(store, VISIT, { now: clock.now() });
    clock.advance(5);
    const second = await recordPageVisit(store, VISIT, { now: clock.now() });
    clock.advance(31);
    const back = await recordPageVisit(store, VISIT, { now: clock.now() });

    expect(first).toEqual({ isFirstVisit: true, isNewVisitor: true, shouldAnnounce: true, noveltyKnown: true });
    expect(second.shouldAnnounce).toBe(false);
    expect(back).toEqual({ isFirstVisit: false, isNewVisitor: false, shouldAnnounce: true, noveltyKnown: true });
    expect(visits).toHaveLength(3);
    expect(visits[0]?.subject).toBe("ada@example.com");
  });

  it("matches a visitor with no IP against earlier rows with no IP", async () => {
    const { store } = createMemoryVisitStore();
    const anonymous = { ...VISIT, ip: null };
    await recordPageVisit(store, anonymous);
    expect((await recordPageVisit(store, anonymous)).shouldAnnounce).toBe(false);
  });

  it("stores bots and internal previews without ever announcing them", async () => {
    const { store, visits } = createMemoryVisitStore({ fail: ["readPriorVisits"] });
    const bot = await recordPageVisit(store, { ...VISIT, isBot: true });
    const internal = await recordPageVisit(store, { ...VISIT, via: "internal" });
    expect(bot.shouldAnnounce || internal.shouldAnnounce).toBe(false);
    expect(visits).toHaveLength(2);
  });

  it("does not let a bot or an internal preview make the customer's visit look old", async () => {
    const { store } = createMemoryVisitStore();
    await recordPageVisit(store, { ...VISIT, isBot: true });
    await recordPageVisit(store, { ...VISIT, via: "internal" });
    expect((await recordPageVisit(store, VISIT)).isFirstVisit).toBe(true);
  });

  it("announces when the prior-visit read fails, and still writes the row", async () => {
    const { store, visits } = createMemoryVisitStore({ fail: ["readPriorVisits"] });
    const result = await recordPageVisit(store, VISIT);
    expect(result).toEqual({ isFirstVisit: false, isNewVisitor: false, shouldAnnounce: true, noveltyKnown: false });
    expect(visits).toHaveLength(1);
  });

  it("stays quiet when the write fails, and never throws", async () => {
    const { store } = createMemoryVisitStore({ fail: ["insertPageVisit"] });
    expect((await recordPageVisit(store, VISIT)).shouldAnnounce).toBe(false);
  });
});

describe("readVisitSummary", () => {
  it("flags truncation and takes the real first visit from its own query", async () => {
    const clock = manualClock();
    const { store } = createMemoryVisitStore({ clock: clock.now });
    for (let i = 0; i < 5; i += 1) {
      await recordPageVisit(store, VISIT);
      clock.advance(60);
    }

    const summary = await readVisitSummary(store, "r1", 3);
    expect(summary.truncated).toBe(true);
    expect(summary.pageViewCount).toBe(3);
    expect(summary.firstAt).toBe("2026-09-01T10:00:00.000Z");

    const full = await readVisitSummary(store, "r1", 10);
    expect(full).toMatchObject({ truncated: false, sessionCount: 5, pageViewCount: 5 });
  });

  it("says it failed instead of saying nobody came", async () => {
    const { store } = createMemoryVisitStore({ fail: ["listPageVisits"] });
    expect(await readVisitSummary(store, "r1")).toMatchObject({ failed: true, sessions: [] });
  });
});

describe("visitor events", () => {
  it("normalises the subject on write and on read, and finds the origin", async () => {
    const clock = manualClock();
    const { store } = createMemoryVisitStore({ clock: clock.now });
    const base = { ...FINGERPRINT, resourceId: null };

    await recordVisitorEvent(store, { ...base, subject: "ADA@example.com", kind: "signed_in" });
    clock.advance(1);
    await recordVisitorEvent(store, { ...base, subject: "ada@example.com", kind: "link_requested", city: "Berlin" });
    clock.advance(1);
    await recordVisitorEvent(store, { ...base, subject: "ada@example.com", kind: "account_created", isBot: true });

    const list = await readVisitorEvents(store, " Ada@Example.com", 2);
    expect(list.events.map((e) => e.kind)).toEqual(["account_created", "link_requested"]);
    expect(list.truncated).toBe(true);

    const origin = await findOriginEvent(store, "ada@example.com", ["link_requested", "account_created"]);
    expect(origin?.city).toBe("Berlin");
  });

  it("never throws from a failed write or read", async () => {
    const { store } = createMemoryVisitStore({ fail: ["insertVisitorEvent", "listVisitorEvents"] });
    await recordVisitorEvent(store, { ...FINGERPRINT, subject: "a@b.c", kind: "signed_in", resourceId: null });
    expect(await readVisitorEvents(store, "a@b.c")).toMatchObject({ failed: true, events: [] });
    expect(await findOriginEvent(store, "a@b.c", ["signed_in"])).toBeNull();
  });
});
```

## What is not tested here

- **The Supabase and Firestore stores against a live database.** The Supabase
  queries are the earlier implementation's, renamed. The migration they
  run against was verified on PostgreSQL 18 ([data-model.md](data-model.md)).
  The Firestore store has never run.
- **`after()` scheduling.** `trackPageVisit` and `trackVisitorEvent` are thin;
  exercise them end to end in the host by opening a resource and checking the
  row and the notification.
- **App Router RSC header behaviour.** `isPageView` is tested against header
  sets, not against a running router.

## Testing checklist

- [ ] All three suites copied unchanged, wired to `npm test`, 56 passing under vitest or bun
- [ ] A new `looksAutomated` pattern arrives with a test row for it, and a real-browser row it must not match
- [ ] One end-to-end open in the running host: row written, bot flag right, announcement received once
