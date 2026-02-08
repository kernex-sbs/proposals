# DistriProc: Process-Aware Distributed Memory via Incremental Container Restore

**A Research Proposal for Process-Level Memory Disaggregation in Container Environments**

---

## Executive Summary

DistriProc enables Linux processes to execute with remotely hosted memory by combining incremental container restoration with network-backed paging, creating a process-aware distributed address space. Unlike existing solutions that either migrate entire containers (CRIU) or require specialized hardware (CXL), DistriProc provides software-defined memory disaggregation that works across heterogeneous edge-cloud environments.

**One-Line Summary**: DistriProc bridges the gap between expensive hardware-based memory disaggregation (CXL) and inflexible container migration by enabling Linux processes to execute while their memory remains distributed across machines.

---

## 1. Motivation & Context

### 1.1 The Memory Wall in Modern Computing

Modern applications increasingly span heterogeneous infrastructure:
- **Edge nodes**: Resource-constrained devices (Raspberry Pi, Jetson, mobile)
- **Cloud servers**: High-capacity compute with elastic memory
- **Accelerators**: GPUs, TPUs requiring large model storage

Yet today, **processes are rigidly bound to local memory**. Three fundamental problems persist:

#### Problem 1: Container Migration Downtime
- **Current State**: CRIU pre-copy migration requires transferring entire memory before execution resumes
- **Real Impact**: Redis migration over 1 Gbps network causes 2.7x service downtime (PCLive, SoCC 2024)
- **Limitation**: All memory must be local before process can run

#### Problem 2: Edge Memory Constraints
- **Current State**: AI inference models (3-20GB) exceed edge device capacity (1-4GB RAM)
- **Real Impact**: Models must be quantized or split, degrading accuracy
- **Limitation**: Cannot leverage cloud memory for edge computation

#### Problem 3: Cold Start Penalty
- **Current State**: Serverless functions with large memory footprints (ML models) take 30-60s to initialize
- **Real Impact**: 60% of serverless execution time wasted on startup (industry reports)
- **Limitation**: Cannot "warm start" with partial memory

### 1.2 Current Solutions Are Insufficient

| Approach | Limitation | Why It's Not Enough |
|----------|-----------|---------------------|
| **CRIU Pre-copy** | Full memory transfer required | Migration downtime scales with memory size |
| **PCLive (2024)** | Pipelined but still migrates everything | Memory still must eventually be local |
| **CRIU Lazy-Pages** | Exists but underoptimized | No hot/cold tracking, no placement policy |
| **CXL Disaggregation** | Rack-scale only, expensive hardware | Not suitable for edge/WAN scenarios |
| **RDMA Memory** | VM-level, high latency (100μs+) | Process semantics lost, not container-native |

### 1.3 Recent Validation

Three converging trends validate our approach:

1. **PCLive (SoCC 2024)**: Demonstrated 38.8x faster restore via pipelining → incremental restore is viable
2. **CXL Research Explosion**: 51 papers in 2024 alone on memory disaggregation → demand is real
3. **CRIU Lazy-Pages**: Documented but rarely used feature → foundation exists, needs optimization

**Key Insight**: PCLive proved processes can start before full memory arrives. We extend this: *processes can RUN indefinitely with remote memory*.

---

## 2. Core Idea

DistriProc proposes:

> **Linux processes whose virtual memory pages may reside on remote machines and are fetched on-demand during execution, while preserving full Linux process semantics (PIDs, VMAs, file descriptors, namespaces).**

### 2.1 What Makes This Different

Unlike classical Distributed Shared Memory (DSM), DistriProc is:

1. **Process-Aware**: Works with real Linux processes, not VMs or abstract memory regions
2. **Container-Native**: Integrates with Docker/Podman/containerd
3. **Application-Transparent**: No code changes required
4. **Hardware-Agnostic**: Runs on commodity networks (TCP/RDMA)
5. **Hybrid-Friendly**: Can leverage CXL when available as fast path

### 2.2 Three Founding Primitives

DistriProc builds on three existing Linux features:

```
1. CRIU Checkpoint/Restore
   └─> Captures/restores complete process state
   
2. userfaultfd (Linux 4.11+)
   └─> User-space page fault handling
   
3. Network Paging
   └─> TCP/RDMA page transfer
```

**Combined**: Process starts with skeleton state → faults on memory access → fetches page remotely → continues execution

---

## 3. System Architecture

### 3.1 High-Level Design

```
┌─────────────────────────────────────────────────────────────┐
│                     Source Node (Origin)                     │
│  ┌────────────┐      ┌──────────────────┐                  │
│  │ Container  │─────>│  Memory Server   │                  │
│  │  (Running) │      │  - Page index    │                  │
│  └────────────┘      │  - Hot/cold map  │                  │
│                      │  - TCP/RDMA srv  │                  │
│                      └──────────────────┘                  │
└──────────────────────────────┬──────────────────────────────┘
                               │
                               │ Network (TCP/RDMA)
                               │ Page requests/responses
                               │
┌──────────────────────────────┴──────────────────────────────┐
│                  Destination Node (Restore)                  │
│  ┌────────────────────────────────────────────────────────┐ │
│  │            DistriProc Runtime Manager                   │ │
│  │  ┌──────────────┐  ┌─────────────┐  ┌──────────────┐  │ │
│  │  │ Restore      │  │ userfaultfd │  │ Page Fetcher │  │ │
│  │  │ Controller   │─>│ Handler     │─>│ (network)    │  │ │
│  │  └──────────────┘  └─────────────┘  └──────────────┘  │ │
│  │         │                  │                 │          │ │
│  └─────────┼──────────────────┼─────────────────┼─────────┘ │
│            │                  │                 │            │
│  ┌─────────▼──────────────────▼─────────────────▼─────────┐ │
│  │              Restored Process                           │ │
│  │  - Namespaces restored                                  │ │
│  │  - Process tree created                                 │ │
│  │  - VMAs registered with userfaultfd                     │ │
│  │  - Executing with partial memory                        │ │
│  └────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### 3.2 Component Details

#### A. Source Node: Memory Server

**Responsibilities**:
- Holds original container checkpoint
- Serves page requests
- Tracks access patterns for optimization

**Implementation**:
```python
class MemoryServer:
    def __init__(self, checkpoint_dir):
        self.pages = PageIndex.from_criu(checkpoint_dir)
        self.access_log = AccessTracker()
        self.cache = LRUCache(size="10GB")
        
    async def handle_page_request(self, page_addr):
        # Log access for hot/cold analysis
        self.access_log.record(page_addr)
        
        # Serve from cache or disk
        if page_addr in self.cache:
            return self.cache[page_addr]
        
        page_data = self.pages.read(page_addr)
        self.cache.insert(page_addr, page_data)
        return page_data
