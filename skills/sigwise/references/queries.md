# Search and condition syntax

Used by `sigwise objects list "<query>"` and `sigwise rules create --when "<condition>"`.

```
is_scammer >= 90 and trust_score < 1
buyer_intent = ready_to_buy, is_scammer < 20
```

- Comparisons are joined by `and` or commas; all must hold.
- Operators: `>= <= > < = !=`.

| Signal type | Value | Operators |
|-------------|-------|-----------|
| `noul` | a percentage: `90` and `0.9` both mean 0.9 | all |
| `score` | the raw score, 0 = first level | all |
| `choice` | an option name (optionally quoted) | `=`, `!=` |

An object with no answer for a signal never matches a comparison on it.

**Search is lenient**: text with no operator is a substring search on object
IDs and names; comparisons that don't parse are skipped. **Rules are strict**:
the condition must parse, name existing signals and use valid operators, or
creation fails with a `400` explaining why.

Sort search results with `--sort flagged` (highest yes/no probability first)
or `--sort trust` (highest score first).
