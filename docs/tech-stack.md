# Tech Stack

A full breakdown of the technology choices behind Market Saarthi, and the reasoning behind each.

---

## Backend API

| Choice | Why |
|---|---|
| **TypeScript** | Type safety across a service that wires together half a dozen external APIs and webhooks. The cost of a runtime surprise in a daily-broadcast pipeline is a missed delivery. |
| **Hono** | A tiny, fast, edge-compatible HTTP framework. Lower overhead than Express, better DX than raw `node:http`, runs anywhere. |
| **postgres.js** | Direct, no-ORM access to Postgres. Few enough tables that an ORM would be more weight than help. |
| **Puppeteer** | Headless Chromium is still the best HTML→PDF rendering engine available — better than any pure-Node PDF library for typography, charts, and CSS. |
| **Vercel AI SDK + Gemini** | Structured outputs, schema validation, multi-provider portability. Gemini is cost-effective for high-volume vision/structured tasks. |
| **Docker** | Chromium + Indian language fonts pre-baked. Reproducible. Deploys anywhere. |

---

## Data layer

| Choice | Why |
|---|---|
| **Supabase (Postgres)** | Managed Postgres with auth, storage, and realtime under one roof. Saves running infra for a small product. |
| **Supabase Storage** | CDN-fronted object store. PDFs are large; we don't want to send them through our own bandwidth. |

---

## Report rendering

| Choice | Why |
|---|---|
| **Next.js 15 + React 19** | App Router for clean routing; SSR-capable for fast first paint; Vercel-native deployment. |
| **Tailwind CSS v4** | Print-friendly styling without a custom CSS pipeline. Easy to design A4 layouts. |
| **AG Charts** | Best-in-class for financial visualizations (candlesticks, gauges, complex axes). |
| **Recharts** | Lighter charts where AG Charts is overkill. |
| **Gauge components** | Custom Fear & Greed and VIX gauges. |

The report app is **only** for rendering. It's a static-data target for the backend's Puppeteer instance — no auth, no interactivity, no analytics noise.

---

## Public frontend

| Choice | Why |
|---|---|
| **Next.js 15** | Same stack as the report app — shared mental model. |
| **Tailwind v4** | Same as report. |
| **Framer Motion** | Marketing-grade animations on the landing/preview surface. |

---

## Messaging & payments

| Choice | Why |
|---|---|
| **WhatsApp Business API (WATI)** | WhatsApp is *the* communication channel in India. WATI provides templated messaging, broadcast, and reliable webhook delivery without us managing Meta's API directly. |
| **PhonePe** | UPI is the default payment rail in India. PhonePe's webhook flow is straightforward and well-documented. |

---

## Operational

| Choice | Why |
|---|---|
| **Cron-driven jobs** | Daily pre-market broadcast, expiry notifications, retry sweeps. Simple, observable, no queue infrastructure needed at this scale. |
| **`report_deliveries` table** | Single source of truth for "did this user get today's report?" — drives retries, support tooling, and reliability metrics. |

---

## What we deliberately don't use

- **No GraphQL.** REST + a handful of endpoints is enough.
- **No Redis.** Postgres handles the working set comfortably; a queue layer would be premature.
- **No Kubernetes.** A single container behind a process manager is the right shape for this product today.
- **No event bus.** Cron + webhooks + a delivery table is observable, debuggable, and trivially correct. We'll add one when scale demands it, not before.
- **No microservices.** Three repos, one bounded context. They're separate because their *runtimes* differ (Node API vs. Vercel-hosted Next.js), not because the domains are independent.

---

## Guiding principle

> **Boring, durable infrastructure on the critical path. Modern, expressive tools at the edges.**

The path from "user wants today's report" to "PDF lands on their phone" runs through Postgres, a webhook, a cron job, and a CDN. Every one of those pieces has been load-bearing in production systems for over a decade. The places we reach for newer tools — Hono, Next.js 15, Gemini — are the places where a regression is recoverable, not the places where a missed delivery costs a customer.
