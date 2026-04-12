---
name: rtl-design
description: Translate requirements into detailed RTL design documents with microarchitecture planning and CBB reuse decisions. Use when designing a module, planning microarchitecture, writing a design spec, analyzing requirements for RTL, or when user asks "how should I implement X in RTL." Also use when user provides a requirement spec and needs a design before coding. Triggers on design planning, architecture decisions, module partitioning, interface definition, or any pre-coding RTL work.
---

# RTL Design Document Generation

## Design Philosophy

Design before coding. A design document is a pre-coding constraint, not post-hoc documentation. Writing Verilog without a design document produces code that works in simulation but fails in integration, timing, or reuse.

Reuse before designing. Check the CBB catalog for existing blocks before designing anything new. Reusing a proven FIFO, arbiter, or CDC module eliminates an entire class of bugs and saves verification effort. Custom logic is a cost; reused logic is a credit.

Requirements drive architecture, not the other way around. Do not start with a clever microarchitecture and then justify it. Start with what the module must do, constrain it with PPA budgets, then choose the simplest microarchitecture that satisfies both.

Every design decision must be traceable to a requirement or a constraint. If you cannot explain why a pipeline stage exists, it should not exist. If you cannot explain why a FIFO is depth-16 instead of depth-8, the depth is wrong. Traceability is what separates a design document from a wishlist.

## Step 1: Requirements Analysis

Extract every constraint from the requirement spec. Missed requirements become change requests during integration, which cost 10x more than catching them here.

### Functional Requirements

For each requirement, extract:
- **Interface**: protocol (AXI4-Stream, APB, custom handshake), port direction, active level
- **Throughput**: sustained rate (transactions per cycle), burst behavior, idle cycles between transactions
- **Latency**: first-response delay, pipeline depth tolerance, worst-case response bound
- **Data format**: bit width, byte ordering, field mapping, alignment, parity or ECC
- **Operating modes**: configuration modes, power states, bypass paths, test modes

**Scenario**: The user provides a requirement that says "implement a packet classifier that inspects the first 4 bytes of each packet and routes to one of 8 output queues." Extract: Interface = AXI4-Stream input, 8x AXI4-Stream output. Throughput = 1 packet header per cycle (must inspect and route within the header arrival window). Latency = routing decision must be available before the packet body arrives (1-2 cycle budget). Data format = first 4 bytes are a key field at bytes [31:0]. Operating modes = bypass mode that routes all traffic to queue 0.

When the requirement spec is ambiguous, make a reasonable assumption, document it explicitly in the design document, and flag it for the user to confirm. Do not stall the design on ambiguities; resolve them with stated assumptions.

### Non-Functional Constraints

These constraints shape microarchitecture decisions directly:
- **Clock frequency**: determines pipeline depth. A 500 MHz target in 7nm may need 3 pipeline stages where 200 MHz needs 1. Ask the user if not specified.
- **Area budget**: LUT/FF count or gate count estimate. Constrained area forces resource sharing; unconstrained area allows parallel paths.
- **Power budget**: dynamic power limits force clock gating, operand isolation, or domain shutdown. Static power limits force smaller implementations.
- **Timing margin**: how much slack is acceptable after synthesis. Tight margins demand careful pipeline partitioning.

**Scenario**: A design targets 400 MHz in a 12nm FPGA. The combinational path through a 32-bit comparator followed by an 8:1 mux is estimated at 3.5 ns. At 400 MHz (2.5 ns period), this path fails timing by 1 ns. The PPA constraint forces a pipeline decision: insert a register between the comparator and the mux, adding 1 cycle of latency but closing timing. Without the frequency constraint, this pipeline stage would be unnecessary overhead.

### Invariants

List every constraint that must hold under all conditions:
- No packet reordering on a given flow ID
- Backpressure must propagate within 2 clock cycles
- All outputs driven to known state during reset
- Throughput must never drop below N transactions per M cycles

Invariants become assertions in the implementation phase. Stating them here ensures they are designed in, not tested in after the fact.

### Non-Goals

State what is explicitly out of scope:
- This module does not handle error correction (detection only)
- This module does not support partial-width transfers
- This module does not implement power gating

