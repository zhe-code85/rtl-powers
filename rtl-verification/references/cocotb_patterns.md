# Cocotb Verification Patterns

Complete working examples for cocotb-based testbenches. Use these as starting templates when building verification environments.

---

## Section 1: Minimal Working Testbench

The smallest functional cocotb testbench: clock generation, reset, and one test.

```python
# test_my_module.py
import cocotb
from cocotb.clock import Clock
from cocotb.triggers import RisingEdge, Timer

@cocotb.test()
async def test_reset(dut):
    """Verify DUT resets to known state."""
    # Start clock: 100 MHz (10ns period)
    clock = Clock(dut.clk, 10, units="ns")
    cocotb.start_soon(clock.start())

    # Apply reset
    dut.rst_n.value = 0
    await Timer(30, units="ns")  # hold reset for 3 cycles
    dut.rst_n.value = 1
    await RisingEdge(dut.clk)
    await RisingEdge(dut.clk)  # settle 2 cycles after reset

    # Check reset state
    assert dut.data_out.value == 0, f"Expected 0 after reset, got {dut.data_out.value}"
    assert dut.valid_out.value == 0, "valid should be low after reset"
```

Corresponding Makefile:

```makefile
# Makefile
SIM ?= icarus
TOPLEVEL_LANG = verilog
VERILOG_SOURCES = $(PWD)/rtl/my_module.v
MODULE = test_my_module
TOPLEVEL = my_module
include $(shell cocotb-config --makefiles)/Makefile.sim
```

---

## Section 2: Coroutine Patterns

### Clock Generation

```python
from cocotb.clock import Clock

# Basic: fixed frequency
clock = Clock(dut.clk, 10, units="ns")  # 100 MHz
cocotb.start_soon(clock.start())

# With phase offset (for multi-clock CDC tests)
clock_a = Clock(dut.clk_a, 10, units="ns")
clock_b = Clock(dut.clk_b, 8, units="ns")  # 125 MHz, different frequency
cocotb.start_soon(clock_a.start())
cocotb.start_soon(clock_b.start())
```

### Reset Sequence

```python
from cocotb.triggers import RisingEdge, Timer

async def apply_reset(dut, clk_name="clk", rst_name="rst_n", cycles=5):
    """Apply synchronous active-low reset for specified cycles."""
    rst = getattr(dut, rst_name)
    clk = getattr(dut, clk_name)

    rst.value = 0
    for _ in range(cycles):
        await RisingEdge(clk)
    rst.value = 1
    await RisingEdge(clk)
    await RisingEdge(clk)  # settle after reset release
```

### Valid-Ready Handshake Driver

```python
from cocotb.triggers import RisingEdge

async def drive_valid_ready(dut, data, clk_name="clk"):
    """Drive a valid-ready handshake transaction.
    Waits for ready before deasserting valid.
    Handles backpressure (ready deasserted mid-transfer).
    """
    clk = getattr(dut, clk_name)

    dut.data_in.value = data
    dut.valid_in.value = 1

    # Wait for transfer: valid AND ready
    while True:
        await RisingEdge(clk)
        if dut.ready_out.value == 1 and dut.valid_in.value == 1:
            break

    dut.valid_in.value = 0
```

### Backpressure-Aware Producer

```python
import cocotb
from cocotb.triggers import RisingEdge

async def producer(dut, data_list, valid_sig, ready_sig, data_sig, clk_name="clk"):
    """Send a list of data words with valid-ready handshaking.
    Respects backpressure from consumer.
    """
    clk = getattr(dut, clk_name)
    for word in data_list:
        data_sig.value = word
        valid_sig.value = 1
        # Wait for acceptance
        while True:
            await RisingEdge(clk)
            if ready_sig.value == 1:
                break
        valid_sig.value = 0
```

### Concurrent Stimulus (Fork/Join)

