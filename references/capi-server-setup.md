# Server-Side Conversions API (Node.js / PayloadCMS)

## 1. The reusable CAPI sender

One function, called from anywhere server-side (API route, PayloadCMS hook, webhook handler).

```ts
// lib/meta/send-capi-event.ts
import crypto from "crypto";

const PIXEL_ID = process.env.NEXT_PUBLIC_META_PIXEL_ID!;
const ACCESS_TOKEN = process.env.META_CAPI_ACCESS_TOKEN!;
const TEST_EVENT_CODE = process.env.META_TEST_EVENT_CODE; // omit in production

function hash(value?: string) {
  if (!value) return undefined;
  return crypto.createHash("sha256").update(value.trim().toLowerCase()).digest("hex");
}

interface SendCapiEventArgs {
  eventName: string;
  eventId: string;           // MUST match the client-side event_id for the same logical event
  eventSourceUrl: string;
  userData: {
    email?: string;
    phone?: string;
    fbp?: string;
    fbc?: string;
    clientIpAddress?: string;
    clientUserAgent?: string;
    externalId?: string;     // e.g. your internal user/order id
  };
  customData: Record<string, unknown>; // value, currency, content_ids, contents, etc.
  actionSource?: "website" | "system_generated" | "other";
}

export async function sendCapiEvent({
  eventName,
  eventId,
  eventSourceUrl,
  userData,
  customData,
  actionSource = "website",
}: SendCapiEventArgs) {
  const payload = {
    data: [
      {
        event_name: eventName,
        event_time: Math.floor(Date.now() / 1000),
        event_id: eventId,
        event_source_url: eventSourceUrl,
        action_source: actionSource,
        user_data: {
          em: hash(userData.email),
          ph: hash(userData.phone),
          fbp: userData.fbp,
          fbc: userData.fbc,
          client_ip_address: userData.clientIpAddress,
          client_user_agent: userData.clientUserAgent,
          external_id: hash(userData.externalId),
        },
        custom_data: customData,
      },
    ],
    ...(TEST_EVENT_CODE ? { test_event_code: TEST_EVENT_CODE } : {}),
  };

  const res = await fetch(
    `https://graph.facebook.com/v20.0/${PIXEL_ID}/events?access_token=${ACCESS_TOKEN}`,
    {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(payload),
    }
  );

  if (!res.ok) {
    const err = await res.text();
    console.error("Meta CAPI send failed:", eventName, err);
    // Don't throw and break the checkout/order flow over a tracking failure —
    // log it (and ideally alert/retry) but let the business logic continue.
  }

  return res.ok;
}
```

Key points baked into this:
- `em`/`ph` (email/phone) are always hashed — never sent raw.
- Failures are logged, not thrown — a tracking API hiccup should never break checkout.
- `test_event_code` is opt-in via env var so you can verify in Meta's Test Events tool without polluting production data, and it's simply absent in prod.

## 2. Wiring Purchase into PayloadCMS (source of truth)

Fire from wherever the order is actually confirmed as paid — a PayloadCMS collection hook is the most robust place if Orders live in Payload:

```ts
// collections/Orders.ts (excerpt)
import { sendCapiEvent } from "../lib/meta/send-capi-event";

export const Orders: CollectionConfig = {
  slug: "orders",
  hooks: {
    afterChange: [
      async ({ doc, previousDoc, req }) => {
        const justPaid = doc.status === "paid" && previousDoc?.status !== "paid";
        if (!justPaid) return;

        await sendCapiEvent({
          eventName: "Purchase",
          eventId: doc.metaEventId, // generated and stored when the order/checkout session was created
          eventSourceUrl: `${process.env.NEXT_PUBLIC_SITE_URL}/order-confirmation/${doc.id}`,
          userData: {
            email: doc.customerEmail,
            phone: doc.customerPhone,
            fbp: doc.fbp,   // captured client-side at checkout start, stored on the order
            fbc: doc.fbc,
            clientIpAddress: req.ip,
            clientUserAgent: req.headers?.["user-agent"],
            externalId: String(doc.id),
          },
          customData: {
            currency: doc.currency,
            value: doc.total,
            content_ids: doc.items.map((i) => i.productId),
            contents: doc.items.map((i) => ({ id: i.productId, quantity: i.quantity })),
            content_type: "product",
            num_items: doc.items.reduce((sum, i) => sum + i.quantity, 0),
            order_id: String(doc.id),
          },
        });
      },
    ],
  },
  // ...fields
};
```

If instead a payment processor webhook (e.g., Stripe) is the actual confirmation point, put this same call in that webhook handler rather than a Payload hook — fire it wherever payment success is genuinely first known, not earlier.

**Important**: generate and store `metaEventId` (and `fbp`/`fbc`) on the order/checkout session *when checkout starts* (client-side), so the same ID is available later when the server confirms payment. This is what lets the client-side and server-side Purchase events (if both exist) dedupe correctly.

## 3. A generic API route for other events (AddToCart, InitiateCheckout, etc.)

```ts
// app/api/meta-capi/route.ts
import { NextRequest, NextResponse } from "next/server";
import { sendCapiEvent } from "@/lib/meta/send-capi-event";

export async function POST(req: NextRequest) {
  const { eventName, eventId, params, fbp, fbc, email, phone } = await req.json();

  await sendCapiEvent({
    eventName,
    eventId,
    eventSourceUrl: req.headers.get("referer") || process.env.NEXT_PUBLIC_SITE_URL!,
    userData: {
      email,
      phone,
      fbp,
      fbc,
      clientIpAddress: req.headers.get("x-forwarded-for") ?? undefined,
      clientUserAgent: req.headers.get("user-agent") ?? undefined,
    },
    customData: params,
  });

  return NextResponse.json({ ok: true });
}
```

## 4. Advanced matching data quality notes

- Always lowercase + trim email/phone before hashing (`hash()` above does this) — Meta's own hashing on their end expects normalized input, mismatched casing silently breaks matching.
- Phone numbers should be normalized to E.164 format (`+15551234567`) before hashing, since inconsistent formats reduce match rate.
- Send as many matching fields as are legitimately available (email, phone, external_id, fbp, fbc, client IP, user agent) — more fields = better match rate, but never fabricate data that isn't real.
