# 八、并发与多线程

NCC 的并发模型基于两个核心范式：
1. **数据并行**：统一的 `parallel`/`reduce` API，通过执行策略选择 CPU 或 GPU
2. **任务并发**：Go 风格的轻量级任务（`Thread::spawn` + `Channel`）

无需新增关键字，全部通过 `comp` 函数和库实现。

## 核心理念

- **统一的并行 API**：`parallel` 和 `reduce` 接受执行策略，CPU/GPU 是实现后端
- **轻量级任务**：Go 风格的 goroutine（Thread::spawn，2KB 栈，M:N 调度）
- **通道通信**：Go 风格的 Channel，优先消息传递而非共享内存
- **零成本抽象**：只为实际选择的执行策略生成所需代码；任务调度本身是运行时库能力
- **无新增关键字**：所有特性通过库和 `comp` 函数实现

## 一、数据并行（OpenMP/OpenACC 统一）

### `parallel` - 数据并行

```cpp
import parallel;

Vector<int32_t> data(1000000);

// CPU 数据并行
parallel(data.size(), [&](size_t i) {
    data[i] = compute(i);
}, Execution::Cpu);

// 具体调度由运行时库执行；Execution::Cpu 只是策略选择
```

### `parallel` - GPU 后端

```cpp
import parallel;

Vector<float> data(1000000);

// GPU 数据并行
parallel(data.size(), [&](size_t i) {
    data[i] = data[i] * 2.0f + 1.0f;
}, Execution::Gpu);

// 后端代码生成与运行时设备支持待确认
```

### 统一的 API 设计

```cpp
// 一维并行
parallel(n, [&](size_t i) { /* ... */ }, Execution::Cpu);
parallel(n, [&](size_t i) { /* ... */ }, Execution::Gpu);

// 二维并行
parallel(rows, cols, [&](size_t i, size_t j) { /* ... */ }, Execution::Cpu);
parallel(rows, cols, [&](size_t i, size_t j) { /* ... */ }, Execution::Gpu);

// 三维并行
parallel(x, y, z, [&](size_t i, size_t j, size_t k) { /* ... */ }, Execution::Cpu);
parallel(x, y, z, [&](size_t i, size_t j, size_t k) { /* ... */ }, Execution::Gpu);
```

### 并行归约

```cpp
// CPU 归约
int32_t sum = reduce(data, 0, [](int32_t a, int32_t b) {
    return a + b;
}, Execution::Cpu);

// GPU 归约
float total = reduce(data, 0.0f, [](float a, float b) {
    return a + b;
}, Execution::Gpu);
```

### 调度策略（CPU）

```cpp
// 静态调度：编译期分配固定范围
parallel(n, [&](size_t i) { /* ... */ }, Execution::Cpu, Schedule::Static);

// 动态调度：运行时动态分配任务
parallel(n, [&](size_t i) { /* ... */ }, Execution::Cpu, Schedule::Dynamic(chunk_size=100));

// 工作窃取：负载均衡
parallel(n, [&](size_t i) { /* ... */ }, Execution::Cpu, Schedule::WorkStealing);
```

### GPU 内存管理

```cpp
import gpu;

// 主机内存
Vector<float> host(1000000);

// 分配 GPU 内存
GpuBuffer<float> device = gpu_alloc(host.size());

// 主机 → GPU
device.copy_from(host);

// GPU 计算
parallel(device.size(), [device](size_t i) {
    device[i] = device[i] * 2.0f;
}, Execution::Gpu);

// GPU → 主机
device.copy_to(host);
```

### 统一内存（自动迁移）

```cpp
// 统一内存：CPU/GPU 自动迁移
UnifiedBuffer<float> data(1000000);

// CPU 初始化
for (size_t i = 0; i < data.size(); ++i) {
    data[i] = i;
}

// GPU 计算（自动迁移到 GPU）
parallel(data.size(), [data](size_t i) {
    data[i] = data[i] * 2.0f;
}, Execution::Gpu);

// CPU 读取（自动迁移回 CPU）
println("Result: {}", data[0]);
```

### 矩阵乘法示例

