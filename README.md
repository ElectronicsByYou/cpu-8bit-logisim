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

└──────────────────┘          └──────────────────────┘

OPCODE (4 bits)                OPERAND (4 bits)

→ Control Unit                 → Internal bus / PC

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
Clock :  1    2    3    4    5    6    7
──── Fetch ────────  ── Execute ──
State :  0    1    2    3    4    5    6
Action: MAR  MDR   IR  PC+1  ...opcode...
←PC  ←RAM ←MDR  ++

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

### Multiplication — a × b (repeated addition)

```asm
; 0x00 = a = 3  (multiplicand)
; 0x01 = b = 4  (multiplier / counter)
; 0x02 = result = 0
; 0x03 = 1  (constant)

        IMM   0         ; result = 0
        STORE 0x02
        LOAD  R0, 0x01  ; counter = b
LOOP:   JZ    END       ; if counter == 0 → done
        LOAD  R1, 0x02  ; R1 = result
        LOAD  R0, 0x00  ; R0 = a
        ADD             ; R0 = result + a
        STORE 0x02      ; result ← R0
        LOAD  R0, 0x01  ; R0 = counter
        LOAD  R1, 0x03  ; R1 = 1
        SUB             ; counter - 1
        STORE 0x01      ; counter ← R0
        JMP   LOOP
END:    HALT
```

### If / else — compare a and b

```asm
; if (a == b) c = 1 else c = 0

        LOAD  R0, 0x00  ; R0 ← a
        LOAD  R1, 0x01  ; R1 ← b
        SUB             ; R0 = a - b
        JZ    EQUAL     ; if a == b → jump
        IMM   0         ; else c = 0
        STORE 0x02
        JMP   END
EQUAL:  IMM   1         ; c = 1
        STORE 0x02
END:    HALT
```

---

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
- [ ] Complete ROM programming and testing
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

## Author

**Aurélien Corty**
Electronics Engineering Student
Preparing for the French *Agrégation* in Electronics

---

## License

MIT License — feel free to use, modify and share.
