# Enhanced TPUSH/TPOP ISA Design for Intra-Cluster Function Group Data Communication

## Overview

This document specifies an enhanced ISA design for `TPUSH`, `TPOP`, and `TFREE` instructions to support **intra-cluster data communication across InCore kernels** within a function group.

### Cluster Architecture

Each cluster contains **1 Cube core** and **2 buddy Vector cores** that share a hardware flag-based synchronization mechanism:

```
┌─────────────────────── Cluster ───────────────────────┐
│                                                       │
│  ┌──────────┐    flags (8 per dir)    ┌──────────┐   │
│  │  Vector 0 │◄══════════════════════►│          │   │
│  └──────────┘   SET/WAIT V→C, C→V    │   Cube   │   │
│                                       │          │   │
│  ┌──────────┐    flags (8 per dir)    │          │   │
│  │  Vector 1 │◄══════════════════════►│          │   │
│  └──────────┘   SET/WAIT V→C, C→V    └──────────┘   │
│                                                       │
└───────────────────────────────────────────────────────┘
```

- A Vector core can **SET** a flag that the Cube core **WAITs** on, and vice versa.
- There are **8 flags per direction per peer** (Vector→Cube: 8 flags, Cube→Vector: 8 flags), for a total of 16 flags per Vector-Cube pair.
- With 2 buddy Vector cores, the cluster has **32 cross-core flags** in total (2 peers × 2 directions × 8 flags).

### AIV Communication Modes: 1:1 vs 1:2

The TPUSH/TPOP design supports two AIV communication modes:

| AIV_MODE | Description | Use Case |
|---|---|---|
| `AIV_SINGLE` (1:1) | Communication with **one** buddy Vector core only | Simple workloads, single Vector core active |
| `AIV_DUAL` (1:2) | Communication with **both** buddy Vector cores simultaneously | Full cluster utilization, tile split/combined across AIV0 and AIV1 |

**Assumption for 1:1 mode**: In the current design, 1:1 mode (`TILE_NO_SPLIT`) communicates with **AIV0 by default**. This simplifies the API and provides forward compatibility — when upgrading to 1:2 mode, AIV0's role remains the same (upper/left portion), only AIV1 is added.

### Tile Split/Combine Axis

When using 1:2 mode, the user **must specify** how the Cube tile is split (C2V) or combined (V2C):

```cpp
enum TileSplitAxis : uint8_t {
    TILE_NO_SPLIT   = 0,   // 1:1 mode, no split
    TILE_UP_DOWN    = 1,   // Split/combine along rows: AIV0=upper half, AIV1=lower half
    TILE_LEFT_RIGHT = 2,   // Split/combine along cols: AIV0=left half, AIV1=right half
};
```

**Why this must be specified**: The Vector kernel code cannot compute correctly without knowing which portion of the tile it receives or produces. With `TILE_UP_DOWN`, AIV0 knows it has the upper rows; with `TILE_LEFT_RIGHT`, AIV0 knows it has the left columns.

```
TILE_UP_DOWN:                          TILE_LEFT_RIGHT:

  Cube Tile [16, 128]:                   Cube Tile [16, 128]:
  ┌───────────────────┐                  ┌──────────┬──────────┐
  │ upper [8, 128]    │ ↔ AIV0           │ left     │ right    │
  ├───────────────────┤                  │[16, 64]  │[16, 64]  │
  │ lower [8, 128]    │ ↔ AIV1           │↔ AIV0    │↔ AIV1    │
  └───────────────────┘                  └──────────┴──────────┘
```

### Ring Buffer Data Channel

Data is moved between producer and consumer kernels through a **multi-slot ring buffer** with flow control. Each slot holds one fixed-size **Tile**. On both platforms, the **consumer** has a local SRAM slot buffer where `tpop` places data. The platforms differ in the producer-side staging:

| Platform | Producer Staging | Consumer Slot Buffer | Data Path |
|---|---|---|---|
| **A2A3** | **Global Memory (GM)** — `tpush` DMAs tile to GM | **Consumer's on-chip SRAM** — `tpop` DMAs from GM to local | Two-hop: Producer → GM → Consumer SRAM |
| **A5** | **Consumer's on-chip SRAM** — `tpush` DMAs tile directly to consumer's SRAM | Same buffer (zero-copy `tpop`) | One-hop: Producer → Consumer SRAM |

```
A2A3 Platform:                                        A5 Platform:

Producer          GM            Consumer              Producer                    Consumer
┌──────┐   ┌────────────┐   ┌──────────────┐         ┌──────┐                 ┌──────────────┐
│      │──▶│ slot[0..N-1]│──▶│ UB / L1      │         │      │───────────────▶│ UB / L1      │
│ Cube │   │ (off-chip)  │   │ slot[0..N-1] │         │ Cube │  DMA to local  │ slot[0..N-1] │
│ /Vec │   └────────────┘   │ (on-chip)    │         │ /Vec │                 │ (on-chip)    │
└──────┘     tpush→GM        └──────────────┘         └──────┘                 └──────────────┘
                              tpop: GM→local                                   tpop: zero-copy
```

On both platforms, `TILE.data` references the consumer's local SRAM slot buffer after `tpop`. The A5 one-hop path eliminates the GM round-trip, enabling lower-latency data handoff. On A2A3, `tpop` performs a DMA from GM to the consumer's local slot buffer before aliasing `TILE.data`.

The enhanced design extends `TPUSH`/`TPOP` to serve as the primary data communication mechanism between InCore kernels co-scheduled on these cores within the same cluster, enabling:

- **Producer kernel → Ring Buffer → Consumer kernel** tile-level data flow between Cube and buddy Vector cores
- Cross-core synchronization via the hardware flag mechanism (SET/WAIT, 8 flags per direction)
- Multi-slot ring buffer for pipelined execution (`SLOT_NUM` = 8 for unidirectional, 4 for bidirectional)
- Platform-adaptive buffer placement: GM (A3) or consumer-local SRAM (A5)
- Deferred slot release via `tfree` instructions, ensuring consumer data is not overwritten before use

## Motivation: Intra-Cluster Function Group Communication

When the `ExpandMixedKernel` pass decomposes a mixed InCore function into multiple co-scheduled kernels (e.g., a data-movement kernel on Vector cores and a compute kernel on Cube cores), these kernels need an efficient, synchronized data communication channel within the same cluster.

`TPUSH`/`TPOP` with ring buffer flow control provides exactly this capability — the enhanced design formalizes how the compiler should emit `TPUSH`/`TPOP` pairs to connect the expanded kernel group.

## Enhanced Design: Tag-Based Dual-Channel FIFO Protocol

The enhanced TPUSH/TPOP design uses a **multi-slot ring buffer** with tag-based dual-channel flow control for moving fixed-size Tile data between producer and consumer kernels.

### Producer / Consumer Roles

The terms **producer** and **consumer** are conceptual roles, not bound to a specific core type:

- A **Cube core** can be a producer (e.g., matmul output → Vector for post-processing), or a consumer (e.g., receiving preprocessed data from Vector).
- A **Vector core** can be a producer (e.g., data loading / preprocessing → Cube), or a consumer (e.g., receiving matmul results from Cube).
- In some applications, **both cores are simultaneously producer and consumer** in opposite directions, forming a bidirectional data flow.

### Ring Buffer Structure

Each ring buffer is a **unidirectional** channel from one producer to one consumer. The number of slots is a compile-time constant parameter `SLOT_NUM`, specified during kernel initialization:

| Communication Pattern | SLOT_NUM | Flags Used | Description |
|---|---|---|---|
| **Unidirectional** (one direction only) | **8** | 8 flags for P2C + C2P | All 8 flags per direction dedicated to a single ring buffer |
| **Bidirectional** (both directions simultaneously) | **4** per direction | 4 flags for each of the 2 ring buffers | The 8 available flags are split equally between the two directions |

```
Unidirectional (SLOT_NUM=8):

    Cube (producer)  ──────▶  Vector (consumer)
    Ring Buffer: slot[0..7], using flags 0..7

Bidirectional (SLOT_NUM=4 per direction):

    Cube  ──── Ring Buffer A (slot[0..3], flags 0..3) ────▶  Vector
    Cube  ◀──── Ring Buffer B (slot[0..3], flags 4..7) ────  Vector
```

