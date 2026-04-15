# RTL Rules

This file is the single source of mandatory RTL coding and delivery rules for `rtl-impl`.
Project-specific rule files may add tighter constraints, but they must not relax or override the rules in this file.

## Delivery

- Default HDL is `Verilog`.
- Deliver synthesizable `Verilog` RTL only. Do not switch implementation syntax to `SystemVerilog`.
- Match nearby RTL naming, reset, and coding conventions unless they conflict with this file.

## Comments

- Comments must explain module intent, non-obvious implementation choices, wrapper behavior, pipeline cuts, and important assumptions.
- Do not write line-by-line narration comments.

## Reset

- Asynchronous reset branches must only assign constant values.

## Combinational Logic

- Combinational logic must assign a defined value on every path. Do not infer latches.
- When using next-value coding, each combinational block may drive only one next-value signal. Do not assign multiple `*_comb` signals in the same `always @(*)` block.

## Arithmetic

- Make widths, truncation, concatenation, and sign extension explicit. Do not rely on implicit tool inference.
- Use explicit `$signed()` where signed arithmetic needs sign extension or width extension.
- Avoid `/` with high-width variable operands in RTL.

## CDC

- Avoid hidden CDC logic. Use approved structures or dedicated modules.
