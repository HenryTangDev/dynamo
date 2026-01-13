---
marp: true
theme: default
paginate: true
backgroundColor: #fff
---

<!--
To convert to PowerPoint:
1. Using Marp CLI: marp SLIDES.md --pptx
2. Using pandoc: pandoc SLIDES.md -o SLIDES.pptx
3. Using reveal.js: https://revealjs.com/
-->

# NVIDIA Dynamo
## High-Throughput Distributed LLM Inference

**A Technical Deep Dive**

---

# Agenda

1. Overview & Problem Statement
2. System Architecture
3. Core Components
4. KV-Aware Routing
5. Disaggregated Serving
6. Performance & Scalability
7. Future Roadmap

---

# Problem Statement

## Challenges in LLM Inference at Scale

- 🔥 **Memory Constraints**: Models exceed single GPU capacity
- ⚡ **Latency Requirements**: Sub-100ms TTFT for interactive workloads
- 📈 **Variable Workloads**: Mix of long/short contexts, high/low throughput
- 🔄 **Resource Utilization**: Prefill vs. decode have different bottlenecks
- 🌐 **Multi-Node Coordination**: Distributed serving complexity

---

# Solution: Dynamo

## Key Capabilities

✅ **Multi-Engine Support**: vLLM, SGLang, TensorRT-LLM
✅ **KV-Aware Routing**: Cache-conscious request routing
✅ **Disaggregated Serving**: Independent prefill/decode scaling
✅ **Distributed Runtime**: etcd + NATS for coordination
✅ **Multi-Tier Storage**: GPU → CPU → SSD → Remote

---

# High-Level Architecture

```
┌─────────────────────────────────────────────────┐
│              USER / CLIENT                       │
└────────────────┬────────────────────────────────┘
                 │ HTTP/REST (OpenAI-compatible)
                 ↓
┌─────────────────────────────────────────────────┐
│         FRONTEND (Rust + Python)                │
│   • HTTP Server (Axum)                          │
│   • Preprocessor (Tokenization, Templates)      │
└────────────────┬────────────────────────────────┘
                 │ PreprocessedRequest
                 ↓
┌─────────────────────────────────────────────────┐
│         KV-AWARE ROUTER (Rust)                  │
│   • Radix Tree Cache Tracking                   │
│   • Intelligent Worker Selection                │
└────────────────┬────────────────────────────────┘
                 │ Routed Request
                 ↓
┌─────────────────────────────────────────────────┐
│         WORKERS (Python + Engine)               │
│   • vLLM / SGLang / TRT-LLM                     │
│   • KV Cache Management                         │
└────────────────┬────────────────────────────────┘
                 │ Response Stream
                 ↓
┌─────────────────────────────────────────────────┐
│    INFRASTRUCTURE (etcd, NATS, TCP)             │
└─────────────────────────────────────────────────┘
```

---

# Layered Architecture

```
┌──────────────────────────────────────────────┐
│   APPLICATION LAYER (Python)                 │
│   Business Logic, Engine Integrations        │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│   FRAMEWORK LAYER (Rust)                     │
│   LLM Abstractions, Routing, Preprocessing   │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│   DISTRIBUTED RUNTIME (Rust)                 │
│   Service Discovery, Network Transport       │
└──────────────────────────────────────────────┘
                    ↓
┌──────────────────────────────────────────────┐
│   INFRASTRUCTURE (External)                  │
│   etcd, NATS, TCP, Prometheus                │
└──────────────────────────────────────────────┘
```

**Principle**: Performance-critical in Rust, Business logic in Python

---

# Component Model

## Service-Oriented Architecture

```
Namespace ("dynamo.vllm")
    ↓
Component ("worker")
    ↓
Endpoint ("generate")
    ↓
Instance (lease_id: 12345)

Full Path: dynamo.vllm.worker.generate-12345
```

**Registration**: etcd `/services/{namespace}/{component}/{endpoint}-{lease_id}`
**Discovery**: Automatic via etcd watchers
**Communication**: NATS service groups (load balanced)

---

# Request Flow (Aggregated Mode)

```
1. HTTP POST /v1/chat/completions
         ↓
2. Preprocessor
   • Apply chat template (Jinja2)
   • Tokenize (HuggingFace)
   • Extract sampling params
         ↓
3. KV Router
   • Compute block hashes
   • Query radix tree
   • Score workers
   • Select best worker
         ↓
4. Worker (vLLM/SGLang/TRT-LLM)
   • Check KV cache
   • Prefill + Decode
   • Stream tokens
         ↓
5. Response to User (OpenAI format)
```

