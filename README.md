# Terlido skills for AI agents

Two skills that teach an AI agent how to run [Terlido](https://terlido.com/agents?utm_source=skills&utm_medium=github&utm_campaign=terlido-skills) through its MCP server: cold email from your own mailbox, and newsletters.

An agent that only sees the tool list can call one tool. It does not know the right setup order, or why a campaign says "running" and sends nothing. These skills put that knowledge in one place the agent loads first.

| Skill | What it covers |
|---|---|
| [`terlido-outreach`](skills/terlido-outreach/SKILL.md) | Mailboxes, warmup, DNS, contacts, lists, segments, campaigns, the sequence format, schedules, senders, guarded enrollment and launch, the review queue, inbox replies, and a table for "running but not sending". |
| [`terlido-newsletter`](skills/terlido-newsletter/SKILL.md) | Publications, the sending mailbox, consent, imports, lists, segments, broadcasts (prepare, test, schedule), content rules, unsubscribes and the blocklist, automations, webhooks, and a table for diagnosing sends. |

Each skill gives the setup order, says which steps only the web app can do, and lists the actions that need your yes first.

## Install

With the `skills` CLI:

```bash
npx skills add nichovski/terlido-skills
```

Or by hand: copy a folder from `skills/` into your agent's skills folder (for Claude Code that is `~/.claude/skills/`).

## Connect your agent to Terlido

1. Add the MCP server to your agent: `https://terlido.com/api/mcp`
2. Sign in and approve the scopes on the consent screen. Terlido uses OAuth 2.1, so there is no API key to create.
3. Ask your agent to load the skill before it starts.

You need a Terlido workspace where you are an owner or admin, on a plan with API access or in the 14-day trial. See [Connect AI agents](https://help.terlido.com/docs/agents/connect).

## A first task to try

After you connect one mailbox in the web app, give your agent this:

```
Use the terlido-outreach skill. I have a CSV with 20 leads and this offer: <one line>.
Check my mailbox and DNS first and tell me what is missing.
Then draft a two-step sequence, show it to me, and wait for my yes before you enroll or launch anything.
```

The agent sends inside the same limits as a person. There is no separate send path for agents. Agent calls are recorded, and you can see and revoke each connected agent in the web app. See [Agent audit and send budget](https://help.terlido.com/docs/agents/security).

## Links

- [Terlido for agents](https://terlido.com/agents?utm_source=skills&utm_medium=github&utm_campaign=terlido-skills)
- [MCP tools reference](https://help.terlido.com/docs/agents/mcp-tools)
- Questions: hi@terlido.com

## Licence

MIT. See [LICENSE](LICENSE).