Non-goals prevent scope creep and protect the design budget.

## Step 2: CBB Reuse Assessment

Before designing any sub-module, check the CBB catalog. This is mandatory, not optional.

### Check CBB Catalog Availability

Determine whether a CBB catalog exists in the project. If the user has not built one, prompt them to run the `cbb-asset` skill first to create one. Design decisions without a catalog are blind decisions.

### Search by Functional Need

For each sub-function identified in the design, search the catalog cross-reference table:
- **Storage**: FIFO, register file, dual-port RAM, shift register
- **Arbitration**: round-robin arbiter, priority arbiter, fixed arbiter
- **Datapath**: multiplexer, barrel shifter, priority encoder, parity generator
- **Clock domain**: synchronizer, Gray code converter, handshake CDC, FIFO CDC
- **Protocol**: AXI4-Stream adapter, APB bridge, custom handshake wrapper
- **Control**: FSM template, counter with threshold, timer, watchdog

Match by function first, then filter by parameter range and interface compatibility.

### Evaluate Compatibility

For each candidate CBB, evaluate:
- **Parameter range**: does the CBB support the required depth, width, or ratio? A FIFO CBB that supports DEPTH 2-256 does not help if you need DEPTH=1024.
- **Interface match**: does the CBB use the same handshake protocol? A valid/ready FIFO cannot directly replace a valid-only buffer without glue logic.
- **Feature coverage**: does the CBB provide all needed features (almost-full flag, programmable threshold, error output)? Missing features mean glue logic or wrapper design.
- **PPA characteristics**: what are the known timing path, area, and power characteristics of the CBB? A CBB that fails timing at your target frequency is not a match.

### Reuse Decision Matrix

For each sub-function, make one of three decisions:

| Decision | Condition | Action |
|----------|-----------|--------|
| Exact match | CBB satisfies all functional, parametric, and interface requirements | Instantiate directly with appropriate parameters |
| Partial match | CBB satisfies core function but needs wrapper or glue logic | Wrap with adapter: protocol converter, width adjuster, or feature supplement. Document wrapper complexity. |
| No match | No CBB fits, or wrapping cost exceeds custom design cost | Design custom. Mark the module boundary with a `// CBB REPLACEABLE:` comment so it can be harvested later. |

Document every reuse decision with: CBB name, parameter configuration, connection method, and rationale for the decision.

### Why Reuse Decisions Matter for PPA

A reused CBB carries known PPA characteristics. You can budget timing and area with confidence. A custom module carries unknown PPA until synthesis. Every custom module introduces schedule risk: it may need redesign if timing fails. Minimize custom modules to minimize risk.

## Step 3: Microarchitecture Planning

Transform requirements and reuse decisions into a concrete microarchitecture.

### Module Partitioning

Decompose the design into sub-modules by function:

- **Datapath modules**: data transformation, arithmetic, muxing, shifting. These are the high-throughput paths where pipeline depth matters.
- **Control modules**: FSMs, sequencers, arbiters. These generate control signals for the datapath. Keep control and datapath separate for clarity and timing.
- **Storage modules**: FIFOs, register files, buffers. These are the reuse candidates most likely to match CBBs.
- **Interface modules**: protocol adapters, CDC modules, pad wrappers. These translate between internal and external protocols.

Rule: each sub-module should have a single clear purpose. A module that does arbitration and data transformation will be hard to reuse, hard to verify, and hard to meet timing.

### Data Flow Analysis

Trace data from input to output:
- **Throughput calculation**: at each pipeline stage, compute the sustained transfer rate. If stage 1 produces 1 word per cycle and stage 2 consumes 1 word every 2 cycles, you need a buffer between them.
- **Backpressure propagation**: for every valid/ready interface, trace how stall conditions propagate backward. A broken backpressure chain causes data loss.
- **Buffer depth calculation**: for each inter-stage buffer, compute the required depth from the worst-case burst size and throughput mismatch. Use the formula: `depth = max_burst * (1 - consumer_rate / producer_rate)` for the general case, rounded up to the next power of 2.

### Pipeline Decisions

