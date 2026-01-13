# Dynamo Architecture

This document provides a comprehensive overview of Dynamo's architecture, design patterns, and system interactions.

## Table of Contents

- [Layered Architecture](#layered-architecture)
- [Core Architectural Patterns](#core-architectural-patterns)
- [Request Flow](#request-flow)
- [KV-Aware Routing](#kv-aware-routing)
- [Disaggregated Serving](#disaggregated-serving)
- [Block Manager (KVBM)](#block-manager-kvbm)
- [Pipeline Architecture](#pipeline-architecture)
- [Service Discovery](#service-discovery)
- [Design Decisions](#design-decisions)

---

## Layered Architecture

Dynamo is organized into four distinct architectural layers:

```
┌─────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                         │
│         (Python Components - Business Logic)                 │
│  frontend/ | router/ | vllm/ | sglang/ | trtllm/ | planner/ │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                     FRAMEWORK LAYER                          │
│              (Rust Core - LLM Abstractions)                  │
│    lib/llm: Backend | KvRouter | Preprocessor | BlockMgr    │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   DISTRIBUTED RUNTIME                        │
│         (Rust Infrastructure - lib/runtime)                  │
│   Component Model | Service Discovery | Network Transport    │
└─────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────┐
│                   INFRASTRUCTURE LAYER                       │
│              etcd | NATS | TCP | ZMQ | Metrics              │
└─────────────────────────────────────────────────────────────┘
```

### Layer Responsibilities

**Application Layer (Python)**
- Engine-specific integrations (vLLM, SGLang, TensorRT-LLM)
- Business logic and request handling
- Dynamic scaling decisions (Planner)
- HTTP API endpoints

**Framework Layer (Rust)**
- LLM-specific abstractions and utilities
- KV-aware routing intelligence
- Request preprocessing (tokenization, chat templates, multimodal)
- Backend execution management

**Distributed Runtime (Rust)**
- Service-oriented architecture primitives
- Network communication (NATS, TCP, etcd)
- Service discovery and registration
- Metrics and observability

**Infrastructure Layer (External)**
- etcd: Distributed configuration and service discovery
- NATS: High-throughput messaging and event streaming
- TCP: Large response streaming
- Prometheus/OTEL: Metrics and tracing

---

## Core Architectural Patterns

### 1. Component-Based Service Architecture

Every service in Dynamo follows a hierarchical naming pattern:

```
Namespace ("dynamo.vllm")
   ↓
Component ("worker")
   ↓
Endpoint ("generate")
   ↓
Instance (lease_id: 12345)

Full identifier: dynamo.vllm.worker.generate-12345
```

**Key Properties:**
- Components register in **etcd** at `/services/{namespace}/{component}/{endpoint}-{lease_id}`
- Endpoints expose via **NATS** service groups for automatic load balancing
- Automatic service discovery via etcd watchers
- Graceful degradation on instance failure (lease expiration)

**Example Registration:**
```rust
// Worker side
let drt = DistributedRuntime::new(runtime, config).await?;
let namespace = drt.namespace("dynamo.vllm");
let component = namespace.component("worker").build()?;
let endpoint = component.endpoint("generate");

endpoint.serve(|request| async move {
    // Handle request
    Ok(response)
}).await?;

// Automatically registers:
// - etcd: /services/dynamo.vllm/worker/generate-12345
// - NATS service group: dynamo.vllm.worker.generate
```

**Example Client:**
```rust
// Client side
let client = Client::new(drt, "dynamo.vllm", "worker", "generate");
let response = client.call(request).await?;

// Automatically:
// 1. Discovers instances via etcd watch
// 2. Load balances across available instances
// 3. Sends via NATS, receives via TCP
```

### 2. Pipeline Architecture

Request processing uses a dataflow pipeline pattern:

```
Request
  ↓
Transform1 (Preprocessor)
  ↓
Transform2 (Router)
  ↓
Transform3 (Backend)
  ↓
Transform4 (ResponseFormatter)
  ↓
Response
```

Each transform implements the `Operator` trait:

```rust
#[async_trait]
trait Operator<In, Out, NextIn, NextOut> {
    async fn generate(
        &self,
        input: In,
        next: Engine<NextIn, NextOut>
    ) -> Result<Out>;
}
```

**Key Features:**
- **Streaming**: Each operator can emit multiple outputs (async generators)
- **Context propagation**: Request metadata (trace_id, user_id) flows through pipeline
- **Error handling**: Errors annotated with context from each stage
- **Metrics**: Automatic collection at each stage

### 3. Radix Tree for KV Cache Tracking

The router maintains a radix tree per worker to track KV cache state:

```
Worker-1 RadixTree:
  root
   ├─ hash_a (tokens 0-15)
   │   ├─ hash_b (tokens 16-31)
   │   │   └─ sequence_123
   │   └─ hash_c (tokens 16-31)
   │       └─ sequence_456
   └─ hash_d (tokens 0-15)
       └─ hash_e (tokens 16-31)
           └─ sequence_789
```

**Benefits:**
- O(k) lookup where k = number of blocks in query
- Efficient prefix matching
- Shared structure for common prefixes
- Reference counting for cache eviction

### 4. Event-Driven KV Synchronization

Workers publish KV events to keep the router synchronized:

```
Worker completes request
   ↓
Publishes to NATS: "kv-events.{namespace}.{worker_id}"
   {
     worker_id: "worker-1",
     blocks: [{hash, token_range, refcount}, ...],
     sequence_finished: true
   }
   ↓
Router subscribes and updates radix tree
   ↓
Next similar request routes to same worker
```

---

## Request Flow

### Aggregated Mode (Single Worker)

```
1. HTTP Request
   ↓
   Frontend (Rust - Axum HTTP server)
   - POST /v1/chat/completions
   - OpenAI-compatible API

2. Preprocessing
   ↓
   Preprocessor (Rust - lib/llm/src/preprocessor.rs)
   - Apply chat template (Jinja2 via minijinja)
   - Load images/video if multimodal
   - Tokenize prompt (HuggingFace tokenizers)
   - Output: PreprocessedRequest {
       token_ids: [128000, 128006, 882, ...],
       sampling_options: {temperature, top_p, ...},
       stop_conditions: {max_tokens, stop_sequences, ...},
       metadata: {model, trace_id, ...}
     }

3. Routing Decision
   ↓
   KvRouter (Rust - lib/llm/src/kv_router.rs)
   - Compute block hashes from token_ids
   - Query radix tree for each worker
   - Score = overlap_ratio * weight + (1 - load) * (1 - weight)
   - Apply temperature for randomization
   - Select best worker

4. Worker Processing
   ↓
   Worker Handler (Python - e.g., vllm/handlers.py)
   - Convert to engine format (SamplingParams, TokensPrompt)
   - Submit to engine (vLLM/SGLang/TRT-LLM)

5. Engine Execution
   ↓
   Backend + Engine
   - PREFILL: Process all prompt tokens, compute KV cache
   - DECODE: Generate tokens one-by-one
   - Check stop conditions (max_tokens, EOS, stop sequences)
   - Stream each token via TCP

6. Response Streaming
   ↓
   - Convert to OpenAI format
   - HTTP SSE stream to user

7. KV Event Publishing
   ↓
   - Worker publishes KV state to NATS
   - Router updates radix tree
   - Future similar requests benefit from cache
```

### Disaggregated Mode (Prefill + Decode Workers)

```
Long context request (4096 tokens)
   ↓
Router Decision:
   - Context length > threshold?
   - Decode worker has cache hit?
   - Prefill queue not overloaded?
   → YES: Remote prefill

1. Publish to Prefill Queue
   ↓
   NATS JetStream: "prefill_queue"
   {
     request: {preprocessed_request},
     target_decode_worker: "decode-worker-1",
     kv_metadata: {
       nixl_addr: "10.0.1.5:9000",
       existing_blocks: [hash1, hash2, ...],
       new_block_offset: 128
     }
   }

2. Prefill Worker Processing
   ↓
   - Pull from queue
   - Read existing KV from decode worker (NIXL RDMA)
   - Compute KV for new tokens
   - Write KV back to decode worker (NIXL RDMA)
   - Notify decode worker: "KV ready"

3. Decode Worker Processing
   ↓
   - Receive "KV ready" notification
   - Verify KV blocks in local GPU memory
   - Run decode phase (token generation)
   - Stream response to user

4. Performance Benefits
   ↓
   - Decode worker free during prefill
   - Can serve other decode requests concurrently
   - Throughput: 2-3x higher
   - Slight latency overhead (~50ms for KV transfer)
```

---

## KV-Aware Routing

### Scoring Algorithm

For each request, the router scores all available workers:

```python
def score_worker(worker, request):
    # 1. Compute block hashes for request
    block_hashes = compute_block_hashes(
        request.token_ids,
        block_size=16
    )

    # 2. Query worker's radix tree
    overlap = worker.radix_tree.query(block_hashes)
    overlap_ratio = overlap.matched_blocks / len(block_hashes)

    # 3. Get worker load
    load = worker.active_blocks / worker.capacity

    # 4. Combined score
    score = (
        overlap_ratio * overlap_weight +
        (1.0 - load) * (1.0 - overlap_weight)
    )

    return score

# 5. Apply temperature (randomization)
if temperature > 0:
    scores = softmax([score_worker(w, req) for w in workers] / temperature)
    selected = weighted_random(workers, scores)
else:
    selected = max(workers, key=lambda w: score_worker(w, req))
```

### Configuration Parameters

- `overlap_score_weight` (default: 0.7): Weight for cache hits vs. load balancing
  - 1.0 = Always prefer cache hits (may overload workers)
  - 0.0 = Pure load balancing (ignores cache)
  - 0.7 = Balance between cache reuse and load distribution

- `router_temperature` (default: 0.0): Randomization in worker selection
  - 0.0 = Deterministic (always select highest score)
  - 0.2 = Slight randomization (explore other workers)
  - 1.0 = High randomization (exploration)

- `router_track_active_blocks` (default: true): Track active KV blocks for load estimation

---

## Disaggregated Serving

### Architecture

```
┌─────────────────┐         ┌─────────────────┐
│ Prefill Worker  │         │ Decode Worker   │
│  (GPU Cluster)  │ ←NIXL→  │  (GPU Cluster)  │
│                 │  RDMA   │                 │
│ - Compute-bound │         │ - Memory-bound  │
│ - Long seqs     │         │ - Single token  │
│ - High memory   │         │ - Low latency   │
└─────────────────┘         └─────────────────┘
```

### Use Cases

**When to disaggregate:**
1. **Long context prompts**: Prefill takes significant time, blocks decode workers
2. **High cache hit rate**: Decode worker already has prefix cached
3. **Variable workload**: Mix of long prefills and short decodes
4. **Resource optimization**: Scale prefill/decode independently

**When NOT to disaggregate:**
1. **Short prompts**: KV transfer overhead > prefill time
2. **Cold cache**: No existing KV in decode worker
3. **Low GPU utilization**: Workers not saturated
4. **Single-node deployment**: NIXL overhead not worth it

### NIXL (NVIDIA Integrated Cross-Lane)

High-performance KV cache transfer mechanism:

- **RDMA over InfiniBand/RoCE**: Zero-copy GPU-to-GPU transfer
- **Bandwidth**: 400 Gbps+ (H100 NVLink)
- **Latency**: 10-50 µs for metadata, ~50ms for full KV transfer
- **No CPU involvement**: Direct GPU memory access

---

## Block Manager (KVBM)

### Multi-Tier Storage Hierarchy

```
GPU HBM (Fastest, ~80GB)
  ↓ NIXL/NVLink
CPU DRAM (Fast, ~1TB)
  ↓ PCIe
Local SSD (Medium, ~8TB)
  ↓ Network
Remote Storage (Slow, unlimited)
```

### Block Lifecycle

```
1. Allocation
   BlockPool::allocate(num_blocks)
   - Try GPU pool first
   - Evict cold blocks if OOM (LRU policy)
   - Fallback to CPU pool

2. Usage
   - Mark block as active
   - Increment reference count
   - Update LRU timestamp

3. Eviction (memory pressure)
   - Select victim: LRU or oldest
   - Copy: GPU → CPU (NIXL) or CPU → SSD (POSIX)
   - Update block location metadata
   - Free GPU memory

4. Promotion (cache hit)
   - Read: CPU → GPU (NIXL) or SSD → CPU
   - Allocate GPU slot (may trigger eviction)
   - Update location metadata

5. Deallocation
   - Decrement reference count
   - If refcount == 0: mark as free
   - Memory reclaimed lazily
```

### Configuration

- **Block size**: 16 tokens (default), must match across all workers
- **GPU pool size**: Auto-detected or manual override
- **CPU pool size**: Unlimited (bounded by system RAM)
- **Eviction policy**: LRU (Least Recently Used)
- **Offload threshold**: Start evicting when GPU > 90% full

---

## Pipeline Architecture

### Operator Pattern

Each pipeline stage implements:

```rust
#[async_trait]
impl Operator<SingleIn<Req>, ManyOut<Resp>, NextIn, NextOut> for Stage {
    async fn generate(
        &self,
        input: SingleIn<Req>,
        next: Engine<NextIn, NextOut>
    ) -> Result<ManyOut<Resp>> {
        // Process input
        let processed = self.process(input)?;

        // Call next stage
        let next_stream = next.generate(processed).await?;

        // Transform output stream
        let output_stream = next_stream.map(|item| {
            self.transform(item)
        });

        Ok(output_stream)
    }
}
```

### Example Pipeline

```rust
// Build pipeline
let pipeline = Preprocessor::new(tokenizer)
    .then(KvRouter::new(indexer))
    .then(Backend::new(engine))
    .then(ResponseFormatter::new());

// Execute
let response_stream = pipeline.generate(request).await?;

// Stream results
while let Some(chunk) = response_stream.next().await {
    send_to_client(chunk)?;
}
```

---

## Service Discovery

### Registration Flow

```
1. Worker starts
   ↓
2. Create DistributedRuntime
   - Connect to etcd
   - Get primary lease (TTL: 10s)
   - Start keepalive (every 5s)
   ↓
3. Create Namespace + Component + Endpoint
   ↓
4. endpoint.serve(handler)
   - Register in NATS service group
   - Store metadata in etcd
   - Publish readiness
   ↓
5. Automatic cleanup on:
   - Graceful shutdown
   - Lease expiration (crash)
   - Network partition
```

### Discovery Flow

```
1. Client::new(namespace, component, endpoint)
   ↓
2. Start etcd watcher
   - Path: /services/{ns}/{component}/{endpoint}
   - Receive initial instances
   ↓
3. Background task
   - Watch for instance changes
   - Add new instances to pool
   - Remove expired instances
   ↓
4. client.call(request)
   - Select instance (load balancing)
   - Send to NATS subject
   - Receive response via TCP
```

### etcd Key Hierarchy

```
/services/{namespace}/{component}/{endpoint}-{lease_id}
  metadata: {
    transport: "nats_tcp://...",
    instance_id: 12345,
    model: "llama-3-8b",
    gpu_count: 2,
    block_size: 16,
    max_model_len: 8192
  }

/config/{model_name}/{worker_type}
  configuration: {...}

/locks/{resource}
  distributed locks

/metadata/kvbm/{worker_id}
  NIXL memory descriptors
```

---

## Design Decisions

### 1. Rust Core + Python Bindings

**Rationale:**
- Performance-critical paths in Rust (runtime, routing, tokenization)
- Business logic in Python (engine integration, easier development)
- Best of both worlds: performance + productivity

**Implementation:**
- PyO3 for Python bindings
- Maturin for building wheels
- Shared memory between Rust/Python when possible

### 2. etcd for Service Discovery

**Rationale:**
- Strong consistency for service metadata
- Watch API for dynamic updates
- Lease mechanism for failure detection
- Battle-tested in Kubernetes

**Alternatives considered:**
- Consul: More features, more complexity
- ZooKeeper: Older, Java-based
- DNS: No strong consistency, no watches

### 3. NATS for Messaging

**Rationale:**
- High throughput (millions of messages/sec)
- Service groups for automatic load balancing
- JetStream for durable streams (prefill queue, KV events)
- Low latency (<1ms)

**Key Roles in Dynamo:**

1. **Request Routing (Core NATS)**
   - RPC pattern for worker communication
   - Service groups provide automatic load balancing
   - Subject: `{namespace}.{component}.{endpoint}`
   - Round-robin delivery to worker instances

2. **Event Streaming (PubSub)**
   - KV cache state updates: `kv-events.*`
   - Router synchronization: `router-sync.*`
   - Wildcard subscriptions for scalability

3. **Durable Queues (JetStream)**
   - Prefill queue for disaggregated serving
   - Guaranteed delivery with acknowledgments
   - Persistent storage for reliability

4. **State Storage (JetStream KV)**
   - Router state snapshots: `radix-bucket`
   - Recovery without rebuilding state
   - Distributed configuration

**Alternatives considered:**
- Kafka: Higher latency (5-10ms), heavier weight (requires Zookeeper)
- RabbitMQ: Lower throughput, more complex configuration
- Redis Streams: Not designed for distributed systems, lacks service groups

### 4. TCP for Response Streaming

**Rationale:**
- Avoids NATS message size limits (1MB default)
- Lower latency for large payloads
- Direct connection after initial routing

**Why not use NATS for responses:**
- Large responses (100s of tokens) exceed message limits
- Extra serialization overhead
- TCP streaming more efficient

### 5. Radix Tree for KV Indexing

**Rationale:**
- Efficient prefix matching: O(k) where k = prefix length
- Shared structure for common prefixes
- Memory efficient (shared nodes)

**Alternatives considered:**
- HashMap: O(1) exact match, but no prefix queries
- Trie: Similar to radix tree, more nodes
- Bloom filter: False positives, no exact tracking

### 6. Pipeline Architecture

**Rationale:**
- Composable transformations
- Easy to add/remove stages
- Automatic metrics collection
- Streaming by default

**Alternatives considered:**
- Middleware pattern: Less composable
- Monolithic handler: Hard to extend
- Actor model: More complexity

### 7. Multi-Tier Storage (KVBM)

**Rationale:**
- Optimize cost vs. latency tradeoff
- Transparent to application layer
- Scales beyond GPU memory limits

**Alternatives considered:**
- GPU-only: Limited capacity
- CPU-only: High latency
- External KV store: Network overhead

---

## Performance Characteristics

### Throughput

- **Single worker (aggregated)**: 1,000-5,000 requests/sec (depending on model size)
- **Disaggregated**: 2-3x higher (prefill/decode parallelized)
- **Multi-worker**: Linear scaling (10 workers = 10,000-50,000 req/s)

### Latency

- **TTFT (Time to First Token)**:
  - Cold cache: 50-500ms (depends on context length)
  - Warm cache (90% hit): 10-50ms

- **TPOT (Time Per Output Token)**:
  - Decode: 5-20ms/token (depends on model size, batch size)

- **Disaggregated overhead**:
  - KV transfer: 50ms (4K context, NIXL RDMA)
  - Notification: <1ms (NATS)

### Scalability

- **Horizontal**: Linear scaling with number of workers
- **Vertical**: Supports multi-GPU workers (tensor parallelism)
- **Distributed**: Multi-node via NATS + etcd
- **Multi-model**: Namespace isolation, independent scaling

---

## Observability

### Metrics Hierarchy

```
MetricsRegistry (Root)
├─ DistributedRuntime metrics
│  ├─ etcd_operations_total
│  ├─ nats_messages_sent_total
│  └─ tcp_connections_active
├─ Namespace metrics
│  └─ Component metrics
│     └─ Endpoint metrics
│        └─ Instance metrics
│           ├─ requests_total
│           ├─ request_duration_seconds
│           ├─ kv_cache_hit_ratio
│           └─ tokens_generated_total
```

### Tracing

- **OpenTelemetry**: Distributed tracing across services
- **Trace context**: Propagated via HTTP headers and NATS metadata
- **Spans**: Created at each pipeline stage
- **Export**: OTLP to Jaeger/Tempo/DataDog

### Health Checks

- `/health`: Overall system health (aggregated from all components)
- `/live`: Liveness probe (service is running)
- `/ready`: Readiness probe (service is ready to accept traffic)

---

## Security Considerations

### TLS/mTLS

- Frontend supports TLS (HTTPS)
- NATS supports TLS for encryption in transit
- etcd supports TLS for secure communication

### Authentication

- API keys via HTTP headers
- JWT tokens for user authentication
- Service-to-service auth via etcd ACLs

### Multi-Tenancy

- Namespace isolation
- Resource quotas per namespace
- Separate metrics/logs per tenant

---

## Future Directions

### Planned Enhancements

1. **Speculative Decoding**: Generate multiple candidates in parallel
2. **Batch Prefill Optimization**: Batch multiple prefill requests
3. **LoRA Adapter Support**: Dynamic model switching
4. **Function Calling**: Tool use and agentic workflows
5. **Multi-Modal**: Better image/video/audio support

### Research Areas

1. **Adaptive Routing**: ML-based worker selection
2. **Cache Eviction**: Smarter policies (value-based, not just LRU)
3. **Request Migration**: Live migration between workers
4. **Heterogeneous Workers**: Mix of different GPU types

---

## References

- [Design Docs](docs/design_docs/)
- [Backend Guides](docs/backends/)
- [Kubernetes Deployment](docs/kubernetes/)
- [Benchmarking Guide](docs/benchmarks/benchmarking.md)
