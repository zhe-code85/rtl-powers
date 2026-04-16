---
name: rtl-impl
description: Use when implementing synthesizable Verilog RTL for a stable module or leaf block. Triggers on requests such as "write this FIFO", "turn this design into RTL", "wrap this IP", or "wire up existing CBB/IP", and also when implementation readiness must be checked before coding, when CBB/IP reuse decisions are needed, or when an implementation trace is required for downstream verification or review.
---

# RTL Implementation For Verilog Delivery

Implement synthesizable RTL from stable design inputs. This skill sits between `rtl-design` and `rtl-verification`.

- `rtl-design` owns architecture, boundaries, and design intent.
- `rtl-impl` owns implementation readiness, reuse decisions, coding structure, and RTL delivery.
- `rtl-verification` owns weakness analysis, test planning, and closure.

Use this skill only when the target is already a stable implementation object. If the work still needs boundary changes, contract changes, or major decomposition, return to `rtl-design`.

Before writing or editing RTL, open [references/rtl-rules.md](references/rtl-rules.md). That file is the mandatory rule set for coding and delivery. Open [references/patterns.md](references/patterns.md) only after the gates below are closed and only for the patterns needed by the current task.

## Step 0: Choose The Entry Path

Use `blocker-first clarification` when either condition is true:
- the user explicitly asks to list or confirm the blocking implementation questions first
- the prompt already exposes the local implementation blockers

In that path, the next user-facing response is the blocking implementation questions. Keep them short and implementation-specific. Ask only what is needed to close implementation readiness, reuse, or mandatory-rule gaps.

Use `normal implementation flow` when the prompt does not already expose those blockers.

## Step 1: Confirm The Implementation Target

Continue only when all of these are true:
- the target is one module or leaf block with a clear responsibility
- external interfaces are stable enough to code
- key control, datapath, storage, and timing decisions are already bounded

Return to `rtl-design` when the work still needs:
- decomposition into major child blocks
- moving interface semantics such as ordering, backpressure, flush, retry, or ownership
- new architecture decisions about pipeline or storage organization

## Step 2: Gather Available Inputs

If Step 0 selected `blocker-first clarification`, ask the blocking questions first and gather broader repository context after the user answers.

Otherwise gather the implementation inputs that already exist:
- `rtl-design` module-design document, if present
- parent architecture notes that constrain the block
- user-provided requirements and corrections
- project rule files and nearby RTL style
- CBB/IP catalog, wrapper docs, or approved reuse inventory

A formal design document is helpful but optional. Implementation-ready information is mandatory.

## Step 3: Run The Implementation Readiness Gate

Before coding, confirm these areas are covered:
- `module boundary`
  One implementation object with a clear purpose and non-goals.
- `interface contract`
  Ports, directions, widths, handshake style, timing semantics, ordering rules, backpressure behavior, flush/error behavior, and parameter ranges.
- `clock/reset/integration assumptions`
  Clock domains, reset behavior, CDC boundaries, DFT-visible behavior, and inherited integration assumptions that affect RTL shape.
- `microarchitecture-critical decisions`
  FSMs, counters, buffering, storage ownership, pipeline cuts, arithmetic strategy, and hazard handling.
- `implementation policy`
  Coding-style constraints, reset rules, arithmetic restrictions, lint or synthesis constraints, and comment language.
- `reuse constraints`
  Whether CBB/IP reuse is required, preferred, or disallowed.
- `parameterization rules`
  Valid ranges, boundary values, and contract semantics such as wrap or saturate behavior.

Close the gate this way:
- ask the user about local implementation gaps
- return to `rtl-design` when the gap changes architecture, ownership, or boundaries
- carry user-confirmed assumptions into the implementation summary and trace

Clarification style:
- prefer one focused question when one item blocks progress
- use a short list only when several blockers are tightly coupled
- keep questions narrow and implementation-specific
- keep code, partial skeletons, and strategy proposals out of the clarification response
- keep recommendation content for after the blockers are answered, unless the user explicitly asks for a recommendation

