# SigWise agent skills

Skills that teach AI coding agents (Claude Code, Codex, and other agents that
read `SKILL.md` files) to work with [SigWise](https://sigwise.ai).

| Skill | What it does |
|-------|--------------|
| [`sigwise`](skills/sigwise/SKILL.md) | Manage a SigWise account with the [`sigwise` CLI](https://github.com/thesigwise/sigwise-cli): accounts, signals, events, objects, rules, webhooks, API keys, billing. |
| [`sigwise-integrate`](skills/sigwise-integrate/SKILL.md) | Add SigWise to application code with the official SDKs: send events, gate content inline, read answers, verify webhooks. |

## Install

Point your agent at the quickstart and let it do the rest:

```
Add SigWise to my app: https://sigwise.ai/SKILL.md
```

Or install the CLI and the skills yourself:

```bash
curl -fsSL https://raw.githubusercontent.com/thesigwise/sigwise-cli/main/scripts/install.sh | sh
sigwise init                          # project: .claude/skills
sigwise skills install --global       # user: ~/.claude/skills
sigwise skills install --agent codex  # project: .agents/skills
```

Or copy `skills/<name>/` into your agent's skills directory by hand.

Then ask your agent, for example:

```
/sigwise set up scam detection for our marketplace
/sigwise who are our riskiest users this week?
/sigwise-integrate send our chat messages to SigWise and block scams before they're delivered
```

## Layout

```
skills/
  <name>/
    SKILL.md          front matter (name, description) + instructions
    references/       detail the agent loads when it needs it
```

`sigwise skills install` downloads this repository's `main` branch and copies
every directory under `skills/` that has a `SKILL.md`.

## Contributing

Keep `SKILL.md` short and procedural; put long references in `references/`.
Check every command against `sigwise <command> --help` and every SDK call
against the SDK's README before changing it.

## License

MIT
