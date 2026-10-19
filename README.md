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

## Program Counter

Holds the address of the instruction the CPU will fetch next. It does not store the instruction itself, only where it lives in instruction memory. Its output feeds the address input of the instruction memory.

- Sequential logic: changes only on the rising clock edge
- 4 bits wide, so it covers 16 addresses (0 to 15) and wraps from 15 back to 0
- Inputs: `clk`, `rst`, `load`, `pc_in[3:0]`
- Output: `pc_out[3:0]`

| Condition | Behaviour |
|-----------|-----------|
| `rst = 1` | `pc_out` goes to 0 (start of the program) |
| `load = 1` | `pc_out` jumps to `pc_in` (used later for branches and jumps) |
| otherwise | `pc_out` increments by 1 every clock |

- Priority is reset first, then load, then increment
- Written with `always @(posedge clk)` and non-blocking `<=`
- Without `load`, the CPU can only run straight down the program, and loops and function calls need it
