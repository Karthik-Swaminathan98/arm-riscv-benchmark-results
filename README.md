# arm-riscv-dsp-benchmark

> Cycle-accurate benchmark of **CMSIS-DSP/NN vs NMSIS/Andes DSP/NN** on ARM Cortex-M4
> and RISC-V Andes D25F — **real silicon**, not simulation.

Conducted as part of a Master's thesis at **TU Chemnitz** in collaboration with
**Infineon Technologies**, Dresden (2025).

![Language](https://img.shields.io/badge/language-C%20%2F%20C%2B%2B-blue)
![Platform](https://img.shields.io/badge/platform-ARM%20Cortex--M4%20%7C%20RISC--V-informational)
![Libraries](https://img.shields.io/badge/libraries-CMSIS%20%7C%20NMSIS%20%7C%20Andes-orange)
![Status](https://img.shields.io/badge/status-thesis%20complete-success)

---

## Key Results (normalised to CMSIS baseline = 1.0)

> Values > 1.0 = RISC-V library outperforms CMSIS. Values < 1.0 = CMSIS wins.

### DSP kernels

| Function | Library | Cycle speedup vs CMSIS | Stack efficiency vs CMSIS |
|---|---|---|---|
| FFT F32 (1024) | ANDES | 1.47× faster | 9.96× less stack |
| FFT F32 (1024) | NMSIS | 1.47× faster | 3.41× less stack |
| Magnitude Q15 | ANDES | 3.32× faster | 14.8× less stack |
| FIR Q15 | NMSIS | 2.14× faster | 2.64× less stack |
| FIR F32 | ANDES | 0.74× (CMSIS wins) | 2.29× less stack |

### NN kernels (NMSIS vs CMSIS)

| Operator | Cycle speedup | Instruction speedup |
|---|---|---|
| Convolution S16 | 1.25× | 1.35× |
| Convolution S8 | 1.31× | 1.52× |
| Fully Connected S8 | 1.13× | 1.14× |
| Softmax S8 | 1.15× | 1.23× |
| Pooling S8 | 1.08× | 1.26× |

---

## Hardware

| Feature | CY8CKIT-062 (ARM) | Telink B91 (RISC-V) |
|---|---|---|
| Core | Cortex-M4 (Armv7E-M) | Andes D25F (RV32IMACFDBP) |
| Pipeline | 3-stage | 5-stage |
| Clock (bench) | 25 MHz | 24 MHz |
| SRAM | 288 KB | 256 KB |
| Key extensions | FPU, DSP, SIMD | FPU, DSP, P-extension SIMD |
| Toolchain | GNU ARM 13.3, ModusToolbox 3.4 | riscv-32-elf-gcc 7.4, AndeSight RDS |

Both boards ran code **entirely from RAM** (no flash wait-states).
Interrupts disabled during measurement. Each kernel built as a separate binary.

---

## Measurement methodology

### Cycle count

**ARM Cortex-M4** — DWT (Data Watchpoint and Trace) hardware cycle counter:

```c
CoreDebug->DEMCR |= CoreDebug_DEMCR_TRCENA_Msk;
DWT->CYCCNT = 0;
DWT->CTRL  |= DWT_CTRL_CYCCNTENA_Msk;

uint32_t start = DWT->CYCCNT;
benchmark_function();
uint32_t cycles = DWT->CYCCNT - start;
```

**RISC-V Andes D25F** — NDS_MCYCLE CSR:

```c
write_csr(NDS_MCYCLE, 0);
uint32_t start = read_csr(NDS_MCYCLE);
benchmark_function();
uint32_t cycles = read_csr(NDS_MCYCLE) - start;
```

### Instruction count

**ARM** — estimated from DWT auxiliary counters:
```
Instructions = CYCCNT − CPICNT − EXCCNT − SLEEPCNT − LSUCNT + FOLDCNT
```

**RISC-V** — directly from NDS_MINSTRET CSR:
```c
write_csr(NDS_MINSTRET, 0);
uint32_t start_inst = read_csr(NDS_MINSTRET);
benchmark_function();
uint32_t instr = read_csr(NDS_MINSTRET) - start_inst;
```

### Stack usage — stack-paint technique

```c
void fill_stack_pattern_to_sp(void) {
    register uint32_t *sp;
    __asm volatile ("mov %0, sp" : "=r" (sp));
    uint32_t *p = (uint32_t*)&__StackLimit;
    while (p < sp) { *p++ = 0xAAAAAAAA; }
}

uint32_t measure_stack_usage(void) {
    register uint32_t *sp;
    __asm volatile ("mov %0, sp" : "=r" (sp));
    uint32_t *p = (uint32_t*)&__StackLimit;
    while (p < sp && *p == 0xAAAAAAAA) { p++; }
    return ((uint32_t)sp - (uint32_t)p);
}
```

Identical code runs on both ARM and RISC-V targets.

### Code size

Measured using linker `.map` file + `objdump` disassembly.
A Python script builds a dependency-aware call graph and computes
**total binary footprint** including all transitive dependencies.

See: [embedded-benchmark-analysis](https://github.com/Karthik-Swaminathan98/embedded-benchmark-analysis)

---

## Repository structure

```
arm-riscv-dsp-benchmark/
├── arm/
│   ├── dsp/                    # CMSIS-DSP kernels (FFT, FIR, Magnitude, Fast Math)
│   ├── nn/                     # CMSIS-NN kernels (Conv, FC, Pooling, Softmax)
│   └── common/
│       ├── dwt_counter.h       # DWT cycle/instruction counter
│       ├── stack_paint.c/.h    # Stack usage measurement
│       └── benchmark_loop.h    # Shared harness
├── riscv/
│   ├── dsp/                    # NMSIS-DSP + Andes-DSP kernels
│   ├── nn/                     # NMSIS-NN kernels
│   └── common/
│       ├── csr_counter.h       # NDS_MCYCLE / NDS_MINSTRET reader
│       └── stack_paint.c       # Stack measurement (shared)
├── analysis/
│   ├── code_size_analyser.py   # objdump call-graph + .map parser
│   └── plot_results.py         # Chart generation
└── results/
    └── summary_table.csv       # Full numerical results
```

---

## Libraries used

| Library | Architecture | Source |
|---|---|---|
| CMSIS-DSP v1.10 | ARM | [ARM-software/CMSIS_6](https://github.com/ARM-software/CMSIS_6) |
| CMSIS-NN | ARM | [ARM-software/CMSIS-NN](https://github.com/ARM-software/CMSIS-NN) |
| NMSIS-DSP/NN | RISC-V | [Nuclei-Software/NMSIS](https://github.com/Nuclei-Software/NMSIS) |
| Andes-DSP | RISC-V | Precompiled (P-extension) via Andes Technology |

---

## Acknowledgements

Master's thesis at **Technische Universität Chemnitz**
(Chair of Computer Architectures and Systems)
in collaboration with **Infineon Technologies**, Dresden.

Supervised by Prof. Dr. Alejandro Masrur · Mr. Daniel Markert ·
Dr. Elias Trommer · Mr. Jerome Almon Swamidasan

---

## Author

**Karthik Swaminathan** — Embedded Firmware Engineer
M.Sc. Embedded Systems · TU Chemnitz
[LinkedIn](https://linkedin.com/in/karthik-swaminathan98) · karthik94870@gmail.com
