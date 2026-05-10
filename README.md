# 8-bit CPU from scratch — Logisim Evolution

> A fully functional 8-bit CPU built from scratch in Logisim Evolution,
> starting from logic gates up to a complete microprogrammed control unit.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Instruction Set](#instruction-set)
- [Control Unit](#control-unit)
- [ALU Operations](#alu-operations)
- [FDE Cycle](#fde-cycle)
- [Program Examples](#program-examples)
- [Getting Started](#getting-started)
- [Project Roadmap](#project-roadmap)
- [Author](#author)

---

## Overview

This project implements a complete 8-bit CPU from the ground up,
following a bottom-up approach :
Transistors → Logic gates → Latches → Flip-flops → Registers
→ ALU → Bus → Control Unit → Full CPU

Built entirely in **Logisim Evolution**, the CPU features :
- A custom 16-instruction ISA
- ROM-based microprogrammed control unit (28 control signals)
- BCD decimal display for registers and ALU output
- Step-by-step and automatic execution modes
- Full jump logic with flag-based conditional branches

---

## Architecture
### Key Components

| Component | Description |
|-----------|-------------|
| **PC** | 8-bit program counter — points to next instruction |
| **MAR** | Memory Address Register — selects RAM location |
| **MDR** | Memory Data Register — bidirectional RAM buffer |
| **IR** | Instruction Register — holds current instruction |
| **R0 / R1** | General-purpose registers — ALU operands |
| **ALU** | 8-bit arithmetic and logic unit |
| **Flags** | Z, C, N, V — ALU status bits |
| **SP** | Stack Pointer — manages LIFO stack |
| **Control Unit** | ROM-based microprogrammed sequencer |

---

## Instruction Set

### Instruction Format

bit7  bit6  bit5  bit4          bit3  bit2  bit1  bit0

└─────────┘          └─────────┘

OPCODE (4 bits)        |        OPERAND (4 bits)

Control Unit           |       Internal bus / PC

### Full ISA Table

| Opcode | Binary | Mnemonic  | Action              | Cycles |
|--------|--------|-----------|---------------------|--------|
| 0      | 0000   | NOP       | No operation        | 5      |
| 1      | 0001   | LOAD R0   | R0 ← RAM[operand]   | 7      |
| 2      | 0010   | LOAD R1   | R1 ← RAM[operand]   | 7      |
| 3      | 0011   | ADD       | R0 ← R0 + R1        | 7      |
| 4      | 0100   | SUB       | R0 ← R0 - R1        | 7      |
| 5      | 0101   | AND       | R0 ← R0 AND R1      | 7      |
| 6      | 0110   | OR        | R0 ← R0 OR R1       | 7      |
| 7      | 0111   | STORE     | RAM[operand] ← R0   | 7      |
| 8      | 1000   | JMP       | PC ← operand        | 5      |
| 9      | 1001   | JZ        | PC ← op if Z=1      | 5      |
| 10     | 1010   | JC        | PC ← op if C=1      | 5      |
| 11     | 1011   | PUSH      | RAM[SP]←R0, SP--    | 7      |
| 12     | 1100   | POP       | R0←RAM[SP], SP++    | 7      |
| 13     | 1101   | MOV       | R0 ← R1             | 5      |
| 14     | 1110   | IMM       | R0 ← operand        | 5      |
| 15     | 1111   | HALT      | Stop CPU            | 5      |

### Encoding Examples

| Assembly      | Binary   | Hex  | Meaning              |
|---------------|----------|------|----------------------|
| LOAD R0, 0x03 | 00010011 | 0x13 | Load RAM[3] into R0  |
| ADD           | 00110000 | 0x30 | R0 = R0 + R1         |
| JMP 0x08      | 10001000 | 0x88 | Jump to address 0x08 |
| IMM 5         | 11100101 | 0xE5 | Load constant 5      |
| HALT          | 11110000 | 0xF0 | Stop execution       |

---

## Control Unit

ROM-based microprogrammed control unit with **28 control signals**.

### ROM Structure

Address (8 bits) = { state[3:0] , opcode[3:0] }
Output  (28 bits) = one bit per control signal
Size = 256 × 28 bits

### FDE Cycle — ROM States

| State | Action       | Active Signals        | ROM Value  |
|-------|--------------|-----------------------|------------|
| 0     | MAR ← PC     | PC_OUT, MAR_IN        | 0x00000009 |
| 1     | MDR ← RAM    | RAM_READ, MDR_IN      | 0x00020180 |
| 2     | IR  ← MDR    | MDR_OUT, IR_IN        | 0x00008010 |
| 3     | PC  ← PC+1   | PC_INC                | 0x00400000 |
| 4+    | Execute      | depends on opcode     | —          |



### Jump Logic

Conditional jumps combine `JMP_ENABLE` with flag values
through external AND gates and a MUX :
JMP_ENABLE AND FLAG_Z → JZ
JMP_ENABLE AND FLAG_C → JC
JMP_ENABLE AND FLAG_N → JN
JMP_ENABLE AND FLAG_V → JV
JMP_ENABLE            → JMP (unconditional)

---

## ALU Operations

| ALU_OP | Operation | Description              | Flag affected |
|--------|-----------|--------------------------|---------------|
| 000    | ADD       | R0 + R1                  | Z, C, N, V    |
| 001    | SUB       | R0 - R1                  | Z, C, N, V    |
| 010    | AND       | R0 AND R1 (bitwise)      | Z, N          |
| 011    | OR        | R0 OR  R1 (bitwise)      | Z, N          |
| 100    | XOR       | R0 XOR R1 (bitwise)      | Z, N          |
| 101    | NOT       | NOT R0                   | Z, N          |
| 110    | SHL       | R0 << 1  (× 2)           | Z, C, N       |
| 111    | SHR       | R0 >> 1  (÷ 2)           | Z, C, N       |

### Flags

| Flag | Name     | Set when                          |
|------|----------|-----------------------------------|
| Z    | Zero     | Result = 0                        |
| C    | Carry    | Unsigned overflow (result > 255)  |
| N    | Negative | Bit 7 of result = 1               |
| V    | Overflow | Signed overflow outside [-128,127]|

---

## FDE Cycle

Every instruction follows the same 4-cycle fetch,
followed by 1 to 3 execute cycles depending on the opcode.

---

## Program Examples

### Addition — a + b

```asm
; Memory layout
; 0x00 = a = 3
; 0x01 = b = 5
; 0x02 = result

LOAD R0, 0x00   ; R0 ← a
LOAD R1, 0x01   ; R1 ← b
ADD             ; R0 = a + b = 8
STORE 0x02      ; RAM[0x02] ← 8
HALT

; Hex file :
; v2.0 raw
; 03 05 00 10 21 30 72 F0
```

### While loop — countdown from 5

```asm
; 0x00 = 1  (constant decrement)
; 0x01 = i  (counter variable)

        IMM   5         ; i = 5
        STORE 0x01
LOOP:   LOAD  R0, 0x01  ; R0 ← i
        LOAD  R1, 0x00  ; R1 ← 1
        SUB             ; R0 = i - 1
        STORE 0x01      ; i ← R0
        JZ    END       ; if i == 0 → exit
        JMP   LOOP      ; else → repeat
END:    HALT
```


## Getting Started

### Prerequisites

- [Logisim Evolution](https://github.com/logisim-evolution/logisim-evolution) v3.x+
- Python 3.x (for the assembler)
- Java 11+ (required by Logisim Evolution)

### Run a program

```bash
# 1. Open the circuit
#    File → Open → logisim/cpu_main.circ

# 2. Load a program into RAM
#    Right-click RAM → Load Image
#    Select programs/addition.hex

# 3. Reset the CPU
#    Press RESET button

# 4. Execute
#    STEP mode : click once per clock cycle
#    AUTO mode : flip the RUN switch
```

### Write your own program

```bash
# Assemble a .asm file to .hex
python programs/assembler.py programs/my_program.asm

# Then load the generated .hex into RAM
```

---

## Project Roadmap

### Logisim Phase
- [x] Logic gates from scratch (AND, OR, NOT, NAND, NOR, XOR)
- [x] Half adder and full adder
- [x] 8-bit ripple carry adder
- [x] Complete ALU — 8 operations + Z C N V flags
- [x] SR latch, D latch, D flip-flop from NAND gates
- [x] 8-bit registers with tristate buffers
- [x] 8-bit internal bus
- [x] RAM, MAR, MDR
- [x] Instruction Register (IR)
- [x] BCD decimal display (Double Dabble + division method)
- [x] ROM-based control unit (28 signals)
- [x] Program Counter with full jump logic
- [x] Complete ROM programming and testing
- [ ] Python assembler
- [ ] Full program test suite

### FPGA Phase (Basys 3)
- [ ] Verilog port of the CPU
- [ ] Testbench and simulation (ModelSim)
- [ ] Timing analysis (Vivado)
- [ ] 5-stage pipeline (IF/ID/EX/MEM/WB)
- [ ] Forwarding and hazard detection
- [ ] Branch prediction
- [ ] L1 cache (direct-mapped)
- [ ] UART peripheral
- [ ] VGA controller
- [ ] RISC-V RV32I implementation
- [ ] Minimal SoC (CPU + UART + VGA)

---

## Repository Structure

cpu-8bit-logisim/

        ├── README.md               ← this file
        
        ├── logisim/
        
                │   ├── cpu_main.circ       ← main Logisim circuit
                
                │   ├── alu.circ            ← ALU subcircuit
                
                │   └── rom_content.txt     ← ROM values
        
        ├── docs/
        
                │   ├── architecture.md     ← detailed architecture
                
                │   ├── ISA.md              ← full instruction reference
                
                │   ├── control_signals.md  ← 28-signal table
                
                │   └── examples/
                
                │       ├── addition.md
                
                │       ├── while_loop.md
                
                │       └── multiplication.md
        
        ├── programs/
        
                │   ├── assembler.py        ← Python assembler
                
                │   ├── addition.hex
                
                │   ├── while_loop.hex
                
                │   └── multiplication.hex
        
        └── images/
        
                ├── cpu_overview.png
                
                ├── alu.png
                
                ├── control_unit.png
                
                └── demo.gif
        

---

## Limitations

This CPU was designed as a pedagogical project, not a production processor.
The constraints below are intentional — they reflect real architectural
trade-offs made when designing minimal hardware from scratch,
and each one motivates a concrete improvement in the next phase.

---

### 1. Memory — 16 addressable locations only

The operand field is **4 bits wide**, which restricts every LOAD, STORE,
and branch target to addresses `0x00`–`0x0F` — 16 bytes total,
shared between instructions and data.

In practice a typical program occupies 8–12 instruction slots,
leaving only 4–8 bytes for variables. This makes several classes
of programs impossible or impractical :

- **Multiplication by repeated addition** requires at least 4 variables
  (result, multiplicand, counter, constant 1) plus 12+ instructions —
  one slot too many. The 16-byte limit is the hard blocker.
- **Multiplication by bit-shifting** (powers of 2 only) is theoretically
  possible but would require SHL as a dedicated ISA instruction,
  displacing an existing opcode since all 16 are already assigned.
- **Arrays and strings** are impossible. There is no indirect addressing —
  every memory access uses a fixed literal operand baked into the instruction.
  `LOAD R0, [R1]` does not exist.
- **Nested function calls** beyond one or two levels are not feasible.
  Code and data compete for the same 16 slots, leaving almost no room
  for a real runtime call stack.

---

### 2. Registers — only two (R0 and R1)

Every ALU result is written back to R0, overwriting its previous value.
Any intermediate result needed later must be explicitly spilled to RAM
before the next operation — consuming both an instruction slot and a data slot.

With only two registers :

- Complex expressions require many LOAD/STORE round-trips to RAM.
- Every loop body carries 2–4 extra instructions just to reload
  values that were overwritten during the previous iteration.
- Expressions involving more than two operands at once, such as
  `(a + b) * (c - d)`, cannot be computed without RAM spilling
  or hardware stack support.

---

### 3. ISA — all 16 opcodes are used

The 4-bit opcode field allows exactly 16 instructions, and all 16
are currently assigned. There is no room to add new instructions
without replacing an existing one.

Notable missing instructions :

| Missing instruction | Impact |
|---------------------|--------|
| **JN** (jump if negative) | Signed comparisons require checking N and V together — `if (a < b)` in signed arithmetic cannot be implemented without it |
| **JV** (jump if overflow) | The V flag is computed and stored but can never trigger a branch |
| **MUL / DIV** | No hardware multiplier — software loops exceed available memory |
| **SHL / SHR** as ISA opcodes | The ALU supports shifts internally (ALU_OP 110/111) but no opcode exposes them to programs |
| **CMP** (compare without storing) | SUB always writes to R0, destroying the minuend |
| **XOR / NOT** as ISA opcodes | Present in the ALU but not accessible as standalone instructions |

---

### 4. Conditional branches — unsigned comparisons only

`JZ` and `JC` handle unsigned equality and carry correctly, but
**signed inequality comparisons are not supported**.

In two's complement arithmetic, the condition `a >= b` (signed) requires
checking that `N == V` after a subtraction — a compound condition
that neither `JZ` nor `JC` can express. Programs that need signed
inequalities (`<`, `>`, `<=`, `>=`) must work around this limitation
by restricting inputs to non-negative values (0–127),
which effectively reduces the useful signed range to unsigned.

---

### 5. No timing model in Logisim

Logisim Evolution is a **functional simulator**, not a timing simulator.
Every gate, register, and bus propagates its value instantaneously
within a simulation step. This is the right choice for a pedagogical tool
but hides important real-world concerns :

- **No propagation delays.** In real CMOS hardware, every gate introduces
  a delay of 0.1–1 ns. The critical path — the longest combinational chain
  from one register output to the next register input — sets the maximum
  clock frequency. On this CPU the critical path runs through the
  ripple-carry adder, the internal bus, and the register enable logic.
  In Logisim this path takes zero time.

- **No setup and hold times.** Real flip-flops require data to be stable
  for a minimum interval before and after the clock edge (setup time and
  hold time). Violating these constraints causes metastability.
  Logisim samples data instantly on the clock edge with no such requirement.

- **No glitches or combinational hazards.** In real logic, intermediate
  signal transitions before settling can cause momentary incorrect values.
  Logisim resolves all levels to their final stable state before any
  register samples them — hardware does not.

- **Clock frequency is unconstrained.** In simulation the CPU can run
  at any speed. On real silicon or FPGA the clock period must exceed
  the critical path delay, typically imposing a maximum of tens to
  hundreds of MHz depending on the logic depth and technology node.

---

## Perspectives — What a wider CPU would unlock

Each limitation above points directly to a natural next step.
This section describes what becomes possible when the constraints
of an 8-bit word, a 4-bit operand, and a functional simulator are lifted —
without assuming any specific implementation platform.

---

### More addressable memory

The root cause of the memory limitation is the **4-bit operand**.
Widening the instruction format to 32 bits would allow a 12-bit or larger
immediate field, giving access to thousands of memory locations instead of 16.

With more addressable memory :

- Instructions and data no longer compete for the same 16 slots.
- Variables, arrays, and strings become feasible.
- Programs can be hundreds of instructions long.
- Multiplication by repeated addition becomes trivial —
  the loop body, the counter, and the accumulator
  all fit comfortably without hitting a ceiling.

---

### More registers

With 32 registers instead of 2, intermediate results can stay
in registers across multiple operations.
The constant LOAD/STORE round-trips that bloat every loop body disappear.
Complex expressions involving several operands at once become natural,
and function calls can pass arguments and return values in registers
without touching memory at all.

---

### More instructions

Widening the opcode field from 4 bits to 7 bits allows dozens of instructions
instead of 16, with room to spare for extensions.

The most immediately useful additions would be :

- **Signed branch instructions** — dedicated opcodes for `<`, `>=`, etc.
  in signed arithmetic, removing the N/V workaround entirely.
- **MUL / DIV** — hardware multiplier and divider,
  making multiplication a single instruction rather than an impossible loop.
- **SHL / SHR** as proper ISA instructions — the ALU already supports
  shifts internally but has no opcode to invoke them from a program.
- **CMP** — compare two values and set flags without overwriting a register.
- **Indirect addressing** — `LOAD R0, [R1]` — the address comes from
  a register rather than a literal, enabling arrays and pointer arithmetic.

---

### Real performance constraints

The most conceptually important shift is moving from a functional simulator
to real hardware, where **time matters**.

In Logisim, every gate propagates its value instantaneously.
There are no propagation delays, no setup times, no hold times, and no glitches.
The simulator resolves all logic to its final stable state before any register
samples it. This is ideal for understanding what a circuit does,
but it hides everything about how fast it can do it.

On real hardware, every combinational gate introduces a small delay
(typically a fraction of a nanosecond). These delays add up along
every path from one register output to the next register input.
The longest such path — the **critical path** — determines the maximum
clock frequency : the clock period must be long enough for signals
to propagate all the way through and settle before the next edge arrives.

On this 8-bit CPU the critical path runs through the ripple-carry adder.
Each full adder stage waits for the carry from the previous one,
so the total delay grows linearly with the number of bits.
On real hardware this would cap the clock frequency significantly.
In Logisim it costs nothing.

This gap between simulation and reality motivates the two main
performance techniques used in real processor design :

**Pipelining** breaks the sequential fetch–decode–execute cycle
into overlapping stages. While one instruction is being executed,
the next is already being decoded, and the one after that is already
being fetched. Instead of one instruction completing every 8 cycles,
one instruction completes every cycle in steady state.
The hardware units that were sitting idle in the sequential design
are now all busy simultaneously.

```
Cycle :     1    2    3    4    5    6    7    8
Instr 1 :  IF   ID   EX  MEM   WB
Instr 2 :       IF   ID   EX  MEM   WB
Instr 3 :            IF   ID   EX  MEM   WB
Instr 4 :                 IF   ID   EX  MEM   WB
```

Pipelining introduces new problems — an instruction may need a result
that the previous instruction has not yet finished computing,
or a branch may change the PC while later instructions are already
partway through the pipeline. Handling these cases correctly
(by stalling, forwarding results early, or predicting branch outcomes)
is one of the central challenges of CPU microarchitecture.

**Clock frequency optimisation** requires analysing the critical path
and reducing its length — either by restructuring the logic
(replacing a ripple-carry adder with a carry-lookahead adder, for example)
or by inserting additional register stages to cut a long path into
shorter segments that each fit within the clock period.
This kind of timing-driven design does not exist in Logisim
but is unavoidable on any real implementation target.

## Author

**Aurélien Corty**
Electronics Engineering Student


---

## License

MIT License — feel free to use, modify and share.
