# pypto-serving 实现计划

基于 Lingqu 分布式运行时 (`pypto_runtime_distributed`) 构建 LLM 推理引擎。

**核心参考文档：**
- `machine_hierarchy_and_function_hierarchy.md` — 层级模型与 `pl.function` / `pl.at` 语法
- `linqu_runtime_design.md` — L3–L7 分布式运行时设计（LevelRuntime, ring buffer, DAG scheduler）
- `linqu_data_system.md` — Lingqu 四层数据服务（shmem, block, dfs, db）
- `pypto_serving_design goal.md` — 设计目标
- `pypto_serving_reference_sglang_vllm.md` — vLLM/SGLang 参考

**前提假设：**
- L2 (Chip) 层已有 `model_prefill` 和 `model_decode` 两个占位函数（目前为空实现），通过 `simpler` 运行时在芯片上执行。
- L3–L7 分布式运行时已在 `pypto_runtime_distributed/` 中实现并验证（LevelRuntime, tree_reduce, TraceWriter 等）。

---

## 一、架构概览：在 Lingqu 层级上构建推理引擎

### 1.1 层级职责映射

推理引擎利用 Lingqu 的 L2–L7 层级，每一层承担不同的推理职责：

| Level | Lingqu 名称 | 推理引擎职责 | 线程模型 |
|-------|-------------|-------------|---------|
| **L7** | Global Coordinator | 全局请求路由 / 负载均衡 / QoS 调度 | 1 orchestrator + scheduler + workers |
| **L6** | Cluster-lv2 | 跨机架协调、全局 Radix Tree 元数据同步 | 同上 |
| **L5** | Cluster-lv1 | 超节点内 Prefill/Decode 服务调度 | 同上 |
| **L4** | Cluster-lv0 | Pod 内批处理调度、KV Cache 复用 | 同上 |
| **L3** | Host | **单机推理引擎核心**：请求解析、Radix 查找、Prefill/Decode 编排、autoregressive loop | 同上 |
| **L2** | Chip | `model_prefill` / `model_decode` 芯片级 kernel 执行 | simpler 运行时管理 |

### 1.2 系统架构图

```
┌─────────────────────────────────────────────────────────────────────────────┐
│  L7  Global Coordinator                                                      │
│  - 全局请求入口、QoS 分级、路由到 L6/L5 服务实例                                │
│  - @pl.function(level=Level.GLOBAL, role=Role.ORCHESTRATOR)                  │
├─────────────────────────────────────────────────────────────────────────────┤
│  L6  Cluster-lv2                                                             │
│  - 跨机架 Radix Tree 元数据聚合（lingqu_db）                                  │
│  - Prefill/Decode 服务多实例调度                                              │
├─────────────────────────────────────────────────────────────────────────────┤
│  L5  Cluster-lv1  (Supernode)                                                │
│  - 超节点内 batch scheduler                                                   │
│  - KV Cache 预取 / 迁移协调（lingqu_shmem）                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│  L4  Cluster-lv0  (Pod)                                                      │
│  - Pod 内多主机推理并行调度                                                    │
│  - lingqu_dfs 共享模型权重 / checkpoint 文件                                  │
├──────────────┬──────────────────────────────────────────────────────────────┤
│  L3  Host    │  单机推理引擎核心                                              │
│              │  ┌───────────────────────────────────────────────────────┐   │
│              │  │ Orchestrator Thread                                    │   │
│              │  │ - 请求解析 → Radix Tree 前缀匹配                       │   │
│              │  │ - submit_worker(model_prefill_host) → Prefill          │   │
│              │  │ - autoregressive_loop: submit_worker(model_decode_host)│   │
│              │  │ - KV Cache 管理 (alloc/evict/persist)                  │   │
│              │  └───────────────────────────────────────────────────────┘   │
│              │  ┌───────────────────────────────────────────────────────┐   │
│              │  │ Worker Threads (CPU)                                    │   │
│              │  │ - model_prefill_host: h2d → L2 model_prefill → d2h    │   │
│              │  │ - model_decode_host:  h2d → L2 model_decode  → d2h    │   │
│              │  │ - kv_evict_worker:  GPU→persistent via lingqu_block    │   │
│              │  │ - kv_load_worker:   persistent→GPU via lingqu_block    │   │
│              │  └───────────────────────────────────────────────────────┘   │
├──────────────┴──────────────────────────────────────────────────────────────┤
│  L2  Chip (simpler runtime, DO NOT MODIFY)                                   │
│  - model_prefill(tokens, kv_blocks) → kv_out, first_token    [占位函数]      │
│  - model_decode(token, kv_blocks) → logits, next_token        [占位函数]      │
│  - autoregressive_loop_wrapper (future: 设备侧自回归闭环)                     │
└─────────────────────────────────────────────────────────────────────────────┘

Data Services:
  lingqu_db    ←── Radix Tree 元数据 (Redis-style K/V, L3–L7 可访问)
  lingqu_dfs   ←── 模型权重, checkpoint, 持久化 KV 文件 (POSIX API, L3–L7)
  lingqu_block ←── KV Cache 持久化到 SSD (async DMA, L0–L7)
  lingqu_shmem ←── 跨节点 KV Cache 共享 (L0–L7, GM space at L0–L2)
```