```

**Key Features**:
- Page prefetching based on access patterns
- Compression for cold pages
- RDMA fast path for hot pages

#### B. Destination Node: DistriProc Runtime

**Phase 1: Skeleton Restore**
```bash
# 1. Restore global state (namespaces, cgroups)
criu restore --lazy-pages --images-dir=/checkpoint

# 2. DistriProc takes over
distri-proc-daemon \
  --checkpoint-dir=/checkpoint \
  --page-server=source.host:9000 \
  --prefetch-policy=sequential \
  --rdma-enabled
```

**Phase 2: Page Fault Handling**
```c
// userfaultfd handler pseudocode
void handle_page_fault(void *addr) {
    // 1. Receive fault notification
    struct uffd_msg msg;
    read(uffd, &msg, sizeof(msg));
    
    // 2. Determine page address
    void *page_addr = (void *)msg.arg.pagefault.address;
    
    // 3. Fetch from remote
    void *page_data = fetch_remote_page(page_addr);
    
    // 4. Map into process
    struct uffdio_copy copy = {
        .dst = (unsigned long)page_addr,
        .src = (unsigned long)page_data,
        .len = PAGE_SIZE,
    };
    ioctl(uffd, UFFDIO_COPY, &copy);
}
```

**Phase 3: Background Migration**
```python
class BackgroundMigrator:
    async def run(self):
        while self.has_remaining_pages():
            # Prioritize hot pages
            page = self.policy.next_page()
            
            # Fetch without blocking process
            data = await self.fetch(page)
            
            # Map preemptively
            self.map_page(page, data)
            
            # Rate limit to avoid network saturation
            await self.rate_limiter.wait()
```

### 3.3 Memory Transport Layer

**Tiered Transport Strategy**:

```
┌──────────────────────────────────────────────────────────┐
│                    Transport Hierarchy                    │
├──────────────────────────────────────────────────────────┤
│  Tier 1: Local Cache        │ Latency: ~100ns            │
│  - Recently accessed pages   │ Bandwidth: Memory speed   │
├──────────────────────────────────────────────────────────┤
│  Tier 2: RDMA (Hot Pages)    │ Latency: 1-10μs           │
│  - Frequently accessed       │ Bandwidth: 100 Gbps       │
├──────────────────────────────────────────────────────────┤
│  Tier 3: TCP (Cold Pages)    │ Latency: 100-500μs        │
│  - Infrequently accessed     │ Bandwidth: 1-10 Gbps      │
├──────────────────────────────────────────────────────────┤
│  Tier 4: Optional CXL        │ Latency: 150-200ns        │
│  - For rack-scale fast path  │ Bandwidth: 64-256 GB/s    │
└──────────────────────────────────────────────────────────┘
```

**Protocol Design**:
```protobuf
// Page request protocol
message PageRequest {
  uint64 page_addr = 1;
  bool prefetch_hint = 2;
  uint32 batch_size = 3;
}

message PageResponse {
  uint64 page_addr = 1;
  bytes data = 2;
  bool compressed = 3;
  repeated uint64 suggested_prefetch = 4;
}
```

---

## 4. Novel Contributions

### 4.1 Process-Aware DSM

**Unlike classical DSM** (TreadMarks, Munin, GiantVM):
- Memory is tied to **real Linux processes** with PIDs, not abstract shared regions
- Preserves **process tree relationships** (parent/child, sessions)
- Maintains **file descriptor semantics** (open files, sockets, pipes)
- Respects **namespace isolation** (PID, mount, network, IPC)

**Comparison Table**:

| Feature | Classical DSM | VM-level (GiantVM) | DistriProc |
|---------|---------------|-------------------|------------|
| Granularity | Page-level | VM-level | Process-level |
| Semantics | Shared memory | Virtual machine | Linux process |
| Container Support | No | No | **Yes** |
| Namespace Aware | No | No | **Yes** |
| Application Changes | Often required | No | No |

### 4.2 Incremental Execution Model

**Key Innovation**: Processes execute while memory ownership migrates

```
Traditional Migration:
Time: |----Checkpoint----|----Transfer----|----Restore----|----Run----|
State: Running           Frozen           Frozen          Running

PCLive (Pipelined):
Time: |----Checkpoint----|----Restore----|----Run----|
State: Running           Frozen          Running
Note: Memory still transfers fully (just pipelined)

