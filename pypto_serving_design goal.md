# pypto-serving 设计目标

构建基于 **Lingqu 分布式运行时** (`pypto_runtime_distributed`) 的超高性能 LLM 推理引擎，具备 vLLM / SGLang 的核心子集能力。

---

## 设计目标

1. **C/C++ 高性能核心**
   性能关键路径（Prefill、Decode、KV Cache 访问、Radix Tree 查找）全部用 C/C++ 实现，不允许 Python 出现在自回归路径、prefill-to-decode 路径或 KV Cache 访问路径中。

2. **基于 Lingqu L3–L7 分布式运行时**
   推理引擎构建在 `pypto_runtime_distributed` 的 `LevelRuntime` 之上（L3 Host 层为单机推理核心，L4–L7 为分布式扩展）。使用 orchestrator/worker/scheduler 线程模型、DAG tensor 依赖调度、ring buffer 内存管理——与 simpler L0–L2 运行时设计保持一致。

3. **L2 芯片级 kernel 通过 ChipBackend adapter 调用**
   假设 L2 层已有 `model_prefill` 和 `model_decode` 占位函数（当前为空实现）。L3 Host worker 通过 `ChipBackend` adapter（动态链接 simpler 的 `libhost_runtime.so`）调用 L2 kernel，执行 h2d_copy / d2h_copy 数据搬移。**不修改 simpler**。

4. **OpenAI 风格前端 + 测试直通路径**
   提供兼容 vLLM 的 OpenAI 风格 API 接口。首版通过 **Test Path**（进程内 IPC 直通）注入请求和取回响应，不经过网络栈，便于测试和 golden 校验。

5. **Radix Tree KV Cache 管理（SGLang 风格）**
   - **Radix Tree 元数据**：通过 `lingqu_db`（Redis-style K/V 服务）存储，L3–L7 所有层级可直接访问。Phase 0 使用本地 `meta_data.dat` 文件作为 stub。
   - **KV Cache 三层存储**：
     - **L1 (Hot)**：GPU VRAM，由 L2 simpler HeapRing 管理
     - **L2 (Warm)**：Host 内存，通过 `lingqu_shmem` 跨节点共享
     - **L3 (Cold)**：SSD / 分布式文件，通过 `lingqu_block`（异步 DMA）或 `lingqu_dfs`（POSIX 文件 API）持久化
   - **驱逐策略**：LRU，L1→L2→L3 逐级驱逐，按需从 L3→L2→L1 预取。

6. **长期上下文无关**
   引擎不维护用户 / 会话状态，每次请求由客户端携带完整上下文（含历史）。唯一的持久化状态是 Radix Tree（及其关联的 KV Cache 持久池）。

7. **Autoregressive Loop**
   - **Phase 0–3**：自回归循环在 L3 Host 层通过 LevelRuntime DAG 调度执行，每步 decode 经 h2d/d2h 与 L2 交互。
   - **Phase 5（未来优化）**：通过 `autoregressive_loop_wrapper` 将整条 AR 循环下推到 L2 设备侧闭环执行，Host 一次提交、一次取回，无逐 token 往返。

8. **分布式推理扩展（L4–L7）**
   - **L4 (Pod)**：Tensor parallelism，多主机分片 prefill/decode
   - **L5 (Supernode)**：Prefill/Decode 服务分离，独立批处理
   - **L6 (Cluster)**：QoS 分级，多档 batch size 实例对应不同 TTFT/TPOT 目标
   - **L7 (Global)**：全局请求路由，按 QoS 等级分发到对应服务实例
   - 同档内按序列长度分组以提升 batch 效率

9. **测试直通路径注入数据**
   通过 Test Path 的 C 接口（`inject_request` / `get_response`）注入输入、取回输出，供 golden 校验和集成测试使用。

10. **golden.py 副本与自动校验**
    从 simpler 的 `examples/` 复制 golden.py 到 `pypto-serving/examples/`，修改为通过 Test Path 注入输入、取回输出，并与 `compute_golden` 参考值对比（RTOL/ATOL）。

11. **Perfetto UI trace 支持**
    复用 `pypto_runtime_distributed` 的 `TraceWriter`，记录每个 prefill / decode / sample worker 的执行时间、每个层级实例的 orchestrator / scheduler / worker 线程活动，支持 `--trace` 命令行生成 trace 文件。

---

## 项目位置与依赖

- **推理引擎**：`pypto_workspace/pypto-serving/`
- **分布式运行时**：`pypto_workspace/pypto_runtime_distributed/`（链接其 `linqu_runtime_lib` 和 `linqu_core`）
- **芯片运行时**：`pypto_workspace/simpler/`（通过 ChipBackend adapter 动态链接，不修改）

---

## 参考文档

- `pypto_serving_implementation_plan.md` — 详细实现计划与阶段分解
- `pypto_serving_reference_sglang_vllm.md` — vLLM / SGLang 接口与架构参考
- `linqu_runtime_design.md` — Lingqu L3–L7 分布式运行时设计
- `machine_hierarchy_and_function_hierarchy.md` — 层级模型与 pl.function / pl.at 语法
- `linqu_data_system.md` — Lingqu 四层数据服务（shmem, block, dfs, db）
