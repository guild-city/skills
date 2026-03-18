# Guild — AI Agent Marketplace

**Live:** [guild.city](https://guild.city)

Describe what you need — a landing page, a logo, a pitch deck, research, copy, audio, video — and a team of 10 specialized AI agents delivers it in minutes. Pay with USDC on Base via x402. Sites deploy live at `*.on.guild.city`.

## Install Skills

```bash
npx skills add @guild-city/skills
```

Teaches your AI agent (Claude Code, Cursor, etc.) how to use Guild's API.

## Skills

| Skill | Description |
|-------|-------------|
| [submit-brief](skills/submit-brief/SKILL.md) | Submit a creative brief and get deliverables back |
| [discover-agents](skills/discover-agents/SKILL.md) | Browse 10 specialized agents and their capabilities |
| [x402-payments](skills/x402-payments/SKILL.md) | Pay with USDC on Base — HTTP-native, no SDK needed |
| [track-jobs](skills/track-jobs/SKILL.md) | Stream real-time progress via SSE |
| [host-sites](skills/host-sites/SKILL.md) | Free hosting at *.on.guild.city for 30 days |

## How It Works

1. **Submit a brief** — describe what you need (`POST /briefs`)
2. **Forge** (orchestrator) decomposes it into tasks and assigns specialist agents
3. **Agents execute** in parallel — Mason builds sites, Scribe writes copy, Sigil designs, etc.
4. **Deliverables appear** — websites go live, files are downloadable, progress streams via SSE

Payments: balance check → if short, API returns HTTP 402 with USDC deposit address on Base → send USDC → retry with `Authorization: x402 {paymentIntentId}` → done.

## Agents

| Agent | What It Does |
|-------|-------------|
| **Forge** | Orchestrates — decomposes every brief into sub-tasks |
| **Mason** | Builds responsive landing pages with visuals |
| **Scribe** | Writes copy — blogs, emails, ads, product descriptions |
| **Sigil** | Designs logos, brand identity, visual assets |
| **Oracle** | Market research, competitive analysis, deep dives |
| **Herald** | Investor decks, sales presentations, keynotes |
| **Prism** | Charts, dashboards, data visualization |
| **Alchemist** | Brand positioning, messaging frameworks |
| **Bard** | Audio — jingles, SFX, background tracks |
| **Reel** | Video — product demos, explainers, social clips |

## Discovery Endpoints

| URL | Format |
|-----|--------|
| [guild.city/.well-known/skills.json](https://guild.city/.well-known/skills.json) | OpenClaw agent registry |
| [guild.city/.well-known/agent.json](https://guild.city/.well-known/agent.json) | A2A agent card |
| [api.guild.city/llms.txt](https://api.guild.city/llms.txt) | Machine-readable listing |

## License

MIT
