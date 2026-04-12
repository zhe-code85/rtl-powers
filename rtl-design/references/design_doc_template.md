# RTL Design Document Template

This file serves dual purpose: (1) a blank template with annotations explaining what to fill in, and (2) a complete example showing a filled-in design document for an AXI Packet Buffer module.

---

## Part 1: Annotated Template

Use this structure for every design document. Fill in every section. If a section does not apply, state "Not applicable because [reason]" rather than omitting it.

```markdown
# Design Document: [Module Name]

## 1. Requirements Summary

### 1.1 Functional Requirements
<!-- List every functional requirement extracted from the spec.
     For each: what the module does, interface protocol, data format,
     throughput, latency, and operating modes. -->

### 1.2 Non-Functional Constraints
<!-- Clock frequency target, area budget (LUT/FF or gate count),
     power budget, timing margin requirement. These directly
     constrain pipeline depth and resource usage. -->

### 1.3 Invariants
<!-- Constraints that must hold under all conditions.
     These become assertions during implementation. -->

### 1.4 Non-Goals
<!-- What is explicitly out of scope. Prevents scope creep. -->

## 2. CBB Reuse Assessment

### 2.1 Catalog Status
<!-- State whether a CBB catalog exists. If not, note that
     the assessment is based on known reusable blocks only. -->

### 2.2 Reuse Decisions
<!-- Table: Sub-function | CBB Match | Decision | Rationale
     Decision is one of: Exact, Partial, Custom -->

### 2.3 Custom Module Boundaries
<!-- For each custom module, state the interface boundary and
     mark it as replaceable for future CBB harvesting. -->

## 3. Microarchitecture

### 3.1 Module Block Diagram
<!-- ASCII art showing module hierarchy and signal flow.
     Every signal here must appear in the interface table. -->

### 3.2 Interface Definition
<!-- Port table: Port | Direction | Width | Clock Domain | Description
     Include every top-level port and every inter-module interface. -->

### 3.3 Parameter Definition
<!-- Parameter table: Parameter | Default | Range | Description
     Every parameter must have a defined valid range. -->

### 3.4 FSM Definition
<!-- State table and transition table for every FSM.
     Every state must have transitions for all input conditions. -->

### 3.5 Timing Description
<!-- Cycle-by-cycle behavior for primary operations.
     Shows when data enters, is processed, and exits. -->

### 3.6 Clock Domain Partitioning
<!-- List every clock domain and every CDC crossing.
     For each crossing: method, width, expected latency. -->

### 3.7 Reset Strategy
<!-- Reset type, domains, and defined reset state for every output. -->

## 4. CBB Instantiation List
<!-- Table: Instance | CBB Name | Parameters | Connection
     Exact configuration for every reused CBB. -->

## 5. Design Constraints and Trade-offs
<!-- State every non-obvious decision with rationale.
     PPA trade-offs must include the numbers behind the decision. -->

## 6. Risk Assessment
<!-- Table: Risk | Likelihood | Impact | Fallback
     Every design carries risk. State it explicitly. -->
```

---

## Part 2: Example Design Document -- AXI Packet Buffer

# Design Document: AXI Packet Buffer (axi_packet_buffer)

## 1. Requirements Summary

### 1.1 Functional Requirements

- Accept variable-length packets via AXI4-Stream slave interface (s_axis)
- Buffer up to `DEPTH` words of `DATA_WIDTH` bits each
- Process buffered data through a configurable threshold check on each word
- Transmit processed packets via AXI4-Stream master interface (m_axis)
- Support backpressure on both input and output sides independently
- Indicate almost-full status when buffer occupancy reaches `FULL_THRESHOLD`
- Packet length is indicated by `tlast` signal on the AXI4-Stream interface
- Throughput: 1 word per clock cycle sustained when backpressure is absent
- Latency: 2 clock cycles from s_axis input acceptance to m_axis output availability in bypass mode

### 1.2 Non-Functional Constraints

- Clock frequency: 250 MHz target in 28nm ASIC library (4 ns period)
- Area budget: under 5000 gate equivalents for default configuration (DATA_WIDTH=32, DEPTH=16)
- Power: no specific budget, but minimize toggle activity on idle paths
- Timing margin: target 20% slack (800 ps margin on critical path)

### 1.3 Invariants

- No packet data is ever lost: backpressure is always honored
- Packet integrity is preserved: words of a single packet are never interleaved with words from another packet
- Output data order matches input data order (no reordering)
- After reset, all outputs are driven to known idle state: tvalid=0, tdata=0, tlast=0, almost_full=0

### 1.4 Non-Goals

