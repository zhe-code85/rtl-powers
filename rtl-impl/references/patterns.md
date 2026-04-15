# RTL Coding Pattern

These are reference patterns, not the main workflow. The main workflow lives in `../SKILL.md`.
Mandatory RTL coding and delivery rules live in `rtl-rules.md`. This file only provides preferred patterns that comply with those rules.

Keep this file minimal. Use one preferred coding style unless a project rule overrides it.

## Preferred Sequential Pattern

- Use one sequential register block plus combinational next-value blocks.
- Keep the registered state on the base signal name.
- Keep the combinational next value on the `*_comb` signal name.
- Drive only one `*_comb` signal per combinational block.
- Do not assign multiple next-value signals in the same `always @(*)` block.
- Default every `*_comb` to hold the current registered value so every path stays assigned.
- Make widths, truncation, concatenation, and sign extension explicit.
- Use explicit `$signed()` when signed arithmetic needs sign extension or width extension.
- Avoid `/` with high-width variable operands in RTL.
- Put reset handling only in the sequential block.

```verilog
module next_value_template #(
    parameter WIDTH = 8
)(
    input  wire             clk,
    input  wire             rst_n,
    input  wire             en,
    input  wire [WIDTH-1:0] data_i,
    output wire [WIDTH-1:0] data_o
);

    reg [WIDTH-1:0] data;
    reg [WIDTH-1:0] data_comb;

    always @(*) begin
        data_comb = data;

        if (en) begin
            data_comb = data_i;
        end
    end

    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            data <= {WIDTH{1'b0}};
        end else begin
            data <= data_comb;
        end
    end

    assign data_o = data;

endmodule
```

## FSM Pattern

- Keep the registered FSM state on the base signal name and compute `state_comb` in its own combinational block.
- Keep transition conditions local to the state block so ownership stays obvious during review.
- Decode control outputs in separate combinational logic when that keeps state transition logic shorter and clearer.

## Reuse Wrapper Pattern

- Keep the documented reuse boundary explicit: adaptation logic outside the reused block, named-port instantiation at the wrapper boundary, and local comments for interface or width translation.
- Terminate protocol adaptation, status remapping, and width shaping in the wrapper rather than editing the reused core contract.
- Record wrapper-only assumptions in the implementation trace so verification can target them directly.

```verilog
module fifo_wrapper_example (
    input  wire        clk,
    input  wire        rst_n,
    input  wire        in_valid,
    output wire        in_ready,
    input  wire [7:0]  in_data,
    output wire        out_valid,
    input  wire        out_ready,
    output wire [7:0]  out_data
);

    wire fifo_wen;
    wire fifo_ren;
    wire fifo_full;
    wire fifo_empty;

    assign fifo_wen = in_valid & in_ready;
    assign fifo_ren = out_valid & out_ready;
    assign in_ready = ~fifo_full;
    assign out_valid = ~fifo_empty;

    fifo_sync_std u_fifo_sync_std (
        .clk    (clk),
        .rst_n  (rst_n),
        .wen_i  (fifo_wen),
        .wdata_i(in_data),
        .full_o (fifo_full),
        .ren_i  (fifo_ren),
        .rdata_o(out_data),
        .empty_o(fifo_empty)
    );

endmodule
```

## CDC Structure Reminder

- Do not hide CDC behavior inside ad hoc RTL. Instantiate an approved synchronizer or CDC module when a signal crosses domains.
- Keep source-domain generation, CDC transport, and destination-domain consumption visibly separated in the code and trace.
