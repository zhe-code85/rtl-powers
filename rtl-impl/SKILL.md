---
name: rtl-impl
description: Implement PPA-optimized Verilog RTL from design documents with CBB reuse. Use when writing RTL code, implementing a module, coding a design, converting a design document into Verilog, or when user asks to write/implement/generate RTL. Also use when user provides a design doc and asks for implementation. Triggers on RTL coding, Verilog implementation, synthesizable code generation, or any request to produce .v files from a design.
---

# rtl-impl: Design Document to Synthesizable Verilog

## Implementation Philosophy

RTL implementation is the bridge between design intent and silicon reality. The design document owns *what* to build; this skill owns *how* to build it in Verilog. Every coding decision carries PPA consequences — poor choices at coding time compound through synthesis and cannot be fully recovered later.

Core principles:

- **Code follows design.** If the design document is incomplete or ambiguous, stop and resolve the gap. Do not guess at intent and fill in details silently. Point the user back to `rtl-design` for clarification. If the user provides only a verbal description with no design document, ask whether to run `rtl-design` first or proceed with a note that a formal design document was skipped.
- **PPA awareness at coding time.** Think about timing paths, area cost, and switching activity while writing RTL — not after synthesis reports come back. A functionally correct module that fails timing by 30% is not a successful implementation. Every `if`, every `+`, every register has a PPA cost that must be justified.
- **Synthesizable, maintainable, reviewable.** Every line of Verilog must synthesize cleanly on the first pass, read clearly to another engineer in a code review, and survive modification without silent breakage. Clever code that saves two lines but confuses a reviewer is a net loss.

### Why This Order Matters

The skill steps are ordered deliberately: inputs first, reuse gate before coding, patterns applied during coding, anti-patterns checked after, project rules integrated throughout. Skipping the reuse gate leads to duplicated effort. Ignoring anti-patterns leads to synthesis surprises. Applying project rules last leads to rework. Follow the order.

---

## Step 0: Gather Inputs

Before writing any Verilog, collect these inputs:

1. **Design document** — the primary input. Contains module partitioning, interface ports, parameters, FSM definitions, timing descriptions, and CBB reuse decisions. The design document is a constraint, not a suggestion. If it says "use fifo_sync for the input buffer," use fifo_sync — do not substitute a different structure without justification.

2. **CBB summary catalog** — needed to resolve instantiation details. If the design document references CBBs, read the catalog for port lists, parameter meanings, and variant details. If no catalog exists, prompt the user to run `cbb-asset` to generate one.

3. **Project rules** — optional but critical when present. Read project rule files (e.g., `rtl_rules.yaml`) when available. These encode team conventions that override generic defaults. A project rule that says "synchronous active-low reset only" means exactly that — do not use asynchronous reset even if the design document mentions it.

4. **Surrounding module context** — if extending an existing codebase, read nearby modules for naming conventions, parameter style, and structural patterns. Consistency with existing code reduces review friction. If the codebase uses `always_ff` / `always_comb` two-process style, match it.

---

## Step 1: CBB Reuse Gate (Mandatory Pre-Coding Step)

Before writing any custom RTL, run the CBB reuse gate. This is not optional — it prevents reimplementation of proven structures and ensures the design document's reuse decisions are honored.

### Why This Step Exists

Reimplementing a FIFO, CDC synchronizer, or RAM wrapper from scratch wastes verification effort and introduces bugs that CBBs have already solved. The reuse gate forces a disciplined check before custom code is written.

### Gate Procedure

1. **Check catalog existence.** If the design document references CBBs but no catalog is available, prompt the user to generate one with `cbb-asset`. If the user declines, note in the output that reuse verification was skipped and proceed cautiously.

2. **Match requirements to catalog entries by functional specificity.** Match on the specific variant needed, not broad category. A design needing a "FWFT FIFO with configurable depth" does not match a "standard FIFO" entry — it needs the FWFT variant. Distinguish between:
   - FIFO: standard vs FWFT vs async vs packet
   - RAM: simple dual-port vs true dual-port vs ROM
   - CDC: bit sync vs bus sync (Gray/handshake) vs pulse sync
   - Pipeline: register vs skid buffer vs elastic buffer

