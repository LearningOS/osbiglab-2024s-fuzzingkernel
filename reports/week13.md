# Week 13

## On rCore-Tutorial

- 整理了之前的工作：[nine-point-eight-p/x86-qemu-fuzzer](https://github.com/nine-point-eight-p/x86-qemu-fuzzer)，其中有简单的文档，还需进一步完善，并补充 kernel

- 总结了一个简单的文档（还需完善），基于 polyhal 的 rCore-Tutorial：

  - 目前至少有 3 份代码：
    - [xushanpu123/rCore-Tutorial-v3](https://github.com/xushanpu123/rCore-Tutorial-v3/tree/main)：看到了 ch1-3、ch8-9 的相关代码，其中 ch3 修改不完全，用户程序似乎只适配 riscv64。
    - [werkeytom/rCore-Tutorial-v3](https://github.com/werkeytom/rCore-Tutorial-v3/tree/ch3-feature)：ch3-5 的相关代码，但是在 `ARCH=x86_64` 下运行各有问题。ch3 可以运行，但是立即崩溃。ch4、ch5 在 `link_app.S` 中的 align 报错，疑似应同 ch3 一样改为 `.align 8`；修改后可以运行，但是报错 page fault。
    - [Huzhiwen1208/rCore-Tutorial-v3](https://github.com/Huzhiwen1208/rCore-Tutorial-v3/tree/main)
  - [这个文档](https://github.com/orgs/rcore-os/discussions/45)中说除了 ch4、ch5、ch8、ch9 都已经完成了各个架构的适配，可能还需整合一下。
  - 协助 @xushanpu123 使用 libafl qemu 编写 fuzzer，运行他修改的 rCore-Tutorial。当时的初步结果是可以运行，但是立即崩溃。

- 将 `libafl_qemu.h` 实现为一个 Rust 库 `libafl_qemu_cmd`，方便 Rust 编写的 harness 直接调用。基于 ch3 的 `03sleep.rs` 修改的示例：

  ```rust
  #![no_std]
  #![no_main]
  
  #[macro_use]
  extern crate user_lib;
  
  use user_lib::{get_time, yield_};
  
  use libafl_qemu_cmd::*;
  
  const BUFFER_SIZE: usize = 64;
  const MAX_TIME: u32 = 3000;
  
  #[no_mangle]
  fn main() -> i32 {
      let buffer = [0u8; BUFFER_SIZE];
      let size = start_virt(buffer.as_ptr(), BUFFER_SIZE); // Fuzzing begins
      if size < 4 {
          return 0;
      }
      let current_timer = get_time();
      let mut wait_for = [0u8; 4];
      wait_for.copy_from_slice(&buffer[..4]);
      let end_timer = current_timer + (u32::from_le_bytes(wait_for) % MAX_TIME) as isize;
      while get_time() < end_timer {
          yield_();
      }
      println!("Test sleep OK!");
      end(EndStatus::Ok); // Fuzzing ends
      0
  }
  ```