### 1.3 KV Cache 三层存储

与 vLLM/SGLang 的 L1/L2/L3 缓存层级对应，Lingqu 数据服务自然提供三层：

| 缓存层 | 存储位置 | Lingqu 服务 | 延迟 | 容量 |
|--------|---------|------------|------|------|
| **L1 (Hot)** | GPU VRAM | L2 simpler HeapRing | ns | GB 级 |
| **L2 (Warm)** | Host Memory | `lingqu_shmem` (跨节点可访问) | μs | 百 GB 级 |
| **L3 (Cold)** | SSD / DFS | `lingqu_block` (SSD) 或 `lingqu_dfs` (分布式文件) | ms | TB 级 |

驱逐策略：LRU，L1→L2→L3 逐级驱逐；按需从 L3→L2→L1 预取。

Radix Tree 元数据存储在 `lingqu_db` 中，所有层级 (L3–L7) 可直接访问。

### 1.4 L2 占位函数接口

目前假设以下 L2 函数已存在（空实现）：

```cpp
// L2 model_prefill: 处理完整 prompt，生成 KV Cache，预测第一个 token
// 输入: token_ids (tensor), kv_block_handles (tensor list)
// 输出: kv_out (updated KV blocks), first_token_logits (tensor)
void model_prefill(LinquTensor token_ids, LinquTensor kv_blocks,
                   LinquTensor kv_out, LinquTensor first_token_logits);

// L2 model_decode: 单步解码，用上一 token + KV Cache 预测下一 token
// 输入: prev_token (tensor), kv_blocks (tensor)
// 输出: next_token_logits (tensor), updated_kv (tensor)
void model_decode(LinquTensor prev_token, LinquTensor kv_blocks,
                  LinquTensor next_token_logits, LinquTensor updated_kv);
```

L3 Worker 通过 `ChipBackend` adapter 调用它们（Phase 0 中 adapter 为 stub）。

### 1.5 初始部署拓扑：PC16 x2

首版实现面向一个最小 Lingqu 系统：

| Level | 实例数 | 物理含义 | LevelRuntime 配置 |
|-------|--------|---------|------------------|
| **L4** | 1 | Pod（2 台 PC16 服务器） | `LevelRuntime(level=4, sched=1, workers=2)` |
| **L3[0]** | 1 | Prefill PC16（16 NPU 芯片） | `LevelRuntime(level=3, sched=1, workers=16)` |
| **L3[1]** | 1 | Decode PC16（16 NPU 芯片） | `LevelRuntime(level=3, sched=1, workers=16)` |
| **L2** | 32 (16×2) | NPU 芯片（stub 占位） | simpler 运行时（本阶段为 stub） |

拓扑常量：
```
NUM_L2_PER_L3 = 16   // 每台 PC16 服务器 16 块 NPU
NUM_L3_PER_L4 = 2    // Pod 内 2 台 PC16
L3_PREFILL_IDX = 0   // Prefill 服务器索引
L3_DECODE_IDX  = 1   // Decode 服务器索引
```

**Prefill/Decode 分离**：L4 Pod 级别实现 Prefill/Decode disaggregation：
- L3[0] 专责 Prefill：接收完整 prompt，16 个 NPU 并行执行 tensor parallelism，输出 KV Cache + first token
- L3[1] 专责 Decode：接收 KV Cache，运行 autoregressive loop，每步 16 个 NPU 并行 decode

**两个独立的 L3 LevelRuntime 实例**：模拟物理服务器分离，各自拥有独立的 task ring、heap ring、16 个 worker 线程。

**PyPTO 编码规则**：每个层级严格遵循「一个编排函数 + 多个 worker 函数」模式：
- Orchestrator 函数：构建任务 DAG，提交 worker 和子编排器，等待 future。不直接计算数据。
- Worker 函数：纯计算函数，在 worker 线程上并行执行。不提交后续任务。

**请求流程**：
```
Client → L4 pod_orchestrate
         ├─→ L3[0].submit_orchestrator(prefill_orchestrate)
         │     └─→ submit_worker × 16 (model_prefill_host)
         │          └─→ ChipBackend.model_prefill (L2 stub)
         │     ← PrefillResult{kv_cache, first_token}
         └─→ L3[1].submit_orchestrator(decode_orchestrate)
               └─→ AR loop:
                     submit_worker × 16 (model_decode_host)
                     submit_worker(sample_token)
               ← DecodeResult{output_tokens}
         ← Response
```

**源码组织**（按层级 Lx_ 前缀）：
- PyPTO DSL 规范：`L2_chip_workers.py`, `L3_prefill_server.py`, `L3_decode_server.py`, `L4_pod_orchestrator.py`, `serving_main.py`
- C++ Worker 实现：`L3_workers.h/cpp`
- C++ 编排实现：`inference_engine.h/cpp` (L3), `pod_orchestrator.h/cpp` (L4), `serving_system.h/cpp` (顶层)

---