```python
import cocotb
from cocotb.triggers import RisingEdge, First, Event

@cocotb.test()
async def test_concurrent_read_write(dut):
    """Write and read simultaneously to test concurrent access."""
    clock = Clock(dut.clk, 10, units="ns")
    cocotb.start_soon(clock.start())
    await apply_reset(dut)

    write_data = list(range(20))

    async def writer():
        for val in write_data:
            dut.wr_en.value = 1
            dut.wr_data.value = val
            await RisingEdge(dut.clk)
            # Handle full condition
            while dut.full.value == 1:
                dut.wr_en.value = 0
                await RisingEdge(dut.clk)
        dut.wr_en.value = 0

    async def reader():
        results = []
        for _ in range(len(write_data)):
            await RisingEdge(dut.clk)
            while dut.empty.value == 1:
                await RisingEdge(dut.clk)
            dut.rd_en.value = 1
            await RisingEdge(dut.clk)
            results.append(int(dut.rd_data.value))
        dut.rd_en.value = 0
        return results

    # Run both concurrently
    write_task = cocotb.start_soon(writer())
    read_task = cocotb.start_soon(reader())

    await write_task
    results = await read_task

    assert results == write_data, f"Data mismatch: {results} != {write_data}"
```

---

## Section 3: Driver / Monitor / Scoreboard

### Lightweight Driver

```python
class AxiLiteDriver:
    """Lightweight AXI4-Lite driver without cocotb-bus dependency.
    Handles AW/W/B and AR/R channels.
    """
    def __init__(self, dut, prefix="s_axi_", clk="clk"):
        self.dut = dut
        self.prefix = prefix
        self.clk = getattr(dut, clk)

    async def write(self, addr, data):
        """Single AXI-Lite write transaction."""
        d = self.dut
        p = self.prefix

        # AW phase
        getattr(d, f"{p}awaddr").value = addr
        getattr(d, f"{p}awvalid").value = 1
        while getattr(d, f"{p}awready").value != 1:
            await RisingEdge(self.clk)
        await RisingEdge(self.clk)
        getattr(d, f"{p}awvalid").value = 0

        # W phase
        getattr(d, f"{p}wdata").value = data
        getattr(d, f"{p}wvalid").value = 1
        while getattr(d, f"{p}wready").value != 1:
            await RisingEdge(self.clk)
        await RisingEdge(self.clk)
        getattr(d, f"{p}wvalid").value = 0

        # B phase
        getattr(d, f"{p}bready").value = 1
        while getattr(d, f"{p}bvalid").value != 1:
            await RisingEdge(self.clk)
        bresp = int(getattr(d, f"{p}bresp").value)
        await RisingEdge(self.clk)
        getattr(d, f"{p}bready").value = 0
        assert bresp == 0, f"Write error: bresp={bresp}"

    async def read(self, addr):
        """Single AXI-Lite read transaction."""
        d = self.dut
        p = self.prefix

        getattr(d, f"{p}araddr").value = addr
        getattr(d, f"{p}arvalid").value = 1
        while getattr(d, f"{p}arready").value != 1:
            await RisingEdge(self.clk)
        await RisingEdge(self.clk)
        getattr(d, f"{p}arvalid").value = 0

        getattr(d, f"{p}rready").value = 1
        while getattr(d, f"{p}rvalid").value != 1:
            await RisingEdge(self.clk)
        rdata = int(getattr(d, f"{p}rdata").value)
        rresp = int(getattr(d, f"{p}rresp").value)
        await RisingEdge(self.clk)
        getattr(d, f"{p}rready").value = 0
        assert rresp == 0, f"Read error: rresp={rresp}"
        return rdata
```

### Monitor

