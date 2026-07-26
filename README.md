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
│   ├── baseband_pipeline.md          # modulation/demod/chan-est/sync algorithm notes
│   ├── workload_partitioning.md      # stage-to-core mapping, handoff protocol
│   └── verification_plan.md          # test plan, BER/SNR validation methodology
│
├── rtl/
│   ├── core/
│   │   └── rv32im_core.sv            # single-core RTL (or submodule to an existing core)
│   ├── cluster/
│   │   ├── interconnect.sv           # crossbar / ring fabric
│   │   ├── barrier_unit.sv           # hardware sync barriers
│   │   ├── mailbox.sv                # inter-core message passing
│   │   └── cluster_top.sv
│   ├── mem/
│   │   ├── core_local_mem.sv         # per-core scratchpad I/D memory
│   │   └── shared_l2.sv
│   ├── dma/
│   │   └── streaming_dma.sv
│   └── common/
│       └── pkg_cluster_params.sv     # core count, mem sizes, interconnect topology
│
├── firmware/
│   ├── common/
│   │   ├── fixed_point.h             # Q-format fixed-point helpers
│   │   └── mailbox_api.h
│   ├── sync/
│   │   └── sync_core.c                # CFO correction, timing sync (Gardner/early-late)
│   ├── channel_est/
│   │   └── chanest_core.c             # pilot-based LS/MMSE estimation
│   ├── demod/
│   │   └── demod_core.c               # matched filter + symbol detection
│   ├── mod/
│   │   └── mod_core.c                 # QPSK/16-QAM mapping, pulse shaping
│   └── linker/
│       └── cluster.ld
│
├── verif/
│   ├── ref_model/
│   │   ├── baseband_chain.py         # NumPy/SciPy floating-point reference pipeline
│   │   └── channel_model.py          # AWGN + multipath channel injector for test vectors
│   ├── tb/
│   │   ├── core_tb.sv                # per-core ISA-level sim
│   │   ├── cluster_tb.sv             # full-cluster system sim
│   │   └── dma_tb.sv
│   ├── sva/
│   │   ├── barrier_assertions.sv     # no stage advances before all cores signal ready
│   │   └── mailbox_assertions.sv     # no overwrite of unread mailbox message
│   └── vectors/
│       └── gen_test_vectors.py       # generates I/Q test vectors at target SNR points
│
├── sim/
│   └── verilator/
│       ├── Makefile
│       └── sim_main.cpp
│
├── analysis/
│   ├── ber_vs_snr.py                 # sweeps SNR, plots BER curve from sim output
│   └── constellation_plot.py         # recovered constellation visualization
│
├── scripts/
│   ├── build_firmware.sh
│   └── run_regression.py
│
└── .github/
    └── workflows/
        └── ci.yml                    # lint + core-level smoke tests on push
```

---