## 二、项目结构（实际实现）

```
pypto_workspace/
├── pypto_runtime_distributed/          # Lingqu 分布式运行时（已实现）
│   ├── src/
│   │   ├── core/tensor.h               # LinquTensor
│   │   ├── runtime/level_runtime.h     # LevelRuntime (L3–L7)
│   │   ├── runtime/tree_reduce.h       # tree_reduce utility
│   │   └── profiling/trace_writer.h    # Perfetto trace
│   └── tests/unit/
│       ├── test_dfs_sum_hierarchy.cpp
│       ├── test_dfs_sum_hierarchy_pl_function.py
│       └── test_dfs_sum_hierarchy_pl_at.py
│
├── pypto-serving/                      # LLM 推理引擎（本项目）
│   ├── CMakeLists.txt
│   ├── pypto_serving_implementation_plan.md
│   ├── pypto_serving_design goal.md
│   ├── src/
│   │   ├── common/
│   │   │   └── request.h               # Request/PrefillResult/DecodeResult/Response 数据结构
│   │   ├── engine/                     # 推理引擎核心（Phase 0/3）
│   │   │   ├── inference_engine.h/cpp  # L3 orchestrator: prefill + decode + AR loop
│   │   │   ├── pod_orchestrator.h/cpp  # L4 orchestrator: prefill→decode 流水线
│   │   │   ├── serving_system.h/cpp    # 顶层 ServingSystem: 生命周期管理 + trace
│   │   │   ├── request_server.h/cpp    # Phase 3: Radix 查找→Prefill→AR→Radix 更新
│   │   │   ├── stop_condition.h/cpp    # Phase 3: EOS/stop_sequence/max_tokens 停止检查
│   │   │   └── L3_workers.h/cpp        # L3 worker 函数: prefill_host/decode_host/sample/allreduce
│   │   ├── kv/                         # Radix Tree + KV Cache 管理（Phase 2/8）
│   │   │   ├── radix_tree.h/cpp        # Radix Tree: 前缀插入/查找/分支/删除/序列化
│   │   │   ├── kv_cache_manager.h/cpp  # 三层 KV 管理 (L1 GPU/L2 Host/L3 SSD) + LRU
│   │   │   ├── kv_persistence.h/cpp    # lingqu_block/lingqu_db 本地文件 stub
│   │   │   └── persistence_manager.h/cpp # Phase 8: 自动刷盘 + save/load 协调
│   │   ├── frontend/                   # 请求入口（Phase 1）
│   │   │   ├── test_path.h             # TestPath C API 定义 (extern "C")
│   │   │   ├── test_path.cpp           # TestPath 实现: wire format + ServingSystem::infer()
│   │   │   └── test_path.py            # Python ctypes 绑定: TestPathClient
│   │   ├── l2_stubs/                   # L2 芯片后端（Phase 0/4）
│   │   │   ├── chip_backend.h          # ChipBackend 抽象接口
│   │   │   ├── chip_backend_stub.h/cpp # 确定性 stub（Phase 0 测试用）
│   │   │   └── chip_backend_dlopen.h/cpp # Phase 4: dlopen simpler + stub fallback
│   │   ├── distributed/               # L5–L7 分布式调度（Phase 7）
│   │   │   └── service_pool.h/cpp      # ServicePool(L5) + ClusterCoordinator(L6) + GlobalRouter(L7)
│   │   ├── L2_chip_workers.py          # PyPTO DSL: L2 worker 规范
│   │   ├── L3_prefill_server.py        # PyPTO DSL: L3 Prefill orchestrator + workers
│   │   ├── L3_decode_server.py         # PyPTO DSL: L3 Decode orchestrator + workers
│   │   ├── L4_pod_orchestrator.py      # PyPTO DSL: L4 Pod orchestrator
│   │   └── serving_main.py             # PyPTO DSL: 顶层入口
│   ├── tests/
│   │   ├── test_phase0_smoke.cpp       # Phase 0: L4→L3 prefill→decode 全链路
│   │   ├── test_phase1_testpath.cpp    # Phase 1: TestPath C API wire format 集成
│   │   ├── test_phase1_e2e.py          # Phase 1: Python→TestPath→Engine→Python 端到端
│   │   ├── test_phase2_radix_kv.cpp    # Phase 2: Radix Tree + KV Cache 14 个用例
│   │   ├── test_phase3_serve.cpp       # Phase 3: serve_request + StopChecker 8 个用例
│   │   ├── test_phase4_adapter.cpp     # Phase 4: ChipBackendDlopen + KV tracking 4 个用例
│   │   ├── test_phase7_distributed.cpp # Phase 7: L5/L6/L7 + 全链路 6 个用例
│   │   └── test_phase8_persistence.cpp # Phase 8: 持久化 + 压力测试 5 个用例
│   └── examples/
│       └── golden_paged_attention/
│           └── golden.py               # Phase 6: Golden 校验框架 5 个用例
│
└── pypto_top_level_design_documents/   # 设计文档（同步副本）
```

