# 八、并发与多线程

NCC 的并发模型只有一个主旨：**结构化并发**。每个任务属于一个明确的
`TaskScope`，作用域结束前任务必须完成、取消或被显式转移。

数据并行（`parallel`/`reduce`）、轻量任务和 `Channel` 都是这个作用域中的库 API。

无需新增关键字，全部通过普通库函数实现；`comp` 仅用于编译期配置和代码生成。

## 核心理念

- **统一的并行 API**：`parallel` 和 `reduce` 接受 `Device` 参数，CPU/GPU 是实现后端
- **唯一的执行单元入口**：`TaskScope::spawn`，不提供 `Thread::spawn`
- **作用域任务**：轻量任务使用 M:N 调度和工作窃取，但不默认脱离作用域
- **所有权决定通道关闭**：`Sender` 计数归零即关闭，不靠人工协调 join 顺序
- **协程只做惰性序列**：保留 `co_yield`/`co_return`，删除 `co_await`，
  并发与惰性序列各占一块，不重叠
- **异步句柄不可静默丢弃**：`Completion` 析构即等待，丢弃退化为同步而非未定义行为
- **零成本抽象**：只为实际选择的执行策略生成所需代码
- **无新增关键字**：所有特性通过库和 `comp` 函数实现

## 一、数据并行

### CPU 数据并行

```cpp
import parallel;

Vector<int32_t> data(1000000);

// 返回 [[nodiscard]] 的 Completion；接收句柄可与其他工作重叠
auto done = parallel<Device::Cpu>(data.size(), [&](size_t i) {
    data[i] = compute(i);
});
done.wait();

// 丢弃返回值也是正确的：句柄析构时阻塞等待，等价于同步执行
parallel<Device::Cpu>(data.size(), [&](size_t i) {
    data[i] = compute(i);
});

// parallel 是运行时库函数，不标记 comp
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

// 动态调度：运行时动态分配任务（参数为 chunk 大小）
parallel<Device::Cpu>(n, [&](size_t i) { /* ... */ }, Schedule::Dynamic(100));

// 工作窃取：负载均衡
parallel<Device::Cpu>(n, [&](size_t i) { /* ... */ }, Schedule::WorkStealing);
```

### GPU 数据并行

GPU 相关的并行计算请参考 [09-gpu.md](09-gpu.md)。

## 二、结构化任务并发

### TaskScope 契约

```cpp
TaskScope scope;

Task<int32_t> task = scope.spawn([] {
    return compute_result();
});

int32_t result = task.join();
```

`TaskScope` 拥有在其中创建的任务。`spawn` 返回 `Task<T>`，其中 `T` 是可调用对象的
返回类型；`Task<T>` 是 move-only 的句柄，不拥有任务本身。捕获局部引用仍遵循 C++
规则，结构化作用域不增加借用检查。

### Task 生命周期

```cpp
comp class Task<type T> {
public:
    T join();                  // 等待完成并返回结果；任务的异常在此重抛
    void detach();             // 放弃句柄，任务仍归 scope 所有
    void request_cancel();     // 请求取消，不等待
    bool is_ready() const;

    ~Task();                   // 等价于 detach()
};
```

| 操作 | 行为 |
| --- | --- |
| `join()` | 阻塞至完成；正常返回结果，任务抛出的异常在此重抛 |
| `detach()` | 放弃句柄；任务仍由 `scope` 拥有并等待 |
| 句柄析构 | 等价于 `detach()`，不阻塞 |
| `scope` 析构 | 对未完成任务先 `request_cancel()`，再等待全部结束 |

