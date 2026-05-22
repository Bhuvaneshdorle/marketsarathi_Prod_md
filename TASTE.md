# Taste

> One person. Four repos. A daily-delivery product that can't miss a day. This is how I think about code.

I'm **Bhuvanesh Dorle** — founder, designer, engineer, ops, and customer support for Market Saarthi. Every line in the stack is mine. Every 5 AM IST production incident is mine. This document is not a résumé; it's a tour of the engineering decisions that keep a one-person product reliable.

Everything below is backed by code that ships in production today.

---

## Principles

### 1. Fast ack, async work, durable log

WhatsApp providers don't wait. If a webhook doesn't return inside ~5 seconds, they retry. Retries cause duplicates. Duplicates cause angry customers.

Every webhook in the backend follows the same three-beat pattern: **validate fast → ack → process in background → log everything**.

```ts
// market-pulse-backend/src/routes/wati-webhook.ts
watiWebhook.post('/wati-webhook', async (c) => {
  // --- Fast validation (return inside milliseconds) ---
  let payload: any;
  try {
    payload = await c.req.json();
  } catch (parseError: any) {
    return c.json({ status: 'error', reason: 'invalid JSON' }, 400);
  }

  // Filter outbound + non-message events early
  if (payload.eventType && payload.eventType !== 'message') {
    return c.json({ status: 'ignored', reason: `event type: ${payload.eventType}` });
  }
  if (payload.owner === true) { /* skip business-sent */ }

  if (normalizedMessage !== TRIGGER_MESSAGE) {
    return c.json({ status: 'ignored', reason: 'not trigger message' });
  }

  // Ack immediately — process in background
  processWebhookInBackground(waId, name).catch((err) => {
    console.error('[WEBHOOK-BG] Unhandled error:', err);
  });

  return c.json({ status: 'accepted', waId });
});
```

The `.catch()` on the floating promise is deliberate. An unhandled rejection in background work shouldn't crash the process — it should be logged and move on.

---

### 2. Errors should tell you *where* they died

A 2 AM error log that says `Error: ECONNRESET` is useless. A 2 AM log that says `[WEBHOOK-BG] Error at step [wati_send_pdf] for 91XXXXXXXXXX: ...` is a fix you can ship before breakfast.

I label every step in long async chains:

```ts
async function processWebhookInBackground(waId: string, name: string | null) {
  let step = 'db_upsert';
  try {
    await sql`INSERT INTO users (wa_id, name) VALUES (${waId}, ${name})
              ON CONFLICT (wa_id) DO NOTHING`;

    step = 'db_check_trial';
    const result = await sql`SELECT ended_at >= NOW() AS is_active, ...`;

    step = 'wati_send_pdf';
    // ... retry loop ...
  } catch (error: any) {
    console.error(`[WEBHOOK-BG] Error at step [${step}] for ${waId}:`, error?.message || error);
  }
}
```

Cheap. Boring. Saves hours.

---

### 3. Idempotency at the database layer, not the application layer

If my application code is the only thing preventing duplicate deliveries, I will eventually ship a bug that duplicates a delivery. Idempotency belongs in the schema:

```ts
// market-pulse-backend/src/cron/daily-report.ts
await sql`
  INSERT INTO report_deliveries (wa_id, pdf_url, source, status)
  VALUES (${user.wa_id}, ${pdfUrl}, 'cron', 'success')
  ON CONFLICT (wa_id, report_date) WHERE status = 'success' DO NOTHING
`;
```

A **partial unique index** on `(wa_id, report_date) WHERE status = 'success'` means:
- A user can have many `failed` rows for the same day (good for retries).
- But only **one** successful delivery per day. The DB enforces it.
- The `ON CONFLICT … DO NOTHING` is the cleanup; the index is the safety net.

Same idea for expiry notifications — `expiry_notified_at IS NULL` in the WHERE clause + `UPDATE … SET expiry_notified_at = NOW()` after the send. A user is *physically incapable* of being re-spammed.

---

### 4. Failure domains stay isolated

The daily cron job does two things: send today's report, then notify expired users. Most people would put both in one `try` block. That means a PDF render bug stops expiry notifications too — a separate, unrelated SLA.

```ts
async function sendDailyReport() {
  // PART 1: PDF generation & delivery
  try {
    const pdfPath = await generatePdf();
    // ... full delivery flow ...
  } catch (error) {
    console.error('[CRON] PDF generation/delivery failed:', error);
  }

  // PART 2: Expiry notifications (independent — always runs)
  await sendExpiryNotifications();
}
```

The comment is the point. The PDF flow can crash and burn; expiry notifications still go out. Two SLAs, two failure domains.

---

### 5. Retries are not "just try again"

A retry without bounds is a thundering herd. A retry without a cap is a leak. A retry without a backoff is a DDoS on your own provider.

