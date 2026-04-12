# CBB Catalog Schema Reference

This document provides concrete examples for every structural element of a CBB summary catalog. Use it as a template and reference when generating catalogs with the `cbb-asset` skill.

---

## 1. Filelist Parsing Examples

### Standard .f Filelist

```
# CBB library filelist
# Generated: 2026-03-15

# FIFOs
./rtl/cbb/fifo/fifo_sync_std.v
./rtl/cbb/fifo/fifo_sync_fwft.v
./rtl/cbb/fifo/fifo_async.v

# CDC synchronizers
./rtl/cbb/cdc/cdc_sync_bit.v
./rtl/cbb/cdc/cdc_sync_bus.v

# Pipeline elements
./rtl/cbb/pipeline/pipeline_reg.v
./rtl/cbb/pipeline/skid_buffer.v

# RAM wrappers
./rtl/cbb/mem/ram_1r1w_wrapper.v
```

### Mixed Absolute and Relative Paths

```
/home/user/project/rtl/cbb/fifo_sync_std.v
./rtl/cbb/fifo_sync_fwft.v
../shared/cdc/cdc_sync_bit.v
+incdir+./rtl/includes
+define+SIMULATION
```

### Parsing Notes

- Lines starting with `#` are comments. Skip them.
- Blank lines are separators. Skip them.
- `+incdir+` lines are include directives. Record them but do not treat as source files.
- `+define+` lines are compile-time defines. Record them but do not treat as source files.
- Relative paths resolve against the directory containing the filelist file.
- Report unresolved paths explicitly. Do not silently skip.

---

## 2. Per-CBB Entry Template with Annotated Fields

```markdown
## module_name

- **Type**: <category from the type classification table>
  <!-- fifo | ram | cdc | pipeline | arithmetic | protocol | clocking | reset | utility | other -->

- **Variant**: <specific functional variant>
  <!-- Must be specific: "FWFT synchronous FIFO", not just "FIFO" -->

- **File**: <path as listed in filelist>
  <!-- Copy from filelist verbatim for traceability -->

- **Clock domains**: <1-clk | 2-clk (clk_a, clk_b) | none>
  <!-- 1-clk for synchronous, 2-clk for cross-domain. Name the clocks. -->

- **Reset**: <sync-high | sync-low | async-high | async-low | none>
  <!-- Infer from port name: rst_n = async-low, rst = sync-high. Confirm from code. -->

### Ports
| Port | Dir | Width | Role |
|---|---|---|---|
| clk | input | 1 | system clock |
| rst_n | input | 1 | asynchronous reset, active low |
| din | input | DATA_W | write data input |
| wr_en | input | 1 | write enable |
| dout | output | DATA_W | read data output |
| rd_en | input | 1 | read enable |
| full | output | 1 | FIFO full flag |
| empty | output | 1 | FIFO empty flag |

### Parameters
| Param | Default | Range | Purpose |
|---|---|---|---|
| DATA_W | 8 | >=1 | data bus width in bits |
| DEPTH | 16 | power of 2 | FIFO depth in entries |

### Use When
- <Scenario 1: specific use case with context>
- <Scenario 2: another distinct use case>

### Avoid When
- <Anti-pattern 1: why it fails and what to use instead>
- <Anti-pattern 2: why it fails and what to use instead>

### Sub-modules
- <name>: <inferred purpose>

### Notes
- <Caveat, assumption, or unusual characteristic>
- <Mark inferences with [inferred from ...]>
```

---

## 3. Example Catalog Entries

### fifo_sync_std

- **Type**: fifo
- **Variant**: Standard synchronous FIFO
- **File**: ./rtl/cbb/fifo/fifo_sync_std.v
- **Clock domains**: 1-clk (clk)
- **Reset**: async-low (rst_n)

