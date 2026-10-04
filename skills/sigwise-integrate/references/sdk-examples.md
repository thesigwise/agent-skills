# SDK examples by language

All SDKs read `SIGWISE_API_KEY`, `SIGWISE_SECRET` and (optionally)
`SIGWISE_BASE_URL` when constructed without credentials. Each returns decoded
JSON matching the API reference (https://sigwise.ai/docs/reference.md).

## Node.js / TypeScript (`@sigwise/node`)

```ts
import { SigWise, SigWiseError, constructWebhookEvent } from "@sigwise/node";
const sigwise = new SigWise();

// async ingest; options: { headers, timeout, maxRetries, signal }
await sigwise.events.ingest("user-42",
  { object_type: "user", events: [{ type: "message", content: text }] },
  { headers: { "Idempotency-Key": `msg-${messageId}` } });

// inline verdict
const v = await sigwise.events.ingest("user-42",
  { wait: true, signals: ["is_scammer"], events: [{ type: "message", content: text }] },
  { timeout: 2000 });
const p = "answers" in v ? v.answers.find(a => a.signal === "is_scammer")?.noul ?? 0 : 0;

// read
try {
  const obj = await sigwise.objects.get("user-42");
  const trust = obj.analysis.find(a => a.key === "trust_score")?.score;
} catch (e) {
  if (e instanceof SigWiseError && e.status === 404) { /* no data yet */ } else throw e;
}

// webhook (Express): raw body!
app.post("/webhooks/sigwise", express.raw({ type: "application/json" }), (req, res) => {
  let event;
  try {
    event = constructWebhookEvent(req.body, req.header("x-webhook-signature"),
      req.header("x-webhook-timestamp"), process.env.SIGWISE_WEBHOOK_SECRET!);
  } catch { return res.sendStatus(400); }
  queue.push(event); // do the work async
  res.sendStatus(204);
});
```

## Python (`sigwise`)

```python
from sigwise import SigWise, construct_webhook_event, WebhookVerificationError
sigwise = SigWise()

sigwise.events.ingest("user-42", object_type="user",
                      events=[{"type": "message", "content": text}])

v = sigwise.events.ingest("user-42", wait=True, signals=["is_scammer"],
                          events=[{"type": "message", "content": text}], timeout=2.0)
p = next((a.get("noul", 0) for a in v.get("answers", []) if a["signal"] == "is_scammer"), 0)

obj = sigwise.objects.get("user-42")
trust = next((a.get("score") for a in obj["analysis"] if a["key"] == "trust_score"), None)

# Flask
@app.post("/webhooks/sigwise")
def hook():
    try:
        event = construct_webhook_event(request.get_data(),
            request.headers.get("X-Webhook-Signature"),
            request.headers.get("X-Webhook-Timestamp"),
            os.environ["SIGWISE_WEBHOOK_SECRET"])
    except WebhookVerificationError:
        return "", 400
    enqueue(event)
    return "", 204
```

## Go (`github.com/thesigwise/sigwise-go`, package `sigwise`)

```go
client := sigwise.New("", "") // empty: read SIGWISE_API_KEY / SIGWISE_SECRET

res, err := client.Events.Ingest(ctx, "user-42", &sigwise.IngestRequest{
	ObjectType: sigwise.String("user"),
	Events:     []sigwise.EventInput{{Type: sigwise.EventTypeMessage, Content: sigwise.String(text)}},
})

res, err = client.Events.Ingest(ctx, "user-42", &sigwise.IngestRequest{
	Wait:    sigwise.Bool(true),
	Signals: []string{"is_scammer"},
	Events:  []sigwise.EventInput{{Type: sigwise.EventTypeMessage, Content: sigwise.String(text)}},
})
if err == nil && res.ModerationVerdict != nil {
	for _, a := range res.ModerationVerdict.Answers {
		if a.Signal == "is_scammer" && a.Noul != nil && *a.Noul >= 0.9 { /* block */ }
	}
}

obj, err := client.Objects.Get(ctx, "user-42")
if sigwise.IsStatus(err, 404) { /* no data yet */ }

// webhook
body, _ := io.ReadAll(r.Body)
ev, err := sigwise.ParseWebhookEvent(body, r.Header.Get("X-Webhook-Signature"),
	r.Header.Get("X-Webhook-Timestamp"), os.Getenv("SIGWISE_WEBHOOK_SECRET"), 5*time.Minute)
if err != nil { http.Error(w, "bad signature", 400); return }
// ev.AnalysisCompletedEvent or ev.RuleTriggeredEvent is set
```

Use a `context.WithTimeout` for the inline call. Per-client headers:
`sigwise.WithHeader(k, v)`.

## PHP (`sigwise/sdk`)

```php
$sigwise = new SigWise\Client(); // reads the environment

$sigwise->events->ingest('user-42', [
    'object_type' => 'user',
    'events' => [['type' => 'message', 'content' => $text]],
], ['headers' => ['Idempotency-Key' => "msg-$messageId"]]);

$v = $sigwise->events->ingest('user-42', [
    'wait' => true, 'signals' => ['is_scammer'],
    'events' => [['type' => 'message', 'content' => $text]],
]);
$p = $v['answers'][0]['noul'] ?? 0;

$event = SigWise\Webhook::constructEvent(
    file_get_contents('php://input'),
    $_SERVER['HTTP_X_WEBHOOK_SIGNATURE'] ?? null,
    $_SERVER['HTTP_X_WEBHOOK_TIMESTAMP'] ?? null,
    getenv('SIGWISE_WEBHOOK_SECRET'),
); // throws SigWise\WebhookVerificationException
```

## Ruby (`sigwise`)

```ruby
sigwise = SigWise::Client.new # reads the environment

sigwise.events.ingest("user-42",
  { object_type: "user", events: [{ type: "message", content: text }] },
  { headers: { "Idempotency-Key" => "msg-#{message.id}" } })

v = sigwise.events.ingest("user-42",
  { wait: true, signals: ["is_scammer"], events: [{ type: "message", content: text }] },
  { timeout: 2 })
p = v.dig("answers", 0, "noul") || 0

# Rails controller
event = SigWise::Webhook.construct_event(request.raw_post,
  request.headers["X-Webhook-Signature"], request.headers["X-Webhook-Timestamp"],
  ENV.fetch("SIGWISE_WEBHOOK_SECRET"))
```

When a detail here disagrees with the SDK's README, the README wins.
