# Building gem5 for RISC-V and ARM

Instructions for building the gem5 simulator configured for RISC-V and ARM architectures.

## Prerequisites & Dependencies

The following tools and dependencies are required to build gem5:
- C++ Compiler: `g++` (>= 10) or `clang` (>= 11)
- Python 3 and headers (`python3-dev`)
- SCons
- zlib, m4, protobuf

On Ubuntu/Debian-based distributions, install them with:
```bash
sudo apt update
sudo apt install build-essential git m4 scons zlib1g zlib1g-dev \
    libprotobuf-dev protobuf-compiler libprotoc-dev libgoogle-perftools-dev \
    python3-dev python-is-python3 libboost-all-dev
```

---

## Compilation

### Build for RISC-V

To compile the optimized gem5 binary for the RISC-V architecture:

```bash
scons build/RISCV/gem5.opt -j $(nproc)
```

The resulting executable will be located at `build/RISCV/gem5.opt`.

---

### Build for ARM

To compile the optimized gem5 binary for the ARM architecture:

```bash
scons build/ARM/gem5.opt -j $(nproc)
```

The resulting executable will be located at `build/ARM/gem5.opt`.

---

### Additional Notes
- The `-j $(nproc)` flag enables parallel compilation using all available CPU cores to speed up build times.
- If additional debug symbols are required, replace `.opt` with `.debug` or `.fast` (e.g., `build/RISCV/gem5.debug`).

---

## gem5 Extensions and Modifications (Vector Lanes Model)

Location of files modified and added for the vector lanes extension plan, indicating what was changed and its purpose:

### 1. Processor Core (O3CPU - ISA-Agnostic)
- [`src/cpu/static_inst.hh`](src/cpu/static_inst.hh): Virtual method `numChimePasses(ThreadContext *tc)` in `StaticInst` to compute the number of passes (*chimes*) based on configured vector lanes.
- [`src/cpu/o3/dyn_inst.hh`](src/cpu/o3/dyn_inst.hh): Forwarding method `numChimePasses()` in `DynInst` querying chime passes via the thread context.
- [`src/cpu/o3/inst_queue.cc`](src/cpu/o3/inst_queue.cc): Scheduling adjustments in `scheduleReadyInsts()` to scale completion latency and functional unit (FU) occupancy when `passes > 1`.
- [`src/cpu/o3/FuncUnitConfig.py`](src/cpu/o3/FuncUnitConfig.py): Realistic execution latencies for SIMD and vector functional units.

### 2. RISC-V Vector (RVV) Support
- [`src/arch/riscv/RiscvISA.py`](src/arch/riscv/RiscvISA.py): `lanes` parameter and `RiscvVectorLanes` parameter type definition.
- [`src/arch/riscv/isa.hh`](src/arch/riscv/isa.hh) and [`src/arch/riscv/isa.cc`](src/arch/riscv/isa.cc): Storage, `getNumVecLanes()` accessor, and validation for the `lanes` parameter.
- [`src/arch/riscv/insts/vector.hh`](src/arch/riscv/insts/vector.hh) and [`src/arch/riscv/insts/vector.cc`](src/arch/riscv/insts/vector.cc): Override of `numChimePasses()` in `VectorMicroInst` computing passes from `microVl` and configured `lanes`.

### 3. ARM SVE Support
- [`src/arch/arm/ArmISA.py`](src/arch/arm/ArmISA.py) and [`src/arch/arm/ArmSystem.py`](src/arch/arm/ArmSystem.py): `sve_lanes_se` (SE mode) and `sve_lanes` (FS mode) parameters, along with the `SveVectorLanes` parameter type.
- [`src/arch/arm/isa.hh`](src/arch/arm/isa.hh), [`src/arch/arm/isa.cc`](src/arch/arm/isa.cc), [`src/arch/arm/system.hh`](src/arch/arm/system.hh), and [`src/arch/arm/system.cc`](src/arch/arm/system.cc): Storage, accessors (`getNumSveLanes()`, `sveLanes()`), and system-wide propagation of `sve_lanes`.
- [`src/arch/arm/insts/static_inst.hh`](src/arch/arm/insts/static_inst.hh) and [`src/arch/arm/insts/static_inst.cc`](src/arch/arm/insts/static_inst.cc): Helper function `numSveChimePasses(ThreadContext *tc, unsigned eCount)` for SVE chime computation.
- [`src/arch/arm/isa/templates/sve.isa`](src/arch/arm/isa/templates/sve.isa) and [`src/arch/arm/isa/templates/sve_mem.isa`](src/arch/arm/isa/templates/sve_mem.isa): Overrides of `numChimePasses()` across SVE arithmetic and memory instruction templates.

### 4. Testing and Validation Environment
All experiments, testing, and validation of the vector lanes model were performed in [`util/vector_lanes_bench/`](util/vector_lanes_bench/):
- **Assembly Microbenchmarks**:
  - `bench_vector_lanes.S` and `Makefile`: Latency and throughput microbenchmarks for RISC-V RVV (`chain` and `indep` modes, varying `SEW` and `LMUL`).
  - `bench_sve_lanes.S` and `Makefile_sve`: Latency and throughput microbenchmarks for ARM SVE (`chain` and `indep` modes, 8-bit, 32-bit, and 64-bit element sizes).
- **Simulation and Automation Scripts**:
  - `run_lanes_bench.py`: gem5 configuration script for RISC-V simulation with L1/L2 caches.
  - `run_sve_lanes_bench.py`: gem5 configuration script for ARM SVE simulation with L1/L2 caches.
  - `run_all_benchmarks.sh`: Automated compilation and execution script for the entire microbenchmark suite.
- **Analytical Models and Documentation**:
  - `expected_cycles.py` and `expected_cycles_sve.py`: Analytical cycle predictors used to verify simulation results against theoretical formulas.
  - `MICROBENCH_GUIDE.md`: Guide for compiling and running the microbenchmarks.
  - `REPORT.md`: Detailed validation report and empirical results analysis.
  - `results/`: Directory containing generated simulation outputs and statistics (`stats.txt`, `config.ini`).

### 5. Configuration Scripts and Evaluation Workloads
- [`gem5_config_riscv.py`](gem5_config_riscv.py) and [`gem5_config_arm.py`](gem5_config_arm.py): Configuration scripts to run full vector workloads with customizable lane counts on RISC-V and ARM.
- [`stats_analisis/`](stats_analisis/): Collected execution statistics from algorithm runs.
