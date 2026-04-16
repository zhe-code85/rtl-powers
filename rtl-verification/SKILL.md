---
name: rtl-verification
description: Use when verifying RTL modules or blocks, building cocotb or Verilog testbenches, creating verification plans, analyzing design risk, defining assertions or coverage, or checking regression closure for RTL behavior. Also use when the user says things like "write a testbench for this module", "how do I verify this block", "what is not tested yet", "add assertions", "analyze this failing regression", or "help close verification for this RTL".
---

# RTL Verification

Verify design intent, not implementation trivia. Start from the DUT's externally visible contract, identify structural weaknesses, then build tests, assertions, and coverage that close those risks.

- `rtl-design` owns architecture, module boundaries, and design intent.
- `rtl-impl` owns implementation structure and RTL delivery.
- `rtl-verification` owns weakness analysis, verification planning, testbench strategy, assertion strategy, and closure.

Default preference is cocotb, but the final framework choice follows Step 1 environment discovery. Use a Verilog testbench only when cocotb is unavailable, the DUT is trivial, or project constraints require it. Open `references/cocotb_patterns.md` only when building cocotb collateral. Open `references/assertion_patterns.md` only when you need concrete assertions, checkers, or coverage patterns. Open `references/verification_planning.md` when you need detailed case decomposition, test allocation, or output templates. If the project mandates UVM or formal property verification, use this skill for weakness analysis, verification planning, assertion intent, and closure triage, then hand off the UVM- or FPV-specific implementation to the project flow.

For runnable verification flows, follow the artifact and waveform rules in Step 6. Use one retained compressed waveform per executed case and do not save waveforms as `VCD`.

## Default Flow

### Step 0: Confirm The Verification Target

Start by identifying the exact verification object.

Continue only if all of these are true:
- The target is a stable module, block, or wrapper boundary.
- External interface semantics are frozen enough to verify.
- The user wants verification analysis, a testbench, a verification plan, or closure work.

Stop and return to `rtl-design` or `rtl-impl` if any of these are still moving:
- Port list, ordering rules, handshake semantics, flush semantics, or ownership rules.
- Major microarchitecture changes that would invalidate the planned verification intent.
- Undefined parameter ranges or reset behavior that prevent meaningful testbench design.

### Step 1: Discover The Verification Environment

Run this step only when the request requires runnable collateral, regression execution, or a concrete framework choice. For `analysis-only` or planning-only work, record simulator and framework as `not required yet` unless the user explicitly asks for an executable environment.

Runnable collateral assumes the project verification environment is already activated in the current shell:
- Use the currently activated project environment for `python`, `pytest`, `cocotb`, and simulator tools.
- Do not hardcode machine-local `venv` paths into the harness, generated commands, or skill output just to make the current shell work.
- If `python`, `pytest`, `cocotb`, and simulator tools resolve from inconsistent environments, or required tools are missing because the verification environment is not activated, stop and tell the user to activate the project verification environment first.

Treat prompt-provided environment facts as working inputs:
- If the prompt explicitly says an existing `Makefile`, `pytest` entry, reusable harness, simulator, or framework is available, use that as the current planning baseline first.
- Do not start by disproving prompt-provided setup facts through broad repository existence searches unless the user explicitly asks for repository reconciliation or the mismatch blocks a concrete file edit in the current tree.
- When local repository state and prompt-provided setup facts differ, say so only after you have first answered the requested planning or verification question using the prompt's stated scenario.

When this step applies, discover the actual execution environment in this order:
1. existing project harness: check `Makefile`, `pytest`, `conftest.py`, CI scripts, wrapper TB files, README commands, and simulator launch scripts
2. harness decision: if a usable harness already exists, extend that harness and keep its simulator/framework path unless it is broken or conflicts with an explicit project rule
3. lightweight environment discovery: if no usable harness answers the question, use direct command or environment checks instead of writing a new discovery script
4. decision and recording: when a new runnable path must be chosen, use the priority rules below and record the result in the verification plan