**Latency**: 50-500ms TTFT (cold), 10-50ms (warm cache)

---

# KV-Aware Routing: The Secret Sauce

## Problem
Random routing → cache misses → recompute KV → wasted GPU cycles

## Solution
Route similar requests to same worker → cache hits → skip recomputation

---

# Radix Tree Cache Tracking

```
Worker-1 RadixTree:
  root
   ├─ hash_0xabcd (tokens 0-15)
   │   ├─ hash_0xef01 (tokens 16-31)
   │   │   └─ sequences: [seq_1, seq_5]
   │   └─ hash_0x1234 (tokens 16-31)
   │       └─ sequences: [seq_2]
   └─ hash_0x5678 (tokens 0-15)
       └─ hash_0x9abc (tokens 16-31)
           └─ sequences: [seq_3, seq_4]
```

**Complexity**: O(k) lookup, k = number of blocks
**Memory**: Shared structure for common prefixes

---

# Worker Scoring Algorithm

```python
def score_worker(worker, request):
    # 1. Compute block hashes
    blocks = hash_blocks(request.tokens, block_size=16)

    # 2. Query radix tree for overlap
    overlap = worker.radix_tree.query(blocks)
    overlap_ratio = overlap.matched / len(blocks)

    # 3. Get worker load
    load = worker.active_blocks / worker.capacity

    # 4. Combined score
    score = (
        overlap_ratio * 0.7 +      # Cache hits
        (1.0 - load) * 0.3          # Load balance
    )

    return score
```

**Configuration**:
- `overlap_score_weight` = 0.7 (balance cache vs. load)
- `router_temperature` = 0.0 (deterministic selection)

---

# KV Event Flow

```
┌─────────────────┐
│  Worker         │
│  Completes Req  │
└────────┬────────┘
         │
         ↓ Publish to NATS "kv-events.worker-1"
         │
┌────────┴────────┐
│  Event:         │
│  {              │
│    worker_id,   │
│    blocks: [...],│
│    sequence_id  │
│  }              │
└────────┬────────┘
         │
         ↓ Router subscribes
         │
┌────────┴────────┐
│  KvIndexer      │
│  Updates        │
│  Radix Tree     │
└────────┬────────┘
         │
         ↓
  Next similar request
  routes to same worker
      (cache hit!)
```

---

# Performance Impact

## Cache Hit Rate vs. Latency

| Cache Hit Rate | TTFT Reduction | Throughput Gain |
|---------------|----------------|-----------------|
| 0% (random)   | 0ms (baseline) | 1x             |
| 50%           | ~200ms         | 1.5x           |
| 70%           | ~300ms         | 2x             |
| 90%+          | ~400ms         | 3x+            |

**Real-world**: 70-90% hit rate typical with KV-aware routing

---

# Disaggregated Serving

## Problem

Prefill and decode have **different** resource requirements:

| Phase   | Bottleneck     | Characteristics |
|---------|----------------|-----------------|
| Prefill | Compute-bound  | Long sequences, high memory |
| Decode  | Memory-bound   | Single token, latency-sensitive |

Running both on same worker → **suboptimal resource utilization**

---

# Disaggregated Architecture

```
┌─────────────────────┐         ┌─────────────────────┐
│  Prefill Worker     │         │  Decode Worker      │
│  (GPU Cluster)      │ ←NIXL→  │  (GPU Cluster)      │
│                     │  RDMA   │                     │
│  • Compute-heavy    │         │  • Memory-heavy     │
│  • Process long ctx │         │  • Low latency      │
│  • Write KV cache   │         │  • Read KV cache    │
└─────────────────────┘         └─────────────────────┘
```

**NIXL (NVIDIA Integrated Cross-Lane)**:
- RDMA over InfiniBand/RoCE
- 400+ Gbps bandwidth (H100 NVLink)
- ~50ms for full KV transfer (4K context)
- Zero-copy GPU-to-GPU

---

# Disaggregated Flow

```
1. Request arrives (4096 tokens)
         ↓
2. Router Decision: Context > threshold?
         ↓ YES
3. Publish to Prefill Queue (NATS)
         ↓
4. Prefill Worker pulls request
   • Read existing KV (NIXL from decode worker)
   • Compute new KV for tokens 2048-4096
   • Write KV back (NIXL to decode worker)
   • Notify decode worker "ready"
         ↓
5. Decode Worker receives notification
   • KV already in local GPU memory
   • Run decode phase
   • Stream response
```

**Benefit**: Decode worker free during prefill → serve other requests

---

# Performance: Aggregated vs. Disaggregated

