# RCCL — topology detection, ring algorithm, and multi-link parallelism

> **Note:** This is general RCCL/NCCL-family design knowledge, not sourced
> from this repo. RCCL's actual implementation lives in
> [`ROCm/rocm-systems/projects/rccl`](https://github.com/ROCm/rocm-systems/tree/develop/projects/rccl).
> Treat this as a conceptual reference, not a spec.

## Topology detection

At communicator init, RCCL discovers the system's compute and interconnect
layout, builds a weighted graph of it, then searches that graph for
communication paths before any collective ever runs.

```mermaid
flowchart TB
    subgraph DISCOVER[" Discovery inputs "]
        direction LR
        HWLOC["hwloc<br/><i>CPU/NUMA topology, PCIe tree</i>"]
        ROCR["ROCr / rocm-smi<br/><i>GPU enumeration, xGMI links</i>"]
        NET["Network plugin<br/><i>libfabric / UCX NIC discovery</i>"]
    end

    subgraph GRAPH[" Topology graph "]
        direction TB
        NODES["Nodes: GPUs, CPUs, NICs, PCIe switches"]
        EDGES["Edges: link type + bandwidth<br/>xGMI &gt; PCIe P2P &gt; PCIe via host &gt; network"]
    end

    subgraph SEARCH[" Path search & channel construction "]
        direction TB
        PATHS["Enumerate candidate rings/trees<br/>between all GPU pairs"]
        SCORE["Score paths by min-bandwidth link<br/>on the path (bottleneck)"]
        CHANNELS["Select N parallel channels<br/>(rings/trees) maximizing aggregate BW"]
    end

    DISCOVER --> GRAPH --> SEARCH
    SCORE --> CHANNELS --> RUN["Runtime: RCCL communicator<br/>ready for collectives"]

    classDef in fill:#f7f3ec,stroke:#756a58,color:#221d16;
    classDef mid fill:#fff4ec,stroke:#c8451f,color:#221d16;
    classDef out fill:#eaf4f1,stroke:#1f6f63,color:#221d16;
    class HWLOC,ROCR,NET in;
    class NODES,EDGES,PATHS,SCORE mid;
    class CHANNELS,RUN out;
```

## Ring algorithm

Each ring/channel connects every GPU to exactly two neighbors. Collectives
like all-reduce run in two pipelined phases — scatter-reduce, then allgather
— so every link in the ring stays saturated simultaneously instead of
funneling through one GPU.

```mermaid
flowchart LR
    G0((GPU 0)) --> G1((GPU 1)) --> G2((GPU 2)) --> G3((GPU 3))
    G3 --> G4((GPU 4)) --> G5((GPU 5)) --> G6((GPU 6)) --> G7((GPU 7))
    G7 --> G0
```

```mermaid
sequenceDiagram
    participant G0 as GPU 0
    participant G1 as GPU 1
    participant G2 as GPU 2
    participant G3 as GPU 3

    Note over G0,G3: Phase 1 — Scatter-Reduce (N−1 steps)
    G0->>G1: chunk A0 (add into A1)
    G1->>G2: chunk B1 (add into B2)
    G2->>G3: chunk C2 (add into C3)
    G3->>G0: chunk D3 (add into D0)
    Note over G0,G3: repeat N−1 times — each GPU ends with one fully-reduced chunk

    Note over G0,G3: Phase 2 — Allgather (N−1 steps)
    G0->>G1: reduced chunk D (forward)
    G1->>G2: reduced chunk A (forward)
    G2->>G3: reduced chunk B (forward)
    G3->>G0: reduced chunk C (forward)
    Note over G0,G3: repeat N−1 times — every GPU ends with all reduced chunks
```

## Multi-link parallelism

A single ring only uses **one** physical link per hop. On a densely
connected node — e.g. an 8-GPU MI300X OAM tray, where each GPU has direct
xGMI links to several (up to all 7) of the other GPUs — a lone ring leaves
the rest of those links idle. RCCL's fix is to run several **channels**
concurrently, each a fully independent ring (or tree) routed over a
*different* one of a GPU's available links, each driven by its own set of
GPU compute units.

```mermaid
flowchart LR
    subgraph SINGLE[" Single channel — 1 ring "]
        direction LR
        A0((GPU 0)) -->|Link A| A1((GPU 1)) --> A2((GPU 2)) --> A3((GPU 3))
        A3 --> A4((GPU 4)) --> A5((GPU 5)) --> A6((GPU 6)) --> A7((GPU 7)) --> A0
        NOTE1["1 of GPU 0's links active<br/>bandwidth ≈ 1× link BW"]
    end

    classDef dim fill:#f7f3ec,stroke:#d8cfbd,color:#756a58;
    class NOTE1 dim;
```

```mermaid
flowchart TB
    G0(("GPU 0"))
    G1((GPU 1))
    G2((GPU 2))
    G3((GPU 3))
    G4((GPU 4))

    G0 -->|Channel 0 hop| G1
    G0 -->|Channel 1 hop| G2
    G0 -->|Channel 2 hop| G3
    G0 -->|Channel 3 hop| G4

    NOTE2["Each labeled edge is GPU 0's outgoing<br/>hop for a distinct, fully independent ring<br/>spanning all 8 GPUs.<br/>4 channels run concurrently on 4 different<br/>physical xGMI links → bandwidth ≈ 4× link BW"]

    classDef ch0 stroke:#c8451f,stroke-width:2px;
    classDef ch1 stroke:#1f6f63,stroke-width:2px;
    classDef ch2 stroke:#7a5cc4,stroke-width:2px;
    classDef ch3 stroke:#b8860b,stroke-width:2px;
    classDef note fill:#f7f3ec,stroke:#d8cfbd,color:#756a58;
    class NOTE2 note;
```

Because each channel maps to its own compute units and rides a different
physical link, the channels execute genuinely in parallel at the hardware
level — not interleaved on one link — so achieved bandwidth scales toward
N × (single-link bandwidth), up to whatever bottleneck comes next (GPU
memory-controller aggregate bandwidth, or the weakest link if the topology
isn't fully symmetric).

| Channels active | GPU 0 links used | Aggregate bandwidth (approx.) |
|---|---|---|
| 1 (single ring) | 1 of 7 | 1× link BW |
| 4 (multi-channel) | 4 of 7 | ~4× link BW |
| 7 (fully saturated) | 7 of 7 | ~7× link BW (theoretical ceiling for GPU 0) |

The same principle extends one layer up to the network: **multi-rail**
support lets a multi-node collective drive several NICs concurrently
(one network channel per HCA) instead of funneling all inter-node traffic
through a single NIC.

This is specifically valuable on dense, direct-link-rich topologies (like
MI300X's 8-GPU tray). On smaller GPU counts or older, less-connected
interconnects, there are fewer independent physical paths to exploit, so
channel count — and the benefit from this technique — naturally drops.
