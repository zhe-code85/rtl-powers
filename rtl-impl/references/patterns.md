# PPA-Aware RTL Coding Patterns

These are reference patterns, not the main workflow. The main workflow lives in `../SKILL.md`.

Use these examples as `Verilog`-style implementation templates for `rtl-impl`. If project helper macros or utility functions exist, adapt the examples to match those local `Verilog` conventions without switching the RTL itself to `SystemVerilog`.

Annotated examples organized by category. Each pattern includes PPA rationale explaining why the pattern helps timing, area, or power. Use these as starting templates when implementing custom RTL.

## Quick Index

- Section 1: Datapath patterns
- Section 2: Control patterns
- Section 3: Storage patterns
- Section 4: Interface patterns
- Section 5: Arithmetic patterns
- Section 6: CBB instantiation and wrapper patterns

---

## Section 1: Datapath Patterns

### 1.1 Pipeline Boundary with Valid-Ready Handshaking

Insert at natural pipeline boundaries to support backpressure. The `valid`/`ready` handshake allows downstream modules to stall without complex freeze logic propagating backward through the entire pipeline.

```verilog
// Pipeline stage with one-entry skid buffering.
// When the output register is blocked, one extra beat can be parked in skid_data.
// If the blocked beat is consumed and a new beat arrives in the same cycle,
// the design refills skid_data so the handshake stays lossless.
// PPA: clean timing boundary; one extra register slice avoids a long stall path.

module pipeline_stage #(
    parameter WIDTH = 32
)(
    input  wire             clk,
    input  wire             rst_n,
    input  wire [WIDTH-1:0] din,
    input  wire             din_valid,
    output wire             din_ready,
    output wire [WIDTH-1:0] dout,
    output wire             dout_valid,
    input  wire             dout_ready
);

    // Skid register: holds data when downstream deasserts ready mid-transfer
    reg [WIDTH-1:0] skid_data;
    reg             skid_valid;

    // Output register visible at the downstream interface
    reg [WIDTH-1:0] out_data;
    reg             out_valid;

    wire out_fire = out_valid & dout_ready;

    // Upstream can send whenever the skid entry is free.
    assign din_ready = ~skid_valid;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            skid_valid <= 1'b0;
            out_valid  <= 1'b0;
            out_data   <= {WIDTH{1'b0}};
            skid_data  <= {WIDTH{1'b0}};
        end else begin
            // Consume the current output beat first. Refill from skid or input.
            if (out_fire) begin
                if (skid_valid) begin
                    out_data  <= skid_data;
                    out_valid <= 1'b1;

                    // If a new beat arrives while draining skid, keep skid occupied.
                    if (din_valid) begin
                        skid_data  <= din;
                        skid_valid <= 1'b1;
                    end else begin
                        skid_valid <= 1'b0;
                    end
                end else if (din_valid) begin
                    out_data  <= din;
                    out_valid <= 1'b1;
                end else begin
                    out_valid <= 1'b0;
                end
            end else if (!out_valid && din_valid) begin
                out_data  <= din;
                out_valid <= 1'b1;
            end else if (out_valid && !dout_ready && din_valid && !skid_valid) begin
                skid_data  <= din;
                skid_valid <= 1'b1;
            end
        end
    end

    assign dout       = out_data;
    assign dout_valid = out_valid;

endmodule
```

### 1.2 Resource Sharing: Single Multiplier Time-Multiplexed

Bad: Two parallel multipliers when throughput allows sharing.

```verilog
// BAD: Two multipliers used for sequential operations.
// PPA cost: 2x DSP blocks when only 1 is needed.
// Area wasted when mul_a and mul_b never compute simultaneously.
wire [31:0] result_a = data_a * coeff_a;
wire [31:0] result_b = data_b * coeff_b;
```

Good: One multiplier shared across time slots.

