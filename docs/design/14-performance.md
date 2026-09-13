# 十四、性能基准与优化

NCC 的性能章节给出可重复 benchmark 目标，不构成所有程序的无条件保证。

## 性能目标

### 编译速度

| 指标 | 目标 | 对比 |
|------|------|------|
| 冷编译 | < 5 秒/万行代码 | 目标基线 |
| 增量编译 | < 1 秒（单文件修改） | 目标基线 |
| 并行度 | 接近线性扩展 | 8核约 7-8 倍速度 |
| 缓存命中率 | > 90% | 跨项目共享全局缓存 |

**编译速度是目标指标，不构成对所有项目的保证。**

### 运行时性能

| 场景 | 目标 | 基准 |
|------|------|------|
| 计算密集 | ≤ 5% 开销 | 与 C++ -O3 对比 |
| 内存操作 | ≤ 3% 开销 | Vector/String 与 std::vector/string 对比 |
| 函数调用 | 零开销 | 内联后无差异 |
| 虚函数调用 | 零开销 | 虚表机制与 C++ 相同 |
| 异常处理 | 零开销（无异常路径） | 与 C++ 异常相同 |

这些数值是目标基线，不是语言保证；必须在相同算法、数据、硬件、编译配置、预热
和计时范围下与 C++ 对照验证。

### GPU 性能

| 场景 | 目标 | 基准 |
|------|------|------|
| 数据并行 | 与手写 CUDA 相当 | ≤ 10% 差距 |
| Kernel 启动开销 | < 10 μs | CUDA 原生约 5-10 μs |
| 内存带宽利用率 | > 80% | 理论峰值带宽 |
| 多 GPU 扩展 | 接近线性 | 4 GPU 约 3.8x |

**GPU 目标：尽力优化到最快，依赖 MLIR GPU Dialect 的优化。**

### 并发性能

| 指标 | 目标 | 对比 |
|------|------|------|
| 任务创建开销 | < 1 μs | 目标基线 |
| Channel 吞吐量 | > 10M ops/s | 目标基线 |
| 上下文切换 | < 100 ns | 用户态调度 |
| 百万级并发 | 目标：支持 | 任务容量测试 |

**并发性能以统一 benchmark 测量为准。**

## MLIR 优化 Pass

NCC 依赖 MLIR/LLVM 提供的优化能力：

### 标准优化 Pass

```
1. 内联优化（Inlining）
   - 小函数自动内联
   - 消除函数调用开销
   
2. 常量传播（Constant Propagation）
   - 编译期计算常量表达式
   - 消除死代码
   
3. 死代码消除（Dead Code Elimination）
   - 移除未使用的代码
   - 减小二进制体积
   
4. 公共子表达式消除（CSE）
   - 避免重复计算
   
5. 循环优化
   - Loop Fusion（循环融合）
   - Loop Tiling（循环分块）
   - Loop Vectorization（向量化）
   - Loop Unrolling（循环展开）
   
6. 内存优化
   - Scalar Replacement（标量替换）
   - Memory Promotion（内存提升到寄存器）
   - Buffer Allocation（缓冲区分配优化）
   
7. 向量化
   - SIMD 指令生成
   - 自动向量化循环
```

### NCC 特定优化

```cpp
// 优化前：循环内可能多次扩容
Vector<int32_t> v;
for (int i = 0; i < 1000; ++i) {
    v.push(i);
}

// 优化器识别到循环次数是编译期已知的常量，将扩容提到循环外：
Vector<int32_t> v;
v.reserve(1000);              // 一次分配，循环内不再检查扩容
for (int i = 0; i < 1000; ++i) {
    v.push(i);                // 已知容量充足，扩容分支被消除
}
```

