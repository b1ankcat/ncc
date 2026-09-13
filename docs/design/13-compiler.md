# 十三、编译器架构

## 核心设计：MLIR 统一后端

NCC 使用 **MLIR (Multi-Level Intermediate Representation)** 作为编译器后端，统一处理 CPU 和 GPU 代码生成。

**选择 MLIR 的理由：**
1. **CPU/GPU 统一**：单一 IR 同时支持 CPU 和 GPU 目标
2. **渐进式降低**：高层语义逐步降低到硬件指令
3. **可扩展方言**：支持自定义操作和优化
4. **成熟生态**：LLVM 项目支持，被 TensorFlow/PyTorch 采用

## 编译流程

```
源代码 (*.ncc)
    ↓
Lexer (词法分析)
    ↓
Parser (语法分析)
    ↓
AST (抽象语法树)
    ↓
HIR (高层 IR) - 展开语法糖
    ↓
Type Checking + Comp Execution (类型检查 + 编译期求值)
    ↓
Coroutine Lowering (协程 → 状态机)
    ↓
MLIR Generation
    ↓
┌─────────────────────────────────────────────┐
│ MLIR Dialect Lowering (方言降低)            │
│                                             │
│   ncc dialect (NCC 特定操作)                │
│       ↓                                     │
│   affine/scf (循环和控制流)                 │
│       ↓                                     │
│   ┌───────────────┬───────────────────┐     │
│   │   CPU 路径    │     GPU 路径      │     │
│   │      ↓        │        ↓          │     │
│   │  llvm dialect │   gpu dialect     │     │
│   │      ↓        │        ↓          │     │
│   │   LLVM IR     │  ┌─────┬───────┐  │     │
│   │               │  │ PTX │ ROCDL │  │     │
│   │               │  └─────┴───────┘  │     │
│   └───────────────┴───────────────────┘     │
└─────────────────────────────────────────────┘
         ↓                    ↓
   x86/ARM 机器码     NVIDIA/AMD GPU 机器码
```

## 编译阶段详解

### 1. Lexer（词法分析）

将源代码转换为 token 流：

```cpp
// 输入
Vector<int32_t> v{1, 2};

// Token 流
IDENTIFIER("Vector")
LESS
IDENTIFIER("int32_t")
GREATER
IDENTIFIER("v")
LBRACE
NUMBER(1)
COMMA
NUMBER(2)
RBRACE
SEMICOLON
```

### 2. Parser（语法分析）

构建 AST：

```
VarDecl
  ├─ type: GenericType
  │   ├─ name: "Vector"
  │   └─ args: [int32_t]
  ├─ name: "v"
  └─ init: BraceInit
      ├─ 1
      └─ 2
```

### 3. HIR（高层 IR）

简化 AST，展开语法糖：

```
- 聚合初始化 → 字段赋值序列
- for 循环 → while 循环
- 运算符重载 → 函数调用
- 范围 for → 迭代器循环（含 Generator 与 Receiver 的遍历）
```

`match(value, handler...)` 按普通函数调用解析，绑定到库中的 comp 函数。
穷尽检查和分发代码生成由该函数通过公开的 comp/反射能力完成；编译器不设置
专用 match AST 节点。生成的枚举分发与用户写出的同等控制流使用相同的降低规则。

**协程降低**在 HIR 之后、MLIR 生成之前进行：含 `co_yield` / `co_return` 的函数
变换为状态机，局部变量提升到协程帧，`co_yield` 处切分基本块并记录恢复点。由于
[没有 `co_await` 与 awaiter 协议](00-overview.md)，不需要 symmetric transfer 与
`await_transform` 的处理，挂起点集合是封闭的（只有 `co_yield` 和函数结束）。
帧默认堆分配；生成器为不逃逸的局部变量时允许把帧提升到调用者栈上，此优化不作承诺。

### 4. Type Checking + Comp Execution

**多阶段类型检查与 comp 执行：**

```
阶段 1：基础类型收集
  - 收集所有用户定义类型（struct/class/enum）
  - 收集函数签名
  - 不执行 comp 函数

阶段 2：Comp 函数执行（迭代直到收敛）
  - 执行所有 comp 类型生成（Vector<int32_t> 等）
  - 执行反射驱动的代码生成
  - 生成新类型定义加入类型表
  - 如果生成了新类型，重复此阶段

阶段 3：完整类型检查
  - 检查所有表达式类型
  - 检查函数调用
  - 检查 lambda 捕获（GPU 安全性）

阶段 4：单态化（Monomorphization）
  - 为每个泛型实例生成具体代码
  - Vector<int32_t>, Vector<float> → 独立的类型和函数
```

