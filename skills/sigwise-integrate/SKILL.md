---
name: sigwise-integrate
description: Add SigWise to an application's code with the official SDKs (Node.js/TypeScript, Python, Go, PHP, Ruby). Use when the user wants to send user/listing/order events or messages to SigWise from their backend, gate or moderate content before publishing (scam, spam, abuse), read trust scores or other signal answers in their app, or receive SigWise webhooks. For account setup, signals, rules and keys from the terminal, use the sigwise skill.
---

# Integrate SigWise into an app

SigWise scores objects (users, listings, orders) from the events and messages
you send it, and returns typed answers per **signal**: `noul` (probability
0–1), `score` (position on labelled levels), `choice` (one of N options).

There are three integration points; most apps need the first and one of the
others:

1. **Send events** when things happen (sign-up, profile edit, message sent,
   listing created). Asynchronous and cheap on your side.
2. **Gate content inline** (`wait: true`): score a message or listing before
   publishing it and block / hold / publish on the verdict.
3. **Consume answers**: read them on demand (`objects.get`) or receive
   `analysis.completed` / `rule.triggered` webhooks.

## Before writing code

1. Find the backend language and framework, and where the relevant actions
   happen (message send handler, listing create, profile update, sign-up).
2. Check signals exist and fit: `sigwise signals list` (sigwise skill). New
   accounts have `is_scammer`, `trust_score`, `buyer_intent`.
3. Credentials: the SDK reads `SIGWISE_API_KEY` (public key ID) and
   `SIGWISE_SECRET` (signing secret) from the environment. If the user has
   none for this app, they can mint one: `sigwise keys create --name "<app> production"`.
   Put them in the app's secret store or untracked `.env`, add placeholders to
   `.env.example`, and never commit the secret or ship it to a browser or
   mobile client. The secret signs requests server-side only.

## Install the SDK

| Language | Install | Import |
|----------|---------|--------|
| Node.js / TS (18+) | `npm install @sigwise/node` | `import { SigWise } from "@sigwise/node"` |
| Python (3.8+) | `pip install "git+https://github.com/thesigwise/sigwise-python"` | `from sigwise import SigWise` |
| Go (1.21+) | `go get github.com/thesigwise/sigwise-go` | `sigwise "github.com/thesigwise/sigwise-go"` |
| PHP (8.1+, ext-curl) | `composer config repositories.sigwise vcs https://github.com/thesigwise/sigwise-php && composer require sigwise/sdk:dev-main` | `new SigWise\Client(...)` |
| Ruby (2.7+) | Gemfile: `gem "sigwise", git: "https://github.com/thesigwise/sigwise-ruby", branch: "main"` | `require "sigwise"` |

Any other language: sign requests yourself, see
https://sigwise.ai/docs/guide/authentication.md. Exact method and type names
for every SDK: [references/sdk-examples.md](references/sdk-examples.md) and
each SDK's README.

Create **one client per process** (module-level singleton), constructed with no
arguments so it reads the environment:

```ts
// lib/sigwise.ts
import { SigWise } from "@sigwise/node";
export const sigwise = new SigWise(); // SIGWISE_API_KEY, SIGWISE_SECRET, optional SIGWISE_BASE_URL
```

## 1. Send events

```ts
await sigwise.events.ingest(`user-${user.id}`, {
  object_type: "user",
  events: [
    { type: "message", content: message.body, metadata: { to: `user-${message.toId}`, listing: listing.id } },
    { type: "event", name: "profile.updated", metadata: { field: "bio", name: user.displayName } },
  ],
});
```

- **Object IDs** are yours; prefix by kind (`user-42`, `listing-7`) and keep
  them stable. `metadata.name` becomes the display name in the console.
- **`type: "message"`** carries free text in `content`; **`type: "event"`**
  is a named action (`listing.created`, `payout.requested`) with `metadata`.
- **Batch** events of one request into one call; the API debounces bursts per
  object into one analysis.
