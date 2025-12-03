# Roadmap: Porting HVM2/Bend to Intel GPUs (SYCL/OneAPI)

**Objective:** Implement a SYCL backend for the HVM2 runtime to enable native execution on Intel Xe architecture (Arc, Data Center, and Battlemage/Xe2).

**Target Hardware:** Intel Arc (Alchemist), Intel Core Ultra (Meteor Lake/Lunar Lake), **Intel Battlemage (Xe2)**.

---

## 🛠 Phase 1: Environment & Discovery
*Goal: Set up the build environment and identify the CUDA logic to be replicated.*

- [ ] **Install Intel OneAPI Base Toolkit**
    - Ensure `icpx` (Intel DPC++ Compiler) is in the PATH.
    - Verify installation with `sycl-ls` (should show your Intel GPU).
- [ ] **Clone Repositories**
    - Fork `HigherOrderCO/hvm-core` (The runtime).
    - Fork `HigherOrderCO/Bend` (The frontend/CLI).
- [ ] **Codebase Audit**
    - Locate the CUDA kernel generation logic (search for `cuda_codegen.rs` or embedded strings starting with `extern "C" __global__`).
    - Identify the specific atomic operations used (likely `atomicCAS` and `atomicAdd`).

## 🏗 Phase 2: Kernel Translation (CUDA to SYCL)
*Goal: Create a `runtime.sycl.cpp` that performs the interaction combinator reductions.*

- [ ] **Initialize the SYCL Queue**
    - Create a selector that prioritizes GPU: `sycl::gpu_selector_v`.
    - Enable profiling to measure execution time.
- [ ] **Memory Management (Unified Shared Memory)**
    - *Challenge:* HVM relies on pointer dereferencing.
    - *Solution:* Use USM (Device Allocations) to mimic C-style pointers.
    - Map `cudaMalloc` → `sycl::malloc_device`.
    - Map `cudaMemcpy` → `q.memcpy`.
- [ ] **Port the Reduction Kernel**
    - Convert the Grid/Block hierarchy:
        - `threadIdx.x` → `item.get_local_id(0)`
        - `blockIdx.x` → `item.get_group(0)`
    - Convert `__syncthreads()` → `item.barrier(sycl::access::fence_space::local_space)`.
- [ ] **Implement Atomics (Crucial)**
    - HVM is lock-free and relies on comparing pointers.
    - Replace `atomicCAS` with `sycl::atomic_ref`:
      ```cpp
      // Example target
      auto atom = sycl::atomic_ref<u32, sycl::memory_order::relaxed, sycl::memory_scope::device, sycl::access::address_space::global_space>(*ptr);
      atom.compare_exchange_strong(expected, desired);
      ```

## 🔌 Phase 3: Rust Integration
*Goal: Allow the `bend` and `hvm` CLI tools to compile and run the SYCL code.*

- [ ] **Update `Cargo.toml`**
    - Add the `cc` build dependency to compile C++ files.
- [ ] **Configure `build.rs`**
    - Add logic to detect a `--sycl` feature flag.
    - Configure the builder to use `icpx` and link against `sycl`.
- [ ] **Implement `gen-sycl`**
    - Duplicate the `gen-c` or `gen-cu` logic in the Rust source.
    - Ensure it outputs the proper SYCL boilerplate instead of CUDA.

## ⚡ Phase 4: Battlemage (Xe2) Optimization
*Goal: Tune the kernel for Intel's specific architecture.*

- [ ] **Sub-Group Optimization**
    - Don't assume a Warp size of 32 (Intel uses 16 or 32 depending on hardware).
    - Use `item.get_sub_group()` for warp-level primitives.
- [ ] **Register Pressure**
    - Intel compilers are aggressive with unrolling. Monitor spill counts using `-fsycl-link-targets`.
- [ ] **XVE Utilization**
    - Ensure memory accesses are coalesced to maximize the wide vector engines on Xe2.

## ✅ Phase 5: Validation & Benchmarks
- [ ] **Hello World**
    - Run a simple recursive sum (Fibonacci or Sum) to verify correctness.
- [ ] **Bitonic Sorter (The Standard)**
    - Run the Bitonic Sorter example from the Bend README.
    - Compare timing against `run-c` (CPU) and `run-cu` (NVIDIA).

## 📚 References
* [Intel OneAPI DPC++ Reference](https://www.intel.com/content/www/us/en/developer/tools/oneapi/data-parallel-c-plus-plus.html)
* [Migrating from CUDA to SYCL](https://www.intel.com/content/www/us/en/developer/articles/technical/migrating-from-cuda-to-sycl.html)
* [ZLUDA (for reference on mapping logic)](https://github.com/vosen/ZLUDA)