DistriProc:
Time: |----Checkpoint----|----Run + Gradual Migration----|
State: Running           Running (with remote pages)
Note: Memory ownership migrates over time, but never required to be complete
```

### 4.3 Container-Integrated Remote Paging

**Not VM-level, not application-level → OS-process-level**

Benefits:
1. **Container Ecosystem Integration**: Works with Docker, Kubernetes, Podman
2. **Granular Control**: Per-process memory policies
3. **Resource Accounting**: cgroup-aware memory tracking
4. **Security**: Namespace isolation preserved

### 4.4 Edge-Cloud Memory Decoupling

**Unique Use Case**: Compute on edge, memory in cloud

Example: AI inference at edge
```
┌─────────────────┐                    ┌──────────────────┐
│  Edge Device    │                    │   Cloud Server   │
│  (Jetson Nano)  │<---Remote Paging-->│                  │
│                 │                    │  Model Params    │
│  - Inference    │                    │  - Weights: 8GB  │
│  - Local RAM:   │                    │  - Activations   │
│    2GB          │                    │  - KV cache      │
└─────────────────┘                    └──────────────────┘
```

---

## 5. Related Work & Positioning

### 5.1 Detailed Comparison

#### vs. PCLive (SoCC 2024)

| Aspect | PCLive | DistriProc |
|--------|--------|-----------|
| **Core Technique** | Pipelined restore | Persistent remote paging |
| **Memory Transfer** | Eventually complete | Indefinitely distributed |
| **Use Case** | Fast migration | Edge computing, memory pooling |
| **Downtime** | ~2.7x reduction | Potentially zero (instant start) |
| **Post-Migration** | All memory local | Memory remains remote |

**Our Advantage**: PCLive still requires destination to eventually hold all memory. We break this assumption.

#### vs. CRIU Lazy-Pages

| Aspect | CRIU Lazy-Pages | DistriProc |
|--------|----------------|-----------|
| **Status** | Basic implementation | Full system with optimizations |
| **Hot/Cold Tracking** | No | **Yes** (echo 1 > /proc/pid/clear_refs) |
| **Prefetching** | None | **Sequential, ML-based** |
| **Transport** | TCP only | **TCP, RDMA, optional CXL** |
| **Page Placement** | FIFO | **Hot-page identification** |
| **Production Ready** | No | **Goal of this work** |

**Our Contribution**: We take the underused CRIU feature and make it production-grade.

#### vs. CXL-Based Disaggregation

| Aspect | CXL Systems | DistriProc |
|--------|------------|-----------|
| **Hardware** | CXL switches ($$$) | Commodity network |
| **Distance** | Rack-scale (<100m) | **Cross-datacenter, WAN** |
| **Latency** | 150-200ns | 1-500μs (acceptable tradeoff) |
| **Deployment** | Requires infrastructure overhaul | **Software-only** |
| **Cost** | High (CXL 3.0 immature) | **Low (existing RDMA/TCP)** |

**Our Positioning**: Software-first disaggregation as CXL alternative. Hybrid mode can use CXL as fast path when available.

#### vs. Traditional RDMA Memory Disaggregation

| Aspect | RDMA Systems (Infiniswap, Memtrade) | DistriProc |
|--------|-------------------------------------|-----------|
| **Abstraction** | Block/swap device | **Process memory** |
| **Granularity** | Page swapping | **Live page faulting** |
| **Latency** | 100μs+ swap latency | **1-10μs RDMA page fetch** |
| **Process Awareness** | No | **Full (PIDs, namespaces)** |
| **Container Support** | Limited | **Native** |

### 5.2 Research Gap Analysis

**Gap 1**: Process-level DSM for containers doesn't exist
- Classical DSM: Dead since early 2000s (too complex, slow networks)
- Modern DSM: VM-level (GiantVM) or application-level (databases)
- **DistriProc**: Process-native, container-friendly

**Gap 2**: CRIU lazy-pages is documented but underexplored
- Only 3 academic papers mention it (since 2017)
- No optimization work published
- **DistriProc**: First comprehensive optimization + evaluation

**Gap 3**: No edge-cloud memory decoupling systems
- CXL work: datacenter-focused
- RDMA work: cloud-focused
- **DistriProc**: Edge → Cloud scenarios

---

## 6. Implementation Plan

### 6.1 Phase 1: Minimal Viable Prototype (6 weeks)

**Goal**: Demonstrate a process running with remote memory

**Setup**:
- Two Linux machines (Ubuntu 24.04, kernel 6.8+)
  - Node A: Laptop (source)
  - Node B: Raspberry Pi 4 (destination)
- Network: Gigabit Ethernet

**Deliverables**:
```
Week 1-2: Infrastructure
- CRIU installation and testing
- Basic checkpoint/restore verification
- Network connectivity setup

Week 3-4: Core Implementation
- userfaultfd handler (C)
- TCP page server
- Basic restore skeleton

Week 5-6: Integration & Demo
- End-to-end test with simple workload
- Redis instance with remote memory
- Performance logging
```

**Milestone**: Run Redis with 50% memory remote, serve requests successfully

**Code Structure**:
```
distri-proc/
├── src/
│   ├── page_server/      # Remote memory server
│   ├── uffd_handler/     # Page fault handler
│   ├── restore_mgr/      # Restore orchestration
│   └── transport/        # TCP/RDMA abstraction
├── tests/
│   ├── unit/
│   └── integration/
└── examples/
    ├── redis/
    └── nginx/
```

**Success Criteria**:
- [ ] Process starts within 100ms of restore
- [ ] Page faults handled correctly
- [ ] No crashes for 1-hour run
- [ ] Basic latency measurements collected

### 6.2 Phase 2: Optimization (5 weeks)

**Week 7-8: Hot/Cold Classification**

Implement page access tracking:
```bash
# Mark all pages as unaccessed
echo 1 > /proc/$PID/clear_refs

# Run workload for 10 seconds
sleep 10

# Read access bits
cat /proc/$PID/smaps | grep Referenced