## Aggregated (Single Worker)
- Prefill 4096 tokens: **800ms**
- Decode 100 tokens: **1000ms**
- Total: **1800ms**
- Worker tied up for entire duration

## Disaggregated (Dedicated Workers)
- Prefill 4096 tokens (dedicated): **800ms**
- KV transfer (NIXL): **50ms**
- Decode 100 tokens: **1000ms**
- Total: **1850ms** (slightly slower)

**BUT**: Decode worker available during prefill!
**Throughput**: **2-3x higher** (can serve concurrent requests)

---

# Block Manager (KVBM)

## Multi-Tier Storage Hierarchy

```
┌──────────────────────────────────────┐
│  GPU HBM (~80GB H100)                │  ← Active KV
│  Fastest, Most Expensive              │
├──────────────────────────────────────┤
│  CPU DRAM (~1TB)                     │  ← Recently evicted
│  Fast, Expensive                      │
├──────────────────────────────────────┤
│  Local SSD (~8TB NVMe)               │  ← Warm storage
│  Medium, Moderate Cost                │
├──────────────────────────────────────┤
│  Remote Storage (S3, NFS)            │  ← Cold storage
│  Slow, Cheap, Unlimited               │
└──────────────────────────────────────┘
```

**Transfers**:
- GPU ↔ CPU: NIXL (RDMA)
- CPU ↔ SSD: POSIX I/O
- SSD ↔ Remote: Network (S3 API)

---

# Block Lifecycle

```
1. ALLOCATION
   Try GPU pool → If full, evict cold blocks → Fallback to CPU

2. USAGE
   Mark active → Increment refcount → Update LRU timestamp

3. EVICTION (memory pressure)
   Select victim (LRU) → Copy GPU→CPU → Update metadata

4. PROMOTION (cache hit)
   Read CPU→GPU → Allocate GPU slot → Update metadata

5. DEALLOCATION
   Decrement refcount → If 0, mark free → Reclaim lazily
```

**Policy**: LRU (Least Recently Used)
**Threshold**: Start evicting at 90% GPU memory full

---

# Service Discovery

## Registration Flow

```
Worker Startup
    ↓
1. Connect to etcd (get lease, TTL=10s)
    ↓
2. Create Namespace + Component + Endpoint
    ↓
3. endpoint.serve(handler)
    • Register in NATS service group
    • Store metadata in etcd:
      /services/dynamo.vllm/worker/generate-12345
    • Start lease keepalive (every 5s)
    ↓
4. Ready to serve requests
    ↓
5. On shutdown/crash:
    • Graceful: explicit unregister
    • Crash: lease expires → auto cleanup
```

---

# Technology Stack

## Infrastructure
- **etcd**: Service discovery, distributed config
- **NATS**: Messaging backbone (see next slide)
- **TCP**: Large response streaming
- **Prometheus**: Metrics collection
- **OpenTelemetry**: Distributed tracing

---

# NATS: The Messaging Backbone

## Multiple Roles in Dynamo

```
┌─────────────────────────────────────────────┐
│         NATS Server (JetStream)             │
└─────────────────────────────────────────────┘
    ↓         ↓           ↓            ↓
  RPC    KV Events   Prefill Queue   State
```

### 1. Request Routing (Service Groups)
```
Router → NATS "dynamo.vllm.worker.generate"
           ↓ (round-robin)
    [Worker-1, Worker-2, Worker-3]
```
**Automatic load balancing, failover**

### 2. KV Event Streaming (PubSub)
```
Workers → NATS "kv-events.*" → Router
```
**Cache state synchronization**

### 3. Prefill Queue (JetStream)
```
Router → NATS "prefill_queue" → Prefill Workers
```
**Durable, guaranteed delivery**

### 4. State Storage (JetStream KV)
```
Router state snapshots → "radix-bucket"
```
**Recovery without rebuilding**

---

# Why NATS?

## Performance
- **Throughput**: 11M+ msgs/sec
- **Latency**: <1ms
- **Scalability**: Thousands of connections

## Features
- **Service Groups**: Built-in load balancing
- **JetStream**: Persistence + guarantees
- **Clustering**: High availability
- **Lightweight**: No Zookeeper needed

## vs. Alternatives

| Feature | NATS | Kafka | RabbitMQ |
|---------|------|-------|----------|
| Latency | <1ms ✅ | 5-10ms | 2-5ms |
| Service Groups | Built-in ✅ | Manual | Manual |
| Complexity | Low ✅ | High | Medium |

**Perfect fit for real-time RPC patterns**

## Core Runtime
- **Rust**: Distributed runtime, routing, preprocessing
- **Python**: Business logic, engine integration
- **PyO3**: Rust ↔ Python bindings

