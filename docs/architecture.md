# Architecture

A deeper look at how Market Saarthi is engineered.

---

## System overview

Market Saarthi is composed of three deployable units behind a single product:

1. **Backend API** — the brain. Aggregates data, runs AI, generates PDFs, talks to WhatsApp, handles payments, runs cron jobs.
2. **Report App** — a dedicated Next.js renderer whose only job is to display the 14-page report perfectly. It exists to be screenshotted, not browsed.
3. **Frontend** — a public-facing site for prospects and the in-browser report preview.

Everything is glued together by a Postgres database (Supabase) and a CDN-backed object store (Supabase Storage).

---

## Why a separate "report app"?

PDF generation is the highest-stakes part of the product. We wanted:

- **Pixel-perfect typography and charts** — server-side rasterized canvas charts are limiting; CSS-driven layouts in a real browser are not.
- **A11y / print fidelity** — A4 dimensions (`794 × 1123 px`), print-aware Tailwind, real font loading.
- **Live previewability** — the same app that renders the PDF is the one we use to design and QA each day's report.

So the report is built as a normal React app. Puppeteer launches headless Chromium, navigates to the report URL, waits for hydration and chart settle, and prints to PDF. No HTML→PDF conversion library; we use the browser itself.

---

## Data flow (the morning broadcast)

```
┌────────────────────────────────────────────────────────────────────┐
│   Cron trigger (pre-market, IST)                                   │
└──────────────────────────┬─────────────────────────────────────────┘
                           ▼
┌────────────────────────────────────────────────────────────────────┐
│   Backend: orchestrator                                            │
│   • Pull subscribers where ended_at >= NOW()                       │
│   • Fan-in: aggregate market data from multiple providers           │
│   • Run Gemini pattern recognition over NSE chart data              │
│   • Persist daily snapshot → /api/market/pulse                      │
└──────────────────────────┬─────────────────────────────────────────┘
                           ▼
┌────────────────────────────────────────────────────────────────────┐
│   Headless Chromium (Puppeteer)                                    │
│   • Navigate to report app                                          │
│   • waitFor: networkidle + chart layout + font load                 │
│   • page.pdf({ format: 'A4', printBackground: true })               │
└──────────────────────────┬─────────────────────────────────────────┘
                           ▼
┌────────────────────────────────────────────────────────────────────┐
│   Supabase Storage                                                  │
│   • Upload MarketSarathi_YYYY-MM-DD.pdf                             │
│   • Generate signed CDN URL                                         │
└──────────────────────────┬─────────────────────────────────────────┘
                           ▼
┌────────────────────────────────────────────────────────────────────┐
│   Fan-out: WhatsApp Business API                                    │
│   • For each subscriber: send template message with PDF link        │
│   • Log every send into report_deliveries                           │
│   • On failure: mark for retry                                      │
└────────────────────────────────────────────────────────────────────┘
```

The PDF is generated **once per day**. The fan-out reuses the same artifact for every recipient — it's the cheap part. Generation is the expensive part, so we cache aggressively.

---

## Inbound flow (on-demand request)

When a subscriber proactively messages the Market Saarthi number:

1. **WhatsApp provider posts a webhook** to the backend.
2. The handler **acknowledges with `200 OK` immediately** — provider timeouts are unforgiving, so we never do real work in the response cycle.
3. The actual work runs in the background:
   - Upsert the user (new users get a free trial window).
   - Check subscription status. Expired? Send the appropriate template. Active? Continue.
   - If today's PDF is already generated, send it. Otherwise, generate-then-send.
4. Result is logged to `report_deliveries`.

This pattern — **fast ack, async work, durable log** — is the same one that keeps Stripe webhooks reliable. It's the right shape for any provider integration.

---

## Subscription & payments

- Payments via **PhonePe** (UPI — the default for Indian B2C).
- PhonePe webhook hits `/api/payment/phonepay/webhook`.
- Successful payment → `users.ended_at` extended by the purchased plan duration.
- A nightly cron checks for users whose `ended_at` is within an expiry window. If `expiry_notified_at IS NULL`, send a one-shot reminder and set `expiry_notified_at = NOW()`. Idempotent.

No subscription is ever silently cancelled. No user is ever notified twice.

---

## AI: pattern recognition

Each day, the backend pulls historical OHLC data for a curated NSE watchlist and asks Google Gemini to identify technical chart patterns:

- Double Bottom / Double Top
- Head & Shoulders (and inverse)
- Rising / Falling Wedges
- Triangles, Flags, Cup & Handle

For every detected pattern, Gemini returns a short, plain-English explanation: what the pattern is, what it suggests, and the levels to watch. These end up on the AI Patterns page of the report.

The LLM is used as a **structured-output classifier**, not as a free-form analyst. Its outputs are constrained, validated, and rendered into pre-designed report slots.

---

## Reliability building blocks

| Concern | How it's handled |
|---|---|
| Provider webhook timeouts | Fast `200 OK` ack, async processing |
| Duplicate deliveries | Daily-PDF cache keyed by IST date |
| Failed sends | `report_deliveries` table + retry cron |
| Expired-user spam | `expiry_notified_at` idempotency flag |
| Cold-start PDF render | Puppeteer browser pool kept warm in container |
| Chart/font flicker in PDFs | Explicit `waitFor` on layout + font-ready events |
| Container portability | Docker image with Chromium + Indian fonts baked in |

---

## What's deliberately *not* in the system

- **No mobile app.** WhatsApp is the app.
- **No user dashboard.** Reports go to your phone. That's the product.
- **No real-time streaming.** Pre-market is a once-a-day moment, not a live feed.
- **No paid LLM dependence on the critical path.** Pattern recognition is the only AI step, and the report ships even if it fails (the page degrades, the PDF goes out).

The product surface is intentionally tiny. Everything in the stack exists to make that one daily moment work.