# Classify:
# Hot: accessed > 5 times in 10s window
# Warm: accessed 1-5 times
# Cold: not accessed
```

**Week 9-10: Prefetching Strategies**

1. **Sequential Prefetcher**:
```python
class SequentialPrefetcher:
    def on_fault(self, addr):
        # Fetch faulted page + next N pages
        pages = [addr + i*PAGE_SIZE for i in range(N)]
        return self.batch_fetch(pages)
```

2. **Stride Prefetcher**:
```python
class StridePrefetcher:
    def __init__(self):
        self.history = []
    
    def on_fault(self, addr):
        # Detect stride pattern
        if len(self.history) >= 2:
            stride = addr - self.history[-1]
            if stride == self.history[-1] - self.history[-2]:
                # Prefetch based on stride
                return [addr + i*stride for i in range(1, 4)]
```

**Week 11: RDMA Integration**

```c
// RDMA read for hot pages
struct ibv_sge sge = {
    .addr = (uintptr_t)local_buf,
    .length = PAGE_SIZE,
    .lkey = mr->lkey
};

struct ibv_send_wr wr = {
    .opcode = IBV_WR_RDMA_READ,
    .send_flags = IBV_SEND_SIGNALED,
    .wr.rdma = {
        .remote_addr = remote_page_addr,
        .rkey = remote_key
    }
};
```

**Deliverables**:
- [ ] Hot page identification (>80% accuracy)
- [ ] 2x prefetch hit rate vs. no prefetching
- [ ] RDMA integration working
- [ ] Latency reduced by 50% for hot pages

### 6.3 Phase 3: Evaluation (4 weeks)

**Week 12-13: Benchmarking**

Workloads:
1. **Redis** (key-value store)
   - YCSB workload A (50/50 read/write)
   - Dataset: 10GB
   - Measure: throughput, P99 latency

2. **ML Inference** (PyTorch)
   - ResNet-50 inference
   - Model: 100MB
   - Dataset: ImageNet (streamed)
   - Measure: inference latency, throughput

3. **NGINX** (web server)
   - Serve static content (1GB total)
   - Apache Bench load test
   - Measure: requests/sec, latency distribution

**Week 14-15: Analysis & Paper Writing**

Metrics to collect:
```python
class Metrics:
    # Performance
    page_fault_latency: List[float]
    prefetch_hit_rate: float
    network_bandwidth: float
    
    # System
    cpu_overhead: float
    memory_local_vs_remote: Tuple[int, int]
    
    # Application
    throughput: float
    p50_latency: float
    p99_latency: float
```

**Deliverables**:
- [ ] Complete benchmark results
- [ ] Comparison with CRIU baseline
- [ ] Comparison with PCLive (if available)
- [ ] Paper draft

### 6.4 Phase 4 (Optional): Advanced Features (4 weeks)

**Week 16-17: Write Handling**

Challenge: How to handle modified pages?

**Strategy 1: Write-Through**
```python
def handle_write_fault(addr):
    # Update local page
    local_page[addr] = new_data
    
    # Immediately propagate to source
    async_write_back(addr, new_data)
```

**Strategy 2: Write-Back with Dirty Tracking**
```python
def handle_write_fault(addr):
    # Mark page as dirty
    dirty_pages.add(addr)
    
    # Batch write-back every 1s
    if time.now() - last_writeback > 1.0:
        batch_writeback(dirty_pages)
        dirty_pages.clear()
```

**Week 18-19: Multi-Node Support**

```
Source Node A ──┐
                ├──> Destination Node (runs process)
Source Node B ──┘
                └──> Memory split across sources
```

**Implementation**:
```python
class MultiSourceManager:
    def __init__(self, sources: List[str]):
        self.sources = sources
        self.page_map = {}  # addr -> source_id
    
    def fetch_page(self, addr):
        source = self.page_map[addr]
        return self.sources[source].fetch(addr)
```

---

## 7. Evaluation Methodology

### 7.1 Experimental Setup

**Hardware**:
- Source Node: 
  - Dell Server, 64GB RAM, 16-core Intel Xeon
  - 25 Gbps NIC (Mellanox ConnectX-5)
  
- Destination Node:
  - Raspberry Pi 4, 4GB RAM, 4-core ARM Cortex-A72
  - Gigabit Ethernet
  
- Alternative Destination:
  - AWS EC2 t3.medium (for cloud evaluation)

**Network**:
- LAN: Gigabit Ethernet (latency ~0.2ms)
- WAN: Simulated with `tc` (latency 10-50ms)
- RDMA: InfiniBand (latency <2μs)

### 7.2 Baseline Comparisons

1. **CRIU Baseline**: Full pre-copy migration
2. **CRIU Lazy-Pages**: Basic implementation without optimizations
3. **PCLive**: If implementation available, otherwise cite paper numbers
4. **Local Execution**: Upper bound (all memory local)

### 7.3 Key Metrics

**Primary Metrics**:
```
1. Startup Latency
   - Time from restore command to process ready
   - Target: <1 second for 10GB memory

2. Page Fault Latency
   - Per-fault handling time
   - Target: <10μs for RDMA, <500μs for TCP

3. Application Throughput
   - Redis: ops/sec
   - ML: inferences/sec
   - Target: >70% of local performance

4. Steady-State P99 Latency
   - Target: <2x local latency