```
Ring Buffer  —  SLOT_NUM fixed-size Tile slots, indexed by tag

    ┌──────────────────────────────────────────────────────┐
    │  slot[0]    slot[1]    ...    slot[SLOT_NUM-1]       │   A3: Global Memory
    └──────────────────────────────────────────────────────┘   A5: Consumer's UB or L1

Signal Channels (mapped to hardware cross-core flags):
    P2C  —  Producer → Consumer  (data ready signal, indexed by tag)
    C2P  —  Consumer → Producer  (space free signal, indexed by tag)
```

Each ring buffer slot holds exactly one Tile and is identified by a **tag** (0 .. SLOT_NUM-1). The two signal channels `P2C` and `C2P` carry per-tag notifications:
- `SET P2C: tag` — producer signals "data in `slot[tag]` is ready" (issued by `tpush`)
- `SET C2P: tag` — consumer signals "`slot[tag]` is free for reuse" (issued by `tfree`, **not** by `tpop`)
- `WAIT P2C: tag` — consumer blocks until `slot[tag]` is ready (issued by `tpop`)
- `WAIT C2P: tag` — producer blocks until `slot[tag]` is free (issued by `tpush`)

### API Definition

#### Platform Constant

```cpp
enum PlatformID : uint8_t {
    PLATFORM_A2A3 = 0,   // A2/A3 platform: ring buffer in Global Memory
    PLATFORM_A5   = 1,   // A5 platform: ring buffer in consumer's on-chip SRAM
};
```

`PLATFORM_ID` is a **compile-time constant** generated by the compiler and embedded into the kernel binary. It is used by the initialization APIs and `tpush_*/tpop_*` instructions to select the appropriate behavior:

| PLATFORM_ID | Producer Ring Buffer | Consumer Slot Buffer | `tpush_*` Behavior | `tpop_*` Behavior |
|---|---|---|---|---|
| `PLATFORM_A2A3` | GM (`GM_SLOT_BUFFER`, pre-allocated per-cluster) | Consumer's on-chip SRAM (`CONSUMER_BUFFER`) | DMA tile → GM slot | DMA GM slot → consumer's local SRAM slot; `TILE.data` references local slot |
| `PLATFORM_A5` | Consumer's on-chip SRAM (UB or L1) | Same as producer target (single buffer) | DMA tile → consumer's local SRAM slot | Zero-copy: `TILE.data` references local SRAM slot directly |

On **both platforms**, the consumer's `TILE.data` always references a local SRAM slot buffer after `tpop`. The difference is only in the data transport path:
- **A2A3**: Two-hop — producer DMAs tile to GM (`tpush`), then consumer DMAs from GM to its local SRAM slot buffer (`tpop`). The `GM_SLOT_BUFFER` serves merely as a **staging buffer** to facilitate data transfer through GM; it is not the buffer that kernel code operates on.
- **A5**: One-hop — producer DMAs tile directly to consumer's local SRAM slot buffer (`tpush`); `tpop` is zero-copy.

**Key design principle**: By organizing the consumer's local SRAM slot buffer identically on A2/A3 and A5, the **kernel program is the same on both platforms**. The consumer kernel always sees data in `local_slot_buf[pop_tag]` regardless of whether it arrived via GM staging (A2A3) or direct cross-core DMA (A5). Only the transport layer inside `tpush`/`tpop` differs — the kernel-visible `tpop → compute on TILE.data → tfree` sequence is platform-independent.

#### Direction Constants

```cpp
enum Direction : uint8_t {
    DIR_C2V = 0,   // Cube → Vector: Cube is producer, Vector is consumer
    DIR_V2C = 1,   // Vector → Cube: Vector is producer, Cube is consumer
};
```

A kernel uses `DIR_C2V` or `DIR_V2C` to specify the data flow direction. For bidirectional communication, both directions are active simultaneously (`DIR_C2V | DIR_V2C`).

#### Tile Split/Combine Axis Constants

```cpp
enum TileSplitAxis : uint8_t {
    TILE_NO_SPLIT   = 0,   // 1:1 mode, full tile to/from single AIV
    TILE_UP_DOWN    = 1,   // 1:2 mode, split/combine along rows (M axis)
    TILE_LEFT_RIGHT = 2,   // 1:2 mode, split/combine along cols (N axis)
};
```

#### `DIR_MASK`

A bitmask indicating which directions are active for this kernel:

| DIR_MASK | Value | Meaning | SLOT_NUM per direction |
|---|---|---|---|
| `DIR_C2V` | `0b01` | Unidirectional: Cube → Vector only | 8 |
| `DIR_V2C` | `0b10` | Unidirectional: Vector → Cube only | 8 |
| `DIR_C2V \| DIR_V2C` | `0b11` | Bidirectional: both directions | 4 |

#### `GM_SLOT_BUFFER` and `CONSUMER_BUFFER_BASE` / `CONSUMER_BUFFER_SIZE`

The **consumer-local SRAM slot buffer** (`CONSUMER_BUFFER_BASE`/`CONSUMER_BUFFER_SIZE`) is the primary data structure that kernel code operates on — it is present on **both** platforms with identical layout. The `GM_SLOT_BUFFER` on A2A3 is merely a **staging buffer** that facilitates data transfer through GM; kernel code never directly references GM slot data.

| Platform | Producer → ? | Consumer Slot Buffer | GM Staging |
|---|---|---|---|
| **A2A3** | Producer DMAs to `GM_SLOT_BUFFER` (GM) | Consumer's local SRAM, same layout as A5 | `GM_SLOT_BUFFER` — staging only, not kernel-visible |
| **A5** | Producer DMAs directly to consumer's SRAM | Consumer's local SRAM (`CONSUMER_BUFFER_BASE`) | Not needed |

This design ensures that **kernel programs look the same on A2/A3 and A5**: the consumer always reads from `local_slot_buf[tag]` in its own SRAM. The platform difference is hidden inside the `tpush`/`tpop` transport implementation.

**A2A3 GM_SLOT_BUFFER (staging only)**: `GM_SLOT_BUFFER` is a **pre-allocated, per-cluster reserved region** in GM, used exclusively as a staging area for producer-to-consumer data transfer. The pypto L0–L2 runtime (`simpler`) reserves this space at startup — outside of the per-task memory ring — so that the GM footprint is bounded and does not grow linearly with the number of pending tasks.

**Pre-allocation scheme**:

```
PER_CLUSTER_SLOT_BUFFER_SIZE = NUM_BUDDIES(2) * L0C_SIZE
    // Conservative upper bound: each buddy pair needs at most L0C_SIZE
    // for bidirectional ring buffers (2 * SLOT_NUM * SLOT_SIZE ≤ L0C_SIZE)

GM_SLOT_BUFFER_RESERVED_SPACE = NUM_CORE_CLUSTERS * PER_CLUSTER_SLOT_BUFFER_SIZE
    // Total reserved GM for all clusters

GM_SLOT_BUFFER_BASE = runtime_reserve_gm(GM_SLOT_BUFFER_RESERVED_SPACE)
    // Reserved once at runtime init, outside the task memory ring
```

Each kernel receives `GM_SLOT_BUFFER_BASE` and `PHYSICAL_CORE_CLUSTER_IDX` as runtime arguments. The kernel computes its per-cluster buffer address:

```
my_gm_slot_buffer = GM_SLOT_BUFFER_BASE + PHYSICAL_CORE_CLUSTER_IDX * PER_CLUSTER_SLOT_BUFFER_SIZE
```

This ensures:
- **Bounded GM usage**: total GM reserved is `NUM_CORE_CLUSTERS * NUM_BUDDIES * L0C_SIZE`, independent of task queue depth
- **No per-task allocation**: the orchestration does not allocate `GM_SLOT_BUFFER` per task; it reuses the pre-allocated region indexed by physical cluster ID
- **Safe reuse**: since a core cluster can only execute one function group at a time, the pre-allocated buffer for that cluster is exclusively owned during execution

