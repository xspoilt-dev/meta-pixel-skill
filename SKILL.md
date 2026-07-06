---
name: meta-pixel-capi-nextjs
description: Implement, audit, or debug Meta (Facebook) Pixel and Conversions API (CAPI) event tracking in Next.js / Node.js / PayloadCMS e-commerce apps, at the reliability level of plugins like PixelYourSite. Covers standard events (PageView, ViewContent, AddToCart, InitiateCheckout, AddPaymentInfo, Purchase, Search, Lead), server-side Purchase tracking so orders aren't lost to ad blockers or dropped connections, browser+server event deduplication via shared event_id, advanced matching (hashed email/phone/fbp/fbc), and consistent event parameters (currency, value, content_ids, contents). Trigger whenever the user mentions Meta/Facebook pixel, fbq, Conversions API, CAPI, ad tracking events, pixel reliability, event dedup, or is debugging events not showing up (or duplicating) in Meta Events Manager. Also trigger for e-commerce checkout/cart tracking reliability questions even without "Meta" being said explicitly.
---

# Meta Pixel + Conversions API for Next.js / PayloadCMS

## Why this skill exists

Hand-rolled `fbq('track', ...)` calls scattered through a codebase tend to have the same failure modes: events fire inconsistently, parameters are missing or malformed, Purchase is only tracked client-side (so it's lost whenever a browser closes early, an ad blocker strips the pixel, or Safari's ITP blocks it), and there's no deduplication when both a script and a plugin/CAPI send the same event twice. This skill encodes the pattern that reliable setups (like PixelYourSite) use: **fire from both browser and server with a shared event ID, and always fire the money-events (Purchase) server-side as the source of truth.**

## Core architecture (build in this order)

1. **Env + config** — Pixel ID and CAPI access token, one place.
2. **Client pixel loader** — base `fbq` script + `fbp`/`fbc` cookie capture.
3. **Server CAPI sender** — a single reusable function that POSTs to the Graph API.
4. **Shared event helper** — one function the rest of the app calls (`trackEvent(...)`) that generates the `event_id`, fires client-side, and triggers the server-side send — so callers never build raw fbq/CAPI payloads by hand.
5. **Wire up standard events** at the right places in the PayloadCMS/Next.js flow (see event map below).
6. **Advanced matching** — hash and attach user data server-side.
7. **Verify** in Meta Events Manager's Test Events tool before shipping.

Read `references/nextjs-integration.md` for steps 1–2, `references/capi-server-setup.md` for steps 3–4 and 6, and `references/standard-events.md` for step 5 (exact event names + required/recommended parameters for e-commerce). Read `references/troubleshooting.md` when something isn't showing up right or looks duplicated.

## The one rule that matters most: server-side Purchase

Always fire `Purchase` from a server-side hook (e.g., a PayloadCMS `afterChange` hook on the Order collection, or your payment webhook handler) — not only from the client-side "thank you" page. The client-side Purchase call, if you keep it for immediacy in Events Manager, must reuse the **same `event_id`** as the server call so Meta dedupes them into one event. Never let Purchase depend solely on the browser successfully loading a confirmation page.

## Quick checklist when reviewing or writing pixel code

- [ ] Every `trackEvent` call site passes the same shape: event name, an `event_id`, and a params object with `currency` + `value` for e-commerce events
- [ ] Purchase is triggered server-side on order creation/payment confirmation, not just client-side
- [ ] Client and server calls for the same logical event share one `event_id`
- [ ] `fbp` and `fbc` cookies are read and forwarded to the server call (needed for CAPI attribution)
- [ ] PII (email, phone, name) is SHA-256 hashed, lowercased, and trimmed before sending — never sent raw
- [ ] `content_ids`, `content_type`, and `contents` match the actual product catalog IDs used in Meta Catalog (if running dynamic ads/catalog)
- [ ] No event fires twice from duplicate mounts (e.g., React Strict Mode double-invoking effects) — dedupe by checking `event_id` was already sent this session for view-type events, or guard with a ref
- [ ] Access token and Pixel ID come from env vars, never hardcoded
- [ ] Tested in Meta Events Manager → Test Events before considering it done

## Gathering context before writing code

Before generating code, check the project for:
- Existing pixel code (search for `fbq(`, `NEXT_PUBLIC_FB_PIXEL`, `graph.facebook.com`)
- The PayloadCMS Order/Product collection shape (field names for price, currency, line items) — read the actual collection config rather than assuming field names
- Where checkout/payment is handled (Stripe webhook? Payload hook? custom route?) — Purchase must hook into whichever place is the actual source of truth for a completed order
- Whether a Pixel ID / access token already exists in `.env` — ask the user for it if not, don't invent placeholder IDs into committed code paths without flagging them clearly as placeholders

If any of this isn't visible in the codebase, ask the user rather than guessing field names or IDs.
