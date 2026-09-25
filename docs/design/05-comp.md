# 五、`comp` 系统

`comp` 是本设计唯一新增的关键字，统一取代 `constexpr` / `consteval`，并作为
反射与代码生成的入口。**`template` 关键字被彻底删除**，所有泛型机制统一为
`comp` 函数 + 反射生成。

## 核心原则

**统一机制**：底层只有一个泛型机制 —— `comp` 函数接受 `type` 参数。
- `comp type` 函数用于生成类型
- `comp class` 是纯语法糖，编译器自动脱糖为 `comp type` 函数 + `define_class` 调用
- 所有泛型都通过编译期调用 `comp` 函数实现，无例外

## `comp` 修饰函数/类型

```cpp
// 编译期函数：取代 constexpr / consteval
comp int32_t square(int32_t x) {
    return x * x;
}

// 编译期类型：类型的构造/使用被限定在编译期
comp struct Config {
    int32_t version;
};
```

## 泛型定义：`comp` 函数 + `type` 参数

`template` 关键字删除，泛型定义改用 `comp` 函数接受 `type` 参数，通过 `<>`
语法调用时触发编译期代码生成：

函数声明采用 `comp 返回类型 函数名<泛型参数>(普通参数)`，其中普通参数使用
C++ 的 `类型 参数名` 写法；没有独立的 `fn` 关键字。类型参数列表属于总纲
已批准的 comp 扩展。求值与实例化的阶段划分见[求值阶段与实例化](#求值阶段与实例化)。

```cpp
// 泛型类型定义（仅演示布局生成，省略全部生命周期操作）
comp type Vector(type T) {
    // 用反射 + splice 生成具体的 Vector_T 类型
    return [: define_aggregate("Vector", {
        {^^T*, "data"},
        {^^size_t, "size"},
        {^^size_t, "capacity"}
    }) :];
}

// 使用：<> 触发编译期调用 Vector(^^int32_t)
Vector<int32_t> v;  // 等价于编译期调用 Vector(^^int32_t)

// 泛型函数定义
comp T max<type T>(T a, T b) {
    return a > b ? a : b;
}

// 使用
max<int32_t>(1, 2);  // 触发编译期调用 max(^^int32_t)
```

## `<>` 与 `()` 调用的统一

**核心规则：`X<Args>` 是 `X(^^Args...)` 的语法糖**

两种写法**完全等价**，可以自由混用：

```cpp
// 类型构造
comp type Vector(type T) { /* ... */ }

// 三种等价写法
Vector<int32_t> v1;              // 语法糖形式
Vector(^^int32_t) v2;            // 直接调用形式
using IntVec = Vector(^^int32_t);
IntVec v3;                       // 类型别名

// 类型参数可以作为值传递
comp type make_container(type T) {
    return Vector(T);  // T 已经是 TypeInfo，直接使用
}

// 类型关系查询
comp bool derives_from(type Derived, type Base) { /* 编译期类型检查 */ }

// 两种等价写法
bool b1 = derives_from<File, Writer>();
bool b2 = derives_from(^^File, ^^Writer);

// 智能指针
comp type unique_ptr(type T) { /* 生成 unique_ptr<T> 类型 */ }
comp auto make_unique(type T) { /* 生成该类型专用的工厂函数 */ }

unique_ptr<File> p1;                        // 语法糖
unique_ptr(^^File) p2;                      // 直接调用
unique_ptr<File> q = make_unique<File>("out.txt");
```

**`<>` vs `()` 对比**：

| 语法 | 含义 | 参数处理 | 使用场景 |
|------|------|----------|----------|
| `X<A, B>` | 编译期调用 `X(^^A, ^^B)` 的语法糖 | 自动加 `^^` 提升为 `TypeInfo` | 类型名位置、类似 C++ 的用法 |
| `X(^^A, ^^B)` | 编译期调用 `X` 函数 | 显式传递 `TypeInfo` | 类型参数需要传递、存储或计算时 |

**何时使用哪种写法**：

```cpp
// 推荐用 <> 的场景：类型名位置
Vector<int32_t> v;
Optional<String> opt;
unique_ptr<File> p;

// 推荐用 () 的场景：类型作为值传递
comp type choose_container(type T, bool use_vector) {
    if (use_vector) {
        return Vector(T);  // T 已经是 TypeInfo
    } else {
        return List(T);
    }
}

// 类型参数存储
comp type T = Vector(^^int32_t);  // 将类型存储为变量
T v;  // 使用存储的类型

// 类型参数数组
comp Vector<TypeInfo> types = {^^int32_t, ^^String, ^^double};
for (auto T : types) {
    println("Size of {}: {}", name_of(T), size_of(T));
}
```

**适用范围**：
- `Vector<T>`、`Optional<T>`、`unique_ptr<T>` - 类型生成
- `make_unique<T>(...)`、`make_shared<T>(...)` - 函数生成，再运行时调用
- `is<T>(value)`、`cast<T>(value)` - 类型查询/转换
- 所有接受 `type` 参数的 comp 函数

## 非类型参数与默认值

`comp` 函数除了接受类型参数（`type T`），还可以接受编译期常量值参数（对应 C++26 的 non-type template parameters）。
参数可以有默认值，编译器会自动识别编译期常量。

### 基本用法

```cpp
// 固定大小数组
comp type Array(type T, size_t N) {
    return [: define_aggregate("Array", {
        {make_array_type(^^T, N), "data"}
    }) :];
}

Array<int32_t, 10> arr1;         // 语法糖
Array(^^int32_t, 10) arr2;       // 直接调用

// 设备标记（用于 GPU 内存管理）
enum class Device : int32_t {
    Cpu = 0,
    Gpu = 1,
};

comp type Vector(type T, Device device = Device::Cpu) {
    if (device == Device::Cpu) {
        return [: define_aggregate("Vector_Cpu", {
            {^^T*, "data"},
            {^^size_t, "size"},
            {^^size_t, "capacity"}
        }) :];
    } else {
        return [: define_aggregate("Vector_Gpu", {
            {^^void*, "device_ptr"},
            {^^size_t, "size"}
        }) :];
    }
}

// 使用默认值
Vector<float> cpu_vec;                    // 使用默认 Device::Cpu
Vector<float, Device::Gpu> gpu_vec;       // 显式指定 Device::Gpu

// 直接调用形式也支持默认值
Vector(^^float) cpu_vec2;                 // 使用默认值
Vector(^^float, Device::Gpu) gpu_vec2;    // 显式指定
```

### 自动识别编译期常量

编译器会自动识别编译期可求值的表达式，无需手动标记：

```cpp
comp type Array(type T, size_t N) { /* ... */ }

// 字面量 - 自动识别
Array<int32_t, 10> arr1;

// 常量 - 自动识别
const size_t SIZE = 10;
Array<int32_t, SIZE> arr2;

// 编译期计算 - 自动识别
constexpr size_t compute_size() { return 5 * 2; }
Array<int32_t, compute_size()> arr3;

// comp 常量 - 自动识别
comp size_t BUFFER_SIZE = 1024;
Array<int32_t, BUFFER_SIZE> arr4;

// 运行时变量 - 编译错误
void func(size_t n) {
    Array<int32_t, n> arr;  // 错误：n 不是编译期常量
}
```

**调用语法规则：**

`X<TypeArg, ValueArg>` 编译期调用 `X(^^TypeArg, ValueArg)`

- **类型参数自动加 `^^`**：转换为反射 `TypeInfo` 对象
- **非类型参数原样传递**：保持为编译期常量值
- **默认值自动填充**：省略的参数使用函数声明中的默认值

**支持的非类型参数类型：**
- 整数类型（`int32_t`, `size_t` 等）
- 枚举类型（`enum class Device`）
- 布尔类型（`bool`）
- 浮点类型（`float`, `double`）- 编译期常量
- 指针类型（编译期可求值的指针）

## `comp class` - 泛型类定义语法糖

**重要**：`comp class` 是纯语法糖，编译器会自动将其脱糖（desugar）为 `comp type` 函数 + `define_class` 调用。
底层只有一个机制：`comp type` 函数。

### 语法糖形式

`comp class` 提供了类似 C++ 的类定义语法，适合需要多个成员函数、构造/析构函数的复杂泛型类：

```cpp
comp class CustomPtr<type T> {
    T* ptr_;

public:
    // 构造函数
    explicit CustomPtr(T* p = nullptr) noexcept : ptr_(p) {}
    
    // 析构函数
    ~CustomPtr() noexcept {
        if (ptr_) {
            delete ptr_;
        }
    }
    
    // 禁用拷贝
    CustomPtr(const CustomPtr&) = delete;
    CustomPtr& operator=(const CustomPtr&) = delete;
    
    // 移动构造
    CustomPtr(CustomPtr&& other) noexcept : ptr_(other.ptr_) {
        other.ptr_ = nullptr;
    }

    CustomPtr& operator=(CustomPtr&& other) noexcept {
        if (this != &other) {
            delete ptr_;
            ptr_ = other.ptr_;
            other.ptr_ = nullptr;
        }
        return *this;
    }
    
    // 成员函数
    T& operator*() { return *ptr_; }
    T* operator->() { return ptr_; }
    T* get() { return ptr_; }
};

// 使用
CustomPtr<User> p(new User{1, "Alice"});
println("{}", p->name);
```

### 自动脱糖机制

编译器会将上述 `comp class` 定义自动转换为等价的 `comp type` 函数：

```cpp
// 编译器生成的等价代码（用户不需要写）
comp type CustomPtr(type T) {
    return [: define_class("CustomPtr", {
        .data_members = {
            {^^T*, "ptr_", Access::Private}
        },
        .constructors = {
            {
                .params = {{^^T*, "p", ^^nullptr}},
                .initializers = {{"ptr_", ^^p}},
                .noexcept = true
            }
        },
        .destructor = {
            .body = ^() {
                if (ptr_) {
                    delete ptr_;
                }
            },
            .noexcept = true
        },
        .copy_constructor = Deleted,
        .copy_assignment = Deleted,
        .move_constructor = {
            .initializers = {{"ptr_", ^^other.ptr_}},
            .body = ^(CustomPtr&& other) {
                other.ptr_ = nullptr;
            },
            .noexcept = true
        },
        .move_assignment = {
            .body = ^(CustomPtr&& other) {
                if (this != &other) {
                    delete ptr_;
                    ptr_ = other.ptr_;
                    other.ptr_ = nullptr;
                }
                return *this;
            },
            .noexcept = true
        },
        .methods = {
            {"operator*", ^^T&, {}, ^() { return *ptr_; }},
            {"operator->", ^^T*, {}, ^() { return ptr_; }},
            {"get", ^^T*, {}, ^() { return ptr_; }}
        }
    }) :];
}
```

**关键点**：
- 用户可以选择写 `comp class`（简洁）或直接写 `comp type` 函数（灵活）
- 两种形式功能完全等价，只是表达方式不同
- `comp class` 适合类似 C++ 的传统类定义
- `comp type` 函数适合需要条件生成、循环生成成员等高级场景

### 何时使用哪种形式

**推荐使用 `comp class` 的场景**：
- 复杂类定义，有多个构造函数、析构函数、运算符重载
- 需要资源管理（RAII）
- 类似 C++ 的传统 OOP 风格
- 团队熟悉 C++ 语法

**推荐使用 `comp type` 函数的场景**：
- 简单聚合类型（纯数据结构）
- 需要条件生成成员（根据类型参数决定）
- 需要循环生成成员（批量生成字段/方法）
- 需要组合其他类型生成器
- 更函数式的元编程风格

**示例对比**：

```cpp
// 简单聚合：用 comp type 函数更简洁
comp type Point(type T) {
    return [: define_aggregate("Point", {
        {^^T, "x"},
        {^^T, "y"}
    }) :];
}

// 条件生成：comp type 函数更灵活
comp type Buffer(type T, bool thread_safe) {
    if (thread_safe) {
        return [: define_class("Buffer", {
            .data_members = {
                {^^Vector<T>, "data"},
                {^^Mutex, "mutex"}  // 额外的互斥锁
            },
            .methods = {/* 加锁的方法 */}
        }) :];
    } else {
        return [: define_class("Buffer", {
            .data_members = {{^^Vector<T>, "data"}},
            .methods = {/* 无锁的方法 */}
        }) :];
    }
}

// 复杂类：comp class 更可读
comp class LockGuard<type T> {
    T& mutex_;
public:
    explicit LockGuard(T& m) : mutex_(m) { mutex_.lock(); }
    ~LockGuard() { mutex_.unlock(); }
    LockGuard(const LockGuard&) = delete;
    LockGuard& operator=(const LockGuard&) = delete;
};
```

此示例仅管理单个对象，要求 T 的析构不抛异常。裸指针成员不会让编译器自动
识别独占资源；删除拷贝、提供移动和清空源指针都由类作者按 C++ 规则实现。

### 内存池复用通用原语

`MemoryPool<T>` 与 Vector 使用同一份核心库 comp 生成器
`StorageOps<T, Device::Cpu>`。生成器选择类型操作，运行时池对象才分配和构造，
不再用 `T data` 的假对象加指针重解释来拼接空闲链表。

实际存储由块记录组成：每块持有原始存储、独立的空闲槽位索引和活动位图。
原始槽位没有活动 T，只有构造成功后才设置活动位。池拥有全部块以及尚未归还的
活动对象；本设计不支持拷贝或移动池，避免转移时误共享资源。

1. 创建池时拒绝零 chunk_size，并检查容量乘法溢出。只有 `allocate()` 的默认
   构造路径要求 T 可默认构造；带参数的 `emplace(args...)` 不要求默认构造。
2. 新块通过 `Ops::allocate_raw(chunk_size)` 分配并满足 `alignof(T)`；
   块记录、位图或空闲索引建立失败时，RAII 临时记录释放尚未提交的全部资源。
3. 取得空闲槽位后，调用 `Ops::construct_at`。构造成功才标记活动并移出空闲表；
   构造抛异常则槽位仍为空闲，异常向调用方传播。
4. 归还时定位所属块和槽位，对活动对象调用 `Ops::destroy_at`，清除活动位并
   放回空闲表。块归属查询由池显式维护，不用已被删除的指针转换关键字。
5. 池析构时按活动位图销毁所有未归还对象，再逐块调用 `Ops::release_raw`；
   已归还槽位不能再次析构。要求 T 的析构不抛异常。
6. 归还参数必须是该池当前活动对象的地址，不能传入内部子对象、其他池对象或
   重复归还。此为库 API 前置条件，不引入全局裸指针所有权推断。

`allocate`、`emplace` 和 `deallocate` 是生成类型的普通运行时成员函数。
这里给出块与槽位算法，具体索引容器可用现有 Vector 实现；其元素寿命与异常
保证必须遵循 [通用容器原语](03-memory.md#通用容器原语)。

### 与标准类定义的对比

| 特性 | 标准类定义 | comp class（语法糖） | comp type 函数 |
|------|-----------|---------------------|----------------|
| 用途 | 具体类型实现 | 泛型类定义（类 C++ 风格） | 泛型类定义（函数式风格） |
| 类型参数 | 无 | 通过 `<type T>` 声明 | 函数参数 `type T` |
| 实例化 | 直接使用 | `CustomPtr<User>` 触发生成 | `CustomPtr(^^User)` 触发生成 |
| 成员函数 | 直接定义 | 直接定义（可使用类型参数） | 通过 `define_class` 描述 |
| 条件生成 | 不支持 | 不支持（静态结构） | 支持（可用 if/for） |
| 适用场景 | 非泛型类 | 需要类型参数的复杂类 | 需要动态生成结构的类 |
| 底层机制 | 直接编译 | 脱糖为 comp type 函数 | 直接调用反射 API |

`comp class` 和 `comp type` 函数底层完全等价，都会生成普通类定义。
两种写法都没有不同的生命周期或优化语义；需要资源管理时都必须明确定义
构造、析构、拷贝和移动操作。

### 核心容器类型

`Vector<T, Device>`, `String`, `HashMap<K, V>` 等核心容器是预定义的库实现，
使用与 `comp class` 相同的公开生成机制和 `StorageOps`：

- 支持 `Device` 参数（CPU/GPU 内存）
- 使用公开的类型属性触发 SIMD、GPU 和内存优化
- 与反射系统深度集成

用户自定义的 `comp class` 类型与核心容器使用相同的生成 API；编译器不得按
类型名称授予特殊语义，只能依据公开的类型属性和调用契约优化。

## 变参泛型

```cpp
// 类型参数包（... 是 C++26 已有语法）
comp void log<type... Args>(Args... args) {
    // 用包展开逐个处理：参数类型各不相同，不能收进同一个初始化列表
    (println("{}", args), ...);
}

log<int32_t, String, double>(42, "hello", 3.14);

// 两种调用形式等价
log(^^int32_t, ^^String, ^^double)(42, "hello", 3.14);
```

参数包展开使用 C++17 已有的折叠表达式。`{args...}` 只适用于所有参数类型相同
的情况；异质参数包必须用折叠表达式或递归展开。

## 类型参数推导

编译器可以从函数调用的实参类型推导类型参数，减少显式类型标注：

```cpp
// 泛型函数定义
comp T max<type T>(T a, T b) {
    return a > b ? a : b;
}

// 显式指定类型参数
auto m1 = max<int32_t>(1, 2);

// 从实参推导类型参数
auto m2 = max(1, 2);        // 推导为 max<int32_t>
auto m3 = max(1.5, 2.5);    // 推导为 max<double>

// 容器构造
comp auto make_vector<type T>(T... values) {
    Vector<T> v;
    (v.push_back(values), ...);
    return v;
}

// 从实参推导
auto v1 = make_vector(1, 2, 3);           // Vector<int32_t>
auto v2 = make_vector("a", "b", "c");     // Vector<const char*>

// 部分推导：指定部分参数，推导其余参数
comp auto create_pair<type T1, type T2>(T1 a, T2 b) {
    return Pair(^^T1, ^^T2){a, b};
}

auto p1 = create_pair(42, "hello");           // 推导两个参数
auto p2 = create_pair<int32_t>(42, "hello");  // 指定第一个，推导第二个
```

**推导规则**：
- 类型参数可以从对应位置的实参类型推导
- 推导按照 C++ 的模板实参推导规则（去除引用、const 等）
- 多个实参对应同一类型参数时，必须推导为相同类型，否则报错
- 显式指定的类型参数不参与推导
- 只有函数的类型参数可以推导，类型生成函数（如 `Vector`）不支持推导

**限制**：
```cpp
// 类型生成函数不支持推导
Vector v1 = {1, 2, 3};  // 错误：无法推导 T

// 必须显式指定
Vector<int32_t> v2 = {1, 2, 3};  // 正确
```

## 高阶 comp 函数

`comp` 函数可以返回 `comp` 函数，支持高阶元编程：

```cpp
// 返回类型生成器的函数
comp auto make_optional_factory(type T) {
    return [=](type Inner) {
        return Optional(Inner);
    };
}

// 使用
comp auto opt_factory = make_optional_factory(^^int32_t);
opt_factory(^^String) v1;  // Optional<String>

// 返回函数生成器
comp auto make_logger<type T>() {
    return [](const T& value) {
        println("Value: {}", value);
    };
}

// 使用
auto log_int = make_logger<int32_t>();
log_int(42);  // 输出: Value: 42

// 组合类型生成器
comp type VectorOf(type T) {
    return Vector(T);
}

comp type ListOf(type T) {
    return List(T);
}

comp auto choose_container(bool use_vector) {
    if (use_vector) {
        return VectorOf;
    } else {
        return ListOf;
    }
}

// 使用
comp auto container_factory = choose_container(true);
container_factory(^^int32_t) data;  // Vector<int32_t>
```

**应用场景**：
- 构建可复用的类型生成器组合
- 实现类型级别的策略模式
- 创建领域特定的类型 DSL

## 裸 `comp { ... }` 块

保留顶层裸块，用于批量的编译期操作，编译器在翻译单元处理到此处时立即执行。

**`comp` 块与 `comp` 函数的统一规则**：

裸 `comp {}` 块在语义上等价于一个立即调用的匿名 `comp` 函数：

```cpp
// 用户写的
comp {
    int32_t x = 10 + 20;
    println("Computed at compile time: {}", x);
}

// 等价于
comp void _anonymous_comp_block_1() {
    int32_t x = 10 + 20;
    println("Computed at compile time: {}", x);
}
_anonymous_comp_block_1();  // 编译期立即调用
```

**关键点**：
- `comp {}` 块和 `comp` 函数遵循**完全相同**的求值规则
- 都在编译期执行
- 都可以调用反射 API、生成类型、注册信息
- 都有相同的副作用行为（编译期输出、编译期断言等）
- 都有相同的限制（不能访问运行时才能确定的值）

**使用场景**：

```cpp
// 批量处理当前模块的类型
comp {
    for (auto T : types_of(^^current_module)) {
        if (satisfies(T, ^^Serializable)) {
            generate_serializer(T);
        }
    }
}

// 注册测试函数
comp {
    for (auto func : functions_of(^^current_module)) {
        if (name_of(func).starts_with("test_")) {
            test_registry_add(func);
        }
    }
}

// 编译期配置初始化
comp {
    if (profile() == "debug") {
        enable_logging();
        enable_assertions();
    }
}
```


## `comp` 条件编译

在 `comp` 函数内部，`if` 语句自动具有编译期分支剪枝能力（对应 C++17 `if constexpr`）：

```cpp
comp void process() {
    // profile() 是核心库提供的编译期查询，返回当前构建配置
    if (profile() == "debug") {
        println("Debug mode enabled");
        enable_logging();
    } else if (profile() == "release") {
        println("Release mode");
        enable_optimizations();
    }
}
```

条件编译使用的目标与配置查询统一写成函数形式：`profile()`、`target_os()`、
`target_arch()`、`target_triple()`。这些是编译期常量函数，分支在实例化时剪枝。

## 批量处理：遍历当前模块内的类型

```cpp
comp {
    // 获取所有函数，注册以 test_ 开头的
    for (auto func : functions_of(^^current_module)) {
        if (name_of(func).starts_with("test_")) {
            test_registry_add(func);
        }
    }

    // 为实现了 Serializable 的类型生成序列化代码
    for (auto T : types_of(^^current_module)) {
        if (satisfies(T, ^^Serializable)) {
            generate_serializer(T);
        }
    }
}
```

## `^^` 运算符的使用规则

**核心规则**：`^^` 可以在任何上下文中使用，不限于 `comp` 上下文。

### 在任何位置获取类型信息

```cpp
// 在 comp 函数中
comp void process(type T) {
    println("Type: {}", name_of(T));
}

// 在普通函数中传递类型信息
void print_type_name(TypeInfo t) {
    println("Type name: {}", name_of(t));
}

void example() {
    print_type_name(^^int32_t);  // 允许！^^T 在任何位置都有效
    print_type_name(^^String);
    print_type_name(^^Vector<int32_t>);
}

// 在类成员中存储类型信息
struct TypeHolder {
    TypeInfo type;
};

TypeHolder holder = {^^int32_t};

// 在容器中存储类型信息
Vector<TypeInfo> types = {^^int32_t, ^^String, ^^double};

void print_sizes() {
    for (auto T : types) {
        println("Size of {}: {}", name_of(T), size_of(T));
    }
}
```

### 表达式反射的限制

表达式反射 `^^(expr)` 仍然只能在 `comp` 上下文中使用：

```cpp
// ✓ 允许：在 comp 上下文中
comp void analyze() {
    int32_t x = 10, y = 20;
    ExprInfo expr = ^^(x + y);
    println("Expression: {}", stringify(expr));
}

// ✗ 不允许：在普通函数中
void func() {
    int32_t x = 10, y = 20;
    ExprInfo expr = ^^(x + y);  // 错误：表达式反射只能在 comp 中使用
}
```

### Splice 的限制

Splice `[: ... :]` 仍然只能在 `comp` 上下文中使用：

```cpp
// ✓ 允许：在 comp 中使用 splice
comp type Point(type T) {
    return [: define_aggregate("Point", {
        {^^T, "x"},
        {^^T, "y"}
    }) :];
}

// ✗ 不允许：在普通函数中使用 splice
void func(TypeInfo T) {
    auto point = [: define_aggregate("Point", {...}) :];  // 错误：splice 只能在 comp 中
}
```

### 运行时类型查询

运行时可以通过 `dynamic_type_of` 获取对象的动态类型：

```cpp
class Shape {
public:
    virtual ~Shape() = default;
};

class Circle : public Shape {
    double radius;
};

void inspect(const Shape& s) {
    TypeInfo T = dynamic_type_of(s);  // 运行时获取动态类型
    
    println("Dynamic type: {}", name_of(T));
    
    // 使用反射 API 查询类型信息
    for (auto field : fields_of(T)) {
        println("  Field: {}", name_of(field));
    }
}
```

### 总结

| 操作 | comp 上下文 | 普通上下文 | 说明 |
|------|-----------|----------|------|
| `^^Type` | ✓ | ✓ | 类型反射随处可用 |
| `^^(expr)` | ✓ | ✗ | 表达式反射仅限 comp |
| `[: ... :]` | ✓ | ✗ | Splice 仅限 comp |
| `TypeInfo` 作为值 | ✓ | ✓ | 可自由传递和存储 |
| `dynamic_type_of` | ✓ | ✓ | 运行时类型查询随处可用 |
| 反射查询 API | ✓ | ✓ | `name_of`, `size_of` 等随处可用 |

## 代码生成：合成聚合类型

用反射库的 `define_aggregate` 从字段描述生成聚合类型，用 splice 拼回成员/
方法体，不再拼接字符串源码：

```cpp
comp {
    using Generated = [: define_aggregate(
        ^^GeneratedBase,
        {data_member_spec(^^int32_t, {.name = "a"}),
         data_member_spec(^^int32_t, {.name = "b"})}) :];
}
```

## 不需要宏系统

```cpp
// 传统语言需要 derive 宏
// @derive(Debug, Serialize)
// struct User { ... };

// 这里的做法：comp 块里直接调用反射生成函数
comp {
    enable_debug(^^User);
    enable_serialize(^^User);
}

// 或者批量处理：裸 comp 块遍历当前模块内所有类型
comp {
    for (auto T : types_of(^^current_module)) {
        if (has_annotation(T, ^^auto_interfaces)) {
            enable_debug(T);
            enable_serialize(T);
        }
    }
}

## 求值阶段与实例化

### 两阶段名称查找（Two-phase lookup）

comp 泛型函数中的名称分为两类，在不同阶段解析：

| 名称类型 | 解析时机 | 使用的依赖闭包 |
| --- | --- | --- |
| **非依赖名称** | 定义时 | 定义处模块的依赖闭包 |
| **依赖名称** | 实例化时 | 实例化点模块的依赖闭包 |

**判断标准**：
- 参数、局部变量的类型涉及类型参数 → 依赖
- 调用的函数实参类型涉及类型参数 → 依赖（触发实参关联查找）
- 直接使用类型参数本身 → 依赖
- 其他 → 非依赖

```cpp
// utils.ncc
export module utils;

comp int32_t helper() { return 42; }

export comp int32_t compute<type T>(T value) {
    auto x = helper();        // 非依赖：定义时在 utils 的依赖闭包中解析
    auto y = process(value);  // 依赖：实例化时在调用点的依赖闭包中解析
    return x + y;
}

// main.ncc
import utils;
import geometry;  // 导出 Vec2 和 process(Vec2)

comp int32_t result = utils::compute(Vec2{1, 2});
// process(value) 在 main.ncc 实例化时解析，找到 geometry::process
```

这与 [模块系统的名称查找规则](01-modules.md#名称查找) 一致：实参关联查找发生在
实例化点，使用该点的传递依赖闭包。

### 实例化触发时机

泛型类型和函数采用**延迟实例化**：只在需要完整定义时才触发。

**触发实例化的情况**：

```cpp
comp class Container<type T> { T value; };
comp void func<type T>() { /* ... */ }

// ✓ 触发实例化（需要完整定义）
Container<int32_t> c;              // 定义对象
sizeof(Container<int32_t>);        // 查询大小
func<int32_t>();                   // 调用泛型函数
Container<int32_t>& r = c;         // 引用绑定需要完整类型

// ✗ 不触发实例化（只需声明）
Container<int32_t>* p;             // 指针声明
Container<int32_t>& get_ref();     // 函数声明返回引用
```

成员函数的实例化独立于类实例化：

```cpp
comp class Widget<type T> {
    T value;
    void process() { /* ... */ }
};

Widget<int32_t> w;   // 实例化 Widget<int32_t> 类
w.process();         // 此时才实例化 process()
```

### 递归深度限制

编译期递归调用受深度限制保护，防止无限递归导致编译器崩溃或资源耗尽。

**默认限制**：512 层递归

```cpp
comp int32_t factorial(int32_t n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}

comp int32_t x = factorial(5);      // ✓ 5 层递归
comp int32_t y = factorial(1000);   // ✗ 编译错误：超过递归深度限制
```

超过限制时编译器报错，错误消息包含调用栈以便调试：

```
error: comp recursion depth exceeded (limit: 512)
  in instantiation of function 'factorial' at depth 512
  call stack:
    factorial(1000) -> factorial(999) -> ... -> factorial(488)
```

**调整限制**：编译器标志 `-fcomp-recursion-limit=N` 可调整限制（如 1024、2048），
但过大的值可能导致编译器内存耗尽。

### 求值上下文与时机

**规则：标记为 `comp` 的函数总是在编译期求值**，无论调用上下文。

```cpp
comp int32_t compute() {
    return 42;
}

comp int32_t x = compute();   // 编译期求值
int32_t y = compute();        // 也在编译期求值，结果作为常量内联
```

**副作用**：comp 函数体内的副作用（如调用 `println`）在编译期发生：

```cpp
comp int32_t log_and_return(int32_t x) {
    println("Computing: {}", x);  // 编译期输出
    return x * 2;
}

comp int32_t a = log_and_return(5);  // 编译时输出 "Computing: 5"
```

**限制**：comp 函数不能执行编译期无法完成的操作：
- 读写文件（除非在 `build.ncc` 中，见 [构建系统](11-build-system.md)）
- 网络访问
- 调用运行时才能确定的外部函数

违反限制时编译器报错。

### 顺序保证

`comp` 块内的语句**按顺序求值**，与普通代码块一致：

```cpp
comp {
    auto a = expr1();  // 第 1 步
    auto b = expr2();  // 第 2 步（可能依赖 a）
    auto c = expr3();  // 第 3 步（可能依赖 a 和 b）
}
```

后续语句可以依赖前面语句的结果。这保证了代码生成和类型定义的确定性顺序。

### 跨模块 comp 依赖

comp 函数可以调用其他模块的 comp 函数：

```cpp
// moduleA.ncc
export module moduleA;
export comp int32_t get_size() { return 10; }

// moduleB.ncc
import moduleA;
export comp int32_t array_size = moduleA::get_size();
```

**构建系统保证**：
1. 编译 `moduleB` 前，`moduleA` 已完全编译
2. `moduleA.ncc.meta` 包含 `get_size` 的完整 AST
3. 编译器从 `.ncc.meta` 读取并求值 `get_size()`

**循环依赖检测**：

```cpp
// moduleA.ncc
import moduleB;
export comp int32_t a = moduleB::b + 1;

// moduleB.ncc
import moduleA;
export comp int32_t b = moduleA::a + 1;  // ✗ 编译错误：comp 依赖循环
```

编译器在求值 comp 常量时检测循环，报错并列出依赖链：

```
error: circular comp dependency detected
  moduleA::a depends on moduleB::b
  moduleB::b depends on moduleA::a
```

### 实例化缓存与增量编译

编译器为每个 `<comp 函数, 类型参数>` 组合缓存生成的代码：

```cpp
// 第一次编译
Vector<int32_t> v1;  // 实例化并缓存 Vector<int32_t>

// 第二次编译（Vector 定义未变）
Vector<int32_t> v2;  // 直接使用缓存
```

缓存位置：
- 项目内：`target/.ncc-cache/`
- 跨项目：`~/.ncc/cache/`（见 [包管理](10-packages.md#全局缓存结构)）

增量编译时，只有依赖的 comp 定义改变时才重新实例化。

## 设计改进总结

本章实现的七个"免费"改进（不增加新语法，只是统一和简化现有机制）：

### 1. 统一 `<>` 和 `()` 调用语法

**改进**：`X<Args>` 是 `X(^^Args...)` 的纯语法糖，两种写法完全等价。

**收益**：
- 类型参数成为真正的一等公民，可以传递、存储、计算
- 消除了"特殊语法"的印象
- 函数式风格和 C++ 风格可以自由选择

```cpp
Vector<int32_t> v1;              // 语法糖
Vector(^^int32_t) v2;            // 直接调用
comp type T = Vector(^^int32_t); // 类型作为值
T v3;
```

### 2. `comp class` 是纯语法糖

**改进**：明确 `comp class` 自动脱糖为 `comp type` 函数 + `define_class` 调用。

**收益**：
- 底层只有一个机制：`comp type` 函数
- 消除"一个功能两个语法"的困惑
- 用户可以根据场景选择最合适的形式

```cpp
// 这两者底层完全等价
comp class Ptr<type T> { /* ... */ };
comp type Ptr(type T) { return [: define_class(...) :]; }
```

### 3. 类型参数默认值

**改进**：`comp` 函数参数支持默认值，与普通函数一致。

**收益**：
- 减少样板代码
- 常见配置可以省略

```cpp
comp type Vector(type T, Device device = Device::Cpu) { /* ... */ }
Vector<float> v1;              // 使用默认 CPU
Vector<float, Device::Gpu> v2; // 显式指定 GPU
```

### 4. 类型参数推导

**改进**：从函数实参推导类型参数，减少显式标注。

**收益**：
- 代码更简洁
- 类型安全不降低

```cpp
comp T max<type T>(T a, T b) { return a > b ? a : b; }
auto m = max(1, 2);  // 推导为 max<int32_t>
```

### 5. 高阶 comp 函数

**改进**：允许 `comp` 函数返回 `comp` 函数。

**收益**：
- 构建可复用的类型生成器组合
- 实现类型级别的策略模式

```cpp
comp auto choose_container(bool use_vector) {
    return use_vector ? VectorOf : ListOf;
}
```

### 6. `^^Type` 随处可用

**改进**：`^^Type` 可以在任何上下文中使用，不限于 `comp` 上下文。

**收益**：
- `TypeInfo` 成为真正的一等值
- 可以在运行时传递和存储类型信息
- 反射 API 随处可用

```cpp
void print_info(TypeInfo t) {  // 普通函数
    println("{}: {}", name_of(t), size_of(t));
}
print_info(^^int32_t);  // 允许！
```

**限制**：
- `^^(expr)` 表达式反射仍限于 comp 上下文
- `[: :]` splice 仍限于 comp 上下文

### 7. 自动识别编译期常量

**改进**：编译器自动识别编译期可求值的表达式。

**收益**：
- 无需手动标记 constexpr
- const 变量、字面量、编译期函数调用都自动识别

```cpp
const size_t SIZE = 10;
Array<int32_t, SIZE> arr;  // 自动识别为编译期常量
```

### 8. 统一 comp 块和 comp 函数规则

**改进**：明确 `comp {}` 块等价于匿名 `comp` 函数。

**收益**：
- 求值规则完全一致，无特殊情况
- 行为可预测

```cpp
comp { /* ... */ }
// 等价于
comp void _anon() { /* ... */ }
_anon();
```

## 总结

NCC 的 `comp` 系统通过以下设计实现了最大的一致性：

1. **单一底层机制**：只有 `comp` 函数，`comp class` 和 `<>` 都是语法糖
2. **类型作为值**：`TypeInfo` 可以像普通值一样传递和存储
3. **随处可用的反射**：`^^Type` 和反射 API 不限于 comp 上下文
4. **函数式与 C++ 风格并存**：`Vector(^^T)` vs `Vector<T>` 任选
5. **推导和默认值**：减少样板代码
6. **高阶能力**：comp 函数可以返回 comp 函数

这些设计都不需要新增语法，只是去掉人为限制、统一已有机制，实现"免费"的通用性提升。
```
