# Week 8

## xv6 for x86_64

### 目标

因为 LibAFL Qemu 目前支持的指令集有限，所以先尝试在 x86_64 下进行 kernel fuzzing。

### 内核

- 使用了 [jserv/xv6-x86_64](https://github.com/jserv/xv6-x86_64)（可使用其它实现，但以下内容不可参考）。
- 在 `user/` 下添加 harness，一个用户程序。此处使用原来的 qemu_systemmmode 示例提供的 `main.c`。
- 修改 `init.c`，使 init proc 直接跳转至 harness（或可直接使 harness 作为 init proc，暂未尝试）。
- 修改 `Makefile`：
  - 给 `CFLAGS` 添加 `-D $(TARGET_DEFINE)` 及 `-I<path to libafl_qemu.h>`，从而可以使用 `sync_exit` 模式。
  - 移除 `UPROGS` 中的无关程序，避免 `mkfs` 打包镜像时因某些用户程序体积太大而失败，同时也减小生成的 `fs.img` 的体积。
- 修改 fuzzer 的 `Makefile.toml`，使其运行前编译内核，并能够用正确的参数启动 Qemu。
  - 参数设置基本按照 kernel 在 Qemu 下运行时的参数即可。
  - 目前还是需要通过额外增加 `-L` 选项，指定 bios 等二进制文件的路径。如果安装较新的 Qemu（8.0+），则应能在 `/usr/share/qemu` 或 `/usr/local/share/qemu` 下找到全部所需文件。注意通过 apt 安装的 Qemu 版本可能非常老旧。

### Fuzzer

- 如何确定 fuzzing 的起止点？可能的方案：

  - 以用户态程序作为 harness，注入数据进行随机 syscall
  - 在 trap handler 中注入数据进行随机 syscall

  为了简单，先尝试第一种方案。

- 如何控制 Qemu 的运行？

  - `classic`、`breakpoint` 都基于断点，但是在使用虚存和进程调度的情况下，暂不确定能否在 harness 内设置合适的断点。
  - `sync_exit` 依靠 harness 主动发出 start 和 end 指令来调控 host 的运行，是一种比较灵活的解决方案。

- `sync_exit` 模式下，需要使用 `libafl_qemu.h` 中定义的指令函数，但是 GNU as 无法编译 `libafl_qemu.h` 中的 `.dword`

  - 解决方案：更改为 `.4byte` 即可。
  - 解决过程：
    - 起初将 `.dword` 改为 `.word`，可编译运行，但运行时 harness 中的 `sync_exit` 相关函数产生了 illegal inst 异常，被 kernel 处理。
    - 阅读 qemu-libafl-bridge（qemu patched by LibAFL）代码，在 `accel/tcg/translator.c` 的 `translator_loop` 中找到判断 backdoor 的相关代码，发现只是逐字节判断。
    - 之前老师、学长已指出 `.word` 可能不合理，查看 kernel 反汇编结果，发现 `.word` 只有两个字节，因此出错。

- `sync_exit` 模式下，如何处理 Qemu 状态恢复？

  - 解决方案：
    - 使用 `QemuSnapshotManager`，即 Qemu 原生 snapshot
    - 使用 `qemu-img convert -f raw -O qcow2 <input_img> <output_img>`，将 `xv6.img` 和 `fs.img` 转换为 `qcow2` 格式
    - 从 Qemu 启动参数中移除 `-drive ... format=raw` 中的 `format=raw`

  - 解决过程：
    - 解决上一个问题后，发现运行若干轮测试后会不断发生 crash。具体来说，`ExitHandler` 通过 `InputCommand` 向 harness 写入数据时，通过 `libafl_qemu_current_cpu() -> CPUStatePtr` 会得到一个空指针，导致 crash。
    - 阅读 qemu-libafl-bridge 代码，在 `cpu-target.c` 中找到 `libafl_qemu_current_cpu` 的实现。当 `current_cpu` 为空时（推测是 Qemu 停止运行），它进一步调用 `libafl/exit.c` 中的 `libafl_last_exit_cpu`，该函数只有当 Qemu 发生 expected exit 时才会返回相应的 CPU state，否则返回空指针。
    - 通过观察和 print 确定了 sync backdoor 的正常处理顺序，大致为：
      - `accel/tcg/translator.c:translator_loop`
      - `accel/tcg/tcg-runtime.c:gen_helper_libafl_qemu_handle_sync_backdoor`
      - `libafl/exit.c:libafl_exit_request_sync_backdoor`，在 `last_exit_reason` 中保存类别为 sync backdoor
      - `libafl/exit.c:prepare_qemu_exit`，确定为 expected exit，在 `last_exit_reason` 中保存 CPU 状态及 next PC，Qemu 停止运行
    - 对比 fuzzer 正常/异常工作时的 print 结果，发现没有进行正常的处理流程，推测是出现了 unexpected exit，原因可能是原代码使用的 `FastSnapshotManager` 没有正确地恢复状态。
    - 替换为 `QemuSnapshotManager` 可正常工作。

- 经验：出现 corpus 加载失败等表面问题，根源一般是 fuzzer 存在 bug。

## 期中总结

- 理论学习：fuzzing 的基本知识
  - Slides
  - Papers: Tardis, LibAFL, LibAFL Qemu
- 动手实践：
  - Fuzzing 101 with LibAFL
  - LibAFL fuzzer examples
    - user mode fuzzers
    - baby_no_std
    - qemu_systemmode
  - Move to x86
    - Helloworld OS from Zhu Yi
    - xv6 for x86_64

## 计划

- First kernel fuzzing!
- 考虑 Qemu 原生 snapshot 带来的性能损失