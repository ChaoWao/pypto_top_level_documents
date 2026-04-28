# UBL128 Serving: Prefill/Decode 解耦推理系统设计（带 Prefix 支持）

> 本文档描述一个 prefill/decode 解耦的推理服务系统的完整设计，并支持 prefix caching。

## 1. 硬件系统

### 1.1 UBL128 HBD（UB High Bandwidth Domain）

UBL128 是一个 **UB High Bandwidth Domain (HBD)**，是本系统中最小的紧耦合计算单元，规格如下：

- **构成**：8 台 PC16 服务器
- **CPU 总数**：8 × 2 = 16 颗 CPU
- **NPU 总数**：8 × 16 = **128 颗 Ascend 950 芯片**
- **域内互联**：采用 **单层 Scale Up 交换机**，为每颗 NPU 提供 **3.2 Tbps** 的交换带宽
- 域内 128 颗 NPU 之间构成 full any-to-any 的高带宽互联，组成 **Scale Up (SU) 网络**

### 1.2 PC16 服务器

每台 PC16 服务器内部由 2 颗 CPU 和 16 颗 Ascend 950 NPU 组成，对外提供 3 类网络接口。

**服务器内部 NPU 拓扑：**

- 16 颗 NPU 分成 **两组，每组 8 颗（2 × 8 NPU）**，每组与 1 颗 CPU 形成一个 NUMA 单元
- 同一组内的 8 颗 NPU 在 **服务器内部** 采用 **400 Gbps 全互联（full mesh）**
- 每颗 CPU 通过 **8 × 400 Gbps UB**（合计 3.2 Tbps）分别连接到本组内的 **8 颗 NPU**
  - CPU 0 ↔ NPU 0..7（一对一 400 Gbps UB）
  - CPU 1 ↔ NPU 8..15（一对一 400 Gbps UB）
  - 用于 CPU 主存 ↔ NPU HBM 的高带宽数据搬运（如 KV swap、prefix load、参数加载等）
- 跨组的 NPU 通信走 **服务器外部** 的 SU 交换机

**服务器对外接口：**

| 接口 | 带宽 | 连接的网络 | 用途 |
|------|------|-----------|------|
| 每颗 NPU × 1 | 3.2 Tbps | Scale Up (SU) 单层交换机 | 构成 UBL128 域内 128 NPU 间 any-to-any 高带宽网络 |
| 每颗 NPU × 1 | 400 Gbps | Scale Out (SO) 网络（UBG 协议） | 跨 UBL128 的 NPU↔NPU、NPU↔CPU 通信 |
| 每颗 CPU × 8 | 8 × 400 Gbps UB | 服务器内部 → 同 NUMA 的 8 颗 NPU | CPU↔NPU 高带宽数据通路（KV swap / 加载 / DMA） |
| 每台服务器：CPU 网卡 1 | 400 Gbps | RoCE 数据中心网络（DCN） | 通用以太网通信、控制面、外部服务接入 |
| 每台服务器：CPU 网卡 2 | 400 Gbps | Scale Out (SO) 网络 | CPU 接入 SO 网络 |

### 1.3 三类网络总览

系统中共存在 **三类物理网络**，定位互不重叠：

1. **Scale Up (SU) 网络** —— UBL128 域内
   - 单层无阻塞交换机
   - 128 颗 NPU，每颗 3.2 Tbps
   - 用于 HBD 内部张量并行 / KV 高带宽传输 / 紧耦合集合通信

2. **Scale Out (SO) 网络** —— 跨 UBL128
   - 协议：**UBG**
   - 拓扑：**两层无收敛 CLOS**
   - 端口规模：可提供 **16384 个交换端口**
   - 接入对象：每个 UBL128 中的 **每颗 CPU 和每颗 NPU** 都有一条 400 Gbps UBG 接口
   - 能力：实现多个 UBL128 HBD 之间，CPU 与 NPU 的 **any-to-any** 互联互通
   - 用途：跨 HBD 的 KV cache 迁移、prefill→decode 的 KV 推送、prefix 共享、跨域并行

3. **RoCE 数据中心网络（DCN）**
   - 每台 PC16 服务器一条 400 Gbps 网卡接入
   - 仅 CPU 侧可见
   - 用途：服务发现、调度控制面、对外 HTTP/gRPC 入口、与存储/对象服务通信等

### 1.4 拓扑示意（自底向上 4 层视图）

> 说明：以下 4 张 ASCII 图中，**所有方框（`+--+ | | +--+`）内部只放 ASCII 字符**，方框中心列严格对齐；所有中文（标题、连线标注、说明）一律放在方框之外，避免 CJK 双宽度字符破坏对齐。

#### 1.4.1 PC16 服务器（最小算力机柜单元）

