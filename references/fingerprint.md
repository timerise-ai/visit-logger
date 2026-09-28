# Fingerprint

What one request can tell you about who sent it, and the traps in reading it.
Two files: the pure types and formatters, which the admin UI may import, and
the server-only reader.

## The shape

| Field | Source | Notes |
|---|---|---|
| `ip` | the edge adapter | personal data; see [operations.md](operations.md) |
| `country` | edge geo header | ISO 3166-1 alpha-2 |
| `region` | edge geo header | ISO 3166-2 **code**: `CA`, `ENG`, `12`. Not a name |
| `city` | edge geo header | percent-encoded on Vercel, raw UTF-8 on Cloudflare |
| `timezone` | edge geo header | IANA name; the best "when to call them" hint there is |
| `browser`, `browserVersion`, `os`, `deviceType` | `userAgent()` | version stored in full, displayed as the major |
| `userAgent` | `user-agent` | stored verbatim (capped), so a later parser can re-read it |
| `referrer` | `referer` | origin only for most cross-site navigations, by browser default |
| `isBot` | `userAgent().isBot` + `looksAutomated` | labels, never blocks |

Every field is `null` when unknown, never `undefined`: Firestore rejects
`undefined`, and SQL has no such thing.

## Types and formatters

```ts
// file: lib/visits/types.ts
/**
 * Visit logging: shapes and formatters.
 *
 * Pure, with no server imports, so an admin component can format a location or
 * a browser line without pulling `next/server` into the client bundle.
 */

/** Who is on the other end of one request. Every field is null when unknown. */
export interface VisitFingerprint {
  ip: string | null;
  /** ISO 3166-1 alpha-2, e.g. "PL". */
  country: string | null;
  /** ISO 3166-2 subdivision code ("CA", "ENG"), not a name. */
  region: string | null;
  city: string | null;
  /** IANA name, e.g. "Europe/Warsaw". */
  timezone: string | null;
  browser: string | null;
  /** Full version as parsed, e.g. "131.0.0.0". Format with the major only. */
  browserVersion: string | null;
  os: string | null;
  /** "mobile", "tablet", "desktop", ...; null for automated clients. */
  deviceType: string | null;
  userAgent: string | null;
  referrer: string | null;
  /** Crawler, link preview, script, headless browser. Stored, never counted. */
  isBot: boolean;
}

/**
 * How the visitor got in.
 *
 * `internal` is your own staff looking at the resource. It is stored for audit
 * but never counted, listed or announced, so QA cannot look like engagement.
 */
export type VisitVia = "link" | "session" | "internal";

/** One page view of one resource. */
export interface PageVisit extends VisitFingerprint {
  id: string;
  /** What was visited; history and announcements group by this. */
  resourceId: string;
  /** A label for the resource as it was at visit time (it can be renamed later). */
  resourceKey: string | null;
  /** Who, when known: a normalised email or a user id. */
  subject: string | null;
  via: VisitVia;
  path: string | null;
  createdAt: string;
}

/**
 * One sitting: consecutive page views by the same visitor (IP + user agent)
 * with no gap longer than the session window. A reload, or a click through
 * three screens, is one visit rather than three.
 */
export interface VisitSession extends VisitFingerprint {
  /** Id of the first page view in the sitting. */
  id: string;
  subject: string | null;
  via: VisitVia;
  startedAt: string;
  endedAt: string;
  pageViews: number;
}

export interface VisitSummary {
  /** Sittings, newest first. `internal` visits are never included. */
  sessions: VisitSession[];
  /** Sittings by people, bots excluded. */
  sessionCount: number;
  /** Page views by people, bots excluded. */
  pageViewCount: number;
  /** The first human page view ever, even when the list was truncated. */
  firstAt: string | null;
  lastAt: string | null;
  /** More page views exist than were read; counts are a lower bound. */
  truncated: boolean;
  /** The read failed. Render "could not load", never "not opened yet". */
  failed: boolean;
}

/**
 * A moment in someone's lifecycle worth a fingerprint: asking for a sign-in
 * link, the first sign-in, a later sign-in, creating their first resource.
 * `K` is the host's own union of kinds.
 */
export interface VisitorEvent<K extends string = string> extends VisitFingerprint {
  id: string;
  subject: string;
  kind: K;
  resourceId: string | null;
  createdAt: string;
}

export interface VisitorEventList<K extends string = string> {
  /** Newest first. */
  events: VisitorEvent<K>[];
  truncated: boolean;
  failed: boolean;
}

/**
 * "Kraków, 12, PL", or null when the edge supplied nothing. Language-neutral:
 * the "Unknown location" fallback is a host string, not this function's.
 */
export function formatVisitLocation(
  visit: Pick<VisitFingerprint, "city" | "region" | "country">,
): string | null {
  const parts = [visit.city, visit.region, visit.country].filter(Boolean);
  return parts.length > 0 ? parts.join(", ") : null;
}

/** Separator between the fragments of a formatted line. A middle dot. */
export const FRAGMENT_SEPARATOR = " \u00b7 ";

/**
 * Browser, OS and device type joined by FRAGMENT_SEPARATOR, or null when
 * nothing was parsed.
 */
export function formatVisitClient(
  visit: Pick<VisitFingerprint, "browser" | "browserVersion" | "os" | "deviceType">,
): string | null {
  // Chrome, Edge and Firefox freeze everything after the major ("131.0.0.0").
  const major = visit.browserVersion?.split(".")[0] ?? null;
  const browser = [visit.browser, major].filter(Boolean).join(" ");
  const parts = [browser, visit.os, visit.deviceType].filter(Boolean);
  return parts.length > 0 ? parts.join(FRAGMENT_SEPARATOR) : null;
}

/** Where a request came from, formatted once for every notifier and panel. */
export interface VisitOrigin {
  location: string | null;
  client: string | null;
  timezone: string | null;
  ip: string | null;
  isBot: boolean;
}

export function describeOrigin(fingerprint: VisitFingerprint): VisitOrigin {
  return {
    location: formatVisitLocation(fingerprint),
    client: formatVisitClient(fingerprint),
    timezone: fingerprint.timezone,
    ip: fingerprint.ip,
    isBot: fingerprint.isBot,
  };
}
```