## Step 4: Run The Reuse Gate

Run the reuse gate after implementation readiness is closed and before custom coding starts.

1. Determine from the prompt, design inputs, or project rules whether reuse is required, preferred, or disallowed.
2. Ask the user about reuse preference only when the current inputs do not already answer it.
3. When reuse is required or preferred, scan the repository for explicit catalogs, wrapper docs, IP lists, or approved inventories.
4. Choose only documented reuse candidates.
5. When reuse is required and no explicit inventory exists, ask the user for the catalog before proceeding.

Reuse classes:
- `direct reuse`
  The documented asset already matches the required function, interface semantics, and key parameters.
- `reuse with wrapper`
  The core function matches and wrapper RTL provides adaptation.
- `custom RTL`
  No suitable documented reuse exists, or a custom block is the cleaner implementation boundary.

Record every reuse decision in the implementation summary and trace.

## Step 5: Freeze The Implementation Strategy

Before coding, write a short implementation-granularity summary that freezes:
- module and submodule split
- direct reuse vs wrapper vs custom boundaries
- major FSM, datapath, storage, and pipeline structure
- reset, CDC, and arithmetic strategy
- user-confirmed assumptions that affect code shape

Use this step to prevent coding drift. Keep it short. If this summary exposes new local gaps, return to Step 3. If it exposes boundary changes, return to `rtl-design`.

Emit a code skeleton first only when the user asks for it or when a brief structure review will clearly prevent rework.

## Step 6: Write Synthesizable RTL

Deliver synthesizable `Verilog` RTL. Follow [references/rtl-rules.md](references/rtl-rules.md) as the mandatory rule set and load only the relevant patterns from [references/patterns.md](references/patterns.md).

Make these decisions explicit in the code:
- FSM ownership and safe default behavior
- storage ownership and lifetime
- handshake and stall propagation
- latency-bearing pipeline cuts
- reset scope
- wrapper behavior and adaptation logic

If a user revises a frozen assumption during coding, return to Step 3 before continuing.

## Step 7: Maintain The Implementation Trace

When RTL is created or materially changed, maintain a lightweight implementation trace paired with the code. Use the project documentation location when one exists. Otherwise place the trace in the normal RTL docs area.

The trace records:
- implementation target
- input sources used
- assumptions confirmed with the user
- comment language
- CBB/IP reuse decisions
- implementation choices that shaped the code
- RTL output files
- verification follow-up points
- open risks or unresolved items

Use [references/implementation_handoff_template.md](references/implementation_handoff_template.md).

## Delivery

Default delivery includes:
- synthesizable `Verilog` RTL and any needed wrapper modules
- a short implementation summary
- a lightweight implementation trace
- a verification handoff
- synthesis and integration notes

## Self-Check

Verify that:
- the target was implementation-ready rather than still an architecture problem
- clarification stayed narrow and implementation-specific
- [references/rtl-rules.md](references/rtl-rules.md) was followed
- comment language was confirmed
- reuse preference was checked
- documented support existed for any selected CBB/IP reuse
- the RTL matches the confirmed interface and integration assumptions
- the trace captures assumptions and reuse decisions without repeating `rtl-design`
- the output is specific enough for `rtl-verification` to continue from the implementation

## Stop And Ask For Guidance When

Stop and ask when:
- the object is not yet a stable implementation boundary
- interface semantics are still changing
- coding requirements conflict with [references/rtl-rules.md](references/rtl-rules.md)
- a required readiness or reuse gate is still open
- comment language has not been confirmed
- reuse is required but no explicit CBB/IP inventory is available
- a project rule conflicts with the confirmed inputs
- a critical arithmetic, CDC, or memory decision is still ambiguous

The default failure mode of this skill is to ask a small number of targeted implementation questions.
