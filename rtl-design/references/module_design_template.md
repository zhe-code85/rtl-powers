# Module Design Template

Use this template in `module-design mode`.

## 1. Scope And Parent Context

### 1.1 Module Identity
<!-- Name the module and state its parent architecture document or parent block. -->

### 1.2 Responsibility And Non-Goals
<!-- State what this module owns and what remains outside its boundary. -->

### 1.3 Required Parent Contracts
<!-- List the parent-level ordering, latency, buffering, configuration, error, or backpressure contracts this module must satisfy. -->

### 1.4 External Interfaces
<!-- Table: Port/Interface | Protocol | Direction | Width | Timing Semantics | Description -->

### 1.5 Parameters And Configuration
<!-- Table: Parameter | Default | Valid Range | Effect On Behavior -->

### 1.6 Assumptions And Open Questions
<!-- Inherited reset/test/clock/power assumptions and any remaining unknowns. -->

## 2. Microarchitecture

### 2.1 Block Diagram
<!-- ASCII art or Mermaid diagram of the module and any local child blocks. -->

### 2.2 Local Child Blocks
<!-- Table: Block | Type | Responsibility | Key Interfaces | Notes -->

### 2.3 Control Definition
<!-- For each FSM or scheduler, provide:
     1. State table: State | Role | Entry Condition | Exit Condition
     2. Mermaid stateDiagram-v2 if readable, otherwise a compact ASCII state diagram
     3. Transition notes for non-obvious arbitration, retry, flush, or recovery behavior
     Also describe counters, scoreboards, ownership rules, and control-state transitions. -->

### 2.4 Datapath And Storage Organization
<!-- Describe transformations, storage ownership, buffering, hazard handling, and data availability rules. -->

### 2.5 Timing And Pipeline Plan
<!-- Use a table when possible:
     Stage/Path | Source Boundary | Combinational Work | Preferred Cut Point | Added Latency | Why This Cut Exists
     Also describe hot paths, backpressure propagation, buffering rationale, and any architecturally safe multi-cycle behavior. -->

### 2.6 CDC, Reset, Test, And Integration Assumptions
<!-- State only the assumptions that affect this module. Do not redesign platform-level policy here. -->

### 2.7 Implementation-Critical Boundaries
<!-- Deferred implementation notes, structure classes needed, coding-sensitive boundaries, and anything rtl-impl must not reinterpret. -->

## 3. Verification Guidance

### 3.1 Invariants
<!-- Assertions or properties that must always hold. -->

### 3.2 Corner Cases
<!-- Boundary conditions, stalls, flushes, retries, empty/full transitions, or mode changes. -->

### 3.3 Illegal Cases And Error Handling
<!-- Unsupported inputs, protocol violations, recovery behavior, and error visibility. -->

### 3.4 Observability And Debug Hooks
<!-- State visibility, counters, status bits, error flags, or trace points needed for bring-up and debug. -->

### 3.5 Verification Handoff Summary
<!-- Table: Item Type | ID/Name | Description | Trigger/Condition | Expected Check
     Item Type is one of: invariant, corner, illegal, observability, risk-focus.
     This table is the structured handoff to rtl-verification. -->

## 4. Trade-Offs And Risks

### 4.1 Key Trade-Offs
<!-- Area/performance/power/latency trade-offs with rationale. -->

### 4.2 Risks
<!-- Table: Risk | Likelihood | Impact | Fallback | Verification Focus -->