#### Ports
| Port | Dir | Width | Role |
|---|---|---|---|
| clk | input | 1 | system clock |
| rst_n | input | 1 | asynchronous reset, active low |
| din | input | DATA_W | write data input |
| wr_en | input | 1 | write enable, qualified by ~full |
| dout | output | DATA_W | read data output, valid when rd_en asserted and ~empty |
| rd_en | input | 1 | read enable, qualified by ~empty |
| full | output | 1 | FIFO is full, deasserts wr_en externally |
| empty | output | 1 | FIFO is empty, deasserts rd_en externally |

#### Parameters
| Param | Default | Range | Purpose |
|---|---|---|---|
| DATA_W | 8 | >=1 | data bus width in bits |
| DEPTH | 16 | power of 2, >=2 | FIFO depth in entries |

#### Use When
- Buffering data between a producer and consumer in the same clock domain
- Smoothing bursty data rates where the average throughput matches but instantaneous rates differ
- Decoupling module timing paths by inserting a FIFO as a pipeline buffer

#### Avoid When
- Consumer requires data on the same cycle as read enable (output is registered) — use fifo_sync_fwft instead
- Crossing clock domains between writer and reader — use fifo_async instead
- Zero-latency read is required (standard FIFO has 1-cycle read latency) — use fifo_sync_fwft instead

#### Sub-modules
- ram_1r1w: internal storage array [inferred from FIFO architecture]
- ptr_bin: binary read/write pointer management

#### Notes
- Read latency: 1 cycle (dout is registered). Assert rd_en, data appears on dout next cycle.
- Write is combinational to storage: assert wr_en, data stored in the same cycle.
- DEPTH must be power of 2 due to binary pointer wrap-around arithmetic.

---

### fifo_sync_fwft

- **Type**: fifo
- **Variant**: FWFT (First-Word Fall-Through) synchronous FIFO
- **File**: ./rtl/cbb/fifo/fifo_sync_fwft.v
- **Clock domains**: 1-clk (clk)
- **Reset**: async-low (rst_n)

#### Ports
| Port | Dir | Width | Role |
|---|---|---|---|
| clk | input | 1 | system clock |
| rst_n | input | 1 | asynchronous reset, active low |
| din | input | DATA_W | write data input |
| wr_en | input | 1 | write enable, qualified by ~full |
| dout | output | DATA_W | read data output, combinational from RAM when ~empty |
| rd_en | input | 1 | read enable / acknowledge consumed data |
| full | output | 1 | FIFO is full |
| empty | output | 1 | FIFO is empty |

#### Parameters
| Param | Default | Range | Purpose |
|---|---|---|---|
| DATA_W | 8 | >=1 | data bus width in bits |
| DEPTH | 16 | power of 2, >=2 | FIFO depth in entries |

#### Use When
- Consumer needs data available immediately (zero read latency) — dout is valid as soon as empty deasserts
- Implementing a lookahead pipeline where the next data word must be pre-fetched
- AXI stream interfaces that expect data to be valid without a read request cycle

#### Avoid When
- Area is tightly constrained — FWFT uses an additional output register or combinational RAM read path
- Crossing clock domains — use fifo_async instead
- Large depth where the combinational read path creates timing pressure — standard FIFO may close timing more easily

#### Sub-modules
- ram_1r1w: internal storage array [inferred from FIFO architecture]
- ptr_bin: binary read/write pointer management

#### Notes
- Zero read latency: dout is valid immediately when empty is low, without asserting rd_en.
- Asserting rd_en acknowledges the current data and advances to the next word.
- Slightly higher area than fifo_sync_std due to the FWFT output logic.

---

### fifo_async

- **Type**: fifo
- **Variant**: Asynchronous FIFO (dual-clock domain)
- **File**: ./rtl/cbb/fifo/fifo_async.v
- **Clock domains**: 2-clk (wr_clk, rd_clk)
- **Reset**: async-low (rst_n)

#### Ports
| Port | Dir | Width | Role |
|---|---|---|---|
| wr_clk | input | 1 | write domain clock |
| rd_clk | input | 1 | read domain clock |
| rst_n | input | 1 | asynchronous reset, active low |
| din | input | DATA_W | write data input |
| wr_en | input | 1 | write enable, qualified by ~full |
| dout | output | DATA_W | read data output |
| rd_en | input | 1 | read enable, qualified by ~empty |
| full | output | 1 | FIFO full flag (synchronous to wr_clk) |
| empty | output | 1 | FIFO empty flag (synchronous to rd_clk) |

