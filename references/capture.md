# Capture points

Where the host calls the logger: the places a resource is served, and the
places a person signs in. Each example marks the host's own functions with
`declare function`. Those are seams, left for the host to replace. They
type-check here and fail loudly if copied unreplaced.

| Capture point | Status | Page view test |
|---|---|---|
| Route handler serving a gated static build (demo, report) | shipped in the earlier implementation | HTML content type **and** `isPageView` |
| App Router page (Server Component) | designed here | `isPageView(await headers())` |
| `proxy.ts` | not recommended | the viewer is not known yet, the route may 404, and the matcher sees every asset |
| Client beacon (`navigator.sendBeacon`) | not shipped | only if time-on-page matters; it is a client analytics feature |

## A gated static build, served by a route handler

The shape it was built for: a folder of built HTML, CSS and JS served by one route
handler to whoever may see it. Every asset goes through the same handler, so
the HTML check is essential. Without it, one page view is fourteen rows.

```ts
// file: app/share/[id]/route.ts
import { type NextRequest, NextResponse } from "next/server";

import { originLines, openedHeadline } from "@/lib/visits/announce";
import { isPageView } from "@/lib/visits/fingerprint";
import { getVisitStore } from "@/lib/visits/store";
import { trackPageVisit } from "@/lib/visits/track";
import type { VisitVia } from "@/lib/visits/types";

// Host seams: replace each with your own.
declare function authoriseViewer(
  request: NextRequest,
  id: string,
): Promise<{ subject: string | null; via: VisitVia; label: string | null } | null>;
declare function readSharedFile(id: string, path: string[]): Promise<{ body: Uint8Array; contentType: string } | null>;
declare function postToSlack(text: string): Promise<void>;

export const dynamic = "force-dynamic";

/**
 * Serves a gated static build (a demo, a report, a proposal) to whoever may
 * see it, and records who opened it. Assets are served by the same handler;
 * only HTML documents are page views.
 */
export async function GET(
  request: NextRequest,
  { params }: { params: Promise<{ id: string; path?: string[] }> },
) {
  const { id, path = [] } = await params;

  const viewer = await authoriseViewer(request, id);
  // 404 rather than 403: do not confirm what exists.
  if (!viewer) return new NextResponse("Not found", { status: 404 });

  const file = await readSharedFile(id, path);
  if (!file) return new NextResponse("Not found", { status: 404 });

  const isHtml = file.contentType.startsWith("text/html");
  if (isHtml && isPageView(request.headers)) {
    trackPageVisit({
      getStore: getVisitStore,
      request,
      visit: {
        resourceId: id,
        resourceKey: viewer.label,
        subject: viewer.subject,
        via: viewer.via,
        path: new URL(request.url).pathname,
      },
      announce: async (assessment, fingerprint) => {
        if (!viewer.subject) return;
        await postToSlack(
          [openedHeadline(viewer.subject, assessment), ...originLines(fingerprint)].join("\n"),
        );
      },
    });
  }

  return new NextResponse(new Uint8Array(file.body), {
    headers: {
      "Content-Type": file.contentType,
      // Gated content: never let a shared cache keep it.
      "Cache-Control": isHtml ? "private, no-store" : "private, max-age=300",
      "X-Robots-Tag": "noindex, nofollow",
    },
  });
}
```

If the resource is reached through a link that carries a token in the query
string, redirect to the clean URL after setting the cookie, **before** logging.
Otherwise the redirect and the landing page are two page views, and the token
sits in browser history and in the `Referer` of every asset request.

## An App Router page

```tsx
// file: app/proposals/[id]/page.tsx
import { headers } from "next/headers";
import { notFound } from "next/navigation";

import { isPageView } from "@/lib/visits/fingerprint";
import { getVisitStore } from "@/lib/visits/store";
import { trackPageVisit } from "@/lib/visits/track";
import type { VisitVia } from "@/lib/visits/types";

// Host seam: who may see this resource, and how they got in.
declare function authoriseViewer(
  id: string,
): Promise<{ subject: string | null; via: VisitVia; title: string } | null>;

/**
 * A resource rendered by the App Router rather than served as a file. The
 * page runs for the document request, for a client-side navigation (an RSC
 * fetch) and for a `<Link>` prefetch. `headers()` hides the RSC headers, so
 * here `isPageView` counts the document request only. Reading `headers()`
 * makes the page dynamic, which it must be: on a static page `after()` runs
 * at build time.
 */
export default async function ProposalPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params;
  const viewer = await authoriseViewer(id);
  if (!viewer) notFound();

  const requestHeaders = await headers();
  if (isPageView(requestHeaders)) {
    trackPageVisit({
      getStore: getVisitStore,
      request: requestHeaders,
      visit: {
        resourceId: id,
        resourceKey: viewer.title,
        subject: viewer.subject,
        via: viewer.via,
        path: `/proposals/${id}`,
      },
    });
  }

  return <main>{viewer.title}</main>;
}
```

The page component runs for the document request, for a client-side navigation
to it (an RSC request), and for a `<Link>` prefetch. Only the first is counted
here. On Next 16.3.6 the `headers()` a Server Component reads leaves out the
flight headers (`rsc`, `next-router-prefetch`, `next-router-segment-prefetch`),
so a client-side navigation reaches the page as `Sec-Fetch-Dest: empty` with
no `rsc`, and `isPageView` drops it together with the prefetches it can no
longer tell apart. Reproduced against `next start`: the same page logged
`{"rsc":null,"dest":"empty"}` for a request sent with `rsc: 1`. A shared link's
first open is always a document request, so the announcement is unaffected;
what is not logged is moving to the resource from another page of the app.
Counting those needs the raw headers, which only `proxy.ts` sees: classify
there with `isPageView` and pass the verdict to the page in a request header
the proxy always overwrites. That is a design, not shipped and not run.