**Consumer slot buffer (both platforms)**: On **both** A2A3 and A5, each consumer InCore function has a `CONSUMER_BUFFER_BASE` and `CONSUMER_BUFFER_SIZE` that reserve a segment of local SRAM for the ring buffer slots. The resolved `CONSUMER_BUFFER_BASE` values are passed as **explicit arguments** (`C2V_CONSUMER_BUF`, `V2C_CONSUMER_BUF`) to the initialization functions on **both** platforms. This is the buffer that kernel code operates on — it is organized identically across platforms, so that **the same kernel program works on both A2/A3 and A5**.

- **A2A3**: The consumer's local slot buffer receives data via `tpop` DMA from GM. The `GM_SLOT_BUFFER` is merely a staging area — kernel code never references GM slot data directly. The producer does **not** need to know the consumer's local buffer address — it writes to GM.
- **A5**: The consumer's local slot buffer is also the DMA target for the producer's `tpush`. The producer **does** need the consumer's buffer address (see "Cross-Core Address Problem on A5" section below).

```
Runtime init (A2A3):
    GM_SLOT_BUFFER_BASE = runtime_reserve_gm(
        NUM_CORE_CLUSTERS * NUM_BUDDIES * L0C_SIZE)

Orchestration function (A2A3):
    // GM_SLOT_BUFFER_BASE and PHYSICAL_CORE_CLUSTER_IDX passed by runtime
    gm_slot_buf = GM_SLOT_BUFFER_BASE + PHYSICAL_CORE_CLUSTER_IDX * PER_CLUSTER_SLOT_BUFFER_SIZE

    // CONSUMER_BUFFER_BASE values resolved by compiler (same mechanism as A5)
    cube_c2v_consumer_buf  = 0                                 // Cube is producer in C2V, not a consumer
    cube_v2c_consumer_buf  = resolve(cube_kernel.CONSUMER_BUFFER_BASE_V2C)   // Cube's L1 slot buffer
    vec_c2v_consumer_buf   = resolve(vector_kernel.CONSUMER_BUFFER_BASE_C2V) // Vector's UB slot buffer
    vec_v2c_consumer_buf   = 0                                 // Vector is producer in V2C, not a consumer

    for ...:
        cube_kernel(  ..., GM_SLOT_BUFFER=gm_slot_buf,    // extern mem_ref, no dep tracking
                           C2V_CONSUMER_BUF=cube_c2v_consumer_buf,
                           V2C_CONSUMER_BUF=cube_v2c_consumer_buf, ...)
        vector_kernel(..., GM_SLOT_BUFFER=gm_slot_buf,    // extern mem_ref, no dep tracking
                           C2V_CONSUMER_BUF=vec_c2v_consumer_buf,
                           V2C_CONSUMER_BUF=vec_v2c_consumer_buf, ...)

Orchestration function (A5):
    // CONSUMER_BUFFER_BASE values resolved by compiler and passed explicitly
    cube_c2v_consumer_buf  = resolve(vector_kernel.CONSUMER_BUFFER_BASE_C2V) // Vector's UB (for producer to DMA into)
    cube_v2c_consumer_buf  = resolve(cube_kernel.CONSUMER_BUFFER_BASE_V2C)   // Cube's own L1
    vec_c2v_consumer_buf   = resolve(vector_kernel.CONSUMER_BUFFER_BASE_C2V) // Vector's own UB
    vec_v2c_consumer_buf   = resolve(cube_kernel.CONSUMER_BUFFER_BASE_V2C)   // Cube's L1 (for producer to DMA into)

    for ...:
        cube_kernel(  ..., GM_SLOT_BUFFER=nullptr,
                           C2V_CONSUMER_BUF=cube_c2v_consumer_buf,
                           V2C_CONSUMER_BUF=cube_v2c_consumer_buf, ...)
        vector_kernel(..., GM_SLOT_BUFFER=nullptr,
                           C2V_CONSUMER_BUF=vec_c2v_consumer_buf,
                           V2C_CONSUMER_BUF=vec_v2c_consumer_buf, ...)
```

**Key difference**: On A2A3, only the **consumer** side of each `CONSUMER_BUF` argument is meaningful (each kernel receives its own local SRAM slot buffer address). The producer side can be `0` because the producer writes to GM, not to the consumer's SRAM. On A5, **both** the consumer and the paired producer receive the consumer's `CONSUMER_BUFFER_BASE` — the producer needs it to DMA directly into the consumer's SRAM.

#### `aic_initialize_pipe(DIR_MASK, SLOT_SIZE, GM_SLOT_BUFFER, C2V_CONSUMER_BUF, V2C_CONSUMER_BUF)`

**Called on the Cube (AIC) core at kernel startup.** Initializes the ring buffer pipe(s) for the specified direction(s).

| Parameter | Type | Description |
|---|---|---|
| `DIR_MASK` | `uint8_t` | Bitmask of active directions (`DIR_C2V`, `DIR_V2C`, or both) |
| `SLOT_SIZE` | `uint32_t` | Size of each ring buffer slot in bytes (= Tile size) |
| `GM_SLOT_BUFFER` | `__gm__ void*` | Pre-allocated per-cluster GM staging buffer, computed as `GM_SLOT_BUFFER_BASE + PHYSICAL_CORE_CLUSTER_IDX * PER_CLUSTER_SLOT_BUFFER_SIZE`. Passed as **extern mem_ref** (no runtime dependency tracking — synchronization is via hardware flags). Active on A2A3; `nullptr` on A5 |
| `C2V_CONSUMER_BUF` | `uint32_t` | Consumer's SRAM base address for C2V direction (Vector's UB). Active on **both** platforms |
| `V2C_CONSUMER_BUF` | `uint32_t` | Consumer's SRAM base address for V2C direction (Cube's own L1). Active on **both** platforms |

**Note**: The split axis (`TILE_UP_DOWN` / `TILE_LEFT_RIGHT`) is **not** specified during initialization. It is specified per-instruction on `tpush_to_aiv` and `tpop_from_aiv`.

**Note on `GM_SLOT_BUFFER` passing convention**: `GM_SLOT_BUFFER` is **not** tracked as an INOUT argument by the runtime's dependency system. It is passed as an **extern mem_ref** — a pre-allocated region whose coherence is managed entirely by the TPUSH/TPOP hardware flag protocol (P2C/C2P), not by the runtime's task dependency tracker. This avoids false dependencies between tasks that share the same cluster's GM staging buffer.

**Description**: Binds the ring buffer pipe(s) to the appropriate backing memory based on `PLATFORM_ID`, computes `SLOT_NUM` from `DIR_MASK` (8 if unidirectional, 4 if bidirectional), and initializes internal state. `C2V_CONSUMER_BUF` and `V2C_CONSUMER_BUF` are active on **both** platforms — they specify the consumer's local SRAM slot buffer. For each direction where the Cube is the **consumer** (`DIR_V2C`), it signals all slots as free to the Vector producer.