> 关键：**每颗 CPU 通过 8 × 400 Gbps UB 链路一对一直连本组的 8 颗 NPU**（CPU0↔NPU0..7，CPU1↔NPU8..15），构成两个 CPU-NPU NUMA 域。

```
=========================  PC16 Server  (2 CPU + 16 NPU)  =========================

         +---------+                              +---------+
         |  NIC#1  |                              |  NIC#2  |
         +----+----+                              +----+----+
              |                                        |
              | 400G RoCE                              | 400G UBG
              v                                        v
            (DCN)                                    (SO)

         +---------+                              +---------+
         |  CPU 0  |                              |  CPU 1  |
         +----+----+                              +----+----+
              |                                        |
              | 8 x 400G UB (1:1)                      | 8 x 400G UB (1:1)
              v                                        v
      +---------------+                        +---------------+
      | NPU Group 0   |                        | NPU Group 1   |
      | NPU 0..NPU 7  |                        | NPU 8..NPU 15 |
      | 400G fullmesh |                        | 400G fullmesh |
      +---------------+                        +---------------+
         (CPU0 NUMA)                              (CPU1 NUMA)

  Each NPU has 4 connections:
    1 x 400 Gbps UB    <-->  its CPU                    (intra-server, 1:1)
    7 x 400 Gbps UB    <-->  the other 7 NPUs in group   (intra-server, full mesh)
    1 x 3.2 Tbps       --->  UBL128 SU single-layer sw   (intra-domain any-to-any)
    1 x 400 Gbps       --->  SO UBG CLOS                 (inter-domain any-to-any)
====================================================================================
```

#### 1.4.2 SSU12 存储框（持久化 KV / Prefix / 模型权重的存储侧机柜）

> SSU = 存储服务单元；SSU12 = 一个机框内含 12 个 SSU。每个 SSU 各自独立 **直出 1 条 400 Gbps UBG** 上行至 SO 网络（不经过任何聚合层）。

```
===================  SSU12 Storage Rack  (1 frame = 12 SSU)  ===================

      +-------+  +-------+  +-------+  +-------+  +-------+  +-------+
      | SSU 0 |  | SSU 1 |  | SSU 2 |  | SSU 3 |  | SSU 4 |  | SSU 5 |
      +---+---+  +---+---+  +---+---+  +---+---+  +---+---+  +---+---+
          |          |          |          |          |          |
      +-------+  +-------+  +-------+  +-------+  +-------+  +-------+
      | SSU 6 |  | SSU 7 |  | SSU 8 |  | SSU 9 |  |SSU 10 |  |SSU 11 |
      +---+---+  +---+---+  +---+---+  +---+---+  +---+---+  +---+---+
          |          |          |          |          |          |
          v          v          v          v          v          v

  ============   12 x 400 Gbps UBG  (each SSU 1 link, no aggregation)  ===========>  SO Network

  * Each SSU directly outputs 1 x 400G UBG -> SO; no in-frame aggregation
  * One SSU12 frame = 12 x 400G external uplinks
  * Purpose: persistent KV / Prefix / model weights
=================================================================================
```

#### 1.4.3 UBL128 HBD（高带宽计算域）

```
=================  UBL128 HBD  (1 HBD = 8 PC16 = 16 CPU + 128 NPU = 128 Ascend 950)  =================

    +------------------------------------------------------------+
    |        UBL128 SU Switch  (intra-domain any-to-any)         |
    |      per NPU 1 x 3.2 Tbps  x  128 NPU  =  409.6 Tbps       |
    +--+-------+-------+-------+-------+-------+-------+-------+-+
       |       |       |       |       |       |       |       |
    +----+  +----+  +----+  +----+  +----+  +----+  +----+  +----+
    |PC16|  |PC16|  |PC16|  |PC16|  |PC16|  |PC16|  |PC16|  |PC16|
    | #0 |  | #1 |  | #2 |  | #3 |  | #4 |  | #5 |  | #6 |  | #7 |
    +----+  +----+  +----+  +----+  +----+  +----+  +----+  +----+

  UBL128 external uplinks:
    128 x NPU :  each 1 x 400 Gbps UBG  --> SO              (inter-domain data plane)
     16 x CPU :  each PC16 1 x 400G RoCE --> DCN             (control plane)
                 each PC16 1 x 400G UBG  --> SO              (CPU data plane)
    128 x NPU intra: 1 x 3.2 Tbps -> SU switch              (stays inside HBD)
======================================================================================================
```

#### 1.4.4 最小生产部署单元 = 2 × UBL128 + 12 × SSU12 + 16 × 鲲鹏 CPU 服务器

