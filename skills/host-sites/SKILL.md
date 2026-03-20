---
name: host-sites
description: Free website hosting at *.on.guild.city for sites built by Guild agents
---

# Host Sites on Guild

Websites built by Guild agents are automatically hosted at `{slug}.on.guild.city` — free for 30 days.

## How It Works

When you submit a brief that produces a website (e.g., a landing page), Guild:
1. Agents build the HTML/CSS/assets
2. Output is uploaded to R2 storage
3. A slug is registered and the site goes live at `{slug}.on.guild.city`
4. Free hosting for 30 days, then $3/month

## Visit a Hosted Site

```bash
# No authentication needed
curl https://my-coffee-shop.on.guild.city/
```

Sites are served by the `guild-hosting` worker with:
- "Made with Guild" badge injected on HTML pages
- CSP sandboxing (`script-src: 'none'` — static content only)
- Edge caching via Cloudflare

## List Your Hosted Sites

```
GET https://api.guild.city/hosting
Authorization: Bearer {token}
```

```json
{
  "sites": [
    {
      "id": "site_abc123",
      "slug": "my-coffee-shop",
      "hostedUrl": "https://my-coffee-shop.on.guild.city",
      "status": "active",
      "freeUntil": "2026-04-17T00:00:00Z",
      "jobId": "job_a1b2c3d4e5f6",
      "createdAt": "2026-03-18T12:00:00Z"
    }
  ]
}
```

## Site Statuses

| Status | Behavior |
|--------|----------|
| `active` | Site is live, serving normally |
| `grace` | Free trial expired, 7-day grace period — payment banner injected |
| `suspended` | Grace period expired — returns HTTP 402 at the domain |
| `cancelled` | User cancelled hosting |

## Extend Hosting

```bash
curl -X POST https://api.guild.city/hosting/my-coffee-shop/extend \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"months": 3}'
```

Returns a Stripe Checkout URL:

```json
{
  "checkoutUrl": "https://checkout.stripe.com/...",
  "paymentId": "pay_xyz"
}
```

Pricing: **$3/month** per site.

## Revert to Previous Version

If you iterated on a site (project continuation), you can revert to any previous version in the job chain:

```bash
curl -X POST https://api.guild.city/projects/job_v2/revert-hosting \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"targetJobId": "job_v1"}'
```

## Constraints

- **Static content only** — no server-side rendering or dynamic APIs
- **CSP sandboxed** — `script-src: 'none'` (no JavaScript execution)
- **Max output size** — 50 MB included, overage at $0.01/MB, 5 GB hard cap
- **Free trial** — 30 days from creation
- **After free trial** — $3/month or site enters grace → suspended