- On `PLATFORM_A2A3`:
  - **C2V (Cube is producer)**: producer DMA target is `GM_SLOT_BUFFER` (staging). `C2V_CONSUMER_BUF` is not used by Cube (it's the Vector consumer's local buffer).
  - **V2C (Cube is consumer)**: `tpop` reads from `GM_SLOT_BUFFER` (staging) and DMAs into `V2C_CONSUMER_BUF` (Cube's own L1).
- On `PLATFORM_A5`:
  - **C2V (Cube is producer)**: producer DMA target is `C2V_CONSUMER_BUF` — the Vector's UB address.
  - **V2C (Cube is consumer)**: `tpop` zero-copy aliases `V2C_CONSUMER_BUF` — Cube's own L1 address.

**Pseudocode**:

```
function aic_initialize_pipe(DIR_MASK, SLOT_SIZE, GM_SLOT_BUFFER, C2V_CONSUMER_BUF, V2C_CONSUMER_BUF):
    if DIR_MASK == (DIR_C2V | DIR_V2C):
        SLOT_NUM = 4
    else:
        SLOT_NUM = 8

    if DIR_MASK & DIR_C2V:
        // Cube is PRODUCER in C2V direction
        if PLATFORM_ID == PLATFORM_A2A3:
            c2v_push_buf = GM_SLOT_BUFFER                         // producer writes to GM
        else:  // PLATFORM_A5
            c2v_push_buf = C2V_CONSUMER_BUF                       // producer writes directly to Vector's UB
        c2v_target_tag = 0

    if DIR_MASK & DIR_V2C:
        // Cube is CONSUMER in V2C direction
        v2c_local_slot_buf = V2C_CONSUMER_BUF                     // consumer's local L1 slot buffer (both platforms)
        if PLATFORM_ID == PLATFORM_A2A3:
            buf_offset = (DIR_MASK & DIR_C2V) ? SLOT_NUM * SLOT_SIZE : 0
            v2c_gm_buf = GM_SLOT_BUFFER + buf_offset              // GM source for tpop DMA
        v2c_pop_tag  = 0
        v2c_free_tag = 0
        // Signal all slots as free to Vector producer(s)
        for (i = 0; i < SLOT_NUM; i++):
            SET flag_V2C_free[AIV0]: i
            SET flag_V2C_free[AIV1]: i
```

#### `aiv_initialize_pipe(DIR_MASK, SLOT_SIZE, GM_SLOT_BUFFER, C2V_CONSUMER_BUF, V2C_CONSUMER_BUF)`

**Called on a Vector (AIV) core at kernel startup.** Initializes the ring buffer pipe(s) for the specified direction(s).

| Parameter | Type | Description |
|---|---|---|
| `DIR_MASK` | `uint8_t` | Bitmask of active directions (`DIR_C2V`, `DIR_V2C`, or both) |
| `SLOT_SIZE` | `uint32_t` | Size of each ring buffer slot in bytes (= Tile size) |
| `GM_SLOT_BUFFER` | `__gm__ void*` | Pre-allocated per-cluster GM staging buffer, computed as `GM_SLOT_BUFFER_BASE + PHYSICAL_CORE_CLUSTER_IDX * PER_CLUSTER_SLOT_BUFFER_SIZE`. Passed as **extern mem_ref** (no runtime dependency tracking). Active on A2A3; `nullptr` on A5 |
| `C2V_CONSUMER_BUF` | `uint32_t` | Consumer's SRAM base address for C2V direction (Vector's own UB). Active on **both** platforms |
| `V2C_CONSUMER_BUF` | `uint32_t` | Consumer's SRAM base address for V2C direction (Cube's L1). Active on **both** platforms |

**Note**: The split axis (`TILE_UP_DOWN` / `TILE_LEFT_RIGHT`) is **not** specified during initialization. It is specified per-instruction on `tpush_to_aic` and `tpop_from_aic`.

**Description**: Binds the ring buffer pipe(s) to the appropriate backing memory based on `PLATFORM_ID`, computes `SLOT_NUM`, and initializes internal state. `C2V_CONSUMER_BUF` and `V2C_CONSUMER_BUF` are active on **both** platforms — they specify the consumer's local SRAM slot buffer. For each direction where the Vector is the **consumer** (`DIR_C2V`), it signals all slots as free to the Cube producer.

**Pseudocode**:

```
function aiv_initialize_pipe(DIR_MASK, SLOT_SIZE, GM_SLOT_BUFFER, C2V_CONSUMER_BUF, V2C_CONSUMER_BUF):
    if DIR_MASK == (DIR_C2V | DIR_V2C):
        SLOT_NUM = 4
    else:
        SLOT_NUM = 8

    my_aiv_idx = get_my_aiv_idx()   // 0 or 1

    if DIR_MASK & DIR_C2V:
        // Vector is CONSUMER in C2V direction
        c2v_local_slot_buf = C2V_CONSUMER_BUF                     // consumer's local UB slot buffer (both platforms)
        if PLATFORM_ID == PLATFORM_A2A3:
            c2v_gm_buf = GM_SLOT_BUFFER                           // GM source for tpop DMA
        c2v_pop_tag  = 0
        c2v_free_tag = 0
        // Signal all slots as free to Cube producer (only signal my own flag)
        for (i = 0; i < SLOT_NUM; i++):
            SET flag_C2V_free[my_aiv_idx]: i

    if DIR_MASK & DIR_V2C:
        // Vector is PRODUCER in V2C direction
        if PLATFORM_ID == PLATFORM_A2A3:
            buf_offset = (DIR_MASK & DIR_C2V) ? SLOT_NUM * SLOT_SIZE : 0
            v2c_push_buf = GM_SLOT_BUFFER + buf_offset            // producer writes to GM
        else:  // PLATFORM_A5
            v2c_push_buf = V2C_CONSUMER_BUF                       // producer writes directly to Cube's L1
        v2c_target_tag = 0
```

#### Buffer Layout and Cross-Core Address Problem on A5

##### A2A3: Pre-allocated Per-Cluster GM Layout

On A2A3, the ring buffer resides in GM. Instead of per-task dynamic allocation, the runtime **pre-allocates** a contiguous `GM_SLOT_BUFFER` region at startup, sized to cover all core clusters. Each cluster's portion is indexed by `PHYSICAL_CORE_CLUSTER_IDX`, and both InCore functions within a cluster share the same physical GM addresses.

```
GM_SLOT_BUFFER_BASE (reserved at runtime init, outside task memory ring):

    ┌──────────────────────────────────────────────────────────────────┐
    │  Cluster 0                    │  Cluster 1                      │
    │  PER_CLUSTER_SLOT_BUFFER_SIZE │  PER_CLUSTER_SLOT_BUFFER_SIZE   │  ...
    │  = NUM_BUDDIES * L0C_SIZE     │  = NUM_BUDDIES * L0C_SIZE       │
    ├───────────────┬───────────────┼───────────────┬─────────────────┤
    │  Buddy 0 buf  │  Buddy 1 buf  │  Buddy 0 buf  │  Buddy 1 buf   │  ...
    │  (C2V + V2C)  │  (C2V + V2C)  │  (C2V + V2C)  │  (C2V + V2C)   │
    └───────────────┴───────────────┴───────────────┴─────────────────┘
    │←────── cluster_idx=0 ────────►│←────── cluster_idx=1 ──────────►│

Per-cluster layout (bidirectional, per buddy):

    ┌─────────────────────────────┬─────────────────────────────┐
    │  C2V ring buffer            │  V2C ring buffer            │
    │  slot[0] .. slot[SLOT_NUM-1]│  slot[0] .. slot[SLOT_NUM-1]│
    │  offset: 0                  │  offset: SLOT_NUM*SLOT_SIZE │
    └─────────────────────────────┴─────────────────────────────┘

Address computation:
    my_gm_slot_buffer = GM_SLOT_BUFFER_BASE + PHYSICAL_CORE_CLUSTER_IDX * PER_CLUSTER_SLOT_BUFFER_SIZE
```

##### Consumer SRAM Slot Buffer (Both Platforms)

On **both** A2A3 and A5, the consumer InCore function reserves a segment of its local SRAM (UB for Vector, L1 for Cube) as a **slot buffer** for TPUSH/TPOP ring buffer data. This is where `tpop` places (or aliases) the received data, and where subsequent compute instructions reference it until `tfree`.

##### `CONSUMER_BUFFER_BASE` / `CONSUMER_BUFFER_SIZE` Constant Symbols

Two **constant symbols** are attached to **each InCore function** that participates in TPUSH/TPOP communication:

```cpp
const uint32_t CONSUMER_BUFFER_BASE;   // base address of the consumer's ring buffer in its local SRAM
const uint32_t CONSUMER_BUFFER_SIZE;   // total size in bytes (= SLOT_NUM * SLOT_SIZE)
```

These symbols represent a **reserved memory segment** in the consumer InCore function's local SRAM (UB for Vector, L1 for Cube). They are used on **both** A2A3 and A5. The key properties are:

1. **Per-function constants**: Each InCore function that acts as a **consumer** in any TPUSH/TPOP direction has its own `CONSUMER_BUFFER_BASE` and `CONSUMER_BUFFER_SIZE`. The resolved values are passed as `C2V_CONSUMER_BUF` / `V2C_CONSUMER_BUF` arguments to the initialization functions on **both** platforms.

2. **Value origin**:
   - **Auto-generated kernels** (`auto_incore` / `ExpandMixedKernel`): The values are generated by the `ExpandMixedKernel` pass, which has visibility into both producer and consumer functions' memory layouts and can assign a non-overlapping SRAM region for the ring buffer.
   - **Manually written kernels**: The programmer specifies `CONSUMER_BUFFER_BASE` and `CONSUMER_BUFFER_SIZE` as explicit constant declarations in the InCore function. The values must be chosen to not conflict with other SRAM usage.