```python
from cocotb.triggers import RisingEdge

class HandshakeMonitor:
    """Observes valid-ready interface and records transactions."""
    def __init__(self, dut, valid_sig, ready_sig, data_sig, clk_name="clk"):
        self.dut = dut
        self.valid_sig = getattr(dut, valid_sig)
        self.ready_sig = getattr(dut, ready_sig)
        self.data_sig = getattr(dut, data_sig)
        self.clk = getattr(dut, clk_name)
        self.transactions = []

    async def run(self):
        """Continuously monitor and record transfers."""
        while True:
            await RisingEdge(self.clk)
            if self.valid_sig.value == 1 and self.ready_sig.value == 1:
                self.transactions.append(int(self.data_sig.value))

    def get_transactions(self):
        """Return recorded transactions and clear buffer."""
        result = self.transactions.copy()
        self.transactions.clear()
        return result
```

### Scoreboard

```python
class Scoreboard:
    """Compare expected (reference model) vs actual (DUT) outputs."""
    def __init__(self):
        self.expected = []
        self.mismatches = []

    def add_expected(self, value):
        """Add an expected output value."""
        self.expected.append(value)

    def compare(self, actual):
        """Compare actual output against next expected value."""
        if not self.expected:
            self.mismatches.append(f"Unexpected output: {actual} (no expected value)")
            return False

        exp = self.expected.pop(0)
        if actual != exp:
            self.mismatches.append(f"Mismatch: expected {exp}, got {actual}")
            return False
        return True

    def report(self):
        """Report all mismatches. Raise if any found."""
        if self.mismatches:
            msg = "\n".join(self.mismatches)
            assert False, f"Scoreboard failures:\n{msg}"

    @property
    def remaining(self):
        """Number of unmatched expected values."""
        return len(self.expected)
```

---

## Section 4: Cocotb-bus Usage

For standard bus interfaces (AXI, APB), cocotb-bus provides pre-built drivers with protocol checking.

```python
# Requires: pip install cocotb-bus
from cocotbext.axi import AxiLiteBus, AxiLiteMaster

@cocotb.test()
async def test_with_cocotb_bus(dut):
    """Use cocotb-bus AXI-Lite master for cleaner bus driving."""
    clock = Clock(dut.clk, 10, units="ns")
    cocotb.start_soon(clock.start())
    await apply_reset(dut)

    # Create AXI-Lite master connected to DUT signals
    axi_master = AxiLiteMaster(
        AxiLiteBus.from_prefix(dut, "s_axi"),
        dut.clk,
        dut.rst_n
    )

    # Write and read using high-level API
    await axi_master.write_dword(0x0000, 0xDEADBEEF)
    value = await axi_master.read_dword(0x0000)
    assert value == 0xDEADBEEF
```

When to use cocotb-bus vs lightweight:
- **cocotb-bus**: Standard bus protocol (AXI, APB, Wishbone), burst support, protocol checking built in
- **Lightweight**: Custom protocols, non-standard interfaces, minimal dependency requirements

---

## Section 5: Regression Structure

### pytest Integration

```python
# conftest.py
import cocotb
import pytest

# No special conftest needed — cocotb discovers tests automatically
# when run via Makefile. For pytest-native discovery:
```

```python
# test_fifo_regression.py
import cocotb
from cocotb.clock import Clock
from cocotb.triggers import RisingEdge
from cocotb.regression import TestFactory

async def test_fifo_basic(dut, depth, width):
    """Parameterized FIFO test — runs for each depth/width combo."""
    clock = Clock(dut.clk, 10, units="ns")
    cocotb.start_soon(clock.start())

    dut.rst_n.value = 0
    await RisingEdge(dut.clk)
    await RisingEdge(dut.clk)
    dut.rst_n.value = 1
    await RisingEdge(dut.clk)

    # Fill FIFO to depth
    for i in range(depth):
        dut.wr_en.value = 1
        dut.wr_data.value = i & ((1 << width) - 1)
        await RisingEdge(dut.clk)
        assert dut.full.value == 0, f"FIFO full at {i}/{depth}"

    dut.wr_en.value = 0
    assert dut.full.value == 1, f"FIFO not full after {depth} writes"

    # Drain FIFO
    for i in range(depth):
        dut.rd_en.value = 1
        await RisingEdge(dut.clk)
        expected = i & ((1 << width) - 1)
        actual = int(dut.rd_data.value)
        assert actual == expected, f"Read mismatch at {i}: expected {expected}, got {actual}"

    dut.rd_en.value = 0
    assert dut.empty.value == 1, "FIFO not empty after draining"

# Register parameterized variants
factory = TestFactory(test_fifo_basic)
factory.add_option("depth", [4, 8, 16, 32])
factory.add_option("width", [8, 16, 32])
factory.generate_tests()
```