#### Parameters
| Param | Default | Range | Purpose |
|---|---|---|---|
| DATA_W | 8 | >=1 | data bus width in bits |
| DEPTH | 16 | power of 2, >=4 | FIFO depth in entries (minimum 4 for Gray-code safety) |
| SYNC_STAGES | 2 | >=2 | number of synchronization stages for pointer CDC |

#### Use When
- Transferring data between two unrelated clock domains where both clocks are free-running
- Clock domain crossing for bursty data streams where throughput matters
- Replacing ad hoc double-flop CDC on multi-bit buses — the Gray-code pointer avoids data corruption

#### Avoid When
- Both sides run on the same clock — use fifo_sync_std instead (avoids Gray-code overhead and synchronization latency)
- Transferring single control signals or pulses — use cdc_sync_bit or cdc_sync_pulse instead (lower latency, lower area)
- One clock is gated or intermittent — the async FIFO assumes both clocks are continuously active for pointer synchronization

#### Sub-modules
- ram_1r1w: internal dual-port storage array
- ptr_gray_wr: Gray-code write pointer with binary-to-Gray conversion
- ptr_gray_rd: Gray-code read pointer with Gray-to-binary conversion
- sync_2ff: 2-stage synchronizer for crossing Gray-coded pointers between domains

#### Notes
- Full/empty detection uses Gray-code pointer comparison, adding 2-3 cycles of synchronization latency on status flags.
- DEPTH minimum is 4 (not 2) to ensure Gray-code pointers have sufficient distance for safe CDC.
- Write-side signals (full, wr_en, din) are synchronous to wr_clk. Read-side signals (empty, rd_en, dout) are synchronous to rd_clk.

---

### cdc_sync_bit

- **Type**: cdc
- **Variant**: Single-bit level synchronizer
- **File**: ./rtl/cbb/cdc/cdc_sync_bit.v
- **Clock domains**: 2-clk (src_clk, dst_clk)
- **Reset**: sync-high (rst)

#### Ports
| Port | Dir | Width | Role |
|---|---|---|---|
| src_clk | input | 1 | source domain clock |
| dst_clk | input | 1 | destination domain clock |
| rst | input | 1 | synchronous reset, active high |
| din | input | 1 | single-bit input from source domain |
| dout | output | 1 | synchronized output in destination domain |

#### Parameters
| Param | Default | Range | Purpose |
|---|---|---|---|
| SYNC_STAGES | 2 | >=2 | number of flip-flop stages for metastability settling |

#### Use When
- Synchronizing a single-bit level signal across clock domains (e.g., enable, mode select, static config)
- The input signal is a slow-changing level (not a pulse) that remains stable for multiple destination clock cycles
- Synchronizing a control bit where occasional latency is acceptable but correctness is critical

#### Avoid When
- Synchronizing a multi-bit bus — each bit arrives independently, producing invalid intermediate values. Use cdc_sync_bus instead.
- Synchronizing a short pulse that may be missed by the destination clock — use a pulse synchronizer instead.
- The signal changes faster than the destination clock can sample it — no amount of synchronization helps if transitions are missed.

#### Sub-modules
- (none — implemented as a shift register of SYNC_STAGES flip-flops)

#### Notes
- Introduces SYNC_STAGES cycles of latency in the destination domain.
- Only safe for single-bit level signals. Multi-bit buses require Gray coding or handshake protocols.
- MTBF depends on SYNC_STAGES and clock frequency ratios. More stages improve MTBF at the cost of latency.

---

### cdc_sync_bus

- **Type**: cdc
- **Variant**: Multi-bit bus synchronizer (Gray-code encoded)
- **File**: ./rtl/cbb/cdc/cdc_sync_bus.v
- **Clock domains**: 2-clk (src_clk, dst_clk)
- **Reset**: sync-high (rst)

