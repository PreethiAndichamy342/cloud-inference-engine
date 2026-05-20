# Architecture — HIPAA Inference Routing & Multi-Cloud Placement Engine

This document describes the system design of the **HIPAA-compliant inference routing** engine: how requests flow through the PHI gate, how the multi-cloud placement engine selects a destination, and how SLA enforcement prevents latency violations.

---

## System Diagram

![End-to-End Architecture](docs/architecture-with-sla.png)

*Complete PHI gate architecture showing policy filtering, SLA-aware routing with p99 latency, and multi-cloud execution across AWS, GCP, and on-premises environments.*

---

## End-to-End Data Flow

Each inference request travels through eight stages:

1. **Client submits request** — carries `model_id`, `payload`, `tenant_id`, `data_sensitivity`, and optional `strategy` / `max_latency_ms`.
2. **FastAPI receives request** (`src/api/main.py`) — validates the schema and passes it to the PlacementRouter.
3. **PHI Gate runs** — the sensitivity tier is read (`public` → `phi_strict`). If the payload contains PHI entities, they are detected and the tier is enforced before any cloud is contacted.
4. **PolicyEngine filters servers** (`src/engine/router.py`) — eliminates every server that fails model-support, sensitivity-clearance, BAA, or on-prem-only checks. This is Phase 1 of the three-phase routing model.
5. **SLA filter applied** — if `max_latency_ms` is set, any server whose rolling p99 exceeds that ceiling is dropped from the candidate list.
6. **Scoring selects winner** — remaining candidates are ranked by the chosen strategy (Phase 2). The top-ranked server is selected.
7. **Request dispatched to cloud adapter** (`src/clouds/`) — the appropriate adapter forwards the payload to the target inference endpoint and records the round-trip latency.
8. **Decision logged** — `request_id`, `selected_server`, `routing_latency_ms`, `phi_entities_detected`, and metadata are written to the in-memory log. No payload or PHI text is ever stored.

---

## System Components

```
Client
  │
  ▼
FastAPI app  (src/api/main.py)
  │
  ├── PlacementRouter  (src/engine/router.py)
  │     ├── PolicyEngine   — compliance + model-support filtering
  │     └── Scoring        — least-loaded / latency / cost / round-robin
  │
  ├── HealthWatcher  (src/engine/health.py)
  │     └── Background threads poll each adapter's health endpoint
  │
  └── Cloud adapters  (src/clouds/)
        ├── OnPremAdapter   — vLLM OpenAI-compatible API
        ├── OllamaAdapter   — Ollama (uses /api/tags for health)
        ├── CloudAdapter    — abstract base
        ├── AwsAdapter      — AWS-specific adapter
        └── GcpAdapter      — GCP-specific adapter
```

| Component | File | Responsibility |
|-----------|------|----------------|
| FastAPI app | `src/api/main.py` | HTTP layer, request validation, dashboard |
| PlacementRouter | `src/engine/router.py` | Orchestrates policy filtering and scoring |
| PolicyEngine | `src/engine/router.py` | Phase 1 compliance gate |
| HealthWatcher | `src/engine/health.py` | Background polling, circuit breaker state |
| Cloud adapters | `src/clouds/` | Per-environment inference dispatch |
| PHIVault | `src/engine/` | Secure entity map storage (token → original) |
| Redis cache | optional | Routing result cache for non-PHI requests |

---

## Three-Phase Routing Model

The **multi-cloud placement engine** decides where to send each request in three phases. Every phase must pass before the next runs.

### Phase 1 — Compliance Filtering (PHI gate architecture)

The `PolicyEngine` eliminates servers that fail any of these checks. This phase always runs first, regardless of strategy.

| Check | Rule |
|-------|------|
| Model support | Server must list the requested `model_id` in `supported_models` |
| Sensitivity clearance | Server's `max_sensitivity` must be ≥ request's `data_sensitivity` |
| BAA requirement | `sensitive`, `phi`, or `phi_strict` data requires `has_baa=True` |
| On-prem only | `phi_strict` requests are restricted to `cloud_env=ON_PREM` servers |