**构建系统**：CMakeLists.txt 链接 `pypto_runtime_distributed` 的预编译静态库（`linqu_runtime_lib`, `linqu_core`, `linqu_ring`, `linqu_profiling`, `linqu_discovery`, `linqu_transport`），同时生成 `pypto_testpath.so` 共享库供 Python ctypes 调用。

**依赖关系**：`pypto-serving` 链接 `pypto_runtime_distributed` 的 `linqu_runtime_lib` 和 `linqu_core`，使用 `LevelRuntime`、`LinquTensor`、`tree_reduce`、`TraceWriter` 等 API。不修改 `simpler`。

---

## 三、L3 单机推理引擎核心设计

### 3.1 用 pl.function 语法表达的推理流程

```python
@pl.function(level=Level.HOST, role=Role.WORKER)
def model_prefill_host(token_ids: Tensor, kv_blocks: Tensor,
                       kv_out: Tensor, first_logits: Tensor):
    """L3 worker: 调用 L2 model_prefill（通过 ChipBackend adapter）。"""
    chip_backend.h2d_copy(token_ids, kv_blocks)
    chip_backend.call_l2("model_prefill", token_ids, kv_blocks, kv_out, first_logits)
    chip_backend.d2h_copy(kv_out, first_logits)


@pl.function(level=Level.HOST, role=Role.WORKER)
def model_decode_host(prev_token: Tensor, kv_blocks: Tensor,
                      next_logits: Tensor, updated_kv: Tensor):
    """L3 worker: 调用 L2 model_decode（通过 ChipBackend adapter）。"""
    chip_backend.h2d_copy(prev_token, kv_blocks)
    chip_backend.call_l2("model_decode", prev_token, kv_blocks, next_logits, updated_kv)
    chip_backend.d2h_copy(next_logits, updated_kv)


@pl.function(level=Level.HOST, role=Role.WORKER)
def sample_token(logits: Tensor, temperature: float, top_p: float,
                 token_out: Tensor):
    """L3 worker: 从 logits 采样下一个 token。"""
    # softmax → top-p filtering → temperature scaling → multinomial sample
    ...


@pl.function(level=Level.HOST, role=Role.ORCHESTRATOR)
def serve_request(rt_l3: LevelRuntime, request: Request) -> Response:
    """L3 orchestrator: 处理单个推理请求。

    1. Radix Tree 前缀查找
    2. Prefill 未命中部分
    3. Autoregressive decode loop
    4. 返回生成的 token 序列
    """
    # Step 1: Radix prefix lookup
    matched_kv, unmatched_tokens = radix_tree.prefix_match(request.token_ids)

    # Step 2: Prefill unmatched tokens
    if len(unmatched_tokens) > 0:
        kv_out = rt_l3.make_tensor(kv_size)
        first_logits = rt_l3.make_tensor(vocab_size)
        rt_l3.submit_worker("model_prefill_host",
            fn=lambda: model_prefill_host(unmatched_tokens, matched_kv,
                                          kv_out, first_logits),
            inputs=[unmatched_tokens, matched_kv],
            outputs=[kv_out, first_logits])
        # DAG scheduler 自动等待 prefill 完成
    else:
        first_logits = cached_logits

    # Step 3: Sample first token
    token = rt_l3.make_tensor(1)
    rt_l3.submit_worker("sample_token",
        fn=lambda: sample_token(first_logits, request.temperature,
                                request.top_p, token),
        inputs=[first_logits], outputs=[token])

    # Step 4: Autoregressive decode loop
    output_tokens = [token]
    for step in range(request.max_tokens):
        next_logits = rt_l3.make_tensor(vocab_size)
        updated_kv = rt_l3.make_tensor(kv_step_size)

        rt_l3.submit_worker("model_decode_host",
            fn=lambda: model_decode_host(token, kv_out,
                                         next_logits, updated_kv),
            inputs=[token, kv_out], outputs=[next_logits, updated_kv])

        next_token = rt_l3.make_tensor(1)
        rt_l3.submit_worker("sample_token",
            fn=lambda: sample_token(next_logits, request.temperature,
                                    request.top_p, next_token),
            inputs=[next_logits], outputs=[next_token])

        # Check stop conditions (EOS, stop sequences, max_tokens)
        if is_stop(next_token, request.stop_sequences):
            break
        token = next_token
        kv_out = updated_kv
        output_tokens.append(next_token)

    # Step 5: Update Radix Tree with new KV
    radix_tree.insert(request.token_ids + output_tokens, kv_out)

    return Response(output_tokens)
```

### 3.2 autoregressive_loop_wrapper（设备侧闭环，未来优化）

Phase 0–3 中，自回归循环在 L3 Host 层执行，每步 decode 通过 `h2d_copy` / `d2h_copy` 与 L2 交互。这是**功能正确但非最优**的路径。

未来优化（Phase 4+）：将整条 AR 循环下推到 L2 设备侧执行（`autoregressive_loop_wrapper`），Host 只参与一次提交和一次取回：

```
Host (L3):  submit_worker("autoregressive_loop_wrapper", ...)
                │
                ▼
Device (L2):  Prefill → Decode step 1 → Decode step 2 → ... → EOS
                │
                ▼ (一次性返回)
Host (L3):  collect complete token sequence
```