#### Ports
| Port | Dir | Width | Role |
|---|---|---|---|
| src_clk | input | 1 | source domain clock |
| dst_clk | input | 1 | destination domain clock |
| rst | input | 1 | synchronous reset, active high |
| din | input | DATA_W | multi-bit bus input from source domain |
| dout | output | DATA_W | synchronized bus output in destination domain |
| valid | output | 1 | data has been captured and is stable in destination domain |

#### Parameters
| Param | Default | Range | Purpose |
|---|---|---|---|
| DATA_W | 8 | >=2 | bus width in bits |
| SYNC_STAGES | 2 | >=2 | number of synchronization stages |

#### Use When
- Transferring a multi-bit data bus across clock domains where only one bit changes per transfer (e.g., counters, pointers)
- Synchronizing Gray-coded pointers or counters between domains
- The source updates the bus at a rate slow enough that only adjacent Gray values are produced

#### Avoid When
- The bus changes multiple bits simultaneously (arbitrary data) — use a handshake-based bus synchronizer instead
- High-throughput streaming is required — an async FIFO provides buffering and higher throughput
- Single-bit signal — use cdc_sync_bit instead (lower area, no Gray encoding needed)

#### Sub-modules
- gray_encode: binary to Gray-code conversion in source domain [inferred from CDC architecture]
- sync_2ff: multi-bit synchronizer (SYNC_STAGES deep, DATA_W wide)
- gray_decode: Gray-code to binary conversion in destination domain

#### Notes
- Assumes source data changes by only one bit per update (Gray-code property). Arbitrary multi-bit changes will corrupt the synchronized output.
- The `valid` flag indicates when new data has been captured. It is synchronous to dst_clk.
- Latency: SYNC_STAGES cycles in destination domain, plus encode/decode combinational delay.

---

### pipeline_reg

- **Type**: pipeline
- **Variant**: Simple pipeline register stage
- **File**: ./rtl/cbb/pipeline/pipeline_reg.v
- **Clock domains**: 1-clk (clk)
- **Reset**: none

#### Ports
| Port | Dir | Width | Role |
|---|---|---|---|
| clk | input | 1 | system clock |
| din | input | DATA_W | data input |
| dout | output | DATA_W | registered data output |
| en | input | 1 | clock enable (hold when low) |

#### Parameters
| Param | Default | Range | Purpose |
|---|---|---|---|
| DATA_W | 8 | >=1 | data width in bits |

#### Use When
- Breaking a long combinational path by inserting a pipeline register
- Timing closure requires retiming but the synthesizer cannot move registers across module boundaries
- Simple data delay where no backpressure handling is needed

#### Avoid When
- Backpressure from downstream must be handled — use skid_buffer instead
- Multi-stage pipeline with depth > 1 is needed — instantiate multiple pipeline_reg stages or use elastic_buffer
- Data must pass through combinationally when enabled — pipeline_reg always adds 1 cycle of latency

#### Sub-modules
- (none — single register stage)

#### Notes
- When `en` is low, dout holds its previous value. No reset — dout is undefined until the first valid write.
- Minimal area: one register bank of DATA_W flip-flops with clock enable.

---

### skid_buffer

- **Type**: pipeline
- **Variant**: Skid buffer (1-entry pipeline with backpressure)
- **File**: ./rtl/cbb/pipeline/skid_buffer.v
- **Clock domains**: 1-clk (clk)
- **Reset**: async-low (rst_n)

#### Ports
| Port | Dir | Width | Role |
|---|---|---|---|
| clk | input | 1 | system clock |
| rst_n | input | 1 | asynchronous reset, active low |
| din | input | DATA_W | data input from upstream |
| valid_in | input | 1 | upstream data is valid |
| ready_out | output | 1 | skid buffer can accept new data (backpressure to upstream) |
| dout | output | DATA_W | data output to downstream |
| valid_out | output | 1 | data is available for downstream |
| ready_in | input | 1 | downstream can accept data |

