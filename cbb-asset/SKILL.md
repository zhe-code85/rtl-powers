---
name: cbb-asset
description: Parse CBB filelists and produce structured asset catalogs for RTL reuse. Use when user provides a CBB filelist, asks to document reusable modules, build an asset catalog, or mentions CBB/IP library. Also use when rtl-design or rtl-impl need to know what CBBs are available — if no catalog exists, prompt user to run this skill first. Triggers on any mention of CBB, IP library, reusable assets, or module inventory.
---

# cbb-asset: CBB Filelist to Structured Reuse Catalog

## Why CBB Asset Documentation Matters

An LLM cannot reuse what it cannot discover. In RTL projects, reusable Common Building Blocks (CBBs) sit in directories alongside glue logic, testbench helpers, and half-finished modules. Without a structured catalog, the LLM will re-implement standard structures from scratch — FIFOs, CDC synchronizers, RAM wrappers — because it cannot distinguish a proven CBB from an ad hoc module, and it cannot tell a standard FIFO from a FWFT FIFO from an async FIFO.

This skill solves the discovery problem. It takes a curated CBB filelist (not a repo scan), reads only the top-level modules, and produces a structured markdown catalog optimized for LLM query. Downstream skills (`rtl-design`, `rtl-impl`) use this catalog to make reuse decisions during design and implementation.

The reuse-first principle depends on this catalog existing. Without it, every downstream skill is blind.

---

## Input: CBB Filelist

The user provides a filelist — a text file listing Verilog source paths, one per line. This is the standard `.f` (filelist) format used by all major simulators and synthesis tools.

### Filelist Format

```
# Lines starting with # are comments
# One path per line
/path/to/project/rtl/cbb/fifo_sync_std.v
/path/to/project/rtl/cbb/fifo_sync_fwft.v
./rtl/cbb/fifo_async.v
+incdir+/path/to/includes
```

### Parsing Rules

1. **Read the filelist file.** Accept absolute paths, relative paths (relative to the filelist location or project root), and `+incdir+` directives. Skip comment lines (starting with `#`) and blank lines. Skip `+define+` and other plus-arg directives — they are compile-time flags, not source files.

2. **Resolve paths.** If a path is relative, resolve it against the directory containing the filelist. If the filelist location is unknown, resolve against the current working directory. Report any paths that cannot be resolved — do not silently skip them.

3. **Filter to Verilog sources.** Only process `.v`, `.sv`, `.vh`, `.svh` files. Skip any non-Verilog entries.

4. **Identify top-level modules.** A top-level CBB module is the primary module defined in each file — the one whose module name matches the file's base name (or is the only module declaration in the file). Everything else is a sub-module.

---

## Per-Module Analysis Workflow

For each top-level module in the filelist, perform a focused analysis. Read the module declaration only — ports, parameters, and the first-level instantiation block. Do not descend into sub-module internals.

### Step 1: Read Module Declaration

Extract the module's interface surface:

- **Module name** and file path
- **Parameters** — name, default value, and valid range (if documented in comments)
- **Ports** — name, direction (input/output/inout), width (static or parameterized), and any protocol role (clock, reset, data, handshake, etc.)
- **Clock and reset** — identify which ports carry clock and reset signals. Note active level (high/low) and synchronicity (sync/async).

### Step 2: Identify Clock-Reset and Interface Protocol

Clock and reset ports define the module's timing domain. Interface ports define the protocol.

- **Clock domain.** Count distinct clock inputs. One clock = synchronous; two clocks = cross-domain. Record clock names.
- **Reset type.** Synchronous or asynchronous, active-high or active-low. Infer from port names (`rst_n` = async active-low, `rst` = likely sync active-high) and confirm from module code if visible.
- **Interface protocol.** Infer from port groupings:
  - **Valid-ready handshake**: `*_valid`, `*_ready` pairs on the same data bus
  - **AXI-like**: `*_aw*`, `*_w*`, `*_b*`, `*_ar*`, `*_r*` channel groups
  - **APB-like**: `psel`, `penable`, `pwrite`, `paddr`, `pwdata`, `prdata`
  - **Simple data bus**: data in/out with enable or valid, no handshake

### Step 3: Extract Specific Functionality with Variant Distinction

This is the critical step. Broad categories like "FIFO" or "CDC" are insufficient for reuse decisions. The LLM must know exactly which variant the CBB implements.

#### FIFO Variants

| Variant | Distinguishing Evidence |
|---|---|
| Standard FIFO | `din`/`dout` data ports, `wr_en`/`rd_en` enables, `full`/`empty` flags, single clock |
| FWFT (First-Word Fall-Through) FIFO | Standard FIFO ports plus `dout` is valid (not just registered) when `empty` is low without asserting `rd_en`; look for comment "FWFT" or "show-ahead" |
| Async FIFO | Two clock inputs (e.g., `wr_clk`, `rd_clk`), Gray-code pointer sync visible in comments or sub-module names |
| Packet FIFO | Standard FIFO ports plus `sop`/`eop` (start/end of packet) or packet-length sideband signals |

