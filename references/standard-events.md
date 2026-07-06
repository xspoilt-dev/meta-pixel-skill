# Standard Events for E-commerce (Meta Pixel + CAPI)

Fire these both client-side (`fbq('track', ...)`) and server-side (Graph API `/events` call) using the shared `trackEvent` helper, with matching `event_id` for any event that exists in both places.

Every event below must include `currency` and `value` when money is involved — Meta cannot optimize ad delivery well without them, and campaigns silently under-perform when these are missing or inconsistent (e.g., some events in cents, some in whole units — pick one and be consistent everywhere).

## PageView
Fires on every page load. No special params needed beyond the base pixel call. Client-side only (server-side PageView is not standard practice — skip it).

## ViewContent
Fires on a product detail page.
```
content_ids: [productId]        // must match Meta Catalog product ID if catalog is in use
content_type: "product"
content_name: product.title
value: product.price
currency: "USD"                 // or your store's actual currency
```
Trigger point: product page server component/route, or client-side on mount of the product page — but prefer firing via a `useEffect` guarded so it doesn't double-fire on re-renders.

## AddToCart
Fires when a user adds a product to cart.
```
content_ids: [productId]
content_type: "product"
contents: [{ id: productId, quantity: qty }]
value: price * qty
currency: "USD"
```
Trigger point: wherever the "add to cart" mutation/action succeeds — fire after confirmation of success, not on button click, so failed adds aren't counted.

## InitiateCheckout
Fires when checkout begins (cart → checkout page transition).
```
content_ids: [...cart item ids]
contents: [{ id, quantity }, ...]
num_items: total quantity across cart
value: cart subtotal
currency: "USD"
```

## AddPaymentInfo
Fires when payment details are submitted (e.g., Stripe payment method attached), before final confirmation.
```
content_ids: [...cart item ids]
value: cart total
currency: "USD"
```

## Purchase — the most important one, fire server-side
Fires on confirmed order completion. **This must be triggered from the server-side source of truth** — a PayloadCMS `afterChange` hook on Order (when status flips to "paid"/"completed"), or a payment webhook handler (e.g., Stripe `checkout.session.completed`). See `capi-server-setup.md` for the hook pattern.
```
content_ids: [...ordered product ids]
contents: [{ id, quantity, item_price }, ...]
content_type: "product"
num_items: total quantity
value: order total (must match what's charged)
currency: "USD"
order_id: your internal order ID   // helps Meta dedupe and helps you cross-reference
```
If you also fire a client-side Purchase on the "thank you" page for immediacy, it MUST reuse the same `event_id` as the server-side call, and the same `order_id`/value, or Meta will double-count.

## Search (optional but recommended if there's on-site search)
```
search_string: query
content_ids: [...matching result ids] (optional)
```

## Lead / CompleteRegistration (if there's any signup/newsletter/quote form)
```
content_name: "newsletter_signup" // or whatever form
value / currency: optional, only if the lead has an estimated monetary value
```

## Parameter consistency rules

- `currency` must be a valid ISO 4217 code and identical across all events in a session (don't mix "USD" and "usd" or switch currencies mid-flow).
- `value` must be a plain number, not a formatted string ("49.99" as a string is fine, "$49.99" is not).
- `content_ids` must be strings matching your Meta Catalog feed IDs exactly, if you run a catalog. If there's no catalog, still pass a consistent internal product ID.
- Don't invent parameters not in Meta's schema — extra custom data can go in a `custom_data` sub-object for CAPI, but stick to documented standard event parameters for the pixel call.