3. **Make the reuse decision per module:**

   | Match Level | Action | When to Apply |
   |---|---|---|
   | Exact match | Instantiate directly | CBB provides exactly the needed function, interface, and parameterization |
   | Partial match | Wrap CBB with glue logic | Core function matches but interface adaptation, width conversion, or protocol bridging needed |
   | No match | Custom RTL with replaceable boundary | No CBB covers the function; design the module so it can be replaced later if a CBB becomes available |

   For partial matches: evaluate whether the wrapper adds more complexity than a clean custom implementation. A wrapper that needs 50% of the logic of a custom module is acceptable. A wrapper that needs 90% probably is not — go custom instead.

4. **Document the reuse decision.** Include in the output summary: which CBBs were instantiated, which were wrapped (and why), which modules are custom (and why no CBB matched), and the rationale for each decision.

### Instantiation Rules

- Use named port connections (`.port(wire)`) for all CBB instantiations. Never use positional connections — port order may change between CBB versions.
- Connect all CBB ports, even unused ones. Tie off unused outputs to dangling wires with descriptive names (e.g., `wire fifo_almost_full_unused;`). This prevents synthesis warnings and documents the intentional non-use.
- Pass parameters explicitly even when using defaults — this documents intent and prevents breakage if CBB defaults change in a future version.
- Respect CBB parameter names and meanings from the catalog. Do not guess parameter semantics.

---

## Step 2: PPA-Aware Coding Patterns

After the reuse gate determines which modules are instantiated vs custom, apply PPA-aware patterns to all custom RTL. The pattern categories below correspond to the detailed annotated examples in `references/patterns.md`.

### Datapath

Datapath logic moves and transforms data. PPA priorities: timing closure on critical paths, area through resource sharing, power through clock gating idle paths.

- **Pipeline boundary placement.** Break long combinational paths at natural boundaries: after wide arithmetic, before large mux trees, at protocol transitions. Insert valid-ready handshaking at pipeline boundaries to support backpressure without stall logic complexity. Why: a pipeline stage costs one register per bit but reduces combinational depth by 50% or more on critical paths.

- **Operator chaining.** For reductions and trees (parity, priority encode, multi-input add), use balanced tree structures instead of linear chains. A balanced adder tree for N inputs has depth log2(N); a linear chain has depth N-1. The area is identical but timing improves dramatically. For 8 inputs: linear chain = 7 adders in series; balanced tree = 3 levels.

- **Resource sharing.** When throughput allows, time-multiplex expensive operators (multipliers, large comparators) across multiple data streams. One multiplier running at 2x throughput uses half the area of two parallel multipliers. Only share when the sharing control logic (a mux and a toggle counter) is simpler than the duplicated resource. Do not share when the multiplexing creates a timing-critical path.

- **Mux fanout discipline.** Limit the width of multiplexer trees. A 32:1 mux on a 64-bit bus creates enormous fanout on the select signal. Decompose into hierarchical mux stages separated by registers, or restructure the select logic to reduce the mux ratio. If a select signal fans out to more than ~8 wide muxes, replicate it through a register.

### Control Logic

Control logic orchestrates datapath behavior. PPA priorities: correct FSM behavior (no dead states), minimal combinational depth for next-state logic, clean default handling.

- **FSM encoding.** Use one-hot encoding for FSMs with more than ~8 states — it reduces next-state logic to single-bit comparisons and maps well to FPGA LUTs. Use binary encoding for small FSMs (4-8 states) where decode logic is trivial and one-hot wastes flip-flops. Why: one-hot eliminates the wide comparator on state equality and reduces LUT depth for next-state logic.

- **Default state handling.** Every FSM must have an explicit default state for unhandled transitions. The default should be a safe recovery state (typically IDLE), not a hang state. Use `default` in case statements — do not rely on every state being explicitly handled. Without a default, a single-bit upset in the state register can cause permanent deadlock.

- **Branch structure.** Flatten complex conditional branches into priority-encoded if/else chains only when priority matters. For mutually exclusive conditions, use case statements or parallel if blocks to give the synthesizer freedom to optimize. Why: priority encoding forces a specific evaluation order; parallel encoding lets synthesis choose the fastest path.

