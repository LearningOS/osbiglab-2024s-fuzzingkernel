# Week 12

## Fuzzing xv6 modules

- 整理了现有代码内容，可以方便地应用于 kernel。

- 使用简单的 deserialization 方式进行 fuzzing：

  - fork-wait：

    ```c
    void fuzz_fork_wait(unsigned char *data, unsigned int size) {
      unsigned i = 0;
      int pids[32] = {};
      while (i < size) {
        unsigned op = data[i++] / 2;
        if (op == 0) {
          if (i == size) break; // reach end
          
          int idx = data[i++] % 32;
          if (pids[idx] != 0)
            continue; // occupied, invalid case
          
          int pid = fork();
          if (pid < 0) continue; // fork failed
          if (pid == 0) exit();
          pids[idx] = pid;
        } else {
          int pid = wait();
          if (pid < 0)
            continue; // no child process now, invalid case
          
          int found = 0;
          for (int j = 0; j < 32; ++j) {
            if (pids[j] == pid) {
              pids[j] = 0;
              found = 1;
              break;
            }
          }
          if (found == 0) {
            printf(1, "unknown child pid!\n");
            while (1) {}
          }
        }
      }
    }
    ```

  - mkdir-chdir：

    ```c
    void fuzz_dir(unsigned char *data, unsigned int size) {
      for (unsigned i = 0; i + 7 < size; i += 8) {
        unsigned op = data[i] / 2;
        char buffer[8] = {};
        memmove(buffer, &data[i + 1], 7);
        if (op == 0) {
          mkdir(buffer);
        } else {
          chdir(buffer);
        }
      }
    }
    ```

- 结果：

  - 运行速度明显比 helloworld OS 更慢，约为每秒数十次。
  - 人工设置 bug，难以把握其影响程度。过于明显的错误会直接导致崩溃，没有意义；否则因为运行效率不高，很难通过崩溃或超时找到 bug。
  - mkdir-chdir 发现磁盘内容并不会恢复，需要具体研究 snapshot 机制。
  - 设想的简单 deserialization 方式：连续的 syscall ID + args（根据数据类型决定长度，buffer 需指定长度）

