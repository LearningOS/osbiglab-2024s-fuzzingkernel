# Week 6

## LibAFL in no-std

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