- **Don't block the user's request on it.** Fire after commit, from a queue /
  background job, or fire-and-forget with error logging. A failure to report
  an event should never fail the user's action.
- **Retries**: send an `Idempotency-Key` header and reuse it on retry,
  otherwise a retried POST records events twice. In Node, pass it as the
  options argument: `ingest(id, body, { headers: { "Idempotency-Key": key } })`
  (derive the key from your own record, e.g. the message ID). The SDKs never
  auto-retry POSTs.
- Don't send secrets, passwords or payment card data in `content`/`metadata`.
  Send what a human moderator would need to judge.

## 2. Gate content inline

```ts
const verdict = await sigwise.events.ingest(`user-${user.id}`, {
  wait: true,
  signals: ["is_scammer"],          // score only what the decision needs
  events: [{ type: "message", content: draft }],
});
const p = "answers" in verdict ? (verdict.answers.find(a => a.signal === "is_scammer")?.noul ?? 0) : 0;
if (p >= 0.9) return block();
if (p >= 0.6) return holdForReview();
return publish();
```

- Returns `200` with `answers` (an async ingest returns `202` without them).
  `include_history: false` judges only this content, ignoring the author's
  past.
- Give it a **timeout** that fits the latency budget (Node: `{ timeout: 2000 }`
  as the options argument), and
  decide fail-open vs fail-closed with the user; it's a product decision.
  Errors: `402` balance empty, `429` too many sync analyses in flight, `502` /
  `503 analyzer_busy` upstream problems. In all of these the events **were
  recorded**; don't resend them.
- Thresholds are a starting point; tell the user to tune them on real traffic.

## 3. Consume answers

**On demand** (`GET /v1/objects/{id}`, cached for a few seconds):

```ts
const obj = await sigwise.objects.get(`user-${id}`);
const trust = obj.analysis.find(a => a.key === "trust_score")?.score; // undefined until analyzed
```

Signals not yet answered are listed in `obj.pending`. A brand-new object
returns `404` until its first event is recorded; handle it as "no data yet".

**Webhooks** (push): register the endpoint with
`sigwise webhooks create --url https://app.example.com/webhooks/sigwise`
(prints the signing secret once; store it as `SIGWISE_WEBHOOK_SECRET`). In the
handler:

- Read the **raw body** (before JSON parsing) and verify with the SDK helper:
  Node `constructWebhookEvent(rawBody, sigHeader, tsHeader, secret)`, Python
  `construct_webhook_event(...)`, Go `sigwise.ParseWebhookEvent(...)`, PHP
  `SigWise\Webhook::constructEvent(...)`, Ruby `SigWise::Webhook.construct_event(...)`.
  Headers: `X-Webhook-Signature`, `X-Webhook-Timestamp`.
- Reject on failure with `400`; respond `2xx` fast and do the work async.
- Handle duplicates idempotently (key on `X-Webhook-Delivery`).
- Payload: `{"event":"analysis.completed","object_id":"user-42","answers":[{"signal":"is_scammer","type":"noul","noul":0.91}], ...}`.

For alerting without code, a rule may be enough:
`sigwise rules create --when "is_scammer >= 90" --email abuse@acme.com`.

## Verify the integration

1. Run the app's flow (or a unit test with the SDK mocked) and check
   `sigwise objects get <id> --wait 30s` shows events and answers.
2. For webhooks in local dev, use a tunnel URL, then
   `sigwise webhooks deliveries` to see status codes and errors, and
   `sigwise webhooks replay <id>` after fixing the handler.
3. Add tests that mock the SDK client: assert the right object ID, event
   types and that SigWise failures don't break the user flow.

## Docs

Every page is Markdown at its URL + `.md`. Index: https://sigwise.ai/llms.txt.
Key pages: ingesting events (`/docs/guides/ingesting-events.md`), direct
moderation (`/docs/guides/moderation.md`), reading results
(`/docs/guides/reading-results.md`), webhooks (`/docs/guides/webhooks.md`),
errors and retries (`/docs/guides/errors.md`).