- This module does not perform error correction or detection on packet data
- This module does not support packet segmentation or reassembly
- This module does not implement clock domain crossing (single clock domain only)

## 2. CBB Reuse Assessment

### 2.1 Catalog Status

CBB catalog available. Assessment based on full catalog search.

### 2.2 Reuse Decisions

| Sub-function | CBB Match | Decision | Rationale |
|-------------|-----------|----------|-----------|
| Input buffer (packet storage) | fifo_sync_fwft | Exact | Synchronous FWFT FIFO supports required DATA_WIDTH, DEPTH, and provides programmable threshold for almost-full |
| Threshold comparator | -- | Custom | Simple combinational comparison, too small to warrant a CBB. Marked replaceable. |
| Output mux (bypass vs buffered) | -- | Custom | 2:1 mux controlled by bypass flag. Trivial logic, not CBB-worthy. Marked replaceable. |

### 2.3 Custom Module Boundaries

- `threshold_cmp`: inputs are `fifo_data[DATA_WIDTH-1:0]` and `threshold_val[DATA_WIDTH-1:0]`, output is `above_threshold`. Replaceable if a parameterizable comparator CBB becomes available.
- `out_mux`: inputs are `bypass_data[DATA_WIDTH-1:0]` and `fifo_data[DATA_WIDTH-1:0]` with `sel`, output is `m_axis_tdata[DATA_WIDTH-1:0]`. Replaceable with standard mux CBB.

## 3. Microarchitecture

### 3.1 Module Block Diagram

```
                    +--------------------------------------------------+
                    |              axi_packet_buffer                   |
                    |                                                  |
  s_axis_tdata  --->+-> [u_fifo_rx: fifo_sync_fwft] --+--> [out_mux] -+---> m_axis_tdata
  s_axis_tvalid --->+    (FWFT mode)                  |    (bypass/   |---> m_axis_tvalid
  s_axis_tready <---+                                |     fifo)     |---> m_axis_tlast
                    |                                |               |
                    |   [threshold_cmp] <------------+               |
                    |      |                                         |
                    |      v                                         |
                    |   almost_full_o ---> (top-level output)        |
                    |                                                  |
  bypass_sel   ---->+---> (out_mux select)                            |
  clk          ---->+---> (all registers)                             |
  rst_n        ---->+---> (all registers)                             |
                    |                                                  |
                    +--------------------------------------------------+
```

### 3.2 Interface Definition

| Port | Direction | Width | Clock Domain | Description |
|------|-----------|-------|-------------|-------------|
| clk | input | 1 | -- | System clock, 250 MHz |
| rst_n | input | 1 | -- | Active-low synchronous reset |
| s_axis_tdata | input | DATA_WIDTH | clk | AXI4-Stream slave data input |
| s_axis_tvalid | input | 1 | clk | AXI4-Stream slave valid |
| s_axis_tready | output | 1 | clk | AXI4-Stream slave ready |
| s_axis_tlast | input | 1 | clk | AXI4-Stream slave last word of packet |
| m_axis_tdata | output | DATA_WIDTH | clk | AXI4-Stream master data output |
| m_axis_tvalid | output | 1 | clk | AXI4-Stream master valid |
| m_axis_tready | input | 1 | clk | AXI4-Stream master ready |
| m_axis_tlast | output | 1 | clk | AXI4-Stream master last word of packet |
| bypass_sel | input | 1 | clk | 1=bypass mode (input directly to output), 0=buffered mode |
| almost_full_o | output | 1 | clk | Buffer occupancy reached FULL_THRESHOLD |

### 3.3 Parameter Definition

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| DATA_WIDTH | 32 | [8, 16, 32, 64, 128] | Width of data bus in bits. Must be a power-of-2 multiple of 8. |
| DEPTH | 16 | [2, 4, 8, 16, 32, 64, 128, 256] | FIFO buffer depth. Must be a power of 2. |
| FULL_THRESHOLD | 14 | [1, DEPTH-1] | Occupancy level at which almost_full_o asserts. Must be less than DEPTH. |

### 3.4 FSM Definition

#### Receive FSM (rx_fsm)

| State | Encoding | Description |
|-------|----------|-------------|
| IDLE | 2'b00 | Waiting for s_axis_tvalid. No data being buffered. |
| RECEIVE | 2'b01 | Actively receiving packet words into FIFO. |
| STALL | 2'b10 | Packet partially received but downstream backpressure active. |

