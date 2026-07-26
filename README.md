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