Choose the pipeline structure based on the clock frequency target and combinational path depth:

| Structure | When to Use | PPA Tradeoff |
|-----------|-------------|--------------|
| Single-cycle combinational | Path delay < 60% of clock period, simple logic | Lowest latency, lowest area, but fails timing on complex paths |
| Multi-cycle (state machine) | Complex operation that cannot complete in one cycle, low throughput requirement | Acceptable latency for control-path operations, but wastes cycles on throughput-critical paths |
| Pipelined (registered stages) | Throughput requirement is 1 transaction per cycle, path delay exceeds single-cycle budget | Maximum throughput at the cost of pipeline registers (area) and latency (initial fill delay) |

Decision rule: pipeline the datapath, multi-cycle the control path. Never pipeline a control FSM unless it runs at a very high rate.

### Clock Domain Partitioning

Identify every clock domain boundary in the design:
- List all clock inputs and their expected frequencies
- Identify every signal that crosses a domain boundary
- For each crossing, choose a CDC strategy:
  - **Single-bit control**: 2-stage synchronizer (standard, low-latency)
  - **Multi-bit data with flow control**: async FIFO or handshake CDC (safe, higher latency)
  - **Multi-bit data with continuous flow**: Gray code counter (efficient, depth must be power of 2)
  - **Pulse across domains**: pulse synchronizer (toggle-based, req/ack)

Document every CDC crossing with: source domain, destination domain, signal width, CDC method, and expected latency.

### Reset Strategy

Define reset behavior for the entire module:
- **Reset type**: synchronous (assert and release on clock edge) vs asynchronous (assert any time, release synchronized to clock). Synchronous reset is safer for synthesis; asynchronous reset is required by some IP.
- **Reset domains**: does the entire module share one reset, or do sub-modules have independent resets? Independent resets require careful sequencing during release.
- **Reset state**: what is the defined state of every output after reset? Unspecified reset states cause integration bugs.

### Risk Assessment and Fallback Plans

For each design decision, state the risk and the fallback:

| Risk | Likelihood | Impact | Fallback |
|------|-----------|--------|----------|
| Custom FIFO fails timing at 500 MHz | Medium | High | Replace with 2-phase pipelined version or reduce frequency requirement |
| CDC handshake latency exceeds budget | Low | Medium | Replace handshake CDC with async FIFO CDC |
| Area exceeds budget by >20% | Medium | Medium | Apply resource sharing on datapath, reduce pipeline depth |

A design without stated risks is a design that will surprise during implementation.

## Step 4: Detailed Design Document

Produce the design document following the template in `references/design_doc_template.md`. The document must contain every section below.

### Module Block Diagram

Draw the module hierarchy as a text block diagram showing:
- Top-level module boundary with all external ports
- Sub-module instances with their CBB or custom designation
- Signal connections between sub-modules
- Clock and reset distribution

Use ASCII art. Every signal in the diagram must appear in the interface table. Every signal in the interface table must appear in the diagram. Discrepancies between the diagram and the port table are integration bugs waiting to happen.

### Interface Definition

For every port on the top-level module and on every internal interface between sub-modules, define:

| Port | Direction | Width | Clock Domain | Description |
|------|-----------|-------|-------------|-------------|
| clk | input | 1 | -- | System clock |
| rst_n | input | 1 | -- | Active-low synchronous reset |
| s_axis_tdata | input | DATA_WIDTH | clk | AXI4-Stream data input |
| s_axis_tvalid | input | 1 | clk | AXI4-Stream valid |
| s_axis_tready | output | 1 | clk | AXI4-Stream ready |
| m_axis_tdata | output | DATA_WIDTH | clk | AXI4-Stream data output |
| m_axis_tvalid | output | 1 | clk | AXI4-Stream valid |
| m_axis_tready | input | 1 | clk | AXI4-Stream ready |

Include clock domain for every port. During integration, clock domain mismatches are the most expensive bugs to fix.

### Parameter Definition

