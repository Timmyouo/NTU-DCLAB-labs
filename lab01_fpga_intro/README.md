# Lab 1 — LFSR Slot-Machine: Synthesizable RTL & FSM Bring-Up

**Course:** Digital Circuits Lab, National Taiwan University (Fall 2025)
**Board:** Terasic DE2-115 (Intel Cyclone IV E, EP4CE115F29C7)
**Language:** SystemVerilog
**Toolchain:** Quartus II 15.0 · Synopsys VCS · SpyGlass / nLint

---

## Overview

A first-lab design exercise that combines a **3-state Mealy/Moore FSM** with a **16-bit Galois LFSR** to drive a **slot-machine-style random number display** on the DE2-115 seven-segment LEDs. Pressing `i_start` starts a "roll" whose update interval grows monotonically until the value freezes; pressing again triggers a new roll.

The lab serves as the course's introduction to a **synthesizable SystemVerilog coding style** — strict `_r` / `_w` register naming, single-driver `always_ff` / `always_comb` separation, no inferred latches — and to the **Quartus → Lint → Simulation → Hardware** sign-off flow used throughout the rest of the course.

---

## Microarchitecture

### FSM (`src/Top.sv`)

| State | Behaviour | Next-state condition |
|-------|-----------|----------------------|
| `S_IDLE` | Display `0`; LFSR keeps shifting every cycle to accumulate entropy. | `i_start` → `S_GEN` |
| `S_GEN` | Output is updated every `threshold` cycles. After 10 updates, `threshold += DELTA_THRESHOLD` per step (visual slow-down). After 20 updates → `S_HOLD`. | counter expires → `S_HOLD` |
| `S_HOLD` | Output is frozen (locked-in result). | `i_start` → `S_GEN` (re-roll) |

A single registered output drives the seven-segment decoder, eliminating glitches between state transitions.

### 16-bit Galois LFSR

Taps on bits `[0, 2, 3, 5]`:

```systemverilog
seed_next = {seed[0] ^ seed[2] ^ seed[3] ^ seed[5], seed[15:1]};
```

The seed advances **even in `S_IDLE`**, so the starting value of each roll is non-deterministic from the user's perspective (a common pseudo-randomness trick when no true entropy source is available).

### Timing constants

- Initial display interval: `INIT_THRESHOLD = 1_000_000` cycles ≈ **20 ms** at 50 MHz.
- Per-step slow-down after the 10th update: `DELTA_THRESHOLD = 500_000` cycles.

These are exposed as `parameter`s on the top module so the visual feel can be retuned without re-writing logic.

---

## Skills Demonstrated

- Writing fully **synthesizable SystemVerilog** with no latches, no multi-driver nets, and a clean `always_ff` / `always_comb` split.
- Designing a small but complete **FSM + datapath** (counter, threshold updater, LFSR, output register) and integrating it with on-board peripherals.
- Going through a full **lint → simulation → synthesis → on-board verification** loop on Quartus + VCS + SpyGlass.

---

## Repository Layout

```
lab01_fpga_intro/
├── src/
│   ├── Top.sv              # Top-level module (LFSR + FSM + output register)
│   └── DE2_115/            # Pin assignments, seven-segment decoder, SDC
├── sim/
│   ├── tb_Top.sv           # SystemVerilog testbench
│   ├── Top_test.sv         # Alternative SV test
│   └── Top_test.py         # Python-based simulation helper
├── include/
│   └── LAB1_include.sv     # Shared parameters / typedefs
├── lint/
│   └── Makefile            # SpyGlass / nLint invocation
└── team12_lab1_report.pdf  # Lab report
```

---

## Build & Verify

### Quartus synthesis

Open the project (`.qpf` / `.qsf`) in **Quartus II 15.0**, select target device **EP4CE115F29C7**, compile, and program the `.sof` to the board.

### VCS simulation

```bash
cd sim
vcs tb_Top.sv ../src/Top.sv -full64 -R -debug_access+all -sverilog +access+rw
# Open waveform viewer (nWave / DVE) to inspect signals
```

### Lint (SpyGlass / nLint)

```bash
cd lint && make
```

Common rules cleaned up before tape-out-style sign-off: combinational loop (22011), inferred latch (23003), incomplete case statement (23007).