## Engines
- **vLLM**: PagedAttention, continuous batching
- **SGLang**: Radix attention, native caching
- **TensorRT-LLM**: CUDA kernels, inflight batching

---

# Key Design Decisions

## 1. Rust Core + Python Bindings
**Why**: Performance where it matters, productivity for integration

## 2. etcd for Discovery
**Why**: Strong consistency, watch API, battle-tested (Kubernetes)

## 3. NATS for Messaging
**Why**: High throughput (millions msg/sec), low latency (<1ms)

## 4. Radix Tree for KV Indexing
**Why**: O(k) prefix matching, memory efficient

## 5. Multi-Tier Storage
**Why**: Optimize cost vs. latency, scale beyond GPU limits

---

# Performance Characteristics

## Throughput
- Single worker: **1,000 - 5,000 req/sec** (model dependent)
- Disaggregated: **2-3x higher** (prefill/decode parallel)
- Multi-worker: **Linear scaling** (10 workers = 10K-50K req/s)

## Latency
- **TTFT** (Time to First Token):
  - Cold cache: 50-500ms
  - Warm cache (90% hit): 10-50ms
- **TPOT** (Time Per Output Token): 5-20ms

## Scalability
- **Horizontal**: Linear with worker count
- **Vertical**: Multi-GPU via tensor parallelism
- **Distributed**: Multi-node via NATS + etcd

---

# Real-World Metrics

