# inference-placement-engine

**HIPAA-compliant inference routing** and **multi-cloud ML placement** engine — policy-driven server selection across AWS, GCP, and on-prem with PHI gate enforcement and p99 SLA guarantees.

---

![End-to-End Architecture](docs/architecture-with-sla.png)

---

## Problem

Healthcare ML workloads span a spectrum of data sensitivity. A de-identified risk score can run on any public cloud, but a request containing full PHI must never leave a HIPAA-compliant environment with a signed BAA. This engine makes placement automatic and policy-driven — compliance rules are enforced at the router before any cloud is contacted.

---

## Key Features

- **PHI Gate** — detects sensitivity tiers (`public` → `phi_strict`) and blocks non-compliant destinations before dispatch
- **Three-Phase Routing** — compliance filtering → SLA filtering → strategy scoring, in that order, every time
- **P99 Latency Enforcement** — rolling p99 per server; requests with `max_latency_ms` reject servers that exceed the ceiling
- **Circuit Breakers** — per-adapter `CLOSED/OPEN/HALF_OPEN` state prevents degraded servers from inflating latency metrics
- **Live Dashboard** — real-time view of routed requests, server health, and routing decisions at `/dashboard`

**Stack:** FastAPI · Python 3.11 · Ollama · Redis (optional)

---

## Quick Start

```bash
git clone https://github.com/PreethiAndichamy342/inference-placement-engine.git
cd inference-placement-engine
pip install -r requirements.txt && ollama pull tinyllama
bash scripts/start_demo.sh
uvicorn src.api.main:app --reload --port 8000
```

Then open **http://localhost:8000/dashboard** or test the compliance-first **healthcare ML routing** API:

```bash
curl -X POST http://localhost:8000/route \
  -H "Content-Type: application/json" \
  -d '{"model_id":"tinyllama:latest","payload":{"prompt":"Patient DOB 1980"},"tenant_id":"hospital_A","data_sensitivity":"phi_strict","strategy":"compliance_first"}'
```

---

## Documentation

| Document | Description |
|----------|-------------|
| [ARCHITECTURE.md](ARCHITECTURE.md) | System components, 8-step data flow, three-phase routing model, PHI-aware placement design |
| [SETUP.md](SETUP.md) | Prerequisites and installation for macOS, Linux, Windows, Chromebook |
| [API_REFERENCE.md](API_REFERENCE.md) | All endpoints with curl examples for the compliance-first routing API |
| [CODE_OF_CONDUCT.md](CODE_OF_CONDUCT.md) | Contributor standards and enforcement |

---

## Medium Series

1. [System Overview](https://preethiandichamy.medium.com/healthcare-call-center-ai-designing-a-multi-cloud-inference-architecture-where-phi-never-moves-d997bdb3ed1f)
2. [Placement Engine](https://preethiandichamy.medium.com/episode-2-the-placement-engine-e4ece0bad3b0)
3. [P99 Latency](https://preethiandichamy.medium.com/route-to-cheapest-then-fastest-both-failed-here-is-what-was-missed-904d5d7957f6)
4. [PHI Gate](https://preethiandichamy.medium.com/healthcare-call-center-ai-the-phi-gate-that-decides-what-leaves-and-what-never-should-95ed2dcb52fd)
5. [Kafka Failover](https://preethiandichamy.medium.com/healthcare-call-center-ai-the-message-queue-that-fails-over-without-anyone-noticing-c5afaa0dc1f7)
6. [Unified Compliance](https://preethiandichamy.medium.com/healthcare-call-center-ai-one-compliance-view-across-four-environments-that-never-stop-moving-e03eff5d955e)
7. [Design Retrospective](https://preethiandichamy.medium.com/healthcare-call-center-ai-what-we-would-build-differently-today-16340566eec3)

---

© 2026 Preethi Andichamy — Apache License 2.0
