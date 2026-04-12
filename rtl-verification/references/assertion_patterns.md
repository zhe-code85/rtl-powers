# Assertion Patterns for RTL Verification

Complete examples of SVA assertions and cocotb checkers organized by category. Use these as templates when adding assertion coverage to a design.

---

## Section 1: SVA Protocol Checkers

### Valid-Ready Handshake (Producer Side)

```systemverilog
// Assert: once valid is asserted, it must stay asserted until ready is seen.
// Rationale: protocol violation — producer must not deassert valid before transfer.
// Placement: inside the producer module or as a bind module.

property p_valid_stable_until_ready;
    @(posedge clk) disable iff (!rst_n)
    valid && !ready |=> valid;  // if not accepted, valid stays
endproperty

assert_valid_stable: assert property (p_valid_stable_until_ready)
    else $error("Valid deasserted before ready received");

// Cover: ensure both simultaneous and non-simultaneous handshakes occur
cover_valid_ready_simultaneous: cover property (
    @(posedge clk) disable iff (!rst_n)
    valid && ready
);

cover_valid_before_ready: cover property (
    @(posedge clk) disable iff (!rst_n)
    valid && !ready ##1 valid && ready
);
```

### Valid-Ready Handshake (Consumer Side)

```systemverilog
// Assert: once ready is deasserted, it must not re-assert in the same transfer.
// This prevents the consumer from "changing its mind" mid-transaction.
// Note: this is a stricter variant — some protocols allow ready to toggle.
// Apply only if your protocol requires stable ready.

property p_ready_stable_during_transfer;
    @(posedge clk) disable iff (!rst_n)
    valid && !ready |=> !ready until_with (valid && ready);
endproperty

assert_ready_protocol: assert property (p_ready_stable_during_transfer);
```

### FIFO Pointer Behavior

```systemverilog
// Assert: write pointer increments only when writing to a non-full FIFO.
// Rationale: catch overflow bugs where wr_en is ignored but pointer moves.
// Placement: inside the FIFO module.

property p_wr_ptr_increments_on_write;
    @(posedge clk) disable iff (!rst_n)
    wr_en && !full |=> ($past(wr_ptr) + 1'b1) == wr_ptr;
endproperty

assert_wr_ptr_correct: assert property (p_wr_ptr_increments_on_write)
    else $error("Write pointer behavior incorrect: wr_en=%b full=%b", wr_en, full);

// Assert: full flag means write pointer equals read pointer (with MSB toggle)
property p_full_means_ptr_match;
    @(posedge clk) disable iff (!rst_n)
    full |-> wr_ptr[MSB] != rd_ptr[MSB] &&
             wr_ptr[MSB-1:0] == rd_ptr[MSB-1:0];
endproperty

assert_full_correct: assert property (p_full_means_ptr_match);
```

### AXI Protocol Checker (Write Response)

```systemverilog
// Assert: BVALID must not be asserted unless a write was accepted.
// Rationale: catch spurious write responses.
// Placement: in AXI subordinate wrapper.

// Track outstanding writes
logic outstanding_write;

always_ff @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        outstanding_write <= 1'b0;
    else begin
        if (s_axi_awvalid && s_axi_awready && s_axi_wvalid && s_axi_wready)
            outstanding_write <= 1'b1;
        else if (s_axi_bvalid && s_axi_bready)
            outstanding_write <= 1'b0;
    end
end

property p_bvalid_requires_write;
    @(posedge clk) disable iff (!rst_n)
    s_axi_bvalid |-> outstanding_write;
endproperty

assert_bvalid_valid: assert property (p_bvalid_requires_write)
    else $error("BVALID without outstanding write");
```

---

## Section 2: SVA Invariant Checkers

### One-Hot Encoding Check

```systemverilog
// Assert: signal is one-hot encoded (exactly one bit set, or zero for idle).
// Rationale: catch FSM corruption or illegal state encoding.
// Placement: after the state register.

property p_one_hot(signal);
    @(posedge clk) disable iff (!rst_n)
    $onehot(signal) || signal == '0;
endproperty

assert_fsm_onehot: assert property (p_one_hot(fsm_state))
    else $error("FSM state not one-hot: %b", fsm_state);
```

### Mutual Exclusion Check

```systemverilog
// Assert: two grants are never active simultaneously.
// Rationale: catch arbiter bugs that grant to multiple requestors.
// Placement: in the arbiter module.

property p_mutual_exclusion;
    @(posedge clk) disable iff (!rst_n)
    !(grant[0] && grant[1]) &&
    !(grant[0] && grant[2]) &&
    !(grant[1] && grant[2]);
endproperty

assert_no_double_grant: assert property (p_mutual_exclusion)
    else $error("Multiple grants active simultaneously: %b", grant);
```

### Range / Overflow Check

```systemverilog
// Assert: counter stays within valid range.
// Rationale: catch wraparound bugs in saturating counters.
// Placement: near the counter logic.

property p_counter_in_range;
    @(posedge clk) disable iff (!rst_n)
    counter <= MAX_COUNT;
endproperty

assert_counter_range: assert property (p_counter_in_range)
    else $error("Counter overflow: counter=%0d max=%0d", counter, MAX_COUNT);
```

