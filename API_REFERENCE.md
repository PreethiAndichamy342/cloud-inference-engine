# API Reference — Healthcare ML API & Compliance-Aware Routing

This document is the complete reference for the **healthcare ML API** exposed by the inference placement engine. Every endpoint is designed around a **compliance-aware routing API** model: HIPAA rules are enforced at the routing layer before any inference request reaches a cloud backend.

Interactive docs are available at **http://localhost:8000/docs** when the server is running.

---

## Endpoints

- [GET /health](#get-health)
- [GET /metrics](#get-metrics)
- [POST /route](#post-route)
- [POST /de-identify](#post-de-identify)
- [GET /logs](#get-logs)
- [GET /logs/stats](#get-logsstats)
- [GET /circuit-status](#get-circuit-status)
- [GET /test-prompts](#get-test-prompts)
- [GET /health-check/{server_id}](#get-health-checkserver_id)
- [GET /server-logs/{server_id}](#get-server-logsserver_id)
- [POST /force-health-poll/{server_id}](#post-force-health-pollserver_id)

---

## GET /health

Returns app liveness and the count of healthy servers.

```bash
curl -s http://localhost:8000/health | python3 -m json.tool
```

**Response:**

```json
{
  "status": "ok",
  "healthy_server_count": 3,
  "total_server_count": 3,
  "checked_at": "2026-05-05T16:55:55Z"
}
```

Returns `status: degraded` (still HTTP 200) when no servers are healthy, so load-balancer health checks don't immediately pull the instance.

---

## GET /metrics

Returns a snapshot of load, latency, cost, and status for every registered server.

```bash
curl http://localhost:8000/metrics
```

**Response:**

```json
{
  "servers": [
    {
      "server_id": "on-prem-01",
      "cloud_env": "on_prem",
      "region": "local",
      "status": "healthy",
      "current_load": 0.0,
      "p99_latency_ms": 0.0,
      "cost_per_token": 0.0,
      "gpu_count": 1,
      "gpu_type": "A100"
    }
  ],
  "collected_at": "2026-05-05T16:55:55Z"
}
```

---

## POST /route

Routes an inference request to the best eligible server. Compliance filtering always runs first; servers that fail the PHI gate are never contacted.

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model_id` | string | yes | Model to invoke, e.g. `"tinyllama:latest"` |
| `payload` | object | yes | Input passed verbatim to the inference server |
| `tenant_id` | string | yes | Identifier of the requesting organisation |
| `data_sensitivity` | string | no | `public` / `internal` / `sensitive` / `phi` / `phi_strict` (default: `internal`) |
| `strategy` | string | no | `compliance_first` / `least_loaded` / `latency_optimized` / `cost_optimized` / `round_robin` (default: `compliance_first`) |
| `task_type` | string | no | `general` / `clinical_nlp` / `medical_imaging` / `risk_scoring` / etc. |
| `max_latency_ms` | float | no | Soft SLA ceiling — servers with p99 above this are excluded |
| `priority` | int | no | 1–10, higher = more important (default: 5) |
| `region_hint` | string | no | Preferred cloud region hint |
| `metadata` | object | no | Arbitrary key-value metadata passed through to the log |

**Example — public request:**

```bash
curl -X POST http://localhost:8000/route \
  -H "Content-Type: application/json" \
  -d '{
    "model_id": "tinyllama:latest",
    "payload": {"prompt": "What is diabetes?"},
    "tenant_id": "hospital_A",
    "data_sensitivity": "public",
    "strategy": "least_loaded"
  }'
```

**Example — PHI request:**

```bash
curl -X POST http://localhost:8000/route \
  -H "Content-Type: application/json" \
  -d '{
    "model_id": "tinyllama:latest",
    "payload": {"prompt": "Patient John DOB 1980 has hypertension"},
    "tenant_id": "hospital_A",
    "data_sensitivity": "phi_strict",
    "strategy": "compliance_first"
  }'
```

**Example — high-priority stat request with SLA ceiling:**

```bash
curl -X POST http://localhost:8000/route \
  -H "Content-Type: application/json" \
  -d '{
    "model_id": "tinyllama:latest",
    "payload": {"prompt": "Rapid sepsis risk score for ICU patient"},
    "tenant_id": "hospital_A",
    "data_sensitivity": "sensitive",
    "strategy": "latency_optimized",
    "priority": 9,
    "max_latency_ms": 500
  }'
```

The router runs compliance filtering first, then drops any server whose `p99_latency_ms` exceeds `max_latency_ms`. If no server survives the SLA cut, the request is rejected with HTTP 503 and a message showing the best available p99.

**Response:**

```json
{
  "request_id": "7de10d69-...",
  "rejected": false,
  "strategy_used": "compliance_first",
  "selected_server": {
    "server_id": "on-prem-01",
    "cloud_env": "on_prem",
    "region": "local",
    "endpoint": "http://localhost:11436/v1/completions",
    "status": "healthy"
  },
  "candidate_count": 1,
  "score_breakdown": {
    "on-prem-01": {"current_load": 0.0, "p99_latency_ms": 0.0, "cost_per_token": 0.0, "gpu_count": 1.0}
  },
  "routing_latency_ms": 0.156,
  "phi_entities_detected": 0,
  "decided_at": "2026-05-05T16:55:55Z"
}
```

---

## POST /de-identify

De-identifies free text and returns the anonymised version with a per-type entity breakdown. The entity map (token → original value) is **not** returned — it is stored securely in PHIVault keyed by `request_id` when routing via `POST /route`.

**Request body:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `text` | string | yes | Free text to de-identify |

**Example:**

```bash
curl -X POST http://localhost:8000/de-identify \
  -H "Content-Type: application/json" \
  -d '{"text": "Patient Jane Smith, DOB 1985-03-12, SSN 123-45-6789"}'
```

**Response:**

```json
{
  "anonymized_text": "Patient <PERSON>, DOB <DATE>, SSN <US_SSN>",
  "entity_count": 3,
  "entities_by_type": {"PERSON": 1, "DATE": 1, "US_SSN": 1}
}
```

---

## GET /logs

Queries the in-memory routing decision log (last 500 entries). No payload or PHI text is ever stored.

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `sensitivity` | string | Filter by `data_sensitivity` tier |
| `tenant_id` | string | Filter by exact tenant ID |
| `cloud_env` | string | Filter by cloud environment |
| `rejected` | boolean | `true` = rejected only, `false` = accepted only |
| `limit` | int | Max entries to return, 1–500 (default: 50) |
| `search` | string | Case-insensitive match on `request_id` or `tenant_id` |

**Example:**

```bash
curl "http://localhost:8000/logs?sensitivity=phi_strict&limit=10"
```

---

## GET /logs/stats

Returns aggregated counts from the routing log grouped by `data_sensitivity` and `cloud_env`.

```bash
curl http://localhost:8000/logs/stats
```

**Response:**

```json
{
  "total": 42,
  "rejected_count": 2,
  "by_sensitivity": {"public": 18, "phi_strict": 12, "internal": 12},
  "by_cloud_env": {"on_prem": 14, "aws": 16, "gcp": 12}
}
```

---

## GET /circuit-status

Returns the circuit breaker state for all registered servers: `CLOSED` (normal), `OPEN` (fast-failing), or `HALF_OPEN` (probing).

```bash
curl http://localhost:8000/circuit-status
```

**Response:**

```json
{
  "servers": [
    {
      "server_id": "aws-sim",
      "state": "CLOSED",
      "consecutive_failures": 0,
      "failure_threshold": 3,
      "last_failure_time": null
    }
  ],
  "collected_at": "2026-05-05T16:55:55Z"
}
```

---

## GET /test-prompts

Returns fabricated sample prompts for each sensitivity tier, suitable for exercising the de-identification pipeline and routing logic from the dashboard. All PHI in `phi` and `phi_strict` tiers is entirely fabricated.

```bash
curl http://localhost:8000/test-prompts
```

---

## GET /health-check/{server_id}

Directly probes a server's health endpoint and returns the result with timing. Does **not** update the server's persisted status.

```bash
curl http://localhost:8000/health-check/aws-sim
```

**Response:**

```json
{
  "server_id": "aws-sim",
  "status": "healthy",
  "latency_ms": 4.231,
  "error": null,
  "checked_at": "2026-05-05T16:55:55Z"
}
```

---

## GET /server-logs/{server_id}

Returns routing log entries that were dispatched to a specific server, newest first.

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `limit` | int | Max entries to return, 1–200 (default: 20) |

```bash
curl "http://localhost:8000/server-logs/on-prem-01?limit=5"
```

---

## POST /force-health-poll/{server_id}

Triggers an immediate health check and updates the server's persisted status. Use this to force a recovery check without waiting for the next background poll interval (default: 30 s).

```bash
curl -X POST http://localhost:8000/force-health-poll/gcp-sim
```

**Response:**

```json
{
  "server_id": "gcp-sim",
  "previous_status": "unavailable",
  "new_status": "healthy",
  "latency_ms": 3.812,
  "polled_at": "2026-05-05T16:55:55Z"
}
```
