# Client-Side Pixel Setup (Next.js App Router)

## 1. Env vars

```
NEXT_PUBLIC_META_PIXEL_ID=xxxxxxxxxxxxxxx
META_CAPI_ACCESS_TOKEN=xxxxxxxxxxxxxxxxxxxxx   # server-only, never prefix with NEXT_PUBLIC_
META_TEST_EVENT_CODE=TEST12345                 # optional, only while testing in Events Manager
```

The access token must NEVER be exposed client-side. Only the Pixel ID is public.

## 2. Base pixel script — `app/layout.tsx`

Use `next/script` with `strategy="afterInteractive"` so it doesn't block page render but still loads early enough to catch the first PageView.

```tsx
import Script from "next/script";

export default function RootLayout({ children }: { children: React.ReactNode }) {
  const pixelId = process.env.NEXT_PUBLIC_META_PIXEL_ID;

  return (
    <html lang="en">
      <body>
        <Script id="meta-pixel-base" strategy="afterInteractive">
          {`
            !function(f,b,e,v,n,t,s)
            {if(f.fbq)return;n=f.fbq=function(){n.callMethod?
            n.callMethod.apply(n,arguments):n.queue.push(arguments)};
            if(!f._fbq)f._fbq=n;n.push=n;n.loaded=!0;n.version='2.0';
            n.queue=[];t=b.createElement(e);t.async=!0;
            t.src=v;s=b.getElementsByTagName(e)[0];
            s.parentNode.insertBefore(t,s)}(window, document,'script',
            'https://connect.facebook.net/en_US/fbevents.js');
            fbq('init', '${pixelId}');
            fbq('track', 'PageView');
          `}
        </Script>
        {children}
      </body>
    </html>
  );
}
```

For route changes in the App Router (client-side navigations don't reload `layout.tsx`), add a PageView fire on route change using `usePathname` in a small client component:

```tsx
"use client";
import { usePathname, useSearchParams } from "next/navigation";
import { useEffect } from "react";

export function PixelRouteTracker() {
  const pathname = usePathname();
  const searchParams = useSearchParams();

  useEffect(() => {
    if (typeof window.fbq === "function") {
      window.fbq("track", "PageView");
    }
  }, [pathname, searchParams]);

  return null;
}
```
Mount `<PixelRouteTracker />` once in the root layout, inside `<body>`.

## 3. Capturing `fbp` / `fbc` cookies for CAPI matching

Meta sets the `_fbp` cookie automatically once the pixel loads. `_fbc` is only set if the user arrived via a Meta ad click (`fbclid` in URL) — capture it manually if the base pixel doesn't set it in your setup:

```ts
// lib/meta/cookies.ts
export function getFbCookies() {
  if (typeof document === "undefined") return { fbp: undefined, fbc: undefined };
  const get = (name: string) =>
    document.cookie.match(new RegExp(`(?:^|; )${name}=([^;]*)`))?.[1];
  return { fbp: get("_fbp"), fbc: get("_fbc") };
}
```
Pass `fbp`/`fbc` up to the server whenever you call your API route that triggers a server-side CAPI event (e.g., include them in the AddToCart mutation payload), so the server call has them available. See `capi-server-setup.md`.

## 4. The shared `trackEvent` client helper

This is what feature code should call — never raw `fbq()` directly — so `event_id` generation is consistent and reused for the paired server call.

```ts
// lib/meta/track-event.ts
import { v4 as uuidv4 } from "uuid";

export function trackEvent(eventName: string, params: Record<string, unknown> = {}) {
  const eventId = uuidv4();

  if (typeof window !== "undefined" && typeof window.fbq === "function") {
    window.fbq("track", eventName, params, { eventID: eventId });
  }

  return eventId; // caller passes this same ID to the server-side CAPI call
}
```

Usage pattern at a call site (e.g., AddToCart button handler, after the mutation succeeds):
```ts
const eventId = trackEvent("AddToCart", {
  content_ids: [productId],
  content_type: "product",
  value: price * qty,
  currency: "USD",
});

await fetch("/api/meta-capi", {
  method: "POST",
  body: JSON.stringify({
    eventName: "AddToCart",
    eventId,
    params: { content_ids: [productId], value: price * qty, currency: "USD" },
    fbp: getFbCookies().fbp,
    fbc: getFbCookies().fbc,
  }),
});
```