- **Control-datapath separation.** Keep FSM next-state logic in its own `always_comb` block, separate from datapath multiplexing logic. Mixing control and datapath in a single always block makes the code harder to review and prevents independent optimization of each path.

### Storage

Storage elements hold state between cycles. PPA priorities: correct reset behavior, minimal switching power, appropriate memory type selection.

- **Register arrays vs SRAM.** Use register arrays for shallow storage (typically < 32 entries) where single-cycle read latency is required. Infer SRAM for deep storage (> 32 entries) where area density matters more than single-cycle access. The boundary depends on the target technology — check project rules for guidance. Why: a 256-entry register array of 32-bit words costs 8192 flip-flops; the same storage as BRAM costs one block RAM.

- **Reset strategy.** Reset only the registers that need deterministic startup values: FSM state registers, control registers, pointers, and configuration registers. Do not reset pipeline stage registers or datapath registers that are written before they are read in normal operation. Why: unnecessary resets add routing congestion for the reset tree and prevent synthesis from retiming registers across pipeline boundaries to balance timing.

- **Clock gating.** Gate clocks on large register banks (e.g., FIFO memory arrays) when the bank is idle. Use ICG (integrated clock gating) cells in ASIC; use clock-enable registers in FPGA. Do not manually gate clocks with AND gates — this creates glitch risk and breaks timing analysis.

### Interface

Interface logic connects modules and crosses domain boundaries. PPA priorities: protocol correctness, clean CDC handoff, minimal latency overhead.

- **Handshake protocols.** Use valid-ready handshaking for intra-chip data interfaces. Valid-ready supports variable-latency producers and consumers without stall chain complexity. For fixed-latency paths where backpressure is impossible, a simple valid signal suffices. Why: valid-ready localizes backpressure to each link rather than propagating stall signals through the entire datapath.

- **CDC boundaries.** Instantiate dedicated CDC synchronizer modules — never inline synchronizer flip-flops in random logic. Use the CBB catalog's CDC modules based on signal type: `cdc_sync_bit` for single-bit control signals, `cdc_sync_bus` (Gray or handshake variant) for multi-bit data or pointers, `cdc_sync_pulse` for pulse signals. Never double-flop a multi-bit bus — individual bits may settle in different cycles, producing transient garbage values.

- **Bus width decisions.** Wider buses reduce throughput pressure but increase routing congestion and area. Choose the minimum width that meets throughput requirements with acceptable latency. Document the width decision and its throughput implication.

- **AXI interfaces.** When connecting to AXI buses, keep the subordinate interface thin (AXI4-Lite for control registers, AXI4 for data bursts). Do not implement full AXI features (cache, burst, lock, QoS, region) unless the design document explicitly requires them. Minimal AXI saves area and verification effort.

### Arithmetic

Arithmetic operators are the most PPA-sensitive structures in RTL. PPA priorities: DSP mapping, timing on critical arithmetic paths, avoiding operators that synthesize poorly.

- **Multiply-accumulate.** Structure MAC patterns to map cleanly to DSP blocks: keep the form `P + A * B` where operand widths match DSP input widths (typically 18x18 or 18x25 depending on platform). Avoid intermediate truncation or rounding inside the accumulation — let the DSP block handle full precision and truncate at the output. Why: any operation between the multiply and the accumulate (shift, truncate, round) prevents the synthesizer from mapping to a single DSP block.

- **Division.** Avoid the `/` operator entirely for variable divisors — it synthesizes to enormous combinational logic that will not meet timing. Use an iterative divider (one bit per cycle) or an approved IP divider block. Constant division by a power of 2 can use right-shift; other constants use shift-and-add decomposition.

- **Saturation logic.** Implement saturation as a comparison followed by a mux, not as arithmetic with overflow detection. The comparison-mux structure maps to LUTs and carry chains efficiently. Why: overflow detection requires examining all bit positions simultaneously; comparison is a single carry chain.

- **Bitwidth discipline.** Explicitly size all intermediate results using width-specifying constructs. Do not rely on Verilog default 32-bit sizing for arithmetic intermediates — this creates unnecessary width in the synthesized netlist and can cause subtle bugs when widths do not match expectations. For example, `wire [15:0] a, b; wire [31:0] p = a * b;` — without the explicit `[31:0]` on `p`, the multiply result would be truncated to 16 bits before assignment.

