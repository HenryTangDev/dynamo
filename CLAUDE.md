# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

NVIDIA Dynamo is a high-throughput, low-latency distributed inference framework for serving generative AI models across multi-node environments. Built in Rust (core runtime) and Python (components), it supports multiple inference engines (vLLM, SGLang, TensorRT-LLM) and provides LLM-specific capabilities like disaggregated serving, KV-aware routing, and distributed cache management.

## Development Setup

### System Requirements
- Ubuntu 24.04 (x86_64 CPU recommended)
- Rust toolchain (install via `rustup`)
- Python 3.10+
- Protocol Buffers compiler (`protoc`)
- Build essentials: `build-essential`, `libhwloc-dev`, `libudev-dev`, `pkg-config`, `libclang-dev`, `protobuf-compiler`, `python3-dev`, `cmake`

### Initial Setup

```bash
# Install system dependencies
sudo apt install -y build-essential libhwloc-dev libudev-dev pkg-config libclang-dev protobuf-compiler python3-dev cmake

# Install Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env

# Install uv (Python package manager)
curl -LsSf https://astral.sh/uv/install.sh | sh

# Create and activate Python virtual environment
uv venv venv
source venv/bin/activate

# Install maturin (Rust<->Python bindings tool)
uv pip install pip maturin

# Build Rust bindings
cd lib/bindings/python
maturin develop --uv
cd ../../..

# Install Dynamo with desired engine
uv pip install -e .
# Or with specific engine: uv pip install -e ".[sglang]" or ".[vllm]" or ".[trtllm]"
```

### Infrastructure Services

Dynamo requires NATS (with JetStream) and optionally etcd for distributed coordination:

```bash
# Quick setup via Docker Compose
docker compose -f deploy/docker-compose.yml up -d

# Or install manually:
# - nats: Run with `nats-server -js`
# - etcd: Run with `./etcd`

# For local development without etcd, use file-based KV store:
# Pass --store-kv file to both frontend and workers
# Configure directory: export DYN_FILE_KV=/data/kv/dynamo
```

**NATS Roles in Dynamo:**
- **Request Routing**: Load-balanced RPC to workers via service groups
- **KV Events**: PubSub for cache state updates (`kv-events.*`)
- **Prefill Queue**: JetStream durable queue for disaggregated serving
- **State Storage**: JetStream KV buckets for router snapshots
- **Router Sync**: Coordinate multiple router replicas

