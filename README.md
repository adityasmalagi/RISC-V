# 4-bit RISC-V Style CPU in Verilog

A 4-bit CPU in Verilog, built one module at a time. Tool: Xilinx Vivado.

---
## ALU

Does the calculation. It is combinational logic: the output depends only on the current inputs, with no clock. Results wrap around in 4 bits (`15 + 1 = 0`).

| op | Operation |
|----|-----------|
| 00 | ADD |
| 01 | SUB |
| 10 | AND |
| 11 | OR |

- Inputs: `a[3:0]`, `b[3:0]`, `op[1:0]`
- Outputs: `result[3:0]`, `zero`
- `always @(*)` with blocking `=` makes it combinational
- `case(op)` picks the operation
- `zero` goes high when the result is 0

---

## Register File

Fast storage inside the CPU: 4 registers (R0 to R3), 4 bits each.

- Two read ports, combinational (no clock)
- One write port, on the rising clock edge when `wr_en = 1`
- R0 is hardwired to 0, so writes to it are ignored
- Inputs: `clk`, `wr_en`, `rd_addr`, `rs1_addr`, `rs2_addr`, `wr_data`
- Outputs: `rs1_data`, `rs2_data`
- The read ports are `assign` statements
- The write uses `always @(posedge clk)` with non-blocking `<=`
- R0 is forced back to 0 every clock, so it overrides any write to it

**Verified:** wrote R1=5, R2=9, R3=3, tried writing 7 to R0 (stayed 0), and read back all values correctly.

---

## Combinational vs Sequential

| Style | Used for | Verilog |
|-------|----------|---------|
| Combinational | ALU, register reads | `assign` or `always @(*)` with `=` |
| Sequential | Register write | `always @(posedge clk)` with `<=` |