## The reader

```ts
// file: lib/visits/fingerprint.ts
import { userAgent } from "next/server";

import type { VisitFingerprint } from "./types";

/**
 * Who is on the other end of a request: IP, geolocation, browser, timezone.
 *
 * Geolocation comes from headers the hosting edge adds, not from an IP-lookup
 * service, so there is no third-party call on the request path. Server-only:
 * `userAgent` comes from `next/server`.
 */

/** Long free-text fields are capped before they reach the database. */
const MAX_TEXT = 512;

export function clip(value: string | null | undefined): string | null {
  const clean = value?.trim();
  return clean ? clean.slice(0, MAX_TEXT) : null;
}

export interface GeoFields {
  country: string | null;
  region: string | null;
  city: string | null;
  timezone: string | null;
}

/**
 * Where the platform in front of the app puts the client IP and its location.
 * The one seam in this file: pick the adapter for the edge the request really
 * came through, or the IP is your CDN's and the city is its data centre's.
 */
export interface EdgeHeaders {
  ip(headers: Headers): string | null;
  geo(headers: Headers): GeoFields;
}

/** Vercel percent-encodes city and region so non-ASCII names survive the header. */
function percentDecoded(value: string | null): string | null {
  const raw = clip(value);
  if (!raw) return null;
  try {
    return decodeURIComponent(raw);
  } catch {
    return raw;
  }
}

/**
 * Vercel overwrites `x-forwarded-for` with the client's address, so the
 * left-most entry cannot be spoofed there. Behind any other proxy it can be:
 * do not reuse this adapter anywhere but Vercel.
 */
export const vercelEdge: EdgeHeaders = {
  ip: (headers) =>
    clip(headers.get("x-forwarded-for")?.split(",")[0]) ?? clip(headers.get("x-real-ip")),
  geo: (headers) => ({
    country: clip(headers.get("x-vercel-ip-country")),
    region: percentDecoded(headers.get("x-vercel-ip-country-region")),
    city: percentDecoded(headers.get("x-vercel-ip-city")),
    timezone: clip(headers.get("x-vercel-ip-timezone")),
  }),
};

/**
 * Cloudflare sends raw UTF-8 in header values, which the Fetch `Headers` object
 * hands back decoded as Latin-1 ("KrakÃ³w"). Re-read the bytes as UTF-8.
 */
export function utf8FromLatin1(value: string | null): string | null {
  const raw = clip(value);
  if (!raw) return null;

  const codes = Array.from(raw, (char) => char.codePointAt(0) ?? 0);
  // Pure ASCII needs nothing; anything above U+00FF is already real Unicode.
  if (codes.every((code) => code < 0x80) || codes.some((code) => code > 0xff)) return raw;

  try {
    return new TextDecoder("utf-8", { fatal: true }).decode(Uint8Array.from(codes));
  } catch {
    return raw; // genuinely Latin-1 after all
  }
}

/**
 * Cloudflare proxy. City, region and timezone need the "Add visitor location
 * headers" managed transform; without it only `cf-ipcountry` arrives.
 * Designed from Cloudflare's documentation; not run in production.
 */
export const cloudflareEdge: EdgeHeaders = {
  ip: (headers) => clip(headers.get("cf-connecting-ip")),
  geo: (headers) => ({
    country: clip(headers.get("cf-ipcountry")),
    region: utf8FromLatin1(headers.get("cf-region-code")),
    city: utf8FromLatin1(headers.get("cf-ipcity")),
    timezone: clip(headers.get("cf-timezone")),
  }),
};

/** Local development, or a host with no trusted edge: record nothing rather than a spoofable guess. */
export const noEdge: EdgeHeaders = {
  ip: () => null,
  geo: () => ({ country: null, region: null, city: null, timezone: null }),
};

/**
 * Automation the framework's crawler list misses: headless browsers, HTTP
 * libraries, uptime monitors, link scanners. `\bbot\b` and `bot/` rather than
 * a bare `bot`, which would flag CUBOT phones.
 */
const AUTOMATION =
  /headless|phantomjs|puppeteer|playwright|selenium|webdriver|lighthouse|curl\/|wget\/|python|go-http-client|java\/|okhttp|axios\/|node-fetch|undici|libwww|httpclient|scrapy|crawler|spider|\bbot\b|bot\/|monitor|uptime|preview|scanner/i;

export function looksAutomated(ua: string | null, frameworkSaysBot: boolean): boolean {
  // No user agent at all is a script, never a browser.
  if (!ua) return true;
  return frameworkSaysBot || AUTOMATION.test(ua);
}

/** Accepts a `Request` or bare `Headers` (e.g. from `next/headers`). */
export function describeVisitor(
  source: Request | Headers,
  edge: EdgeHeaders = vercelEdge,
): VisitFingerprint {
  const headers = source instanceof Headers ? source : source.headers;
  const parsed = userAgent({ headers });
  const ua = clip(parsed.ua);
  const isBot = looksAutomated(ua, parsed.isBot);

  return {
    ip: edge.ip(headers),
    ...edge.geo(headers),
    browser: clip(parsed.browser.name),
    browserVersion: clip(parsed.browser.version),
    os: clip(parsed.os.name),
    // The parser leaves `type` undefined for desktop rather than saying so, and
    // guesses wildly for scripts ("wearable" for python-requests).
    deviceType: isBot ? null : (clip(parsed.device.type) ?? "desktop"),
    userAgent: ua,
    referrer: clip(headers.get("referer")),
    isBot,
  };
}

/**
 * Speculative loads: browser prefetch and prerender, mail-client link
 * previews, and the App Router's own `<Link>` prefetches.
 */
function isPrefetch(headers: Headers): boolean {
  return (
    Boolean(headers.get("sec-purpose")?.includes("prefetch")) ||
    headers.get("purpose") === "prefetch" ||
    headers.get("x-purpose") === "preview" ||
    headers.has("next-router-prefetch") ||
    headers.has("next-router-segment-prefetch")
  );
}

/**
 * Whether this request is a person opening a page, rather than a subresource,
 * a prefetch, an embed or a Server Action.
 *
 * `Sec-Fetch-Dest: document` is the reliable signal for a full navigation.
 * An App Router client-side navigation is an RSC fetch (`rsc: 1`, destination
 * `empty`) and counts too. Requests without the header at all (old clients,
 * most bots) are let through and left to bot detection.
 */
export function isPageView(headers: Headers): boolean {
  if (isPrefetch(headers)) return false;
  if (headers.has("next-action")) return false;
  if (headers.get("rsc") === "1") return true;

  const dest = headers.get("sec-fetch-dest");
  return !dest || dest === "document";
}
```

