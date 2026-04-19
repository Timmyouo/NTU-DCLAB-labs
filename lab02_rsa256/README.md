# Lab 2 — 256-bit RSA Hardware Accelerator with Avalon-MM / RS-232 Interface

**Course:** Digital Circuits Lab, National Taiwan University (Fall 2025)
**Board:** Terasic DE2-115 (Intel Cyclone IV E, EP4CE115F29C7)
**Languages:** SystemVerilog · Verilog · Python (host)
**Toolchain:** Quartus II 15.0 · Qsys / Platform Designer · Synopsys VCS · `pyserial`

---

## Overview

A complete hardware implementation of **256-bit RSA decryption** — `m = c^d mod n` — on FPGA, with end-to-end host integration over RS-232.

The design is structured in three clean layers, mirroring how a real cryptographic IP block would be productised:

1. **`Rsa256Core`** — pure arithmetic datapath: Montgomery-multiplication-based modular exponentiation.
2. **`Rsa256Wrapper`** — protocol / control layer: an **Avalon-MM master** that talks to the on-chip RS-232 IP, sequences operand transfers, drives the core, and ships the result back.
3. **Qsys system** — Avalon interconnect with the Intel RS-232 IP slave and the wrapper as a custom Avalon master.

The PC side runs a Python (and equivalent C++) script that streams `(n, d, c)` triples to the FPGA and prints the recovered plaintext.

---

## Algorithm & Datapath

### Why Montgomery?

A naïve `mod n` after every 256×256 multiplication would require a full 512-bit division — expensive in both area and latency on FPGA. **Montgomery multiplication** replaces the division with shifts and conditional subtractions, making the inner loop a tight bit-serial datapath ideal for hardware.

### Sub-modules inside `Rsa256Core`

| Module | Role | Latency |
|--------|------|---------|
| `RsaPrep` | Pre-compute `c · 2^256 mod n` (entry into Montgomery domain) | ~256 cycles |
| `RsaMont` | One 256-bit Montgomery multiply: `a · b · 2^-256 mod n` | ~256 cycles |

### Top-level FSM

| State | Action |
|-------|--------|
| `S_IDLE` | Wait for `i_start` |
| `S_PREP` | Run `RsaPrep` to enter Montgomery domain |
| `S_MONT` | Loop 256 bits of `d`: square (`t²`) every step, multiply (`m × t`) when `d[i] = 1` — left-to-right binary exponentiation |
| `S_DONE` | Assert `o_finished`; output `o_a_pow_d` |

Two Montgomery multiplications per exponent bit, ~256 cycles each → roughly **2 × 256 × 256 ≈ 130 k cycles** per decryption (≈ 2.6 ms at 50 MHz, neglecting protocol overhead).

---

## System Integration

### Avalon-MM custom master (`Rsa256Wrapper`)

The wrapper is a **bus master** rather than a memory-mapped slave: it actively issues `read` / `write` transactions to the RS-232 IP's status and data registers, using the **Avalon `waitrequest` / `readdatavalid` handshake**. This is a common pattern for FPGA accelerators that need to pull data straight out of an on-chip peripheral without involving a soft-CPU.

### Host protocol (RS-232, 115200 8N1)

```
PC → FPGA : 32 bytes  n  (modulus, big-endian)
PC → FPGA : 32 bytes  d  (private exponent, big-endian)
PC → FPGA : 32 bytes  c  (ciphertext block, big-endian)
FPGA      : compute  m = c^d mod n
FPGA → PC : 31 bytes  m  (plaintext, leading zero byte omitted)
            (loop for next ciphertext block)
```

### Qsys assembly

In Platform Designer the wrapper is exported as a custom Avalon master and connected to the Intel RS-232 IP slave through the Avalon interconnect, with clock and reset bridges. The PLL and reset controller are also instantiated in Qsys.

---

## Skills Demonstrated

- **Cryptographic hardware**: implementing Montgomery multiplication and binary modular exponentiation as a multi-cycle FSM-driven datapath.
- **Bus protocols**: hand-rolling an **Avalon-MM master** (request / waitrequest / readdatavalid handshake) and integrating it through **Qsys / Platform Designer**.
- **Mixed-language verification**: SystemVerilog testbench for `Rsa256Core`, plus an integration testbench for `Rsa256Wrapper` driven by hex vectors (`wrapper_input.txt` / `wrapper_output.txt`); cross-checked against a pure-Python RSA reference.
- **Host-FPGA bring-up**: writing a Python serial driver to feed real RSA test vectors to the board over RS-232 and validate plaintext byte-for-byte.

---

## Repository Layout

```
lab02_rsa256/
├── src/
│   ├── Rsa256Core.sv           # Core FSM + Montgomery sub-modules
│   ├── Rsa256Wrapper.sv        # Avalon master + RS-232 protocol controller
│   ├── DE2_115/                # Pin assignments, SDC
│   ├── tb_verilog/
│   │   ├── tb.sv               # Testbench for Rsa256Core
│   │   ├── test_wrapper.sv     # Testbench for Rsa256Wrapper
│   │   ├── PipelineCtrl.v      # Pipeline-control helper for TB
│   │   ├── PipelineTb.v        # Pipeline TB utility
│   │   ├── wrapper_input.txt   # Hex input vectors
│   │   └── wrapper_output.txt  # Expected output vectors
│   └── pc_python/
│       ├── rs232.py            # Host serial driver (send n, d, c; receive plaintext)
│       ├── rs232.cpp           # C++ equivalent
│       ├── key.bin / enc.bin   # Test vectors
│       └── golden/
│           ├── rsa.py          # Pure-Python RSA reference
│           ├── key.bin / enc*.bin
│           └── dec*.txt        # Expected plaintext outputs
├── Lab2_lecture.pdf            # Lab spec
├── Lab2_qsys_tuto1.pdf         # Qsys tutorial
└── team12_lab2_report.pdf      # Lab report
```

---

## Build & Verify

### Quartus + Qsys

1. In Platform Designer, instantiate the **Intel RS-232 IP**, an Altera PLL, and `Rsa256Wrapper` as a custom Avalon master; connect them through the Avalon interconnect (clock / reset bridges as needed).
2. Open the Quartus project (`.qpf` / `.qsf`), compile, and program the `.sof` to the DE2-115.
3. Connect the PC to the DE2-115 via an RS-232 cable.

### VCS simulation

```bash
# Test Rsa256Core in isolation
vcs src/tb_verilog/tb.sv src/Rsa256Core.sv \
    -full64 -R -debug_access+all -sverilog +access+rw

# Test Rsa256Wrapper end-to-end (file order matters)
vcs src/tb_verilog/test_wrapper.sv \
    src/tb_verilog/PipelineCtrl.v src/tb_verilog/PipelineTb.v \
    src/Rsa256Wrapper.sv src/Rsa256Core.sv \
    -full64 -R -debug_access+all -sverilog +access+rw
```

### Host driver

```bash
# Linux / macOS
python src/pc_python/rs232.py /dev/tty.usbserial-XXXX
# Windows
python src/pc_python/rs232.py COM3
```

Decrypts `enc.bin` using the private key in `key.bin` and prints the plaintext.

### Golden reference

```bash
python src/pc_python/golden/rsa.py d < enc.bin > plain.txt
diff plain.txt src/pc_python/golden/dec1.txt
```
