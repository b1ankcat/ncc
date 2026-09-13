# 九、GPU 并行与异构计算

NCC 通过统一的 `Device` 类型参数和 `parallel` API 支持 CPU/GPU 异构计算。

## 核心理念

- **统一的内存类型**：`Vector<T, Device>` 通过类型参数区分 CPU/GPU 内存
- **非拥有设备视图**：`DeviceView<T>` 只携带设备地址、长度和设备标识，不复制或拥有数据
- **显式内存迁移**：用户必须显式调用 `.to_device()` / `.to_host()`
- **编译期捕获检查**：GPU lambda 的捕获列表在编译期静态分析
- **运行时后端选择**：支持 CUDA/ROCm/OneAPI，运行时自动检测

## 设备类型参数

```cpp
enum class Device : int32_t {
    Cpu = 0,
    Gpu = 1,
};

// CPU 内存（默认）
Vector<float> cpu_vec(1000);               // 等价于 Vector<float, Device::Cpu>
Vector<float, Device::Cpu> cpu_vec2(1000);

// GPU 内存
Vector<float, Device::Gpu> gpu_vec(1000);
```

## GPU 数据并行

```cpp
import parallel;

// CPU 内存（默认）
Vector<float> cpu_data(1000000);

// 错误：不能直接在 GPU parallel 中捕获 CPU 数据
// parallel<Device::Gpu>(cpu_data.size(), [&](size_t i) {
//     cpu_data[i] = i;  // 编译错误：GPU lambda 捕获了 Device::Cpu 内存
// });

// 正确：显式迁移到 GPU 内存
Vector<float, Device::Gpu> gpu_data = cpu_data.to_device();

// GPU 数据并行：lambda 捕获非拥有视图，不复制 Vector
DeviceView<float> view = gpu_data.view();
auto done = parallel<Device::Gpu>(view.size(), [view](size_t i) {
    view[i] = view[i] * 2.0f + 1.0f;
});
done.wait();

// 读回到 CPU
Vector<float> result = gpu_data.to_host();
```

### `DeviceView<T>` 非拥有视图

```cpp
DeviceView<float> view = gpu_data.view();
DeviceView<const float> read_only = gpu_data.view();
```

`Vector<T, Device::Gpu>` 始终是拥有 GPU 内存的值类型；拷贝是否深拷贝由
Vector 契约决定。`DeviceView<T>` 只包含设备地址、长度、设备标识和必要的
stride/shape 信息，按值捕获只复制这些描述信息。视图不延长 Vector 生命周期，
提交的异步操作完成前所有者必须保持有效；`parallel` 返回的完成句柄必须
`wait()` 后才能释放所有者。CPU 指针和 CPU Vector 不能转成 GPU 视图。

## 内存迁移 API

### `to_device()` - CPU → GPU

```cpp
// 同步迁移（默认）
Vector<float, Device::Gpu> to_device() const;

// 异步迁移
Future<Vector<float, Device::Gpu>> to_device_async() const;

// 指定目标 GPU
Vector<float, Device::Gpu> to_device(GpuId gpu_id) const;
Vector<float, Device::Gpu> to_device(GpuId gpu_id, bool async) const;
```

**同步迁移示例：**

```cpp
Vector<float> cpu_data(1000000);

// 默认：同步迁移到当前 GPU
Vector<float, Device::Gpu> gpu_data = cpu_data.to_device();

// 指定 GPU 设备
Vector<float, Device::Gpu> gpu_data2 = cpu_data.to_device(GpuId{1});
```

**异步迁移示例：**

```cpp
Vector<float> cpu_data(1000000);

// 异步迁移（不阻塞）
auto future = cpu_data.to_device_async();

// 可以继续执行其他 CPU 工作
do_cpu_work();

// 等待 GPU 拷贝完成
Vector<float, Device::Gpu> gpu_data = future.get();

// 或者使用简化语法
Vector<float, Device::Gpu> gpu_data2 = cpu_data.to_device(GpuId{0}, true);
```

**语义规格：**

