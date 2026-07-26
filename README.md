# Multi-Core-RISC-V-Cluster

A multi-core RISC-V compute cluster (4–8x RV32IM cores with a shared
interconnect and DMA-fed sample buffers) built to execute a real
software-defined-radio baseband pipeline as its target workload —
modulation, demodulation, channel estimation, and symbol/carrier
synchronization — partitioned and parallelized across cores.

## Overview

- **Cluster**: N-core RV32IM cluster (parametrized core count), shared
  L2/scratchpad, crossbar or ring interconnect, per-core private I/D
  memory for deterministic DSP timing
- **Inter-core sync**: hardware barriers/semaphores + a shared mailbox
  for pipeline-stage handoff between cores
- **DMA**: streaming DMA engine feeding I/Q sample buffers in/out of
  core-local memory without CPU-cycle overhead
- **Baseband workload** (the thing the cluster actually runs):
  - **Modulation**: QPSK/16-QAM symbol mapping and pulse shaping
  - **Demodulation**: matched filtering + symbol detection
  - **Channel estimation**: pilot-based LS/MMSE estimation
  - **Synchronization**: carrier frequency offset (CFO) correction and
    timing/symbol synchronization (e.g., Gardner/early-late)
- **Workload partitioning**: pipeline stages mapped to cores (e.g.,
  Core0 = sync, Core1 = channel estimation, Core2/3 = demod), with
  double-buffered handoff between stages
- **Verification**: per-core ISA-level checks + system-level bit-exact
  comparison against a Python/NumPy floating-point reference chain

## Architecture Overview

```
                         ┌────────────────────────┐
   I/Q samples in ──────▶│   Streaming DMA Engine  │
                         └───────────┬────────────┘
                                     ▼
                 ┌───────────────────────────────────┐
                 │        Shared Interconnect          │  crossbar / ring
                 │   (core-to-core + core-to-mem)      │
                 └──┬───────┬───────┬───────┬──────────┘
                    ▼       ▼       ▼       ▼
              ┌─────────┐┌─────────┐┌─────────┐┌─────────┐
              │ Core 0  ││ Core 1  ││ Core 2  ││ Core 3  │
              │  Sync   ││ Channel ││ Demod   ││  (Mod / │
              │ (CFO +  ││  Est.   ││(matched ││ spare / │
              │ timing) ││ (LS/    ││ filter +││ control)│
              │         ││  MMSE)  ││ detect) ││         │
              └────┬────┘└────┬────┘└────┬────┘└────┬────┘
                   │  hardware barrier / mailbox for   │
                   └──── stage handoff (double-buffered)┘
                                     ▼
                         ┌────────────────────────┐
                         │  Streaming DMA Engine   │──▶ recovered bits /
                         └────────────────────────┘    constellation out
```

---

## Repository Structure

```
mc-riscv-cluster/
├── README.md
├── LICENSE
├── .gitignore
├── Makefile                          
├── docs/
│   ├── cluster_architecture.md
│   ├── baseband_pipeline.md          
│   ├── workload_partitioning.md      
│   └── verification_plan.md          
│
├── rtl/
│   ├── core/
│   │   └── rv32im_core.sv            
│   ├── cluster/
│   │   ├── interconnect.sv           
│   │   ├── barrier_unit.sv           
│   │   ├── mailbox.sv                
│   │   └── cluster_top.sv
│   ├── mem/
│   │   ├── core_local_mem.sv         
│   │   └── shared_l2.sv
│   ├── dma/
│   │   └── streaming_dma.sv
│   └── common/
│       └── pkg_cluster_params.sv     
│
├── firmware/
│   ├── common/
│   │   ├── fixed_point.h             
│   │   └── mailbox_api.h
│   ├── sync/
│   │   └── sync_core.c               
│   ├── channel_est/
│   │   └── chanest_core.c             
│   ├── demod/
│   │   └── demod_core.c               
│   ├── mod/
│   │   └── mod_core.c                 
│   └── linker/
│       └── cluster.ld
│
├── verif/
│   ├── ref_model/
│   │   ├── baseband_chain.py        
│   │   └── channel_model.py         
│   ├── tb/
│   │   ├── core_tb.sv                
│   │   ├── cluster_tb.sv            
│   │   └── dma_tb.sv
│   ├── sva/
│   │   ├── barrier_assertions.sv     
│   │   └── mailbox_assertions.sv     
│   └── vectors/
│       └── gen_test_vectors.py       
│
├── sim/
│   └── verilator/
│       ├── Makefile
│       └── sim_main.cpp
│
├── analysis/
│   ├── ber_vs_snr.py                 
│   └── constellation_plot.py       
│
├── scripts/
│   ├── build_firmware.sh
│   └── run_regression.py
│
└── .github/
    └── workflows/
        └── ci.yml                    
```

---

## Tools

- Verilator ≥ 5.0
- RISC-V GNU toolchain (`riscv32-unknown-elf-gcc`) — firmware per core
- Python 3.10+ with NumPy/SciPy/Matplotlib (reference model, BER/SNR analysis)
- GTKWave for waveform debug

## Verification Approach

- **Reference model parity**: `verif/ref_model/baseband_chain.py` is a
  floating-point NumPy implementation of the same pipeline; fixed-point
  RTL/firmware output is compared against it within a defined error
  bound at each stage (sync, channel estimation, demod).
- **Per-stage unit tests**: each core's algorithm (sync, chan-est,
  demod, mod) is validated in isolation against known test vectors
  before integration.
- **System-level tests**: full 4-stage pipeline run across cores with
  injected AWGN + multipath channels at multiple SNR points; output
  compared bit-for-bit against the reference model's recovered bits.
- **Assertions (SVA)**: barrier/mailbox protocol correctness — no core
  advances to the next pipeline stage before all producers signal
  ready, no mailbox message is overwritten before being read.
- **Metrics**: BER vs. SNR curve and recovered constellation plots are
  generated automatically from regression output — a concrete,
  visual pass/fail signal beyond "testbench passed."