3. **Address allocator reservation**: The downstream **memory address allocator** (e.g., `AllocateMemoryAddr` pass) must treat the segment `[CONSUMER_BUFFER_BASE, CONSUMER_BUFFER_BASE + CONSUMER_BUFFER_SIZE)` as **occupied / in-use** in the consumer function's SRAM on **both** platforms. It must **not** allocate any other symbols (tiles, temporaries, etc.) into this region.

4. **Cross-function visibility (A5 only)**: On A5, the producer needs to know the consumer's `CONSUMER_BUFFER_BASE` to DMA data directly into the consumer's SRAM. The compiler ensures this by generating both functions in the same compilation unit and emitting the consumer's `CONSUMER_BUFFER_BASE` as a constant in the producer's initialization code. On A2A3, the producer writes to GM and does **not** need the consumer's local buffer address.

##### A5-Specific: Cross-Core Address Problem

On A5, the ring buffer for each direction resides in the **consumer's on-chip SRAM**. The producer needs to know this address to DMA data into it — but it lives in another core's address space:

```
A5 Problem: C2V direction (Cube produces → Vector consumes)

    Cube InCore function:                    Vector InCore function:
    ┌─────────────────────┐                 ┌─────────────────────┐
    │  tpush_to_aiv       │   ??? how to    │  consumer_buf =     │
    │  DMA to Vector's UB │ ──────────────▶ │  UB[BASE..BASE+SIZE]│
    │  at what address?   │   get address?  │  // local segment   │
    └─────────────────────┘                 └─────────────────────┘
```

The solution is the `CONSUMER_BUFFER_BASE` mechanism described above — the resolved address is passed as an explicit argument (`C2V_CONSUMER_BUF`) to the producer's initialization, so the producer knows where to DMA.

On A2A3, this problem does not exist: the producer writes to GM, and only the consumer reads from GM into its own local `CONSUMER_BUFFER`. No cross-core address is needed.

##### Buffer Layout Summary

Both platforms have consumer-local SRAM slot buffers with the same layout. A2A3 additionally uses GM as an intermediate staging buffer for producer → consumer data transfer.

```
A2A3 (GM staging buffer + consumer-local SRAM slot buffer):

    GM_SLOT_BUFFER (producer writes here via tpush):
    ┌─────────────────────────────┬─────────────────────────────┐
    │  C2V ring buffer            │  V2C ring buffer            │
    │  slot[0] .. slot[SLOT_NUM-1]│  slot[0] .. slot[SLOT_NUM-1]│
    └──────────────┬──────────────┴──────────────┬──────────────┘
                   │ tpop: DMA GM→local          │ tpop: DMA GM→local
                   ▼                             ▼
    Vector UB (for C2V, Vector is consumer):  Cube L1 (for V2C, Cube is consumer):
    ┌──────────┬──────────────────────┐       ┌──────────┬──────────────────────┐
    │ normal   │  CONSUMER_BUFFER     │       │ normal   │  CONSUMER_BUFFER     │
    │ tiles    │  slot[0]..slot[N-1]  │       │ tiles    │  slot[0]..slot[N-1]  │
    └──────────┴──────────────────────┘       └──────────┴──────────────────────┘
    ◄── allocator avoids this region ──►      ◄── allocator avoids this region ──►

A5 (consumer-local SRAM slot buffer only, no GM):

    Vector UB (for C2V, Vector is consumer):
    ┌──────────┬──────────────────────────────┬───────────┐
    │ normal   │  CONSUMER_BUFFER segment     │ normal    │
    │ tiles    │  [BASE .. BASE+SIZE)         │ tiles     │
    │          │  slot[0] .. slot[SLOT_NUM-1] │           │
    └──────────┴──────────────────────────────┴───────────┘
    ◄─── allocator avoids this region ───►

    Cube L1 (for V2C, Cube is consumer):
    ┌──────────┬──────────────────────────────┬───────────┐
    │ normal   │  CONSUMER_BUFFER segment     │ normal    │
    │ tiles    │  [BASE .. BASE+SIZE)         │ tiles     │
    │          │  slot[0] .. slot[SLOT_NUM-1] │           │
    └──────────┴──────────────────────────────┴───────────┘
```

#### Data Transfer Instructions

The ISA defines **six instructions**: four for data transfer (`tpush`/`tpop`) and two for deferred slot release (`tfree`). Each instruction is executed on a specific core type with an implicit direction.

| Instruction | Executed On | Role | Direction | Description |
|---|---|---|---|---|
| `tpush_to_aiv(TILE, SPLIT)` | **Cube** | Producer | C2V | Push tile from Cube to buddy Vector(s) |
| `tpush_to_aic(TILE, SPLIT)` | **Vector** | Producer | V2C | Push tile from Vector to buddy Cube |
| `tpop_from_aic(TILE, SPLIT)` | **Vector** | Consumer | C2V | Pop tile that Cube pushed (does **not** free the slot) |
| `tpop_from_aiv(TILE, SPLIT)` | **Cube** | Consumer | V2C | Pop tile that Vector(s) pushed (does **not** free the slot) |
| `tfree_from_aic(SPLIT)` | **Vector** | Consumer | C2V | Free the oldest un-freed C2V slot after consumer is done using it |
| `tfree_from_aiv(SPLIT)` | **Cube** | Consumer | V2C | Free the oldest un-freed V2C slot after consumer is done using it |

**`SPLIT` parameter values**:
- `TILE_NO_SPLIT`: 1:1 mode (full tile to/from single AIV)
- `TILE_UP_DOWN`: 1:2 mode (split/combine along rows)
- `TILE_LEFT_RIGHT`: 1:2 mode (split/combine along cols)

#### `tpush_to_aiv(TILE, SPLIT)` — C2V Direction

**Executed on Cube (AIC).** Pushes a tile into the C2V ring buffer destined for buddy Vector core(s).

**1:1 Mode** (`SPLIT == TILE_NO_SPLIT`):
```
function tpush_to_aiv(TILE, TILE_NO_SPLIT):
    // Wait for single AIV's free flag
    WAIT flag_free[C2V, target_aiv]: target_tag

    // DMA full tile to target AIV's slot
    dst_addr = ring_buf_base + target_tag * SLOT_SIZE
    MTE_copy(src=TILE.data, dst=dst_addr, size=SLOT_SIZE)
    WAIT mte_flag

    // Signal single AIV ready
    SET flag_ready[C2V, target_aiv]: target_tag
    target_tag = (target_tag + 1) % SLOT_NUM
```

**1:2 Mode** (`SPLIT == TILE_UP_DOWN` or `TILE_LEFT_RIGHT`):
```
function tpush_to_aiv(TILE, SPLIT):
    // Wait for BOTH AIV0 and AIV1 free flags
    WAIT flag_free[C2V, AIV0]: target_tag
    WAIT flag_free[C2V, AIV1]: target_tag

    // Split tile and DMA halves to both AIVs
    if SPLIT == TILE_UP_DOWN:
        // Upper half to AIV0, lower half to AIV1
        MTE_copy(src=TILE.upper_half, dst=aiv0_slot[target_tag], size=HALF_TILE_SIZE)
        MTE_copy(src=TILE.lower_half, dst=aiv1_slot[target_tag], size=HALF_TILE_SIZE)
    else:  // TILE_LEFT_RIGHT
        // Left half to AIV0, right half to AIV1
        MTE_copy(src=TILE.left_half,  dst=aiv0_slot[target_tag], size=HALF_TILE_SIZE)
        MTE_copy(src=TILE.right_half, dst=aiv1_slot[target_tag], size=HALF_TILE_SIZE)
    WAIT mte_flags

    // Signal BOTH AIV0 and AIV1 ready
    SET flag_ready[C2V, AIV0]: target_tag
    SET flag_ready[C2V, AIV1]: target_tag
    target_tag = (target_tag + 1) % SLOT_NUM
```

#### `tpush_to_aic(TILE, SPLIT)` — V2C Direction

**Executed on Vector (AIV).** Pushes a tile into the V2C ring buffer destined for the buddy Cube core.