**析构等同 detach 而非等待**，因为 `TaskScope` 提供了兜底：任务的生命周期上限始终
是作用域，句柄提前销毁不会产生游离任务。这与 `Completion` 的"析构即等待"不同——
后者没有任何兜底者，见[异步句柄对照](#异步句柄对照)。

**未被 join 的任务如果抛出异常**，异常保存在任务中；`scope` 析构时重抛第一个未被
观察的异常。多个任务抛出时抛出聚合异常，保留全部：

```cpp
class AggregateException : public RuntimeError {
public:
    Vector<Exception> nested() const;
};
```

`scope` 析构期间重抛异常遵循 C++ 规则：若析构本身发生在异常传播中，程序终止。
需要精确控制错误处理的场合应显式 `join()` 每个任务，而不是依赖作用域汇总。

### 与调度器交互

```cpp
Thread::sleep(Duration::from_millis(10));   // 让出 worker，不占用 OS 线程
Thread::yield_now();                         // 主动让出，交还调度器
```

`Thread` 只提供当前任务与调度器对话的操作，**不提供 `Thread::spawn`**：创建执行
单元只有 `TaskScope::spawn` 一个入口。`Thread::sleep` 挂起当前任务并释放 worker
给其他任务，不是 OS 级 `nanosleep`。

需要真正的 OS 线程时（必须绑核的实时线程、要求特定栈大小的第三方回调），显式经由
[`import pthread`](12-interop.md#使用-import-批量导入) 调用 C API。这样脱离调度器
是一个显眼的动作，且明确不适用 Channel 关闭协议、取消传播和 `blocking()` 记账。

### 启动轻量级任务

```cpp
import thread;

TaskScope scope;

// 无返回值：Task<void>
Task<void> background = scope.spawn([]{
    println("Running in background");
    compute_work();
});
background.join();

// 带返回值：返回类型由可调用对象推导
Task<int32_t> computation = scope.spawn([]{
    return compute_result();
});
int32_t result = computation.join();
```

### 轻量级任务特性

- **栈初始 2KB**，按需增长
- **M:N 调度**：M 个任务映射到 N 个 OS 线程
- **工作窃取**：空闲线程从其他线程窃取任务
- **百万级并发**：可以轻松创建数十万个任务

### 通道通信

通道由 `create()` 产生一对句柄：`Sender<T>` 负责发送，`Receiver<T>` 负责接收。
两者都是 move-only；`Sender` 额外支持 `clone()`。

```cpp
import channel;

// 无缓冲通道（同步）
auto [tx, rx] = Channel<int32_t>::create();

TaskScope scope;
auto producer = scope.spawn([tx = move(tx)]{
    tx.send(42);        // 阻塞直到接收
});                     // 任务结束，tx 析构，通道关闭

Optional<int32_t> value = rx.recv();   // 阻塞直到发送或通道关闭
producer.join();

// 有缓冲通道（异步）
auto [tx2, rx2] = Channel<int32_t>::create(100);   // 缓冲区大小 100

tx2.send(1);            // 不阻塞（缓冲区未满）
tx2.send(2);

if (auto v = rx2.recv()) {
    println("{}", *v);  // 1
}
```

**接收 API：**

```cpp
Optional<T> recv();       // 阻塞直到有值；通道关闭且排空后返回空值
Optional<T> try_recv();   // 立即返回；无可用值时为空
bool is_closed() const;   // 是否已关闭（缓冲区可能仍有残留值）
```

`try_recv()` 的空值不区分"暂时没有"与"已关闭且排空"，需要区分时再查
`is_closed()`。两次调用之间状态可能改变，因此该组合不适用于要求精确判定的场合；
那些场合应使用阻塞 `recv()` 或范围 `for`，它们由通道内部同步保证判定准确。

**发送 API：**

```cpp
void send(T value);       // 阻塞直到写入；向已关闭通道发送抛出 ChannelClosed
Sender<T> clone() const;  // 增加发送端计数
void close();             // 放弃当前 Sender，等价于提前析构
```

`send` 抛异常而 `recv` 返回空值，这个不对称是有意的：接收端读到通道结束是正常的
流程终止，而向已关闭的通道发送意味着发送端对通道状态的判断已经出错。

### 通道关闭协议

**通道在最后一个 `Sender<T>` 析构时自动关闭。** 关闭时机由所有权决定，不需要
人工协调 join 顺序：

```cpp
auto [tx, rx] = Channel<int32_t>::create(10);

TaskScope scope;
auto producer = scope.spawn([tx = move(tx)]{
    for (int i = 0; i < 10; ++i) {
        tx.send(i);
    }
});                        // tx 析构，这是最后一个 Sender，通道关闭

// 迭代直到通道关闭且排空
for (auto value : rx) {
    println("{}", value);
}
producer.join();
```

多个生产者各持有一份 `clone()`，全部结束后通道自动关闭：

```cpp
auto [tx, rx] = Channel<Work>::create(100);

TaskScope scope;
for (size_t i = 0; i < 4; ++i) {
    scope.spawn([tx = tx.clone(), i]{
        for (size_t j = 0; j < 100; ++j) {
            tx.send(Work{.id = i * 100 + j});
        }
    });
}
tx.close();      // 主线程放弃自己那份；此后计数只由 4 个任务持有

for (auto work : rx) {   // 4 个任务全部结束后自然终止
    process(work);
}
```

主线程那句 `tx.close()` 不可省略：只要它还持有一份 `Sender`，计数就不会归零，
`for` 会永久阻塞。显式 `close()` 的语义就是"提前放弃这一份"，与析构等价。

范围 `for` 遍历 `Receiver` 时逐个取值直到通道关闭且排空，等价于
`while (auto v = rx.recv())`。

`Receiver<T>` 同样支持 `clone()`，用于多消费者竞争消费：每个值只会交给其中一个
消费者，不会广播。接收端计数不影响关闭时机——只有 `Sender` 计数决定通道何时关闭；
所有 `Receiver` 都析构后，`send` 抛出 `ChannelClosed`（无人可接收）。

### Select 多路复用

```cpp
auto [tx1, rx1] = Channel<int32_t>::create();
auto [tx2, rx2] = Channel<String>::create();

// Channel::select 是库函数，非关键字
auto result = Channel::select(
    rx1.recv_case([](Optional<int32_t> value) {
        if (!value) { return -1; }              // 该通道已关闭且排空
        println("Received int: {}", *value);
        return 1;
    }),
    rx2.recv_case([](Optional<String> msg) {
        if (!msg) { return -1; }
        println("Received string: {}", *msg);
        return 2;
    }),
    tx1.send_case(42, []{
        println("Sent to ch1");
        return 3;
    }),
    Channel::default_case([]{
        println("No channel ready");
        return 0;
    })
);
```

**select 语义：**

- **返回类型**：所有 case 处理函数的返回类型必须一致（含引用限定），全部为 `void`
  也合法。类型不一致在编译期报错。
- **阻塞行为**：不含 `default_case` 时阻塞，直到某个 case 就绪；含 `default_case`
  时非阻塞，无 case 就绪立即执行 default。
- **关闭的通道**：`recv_case` 的处理函数接收 `Optional<T>`，通道关闭且排空后以空值
  调用一次——关闭本身就是一种"就绪"，否则 select 会永久忽略该通道。`send_case`
  所在通道关闭时，该 case 抛出 `ChannelClosed`。
- **多个就绪**：随机选择一个执行，不保证顺序，避免固定优先级导致饥饿。
- **只执行一个**：每次 `select` 调用恰好执行一个 case 处理函数。

### 生产者-消费者模式

```cpp
void producer_consumer() {
    TaskScope scope;
    auto [tx, rx] = Channel<Work>::create(100);

    // 4 个生产者，各持一份 Sender
    for (size_t i = 0; i < 4; ++i) {
        scope.spawn([tx = tx.clone(), i]{
            for (size_t j = 0; j < 100; ++j) {
                tx.send(Work{.id = i * 100 + j});
            }
        });
    }
    tx.close();      // 放弃主线程那份，否则通道永不关闭

    // 8 个消费者竞争消费，各持一份 Receiver
    for (size_t i = 0; i < 8; ++i) {
        scope.spawn([rx = rx.clone()]{
            for (auto work : rx) {
                process(work);
            }
        });
    }
    rx.close();
}
```

不需要按顺序 join：生产者全部结束后 `Sender` 计数归零，通道关闭；消费者的
范围 `for` 随之结束；`scope` 析构时等待全部任务完成。与之前"先 join 生产者、
再 close、再 join 消费者"的写法相比，正确性不再依赖人工安排的顺序。

### 工作池模式

`Job` 是用户定义的工作项类型，不是核心库类型；`Task<T>` 才是任务句柄。

```cpp
struct Job {
    void execute();
};

struct WorkerPool {
    TaskScope scope;
    Sender<Job> submit_tx;

    explicit WorkerPool(size_t num_workers) {
        auto [tx, rx] = Channel<Job>::create(1000);
        submit_tx = move(tx);

        for (size_t i = 0; i < num_workers; ++i) {
            scope.spawn([rx = rx.clone()]{
                for (auto job : rx) {     // 多 worker 竞争消费
                    job.execute();
                }
            });
        }
        // rx 在构造函数结束时析构，接收端只剩各 worker 持有的克隆
    }

    // 池持有 scope 与发送端，不可拷贝或移动
    WorkerPool(const WorkerPool&) = delete;
    WorkerPool& operator=(const WorkerPool&) = delete;
    WorkerPool(WorkerPool&&) = delete;
    WorkerPool& operator=(WorkerPool&&) = delete;

    void submit(Job job) {
        submit_tx.send(move(job));
    }

    ~WorkerPool() {
        submit_tx.close();   // 通道关闭 → worker 的 for 结束
        // scope 析构等待全部 worker 完成
    }
};
```

析构顺序是关键：成员按声明逆序销毁，`submit_tx` 先于 `scope` 析构，因此通道先关闭、
worker 自然退出，`scope` 随后等待。显式 `close()` 使这个依赖清晰可见，不依赖成员
声明顺序的巧合。worker lambda 不捕获 `this`，池对象的地址不参与任务执行。

## 三、惰性序列：生成器协程

绿色线程解决并发，协程解决**惰性序列**——两者不重叠，因此并存不违反"一个问题
一个解法"。为保证不重叠，NCC 保留 `co_yield` / `co_return`，删除 `co_await` 与
awaiter 协议（见 [概览](00-overview.md)）：挂起点只有 `co_yield` 和 `co_return`，
语言层面无法用协程搭建第二套异步机制。

### 编写生成器

```cpp
import generator;

// 惰性产生斐波那契数列，不预先计算也不分配容器
Generator<uint64_t> fibonacci() {
    uint64_t a = 0;
    uint64_t b = 1;
    for (;;) {
        co_yield a;
        auto next = a + b;
        a = b;
        b = next;
    }
}

for (auto n : fibonacci()) {
    if (n > 1000) break;
    println("{}", n);
}
```

递归结构的遍历是协程最有价值的场景——用推式回调或手写迭代器都要显式维护栈：

```cpp
Generator<const Node&> in_order(const Node& node) {
    if (node.left) {
        for (const auto& n : in_order(*node.left)) { co_yield n; }
    }
    co_yield node;
    if (node.right) {
        for (const auto& n : in_order(*node.right)) { co_yield n; }
    }
}
```

### 生成器的规则

- **`Generator<T>` 不是核心库特权类型**：用户可定义自己的生成器类型，只需提供
  含 `yield_value` 的 promise。由于没有 `co_await`，能定义出的东西只有惰性序列。
- **move-only**，不可拷贝；帧的生命周期就是生成器对象的生命周期。
- **在哪个任务恢复取决于在哪调用 `next()`**，语言不作限制；生成器本身不引入并发。
- **帧默认堆分配**，生成器为局部变量且不逃逸时允许消除（与 C++ 的 HALO 同理，
  不作承诺）。这是 opt-in 的开销：不写协程的程序不付任何代价。
- **生成器体内可以做 I/O**：绿色线程照常让出，帧不受影响。两个机制在此正交。
- **不是设备可传输类型**：`parallel<Device::Gpu>` 的 lambda 不能捕获生成器，
  kernel 内不能定义协程。

### 组合：管道适配器

`|` 适配器作用于任何满足序列接口的对象（生成器、容器、视图），在编译期融合：

```cpp
auto records = lines(file)
    | filter([](const String& l) { return !l.empty(); })
    | map([](const String& l) { return parse(l); })
    | take(10);

for (const auto& r : records) { println("{}", r); }
```

**分工**：生成器协程负责**编写**序列源（控制流复杂时尤其如此），`|` 适配器负责
**组合**。适配器链在编译期展开为单个循环；对容器和视图为零开销，对生成器保留
一次帧恢复的代价。核心库不提供第二套推式接口——那会让"如何编写序列"出现两个答案。

### 与 Channel 的分界

| 需求 | 机制 |
| --- | --- |
| 单生产者、消费者按需拉取、无并发 | `Generator<T>` |
| 跨任务传递、多生产者、需要缓冲 | `Channel<T>` |

生成器是同一个任务内的控制流转移，无同步开销；Channel 涉及跨任务同步。惰性计算
序列用生成器，任务间通信用 Channel。

## 四、阻塞与调度

### 核心库 I/O 自动让出

核心库的 `File`、`TcpStream` 等 I/O 类型实现在异步后端上（Linux 用 io_uring 或
epoll，Windows 用 IOCP，macOS 用 kqueue）。用户写普通的阻塞风格代码，调用点让出
worker 给其他任务：

```cpp
scope.spawn([conn = move(conn)]{
    String request = conn.read_line();   // 让出 worker，不占用 OS 线程
    conn.write(handle(request));         // 同上
});
```

这是"绿色线程 + 同步写法"模型的地基：看起来阻塞的调用实际是让出点。

### 阻塞的 C 调用

第三方 C 库直接进系统调用，调度器无从感知——`sqlite3_step`、`curl_easy_perform`
会占住 worker 直到返回。规范形式是用 `blocking()` 把调用移到专用阻塞线程池：

```cpp
// 规范形式：显式包裹
auto row = blocking([&]{
    return sqlite3_step(stmt);
});
```

`blocking()` 是普通库函数，不需要编译器支持：它把可调用对象派发到阻塞线程池，
挂起当前任务直到完成。

**内置绑定已预先标注**：`import libc` / `import posix` / `import pthread` 生成的
声明中，已知会阻塞的函数由编译器标记，调用时自动包装，用户无需手写 `blocking()`。
自己声明的 `extern "C"` 函数可加 `[[ncc::blocking]]` 获得同样待遇：

```cpp
extern "C" {
    [[ncc::blocking]] int32_t sqlite3_step(sqlite3_stmt* stmt);
}
```

`[[ncc::blocking]]` 只是便利标注，规范形式仍是 `blocking()` 函数。它符合
[attribute 可忽略判据](00-overview.md)：删掉它程序仍然正确，只是并发度下降。

**兜底检测**：监控线程发现某个 worker 超过阈值未让出时，补充一个 worker 顶上。
这把漏标的损害限制在"短暂降低并发度"，而不是死锁。这是启发式机制，不是正确性
保证——阻塞调用仍应显式标注。

### 调度语义

- **协作式调度，非抢占**。让出点是：核心库 I/O、`Channel` 操作、`Thread::sleep`、
  `Thread::yield_now`、`blocking()`、锁等待。
- **纯计算的长任务不会自动让出**，可能饿死同一 worker 上的其他任务。计算密集的
  循环应周期性调用 `Thread::yield_now()`，或改用 `parallel` 分块。
- **工作窃取**：空闲 worker 从其他 worker 的队列尾部窃取任务。
- **worker 数量**默认等于硬件线程数，可由调度器配置覆盖。
- **阻塞线程池独立于 worker 池**，按需增长，用于 `blocking()` 派发。

选择协作式调度是因为抢占需要信号或安全点插桩：前者与 C 互操作冲突（信号可能
打断第三方 C 库的系统调用），后者在每个循环回边插入检查，与"零成本抽象"相悖。
代价是上面第二条——长计算任务需要程序员配合。

## 五、同步原语

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
    RwLock<HashMap<String, String>> data;
    
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

### Semaphore - 并发数限制

```cpp
Semaphore limit(100);       // 最多 100 个许可

void handle(TcpStream conn) {
    auto permit = limit.acquire();   // 无许可时让出 worker 等待
    handle_connection(conn);
    // permit 析构时归还许可
}
```

`acquire()` 返回 RAII 许可对象，析构时归还；`try_acquire()` 返回
`Optional<Permit>`，无许可时立即返回空。用于给无界的任务创建加背压——
`TaskScope` 负责生命周期，不负责限流。

## 六、使用场景

### 场景 1：数据并行（图像处理）

```cpp
void apply_filter(Image& img) {
    auto done = parallel<Device::Cpu>(img.height(), img.width(), [&](size_t y, size_t x) {
        img(y, x) = blur(img, y, x);
    });
    done.wait();
}
```

### 场景 2：任务并发（Web 服务器）

```cpp
void http_server() {
    TaskScope scope;
    TcpListener listener("127.0.0.1:8080");

    for (;;) {
        TcpStream conn = listener.accept();   // 让出 worker 直到有连接

        // 每个连接启动一个轻量级任务；句柄立即析构，等同 detach
        scope.spawn([conn = move(conn)]{
            handle_connection(conn);
        });
    }
}
```

`scope` 只保留**未完成**任务的记录，已完成的任务在结束时从作用域中移除，因此
长期运行的 accept 循环不会让作用域无界增长。需要限制并发连接数时用信号量或
有界通道，而不是依赖作用域——作用域负责生命周期，不负责背压。

### 场景 3：Pipeline（流水线）

```cpp
void pipeline() {
    TaskScope scope;
    auto [raw_tx, raw_rx] = Channel<RawData>::create(100);
    auto [proc_tx, proc_rx] = Channel<ProcessedData>::create(100);
    auto [result_tx, result_rx] = Channel<Result>::create(100);

    // Stage 1: 读取数据
    scope.spawn([tx = move(raw_tx)]{
        for (auto data : read_input()) {
            tx.send(data);
        }
    });                                  // tx 析构 → raw 通道关闭

    // Stage 2: 处理数据（多个 worker 竞争消费）
    for (size_t i = 0; i < 4; ++i) {
        scope.spawn([rx = raw_rx.clone(), tx = proc_tx.clone()]{
            for (auto data : rx) {
                tx.send(process(data));
            }
        });                              // 最后一个 worker 结束 → proc 通道关闭
    }
    raw_rx.close();
    proc_tx.close();

    // Stage 3: 保存结果
    scope.spawn([rx = move(proc_rx), tx = move(result_tx)]{
        for (auto data : rx) {
            tx.send(save(data));
        }
    });                                  // tx 析构 → result 通道关闭

    // 主线程消费最终结果，不需要安排各级的关闭顺序
    for (auto result : result_rx) {
        println("Result: {}", result);
    }
}
```

每一级的关闭由该级发送端的所有权自然触发：主线程只需放弃自己不再使用的句柄
（`raw_rx.close()`、`proc_tx.close()`），然后消费结果即可。

> 对比：如果改用单一 `Channel` 对象加显式 `close()`，正确性就取决于人工安排的
> join 与 close 顺序。典型写法是"join 生产者 → close stage1 → join workers →
> close stage2 → 消费 stage3"，但这会死锁：主线程要 join 完 workers 才去消费
> `stage3`，而 stage3 缓冲填满后 saver 阻塞在 `send`，于是 `proc` 通道不再被消费、
> workers 阻塞在自己的 `send` 上，join 永不返回。所有权驱动的关闭从机制上排除了
> 这类顺序错误。

### 场景 4：混合使用（CPU 并行 + 任务并发）

```cpp
void hybrid_computation() {
    TaskScope scope;
    Vector<Image> images = load_images();
    
    // 1. CPU 并行预处理
    auto preprocess_done = parallel<Device::Cpu>(images.size(), [&](size_t i) {
        images[i] = preprocess(images[i]);
    });
    preprocess_done.wait();
    
    // 2. 任务并发上传结果
    auto [tx, rx] = Channel<Result>::create(100);

    for (auto& img : images) {
        scope.spawn([tx = tx.clone(), img]{
            tx.send(upload(img));
        });
    }
    tx.close();      // 放弃主线程那份，上传任务全部结束后通道关闭

    // 收集结果：不需要计数，通道关闭即遍历结束
    for (auto result : rx) {
        println("Uploaded: {}", result);
    }
}
```

## 七、性能优化

### 任务粒度

```cpp
// ✗ 差：任务太细粒度
TaskScope scope;
for (size_t i = 0; i < 1000000; ++i) {
    scope.spawn([i]{ compute(i); });  // 百万个任务，调度开销盖过计算
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

## 八、最佳实践

### 1. 数据并行用 parallel

```cpp
// ✓ 好：交给 parallel 分块
parallel<Device::Cpu>(data.size(), [&](size_t i) { /* ... */ });

// ✗ 差：手工分块 + 逐个 spawn，重复实现调度器已有的工作
TaskScope scope;
size_t chunk = data.size() / num_workers;
for (size_t w = 0; w < num_workers; ++w) {
    scope.spawn([&, w]{
        for (size_t i = w * chunk; i < (w + 1) * chunk; ++i) { /* ... */ }
    });
}
// 尾部余数要另行处理；负载不均时也无法窃取
```

### 2. 任务并发用 TaskScope + Channel

```cpp
// ✓ 好：结果随消息转移，所有权清晰
auto [tx, rx] = Channel<Result>::create();
TaskScope scope;
scope.spawn([tx = move(tx)]{ tx.send(compute()); });

if (auto r = rx.recv()) {
    use(*r);
}

// ✓ 也好：单个结果直接用 Task<T> 的返回值，不必动用通道
TaskScope scope2;
Task<Result> task = scope2.spawn([]{ return compute(); });
Result r2 = task.join();

// ✗ 差：用锁包一个只写一次的结果，还要额外协调"何时写完了"
Mutex<Optional<Result>> shared;
scope2.spawn([&]{ *shared.lock() = compute(); });
```

单个结果用 `Task<T>::join()` 最直接；需要流式传递多个结果时才用通道。

### 3. 按场景选择通信方式

三种机制各有适用场景，不存在"通道总是更好"：

| 场景 | 推荐 | 理由 |
| --- | --- | --- |
| 传递工作项、结果、事件流 | `Channel<T>` | 所有权随消息转移，无需推理临界区 |
| 单个计数器、标志位 | `Atomic<T>` | 无锁，开销远低于通道 |
| 需要在多个字段间维持不变量 | `Mutex<T>` / `RwLock<T>` | 临界区能覆盖整组修改 |

```cpp
// ✓ 传递数据流：通道
auto [work_tx, work_rx] = Channel<Work>::create(100);

// ✓ 纯计数：原子变量，用通道反而更慢
Atomic<int64_t> processed(0);
processed.fetch_add(1, MemoryOrder::Relaxed);

// ✓ 多字段不变量：锁比通道更直接
struct Stats {
    Mutex<StatsData> data;   // total 与 buckets 必须一致更新

    void record(int32_t value) {
        auto guard = data.lock();
        guard->total += value;
        guard->buckets[bucket_of(value)] += 1;
    }
};
```

真正要避免的是**用锁保护本该由所有权转移解决的问题**：若一份数据在任意时刻只
应由一个任务持有，用通道移交比用锁共享更难出错。

## 异步句柄对照

三种句柄代表不同的东西，析构行为也不同：

| 类型 | 代表 | 终结操作 | 析构行为 | 归属 |
| --- | --- | --- | --- | --- |
| `Task<T>` | 绿色线程任务 | `join()` / `detach()` | 等同 `detach()`，不阻塞 | `TaskScope` |
| `Completion` | 并行或设备操作 | `wait()` | **阻塞等待完成** | 无（绑定操作本身） |
| `Completion<T>` | 带结果的异步操作 | `get()` | **阻塞等待完成** | 无 |

```cpp
class [[nodiscard]] Completion {
public:
    void wait();                  // 阻塞至完成；操作失败时重抛异常
    bool is_ready() const;
    ~Completion();                // 未 wait 则阻塞等待
};

comp class [[nodiscard]] Completion<type T> {
public:
    T get();                      // 阻塞并取值，只能调用一次
    bool is_ready() const;
    ~Completion();
};
```

**为什么析构行为不同**：`Task<T>` 有 `TaskScope` 兜底，句柄销毁后任务的生命周期
上限仍是作用域，不会游离；`Completion` 没有任何兜底者，析构不等待就意味着操作
可能仍在读写已释放的内存。因此前者析构即 detach，后者析构即等待。

两者都标注 `[[nodiscard]]`。丢弃 `Completion` 会得到警告，但语义仍然正确——
**退化为同步执行**，只是失去并行收益：

```cpp
// 丢弃返回值：等价于同步执行这个并行循环，正确但无重叠
parallel<Device::Cpu>(n, [&](size_t i) { data[i] = compute(i); });

// 需要重叠时接收句柄，显式 wait
auto done = parallel<Device::Cpu>(n, [&](size_t i) { data[i] = compute(i); });
do_other_work();
done.wait();
```

## 九、按场景选择 API

| 场景 | API |
|------|-----|
| CPU / GPU 数据并行 | `parallel<Device::Cpu>` / `parallel<Device::Gpu>` |
| 任务并发 | `TaskScope::spawn` → `Task<T>` |
| 任务间通信 | `Sender<T>` / `Receiver<T>` |
| 惰性序列 | `Generator<T>` + `co_yield` |
| 阻塞的 C 调用 | `blocking()` / `[[ncc::blocking]]` |
| 计数与标志 | `Atomic<T>` |
| 多字段不变量 | `Mutex<T>` / `RwLock<T>` |
| 限制并发数 | `Semaphore` |