```

**Secondary Metrics**:
```
1. Network Bandwidth Usage
2. CPU Overhead (% spent in page fault handling)
3. Memory Efficiency (local cache hit rate)
4. Prefetch Accuracy
5. Background Migration Progress
```

### 7.4 Experimental Matrix

| Workload | Local Memory % | Network | Expected Result |
|----------|---------------|---------|----------------|
| Redis | 10% | LAN | Startup <100ms, throughput >50% |
| Redis | 50% | LAN | Startup <500ms, throughput >80% |
| Redis | 10% | WAN | Startup <100ms, throughput >30% |
| ML Inference | 20% | LAN | Latency <2x local |
| NGINX | 30% | LAN | Throughput >70% local |

### 7.5 Evaluation Questions

1. **How much memory can be remote before performance degrades significantly?**
   - Hypothesis: 50% remote is acceptable for most workloads

2. **Which prefetching strategy works best?**
   - Test: Sequential vs. Stride vs. ML-based

3. **Does RDMA provide meaningful improvement over TCP?**
   - Compare latency distributions

4. **How does it scale with network latency?**
   - Test LAN (0.2ms) vs. WAN (10-50ms)

5. **What's the CPU overhead of page fault handling?**
   - Measure with `perf` profiling

---

## 8. Use Cases & Applications

### 8.1 Zero-Downtime Container Migration

**Scenario**: Kubernetes node maintenance

```yaml
# Traditional approach
kubectl drain node-1  # 30-60s downtime per pod

# DistriProc approach
kubectl annotate pod my-app distri-proc/enable=true
kubectl drain node-1 --grace-period=1  # <1s downtime
```

**Benefits**:
- Instant failover
- Continuous service availability
- SLA compliance

### 8.2 Edge AI with Cloud-Backed Memory

**Scenario**: Drone image recognition

```
Drone (Edge)                    Cloud Server
├─ Camera (input)               ├─ Model weights (8GB)
├─ CPU (inference)              ├─ Historical data
└─ RAM: 2GB (activations)       └─ Feature database
    ↑                                 ↓
    └──── Remote paging over 5G ────┘
```

**Benefits**:
- Run large models on small devices
- No model quantization needed
- Dynamic model updates

### 8.3 Instant Startup for Large AI Workers

**Scenario**: On-demand LLM inference

```python
# Traditional: 30-60s cold start
llm = LargeLanguageModel("llama-70b")  # Load 140GB model
llm.generate(prompt)

# DistriProc: <1s warm start
llm = DistriProc.restore("llama-70b-snapshot")
llm.generate(prompt)  # Fault in only needed pages
```

**Benefits**:
- Sub-second startup
- Pay only for pages used
- Better resource utilization

### 8.4 Live Debugging via Remote Process Cloning

**Scenario**: Production debugging

```bash
# Clone production process to dev machine
distri-proc clone \
  --source=prod-server:redis-prod \
  --dest=localhost \
  --memory=lazy

# Debug on local machine without affecting production
gdb attach $(pgrep redis-server)
```

**Benefits**:
- No production disruption
- Full state access
- Reproducible debugging

### 8.5 Memory Pooling Across Machines

**Scenario**: Research cluster with uneven memory usage

```
Node A: 128GB RAM, 30% used  ─┐
Node B: 64GB RAM, 90% used    ├─> Shared memory pool
Node C: 256GB RAM, 20% used  ─┘
```

**Benefits**:
- Higher cluster-wide utilization
- Elastic memory allocation
- Cost savings

### 8.6 Failure Recovery Without Restart

**Scenario**: Long-running computation

```python
# Checkpoint every hour (minimal overhead)
checkpoint_daemon.set_interval(3600)

# On failure, instantly restore
if node_fails:
    distri-proc restore --checkpoint=latest
    # Process resumes with remote memory
```

**Benefits**:
- Fast recovery (<1s vs. restart)
- Minimal work lost
- Continuous progress

### 8.7 Serverless Cold Start Optimization

**Scenario**: AWS Lambda with ML model

```javascript
// Traditional: Load model on every cold start
exports.handler = async (event) => {
    const model = await loadModel();  // 30s cold start!
    return model.predict(event.data);
};

// DistriProc: Pre-warmed snapshot
exports.handler = async (event) => {
    // Restored from snapshot with lazy memory
    return model.predict(event.data);  // <1s startup
};
```

---

## 9. Risk Analysis & Mitigation

### 9.1 Technical Risks

| Risk | Probability | Impact | Mitigation |
|------|-------------|--------|-----------|
| **Page fault latency too high** | High | High | RDMA fast path, aggressive prefetching |
| **Network becomes bottleneck** | Medium | High | Adaptive rate limiting, compression |
| **Write consistency issues** | Medium | Medium | Write-through for critical pages |
| **CRIU compatibility problems** | Low | Medium | Use stable CRIU 3.19+ |
| **userfaultfd limitations** | Low | Low | Test with newer kernels (6.8+) |

### 9.2 Detailed Risk Mitigation

#### Risk 1: Excessive Page Fault Latency

**Problem**: If every page fault takes 100-500μs (TCP), application performance degrades.

**Mitigation Strategy**:
```
1. Hot Page Identification
   - Track access frequency
   - Eagerly fetch hot pages to local cache
   
2. Prefetching
   - Sequential: Next N pages
   - Stride-based: Pattern detection
   - ML-based: Learn access patterns
   
3. RDMA for Hot Pages
   - 1-10μs latency vs. 100-500μs TCP
   - 10-50x improvement for critical pages
   
4. Adaptive Caching
   - LRU eviction for local cache
   - Target 70%+ hit rate
```

**Acceptable Performance Envelope**:
- 80% pages: <10μs (RDMA or cache)
- 15% pages: <100μs (TCP, infrequent)
- 5% pages: <500μs (cold pages, rare)

#### Risk 2: Network Saturation

**Problem**: Fetching pages at full speed saturates network.

**Mitigation**:
```python
class RateLimiter:
    def __init__(self, target_bw_mbps=800):
        self.target = target_bw_mbps * 1_000_000 / 8  # bytes/sec
        self.tokens = self.target
        
    async def wait_for_page(self):
        while self.tokens < PAGE_SIZE:
            await asyncio.sleep(0.001)
        self.tokens -= PAGE_SIZE