```cpp
void matmul_cpu(Matrix& c, const Matrix& a, const Matrix& b) {
    parallel(c.rows(), c.cols(), [&](size_t i, size_t j) {
        float sum = 0.0f;
        for (size_t k = 0; k < a.cols(); ++k) {
            sum += a(i, k) * b(k, j);
        }
        c(i, j) = sum;
    }, Execution::Cpu);
}

void matmul_gpu(Matrix& c, const Matrix& a, const Matrix& b) {
    parallel(c.rows(), c.cols(), [&](size_t i, size_t j) {
        float sum = 0.0f;
        for (size_t k = 0; k < a.cols(); ++k) {
            sum += a(i, k) * b(k, j);
        }
        c(i, j) = sum;
    }, Execution::Gpu);
}
```

## 二、任务并发（Go 风格）

### 启动轻量级任务

```cpp
import thread;

// Thread::spawn 创建轻量级任务（类似 Go goroutine）
auto task = Thread::spawn([]{
    println("Running in background");
    compute_work();
});

task.join();

// 带返回值
auto task = Thread::spawn([]{
    return compute_result();
});

int32_t result = task.join();
```

### 轻量级任务特性

- **栈初始 2KB**，按需增长（类似 Go）
- **M:N 调度**：M 个任务映射到 N 个 OS 线程
- **工作窃取**：空闲线程从其他线程窃取任务
- **百万级并发**：可以轻松创建数十万个任务

### 通道通信

```cpp
import channel;

// 无缓冲通道（同步）
Channel<int32_t> ch;

Thread::spawn([&]{
    ch.send(42);  // 阻塞直到接收
});

int32_t value = ch.recv();  // 阻塞直到发送

// 有缓冲通道（异步）
Channel<int32_t> ch(100);  // 缓冲区大小 100

ch.send(1);  // 不阻塞（缓冲区未满）
ch.send(2);

int32_t v1 = ch.recv();  // 1
int32_t v2 = ch.recv();  // 2
```

### 通道迭代

```cpp
Channel<int32_t> ch(10);

Thread::spawn([&]{
    for (int i = 0; i < 10; ++i) {
        ch.send(i);
    }
    ch.close();  // 关闭通道
});

// 迭代直到通道关闭
for (auto value : ch) {
    println("{}", value);
}
```

### Select 多路复用

```cpp
Channel<int32_t> ch1;
Channel<String> ch2;

// Channel::select 是库函数，非关键字
auto result = Channel::select(
    ch1.recv_case([](int32_t value) {
        println("Received int: {}", value);
        return 1;
    }),
    ch2.recv_case([](String msg) {
        println("Received string: {}", msg);
        return 2;
    }),
    ch1.send_case(42, []{
        println("Sent to ch1");
        return 3;
    }),
    Channel::default_case([]{
        println("No channel ready");
        return 0;
    })
);
```

### 生产者-消费者模式

```cpp
void producer_consumer() {
    Channel<Work> queue(100);
    
    // 4 个生产者
    for (size_t i = 0; i < 4; ++i) {
        Thread::spawn([&, i]{
            for (size_t j = 0; j < 100; ++j) {
                queue.send(Work{.id = i * 100 + j});
            }
        });
    }
    
    // 8 个消费者
    for (size_t i = 0; i < 8; ++i) {
        Thread::spawn([&]{
            for (auto work : queue) {
                process(work);
            }
        });
    }
    
    // 等待生产者完成
    Thread::sleep(1s);
    queue.close();
}
```

### 工作池模式

```cpp
struct WorkerPool {
    Channel<Task> tasks;
    Vector<Thread::Handle> workers;
    
    WorkerPool(size_t num_workers) : tasks(1000) {
        for (size_t i = 0; i < num_workers; ++i) {
            workers.push(Thread::spawn([this]{
                for (auto task : tasks) {
                    task.execute();
                }
            }));
        }
    }
    
    void submit(Task task) {
        tasks.send(task);
    }
    
    ~WorkerPool() {
        tasks.close();
        for (auto& worker : workers) {
            worker.join();
        }
    }
};
```

## 三、同步原语

