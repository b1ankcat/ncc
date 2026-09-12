# 八、并发与多线程

NCC 的并发模型基于两个核心范式：
1. **数据并行**：统一的 `parallel`/`reduce` API，通过 `Device` 参数选择 CPU 或 GPU
2. **任务并发**：Go 风格的轻量级任务（`Thread::spawn` + `Channel`）

无需新增关键字，全部通过 `comp` 函数和库实现。

## 核心理念

- **统一的并行 API**：`parallel` 和 `reduce` 接受 `Device` 参数，CPU/GPU 是实现后端
- **轻量级任务**：Go 风格的 goroutine（Thread::spawn，2KB 栈，M:N 调度）
- **通道通信**：Go 风格的 Channel，优先消息传递而非共享内存
- **零成本抽象**：只为实际选择的执行策略生成所需代码
- **无新增关键字**：所有特性通过库和 `comp` 函数实现

## 一、数据并行

### CPU 数据并行

```cpp
import parallel;

Vector<int32_t> data(1000000);

// CPU 数据并行
parallel<Device::Cpu>(data.size(), [&](size_t i) {
    data[i] = compute(i);
});

// parallel 是 comp 函数，签名：
// comp void parallel<Device device>(size_t n, auto func, Schedule schedule = Schedule::Auto);
```

### 统一的 API 设计

```cpp
// 一维并行
parallel<Device::Cpu>(n, [&](size_t i) { /* ... */ });

// 二维并行
parallel<Device::Cpu>(rows, cols, [&](size_t i, size_t j) { /* ... */ });

// 三维并行
parallel<Device::Cpu>(x, y, z, [&](size_t i, size_t j, size_t k) { /* ... */ });
```

### 并行归约

```cpp
// CPU 归约
int32_t sum = reduce<Device::Cpu>(data, 0, [](int32_t a, int32_t b) {
    return a + b;
});

// 求最大值
float max_val = reduce<Device::Cpu>(data, -INFINITY, [](float a, float b) {
    return a > b ? a : b;
});
```

### 调度策略（CPU）

```cpp
// 静态调度：编译期分配固定范围
parallel<Device::Cpu>(n, [&](size_t i) { /* ... */ }, Schedule::Static);

// 动态调度：运行时动态分配任务
parallel<Device::Cpu>(n, [&](size_t i) { /* ... */ }, Schedule::Dynamic(chunk_size=100));

// 工作窃取：负载均衡
parallel<Device::Cpu>(n, [&](size_t i) { /* ... */ }, Schedule::WorkStealing);
```

### GPU 数据并行

GPU 相关的并行计算请参考 [09-gpu.md](09-gpu.md)。

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

## 四、使用场景

### 场景 1：数据并行（图像处理）

```cpp
void apply_filter(Image& img) {
    parallel<Device::Cpu>(img.height(), img.width(), [&](size_t y, size_t x) {
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

### 场景 4：混合使用（CPU 并行 + 任务并发）

```cpp
void hybrid_computation() {
    Vector<Image> images = load_images();
    
    // 1. CPU 并行预处理
    parallel<Device::Cpu>(images.size(), [&](size_t i) {
        images[i] = preprocess(images[i]);
    });
    
    // 2. 任务并发上传结果
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

## 五、性能优化

### 任务粒度

```cpp
// ✗ 差：任务太细粒度
for (size_t i = 0; i < 1000000; ++i) {
    Thread::spawn([i]{ compute(i); });  // 百万个任务，开销大
}

// ✓ 好：使用数据并行
parallel<Device::Cpu>(1000000, [](size_t i) {
    compute(i);  // 自动分块
});
```

### 内存访问模式

```cpp
// ✓ 好：连续访问（CPU 缓存友好）
parallel<Device::Cpu>(n, [&](size_t i) {
    result[i] = data[i] * 2;  // 连续内存访问
});

// ✗ 差：随机访问（缓存不友好）
parallel<Device::Cpu>(n, [&](size_t i) {
    result[i] = data[random_index[i]];  // 随机跳跃
});
```

## 六、最佳实践

### 1. 数据并行用 parallel

```cpp
// ✓ 好
parallel<Device::Cpu>(data.size(), [&](size_t i) { /* ... */ });

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

## 七、对比总结

| 场景 | 解决方案 | API |
|------|---------|-----|
| CPU 数据并行 | OpenMP 风格 | `parallel<Device::Cpu>` |
| GPU 数据并行 | 参见 GPU 文档 | `parallel<Device::Gpu>` |
| 任务并发 | Go 风格 | `Thread::spawn` + `Channel` |
| 同步原语 | 标准 C++ | `Mutex`、`Atomic`、`RwLock` |

**核心优势**：
- ✅ 统一的 CPU/GPU API（`parallel` + `Device` 参数）
- ✅ 轻量级任务系统（Go 风格）
- ✅ 无新增关键字（全部库 + comp 函数）
- ✅ 零成本抽象（只为选定策略生成所需代码）

## 下一步

- 查看 [09-gpu.md](09-gpu.md) 了解 GPU 并行与异构计算
- 查看 [10-compiler.md](10-compiler.md) 了解编译器架构
- 查看 [12-examples.md](12-examples.md) 查看完整的并发示例