#### RAM Variants

| Variant | Distinguishing Evidence |
|---|---|
| Simple Dual-Port (1R1W) | Separate read and write ports with independent addresses, single clock, no simultaneous read-write to same address |
| True Dual-Port (2R2W) | Two complete read-write port pairs (A and B), each with address, write-enable, write-data, read-data |
| ROM | Write port absent; only read address and read data; contents initialized via parameter or `$readmemh` |
| Single-Port (1R1W shared) | One address port shared between read and write, with write-enable distinguishing the operation |

#### CDC Variants

| Variant | Distinguishing Evidence |
|---|---|
| Bit synchronizer | Single-bit input, two clock domains, 2-stage flop chain in sub-module or inline |
| Bus synchronizer (Gray) | Multi-bit input, two clock domains, Gray-code encode/decode visible in port names or sub-modules |
| Bus synchronizer (handshake) | Multi-bit input, two clock domains, `req`/`ack` handshake signals |
| Pulse synchronizer | Single-bit pulse input (not level), two clock domains, toggle-based protocol in sub-module or comments |

#### Pipeline Variants

| Variant | Distinguishing Evidence |
|---|---|
| Register stage | `din`/`dout` with `valid_in`/`valid_out` (or just clocked register), no backpressure |
| Skid buffer | `din`/`dout` with `valid`/`ready` on both sides, internal "skid" register for holding data during backpressure |
| Elastic buffer | `din`/`dout` with `valid`/`ready` on both sides, parameterized depth > 1, internally a shallow FIFO with handshake |

#### Other Common CBB Types

- **Arithmetic helpers**: multipliers, adder trees, saturators, rounders — distinguish by operator type and pipeline depth
- **Protocol adapters**: AXI-to-APB, APB-to-regfile, width converters — distinguish by protocol pair and data width
- **Clocking**: PLL wrappers, clock dividers, clock muxes — distinguish by function and output count
- **Reset controllers**: reset synchronizers, reset generators — distinguish by sync/async and sequence behavior

### Step 4: Functional Inference

When module comments are sparse, infer functionality from:

- **Port naming conventions.** `wr_en` + `rd_en` + `full` + `empty` = FIFO. `sync_2ff` in a sub-module name = 2-stage synchronizer. `gray` in port names = Gray-code CDC.
- **Parameter names.** `DATA_W`, `DEPTH`, `ADDR_W` = FIFO or RAM. `SYNC_STAGES` = synchronizer depth. `PIPELINE_DEPTH` = pipeline register stages.
- **Sub-module names** (recorded, not analyzed). `gray_ptr_sync` sub-module = async FIFO pointer synchronization. `ram_1r1w` sub-module = FIFO using simple dual-port RAM.
- **Header comments.** Module-purpose comments often state the function directly.

### Step 5: Classify Type

Assign a type category to each CBB:

| Type | Examples |
|---|---|
| `fifo` | Synchronous FIFOs, async FIFOs, packet FIFOs |
| `ram` | Dual-port RAMs, ROMs, register files |
| `cdc` | Bit syncs, bus syncs, pulse syncs, handshake adapters |
| `pipeline` | Register stages, skid buffers, elastic buffers |
| `arithmetic` | Multipliers, adder trees, saturators, rounders |
| `protocol` | Bus adapters, protocol bridges, width converters |
| `clocking` | PLL wrappers, clock dividers, clock muxes |
| `reset` | Reset synchronizers, reset generators |
| `utility` | One-hot encoders, priority encoders, parity checkers |
| `other` | Anything not fitting the above categories |

---

## Sub-Module Boundary Rule

CBBs frequently instantiate helper sub-modules. A `fifo_sync_std` might instantiate `ram_1r1w` and `ptr_gray`. A `cdc_sync_bus` might instantiate `gray_encode`, `sync_2ff`, and `gray_decode`.

### Detection Rules

A module is a sub-module if:
- It appears as an instantiated module inside a top-level CBB (i.e., it is not the primary module of its file, or its file was not in the filelist)
- Its module name differs from the file's primary module name
- It is instantiated more than once with different parameters (e.g., `sync_2ff #(.WIDTH(1))` and `sync_2ff #(.WIDTH(8))`)

### Treatment

For sub-modules, record **name and inferred purpose only**. Do not read the sub-module file. Do not analyze its ports or parameters. Do not create a catalog entry for it.

```
Sub-modules:
  - ram_1r1w: internal storage array (inferred from FIFO architecture)
  - ptr_gray: Gray-code pointer for async clock domain crossing
```

This boundary exists for a reason: sub-modules are implementation details of the CBB. Downstream skills instantiate the CBB, not its sub-modules. Analyzing sub-module internals creates noise in the catalog and wastes context budget.

**Exception**: If a sub-module is also listed as a separate top-level entry in the filelist (i.e., it is independently reusable), treat it as a top-level CBB and create a full catalog entry. This happens when a RAM wrapper is both used inside a FIFO and independently instantiatable.