| Current State | Condition | Next State | Outputs |
|---------------|-----------|------------|---------|
| IDLE | s_axis_tvalid == 1 && !fifo_full | RECEIVE | fifo_wr_en=1, s_axis_tready=1 |
| IDLE | s_axis_tvalid == 0 | IDLE | fifo_wr_en=0, s_axis_tready=1 |
| IDLE | s_axis_tvalid == 1 && fifo_full | IDLE | fifo_wr_en=0, s_axis_tready=0 (backpressure) |
| RECEIVE | s_axis_tvalid == 1 && s_axis_tlast == 0 && !fifo_full | RECEIVE | fifo_wr_en=1, s_axis_tready=1 |
| RECEIVE | s_axis_tvalid == 1 && s_axis_tlast == 1 | IDLE | fifo_wr_en=1, s_axis_tready=1, pkt_end=1 |
| RECEIVE | s_axis_tvalid == 0 | RECEIVE | fifo_wr_en=0, s_axis_tready=1 |
| RECEIVE | fifo_full == 1 && s_axis_tvalid == 1 | STALL | fifo_wr_en=0, s_axis_tready=0 |
| STALL | !fifo_full && s_axis_tvalid == 1 | RECEIVE | fifo_wr_en=1, s_axis_tready=1 |
| STALL | !fifo_full && s_axis_tvalid == 0 | IDLE | fifo_wr_en=0, s_axis_tready=1 |

#### Transmit FSM (tx_fsm)

| State | Encoding | Description |
|-------|----------|-------------|
| WAIT | 2'b00 | No complete packet in FIFO. m_axis_tvalid deasserted. |
| SEND | 2'b01 | Transmitting packet words from FIFO to m_axis. |
| HOLD | 2'b10 | Data valid but m_axis_tready deasserted (backpressure from consumer). |

| Current State | Condition | Next State | Outputs |
|---------------|-----------|------------|---------|
| WAIT | pkt_ready == 1 && bypass_sel == 0 | SEND | fifo_rd_en=1, m_axis_tvalid=1 |
| WAIT | pkt_ready == 0 | WAIT | fifo_rd_en=0, m_axis_tvalid=0 |
| SEND | m_axis_tready == 1 && !fifo_tlast | SEND | fifo_rd_en=1, m_axis_tvalid=1 |
| SEND | m_axis_tready == 1 && fifo_tlast | WAIT | fifo_rd_en=1, m_axis_tvalid=1 (last word) |
| SEND | m_axis_tready == 0 | HOLD | fifo_rd_en=0, m_axis_tvalid=1 (hold data) |
| HOLD | m_axis_tready == 1 && !fifo_tlast | SEND | fifo_rd_en=1, m_axis_tvalid=1 |
| HOLD | m_axis_tready == 1 && fifo_tlast | WAIT | fifo_rd_en=1, m_axis_tvalid=1 (last word) |

### 3.5 Timing Description

#### Packet Receive Flow (normal operation, no backpressure)

```
Cycle 0: s_axis_tvalid=1, s_axis_tdata=word0, s_axis_tlast=0
         -> rx_fsm: IDLE -> RECEIVE, fifo_wr_en=1, s_axis_tready=1

Cycle 1: s_axis_tvalid=1, s_axis_tdata=word1, s_axis_tlast=0
         -> rx_fsm: RECEIVE, fifo_wr_en=1, word0 available on FWFT output

Cycle 2: s_axis_tvalid=1, s_axis_tdata=word2, s_axis_tlast=0
         -> rx_fsm: RECEIVE, fifo_wr_en=1, word1 on FWFT output

Cycle N: s_axis_tvalid=1, s_axis_tdata=wordN, s_axis_tlast=1
         -> rx_fsm: RECEIVE -> IDLE, fifo_wr_en=1, pkt_end pulse

Cycle N+1: Packet complete in FIFO, tx_fsm: WAIT -> SEND
           m_axis_tvalid=1, m_axis_tdata=word0, m_axis_tlast=0
```

#### Bypass Mode Timing

```
Cycle 0: s_axis_tvalid=1, s_axis_tdata=word0, bypass_sel=1
         -> m_axis_tdata=word0 (combinational path through out_mux)
         -> m_axis_tvalid=1, s_axis_tready=m_axis_tready (pass-through)

Cycle 1: Consumer asserts m_axis_tready=1
         -> Transfer complete, next word accepted
```

### 3.6 Clock Domain Partitioning

Single clock domain (clk). No CDC crossings in this module.

### 3.7 Reset Strategy

- **Reset type**: Active-low synchronous reset (rst_n). All registers reset on the rising edge of clk when rst_n is low.
- **Reset domains**: Single reset domain. Entire module shares rst_n.
- **Reset state**:
  - rx_fsm: IDLE
  - tx_fsm: WAIT
  - s_axis_tready: 1 (ready to accept data immediately after reset)
  - m_axis_tvalid: 0
  - m_axis_tdata: 0
  - m_axis_tlast: 0
  - almost_full_o: 0