---

## Step 3: Common PPA Anti-Patterns

These patterns appear in functionally correct RTL that still fails PPA goals. Recognize and avoid them proactively.

| Anti-Pattern | Why It Hurts | What To Do Instead |
|---|---|---|
| Long combinational chains (depth > 10 logic levels) | Fails timing at any reasonable clock frequency | Insert pipeline registers at natural boundaries |
| Mux fanout explosion (one select drives many wide muxes) | Creates high-fanout net that buffers slowly and consumes routing | Replicate the select through a register; decompose mux tree into stages |
| Unnecessary latch inference (incomplete assignments in combinational blocks) | Unintended storage, makes timing analysis difficult, tool-specific behavior | Assign all outputs in all branches of `always_comb` blocks |
| Reset logic preventing retiming (resetting pipeline registers that don't need it) | Synthesis cannot move registers across pipeline boundaries to balance timing | Reset only control and state registers; leave pipeline data unreset |
| Arithmetic operators that won't map to DSP (`*` with non-standard widths, `/` with variable) | Massive LUT fabric usage instead of dedicated DSP blocks | Use DSP-width operands; replace variable `/` with iterative divider IP |
| Wide mux trees on critical paths (32:1 mux on 64-bit bus) | Single-level mux creates long propagation delay through LUT chain | Hierarchical mux with register stages between levels |
| Asynchronous reset without synchronizer on deassertion | Reset release metastability causes random state corruption | Synchronize async reset deassertion through a 2-stage synchronizer |
| Missing default cases in FSM and mux logic | Latches or undefined behavior for uncovered inputs | Always include `default:` with safe recovery to IDLE |

### Scenario: Identifying Anti-Patterns in Existing Code

When modifying existing RTL, check for these anti-patterns in the surrounding code before adding changes. Fixing an anti-pattern while making nearby modifications is legitimate improvement. Adding new code that perpetuates an existing anti-pattern is not.

---

## Step 4: Project Rules Integration

Project rules encode team and platform conventions. When available, read project rule files and apply them as constraints that override generic defaults. This is not optional when rule files are present.

### How to Apply Project Rules

1. **Detect rule files.** Look for YAML or markdown files in the project configuration directory (e.g., `rtl_rules.yaml`, `rtl_rules_strict.yaml`). If the user mentions project rules, ask for the file path.

2. **Apply rules by category:**

   | Rule Category | What It Controls | Override Priority |
   |---|---|---|
   | `coding_style` | Sequential block style (two-process DQ vs single-process), naming conventions | High — must match existing codebase |
   | `arithmetic` | Variable division, modulo, constant division restrictions | High — prevents synthesis failures |
   | `reset_clock` | Sync vs async reset, active level, explicit reset values | High — affects all sequential logic |
   | `reuse_policy` | Require CBB search before custom, prefer approved IP | Medium — affects reuse gate decisions |
   | `implementation` | Max arithmetic complexity, pipeline preferences | Medium — affects coding pattern selection |
   | `fpga` / `asic` | DSP/BRAM/vendor IP preferences, platform-specific rules | Medium — affects inference and instantiation |
   | `verification` | Required verification levels, assertion review | Low — noted for downstream verification skill |

3. **Conflict resolution.** When project rules conflict with this skill's default patterns, project rules win. Document the override in the output summary. If a rule seems wrong for the specific design, note the concern and apply the rule anyway — the user can explicitly override.

### Common Rule Patterns

- **`dq_two_process` style**: Separate `always_ff` for sequential logic and `always_comb` for combinational logic. Never mix sequential and combinational assignments in one block. This is the most common project rule and must be followed strictly.
- **`synchronous_active_low` reset**: Use `if (!rst_n)` in `always_ff @(posedge clk)` blocks. No `always_ff @(negedge rst_n)` sensitivity. This means: no asynchronous reset in the sensitivity list, only synchronous reset checked on clock edges.
- **`allow_variable_division: false`**: Replace any `/ divisor` where divisor is not a compile-time constant with an iterative divider or IP block. Zero exceptions.
- **`prefer_dsp_for_multiplier: true`**: Structure multiply operations as `P + A * B` forms that map to DSP48/EDSP blocks. Avoid multiply constructs that synthesize to LUT fabric.
- **`max_recommended_comb_depth: 12`**: If the design has combinational paths exceeding this depth, insert pipeline registers. This is a guideline, not a hard constraint, but exceeding it substantially guarantees timing failure.

---

## Step 5: Output Format

Every implementation must produce structured output. This is not optional — downstream skills (rtl-verification, synthesis) depend on this summary.

### Implementation Summary
- Module name and brief function description
- CBB reuse decisions: instantiated (which CBB, parameters), wrapped (which CBB, what glue logic), custom (why no CBB matched), with rationale for each
- Project rules applied: which rules were present, which were followed, any noted conflicts

### RTL Structure Decisions
- Pipeline depth and stage boundary locations with rationale
- FSM encoding choices and state count per FSM
- Memory type decisions (register array vs SRAM inference) with depth/width justification
- Interface protocol choices and width decisions with throughput analysis
- Arithmetic implementation strategy (DSP-mapped, iterative, IP)

### Notes for Verification Follow-Up
- Suggested assertion locations (boundary checks, protocol monitors, invariant assertions)
- Known corner cases that need directed tests (counter wrap, FIFO full+drain, simultaneous events)
- Assumptions made during implementation that verification should confirm

### Notes for Synthesis Follow-Up
- Expected critical paths and why they might be tight (wide arithmetic, deep mux trees)
- Multi-cycle path or false path annotations that synthesis constraints may need
- Clock domain crossing points that need constraint annotations

---

## Coding Checklist

Before delivering RTL, verify every item:

- [ ] All ports declared with explicit widths (`[WIDTH-1:0]`, not bare integers)
- [ ] All parameters have defaults and documented valid ranges
- [ ] Sequential logic uses `always_ff`, combinational uses `always_comb` (two-process DQ style)
- [ ] No incomplete assignments in `always_comb` blocks — all outputs assigned in all branches
- [ ] No unintended latches
- [ ] FSM has explicit `default` state with safe recovery behavior
- [ ] Arithmetic intermediates have explicit bitwidth (no reliance on Verilog default 32-bit)
- [ ] No variable-operand division or modulo
- [ ] CBB instantiations use named port connections, all ports connected
- [ ] No inline CDC synchronizers — dedicated CDC modules used instead
- [ ] Reset strategy matches project rules (sync vs async, active level, which registers reset)
- [ ] Module header comment documents function, interface, and key parameters
- [ ] No combinational loops or feedback paths
- [ ] No `casex` — use `casez` for don't-care bits, `case` for everything else

---

## When to Escalate

These situations require stopping and asking for guidance rather than proceeding independently:

- **Design document unclear or missing.** Stop and recommend running `rtl-design` first. Do not guess at design intent — an incorrect implementation wastes more time than requesting clarification.
- **No CBB catalog available but design references CBBs.** Recommend running `cbb-asset` to generate one before proceeding. Without the catalog, CBB parameter meanings must be guessed.
- **Project rules and PPA best practices conflict.** Apply project rules but flag the conflict clearly in the output. Do not silently ignore a rule to optimize PPA.
- **Complex arithmetic beyond simple DSP mapping.** Flag for design review — complex arithmetic structures (CORDIC, floating-point, non-standard rounding, barrel shifters wider than 32 bits) need careful analysis beyond standard patterns.
- **Multi-clock design with non-standard CDC requirements.** Flag for review — custom CDC logic is high-risk and should not be implemented ad hoc. Use proven CDC CBBs or approved IP.
- **Verification-critical modules.** If the module controls safety, security, or system-critical state machines, note this prominently so rtl-verification can apply appropriate rigor.

---

## References

- **`references/patterns.md`** — Annotated Verilog examples for all coding patterns referenced in Step 2. Organized by category (datapath, control, storage, interface, arithmetic, CBB instantiation) with good-vs-bad comparisons and PPA rationale. Consult this file for concrete implementation templates when applying patterns.
