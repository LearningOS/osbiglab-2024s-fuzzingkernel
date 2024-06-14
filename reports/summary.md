# Summary

## 学习

了解 fuzzing 的基础知识与相关工具的原理。

- fuzzing 基础：
  - 张超老师的 PPT，了解 fuzzing 的基本知识
  - 结合 LibAFL 跟随 [Fuzzing101 with LibAFL](https://epi052.gitlab.io/notes-to-self/blog/2021-11-01-fuzzing-101-with-libafl/) 了解 coverage-guided fuzzing 基本流程的具体实现
- 已有成果（测试工具）：
  - 阅读 [Tardis](https://ieeexplore.ieee.org/document/9921188)、[LibAFL](https://dl.acm.org/doi/10.1145/3548606.3560602)、[LibAFL QEMU](https://hal.science/hal-04500872) 等论文，了解了这些工作的目标、方法及局限性
  - 在 debug 过程中分析了 [LibAFL](https://github.com/AFLplusplus/LibAFL)、[LibAFL QEMU](https://github.com/AFLplusplus/LibAFL/blob/main/libafl_qemu) 的部分代码，深入了解 LibAFL 的具体实现
- 虚拟机：
  - 在 debug 过程中分析了部分 [LibAFL 修改过的 QEMU 源码](https://github.com/AFLplusplus/qemu-libafl-bridge)，学习 LibAFL 如何通过控制虚拟机的运行进行测试

## 实践

使用一些工具在简单的情景下测试内核。

- TSFFS 与 Simics：运行示例
- LibAFL
  - 跟随 Fuzzing101 with LibAFL 运行一般用户态软件测试
  - 使用 LibAFL 运行[示例]([https://github.com/AFLplusplus/LibAFL/tree/main/fuzzers)：
    - 运行 baby_fuzzer、baby_no_std 等，熟悉方法
    - 运行 qemu_systemmode，学习 3 种控制方式的原理和实现
  - 在 x86_64 下使用 LibAFL QEMU 的 sync backdoor 模式测试内核：
    - helloworld_os：@zhuyi 编写的 OS。测试了内存相关模块。
    - [xv6-x86_64](https://github.com/jserv/xv6-x86_64)：从 x86_32 迁移到 x86_64 的 xv6。测试了部分系统调用。
    - [rcore-tutorial-v3-with-hal-component](https://github.com/Byte-OS/rcore-tutorial-v3-with-hal-component)：通过 polyhal 移植到 x86_64 的 rcore-tutorial。测试了部分系统调用。