**1:1 Mode** (`SPLIT == TILE_NO_SPLIT`):
```
function tpush_to_aic(TILE, TILE_NO_SPLIT):
    // Wait for my free flag
    WAIT flag_free[V2C, my_aiv_idx]: target_tag

    // DMA full tile to slot
    dst_addr = ring_buf_base + target_tag * SLOT_SIZE
    MTE_copy(src=TILE.data, dst=dst_addr, size=SLOT_SIZE)
    WAIT mte_flag

    // Signal my ready flag
    SET flag_ready[V2C, my_aiv_idx]: target_tag
    target_tag = (target_tag + 1) % SLOT_NUM
```

**1:2 Mode** (`SPLIT == TILE_UP_DOWN` or `TILE_LEFT_RIGHT`):
```
function tpush_to_aic(TILE, SPLIT):
    // Wait for my free flag
    WAIT flag_free[V2C, my_aiv_idx]: target_tag

    // Compute my write offset based on my AIV index and split axis
    if SPLIT == TILE_UP_DOWN:
        // AIV0 writes upper half (offset 0), AIV1 writes lower half
        dst_offset = my_aiv_idx * HALF_TILE_SIZE
    else:  // TILE_LEFT_RIGHT
        // AIV0 writes left half (offset 0), AIV1 writes right half
        dst_offset = my_aiv_idx * HALF_TILE_SIZE

    // DMA my half to shared L1 slot with correct offset
    dst_addr = ring_buf_base + target_tag * FULL_TILE_SIZE + dst_offset
    MTE_strided_copy(src=TILE.data, dst=dst_addr, ...)
    WAIT mte_flag

    // Signal my ready flag
    SET flag_ready[V2C, my_aiv_idx]: target_tag
    target_tag = (target_tag + 1) % SLOT_NUM
```

#### `tpop_from_aic(TILE, SPLIT)` — C2V Direction

**Executed on Vector (AIV).** Pops a tile from the C2V ring buffer (data that Cube pushed).

**Both 1:1 and 1:2 modes**: Each AIV receives its portion independently. Note that `tpop` does **not** signal the slot as free — the consumer must explicitly call `tfree_from_aic` after it has finished using the data. On both platforms, `TILE.data` references the consumer's **local SRAM slot buffer** after `tpop`.
```
function tpop_from_aic(TILE, SPLIT):
    // Wait for my ready flag
    WAIT flag_ready[C2V, my_aiv_idx]: pop_tag

    // Receive my tile (full in 1:1, half in 1:2)
    local_dst = local_slot_buf[pop_tag]
    if PLATFORM_A2A3:
        // DMA from GM slot to consumer's local SRAM slot buffer
        gm_src = gm_slot[pop_tag]
        MTE_load(dst=local_dst, src=gm_src, size=tile_size)
        WAIT mte_flag
    // else PLATFORM_A5: data already in local_slot_buf (written by producer's tpush)

    TILE.data = local_dst   // both platforms: reference consumer's local SRAM slot

    // Advance pop_tag — but do NOT signal free (deferred to tfree)
    pop_tag = (pop_tag + 1) % SLOT_NUM
```

**Note**: In 1:2 mode, each AIV's kernel knows which portion it received based on `SPLIT`:
- `TILE_UP_DOWN`: AIV0 has upper rows, AIV1 has lower rows
- `TILE_LEFT_RIGHT`: AIV0 has left cols, AIV1 has right cols

**Why `tpop` does not signal free**: After `tpop`, `TILE.data` references the consumer's local SRAM slot buffer. Subsequent compute instructions operate on this data in-place. If `tpop` immediately signaled the slot as free, the producer could cause a new `tpush` → DMA chain that overwrites the consumer's slot buffer before the consumer finishes using it. On A5, the producer's `tpush` DMAs directly into the consumer's slot buffer; on A2A3, a subsequent `tpop` for the same tag would overwrite the local slot. The separate `tfree_from_aic` instruction lets the consumer explicitly indicate that the slot is no longer referenced and can be reused.

#### `tpop_from_aiv(TILE, SPLIT)` — V2C Direction

**Executed on Cube (AIC).** Pops a tile from the V2C ring buffer (data that Vector(s) pushed).

Note that `tpop` does **not** signal the slot as free — the consumer must explicitly call `tfree_from_aiv` after it has finished using the data. On both platforms, `TILE.data` references the consumer's **local SRAM slot buffer** (L1) after `tpop`.

**1:1 Mode** (`SPLIT == TILE_NO_SPLIT`):
```
function tpop_from_aiv(TILE, TILE_NO_SPLIT):
    // Wait for single AIV's ready flag
    WAIT flag_ready[V2C, target_aiv]: pop_tag

    // Receive full tile into consumer's local slot buffer
    local_dst = local_slot_buf[pop_tag]
    if PLATFORM_A2A3:
        // DMA from GM slot to consumer's local SRAM slot buffer
        gm_src = gm_slot[pop_tag]
        MTE_load(dst=local_dst, src=gm_src, size=SLOT_SIZE)
        WAIT mte_flag
    // else PLATFORM_A5: data already in local_slot_buf (written by producer's tpush)

    TILE.data = local_dst   // both platforms: reference consumer's local L1 slot

    // Advance pop_tag — but do NOT signal free (deferred to tfree)
    pop_tag = (pop_tag + 1) % SLOT_NUM
```

**1:2 Mode** (`SPLIT == TILE_UP_DOWN` or `TILE_LEFT_RIGHT`):
```
function tpop_from_aiv(TILE, SPLIT):
    // Wait for BOTH AIV0 and AIV1 ready flags
    WAIT flag_ready[V2C, AIV0]: pop_tag
    WAIT flag_ready[V2C, AIV1]: pop_tag

    // Combined tile is in GM slot (A2A3) or local slot (A5)
    local_dst = local_slot_buf[pop_tag]
    if PLATFORM_A2A3:
        // DMA from GM slot to consumer's local SRAM slot buffer
        gm_src = gm_slot[pop_tag]
        MTE_strided_load(dst=local_dst, src=gm_src, ...)
        WAIT mte_flag
    // else PLATFORM_A5: combined data already in local_slot_buf

    TILE.data = local_dst   // both platforms: reference consumer's local L1 slot

    // Advance pop_tag — but do NOT signal free (deferred to tfree)
    pop_tag = (pop_tag + 1) % SLOT_NUM
```

#### `tfree_from_aic(SPLIT)` — C2V Direction (Consumer Free)

**Executed on Vector (AIV).** Signals that the consumer has finished using the data from the most recently un-freed C2V slot, making it available for the Cube producer to reuse. Must be called once for each preceding `tpop_from_aic`.

The consumer maintains a separate `free_tag` that trails `pop_tag`. Each `tfree` advances `free_tag` by one and signals the corresponding C2P free flag.

```
function tfree_from_aic(SPLIT):
    // Signal my free flag for the oldest un-freed slot
    SET flag_free[C2V, my_aiv_idx]: free_tag
    free_tag = (free_tag + 1) % SLOT_NUM
```

**Note**: `SPLIT` is accepted for API consistency but does not affect the free signaling — each AIV frees its own flag independently regardless of 1:1 or 1:2 mode.

**Invariant**: `free_tag` must never advance past `pop_tag`. The number of outstanding un-freed slots (`pop_tag - free_tag`, modulo `SLOT_NUM`) must be in the range `[0, SLOT_NUM)`. Calling `tfree` without a matching prior `tpop` is undefined behavior.

#### `tfree_from_aiv(SPLIT)` — V2C Direction (Consumer Free)

**Executed on Cube (AIC).** Signals that the consumer has finished using the data from the most recently un-freed V2C slot, making it available for the Vector producer(s) to reuse. Must be called once for each preceding `tpop_from_aiv`.

**1:1 Mode** (`SPLIT == TILE_NO_SPLIT`):
```
function tfree_from_aiv(TILE_NO_SPLIT):
    // Signal single AIV free
    SET flag_free[V2C, target_aiv]: free_tag
    free_tag = (free_tag + 1) % SLOT_NUM
```

**1:2 Mode** (`SPLIT == TILE_UP_DOWN` or `TILE_LEFT_RIGHT`):
```
function tfree_from_aiv(SPLIT):
    // Signal BOTH AIV0 and AIV1 free
    SET flag_free[V2C, AIV0]: free_tag
    SET flag_free[V2C, AIV1]: free_tag
    free_tag = (free_tag + 1) % SLOT_NUM
```

