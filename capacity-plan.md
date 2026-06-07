# Vision Moderation — Capacity Plan

---

## 1. Latency Budget Breakdown

Target: **p95 ≤ 250 ms**, synchronous `/v1/classify` endpoint, T4 GPU inference path.

| Stage | Budget (ms) | Notes |
|---|---:|---|
| Network in | 5 | TCP receive; TLS overhead amortised across keep-alive connections |
| Auth + routing | 3 | JWT signature verify ~1 ms; load-balancer routing ~2 ms |
| Payload parse | 4 | JSON decode + image header validation |
| Feature lookup (Redis) | 8 | p95 observed; runs concurrently with pre-processing where the async client allows |
| Pre-processing | 10 | Resize, normalise (half of the 15 ms combined pre+post budget) |
| Model inference (T4 GPU) | 35 | p95 ≈ 1.6× the 22 ms median; accounts for GPU scheduling jitter and batch-assembly variance |
| Post-processing | 5 | Threshold application, label mapping, confidence score rounding |
| Serialisation | 3 | JSON-encode response body (~0.5 KB) |
| Network out | 5 | TCP send; small response fits in one MTU |
| **Headroom** | **172** | Positive; absorbs Redis tail spikes, 5 ms batch-wait window, and minor phase variance |
| **Total** | **250** | |

**Sum check:** 5 + 3 + 4 + 8 + 10 + 35 + 5 + 3 + 5 = **78 ms**.
Headroom = 250 − 78 = **172 ms**. ✓

---

## 2. CPU vs GPU Decision

**Decision: T4 GPU — `g4dn.xlarge`**

At 22 ms median inference, one T4 processes ≈ 45 single-item inferences per second; with dynamic batching
(batch = 8, see §4) GPU throughput rises to 8 / 0.022 ≈ **364 req/s from the GPU alone**.
Accounting for CPU-side preprocessing across 3 available vCPUs (3 × 1/0.010 = 300 req/s)
and async Redis overlap, a conservative **120 RPS per replica** is achievable
while consuming only 78 ms of the 250 ms p95 budget — leaving **172 ms of headroom**.

A CPU-only alternative (`c5.2xlarge`, 8 vCPUs, ≈ $0.340/hr on-demand, AWS EC2
us-east-1 pricing¹) yields ~50 RPS per replica at 75 ms median inference and a
**57 ms p95 headroom** — 3× thinner. Meeting 300 RPS sustained requires 8 CPU
replicas at **≈ $1,960/month**. The GPU path (`g4dn.xlarge`, ≈ $0.526/hr¹) needs
only 4 replicas at **≈ $1,516/month** — 23% cheaper and with far more latency
margin. The GPU wins on both cost and reliability; CPU is ruled out.

> ¹ Source: [AWS EC2 On-Demand Pricing](https://aws.amazon.com/ec2/pricing/on-demand/), us-east-1,
> Linux, June 2025. Prices rounded to three decimal places; within ±30% of actual.

---

## 3. Replica Sizing

**Instance type:** `g4dn.xlarge` — 4 vCPUs, 16 GB RAM, 1 × T4 GPU (16 GB VRAM).
The 180 MB ONNX model loads into VRAM with ample room for batch activations.

| Scenario | Target RPS | Per-replica RPS | Min replicas (⌈target / per-replica⌉) | + 30% headroom (⌈min × 1.3⌉) | Monthly cost |
|---|---:|---:|---:|---:|---:|
| Sustained | 300 | 120 | 3 | **4** | **≈ $1,516** |
| Spike (5 min) | 500 | 120 | 5 | **7** | ≈ $2,653 |

Both figures are well within the **$4,000/month** compute budget; sustained cost is
38% of the envelope, leaving room for Redis, ingress, and monitoring overhead.

**Spike strategy: horizontal autoscaling (HPA).**
The 5-minute spike window is long enough for Kubernetes HPA to scale from
4 → 7 replicas: trigger fires at 70% GPU utilisation per replica (~84 RPS);
scale-out completes in ≈ 60–90 s. One additional replica is pre-warmed when
current RPS exceeds 75% of the scale-out threshold to reduce reaction lag.
No request shedding is required to handle the 500 RPS spike.

---

## 4. Batching Decision

**Dynamic batching is enabled for both endpoints, with separate parameters per path.**

For the synchronous `/v1/classify` endpoint (90% of traffic), the configuration is
**max\_batch\_size = 8, max\_wait = 5 ms**. The 172 ms of headroom in the latency
budget comfortably absorbs the worst-case 5 ms wait: even a request that arrives
just after a batch dispatch and waits the full window reaches the GPU at 83 ms,
leaving 167 ms of slack. The throughput benefit is material: packing 8 requests
into one GPU forward pass reduces per-request GPU occupancy from 22 ms to
≈ 2.75 ms, roughly tripling per-replica capacity without additional hardware.
A max batch size of 8 was chosen because the T4's 16 GB of VRAM easily
accommodates the 180 MB model plus eight sets of intermediate activations
with room to spare; larger batches yield diminishing throughput gains and
increase the risk of inference-time spikes from VRAM pressure.

For the asynchronous `/v1/batch` endpoint (10% of traffic, called by partners
aggregating uploads), the configuration is **max\_batch\_size = 32, max\_wait = 50 ms**.
There is no p95 latency SLO on this path beyond an overall request timeout of
10 s, so deeper batching maximises GPU utilisation without risk of violating
the partner SLA. The 50 ms wait window ensures that a large partner upload
is not split across many small GPU batches, which would reduce throughput
and increase per-image cost at peak GPU load.
