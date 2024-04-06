# Week 6

## LibAFL Paper

### LibAFL

[LibAFL: A Framework to Build Modular and Reusable Fuzzers](https://dl.acm.org/doi/abs/10.1145/3548606.3560602)

- Motivation

- Detailed introduction to entities

### LibAFL QEMU

[LibAFL QEMU: A Library for Fuzzing-oriented Emulation](https://hal.science/hal-04500872/)

- Library:
  - `libafl_qemu_build`: offers an API to build QEMU and generate automatically bindings
  - `libafl_qemu_sys`: exposes C API, invokes the `libafl_qemu_build` API during the build process
  - `libafl_qemu`: depends on `libafl_qemu_sys`, exposes safe Rust API
- `libafl_qemu`:
  - low-level API: in `Emulator`, with safe Rust API (mentioned before)
  - high-level API: abstraction over low-level API, offering hooking capabilities
  - fuzzing-oriented capabilities: built on top of the low and high-level APIs, implement a set of pre-made instrumentations (*instrumentation helpers*)
    - only support some architectures
  - select architecture with feature flags
  - support both user mode and system mode
- Hooks: plugins for specific instrumentation
  - many kinds...
  - correspond to `QemuHelper`, living in `QemuHooks` (compile-time tuple list)
  - communicate with host:
    - backdoor: use a hypercall that triggers the backdoor hook
    - sync backdoor: stop the VM and return to the host-side harness
- Control: how to drive the fuzzers target
  - breakpoints: you know where to start and stop the fuzzing
  - sync backdoor: you don't know where to start and stop the fuzzing (e.g., kernel fuzzing)
  - custom monitor: debugger-like

## LibAFL Examples

### baby_no_std

由 LibAFL 提供的在 no-std 环境下的 fuzzer 示例。

#### 环境

- UNIX-like
- Cargo-make，使用 `cargo install cargo-make` 安装

#### 过程

1. 从 [LibAFL](https://github.com/AFLplusplus/LibAFL) 仓库 clone 源码

2. 在 `fuzzers/baby_no_std ` 目录下，修改 `Cargo.toml`，将 `[dependencies]` 中 `libafl` 一项修改为：

   ```
   libafl = { default-features = false, path = "../../libafl/", features = ["serdeany_autoreg"] }
   ```

3. 运行 `cargo make test`，观察到类似输出即可：

   ```
   [UserStats #0] run time: 0h-0m-0s, clients: 1, corpus: 0, objectives: 0, executions: 0, exec/sec: 0.000
   [Testcase #0] run time: 0h-0m-0s, clients: 1, corpus: 1, objectives: 0, executions: 1, exec/sec: 0.000
   [UserStats #0] run time: 0h-0m-0s, clients: 1, corpus: 1, objectives: 0, executions: 1, exec/sec: 0.000
   [Testcase #0] run time: 0h-0m-0s, clients: 1, corpus: 2, objectives: 0, executions: 2, exec/sec: 0.000
   [UserStats #0] run time: 0h-0m-0s, clients: 1, corpus: 2, objectives: 0, executions: 2, exec/sec: 0.000
   [Testcase #0] run time: 0h-0m-0s, clients: 1, corpus: 3, objectives: 0, executions: 4425, exec/sec: 0.000
   Aborted
   ```

#### 注意

1. 目前只能使用 LibAFL 仓库源码进行实验，如使用 crates.io 的版本，在 dev 模式下会产生链接错误，原因未知。

2. 可以将 `Makefile.toml` 中 `[task.test_unix]` 修改为：

   ```
   [tasks.test_unix]
   script='''
   cargo run --profile ${PROFILE} -Zbuild-std=core,alloc --target x86_64-unknown-linux-gnu || true
   '''
   dependencies = ["build"]
   ```

   添加了 `--profile ${PROFILE}`，让 `cargo run` 使用 `cargo build` 的结果，否则默认是 dev 模式。

3. 可以将 `src/main.rs` 中 `panic` 函数修改为：

   ```rust
   #[panic_handler]
   fn panic(info: &PanicInfo) -> ! {
       #[cfg(unix)]
       unsafe {
           printf(b"panic: %s\n\0".as_ptr().cast(), info.to_string().as_ptr());
           abort();
       }
       #[cfg(not(unix))]
       loop {
           // On embedded, there's not much left to do.
       }
   }
   ```

   输出 `info`，可以得到更多有效信息，如 fuzzer 异常崩溃、`harness` panic 时输出的 `=)`。

### qemu_systemmode

#### 环境

- UNIX-like
- Cargo-make，使用 `cargo install cargo-make` 安装

#### 过程

1. 从 [LibAFL](https://github.com/AFLplusplus/LibAFL) 仓库 clone 源码
2. 安装编译器 `gcc-arm-none-eabi`
3. 设置 `export LLVM_CONFIG_PATH=/usr/bin/llvm-config-{version}`，运行 `cargo make run`

#### 注意

1. 最近 `qemu_systemmode` 有更新，新增两种不同的控制方式，还未尝试：

   - `classic`: The low-level way to interact with QEMU.

   - `breakpoint`: Interaction with QEMU using the command system, leveraging breakpoints.
   - `sync_exit`: Interaction with QEMU using the command system, leveraging sync exits.

## Tardis

与软院课题组进行交流：

- 了解了更多 Tardis 架构方面的细节（QEMU + AFL-style + Rust）
- 咨询了他们对 Gustave、LibAFL 等其它 fuzzing 工具的看法
  - Gustave 也基于 QEMU 实现，但他们认为适配难度较高
  - 他们认为 LibAFL 更适合软件，但就本周实验情况来说应该也能尝试

## 计划

- 进一步理解 LibAFL QEMU 的工作过程和使用方法
- 尝试用 LibAFL QEMU 运行 rcore-tutorial