Treat existing harness evidence carefully:
- A repository-level `Makefile`, `pytest` entry, or reusable `cocotb` harness is evidence to extend the current flow, not a reason to build a second parallel runner.
- Existing harnesses should stay environment-agnostic where possible. Prefer `python`, `python3`, `pytest`, and simulator names that rely on the activated project environment rather than embedding a personal `venv` path.
- An existing harness may fix the flow shape without fixing the simulator backend. If the harness is simulator-agnostic or configurable, keep the harness and still apply the simulator priority rules below.
- If the current harness already uses `cocotb`, keep `cocotb` unless it is broken or an explicit project rule requires a different framework.
- For example: existing `Makefile` plus `pytest` plus extensible `cocotb` harness, with both `VCS` and `Verilator` available, means reuse that harness and choose `VCS` as the simulator backend rather than creating a new flow or defaulting to `Verilator`.

Apply these priority rules:

- Simulator
  When no usable harness already fixes the simulator choice, prefer `VCS` when it is available in the project environment or scripts.
  If `VCS` is not available, fall back to `Verilator`.
  If neither simulator is available, stop and tell the user that no supported simulator was found.
- Testbench framework
  When no usable harness already fixes the framework choice, check whether `cocotb` is available in the environment or already used by the project after the simulator is chosen.
  If `cocotb` is available, use it with the chosen simulator.
  If `cocotb` is unavailable, fall back to a hand-written self-checking Verilog testbench.

Record the chosen environment in the plan:
- simulator: `VCS` or `Verilator`
- framework: `cocotb` or `Verilog TB`
- reuse: whether an existing harness can be extended

### Step 2: Collect Verification Inputs

Gather whatever inputs already exist:
- Feature list from the spec, requirements doc, module-design doc, or user-provided behavior notes
- RTL source for the DUT and any required wrappers
- `rtl-design` document, if present
- `rtl-impl` implementation trace or handoff, if present
- User-provided requirements, bug reports, or corner cases
- Existing testbench files, assertions, coverage reports, and regression logs
- Project simulator and framework constraints

Start with the feature list before anything else:
- Read the requirements or design document and extract the explicit feature list.
- If no formal feature list exists, derive one from the DUT contract and user-provided behavior.
- For each feature, write one `positive` example and one `adverse` example.
- `positive` example: the feature is exercised correctly and should succeed.
- `adverse` example: the feature is violated, blocked, saturated, or driven with an invalid condition, and the DUT should reject it, hold state, raise an error, apply backpressure, or otherwise behave safely according to the contract.
- If a feature depends on a timer, watchdog, response window, or wait counter, its `adverse` example must include a timeout scenario.
- If a feature truly has no meaningful adverse case, record that waiver explicitly in the plan instead of silently omitting it.

Treat this feature list plus its positive and adverse examples as a first-class verification input. The feature list answers "what behavior must exist"; the weakness analysis answers "where the implementation is likely to break".

If the request is to write tests, do not skip input collection. Verification without a stable contract becomes cargo-cult stimulus.

For `analysis-only` or planning-only requests:
- Deliver the weakness analysis and verification plan directly from the available DUT contract, prompt facts, and design clues.
- If local RTL or design documents are partial or missing, continue with a bounded plan based on the available contract clues and state the missing inputs explicitly.
- Do not turn the response into repository-status commentary, skill-maintenance commentary, or environment-discovery narration.

### Step 3: Run Design Weakness Analysis

Read the RTL before planning tests. Every verification decision should trace back to what the RTL actually does and where it is vulnerable.

Inspect at least these areas:
- Module port list: clock domains, reset domains, handshake interfaces
- Parameters: boundary values such as `DEPTH=1`, `WIDTH=1`, `RATIO=1`
- FSM structure: states, transitions, default handling, recovery paths
- Datapath: pipeline cuts, buffer ownership, backpressure paths, truncation points
- CDC structures: synchronizers, handshakes, Gray coding, pulse transfers
- Reset structure: sync vs async behavior, partial reset, intentionally unreset registers

Document every weakness with location and risk level.
For each weakness, capture at least:
- ID
- category
- location
- failure symptom
- risk level
- planned verification hook such as directed test, assertion, checker, or coverage point

