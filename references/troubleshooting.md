# Troubleshooting

## Events duplicated in Events Manager
Cause: client and server sent the same logical event without a shared `event_id`, or `event_id` is regenerated per-call instead of being created once and reused.
Fix: generate the `event_id` at the point the action starts (client-side), store/pass it through to whatever fires the server-side version, and confirm both payloads use the exact same string.

## Events missing entirely
Check in order:
1. Is `fbq` actually defined on `window` by the time the call fires? (race condition if firing before the base script tag loads — guard with `typeof window.fbq === "function"`)
2. Is an ad blocker stripping the pixel in your own testing? Check via server-side CAPI logs, not just Events Manager's "browser" tab.
3. Is the API route/webhook actually being hit? Add a log line before the `fetch` to Graph API and check server logs.
4. Is `META_CAPI_ACCESS_TOKEN` valid and does it have permission for this Pixel ID? A 401/403 from Graph API means a bad or expired token — regenerate in Events Manager → Settings → Conversions API.

## Purchase value doesn't match what Meta shows
Usually a units mismatch (cents vs whole currency units) or the value is computed differently client-side vs server-side (e.g., one includes tax/shipping, the other doesn't). Pick one canonical total (usually the actual charged amount) and use it everywhere.

## Low match quality / poor attribution
- Confirm `em`/`ph` are being hashed with SHA-256 of lowercased+trimmed values, not sent raw or double-hashed.
- Confirm `fbp`/`fbc` cookies are actually present and being forwarded — check they're not being read too early before the pixel sets `_fbp`, and that `_fbc` capture logic actually runs for ad-click traffic (`fbclid` param present).
- Confirm `client_ip_address` and `client_user_agent` are passed through from the real request, not a proxy/CDN's IP.

## Testing before shipping
Use Meta Events Manager → your Pixel → Test Events tab, paste in the `META_TEST_EVENT_CODE` shown there into your env var temporarily, then trigger each event on a staging/dev build and confirm:
- It appears under both "Browser" and "Server" columns for events that fire both ways
- Parameters are populated (not blank/undefined)
- No duplicate rows for the same user action

Remove or unset `META_TEST_EVENT_CODE` before deploying to production — leaving it set causes production events to be flagged as test traffic and excluded from real reporting.

## React Strict Mode double-firing
In development, React 18 Strict Mode intentionally double-invokes effects. A `useEffect` that calls `trackEvent("ViewContent", ...)` on mount may appear to fire twice locally — this is dev-only and won't happen in production builds, but if it's a concern, guard with a `useRef` flag that's checked before firing.