---

## 四、分布式推理扩展（L4–L7）

### 4.1 Tensor Parallelism（L4 Pod 级别）

对于大模型（参数量超过单芯片容量），使用 L4 Pod 级别的 tensor parallelism：

```python
@pl.function(level=Level.POD, role=Role.ORCHESTRATOR)
def pod_prefill(rt_l3: LevelRuntime, rt_l4: LevelRuntime,
                request: Request) -> Tensor:
    """L4 orchestrator: 将 prefill 分片到 Pod 内多个 Host。"""
    shard_futures = []
    for host_idx in range(num_hosts_in_pod):
        shard_futures.append(
            rt_l3.submit_orchestrator("serve_shard",
                fn=lambda: serve_prefill_shard(rt_l3, request, host_idx)))

    shard_results = [f.get() for f in shard_futures]
    return pl.tree_reduce(rt_l4, shard_results, allreduce_fn, "allreduce")
```

### 4.2 Prefill/Decode 分离（L5–L6 级别）

```
L6 Cluster Coordinator
  ├── Prefill Service Pool (L5 Supernode)
  │   ├── Prefill Instance 0 (L4 Pod) — batch_size=8, 低延迟档
  │   ├── Prefill Instance 1 (L4 Pod) — batch_size=32, 高吞吐档
  │   └── ...
  └── Decode Service Pool (L5 Supernode)
      ├── Decode Instance 0 (L4 Pod) — batch_size=16, 低延迟档
      ├── Decode Instance 1 (L4 Pod) — batch_size=128, 高吞吐档
      └── ...
```

QoS 分级通过不同 batch size 的实例实现：
- **TTFT（Time To First Token）** 由 Prefill 档位和 batch size 决定
- **TPOT（Time Per Output Token）** 由 Decode 档位和 batch size 决定

### 4.3 Lingqu 数据服务在推理中的作用

| Lingqu 服务 | 推理引擎用途 | 访问层级 |
|-------------|------------|---------|
| `lingqu_shmem` | KV Cache L2 (warm) 跨节点共享；模型权重广播 | L0–L7 |
| `lingqu_block` | KV Cache L3 (cold) 持久化到 SSD；异步读写 | L0–L7 |
| `lingqu_dfs` | 模型文件、checkpoint、tokenizer 配置 | L3–L7 |
| `lingqu_db` | Radix Tree 元数据（Redis-style K/V）；请求队列 | L3–L7 |

---

## 五、实现阶段与任务分解

### Phase 0：项目骨架与运行时集成 ✅

| 序号 | 任务 | 产出 | 状态 |
|------|------|------|------|
| 0.1 | 在 `pypto-serving/` 创建目录结构和 CMakeLists.txt，链接 `pypto_runtime_distributed` 的 `linqu_runtime_lib` 和 `linqu_core` | `CMakeLists.txt` | ✅ 已完成 |
| 0.2 | 创建 L2 占位函数 `model_prefill` / `model_decode`（确定性 stub） | `chip_backend.h`, `chip_backend_stub.h/cpp` | ✅ 已完成 |
| 0.3 | 创建 L3 `InferenceEngine` + L4 `PodOrchestrator` + `ServingSystem` | `inference_engine.h/cpp`, `pod_orchestrator.h/cpp`, `serving_system.h/cpp` | ✅ 已完成 |
| 0.4 | 验证：L4→L3 prefill→decode 全链路冒烟测试 | `test_phase0_smoke.cpp` | ✅ 已通过 |

**实现说明**：
- CMakeLists.txt 使用 `IMPORTED` 静态库方式链接 `pypto_runtime_distributed` 预编译产出，避免 `add_subdirectory` 的 CMAKE_SOURCE_DIR 冲突
- L3 workers（`L3_workers.h/cpp`）包含 4 个 worker 函数：`model_prefill_host`, `model_decode_host`, `sample_token`, `logits_allreduce_pair`
- `ChipBackendStub` 生成基于 chip_id 的确定性输出，确保测试可复现
- PyPTO DSL 规范文件：`L2_chip_workers.py`, `L3_prefill_server.py`, `L3_decode_server.py`, `L4_pod_orchestrator.py`, `serving_main.py`

**验收**：`pypto-serving` 链接 `pypto_runtime_distributed`，L4→L3[0] prefill→L3[1] decode 全链路通过。

---

### Phase 1：测试直通路径（Test Path） ✅

| 序号 | 任务 | 产出 | 状态 |
|------|------|------|------|
| 1.1 | 定义 `TestPath` C 接口：`testpath_init/start/inject_request/get_response/stop/shutdown` | `test_path.h` | ✅ 已完成 |
| 1.2 | TestPath 直接调用 `ServingSystem::infer()`，同步返回响应 | `test_path.cpp` | ✅ 已完成 |
| 1.3 | Python ctypes 绑定：`TestPathClient` 类，支持 `client.infer(token_ids=[...])` | `test_path.py` | ✅ 已完成 |
| 1.4 | C++ 集成测试：wire format 序列化/反序列化 + pipeline 复用（2 次请求） | `test_phase1_testpath.cpp` | ✅ 已通过 |
| 1.5 | Python E2E 测试：4 个用例（基本推理/短 prompt/单 token/确定性验证） | `test_phase1_e2e.py` | ✅ 已通过 |

