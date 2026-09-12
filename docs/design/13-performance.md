# 十三、性能基准与优化

NCC 的性能目标：**与手写 C++ 性能差距在 5% 以内，GPU 代码尽力优化到最快。**

## 性能目标

### 编译速度

| 指标 | 目标 | 对比 |
|------|------|------|
| 冷编译 | < 5 秒/万行代码 | Go: ~2秒/万行, C++: ~20秒/万行 |
| 增量编译 | < 1 秒（单文件修改） | Go: ~0.5秒, C++: ~3秒 |
| 并行度 | 接近线性扩展 | 8核约 7-8 倍速度 |
| 缓存命中率 | > 90% | 跨项目共享全局缓存 |

**编译速度定位：介于 Go（最快）和 C++（最慢）之间，目标是 Rust 级别。**

### 运行时性能

| 场景 | 目标 | 基准 |
|------|------|------|
| 计算密集 | ≤ 5% 开销 | 与 C++ -O3 对比 |
| 内存操作 | ≤ 3% 开销 | Vector/String 与 std::vector/string 对比 |
| 函数调用 | 零开销 | 内联后无差异 |
| 虚函数调用 | 零开销 | 虚表机制与 C++ 相同 |
| 异常处理 | 零开销（无异常路径） | 与 C++ 异常相同 |

**性能承诺：依赖 MLIR/LLVM 的优化能力，保证与 C++ 性能差距 ≤ 5%。**

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
| 任务创建开销 | < 1 μs | Go goroutine: ~0.5μs |
| Channel 吞吐量 | > 10M ops/s | Go channel: ~20M ops/s |
| 上下文切换 | < 100 ns | 用户态调度 |
| 百万级并发 | ✅ 支持 | 类似 Go |

**并发定位：接近 Go 的性能，用户态调度 + 工作窃取。**

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
// 优化前
Vector<int32_t> v;
for (int i = 0; i < 1000; ++i) {
    v.push(i);
}

// NCC 优化器识别模式：
// 1. 预分配容量（避免多次扩容）
// 2. 批量初始化（避免逐个 push）

// 优化后（等价代码）
Vector<int32_t> v;
v.reserve(1000);  // 预分配
for (int i = 0; i < 1000; ++i) {
    v.data()[i] = i;  // 直接写入，无边界检查
}
v.set_size(1000);
```

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
benchmark("vector_push", []() {
    Vector<int32_t> v;
    for (int i = 0; i < 10000; ++i) {
        v.push(i);
    }
});

benchmark("string_concat", []() {
    String s;
    for (int i = 0; i < 1000; ++i) {
        s += "x";
    }
});

// 运行
int main() {
    run_benchmarks();
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
    
    parallel<Device::Gpu>(n, [=](size_t i) {
        y[i] = a * x[i] + y[i];
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
// COW (Copy-On-Write) - 避免不必要的拷贝
String s1 = "Hello, World!";
String s2 = s1;  // 零拷贝，共享数据
s2 += "!";       // 此时才拷贝

// SSO (Small String Optimization) - 短字符串无堆分配
String short_str = "abc";  // 内联存储，sizeof = 24 字节

// 性能特征
sizeof(String) = 24 字节
  - 8 字节：指针或内联数据开始
  - 8 字节：长度
  - 8 字节：容量/引用计数
```

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
// ✗ 差：不必要的拷贝
Vector<Data> get_data() {
    Vector<Data> v;
    // ... 填充 v
    return v;  // 可能触发拷贝
}

// ✓ 好：RVO (Return Value Optimization)
Vector<Data> get_data() {
    Vector<Data> v;
    // ... 填充 v
    return v;  // 编译器优化为零拷贝
}

// ✓ 更好：预分配
void get_data(Vector<Data>& out) {
    out.reserve(expected_size);
    // ... 填充 out
}
```

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
// ✗ 差：未向量化
for (size_t i = 0; i < n; ++i) {
    result[i] = a[i] + b[i];
}

// ✓ 好：提示编译器可向量化
#pragma clang loop vectorize(enable)
for (size_t i = 0; i < n; ++i) {
    result[i] = a[i] + b[i];
}

// ✓ 更好：使用 parallel（自动向量化）
parallel<Device::Cpu>(n, [&](size_t i) {
    result[i] = a[i] + b[i];
});
```

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
parallel<Device::Gpu>(n, [gpu_data](size_t i) { /* ... */ });
auto result1 = gpu_data.to_host();  // 传输 1

gpu_data = result1.to_device();
parallel<Device::Gpu>(n, [gpu_data](size_t i) { /* ... */ });
auto result2 = gpu_data.to_host();  // 传输 2

// ✓ 好：批量处理，减少传输
auto gpu_data = cpu_data.to_device();

parallel<Device::Gpu>(n, [gpu_data](size_t i) { /* 操作 1 */ });
parallel<Device::Gpu>(n, [gpu_data](size_t i) { /* 操作 2 */ });

auto result = gpu_data.to_host();  // 只传输一次
```

## 性能对比数据（预期）

### 基准测试对比

| 测试 | NCC | C++ | Rust | Go | 差距 |
|------|-----|-----|------|-----|------|
| Vector push | 100% | 98% | 95% | 85% | +2% |
| String concat | 100% | 102% | 98% | 90% | -2% |
| 函数调用 | 100% | 100% | 100% | 95% | 0% |
| 虚函数调用 | 100% | 100% | N/A | N/A | 0% |
| 异常处理 | 100% | 100% | N/A | N/A | 0% |

*基准：C++ -O3 = 100%，数值越高越快*

### GPU 性能对比

| 测试 | NCC | 手写 CUDA | 差距 |
|------|-----|----------|------|
| SAXPY | 95% | 100% | -5% |
| 矩阵乘法 | 92% | 100% | -8% |
| 卷积 | 90% | 100% | -10% |

*基准：手写 CUDA = 100%*

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

- 查看 [11-compiler.md](11-compiler.md) 了解编译器架构
- 查看 [09-gpu.md](09-gpu.md) 了解 GPU 优化细节
- 查看 [08-concurrency.md](08-concurrency.md) 了解并发性能