### Mutex<T> - 互斥锁

```cpp
import mutex;

struct Counter {
    Mutex<int32_t> value;
    
    void increment() {
        auto guard = value.lock();  // RAII 加锁
        *guard += 1;
        // guard 析构时自动解锁
    }
    
    int32_t get() {
        auto guard = value.lock();
        return *guard;
    }
};
```

### Atomic - 原子操作

```cpp
import atomic;

Atomic<int32_t> counter(0);

// 原子操作（无锁）
counter.fetch_add(1, MemoryOrder::Relaxed);
counter.fetch_sub(1, MemoryOrder::AcqRel);

int32_t old = counter.exchange(42, MemoryOrder::SeqCst);

// Compare-and-swap
int32_t expected = 10;
bool success = counter.compare_exchange_strong(expected, 20);
```

### RwLock - 读写锁

```cpp
struct Cache {
    RwLock<Map<String, String>> data;
    
    Optional<String> get(const String& key) {
        auto guard = data.read_lock();  // 共享读锁
        return guard->get(key);
    }
    
    void set(const String& key, const String& value) {
        auto guard = data.write_lock();  // 独占写锁
        guard->insert(key, value);
    }
};
```

## 四、实现原理

### parallel 的实现边界（待确认）

```cpp
// parallel 是普通运行时库函数；comp 只可用于静态策略选择或生成专用内核。
// 线程数、分块和 join 都依赖运行时硬件与输入，不能放进 comp 求值。
void parallel(size_t n, auto func, Execution execution,
              Schedule schedule = Schedule::Auto);
```

### GPU 后端（待确认）

```cpp
// GPU 后端可以由编译器在 comp 上下文中选择，也可以由运行时库调度。
// 具体 CUDA/OpenCL/Metal 接口尚未定稿，这里不规定新的语言语法。
```

### Thread::spawn 运行时

```cpp
// 运行时系统（库实现，非语言特性）
struct Runtime {
    Vector<OSThread> os_threads;        // OS 线程池
    LockFreeQueue<Task*> global_queue;  // 全局任务队列
    Vector<LockFreeQueue<Task*>> local_queues;  // 每线程本地队列
    
    void worker(size_t tid) {
        while (running) {
            // 1. 尝试从本地队列取任务
            Optional<Task*> task = local_queues[tid].pop();
            
            if (!task) {
                // 2. 尝试从全局队列取任务
                task = global_queue.pop();
            }
            
            if (!task) {
                // 3. 工作窃取：从其他线程偷任务
                for (size_t i = 0; i < local_queues.size(); ++i) {
                    if (i != tid) {
                        task = local_queues[i].steal();
                        if (task) break;
                    }
                }
            }
            
            if (task) {
                task.value()->execute();
            } else {
                // 无任务，休眠
                Thread::sleep(1ms);
            }
        }
    }
};
```

## 五、使用场景

### 场景 1：数据并行（图像处理）

```cpp
void apply_filter(Image& img) {
    // CPU 版本
    parallel(img.height(), img.width(), [&](size_t y, size_t x) {
        img(y, x) = blur(img, y, x);
    });
    
    // GPU 版本（API 完全相同）
    parallel(img.height(), img.width(), [&](size_t y, size_t x) {
        img(y, x) = blur(img, y, x);
    });
}
```

### 场景 2：任务并发（Web 服务器）

```cpp
void http_server() {
    TcpListener listener("127.0.0.1:8080");
    
    loop {
        TcpStream conn = listener.accept();
        
        // 每个连接启动一个轻量级任务
        Thread::spawn([conn = move(conn)]{
            handle_connection(conn);
        });
    }
}
```

### 场景 3：Pipeline（流水线）