**实现说明**：
- Wire format：`[max_tokens:u32][temperature:f32][top_p:f32][stop_token:i32][vocab_size:i32][kv_size:i32][kv_step:i32][n_tokens:u32][token_0:u64]...[token_N:u64]`
- 响应 format：`[n_tokens:u32][token_0:u64]...[token_N:u64]`
- `pypto_testpath.so` 共享库供 Python ctypes 加载
- Python `TestPathClient` 支持上下文管理器（`with` 语句）

**验收**：Python → TestPath → L4→L3 LevelRuntime → L2 stub → TestPath → Python 全链路通。

---

### Phase 2：Radix Tree 与 KV Cache 管理 ✅

| 序号 | 任务 | 产出 | 状态 |
|------|------|------|------|
| 2.1 | Radix Tree：节点/边/前缀匹配/分支分裂/删除/序列化 | `radix_tree.h/cpp` | ✅ 已完成 |
| 2.2 | KV Cache Manager：三层管理 (L1 GPU / L2 Host / L3 SSD) | `kv_cache_manager.h/cpp` | ✅ 已完成 |
| 2.3 | KV 持久化 stub：`LocalFilePersistence`（lingqu_block 本地文件模拟） | `kv_persistence.h/cpp` | ✅ 已完成 |
| 2.4 | Radix 元数据持久化 stub：`LocalRadixPersistence`（lingqu_db 本地文件模拟） | 与 2.3 合并 | ✅ 已完成 |
| 2.5 | LRU 驱逐策略：L1→L2→L3 逐级降级，ref_count 保护 | 与 2.2 合并 | ✅ 已完成 |
| 2.6 | 14 个单元测试：前缀/分支/扩展/删除/序列化/eviction/promote/demote/persistence | `test_phase2_radix_kv.cpp` | ✅ 全部通过 |

**实现说明**：
- `RadixTree`：线程安全（内部 mutex），支持 `insert/find_prefix/find_exact/remove/ref/unref/touch/eviction_candidates/serialize/deserialize`
- `KVCacheManager`：每层独立 LRU 链表（front=oldest），`alloc/free/data/promote/demote/evict`，三层容量独立配置
- `LocalFilePersistence`：每个 KV block 写为 `kv_block_<handle>.bin`，支持 write/read/delete
- `LocalRadixPersistence`：整棵 Radix Tree 序列化为单个 `.bin` 文件

**验收**：14 个测试全部通过：前缀插入/查找/分支、KV 块 alloc/free/evict/promote/demote、persistence 往返。

---

### Phase 3：Prefill/Decode 流程与 Autoregressive Loop ✅

| 序号 | 任务 | 产出 | 状态 |
|------|------|------|------|
| 3.1 | 请求解析：`RequestServer::make_stop_config()` | `request_server.h/cpp` | ✅ 已完成 |
| 3.2-3.4 | L3 workers 已在 Phase 0 实现（`L3_workers.h/cpp`） | — | ✅ 复用 Phase 0 |
| 3.5 | `RequestServer::serve_request()`：Radix 查找→KV alloc→Prefill→AR loop→Radix 更新 | `request_server.h/cpp` | ✅ 已完成 |
| 3.6 | `StopChecker`：max_tokens / EOS / 多 token stop_sequence + reason 追踪 | `stop_condition.h/cpp` | ✅ 已完成 |
| 3.7 | Perfetto trace 集成：`ServingSystem::serve()` 传递 trace_pid | `serving_system.cpp` | ✅ 已完成 |
| — | 8 个集成测试：stop 条件 4 + serve pipeline 4（含 prefix sharing 验证） | `test_phase3_serve.cpp` | ✅ 全部通过 |

**实现说明**：
- `RequestServer` 组合 `RadixTree` + `KVCacheManager`，实现 Radix 前缀查找→KV 块分配→Prefill→AR loop→Radix 更新的完整流程
- `StopChecker` 支持三种停止条件的组合，提供 `stop_reason()` 用于统计
- 前缀共享验证：第二个请求（共享 4-token 前缀）成功获得 cache hit
- `ServingSystem::serve()` 作为集成入口，连接 RequestServer 和现有 L4→L3 pipeline

**验收**：8 个测试通过，前缀命中验证成功，Perfetto trace 生成正确。

---

### Phase 4：L2 真实 Kernel 集成 ✅（框架 + stub 模式）

| 序号 | 任务 | 产出 | 状态 |
|------|------|------|------|
| 4.1 | `ChipBackendDlopen`：dlopen 搜索 `libhost_runtime.so`，找不到则 stub fallback | `chip_backend_dlopen.h/cpp` | ✅ 已完成 |
| 4.2-4.3 | `model_prefill_host` / `model_decode_host` 接口已就绪，通过 ChipBackend 抽象调用 | — | ✅ 接口已定义 |
| 4.4 | KV Cache + serve pipeline 端到端 tracking（prefix sharing + block 分配） | `test_phase4_adapter.cpp` | ✅ 已验证 |
| 4.5 | 端到端验证（stub 模式下 dlopen fallback 确定性一致） | 4 个测试 | ✅ 全部通过 |

