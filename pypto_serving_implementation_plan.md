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

## 二、项目结构

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
│   ├── src/
│   │   ├── engine/                     # 推理引擎核心
│   │   │   ├── inference_engine.h/cpp  # L3 orchestrator: prefill + AR loop
│   │   │   ├── model_prefill_host.h    # L3 worker: h2d → L2 prefill → d2h
│   │   │   ├── model_decode_host.h     # L3 worker: h2d → L2 decode → d2h
│   │   │   ├── autoregressive_loop.h   # AR loop 逻辑 + stop conditions
│   │   │   └── batch_scheduler.h       # L4/L5 batch 调度
│   │   ├── radix/                      # Radix Tree + KV 管理
│   │   │   ├── radix_tree.h/cpp        # Radix Tree 数据结构
│   │   │   ├── kv_cache_manager.h      # 三层 KV 管理 (L1/L2/L3)
│   │   │   └── kv_persistence.h        # lingqu_block/dfs 持久化
│   │   ├── frontend/                   # 请求入口
│   │   │   ├── request_parser.h/cpp    # OpenAI 格式解析
│   │   │   └── test_path.h/cpp         # 测试直通路径 (无网络)
│   │   └── cluster/                    # L4–L7 分布式调度
│   │       ├── global_router.h         # L7 全局路由
│   │       ├── prefill_service.h       # Prefill 服务实例
│   │       └── decode_service.h        # Decode 服务实例
│   ├── tests/
│   │   ├── test_radix_tree.cpp
│   │   ├── test_single_request.cpp
│   │   └── test_batch_decode.cpp
│   └── examples/
│       └── golden_paged_attention/
│           └── golden.py               # 从 simpler 复制、改用 test path
│
└── pypto_top_level_design_documents/   # 设计文档
```

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

### Phase 0：项目骨架与运行时集成

| 序号 | 任务 | 产出 |
|------|------|------|
| 0.1 | 在 `pypto-serving/` 创建目录结构和 CMakeLists.txt，链接 `pypto_runtime_distributed` 的 `linqu_runtime_lib` 和 `linqu_core` | 可编译的空项目 |
| 0.2 | 创建 L2 占位函数 `model_prefill` / `model_decode`（空实现，接口定义正确） | `l2_model_stubs.h/cpp` |
| 0.3 | 创建 L3 `InferenceEngine` 类，内含一个 `LevelRuntime(level=3, ...)`，能 start/stop | 最小可运行 binary |
| 0.4 | 验证：L3 LevelRuntime 能 submit_worker 并执行一个空任务 | 单元测试通过 |

**验收**：`pypto-serving` 链接 `pypto_runtime_distributed`，L3 LevelRuntime 工作正常。

---

### Phase 1：测试直通路径（Test Path）

| 序号 | 任务 | 产出 |
|------|------|------|
| 1.1 | 定义 `TestPath` C 接口：`inject_request(buf, len)` / `get_response(buf, len)` | `test_path.h` |
| 1.2 | L3 orchestrator 从 TestPath 队列取请求，执行后写回响应 | 请求/响应流 |
| 1.3 | Python 绑定（ctypes）：从 golden 调用 inject / get | `test_path.py` |
| 1.4 | 端到端验证：注入 token ids → L3 orchestrator 调用 L2 stub → 返回结果 | 集成测试 |

**验收**：Python → TestPath → L3 LevelRuntime → L2 stub → TestPath → Python 全链路通。

---

### Phase 2：Radix Tree 与 KV Cache 管理

| 序号 | 任务 | 产出 |
|------|------|------|
| 2.1 | Radix Tree 数据结构（C++）：节点、边、token 到节点映射、前缀匹配 | `radix_tree.h/cpp` |
| 2.2 | KV Cache Manager：三层管理 (GPU L1 / Host L2 / Persistent L3) | `kv_cache_manager.h/cpp` |
| 2.3 | KV 持久化：通过 `lingqu_block` 异步读写 SSD（Phase 0 中用本地文件 stub） | `kv_persistence.h/cpp` |
| 2.4 | Radix 元数据持久化：通过 `lingqu_db` 存储（Phase 0 中用本地 meta_data.dat） | 与 2.1 结合 |
| 2.5 | LRU 驱逐策略：L1→L2→L3 逐级驱逐，按需预取 | 与 2.2 结合 |
| 2.6 | 单元测试：前缀插入/查找/分支、KV 块 alloc/free/evict | 测试用例 |

**验收**：Radix Tree 前缀匹配、KV 块三层管理逻辑正确。

---

### Phase 3：Prefill/Decode 流程与 Autoregressive Loop

| 序号 | 任务 | 产出 |
|------|------|------|
| 3.1 | 请求解析：从 TestPath buffer 解析 messages / max_tokens / stop / temperature | `request_parser.h/cpp` |
| 3.2 | `model_prefill_host` L3 worker：h2d → L2 model_prefill → d2h（Phase 0 stub） | worker 函数 |
| 3.3 | `model_decode_host` L3 worker：h2d → L2 model_decode → d2h（Phase 0 stub） | worker 函数 |
| 3.4 | `sample_token` L3 worker：softmax → top-p → temperature → sample | worker 函数 |
| 3.5 | `serve_request` L3 orchestrator：Radix 查找 → Prefill → AR loop → Radix 更新 | orchestrator 函数 |
| 3.6 | Stop 条件：EOS token、用户 stop 序列、max_tokens | `stop_condition.h/cpp` |
| 3.7 | Perfetto trace 集成：每个 prefill/decode/sample worker 记录时间 | trace 输出 |

**验收**：通过 TestPath 注入 prompt，返回 decode 序列（L2 stub 输出伪数据）。

---

### Phase 4：L2 真实 Kernel 集成

| 序号 | 任务 | 产出 |
|------|------|------|
| 4.1 | 实现 `ChipBackend` adapter：dlopen simpler 的 `libhost_runtime.so`，调用 L2 API | `chip_backend.h/cpp` |
| 4.2 | `model_prefill_host` 对接真实 L2 kernel：h2d_copy → simpler API → d2h_copy | 真实 prefill |
| 4.3 | `model_decode_host` 对接真实 L2 kernel | 真实 decode |
| 4.4 | KV Cache GPU 池与 simpler HeapRing 的对接 | L1 KV 管理 |
| 4.5 | 端到端验证：真实 model weights → prefill → decode → 可读文本输出 | 集成测试 |

**验收**：L3 Host 层完整推理流程，使用真实 L2 kernel 产生正确的模型输出。

---

### Phase 5：autoregressive_loop_wrapper（设备侧闭环）

| 序号 | 任务 | 产出 |
|------|------|------|
| 5.1 | 在 L2 simpler 中实现 `autoregressive_loop_wrapper` 原语 | L2 扩展 |
| 5.2 | L3 orchestrator 改为单次提交 wrapper（含 max_tokens, stop, temperature） | orchestrator 更新 |
| 5.3 | 性能对比：逐步 h2d/d2h vs. wrapper 闭环 | 基准测试 |

**验收**：Host 一次提交、一次取回，自回归循环在设备侧完成。

---

### Phase 6：golden.py 副本与自动化校验

| 序号 | 任务 | 产出 |
|------|------|------|
| 6.1 | 从 simpler 复制 golden.py 到 `pypto-serving/examples/golden_paged_attention/` | 副本 |
| 6.2 | 修改 generate_inputs 为 TestPath 格式 | 适配 |
| 6.3 | 通过 TestPath 注入 → Engine 执行 → TestPath 取回 → compute_golden 对比 | 校验流 |
| 6.4 | 支持 ALL_CASES / DEFAULT_CASE / --case / --all | runner |

**验收**：至少一个 golden case 通过（引擎输出与参考值在 RTOL/ATOL 内一致）。

---

### Phase 7：分布式推理（L4–L7）

| 序号 | 任务 | 产出 |
|------|------|------|
| 7.1 | L4 Pod 级别 tensor parallelism：多主机分片 prefill/decode | `pod_parallel.h/cpp` |
| 7.2 | L5 Prefill/Decode 分离：独立服务池 | `prefill_service.h`, `decode_service.h` |
| 7.3 | L6 QoS 分级：多档 batch size 实例，不同 TTFT/TPOT 目标 | QoS 调度器 |
| 7.4 | L7 全局路由：请求按 QoS 等级路由到对应 L6→L5→L4 实例 | `global_router.h/cpp` |
| 7.5 | 同档内按序列长度分组 batch | 长度分组策略 |
| 7.6 | `lingqu_db` 分布式 Radix 元数据：跨节点前缀共享 | 分布式 Radix |
| 7.7 | `lingqu_shmem` 跨节点 KV Cache 共享 | warm cache 共享 |

**验收**：多节点推理工作正常，Prefill/Decode 分离，QoS 分级可验证。

---

### Phase 8：持久化与生产加固

| 序号 | 任务 | 产出 |
|------|------|------|
| 8.1 | `lingqu_db` Radix 元数据持久化（启动加载 / 定期刷盘） | 持久化策略 |
| 8.2 | `lingqu_block` KV Cache SSD 持久化（分片阵列管理） | 块管理 |
| 8.3 | `lingqu_dfs` 模型权重分布式加载 | 权重管理 |
| 8.4 | Perfetto trace 全链路（L3–L7 每个推理步骤） | 性能分析 |
| 8.5 | 压力测试与容量调优 | 性能报告 |

---

## 六、关键约束检查表

- [ ] **性能关键路径全为 C/C++**：Prefill、Decode、Radix 查找、KV 读写均无 Python
- [ ] **自回归循环**：Phase 0–3 在 L3 Host 执行（每步 h2d/d2h）；Phase 5 下推到 L2 闭环
- [ ] **使用 Lingqu 分布式运行时**：所有 L3–L7 调度通过 `LevelRuntime` 的 orchestrator/worker/scheduler 模型
- [ ] **不修改 simpler**：通过 `ChipBackend` adapter 动态链接 simpler API
- [ ] **Lingqu 数据服务**：KV 持久化用 `lingqu_block`/`lingqu_dfs`，元数据用 `lingqu_db`
- [ ] **Test Path 先于网络**：首版通过 TestPath 注入/返回，便于测试
- [ ] **Perfetto trace 支持**：复用 `pypto_runtime_distributed` 的 TraceWriter

---

## 七、依赖与顺序小结

```
Phase 0 (骨架 + 运行时集成)
    │
    ├──► Phase 1 (Test Path) ──┬──► Phase 6 (golden 校验)
    │                          │
    ├──► Phase 2 (Radix + KV)  │
    │         │                │
    │         ▼                │
    │    Phase 3 (Prefill/Decode/AR loop)
    │         │                │
    │         ▼                │
    │    Phase 4 (L2 真实 kernel) ──┘
    │         │
    │         ▼
    │    Phase 5 (设备侧 AR 闭环)
    │
    └──► Phase 7 (分布式 L4–L7) ──► Phase 8 (持久化/生产加固)
```

**建议**：0 → 1 → 2 可部分并行（1 和 2 无互相依赖）；3 依赖 1+2；4 依赖 3；5 依赖 4；6 依赖 1 且最好在 4 之后；7 在 4/5 稳定后开展。

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
