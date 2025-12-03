# Contributing to Bendsycle

Thank you for your interest in porting Bend to Intel GPUs! Bendsycle is a community effort to bring the HVM2 runtime to the SYCL/OneAPI ecosystem.

## 🚧 Current Status
We are currently in **Phase 2** of the Roadmap: *Kernel Translation*.
- **Goal:** Get a basic interaction combinator kernel compiling with `icpx`.
- **Target Hardware:** Intel Arc Battlemage (B580) and Alchemist.

## 🛠 Development Environment
To contribute, you will need a Linux environment (Ubuntu 24.04 recommended) with:
1.  **Intel OneAPI Base Toolkit** (specifically `icpx` and `dpct`).
2.  **Rust** (latest stable).
3.  **Intel Compute Runtime** (Level Zero).

### Setting up the Dev Container (Optional)
If you use VS Code, we recommend using the Dev Container definition included in `.devcontainer` to ensure you have the correct OneAPI versions.

## 🧠 How to Help
We need help in three specific areas:

### 1. The "Atomic" Problem
HVM relies on `atomicCAS` (Compare and Swap) on global memory. SYCL handles atomics differently than CUDA.
* **Task:** Refactor `src/cuda/runtime.cu` logic into `src/sycl/runtime.cpp` using `sycl::atomic_ref`.
* **Constraint:** Must match the memory ordering guarantees of the CUDA implementation to prevent race conditions in graph rewriting.

### 2. Memory Management (USM)
We are using **Unified Shared Memory (USM)** to simplify the port.
* Look for `cudaMalloc` calls in the Rust FFI bindings.
* Replace them with `sycl::malloc_device` ensuring the pointer is accessible by the device selector.

### 3. Build System
We need to wire up `build.rs` to detect Intel GPUs.
* If you know `cc-rs` well, help us configure the build script to link `libsycl` dynamically only when the `--sycl` feature is enabled.

## 📜 Submission Guidelines
1.  **Fork** the repository.
2.  **Create a branch** for your feature (`git checkout -b feature/atomic-fix`).
3.  **Sign off** your commits (DCO).
4.  Open a **Pull Request** and tag the maintainers.

## 💬 Community
Join the discussion on the [HigherOrderCO Discord](https://discord.gg/higherorderco) in the `#development` channel (mention you are working on the SYCL port).