If no server passes all four checks, the request is rejected immediately with HTTP 503 — no inference is attempted.

### Phase 2 — SLA Filtering

If `max_latency_ms` is provided, any server whose rolling p99 latency (measured over the last 100 requests) exceeds the ceiling is removed from the candidate list. If no server survives, the request is rejected with the best available p99 reported back so the caller knows how close they are.

### Phase 3 — Strategy Scoring

Servers that survive Phases 1 and 2 are ranked by the chosen strategy:

| Strategy | Ranking criterion |
|----------|-------------------|
| `compliance_first` | Highest `max_sensitivity` clearance wins |
| `least_loaded` | Lowest `current_load` wins |
| `latency_optimized` | Lowest `p99_latency_ms` wins |
| `cost_optimized` | Lowest `cost_per_token` wins |
| `round_robin` | Cycles through eligible servers in registration order |

### End-to-end example — PHI_STRICT request

```
Request: data_sensitivity=phi_strict, strategy=compliance_first

Phase 1 — Compliance filtering:
  aws-sim  → REJECTED (max_sensitivity=INTERNAL, has_baa=False, cloud_env=AWS)
  gcp-sim  → REJECTED (max_sensitivity=INTERNAL, has_baa=False, cloud_env=GCP)
  on-prem  → PASSES  (max_sensitivity=PHI_STRICT, has_baa=True, cloud_env=ON_PREM)

Phase 2 — SLA filtering:
  on-prem  → PASSES (no max_latency_ms set)

Phase 3 — Scoring:
  on-prem  → selected (only eligible candidate)

Result: routed to on-prem. PHI never leaves the compliant environment.
```

---

## Demo Topology

`scripts/start_demo.sh` starts three local Ollama processes that simulate a real multi-cloud environment:

| Simulated env | Port  | Model            | max_sensitivity | has_baa |
|---------------|-------|------------------|-----------------|---------|
| aws-sim       | 11434 | tinyllama:latest | INTERNAL        | false   |
| gcp-sim       | 11435 | tinyllama:latest | INTERNAL        | false   |
| on-prem       | 11436 | tinyllama:latest | PHI_STRICT      | true    |

This topology is intentionally minimal: `aws-sim` and `gcp-sim` can handle public and internal data, while `on-prem` is the only BAA-signed environment that accepts PHI. Any `phi_strict` request will always land on `on-prem` regardless of load or latency, demonstrating the compliance-first guarantee of the **HIPAA inference routing** model.

---

## Circuit Breaker

Each adapter has an independent circuit breaker with three states:

| State | Behaviour |
|-------|-----------|
| `CLOSED` | Normal operation — requests forwarded |
| `OPEN` | Fast-fail — requests rejected without contacting the server |
| `HALF_OPEN` | Probing — one request allowed through to test recovery |

The breaker opens after three consecutive failures and resets after a successful probe. This prevents a degraded server from inflating p99 latency measurements that feed back into the SLA filter.

---

## Related Articles

| Topic | Article |
|-------|---------|
| PHI gate architecture and sensitivity tiers | [PHI Gate](https://preethiandichamy.medium.com/healthcare-call-center-ai-the-phi-gate-that-decides-what-leaves-and-what-never-should-95ed2dcb52fd) |
| p99 latency and SLA-aware server filtering | [P99 Latency](https://preethiandichamy.medium.com/route-to-cheapest-then-fastest-both-failed-here-is-what-was-missed-904d5d7957f6) |
| Multi-cloud placement engine design | [Placement Engine](https://preethiandichamy.medium.com/episode-2-the-placement-engine-e4ece0bad3b0) |
| Full system overview | [System Overview](https://preethiandichamy.medium.com/healthcare-call-center-ai-designing-a-multi-cloud-inference-architecture-where-phi-never-moves-d997bdb3ed1f) |
| Unified compliance view | [Unified Compliance](https://preethiandichamy.medium.com/healthcare-call-center-ai-one-compliance-view-across-four-environments-that-never-stop-moving-e03eff5d955e) |