See [COMPONENTS.md](COMPONENTS.md#nats) for detailed NATS usage patterns.

### Running Dynamo Locally

```bash
# Sanity check (verify system configuration)
./deploy/sanity_check.py

# Start frontend (HTTP API + router)
python -m dynamo.frontend --http-port 8000 [--store-kv file]

# Start worker (example with SGLang)
python -m dynamo.sglang --model deepseek-ai/DeepSeek-R1-Distill-Llama-8B [--store-kv file]

# Test the API
curl localhost:8000/v1/chat/completions -H "Content-Type: application/json" -d '{
  "model": "deepseek-ai/DeepSeek-R1-Distill-Llama-8B",
  "messages": [{"role": "user", "content": "Hello!"}],
  "stream": false,
  "max_tokens": 300
}'
```

## Build and Test Commands

### Rust

```bash
# Format check
cargo fmt -- --check

# Lint (clippy)
cargo clippy --no-deps --all-targets -- -D warnings

# Build
cargo build [--release]

# Run tests
cargo test --locked --all-targets    # Unit and integration tests
cargo test --locked --doc             # Doc tests
cargo doc --no-deps                   # Generate documentation

# Build specific workspace member
cd lib/llm && cargo build

# License and dependency checks
cargo install cargo-deny@0.16.4
cargo-deny --no-default-features check --hide-inclusion-graph licenses bans --config deny.toml
```

### Python

```bash
# Install pre-commit hooks (for formatting/linting)
pip install pre-commit
pre-commit install
pre-commit run --all-files

# Run Python tests
pytest [path/to/test]

# Run tests with markers
pytest -m "unit"                      # Unit tests only
pytest -m "integration"               # Integration tests
pytest -m "vllm"                      # vLLM-specific tests
pytest -m "gpu_1"                     # Tests requiring 1 GPU

# Format Python code (automatically via pre-commit)
# Uses: black, isort, ruff

# Type checking (automatically via pytest --mypy)
```

### Running Single Tests

```bash
# Rust: Run specific test
cargo test test_name
cargo test --package dynamo-llm test_name

# Python: Run specific test
pytest path/to/test_file.py::test_function_name
pytest -k "test_pattern"
```

### GitHub Actions Locally

```bash
# Install act: https://nektosact.com/introduction.html
# Run specific workflow
act -j pre-merge-rust
```

## Architecture Overview

### High-Level Component Flow

```
User Request (OpenAI API)
    ↓
Frontend (HTTP Server) - Axum-based Rust server, OpenAI-compatible REST API
    ↓
Preprocessor - Chat templates, multimodal encoding, tokenization
    ↓
Router (KV-Aware) - Radix tree-based routing, cache hit optimization, load balancing
    ↓
Workers (vLLM/SGLang/TRT-LLM) - Prefill workers, Decode workers, or Aggregated
    ↓
Distributed Runtime (etcd + NATS + TCP) - Service discovery, messaging, response streaming
```

### Key Architectural Concepts

**Disaggregated Serving**: Separate workers for prefill (compute-bound, long sequences) and decode (memory-bound, single token generation). Enables independent scaling and resource optimization. KV cache is transferred between prefill and decode workers via NIXL (high-performance RDMA).

**KV-Aware Routing**: Router maintains a radix tree of KV cache state across all workers. Routes requests to workers with the highest cache overlap to minimize recomputation. Workers publish KV events to NATS after each request to keep router state synchronized.

**Distributed Runtime**: Core infrastructure (`lib/runtime`) provides service discovery (etcd), messaging (NATS), and network transports (TCP, ZMQ). Components register endpoints in etcd and communicate via NATS subjects. Health monitoring and metrics collection built-in.

**KVBM (KV Block Manager)**: Multi-tier memory management system (`lib/llm/block_manager/`) supporting GPU HBM → CPU DRAM → SSD → Remote storage. Handles block allocation, offloading, and distributed sharing via NIXL.

**Component Model**: Service-oriented architecture where each component exposes endpoints in a namespace (e.g., `dynamo.vllm.worker.generate`). Clients discover endpoints via etcd watchers and load balance requests via NATS service groups.

### Code Organization

**Rust Core (`lib/`)**:
- `runtime/`: Distributed runtime infrastructure (etcd, NATS, TCP, service discovery, pipeline architecture)
- `llm/`: LLM-specific logic (backend abstraction, KV router, preprocessor, block manager, tokenizers, protocols)
- `config/`: Configuration management (TOML/JSON, environment variables via figment)
- `tokens/`: Token counting and definitions
- `memory/`: NUMA-aware memory allocation
- `parsers/`: Reasoning and tool-use parsing
- `async-openai/`: OpenAI protocol bindings
- `bindings/python/`: PyO3 Python bindings (built via maturin)

**Python Components (`components/src/dynamo/`)**:
- `frontend/`: HTTP API server wrapper and router integration
- `router/`: Standalone KV-aware router service
- `vllm/`: vLLM engine integration (handlers, args, publisher, multimodal)
- `sglang/`: SGLang engine integration (ZMQ-based, radix attention)
- `trtllm/`: TensorRT-LLM integration (C++ engine bindings)
- `planner/`: Dynamic worker scaling (SLA-based, Kubernetes connector)
- `common/`: Shared utilities (config dump, LoRA management, OTEL tracing)

**Key Files to Know**:
- `lib/llm/src/kv_router.rs` (36KB): KV-aware routing engine with radix tree indexer and scheduler
- `lib/llm/src/backend.rs` (23KB): Backend engine abstraction and decoder state machine
- `lib/llm/src/preprocessor.rs` (50KB+): Request preprocessing pipeline with chat templates and multimodal handling
- `lib/runtime/src/distributed.rs`: Main distributed runtime orchestrator
- `components/src/dynamo/vllm/handlers.py` (59KB): vLLM request/response handling for prefill and decode
- `components/src/dynamo/vllm/main.py` (37KB): vLLM worker initialization and lifecycle

## Configuration

Configuration uses figment (Rust) with precedence: code defaults < TOML files < environment variables.

**Important Environment Variables**:
- `DYN_LOG`: Logging level (same syntax as `RUST_LOG`, e.g., `export DYN_LOG=debug`)
- `DYN_FILE_KV`: Directory for file-based KV store (default: `$TMPDIR/dynamo_store_kv`)
- `CUDA_VISIBLE_DEVICES`: Select GPUs for workers
- `OTEL_*`: OpenTelemetry tracing configuration

## Contribution Guidelines

### Issue-First Workflow (External Contributors)

- **Direct PR allowed** (no issue needed): Changes <100 lines addressing simple concerns (typos, simple bug fixes)
- **Issue required first**: Changes ≥100 lines, new features, architecture changes, or multi-component changes

Create a GitHub issue first using the Contribution Request template. Include:
- Problem description with context
- Proposed solution and approach
- Estimated PR size (XS/S/M/L/XL/XXL)
- Files affected and components touched
- Type of change (bug fix, new feature, refactoring, performance)

Wait for `approved-for-pr` label before submitting PR.

### PR Requirements

1. Link PR to approved issue using "Fixes #123" or "Closes #123"
2. Wait for Code Rabbit automated review and address suggestions (including nitpicks)
3. Ensure all CI tests pass
4. Review CODEOWNERS file to identify required reviewers
5. Sign off commits with DCO: `git commit -s` (adds `Signed-off-by: Name <email>`)

### Code Quality Standards

- Follow existing conventions and formatting (enforced by pre-commit hooks)
- Keep PRs focused on a single concern
- Build log must be clean (no warnings or errors)
- All tests must pass
- Demonstrate understanding of all code changes (AI-generated code is allowed but submitter must fully understand it)

## Testing Strategy

**Rust**: Unit tests in `lib/*/src/` modules, integration tests in `lib/*/tests/`, doc tests in inline documentation.

**Python**: Pytest with markers for test categorization:
- `unit`, `integration`, `e2e`: Test scope
- `gpu_0`, `gpu_1`, `gpu_2`, `gpu_4`, `gpu_8`: GPU requirements
- `vllm`, `sglang`, `trtllm`: Engine-specific
- `router`, `planner`, `kvbm`: Component-specific
- `pre_merge`, `post_merge`, `nightly`, `weekly`: CI scheduling

## Debugging

**Enable verbose logging**:
```bash
export DYN_LOG=debug
export RUST_BACKTRACE=1
```

**Check system configuration**:
```bash
./deploy/sanity_check.py [--thorough-check] [--json-output]
```

**Frontend OpenAPI spec generation** (without running server):
```bash
cargo run -p dynamo-llm --bin generate-frontend-openapi
# Outputs to: docs/frontends/openapi.json
```

**Inspect configuration at runtime**: Worker configuration is dumped to files via `dynamo.common.config_dump` module.

## Common Patterns

**Adding a new endpoint to a worker**:
1. Create handler function in `components/src/dynamo/{engine}/handlers.py`
2. Register endpoint in `main.py`: `component.endpoint("endpoint_name").serve(handler)`
3. Endpoint automatically discoverable via etcd at: `{namespace}.{component}.{endpoint_name}`

**Testing KV-aware routing**:
1. Start multiple workers with same model
2. Send requests with overlapping prefixes
3. Router logs show cache hit scores and worker selection
4. Check KV events in NATS: `nats sub "kv_events.>"`

**Working with disaggregated serving**:
- Prefill workers: `python -m dynamo.{engine} --disaggregated-serving prefill`
- Decode workers: `python -m dynamo.{engine} --disaggregated-serving decode`
- Aggregated (default): `python -m dynamo.{engine}` (handles both prefill and decode)

**Adding support for a new model**:
1. Check if tokenizer is supported (HuggingFace tokenizers in `lib/llm/src/tokenizers.rs`)
2. Add chat template if needed (`lib/llm/src/preprocessor.rs` or model's tokenizer_config.json)
3. For multimodal models, check `{engine}/multimodal_handlers/` for processor support
4. Test with frontend: ensure proper token encoding and template application

## Documentation

- **Official Docs**: https://docs.nvidia.com/dynamo/latest/index.html
- **Design Proposals**: https://github.com/ai-dynamo/enhancements
- **Support Matrix**: docs/reference/support-matrix.md
- **Recipes**: recipes/ directory
- **Examples**: examples/ directory
- **Architecture Deep-Dives**: docs/design_docs/
- **Backend-Specific Guides**: docs/backends/{vllm,sglang,trtllm}/

Build docs locally:
```bash
# Install docs dependencies
uv pip install -r pyproject.toml --group docs
cd docs
make html  # Output in docs/_build/html
```

## Kubernetes Deployment

See docs/kubernetes/README.md for detailed deployment guide.

```bash
# Install Dynamo operator
kubectl apply -f deploy/operator/

# Deploy workers via custom resources
kubectl apply -f examples/backends/{engine}/

# Monitor deployments
kubectl get dynamoworkloads -A
```

**Autoscaling**: Planner component monitors queue depths and worker utilization to scale prefill/decode workers independently. Configure via `planner.yaml` with SLA targets.

## Performance Benchmarking

```bash
# Compare deployment topologies
# See: docs/benchmarks/benchmarking.md

# SLA-driven profiling
# See: docs/planner/sla_planner_quickstart.md

# Use AIPerf for load testing
# See: benchmarks/llm/README.md
```

## Engine-Specific Notes

**vLLM**: Attempts to allocate full context length at startup. Use `--context-length` to reduce if OOM. Set `CUDA_VISIBLE_DEVICES` for multi-GPU. Supports disaggregated serving and NIXL KV transfer.

**SGLang**: Requires `libnuma-dev`. Native radix attention. Uses ZMQ for communication. Pass SGLang flags directly (see https://docs.sglang.ai/advanced_features/server_arguments.html).

**TensorRT-LLM**: Use NGC PyTorch container (version must match TensorRT-LLM version). Launch with `--shm-size=1g --ulimit memlock=-1`. Requires `libopenmpi-dev` and `libzmq3-dev` (for disaggregated serving). Install via `pip` (not `uv`) due to git dependencies.

## Troubleshooting

**"No workers available"**: Check etcd connection, ensure workers registered (`etcdctl get --prefix /services/`), verify NATS connectivity.

**KV routing not working**: Ensure `--kv-overlap-score-weight > 0`, check KV event publishing is enabled, verify NATS JetStream is running.

**Disaggregated serving errors**: Verify NIXL is installed (for KV transfer), check prefill queue in NATS (`nats stream info prefill_queue`), ensure both prefill and decode workers are running.

**Build failures**: Install all system dependencies listed above, ensure protoc is in PATH, check Rust version matches `rust-toolchain.toml`.

**Import errors**: Rebuild Python bindings (`cd lib/bindings/python && maturin develop --uv`), ensure virtual environment is activated.
