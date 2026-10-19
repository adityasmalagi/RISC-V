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

## Instruction Memory

Stores the program. The PC gives an address and the 8-bit instruction stored at that address comes out. It only looks up and returns the instruction, it never decides anything.

- ROM (read only): the CPU reads it while running and never writes to it
- Combinational: `inst` changes as soon as `addr` changes, no clock needed
- Input: `addr[3:0]` from the PC
- Output: `inst[7:0]`
- 4-bit address gives 16 locations, each 8 bits wide
- Written as a `case` on the address inside `always @(*)` with blocking `=`
- Every address without an instruction hits `default` and returns `00000000` (NOP)

### Instruction format
```
[7:6] opcode | [5:4] Rd | [3:2] Rs1 | [1:0] Rs2
```

The memory does not understand these fields. It only outputs the 8 bits, and the Control Unit and Register File split them up later.

### Stored program
| Addr | Instruction | Meaning |
|------|-------------|---------|
| 0 | `00_01_00_10` | ADD R1 = R0 + R2 |
| 1 | `01_11_01_00` | SUB R3 = R1 - R0 |
| 2 | `10_01_00_00` | LOAD R1 = Mem[R0] |
| 3 | `11_00_10_11` | STORE Mem[R2] = R3 |
| 4 | `00_10_01_11` | ADD R2 = R1 + R3 |
| 5 to 15 | `00_00_00_00` | NOP |

A NOP decodes as `ADD R0 = R0 + R0`. R0 is always 0 and ignores writes, so nothing changes.

### Why it is separate from data memory
Instruction memory is read only and holds the program. Data memory is read and write and holds data. Keeping them separate is called Harvard architecture, and it lets the CPU fetch an instruction and access data in the same cycle.

**Verified in simulation:** addr 1 to 4 return `74`, `90`, `cb`, `27` (hex), and addr 5, 6 and 15 return `00`.