## What the headers really carry

Verified against Vercel's and Cloudflare's header documentation and, where
marked, by running it.

| Header | Carries | Trap |
|---|---|---|
| `x-forwarded-for` (Vercel) | the client's public IP; Vercel overwrites any client-sent value | behind any other proxy the left-most entry is client-controlled |
| `x-real-ip`, `x-vercel-forwarded-for` (Vercel) | the same IP | the second survives a proxy you put in front of Vercel |
| `x-vercel-ip-country` | ISO 3166-1 alpha-2 | absent in local development |
| `x-vercel-ip-country-region` | up to three characters: the ISO 3166-2 subdivision code; in the UK the country ("England"), not the county | render it as a code or map it; it is not a name |
| `x-vercel-ip-city` | city name, non-ASCII percent-encoded (RFC 3986) | decode it, and survive a malformed escape (tested) |
| `x-vercel-ip-timezone` | IANA name | none |
| `x-vercel-ip-latitude`, `-longitude`, `-postal-code`, `-continent` | available, not stored | add columns only with a purpose; they sharpen the IP into a location |
| `cf-connecting-ip` | client IP at the Cloudflare edge | spoofable on any route that bypasses Cloudflare |
| `cf-ipcountry` | country, sent whenever IP geolocation is on | none |
| `cf-ipcity`, `cf-region-code`, `cf-timezone` | need the "Add visitor location headers" managed transform | raw UTF-8: Node's HTTP parser hands it back as Latin-1, `Kraków` becomes `KrakÃ³w` (reproduced on Node 22); `utf8FromLatin1` repairs it |

