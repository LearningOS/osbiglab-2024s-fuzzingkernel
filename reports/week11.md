# Week 11

## Fuzzing PMM module

- LibAFL 进行了一些修复，现在 LibAFL Qemu 基本可以正常使用了。

- HelloworldOS 新增了 PMM 模块，接口简单，可以测试。
- 对 OS 的修改：
  - 启动参数中 `-machine` 从 q35 改为默认值 i440FX（不指定即可），可能是因为 LibAFL 没有做好对 ICH9 的支持，导致恢复状态时出错（即 week7 运行 HelloworldOS 时出现的问题）。相关参考：[【虚拟化qemu】Q35 与 I440FX - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/664866414)
  - 修复了 log 等级为 `LOG_OFF` 时依然会输出颜色控制字符的问题。

- 编写 harness：将一个字节视为一个参数，交替进行申请和释放。

  ```c
  // modules/test/test.c
  
  typedef struct ppn_test_entry {
  	bool valid;
  	uint32_t ppn;
  	uint32_t size;
  } ppn_test_entry;
  
  static void pmm_fuzz(uint8_t *buffer, size_t size) {
  	ppn_test_entry entries[TEST_BUF_SIZE] = {};
  
  	for (size_t i = 0; i < size; ++i) {
  		uint8_t idx = buffer[i] % TEST_BUF_SIZE;
  		if (entries[idx].valid) {
  			pmm_free(entries[idx].ppn, entries[idx].size);
  			entries[idx].valid = false;
  		} else {
  			entries[idx].ppn = pmm_alloc(buffer[i]);
  			entries[idx].size = buffer[i];
  			entries[idx].valid = true;
  		}
  	}
  
  	buffer[0] = 0; // avoid optimization
  }
  
  void test(void)
  {
  	pr_info("start test\n");
  	size_t size = LIBAFL_QEMU_START_VIRT((size_t) buffer, INPUT_BUF_SIZE);
  	pmm_fuzz(buffer, size);
  	LIBAFL_QEMU_END(LIBAFL_QEMU_END_OK);
  }
  ```

- 结果：正常运行，效率合理，执行几次即可达到很高的覆盖率，目前还没有找到 bug。

  ```
  [UserStats   #1]  (GLOBAL) run time: 0h-8m-48s, clients: 1, corpus: 92, objectives: 0, executions: 471843, exec/sec: 893.8, edge%
                    (CLIENT) corpus: 92, objectives: 0, executions: 471843, exec/sec: 893.8, edges: 1889/1894 (99%)
  [Testcase    #1]  (GLOBAL) run time: 0h-8m-48s, clients: 1, corpus: 93, objectives: 0, executions: 472698, exec/sec: 895.5, edge%
                    (CLIENT) corpus: 93, objectives: 0, executions: 472698, exec/sec: 895.5, edges: 1889/1894 (99%)
  ```

- 问题：使用 coverage-guided 的方法，覆盖分支不等于没有问题；有些分支可能很难达到。

## Testcase deserialization

- 需要设计一种格式，将输入数据转换为实际的由 syscall 序列组成的测例

- 现有的方案：参考 [syzkaller](https://github.com/google/syzkaller/blob/master/docs/syscall_descriptions.md) 和 [Rtkaller](https://github.com/Rrooach/Rtkaller/blob/master/docs/features.md)，后者基于前者修改，二者类似（毕竟至今仍未得到 Tardis）

  - 抽象表示：used to [analyze](https://github.com/google/syzkaller/blob/master/prog/analysis.go), [generate](https://github.com/google/syzkaller/blob/master/prog/rand.go), [mutate](https://github.com/google/syzkaller/blob/master/prog/mutation.go), [minimize](https://github.com/google/syzkaller/blob/master/prog/minimization.go), [validate](https://github.com/google/syzkaller/blob/master/prog/validation.go), etc programs.

    ```go
    // prog/prog.go
    
    type Prog struct {
    	Target   *Target
    	Calls    []*Call
    	Comments []string
    
    	// Was deserialized using Unsafe mode, so can do unsafe things.
    	isUnsafe bool
    }
    
    type Call struct {
    	Meta    *Syscall
    	Args    []Arg
    	Ret     *ResultArg
    	Props   CallProps
    	Comment string
    }
    
    type Arg interface {
    	Type() Type
    	Dir() Dir
    	Size() uint64
    
    	validate(ctx *validCtx, dir Dir) error
    	serialize(ctx *serializer)
    }
    ```

    `Arg` 的实现有 `ConstArg`、`PointerArg`、`DataArg`、`GroupArg` 等。作为框架可以支持各种调用，但是对于具体的一个 fuzzer 不一定都有用。

  - 具体表示：does not contain rich type information (irreversible), used for actual execution (interpretation) of programs by [executor](https://github.com/google/syzkaller/blob/master/executor/executor.cc).

    ```go
    // prog/decodeexec.go
    
    type ExecProg struct {
    	Calls []ExecCall
    	Vars  []uint64
    }
    
    type ExecCall struct {
    	Meta    *Syscall
    	Props   CallProps
    	Index   uint64
    	Args    []ExecArg
    	Copyin  []ExecCopyin
    	Copyout []ExecCopyout
    }
    
    type ExecCopyin struct {
    	Addr uint64
    	Arg  ExecArg
    }
    
    type ExecCopyout struct {
    	Index uint64
    	Addr  uint64
    	Size  uint64
    }
    
    type ExecArg interface{} // one of ExecArg*
    
    type ExecArgConst struct {
    	Size           uint64
    	Format         BinaryFormat
    	Value          uint64
    	BitfieldOffset uint64
    	BitfieldLength uint64
    	PidStride      uint64
    }
    
    type ExecArgResult struct {
    	Size    uint64
    	Format  BinaryFormat
    	Index   uint64
    	DivOp   uint64
    	AddOp   uint64
    	Default uint64
    }
    
    type ExecArgData struct {
    	Data     []byte
    	Readable bool
    }
    
    type ExecArgCsum struct {
    	Size   uint64
    	Kind   uint64
    	Chunks []ExecCsumChunk
    }
    
    type ExecCsumChunk struct {
    	Kind  uint64
    	Value uint64
    	Size  uint64
    }
    ```

- 问题：迁移已有实现？做一个简单实现？

## Review on TSFFS

- 与其它框架对比：
  - 优点：简单，只需添加 SIMICS script，修改少量代码
  - 缺点：效率，sanitizers
- fuzzing 模式：
  - compiled-in：类似于 LibAFL Qemu 的 sync_exit，不是指 static instrumentation。支持同时存在多个 fuzzing 段，通过编号区分，运行时通过外部设置指定要运行的部分。
  - closed-box：在 SIMICS script 中手动调用 `start` 和 `end`，将测试数据注入内存空间。
  - manual：直接获取测试数据，自行处理。
- 问题：似乎使用 SIMICS 人数不多，除了文档帮助较少。