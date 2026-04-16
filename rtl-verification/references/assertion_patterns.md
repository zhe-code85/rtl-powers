# Assertion Patterns for RTL Verification

Use this file when turning verification intent into concrete assertions, checkers, or coverage.

## Contents

- `Protocol Assertions`
- `Invariant Assertions`
- `Cocotb-Side Checkers`
- `Coverage Patterns`
- `Common Anti-Patterns`

## Protocol Assertions

Valid-ready producer rule:

```systemverilog
property p_valid_stable_until_ready;
    @(posedge clk) disable iff (!rst_n)
    valid && !ready |=> valid;
endproperty

assert_valid_stable: assert property (p_valid_stable_until_ready)
    else $error("Valid deasserted before ready");
```

FIFO write legality:

```systemverilog
property p_wr_ptr_increments_on_write;
    @(posedge clk) disable iff (!rst_n)
    wr_en && !full |=> ($past(wr_ptr) + 1'b1) == wr_ptr;
endproperty

assert_wr_ptr_correct: assert property (p_wr_ptr_increments_on_write);
```

Multi-clock request/ack progress:

```systemverilog
property p_req_eventually_ack;
    @(posedge clk_a) disable iff (!rst_n_a)
    $rose(req_a) |-> ##[1:32] ack_b_sync;
endproperty

assert_req_eventually_ack: assert property (p_req_eventually_ack);
```

Use protocol assertions for:
- handshake legality
- bounded progress
- ordering and mutual exclusion
- timeout and recovery requirements

## Invariant Assertions

One-hot legality:

```systemverilog
property p_one_hot(signal);
    @(posedge clk) disable iff (!rst_n)
    $onehot(signal) || signal == '0;
endproperty

assert_fsm_onehot: assert property (p_one_hot(fsm_state));
```

Range limit:

```systemverilog
property p_counter_in_range;
    @(posedge clk) disable iff (!rst_n)
    counter <= MAX_COUNT;
endproperty

assert_counter_range: assert property (p_counter_in_range);
```

Reset value:

```systemverilog
property p_reset_value(signal, reset_val);
    @(posedge clk)
    $fell(rst_n) |=> signal == reset_val;
endproperty

assert_ctrl_reset: assert property (p_reset_value(ctrl_state, CTRL_IDLE));
```

Use invariant assertions for:
- state encoding
- counter/range safety
- reset postconditions
- architectural “must always hold” constraints

## Cocotb-Side Checkers

Signal health checker:

```python
from cocotb.triggers import RisingEdge


async def check_no_x_signals(dut, signal_names, clk_name="clk", cycles=20):
    clk = getattr(dut, clk_name)
    for _ in range(cycles):
        await RisingEdge(clk)
        for name in signal_names:
            value = getattr(dut, name).value
            assert not value.is_resolvable is False, f"{name} has X/Z"
```

Scoreboard shape:

```python
def check_transactions(actual, expected):
    assert actual == expected, f"Transaction mismatch: {actual} != {expected}"
```

Prefer cocotb-side checking when:
- the property is transaction-level
- the check is easier in Python than in SVA
- a lightweight reference model already exists in Python

## Coverage Patterns

Cover property:

```systemverilog
cover_timeout_path: cover property (
    @(posedge clk) disable iff (!rst_n)
    req && !ack ##[1:16] timeout
);
```

Covergroup shape:

```systemverilog
covergroup cg_status @(posedge clk);
    coverpoint state;
    coverpoint timeout_seen;
endgroup
```

Python-side coverage counter:

```python
coverage = {"timeout": 0, "flush_collision": 0}


def hit(name):
    coverage[name] += 1
```

Use coverage to prove:
- planned modes were exercised
- timeout and recovery paths were hit
- boundary tuples actually ran
- adverse scenarios were not only planned but observed

## Common Anti-Patterns

- Do not assert unstable implementation trivia when the real contract is external behavior.
- Do not write assertions that fire every cycle with no debug value.
- Do not omit reset disable conditions on properties that are not meaningful during reset.
- Do not treat code coverage as the definition of verification closure.
- Do not replace a needed reference model with brittle one-off expected-value checks.
