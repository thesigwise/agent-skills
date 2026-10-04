# sigwise CLI reference

Global flags work on every command and may come before or after it:
`--account <name>`, `--base-url <url>`, `--json`, `--verbose` (log HTTP
requests to stderr), `--no-color`, `-y/--yes` (skip confirmations).

Exit codes: `0` ok, `1` API or network error, `2` usage error. With `--json`,
API errors are printed as the API's JSON error envelope on stderr:
`{"error":{"code":"not_found","message":"..."}}`.

## Accounts

```bash
sigwise signup [--org NAME] [--email E] [--name ACCOUNT] [--no-key] [--no-switch]
sigwise login  [--email E] [--code 2FA] [--name ACCOUNT] [--no-key] [--no-switch]
sigwise login  --api-key KEY_ID [--secret S] --name ACCOUNT     # save a key instead
sigwise logout [ACCOUNT] [--all] [--keep-key]
sigwise whoami
sigwise accounts list | current | show [NAME]
sigwise accounts use NAME
sigwise accounts add --name NAME --api-key KEY_ID     # secret prompted / read from stdin
sigwise accounts rename OLD NEW
sigwise accounts remove NAME                          # forget locally, revoke nothing
```

Config file: `~/.config/sigwise/config.json` (`$SIGWISE_CONFIG_DIR` overrides).
Env: `SIGWISE_API_KEY` + `SIGWISE_SECRET` (use a key without a saved account),
`SIGWISE_BASE_URL`, `SIGWISE_ACCOUNT`.

## Signals

```bash
sigwise signals list
sigwise signals get KEY
# noul
sigwise signals set is_scammer --type noul \
  --instructions "Decide if this user is likely trying to defraud other users." \
  --true "asks to pay off-platform, overpayment, phishing links" \
  --false "ordinary questions and negotiation"
# score (levels low → high)
sigwise signals set trust_score --type score --instructions "Rate how far others can trust this user." \
  --level "high risk" --level neutral --level trusted
# choice (option or option=description)
sigwise signals set buyer_intent --type choice --instructions "Where is this user in a purchase?" \
  --option browsing --option "comparing=asks about alternatives" --option "ready_to_buy=asks how to pay"
# raw criteria JSON, or a whole definition from a file
sigwise signals set KEY --type choice --instructions "..." --criteria '{"a":null,"b":"desc"}'
sigwise signals set KEY --file signal.json
# partial update keeps the rest of the definition
sigwise signals set KEY --instructions "new wording"
sigwise signals enable KEY | disable KEY | delete KEY
sigwise signals backfill KEY --status      # estimate + progress
sigwise signals backfill KEY               # start (asks; --yes to skip)
```

## Events and objects

```bash
sigwise events send OBJECT_ID -m "message text" [-m ...] [-e action.name] \
  [--meta key=value ...] [-t user] [--at 2026-01-02T15:04:05Z]
sigwise events send OBJECT_ID -f events.json       # {"events":[...]}, [...], or NDJSON; - for stdin
sigwise events send OBJECT_ID -m "..." --wait [--signals a,b] [--no-history]
sigwise events list OBJECT_ID
```

Each send includes a random `Idempotency-Key` (override with
`--idempotency-key`), so retrying the same command never double-records.
`--meta` values that parse as JSON stay typed (`n=3` → number).

```bash
sigwise objects list [QUERY] [--sort recent|flagged|trust] [--limit 50] [--cursor C] [--all]
sigwise objects get ID [--wait 30s]     # --wait polls until nothing is pending
sigwise objects events ID
sigwise objects state ID                # compacted long-term history
sigwise objects analyze ID              # billed
sigwise objects analyze-all             # billed per object; asks
sigwise playground -m "text" [-e name] [--signals a,b] [-t user] [-f file]   # billed, records nothing
```

## Rules

```bash
sigwise rules list | get ID | delete ID
sigwise rules create --when "is_scammer >= 90" --email abuse@acme.com [--subject "..."] [--name N] [--cooldown 1h]
sigwise rules create --when "spam_risk > 80" --slack https://hooks.slack.com/services/...
sigwise rules create --when "buyer_intent = ready_to_buy" --webhook ENDPOINT_ID
sigwise rules update ID [--when ...] [--enabled=false] [--cooldown 30m]
sigwise rules firings [--rule ID] [--object ID] [--limit 50]
```

A webhook action needs an endpoint subscribed to `rule.triggered`. Rules fire
on transitions (when an object starts matching), then respect the cooldown.

## Webhooks

```bash
sigwise webhooks list | get ID | delete ID
sigwise webhooks create --url https://example.com/hook [--events analysis.completed,rule.triggered] [--signal KEY ...] [--description D]
sigwise webhooks update ID [--url U] [--enabled=false]
sigwise webhooks deliveries [--status pending|delivering|delivered|dead] [--endpoint ID]
sigwise webhooks replay DELIVERY_ID
sigwise webhooks verify --secret S --timestamp T --signature sha256=... --body payload.json
```

## API keys

```bash
sigwise keys list
sigwise keys create --name Production [--save ACCOUNT]   # secret printed once
sigwise keys update ID --name N | --deactivate | --activate
sigwise keys rotate ID [--save ACCOUNT]                  # old secret dies immediately
sigwise keys revoke ID
```

`ID` is the UUID in the first column of `keys list`, not the public key ID.

## Account, billing, usage

```bash
sigwise overview
sigwise settings get | set --auto-backfill-signals=true
sigwise billing balance
sigwise billing ledger [--type credit|charge] [--group] [--limit 50]
sigwise billing topup --amount 25 [--open] [--wait 10m]   # user pays in the browser
sigwise billing checkout SESSION_ID [--wait 5m]
sigwise usage [--days 30 | --from YYYY-MM-DD --to YYYY-MM-DD]
sigwise password change | forgot | reset
sigwise mfa status | setup | enable | disable
sigwise notifications list | set low_balance=off
sigwise identities list | unlink github
```

## Agent skills

```bash
sigwise init [--agent claude|codex|agents] [--global]   # sign-in check + install skills
sigwise skills install [--skill NAME] [--global] [--dir DIR]
sigwise skills list
```

## Raw API

```bash
sigwise api /v1/me
sigwise api GET "/v1/objects?q=is_scammer>90&limit=5"
sigwise api PUT /v1/signals/is_spam -d '{"type":"noul","instructions":"Is this spam?"}'
sigwise api POST /v1/objects/user-42/events --file body.json
sigwise api PATCH /v1/settings -F auto_backfill_signals=true
```
