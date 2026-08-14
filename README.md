# eusend for coding agents

The official [Agent Skill](https://agentskills.io) for [eusend](https://eusend.dev), the
EU-native transactional email API — packaged as a Claude Code plugin, and usable in any
skills-compatible agent ([see below](#other-agents)).

As a plugin it bundles two things:

- **A skill** that teaches Claude how eusend actually behaves — the send options, the rules
  that bite (verified senders, idempotent retries, raw-body webhook signatures, the SPF
  record you should _not_ add), which errors are worth retrying, and where the limits are.
  Claude loads it on its own when it notices you're working with eusend.
- **The eusend MCP server**, so Claude can send mail, add domains, and inspect delivery
  against your real account.

## Install

```
/plugin marketplace add eusend-dev/eusend-skill
/plugin install eusend@eusend
```

Then set your API key so the bundled MCP server can authenticate:

```bash
export EUSEND_API_KEY=eu_live_xxxxxxxxxxxx
export EUSEND_FROM=hello@yourdomain.com   # optional: pins a default sender
```

Use an `eu_test_` key while you're wiring things up — sends are accepted, recorded, and fire
the full webhook lifecycle, but never reach a real inbox.

The skill works with no key at all. Only the MCP server needs one.

## Use it

Claude picks the skill up automatically when a task involves eusend. To invoke it directly:

```
/eusend:eusend
```

Some things to try:

```
Add password reset emails to this app using eusend
Why is this send returning DOMAIN_NOT_VERIFIED?
Write a webhook handler for eusend bounces
Set up the DNS records for acme.com
```

## What's in the skill

`SKILL.md` stays short and is the only part always in context. The detail lives in
`references/`, loaded on demand:

| File                      | Covers                                                                     |
| ------------------------- | -------------------------------------------------------------------------- |
| `sending.md`              | Send options, attachments, scheduling, templates, batch, React Email, SMTP |
| `domains-dns.md`          | Adding a domain, DKIM/SPF/DMARC, Return-Path alignment                     |
| `webhooks.md`             | Events, payloads, verified handlers in three languages, reading a bounce   |
| `audiences-broadcasts.md` | Audiences, contacts, broadcasts, unsubscribes, suppressions                |
| `error-codes.md`          | Every code, its status, and whether to retry                               |
| `testing.md`              | Test-mode keys, simulating bounces, testing webhook handlers               |

## Other agents

[Agent Skills](https://agentskills.io) is an open standard, and this one is written to it —
`SKILL.md` carries only spec-standard frontmatter (`name`, `description`, `license`) and the
body uses no Claude-specific syntax. So it works in Cursor, GitHub Copilot, VS Code, Codex,
Gemini CLI, OpenCode, Goose, Amp, Roo Code, and the rest of the skills-compatible ecosystem.

Only the install differs. The `/plugin` command above is Claude Code's marketplace format;
everywhere else you drop the skill folder into a directory the agent scans. Cursor, Codex and
Copilot all read `.agents/skills/`, so one copy covers them:

```bash
git clone https://github.com/eusend-dev/eusend-skill
mkdir -p .agents/skills
cp -r eusend-skill/plugins/eusend/skills/eusend .agents/skills/
```

Use `~/.agents/skills/` instead to install it once for every project. Other agents scan their
own locations — `.github/skills/` for Copilot, `.cursor/skills/` for Cursor, `~/.claude/skills/`
for Claude Code without the plugin — and any of them work; check your agent's docs if
`.agents/skills/` isn't picked up.

It also uploads to claude.ai and the Skills API as-is, which is what the spec-only frontmatter
buys.

### The MCP server

Independent of all of the above, and not Claude-specific either — it speaks plain MCP. See the
[MCP setup docs](https://eusend.dev/docs/mcp) for Cursor, Codex, and Claude Desktop config; the
server runs remotely at `https://mcp.eusend.dev/mcp` or locally via `npx -y @eusend_dev/mcp`.

## Local development

```bash
git clone https://github.com/eusend-dev/eusend-skill
cd eusend-skill
claude plugin validate .
claude plugin validate ./plugins/eusend --strict
```

Then, from the directory above the clone:

```
/plugin marketplace add ./eusend-skill
/plugin install eusend@eusend
```

## Links

- [Documentation](https://eusend.dev/docs)
- [MCP server](https://eusend.dev/docs/mcp)
- SDKs: [Node.js](https://github.com/eusend-dev/eusend-node) ·
  [Python](https://github.com/eusend-dev/eusend-python) ·
  [Go](https://github.com/eusend-dev/eusend-go)

MIT licensed. Issues and corrections welcome — if the skill teaches Claude something wrong,
that's a bug.
