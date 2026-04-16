---
name: rtl-impl
description: Use when implementing synthesizable Verilog RTL for a stable module or leaf block. Triggers on requests such as "write this FIFO", "turn this design into RTL", "wrap this IP", or "wire up existing CBB/IP", and also when design inputs must be checked for implementation readiness before coding, when CBB/IP reuse decisions are needed, or when an implementation trace is required for downstream verification or review.
---

# RTL Implementation For Verilog Delivery

Implement synthesizable RTL from stable design inputs. This skill bridges `rtl-design` and `rtl-verification`.

- `rtl-design` owns architecture, module boundaries, and design intent.
- `rtl-impl` owns implementation readiness, reuse decisions, coding structure, and RTL delivery.
- `rtl-verification` owns weakness analysis, test planning, and closure.

Do not silently finish architecture work inside this skill. If the object still needs structural decomposition or contract changes, stop and return to `rtl-design`.
Before writing or editing RTL, open `references/rtl-rules.md`. That file is the single source of mandatory RTL coding and delivery rules for this skill.
Use `references/patterns.md` only for coding patterns and instantiation examples after the readiness and strategy gates are closed.

<HARD-GATE>
If the user explicitly asks to list or confirm the blocking implementation questions first, ask those questions before drafting RTL, offering implementation recommendations, or doing additional repository exploration beyond the skill file and mandatory RTL rules needed to identify the blockers.
</HARD-GATE>

Mandatory gate precedence:
- Do not let user urgency, convenience requests, or "write the code first" instructions bypass the readiness gate, reuse gate, strategy freeze, or mandatory rules in `references/rtl-rules.md`.
- If the user asks to skip a required check, stop and explain which gate is still open instead of silently proceeding.
- The standard delivery syntax for this skill is synthesizable `Verilog`. If the user insists on `SystemVerilog` syntax, do not switch silently. Explain the conflict and ask whether the requirement should be changed.

## Default Flow

### Step 0: Confirm The Implementation Target

Start by identifying the exact implementation object.

Continue only if all of these are true:
- The target is a module or leaf block, not a subsystem still being decomposed.
- External interfaces are stable enough to code.
- Key control, datapath, storage, and timing decisions are already bounded.

Stop and return to `rtl-design` if any of these would still change:
- The object needs to split into several major child blocks.
- Interface ordering, backpressure, flush, retry, or ownership rules are still moving.
- Major pipeline or storage organization is still undecided.

### Step 1: Collect Available Inputs

Gather whatever implementation inputs already exist:
- `rtl-design` module-design document, if present
- Parent architecture notes, if they constrain the module
- User-provided requirements or corrections
- Project rule files and surrounding RTL style
- CBB or IP catalog, module list, wrapper docs, or approved reuse inventory

If there is no formal design document, continue by iteratively confirming the missing implementation inputs with the user. A design document is helpful, not mandatory. Implementation-ready information is mandatory.

### Step 2: Run The Implementation Readiness Gate

Before writing RTL, verify that the inputs are sufficient to implement responsibly.

Implementation is ready only when these areas are covered:
- `module boundary`
  One implementation object with a clear responsibility and clear non-goals.
- `interface contract`
  Ports, directions, widths, handshake style, timing semantics, ordering rules, backpressure behavior, flush/error behavior, and parameter ranges.
- `clock/reset/integration assumptions`
  Clock domains, reset behavior, CDC boundaries, DFT-visible behavior, and any inherited integration or timing-constraint assumptions that affect the RTL.
- `microarchitecture-critical decisions`
  Enough detail on FSMs, counters, buffering, storage ownership, pipeline cuts, arithmetic strategy, and hazard handling to code without inventing new architecture.
- `implementation policy`
  Coding style constraints, reset rules, arithmetic restrictions, lint or synthesis constraints, and comment language.
- `reuse constraints`
  Whether CBB/IP reuse is required, preferred, or disallowed.
- `parameterization rules`
  Valid parameter ranges, boundary values such as `WIDTH=1` or `DEPTH=0/1`, and whether overflow, saturation, or wrapping behavior is part of the contract.

Gate behavior:
- If the missing information is local and implementation-specific, ask the user and continue once confirmed.
- If the missing information changes architecture, ownership, or module boundaries, stop and return to `rtl-design`.
- If the user accepts explicit assumptions, carry them into the implementation summary and implementation trace.

Clarification format rules:
- Prefer one focused question when a single item blocks implementation readiness.
- A short list-question format is allowed when several implementation blockers are tightly coupled.
- Keep clarification narrow. Ask only for information that is required to pass the implementation readiness gate, reuse gate, or mandatory-rule checks.
- If the prompt already exposes the local blockers, ask them directly instead of doing broader repository discovery first.
- Do not expand into architecture exploration, speculative design alternatives, or broad requirement interviews inside `rtl-impl`.
- Do not mix clarification questions with RTL code, partial RTL skeletons, or tentative implementation strategy in the same response.
- Do not include recommended default implementation choices in the clarification response unless the user explicitly asks for a recommendation after the blocking questions are listed.

Typical questions this skill should ask when needed:
- Is the comment language for code comments Chinese or English?
- Are there project rules for reset style, naming, arithmetic, or coding templates?
- Are there latency, throughput, or pipeline constraints not captured in the current inputs?
- Are error, flush, or backpressure semantics fully frozen?
- Are there DFT-visible modes, scan assumptions, or timing exceptions that change the implementation boundary?
- For parameterized modules, what ranges and boundary semantics are valid for each exposed parameter?
- Does anyone expect implementation syntax or coding templates that conflict with `references/rtl-rules.md`?

### Step 3: Run The CBB/IP Reuse Gate