Look especially for these weakness classes:
- `CDC risk`: missing synchronizer, unsafe multi-bit crossing, pulse transfer hazard, unbounded req/ack progress
- `Reset risk`: unsynchronized release, partial reset ambiguity, unintentionally unreset state, reset glitching into normal operation
- `FSM hazard`: missing recovery path, non-exclusive transitions, combinational output glitch risk, missing timeout or wait recovery
- `Datapath hazard`: overflow or underflow, incomplete backpressure, implicit truncation or sign handling, incomplete stall or flush coverage
- `Arbitration risk`: starvation, missing mutual exclusion, deadlock-prone request/grant sequencing
- `Edge-case trap`: wraparound, zero-length behavior, power-of-2 versus non-power-of-2 divergence, simultaneous control priority, timeout boundary ambiguity

Risk prioritization:
- `High`: can cause functional error, deadlock, or data loss in normal operation
- `Medium`: can corrupt data or break behavior under specific boundary conditions
- `Low`: rare edge case or parameter corner that still deserves at least one directed scenario

Feed the full weakness list into Step 4. Do not write tests first and hope the list emerges later.

### Step 4: Build The Verification Plan

Turn the weakness list into a verification matrix before writing collateral.

Use these levels:

| Level | Purpose | Typical Goal |
| --- | --- | --- |
| Sanity | Compile, reset, basic IO legality | DUT enters known idle behavior |
| Directed | Target a specific weakness | One risk, one deterministic scenario |
| Boundary | Exercise limits and extrema | Max/min parameters, empty/full, wraparound |
| Corner | Combine multiple adverse conditions | No deadlock or corruption under overlap |
| Stress | Sustained or randomized operation | Stable behavior over long regressions |

Map every weakness explicitly:
- Every `High` weakness gets at least one directed test and one assertion.
- Every `Medium` weakness gets at least one boundary or corner test.
- Every `Low` weakness gets at least one directed scenario.
- Shared tests are allowed, but coverage must still be traceable per weakness.

Map every feature explicitly:
- Every feature from Step 2 gets at least one `positive` scenario.
- Every feature from Step 2 gets at least one `adverse` scenario, unless the contract explicitly states there is no meaningful invalid, blocked, or stressed case.
- A single test may cover both a feature example and a weakness, but both mappings must remain visible in the plan.

Plan parameterized verification explicitly:
- Separate smoke points from boundary points and cross-product points.
- Do not try to brute-force every parameter combination unless the space is tiny.
- Pick representative tuples that cover minimum, maximum, singleton, power-of-2, and non-power-of-2 cases when relevant.
- Record which parameter combinations are executed and which are intentionally deferred.
- Record test allocation explicitly: which scenario goes into which test file or test bucket, and why it belongs there.

Define closure before implementation:
- Directed tests for the chosen DUT configuration pass.
- Assertions hold across all executed levels.
- Functional coverage hits the intended invariants.
- Code coverage targets are either met or waived with justification.
- No unresolved `High` or `Medium` risk item remains hidden.

### Step 5: Plan Assertions And Coverage

Assertions and coverage are part of the plan, not optional polish.

Plan assertions, coverage, and reference models together:
- Use assertions for handshake legality, invariants, protocol ordering, bounded latency, mutual exclusion, range limits, and CDC protocol safety.
- Prefer SVA in or beside RTL when the property is naturally expressed there. Use Python-side checkers for transaction-level or statistical properties that are awkward in SVA.
- If the project uses FPV, mark which assertions are simulation-only checks and which are suitable formal properties.
- Derive functional coverage from the spec and weakness list, not from whatever is easy to sample. Treat code coverage as a safety net, not the definition of correctness.
- Use cover properties, covergroups, or Python-side coverage counters when they help prove that a protocol mode, timeout path, parameter corner, or adverse scenario was actually exercised.
- Require a reference model when direct expected-value checking becomes fragile, such as arithmetic transforms, protocol translation, reordering, or multiple legal interleavings. Prefer a lightweight Python reference model in cocotb flows.

Use `references/assertion_patterns.md` when you need concrete SVA assertions, cocotb checkers, or coverage patterns.

