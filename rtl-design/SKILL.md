---
name: rtl-design
description: Use when a digital design engineer needs architecture or detailed design documentation for an assigned ASIC subsystem, block, or RTL module before implementation. Triggers on subsystem layering, recursive module decomposition, protocol/config/control/datapath partitioning, interface contracts, microarchitecture planning, timing or pipeline decisions, CDC/reset/test assumptions, or when rtl-impl or rtl-verification needs a design handoff document. Also use when the user says things like "design this DMA engine", "split this block/module", "refine this subsystem", "give me an architecture doc", "continue from top-level to module design", or similar colloquial requests to design or decompose RTL hardware.
---

# RTL Design For ASIC Handoff

Generate design documents for digital design engineers. This skill does not replace system engineering or chip-level architecture work. It starts from an assigned subsystem, block, or module and produces a document that can drive RTL implementation and verification.

Do not select specific CBBs in this skill. Describe required structures such as FIFOs, storage arrays, arbiters, register slices, CDC structures, or counters by role and constraints only. Specific reuse mapping is deferred to `rtl-impl`.

## Step 0: Determine The Mode

Choose one mode before designing:

- `architecture mode`: use when the current object still needs decomposition, when module boundaries are not final, or when the main task is to define layering, partitioning, budgets, and interface contracts.
- `module-design mode`: use when the current object has reached an implementation boundary and can be described directly in terms of FSMs, datapaths, storage, timing, corner cases, and verification hooks.

If the user already states the mode, follow it. Otherwise infer the mode from the request.

Choose `module-design mode` only if most of these are true:
- External interface contracts are stable
- Main responsibility is single-purpose and bounded
- The object does not still need to split into several major child blocks
- The design can be described directly as FSM, datapath, storage, and timing behavior

If the mode is still ambiguous after inspecting the request and project context, ask one question:
`Is this object still being decomposed, or is it already ready for leaf/block-level detailed design?`

## Shared Rules

- Treat the current object as an ASIC subsystem, block, or module owned by a digital design engineer.
- Use hardware semantics only: registers, combinational logic, valid/ready rules, storage ownership, clock edges, and cycle-level timing.
- Record project dependencies explicitly. Do not invent chip-level policy for reset, DFT, clocking, test modes, power intent, or physical integration. If the project already defines them, inherit them. If not, list them as assumptions or open questions.
- Do not default reset strategy to a house style. Reset choice is an integration dependency unless the user specifies it.
- If reset, test, or clock interface names, active levels, or exact semantics are not explicitly provided by the user or parent document, describe them as inherited dependencies rather than inventing concrete ports or polarity.
- Keep protocol adaptation and configuration semantics visible, but allow light protocol state handling inside the core control layer when that simplifies the design.
- Make verification guidance part of the design output: invariants, corner cases, illegal cases, observability hooks, and assumptions that the verification plan must honor.

## Architecture Mode

Use this mode for the current design layer when the object still needs structural decomposition.

### Step 1: Capture Scope And Constraints

Extract and state:
- What this object is responsible for in the parent system
- The feature list derived from user requirements, parent constraints, and current-layer ownership
- External interfaces and protocol contracts
- Throughput, latency, buffering, ordering, and QoS requirements
- Area, frequency, power, and timing-margin budgets
- Dependencies inherited from the platform: clocking, reset, test mode behavior, power-domain assumptions, macro boundaries, or integration restrictions
- Known or suspected clock-domain, reset-domain, and power-domain boundaries that could affect partitioning
- Open questions that could change the partitioning

### Step 2: Build The Layer Skeleton

Start with this default skeleton:
- `protocol/config layer`
- `core control layer`
- `datapath/storage layer`

Then analyze the main flow as:
- `ingress`
- `schedule`
- `egress`

Use the flow view to refine the layer view, not to replace it.

### Step 3: Decompose Recursively

Expand the module tree as far as the current information supports. Do not stop at a shallow block diagram if additional structure is already clear.

For every node in the tree, mark one status:
- `ready for module-design`
- `needs further architecture`
- `TBD`

### Step 4: Apply Partitioning Rules

Use these rules to keep the decomposition professional and consistent:

- Separate external protocol adaptation and CSR/config decode from core policy unless the logic is trivial and tightly coupled.
- Separate throughput-critical datapath or storage ownership from heavy arbitration or multi-client control.
- Split out a scheduler or arbiter when fairness, priority, ordering, or backpressure policy becomes a first-class concern.
- Split out storage management when occupancy tracking, hazard handling, replay, buffering, or flow-control semantics dominate the design.
- Do not mix unrelated ingress and egress protocol details into a single control block unless the object is very small and the coupling is essential.
- Keep error, flush, retry, and recovery behavior explicit. They may stay in the core control layer, but they must not be hidden inside datapath prose.
- Separate blocks across power-domain boundaries when isolation, retention, or power-state sequencing would otherwise become implicit side effects inside a functional block.
- Make CDC boundaries explicit at architecture level. If a suspected crossing changes buffering, control ownership, or integration structure, give it its own boundary or wrapper rather than burying it in prose.
- Prefer boundaries that improve implementation and verification clarity over boundaries chosen only for aesthetic symmetry.

