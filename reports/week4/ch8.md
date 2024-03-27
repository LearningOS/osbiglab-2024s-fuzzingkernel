# Chapter 8

## 功能

- 利用银行家算法进行死锁检测
  - 创建 `DeadlockChecker`，功能是对一类资源和一组任务进行死锁检测。内部维护了当前可用资源、已分配资源及各个任务对资源的需求量。任务可通过 `request` 接口申请资源，此时进行死锁检测并返回结果。
  - 为实现简单，对 `Mutex` 和 `Semaphore` 分开检测。在 `ProcessControlBlock` 中为两种资源添加相应的 `DeadlockChecker`，并添加该进程是否进行死锁检测的开关标记。
  - 系统调用实现的改动：
    - `sys_thread_create`：将新的任务添加到 `DeadlockChecker` 中。
    - `sys_mutex_create`、`sys_semaphore_create`：将新的资源添加到 `DeadlockChecker` 中。
    - `sys_mutex_lock`、`sys_semaphore_down`：根据设置进行死锁检测，若失败则返回 `-0xDEAD`；否则申请资源，同时维护 `DeadlockChecker` 中的资源情况。
    - `sys_mutex_unlock`、`sys_semaphore_up`：释放资源，同时维护 `DeadlockChecker` 中的资源情况。

## 简答

### 线程资源回收

1. 主线程退出时，需要回收的资源包括：
   - 所有线程资源：线程号、用户栈、内核栈、Trap 上下文的存储空间等
   - 进程资源：进程号、进程的地址空间中用于存放数据的物理页
2. 主线程退出时，所有线程资源都应回收，但是具体实现不同。其它线程的 `TaskControlBlock` 可能出现的位置：
   - 处于  `Ready` 状态的线程可能位于 `TASK_MANAGER` 的队列中。这些线程同时也存在于 `ProcessControlBlock` 中。因此，需要手动将其从 `TASK_MANAGER` 的队列中移除，之后清空 `ProcessControlBlock` 的线程列表时可使引用计数归零，从而释放相应资源。
   - 处于 `Blocked` 状态的线程可能位于 `MutexBlock` 或 `Semahpore` 的阻塞队列中。这些线程同时也存在于 `ProcessControlBlock` 中，但是 `Mutex` 和 `Semaphore` 也属于 `ProcessControlBlock`。所以在 `exit_current_and_run_next` 的结尾处，当 `ProcessControlBlock` 的引用计数归零而被回收时，阻塞队列中的 `TaskControlBlock` 的引用计数也会清零，从而被回收。

### Mutex 实现

1. 实现 1 不论当前是否有阻塞线程都会释放锁，然后唤醒一个线程；而实现 2 只有在当前没有阻塞线程时才会释放锁，否则直接唤醒一个线程，相当于把资源移交给该线程。

   如果实现 1 的 `lock` 的实现与实验框架相同，那么实现 1 是错误的，因为资源状态的表示与实际情况不符。当被唤醒线程开始执行时，它会认为自己已经拥有了锁，进入临界区访问共享资源。然而因为锁处于释放状态，如果一个新的线程请求锁，它也会获得该资源。这就产生了 race condition。

## Honor Code

- 在完成本次实验的过程（含此前学习的过程）中，我曾分别与 **以下各位** 就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：
  
  > 无

- 此外，我也参考了 **以下资料** ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：
  
  > 去年个人独立完成的实验：
  > 
  > [李羽飞 / rCore-Tutorial-2023S · GitLab (tsinghua.edu.cn)](https://git.tsinghua.edu.cn/liyufei21/rcore-tutorial-2023s)

- 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

- 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。