```verilog
// GOOD: Time-multiplexed multiplier shared between two operations.
// PPA: Saves one DSP block. Control logic is minimal (one mux + toggle).
//      Throughput halved per stream but area halved — acceptable when
//      full throughput per stream is not required.
// The first product is parked until the second product arrives, then the pair
// is published together with result_valid.

module shared_mult #(
    parameter WIDTH = 16
)(
    input  wire             clk,
    input  wire             rst_n,
    input  wire [WIDTH-1:0] data_a,
    input  wire [WIDTH-1:0] coeff_a,
    input  wire [WIDTH-1:0] data_b,
    input  wire [WIDTH-1:0] coeff_b,
    input  wire             data_valid,
    output reg  [2*WIDTH-1:0] result_a,
    output reg  [2*WIDTH-1:0] result_b,
    output reg                result_valid
);

    reg sel;  // 0 = capture A, 1 = capture B and publish the pair
    reg [2*WIDTH-1:0] a_product_hold;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            sel            <= 1'b0;
            result_valid   <= 1'b0;
            result_a       <= {(2*WIDTH){1'b0}};
            result_b       <= {(2*WIDTH){1'b0}};
            a_product_hold <= {(2*WIDTH){1'b0}};
        end else if (data_valid) begin
            // Single multiplier, time-shared
            if (!sel) begin
                a_product_hold <= data_a * coeff_a;
                result_valid   <= 1'b0;
            end else begin
                result_a     <= a_product_hold;
                result_b     <= data_b * coeff_b;
                result_valid <= 1'b1;
            end
            sel <= ~sel;
        end else begin
            result_valid <= 1'b0;
        end
    end

endmodule
```

### 1.3 Balanced Tree vs Linear Chain for Multi-Input Addition

```verilog
// BAD: Linear chain — depth = N-1 adders in series.
// PPA: timing grows linearly with input count.
// Each adder waits for the previous to complete.
wire [15:0] sum = a + b + c + d + e + f + g + h;

// GOOD: Balanced tree — depth = log2(N) adders.
// PPA: timing grows logarithmically. Same number of adders
//      but critically shorter path. Synthesis can place each
//      level on a separate timing group.

// Level 1: 4 pairwise adds
wire [15:0] s1_0 = a + b;
wire [15:0] s1_1 = c + d;
wire [15:0] s1_2 = e + f;
wire [15:0] s1_3 = g + h;

// Level 2: 2 pairwise adds
wire [16:0] s2_0 = {1'b0, s1_0} + {1'b0, s1_1};  // width grows
wire [16:0] s2_1 = {1'b0, s1_2} + {1'b0, s1_3};

// Level 3: final add
wire [17:0] tree_sum = {1'b0, s2_0} + {1'b0, s2_1};
```

---

## Section 2: Control Patterns

### 2.1 One-Hot FSM with Default State and Explicit Encoding

One-hot encoding: one flip-flop per state. Next-state logic reduces to single-bit comparisons. Best for FSMs with more than ~8 states on FPGA.

```verilog
module onehot_fsm (
    input  wire clk,
    input  wire rst_n,
    input  wire start,
    input  wire done,
    output reg  busy,
    output reg  [3:0] fsm_out
);

    // One-hot state encoding — explicit parameter prevents accidental binary encoding
    localparam [3:0] IDLE    = 4'b0001,
                     LOAD    = 4'b0010,
                     PROCESS = 4'b0100,
                     FINISH  = 4'b1000;

    reg [3:0] state, next_state;

    // State register
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            state <= IDLE;
        end else begin
            state <= next_state;
        end
    end

    // Next-state logic — one-hot means single-bit checks
    always @(*) begin
        next_state = state;  // default: hold current state
        case (1'b1)  // synthesis pragma parallel_case
            state[0]:  // IDLE
                if (start) next_state = LOAD;
            state[1]:  // LOAD
                next_state = PROCESS;
            state[2]:  // PROCESS
                if (done) next_state = FINISH;
            state[3]:  // FINISH
                next_state = IDLE;
            default:
                next_state = IDLE;  // safe recovery from corruption
        endcase
    end

    // Output logic — decoded from individual state bits
    always @(*) begin
        busy    = state[1] | state[2];  // LOAD or PROCESS
        fsm_out = state;
    end

endmodule
```

### 2.2 Binary FSM for Small State Spaces

Binary encoding: fewer flip-flops but wider decode logic. Best for 4-8 states where the decode logic is trivial.