Use `references/verification_planning.md` for detailed case decomposition, test allocation, and output template patterns.

### Step 6: Build Verification Collateral In The Chosen Environment

Run this step only when the request includes executable collateral.
If Step 1 was skipped because the request is `analysis-only` or planning-only, skip this step as well.
Use the environment chosen in Step 1. Do not revisit the framework choice unless the discovered environment is incomplete or broken.

For a `cocotb` flow:
- Start with lightweight hand-written drivers and monitors unless a standard bus library provides real value.
- Pair every stimulus source with a checker, scoreboard, or reference model.
- Build the testbench to satisfy the assertions, feature scenarios, and coverage targets already defined in Steps 4 and 5.
- Use `references/cocotb_patterns.md` for directory structure, regression setup, and reusable patterns.

For a `Verilog TB` fallback:
- Keep the testbench self-checking with tasks for reset, stimulus, output checking, and summary reporting.
- Implement the same planned scenarios and checks, even if the framework is lighter.
- Treat it as a fallback, not the default growth path.

Artifact and waveform rules for executable collateral:
- Allocate one artifact directory per executed case. Keep waveform, log, seed, and repro command for that case in the same directory.
- Preserve case isolation. One case directory should correspond to one primary verification question and one primary failure story.
- Keep waveform retention on for every executed case, not only for failing cases or manually selected debug reruns.
- Do not drop waveforms for passing cases just to save space. If retention cost is too high, reduce waveform scope, time window, or compression settings without changing the per-executed-case retention rule.
- Choose a compressed waveform format that fits the selected simulator and project flow:
  - prefer `FSDB` or `VPD` in `VCS`-based flows when supported by the project environment
  - prefer `FST` in `Verilator`-based flows
- If the existing harness already has a compressed waveform format, extend it instead of introducing a second waveform convention.
- If the current harness writes `VCD`, update the harness or run configuration to a compressed waveform format before adding new runnable cases.

### Step 7: Deliver Verification Artifacts

Default delivery depends on the request:

- `analysis-only`
  Deliver a weakness report and a verification plan. Runnable simulator or framework setup is optional unless the user explicitly asks for it. The response should be the analysis and plan themselves, not commentary about editing the skill or changing repository rules.
- `planning + implementation`
  Deliver the weakness report, verification plan, the requested cocotb or Verilog collateral, and per-case artifact directories with retained compressed waveforms for every executed case.
- `closure or debug follow-up`
  Deliver updated risk status, failing or passing checks, and remaining residual risk.

When debugging a failing regression or checker:
1. Localize the failing test, assertion, or checker and identify the first observable divergence.
2. Map the failure back to the verification matrix and the related weakness or feature.
3. Classify the issue as a testbench bug, design bug, or contract gap.
4. If it is a testbench bug, repair the collateral and keep the original intent unchanged.
5. If it is a design bug, return the failing evidence and affected intent to `rtl-impl`.
6. If it is a contract gap or interface ambiguity, return to `rtl-design`.
7. Update the weakness report, plan, and residual-risk summary so closure status stays current.

Always state what is not covered:
- Real metastability behavior and async timing hazards
- Analog or mixed-signal boundary effects
- Gate-level optimization effects unless specifically simulated
- Power-intent behavior unless UPF/CPF verification is in scope
- Untested parameter combinations

## Output Delivery

Default outputs are:
- weakness report
- verification plan
- executable collateral when requested
- residual-risk summary

Use `references/verification_planning.md` for detailed report and plan templates.

## When To Stop And Ask For Guidance

Stop instead of guessing when:
- The DUT contract is still changing
- The request needs executable collateral, regression execution, or a concrete framework choice and the required simulator or framework support is still unknown
- The verification request depends on unresolved design intent
- The user asks for assertions or checks that overfit unstable implementation details
- Parameter legality or reset behavior is too ambiguous to verify responsibly
- The project requires waveform preservation but the harness cannot yet emit a compressed waveform format

The default failure mode of this skill is to surface risk and ask for the missing verification input, not to fabricate a false sense of coverage.
