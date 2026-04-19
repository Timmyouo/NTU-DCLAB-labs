# NTU-DCLAB-labs

Lab assignments from a **digital circuits / FPGA** course at NTU, implemented in **SystemVerilog** on the **Terasic DE2-115 (Cyclone IV E)** development board.

> GitHub: [Timmyouo/NTU-DCLAB-labs](https://github.com/Timmyouo/NTU-DCLAB-labs)

---

## Capstone Project — FPGA Dart Game

The labs culminated in a **two-player physical dart game** on the DE2-115. Players physically throw an Arduino controller equipped with an **MPU6050 IMU**; the throw motion is captured, transmitted via **Bluetooth (HC-05)**, processed on the FPGA with a custom **physics engine** (fixed-point ballistic integration), and rendered over **VGA** with graphics stored in external **SRAM**.

[![Gameplay demo](https://img.youtube.com/vi/cwpWrTNM2AM/hqdefault.jpg)](https://youtu.be/cwpWrTNM2AM)

**Tech:** SystemVerilog · VGA · External SRAM · UART · Arduino · Python asset pipeline  
**Repo:** [Timmyouo/FPGA-Dart-Game](https://github.com/Timmyouo/FPGA-Dart-Game)

---

## Labs

| # | Directory | Topic | Key techniques |
|---|-----------|-------|----------------|
| 1 | [`lab01_fpga_intro/`](lab01_fpga_intro/) | FPGA Intro & LFSR Random Number | Synthesizable SV style, FSM, LFSR, seven-segment display |
| 2 | [`lab02_rsa256/`](lab02_rsa256/) | RSA-256 Hardware Accelerator | Montgomery multiplication, Avalon-MM master, RS-232 host interface |
| 3 | [`lab03_audio_i2c/`](lab03_audio_i2c/) | Audio Record / Playback over I²C | I²C init, WM8731 codec, SRAM audio buffer, variable-speed DSP |

---

## Toolchain

| Tool | Purpose |
|------|---------|
| Quartus II 15.0 | Synthesis, place & route, FPGA programming |
| Qsys (Platform Designer) | SoC integration for Avalon bus (Lab 2) |
| Synopsys VCS | RTL simulation where testbenches are provided |
| Python 2/3 + pyserial | Host-side test scripts (Lab 2) |

Board: **EP4CE115F29C7** on DE2-115, 50 MHz on-board oscillator.


