---
name: rtl-design
description: Use when the user provides RTL requirements, interface constraints, clock/reset expectations, or PPA targets and needs a design document before writing RTL. Triggers on module partitioning, microarchitecture planning, pipeline cuts, FSM design, CDC/reset strategy, interface definition, or when rtl-impl needs architecture clarification before coding.
---

# RTL Design Document Generation

## Design Philosophy

Design before coding. A design document is a pre-coding constraint, not post-hoc documentation. Writing Verilog without a design document produces code that works in simulation but fails in integration, timing, or reuse.

Requirements drive architecture. Start with what the module must do, constrain it with PPA budgets, then choose the simplest microarchitecture that satisfies both. Every design decision must be traceable to a requirement or a constraint — if you cannot explain why a pipeline stage exists, it should not exist.

## Before Starting the Design

Before diving into requirements analysis, first inspect the user's request and local project context for existing modules, interface standards, and design conventions. Only ask the user about existing design context when it is missing and would change an architecture-level decision.

## Step 1: Requirements Analysis

Extract every constraint from the requirement spec. Missed requirements become change requests during integration, which cost 10x more than catching them here.

### Functional Requirements

For each requirement, extract:
- **Interface**: protocol (AXI4-Stream, APB, custom handshake), port direction, active level
- **Throughput**: sustained rate (transactions per cycle), burst behavior, idle cycles between transactions
- **Latency**: first-response delay, pipeline depth tolerance, worst-case response bound
- **Data format**: bit width, byte ordering, field mapping, alignment, parity or ECC
- **Operating modes**: configuration modes, power states, bypass paths, test modes

### Non-Functional Constraints

These constraints shape microarchitecture decisions directly:
- **Clock frequency**: determines pipeline depth. Ask the user if not specified.
- **Area budget**: LUT/FF count or gate count estimate. Constrained area forces resource sharing.
- **Power budget**: dynamic power limits force clock gating, operand isolation, or domain shutdown.
- **Timing margin**: how much slack is acceptable after synthesis.

### Invariants

List every constraint that must hold under all conditions:
- No packet reordering on a given flow ID
- Backpressure must propagate within N clock cycles
- All outputs driven to known state during reset

Invariants become assertions in the implementation phase.

### Non-Goals

State what is explicitly out of scope. Non-goals prevent scope creep.

### Handling Missing Information

When requirements are incomplete, apply this rule: **make conservative assumptions and document them explicitly; only ask the user when the missing information would change an architecture-level decision.**

Information that changes architecture (must ask if missing):
- Clock frequency target
- Interface protocol and handshake type
- Throughput requirement
- Number of clock domains

Information that has standard defaults (assume and document):
- Reset strategy → default: synchronous active-low, all outputs to idle
- Pipeline structure → decide based on frequency target
- Area budget → assume unconstrained unless specified
- Parameter ranges → infer from requirements, document assumptions

### Hardware Description Rules

Describe the design as hardware, not software. Do not write software pseudocode, imperative algorithms, or function-call narratives.

Every behavior description must be framed in terms of:
- Registers and register boundaries
- Combinational logic, muxes, datapath operators, and storage elements
- FSMs, counters, FIFOs, arbiters, and handshakes
- Clock edges, valid/ready behavior, and cycle-level timing

If a design choice cannot be explained as hardware structure or cycle-level behavior, it is not specified well enough for RTL design.

## Step 2: Initial Sub-module Inventory

Before full microarchitecture planning, draft the candidate sub-modules implied by the requirements. This is an initial inventory, not the final specification. Step 3 may merge, split, or refine these module boundaries.

### What to Specify for Each Sub-module

For every sub-module, document:

| Field | Description |
|-------|-------------|
| Name | Module identifier |
| Type | Datapath / Control / Storage / Interface |
| Function | What this module does — the behavioral requirement |
| Interface | Ports with direction, width, clock domain, and protocol |
| Performance | Throughput, latency requirements, and timing constraints |
| Dependencies | Which other sub-modules it connects to and how |