#### Parameters
| Param | Default | Range | Purpose |
|---|---|---|---|
| DATA_W | 8 | >=1 | data width in bits |

#### Use When
- Inserting a pipeline stage between two modules that use valid-ready handshaking
- Downstream may deassert ready unexpectedly and upstream data must be held (skidded) for one cycle
- Breaking a combinational handshake path while preserving backpressure propagation

#### Avoid When
- No backpressure exists in the design — use pipeline_reg instead (simpler, lower area)
- Buffering depth > 1 is needed to absorb burst differences — use an elastic buffer or FIFO instead
- Crossing clock domains — this is a single-clock module

#### Sub-modules
- (none — skid register with handshake control logic)

#### Notes
- Adds exactly 1 cycle of latency when downstream accepts immediately. Latency does not increase under backpressure — the skid register absorbs exactly one stalled transfer.
- Ready_out deasserts for exactly one cycle when skidding data (downstream rejects while new data arrives).
- The smallest valid-ready pipeline element. Use this before escalating to a FIFO.

---

### ram_1r1w_wrapper

- **Type**: ram
- **Variant**: Simple dual-port RAM (1 read port, 1 write port)
- **File**: ./rtl/cbb/mem/ram_1r1w_wrapper.v
- **Clock domains**: 1-clk (clk)
- **Reset**: none

#### Ports
| Port | Dir | Width | Role |
|---|---|---|---|
| clk | input | 1 | system clock |
| wr_addr | input | ADDR_W | write address |
| wr_en | input | 1 | write enable |
| wr_data | input | DATA_W | write data |
| rd_addr | input | ADDR_W | read address |
| rd_en | input | 1 | read enable |
| rd_data | output | DATA_W | read data output (1-cycle latency) |

#### Parameters
| Param | Default | Range | Purpose |
|---|---|---|---|
| DATA_W | 8 | >=1 | data width in bits |
| DEPTH | 256 | power of 2 | memory depth in entries |
| ADDR_W | 8 | log2(DEPTH) | address width (derived from DEPTH if not explicit) |

#### Use When
- Need a simple dual-port memory with independent read and write addresses
- FIFO internals that require separate read/write pointer addressing
- Register file with one read port and one write port

#### Avoid When
- Simultaneous read and write to the same address with read-during-write behavior requirements — check the wrapper's behavior (usually returns old data or don't-care)
- True dual-port access (two independent read-write ports) is needed — use a true dual-port RAM wrapper
- Read-only data (initialized once) — use a ROM instead
- Very shallow storage (< 16 entries) — a register array may be more efficient and provides deterministic timing

#### Sub-modules
- (vendor-specific RAM primitive or inferred register array, selected by parameter or define)

#### Notes
- Read latency: 1 cycle (registered output). Assert rd_en with rd_addr, data appears on rd_data next cycle.
- DEPTH must be power of 2 for address wrap-around.
- No collision handling for simultaneous read/write to the same address. Avoid this case or confirm the wrapper's behavior.

---

## 4. Sub-Module Detection Patterns

### How to Identify Sub-Modules from Module Code

When reading a top-level CBB module, sub-modules appear as instantiated modules in the module body. Detect them by these patterns:

#### Pattern 1: Named Instantiation with Parameters

```verilog
ram_1r1w #(
    .DATA_W (DATA_W),
    .DEPTH  (DEPTH)
) u_ram (
    .clk     (clk),
    .wr_addr (wr_ptr),
    ...
);
```

Sub-module: `ram_1r1w`, instance: `u_ram`. Record name `ram_1r1w` and infer purpose from the instance name and connected signals.

#### Pattern 2: Instantiation Without Parameters

```verilog
sync_2ff u_sync_wr_to_rd (
    .clk  (rd_clk),
    .din  (gray_wr_ptr),
    .dout (gray_wr_ptr_synced)
);
```

Sub-module: `sync_2ff`, instance: `u_sync_wr_to_rd`. Purpose inferred from signal names: synchronizing write pointer to read domain.

#### Pattern 3: Multiple Instances of the Same Sub-Module