优化只是把扩容判断提出循环并消除已知为假的分支，`push` 仍然按
[通用容器原语](03-memory.md#通用容器原语)在未初始化尾槽上构造元素并递增 size。
优化器不会改写成 `data()[i] = ...` 加 `set_size()` 这类形式——那会跳过元素构造，
对非平凡类型是未定义行为。对平凡可复制类型，构造循环本身会被降级为
memset/memcpy，这是后端的既有优化，不改变契约。

### GPU 特定优化

```mlir
// 优化前：朴素并行
gpu.launch blocks(%bx) threads(%tx) {
    %i = compute_index %bx, %tx
    %val = load %input[%i]
    %result = compute %val
    store %result, %output[%i]
}

// 优化后：合并内存访问
gpu.launch blocks(%bx) threads(%tx) {
    // 1. Coalesced memory access（合并访问）
    %i = compute_index %bx, %tx
    %val = coalesced_load %input[%i]
    
    // 2. Shared memory optimization（共享内存优化）
    %shared = shared_memory_alloc
    store %val, %shared[%tx]
    gpu.barrier
    
    // 3. 计算
    %result = compute_optimized %shared
    
    // 4. Coalesced write
    coalesced_store %result, %output[%i]
}
```

## 性能测试框架

### Benchmark 支持

```cpp
import benchmark;

// 定义 benchmark
benchmark::register_case("vector_push", []() {
    Vector<int32_t> v;
    for (int i = 0; i < 10000; ++i) {
        v.push(i);
    }
});

benchmark::register_case("string_concat", []() {
    String s;
    for (int i = 0; i < 1000; ++i) {
        s += "x";
    }
});

// 运行
int main() {
    benchmark::run_all();
}
```

**输出：**

```
Running benchmarks...

vector_push          1,234,567 ops/s    810 ns/op
string_concat          234,567 ops/s  4,260 ns/op

2 benchmarks completed in 5.2s
```

### 性能对比测试

```cpp
import benchmark;

// NCC 实现
benchmark("ncc_vector", []() {
    Vector<int32_t> v;
    for (int i = 0; i < 10000; ++i) {
        v.push(i);
    }
});

// C++ 实现（通过 C wrapper）
benchmark("cpp_vector", []() {
    void* v = std_vector_create();
    for (int i = 0; i < 10000; ++i) {
        std_vector_push(v, i);
    }
    std_vector_destroy(v);
});
```

### GPU Benchmark

```cpp
import benchmark;
import gpu;

benchmark("gpu_saxpy", []() {
    size_t n = 10000000;
    Vector<float, Device::Gpu> x(n), y(n);
    float a = 2.5f;
    
    parallel<Device::Gpu>(n, [x_view = x.view(), y_view = y.view(), a](size_t i) {
        y_view[i] = a * x_view[i] + y_view[i];
    });
    
    gpu::synchronize();  // 等待完成
});

// 对比 CPU 版本
benchmark("cpu_saxpy", []() {
    size_t n = 10000000;
    Vector<float> x(n), y(n);
    float a = 2.5f;
    
    parallel<Device::Cpu>(n, [&](size_t i) {
        y[i] = a * x[i] + y[i];
    });
});
```

## 性能监控

### 编译时性能分析

```bash
# 显示编译时间分解
ccc build --time-trace

# 输出：
# Parsing:           0.5s
# Type checking:     1.2s
# MLIR generation:   0.8s
# MLIR optimization: 2.1s
# LLVM codegen:      1.4s
# Linking:           0.3s
# Total:             6.3s
```

### 运行时性能分析

```bash
# 生成性能分析数据
ccc build --profile

# 运行程序生成 perf 数据
./myapp

# 查看热点函数
ccc perf report
```

## 内存性能

### String 性能

```cpp
// SSO：≤22 字节内联存储，无堆分配，拷贝是 24 字节按位复制
String short_str = "abc";

// 长字符串拷贝为 O(n)：分配 + 复制，没有隐式共享
String s1 = "a fairly long string that exceeds the inline buffer";
String s2 = s1;             // O(n) 深拷贝

// 避免拷贝的两种显式方式
StringView view = s1;       // 零拷贝，不拥有
String s3 = move(s1);       // O(1)，接管缓冲区
```

`sizeof(String) == 24`，两种形态共用这 24 字节（布局见
[字符串设计](02-types.md#string-类型)）。不采用 COW，因此没有原子引用计数的开销，
也没有"某次写入意外触发深拷贝"这类难以预测的性能悬崖。

### Vector 性能

```cpp
// 预分配避免扩容
Vector<int32_t> v;
v.reserve(1000);  // 一次分配
for (int i = 0; i < 1000; ++i) {
    v.push(i);  // 无重新分配
}

// 内存布局
sizeof(Vector<T>) = 24 字节
  - 8 字节：数据指针
  - 8 字节：size
  - 8 字节：capacity

// 增长策略：2x（与 std::vector 相同）
```

### 智能指针开销

```cpp
// unique_ptr - 零开销
sizeof(unique_ptr<T>) = sizeof(T*)  // 8 字节

// shared_ptr - 引用计数（原子操作）
sizeof(shared_ptr<T>) = 16 字节
  - 8 字节：对象指针
  - 8 字节：控制块指针

// 原子引用计数开销：~10-20 ns per 增减
```

## 性能回退检测

### CI 性能测试

```yaml
# .github/workflows/perf.yml
name: Performance Tests

on: [push, pull_request]

jobs:
  benchmark:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      
      - name: Build
        run: ccc build --release
      
      - name: Run benchmarks
        run: ccc bench --output=results.json
      
      - name: Compare with baseline
        run: |
          ccc perf-compare \
            --baseline=main \
            --current=results.json \
            --threshold=5%
      
      # 如果性能下降 > 5%，CI 失败
```

### 性能回退报告

```
Performance regression detected:

  vector_push: 1,234,567 ops/s → 1,100,000 ops/s (-10.9%)
    ❌ REGRESSION: exceeds 5% threshold
    
  string_concat: 234,567 ops/s → 230,000 ops/s (-1.9%)
    ✅ OK: within threshold
    
Investigate vector_push performance drop.
```

## 优化技巧

### 1. 避免不必要的拷贝

```cpp
// 返回具名局部对象：允许 NRVO，否则按标准返回规则移动
Vector<Data> get_data() {
    Vector<Data> v;
    // ... 填充 v
    return v;  // 不写 move(v)，保留 NRVO 的机会
}

// 需要复用调用方已有缓冲区时，可使用输出参数
void get_data(Vector<Data>& out) {
    out.reserve(expected_size);
    // ... 填充 out
}
```

具名局部对象的 NRVO 不是强制保证；未发生 NRVO 时，符合条件的返回表达式
按 C++ 规则选择移动或拷贝操作。Vector 的移动由容器实现，`move()` 本身不
执行资源转移。输出参数适用于已有缓冲区复用，并非普遍优于返回值。

### 2. 使用视图避免拷贝

```cpp
// ✗ 差：子串拷贝
String process(const String& text) {
    String prefix = text.substr(0, 10);  // 拷贝
    return prefix;
}

// ✓ 好：零拷贝视图
StringView process(const String& text) {
    return text.substr(0, 10);  // StringView，零拷贝
}
```

### 3. 循环优化

```cpp
// 朴素循环：是否向量化取决于后端能否证明无别名
for (size_t i = 0; i < n; ++i) {
    result[i] = a[i] + b[i];
}

// ✓ 显式提示：attribute 可忽略，删掉它程序语义不变
[[ncc::vectorize]] for (size_t i = 0; i < n; ++i) {
    result[i] = a[i] + b[i];
}

// ✓ 更好：使用 parallel，兼得分块与向量化
parallel<Device::Cpu>(n, [&](size_t i) {
    result[i] = a[i] + b[i];
});
```

没有 `#pragma`：预处理器已整体删除，向量化提示改用
[attribute](00-overview.md)，并遵循"attribute 必须可忽略"的判据。

### 4. 内存对齐

```cpp
// 确保 SIMD 对齐
alignas(64) float data[1024];  // 缓存行对齐

// Vector 自动对齐（对于 SIMD）
Vector<float> v(1024);  // 内部数据自动对齐
```

### 5. GPU 优化

```cpp
// ✗ 差：多次内存传输
auto gpu_data = cpu_data.to_device();
parallel<Device::Gpu>(n, [view = gpu_data.view()](size_t i) { /* ... */ });
auto result1 = gpu_data.to_host();  // 传输 1

gpu_data = result1.to_device();
parallel<Device::Gpu>(n, [view = gpu_data.view()](size_t i) { /* ... */ });
auto result2 = gpu_data.to_host();  // 传输 2

// ✓ 好：批量处理，减少传输
auto gpu_data = cpu_data.to_device();

parallel<Device::Gpu>(n, [view = gpu_data.view()](size_t i) { /* 操作 1 */ });
parallel<Device::Gpu>(n, [view = gpu_data.view()](size_t i) { /* 操作 2 */ });

auto result = gpu_data.to_host();  // 只传输一次
```

## 验收目标（尚无实测数据）

> ⚠️ **以下是设计阶段设定的验收门槛，不是测量结果。** 编译器尚未实现，
> 任何具体百分比都无从测得。这些数字的作用是：实现完成后，若 benchmark
> 达不到该门槛，视为需要修复的性能缺陷。

### CPU 验收门槛

以相同算法、数据和编译配置下的 C++ `-O3` 为基准：

| 测试 | 门槛 | 理由 |
|------|------|------|
| Vector push / String concat | 不慢于 5% | 容器实现策略与 std 相近 |
| 函数调用、虚函数调用 | 不慢于 1% | 降低到相同的 LLVM IR 与虚表布局 |
| 异常处理（无异常路径） | 不慢于 1% | 复用 LLVM 的 invoke/landingpad |

### GPU 验收门槛

以针对同一算法手写的 CUDA 为基准：

| 测试 | 门槛 |
|------|------|
| SAXPY（带宽受限） | 不慢于 10% |
| 矩阵乘法 | 不慢于 15% |
| 卷积 | 不慢于 15% |

达标与否必须由 [性能测试框架](#性能测试框架)在统一硬件和预热条件下判定，
并纳入 CI 回归检测。

## 性能优化路线图

### 短期（第一版发布）

- ✅ 依赖 MLIR/LLVM 标准优化
- ✅ 基本的内联和常量传播
- ✅ 简单的循环优化

### 中期（v1.1-v1.5）

- ⏳ NCC 特定的模式识别优化
- ⏳ 更激进的内联策略
- ⏳ 逃逸分析（栈分配优化）
- ⏳ GPU kernel fusion（kernel 融合）

### 长期（v2.0+）

- 📋 Profile-Guided Optimization (PGO)
- 📋 链接时优化（LTO）增强
- 📋 自动并行化
- 📋 多 GPU 自动分片

## 下一步

- 查看 [13-compiler.md](13-compiler.md) 了解编译器架构
- 查看 [09-gpu.md](09-gpu.md) 了解 GPU 优化细节
- 查看 [08-concurrency.md](08-concurrency.md) 了解并发性能