**依赖图分析：**

```text
依赖调度算法（非 NCC 源代码）：
1. 初始化待执行集合与已完成集合。
2. 从待执行集合中取出依赖已满足的操作，构成当前批次。
3. 执行当前批次，将完成项移入已完成集合。
4. 若待执行项尚存但没有可执行批次，报告未解决依赖或循环依赖。
5. 重复上述过程，直到待执行集合为空。
```

### 5. MLIR 代码生成

#### NCC Dialect（自定义方言）

定义 NCC 特有的高层操作：

```mlir
// Vector<int32_t> v;
%v = ncc.vector.create : !ncc.vector<i32>

// v.push(42);
ncc.vector.push %v, %c42 : !ncc.vector<i32>, i32

// parallel<Device::Gpu>(n, [](size_t i) { ... })
ncc.parallel<gpu> (%i : index) in [0, %n) {
    // loop body
}

// Tagged enum
%shape = ncc.tagged_enum.create @Shape::Circle(%radius) : !ncc.enum<Shape>
ncc.tagged_enum.dispatch %shape {
    @Circle(%r) => { ... },
    @Rect(%w, %h) => { ... },
    @Point => { ... }
}

// 异常
ncc.throw %exception : !ncc.exception
ncc.try {
    ...
} catch @IOException(%e) {
    ...
}
```

`ncc.tagged_enum.dispatch` 仅表示生成后的枚举分发语义，不是源语言的 match
语法，也不依赖被调用函数的名字；用户生成的同等枚举分发同样可以使用此 IR。

#### 方言降低（Dialect Lowering）

**第一步：NCC → Affine/SCF**

```mlir
// NCC parallel 降低为 SCF parallel
ncc.parallel<cpu> (%i : index) in [0, %n) {
    %val = load %array[%i]
    %result = muli %val, %val
    store %result, %output[%i]
}
↓
scf.parallel (%i) = (%c0) to (%n) step (%c1) {
    %val = memref.load %array[%i]
    %result = arith.muli %val, %val
    memref.store %result, %output[%i]
}
```

**第二步：分支到 CPU/GPU 路径**

```mlir
// CPU 路径：SCF → LLVM Dialect
scf.parallel → llvm.call @openmp_parallel
↓
LLVM IR
↓
x86/ARM 机器码

// GPU 路径：SCF → GPU Dialect
scf.parallel<gpu> → gpu.launch
↓
gpu.module {
    gpu.func @kernel(%arg0: memref<?xf32>) {
        %tid = gpu.thread_id x
        // ...
    }
}
↓
CUDA PTX / ROCm LLGPU
```

#### 异常处理的 MLIR 表示

```mlir
// try-catch 降低为 invoke + landingpad
func.func @process() {
    %result = ncc.try {
        %file = call @open_file(%path) : (!ncc.string) -> !ncc.file
        call @process_file(%file) : (!ncc.file) -> ()
        ncc.yield
    } catch {
    ^bb_catch(%exception: !ncc.exception):
        %is_io = ncc.exception.isa %exception, @IOException
        cf.cond_br %is_io, ^bb_handle_io, ^bb_rethrow
    ^bb_handle_io:
        // handle exception
        cf.br ^bb_exit
    ^bb_rethrow:
        ncc.throw %exception
    }
^bb_exit:
    return
}

// 最终降低为 LLVM 的 invoke/landingpad
↓
llvm.func @process() {
    %result = llvm.invoke @open_file(%path) 
        to ^bb_normal unwind ^bb_unwind
^bb_normal:
    ...
^bb_unwind:
    %landingpad = llvm.landingpad ...
    ...
}
```

## GPU 代码生成

### GPU Dialect 使用