- **同步操作**：阻塞调用线程直到 GPU 内存分配和数据拷贝完成
- **深拷贝**：在 GPU 上分配新内存并拷贝数据，原始 CPU 数据保持不变
- **返回值**：新的 `Vector<T, Device::Gpu>` 对象，拥有独立的 GPU 内存
- **可多次调用**：每次调用都会分配新的 GPU 内存并拷贝数据

### `to_host()` - GPU → CPU

```cpp
// 同步读回（默认）
Vector<T, Device::Cpu> to_host() const;

// 异步读回
Future<Vector<T, Device::Cpu>> to_host_async() const;
```

**示例：**

```cpp
Vector<float, Device::Gpu> gpu_data(1000);
    auto done = parallel<Device::Gpu>(gpu_data.size(), [view = gpu_data.view()](size_t i) {
        view[i] = i * 2.0f;
    });
    done.wait();

// 同步读回 CPU
Vector<float> cpu_result = gpu_data.to_host();

// 异步读回
auto future = gpu_data.to_host_async();
do_other_work();
Vector<float> cpu_result2 = future.get();
```

**语义规格：**

- **同步操作**：阻塞直到 GPU → CPU 拷贝完成
- **深拷贝**：在 CPU 上分配新内存并从 GPU 拷贝数据，原始 GPU 数据保持不变
- **返回值**：新的 `Vector<T, Device::Cpu>` 对象（等价于 `Vector<T>`）
- **不释放 GPU 内存**：原始 GPU 对象仍然有效，可继续使用

## 编译期捕获检查

编译器静态分析 `parallel<Device::Gpu>` 的 lambda 捕获列表：

```cpp
// parallel 的运行时库实现（简化）
void parallel<Device device>(size_t n, auto func) {
    if (device == Device::Gpu) {
        // 实例化 GPU kernel 时执行编译期捕获检查
        for (auto capture : captures_of(^^decltype(func))) {
            auto capture_type = type_of(capture);
            auto capture_mode = capture_mode_of(capture);

            if (!gpu_capture_safe(capture_type, capture_mode)) {
                static_assert(false,
                    "GPU 捕获的值不是设备可传输类型：{}",
                    name_of(capture_type));
            }
        }
    }
    // 生成实际的并行代码...
}

// 设备可传输性：递归检查类型和捕获方式
comp bool gpu_capture_safe(type T, CaptureMode mode) {
    // GPU kernel 不能保存主机对象的引用；视图必须按值捕获
    if (mode == CaptureMode::ByReference) {
        return false;
    }

    // 标量按值复制到设备参数区
    if (is_scalar(T) && !is_pointer(T)) {
        return true;
    }

    // 非拥有视图携带明确的设备地址和布局
    if (is_template_instantiation_of(T, ^^DeviceView)) {
        return true;
    }

    // 聚合类型只有在所有成员都可传输时才能捕获
    if (is_aggregate(T)) {
        for (auto field : nonstatic_data_members_of(^^T)) {
            if (!gpu_capture_safe(type_of(field), CaptureMode::ByValue)) {
                return false;
            }
        }
        return true;
    }

    // 裸指针、引用、this、CPU 容器和未知设备类型默认拒绝
    return false;
}
```

**编译器检查规则：**

1. **标量类型可以值捕获**（复制到设备参数区）；裸指针不属于允许的标量
2. **`DeviceView<T>` 等非拥有 GPU 视图只能按值捕获**
3. **由标量和设备视图组成的聚合类型可以递归捕获**
4. **裸指针、引用、`this`、CPU 容器和未知地址空间类型会报错**

此检查只约束 `parallel<Device::Gpu>` 的调用边界，不检查普通 CPU 代码的指针
寿命或数据竞争，也不提供通用借用检查。GPU 数据访问统一通过 `DeviceView<T>`；
异步操作结束前，拥有视图所指内存的 Vector 必须保持有效。

**错误诊断示例：**