Allocate budgets while partitioning rather than leaving them to the end:
- Give storage-heavy blocks most of the area budget when buffering, descriptors, or queueing dominate the subsystem.
- Reserve the tightest timing budget for the blocks on the likely throughput-critical path, then distribute looser timing targets to sideband, config, or statistics logic.
- Assign latency budgets per major path segment such as ingress classify, schedule/route, and egress launch instead of only stating an end-to-end number.
- If the budget split is still uncertain, mark it as provisional and explain which open question would move the allocation.

### Step 5: Produce The Architecture Document

Use `references/architecture_design_template.md`.

The architecture document must include:
- Feature summary with source, status, owner, and design impact
- Layer map
- Flow map
- Recursive module tree with status per node
- Partition rationale for every major child block
- External and internal interface contracts
- Domain-boundary summary covering clock/reset/power boundaries that matter to decomposition
- Budget allocation and integration-critical boundaries
- Verification hooks and architecture-level invariants
- Clear next-step handoff: which nodes are ready for detailed design next

## Module-Design Mode

Use this mode only after the current object is stable enough to implement directly.

### Step 1: Anchor The Parent Context

State:
- Parent architecture document or parent block, if available
- The module's exact responsibility and non-goals
- Which parent-level contracts this module must satisfy
- Which assumptions remain inherited from the parent instead of being redesigned here

### Step 2: Freeze The Interface Contract

Define:
- Top-level ports and interface semantics
- Valid/ready, request/ack, or bus timing rules
- Ordering, flush, retry, error, and backpressure behavior
- Parameter ranges and configuration effects

If these are still unstable, stop and return to `architecture mode`.

### Step 3: Describe The Microarchitecture

Describe the module in enough detail to hand off to implementation and verification:
- Block diagram and local child blocks, if any
- FSMs, schedulers, counters, scoreboards, and control-state ownership
- Datapath and storage organization
- Pipeline cuts, timing-sensitive paths, and buffering rationale
- CDC, reset, test-mode, and integration assumptions that affect the module
- Implementation-critical boundaries and deferred items

Use explicit formats for control and timing:
- For each FSM or scheduler, provide at least a state table and a state-transition view. Prefer Mermaid `stateDiagram-v2` when it stays readable; otherwise use a compact ASCII diagram plus a transition table.
- For each timing-critical path, identify the source state or register boundary, the combinational work between stages, the preferred cut point, and why that cut point is chosen.
- If a path is intentionally multi-cycle or rate-limited, say why that is architecturally safe and what contract makes it safe. Do not imply STA exceptions unless the design contract clearly justifies them.
- When describing register insertion, explain whether it exists to break a long control chain, isolate a wide datapath, absorb backpressure, or localize fanout.

### Step 4: Add Verification Guidance

State:
- Invariants
- Corner cases
- Illegal cases
- Observability and debug hooks
- Risks and fallback strategies

### Step 5: Produce The Module Design Document

Use `references/module_design_template.md`.

The module design document must be detailed enough that:
- `rtl-impl` can choose structures and coding style without rediscovering the architecture
- `rtl-verification` can derive test intent, assertions, and coverage targets directly from the document

## Handling Missing Information

Ask the user only when the missing information changes the design boundary or invalidates a key contract.

Treat missing information as boundary-changing when any of these would become different:
- The current object would switch mode between `architecture mode` and `module-design mode`
- A child block would merge, split, or move between protocol/config, core control, and datapath/storage layers
- An interface contract would change in ordering, backpressure, retry, flush, or ownership semantics
- A timing plan would change from single-stage to pipelined, or from direct flow to buffered flow
- A suspected CDC, reset-domain, or power-domain boundary would require a new wrapper, queue, synchronizer boundary, isolation point, or ownership split

Must ask if missing and architecture would change:
- Main protocol or handshake style
- Throughput or latency contract
- Whether the current object is still decomposing or already implementable
- Ordering, flush, retry, or error semantics that affect partitioning

Usually inherit or record as assumptions instead of inventing policy:
- Reset topology defined by the parent or project
- Test modes and DFT hooks defined by the platform
- Physical or power-domain assumptions supplied by integration

If an inherited dependency is unknown, record it under assumptions or open questions. Do not silently pick a default that narrows the architecture.

## Output Delivery

Deliver inline markdown by default. Write to a file only if:
- The user explicitly asks for a file, or
- The project already has an established docs location and naming scheme

When writing a module design document, reference the parent architecture document if available.

## Self-Check Before Delivery

Verify:
- The selected mode matches the maturity of the current object
- Architecture documents expand the tree as far as justified and mark node status clearly
- Module design documents only target objects that are already implementation-ready
- Layering follows `protocol/config`, `core control`, `datapath/storage`, with `ingress/schedule/egress` used as a supporting view
- Interface contracts and inherited assumptions are explicit
- Reset, test, and integration dependencies are recorded instead of guessed
- No specific CBB names or instantiation choices appear in the design output
- Verification hooks, invariants, and edge cases are explicit enough for `rtl-verification`

If a check fails:
- Wrong mode or unstable interfaces: stop and move the object back to `architecture mode`
- Missing budget split or timing rationale: add a provisional allocation and say what assumption drives it
- Missing FSM, pipeline, or interface detail in a module document: expand the relevant section before delivery
- Guessed reset/test/clock semantics: replace them with inherited assumptions or explicit open questions
- Weak verification handoff: add a structured summary of invariants, corner cases, illegal cases, observability hooks, and risk focus items