```verilog
module binary_fsm (
    input  wire clk,
    input  wire rst_n,
    input  wire req,
    input  wire ack,
    output reg  grant
);

    localparam [1:0] IDLE   = 2'b00,
                     WAIT   = 2'b01,
                     ACTIVE = 2'b10;

    reg [1:0] state, next_state;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n)
            state <= IDLE;
        else
            state <= next_state;
    end

    always @(*) begin
        next_state = state;
        case (state)
            IDLE:   if (req)  next_state = WAIT;
            WAIT:   if (ack)  next_state = ACTIVE;
            ACTIVE:           next_state = IDLE;
            default: next_state = IDLE;  // explicit recovery
        endcase
    end

    always @(*) begin
        grant = (state == ACTIVE);
    end

endmodule
```

### 2.3 FSM with Explicit Idle Recovery

Every FSM must recover gracefully from unexpected states (e.g., radiation-induced bit flip, reset glitch). The pattern below forces any undefined state back to IDLE within one cycle.

```verilog
// Recovery wrapper: use inside next-state combinational block.
// If state register holds a value not in the defined localparam set,
// immediately transition to IDLE. This is NOT the same as the default
// case in a case statement — this catches actual register corruption.

always @(*) begin
    next_state = state;
    case (state)
        IDLE:    if (go)   next_state = RUN;
        RUN:     if (done) next_state = DONE;
        DONE:              next_state = IDLE;
        default: next_state = IDLE;  // catches both uncovered and corrupted states
    endcase
end
```

---

## Section 3: Storage Patterns

### 3.1 Register Array with Synchronous Reset

Use for shallow storage (< ~32 entries). Single-cycle read/write, no inference ambiguity.

```verilog
module reg_array #(
    parameter DEPTH = 16,
    parameter WIDTH = 32
)(
    input  wire                 clk,
    input  wire                 rst_n,
    input  wire                 wen,
    input  wire [$clog2(DEPTH)-1:0] waddr,
    input  wire [WIDTH-1:0]     wdata,
    input  wire [$clog2(DEPTH)-1:0] raddr,
    output wire [WIDTH-1:0]     rdata
);

    reg [WIDTH-1:0] mem [0:DEPTH-1];
    integer i;

    // Write port — synchronous write
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            for (i = 0; i < DEPTH; i = i + 1) begin
                mem[i] <= {WIDTH{1'b0}};
            end
        end else if (wen) begin
            mem[waddr] <= wdata;
        end
    end

    // Read port — asynchronous read (combinational)
    // PPA: single-cycle read latency. For timing-critical paths,
    //      register the read output and accept 1-cycle latency.
    assign rdata = mem[raddr];

endmodule
```

### 3.2 SRAM Inference: Simple Dual-Port

Use for deep storage (> ~32 entries) where area density matters. The pattern below infers block RAM on FPGA. Key: separate read and write ports, synchronous read.

```verilog
module ram_simple_dual #(
    parameter DEPTH = 256,
    parameter WIDTH = 32
)(
    input  wire                         clk,
    // Write port
    input  wire                         wen,
    input  wire [$clog2(DEPTH)-1:0]     waddr,
    input  wire [WIDTH-1:0]             wdata,
    // Read port
    input  wire                         ren,
    input  wire [$clog2(DEPTH)-1:0]     raddr,
    output reg  [WIDTH-1:0]             rdata
);

    reg [WIDTH-1:0] mem [0:DEPTH-1];

    // Write: synchronous
    always @(posedge clk) begin
        if (wen) begin
            mem[waddr] <= wdata;
        end
    end

    // Read: synchronous — required for BRAM inference.
    // PPA: 1-cycle read latency, but maps to dedicated BRAM instead of
    //      consuming LUTs and flip-flops. Area savings scale with depth.
    always @(posedge clk) begin
        if (ren) begin
            rdata <= mem[raddr];
        end
    end

    // Note: no reset on memory array. BRAM does not support synchronous reset
    // on the storage array. Initialize via a separate init sequence if needed.

endmodule
```

### 3.3 Clock Gating Wrapper

Gate clocks on large register banks when idle. Use ICG cells (ASIC) or clock-enable patterns (FPGA). Never gate clocks with manual AND gates.

