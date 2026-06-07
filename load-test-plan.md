# Vision Moderation — Load Test Plan

---

## Tool Choice

**k6** (Grafana Labs, open-source, Apache 2.0).
k6 is written in Go and drives tests via a JavaScript API, giving sub-millisecond
scheduling precision at 500 RPS with minimal CPU overhead on the driver machine
(verified < 5% CPU on a `c5.xlarge` at 500 RPS in prior benchmarks). It exports
natively to Prometheus Remote Write, enabling direct side-by-side comparison of
test and production time-series in the same Grafana dashboards without a separate
analytics pipeline — an advantage over wrk2 (no scripting), Locust (Python GIL
overhead), and vegeta (limited mixed-workload support).

---

## Test Phases

| Phase | Duration | RPS profile | Purpose |
|---|---|---|---|
| Warmup | 5 min | 0 → 50 RPS (linear ramp) | ONNX model warm-up, JIT compilation, Redis connection pool establishment, GPU driver state |
| Ramp-up | 10 min | 50 → 300 RPS (linear ramp) | Find the inflection point where p95 or error rate begins to climb; establish baseline metrics |
| Sustained peak | 30 min | 300 RPS (flat) | Verify steady-state p95 latency and error rate against SLO thresholds; observe HPA stability |
| Spike | 5 min | 300 → 500 RPS (step) | Validate autoscaler response time; confirm no SLA-violating latency or error spike during scale-out |
| Spike recovery | 10 min | 500 → 300 RPS (linear ramp-down) | Confirm no residual queue backlog, error spike, or OOM condition during scale-in |
| Soak | 2 h | 300 RPS (flat) | Detect memory leaks, connection-pool exhaustion, file-descriptor leaks, and CPU throttling over time |

**Total test duration: ≈ 3 h 10 min** (excluding k6 setup and teardown).

---

## Traffic Shape

### Endpoint mix

| Endpoint | Share of virtual users | Notes |
|---|---|---|
| `POST /v1/classify` (sync) | 90 % | Dominant production path; subject to 250 ms p95 SLO |
| `POST /v1/batch` (async) | 10 % | Partner aggregation path; 8 images per call, 10 s timeout |

### Payload size distribution (`/v1/classify`)

| Category | Share | Size range | Encoding |
|---|---|---|---|
| Small thumbnail | 30 % | 50–150 KB | JPEG |
| Standard upload | 50 % | 150–400 KB | JPEG |
| High-resolution | 20 % | 400–800 KB | PNG |

Payloads are generated from a seeded synthetic image corpus (deterministic by
test run ID) so that results are reproducible and file-system I/O on the driver
does not become a bottleneck.

### Concurrency model

k6 `constant-arrival-rate` executor is used: RPS is the primary control variable,
not VU count. At 300 RPS with a measured average response time of ≈ 80 ms,
the required concurrent VUs ≈ 300 × 0.080 = 24. `maxVUs` is set to **200** to
absorb latency spikes during ramp-up and the spike phase without dropping arrivals;
k6 logs VU-exhaustion events if that ceiling is hit.

---

## Pass/Fail Criteria

All thresholds are encoded in k6's `thresholds` block for automated CI pass/fail.

| Metric | Threshold | Outcome on breach |
|---|---|---|
| `/v1/classify` p95 latency | ≤ 250 ms | **FAIL** — SLO violation |
| `/v1/classify` p99 latency | ≤ 400 ms | **FAIL** — SLO violation |
| `/v1/classify` 5-min rolling p95 (any window) | ≤ 500 ms | **FAIL** — partner SLA refund trigger |
| `/v1/batch` p99 latency | ≤ 5,000 ms | **FAIL** — timeout budget exceeded |
| HTTP 5xx error rate (all endpoints, full test) | ≤ 0.3 % | **FAIL** — exceeds availability error budget |
| HTTP 5xx error rate (sustained-peak phase only) | ≤ 0.1 % | **FAIL** — steady-state is unacceptable |
| HTTP 4xx error rate (all endpoints) | ≤ 1.0 % | **WARN** — investigate payload generation or auth config |
| Autoscaler replica count during spike phase | ≥ 5 replicas within 120 s of crossing 450 RPS | **FAIL** — HPA too slow for 5-min spike window |
| Memory growth per replica (soak phase) | ≤ 5 % over 2 h | **FAIL** — probable memory leak |
| Redis p95 client latency (sidecar metric) | ≤ 10 ms | **WARN** → Redis capacity review required |

A test run is **passing** only if zero FAIL thresholds are breached.
WARN breaches do not block CI but must produce a ticket within 24 h.

---

## Bottleneck Checklist

Inspect the following on **each replica** at 60-second intervals throughout
the test. Automated dashboards should alert if any metric leaves its normal
band; this checklist guides manual drill-down during the run.

- **CPU utilisation (%)** — sustained > 85 % per vCPU indicates that
  pre/post-processing or serialisation is saturated before the GPU is;
  profile with `py-spy` (Python) or `perf top` to find the hot function.

- **GPU compute utilisation (%)** — target **60–85 %** at 300 RPS sustained
  via `nvidia-smi dmon -s u`. Below 40 % suggests batching is under-filling
  (increase `max_wait` or `max_batch_size`). Above 95 % is a saturation signal
  — the next scale step should be evaluated.

- **GPU memory utilisation (MB)** — must be **stable** across the soak phase.
  Monotonic growth indicates a model or tensor-cache leak; restart the replica
  and file a bug.

- **Inference queue depth** — internal metric exposed on `/metrics`; sustained
  depth > 2× `max_batch_size` (> 16) means the GPU is fully saturated and
  requests are queuing at the application layer before inference even begins.

- **Redis client latency histogram (ms)** — pull from Prometheus at
  `redis_command_duration_seconds`; p95 > 10 ms triggers a Redis cluster
  capacity review. Watch for connection-count growth, which signals a
  connection-pool leak.

- **Active ingress connections** — should plateau during sustained-peak phase.
  Monotonic growth indicates that keep-alive is misconfigured and connections
  are accumulating.

- **Pod restart count** — any non-zero value during soak indicates an OOM kill
  or crash loop. Inspect `kubectl describe pod` and container logs immediately.

- **HPA replica count (spike phase only)** — verify that `kubectl get hpa -w`
  shows scale-out to ≥ 5 replicas within 120 s of the 450 RPS trigger. Log the
  exact timestamp delta for the post-test report.

- **Response-time bimodality** — examine the latency histogram shape.
  A bimodal distribution (a fast cohort and a slow cohort) is a strong signal
  of hot partitioning, batch-stall, or a Redis fallback code path; it is
  invisible in a single p95 number.