**实现说明**：
- `ChipBackendDlopen` 自动搜索 `libhost_runtime.so`（当前目录/相对路径/默认路径），若未找到则使用内置 stub 函数
- 期望的 C linkage 函数签名：`simpler_model_prefill(chip_id, tokens, n_tokens, kv_in, kv_size, kv_out, logits, vocab)` 和 `simpler_model_decode(...)`
- Stub fallback 产生确定性输出，经两个独立实例验证一致性
- **注意**：真实 kernel 集成需要 simpler 硬件环境，当前仅框架已就绪

**验收**：4 个测试通过（adapter fallback + determinism + pipeline + KV tracking）。

---

### Phase 5：autoregressive_loop_wrapper（设备侧闭环）⏭ 需要硬件

| 序号 | 任务 | 产出 | 状态 |
|------|------|------|------|
| 5.1 | 在 L2 simpler 中实现 `autoregressive_loop_wrapper` 原语 | L2 扩展 | ⏭ 待硬件环境 |
| 5.2 | L3 orchestrator 改为单次提交 wrapper（含 max_tokens, stop, temperature） | orchestrator 更新 | ⏭ 待 5.1 |
| 5.3 | 性能对比：逐步 h2d/d2h vs. wrapper 闭环 | 基准测试 | ⏭ 待 5.2 |

**说明**：此 Phase 完全依赖 simpler 硬件运行时，需要真实 NPU 设备。当前跳过。

**验收**：待硬件环境就绪后实施。

---

### Phase 6：golden.py 副本与自动化校验 ✅

| 序号 | 任务 | 产出 | 状态 |
|------|------|------|------|
| 6.1 | 创建 golden 框架：`GoldenCase` / `GoldenRunner` | `golden.py` | ✅ 已完成 |
| 6.2 | TestPath 适配：通过 `TestPathClient` 注入/获取 | — | ✅ 已完成 |
| 6.3 | 5 个 golden cases：basic_8tok / short_2tok / single_token / determinism / longer_decode | — | ✅ 5/5 通过 |
| 6.4 | CLI：`--case NAME` / `--all` / `--list` / `--trace` / `--report` | — | ✅ 已完成 |

**实现说明**：
- `GoldenCase` 定义测试参数和预期输出（token 数量和精确值）
- `GoldenRunner` 封装 engine 生命周期、批量执行、JSON report 输出
- 当前为 stub 模式下的确定性验证，Phase 4+ 真实 kernel 后可扩展为 RTOL/ATOL 浮点比对
- 运行命令：`python examples/golden_paged_attention/golden.py --all`

**验收**：5/5 golden cases 通过，JSON report 输出正常。

---

### Phase 7：分布式推理（L4–L7） ✅（框架搭建）

| 序号 | 任务 | 产出 | 状态 |
|------|------|------|------|
| 7.1 | L4 Pod：已在 Phase 0 实现（`PodOrchestrator` 16-chip tensor parallelism） | — | ✅ 复用 Phase 0 |
| 7.2 | L5 `ServicePool`：多实例池 + least-loaded 实例选择 | `service_pool.h/cpp` | ✅ 已完成 |
| 7.3 | L6 `ClusterCoordinator`：QoS 分级路由（LOW_LATENCY / HIGH_THROUGHPUT） | `service_pool.h/cpp` | ✅ 已完成 |
| 7.4 | L7 `GlobalRouter`：全局请求路由，跨集群分发 | `service_pool.h/cpp` | ✅ 已完成 |
| 7.5 | 序列长度分组 batch | — | ⏭ 待后续 |
| 7.6 | `lingqu_db` 分布式 Radix 元数据 | — | ⏭ 待真实集群 |
| 7.7 | `lingqu_shmem` 跨节点 KV Cache 共享 | — | ⏭ 待真实集群 |
| — | 6 个测试：pool basic/multi + cluster basic/qos + router + full L7→L2 | `test_phase7_distributed.cpp` | ✅ 全部通过 |

**实现说明**：
- `ServicePool` (L5)：管理多个 `ServingSystem` 实例，每个实例有独立的 L4+L3+L2 栈；按 `active_requests` least-loaded 选择
- `ClusterCoordinator` (L6)：管理 prefill/decode 两类 ServicePool；QoS 路由策略：LOW_LATENCY→首个 pool（小 batch），HIGH_THROUGHPUT→末尾 pool（大 batch）
- `GlobalRouter` (L7)：管理多个 ClusterCoordinator；当前 round-robin，预留 content-based routing 接口
- 全链路验证：L7 GlobalRouter → L6 Cluster → L5 Pool → L4 Pod → L3 Engine → L2 Stub

**验收**：6 个测试通过，L7→L6→L5→L4→L3→L2 全链路端到端验证成功。

---