```verilog
// FPGA: clock-enable pattern (synthesis maps to clock-enable flip-flops)
// PPA: reduces switching power on idle register banks without
//      creating glitchy clock trees.

module clock_gated_regs #(
    parameter WIDTH = 64,
    parameter DEPTH = 8
)(
    input  wire clk,
    input  wire rst_n,
    input  wire clk_en,        // clock enable: active high
    input  wire [$clog2(DEPTH)-1:0] addr,
    input  wire [WIDTH-1:0]    wdata,
    input  wire                wen,
    output wire [WIDTH-1:0]    rdata
);

    reg [WIDTH-1:0] regs [0:DEPTH-1];

    integer i;
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            for (i = 0; i < DEPTH; i = i + 1)
                regs[i] <= {WIDTH{1'b0}};
        end else if (clk_en) begin  // only update when enabled
            if (wen)
                regs[addr] <= wdata;
        end
    end

    assign rdata = regs[addr];

endmodule
```

---

## Section 4: Interface Patterns

### 4.1 Valid-Ready Handshake (Producer and Consumer Sides)

Standard handshake for intra-chip data transfer. `valid` and `ready` together indicate a transfer. Both sides may deassert independently.

```verilog
// Producer side: drives valid + data, accepts ready
// Transfer occurs on clock edge where valid AND ready are both high.
// PPA: no stall chain propagation. Backpressure is local to each link.

// Producer output signals
//   data_out_valid: asserted when data is available
//   data_out_ready: input from consumer (1 = can accept)
//   data_out:       data bus
// Transfer: data_out_valid & data_out_ready

// Consumer side: drives ready, accepts valid + data
// Consumer may deassert ready at any time; producer must hold data
// until ready is asserted.

module handshake_consumer #(
    parameter WIDTH = 32
)(
    input  wire             clk,
    input  wire             rst_n,
    // Input interface (consumer side)
    input  wire [WIDTH-1:0] data_in,
    input  wire             data_in_valid,
    output reg              data_in_ready,
    // Internal interface
    output reg  [WIDTH-1:0] payload,
    output reg              payload_valid
);

    // Simple consumer: always ready to accept when internal buffer is empty
    wire transfer = data_in_valid & data_in_ready;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            data_in_ready  <= 1'b1;  // ready by default
            payload        <= {WIDTH{1'b0}};
            payload_valid  <= 1'b0;
        end else begin
            if (transfer) begin
                payload       <= data_in;
                payload_valid <= 1'b1;
                data_in_ready <= 1'b0;  // internal buffer full
            end else if (!payload_valid) begin
                data_in_ready <= 1'b1;  // can accept again
            end
        end
    end

endmodule
```

### 4.2 AXI4-Lite Subordinate Interface Skeleton

Minimal AXI4-Lite subordinate for control register access. Only required channels (AW, W, B, AR, R). No burst support.

```verilog
module axi_lite_sub #(
    parameter ADDR_WIDTH = 12,
    parameter DATA_WIDTH = 32
)(
    input  wire                     clk,
    input  wire                     rst_n,
    // AXI4-Lite subordinate interface
    input  wire [ADDR_WIDTH-1:0]    s_axi_awaddr,
    input  wire                     s_axi_awvalid,
    output wire                     s_axi_awready,
    input  wire [DATA_WIDTH-1:0]    s_axi_wdata,
    input  wire [(DATA_WIDTH/8)-1:0] s_axi_wstrb,
    input  wire                     s_axi_wvalid,
    output wire                     s_axi_wready,
    output wire [1:0]               s_axi_bresp,
    output wire                     s_axi_bvalid,
    input  wire                     s_axi_bready,
    input  wire [ADDR_WIDTH-1:0]    s_axi_araddr,
    input  wire                     s_axi_arvalid,
    output wire                     s_axi_arready,
    output wire [DATA_WIDTH-1:0]    s_axi_rdata,
    output wire [1:0]               s_axi_rresp,
    output wire                     s_axi_rvalid,
    input  wire                     s_axi_rready
);

    // Write transaction state
    reg aw_ready, w_ready, b_valid;
    reg [ADDR_WIDTH-1:0] wr_addr;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            aw_ready <= 1'b1;
            w_ready  <= 1'b1;
            b_valid  <= 1'b0;
            wr_addr  <= {ADDR_WIDTH{1'b0}};
        end else begin
            // Accept write address
            if (s_axi_awvalid & aw_ready) begin
                wr_addr  <= s_axi_awaddr;
                aw_ready <= 1'b0;
            end
            // Complete write data phase
            if (s_axi_wvalid & w_ready) begin
                w_ready  <= 1'b0;
                b_valid  <= 1'b1;  // respond OKAY
            end
            // Complete write response
            if (s_axi_bready & b_valid) begin
                b_valid  <= 1'b0;
                aw_ready <= 1'b1;
                w_ready  <= 1'b1;
            end
        end
    end

    assign s_axi_awready = aw_ready;
    assign s_axi_wready  = w_ready;
    assign s_axi_bvalid  = b_valid;
    assign s_axi_bresp   = 2'b00;  // OKAY

    // Read transaction state (simplified: combinational read)
    reg ar_ready, r_valid_reg;
    reg [DATA_WIDTH-1:0] rd_data;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            ar_ready    <= 1'b1;
            r_valid_reg <= 1'b0;
            rd_data     <= {DATA_WIDTH{1'b0}};
        end else begin
            if (s_axi_arvalid & ar_ready) begin
                ar_ready    <= 1'b0;
                r_valid_reg <= 1'b1;
                // Placeholder: mux read data based on address
                rd_data     <= {DATA_WIDTH{1'b0}};
            end
            if (s_axi_rready & r_valid_reg) begin
                r_valid_reg <= 1'b0;
                ar_ready    <= 1'b1;
            end
        end
    end

    assign s_axi_arready = ar_ready;
    assign s_axi_rvalid  = r_valid_reg;
    assign s_axi_rdata   = rd_data;
    assign s_axi_rresp   = 2'b00;  // OKAY

endmodule
```

