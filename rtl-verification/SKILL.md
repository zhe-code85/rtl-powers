---
name: rtl-verification
description: Analyze RTL design weaknesses and create comprehensive verification plans using cocotb (primary) or Verilog testbenches. Use when verifying RTL, writing testbenches, planning verification, checking coverage, testing a module, or ensuring design correctness. Also use when user asks about testbench creation, stimulus generation, assertion writing, or coverage closure. Triggers on verification, testing, testbench, coverage, assertions, cocotb, or any request to validate RTL behavior.
---

# RTL Verification

## Verification Philosophy

Verify design intent, not implementation details. A testbench that checks internal signals is fragile; one that checks observable behavior is resilient to RTL refactoring.

Design weakness analysis drives verification planning. Do not write tests first and hope they cover the risks. Analyze the RTL for structural weaknesses, rank them by severity, then build a verification matrix that targets every identified risk.

Functional correctness is necessary but not sufficient. A module that passes all functional tests may still fail timing, consume excessive area, or break under parameter corner cases. Verification plans must include boundary and stress tests that expose these classes of failure.

## Step 0: Design Weakness Analysis (Mandatory Pre-Planning)

Read the RTL before planning any tests. Every verification decision flows from what the design actually does and where it is vulnerable.

### What to Read

- Module port list: identify clock domains, reset domains, handshake interfaces
- Parameter declarations: spot boundary values (DEPTH=1, WIDTH=0, RATIO=1)
- FSM definitions: enumerate states, transitions, default/unused handling
- Datapath structure: pipeline stages, FIFO depth, backpressure paths
- Clock domain crossings: synchronizer chains, Gray code converters, handshake CDC
- Reset logic: sync vs async assert/deassert, partial resets, reset-less domains

### Structural Weakness Detection

Check for each pattern below. Document every finding with the signal or module location and risk level.

**CDC Risk (High if present)**
- Signal crossing clock domains without a 2+ stage synchronizer
- Multi-bit bus crossing domains without Gray coding or handshake protocol
- Pulse signals crossing domains without a pulse synchronizer
- CDC feedback loops or req/ack with no maximum latency bound

**Reset Domain (High if inconsistent)**
- Async reset release not synchronized to local clock (recovery violation)
- Partial reset that leaves some registers in undefined state while others clear
- Counters or state registers missing reset (intentional vs oversight)
- Reset active during normal operation (glitch susceptibility)

**FSM Hazards (High if functional, Medium if recovery)**
- Missing default state or default next-state assignment
- Non-mutually-exclusive transition conditions (onehot encoding collision)
- No idle/error recovery path from unreachable states
- Output glitches during state transitions (registered vs combinational output)
- Uncovered input condition combinations (partial case/casex coverage)

**Datapath Hazards (High if data loss possible)**
- FIFO or buffer overflow: write when full without proper backpressure stall
- FIFO or buffer underflow: read when empty producing garbage data
- Backpressure path breakage: valid/ready or almost-full not connected through pipeline
- Width truncation: arithmetic result wider than destination without saturation check
- Incomplete stall or flush logic: pipeline stall not propagated to all stages

**Arbitration and Contention (High if deadlock, Medium if starvation)**
- Round-robin or priority arbiter with possible starvation under sustained requests
- Concurrent access to shared resource without mutual exclusion
- Request/grant protocol with potential deadlock cycles (circular dependency)
- Fairness not verified under asymmetric load patterns

**Edge Case Traps (Medium if functional, Low if cosmetic)**
- Counter wraparound: does the design handle 0xFFF -> 0x000 correctly?
- Parameter boundary: DEPTH=1 (single-entry buffer), WIDTH=0 (null bus), RATIO=1 (no divide)
- Simultaneous set and reset on a register (last-write-wins vs undefined)
- Zero-length burst or packet (empty payload path)
- Power-of-2 vs non-power-of-2 depth arithmetic

### Risk Prioritization

- **High**: Can cause functional error, deadlock, or data loss under normal operating conditions. Requires directed test plus assertion.
- **Medium**: Can cause data corruption or incorrect behavior under specific conditions. Requires boundary or corner test.
- **Low**: Occurs only in rare edge cases or parameter combinations. Requires at least one directed scenario.

Feed the complete weakness list into the verification matrix in Step 1.

## Step 1: Verification Planning

### Build the Verification Matrix

Organize tests in escalating levels. Each level has a distinct purpose and closure criterion.

| Level | Purpose | Example | Closure |
|-------|---------|---------|---------|
| Sanity | DUT compiles, resets to known state, basic IO works | Reset brings all outputs to idle | Pass in 10 cycles |
| Directed | Target specific weaknesses with known stimulus | CDC handshake completes across domains | Known-good response per test |
| Boundary | Test at parameter and arithmetic limits | FIFO at full depth, counter at max value | Correct behavior at edges |
| Corner | Combine edge conditions simultaneously | Full FIFO during backpressure with CDC | No deadlock, no data corruption |
| Stress | Sustained operation with random or maximal stimulus | 10K random transactions at max throughput | Zero errors in sustained run |