**Invariant**: Same as `tfree_from_aic` — `free_tag` must never advance past `pop_tag`.

**Pipeline depth control**: The gap between `pop_tag` and `free_tag` determines how many slot buffers the consumer holds simultaneously. A consumer that calls `tfree` immediately after each `tpop` behaves like the original design (minimum pipeline depth = 1 slot). A consumer that defers `tfree` across multiple iterations can hold up to `SLOT_NUM - 1` slots simultaneously, enabling deeper compute-communication overlap at the cost of reduced ring buffer capacity for the producer.

### Flag Assignment

The 8 hardware flags per direction per peer are mapped as follows:

```
Unidirectional (DIR_C2V only, SLOT_NUM=8):

    flag_ready[C2V, aiv_idx] : flags 0..7   (Cube SETs, Vector WAITs)
    flag_free [C2V, aiv_idx] : flags 0..7   (Vector SETs, Cube WAITs)

Bidirectional (DIR_C2V | DIR_V2C, SLOT_NUM=4):

    flag_ready[C2V, aiv_idx] : flags 0..3   (Cube SETs, Vector WAITs)
    flag_free [C2V, aiv_idx] : flags 0..3   (Vector SETs, Cube WAITs)
    flag_ready[V2C, aiv_idx] : flags 4..7   (Vector SETs, Cube WAITs)
    flag_free [V2C, aiv_idx] : flags 4..7   (Cube SETs, Vector WAITs)
```

### Timing Diagram: C2V 1:2 Mode with TILE_UP_DOWN

```
          iter 0              iter 1              iter 2
tag:        0                   1                   2

AIC (Cube, producer):
          ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐
          │ tpush_to_aiv     │  │ tpush_to_aiv     │  │ tpush_to_aiv     │
          │ WAIT f[0]:0      │  │ WAIT f[0]:1      │  │ WAIT f[0]:2      │
          │ WAIT f[1]:0      │  │ WAIT f[1]:1      │  │ WAIT f[1]:2      │
          │ MTE upper→AIV0   │  │ MTE upper→AIV0   │  │ MTE upper→AIV0   │
          │ MTE lower→AIV1   │  │ MTE lower→AIV1   │  │ MTE lower→AIV1   │
          │ SET r[0]:0       │  │ SET r[0]:1       │  │ SET r[0]:2       │
          │ SET r[1]:0       │  │ SET r[1]:1       │  │ SET r[1]:2       │
          └────────┬─────────┘  └────────┬─────────┘  └────────┬─────────┘
                   │                     │                     │
                   ▼ ready               ▼ ready               ▼ ready

AIV0 (Vector, consumer, receives UPPER half):
               ┌──────────────┐                  ┌──────────────┐
               │ tpop_from_aic│                  │ tpop_from_aic│
               │ WAIT r[0]:0  │                  │ WAIT r[0]:1  │
               │ (pop_tag→1)  │                  │ (pop_tag→2)  │
               └──────┬───────┘                  └──────┬───────┘
                      │ use upper half                   │ use upper half
               ┌──────┴───────┐                  ┌──────┴───────┐
               │tfree_from_aic│                  │tfree_from_aic│
               │ SET f[0]:0   │                  │ SET f[0]:1   │
               │ (free_tag→1) │                  │ (free_tag→2) │
               └──────┬───────┘                  └──────┬───────┘

AIV1 (Vector, consumer, receives LOWER half):
               ┌──────────────┐                  ┌──────────────┐
               │ tpop_from_aic│                  │ tpop_from_aic│
               │ WAIT r[1]:0  │                  │ WAIT r[1]:1  │
               │ (pop_tag→1)  │                  │ (pop_tag→2)  │
               └──────┬───────┘                  └──────┬───────┘
                      │ use lower half                   │ use lower half
               ┌──────┴───────┐                  ┌──────┴───────┐
               │tfree_from_aic│                  │tfree_from_aic│
               │ SET f[1]:0   │                  │ SET f[1]:1   │
               │ (free_tag→1) │                  │ (free_tag→2) │
               └──────┴───────┘                  └──────┴───────┘

Legend: r[x] = flag_ready[aiv_x], f[x] = flag_free[aiv_x]
```

### Timing Diagram: V2C 1:2 Mode with TILE_UP_DOWN

```
          iter 0              iter 1              iter 2
tag:        0                   1                   2

AIV0 (Vector, producer, writes UPPER half):
          ┌──────────────────┐  ┌──────────────────┐
          │ tpush_to_aic     │  │ tpush_to_aic     │
          │ WAIT f[0]:0      │  │ WAIT f[0]:1      │
          │ MTE upper→L1+0   │  │ MTE upper→L1+0   │
          │ SET r[0]:0       │  │ SET r[0]:1       │
          └────────┬─────────┘  └────────┬─────────┘

AIV1 (Vector, producer, writes LOWER half):
          ┌──────────────────┐  ┌──────────────────┐
          │ tpush_to_aic     │  │ tpush_to_aic     │
          │ WAIT f[1]:0      │  │ WAIT f[1]:1      │
          │ MTE lower→L1+off │  │ MTE lower→L1+off │
          │ SET r[1]:0       │  │ SET r[1]:1       │
          └────────┬─────────┘  └────────┬─────────┘
                   │                     │
                   ▼ ready               ▼ ready

AIC (Cube, consumer):
               ┌───────────────┐                  ┌───────────────┐
               │ tpop_from_aiv │                  │ tpop_from_aiv │
               │ WAIT r[0]:0   │                  │ WAIT r[0]:1   │
               │ WAIT r[1]:0   │                  │ WAIT r[1]:1   │
               │ (pop_tag→1)   │                  │ (pop_tag→2)   │
               └───────┬───────┘                  └───────┬───────┘
                       │ use combined tile                │ use combined tile
               ┌───────┴───────┐                  ┌───────┴───────┐
               │tfree_from_aiv │                  │tfree_from_aiv │
               │ SET f[0]:0    │                  │ SET f[0]:1    │
               │ SET f[1]:0    │                  │ SET f[1]:1    │
               │ (free_tag→1)  │                  │ (free_tag→2)  │
               └───────┬───────┘                  └───────┬───────┘
```

### Key Properties

1. **No deadlock**: The consumer side (`aiv_initialize_pipe` or `aic_initialize_pipe`) pre-signals all SLOT_NUM slots as free before the main loop begins, so the producer can fill up to SLOT_NUM slots before blocking.
2. **Backpressure**: If the producer is faster than the consumer, `tpush_*` blocks at `WAIT flag_free` when all slots are occupied; if the consumer is faster, `tpop_*` blocks at `WAIT flag_ready` when no data is ready.
3. **In-order delivery**: Producer advances `push_tag`, consumer advances `pop_tag` and `free_tag`, all in strict round-robin order `(tag + 1) % SLOT_NUM`, guaranteeing FIFO semantics.
4. **Decoupled DMA**: `tpush_*` uses MTE for async data transfer with an explicit `mte_flag` wait to ensure completion before signaling the consumer.
5. **Deferred slot release (`tfree`)**: `tpop` does **not** signal the slot as free. The consumer must explicitly call `tfree` after it has finished referencing the popped data. This prevents the producer from overwriting slot buffer contents that the consumer is still using (critical for A5 zero-copy where `tpop` aliases the slot buffer directly). The consumer maintains two tags: `pop_tag` (advanced by `tpop`) and `free_tag` (advanced by `tfree`), with invariant `free_tag ≤ pop_tag` (modular).
6. **Pipeline depth control**: The gap `pop_tag - free_tag` (modular) determines how many slots the consumer holds simultaneously. Calling `tfree` immediately after each `tpop` gives minimum latency (1 slot held). Deferring `tfree` across iterations enables deeper compute-communication overlap.
7. **1:2 Mode Flow Control**:
   - **C2V**: Cube waits for **both** AIV0 and AIV1 free, then signals **both** ready.
   - **V2C**: Each AIV waits for its **own** free, signals its **own** ready. Cube waits for **both** ready, frees **both** via `tfree_from_aiv`.