## 4. CBB Instantiation List

| Instance | CBB Name | Parameters | Connection |
|----------|----------|------------|------------|
| u_fifo_rx | fifo_sync_fwft | DATA_WIDTH=32, DEPTH=16, THRESHOLD=14 | wr_en from rx_fsm, rd_en from tx_fsm, data_in from s_axis_tdata, data_out to out_mux and threshold_cmp |

## 5. Design Constraints and Trade-offs

- **FWFT FIFO chosen over standard FIFO**: First-Word Fall-Through eliminates 1 cycle of read latency. The data appears on the FIFO output in the same cycle it is written, which is required to meet the 2-cycle bypass latency specification. Tradeoff: FWFT FIFOs use slightly more area (output register bypass logic) than standard FIFOs.

- **Depth 16 with FULL_THRESHOLD=14**: The upstream source is expected to issue bursts of up to 8 words. With a 2-word safety margin for pipeline latency between almost-full assertion and upstream stall response, FULL_THRESHOLD=14 provides 2 words of headroom after the threshold is crossed. Depth 16 accommodates the burst plus the margin.

- **Separate rx_fsm and tx_fsm rather than a single FSM**: Dual FSM allows concurrent receive and transmit, achieving full-duplex throughput (1 word/cycle in, 1 word/cycle out simultaneously). A single FSM would serialize these operations and halve throughput.

- **Bypass mode uses combinational mux path**: When bypass_sel=1, data flows from s_axis_tdata through the 2:1 mux to m_axis_tdata combinationally. This saves FIFO read/write latency but adds combinational path delay. At 250 MHz (4 ns period) with the target 28nm library, the mux path delay is negligible (< 100 ps).

## 6. Risk Assessment

| Risk | Likelihood | Impact | Fallback |
|------|-----------|--------|----------|
| FWFT FIFO critical path exceeds 800 ps margin at 250 MHz | Low | High | Switch to standard FIFO with 1 extra cycle latency. Update spec to reflect 3-cycle latency. |
| Area exceeds 5000 gate estimate for DATA_WIDTH=128 configuration | Medium | Medium | For wide configurations only, accept higher area. Document the scaling factor. |
| Backpressure latency from almost_full_o assertion to s_axis_tready deassert exceeds 2 cycles | Low | High | Add pipeline register on almost_full path and increase FULL_THRESHOLD to compensate for the additional latency. |
| fifo_sync_fwft CBB does not support DEPTH=2 corner case correctly | Medium | Medium | Verify with directed test at DEPTH=2 during verification phase. If broken, instantiate standard FIFO for DEPTH=2 with wrapper. |

---

## Part 3: Standard Formats Reference

### Interface Table Format

```markdown
| Port | Direction | Width | Clock Domain | Description |
|------|-----------|-------|-------------|-------------|
| name | input/output | N or [M:0] | clk_name | Functional description |
```

Rules:
- Direction is always from the perspective of this module (input = comes from outside, output = goes to outside).
- Width is specified as an integer for single-bit signals or `[M:0]` notation for buses. Parameterized widths use the parameter name.
- Clock domain identifies which clock edge the signal is synchronous to. Use "--" for asynchronous signals.
- Every clock and reset port must be listed.

### Parameter Table Format

```markdown
| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| NAME | value | [min, max] or enumeration | What it controls and valid values |
```

Rules:
- Range must be explicit. Use `[min, max]` for continuous ranges or `{val1, val2, val3}` for discrete values.
- State whether power-of-2 restriction applies.
- Note any inter-parameter dependencies (e.g., "FULL_THRESHOLD must be less than DEPTH").

### FSM State Table Format

```markdown
| State | Encoding | Description |
|-------|----------|-------------|
| NAME | binary | Functional behavior in this state |
```

### FSM Transition Table Format

```markdown
| Current State | Condition | Next State | Outputs |
|---------------|-----------|------------|---------|
| STATE_A | guard expression | STATE_B | output assignments |
```

Rules:
- Conditions must be mutually exclusive for each current state, or priority must be stated.
- Include a "default" row for each state if conditions do not cover all cases.
- Outputs column lists every output that changes in this transition. Outputs not listed hold their previous value.

### Timing Description Format

```markdown
Cycle N: [input stimulus]
         -> [FSM transition], [internal signal changes]
Cycle N+1: [resulting output]
```

Rules:
- Number cycles starting from 0 (first active clock edge).
- Show both input stimulus and resulting behavior.
- Cover the primary operation from start to finish.
- Separate timing descriptions for different operating modes if they differ significantly.
