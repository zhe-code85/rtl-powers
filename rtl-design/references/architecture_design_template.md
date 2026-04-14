# Architecture Design Template

Use this template in `architecture mode`.

## 1. Scope And Context

### 1.1 Design Object
<!-- Name the subsystem or block being designed and state its role in the parent system. -->

### 1.2 Functional Scope
<!-- State what this object must do and what is explicitly out of scope. -->

### 1.3 Feature Summary
<!-- Table: Feature | Source | Status | Current Owner | Design Impact
     Source is one of: user requirement, parent architecture, project constraint.
     Status is one of: required, optional, deferred, not supported, TBD.
     Current Owner identifies whether the feature is owned by this object, a child block, the parent system, or is still unassigned.
     Design Impact should name the main affected area such as layering, interface contract, storage, timing, buffering, error handling, or verification. -->

### 1.4 External Interfaces And Contracts
<!-- Table: Interface | Protocol | Direction | Key Signals | Throughput/Latency Contract | Notes -->

### 1.5 Non-Functional Constraints
<!-- Frequency, area, power, timing margin, buffering limits, ordering or QoS constraints. -->

### 1.6 Inherited Dependencies And Assumptions
<!-- Clocking, reset, test modes, power-domain assumptions, macro boundaries, or integration constraints inherited from the project or parent block. -->

### 1.7 Open Questions
<!-- Unknowns that could change decomposition or interface contracts. -->

## 2. Layered Architecture

### 2.1 Layer Map
<!-- Prefer a Mermaid flowchart or a compact ASCII layered stack.
     Describe the protocol/config layer, core control layer, and datapath/storage layer. -->

### 2.2 Flow Map
<!-- Prefer a Mermaid flowchart or a compact ASCII data-flow sketch.
     Map ingress, schedule, and egress onto the layer structure. -->

### 2.3 Partition Rationale
<!-- For each major child block, explain why it is separate. Use reasons such as protocol isolation, scheduling complexity, storage ownership, timing closure, or verification clarity. -->

## 3. Recursive Decomposition

### 3.1 Module Tree
<!-- Tree or table: Node | Parent | Primary Role | Status (ready for module-design / needs further architecture / TBD) -->

### 3.2 Child Block Summary
<!-- Table: Block | Layer | Flow Role | Responsibility | Key Interfaces | Budget/Performance Target | Status -->

### 3.3 Interface Contract Matrix
<!-- Table: Source Block | Destination Block | Interface/Protocol | Ordering/Backpressure Rules | Error/Flush Behavior | Notes -->

### 3.4 Budget Allocation
<!-- Table: Block | Area Budget Share | Timing Budget/Target | Latency Budget | Buffering Budget | Allocation Rationale
     Use provisional numbers or relative weights if exact figures are not yet frozen. -->

### 3.5 Domain-Boundary Summary
<!-- Table: Boundary | Type (clock/reset/power/test) | Source Side | Destination Side | Architectural Impact | Status
     Include suspected boundaries as TBD when they may force wrappers, buffering, isolation, retention handling, or ownership splits. -->

## 4. Integration-Critical Design Decisions

### 4.1 Scheduling And Control Model
<!-- Describe arbitration, ownership, fairness, ordering, replay, flush, retry, or recovery behavior at architecture level. -->

### 4.2 Datapath And Storage Boundaries
<!-- State where buffering, storage ownership, transformations, or width adaptation occur. -->

### 4.3 Timing-Sensitive Boundaries
<!-- Identify likely hot paths, cut points, buffering points, and any boundary chosen primarily for implementation convergence. -->

### 4.4 Integration Assumptions
<!-- Record reset, test, clocking, power, and physical assumptions that affect this architecture without redesigning those systems. -->

## 5. Verification Guidance

### 5.1 Architecture-Level Invariants
<!-- List properties that must hold across child blocks and interfaces. -->

### 5.2 Corner And Illegal Cases
<!-- List architecture-visible corner cases, illegal conditions, flush/retry paths, and recovery behavior. -->

### 5.3 Observability And Debug Hooks
<!-- Status outputs, counters, state visibility, error indicators, or readback points that verification and debug should rely on. -->

### 5.4 Verification Handoff Summary
<!-- Table: Item Type | ID/Name | Description | Scope | Expected Check
     Item Type is one of: invariant, corner, illegal, observability, risk-focus.
     This table is the structured handoff to rtl-verification. -->

## 6. Next-Step Handoff

### 6.1 Blocks Ready For Module Design
<!-- List child blocks that can move to module-design mode next. -->

### 6.2 Blocks Requiring More Architecture
<!-- List nodes that still need deeper decomposition and why. -->

### 6.3 Risks
<!-- Table: Risk | Likelihood | Impact | Fallback | Affected Blocks -->