```mlir
// parallel<Device::Gpu>(n, [view](size_t i) { view[i] = i * 2; })

module {
    // GPU kernel
    gpu.module @kernels {
        gpu.func @kernel_parallel(%data: memref<?xi32>, %n: index) 
            kernel {
            %tid = gpu.thread_id x
            %bid = gpu.block_id x
            %bdim = gpu.block_dim x
            
            %base = arith.muli %bid, %bdim : index
            %i = arith.addi %base, %tid : index
            
            %in_range = arith.cmpi ult, %i, %n : index
            scf.if %in_range {
                %val = arith.muli %i, %c2 : i32
                memref.store %val, %data[%i] : memref<?xi32>
            }
            gpu.return
        }
    }
    
    // Host code
    func.func @main() {
        %data = memref.alloc() : memref<?xi32>
        %data_gpu = gpu.memcpy %data, host_to_device
        
        // n == 0 时跳过 kernel；否则向上取整，覆盖尾部元素
        %grid_dim = arith.ceildivui %n, %c256 : index
        %block_dim = %c256
        
        gpu.launch_func @kernels::@kernel_parallel
            blocks in (%grid_dim, %c1, %c1)
            threads in (%block_dim, %c1, %c1)
            args(%data_gpu : memref<?xi32>, %n : index)
        
        %result = gpu.memcpy %data_gpu, device_to_host
        return
    }
}
```

### GPU 后端降低

```
GPU Dialect
    ↓
gpu-to-nvvm (NVIDIA)  或  gpu-to-rocdl (AMD)
    ↓
NVVM Dialect (PTX)    或  ROCDL Dialect
    ↓
PTX Assembly          或  LLGPU
    ↓
CUDA Runtime          或  ROCm Runtime
```

## 优化 Pass

### MLIR 提供的标准优化

```
1. 内联优化（Inlining）
2. 常量传播（Constant Propagation）
3. 死代码消除（Dead Code Elimination）
4. 循环优化：
   - Loop Fusion（循环融合）
   - Loop Tiling（循环分块）
   - Loop Vectorization（向量化）
5. 内存优化：
   - Buffer Allocation（缓冲区分配）
   - Memory Promotion（内存提升）
```

### NCC 特定优化

```mlir
// 优化前：Vector 多次分配
%v1 = ncc.vector.create
ncc.vector.push %v1, %val1
ncc.vector.push %v1, %val2
ncc.vector.push %v1, %val3

// 优化后：预分配容量
%v1 = ncc.vector.create_with_capacity %c3
ncc.vector.push_unchecked %v1, %val1
ncc.vector.push_unchecked %v1, %val2
ncc.vector.push_unchecked %v1, %val3
```

## 增量编译

### 模块级缓存

```
src/main.ncc    → target/.ccc-cache/main.mlir   → target/.ccc-cache/main.o
src/utils.ncc   → target/.ccc-cache/utils.mlir  → target/.ccc-cache/utils.o
src/models.ncc  → target/.ccc-cache/models.mlir → target/.ccc-cache/models.o

只重新编译修改过的模块
```

**缓存键计算：**

```
cache_key = sha256(
    source_file_hash,
    compiler_version,
    optimization_flags,
    target_triple,
    dependency_hashes
)
```

## 编译选项

```bash
# 默认编译（MLIR → LLVM IR → 机器码）
ccc build

# 输出 MLIR
ccc build --emit-mlir

# 输出 LLVM IR
ccc build --emit-llvm

# 输出汇编
ccc build --emit-asm

# 指定优化级别
ccc build --opt-level=3

# GPU 目标
ccc build --target=gpu-cuda
ccc build --target=gpu-rocm

# 查看优化 Pass
ccc build --verbose-passes
```

## 调试支持

### MLIR 可视化

```bash
# 查看每个 Pass 后的 MLIR
ccc build --mlir-print-ir-after-all

# 生成 Pass Pipeline 图
ccc build --mlir-print-ir-module-scope --mlir-print-local-scope
```

### Debug Info 生成

```mlir
// MLIR 包含源码位置信息
%result = ncc.vector.push %v, %val loc("main.ncc":10:5)
```

最终生成 DWARF debug info，支持 gdb/lldb 调试。

## 性能目标

- **编译速度**：< 1秒/1000行代码（增量编译）
- **生成代码性能**：与手写 C++/CUDA 相当（LLVM 优化级别 -O3）
- **GPU kernel 启动开销**：< 10μs

## 与其他编译器对比

| 编译器 | 后端 | CPU 支持 | GPU 支持 | 增量编译 |
|--------|------|---------|---------|---------|
| Rust | LLVM | ✅ | ❌ | ✅ |
| Swift | LLVM | ✅ | ❌ | ✅ |
| Julia | LLVM | ✅ | 部分 | ❌ |
| Mojo | MLIR | ✅ | ✅ | ✅ |
| **NCC** | **MLIR** | **✅** | **✅** | **✅** |

## 下一步

- 查看 [09-gpu.md](09-gpu.md) 了解 GPU 编译细节
- 查看 [11-build-system.md](11-build-system.md) 了解构建系统