8. **Split Axis Semantic**: The `TileSplitAxis` set during init tells each kernel which portion of data it handles:
   - `TILE_UP_DOWN`: AIV0 = upper rows, AIV1 = lower rows
   - `TILE_LEFT_RIGHT`: AIV0 = left cols, AIV1 = right cols

### API Summary

| API | Called On | Role | Direction | Description |
|---|---|---|---|---|
| `aic_initialize_pipe(DIR_MASK, SLOT_SIZE, GM_SLOT_BUFFER, C2V_CONSUMER_BUF, V2C_CONSUMER_BUF)` | Cube (AIC) | Setup | — | Bind ring buffer, init `push_tag`/`pop_tag`/`free_tag`, pre-signal free slots |
| `aiv_initialize_pipe(DIR_MASK, SLOT_SIZE, GM_SLOT_BUFFER, C2V_CONSUMER_BUF, V2C_CONSUMER_BUF)` | Vector (AIV) | Setup | — | Bind ring buffer, init `push_tag`/`pop_tag`/`free_tag`, pre-signal free slots |
| `tpush_to_aiv(TILE, SPLIT)` | Cube (AIC) | Producer | C2V | 1:1: push to single AIV; 1:2: split and push to both AIVs |
| `tpush_to_aic(TILE, SPLIT)` | Vector (AIV) | Producer | V2C | 1:1: push full tile; 1:2: push my half with strided write |
| `tpop_from_aic(TILE, SPLIT)` | Vector (AIV) | Consumer | C2V | Receive my portion (does **not** free slot). A5: zero-copy; A2A3: load from GM |
| `tpop_from_aiv(TILE, SPLIT)` | Cube (AIC) | Consumer | V2C | 1:1: receive from single AIV; 1:2: wait both, receive combined (does **not** free slot) |
| `tfree_from_aic(SPLIT)` | Vector (AIV) | Consumer | C2V | Signal oldest un-freed C2V slot as reusable by producer |
| `tfree_from_aiv(SPLIT)` | Cube (AIC) | Consumer | V2C | Signal oldest un-freed V2C slot as reusable by producer(s) |

### DSL Grammar: `pl.reserve_buffer` — Reserved Address Space Declaration

*(This section remains unchanged from the original document.)*

The compiler must provide a **DSL-level mechanism** for InCore kernel programs to declare reserved address space for the SLOT_BUFFER. This is necessary because:

1. The address allocator must know which SRAM regions are off-limits **before** it runs.
2. For manually written InCore kernels, the programmer needs an explicit way to express "this region of my local SRAM is reserved for TPUSH/TPOP ring buffer."
3. For compiler-generated kernels (`auto_incore` / `ExpandMixedKernel`), the pass emits the same declaration into the generated IR, so the rest of the pipeline treats it uniformly.

---

## Version History

### v3.1 — Deferred Slot Release (`tfree`) and Uniform Consumer Slot Buffer

**Date**: 2026-03-14

**Problem 1 — Premature slot reuse**: In the previous design, `tpop` immediately signaled the slot as free (via `SET flag_free`). The consumer's `TILE.data` references the slot buffer in SRAM. If the producer issues a `tpush` before the consumer finishes using this data, it overwrites the slot contents, corrupting the consumer's in-flight computation.

**Problem 2 — A2A3 had no consumer-local slot buffer**: Previously, A2A3 `tpop` loaded data from GM into an arbitrary `TILE.data` destination (not a structured slot buffer). This made the consumer code path platform-dependent and inconsistent with A5's zero-copy slot buffer model.

**Key Design Decisions**:

1. **Split `tpop` into `tpop` + `tfree`**: `tpop` now only receives data and advances `pop_tag`. It does **not** signal the slot as free. A new `tfree` instruction (`tfree_from_aic`, `tfree_from_aiv`) explicitly signals `SET flag_free` and advances `free_tag`.

2. **Two consumer-side tags**: Each consumer direction now maintains:
   - `pop_tag` — next slot to pop (advanced by `tpop`)
   - `free_tag` — next slot to free (advanced by `tfree`)
   - Invariant: `free_tag ≤ pop_tag` (modular arithmetic)

3. **Pipeline depth control**: The gap `pop_tag - free_tag` determines how many slots the consumer holds. The consumer can defer `tfree` across multiple iterations to overlap compute with communication, trading ring buffer capacity for pipeline depth.

4. **Uniform consumer-local slot buffer on both platforms**: `CONSUMER_BUFFER_BASE` / `CONSUMER_BUFFER_SIZE` and the `C2V_CONSUMER_BUF` / `V2C_CONSUMER_BUF` arguments are now active on **both** A2A3 and A5. On both platforms, `tpop` places data into the consumer's local SRAM slot buffer, and `TILE.data` references this local buffer. The consumer-local slot buffer is organized identically on both platforms, so the **kernel program looks the same on A2/A3 and A5**. The only difference is the transport path hidden inside the `tpush`/`tpop` implementation:
   - **A2A3**: producer → GM staging (`tpush`), GM → consumer SRAM (`tpop` DMA)
   - **A5**: producer → consumer SRAM directly (`tpush` DMA), zero-copy `tpop`

5. **GM_SLOT_BUFFER is merely a staging buffer**: On A2A3, `GM_SLOT_BUFFER` facilitates data transfer through GM but is never directly referenced by kernel code. The kernel always operates on the consumer's local SRAM slot buffer. This staging role is why GM_SLOT_BUFFER is pre-allocated per core cluster at runtime init and not per task.

6. **Platform-uniform consumer API**: The `tpop → compute on TILE.data → tfree` sequence is identical on both platforms. Consumer kernel code does not need platform-specific branches, enabling a single kernel binary to target both A2/A3 and A5 (with only the transport layer adapting).

**Rationale**: `tpop` produces a reference to slot buffer data in the consumer's local SRAM. The consumer must be able to use this reference across multiple subsequent instructions before releasing the slot. Without `tfree`, the only safe pattern would be to immediately copy `tpop` data to a separate buffer — defeating the purpose of zero-copy on A5 and adding unnecessary overhead. By organizing the consumer-local slot buffer identically across platforms, kernel programs are platform-independent — the `tpop → compute → tfree` pattern is the same on A2/A3 and A5, greatly simplifying compiler codegen and manual kernel development.

### v2.1 — 1:1 and 1:2 Modes with Tile Split/Combine Axis

**Date**: 2026-03-16

**Key Design Decisions**:

1. **Added `TileSplitAxis` enum** with three values:
   - `TILE_NO_SPLIT` — 1:1 mode, full tile to/from single AIV (AIV0 by default)
   - `TILE_UP_DOWN` — 1:2 mode, split/combine along rows (upper/lower halves)
   - `TILE_LEFT_RIGHT` — 1:2 mode, split/combine along cols (left/right halves)

2. **Split axis specified per-instruction** (not during init):
   - `initialize_pipe` does not take split params
   - Each `tpush_*` / `tpop_*` instruction takes `SPLIT` as second parameter
   - Rationale: Split axis is per-operation, not per-pipe property

3. **Platform-specific split/combine implementation**:
   - **A5**: Split/combine logic in `tpush` — producer directly writes to consumer's on-chip SRAM with correct layout; `tpop` is sync-only (zero-copy)
   - **A2A3**: Split/combine logic in `tpop` — consumer must load data from GM (not a no-op sync). Implementation options:
     - Strided read from GM and combine on consumer side, OR
     - Strided write to consumer's local buffer during read
     - Choice depends on GM access pattern performance on A3

4. **1:1 mode uses AIV0 by default** for forward compatibility:
   - When upgrading to 1:2 mode, AIV0's role remains the same
   - AIV1 is simply added for the other half

5. **Updated flow control semantics**:
   - **C2V 1:2**: Cube `tpush_to_aiv` waits for **both** AIV free flags, signals **both** ready
   - **V2C 1:2**: Each AIV `tpush_to_aic` waits its own free, signals its own ready; Cube `tpop_from_aiv` waits **both** ready, signals **both** free

6. **User-friendly naming**: Used `UP_DOWN` / `LEFT_RIGHT` instead of `M` / `N` axis — users immediately understand the tile partition without needing to know the M/N axis mapping.

**Rationale**: The split/combine axis **must be specified** so Vector kernel code knows which portion of the tile it receives or produces. Without this information, compute code cannot be correct.