```cpp
void pipeline() {
    Channel<RawData> stage1(100);
    Channel<ProcessedData> stage2(100);
    Channel<Result> stage3(100);
    
    // Stage 1: 读取数据
    Thread::spawn([&]{
        for (auto data : read_input()) {
            stage1.send(data);
        }
        stage1.close();
    });
    
    // Stage 2: 处理数据（多个 worker）
    for (size_t i = 0; i < 4; ++i) {
        Thread::spawn([&]{
            for (auto data : stage1) {
                stage2.send(process(data));
            }
        });
    }
    
    // Stage 3: 保存结果
    Thread::spawn([&]{
        for (auto data : stage2) {
            stage3.send(save(data));
        }
        stage3.close();
    });
    
    // 等待完成
    for (auto result : stage3) {
        println("Result: {}", result);
    }
}
```

### 场景 4：混合使用

```cpp
void hybrid_computation() {
    Vector<Image> images = load_images();
    
    // 1. CPU 并行预处理
    parallel(images.size(), [&](size_t i) {
        images[i] = preprocess(images[i]);
    });
    
    // 2. GPU 并行计算
    parallel(images.size(), [&](size_t i) {
        images[i] = compute_intensive(images[i]);
    });
    
    // 3. 任务并发上传结果
    Channel<Result> results(100);
    
    for (auto& img : images) {
        Thread::spawn([&, img]{
            results.send(upload(img));
        });
    }
    
    // 收集结果
    for (size_t i = 0; i < images.size(); ++i) {
        auto result = results.recv();
        println("Uploaded: {}", result);
    }
}
```

## 六、性能优化

### 任务粒度

```cpp
// ✗ 差：任务太细粒度
for (size_t i = 0; i < 1000000; ++i) {
    Thread::spawn([i]{ compute(i); });  // 百万个任务，开销大
}

// ✓ 好：使用数据并行
parallel(1000000, [](size_t i) {
    compute(i);  // 自动分块
});
```

### CPU vs GPU 选择

```cpp
// 数据量小：CPU
if (data.size() < 10000) {
    parallel(data.size(), [&](size_t i) { process(data[i]); });
}
// 数据量大：GPU
else {
    parallel(data.size(), [&](size_t i) { process(data[i]); });
}
```

### 内存访问模式

```cpp
// ✓ 好：连续访问（CPU 缓存友好）
parallel(n, [&](size_t i) {
    result[i] = data[i] * 2;  // 连续内存访问
});

// ✗ 差：随机访问（缓存不友好）
parallel(n, [&](size_t i) {
    result[i] = data[random_index[i]];  // 随机跳跃
});
```

## 七、最佳实践

### 1. 数据并行用 cpu/parallel

```cpp
// ✓ 好
parallel(data.size(), [&](size_t i) { /* ... */ });

// ✗ 差：手动创建线程
for (size_t i = 0; i < num_threads; ++i) {
    Thread::spawn(...);
}
```

### 2. 任务并发用 Thread::spawn + Channel

```cpp
// ✓ 好：通道通信
Channel<Result> ch;
Thread::spawn([&]{ ch.send(compute()); });
Result r = ch.recv();

// ✗ 差：共享内存
Mutex<Result> result;
Thread::spawn([&]{ /* 写 result */ });
```

### 3. 优先通道，避免锁

```cpp
// ✓ 好：无锁通信
Channel<int32_t> ch;

// ✗ 差：需要锁
Mutex<int32_t> counter;
```

## 八、对比总结

| 场景 | 解决方案 | API |
|------|---------|-----|
| CPU 数据并行 | OpenMP 风格 | `parallel(..., Execution::Cpu)` |
| GPU 数据并行 | OpenACC 风格 | `parallel(..., Execution::Gpu)` |
| 任务并发 | Go 风格 | `Thread::spawn` + `Channel` |
| 同步原语 | 标准 C++ | `Mutex`、`Atomic`、`RwLock` |

**核心优势**：
- ✅ 统一的 CPU/GPU API（`parallel` + `Execution` 策略）
- ✅ 轻量级任务系统（Go 风格）
- ✅ 无新增关键字（全部库 + comp 函数）
- ✅ 零成本抽象（只为选定策略生成所需代码）

## 下一步

- 查看 [09-compiler.md](09-compiler.md) 了解 `comp` 函数的编译器实现
- 查看 [10-packages.md](10-packages.md) 了解并发库的依赖管理
- 查看 [12-examples.md](12-examples.md) 查看完整的并发示例

