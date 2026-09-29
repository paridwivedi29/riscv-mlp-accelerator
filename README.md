

# riscv-mlp-accelerator

**fixed-point quantized MLP accelerator (Vitis HLS) integrated with a RISC-V core over AXI-Lite.**

![HLS](https://img.shields.io/badge/HLS-Vitis-blue) ![ISA](https://img.shields.io/badge/ISA-RV32I-green) ![FPGA](https://img.shields.io/badge/FPGA-Xilinx-orange) ![License](https://img.shields.io/badge/license-MIT-lightgrey)

A pruned and quantized multilayer perceptron inference accelerator, designed in C++/Vitis HLS, exported as RTL, and attached to a pipelined RV32I core as a memory-mapped peripheral. Based on the techniques in *"An Efficient and Low-Power MLP Accelerator Supporting Structured Pruning, Sparse Activations and Asymmetric Quantization"* (Lin, Chang, Huang, NYCU, AICAS 2021).


---

## Highlights

- **Network:** 17 → 32 → 16 → 10 MLP (10-class classification)
- **Structured pruning:** 50% structural pruning on L1 (64→32) and L2 (32→16)
- **Quantization:** fixed-point (`ap_fixed`, rounding + saturation enabled)
- **HLS-to-RTL flow:** Vitis HLS → synthesizable RTL → Vivado
- **SoC integration:** AXI-Lite register-mapped peripheral on a RISC-V memory bus, with a C driver
- **Verification:** C simulation, C/RTL co-simulation, and hardware-vs-software accuracy checks
- **Architectural extension (documented):** custom `mlpu.*` RISC-V instructions for tightly coupled offload

---

## Results

| Stage | Accuracy | Notes |
|---|---|---|
| FP32 baseline | 95% | Dense model |
| + 50% structured pruning | 93% | L1: 64→32, L2: 32→16 |
| + fixed-point quantization | 92% | `ap_fixed<W,I,AP_RND,AP_SAT>` |

| Metric | Value |
|---|---|
| Latency (cycles) | `TODO` |
| Clock (MHz) | `TODO` |
| LUT / FF / DSP / BRAM | `TODO` |
| Power (W) | `TODO` |

---

## Architecture

```
            +-------------------+        AXI-Lite        +----------------------+
            |   RISC-V (RV32I)  | <--------------------> |   MLP Accelerator    |
            |  5-stage pipeline |    control / status    |   (Vitis HLS IP)     |
            +---------+---------+                        |  L1 -> L2 -> L3      |
                      |                                  +----------+-----------+
                      |  memory bus                                 |
                +-----v------+                              +-------v-------+
                |  Data/Inst |                              | Weights (ROM) |
                |   Memory   |                              | Biases        |
                +------------+                              +---------------+
```

**Control flow:** software writes input data and sets `ap_start` via memory-mapped registers; the accelerator asserts `ap_done` (optionally raising an interrupt); software reads back the result / predicted class.

### Register map

| Offset | Register | Description |
|---|---|---|
| `0x00` | `CTRL` | start / done / idle |
| `0x10` | `INPUT` | input buffer |
| `0x20` | `OUTPUT` | output / result buffer |

> Update offsets to match your generated `xmlp_hw.h`.

### Custom instruction extension (documented, optional)

An alternative tightly coupled interface using a custom opcode with `mlpu.config_input`, `mlpu.config_weight`, `mlpu.config_output`, `mlpu.invoke`, `mlpu.result`, `mlpu.status`, and `mlpu.argmax`. The shipped integration uses the simpler AXI-Lite peripheral path; the ISA extension spec lives in [`docs/isa_extension.md`](docs/isa_extension.md).

---

## Repository structure

```
riscv-mlp-accelerator/
├── hls/                # Vitis HLS sources
│   ├── mlp.cpp
│   ├── mlp.h
│   ├── weights.h       # exported quantized weights/biases
│   └── mlp_tb.cpp      # C testbench
├── rtl/                # SystemVerilog wrapper + RISC-V core
├── sw/                 # C driver and demo program
├── python/             # training, pruning, quantization, export scripts
├── vivado/             # block design / constraints / build scripts
├── docs/               # diagrams, ISA extension spec, report
└── README.md
```



---

## Getting started

### Prerequisites

- Vitis HLS / Vivado (`TODO: version`)
- Python 3.x with `numpy`, `torch` (or your framework)
- RISC-V GCC toolchain (`riscv32-unknown-elf-gcc`)

### 1. Train, prune, quantize, export weights

```bash
cd python
python train.py
python prune.py
python quantize.py        # writes hls/weights.h
```

### 2. Run HLS (C sim, synthesis, co-sim, export)

```bash
cd hls
vitis_hls -f run_hls.tcl
```

### 3. Build the SoC in Vivado

```bash
cd vivado
vivado -mode batch -source build.tcl
```

### 4. Build and run the software

```bash
cd sw
make
```

---

## Design notes

- **Quantization:** rounding and saturation (`AP_RND`, `AP_SAT`) are enabled on the fixed-point type, and requantization is handled per layer to avoid overflow/clipping mismatches between software and hardware.
- **Pruning:** structured pruning keeps the datapath dense and regular, which maps cleanly to HLS pipelining.
- **Co-simulation:** `m_axi` ports need an explicit `depth` for C/RTL co-sim to pass.
- **Integration choice:** moved from a custom-instruction interface to a memory-mapped AXI-Lite peripheral for easier integration with off-the-shelf RISC-V cores, and kept the ISA extension as a documented design.

---

## Roadmap

- [ ] Activation-sparsity power gating
- [ ] Asymmetric quantization (per the base paper)
- [ ] Timing / power report on target FPGA
- [ ] Demo video / waveform screenshots

---

## Reference

Lin, Chang, Huang, *An Efficient and Low-Power MLP Accelerator Supporting Structured Pruning, Sparse Activations and Asymmetric Quantization*, IEEE AICAS 2021.


## Author

**Pari Dwivedi** ([@paridwivedi29](https://github.com/paridwivedi29)) · [LinkedIn](https://linkedin.com/in/your-handle)