### Phase 8：持久化与生产加固 ✅

| 序号 | 任务 | 产出 | 状态 |
|------|------|------|------|
| 8.1 | `PersistenceManager`：Radix 元数据 save/load + auto-flush 后台线程 | `persistence_manager.h/cpp` | ✅ 已完成 |
| 8.2 | KV block 持久化：L1↔L3 demote/promote 经本地文件 stub 验证 | 与 Phase 2 stub 配合 | ✅ 已验证 |
| 8.3 | `lingqu_dfs` 模型权重分布式加载 | — | ⏭ 待真实部署 |
| 8.4 | Perfetto trace 全链路：多请求场景生成完整 trace 文件（22KB+） | `test_phase8_persistence.cpp` | ✅ 已验证 |
| 8.5 | 压力测试：10 个顺序请求，2000 req/s | `test_phase8_persistence.cpp` | ✅ 已通过 |

**实现说明**：
- `PersistenceManager` 协调 `LocalRadixPersistence`（Radix metadata）和 `LocalFilePersistence`（KV blocks）
- Auto-flush：后台线程以可配置间隔（默认 5s）定期保存 Radix 元数据；`stop()` 时执行最终 flush
- Radix save/load 往返验证：插入数据→save→新建 tree→load→查询验证
- KV demote/promote 往返：L1 写入数据→demote L3（写文件）→promote L1（读文件）→数据一致

**验收**：5 个测试通过（persistence roundtrip + auto-flush + stress + trace）。

---

## 六、关键约束检查表

- [x] **性能关键路径全为 C/C++**：Prefill、Decode、Radix 查找、KV 读写均无 Python
- [x] **自回归循环**：Phase 0–3 在 L3 Host 执行（每步 h2d/d2h）；Phase 5 下推到 L2 闭环（待硬件）
- [x] **使用 Lingqu 分布式运行时**：所有 L3–L7 调度通过 `LevelRuntime` 的 orchestrator/worker/scheduler 模型
- [x] **不修改 simpler**：通过 `ChipBackend` adapter 动态链接 simpler API（`ChipBackendDlopen`）
- [x] **Lingqu 数据服务**：KV 持久化用 `LocalFilePersistence`（lingqu_block stub），元数据用 `LocalRadixPersistence`（lingqu_db stub）
- [x] **Test Path 先于网络**：首版通过 TestPath C API + Python ctypes 注入/返回
- [x] **Perfetto trace 支持**：复用 `pypto_runtime_distributed` 的 TraceWriter，全链路验证通过

---

## 七、依赖与顺序小结

```
Phase 0 (骨架 + 运行时集成)          ✅
    │
    ├──► Phase 1 (Test Path)          ✅ ──┬──► Phase 6 (golden 校验)  ✅
    │                                      │
    ├──► Phase 2 (Radix + KV)         ✅   │
    │         │                            │
    │         ▼                            │
    │    Phase 3 (Prefill/Decode/AR)  ✅   │
    │         │                            │
    │         ▼                            │
    │    Phase 4 (L2 kernel adapter)  ✅ ──┘
    │         │
    │         ▼
    │    Phase 5 (设备侧 AR 闭环)    ⏭ 待硬件
    │
    └──► Phase 7 (分布式 L4–L7)      ✅ ──► Phase 8 (持久化/加固)  ✅
```

**实际执行顺序**：0 → 1 → 2 → 3 → 4 → 6 → 7 → 8（Phase 5 跳过，需要 simpler 硬件环境）。

**测试统计**：
| 测试文件 | 用例数 | 结果 |
|----------|--------|------|
| `test_phase0_smoke.cpp` | 1 | ✅ |
| `test_phase1_testpath.cpp` | 2 requests | ✅ |
| `test_phase1_e2e.py` | 4 cases | ✅ |
| `test_phase2_radix_kv.cpp` | 14 | ✅ |
| `test_phase3_serve.cpp` | 8 | ✅ |
| `test_phase4_adapter.cpp` | 4 | ✅ |
| `golden.py --all` | 5 | ✅ |
| `test_phase7_distributed.cpp` | 6 | ✅ |
| `test_phase8_persistence.cpp` | 5 | ✅ |
| **合计** | **~49** | **全部通过** |

---

## 八、文档与参考

- **Lingqu 分布式运行时设计**：`linqu_runtime_design.md`
- **层级模型与函数语法**：`machine_hierarchy_and_function_hierarchy.md`
- **Lingqu 数据服务**：`linqu_data_system.md`
- **vLLM/SGLang 参考**：`pypto_serving_reference_sglang_vllm.md`
- **设计目标**：`pypto_serving_design goal.md`
- **simpler 运行时**：`../simpler/docs/pto2_rt.md`
- **DFS 层级测试**：`pypto_runtime_distributed/tests/unit/test_dfs_sum_hierarchy.cpp`
- **pl.function 测试**：`pypto_runtime_distributed/tests/unit/test_dfs_sum_hierarchy_pl_function.py`
- **pl.at 测试**：`pypto_runtime_distributed/tests/unit/test_dfs_sum_hierarchy_pl_at.py`