```cpp
Vector<float> cpu_data(1000);

// ✗ 编译错误
parallel<Device::Gpu>(cpu_data.size(), [&](size_t i) {
    cpu_data[i] = i;
});
// 错误信息：
// error: GPU lambda 捕获了 CPU 内存类型 'Vector<float, Device::Cpu>&'
//     parallel<Device::Gpu>(cpu_data.size(), [&](size_t i) {
//                                             ^
// note: 捕获变量 'cpu_data' 类型为 'Vector<float, Device::Cpu>&'
// note: 使用 .to_device() 将数据迁移到 GPU：
//       auto gpu_data = cpu_data.to_device();

// ✗ 编译错误：裸指针的地址空间未知
float* ptr = cpu_data.data();
parallel<Device::Gpu>(cpu_data.size(), [ptr](size_t i) {
    ptr[i] = 0.0f;
});
// error: raw pointer capture is not device-transferable
// help: create DeviceView<float> from GPU-owned storage

// ✓ 正确：显式迁移并捕获视图
auto gpu_data = cpu_data.to_device();
auto done = parallel<Device::Gpu>(gpu_data.size(), [view = gpu_data.view()](size_t i) {
    view[i] = i;
});
done.wait();

// ✓ 正确：标量值捕获
int32_t scale = 2;
parallel<Device::Gpu>(gpu_data.size(), [view = gpu_data.view(), scale](size_t i) {
    view[i] = view[i] * scale;  // scale 复制到常量内存
});
```

## GPU 后端选择与初始化

### 查询可用设备

```cpp
import gpu;

// 查询可用的 GPU 设备
auto devices = gpu::get_devices();
for (auto& dev : devices) {
    println("Device {}: {} ({})", 
        dev.id, dev.name, dev.backend);  // backend: Cuda, Rocm, OneAPI
}
```

### 设置使用的 GPU

```cpp
// 用户显式选择 GPU 设备（必须是同一后端）
gpu::set_devices({0, 1, 2, 3});  // 使用 GPU 0-3

// 检查：所有设备必须是同一后端（全 NVIDIA 或全 AMD）
// 如果混合后端，运行时抛出异常：
// RuntimeError: 不能混合使用不同后端的 GPU：
//   Device 0: NVIDIA RTX 4090 (Cuda)
//   Device 1: AMD Radeon RX 7900 XTX (Rocm)

// 默认行为：如果用户未调用 set_devices()
// 1. 使用 GPU 0（第一个可用 GPU）
// 2. 自动选择对应的后端（Cuda/Rocm/OneAPI）
// 3. 没有 GPU 时按开发者策略回退 CPU 或抛出 GpuUnavailable
```

### 指定目标 GPU

```cpp
// 迁移到特定 GPU
Vector<float, Device::Gpu> gpu_data = cpu_data.to_device(GpuId{1});

// 在特定 GPU 上执行
    auto done = parallel<Device::Gpu>(gpu_data.size(), [view = gpu_data.view()](size_t i) {
        view[i] = compute(i);
    });
    done.wait();

// 多 GPU 并行（数据自动分片）
gpu::set_devices({0, 1});
Vector<float, Device::Gpu> data(10000000);  // 自动分片到 GPU 0 和 1

parallel<Device::Gpu>(data.size(), [view = data.view()](size_t i) {
    view[i] = compute(i);
});
// 运行时自动在两个 GPU 上并行执行
```

### GPU 后端选择规则

1. **编译目标与运行设备分离**：
   - 编译器只根据目标后端生成代码，不要求构建机实际存在 GPU：
   ```
   error: 使用了 Device::Gpu 但系统未检测到 GPU 设备
       Vector<float, Device::Gpu> data;
                     ^
   note: 安装 NVIDIA/AMD GPU 驱动并重新编译
   note: 或者修改代码使用 Device::Cpu
   ```

2. **运行时后端选择**：
   - `gpu::set_devices()` 设置使用哪些 GPU（编号从 0 开始）
   - 运行时检查：所有设备必须是同一后端（全 CUDA 或全 ROCm）
   - 如果未调用 `set_devices()`，默认使用 GPU 0

3. **单一后端约束**：
   - 同一时间只能使用一种后端（CUDA 或 ROCm 或 OneAPI）
   - 不能在运行时切换后端
   - 不能同时使用 NVIDIA 和 AMD GPU

