# Chapter 3

## 功能

- `sys_task_info`
  - 在 `TaskControlBlock` 中维护任务起始时间、系统调用计数等信息，提供修改和访问的接口
  - 将 `TaskControlBlock` 修改任务状态的功能封装成接口，方便维护信息和外部调用

## 简答

1. 运行输出：

   ```
   [rustsbi] RustSBI version 0.3.0-alpha.4, adapting to RISC-V SBI v1.0.0
   ...
   [kernel] Loading app_0
   [kernel] PageFault in application, kernel killed it.
   [kernel] Loading app_1
   [kernel] IllegalInstruction in application, kernel killed it.
   [kernel] Loading app_2
   [kernel] IllegalInstruction in application, kernel killed it.
   ```

2. `__alltraps` 与 `__restore`

   1. 刚进入 `__restore` 时，`a0` 的值是内核栈上保存的 Trap 上下文的指针。调用 `__restore` 有两种情形：
      
      - 开始运行一个应用程序时，`run_next_app` 调用 `__restore` 切换到用户态，以内核栈顶的初始 Trap 上下文的指针作为参数。
      
      - 处理 Trap 时，`__alltraps` 调用 `trap_handler`，返回后直接顺序执行 `__restore`，由于 `trap_handler` 返回的是 Trap 上下文的引用，所以 `a0` 的值还是原来 Trap 上下文的指针。

   2. 特殊的寄存器分别是：

         - `sstatus`：`SPP` 字段记录 Trap 发生之前 CPU 所处的特权级，`SPIE` 记录 Trap 发生之前的中断使能，这些信息可以用来恢复之前 CPU 的执行状态。
         

         - `sepc`：记录 Trap 发生之前执行的最后一条指令的地址，用来确定返回用户态后从何处开始执行。
         

         - `sscratch`：记录一个机器字长的数据，供操作系统使用。在内核处理 Trap 时，`sscratch` 一般用于存储用户栈的指针，返回用户态时需要恢复。

   3. `x2` 是 `sp`，此时 `sp` 指向内核栈顶，但我们希望保存的是用户栈指针，而用户栈指针被交换到 `sscratch` 中暂存，所以需要先保存通用寄存器，而后从 `sscratch` 中取出到通用寄存器中，再保存到 Trap 上下文中。`x4` 是 `tp`，由多线程应用程序使用，指向线程特定变量，所以暂时无需保存。

   4. 执行后，`sp` 指向用户栈顶，`sscratch` 指向内核栈顶保存的 Trap 上下文。

   5. `sret` 发生状态转换，恢复 `sstatus` 中的 `SPP` 和 `SPIE`，即发生 Trap 之前的特权级和中断使能状态。因为之前 CPU 处在 U 特权级，所以 `sret` 将返回用户态。

   6. 执行后，`sp` 指向内核栈顶保存的 Trap 上下文，`sscratch` 指向用户栈顶。

   7. 用户程序通过调用 `ecall` 产生系统调用，由运行在更高特权级的程序进行处理。用户库已经将其封装为 `syscall` 函数。

## Honor Code

1. 在完成本次实验的过程（含此前学习的过程）中，我曾分别与 **以下各位** 就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：
   
   > 无

2. 此外，我也参考了 **以下资料** ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：
   
   > 去年个人独立完成的实验：
   > 
   > [李羽飞 / rCore-Tutorial-2023S · GitLab (tsinghua.edu.cn)](https://git.tsinghua.edu.cn/liyufei21/rcore-tutorial-2023s)
   > 
   > [LearningOS/lab3-os5-nine-point-eight-p: lab3-os5-nine-point-eight-p created by GitHub Classroom](https://github.com/LearningOS/lab3-os5-nine-point-eight-p)

3. 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

4. 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。
