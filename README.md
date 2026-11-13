# Custom Memory Design for Logic-in-Memory (LiM)

### EC571: Digital VLSI Circuit Design — Spring 2025 Boston University

Public mirror to the older branch of [LiM](https://github.com/RaunakSRajpal/LiM) repo: contains the work from EC571 alone

**Authors:** Raunak S. Rajpal, Chih Han "Josh" Yeh

## Overview

Modern processors operate far faster than memory can supply data, creating a performance bottleneck known as the **Memory Wall** or the **Von Neumann bottleneck**. Logic-in-Memory (LiM) addresses this by embedding logic circuits directly into memory cells, allowing computations to execute _"in-place"_ and reducing costly CPU–memory data transfers. Thus, it all boils down to a PPA trade-off between reducing memory access-times at the cost of adding area, power and complexity to individual cells or vice-versa.

This project implements a Logic-in-Memory (LiM) architecture for searching the minimum or maximum value in a memory array, following the algorithm and design proposed by **[Ottati et al. (2021)](./docs/LiM_2023_Fabrizio_Ottati.pdf)**. Two AND gate variants, a ***Dynamic-Logic AND*** and a ***Special-Purpose AND*** are embedded directly into the 10T NOR-CAM cells and evaluated within a 2×2 memory array using Cadence Virtuoso.

The goal is to demonstrate that performing computation directly inside memory can reduce data movement between processor and memory (the "Memory Wall" problem), improving performance and overall energy efficiency for certain workloads.

<!-- --- -->


## Background

This project builds on the work of:

> F. Ottati, G. Turvani, G. Masera, and M. Vacca, "Custom Memory Design for Logic-in-Memory: Drawbacks and Improvements over Conventional Memories," *Electronics*, vol. 10, no. 18, p. 2291, Sep. 2021. https://doi.org/10.3390/electronics10182291

<!-- --- -->


## Project tree

```
.
├── LICENSE			## documentation/img files
├── README.md
├── docs/
│	├──	EC571_Project_Paper-LiM_RSR_CJY.pdf
│	├──	EC571_LiM_Project_Presentation.pdf
│	└──	LiM_2023_Fabrizio_Ottati.pdf
├── schematics-img/
├── plots/
|
|
|				## cadence schematic design files
├── benchmarks/
│	├──	SRAM 6T
│	├──	CAM 10T
│	└──	LiCAM 13T
├── LiM/
│	├──	dynNOR
│	└──	spAND
├── periphLog/
│	└──	minMax
├── SenseAmp/
│	└──	latchSA
|
|				## Top memory design
└── top_cell/
	└──	...	
```

<!-- > Note: << A full Cadence directory tree can be added here?? >>. The schematic hierarchy includes: cell design, Sense Amplifiers, peripheral circuitry, the 2x2 array top-design, and their testbenches.
 `6T_SRAM`, `CAM10Tcell`, `latchSA`, `dynAND`, `spAND`, `LiCAM13T`, and the 2×2 array testbenches. -->

---


## Search Algorithm

In-Memory Min/Max Search: A **bitwise AND-based max/min value search** is implemented over the LiM array. The algorithm proceeds as follows:

1. A mask vector is applied in parallel to all stored words in the array, one bit at a time, starting from the MSB.
2. At each step, each word gets ***AND-ed*** with the current mask bit.
3. For finding the maximum, words that produce a `0` result are discarded; or `1` for minimum.
4. The process iterates toward the LSB until a single word (the max or min) remains.

> This approach parallelizes comparison across all rows simultaneously, exploiting the array's wired-OR structure.

<!-- --- -->


## Architecture

### Benchmark Cells (Reference)

| Cell | Description |
|------|-------------|
| **6T SRAM** | Standard 6-transistor static RAM cell; baseline for read/write performance |
| **10T NOR-CAM** | Content-Addressable Memory cell with XNOR-based compare logic; baseline for lookup performance |

Both use a **latch-based sense amplifier (SA)** for read operations, and the CAM uses a **current-saving matchline sense amplifier (MLSA)** for search.

### LiM Cell Variants

#### Dynamic CMOS Logic AND (AND DYN)
- Implements AND via De Morgan's Law: `~(D̄ + B̄L) = D · BL`
<!-- Uses a precharge PMOS transistor (clock-gated) and two series NMOS pull-downs -->
- **16 transistors total** (10T CAM core + 6T AND gate)
<!-- - Higher area and power cost; poor memory density
- Masking done by staggering high inputs on bitlines -->

#### Special-Purpose AND (AND SPY)
- Exploits the fact that the mask always selects exactly one column at a time (AND with `1` reduces to a selection)
- Area-optimized variant: uses only **2 additional NMOS transistors** in series
<!-- First transistor gated by inverted data bit `D̄`; second by inverted bitline `B̄L` (mask) -->
- Does not implement a general AND function but correctly realizes the required filtering behavior
- Masking done by staggering low inputs on bitlines

Both variants share the **10T NOR-CAM core** and a reconfigurable **latch-based SA** that supports both memory read and LiM AND modes.

---


## Simulation Environment

- **Tool:** Cadence Virtuoso Layout
- **Simulator:** Cadence Spectre
- **Process library:** FreePDK 45nm-CMOS ; custom transistor sizing in this implementation
- **Supply voltage:** VDD = 1.1 V
- **Clock period:** 100 ns (testbench)
- **Array size simulated:** 2×2 (full characterization)

Simulated operations per cell: **Read '0', Read '1', Write, Compare, AND**

<!-- --- -->


## Results

Energy-Delay Products (EDP) in pJ·ps, compared against Ottati et al. reference values:

**Key observations:**
- The Special-Purpose AND achieves the lowest EDP per (micro-)instruction across all the implementations, consistent with Ottati et al.
- The Dynamic CMOS AND has significantly higher AND EDP due to dynamic logic switching overhead across all row cells during every AND operation.
- Both implementations are slightly higher than reference values, attributable to differences in transistor sizing, process parameters, and the use of a latch-based SA instead of the original MLSA for the AND PHY path.
- Both designs successfully demonstrate correct LiM behavior in 2×2 array simulations.

### Simulation Results

| Memory  | Write   | Read   | Compare | AND     |
|---------|---------|--------|---------|---------|
| SRAM    | 188.7   | 163.1  | —       | —       |
| CAM     | 291.12  | 203.33 | 262.12  | —       |
| SPY-AND | 406.13  | 274.90 | 410.22  | 107.55  |
| DYN-AND | 1122.16 | 410.77 | 2055.39 | 4179.61 |

### Reference Values

| Memory  | Write | Read | Compare | AND  |
|---------|-------|------|---------|------|
| SRAM    | 134   | 118  | —       | —    |
| CAM     | 275   | 230  | 236     | —    |
| SPY-AND | 309   | 260  | 717     | 98   |
| DYN-AND | 596   | 451  | 1152    | 2008 |

<!-- --- -->


## Key Design Decisions & Deviations from Ottati et al.

- The **static CMOS AND** variant (section 3.2) from the original paper was **not implemented** in this work; only the dynamic and special-purpose variants are covered.
- A **reconfigurable latch-based sense amplifier** is used in place of the original current-saving MLSA, supporting both SRAM read and CAM/LiM modes.
- Transistor sizing was chosen to satisfy read/write margin and CAM compare operation robustness via rigorous simulations.

<!-- --- -->


## References

1. F. Ottati, G. Turvani, G. Masera, and M. Vacca, "Custom Memory Design for Logic-in-Memory: Drawbacks and Improvements over Conventional Memories," *Electronics*, vol. 10, no. 18, p. 2291, Sep. 2021. https://doi.org/10.3390/electronics10182291
2. S. Jeloka, N. B. Akesh, D. Sylvester, and D. Blaauw, "A 28 nm Configurable Memory (TCAM/BCAM/SRAM) Using Push-Rule 6T Bit Cell Enabling Logic-in-Memory," *IEEE J. Solid-State Circuits*, vol. 51, no. 4, pp. 1009–1021, Apr. 2016. https://doi.org/10.1109/jssc.2016.2515510
3. T. Kobayashi, K. Nogami, T. Shirotori, and Y. Fujimoto, "A current-controlled latch sense amplifier and a static power-saving input buffer for low-power architecture," *IEEE J. Solid-State Circuits*, vol. 28, no. 4, pp. 523–527, Apr. 1993. https://doi.org/10.1109/4.210039
4. M. Vacca, Y. Tavva, A. Chattopadhyay, and A. Calimera, "Logic-In-Memory Architecture For Min/Max Search," *2018 25th IEEE Int. Conf. on Electronics, Circuits and Systems (ICECS)*, Bordeaux, France, 2018, pp. 853–856. https://doi.org/10.3390/electronics10020155 

---


## OG Contributors

Raunak S. Rajpal	|	rsrajpal@bu.edu		[^Corresponding-Author]  
Chih "Josh" Yeh		|	jy0825@bu.edu


[^Corresponding-Author]: Lead and Corresponding Author [LinkedIn](https://www.linkedin.com/in/raunak-rajpal-666b5820b/)
