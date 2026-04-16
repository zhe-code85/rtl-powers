# Cocotb Verification Patterns

Use this file when building runnable cocotb collateral. Keep the harness small, traceable, and easy to extend.

Assume the project verification environment is already activated before running `make`, `pytest`, or simulator commands. Keep harnesses environment-agnostic where possible, and do not bake a machine-local `venv` path into example commands or generated collateral.

## Contents

- `Artifact Layout`
- `Minimal Harness`
- `Core Helpers`
- `Regression Structure`
- `Common Pitfalls`

## Artifact Layout

Keep one artifact directory per executed case. Store that case's waveform, log, seed, and repro command together.

Preferred layout:

```text
sim_out/
  control_freeze_hold/
    run.log
    waveform.fst
    seed.txt
    repro.sh
  control_zero_ftw_restart/
    run.log
    waveform.fst
    seed.txt
    repro.sh
```

Waveform rules:
- keep waveform files inside the case directory
- keep waveform retention enabled for every executed case, including passing cases
- do not switch to "fail-only waveform retention" to save disk space; reduce waveform scope or rely on the compressed format instead
- use a compressed waveform format such as `FST`, `FSDB`, or `VPD`
- extend the project's existing compressed waveform convention when one already exists
- update any `VCD`-based harness before adding new runnable cases

One workable runner shape:

```bash
CASE=test_reset
ARTIFACT_DIR=$PWD/sim_out/$CASE

mkdir -p "$ARTIFACT_DIR"
make SIM=verilator CASE="$CASE" ARTIFACT_DIR="$ARTIFACT_DIR" WAVE_FORMAT=fst \
  | tee "$ARTIFACT_DIR/run.log"
printf 'make SIM=verilator CASE=%s ARTIFACT_DIR=%s WAVE_FORMAT=fst\n' \
  "$CASE" "$ARTIFACT_DIR" > "$ARTIFACT_DIR/repro.sh"
```

## Minimal Harness

Minimal cocotb test:

```python
import cocotb
from cocotb.clock import Clock
from cocotb.triggers import RisingEdge


async def apply_reset(dut, cycles=5):
    dut.rst_n.value = 0
    for _ in range(cycles):
        await RisingEdge(dut.clk)
    dut.rst_n.value = 1
    await RisingEdge(dut.clk)
    await RisingEdge(dut.clk)


@cocotb.test()
async def test_reset(dut):
    cocotb.start_soon(Clock(dut.clk, 10, units="ns").start())
    await apply_reset(dut)
    assert dut.valid_out.value == 0
    assert dut.data_out.value == 0
```

Minimal Makefile skeleton:

```makefile
SIM ?= verilator
TOPLEVEL_LANG = verilog
VERILOG_SOURCES = $(PWD)/rtl/my_module.v
MODULE = test_my_module
TOPLEVEL = my_module
CASE ?= test_reset
ARTIFACT_DIR ?= $(PWD)/sim_out/$(CASE)
SIM_BUILD = $(ARTIFACT_DIR)/sim_build
COCOTB_RESULTS_FILE = $(ARTIFACT_DIR)/results.xml

# Assume the project verification environment is already active.
# Keep the harness generic instead of hardcoding a local venv path.

# Hook these variables into the local harness so the simulator writes a
# compressed waveform such as waveform.fst, waveform.fsdb, or waveform.vpd
# under $(ARTIFACT_DIR) instead of a shared dump.vcd.
include $(shell cocotb-config --makefiles)/Makefile.sim
```

## Core Helpers

Keep helpers lightweight. Add more structure only when reuse is real.

Clock and reset:

```python
from cocotb.clock import Clock
from cocotb.triggers import RisingEdge


async def start_clock(dut, name="clk", period_ns=10):
    cocotb.start_soon(Clock(getattr(dut, name), period_ns, units="ns").start())


async def apply_reset(dut, clk_name="clk", rst_name="rst_n", cycles=5):
    rst = getattr(dut, rst_name)
    clk = getattr(dut, clk_name)
    rst.value = 0
    for _ in range(cycles):
        await RisingEdge(clk)
    rst.value = 1
    await RisingEdge(clk)
```

Handshake driver:

```python
from cocotb.triggers import RisingEdge


async def drive_valid_ready(dut, data, clk_name="clk"):
    clk = getattr(dut, clk_name)
    dut.data_in.value = data
    dut.valid_in.value = 1
    while True:
        await RisingEdge(clk)
        if dut.valid_in.value == 1 and dut.ready_out.value == 1:
            break
    dut.valid_in.value = 0
```

Monitor and scoreboard shape:

```python
from cocotb.triggers import RisingEdge


class HandshakeMonitor:
    def __init__(self, dut, valid_sig, ready_sig, data_sig, clk_name="clk"):
        self.valid_sig = getattr(dut, valid_sig)
        self.ready_sig = getattr(dut, ready_sig)
        self.data_sig = getattr(dut, data_sig)
        self.clk = getattr(dut, clk_name)
        self.transactions = []

    async def run(self):
        while True:
            await RisingEdge(self.clk)
            if self.valid_sig.value == 1 and self.ready_sig.value == 1:
                self.transactions.append(int(self.data_sig.value))


def check_sequence(actual, expected):
    assert actual == expected, f"Mismatch: {actual} != {expected}"
```

Reference model guidance:
- use a Python reference model when direct expected-value checks become fragile
- keep the model at contract level, not internal implementation level
- place model state beside the scoreboard, not scattered through tests

## Regression Structure

Recommended split:
- one test file per dominant verification bucket when practical
- one case per primary verification question
- shared reset, helpers, and reference models in reusable support files

Useful `pytest` shape:

```python
import pytest


@pytest.mark.parametrize("case_name", ["basic_reset_idle", "boundary_wrap"])
def test_cases(case_name):
    assert case_name
```

CI-friendly runner guidance:
- pass `CASE`, `ARTIFACT_DIR`, and waveform format through the harness
- keep simulator build products under the case directory or a clearly namespaced subdirectory
- keep the repro command stable enough that a failed case can be rerun directly

## Common Pitfalls

- Do not rely on delta-cycle timing guesses when a clock edge or explicit trigger is the real contract.
- Do not bury unrelated checks inside one long case just because the setup is shared.
- Do not build a second harness when the repository already has a usable one.
- Do not sample waveforms into a shared `dump.vcd`; keep one retained compressed waveform per executed case.
- Do not overuse heavy bus libraries when a small hand-written driver is easier to review and maintain.