For every parameter:

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| DATA_WIDTH | 32 | [8, 16, 32, 64] | Data bus width in bits |
| DEPTH | 16 | [2, 4, 8, ..., 256] | Buffer depth (power of 2) |
| FULL_THRESHOLD | 14 | [1, DEPTH-1] | Threshold for almost-full flag |

State the valid range for every parameter. A parameter without a range will eventually be instantiated with an invalid value.

### FSM Definition

For every FSM in the design:

| State | Encoding | Description |
|-------|----------|-------------|
| IDLE | 3'b000 | Waiting for valid input |
| RECEIVE | 3'b001 | Receiving packet data |
| PROCESS | 3'b010 | Processing buffered data |
| TRANSMIT | 3'b011 | Transmitting processed data |

Define transitions as a table:

| Current State | Condition | Next State | Outputs |
|---------------|-----------|------------|---------|
| IDLE | s_axis_tvalid == 1 | RECEIVE | Accept data, assert s_axis_tready |
| RECEIVE | pkt_count == pkt_len | PROCESS | Deassert s_axis_tready, start processing |
| RECEIVE | s_axis_tvalid == 0 | RECEIVE | Hold state, wait for data |
| PROCESS | done == 1 | TRANSMIT | Assert m_axis_tvalid with result |
| TRANSMIT | m_axis_tready == 1 and last == 1 | IDLE | Complete transmission, return to idle |
| TRANSMIT | m_axis_tready == 0 | TRANSMIT | Hold valid, wait for consumer |

Every state must have a defined transition for every input condition. Missing transitions are unspecified behavior.

### Timing Description

Describe cycle-by-cycle behavior for the primary operations:

```
Cycle 0: s_axis_tvalid asserts with first data word
Cycle 1: s_axis_tready asserts (1 cycle latency), data written to buffer
Cycle 2: s_axis_tvalid and s_axis_tready both high, data flows through
...
Cycle N: Last word received, s_axis_tready deasserts, processing begins
Cycle N+1: Processed data appears on m_axis_tdata, m_axis_tvalid asserts
Cycle N+2: m_axis_tready asserts, data consumed by downstream
```

Timing descriptions prevent mismatched assumptions between the designer and the integrator.

### CBB Instantiation List

For every CBB used in the design:

| Instance | CBB Name | Parameters | Connection |
|----------|----------|------------|------------|
| u_fifo_rx | fifo_sync_fwft | DATA_WIDTH=32, DEPTH=16 | Input from s_axis, output to processing |
| u_cdc_status | sync_2stage | -- | status_signal from clk_tx domain to clk_rx domain |

Document the exact parameter configuration and how the CBB connects to the surrounding logic. This is the specification for the implementation step.

### Design Constraints and Trade-off Rationale

State every non-obvious design decision and why it was made:
- "Pipeline depth is 3 stages because synthesis at 500 MHz in the target library shows the combinational path exceeds 2ns in a single-stage implementation."
- "FIFO depth is 16 because the upstream burst size is 8 words and the downstream consumption rate is 50%, giving a required depth of 8 / (1 - 0.5) = 16."
- "Control FSM is multi-cycle rather than pipelined because throughput requirement is 1 transaction per 4 cycles, making pipeline registers wasteful."

Rationale documents decisions so they are not revisited without new information. Without rationale, every design review re-litigates decisions that were already made correctly.

## Output Format

Produce a single structured markdown design document with all sections from Step 4. Use the template in `references/design_doc_template.md` as the starting format. Fill in every section completely. Do not leave any section as "TBD" or "to be determined" -- if a section cannot be filled in, state what information is missing and what assumption was made instead.

The design document is the handoff artifact to the implementation phase (rtl-impl skill) and the verification planning phase (rtl-verification skill). It must contain enough detail that both phases can proceed without returning to the requirements spec.

## Self-Check Before Delivery

Before presenting the design document, verify:
1. Every requirement from Step 1 is addressed in the design.
2. Every sub-module has a reuse decision (exact match, partial match, or custom).
3. Every clock domain crossing has a CDC strategy.
4. Every FSM state has a complete set of transitions.
5. Every parameter has a valid range.
6. The timing description covers the primary operation from input to output.
7. PPA trade-offs are stated with rationale, not just conclusions.
