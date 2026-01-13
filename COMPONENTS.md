# Dynamo Component Reference

This document provides detailed breakdowns of Dynamo's core components, their interactions, and implementation details.

## Table of Contents

- [System Overview](#system-overview)
- [Distributed Runtime](#distributed-runtime)
- [Frontend Layer](#frontend-layer)
- [Routing Layer](#routing-layer)
- [Worker Layer](#worker-layer)
- [Infrastructure Services](#infrastructure-services)
- [Component Interactions](#component-interactions)

---

## System Overview

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         USER / CLIENT                                    │
│           (HTTP/REST, gRPC, or any OpenAI-compatible client)            │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │ HTTP/HTTPS
                                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                         FRONTEND LAYER                                   │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ HTTP Server (Rust - Axum) + Preprocessor                           │ │
│  │ • OpenAI-compatible REST API                                       │ │
│  │ • Chat template application                                        │ │
│  │ • Tokenization                                                     │ │
│  │ • Multimodal encoding                                              │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │ PreprocessedRequest
                                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                         ROUTING LAYER                                    │
│  ┌────────────────────────────────────────────────────────────────────┐ │
│  │ KV-Aware Router (Rust)                                             │ │
│  │ • Radix tree KV cache tracking                                     │ │
│  │ • Worker scoring & selection                                       │ │
│  │ • Prefill/decode coordination                                      │ │
│  └────────────────────────────────────────────────────────────────────┘ │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │ Routed Request
                                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                         WORKER LAYER                                     │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐                 │
│  │ vLLM Worker  │  │SGLang Worker │  │TRT-LLM Worker│                 │
│  │ • Handler    │  │ • Handler    │  │ • Handler    │                 │
│  │ • AsyncLLM   │  │ • Runtime    │  │ • Engine     │                 │
│  │ • KV Pub     │  │ • KV Pub     │  │ • KV Pub     │                 │
│  └──────────────┘  └──────────────┘  └──────────────┘                 │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │ LLMEngineOutput
                                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                    DISTRIBUTED RUNTIME                                   │
│  • Component Model  • Service Discovery  • Network Transport            │
│  • Metrics         • Health Checks       • Configuration                │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                                 ↓
┌─────────────────────────────────────────────────────────────────────────┐
│                    INFRASTRUCTURE SERVICES                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌────────────────────┐    │
│  │   etcd   │  │   NATS   │  │    TCP   │  │ Prometheus / OTEL  │    │
│  └──────────┘  └──────────┘  └──────────┘  └────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Distributed Runtime

**Location**: `lib/runtime/`

The distributed runtime provides the foundational infrastructure for building distributed applications.

### Core Components

#### DistributedRuntime (`src/distributed.rs`)

Main orchestrator that provides:

```rust
pub struct DistributedRuntime {
    // Local async runtime
    runtime: Runtime,

    // Communication
    nats_client: Option<nats::Client>,
    tcp_server: Arc<OnceCell<TcpStreamServer>>,

    // Storage
    store: kv::Manager,  // etcd/file/memory

    // Service discovery
    discovery_client: Arc<dyn Discovery>,

    // Component registry
    component_registry: Registry,

    // Health & metrics
    system_health: Arc<Mutex<SystemHealth>>,
    metrics_registry: MetricsRegistry,
}
```

**Key Methods:**
- `new(runtime, config)` - Initialize with etcd/NATS connections
- `namespace(name)` - Create logical namespace
- `shutdown()` - Graceful shutdown with cleanup

#### Component (`src/component/component.rs`)

Logical unit of work (e.g., "worker", "router", "preprocessor"):

```rust
pub struct Component {
    drt: Arc<DistributedRuntime>,
    name: String,
    namespace: Namespace,
    metrics_registry: MetricsRegistry,
}
```

**Usage:**
```rust
let component = namespace
    .component("worker")
    .labels(vec![("gpu", "0")])
    .build()?;
```

#### Endpoint (`src/component/endpoint.rs`)

Network-accessible service:

```rust
pub struct Endpoint {
    component: Arc<Component>,
    name: String,
    transport: TransportType,
}
```

**Server-side:**
```rust
endpoint.serve(|request: Request| async move {
    // Handle request
    Ok(response)
}).await?;
```

**Client-side:**
```rust
let client = Client::new(drt, namespace, component, endpoint);
let response = client.call(request).await?;
```

### Transport Layer

#### NATS Client (`src/transports/nats.rs`)

High-performance messaging:

**Core Features:**
- **PubSub**: `publish(subject, data)`, `subscribe(subject)`
- **Request/Reply**: Built-in with timeouts
- **Service Groups**: Automatic load balancing
- **JetStream**: Durable streams, KV buckets

**Subject Patterns:**
- RPC: `{namespace}.{component}.{endpoint}`
- Events: `kv-events.{namespace}.{worker_id}`
- Queues: `prefill_queue`, `active_sequences_events`

#### etcd Client (`src/transports/etcd.rs`)

Distributed configuration and coordination:

**Features:**
- **Lease management**: Automatic keepalive, TTL-based expiration
- **Watch API**: Real-time updates on key changes
- **Transactions**: Atomic operations
- **Locking**: Distributed locks for coordination

**Key Structure:**
```
/services/
  {namespace}/
    {component}/
      {endpoint}-{lease_id}  → {transport, metadata, ...}

/config/
  {model_name}/
    {worker_type}  → configuration

/locks/
  {resource}  → distributed lock

/metadata/
  kvbm/
    {worker_id}  → NIXL descriptors
```

#### TCP Server (`src/transports/tcp.rs`)

Streaming response server:

**Why separate from NATS:**
- Avoids message size limits (NATS default: 1MB)
- Lower latency for large payloads
- Direct connection after routing

**Implementation:**
```rust
let tcp_server = TcpStreamServer::new()?;
let addr = tcp_server.local_addr()?;

// Listen for connections
tcp_server.accept_stream(stream_id, |stream| async move {
    while let Some(chunk) = data.next().await {
        stream.write_all(&chunk).await?;
    }
    Ok(())
});
```

### Service Discovery

#### Discovery Trait (`src/discovery/mod.rs`)

Pluggable backends for service discovery:

```rust
#[async_trait]
pub trait Discovery: Send + Sync {
    async fn watch(&self, query: DiscoveryQuery)
        -> Result<Receiver<Vec<Instance>>>;

    async fn register(&self, instance: Instance) -> Result<()>;

    async fn unregister(&self, instance_id: u64) -> Result<()>;
}
```

**Implementations:**
- **KV Store** (`kv_store.rs`): etcd-based (default)
- **Kubernetes** (`kube.rs`): CRD-based for K8s native
- **Mock** (`mock.rs`): In-memory for testing

#### KV Store Discovery (`src/discovery/kv_store.rs`)

etcd-based service discovery:

**Registration:**
```rust
// Worker registers
discovery.register(Instance {
    namespace: "dynamo.vllm",
    component: "worker",
    endpoint: "generate",
    instance_id: lease_id,
    transport: TransportType::Nats(subject),
    metadata: {...}
}).await?;

// Stored in etcd:
// /services/dynamo.vllm/worker/generate-12345
```

**Discovery:**
```rust
// Client watches
let query = DiscoveryQuery::Endpoint {
    namespace: "dynamo.vllm",
    component: "worker",
    endpoint: "generate"
};

let rx = discovery.watch(query).await?;

// Receive updates
while let Ok(instances) = rx.recv().await {
    // Update client pool
}
```

### Pipeline (`src/pipeline/`)

Dataflow processing framework:

#### Operator Trait

```rust
#[async_trait]
pub trait Operator<In, Out, NextIn, NextOut> {
    async fn generate(
        &self,
        input: In,
        next: Engine<NextIn, NextOut>
    ) -> Result<Out>;
}
```

#### Example Pipeline

```rust
// Define stages
struct Stage1;
impl Operator<Req1, Resp1, Req2, Resp2> for Stage1 {
    async fn generate(&self, input: Req1, next: ...) -> Result<Resp1> {
        let processed = self.process(input)?;
        let next_stream = next.generate(processed).await?;
        Ok(transform(next_stream))
    }
}

// Compose pipeline
let pipeline = Stage1::new()
    .then(Stage2::new())
    .then(Stage3::new());

// Execute
let output = pipeline.generate(input).await?;
```

---

## Frontend Layer

**Location**: `lib/llm/src/` and `components/src/dynamo/frontend/`

### HTTP Server (`lib/llm/src/http/`)

Axum-based REST API server:

**Endpoints:**
- `POST /v1/chat/completions` - Chat completion
- `POST /v1/completions` - Text completion
- `POST /v1/embeddings` - Generate embeddings
- `GET /health` - Health check
- `GET /metrics` - Prometheus metrics
- `GET /openapi.json` - OpenAPI specification

**Request Flow:**
```rust
// 1. Receive HTTP request
async fn chat_completions(
    Json(req): Json<ChatCompletionRequest>
) -> Response {
    // 2. Convert to internal format
    let internal_req = convert_openai_to_internal(req)?;

    // 3. Submit to pipeline
    let stream = pipeline.generate(internal_req).await?;

    // 4. Convert to SSE stream
    let sse_stream = stream.map(|output| {
        Event::default()
            .data(serde_json::to_string(&output)?)
    });

    // 5. Return streaming response
    Sse::new(sse_stream)
}
```

### Preprocessor (`lib/llm/src/preprocessor.rs`)

Converts OpenAI requests to engine-ready format:

**Pipeline:**
```
OpenAI Request
  ↓
1. Chat Template Application
   - Load template from model config
   - Apply Jinja2 template via minijinja
   - Input: [{"role": "user", "content": "Hello"}]
   - Output: "<|begin_of_text|><|start_header_id|>user..."
  ↓
2. Multimodal Processing (if applicable)
   - Load images/video from URLs or base64
   - Encode with vision processor
   - Insert placeholders in text
  ↓
3. Tokenization
   - HuggingFace tokenizers (Rust binding)
   - Input: text string
   - Output: token IDs [128000, 128006, ...]
  ↓
4. Build PreprocessedRequest
   {
     request_id: "req-abc",
     token_ids: [128000, ...],
     sampling_options: {temperature, top_p, ...},
     stop_conditions: {max_tokens, stop_sequences, ...},
     metadata: {model, trace_id, ...}
   }
```

**Key Components:**

#### Chat Template Engine
```rust
pub struct ChatTemplateEngine {
    templates: HashMap<String, Template>,
}

impl ChatTemplateEngine {
    pub fn apply(
        &self,
        messages: &[ChatMessage],
        model: &str
    ) -> Result<String> {
        let template = self.get_template(model)?;
        template.render(context! {
            messages => messages,
            bos_token => "<|begin_of_text|>",
            ...
        })
    }
}
```

#### Multimodal Processor
```rust
pub struct MultimodalProcessor {
    vision_encoder: Arc<VisionEncoder>,
}

impl MultimodalProcessor {
    pub async fn process_media(
        &self,
        media: &[MediaItem]
    ) -> Result<ProcessedMedia> {
        // Load images/video
        let loaded = self.load_media(media).await?;

        // Encode with vision model
        let encoded = self.vision_encoder.encode(loaded)?;

        // Create placeholders
        let text_with_placeholders =
            self.insert_placeholders(&loaded)?;

        Ok(ProcessedMedia {
            text: text_with_placeholders,
            embeddings: encoded,
        })
    }
}
```

#### Tokenizer (`lib/llm/src/tokenizers.rs`)
```rust
pub struct Tokenizer {
    inner: Arc<dyn TokenizerImpl>,
}

impl Tokenizer {
    pub fn encode(&self, text: &str) -> Result<Vec<u32>> {
        self.inner.encode(text, AddSpecialTokens(true))
    }

    pub fn decode(&self, ids: &[u32]) -> Result<String> {
        self.inner.decode(ids, SkipSpecialTokens(false))
    }

    pub fn decode_stream(
        &self,
        prompt_ids: &[u32],
        skip_special: bool
    ) -> DecodeStream {
        DecodeStream::new(self.clone(), prompt_ids, skip_special)
    }
}
```

---

## Routing Layer

**Location**: `lib/llm/src/kv_router/`

### KV Router Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           KV Router                                      │
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐│
│  │ KvIndexer (indexer.rs)                                             ││
│  │                                                                     ││
│  │  workers: HashMap<WorkerId, WorkerState>                           ││
│  │    WorkerState {                                                   ││
│  │      radix_tree: RadixTree<BlockHash>                              ││
│  │      active_blocks: HashSet<BlockHash>                             ││
│  │      capacity: usize                                               ││
│  │      config: ModelRuntimeConfig                                    ││
│  │    }                                                                ││
│  │                                                                     ││
│  │  Methods:                                                           ││
│  │  • query(worker_id, block_hashes) → OverlapScores                  ││
│  │  • handle_kv_event(event) → update radix tree                      ││
│  │  • add_worker(worker_id, config)                                   ││
│  │  • remove_worker(worker_id)                                        ││
│  └────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐│
│  │ KvScheduler (scheduler.rs)                                         ││
│  │                                                                     ││
│  │  Methods:                                                           ││
│  │  • select_worker(request, workers) → WorkerId                      ││
│  │  • score_worker(worker, block_hashes) → f64                        ││
│  │  • apply_temperature(scores, temp) → probabilities                 ││
│  └────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐│
│  │ SequenceTracker (sequence.rs)                                      ││
│  │                                                                     ││
│  │  sequences: HashMap<RequestId, SequenceInfo>                       ││
│  │    SequenceInfo {                                                   ││
│  │      worker_id: WorkerId                                           ││
│  │      state: SequenceState (Created/Running/Completed)              ││
│  │      blocks: Vec<BlockHash>                                        ││
│  │      created_at: Timestamp                                         ││
│  │    }                                                                ││
│  └────────────────────────────────────────────────────────────────────┘│
│                                                                          │
│  ┌────────────────────────────────────────────────────────────────────┐│
│  │ Subscriber (subscriber.rs) - Background Task                       ││
│  │                                                                     ││
│  │  • Subscribe to NATS "kv-events.*"                                 ││
│  │  • For each event:                                                 ││
│  │    - Parse KV event                                                ││
│  │    - Update KvIndexer                                              ││
│  │    - Update SequenceTracker                                        ││
│  │    - Snapshot state if needed                                      ││
│  └────────────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────────────┘
```

### Radix Tree Structure

```rust
pub struct RadixTree<K> {
    root: Node<K>,
}

pub struct Node<K> {
    hash: Option<K>,
    children: HashMap<K, Box<Node<K>>>,
    sequences: Vec<SequenceId>,
}
```

**Example Tree:**
```
root
 ├─ hash_0xabcd (block 0: tokens 0-15)
 │   ├─ hash_0xef01 (block 1: tokens 16-31)
 │   │   └─ sequences: [seq_1, seq_5]
 │   └─ hash_0x1234 (block 1: tokens 16-31)
 │       └─ sequences: [seq_2]
 └─ hash_0x5678 (block 0: tokens 0-15)
     └─ hash_0x9abc (block 1: tokens 16-31)
         └─ sequences: [seq_3, seq_4]
```

**Query Algorithm:**
```rust
pub fn query(&self, blocks: &[BlockHash]) -> OverlapScores {
    let mut matched = 0;
    let mut current = &self.root;

    for block_hash in blocks {
        if let Some(child) = current.children.get(block_hash) {
            matched += 1;
            current = child;
        } else {
            break;
        }
    }

    OverlapScores {
        matched_blocks: matched,
        total_blocks: blocks.len(),
        ratio: matched as f64 / blocks.len() as f64,
    }
}
```

### Worker Selection Algorithm

**File**: `scheduler.rs`

```rust
pub struct KvScheduler {
    overlap_weight: f64,
    temperature: f64,
    track_active_blocks: bool,
}

impl KvScheduler {
    pub fn select_worker(
        &self,
        request: &SchedulingRequest,
        workers: &HashMap<WorkerId, WorkerState>,
        indexer: &KvIndexer
    ) -> Result<WorkerId> {
        // 1. Compute block hashes
        let block_hashes = compute_block_hashes(
            &request.token_ids,
            request.block_size
        );

        // 2. Score each worker
        let mut scores = Vec::new();
        for (worker_id, worker) in workers {
            let score = self.score_worker(
                worker_id,
                worker,
                &block_hashes,
                indexer
            )?;
            scores.push((worker_id, score));
        }

        // 3. Apply temperature
        if self.temperature > 0.0 {
            let probs = softmax(
                &scores.iter().map(|(_, s)| s / self.temperature)
                    .collect::<Vec<_>>()
            );
            weighted_random(&scores, &probs)
        } else {
            scores.iter()
                .max_by(|(_, s1), (_, s2)|
                    s1.partial_cmp(s2).unwrap())
                .map(|(id, _)| id)
                .cloned()
                .ok_or(Error::NoWorkersAvailable)
        }
    }

    fn score_worker(
        &self,
        worker_id: &WorkerId,
        worker: &WorkerState,
        block_hashes: &[BlockHash],
        indexer: &KvIndexer
    ) -> Result<f64> {
        // Query overlap
        let overlap = indexer.query(worker_id, block_hashes)?;
        let overlap_ratio = overlap.ratio;

        // Compute load
        let load = if self.track_active_blocks {
            worker.active_blocks.len() as f64 / worker.capacity as f64
        } else {
            0.0
        };

        // Combined score
        let score =
            overlap_ratio * self.overlap_weight +
            (1.0 - load) * (1.0 - self.overlap_weight);

        Ok(score)
    }
}
```

### KV Event Processing

**Publisher** (Worker-side):
```python
# components/src/dynamo/vllm/publisher.py

class KvEventPublisher:
    def __init__(self, nats_client, worker_id):
        self.nats = nats_client
        self.worker_id = worker_id
        self.subject = f"kv-events.dynamo.vllm.{worker_id}"

    async def publish(self, request_id, blocks):
        event = {
            "worker_id": self.worker_id,
            "request_id": request_id,
            "blocks": [
                {
                    "hash": block.hash,
                    "token_range": [block.start, block.end],
                    "refcount": block.refcount
                }
                for block in blocks
            ],
            "sequence_finished": True,
            "timestamp": time.time()
        }

        await self.nats.publish(
            self.subject,
            json.dumps(event).encode()
        )
```

**Subscriber** (Router-side):
```rust
// lib/llm/src/kv_router/subscriber.rs

pub async fn start_kv_router_background(
    indexer: Arc<Mutex<KvIndexer>>,
    nats_client: nats::Client
) -> Result<()> {
    let sub = nats_client
        .subscribe("kv-events.*")
        .await?;

    while let Some(msg) = sub.next().await {
        let event: KvEvent = serde_json::from_slice(&msg.payload)?;

        // Update indexer
        let mut indexer = indexer.lock().await;
        indexer.handle_kv_event(event)?;
    }

    Ok(())
}
```

---

## Worker Layer

**Location**: `components/src/dynamo/{vllm,sglang,trtllm}/`

### vLLM Worker

#### Handler (`vllm/handlers.py`)

```python
class DecodeWorkerHandler:
    def __init__(self, engine, runtime, config):
        self.engine = engine  # vLLM AsyncLLM
        self.runtime = runtime  # DistributedRuntime
        self.config = config
        self.publisher = KvEventPublisher(...)

    async def handle_generate(
        self,
        request: PreprocessedRequest
    ) -> AsyncGenerator[LLMEngineOutput, None]:
        # 1. Build sampling params
        sampling_params = build_sampling_params(
            request,
            self.config.default_sampling_params
        )

        # 2. Create vLLM input
        vllm_input = TokensPrompt(
            prompt_token_ids=request["token_ids"]
        )

        # 3. Submit to engine
        async for output in self.engine.generate(
            vllm_input,
            sampling_params,
            request_id=request["request_id"]
        ):
            # 4. Convert to internal format
            llm_output = self.convert_output(output)

            # 5. Stream back
            yield llm_output

        # 6. Publish KV event
        await self.publisher.publish(
            request["request_id"],
            self.get_kv_blocks(request["request_id"])
        )

    def convert_output(
        self,
        vllm_output: RequestOutput
    ) -> LLMEngineOutput:
        return {
            "request_id": vllm_output.request_id,
            "text": vllm_output.outputs[0].text,
            "token_ids": vllm_output.outputs[0].token_ids,
            "finish_reason": vllm_output.outputs[0].finish_reason,
            "metadata": {
                "worker_id": self.config.worker_id,
                "timestamp": time.time()
            }
        }
```

#### Disaggregated Handlers

**Prefill Handler:**
```python
class PrefillWorkerHandler:
    async def handle_prefill(
        self,
        request: PrefillRequest
    ) -> None:
        # 1. Read existing KV from decode worker
        existing_kv = await self.nixl.read(
            remote_addr=request["decode_worker_addr"],
            block_ids=request["existing_blocks"]
        )

        # 2. Run prefill with existing KV
        kv_cache = await self.engine.prefill(
            request["token_ids"],
            existing_kv=existing_kv
        )

        # 3. Write new KV to decode worker
        await self.nixl.write(
            remote_addr=request["decode_worker_addr"],
            block_offset=request["new_block_offset"],
            kv_data=kv_cache
        )

        # 4. Notify decode worker
        await self.nats.publish(
            f"prefill_complete.{request['decode_worker_id']}",
            {
                "request_id": request["request_id"],
                "kv_blocks_written": kv_cache.block_hashes,
                "status": "ready_for_decode"
            }
        )
```

**Decode Handler (Disaggregated):**
```python
class DisaggregatedDecodeHandler:
    async def handle_decode(
        self,
        request: PreprocessedRequest
    ) -> AsyncGenerator[LLMEngineOutput, None]:
        # 1. Wait for prefill completion
        await self.wait_for_prefill(request["request_id"])

        # 2. Verify KV in local memory
        assert self.has_kv_blocks(request["kv_blocks"])

        # 3. Run decode
        async for output in self.engine.generate(
            token_ids=[],  # No prefill needed
            kv_cache=self.get_kv_blocks(request["kv_blocks"]),
            sampling_params=request["sampling_params"]
        ):
            yield output
```

### SGLang Worker

Similar structure to vLLM, with differences:

**Key Differences:**
- Native ZMQ support (in addition to NATS)
- Radix attention (built-in KV cache management)
- Different request/response format

```python
# components/src/dynamo/sglang/handlers.py

class SGLangWorkerHandler:
    def __init__(self, runtime_endpoint, config):
        self.runtime = runtime_endpoint  # SGLang runtime
        self.config = config

    async def handle_generate(self, request):
        # Convert to SGLang format
        sglang_request = {
            "text": None,  # Already tokenized
            "input_ids": request["token_ids"],
            "sampling_params": {
                "temperature": request["sampling_options"]["temperature"],
                "top_p": request["sampling_options"]["top_p"],
                "max_new_tokens": request["stop_conditions"]["max_tokens"]
            }
        }

        # Submit to SGLang
        async for output in self.runtime.generate(sglang_request):
            yield self.convert_output(output)
```

### TensorRT-LLM Worker

**Key Differences:**
- C++ engine (Python bindings)
- Inflight batching
- Different multimodal handling

```python
# components/src/dynamo/trtllm/handlers.py

class TrtllmWorkerHandler:
    def __init__(self, engine_path, config):
        self.engine = TrtllmEngine.from_dir(engine_path)
        self.config = config

    async def handle_generate(self, request):
        # Build TRT-LLM request
        trt_request = self.engine.create_request(
            input_token_ids=request["token_ids"],
            max_new_tokens=request["stop_conditions"]["max_tokens"],
            temperature=request["sampling_options"]["temperature"],
            top_p=request["sampling_options"]["top_p"]
        )

        # Submit to engine
        for output in self.engine.generate([trt_request]):
            yield self.convert_output(output)
```

---

## Infrastructure Services

### etcd

**Purpose**: Service discovery, distributed configuration, coordination

**Usage in Dynamo:**

1. **Service Registration:**
```bash
# Worker registers endpoint
etcdctl put /services/dynamo.vllm/worker/generate-12345 '{
  "transport": "nats_tcp://dynamo.vllm.worker.generate",
  "instance_id": 12345,
  "metadata": {
    "model": "llama-3-8b",
    "gpu_count": 2,
    "block_size": 16
  }
}'
```

2. **Service Discovery:**
```bash
# Client watches for workers
etcdctl watch --prefix /services/dynamo.vllm/worker/generate
```

3. **Configuration:**
```bash
# Store model config
etcdctl put /config/llama-3-8b/worker '{
  "max_batch_size": 256,
  "max_model_len": 8192,
  "gpu_memory_utilization": 0.9
}'
```

### NATS

**Purpose**: High-performance messaging, event streaming, queues

NATS is the **messaging backbone** of Dynamo, connecting all distributed components.

#### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                NATS Server Cluster                       │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐              │
│  │ NATS-1   │  │ NATS-2   │  │ NATS-3   │              │
│  │ (Leader) │──│ (Follower)──│ (Follower)              │
│  └──────────┘  └──────────┘  └──────────┘              │
│                                                          │
│  Core Features:                                          │
│  • Pub/Sub messaging                                    │
│  • Service groups (load balancing)                      │
│  • Request/Reply                                        │
│                                                          │
│  JetStream Features:                                    │
│  • Persistent streams                                   │
│  • KV buckets                                           │
│  • Consumer groups                                      │
└─────────────────────────────────────────────────────────┘
```

#### Role 1: Request Routing (Service Groups)

**Purpose**: Load-balanced RPC to workers

```
Router publishes to "dynamo.vllm.worker.generate"
         ↓
┌────────────────────────────────────────┐
│  NATS Service Group (automatic LB)     │
└────────────────────────────────────────┘
         ↓
    ┌────┼────┐
    ↓    ↓    ↓
Worker-1 Worker-2 Worker-3
(1/3)    (1/3)    (1/3)
```

**How it works:**
- All workers subscribe to **same subject**
- NATS delivers each message to **only one worker**
- Automatic round-robin load balancing
- Failover if worker crashes

**Code Example:**
```rust
// Worker subscribes (all workers use same subject)
let sub = nats_client
    .subscribe("dynamo.vllm.worker.generate")
    .await?;

// NATS automatically load balances
while let Some(msg) = sub.next().await {
    handle_request(msg).await?;
}
```

#### Role 2: KV Event Streaming (PubSub)

**Purpose**: Broadcast KV cache state updates

```
Worker-1 → NATS "kv-events.dynamo.vllm.worker-1"
Worker-2 → NATS "kv-events.dynamo.vllm.worker-2"
                    ↓
        Router subscribes to "kv-events.*"
                    ↓
        Updates radix tree for routing
```

**Publisher (Worker):**
```python
# After request completion
event = {
    "worker_id": "worker-1",
    "blocks": [
        {"hash": "0xabcd1234", "token_range": [0, 16]},
        {"hash": "0xef567890", "token_range": [16, 32]}
    ],
    "sequence_finished": True
}

await nats_client.publish(
    f"kv-events.dynamo.vllm.{worker_id}",
    json.dumps(event).encode()
)
```

**Subscriber (Router):**
```rust
// Router subscribes to all KV events
let sub = nats_client.subscribe("kv-events.*").await?;

while let Some(msg) = sub.next().await {
    let event: KvEvent = serde_json::from_slice(&msg.payload)?;
    indexer.handle_kv_event(event)?;
}
```

#### Role 3: Prefill Queue (JetStream)

**Purpose**: Durable queue for disaggregated prefill

**Why JetStream:**
- **Durability**: Messages persist if worker crashes
- **At-least-once delivery**: Guarantees processing
- **Backpressure**: Queue depth indicates system load

```rust
// Router publishes to prefill queue
js_context.publish(
    "prefill_queue",
    serde_json::to_vec(&prefill_request)?
).await?;

// Prefill worker pulls and acknowledges
let consumer = js_context
    .create_consumer("prefill_queue")
    .await?;

while let Some(msg) = consumer.messages().await?.next().await {
    let request = parse_request(msg.payload);
    process_prefill(request).await?;
    msg.ack().await?; // Acknowledge completion
}
```

#### Role 4: State Storage (JetStream KV)

**Purpose**: Persist router state for recovery

```rust
// Create KV bucket
let kv = js_context.create_key_value("radix-bucket").await?;

// Store snapshot
kv.put("radix-state", serialized_tree).await?;

// Retrieve on router restart
let snapshot = kv.get("radix-state").await?;
let tree = deserialize_tree(snapshot.value)?;
```

#### Message Patterns

| Pattern | Subject | Use Case | Features |
|---------|---------|----------|----------|
| **Request/Reply** | `dynamo.vllm.worker.generate` | RPC to workers | Service groups, load balancing |
| **PubSub** | `kv-events.*` | KV cache updates | Wildcard subscriptions |
| **Queue** | `prefill_queue` | Disaggregated prefill | JetStream, durability |
| **KV Store** | `radix-bucket/radix-state` | State snapshots | JetStream KV |
| **Broadcast** | `router-sync.*` | Router coordination | All subscribers receive |

#### Configuration

**Development:**
```bash
# Start NATS with JetStream
nats-server -js
```

**Production (Cluster):**
```conf
# nats-cluster.conf
jetstream {
    store_dir: /data/nats
    max_memory: 1GB
    max_file: 100GB
}

cluster {
    name: dynamo-cluster
    listen: 0.0.0.0:6222
    routes: [
        nats://nats-1:6222
        nats://nats-2:6222
        nats://nats-3:6222
    ]
}
```

**Dynamo Config:**
```rust
pub struct NatsConfig {
    pub url: String,              // "nats://localhost:4222"
    pub max_reconnects: i32,       // -1 (unlimited)
    pub reconnect_delay: Duration, // 2 seconds
    pub enable_jetstream: bool,    // true
}
```

#### Performance

- **Throughput**: 11M+ msgs/sec (core), 1M+ msgs/sec (JetStream)
- **Latency**: <1ms (core), <5ms (JetStream)
- **Connections**: Thousands per node
- **Dynamo usage**: 10K-100K msgs/sec average, 500K peak

#### Monitoring

```bash
# Check server status
nats server check

# List streams
nats stream list

# Monitor all subjects
nats sub ">" --count

# View stream info
nats stream info prefill_queue

# View KV bucket
nats kv list radix-bucket
```

**Metrics** (exposed at `/metrics`):
- `nats_msgs_in_total`
- `nats_msgs_out_total`
- `nats_connections_total`
- `jetstream_streams`
- `jetstream_consumers`

#### Failure Handling

**Worker Crash:**
```
Before: Router → NATS → [Worker-1, Worker-2, Worker-3]
Worker-2 crashes
After:  Router → NATS → [Worker-1, Worker-3]

NATS automatically removes crashed worker from service group
No manual intervention needed
```

**NATS Server Crash (Cluster):**
```
NATS-1 (Leader) crashes
  ↓
NATS-2 promoted to Leader
  ↓
Clients auto-reconnect
No message loss (JetStream persists)
```

#### CLI Commands

```bash
# RPC (Request/Reply)
nats req dynamo.vllm.worker.generate '{"request_id": "req-1", ...}'

# Subscribe to events
nats sub "kv-events.>"

# Publish event
nats pub kv-events.dynamo.vllm.worker-1 '{"worker_id": "worker-1", ...}'

# Create stream
nats stream add prefill_queue \
  --subjects "prefill_queue" \
  --retention limits \
  --max-age 1h

# Publish to queue
nats pub prefill_queue '{"request": {...}}'

# Create consumer
nats consumer add prefill_queue prefill_worker
```

### Prometheus

**Metrics Endpoints:**

```bash
# Frontend metrics
curl http://frontend:8000/metrics

# Worker metrics
curl http://worker:8001/metrics
```

**Key Metrics:**
- `dynamo_requests_total{namespace, component, endpoint}`
- `dynamo_request_duration_seconds{namespace, component}`
- `dynamo_kv_cache_hit_ratio{worker_id}`
- `dynamo_tokens_generated_total{worker_id, model}`
- `dynamo_active_requests{worker_id}`

---

## Component Interactions

### Complete Request Flow (Detailed)

See [ARCHITECTURE.md](ARCHITECTURE.md#request-flow) for the complete flow.

### Disaggregated Request Flow (Detailed)

See [ARCHITECTURE.md](ARCHITECTURE.md#disaggregated-serving) for the complete flow.

### KV Event Flow

```
1. Worker completes request
   ↓
2. KvEventPublisher.publish()
   - Collect KV block hashes
   - Build event payload
   - Publish to NATS "kv-events.{namespace}.{worker_id}"
   ↓
3. NATS delivers to subscribers
   ↓
4. Router Subscriber receives event
   ↓
5. KvIndexer.handle_kv_event()
   - Parse event
   - Update worker's radix tree
   - Add/update block hashes
   - Increment reference counts
   - Prune old sequences if needed
   ↓
6. Next similar request
   ↓
7. KvScheduler.select_worker()
   - Compute block hashes
   - Query radix tree
   - Find cache hit on same worker
   - Route to worker
   ↓
8. Worker processes with cache hit
   - Reduced TTFT (Time to First Token)
   - Higher throughput
```

### Service Lifecycle

**Worker Startup:**
```
1. Parse configuration
2. Create DistributedRuntime
   - Connect to etcd
   - Connect to NATS
   - Get primary lease
3. Initialize engine (vLLM/SGLang/TRT-LLM)
4. Load model
5. Create handlers
6. Register endpoints
   - endpoint.serve(handler)
   - Register in etcd
   - Join NATS service group
7. Start KV event publisher
8. Ready to serve
```

**Worker Shutdown:**
```
1. Receive SIGTERM/SIGINT
2. Stop accepting new requests
3. Wait for in-flight requests (graceful timeout)
4. Unregister from etcd (delete key)
5. Close NATS connection
6. Cleanup resources (GPU memory, etc.)
7. Exit
```

**Router Lifecycle:**
```
1. Create DistributedRuntime
2. Initialize KvIndexer
3. Start background subscriber
   - Subscribe to "kv-events.*"
   - Update indexer on events
4. Register routing endpoint
5. Serve requests
   - Receive PreprocessedRequest
   - Select worker via KvScheduler
   - Forward to worker
6. On shutdown:
   - Snapshot radix tree state (optional)
   - Unregister endpoint
   - Stop subscriber
```

---

## Configuration Reference

### Runtime Configuration

**File**: `lib/runtime/src/config.rs`

```rust
pub struct RuntimeConfig {
    // System
    pub num_threads: usize,
    pub system_port: u16,
    pub system_health_path: String,
    pub system_live_path: String,

    // Health
    pub starting_health_status: HealthStatus,
    pub use_endpoint_health_status: bool,
}
```

### Distributed Configuration

```rust
pub struct DistributedConfig {
    pub store_kv: KvSelector,  // etcd/file/memory
    pub nats_config: Option<NatsConfig>,
    pub request_plane: RequestPlaneMode,  // NATS/ZMQ
}
```

### KV Router Configuration

```rust
pub struct KvRouterConfig {
    pub overlap_score_weight: f64,  // 0.7 default
    pub router_temperature: f64,     // 0.0 default
    pub use_kv_events: bool,         // true
    pub router_track_active_blocks: bool,  // true
    pub router_assume_kv_reuse: bool,      // true
    pub router_snapshot_threshold: Option<u32>,
    pub router_reset_states: bool,   // false
}
```

### Engine-Specific Configuration

**vLLM:**
```python
VllmConfig(
    model="meta-llama/Llama-3-8b",
    tensor_parallel_size=2,
    max_model_len=8192,
    gpu_memory_utilization=0.9,
    block_size=16,
    enable_prefix_caching=True
)
```

**SGLang:**
```python
SGLangConfig(
    model_path="meta-llama/Llama-3-8b",
    tp_size=2,
    mem_fraction_static=0.9,
    context_length=8192
)
```

**TensorRT-LLM:**
```python
TrtllmConfig(
    engine_dir="/path/to/engine",
    max_batch_size=256,
    max_input_len=4096,
    max_output_len=2048
)
```

---

## Debugging & Troubleshooting

### Enable Debug Logging

```bash
export DYN_LOG=debug
export RUST_BACKTRACE=1
```

### Check Service Registration

```bash
# List all registered services
etcdctl get --prefix /services/ --print-value-only

# Watch for changes
etcdctl watch --prefix /services/
```

### Monitor NATS Subjects

```bash
# List all subjects
nats sub ">"

# Monitor KV events
nats sub "kv-events.>"

# Monitor prefill queue
nats stream info prefill_queue
```

### Inspect Metrics

```bash
# Frontend metrics
curl http://localhost:8000/metrics | grep dynamo

# Worker metrics
curl http://localhost:8001/metrics | grep dynamo
```

### Common Issues

**"No workers available"**
- Check etcd: `etcdctl get --prefix /services/`
- Check NATS: `nats server ping`
- Verify worker registration in logs

**"KV routing not working"**
- Check `overlap_score_weight > 0`
- Verify KV event publishing enabled
- Check router subscriber logs

**"High latency"**
- Check KV cache hit ratio (should be >70%)
- Monitor worker GPU utilization
- Check network latency (etcd/NATS)

---

For more information, see:
- [ARCHITECTURE.md](ARCHITECTURE.md) - Overall architecture
- [CLAUDE.md](CLAUDE.md) - Development guide
- [docs/](docs/) - Detailed documentation