### 4.3 CDC Boundary with Synchronizer Instantiation

Never inline synchronizer flip-flops. Instantiate a dedicated CDC module from the CBB catalog. The pattern below shows the connection.

```verilog
// Single-bit CDC: use cdc_sync_bit from CBB catalog.
// PPA: Two-stage synchronizer adds 2-cycle latency but eliminates
//      metastability risk. Never double-flop a multi-bit bus.

// Instantiation pattern for single-bit CDC
cdc_sync_bit u_cdc_req_sync (
    .clk_dst  (clk_b),           // destination clock
    .rst_n    (rst_b_n),         // destination reset (sync to clk_b)
    .din      (req_from_clk_a),  // signal in source domain
    .dout     (req_synced)       // synchronized signal in clk_b domain
);

// Multi-bit CDC: use Gray-coded bus sync or handshake.
// NEVER double-flop a multi-bit bus — bits may arrive in different cycles.
// Use cdc_sync_bus with Gray encoding for counters, or handshake protocol
// for arbitrary data.

// Multi-bit CDC instantiation pattern
cdc_sync_bus #(
    .WIDTH (16)
) u_cdc_data_sync (
    .clk_dst  (clk_b),
    .rst_n    (rst_b_n),
    .din      (data_from_clk_a),
    .dout     (data_synced)
);
```

---

## Section 5: Arithmetic Patterns

### 5.1 DSP-Mapped Multiply-Accumulate

Structure the MAC to map directly to DSP48/EDSP blocks. Keep the form `P + A * B` without intermediate truncation.

```verilog
// DSP-mapped MAC pattern.
// PPA: maps cleanly to a single DSP48E1 slice on Xilinx or EDSP on Lattice.
//      Full-precision accumulation prevents rounding error accumulation.
//      Output truncation happens ONCE at the final result.

module dsp_mac #(
    parameter A_WIDTH = 18,
    parameter B_WIDTH = 18,
    parameter P_WIDTH = 48
)(
    input  wire                 clk,
    input  wire                 rst_n,
    input  wire                 clr,
    input  wire                 en,
    input  wire signed [A_WIDTH-1:0] a,
    input  wire signed [B_WIDTH-1:0] b,
    output wire signed [P_WIDTH-1:0] p
);

    reg signed [P_WIDTH-1:0] accum;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            accum <= {P_WIDTH{1'b0}};
        end else if (clr) begin
            accum <= {P_WIDTH{1'b0}};
        end else if (en) begin
            // Core MAC: P + A * B — maps to single DSP block
            accum <= accum + a * b;
        end
    end

    assign p = accum;

endmodule
```

### 5.2 Division: Iterative Restoring Divider

Variable division must NOT use the `/` operator. Use iterative approach (one bit per cycle) or approved IP.

