# arm-riscv-benchmark-results

> Complete cross-architecture performance analysis of **ARM Cortex-M4 (CMSIS) vs RISC-V Andes D25F (NMSIS / Andes)**
> across DSP kernels, Neural Network operators, and full model inference —
> measured on **real silicon**, reproducible, cycle-accurate.

Master's thesis — **TU Chemnitz** × **Infineon Technologies**, Dresden (2025).

**Source code:**
- ARM benchmarks → [ARM-Project](https://github.com/Karthik-Swaminathan98/ARM-Project)
- RISC-V benchmarks → [RISCV-Project](https://github.com/Karthik-Swaminathan98/RISCV-Project)
- Code size analysis → [mcu-function-size-analyser](https://github.com/Karthik-Swaminathan98/mcu-function-size-analyser)

![ARM](https://img.shields.io/badge/ARM-Cortex--M4%20PSoC6-blue)
![RISCV](https://img.shields.io/badge/RISC--V-Andes%20D25F%20Telink%20B91-orange)
![Libraries](https://img.shields.io/badge/libraries-CMSIS--DSP%20%7C%20CMSIS--NN%20%7C%20NMSIS--DSP%20%7C%20NMSIS--NN%20%7C%20Andes--DSP-red)
![Thesis](https://img.shields.io/badge/thesis-TU%20Chemnitz%20%2F%20Infineon%202025-green)

---

## Hardware

| | ARM Platform | RISC-V Platform |
|---|---|---|
| Board | CY8CKIT-062-WIFI-BT (PSoC6) | Telink B91 |
| Core | Cortex-M4 (Armv7E-M) | Andes D25F (RV32IMACFDBP) |
| Pipeline | 3-stage in-order | 5-stage in-order |
| Clock (bench) | 25 MHz | 24 MHz |
| SRAM | 288 KB | 256 KB |
| SIMD extensions | DSP, FPU, VFP | P-extension (packed SIMD), FPU |
| Toolchain | GNU ARM 13.3, ModusToolbox 3.4 | riscv-32-elf-gcc 7.4, AndeSight RDS |
| Cycle counter | DWT->CYCCNT | NDS_MCYCLE CSR |
| Instruction counter | DWT auxiliary (estimated) | NDS_MINSTRET CSR (exact) |

All code executes **from RAM only** — no flash wait-states.
Interrupts disabled during measurement. Each kernel is a standalone binary.

---

## How to read these results

> All DSP and NN kernel results are **normalised to CMSIS (ARM) = 1.0**.
> - Value **> 1.0** → RISC-V library outperforms CMSIS
> - Value **< 1.0** → CMSIS outperforms RISC-V
> - For stack: value > 1.0 means **less stack used** (more efficient)

Model inference results show **absolute numbers** (cycles, bytes) for direct comparison.

---

## Part 1 — DSP Kernel Results

### 1.1 Transform — Complex FFT (F32 and Q15)

#### FFT F32 — Cycle count vs FFT size (N)

| N | CMSIS | ANDES | NMSIS | ANDES speedup | NMSIS speedup |
|---|---|---|---|---|---|
| 32 | 2832 | 2764 | 2029 | 1.02× | 1.40× |
| 64 | 4852 | 3862 | 3446 | 1.26× | 1.41× |
| 128 | 14205 | 9330 | 13257 | 1.52× | 1.07× |
| 256 | 29188 | 20456 | 19303 | 1.43× | 1.51× |
| 512 | 68918 | 35717 | 52520 | 1.93× | 1.31× |
| 1024 | 136254 | 92504 | 93237 | 1.47× | 1.46× |

#### FFT F32 — Stack usage vs FFT size (bytes)

| N | CMSIS | ANDES | NMSIS |
|---|---|---|---|
| 32 | 344 | 8 | 192 |
| 64 | 276 | 8 | 256 |
| 128 | 276 | 8 | 256 |
| 256 | 344 | 40 | 256 |
| 512 | 276 | 40 | 256 |
| 1024 | 344 | 40 | 276 |

**ANDES uses up to 43× less stack than CMSIS for FFT F32 (N=32: 8B vs 344B).**

---

#### Summary — all DSP Transform metrics (normalised, CMSIS=1.0)

| Function | Library | Cycle | Instruction | Exec Time | Stack | Code Size |
|---|---|---|---|---|---|---|
| FFT F32 | ANDES | 1.24× | 1.42× | 1.19× | 9.96× less | 1.05× |
| FFT F32 | NMSIS | 1.43× | 1.49× | 1.37× | 3.41× less | 1.01× |
| FFT Q15 | ANDES | ~1.0× | 0.97× | ~1.0× | 1.85× less | 1.09× |
| FFT Q15 | NMSIS | 1.15× | 1.20× | 1.10× | 1.52× less | 0.62× |

---

### 1.2 Filtering — FIR (F32 and Q15)

#### FIR Q15 — Cycle count vs input size (N)

| N | CMSIS | ANDES | NMSIS | ANDES speedup | NMSIS speedup |
|---|---|---|---|---|---|
| 32 | 1644 | 943 | 2606 | 1.74× | 0.63× |
| 64 | 5022 | 1719 | 3028 | 2.92× | 1.66× |
| 128 | 9854 | 3287 | 5796 | 2.99× | 1.70× |
| 256 | 19519 | 6419 | 11332 | 3.04× | 1.72× |
| 512 | 38846 | 12691 | 22404 | 3.06× | 1.73× |
| 1024 | 77502 | 25235 | 44548 | 3.07× | 1.74× |

#### Summary — all DSP FIR metrics (normalised, CMSIS=1.0)

| Function | Library | Cycle | Instruction | Stack | Code Size |
|---|---|---|---|---|---|
| FIR F32 | ANDES | 0.74× (CMSIS wins) | 0.64× | 2.29× less | 4.33× |
| FIR F32 | NMSIS | 2.53× | 2.37× | 1.60× less | 1.02× |
| FIR Q15 | ANDES | 1.03× | 1.41× | 15× less | 1.13× |
| FIR Q15 | NMSIS | 2.14× | 1.93× | 2.64× less | 1.24× |

> **Why ANDES loses on FIR F32:** Andes uses scalar `fmul.s + fadd.s` with a small 2-tap unroll,
> while CMSIS uses an 8-tap unrolled loop — more pointer arithmetic amortised over more work.
> Despite heavier load-store traffic, CMSIS finishes FIR F32 in fewer cycles.

---

### 1.3 Magnitude — Complex Magnitude (F32 and Q15)

#### Magnitude Q15 — Cycle count vs input size (N)

| N | CMSIS | ANDES | NMSIS |
|---|---|---|---|
| 32 | 2631 | 1726 | 1278 |
| 64 | 5235 | 2674 | 2030 |
| 128 | 10409 | 1650 | 1582 |
| 256 | 13794 | 3131 | 3054 |
| 512 | 29362 | 6067 | 5998 |
| 1024 | 40077 | 11963 | 13886 |

#### Summary — all Magnitude metrics (normalised, CMSIS=1.0)

| Function | Library | Cycle | Instruction | Stack | Code Size |
|---|---|---|---|---|---|
| Magnitude F32 | ANDES | 1.04× | 1.99× | 1.52× less | 0.89× |
| Magnitude F32 | NMSIS | 1.15× | 1.41× | 0.33× (CMSIS wins) | 4.33× larger |
| Magnitude Q15 | ANDES | 3.32× | 3.57× | 14.8× less | 0.92× |
| Magnitude Q15 | NMSIS | 1.76× | 1.93× | 1.17× less | 1.20× |

> **ANDES achieves 3.32× fewer cycles on Magnitude Q15** by processing two samples
> per iteration with `pkbb16` packed store and a Q15-specific sqrt — eliminating the
> general-purpose Q31 path used by CMSIS.

---

### 1.4 Fast Math (sqrt, sin, cos, atan2 — F32 and Q15)

| Function | Library | Cycle | Instruction | Stack | Code Size |
|---|---|---|---|---|---|
| Fast Math F32 | ANDES | 0.91× (CMSIS wins) | 1.43× | 0.94× | 1.50× |
| Fast Math F32 | NMSIS | 0.84× (CMSIS wins) | 1.58× | 0.93× | 1.39× |
| Fast Math Q15 | ANDES | 1.28× | 1.48× | 15× less | 1.25× |
| Fast Math Q15 | NMSIS | 0.85× (CMSIS wins) | 0.92× | 1.56× less | 0.97× |

---

## Part 2 — Neural Network Kernel Results

> Comparison: NMSIS-NN (RISC-V) vs CMSIS-NN (ARM). S8 = INT8. S16 = INT16.

### 2.1 Activation (ReLU6 S8, Activation S16)

| Metric | NMSIS S8 | NMSIS S16 |
|---|---|---|
| Cycle count | 0.78× (CMSIS wins) | 0.92× (CMSIS wins) |
| Instruction count | 1.50× faster | 0.97× (CMSIS wins) |
| Stack usage | 1.00× (same) | 3.00× less |
| Execution time | 0.75× (CMSIS wins) | 0.88× (CMSIS wins) |
| Code size | 0.82× smaller | 0.71× smaller |

> **Why CMSIS wins on S8:** ARM uses conditional execution (`it ge / movge`) to skip
> the high clamp entirely for in-range values. NMSIS always evaluates both `maxw` and `minw`.

---

### 2.2 Convolution (Conv wrapper, Depthwise Conv — S8 and S16)

| Metric | NMSIS S8 | NMSIS S16 |
|---|---|---|
| Cycle count | **1.31× faster** | **1.25× faster** |
| Instruction count | **1.52× faster** | **1.35× faster** |
| Stack usage | 1.06× (comparable) | 1.00× (same) |
| Execution time | **1.26× faster** | **1.20× faster** |
| Code size | 0.72× smaller | 0.91× smaller |

> **Why NMSIS wins:** P-extension operations (`kmada`, `sunpkd820`, `pktt16/pkbb16`)
> collapse ARM's 12–16 instruction unpack-MAC sequence into 6–8 instructions,
> nearly halving dynamic cost per output.

---

### 2.3 Fully Connected (S8 and S16)

| Metric | NMSIS S8 | NMSIS S16 |
|---|---|---|
| Cycle count | **1.13× faster** | 0.85× (CMSIS wins) |
| Instruction count | **1.14× faster** | 0.99× (CMSIS wins) |
| Stack usage | 0.84× (CMSIS wins) | 0.70× (CMSIS wins) |
| Execution time | **1.08× faster** | 0.84× (CMSIS wins) |

> **Why CMSIS wins on S16:** Back-to-back `kmada` instructions create RAW hazards
> in NMSIS, forcing stalls. CMSIS `smlad`-based dual-MAC loop has shorter dependency chains.

---

### 2.4 Pooling — Average Pooling (S8 and S16)

| Metric | NMSIS S8 | NMSIS S16 |
|---|---|---|
| Cycle count | **1.08× faster** | **1.08× faster** |
| Instruction count | **1.26× faster** | **1.74× faster** |
| Stack usage | **1.43× less** | **1.45× less** |
| Execution time | 1.03× faster | 1.03× faster |

---

### 2.5 Softmax (S8 and S16)

| Metric | NMSIS S8 | NMSIS S16 |
|---|---|---|
| Cycle count | **1.24× faster** | **1.15× faster** |
| Instruction count | **1.41× faster** | **1.23× faster** |
| Stack usage | 0.34× (CMSIS wins — 608B buffer) | **1.79× less** |
| Execution time | **1.19× faster** | **1.10× faster** |
| Code size | 1.27× larger | 0.56× smaller |

---

## Part 3 — Model Inference Results

End-to-end inference on bare-metal RISC-V (Andes D25F) and ARM (Cortex-M4).
No OS, no runtime — direct execution using NMSIS-NN and CMSIS-NN quantised kernels.
Measured layer-by-layer: cycle count and stack usage per operator.

---

### 3.1 CIFAR-10 Image Classification

**Platform:** RISC-V Andes D25F (Telink B91) — ANDES vs NMSIS library comparison  
**Model:** Quantised CNN (INT8) — 3-layer ConvNet  
**Input:** 32×32 RGB image

#### CIFAR-10 — Cycle count per layer

| Layer | ANDES | NMSIS | NMSIS speedup |
|---|---|---|---|
| Preprocess | 21,517 | 28,688 | 0.75× |
| Conv1 | 17,681,954 | 5,267,023 | **3.36×** |
| ReLU1 | 378,681 | 106,531 | **3.55×** |
| Pool1 | 1,633,962 | 183,976 | **8.88×** |
| Conv2 | 27,302,881 | 4,321,937 | **6.32×** |
| ReLU2 | 46,396 | 13,341 | **3.48×** |
| Pool2 | 201,941 | 27,419 | **7.36×** |
| Conv3 | 6,307,995 | 1,080,566 | **5.84×** |
| ReLU3 | 25,460 | 6,686 | **3.81×** |
| Pool3 | 98,399 | 11,314 | **8.70×** |
| FC | 23,659 | 13,023 | **1.82×** |
| Softmax | 438 | 442 | ~1.0× |

> **NMSIS significantly outperforms ANDES on CIFAR-10.** The largest gap is at Conv2
> where NMSIS is 6.32× faster — driven by NMSIS fused MAC operations vs ANDES scalar arithmetic.

#### CIFAR-10 — Stack usage per layer (bytes)

| Layer | ANDES | NMSIS |
|---|---|---|
| Conv1 | 116 | 204 |
| Pool1 | 52 | 92 |
| Conv2 | 104 | 216 |
| Pool2 | 52 | 92 |
| Conv3 | 104 | 216 |
| Pool3 | 52 | 92 |
| FC | 116 | 100 |

> **ANDES uses less stack per layer** — consistent with kernel-level findings
> where Andes minimises stack frames through compact prologues.

---

### 3.2 Keyword Spotting — DS-CNN Small

**Platform:** ARM (CMSIS-NN) vs RISC-V (NMSIS-NN)  
**Model:** DS-CNN Small (INT8) — Depthwise Separable CNN for keyword spotting  
**Task:** 10-class keyword recognition  
**Architecture:** 13 layers — 6× DepthwiseConv2D + 5× Conv2D + AvgPool + FC

#### KWS DS-CNN Small — Cycle count per layer

| Layer | CMSIS | NMSIS | NMSIS speedup |
|---|---|---|---|
| Layer 1: DEPTHWISE_CONV_2D | 3,101,817 | 1,523,422 | **2.04×** |
| Layer 2: DEPTHWISE_CONV_2D | 916,249 | 541,043 | **1.69×** |
| Layer 3: CONV_2D | 1,770,642 | 1,449,606 | **1.22×** |
| Layer 4: DEPTHWISE_CONV_2D | 916,249 | 541,043 | **1.69×** |
| Layer 5: CONV_2D | 1,770,642 | 1,449,606 | **1.22×** |
| Layer 6: DEPTHWISE_CONV_2D | 916,249 | 541,043 | **1.69×** |
| Layer 7: CONV_2D | 1,770,642 | 1,449,606 | **1.22×** |
| Layer 8: DEPTHWISE_CONV_2D | 916,249 | 541,043 | **1.69×** |
| Layer 9: CONV_2D | 1,770,642 | 1,449,606 | **1.22×** |
| Layer 10: DEPTHWISE_CONV_2D | 916,249 | 541,043 | **1.69×** |
| Layer 11: CONV_2D | 1,770,642 | 1,449,606 | **1.22×** |
| Layer 12: AVERAGE_POOL_2D | 64,154 | 76,300 | 0.84× |
| Layer 13: FULLY_CONNECTED | 3,207 | 3,942 | 0.81× |
| **TOTAL** | **16,603,633** | **11,556,909** | **1.44× faster** |

#### KWS DS-CNN Small — Stack usage per layer (bytes)

| Layer | CMSIS | NMSIS |
|---|---|---|
| All DEPTHWISE_CONV_2D | 248 | 468 |
| All CONV_2D | 572 | 584 |
| AVERAGE_POOL_2D | 160 | 112 |
| FULLY_CONNECTED | 216 | 508 |
| **TOTAL** | **4,724** | **6,348** |

> **NMSIS runs the full DS-CNN Small model 1.44× faster than CMSIS** — 16.6M vs 11.6M cycles.
> CMSIS uses less total stack (4724B vs 6348B), largely due to the FC layer where
> NMSIS allocates a larger frame.

---

### 3.3 Keyword Spotting — DS-CNN Medium

**Platform:** ARM (CMSIS-NN) vs RISC-V (NMSIS-NN)  
**Model:** DS-CNN Medium (INT8) — larger DS-CNN variant  
**Task:** 10-class keyword recognition  
**Architecture:** 13 layers — same topology as Small, larger feature maps

#### KWS DS-CNN Medium — Cycle count per layer

| Layer | CMSIS | NMSIS | NMSIS speedup |
|---|---|---|---|
| Layer 1: DEPTHWISE_CONV_2D | 14,101,480 | 6,970,463 | **2.02×** |
| Layer 2: DEPTHWISE_CONV_2D | 1,369,318 | 804,113 | **1.70×** |
| Layer 3: CONV_2D | 5,197,590 | 4,626,485 | **1.12×** |
| Layer 4: DEPTHWISE_CONV_2D | 1,235,176 | 736,936 | **1.68×** |
| Layer 5: CONV_2D | 5,197,590 | 4,626,485 | **1.12×** |
| Layer 6: DEPTHWISE_CONV_2D | 1,235,176 | 736,936 | **1.68×** |
| Layer 7: CONV_2D | 5,197,590 | 4,626,485 | **1.12×** |
| Layer 8: DEPTHWISE_CONV_2D | 1,235,176 | 736,936 | **1.68×** |
| Layer 9: CONV_2D | 5,197,590 | 4,626,485 | **1.12×** |
| Layer 10: DEPTHWISE_CONV_2D | 1,235,176 | 736,936 | **1.68×** |
| Layer 11: CONV_2D | 5,197,590 | 4,626,485 | **1.12×** |
| Layer 12: AVERAGE_POOL_2D | 86,141 | 105,758 | 0.81× |
| Layer 13: FULLY_CONNECTED | 6,464 | 9,350 | 0.69× |
| **TOTAL** | **46,492,057** | **33,969,853** | **1.37× faster** |

#### KWS DS-CNN Medium — Stack usage per layer (bytes)

| Layer | CMSIS | NMSIS |
|---|---|---|
| All DEPTHWISE_CONV_2D | 248 | 468 |
| All CONV_2D | 572 | 584 |
| AVERAGE_POOL_2D | 160 | 112 |
| FULLY_CONNECTED | 216 | 508 |
| **TOTAL** | **4,724** | **6,348** |

> **NMSIS runs DS-CNN Medium 1.37× faster** — 46.5M vs 34.0M cycles.
> Depthwise convolution layers show the largest gains (up to 2.02×),
> consistent with NMSIS P-extension SIMD efficiency on quantised operators.

---

## Part 4 — Key Takeaways

### Where RISC-V (NMSIS / Andes) wins

| Domain | Best result | Reason |
|---|---|---|
| FFT F32 | 1.47× fewer cycles (NMSIS) | Fused fmadd.s, fewer load-store pairs |
| Magnitude Q15 | **3.32× fewer cycles** (ANDES) | Q15-specific sqrt, pkbb16 packed output |
| FIR Q15 | 2.14× fewer cycles (NMSIS) | SMALDA/SMALXDA packed MAC, compressed RVC |
| Convolution S8 | 1.31× fewer cycles (NMSIS) | P-extension kmada, sunpkd820 unpack |
| Pooling S8/S16 | 1.74× fewer instructions (NMSIS) | maxw/minw fused compare-select |
| Softmax S16 | 1.23× fewer instructions (NMSIS) | smar64 saturating MAC replaces smlal+clamp |
| KWS DS-CNN Small | **1.44× faster** end-to-end | Consistent depthwise conv advantage |
| KWS DS-CNN Medium | **1.37× faster** end-to-end | Same architecture, larger maps |
| CIFAR-10 Conv | up to **6.32× fewer cycles** (NMSIS) | SIMD fused operations |
| Stack | Up to **14.8× less** (ANDES Mag Q15) | Lean prologues, register-resident loops |

### Where ARM (CMSIS) wins

| Domain | Result | Reason |
|---|---|---|
| FIR F32 | 0.74× (CMSIS faster) | 8-tap loop unrolling, fused MACs |
| Activation S8 (ReLU6) | 0.78× (CMSIS faster) | Conditional execution skips clamp on in-range values |
| Fully Connected S16 | 0.85× (CMSIS faster) | smlad dual-MAC, shorter dependency chains |
| Fast Math F32 | 0.91× (CMSIS faster) | Mature CMSIS floating-point math kernels |
| Softmax S8 stack | 0.34× (CMSIS less stack) | NMSIS allocates 608B temporary buffer |

### Overall conclusion

> RISC-V libraries (NMSIS and Andes) are **broadly competitive with CMSIS** and
> deliver measurable advantages in fixed-point DSP and quantised NN workloads —
> especially at model level where depthwise convolution efficiency compounds across layers.
> ARM retains strengths where its conditional execution model, post-increment addressing,
> or tightly-fused MAC loops align with the kernel's control flow.
> **For TinyML workloads on constrained MCUs, RISC-V with NMSIS presents a compelling
> alternative to ARM with CMSIS — with up to 1.44× better end-to-end inference speed.**

---

## Measurement methodology (summary)

| Metric | ARM (PSoC6) | RISC-V (Telink B91) |
|---|---|---|
| Cycle count | `DWT->CYCCNT` | `NDS_MCYCLE` CSR |
| Instruction count | DWT formula: `CYCCNT - CPICNT - EXCCNT - SLEEPCNT - LSUCNT + FOLDCNT` | `NDS_MINSTRET` CSR (exact) |
| Stack usage | Stack-paint: fill `0xAAAAAAAA`, scan after execution | Same technique |
| Code size | `.map` file + `objdump` call-graph (dependency-aware) | Same technique |
| Execution from | RAM (`.cy_ramfunc` attribute) | RAM (vendor linker script) |
| Build flags | `-O3 -flto -ffunction-sections -fdata-sections` | Same flags |

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