### Map Weaknesses to Tests

For each weakness from Step 0:
- Every High-risk weakness gets at least one directed test AND one assertion.
- Every Medium-risk weakness gets at least one boundary or corner test.
- Every Low-risk weakness gets at least one directed scenario.
- Overlapping weaknesses can share a test, but each weakness must be explicitly covered.

### Closure Criteria

Define what "done" means before writing tests:
- All directed tests pass for the target DUT configuration.
- All assertions hold across every test level.
- Functional coverage hits target for all spec invariants.
- Code coverage: statement >95%, branch >90%, FSM 100% state+transition.
- No unresolved High or Medium risk items.

## Step 2: Cocotb Testbench Architecture (Primary Framework)

Use cocotb for all non-trivial verification. Its Python-based coroutine model, rich ecosystem, and pytest integration make it the best choice for structured verification.

### Project Structure

```
tb/
├── Makefile              # cocotb build/run entry point
├── conftest.py           # pytest configuration for cocotb
├── test_module.py        # test cases grouped by verification level
├── tb_driver.py          # stimulus drivers (cocotb coroutines)
├── tb_monitor.py         # output observers
├── tb_scoreboard.py      # expected vs actual comparison
├── tb_refmodel.py        # reference model (Python)
└── hdl/
    └── tb_wrapper.v      # optional Verilog wrapper for DUT instantiation
```

### When to Use Lightweight vs cocotb-bus

- **Lightweight** (recommended starting point): hand-write Driver/Monitor as plain coroutines. Use for simple interfaces, custom protocols, or when you want zero framework overhead.
- **cocotb-bus**: use for standard bus interfaces (AXI, APB, Wishbone) where the bus driver abstraction provides real value. Adds dependency but handles burst, protocol checking, and interleaving.

See `references/cocotb_patterns.md` for complete working examples of both approaches.

### Stimulus Strategy

1. Start with directed tests targeting each weakness. Each test is a standalone coroutine.
2. Add constrained-random stimulus using Python `random` or hypothesis for coverage holes.
3. For stress tests, use cocotb `fork` (or `start_soon`) to run concurrent stimulus generators and checkers.
4. Always pair stimulus with a checker. Unguarded stimulus proves nothing.

### Reference Model Integration

Build a Python reference model that mirrors DUT behavior at the transaction level:
- Same inputs as DUT (fed by shared Driver)
- Produces expected outputs for Scoreboard comparison
- Does not need cycle-accuracy for most checks; functional equivalence suffices
- For cycle-accurate checks, use phase-aligned monitors

### Regression with pytest

Integrate with pytest for parameterized regression, structured reporting, and CI compatibility. See `references/cocotb_patterns.md` Section 5 for the full setup.

## Step 3: Verilog Testbench Fallback

Use Verilog testbenches only when cocotb is unavailable or for quick sanity checks on simple DUTs. A Verilog testbench is better than no testbench, but it lacks the structural advantages of cocotb (coroutine concurrency, Python ecosystem, pytest integration).

### When to Choose Verilog TB

- cocotb is not installed and cannot be installed
- DUT is a simple combinational block or single-clock single-domain module
- Quick sanity check needed in under 5 minutes
- Project mandates Verilog-only testbenches

### Self-Checking Pattern

Structure the testbench around tasks:
- `task apply_reset`: deassert reset with known timing
- `task apply_stimulus`: feed one test vector
- `task check_output`: compare actual vs expected, flag PASS/FAIL
- `task run_all_tests`: sequential loop over test vectors with pass/fail counter
- Final `$display` summary with pass count and fail count

### Migration Path to Cocotb

When the Verilog TB grows beyond 200 lines or needs concurrent stimulus, migrate:
1. Keep the test vectors and expected results (they are data, not framework)
2. Replace task-based stimulus with cocotb coroutines
3. Replace `$display` checks with Python assert or Scoreboard
4. Add concurrent stimulus that was impossible in Verilog tasks

## Step 4: Assertion Strategy

Assertions are the most efficient way to catch bugs because they fire at the exact cycle the violation occurs, rather than propagating errors downstream.

### Placement Principle

- **Boundary assertions**: check interface protocols at DUT boundaries (valid/ready, FIFO full/empty, CDC handshake). These catch integration bugs.
- **Invariant assertions**: check internal properties that must always hold (one-hot encoding, mutual exclusion, range limits). These catch logic bugs.
- **Protocol assertions**: check sequence-level behavior (burst length, handshake ordering, response latency). These catch timing bugs.

### SVA in RTL (Primary)