**In-request retry** (when a user is actively waiting):

```ts
const MAX_ATTEMPTS = 3;
const RETRY_DELAY_MS = 5000;
let lastError: any = null;

for (let attempt = 1; attempt <= MAX_ATTEMPTS; attempt++) {
  try {
    await sendPdfViaWati(waId, pdfUrl);
    lastError = null;
    break;  // success — stop retrying
  } catch (sendError: any) {
    lastError = sendError;
    if (attempt < MAX_ATTEMPTS) {
      await new Promise(resolve => setTimeout(resolve, RETRY_DELAY_MS));
    }
  }
}

if (lastError) {
  // Persist failure so the retry cron can pick it up later
  await sql`INSERT INTO report_deliveries (..., status, error_message) VALUES (..., 'failed', ...)`;
}
```

**Out-of-band retry cron** picks up what the in-request retries missed, with a cap and a sanity check:

```sql
-- market-pulse-backend/src/cron/retry-failed-reports.ts
SELECT rd.id, rd.wa_id, rd.pdf_url, rd.retry_count, u.name
FROM report_deliveries rd
JOIN users u ON u.wa_id = rd.wa_id
WHERE rd.report_date = CURRENT_DATE
  AND rd.status = 'failed'
  AND rd.retry_count < 3                    -- cap retries
  AND u.ended_at >= NOW()                   -- skip expired subscribers
  AND NOT EXISTS (                          -- skip if already delivered another way
    SELECT 1 FROM report_deliveries rd2
    WHERE rd2.wa_id = rd.wa_id
      AND rd2.report_date = CURRENT_DATE
      AND rd2.status = 'success'
  )
```

Three guards in one query: **retry cap**, **subscription validity**, **cross-source dedupe**. The cron can run as often as I want; it won't double-send.

---

### 6. Operability is a feature

I run this alone. If something breaks at 4:55 AM, I need to know *what* without SSHing into anything.

Every webhook has a `GET` diagnostic twin that reports env-var presence and live dependency health:

```ts
watiWebhook.get('/wati-webhook', async (c) => {
  const checks: Record<string, any> = {
    DATABASE_URL: !!process.env.DATABASE_URL,
    DATABASE_URL_host: process.env.DATABASE_URL?.match(/@([^:\/]+)/)?.[1] || 'unknown',
    WATI_API_KEY: !!process.env.WATI_API_KEY,
    WATI_API_ENDPOINT: !!process.env.WATI_API_ENDPOINT,
    db_connected: false,
    db_error: null,
  };
  try {
    await sql`SELECT 1`;
    checks.db_connected = true;
  } catch (err: any) {
    checks.db_connected = false;
    checks.db_error = err?.message || String(err);
  }
  return c.json(checks);
});
```

`curl /api/wati/wati-webhook` in the browser, get the truth, fix the config. No log diving.

---

### 7. Timezones are a product decision

Indian markets open at 9:15 AM IST. A naive UTC date in a PDF filename means the file for "today" might be named after yesterday for half the day. So:

```ts
export function getTodayPdfFilename(): string {
  const istDate = new Date(new Date().getTime() + (5.5 * 60 * 60 * 1000));
  return `MarketSarathi_${istDate.toISOString().split('T')[0]}.pdf`;
}
```

Database rows use `report_date DATE` columns. Comparisons use `CURRENT_DATE`. The whole product agrees that "today" means "today in Mumbai."

---

## The PDF craft

PDF rendering is the highest-risk, highest-craft part of the product. The 14-page report is the entire product, and it ships once a day. If it looks wrong, customers churn.

I deliberately separated the renderer into its own Next.js app. The backend doesn't `puppeteer` over inline HTML — it navigates a real, deployed Next.js app at A4 dimensions. That separation means:
- The same URL I QA in my browser is what gets PDF'd.
- Charts render with the same JS engine they were designed in.
- Tailwind utility classes, fonts, gradients, animations — all behave identically in preview and production.

But "navigate and print" is naïve. Real life:

```ts
await page.goto(REPORT_URL, { waitUntil: 'networkidle0', timeout: 90000 });

await page.waitForSelector('body', { timeout: 30000 });
await page.waitForFunction('document.readyState === "complete"', { timeout: 30000 });
await new Promise(r => setTimeout(r, 8000));     // settle

// Force lazy content + chart layout
await page.evaluate('window.scrollTo(0, document.body.scrollHeight)');
await new Promise(r => setTimeout(r, 3000));
await page.evaluate('window.scrollTo(0, 0)');

// Wait for images
await page.waitForFunction(`Array.from(document.images).every(img => img.complete)`,
  { timeout: 30000 }).catch(() => console.log('Some images may not have loaded'));

// Wait for web fonts — typography fidelity is non-negotiable
await page.waitForFunction('document.fonts.ready', { timeout: 30000 })
  .catch(() => console.log('Font loading timed out, proceeding anyway'));

// Wait for React hydration
await page.waitForFunction(`!document.querySelector('[data-reactroot]') || ...`,
  { timeout: 15000 }).catch(() => console.log('React hydration check timed out'));
```