```

**Adaptive Strategy**:
- Monitor RTT
- Reduce fetch rate if RTT increases
- Prioritize critical pages

#### Risk 3: Write Consistency

**Problem**: Modified pages need to be synced.

**Approach 1: Write-Through** (simple, high latency)
```
Every write → immediate network write
Pros: Consistent
Cons: High latency
```

**Approach 2: Write-Back** (complex, low latency)
```
Dirty pages tracked locally
Batch write-back every T seconds
Pros: Low latency
Cons: Requires consistency protocol
```

**Chosen Strategy**: Start with read-only workloads, add write-back in Phase 4.

### 9.3 Research Risks

| Risk | Mitigation |
|------|-----------|
| **Results don't show improvement** | Focus on specific use cases where remote memory is acceptable |
| **Implementation too complex** | Start with simple TCP version, add optimizations incrementally |
| **Baseline unavailable for comparison** | Use CRIU numbers, cite PCLive paper |

---

## 10. Expected Outcomes

### 10.1 Quantitative Goals

**Performance Targets**:
```
1. Startup Latency
   - DistriProc: <1s for 10GB memory
   - CRIU baseline: 30-60s
   - Target: 30-60x improvement

2. Throughput (Redis)
   - Local: 100,000 ops/sec (baseline)
   - DistriProc (50% remote): >70,000 ops/sec
   - Target: >70% of local

3. Latency (Redis P99)
   - Local: 1ms
   - DistriProc: <2ms
   - Target: <2x local

4. Network Efficiency
   - Prefetch hit rate: >60%
   - Bandwidth usage: <500 MB/s sustained
```

### 10.2 Qualitative Outcomes

1. **Working Prototype**
   - Production-ready codebase
   - Documentation
   - Example deployments

2. **Open Source Release**
   - GitHub repository
   - Docker images
   - Kubernetes integration

3. **Research Contributions**
   - Conference paper (OSDI/SOSP/EuroSys)
   - Possibly systems track at SIGCOMM/NSDI
   - Workshop presentations

4. **Industry Impact**
   - Potential adoption by container orchestrators
   - Edge computing platforms
   - Serverless providers

### 10.3 Success Criteria

**Minimum Viable Success**:
- [ ] Prototype works for read-only workloads
- [ ] Demonstrates <1s startup vs. 30s baseline
- [ ] Achieves >50% throughput of local execution
- [ ] Published paper at systems venue

**Stretch Goals**:
- [ ] Write support with consistency guarantees
- [ ] Multi-node memory pooling
- [ ] Kubernetes operator
- [ ] Industry partnership

---

## 11. Timeline & Milestones

### 11.1 15-Week Detailed Schedule

```
Week 1-2: Infrastructure Setup
├─ Install CRIU, test checkpoint/restore
├─ Network setup (LAN + RDMA)
└─ Milestone: Basic CRIU working

Week 3-4: Core Implementation
├─ userfaultfd handler (C)
├─ TCP page server
└─ Milestone: First remote page fetch

Week 5-6: Integration
├─ End-to-end restore flow
├─ Redis test application
└─ Milestone: Redis running with remote memory

Week 7-8: Hot/Cold Classification
├─ Access tracking (/proc/pid/clear_refs)
├─ Page classification algorithm
└─ Milestone: 80% hot page accuracy

Week 9-10: Prefetching
├─ Sequential prefetcher
├─ Stride prefetcher
└─ Milestone: 2x prefetch hit rate

Week 11: RDMA Integration
├─ RDMA transport layer
├─ Hot page fast path
└─ Milestone: <10μs hot page latency

Week 12-13: Benchmarking
├─ Redis (YCSB)
├─ ML inference (PyTorch)
├─ NGINX
└─ Milestone: Complete benchmark suite

Week 14-15: Analysis & Writing
├─ Data analysis
├─ Paper draft
└─ Milestone: Camera-ready paper

Optional Phase 4 (Week 16-19):
├─ Write handling
├─ Multi-node support
├─ Kubernetes integration
└─ Milestone: Production-ready system
```

### 11.2 Go/No-Go Decision Points

**Week 6 Decision**:
- Can we restore a process with remote memory?
- If NO → Pivot to optimizing CRIU lazy-pages
- If YES → Continue to optimization phase

**Week 11 Decision**:
- Do we achieve <2ms P99 latency for Redis?
- If NO → Focus on specific workloads where it works
- If YES → Expand to more workloads

**Week 15 Decision**:
- Do we have compelling results for a paper?
- If NO → Consider SOSP/OSDI poster or workshop
- If YES → Submit to top-tier venue

---

## 12. Resource Requirements

### 12.1 Hardware

**Essential**:
- 2x Linux servers (64GB RAM, 10GbE NIC) - $0 (use lab resources)
- 1x Raspberry Pi 4 (8GB) - $75
- Network switch (10GbE) - $200

**Optional** (for RDMA):
- 2x Mellanox ConnectX-5 NICs - $800
- InfiniBand switch - $1,500

**Total**: $275 (basic) to $2,575 (with RDMA)

### 12.2 Software

All open-source:
- Ubuntu 24.04 LTS
- Linux kernel 6.8+
- CRIU 3.19+
- Python 3.11+
- Rust 1.70+ (for performance-critical components)

### 12.3 Personnel

**Primary Researcher**: 15 weeks full-time
- Systems programming (C, Rust)
- Kernel development experience
- Network programming

**Advisor**: 2 hours/week
- Guidance on research direction
- Paper writing

**Optional**: 1 undergraduate assistant
- Benchmarking
- Documentation

### 12.4 Cloud Resources (Optional)

For WAN testing:
- AWS EC2 instances: ~$500/month
- Azure VMs: ~$400/month
- Google Cloud: ~$450/month

**Total budget**: $1,000-$1,500 for 3 months

---

## 13. Future Research Directions

### 13.1 Short-Term (1-2 years)

**1. ML-Based Prefetching**
```python
class MLPrefetcher:
    def __init__(self):
        self.model = LSTM(input_size=64, hidden_size=128)
        
    def predict_next_pages(self, history):
        # Learn access patterns
        features = self.extract_features(history)
        next_pages = self.model(features)
        return next_pages