Run the reuse gate after implementation readiness is satisfied and before custom coding starts.

1. Determine from the prompt, existing design inputs, or project rules whether this implementation must prefer existing `CBB/IP`.
2. Ask the user about reuse preference only if the current inputs do not already answer it.
3. If reuse is required or preferred, scan the repository for explicit catalogs, IP lists, wrapper docs, or approved reuse inventories.
4. Only use documented reuse candidates. Do not guess capabilities from names alone.
5. If reuse is required but no explicit in-repo list exists, stop and ask the user for the CBB/IP list or catalog.

### Reuse Decision Classes

- `direct reuse`
  The CBB/IP already matches the required function, interface semantics, and key parameters.
- `reuse with wrapper`
  The core function matches, but wrapper RTL is still needed for adaptation, width shaping, protocol bridging, or status handling.
- `custom RTL`
  No suitable documented reuse exists, or the wrapper cost is unjustified. Keep the custom block boundary replaceable.

Reuse rules:
- Prefer mature reuse for FIFO, RAM, ROM, CDC, divider, DSP-heavy arithmetic, or other high-risk reusable structures.
- Use named port connections for all CBB/IP instantiations.
- Do not instantiate a CBB/IP if port or parameter meaning is unclear.
- Record every reuse decision in the implementation summary and implementation trace.

### Step 4: Freeze The Implementation Strategy

Before writing RTL, summarize the implementation plan at implementation granularity.

This summary should freeze:
- module and submodule split
- direct reuse vs wrapper vs custom boundaries
- major FSM, datapath, storage, and pipeline structure
- reset, CDC, and arithmetic strategy
- any user-confirmed assumptions that affect code shape

Keep this lightweight. Do not rewrite the `rtl-design` document. The purpose is to prevent coding drift.

If strategy freeze exposes unresolved local implementation questions, return to Step 2 and close them before coding.
If strategy freeze exposes boundary changes or new architecture work, stop and return to `rtl-design`.

Emit a code skeleton first only when one of these is true:
- the user explicitly asks for it
- the module is large enough that the submodule split is still worth confirming
- multiple valid wrapper/custom split options remain
- the implementation risk is high enough that a brief skeleton review will prevent rework

Otherwise, proceed directly to RTL.

### Step 5: Write Synthesizable RTL

Write RTL that is clear, reviewable, and ready for downstream verification.

Deliver synthesizable `Verilog` RTL only unless the user changes the requirement after you explain the skill boundary conflict.
Follow `references/rtl-rules.md` as the mandatory rule set. Open `references/patterns.md` only after Steps 0 to 4 are complete, and only load the patterns relevant to the current work.

- Use the confirmed coding style and project rules that do not conflict with `references/rtl-rules.md`.
- Keep control, datapath, and wrapper responsibilities explicit.
- Make reuse boundaries obvious in code.
- Do not silently add architecture beyond what was confirmed in the readiness gate.

The RTL should make these decisions explicit:
- FSM ownership and safe default behavior
- data storage ownership and lifetime
- handshake and stall propagation
- latency-bearing pipeline cuts
- reset scope and registers intentionally left unreset
- IP wrapper behavior and adaptation logic

If a coding choice trades off timing, area, power, or replaceability, make that rationale visible in comments or the implementation trace.
If the user revises a frozen assumption during coding, return to Step 2 and close the gap before continuing.

### Step 6: Maintain The Implementation Trace

When RTL is created or materially changed, maintain a lightweight implementation trace document paired with the code. This is for traceability, not for repeating `rtl-design`.

Use the project documentation location when one exists. Otherwise place the trace in the project's normal RTL docs area and keep it short, structured, and diff-friendly.

The trace should record:
- implementation target
- input sources used
- assumptions confirmed with the user
- comment language
- CBB/IP reuse decisions
- key implementation choices that shaped the code
- RTL output files
- verification follow-up points
- open risks or unresolved items

Use `references/implementation_handoff_template.md`.

## Output Delivery

Default delivery includes all of these:

- `RTL implementation`
  Synthesizable `Verilog` code and any needed wrapper modules.
- `implementation summary`
  Short summary of the chosen structure, assumptions, and reuse decisions.
- `implementation trace`
  Lightweight traceability record paired with the code.
- `verification handoff`
  Assertion targets, risky corners, and assumptions that `rtl-verification` should validate.
- `synthesis/integration notes`
  Likely critical paths, CDC points, reset assumptions, or inference-sensitive structures.

## Self-Check Before Delivery

Verify:
- The target really was implementation-ready and not still an architecture problem.
- Clarification, if needed, stayed narrow and implementation-specific rather than expanding into architecture discovery.
- Clarification, if needed, did not smuggle in default implementation choices that the user had not yet confirmed.
- The mandatory rules in `references/rtl-rules.md` were followed.
- Comment language was confirmed with the user.
- Reuse preference was explicitly checked.
- Required CBB/IP documentation existed before reuse was selected.
- The RTL matches the confirmed interface and integration assumptions.
- The implementation trace captures assumptions and reuse decisions without repeating the design document.
- The output is specific enough for `rtl-verification` to continue from the implementation, not to rediscover it.

## When To Stop And Ask For Guidance

Stop instead of guessing when:
- the object is not yet a stable implementation boundary
- interface semantics are still changing
- the user or project insists on coding requirements that conflict with `references/rtl-rules.md`
- the user tries to bypass a required readiness, reuse, or strategy gate
- comment language has not been confirmed
- reuse is required but no explicit CBB/IP inventory is available
- a project rule conflicts with the current inputs
- a critical arithmetic, CDC, or memory decision is still ambiguous

The default failure mode of this skill is to ask a small number of targeted implementation questions, not to invent missing requirements.
