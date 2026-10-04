# Designing signals

A signal is a question SigWise answers about every object. Its `key` (letters,
digits, underscores) is what code and conditions refer to, so name it for the
condition you'll write: `is_scammer >= 90` reads well.

## Pick the type by what the answer drives

| Type | Answer | Good for | Criteria shape |
|------|--------|----------|----------------|
| `noul` | `noul`: probability 0–1 that the answer is yes | thresholds, blocking, alerts | `{"true": "what yes looks like", "false": "what no looks like"}` (both optional) |
| `score` | `score`: 0 = first level … n−1 = last, fractions between; plus `probabilities`, `confidence` | ranking, tiers, gating limits | `["lowest level", "...", "highest level"]` |
| `choice` | `choice`: the likeliest option; plus `probabilities`, `confidence` | routing, categorising | `{"option": "description or null", ...}` |

## Writing instructions and criteria

- **Ask one thing.** Split "scammer or spammer?" into `is_scammer` and `is_spam`.
- **Name the evidence**, not the verdict: "asks to pay outside the platform,
  sends shortened links, overpays and asks for a refund" beats "suspicious".
- **Describe both sides.** For `noul`, say what legitimate behaviour looks
  like too, so ordinary negotiation isn't flagged.
- **Write for the platform.** Mention what objects are ("sellers on a
  second-hand car marketplace") and which events matter.
- **Keep levels and options mutually exclusive** and in order (score levels
  low → high).

## Iterate safely

1. `sigwise signals set ...` (or edit with `--instructions` only; the rest is kept).
2. `sigwise playground -m "<realistic bad example>" -m "<realistic good example>" --signals <key>`
   on several cases, including borderline ones. Nothing is recorded.
3. Adjust wording until scores separate cleanly.
4. Existing objects keep their old answers until analyzed again. To answer a
   new signal for them: `sigwise signals backfill <key> --status` (estimate),
   then `sigwise signals backfill <key>` (billed per object).

## Defaults

New accounts start with `is_scammer` (noul), `trust_score` (score: high risk →
neutral → trusted) and `buyer_intent` (choice). Tune or delete them like any
other signal.

## Examples

```bash
sigwise signals set is_spam --type noul \
  --instructions "Is this user posting spam: unsolicited promotion, repeated identical messages, or link farming?" \
  --true "repeats promos, posts unrelated links, mass-messages strangers" \
  --false "normal conversation about the listing"

sigwise signals set churn_risk --type noul \
  --instructions "Will this customer stop using the product within 30 days?" \
  --true "complains, downgrades, exports data, asks how to cancel" \
  --false "active use, asks about new features, invites teammates"

sigwise signals set seller_quality --type score \
  --instructions "Rate this seller's reliability from their listings, replies and disputes." \
  --level "unreliable" --level "mixed" --level "reliable" --level "excellent"

sigwise signals set message_topic --type choice \
  --instructions "What is the user's latest message mainly about?" \
  --option "pricing=asks about price or discounts" --option "shipping" \
  --option "payment" --option "complaint" --option "other"