Each `waitForFunction` has a timeout *and* a `.catch()`. A missing font shouldn't kill the PDF — it should log a warning and ship a slightly-imperfect report. **Soft-degrade beats hard-fail** for a product that has to land in someone's inbox before 9 AM.

Plus print-specific CSS injection: `page-break-inside: avoid` on cards, `orphans/widows: 3` on paragraphs, gray backgrounds forced to white (printers hate gray), emoji forced into the Noto Color Emoji family so sentiment icons render in color on every device.

```ts
await page.pdf({
  format: 'A4',
  printBackground: true,
  margin: { top: '1cm', right: '1cm', bottom: '1cm', left: '1cm' },
  preferCSSPageSize: true,
});
```

`preferCSSPageSize: true` is the tell — print engineering, not screen engineering.

---

## Frontend taste

Two surfaces:

**The report renderer** (private repo) — 14 React components, one per page, A4-sized (`794 × 1123 px`), Tailwind v4, AG Charts for financial visualizations, framer-motion for the subtle motion that survives PDF rasterization. Data flows through a single `MarketDataContext`; loading and error states are *always* rendered (never just a spinner that hangs forever):

```tsx
if (loading && !data) {
  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50">
      <div className="animate-spin rounded-full h-12 w-12 border-b-2 border-blue-600" />
    </div>
  );
}

if (error || !data) {
  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-50">
      <div className="text-center">
        <div className="text-red-600 mb-2">Error: {error || 'No data available'}</div>
        {isUsingSampleData() && (
          <div className="text-sm text-gray-500">
            Make sure NEXT_PUBLIC_USE_SAMPLE_DATA is set to 'true' in your environment
          </div>
        )}
      </div>
    </div>
  );
}
```

The `isUsingSampleData()` branch is the small touch that separates a senior frontend dev from a senior dev: **the error message tells you how to fix it**.

**My personal site** (`dorl3-frontend`) — the second surface, which doubles as a credential. Stack: Next.js 15 + React 19, shadcn/ui primitives (avatar, badge, button, card, dialog, separator, tooltip), Magic UI for motion (blur-fade, dock, terminal), tailwind, framer-motion. Custom local `GeneralSans` font loaded via `next/font/local` with `display: swap` (no FOIT, perfect Lighthouse). Full Open Graph + Twitter + robots metadata. Theme-aware dark mode via a proper provider. A `TubesBackground` WebGL canvas as the hero. Nothing accidental.

The taste rule: **animations should reward attention, not demand it**. Every motion is sub-300ms, easing-out, triggered by scroll/hover/route-change — never auto-loops.

---

## What I refuse to do

- **No Redis I don't need.** Postgres handles the working set. A queue layer is premature.
- **No microservices for the sake of it.** The three repos are split by *runtime* (Node API, Vercel-hosted Next.js apps), not by domain.
- **No ORM.** `postgres.js` tagged-template SQL gives me the SQL I want with parameter safety. ORMs would add weight I can't pay back at this scale.
- **No premature observability stack.** A `report_deliveries` table + structured `console` logs + diagnostic endpoints is enough until it isn't. When it isn't, I'll add OTel.
- **No silent failures.** Every catch logs. Every batch operation tallies sent/failed and prints the summary. If something happens, I'll know.
- **No tests-for-tests-sake.** Tests are missing where I judge the cost-of-bug × probability-of-bug to be low. They will exist where it isn't (the PDF render path is heading there next).

---

## How I work

- **Ship daily.** The product literally requires it. Code that doesn't ship doesn't count.
- **Make the boring thing reliable before making the interesting thing exist.** The cron, the DB schema, the webhook ack pattern — those came before the AI patterns page.
- **Logs over dashboards, until logs aren't enough.** Searchable, prefixed (`[WEBHOOK-BG]`, `[CRON]`, `[RETRY-CRON]`), step-labeled.
- **Write production code; if I can ship it without tests today, I'd better understand it well enough to fix it without tests at 5 AM.**
- **Comment the *why*, not the *what*.** The `// PART 2: Expiry notifications (independent — always runs)` comment exists because three months from now I'll forget why the `await` isn't inside the `try`.

---

## Contact

📧 [bhuvaneshdorle@gmail.com](mailto:bhuvaneshdorle@gmail.com)
🌐 [marketsarathi.com](https://marketsarathi.com/)
👤 [github.com/Bhuvaneshdorle](https://github.com/Bhuvaneshdorle)

If this looks like the kind of engineering you want on your team — or if you want me to build something like this for you — get in touch.
