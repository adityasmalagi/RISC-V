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

## Control Unit

The brain of the CPU. It reads the 2-bit opcode of the instruction and sets the control signals that tell every other module what to do this cycle. It does no calculation and stores nothing.

- Combinational logic: no clock, the signals change as soon as the opcode changes
- Input: `opcode[1:0]` from `inst[7:6]`
- Outputs: `wr_en`, `alu_op[1:0]`, `mem_read`, `mem_write`, `mem_to_reg`
- Written with `always @(*)` and blocking `=`
- Every output is given a default value at the top of the block, so no output is left unassigned and no latch is inferred in synthesis
- The `case` on the opcode only changes the signals that differ from the defaults

### Control signals
| Signal | Purpose |
|--------|---------|
| `wr_en` | 1 = register file writes the result |
| `alu_op` | Operation for the ALU: 00 ADD, 01 SUB |
| `mem_read` | 1 = data memory outputs data (LOAD) |
| `mem_write` | 1 = data memory stores data (STORE) |
| `mem_to_reg` | Write-back select: 0 = ALU result, 1 = memory data |

### Truth table
| Opcode | Instruction | wr_en | alu_op | mem_read | mem_write | mem_to_reg |
|--------|-------------|-------|--------|----------|-----------|------------|
| 00 | ADD | 1 | 00 | 0 | 0 | 0 |
| 01 | SUB | 1 | 01 | 0 | 0 | 0 |
| 10 | LOAD | 1 | 00 | 1 | 0 | 1 |
| 11 | STORE | 0 | 00 | 0 | 1 | 0 |

### Why the values are set this way
- **ADD / SUB:** the ALU computes and the result is written back to the register
- **LOAD:** the ALU computes the address, memory is read, and `mem_to_reg = 1` writes the memory data into the register
- **STORE:** `wr_en = 0` so no register is overwritten, and `mem_write = 1` stores the value into memory

### Connections
```
Instruction Memory -> inst[7:6] -> Control Unit
                                     -> wr_en       -> Register File
                                     -> alu_op      -> ALU
                                     -> mem_read    -> Data Memory
                                     -> mem_write   -> Data Memory
                                     -> mem_to_reg  -> Write-back MUX
```
## Data Memory

The RAM of the CPU. It stores the data values the program works on. STORE writes a register value into memory, and LOAD reads a value from memory into a register.

- RAM (read and write), unlike the instruction memory which is read only
- Write is sequential: it happens on the rising clock edge when `mem_write = 1`, using `always @(posedge clk)` with `<=`
- Read is combinational: the data appears as soon as the address changes when `mem_read = 1`, using `assign`
- When `mem_read = 0`, `rd_data` is forced to 0
- 4-bit address gives 16 locations, each 4 bits wide
- An `initial` block preloads the memory so a LOAD reads real values instead of `X` in simulation. It is for simulation only and does not synthesize

### Ports
| Port | Direction | Purpose |
|------|-----------|---------|
| `clk` | input | Writes happen on the rising edge |
| `mem_read` | input | 1 = output the data at `addr` |
| `mem_write` | input | 1 = store `wr_data` at `addr` |
| `addr[3:0]` | input | Memory location |
| `wr_data[3:0]` | input | Value to store |
| `rd_data[3:0]` | output | Value read from `addr` |

### Control signals
The control unit decides when to read and write:
- LOAD sets `mem_read = 1`
- STORE sets `mem_write = 1`

### Instruction memory vs data memory
| | Instruction Memory | Data Memory |
|---|---|---|
| Type | ROM | RAM |
| Operations | Read only | Read and write |
| Clock | None | Write on clock edge |
| Holds | The program | The data |
| Width | 8 bits | 4 bits |

Keeping the two separate is called Harvard architecture. It lets the CPU fetch an instruction and access data in the same cycle.

### Connections
```
rs1_data       -> addr      (where to read or write)
rs2_data       -> wr_data   (the value a STORE writes)
Control Unit   -> mem_read, mem_write
rd_data        -> Write-back MUX -> Register File
```

## Top Level CPU

Connects all six modules into a single-cycle 4-bit CPU. It has no logic of its own except the write-back mux. The only ports are `clk` and `rst`, and the internal signals are observed in simulation.

- Structural design: each module is instantiated once and connected with named ports
- The instruction is split into fields: `[7:6]` opcode, `[5:4]` Rd, `[3:2]` Rs1, `[1:0]` Rs2
- The PC, register file write and data memory write are clocked. Everything else is combinational
- Write-back mux: `mem_to_reg` selects the ALU result (0) or the data memory output (1)
- Branches are not supported yet, so the PC `load` input is tied to 0
- Data memory address comes from `rs1_data`, so LOAD is `Rd = Mem[Rs1]` and STORE is `Mem[Rs1] = Rs2`
- Registers and data memory are preloaded with an `initial` block for simulation only (R2 = 4, dm[0..3] = 5, 10, 3, 7)

### Data flow
```
PC -> Instruction Memory -> Control Unit
                         -> Register File -> ALU -> Write-back MUX -> Register File
                                          -> Data Memory -> Write-back MUX
```

### Test program
| Addr | Instruction | Meaning | Result |
|------|-------------|---------|--------|
| 0 | `00_01_00_10` | ADD R1 = R0 + R2 | R1 = 4 |
| 1 | `01_11_01_00` | SUB R3 = R1 - R0 | R3 = 4 |
| 2 | `10_01_00_00` | LOAD R1 = Mem[R0] | R1 = 5 |
| 3 | `11_00_10_11` | STORE Mem[R2] = R3 | Mem[4] = 4 |
| 4 | `00_10_01_11` | ADD R2 = R1 + R3 | R2 = 9 |
| 5 to 15 | `00_00_00_00` | NOP | No change |

### Verified in simulation
- The PC counts 0 to 5 and beyond, one step per clock
- Final registers: R0 = 0, R1 = 5, R2 = 9, R3 = 4
- STORE wrote 4 into data memory address 4, and no register was written during the STORE
- LOAD read 5 from address 0 into R1 through the write-back mux

### Bug found during debugging
The PC output stayed at 1 instead of counting because the increment used `pc_in + 1` instead of `pc_out + 1`. It was fixed by incrementing the current PC value.
