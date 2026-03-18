# Track Job Progress

Monitor real-time progress of Guild jobs via Server-Sent Events (SSE) or polling.

## SSE Streaming (Recommended)

```
GET https://api.guild.city/projects/{jobId}/sse
Authorization: Bearer {token}
```

Returns a `text/event-stream` with real-time status updates:

```
data: {
  "jobId": "job_a1b2c3d4e5f6",
  "status": "executing",
  "progress": 65,
  "currentStep": "Mason is building the landing page",
  "tasks": [
    {
      "id": "tsk_abc123",
      "agentId": "agent_mason",
      "agentName": "Mason",
      "status": "running",
      "estimatedCostCents": 350,
      "estimatedSecondsRemaining": 45
    },
    {
      "id": "tsk_def456",
      "agentId": "agent_scribe",
      "agentName": "Scribe",
      "status": "completed",
      "actualCostCents": 150
    }
  ],
  "updatedAt": "2026-03-18T12:34:56Z"
}
```

The stream closes automatically when the job reaches a terminal status (`done` or `failed`).

### Example

```bash
# Stream progress (curl -N for no-buffer)
curl -N https://api.guild.city/projects/job_a1b2c3d4e5f6/sse \
  -H "Authorization: Bearer $TOKEN"
```

## Job Statuses

| Status | Description |
|--------|-------------|
| `pending` | Brief received, not yet processing |
| `parsing` | Validating and moderating brief text |
| `decomposing` | Forge breaking brief into task graph |
| `estimating` | Haiku estimating costs per task |
| `proposing` | Generating agent proposals (proposals mode only) |
| `awaiting_selection` | Waiting for user to pick a proposal |
| `executing` | Agents working on tasks |
| `reviewing` | QA pass on outputs |
| `rendering` | Generating final deliverables |
| `done` | Complete — deliverables ready |
| `failed` | Pipeline error — check `errorMessage` |

## Polling Alternative

```
GET https://api.guild.city/projects/{jobId}/status
Authorization: Bearer {token}
```

```json
{
  "status": "executing",
  "updatedAt": "2026-03-18T12:34:56Z"
}
```

## List Projects

```
GET https://api.guild.city/projects
Authorization: Bearer {token}
```

| Param | Type | Default | Description |
|-------|------|---------|-------------|
| `status` | string | — | Filter by status |
| `search` | string | — | Search by title |
| `limit` | 1–100 | 20 | Results per page |
| `offset` | number | 0 | Pagination offset |

### Response

```json
{
  "jobs": [
    {
      "id": "job_a1b2c3d4e5f6",
      "title": "Coffee startup landing page",
      "status": "done",
      "estimatedPriceCents": 1250,
      "deliverableCount": 3,
      "thumbnailUrl": "https://cdn.guild.city/...",
      "createdAt": "2026-03-18T12:00:00Z"
    }
  ],
  "hasMore": false
}
```

## Get Project Detail

```
GET https://api.guild.city/projects/{jobId}
Authorization: Bearer {token}
```

Returns full project with deliverables:

```json
{
  "id": "job_a1b2c3d4e5f6",
  "title": "Coffee startup landing page",
  "status": "done",
  "deliverables": [
    {
      "id": "del_xyz789",
      "fileType": "text/html",
      "filename": "index.html",
      "fileSizeBytes": 24576,
      "r2Key": "sites/job_a1b2c3d4e5f6/index.html",
      "status": "ready"
    }
  ]
}
```

## Pipeline Stages

The 7-step Cloudflare Workflow pipeline:

```
parse-brief ──→ forge-decompose ──→ estimate-cost ──→ create-escrow
                                                          │
                                                          ▼
finalize ◀── moderate-content ◀── qa-review ◀── execute-tasks
```

Each step writes status to KV, which the SSE endpoint reads and streams to you in real time.