4. **多 GPU 支持（显式启用）**：
   - 可以使用多个同后端 GPU：`gpu::set_devices({0, 1, 2, 3})`
   - 只有启用分片策略时才自动分片，`parallel<Device::Gpu>` 按已选设备调度

### 后端检测示例

```cpp
import gpu;

int main() {
    // 检查 GPU 可用性
    if (gpu::available()) {
        auto info = gpu::get_backend_info();
        println("GPU Backend: {}", info.name);  // "CUDA 12.3" 或 "ROCm 6.0"
        println("Devices: {}", info.device_count);
        
        // 使用 GPU
        gpu::set_devices({0});
        Vector<float, Device::Gpu> data(1000000);
        // ...
    } else {
        println("No GPU available, using CPU");
        Vector<float> data(1000000);
        parallel<Device::Cpu>(data.size(), [&](size_t i) {
            data[i] = compute(i);
        });
    }
}
```

## 统一内存（高级用法）

```cpp
// 使用统一内存（需要硬件支持，CUDA 6.0+ / ROCm 4.0+）
UnifiedBuffer<float> data(1000000);

// CPU 初始化（页面在主机内存）
for (size_t i = 0; i < data.size(); ++i) {
    data[i] = i;
}

// GPU 计算（运行时自动迁移到 GPU）
parallel<Device::Gpu>(data.size(), [view = data.view()](size_t i) {
    view[i] = view[i] * 2.0f;
});

// CPU 读取（运行时自动迁移回 CPU）
println("Result: {}", data[0]);
```

**UnifiedBuffer 特性：**

- **自动页面迁移**：运行时根据访问模式在 CPU/GPU 之间迁移内存页
- **透明访问**：CPU 和 GPU 代码使用相同的指针
- **性能权衡**：页面故障开销较高，适合访问模式不规则的场景
- **硬件要求**：需要 PCIe Atomics 或 XGMI/NVLink 支持

## 矩阵乘法示例

```cpp
void matmul_cpu(Matrix& c, const Matrix& a, const Matrix& b) {
    parallel<Device::Cpu>(c.rows(), c.cols(), [&](size_t i, size_t j) {
        float sum = 0.0f;
        for (size_t k = 0; k < a.cols(); ++k) {
            sum += a(i, k) * b(k, j);
        }
        c(i, j) = sum;
    });
}

void matmul_gpu(Matrix& c, const Matrix& a, const Matrix& b) {
    // 迁移到 GPU
    auto a_gpu = a.to_device();
    auto b_gpu = b.to_device();
    Matrix<float, Device::Gpu> c_gpu(c.rows(), c.cols());
    
    // GPU 计算
    auto done = parallel<Device::Gpu>(c_gpu.rows(), c_gpu.cols(), [a = a_gpu.view(), b = b_gpu.view(), c = c_gpu.view()](size_t i, size_t j) {
        float sum = 0.0f;
        for (size_t k = 0; k < a_gpu.cols(); ++k) {
            sum += a_gpu(i, k) * b_gpu(k, j);
        }
        c(i, j) = sum;
    });
    done.wait();
    
    // 读回结果
    c = c_gpu.to_host();
}
```

## 编译期代码生成

一维 GPU 工作组统一使用 `block_id * block_dim + thread_id` 计算全局索引，
grid 大小按 `ceil(n / block_size)` 计算；`n == 0` 时不启动 kernel。所有全局
内存访问都生成 `index < n` 的边界判断，多维访问按行主序和显式 stride 计算。

编译器为 `Device::Gpu` 生成对应后端的内核代码：

- **CUDA 后端** → 生成 PTX/SASS
- **ROCm 后端** → 生成 GCN/RDNA ISA  
- **OneAPI 后端** → 生成 SPIR-V

后端目标在编译期由 `--target` 确定；实际设备和驱动在运行时检测。没有可用 GPU
时，开发者选择显式 CPU 回退或在执行 GPU 操作时抛出 `GpuUnavailable`。

## 下一步

- 查看 [08-concurrency.md](08-concurrency.md) 了解任务并发
- 查看 [10-compiler.md](10-compiler.md) 了解编译器架构