With Cache Components enabled,
the component that reads `headers()` must sit inside `<Suspense>`, and the
tracking call belongs in that component.

## Lifecycle events with Supabase Auth

Three moments before any resource exists are worth a fingerprint: the sign-in
request, the first sign-in, and later sign-ins. The fourth kind in the
reference set, `resource_created`, is recorded wherever the host creates the
subject's first resource, with the same `trackVisitorEvent` call.

### The first-sign-in trap

`signInWithOtp` with `shouldCreateUser: true` creates the auth user **when the
link is requested**. `created_at` is therefore the request time. The earlier
implementation decided "new account" by `created_at` being under five minutes old at the
callback, so anyone who opened the email more than five minutes after asking
for it was recorded as `signed_in`. The host's "new workspace" notification,
gated on the same test, never fired for them. The auth server's source shows
both halves: the magic-link handler signs the user up at request time; the
verify handler confirms the email on the first verification and leaves
`email_confirmed_at` alone on every later one.

```ts
// file: lib/visits/supabase-auth.ts
import type { User } from "@supabase/supabase-js";

/**
 * Whether this sign-in is the account's first, for the `account_created`
 * event. Supabase Auth only.
 *
 * Not `created_at`. `signInWithOtp` with `shouldCreateUser` creates the auth
 * user when the link is *requested*, so `created_at` is the request time, and
 * anyone who opens the email more than a few minutes later reads as a
 * returning user. `email_confirmed_at` is set by the first verification and
 * left alone by every later one; the callback runs seconds after it.
 */
const FIRST_SIGN_IN_WINDOW_MS = 2 * 60 * 1000;

export function isFirstSignIn(
  user: Pick<User, "email_confirmed_at">,
  now: number = Date.now(),
): boolean {
  if (!user.email_confirmed_at) return false;
  return now - new Date(user.email_confirmed_at).getTime() < FIRST_SIGN_IN_WINDOW_MS;
}
```

The two-minute window covers the gap between verification and the callback,
which is seconds, plus clock skew. It is short on purpose: a second device
signing in a few minutes after the first would otherwise record a second
`account_created`.

Other providers have their own signal: Clerk's `user.created` webhook,
NextAuth's `isNewUser` in the `signIn` event. Use the provider's statement of
"first", never a timestamp comparison against the request.

### The sign-in request

```ts
// file: app/api/auth/link/route.ts
import type { SupabaseClient } from "@supabase/supabase-js";
import { type NextRequest, NextResponse } from "next/server";

import { getVisitStore } from "@/lib/visits/store";
import { trackVisitorEvent } from "@/lib/visits/track";

// Host seams: the Supabase SSR client and the rate limiter it already has.
declare function createRouteClient(request: NextRequest, response: NextResponse): SupabaseClient;
declare function allowSignInRequest(request: NextRequest, email: string): boolean;

/**
 * Sends the magic link. The form submission is the one request certainly made
 * by the person's own browser, so its fingerprint is the best "signed up
 * from" there will be. Recorded only once the link was actually sent.
 */
export async function POST(request: NextRequest) {
  const body = (await request.json().catch(() => null)) as { email?: unknown } | null;
  const email = typeof body?.email === "string" ? body.email.trim().toLowerCase() : "";
  if (!email.includes("@")) return NextResponse.json({ ok: false }, { status: 400 });
  if (!allowSignInRequest(request, email)) return NextResponse.json({ ok: false }, { status: 429 });

  const response = NextResponse.json({ ok: true });
  const supabase = createRouteClient(request, response);
  const { error } = await supabase.auth.signInWithOtp({
    email,
    options: { shouldCreateUser: true, emailRedirectTo: `${new URL(request.url).origin}/auth/callback` },
  });
  if (error) return NextResponse.json({ ok: false }, { status: 400 });

  trackVisitorEvent({
    getStore: getVisitStore,
    request,
    event: { subject: email, kind: "link_requested", resourceId: null },
  });
  return response;
}
```

Record it only after the link was actually sent. A rate-limited or failed
request is not a moment in anyone's lifecycle.

### The callback

```ts
// file: app/auth/callback/route.ts
import type { SupabaseClient } from "@supabase/supabase-js";
import { type NextRequest, NextResponse } from "next/server";

import { getVisitStore } from "@/lib/visits/store";
import { isFirstSignIn } from "@/lib/visits/supabase-auth";
import { trackVisitorEvent } from "@/lib/visits/track";

declare function createRouteClient(request: NextRequest, response: NextResponse): SupabaseClient;

export async function GET(request: NextRequest) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get("code");
  if (!code) return NextResponse.redirect(`${origin}/sign-in`);

  const response = NextResponse.redirect(`${origin}/`);
  const supabase = createRouteClient(request, response);
  const { data, error } = await supabase.auth.exchangeCodeForSession(code);
  if (error) return NextResponse.redirect(`${origin}/sign-in?error=auth_failed`);

  const user = data.session?.user;
  if (user?.email) {
    // A mail scanner may make this click before the person does; `isBot`
    // catches the ones that say so, and `pickOriginEvent` prefers the form.
    trackVisitorEvent({
      getStore: getVisitStore,
      request,
      event: {
        subject: user.email,
        kind: isFirstSignIn(user) ? "account_created" : "signed_in",
        resourceId: null,
      },
    });
  }
  return response;
}
```

## Capture checklist

- [ ] Every capture point behind `isPageView`, plus an HTML check where assets share the route
- [ ] Token-in-URL redirect happens before logging
- [ ] `via` from the host's authorisation, staff first ([adaptation.md](adaptation.md))
- [ ] `link_requested` recorded after the link is sent, not before
- [ ] `account_created` decided by the provider's "first", not by `created_at`
