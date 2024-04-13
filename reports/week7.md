# Week 7

## LibAFL Examples

### qemu_systemmode

阅读代码，包括 classic、breakpoint、sync_exit 三种模式。要点如下：

共同点：

- 核心在于 `run_client` 和其中的 `harness`，前者负责每一个 fuzzer 的构造和运行，后者负责 fuzzer 的一次 fuzzing 过程。
- 由 `QemuExecutor`、`QemuHooks`、`QemuEdgeCoverageHelper` 共同完成动态插桩。
- observer、feedback、objective、state、scheduler、mutator 等组件与一般用户程序 fuzzing 基本相同。
- 三种在 fuzzing 过程中保存/恢复状态的方式：
  - 只保存/恢复 CPU 寄存器
  - qemu 原生 snapshot，据注释说较慢，此方法必须在启动 qemu 时提供写入 snapshot 的磁盘镜像
  - fast snapshot，包括 devices+mem（具体含义？）

不同点：控制 fuzzing 运行的方式，可参考 LibAFL Qemu 论文 III.B. 中 Execution Control 一节

- classic：基于断点，使用 `libafl_qemu::emu::Qemu`，直接设置或移除断点，控制 qemu 本身的运行，结束后处理结果并恢复状态等。
- breakpoint：基于断点，使用 `libafl_qemu::emu::Emulator`，对断点及相关操作做了封装，`ExitHandler` 和 `SnapshotManager` 配合处理结果和恢复状态。
- sync_exit：基于设计的 `sync_backdoor` 指令，qemu 将暂停执行并将控制权移交给 host-side harness。实现上，在 target 方面，提供了 `libafl_qemu.h`，包含一系列基于 `sync_backdoor` 的封装指令，可在目标程序中调用以通知 fuzzer；在 harness 方面，实现了对封装指令的相应处理，所以 harness 无需显式处理断点相关操作。

### Move to x86

试图将 qemu_systemmode 的例子从 arm 迁移到 x86。尝试了：

- xv6（i386）：试图自行裁剪成简单的 OS，但是失败了（跑飞了）。
- 朱懿学长提供的 OS（x86_64）：在大家的帮助下成功了。

#### 流程

- 复制一份 qemu_systemmode 的代码。

- 准备内核源代码。

- 修改内核的 `main.c`，将原来测试的 `main.c` 的内容复制过来，并使 kernel 启动后跳到测试主函数 `main_test`。

- 准备启动 qemu 所需的二进制文件，包括：

  ```
  bios-256k.bin
  efi-e1000e.rom
  kvmvapic.bin
  multiboot_dma.bin
  vgabios-stdvga.bin
  ```

  可使用本机安装的 qemu 的相关文件（`/usr/share/qemu`），或从 qemu 仓库下载。

- 修改 `Makefile.toml`：

  - `[tasks.target]` 中，修改编译内核的指令为 `make -C <kernel_dir>`。

  - `[tasks.run_fuzzer]` 中，参照内核的 Makefile 的写法修改启动 qemu 的指令，例如：

    ```
    args = [
        "-kernel", "${KERNEL_DIR}/kernel.bin",
        "-drive", "if=none,format=qcow2,file=${TARGET_DIR}/dummy.qcow2",
        "-L", "${QEMU_BINARY_DIR}",
        "-machine", "q35",
        "-nographic",
    ]
    ```

    `-drive` 挂载一个空的磁盘镜像，用于保存 qemu 原生 snapshot。`-L` 指定启动所需的二进制文件的路径。

- 修改 `src/fuzzer_*.rs` 中保存/恢复 qemu 状态的代码。目前无法使用 fast snapshot，否则会导致 qemu 的 memory region 部分崩溃，只能使用其他方案（只保存寄存器或使用 qemu 原生 snapshot）。原因暂时未知。

- 运行 `cargo make <feature>`。

#### 注意

- 可能需要设置环境变量 `LLVM_CONFIG=<major_version>`。程序会搜索可用的 `llvm-config`，但不一定成功，此时需要手动指定。

- 可能需要从 github clone 代码，注意网络设置。

## 计划

- 是否可以开始做 kernel fuzzing？
  - 基于 riscv 的 kernel 如何处理？
  - 感觉还没完全明白 kernel fuzzing 的模式……
- 有时间时确定 fast snapshot 崩溃的原因
