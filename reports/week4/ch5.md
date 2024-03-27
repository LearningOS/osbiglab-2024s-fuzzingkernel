# Chapter 5

## 功能

- `sys_spawn`
  - 通过 `TaskControlBlock::spawn` 实现。具体方法是通过 `TaskControlBlock::new` 和应用数据创建一个新的 `TaskControlBlock`，然后将其与父进程关联起来。
- Stride 算法
  - 在 `TaskControlBlock` 中维护当前任务的 pass 值和优先级信息。
  - 创建 `TaskManager` trait 表示调度器，要求提供 `new`、`add` 和 `fetch` 接口。将原来的 `TaskManager` 改为 `SimpleManager`，并实现 `TaskManager` trait。
  - 创建 `StrideManager`，实现了 `TaskManager` trait。内部维护一个优先队列，`fetch` 总是取出 pass 值最小的任务，并更新其 pass 值。根据简答作业进行了简单的溢出处理，但不确定是否正确。

## 简答

### Stride 算法

1. p2 的 stride 应为 250 + 10 = 260，因溢出变为 260 - 256 = 4，所以 p2 的 stride 较小，下一次会调度 p2。

2. 不考虑溢出，使用数学符号计算。记 BigStride 为 $S$，stride 为 $s$，priority 为 $p$，用下标区分进程。设 stride 最小值为 $s_i$，最大值为 $s_j$，且 $s_j - s_i \leq S / 2$。下一轮调度将执行 $i$ 对应的进程，执行后 $s_i' = s_i + S / p_i \leq s_i + S / 2 \leq s_j$，即 $s_i \leq s_i' \leq s_j$。此时存在最小值 $s_k' = s_k \in [s_i, s_i']$，最大值 $s_j' = s_j$，显然 $s_j' - s_k' \leq s_j - s_i \leq S/2$，满足题设条件。

3. 实现如下：

   ```rust
   impl PartialOrd for Stride {
       fn partial_cmp(&self, other: &Self) -> Option<Ordering> {
           let diff = self.0.abs_diff(other.0);
           if diff <= BIG_STRIDE / 2 {
               // No overflow, simply compare
               Some(self.0.cmp(&other.0))
           } else {
               // There's one and only one that has overflowed
               Some(self.0.cmp(&other.0).reverse())
           }
       }
   }
   ```

## Honor Code

- 在完成本次实验的过程（含此前学习的过程）中，我曾分别与 **以下各位** 就（与本次实验相关的）以下方面做过交流，还在代码中对应的位置以注释形式记录了具体的交流对象及内容：

  > 无

- 此外，我也参考了 **以下资料** ，还在代码中对应的位置以注释形式记录了具体的参考来源及内容：

  > 去年个人独立完成的实验：
  >
  > [李羽飞 / rCore-Tutorial-2023S · GitLab (tsinghua.edu.cn)](https://git.tsinghua.edu.cn/liyufei21/rcore-tutorial-2023s)
  >
  > [LearningOS/lab3-os5-nine-point-eight-p: lab3-os5-nine-point-eight-p created by GitHub Classroom](https://github.com/LearningOS/lab3-os5-nine-point-eight-p)

- 我独立完成了本次实验除以上方面之外的所有工作，包括代码与文档。 我清楚地知道，从以上方面获得的信息在一定程度上降低了实验难度，可能会影响起评分。

- 我从未使用过他人的代码，不管是原封不动地复制，还是经过了某些等价转换。 我未曾也不会向他人（含此后各届同学）复制或公开我的实验代码，我有义务妥善保管好它们。 我提交至本实验的评测系统的代码，均无意于破坏或妨碍任何计算机系统的正常运转。 我清楚地知道，以上情况均为本课程纪律所禁止，若违反，对应的实验成绩将按“-100”分计。
