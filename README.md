# 16-bit BFloat16 Arithmetic Unit on FPGA

A hardware implementation of a **Brain Floating Point (BFloat16)** arithmetic unit supporting addition, subtraction, and multiplication, targeting the **Basys3 FPGA** (Xilinx Artix-7) and developed using **Vivado Design Suite**.

---

## Table of Contents

- [Overview](#overview)
- [Number Format](#number-format)
- [Project Structure](#project-structure)
- [Module Hierarchy](#module-hierarchy)
- [How to Use (On Hardware)](#how-to-use-on-hardware)
- [Pin Mapping Summary](#pin-mapping-summary)
- [Simulation](#simulation)
- [Resource Utilization](#resource-utilization)
- [Limitations](#limitations)
- [References](#references)

---

## Overview

This project implements a 16-bit floating-point ALU in Verilog HDL that performs:

| Operation      | Trigger          |
|----------------|------------------|
| Addition       | `sum` button (BTNU) — rising edge |
| Subtraction    | `sub` button (BTND) — rising edge |
| Multiplication | `sw[15]` switch toggled high (while not loading) |

Operands are loaded from the 16 on-board slide switches into two internal registers (`reg_a`, `reg_b`). The 16-bit result is displayed on the 16 on-board LEDs.

---

## Number Format

The **BFloat16** format uses 16 bits with the same exponent width as IEEE 754 float32:

```
Bit 15  : Sign (S)
Bits 14–7 : Exponent (E) — 8 bits, bias = 127
Bits 6–0  : Mantissa (M) — 7 bits (implicit leading 1 for normalized numbers)
```

**Value** = `(-1)^S × 2^(E-127) × 1.M`  (normalized, 0 < E < 255)

Special values follow IEEE 754 conventions:
- `E = 0xFF, M = 0` → Infinity
- `E = 0x00, M = 0` → Zero

---

## Project Structure

```
.
├── top_module.v          # Top-level: I/O handling, register loading, edge detection
├── main.v                # Arithmetic dispatcher (add/sub/mul routing)
├── bfloat_mul.v          # BFloat16 multiplier top-level
├── multiply_mantissa.v   # 10×10-bit mantissa multiplier
├── normalisation_mul.v   # Multiplier result normalization
├── compare_and_shift.v   # Exponent comparison and mantissa alignment (adder)
├── addition.v            # Signed mantissa addition
├── normalisation.v       # Adder result normalization
├── constraints.xdc       # Basys3 pin constraints
└── README.md
```

---

## Module Hierarchy

```
top_module
└── main
    ├── bfloat_mul
    │   ├── multiply_mantissa
    │   └── normalisation_mul
    ├── compare_and_shift
    ├── addition
    └── normalisation
```

### Module Descriptions

| Module | Type | Description |
|---|---|---|
| `top_module` | Sequential | Registers operands from switches; generates one-shot operation pulses via rising-edge detection |
| `main` | Sequential | Stores result in `o_res` register; selects between adder and multiplier outputs |
| `bfloat_mul` | Combinational | Computes sign (XOR), exponent (sum − 127), and normalized mantissa product |
| `multiply_mantissa` | Combinational | Computes 20-bit product of two 10-bit mantissas (maps to DSP48E1) |
| `normalisation_mul` | Combinational | Normalizes 20-bit product; adjusts exponent accordingly |
| `compare_and_shift` | Combinational | Right-shifts smaller mantissa to align exponents; outputs result exponent |
| `addition` | Combinational | Adds/subtracts signed mantissas; produces 11-bit sum |
| `normalisation` | Combinational | Extracts sign, converts to magnitude, left-shifts to normalize |

---

## How to Use (On Hardware)

### Step-by-step

1. **Load Operand A**
   - Set `sw[15:0]` to the BFloat16 bit pattern of operand A
   - Press and release `btn_load_a` (BTNL — left button)

2. **Load Operand B**
   - Set `sw[15:0]` to the BFloat16 bit pattern of operand B
   - Press and release `btn_load_b` (BTNR — right button)

3. **Trigger an Operation**

   | Operation | Action |
   |-----------|--------|
   | A + B | Press and release `sum` (BTNU — up button) |
   | A − B | Press and release `sub` (BTND — down button) |
   | A × B | Toggle `sw[15]` high (while neither load button is pressed) |

4. **Read the Result**
   - The 16-bit BFloat16 result appears on `led[15:0]`

5. **Reset**
   - Press `rst` (BTNC — center button) to clear all registers to zero

### Example: 1.0 + 1.0

| Step | Action |
|------|--------|
| 1 | Set switches to `0011 1111 1000 0000` (= `0x3F80` = 1.0 in BFloat16) |
| 2 | Press BTNL to load A |
| 3 | Switches already set to `0x3F80`; press BTNR to load B |
| 4 | Press BTNU (sum) |
| 5 | LEDs show `0100 0000 0000 0000` (= `0x4000` = 2.0 in BFloat16) ✓ |

---

## Pin Mapping Summary

| Signal | FPGA Pin | Board Resource |
|--------|----------|----------------|
| `clk` | W5 | 100 MHz oscillator |
| `rst` | U18 | BTNC (center) |
| `sum` | T18 | BTNU (up) |
| `sub` | U17 | BTND (down) |
| `btn_load_a` | W19 | BTNL (left) |
| `btn_load_b` | T17 | BTNR (right) |
| `sw[0]`–`sw[15]` | V17–R2 | Slide switches SW0–SW15 |
| `led[0]`–`led[15]` | U16–L1 | LEDs LD0–LD15 |

All I/O operates at **LVCMOS33** (3.3 V logic).

---

## Simulation

Functional simulation was carried out in **Vivado XSim**. The testbench drives `i_a`, `i_b`, and one-cycle pulses on `sum`, `sub`, `mul` directly into the `main` module. Key verified cases:

| A | B | Op | Expected Result |
|---|---|----|-----------------|
| `0x3F80` (1.0) | `0x3F80` (1.0) | + | `0x4000` (2.0) |
| `0x4000` (2.0) | `0x3F80` (1.0) | − | `0x3F80` (1.0) |
| `0x3F80` (1.0) | `0x3F80` (1.0) | × | `0x3F80` (1.0) |
| `0x7F80` (∞) | `0x3F80` (1.0) | + | `0x7F80` (∞) |
| `0x0000` (0) | `0x4040` (3.0) | + | `0x4040` (3.0) |
| `0x3F80` (1.0) | `0xBF80` (−1.0) | + | `0x0000` (0) |
| `0x4100` (8.0) | `0x4000` (2.0) | × | `0x4200` (16.0) |

---

## Resource Utilization

Synthesized for **Artix-7 XC7A35T** at 100 MHz:

| Resource | Used | Available | % |
|----------|------|-----------|---|
| Slice LUTs | ~312 | 20,800 | ~1.5% |
| Slice Registers | ~48 | 41,600 | ~0.1% |
| DSP48E1 Slices | 1 | 90 | ~1.1% |
| I/O Buffers | 39 | 210 | ~18.6% |

Timing: **No violations at 100 MHz** (WNS ≈ +1.8 ns).

---

## Limitations

- **No rounding**: The mantissa is truncated (not rounded to nearest-even), which may accumulate error in chained computations.
- **NaN not explicitly handled**: Inputs with E = 0xFF and M ≠ 0 are treated the same as infinity.
- **Subnormal arithmetic**: Denormalized operands (E = 0) are recognized but full subnormal computation is not guaranteed.
- **Combinational normalization**: The iterative shift loop is synthesized as a combinational priority chain, limiting normalization to 8 left-shift steps.

---

## References

1. Xilinx Inc. *Basys3 Reference Manual*. Digilent, Inc., 2016.
2. IEEE. *IEEE Standard for Floating-Point Arithmetic*. IEEE Std 754-2019.
3. Wang N. et al. *BFloat16: The Secret to High Performance on Cloud TPUs*. Google AI Blog, 2019.
4. Xilinx Inc. *Vivado Design Suite User Guide: Synthesis (UG901)*, 2022.
5. Digilent Inc. *Basys3 Master XDC*. https://digilent.com/reference/programmable-logic/basys-3/start

---

*Course: Digital Electronic Circuits (ECE, 2024) | Instructor: Dr. Srinivas Boppu | Platform: Basys3 FPGA*