### Reset Value Check

```systemverilog
// Assert: critical registers have known values after reset.
// Rationale: catch incomplete or missing reset on state-holding elements.
// Placement: immediately after the register declaration.

property p_reset_value(signal, reset_val);
    @(posedge clk)
    $fell(rst_n) |=> signal == reset_val;
endproperty

assert_ctrl_reset: assert property (p_reset_value(ctrl_state, CTRL_IDLE))
    else $error("Control state not reset to IDLE");
```

---

## Section 3: Cocotb-Side Checkers

### Python Assertion Functions

```python
from cocotb.triggers import RisingEdge

async def check_no_x_signals(dut, signal_names, clk_name="clk", duration_ns=1000):
    """Monitor signals for X/Z values over a time window."""
    clk = getattr(dut, clk_name)
    elapsed = 0
    period_ns = 10  # adjust to clock period

    while elapsed < duration_ns:
        await RisingEdge(clk)
        elapsed += period_ns
        for name in signal_names:
            sig = getattr(dut, name)
            if not sig.value.is_resolvable:
                assert False, f"Signal {name} is X or Z at time {elapsed}ns"
```

### Scoreboard-Based Checking

```python
class FifoScoreboard:
    """Scoreboard for FIFO: compare pushed vs popped data."""

    def __init__(self):
        self.expected = []

    def push(self, value):
        """Record a write to FIFO."""
        self.expected.append(value)

    def pop(self, actual):
        """Verify a read from FIFO matches expected."""
        if not self.expected:
            assert False, f"FIFO popped {actual} but queue is empty"
        expected = self.expected.pop(0)
        assert actual == expected, f"FIFO data mismatch: expected {expected}, got {actual}"

    def verify_empty(self):
        """After all operations, FIFO should be drained."""
        assert len(self.expected) == 0, \
            f"FIFO not empty: {len(self.expected)} items remain: {self.expected}"
```

### Timeout Watchdog

```python
import cocotb
from cocotb.triggers import RisingEdge, Timer, First

class Watchdog:
    """Fail the test if a condition isn't met within a time limit."""

    def __init__(self, dut, timeout_us=100, clk_name="clk"):
        self.dut = dut
        self.timeout_us = timeout_us
        self.clk_name = clk_name

    async def wait_for(self, condition_fn, msg="Timeout waiting for condition"):
        """Wait for condition_fn() to return True, or fail with timeout."""
        clk = getattr(self.dut, self.clk_name)
        elapsed = 0
        period_us = 0.01  # 10ns in us

        while elapsed < self.timeout_us:
            await RisingEdge(clk)
            elapsed += period_us
            if condition_fn():
                return

        assert False, f"{msg} (timeout after {self.timeout_us}us)"
```

Usage:

```python
@cocotb.test()
async def test_transfer_completes(dut):
    """Verify data transfer completes within expected time."""
    # ... setup ...
    wd = Watchdog(dut, timeout_us=10)
    await wd.wait_for(
        lambda: dut.transfer_done.value == 1,
        msg="Transfer did not complete"
    )
```

---

## Section 4: Anti-Patterns

### Noisy Assertions: Fire Every Cycle

```systemverilog
// BAD: This fires every cycle the condition is true, creating log spam.
// Engineers learn to ignore noisy assertions.
assert property (@(posedge clk) disable iff (!rst_n)
    data_valid |-> data != 0
);

// GOOD: Only check when data_valid first rises, or check a specific protocol event.
assert property (@(posedge clk) disable iff (!rst_n)
    $rose(data_valid) |-> data != 0  // check only on assertion edge
);
```

### Testing Implementation, Not Intent

```systemverilog
// BAD: Tests internal implementation detail that may change during refactoring.
assert property (@(posedge clk) disable iff (!rst_n)
    internal_counter == expected_counter_value  // fragile!
);

// GOOD: Tests observable behavior that is invariant across implementations.
assert property (@(posedge clk) disable iff (!rst_n)
    output_data == expected_output  // behavioral, not structural
);
```

### Over-Specified Timing Assertions

```systemverilog
// BAD: Exact cycle count breaks when pipeline depth changes or retiming is applied.
assert property (@(posedge clk) disable iff (!rst_n)
    $rose(start) |-> ##5 result_valid  // fragile: exact latency dependency
);

// GOOD: Use range or event-driven check that tolerates latency changes.
assert property (@(posedge clk) disable iff (!rst_n)
    $rose(start) |-> ##[1:10] result_valid  // range-based
);

// BEST: Check protocol correctness, not latency.
assert property (@(posedge clk) disable iff (!rst_n)
    result_valid |-> $past(start, 1) || $past(start, 2)  // causal link
);
```

### Missing Reset Disable

```systemverilog
// BAD: Assertion fires during reset when signals are undefined.
assert property (@(posedge clk)
    valid |-> data != 0  // fires on X values during reset!
);

// GOOD: Always disable during reset.
assert property (@(posedge clk) disable iff (!rst_n)
    valid |-> data != 0
);
```