### Specification Depth

At this stage, the inventory should be precise enough to support architecture exploration:
- The major responsibilities and interfaces are clear
- Likely module boundaries are visible for pipeline and CDC planning
- Step 3 can refine the design without rediscovering the requirements

But it should NOT include:
- Specific CBB names or instantiation details (that is rtl-impl's responsibility)
- RTL implementation details (signal assignments, encoding schemes)
- Verification strategy (that is rtl-verification's responsibility)

## Step 3: Microarchitecture Planning

Transform requirements and the initial sub-module inventory into a concrete microarchitecture. Refine the sub-module boundaries here until the architecture is stable enough that the final design document can hand off cleanly to rtl-impl and rtl-verification.

### Module Partitioning

Decompose the design into sub-modules by function:
- **Datapath modules**: data transformation, arithmetic, muxing, shifting — high-throughput paths where pipeline depth matters
- **Control modules**: FSMs, sequencers, arbiters — generate control signals for the datapath
- **Storage modules**: FIFOs, register files, buffers
- **Interface modules**: protocol adapters, CDC modules, pad wrappers

Each sub-module should have a single clear purpose. A module that does arbitration and data transformation will be hard to reuse, hard to verify, and hard to meet timing.

### Data Flow Analysis

Trace data from input to output:
- **Throughput calculation**: at each pipeline stage, compute the sustained transfer rate. If stage 1 produces 1 word/cycle and stage 2 consumes 1 word every 2 cycles, a buffer is needed between them.
- **Backpressure propagation**: for every valid/ready interface, trace how stall conditions propagate backward. A broken backpressure chain causes data loss.
- **Buffer depth estimation**: for each inter-stage buffer, estimate minimum depth from the worst-case burst size, throughput mismatch between producer and consumer, and backpressure propagation latency. Include a safety margin (typically 20% or 2 entries, whichever is larger). Round up to the supported implementation granularity — often power-of-2 for FIFO-based buffers, but not universally required.

  Example: upstream bursts up to 8 words, downstream consumes at 50% rate, backpressure takes 2 cycles to propagate. Worst-case accumulation during burst = 8 words at 50% fill rate = 4 words, plus 2 words for propagation delay = 6. If backed by a power-of-2 FIFO, round up to 8; otherwise 6 is sufficient.

### Pipeline Decisions

Choose the pipeline structure based on clock frequency target and combinational path depth:

| Structure | When to Use | PPA Tradeoff |
|-----------|-------------|--------------|
| Single-cycle combinational | Path delay < 60% of clock period, simple logic | Lowest latency and area, but fails timing on complex paths |
| Multi-cycle (state machine) | Complex operation that cannot complete in one cycle, low throughput | Acceptable for control-path operations, wastes cycles on throughput-critical paths |
| Pipelined (registered stages) | Throughput is 1 transaction/cycle, path delay exceeds single-cycle budget | Maximum throughput at the cost of pipeline registers and initial fill latency |

Decision rule: pipeline the datapath, multi-cycle the control path. Never pipeline a control FSM unless it runs at a very high rate.

For every pipelined datapath, explicitly define each stage with:
- Stage name or index
- Register boundary at the stage input/output
- Combinational work performed in that stage
- Valid/ready or data-availability behavior at that stage
- Added latency relative to input acceptance

### PPA Techniques

When constraints demand it, document only the techniques actually used:
- **Power**: module-level clock gating, operand isolation
- **Area**: resource sharing, time-division multiplexing, serialized processing
- **Performance**: deeper pipelining, buffering, wider parallelism

For each technique, record the driving constraint, the tradeoff, and the expected benefit.

### Backend Awareness

Design with implementation reality in mind. The target process shapes architecture decisions.

Cover only the backend-sensitive items that affect architecture:
- Likely critical path or timing hot spot
- Preferred register cut point or mitigation
- High-fanout or clock/reset distribution concerns
- Hard macro or physical partition constraints, if any
- FPGA prototype adaptation, if the design must also map to FPGA

Explicitly call out:
- The likely critical path or timing hot spot
- Why it is timing-sensitive
- The preferred register cut point or architectural mitigation

### Clock Domain Partitioning

Identify every clock domain boundary:
- List all clock inputs and their expected frequencies
- Identify every signal that crosses a domain boundary
- For each crossing, choose a CDC strategy:
  - **Single-bit control**: 2-stage synchronizer
  - **Multi-bit data with flow control**: async FIFO or handshake CDC
  - **Multi-bit data with continuous flow**: Gray code counter (depth must be power of 2)
  - **Pulse across domains**: pulse synchronizer (toggle-based, req/ack)

Document every CDC crossing with: source domain, destination domain, signal width, CDC method, and expected latency.

### Reset Strategy

Define reset behavior for the entire module:
- **Reset type**: synchronous vs asynchronous. State the choice and why.
- **Reset domains**: shared reset or independent sub-module resets.
- **Reset state**: defined state of every output after reset. Unspecified reset states cause integration bugs.

### Risk Assessment

For each design decision, state the risk and the fallback:

| Risk | Likelihood | Impact | Fallback |
|------|-----------|--------|----------|
| (example) Custom FIFO fails timing at target frequency | Medium | High | Replace with pipelined version or reduce frequency requirement |

A design without stated risks is a design that will surprise during implementation.

### DFX Planning

Plan DFX features from two sources:
- **Necessity-driven**: observability or controllability required by the module type
- **Weakness-driven**: debug hooks needed to reduce a specific risk

For each DFX feature, document:
- The feature
- Whether it is necessity-driven or weakness-driven
- The design concern it addresses
- How it is exposed: status bit, output port, counter, interrupt, or readback field

## Step 4: Output Requirements

Produce the design document **strictly following the template** in `references/design_doc_template.md`. Use its section structure exactly — do not add, remove, or reorder top-level sections. If a section does not apply, write "Not applicable because [reason]" rather than omitting it.

The template contains three parts:
1. **Part 1**: Annotated blank template — use this structure
2. **Part 2**: Complete example (AXI Packet Buffer) — reference for depth and style
3. **Part 3**: Standard format reference — use these table formats exactly

### Key Points by Template Section

Use the template structure exactly. The main non-obvious requirements are:
- `§1`: include open questions, pending confirmations, and architecture-driving assumptions
- `§2`: use the final refined sub-module boundaries from Steps 2-3
- `§3.5`: for pipelined datapaths, include a stage table before the cycle-by-cycle timing
- `§3.9`: identify timing hot spots and preferred cut points or mitigations
- `§4`: mark implementation details as deferred to `rtl-impl`
- `§5-§6`: quantify tradeoffs and risks; link DFX to risks where applicable

Use hardware semantics throughout. Do not use software pseudocode.

### Output Delivery

Deliver the design document as inline markdown in the current response by default. Only write to a file if:
- The user explicitly asks for a file, or
- The project has an established `docs/` directory and file naming convention

When writing to a file, name it based on the top-level module name, following the project's existing naming conventions if visible.

After completing the document, inform the user that the design document is ready and can be used as input for the `rtl-impl` skill (implementation) or the `rtl-verification` skill (verification planning).

## Self-Check Before Delivery

Before presenting the design document, verify:
- Requirements, open questions, and assumptions are explicit in `§1`
- Final sub-module specs are complete in `§2`
- Top-level interfaces, parameters, FSMs, timing, CDC, and reset are complete in `§3`
- Pipelined datapaths include a stage table in `§3.5`
- Backend awareness identifies timing hot spots and mitigations in `§3.9`
- DFX features trace to a specific necessity or weakness in `§3.10`
- Trade-offs are quantified in `§5`
- Risks are explicit and connected to mitigations or DFX in `§6`
- The document uses hardware semantics rather than software pseudocode

If a check fails:
- Missing content → go back and fill it in
- Inconsistency (e.g., diagram vs table) → resolve by updating both
- Truly not applicable → write "Not applicable because [reason]"
