# Verification Planning Patterns

Use this file when building the verification matrix, decomposing cases, assigning cases to buckets or files, or writing the weakness report and verification plan outputs.

## Contents

- `Test Case Decomposition`
- `Test Case Allocation And Evolution`
- `Output Templates`

## Test Case Decomposition

Apply these rules during Step 4 planning and Step 6 implementation. Allocation decides which bucket a scenario belongs to. Decomposition decides how many distinct cases should exist inside that bucket.

Use this sequence:
1. group the verification matrix by dominant bucket or intent
2. inside each bucket, group scenarios by primary verification question
3. split cases again whenever setup lifecycle, checker model, or pass criteria diverge
4. factor shared setup into helpers, fixtures, tasks, or drivers instead of merging unrelated checks into one case

Case decomposition rules:
- One case should answer one primary verification question.
- One case should have one primary failure story that is easy to localize.
- A case may cover multiple parameter or stimulus variants only when they preserve the same question, setup model, checker style, and pass criteria.
- Shared reset or common driver code is not a reason to merge scenarios into one case.
- If two scenarios need different scoreboards, reference models, timeout windows, or observability points, they should usually be separate cases even if they stay in the same file.

Split a bucket into multiple cases when any of these are true:
- the scenarios prove different semantic claims such as hold versus restart, legality versus recovery, or boundary behavior versus long-run correctness
- the pass criteria differ
- the setup or stimulus lifecycle differs
- the checker, scoreboard, assertion set, or reference model differs
- one part of the test depends on an earlier part succeeding before the real target can be observed
- the case name would naturally contain "and", "then", or multiple independent phases
- a failure would not immediately tell the engineer which requirement broke

Refactor an existing case into multiple cases when:
- it has grown into a multi-stage script with unrelated assertions
- later checks rely on state prepared only by earlier checks
- rerunning one scenario requires replaying unrelated behavior
- debug requires reading many intermediate signals just to know which sub-story failed

Worked decomposition example inside a `control` bucket:
- `control_freeze_hold`: proves `en=0` holds phase and output
- `control_cfg_commit_no_phase_jump`: proves runtime config commit updates FTW without phase discontinuity
- `control_zero_ftw_restart`: proves restart semantics after zero-FTW commit

Shared setup handling: keep the cases separate, extract repeated reset or helper stimulus into reusable helpers, and do not keep scenarios fused only because they can share a fixture.

## Test Case Allocation And Evolution

Do not allocate tests by convenience, DUT name, or "there is already a file open." Allocate them by dominant verification intent.

Use three layers:
- verification matrix: maps feature or weakness to required verification intent
- test file or bucket: owns one dominant purpose such as `basic`, `control`, `boundary`, `corner`, `stress`, `waveform`, or `structure`
- test case: owns one scenario, one setup story, and one primary pass or fail contract

Default allocation rules:
- Put smoke or reset legality into `basic` or `sanity`.
- Put enable, freeze, commit, handshake, restart, or temporal protocol behavior into `control` or `protocol`.
- Put max, min, wraparound, singleton, zero-length, or parameter extrema into `boundary`.
- Put overlapping adverse conditions into `corner`.
- Put long-run randomized or endurance checks into `stress`.
- Put long-run numerical or sequence correctness into `waveform`, `datapath`, or another clearly scoped bucket.
- Put verification-scoped static checks such as required checker shape, required helper boundaries, or source-shape rules that directly protect verification intent into `structure`. General lint or style enforcement belongs elsewhere.

Merge a new demand into an existing case only if all of these are true:
- same dominant verification goal
- same setup and stimulus model
- same pass criteria
- no hidden dependency on earlier sub-scenarios in the same test
- merging does not turn the case into a multi-stage story that is harder to debug

If any merge criterion fails, do not append to the old case. Create a new case. If the existing file would become mixed-purpose, create a new file or bucket as well.

When a new test demand appears during execution, use this sequence:
1. map it to feature, weakness, and verification level
2. identify its dominant verification intent
3. compare that intent against existing file and case responsibilities
4. merge only if every merge criterion holds
5. otherwise create a new case, and usually a new file when the old file boundary would become mixed

Worked example:
- `basic`: reset and first legal output
- `control`: enable rise, freeze hold, config-commit timing, restart semantics
- `boundary`: `phase_init=max`, wraparound, `ftw=0`, singleton parameter edges
- `waveform`: multi-cycle numerical correctness and output frequency
- `structure`: verification-scoped static checks such as one-required-checker-per-block rules

Anti-patterns:
- do not add a wraparound boundary check into `basic` just because the DUT is the same
- do not merge zero-FTW restart behavior into a freeze-hold case if one check proves hold semantics and the other proves restart semantics with different pass criteria

## Output Templates

### Weakness Report

```text
## Design Weakness Report: [module_name]

### Summary
- High risk: [count]
- Medium risk: [count]
- Low risk: [count]

### Findings
| ID | Category | Location | Symptom | Risk | Verification Hook |
|----|----------|----------|---------|------|-------------------|
| W1 | CDC | sig_x | Missing synchronizer can corrupt data crossing | High | Directed CDC test plus sync assertion |
```

### Verification Plan

```text
## Verification Plan: [module_name]

### Chosen Environment
- Simulator: [VCS/Verilator/not required yet]
- Framework: [cocotb/Verilog TB/not required yet]
- Harness reuse: [yes/no, description]
- Reference model: [yes/no, description]

### Verification Matrix
| Test ID | Level | Target Weakness | Description | Pass Criteria |
|---------|-------|-----------------|-------------|---------------|
| T1 | Sanity | -- | Reset brings DUT to idle | All outputs = idle after reset |
| T2 | Directed | W1 | CDC handshake | Correct data transfer across domains |

### Test Allocation
| Bucket Or File | Dominant Intent | Scenarios Assigned | Why Here |
|----------------|-----------------|--------------------|----------|
| basic | Sanity | Reset and first legal output | Smoke legality only |
| boundary | Boundary | Max input wraparound | Different purpose and pass criteria from basic |

### Case Artifacts
| Case ID | Artifact Directory | Waveform Format | Notes |
|---------|--------------------|-----------------|-------|
| control_freeze_hold | sim_out/control_freeze_hold/ | FST | Log, seed, waveform, repro command kept together for both pass and fail results |

### Case Decomposition
| Bucket | Case ID | Primary Question | Shared Setup | Why Separate |
|--------|---------|------------------|--------------|--------------|
| control | control_freeze_hold | Does disable hold state and output? | Reset plus running configuration | Different pass criteria from restart semantics |
| control | control_zero_ftw_restart | Does restart from init phase stay DC after zero-FTW commit? | Reset plus running configuration | Different primary failure story from freeze hold |

### Assertion Inventory
| Assert ID | Type | Location | Property |
|-----------|------|----------|----------|
| A1 | Boundary | valid/ready | valid should not deassert before ready |

### Coverage Targets
- Functional: [list of coverpoints]
- Code: [targets or waivers]

### Residual Risk
- [blind spots and untested conditions]
```