```verilog
sync_2ff #(.WIDTH(1)) u_sync_full  (.din(gray_full_flag),  ...);
sync_2ff #(.WIDTH(1)) u_sync_empty (.din(gray_empty_flag), ...);
```

Two instances of `sync_2ff` with different parameters. Record one entry: `sync_2ff: used for pointer synchronization (2 instances)`.

#### Pattern 4: Generate-Block Instantiations

```verilog
genvar i;
generate
    for (i = 0; i < DATA_W; i = i + 1) begin : gen_sync
        sync_bit u_sync_bit (
            .din  (din[i]),
            .dout (dout[i])
        );
    end
endgenerate
```

Sub-module: `sync_bit`, replicated DATA_W times. Record: `sync_bit: per-bit synchronizer (DATA_W instances via generate)`.

### Sub-Module Purpose Inference Table

| Sub-Module Name Pattern | Likely Purpose |
|---|---|
| `ram_*`, `mem_*` | Internal storage array |
| `ptr_*`, `cntr_*` | Pointer or counter management |
| `gray_*` | Gray-code encode/decode for CDC |
| `sync_*`, `cdc_*` | Clock domain crossing synchronization |
| `enc_*`, `dec_*` | Encoding or decoding logic |
| `arb_*`, `mux_*` | Arbitration or multiplexing |
| `reg_*`, `ff_*` | Register or flip-flop stage |
| `cmp_*`, `eq_*` | Comparison or equality check |

---

## 5. Cross-Reference Table Format

The cross-reference table is the primary query surface for downstream skills. It maps a design need directly to a CBB and variant.

### Format

```markdown
## Quick Reference: Need -> CBB

| Need | CBB | Variant | Key Params |
|---|---|---|---|
| <what the designer needs> | <module_name> | <specific variant> | <critical parameters> |
```

### Example Table

```markdown
## Quick Reference: Need -> CBB

| Need | CBB | Variant | Key Params |
|---|---|---|---|
| Standard data buffering, same clock domain | fifo_sync_std | Standard sync FIFO | DATA_W, DEPTH |
| Zero-latency read, same clock domain | fifo_sync_fwft | FWFT sync FIFO | DATA_W, DEPTH |
| Data transfer across clock domains | fifo_async | Async FIFO | DATA_W, DEPTH, SYNC_STAGES |
| Single-bit level signal CDC | cdc_sync_bit | Bit synchronizer | SYNC_STAGES |
| Multi-bit bus CDC (Gray-compatible) | cdc_sync_bus | Bus synchronizer (Gray) | DATA_W, SYNC_STAGES |
| Pipeline break, no backpressure | pipeline_reg | Register stage | DATA_W |
| Pipeline break with valid-ready backpressure | skid_buffer | Skid buffer | DATA_W |
| Multi-entry pipeline with backpressure | elastic_buffer | Elastic buffer | DATA_W, DEPTH |
| Simple dual-port memory | ram_1r1w_wrapper | Simple dual-port RAM | DATA_W, DEPTH, ADDR_W |
| True dual-port memory (2R2W) | ram_2r2w_wrapper | True dual-port RAM | DATA_W, DEPTH, ADDR_W |
| Read-only initialized memory | rom_wrapper | ROM | DATA_W, DEPTH, INIT_FILE |
| Single-bit pulse CDC | cdc_sync_pulse | Pulse synchronizer | SYNC_STAGES |
```

### Usage by Downstream Skills

When `rtl-design` or `rtl-impl` needs a CBB, it scans this table by the "Need" column. The match is functional and specific:

1. Designer needs "buffering with zero-latency read" -> matches `fifo_sync_fwft`, not `fifo_sync_std`
2. Designer needs "cross-domain data transfer" -> matches `fifo_async`, not `fifo_sync_std`
3. Designer needs "pipeline stage between handshaking modules" -> matches `skid_buffer`, not `pipeline_reg`

The table must be specific enough that there is only one correct CBB for each distinct need. If two CBBs could serve the same need, the use-when/avoid-when guidance in the full entries must disambiguate them.