### CI-Friendly Makefile Runner

```makefile
# run_regression.mk
SIM ?= icarus
TOPLEVEL_LANG = verilog
VERILOG_SOURCES = $(PWD)/rtl/fifo_sync.v
MODULE = test_fifo_regression
TOPLEVEL = fifo_sync

# CI: run with junit XML output
COCOTB_RESULTS_FILE = results.xml

include $(shell cocotb-config --makefiles)/Makefile.sim
```

```bash
# CI pipeline command
make -f run_regression.mk SIM=icandrone COCOTB_RESULTS_FILE=results.xml
```

---

## Section 6: Common Pitfalls

### Delta Cycles vs Simulation Time

Cocotb triggers fire in delta cycles within a simulation time step. When you `await RisingEdge(clk)`, the coroutine resumes at the delta cycle where clk transitions, NOT at the next simulation time step. Signal reads after `RisingEdge` see values from the previous delta cycle.

```python
# WRONG: read happens in same delta as clock edge — may see old values
await RisingEdge(dut.clk)
value = int(dut.data.value)  # might be stale

# CORRECT: add one more delta for signals to propagate
await RisingEdge(dut.clk)
await RisingEdge(dut.clk)  # now data is stable
value = int(dut.data.value)
```

### Signal Access Gotchas

```python
# Signal values are NOT plain Python ints
# Always cast to int for comparison
val = dut.counter.value
if val == 5:      # works
    pass
if int(val) > 3:  # safer for arithmetic
    pass

# Writing values: use .value assignment
dut.wr_data.value = 42
# NOT: dut.wr_data = 42  (replaces the handle)

# Undefined signals return 'x' or 'z'
val = dut.signal.value
if val.is_resolvable:  # check before int() conversion
    number = int(val)
else:
    # Signal is undefined — handle accordingly
    pass
```

### Race Conditions in Stimulus

```python
# WRONG: stimulus and check in same delta
dut.wr_en.value = 1
dut.wr_data.value = 42
await RisingEdge(dut.clk)  # DUT sees the inputs THIS cycle
result = int(dut.rd_data.value)  # may not be ready yet

# CORRECT: drive, wait for effect, then check
dut.wr_en.value = 1
dut.wr_data.value = 42
await RisingEdge(dut.clk)
dut.wr_en.value = 0
await RisingEdge(dut.clk)  # let output propagate
result = int(dut.rd_data.value)
```

### Timeout Watchdog

```python
from cocotb.triggers import Timer, First

async def with_timeout(coro, timeout_us=100):
    """Run coroutine with timeout. Fail test if timeout expires."""
    result = await First(
        coro,
        Timer(timeout_us, units="us")
    )
    # If Timer won, the coroutine didn't complete
    # Check by seeing if result is from the coroutine
    return result

@cocotb.test()
async def test_no_deadlock(dut):
    """Verify DUT doesn't hang during backpressure."""
    clock = Clock(dut.clk, 10, units="ns")
    cocotb.start_soon(clock.start())
    await apply_reset(dut)

    # This should complete within 50us
    async def write_sequence():
        for i in range(100):
            await drive_valid_ready(dut, i)

    try:
        await with_timeout(write_sequence(), timeout_us=50)
    except Exception:
        assert False, "DUT appears to deadlock under backpressure"
```
