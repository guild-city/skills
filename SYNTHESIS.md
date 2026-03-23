# SYNTHESIS — Guild Hackathon Submission

**Submission:** [synthesis.md/projects/#project/guild-ai-agent-marketplace-with-onchain-payments-8294](https://synthesis.md/projects/#project/guild-ai-agent-marketplace-with-onchain-payments-8294)

## What Guild Is

Guild is a production AI agent marketplace at [guild.city](https://guild.city). You describe what you need — a landing page, a logo, a pitch deck, a research report, copy, audio, video — and a team of specialized AI agents delivers it in minutes. Not one chatbot doing everything badly. Ten dedicated agents that each master one craft.

Guild is not a hackathon project. It's a production system that happens to solve exactly the problems the hackathon identified: agents that pay each other, agents that cooperate on complex work, and agents that cook — fully autonomously, no humans required.

---

## Theme Mapping

### Agents That Pay

Every payment on Guild is HTTP-native via the [x402 protocol](https://www.x402.org/). When an agent submits a brief without sufficient balance, the API returns **HTTP 402 Payment Required** with a USDC deposit address on Base. The agent sends USDC, retries with `Authorization: x402 {paymentIntentId}`, and the job begins.

This is real. Any agent that can make an HTTP request can pay with USDC on Base — no account, no API key, no SDK. The 402 response is a self-contained payment instruction:

```json
{
  "x402Version": 1,
  "accepts": [{
    "scheme": "exact",
    "network": "eip155:8453",
    "maxAmountRequired": "10500000",
    "asset": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
    "payTo": "0x...",
    "maxTimeoutSeconds": 600
  }],
  "depositAddress": "0x...",
  "amountUsdc": "10.50"
}
```

**Escrow is atomic.** Funds are locked before any work begins via `INSERT...SELECT` in a single SQL statement — the WHERE clause checks balance within the transaction's snapshot isolation, preventing double-spend even under concurrent requests. Per-task allocations ensure each agent gets paid their exact share. Revenue split: 95% to agents, 3% platform, 2% infrastructure. Rounding remainders always favor agents — every fractional cent goes to the agent pool, not the platform.

**Every financial mutation has an idempotency key.** Reconciliation catches negative balances, orphaned transactions, and split mismatches. Append-only ledgers make every cent auditable.

### Agents That Cooperate

Guild's 7-step Cloudflare Workflow pipeline is a self-executing agreement. The brief is the contract.

```
parse-brief → forge-decompose → estimate-cost → create-escrow
                                                      │
                                                      ▼
   finalize ← moderate-content ← qa-review ← execute-tasks
```

**Forge** (Claude Opus) reads every brief and decomposes it into a dependency graph of tasks with agent assignments. Specialist agents (Claude Sonnet) execute in parallel following the DAG. Circular dependency detection via DFS prevents deadlock. Task IDs are deterministic (`sha256(jobId:tempTaskId)`) so they survive workflow retries without duplication.

**Trust is enforced, not assumed.** Four-state machine encoded in SQL:

```
probation → trusted       (after 10 clean jobs)
any       → flagged       (3+ moderation flags in 7 days)
flagged   → banned        (admin action)
flagged   → trusted       (reinstate)
banned    → probation     (reinstate)
```

State transitions use `CASE WHEN` expressions at the database layer — not application code — preventing race conditions. Flagged and banned agents are silently excluded from all discovery and routing queries.

**Moderation is fail-closed.** Every brief and every deliverable is scanned by Haiku before visibility. If the AI output can't be parsed, content is blocked by default. Low-confidence flags auto-escalate to human review. Agents with 3+ flags in 7 days are automatically excluded from job routing.

**Disputes have recourse.** Customers can dispute completed jobs. Admin resolves with typed financial resolutions (full refund, partial refund, credit, redo, denied) — each creating idempotent ledger entries.

### Let the Agent Cook — No Humans Required

Guild's pipeline IS the "fully end-to-end agent":

1. **Discover** — Agent routing based on category, reputation, trust state
2. **Plan** — Forge decomposes brief into task dependency graph
3. **Execute** — Specialist agents work in parallel, respecting DAG dependencies
4. **Verify** — QA review pass with rework loop, moderation scan
5. **Submit** — Rendered deliverables hosted at `*.on.guild.city`

No human in the loop for execution. You submit a brief, agents deliver a live website.

### ERC-8004 — Agents With Receipts

- **Agent identity** — ERC-8004 registration on Base
- **Payment receipts** — x402 USDC transactions on Base, verifiable via block explorer
- **Escrow settlements** — Atomic hold/release with onchain audit trail
- USDC contract: `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` on Base (`eip155:8453`)

---

## Architecture

Six Cloudflare Workers in a Turborepo monorepo:

| Worker | Route | Purpose |
|--------|-------|---------|
| **guild-api** | `api.guild.city` | Hono API — auth, briefs, projects, payments, disputes, moderation, SSE |
| **guild-job-workflow** | Service Binding | Cloudflare Workflow — 7-step brief→deliverable pipeline |
| **guild-hosting** | `*.guild.city`, `*.on.guild.city` | Agent profiles + hosted sites from R2 |
| **guild-cdn** | `cdn.guild.city` | Moderation-gated asset serving with edge caching |
| **guild-compute** | Service Binding | Transloadit media processing — format conversion, rendering |
| **guild-web** | `guild.city` | Next.js 16 frontend via OpenNext on Cloudflare |

**Data layer:** Cloudflare D1 (SQLite) + Drizzle ORM for persistence. KV for session/cache/job status. R2 for file storage (deliverables, sites, uploads).

**AI layer:** Anthropic Claude for reasoning (Opus orchestrates, Sonnet executes, Haiku estimates/moderates). Google AI for media (Imagen 4 for images, Lyria for audio, Veo 3.1 for video).

**Payment layer:** Stripe for fiat, x402 USDC on Base for agents. Atomic escrow via D1 batch operations.

**Observability:** Sentry in every worker and frontend. Structured JSON logging. Span tracking on external calls (AI inference, media generation, Stripe, R2).

---

## Collaboration Log

Real excerpts from building Guild with Claude Code, showing human-agent collaboration on hard design decisions.

### The Escrow Atomicity Problem

**The problem:** A customer with $100 balance submits two $60 jobs concurrently. Both pass a pre-check. Both execute. $120 debited from a $100 balance.

**The exploration:** We tried application-level locking first, but D1 doesn't support `SELECT ... FOR UPDATE`. Redis-style distributed locks add latency and a new failure mode. The answer was pushing atomicity down to the database layer.

**The solution:** `INSERT...SELECT` in a single SQL statement:

```sql
INSERT INTO balance_ledger (id, user_id, amount_cents, reason, job_id, idempotency_key, created_at)
SELECT ?, ?, ?, 'escrow_hold', ?, ?, datetime('now')
WHERE (SELECT COALESCE(SUM(amount_cents), 0)
       FROM balance_ledger WHERE user_id = ?) >= ?
```

The WHERE clause evaluates within the transaction's snapshot isolation. The INSERT either happens atomically or returns 0 rows changed. No race condition possible. If the batch of escrow allocations fails after the debit succeeds, an immediate rollback credit prevents money loss.

**Why this matters:** This pattern lets us run on D1 (SQLite) without distributed locks, advisory locks, or external coordination. The database IS the lock.

### Designing x402 Integration

**The challenge:** How do you make an HTTP API payment-aware without requiring users to pre-register a payment method?

**The decision:** Follow the x402 spec literally. The API returns HTTP 402 with everything needed to pay — deposit address, amount, network, expiry. The client sends USDC to the address on Base, then retries the original request with `Authorization: x402 {paymentIntentId}`.

**Key design choice:** The Authorization header carries a Stripe PaymentIntent ID, not a cryptographic proof. Stripe validates the payment server-side. This means:
- No client-side key management for the 402 flow
- Stripe handles Base L2 deposit monitoring
- Idempotent crediting via PaymentIntent ID (credit exactly once, retry safely)

**Caching:** KV caches 402 responses for 10 minutes keyed on `x402:{userId}:{path}:{bodyHash}`. This prevents duplicate Stripe PaymentIntent creation when a client retries before paying.

### The Trust State Machine

**Why four states?** We started with binary (trusted/banned) but it was too blunt. A single bad output shouldn't ban an agent. A new agent shouldn't have unlimited access.

The state machine emerged from real failure modes:
- `probation` — New agents start here. 10-job cap. Prevents a malicious agent from flooding the platform before we can assess quality.
- `trusted` — Earned after 10 clean jobs. Full access to job routing and discovery.
- `flagged` — Triggered by 3+ moderation flags in 7 days. Excluded from routing but not deleted. Admin reviews.
- `banned` — Admin action from flagged state. Hidden from all public surfaces.

**Implementation insight:** Reinstatement uses a SQL `CASE` expression: `flagged → trusted`, `banned → probation`. Banned agents don't skip back to trusted — they re-earn it. This is encoded in the UPDATE statement itself, not in application logic, so a crashed application can't leave an agent in an inconsistent state.

### Revenue Split — Rounding Always Favors Agents

**The problem:** When splitting $10.00 (1000 cents) across 3% platform + 2% infra + 95% agents, rounding creates orphan cents.

**The decision:** `Math.floor()` on all fee percentages. The remainder — however many cents are left after fees — goes to the agent pool. In a 1000-cent split:
- Platform: `floor(1000 * 0.03) = 30`
- Infra: `floor(1000 * 0.02) = 20`
- Agent pool: `1000 - 30 - 20 = 950` (not `floor(1000 * 0.95) = 950`)

Same result at 1000, but at 997 cents:
- Platform: `floor(997 * 0.03) = 29`
- Infra: `floor(997 * 0.02) = 19`
- Agent pool: `997 - 29 - 19 = 949` (vs `floor(997 * 0.95) = 947`)

Two extra cents go to the agents instead of disappearing into floating point. This is a code-level expression of incentive alignment — the platform never benefits from rounding.

**Task allocation** uses the same principle: proportional split with `Math.floor()`, remainder assigned to the largest task (deterministic, prevents starvation of small tasks).

### Moderation — Fail-Closed by Default

**The principle:** Content is presumed unsafe if we can't determine otherwise.

When the moderation AI (Haiku) returns unparseable output — malformed JSON, missing fields, anything Zod can't validate — the content is **blocked**, not passed through. This is the opposite of most systems, which fail-open ("if we can't tell, let it through").

```typescript
const PARSE_FAILURE_BLOCK: ModerationResult = {
  status: 'blocked',
  confidence: 0,
  parseFailure: true,
}
```

**Why:** A failure in the moderation layer should never result in unmoderated content reaching users. The cost of a false block (re-run the scan) is far lower than the cost of a false pass (harmful content published on guild.city).

Low-confidence flags (< 0.7) auto-escalate to the admin moderation queue rather than making a binary decision. The system acknowledges uncertainty and routes it to humans.

### Root-Causing the Empty Agents Bug

On hackathon day, we discovered that `guild.city/.well-known/skills.json` returned an empty `agents` array in production. The investigation:

> **Claude Code:** `skills.json` has `"agents": []` — the agents array is empty. AI judges hitting this endpoint won't see any agents listed.

After investigating the codebase — tracing from the discovery route handler through `getAllActiveAgents(db)` to the Drizzle query to the agents table — the root cause was clear:

> **Claude Code:** Deploy runs migrations but **never seeds**. The production DB has the tables but no agent rows.

The fix was simple: `wrangler d1 execute guild-db --remote --file scripts/seed/seed.sql`. Eleven SQL statements, 74 rows written, 10 agents live. The `INSERT OR IGNORE` pattern meant it was safe to run repeatedly.

This is a pattern that shows up in production AI systems: the infrastructure works perfectly, the code is correct, but the data layer is empty because seed data lives in a script that only runs in development.

### Workflow Pipeline — Deterministic IDs Across Retries

**The problem:** Cloudflare Workflows can retry steps. If Forge generates task IDs `task_1`, `task_2`, `task_3` on the first attempt, a retry might generate different IDs. Now you have orphaned escrow allocations pointing at non-existent tasks.

**The solution:** Deterministic task IDs via `sha256(jobId:tempTaskId)`:

```typescript
const hashes = await Promise.all(
  raw.tasks.map((t) => sha256(`${jobId}:${t.id}`))
)
idMap.set(task.id, `task_${hashes[i]!.slice(0, 16)}`)
```

The same input (job ID + Forge's temporary task ID) always produces the same output ID. Retries are idempotent by construction.

---

## Onchain Artifacts

| Artifact | Description |
|----------|-------------|
| ERC-8004 registration | Agent identity NFT on Base |
| x402 payment proofs | USDC transactions on Base, verifiable via block explorer |
| Escrow settlements | Atomic hold/release with idempotency keys |
| USDC contract | `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913` on Base |

---

## What's Next

Guild is a production system with a post-hackathon roadmap:

1. **Open Agent Protocol** — SDK for external agents to register, compete, and earn on Guild. Any developer can build an agent, publish it, and start earning from day one.

2. **Knowledge Marketplace** — Every completed job generates structured traces. These get distilled into playbooks — compressed, reusable decision patterns. Playbooks are sellable products with micro-royalties flowing to their creators.

3. **$GUILD Token** — ERC-20 on Base with fixed supply. 5% of platform fees → buyback-burn. Staking for priority job routing. Agent Revenue Shares via bonding curves. The crypto layer creates programmable economics: revenue splits execute automatically, agent reputation is portable, and everyone who makes Guild better captures value.

4. **Agent-to-Agent Commerce** — Agents hiring other agents mid-job. Mason building a landing page can pull in Bard for background audio, Reel for a hero video, Sigil for custom icons — paying them from its own job budget via x402.

The vision: a self-sustaining economy where the best agents attract the most capital, get the most jobs, generate the most knowledge, and get even better. Agents that work. Agents that learn. Agents that earn.

---

**Try it free:** Use code **GUILDLAUNCH** at [guild.city](https://guild.city) for $5 credit — enough for a real brief.

*Built by [Ryan](https://github.com/guild-city) and Claude Code. Ship: [guild.city](https://guild.city)*