## What the user agent really says

Run through Next 16.2's `userAgent()`, then `looksAutomated`:

| Client | Parsed | `userAgent().isBot` | `looksAutomated` |
|---|---|---|---|
| Chrome 131 on macOS | Chrome `131.0.0.0`, Mac OS, no device type | false | false |
| Safari on iPhone | Mobile Safari `18.1`, iOS, `mobile` | false | false |
| Googlebot, Slackbot, facebookexternalhit, GPTBot | nothing | **true** | true |
| HeadlessChrome 120 | Chrome Headless, Linux | **false** | true |
| `curl/8.4.0` | nothing | **false** | true |
| `python-requests/2.31.0` | device type `wearable` | **false** | true |
| no `user-agent` header at all | nothing | **false** | true |
| CUBOT Android phone | Chrome, Android, `mobile` | false | false (why `\bbot\b`, not `bot`) |

The rows in bold are why `looksAutomated` exists. Mail-security scanners follow
emailed links with headless browsers before the recipient reads the message;
counted as the recipient, each one is a false "they opened it" at the moment a
salesperson is most likely to act on it.

What no user-agent rule catches: scanners that run a full, normally identified
browser from a cloud address. See "Mail scanners" in
[operations.md](operations.md).

Chrome, Edge and Firefox freeze the minor version and most of the OS version in
the user agent ("131.0.0.0", "Mac OS X 10_15_7" on every Mac). Store the full
string, show the major, and do not build anything on the OS version.

## What counts as a page view

`isPageView` is the filter that decides whether a request is logged at all. Its
contract, row by row, is a test in [testing.md](testing.md):

| Request | Page view? | Why |
|---|---|---|
| `Sec-Fetch-Dest: document` | yes | a full navigation |
| no `Sec-Fetch-*` headers | yes | old clients and most bots; left to `isBot` |
| `rsc: 1`, `Sec-Fetch-Dest: empty` | yes | an App Router client-side navigation |
| `Sec-Fetch-Dest: iframe` / `script` / `image` / ... | no | an embed or a subresource |
| `Sec-Purpose: prefetch` or `prefetch;prerender` | no | Chrome/Safari speculative load |
| `Purpose: prefetch`, `X-Purpose: preview` | no | legacy prefetch, mail-client link preview |
| `next-router-prefetch`, `next-router-segment-prefetch` | no | a `<Link>` prefetch |
| `next-action` | no | a Server Action re-rendering the page |

The last three rows and the `rsc` row are additions for App Router pages. The
header names come from Next 16.2's own router constants; the behaviour has not
been exercised in production. A `router.refresh()` is an RSC request with no
prefetch header and counts as a page view; that is usually what you want.

Those rows need the raw request headers, which a route handler and `proxy.ts`
have and a Server Component does not. On Next 16.3.6, `headers()` in a page
leaves out `rsc` and both prefetch headers (`HIDDEN_REQUEST_HEADERS` in
`request-store.js`), so there a client navigation and a `<Link>` prefetch both
arrive as `Sec-Fetch-Dest: empty` and neither counts. The last row of the test
table pins that; [capture.md](capture.md) says what it means for a page.

A route handler that serves HTML files, as in the earlier implementation, only ever sees the
first six rows. Check the content type as well: only an HTML response is a page.

## Fingerprint checklist

- [ ] Fingerprint read synchronously, before `after()`
- [ ] Edge adapter chosen for the real edge ([adaptation.md](adaptation.md))
- [ ] `looksAutomated` in place; bots stored and labelled, not dropped
- [ ] `isPageView` in front of every capture point, plus an HTML check in file-serving routes
- [ ] No extra geo columns without a stated purpose