```

**2. CXL Hybrid Mode**
```
DistriProc with CXL fast path:
- Rack-local pages: CXL (150-200ns)
- Cross-rack pages: RDMA (1-10μs)
- WAN pages: TCP (100-500μs)
```

**3. eBPF-Based Page Tracking**
```c
// eBPF program for efficient page access tracking
SEC("tracepoint/page_fault")
int trace_page_fault(struct trace_event_raw_mm_fault *ctx) {
    u64 addr = ctx->address;
    access_map.update(&addr, &timestamp);
    return 0;
}
```

### 13.2 Medium-Term (2-4 years)

**1. Kubernetes Native Integration**
```yaml
apiVersion: v1
kind: Pod
metadata:
  annotations:
    distri-proc.io/enabled: "true"
    distri-proc.io/memory-policy: "hot-local"
    distri-proc.io/remote-memory: "50%"
spec:
  containers:
  - name: app
    image: myapp:latest
```

**2. Multi-Node Memory Graphs**
```
Process P1 (Node A)
  └─ Memory split:
      ├─ Hot pages: Local (Node A)
      ├─ Warm pages: Node B (RDMA)
      └─ Cold pages: Node C (TCP)
```

**3. Serverless Integration**
```javascript
// AWS Lambda with DistriProc
exports.handler = DistriProc.wrap(async (event) => {
    // Instant startup with lazy memory loading
    return model.predict(event);
});
```

### 13.3 Long-Term (4+ years)

**1. Hardware Acceleration**
- Custom FPGA for page fault handling
- Smart NIC integration
- CXL memory expanders

**2. Distributed OS Vision**
```
Future datacenter:
├─ Compute pool (CPUs)
├─ Memory pool (DRAM + CXL)
├─ Storage pool (NVMe + object storage)
└─ Accelerator pool (GPUs, TPUs)

All connected via DistriProc-like abstraction
```

**3. Language Runtime Integration**
```java
// JVM with distributed heap
public class DistriProcJVM {
    // Heap pages can be on remote nodes
    // GC-aware remote memory management
}
```

---

## 14. Broader Impact

### 14.1 Academic Impact

**Research Questions Opened**:
1. What are optimal page placement policies for edge-cloud systems?
2. How can we predict memory access patterns in containerized apps?
3. Can we build a consistent distributed process abstraction?

**Community Benefits**:
- Open-source toolkit for memory disaggregation research
- Benchmark suite for container migration
- Production trace datasets

### 14.2 Industry Impact

**Immediate Applications**:
1. **Cloud Providers**: Reduce VM migration downtime
2. **Edge Platforms**: Enable AI inference on constrained devices
3. **CDN Operators**: Instant content cache warm-up

**Economic Impact**:
- Reduced downtime → higher SLA compliance
- Better utilization → lower infrastructure costs
- Faster deployment → improved developer productivity

### 14.3 Educational Impact

**Coursework Integration**:
- Operating systems (process memory management)
- Distributed systems (remote paging protocols)
- Systems programming (userfaultfd, CRIU)

**Open Source Contributions**:
- Documentation for CRIU lazy-pages
- Tutorials for userfaultfd
- Reference implementations

---

## 15. Conclusion

### 15.1 Summary of Contributions

DistriProc makes three key contributions:

1. **Process-Aware DSM for Containers**
   - First system to enable distributed memory for real Linux processes
   - Preserves full process semantics (PIDs, namespaces, FDs)
   - Container ecosystem integration

2. **Incremental Execution Model**
   - Processes run while memory ownership migrates
   - No requirement for complete local memory
   - Enables instant startup for large applications

3. **Software-Defined Memory Disaggregation**
   - Works over commodity networks (TCP/RDMA)
   - No specialized hardware required
   - Hybrid mode can leverage CXL when available

### 15.2 Why This Matters

**Fundamental Question**: Must memory be local to execution?

**Traditional Answer**: Yes (since Unix in 1970s)

**DistriProc's Answer**: No, not anymore.

This enables:
- Distributed operating systems
- Flexible edge computing architectures
- New cloud abstractions

### 15.3 Call to Action

This research sits at the intersection of:
- **Containers** (largest deployment since VMs)
- **Kernel memory management** (40+ years of research)
- **Distributed systems** (next-generation datacenters)

**Timing is Perfect**:
- PCLive (2024) validated incremental restore
- CXL hype creates demand for disaggregation
- CRIU lazy-pages exists but underexplored

**We Have the Tools**:
- userfaultfd (since Linux 4.11)
- CRIU (mature, production-ready)
- RDMA/CXL (fast networks)

**Research Gap is Clear**:
- No process-level DSM for containers
- No edge-cloud memory decoupling
- No optimization of CRIU lazy-pages

---

## 16. References & Further Reading

### 16.1 Primary References

**CRIU & Container Migration**:
1. CRIU Documentation: https://criu.org
2. PCLive (SoCC 2024): "Pipelined Restoration of Application Containers for Reduced Service Downtime"
3. Efficient Live Migration of Linux Containers (IOSR-JCE 2018)

**Memory Disaggregation**:
4. Rcmp (ACM TACO 2024): "Reconstructing RDMA-Based Memory Disaggregation via CXL"
5. Memory Disaggregation: Advances and Open Challenges (2023)
6. Pond (ASPLOS 2023): "CXL-Based Memory Pooling Systems for Cloud Platforms"

**Linux Kernel**:
7. userfaultfd Documentation: https://www.kernel.org/doc/html/latest/admin-guide/mm/userfaultfd.html
8. Linux Memory Management: https://www.kernel.org/doc/html/latest/admin-guide/mm/

### 16.2 Related Projects

- **Infiniswap**: RDMA-based remote memory swapping
- **Memtrade**: Memory marketplace for VMs
- **GiantVM**: VM-level DSM
- **Firecracker**: Lightweight VM for serverless

---

## Appendix A: Technical Details

### A.1 userfaultfd API Example

```c
#include <linux/userfaultfd.h>
#include <sys/ioctl.h>
#include <sys/syscall.h>