Place SystemVerilog Assertions inside or beside the RTL module. They compile with the design, require no testbench, and fire in every simulation. See `references/assertion_patterns.md` for complete examples.

Guidelines:
- One assertion per invariant. Do not combine unrelated checks.
- Use `cover` properties alongside `assert` to confirm the assertion is actually exercised.
- Use `$rose`/`$fell`/`$past` for temporal checks, not for combinational ones.
- Guard assertions with reset: most assertions should disable during reset using `disable iff (!rst_n)`.

### Cocotb Checkers (Supplementary)

Use Python-side checkers for properties that are awkward in SVA:
- Multi-cycle transaction-level checks (packet integrity, reorder verification)
- Statistical checks (throughput measurement, latency distribution)
- Checks requiring reference model comparison

See `references/assertion_patterns.md` Section 3 for cocotb checker patterns.

### Assertion Density

Too many assertions is as bad as too few. Noisy assertions that fire every cycle train engineers to ignore them. Follow these rules:
- Assertions should fire only on violation, never on correct behavior.
- Avoid testing implementation details (internal signal values that could change in a refactor).
- Avoid over-specified timing assertions that break with retiming or pipelining changes.

## Step 5: Coverage Strategy

Coverage quantifies how thoroughly the design has been exercised. Without coverage targets, you do not know when verification is complete.

### Functional Coverage

Derive coverage points directly from spec invariants and design weakness analysis:
- Each FSM state visited (coverpoint on state register)
- Each handshake direction exercised (valid without ready, ready without valid, simultaneous)
- Each FIFO condition (empty, full, almost-empty, almost-full, partial)
- Each parameter boundary value exercised
- Each arbitration outcome (each requestor granted, idle, simultaneous request)

### Cocotb Coverage API

Use `cocotb.coverage` to define cross-coverage and transaction-level coverage that is awkward in SystemVerilog covergroups:
- Define `CoverPoint` and `CoverCross` in Python
- Sample in monitors after each transaction
- Report coverage at test end

### Code Coverage

Run code coverage as a safety net, not a goal:
- Statement coverage: target >95%. Below 90% indicates untested logic paths.
- Branch coverage: target >90%. Uncovered branches may be dead code or missing tests.
- Toggle coverage: useful for identifying undriven or unread signals.
- FSM coverage: target 100% state and transition. Any uncovered state or transition is a test gap.

Code coverage is misleading when:
- Dead code inflates coverage gaps (remove dead code, do not test it)
- Redundant logic hits coverage targets without testing real behavior
- Coverage is high but functional coverage is low (tested code, wrong scenarios)

### Closure Criteria

Coverage closure is achieved when:
- All functional coverage points from the spec are hit
- Code coverage meets targets (with justified waivers for unreachable code)
- All High-risk weakness assertions pass under stress conditions
- Zero unresolved assertion failures across the full regression

## Blind Spots and Risk Assessment

After completing the verification plan, explicitly state what it does NOT cover:

- Asynchronous timing hazards (metastability, recovery/removal): simulation does not model real metastability. Use SDC constraints and CDC tools for these.
- Analog boundary effects: mixed-signal interfaces need separate verification.
- Gate-level behavior: post-synthesis simulation may reveal different behavior due to optimization.
- Power intent: UPF/CPF power domain transitions are not verified by functional tests.
- Parameter combinations not explicitly tested: if the design has 5 parameters with 3 values each, you cannot test all 243 combinations. Document which ones are tested and which are waived.

State residual risk explicitly. A verification plan without stated blind spots is dishonest.

## Output Format

Produce two artifacts for every verification engagement:

### 1. Weakness Report

```
## Design Weakness Report: [module_name]

### Summary
- High risk: [count]
- Medium risk: [count]
- Low risk: [count]

### Findings
| ID | Category | Location | Description | Risk | Mitigation |
|----|----------|----------|-------------|------|------------|
| W1 | CDC      | sig_x    | Missing synchronizer | High | Add 2-stage sync + assertion |
```

### 2. Verification Plan

```
## Verification Plan: [module_name]

### Testbench Framework
- Primary: cocotb + [simulator]
- Fallback: Verilog TB (if applicable)
- Reference model: [yes/no, description]

### Verification Matrix
| Test ID | Level | Target Weakness | Description | Pass Criteria |
|---------|-------|-----------------|-------------|---------------|
| T1 | Sanity | -- | Reset brings DUT to idle | All outputs = idle after reset |
| T2 | Directed | W1 | CDC handshake | Correct data transfer across domains |

### Assertion Inventory
| Assert ID | Type | Location | Property |
|-----------|------|----------|----------|
| A1 | Boundary | valid/ready | valid should not deassert before ready |

### Coverage Targets
- Functional: [list of coverpoints]
- Code: statement >95%, branch >90%, FSM 100%

### Residual Risk
- [stated blind spots and untested conditions]
```