---

## Output: CBB Summary Catalog

Produce a single markdown document containing all CBB entries. Structure it for fast LLM query — a downstream skill should be able to find the right CBB by scanning the cross-reference table, then read the specific entry for details.

### Per-CBB Entry Format

Each entry follows this structure:

```markdown
## module_name

- **Type**: <fifo | ram | cdc | pipeline | arithmetic | protocol | clocking | reset | utility | other>
- **Variant**: <specific variant, e.g., "FWFT synchronous FIFO">
- **File**: <path from filelist>
- **Clock domains**: <1-clk | 2-clk (wr_clk, rd_clk) | none>
- **Reset**: <sync-high | sync-low | async-high | async-low | none>

### Ports
| Port | Dir | Width | Role |
|---|---|---|---|
| clk | input | 1 | system clock |
| ... | ... | ... | ... |

### Parameters
| Param | Default | Range | Purpose |
|---|---|---|---|
| DATA_W | 8 | >=1 | data bus width |
| ... | ... | ... | ... |

### Use When
- <specific scenario where this CBB is the correct choice>

### Avoid When
- <scenario where a different variant or custom RTL is better>

### Sub-modules
- <name>: <inferred purpose>

### Notes
- <any caveats, assumptions, or unusual characteristics>
```

### Cross-Reference Table

After all entries, produce a summary table mapping design needs to CBBs:

```markdown
## Quick Reference: Need -> CBB

| Need | CBB | Variant | Key Param |
|---|---|---|---|
| Standard buffering, same clock | fifo_sync_std | Standard sync FIFO | DATA_W, DEPTH |
| Zero-latency read, same clock | fifo_sync_fwft | FWFT sync FIFO | DATA_W, DEPTH |
| Buffering across clock domains | fifo_async | Async FIFO | DATA_W, DEPTH, wr_clk, rd_clk |
| Single-bit level CDC | cdc_sync_bit | Bit synchronizer | SYNC_STAGES |
| Multi-bit bus CDC | cdc_sync_bus | Bus synchronizer (Gray) | DATA_W, SYNC_STAGES |
| Single-bit pulse CDC | cdc_sync_pulse | Pulse synchronizer | SYNC_STAGES |
| Pipeline register with backpressure | skid_buffer | Skid buffer | DATA_W |
| Shallow pipeline with backpressure | elastic_buffer | Elastic buffer | DATA_W, DEPTH |
| Simple dual-port memory | ram_1r1w_wrapper | Simple dual-port RAM | DATA_W, DEPTH |
| ... | ... | ... | ... |
```

This table is the primary query surface. Downstream skills scan this table first, then drill into the full entry for port and parameter details.

---

## Guardrails

### Only Document What Exists

- Do not invent CBBs that are not in the filelist. If the user's filelist contains 5 FIFOs and no RAM wrappers, the catalog contains 5 FIFOs and no RAM wrappers.
- Do not extrapolate functionality beyond what the code and comments demonstrate. If a FIFO has no `almost_full` flag, do not document one.

### Mark Inference

- When functionality is inferred from port names or sub-module names (not explicitly stated in comments), mark it: `[inferred from port names]` or `[inferred from sub-module name]`.
- When functionality is stated in module header comments, treat it as confirmed (not inferred).
- When inference is ambiguous (multiple interpretations possible), document all plausible interpretations and mark the most likely one.

### Conservative Guidance

- **Use-when** guidance must be specific. "Use when buffering data between producer and consumer in the same clock domain" is specific. "Use when you need a FIFO" is too vague.
- **Avoid-when** guidance must explain the consequence of misuse. "Avoid when crossing clock domains — lacks async pointer synchronization; use fifo_async instead" is specific. "Avoid for async" is too vague.
- When uncertain about a CBB's suitability for a scenario, omit the guidance rather than guess. The catalog must be a reliable reference, not a speculative one.

### Completeness Over Perfection

- If a module cannot be fully analyzed (e.g., heavily parameterized with unclear port roles), create the entry with what is known and mark unknowns as `[analysis incomplete — review source]`.
- Do not skip modules. Every entry in the filelist gets a catalog entry, even if the entry is minimal.
- Report modules that could not be analyzed at all in a separate "Unresolved Modules" section at the end of the catalog.

---

## Workflow Summary

1. **Receive filelist** from user. Validate format and resolve paths.
2. **Parse filelist** into a list of Verilog source files. Skip non-Verilog entries, comments, and blank lines.
3. **Identify top-level modules** — one per file (or primary module when multiple modules share a file).
4. **Read each top-level module declaration.** Extract ports, parameters, clock/reset, interface protocol.
5. **Infer specific functionality** with variant distinction. Use port names, parameter names, sub-module names, and comments.
6. **Record sub-modules** — name and purpose only. Do not analyze internals.
7. **Produce catalog** with per-CBB entries and cross-reference table.
8. **Report unresolved modules** — any module that could not be analyzed.