## Baseten Case Study
*Source: [Baseten Blog](https://www.baseten.co/blog)*

**Configuration**:
- Model: Qwen-Coder-3B
- Setup: Dynamo with KV-aware routing

**Results**:
- **2x faster inference** vs. vanilla vLLM
- **70%+ cache hit rate** on production traffic
- **50% reduction** in GPU cost per request

---

# MLPerf Inference Benchmark

## NVIDIA Blackwell + Dynamo
*Source: [NVIDIA Blog](https://blogs.nvidia.com/blog/mlperf-inference-blackwell-ultra/)*

**Record Performance**:
- **LLama-3.1-70B**: 3.5x faster than previous generation
- **Throughput**: 50,000+ tokens/sec per GPU
- **Latency**: <10ms TTFT with warm cache

**Key Enabler**: Disaggregated serving + KV-aware routing

---

# Observability

## Hierarchical Metrics

```
MetricsRegistry (Root)
├─ DistributedRuntime
│  ├─ etcd_operations_total
│  ├─ nats_messages_sent
│  └─ tcp_connections_active
└─ Namespace ("dynamo.vllm")
   └─ Component ("worker")
      └─ Endpoint ("generate")
         └─ Instance (12345)
            ├─ requests_total
            ├─ request_duration_seconds
            ├─ kv_cache_hit_ratio
            └─ tokens_generated_total
```

**Scraping**: `/metrics` endpoint per component
**Tracing**: OpenTelemetry (Jaeger, Tempo, DataDog)

---

# Health Checks

## Kubernetes Integration

```yaml
livenessProbe:
  httpGet:
    path: /live
    port: 8000
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 5
```

- `/live`: Service is running
- `/health`: Service ready to accept traffic (aggregated from all components)

---

# Multi-Tenancy

## Namespace Isolation

```
┌────────────────────────────────────┐
│  Namespace: "tenant-A"             │
│  ├─ worker-1 (llama-3-8b)          │
│  ├─ worker-2 (llama-3-8b)          │
│  └─ router                         │
└────────────────────────────────────┘

┌────────────────────────────────────┐
│  Namespace: "tenant-B"             │
│  ├─ worker-1 (mistral-7b)          │
│  └─ router                         │
└────────────────────────────────────┘
```

**Isolation**:
- Separate metrics/logs
- Resource quotas per namespace
- Independent scaling
- API key authentication

---

# Deployment Options

## 1. Local Development
```bash
# Start infrastructure
docker compose -f deploy/docker-compose.yml up -d

# Start frontend
python -m dynamo.frontend --http-port 8000

# Start worker
python -m dynamo.sglang --model llama-3-8b
```

## 2. Kubernetes
```bash
# Install operator
kubectl apply -f deploy/operator/

# Deploy workers
kubectl apply -f examples/backends/vllm/
```

---

# Auto-Scaling

## Dynamic Planner

**Monitors**:
- Prefill queue depth (NATS)
- Decode worker latency
- Request arrival rate
- SLA compliance

**Actions**:
- Scale prefill workers (queue depth high)
- Scale decode workers (latency high)
- Zero-downtime via NIXL KV migration

**Integration**: Kubernetes HPA or custom controller

---

# Future Roadmap

## Planned Enhancements

1. **Speculative Decoding**: Generate multiple candidates in parallel
2. **Batch Prefill**: Batch multiple prefill requests together
3. **LoRA Adapters**: Dynamic model switching at runtime
4. **Enhanced Multi-Modal**: Better image/video/audio support
5. **Function Calling**: Tool use and agentic workflows

## Research Areas

- **ML-Based Routing**: Learn optimal worker selection
- **Smart Cache Eviction**: Value-based (not just LRU)
- **Request Migration**: Live migration between workers
- **Heterogeneous Workers**: Mix of GPU types (H100, A100, etc.)

---

# Benchmarking Guide

## Compare Deployment Topologies

```bash
# Vanilla vLLM (baseline)
./benchmarks/llm/perf.sh --engine vllm --model llama-3-8b

# Dynamo Aggregated
./benchmarks/llm/perf.sh --engine dynamo-vllm --mode aggregated

# Dynamo Disaggregated
./benchmarks/llm/perf.sh --engine dynamo-vllm --mode disaggregated

# Analyze results
python benchmarks/llm/analyze.py --compare
```

**Tools**: AIPerf, custom load generators

---

# SLA-Driven Deployment

## Optimize for Your Requirements

```python
# Define SLA
sla_config = {
    "p50_latency_ms": 50,
    "p99_latency_ms": 200,
    "throughput_req_per_sec": 1000
}

# Profile workload
python -m dynamo.planner.profile \
    --sla-config sla.json \
    --workload production_trace.json

# Generate deployment recommendation
# → 5 prefill workers, 10 decode workers
# → block_size=16, overlap_weight=0.7
```

**Guide**: `docs/planner/sla_planner_quickstart.md`

---

# Contributing

## Getting Started

1. **Setup Development Environment**
   ```bash
   # Install dependencies
   sudo apt install build-essential libhwloc-dev ...

   # Install Rust
   curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

   # Build Rust bindings
   cd lib/bindings/python && maturin develop --uv
   ```

2. **Run Tests**
   ```bash
   # Rust
   cargo test --all-targets

   # Python
   pytest -m unit
   ```

3. **Pre-commit Hooks**
   ```bash
   pre-commit install
   ```

---

# Contribution Workflow

## Issue-First Approach

1. **Create GitHub Issue** (for changes ≥100 lines)
   - Problem description
   - Proposed solution
   - Estimated PR size

2. **Get Approval** (`approved-for-pr` label)

3. **Submit PR**
   - Link to issue: "Fixes #123"
   - Pass CI tests
   - Address Code Rabbit review

4. **Sign-off Required** (DCO)
   ```bash
   git commit -s -m "Your message"
   ```

---

# Key Resources

## Documentation
- **Architecture**: `ARCHITECTURE.md`
- **Components**: `COMPONENTS.md`
- **Development**: `CLAUDE.md`
- **API Docs**: https://docs.nvidia.com/dynamo/

## Code
- **GitHub**: https://github.com/ai-dynamo/dynamo
- **Containers**: NGC Catalog (nvidia.com/ai-dynamo)
- **Examples**: `examples/` and `recipes/`

## Community
- **Discord**: https://discord.gg/D92uqZRjCZ
- **Issues**: https://github.com/ai-dynamo/dynamo/issues

---

# Summary

## What Makes Dynamo Unique?

✅ **KV-Aware Routing**: 2-3x throughput via cache reuse
✅ **Disaggregated Serving**: Independent prefill/decode scaling
✅ **Multi-Engine**: vLLM, SGLang, TRT-LLM support
✅ **Production-Ready**: etcd + NATS for reliability
✅ **Observable**: Hierarchical metrics, distributed tracing
✅ **Scalable**: Linear horizontal scaling, multi-node

**Built in Rust for performance, Python for productivity**

---

# Questions?

## Get Started

```bash
# Quick start
uv venv venv && source venv/bin/activate
uv pip install "ai-dynamo[sglang]"

python -m dynamo.frontend --http-port 8000 &
python -m dynamo.sglang --model deepseek-ai/DeepSeek-R1-Distill-Llama-8B &

curl localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model": "DeepSeek-R1-Distill-Llama-8B",
       "messages": [{"role": "user", "content": "Hello!"}]}'
```

**Links**:
- Docs: https://docs.nvidia.com/dynamo/
- GitHub: https://github.com/ai-dynamo/dynamo
- Discord: https://discord.gg/D92uqZRjCZ

---

# Thank You!

**NVIDIA Dynamo**
*High-Throughput Distributed LLM Inference*

---
**Appendix**: Architecture diagrams, configuration reference, troubleshooting