```verilog
// Iterative restoring divider: 1 bit per cycle.
// PPA: Uses one subtractor and a shift register — minimal area.
//      Latency = dividend width cycles. Acceptable for non-critical-path
//      division. For high-throughput, use pipeline divider IP.

module iter_divider #(
    parameter WIDTH = 16
)(
    input  wire                 clk,
    input  wire                 rst_n,
    input  wire                 start,
    input  wire [WIDTH-1:0]     dividend,
    input  wire [WIDTH-1:0]     divisor,
    output reg  [WIDTH-1:0]     quotient,
    output reg  [WIDTH-1:0]     remainder,
    output reg                  done
);

    localparam ITER = WIDTH;

    reg [WIDTH-1:0] rem_reg;
    reg [WIDTH-1:0] quot_reg;
    reg [2*WIDTH-1:0] shifted;
    reg [$clog2(ITER+1)-1:0] count;
    reg busy;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            quotient  <= {WIDTH{1'b0}};
            remainder <= {WIDTH{1'b0}};
            done      <= 1'b0;
            busy      <= 1'b0;
            count     <= {( $clog2(ITER+1) ){1'b0}};
            rem_reg   <= {WIDTH{1'b0}};
            quot_reg  <= {WIDTH{1'b0}};
            shifted   <= {(2*WIDTH){1'b0}};
        end else if (start && !busy) begin
            // Initialize
            shifted  <= {dividend, {WIDTH{1'b0}}};
            count    <= ITER[ $clog2(ITER+1)-1 : 0 ];
            busy     <= 1'b1;
            done     <= 1'b0;
        end else if (busy) begin
            if (count == 0) begin
                quotient  <= quot_reg;
                remainder <= rem_reg;
                done      <= 1'b1;
                busy      <= 1'b0;
            end else begin
                // Shift left, subtract divisor
                shifted  <= shifted << 1;
                rem_reg  <= shifted[2*WIDTH-1 -: WIDTH];
                if (shifted[2*WIDTH-1 -: WIDTH] >= divisor) begin
                    shifted[2*WIDTH-1 -: WIDTH] <=
                        shifted[2*WIDTH-1 -: WIDTH] - divisor;
                    quot_reg <= (quot_reg << 1) | 1'b1;
                end else begin
                    quot_reg <= (quot_reg << 1);
                end
                count <= count - 1'b1;
            end
        end else begin
            done <= 1'b0;
        end
    end

endmodule
```

### 5.3 Saturation Logic

Clamp values to min/max range using comparison + mux, not overflow detection arithmetic.

```verilog
// Saturation: clamp signed result to symmetric range.
// PPA: comparison + mux maps to LUT + carry chain. One level of logic.
//      Much faster than overflow-detection arithmetic approaches.

module saturate #(
    parameter IN_WIDTH  = 24,
    parameter OUT_WIDTH = 16
)(
    input  wire signed [IN_WIDTH-1:0]  din,
    output wire signed [OUT_WIDTH-1:0] dout
);

    // Compute bounds
    localparam signed [IN_WIDTH-1:0] POS_MAX = (1 <<< (OUT_WIDTH - 1)) - 1;
    localparam signed [IN_WIDTH-1:0] NEG_MIN = -(1 <<< (OUT_WIDTH - 1));

    // Clamp logic: compare then select
    assign dout = (din > POS_MAX) ? POS_MAX[OUT_WIDTH-1:0] :
                  (din < NEG_MIN) ? NEG_MIN[OUT_WIDTH-1:0] :
                  din[OUT_WIDTH-1:0];

endmodule
```

---

## Section 6: CBB Instantiation Examples

### 6.1 Instantiating fifo_sync with Parameters

```verilog
// Named port connections. All parameters explicit.
// PPA: Using proven CBB avoids re-verification and ensures known timing.

fifo_sync #(
    .DEPTH (16),
    .WIDTH (DATA_W),
    .FWFT  (1)   // first-word-fall-through mode
) u_tx_fifo (
    .clk        (tx_clk),
    .rst_n      (tx_rst_n),
    // Write side
    .wr_en      (tx_fifo_wr),
    .wr_data    (tx_fifo_wdata),
    .full       (tx_fifo_full),
    // Read side
    .rd_en      (tx_fifo_rd),
    .rd_data    (tx_fifo_rdata),
    .empty      (tx_fifo_empty),
    // Status
    .count      (tx_fifo_count)  // optional: for watermark logic
);
```