> **鲲鹏 CPU 服务器（KP CPU Server）**：通用 ARM CPU 服务器，每台配 **1 张 1825 网卡**，该网卡同时提供：
> - 1 × 400 Gbps 接入 **DCN（RoCE）**
> - 1 × 400 Gbps 接入 **SO 网络（UBG）**
>
> 在系统中承担 **调度 / Router / KV 元数据管理 / 控制面服务 / 长期 prefix index** 等 CPU 侧职责。

```
========================  Production Unit  ========================
       2 x UBL128  +  12 x SSU12  +  16 x KP CPU Server

   +-----------------------------------------------------------------------+
   |                            == DCN (RoCE) ==                           |
   |        CPU-only :  control / HTTP / gRPC / discovery / monitor        |
   +-----------------------------------------------------------------------+
                   |                                   |
                   16 x 400G RoCE                      16 x 400G RoCE
                   v                                   v
   +------------------------------+    +------------------------------+
   |          2 x UBL128          |    |      16 x KP CPU Server      |
   |         (per UBL128:         |    |      each: 1 x NIC1825       |
   |      16 CPU + 128 NPU)       |    |          400G  -> DCN        |
   |                              |    |          400G  -> SO         |
   |    CPU :  400G UBG -> SO     |    |                              |
   |    NPU :  400G UBG -> SO     |    |   role: scheduler / router   |
   |     NPU :  3.2T intra-SU     |    |         KV / prefix index    |
   +------------------------------+    +------------------------------+
                   |                                   |
                   288 x 400G UBG                      16 x 400G UBG
                   v                                   v
   +-----------------------------------------------------------------------+
   |                            == SO Network ==                           |
   |              UBG, 2-layer non-blocking CLOS, 16384 ports              |
   |                      any-to-any : NPU + CPU + SSU                     |
   +-----------------------------------------------------------------------+
                   ^
                   | 144 x 400G UBG (each SSU 1 link)
                   |
   +------------------------------+
   |       12 x SSU12 Rack        |
   |       total 144 x SSU        |
   |     each SSU 400G -> SO      |
   |    KV / prefix / weights     |
   +------------------------------+

  SO ports occupancy:
    2 x UBL128 :  2 x (128 NPU + 16 CPU)   =  288 ports
   16 x KP     :  16 x 1                    =   16 ports
   12 x SSU12  :  12 x 12                   =  144 ports
   ----------------------------------------------------
   Total       :                              448 / 16384 SO ports
====================================================================
```

> **设计含义**：
> - **PC16 → UBL128 → 整体系统** 三级硬件抽象：算力侧聚合在 UBL128 内（域内 SU 极高带宽），对外只通过 **SO 网络** any-to-any 互联；
> - **存储侧（SSU12）** 与 **算力侧（UBL128）** 完全解耦，仅通过 SO 平面相连，可独立扩缩容；
> - **鲲鹏 CPU 服务器** 是 **唯一同时跨 DCN 与 SO 的 CPU 节点**，作为 **控制面 + 数据面元数据** 的桥梁，承担调度/Router/KV 索引等职责；
> - SO 网络 16384 端口规模相对当前 448 端口占用极宽裕，预留了未来扩展 (更多 UBL128 / SSU12 / CPU 服务器) 的空间。

### 1.5 关键带宽与边界总结

| 通信路径 | 经过的网络 | 单链路带宽 | 备注 |
|----------|-----------|-----------|------|
| 同 PC16、同 NUMA 组内 CPU↔NPU | 服务器内 UB（一对一直连） | 400 Gbps（每对） / 3.2 Tbps（CPU 总） | CPU 主存↔NPU HBM 高带宽通路 |
| 同 PC16、同 8-NPU 组内 NPU↔NPU | 服务器内 full mesh | 400 Gbps | 不经过任何交换机 |
| 同 PC16、跨 8-NPU 组 NPU↔NPU | SU 交换机 | 3.2 Tbps（链路上限） | 走域内单层交换 |
| 同 UBL128、跨服务器 NPU↔NPU | SU 交换机 | 3.2 Tbps（链路上限） | 单跳交换 |
| 跨 UBL128 NPU↔NPU / CPU↔NPU | SO（UBG CLOS） | 400 Gbps | 两层 CLOS、无收敛 |
| CPU↔外部服务（控制面/HTTP） | DCN（RoCE） | 400 Gbps/server | 仅 CPU 可见 |

> **设计含义**：UBL128 内部具备 **3.2 Tbps × 128** 的极高带宽，是 prefill 大算力聚合 / decode 张量并行 / 域内 KV 共享的天然边界；SO 网络（400 Gbps × 任意端口、any-to-any）则用于 **跨 HBD 的 KV 迁移、prefill→decode 解耦、prefix cache 跨域共享**；DCN 仅承担控制面与外部入口，不承担数据面流量。
