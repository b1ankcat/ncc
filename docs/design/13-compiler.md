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

标准优化（内联、常量传播、死代码消除、循环与内存优化、向量化）直接复用
MLIR 与 LLVM 的既有 Pass，清单见
[14-performance.md](14-performance.md#标准优化-pass)。下面只列 NCC 自己引入的。

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

缓存分两级：**模块级**键由源文件哈希、编译器版本、优化标志、目标三元组和所依赖
模块的哈希构成；**包级**键还要计入 feature、锁文件与构建脚本输入，完整清单见
[10-packages.md](10-packages.md#全局缓存结构)。模块级缓存位于项目内，包级缓存
跨项目共享。

## 编译选项

```bash
# 默认编译（MLIR → LLVM IR → 机器码）
ncc build

# 输出 MLIR
ncc build --emit-mlir

# 输出 LLVM IR
ncc build --emit-llvm

# 输出汇编
ncc build --emit-asm

# 指定优化级别
ncc build --opt-level=3

# GPU 目标
ncc build --target=gpu-cuda
ncc build --target=gpu-rocm

# 查看优化 Pass
ncc build --verbose-passes
```

## 调试支持

### MLIR 可视化

```bash
# 查看每个 Pass 后的 MLIR
ncc build --mlir-print-ir-after-all

# 生成 Pass Pipeline 图
ncc build --mlir-print-ir-module-scope --mlir-print-local-scope
```

### Debug Info 生成

```mlir
// MLIR 包含源码位置信息
%result = ncc.vector.push %v, %val loc("main.ncc":10:5)
```

最终生成 DWARF debug info，支持 gdb/lldb 调试。

编译速度与生成代码性能的目标见
[14-performance.md](14-performance.md#验收目标)。

## 符号 Mangling 与 ABI

NCC 采用 **Itanium C++ ABI** mangling 规则的扩展，保证与现有工具链（GCC、Clang、
LLVM 调试器、链接器）的兼容性。

### 基本规则

1. **模块名编码为命名空间**：NCC 的模块在 mangling 中表示为 C++ 命名空间
2. **核心库类型在全局命名空间**：`String`、`Vector` 等核心库符号不带模块前缀
3. **泛型实例使用模板参数编码**：`Vector<int32_t>` 编码为模板实例化

### 编码规则

| NCC 符号 | Mangled Name | 说明 |
|---------|--------------|------|
| `geometry::Vec2` | `_ZN8geometry4Vec2E` | 模块名作为命名空间 |
| `String` | `_Z6String` | 核心库类型，全局命名空间 |
| `Vector<int32_t>` | `_Z6VectorIiE` | 泛型实例，`i` 是 `int32_t` |
| `Vector<Vec2>` | `_Z6VectorIN8geometry4Vec2EE` | 嵌套：Vector<geometry::Vec2> |
| `net.http::Client` | `_ZN3net4http6ClientE` | 嵌套模块（`.` → 两层） |

### 类型参数编码

遵循 Itanium ABI 的类型编码：

| NCC 类型 | 编码 |
|---------|------|
| `int32_t` | `i` |
| `int64_t` | `l` |
| `uint32_t` | `j` |
| `uint64_t` | `m` |
| `float` | `f` |
| `double` | `d` |
| `bool` | `b` |
| `String` | `6String` |
| 指针 `T*` | `P<T编码>` |
| 引用 `T&` | `R<T编码>` |

### 函数签名编码

```cpp
// geometry.ncc
export module geometry;
export double distance(Vec2 a, Vec2 b);
// mangled: _ZN8geometry8distanceENS_4Vec2ES0_

// 泛型函数
export comp T max<type T>(T a, T b);
// 实例化 max<int32_t>
// mangled: _ZN8geometry3maxIiEET_S0_S0_
```

### 跨模块实例化去重

编译器为每个泛型实例生成唯一的 mangled name，链接器使用 **weak symbols** 自动合并：

```cpp
// moduleA.ncc 实例化 Vector<int32_t>
// 生成符号：_Z6VectorIiE (weak)

// moduleB.ncc 也实例化 Vector<int32_t>
// 生成符号：_Z6VectorIiE (weak)

// 链接时自动合并为一个符号
```

### ABI 稳定性

**ABI 版本号**：编译器生成的对象文件包含 ABI 版本标记：

```
.section .ncc_abi_version
.byte 1  // 主版本
.byte 0  // 次版本
```

**兼容性规则**：
- 主版本相同：二进制兼容
- 主版本不同：不兼容，需要重新编译
- Mangling 规则变化会增加主版本号

**稳定性保证**：
- 类型的物理布局（大小、对齐、字段偏移）
- 函数调用约定
- 异常处理机制
- Mangled name 格式

### 与 C++ 互操作

NCC 的 mangling 兼容 C++，允许直接链接 C++ 对象文件：

```cpp
// C++ 库（编译为 .o）
namespace geometry {
    struct Vec2 { double x, y; };
    double distance(Vec2 a, Vec2 b);
}

// NCC 代码可以直接链接并调用
// mangled name 一致：_ZN8geometry8distanceENS_4Vec2ES0_
```

### Demangling

使用标准 `c++filt` 工具解码：

```bash
$ echo "_ZN8geometry4Vec2E" | c++filt
geometry::Vec2

$ echo "_Z6VectorIiE" | c++filt
Vector<int>
```

NCC 编译器提供 `ncc demangle` 命令，输出格式与 `c++filt` 一致。