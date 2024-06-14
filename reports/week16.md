# Week 16

## On rCore-Tutorial

- 修复了 `libafl_qemu_cmd` 的一些问题

  - 对 Rust 内联汇编不熟导致的低级错误，写反了源寄存器和目的寄存器

  - 发现用于给命令传递参数的汇编指令没有被正常执行（`.4byte` 之前的 `mov`）：

    ```rust
    unsafe {
        asm!(
        	"mov rax, {1}",
            "mov rdi, {2}",
            "mov rsi, {3}",
            ".4byte {opcode}",
            "mov {0}, rax",
            lateout(reg) ret,
            in(reg) action,
            in(reg) arg1,
            in(reg) arg2,
    		opcode = const OPCODE,
        );
    }
    ```

    在 @yfblock 的帮助下，发现可能与 QEMU 的指令翻译有关，具体在 `accel/tcg/translator.c` 的 `translator_loop` 中。翻译过程是按块（translation block，TB）进行的，由于 `mov` 和 backdoor 指令码在同一个基本块中，当执行到 backdoor 指令码时，可能 `mov` 指令只是经过了翻译而没有被执行。但是不知道如何解释之前 C 为何能正常运行，还需要进一步调试。

    临时的解决方案是将 backdoor 指令码放到一个另外的函数中，再通过 `call` 进行调用：

    ```rust
    #[naked]
    extern "C" fn backdoor() {
        unsafe {
            asm!(
                ".4byte {opcode}",
                "ret",
                opcode = const OPCODE,
                options(noreturn),
            );
        }
    }
    ```

- 运行测试

  - 通过简单的 fork 测试可行性：

    ```rust
    #![no_std]
    #![no_main]
    
    use core::alloc::Vec;
    
    use libafl_qemu_cmd::{self, EndStatus};
    
    use user_lib::{exit, fork, wait};
    
    #[no_mangle]
    fn main() {
        const BUFFER_SIZE: usize = 4;
        
        let mut buffer = [0u8; BUFFER_SIZE];
        
        let _size = libafl_qemu_cmd::start_virt(buffer.as_mut_ptr(), BUFFER_SIZE);
        for i in size as usize..BUFFER_SIZE {
            buffer[i] = 0;
        }
        let child_num = u32::from_le_bytes(buffer);
    
        for _ in 0..child_num {
            let pid = fork();
            if pid == 0 {
                exit(0);
            }
            assert!(pid > 0);
        }
    
        let mut exit_code: i32 = 0;
        for _ in 0..child_num {
            if wait(&mut exit_code) <= 0 {
                panic!("wait stopped early");
            }
        }
    
        if wait(&mut exit_code) > 0 {
            panic!("wait got too many");
        }
    
        libafl_qemu_cmd::end(EndStatus::Ok);
    }
    ```