### 6.2 Instantiating cdc_sync_bus with Connection Pattern

```verilog
// CDC bus sync: Gray-coded for counters, handshake for arbitrary data.
// Tie off unused outputs to named wires for debug visibility.

wire [PTR_WIDTH-1:0] rd_ptr_synced;

cdc_sync_bus #(
    .WIDTH (PTR_WIDTH)
) u_rd_ptr_cdc (
    .clk_dst  (wr_clk),
    .rst_n    (wr_rst_n),
    .din      (rd_ptr_gray),   // Gray-encoded read pointer from read domain
    .dout     (rd_ptr_synced)  // synchronized Gray pointer in write domain
);
```

### 6.3 Wrapping a CBB with Glue Logic

When a CBB partially matches the requirement, wrap it with adapter logic rather than modifying the CBB.

```verilog
// Wrapper: adapts a 32-bit FIFO to a 64-bit interface by
// packing two 32-bit writes into one 64-bit read.
// The CBB (fifo_sync) is unchanged — all adaptation is in the wrapper.

module fifo_32to64 #(
    parameter DEPTH = 32
)(
    input  wire        clk,
    input  wire        rst_n,
    // 32-bit write interface
    input  wire [31:0] wr_data,
    input  wire        wr_en,
    output wire        full,
    // 64-bit read interface
    output wire [63:0] rd_data,
    input  wire        rd_en,
    output wire        empty
);

    // Internal 32-bit FIFO at double depth
    wire [31:0] fifo_dout;
    wire        fifo_empty;
    wire        fifo_rd;
    wire        fifo_full;

    fifo_sync #(
        .DEPTH (DEPTH * 2),
        .WIDTH (32)
    ) u_fifo (
        .clk     (clk),
        .rst_n   (rst_n),
        .wr_en   (wr_en),
        .wr_data (wr_data),
        .full    (fifo_full),
        .rd_en   (fifo_rd),
        .rd_data (fifo_dout),
        .empty   (fifo_empty)
    );

    // Packing state: accumulate two 32-bit words
    reg [63:0] pack_reg;
    reg        pack_valid;

    assign fifo_rd = ~fifo_empty & ~pack_valid;

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            pack_reg   <= 64'b0;
            pack_valid <= 1'b0;
        end else if (fifo_rd) begin
            if (!pack_valid) begin
                pack_reg[31:0] <= fifo_dout;
                pack_valid     <= 1'b0;  // wait for second word
            end else begin
                pack_reg[63:32] <= fifo_dout;
                pack_valid       <= 1'b1;
            end
        end else if (rd_en && pack_valid) begin
            pack_valid <= 1'b0;
        end
    end

    assign full   = fifo_full;
    assign empty  = ~pack_valid;
    assign rd_data = pack_reg;

endmodule
```

---

## Quick-Reference Pattern Index

| Problem | Pattern | Section |
|---|---|---|
| Pipeline boundary with backpressure | Valid-ready skid buffer | 1.1 |
| Reduce multiplier count | Time-multiplexed shared multiplier | 1.2 |
| Multi-input addition timing | Balanced tree | 1.3 |
| Large FSM state encoding | One-hot with default recovery | 2.1 |
| Small FSM (4-8 states) | Binary encoding | 2.2 |
| FSM corrupted state recovery | Explicit default to IDLE | 2.3 |
| Shallow storage (< 32 entries) | Register array | 3.1 |
| Deep storage (> 32 entries) | SRAM inference (simple dual-port) | 3.2 |
| Reduce idle register power | Clock gating wrapper | 3.3 |
| Intra-chip data transfer | Valid-ready handshake | 4.1 |
| Control register access | AXI4-Lite subordinate | 4.2 |
| Signal crossing clock domains | CDC synchronizer instantiation | 4.3 |
| Multiply-accumulate | DSP-mapped MAC | 5.1 |
| Variable division | Iterative restoring divider | 5.2 |
| Clamp to output range | Saturation (compare + mux) | 5.3 |
| Instantiate standard FIFO | fifo_sync named ports | 6.1 |
| Cross-domain pointer sync | cdc_sync_bus | 6.2 |
| Adapt CBB to different interface | Wrapper with glue logic | 6.3 |