int setup_userfaultfd(void *addr, size_t len) {
    // Create userfaultfd
    int uffd = syscall(__NR_userfaultfd, O_CLOEXEC | O_NONBLOCK);
    
    // Enable features
    struct uffdio_api api = {
        .api = UFFD_API,
        .features = UFFD_FEATURE_MISSING_HUGETLBFS |
                    UFFD_FEATURE_MISSING_SHMEM
    };
    ioctl(uffd, UFFDIO_API, &api);
    
    // Register memory region
    struct uffdio_register reg = {
        .range = { .start = (unsigned long)addr, .len = len },
        .mode = UFFDIO_REGISTER_MODE_MISSING
    };
    ioctl(uffd, UFFDIO_REGISTER, &reg);
    
    return uffd;
}

void handle_page_faults(int uffd) {
    struct uffd_msg msg;
    
    while (1) {
        read(uffd, &msg, sizeof(msg));
        
        if (msg.event == UFFD_EVENT_PAGEFAULT) {
            void *page_addr = (void *)msg.arg.pagefault.address;
            
            // Fetch page remotely
            void *page_data = fetch_remote_page(page_addr);
            
            // Copy into process
            struct uffdio_copy copy = {
                .dst = (unsigned long)page_addr,
                .src = (unsigned long)page_data,
                .len = 4096,
                .mode = 0
            };
            ioctl(uffd, UFFDIO_COPY, &copy);
        }
    }
}
```

### A.2 CRIU Lazy-Pages Command Reference

```bash
# Source: Checkpoint without writing memory
criu dump \
  --tree $PID \
  --images-dir /checkpoint \
  --lazy-pages \
  --address 192.168.1.100 \
  --port 9000

# Source: Start page server
criu page-server \
  --images-dir /checkpoint \
  --lazy-pages \
  --address 192.168.1.100 \
  --port 9000

# Destination: Start lazy-pages daemon
criu lazy-pages \
  --images-dir /checkpoint \
  --page-server \
  --address 192.168.1.100 \
  --port 9000

# Destination: Restore with lazy memory
criu restore \
  --images-dir /checkpoint \
  --lazy-pages
```

### A.3 Network Protocol Specification

```protobuf
syntax = "proto3";

package distri_proc;

service PageService {
  rpc FetchPage (PageRequest) returns (PageResponse);
  rpc BatchFetch (BatchRequest) returns (stream PageResponse);
  rpc GetHotPages (HotPagesRequest) returns (PageList);
}

message PageRequest {
  uint64 page_addr = 1;
  uint32 process_id = 2;
  bool prefetch_hint = 3;
}

message PageResponse {
  uint64 page_addr = 1;
  bytes data = 2;
  bool compressed = 3;
  CompressionType compression = 4;
  repeated uint64 suggested_prefetch = 5;
}

message BatchRequest {
  repeated uint64 page_addrs = 1;
  uint32 process_id = 2;
}

enum CompressionType {
  NONE = 0;
  LZ4 = 1;
  ZSTD = 2;
}
```

---

## Appendix B: Evaluation Scripts

### B.1 Redis Benchmark Script

```bash
#!/bin/bash

# Setup
REDIS_MEM="10GB"
LOCAL_MEM="50%"  # 50% local, 50% remote

# Start Redis with DistriProc
distri-proc run \
  --checkpoint=/checkpoints/redis-${REDIS_MEM} \
  --local-memory=${LOCAL_MEM} \
  --page-server=source-node:9000 \
  --rdma-enabled

# Wait for startup
sleep 5

# Run YCSB benchmark
ycsb run redis -s \
  -P workloads/workloada \
  -p recordcount=10000000 \
  -p operationcount=1000000 \
  -threads 16 \
  | tee results/redis-${LOCAL_MEM}.txt

# Collect metrics
distri-proc stats --json > results/stats-${LOCAL_MEM}.json
```

### B.2 Analysis Script

```python
import json
import matplotlib.pyplot as plt

def analyze_results(baseline_file, distri_proc_file):
    with open(baseline_file) as f:
        baseline = json.load(f)
    
    with open(distri_proc_file) as f:
        distri = json.load(f)
    
    # Calculate overhead
    throughput_ratio = distri['throughput'] / baseline['throughput']
    latency_ratio = distri['p99_latency'] / baseline['p99_latency']
    
    print(f"Throughput: {throughput_ratio:.2%} of baseline")
    print(f"P99 Latency: {latency_ratio:.2f}x baseline")
    
    # Plot latency distribution
    plt.figure(figsize=(10, 6))
    plt.hist(baseline['latencies'], alpha=0.5, label='Baseline')
    plt.hist(distri['latencies'], alpha=0.5, label='DistriProc')
    plt.xlabel('Latency (ms)')
    plt.ylabel('Frequency')
    plt.legend()
    plt.savefig('latency_distribution.png')

if __name__ == '__main__':
    analyze_results('results/baseline.json', 'results/distri-proc.json')
```

---

**End of Proposal**

**Contact Information**:
- Project Lead: [Utkarsh Maurya]
- Email: [utkarsh@kernex.sbs]
- GitHub: https://github.com/kernex-sbs/distri-proc

**Last Updated**: February 8, 2026
