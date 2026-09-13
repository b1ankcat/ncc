# 五、`comp` 系统

`comp` 是本设计唯一新增的关键字，统一取代 `constexpr` / `consteval`，并作为
反射与代码生成的入口。**`template` 关键字被彻底删除**，所有泛型机制统一为
`comp` 函数 + 反射生成，通过 `<>` 语法调用。

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
已批准的 comp 扩展；**求值与实例化的精确阶段划分尚未定稿**，见
[未定稿部分](00-overview.md#文档结构)。

下面的 `Vector` 只演示 `comp type` 的调用形式和布局生成，**不是核心库 Vector
的真实定义**：真实 Vector 用 `define_class` + `StorageOps` 生成，并须满足
[通用容器原语](03-memory.md#通用容器原语)与
[define_class](04-reflection.md#完整类定义define_class-api)规定的完整生命周期契约。

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

## `<>` 与 `()` 调用的区分

**核心规则：`X<Args>` 等价于编译期调用 `X(^^Args...)`**

```cpp
// 规则：X 必须是一个 comp 函数

// 类型构造
comp type Vector(type T) { /* ... */ }

Vector<int32_t> v;       // <> 调用：编译期 Vector(^^int32_t)
Vector(^^int32_t)        // () 直接调用，返回类型本身（不是对象）

// 类型关系查询：编译期检查两个类型的关系，与运行时的 is<T>(value) 不同名
comp bool derives_from(type Derived, type Base) { /* 编译期类型检查 */ }

derives_from(^^File, ^^Writer)     // 编译期检查 File 是否派生自 Writer

// 智能指针：类型和工厂都是 comp 函数
comp type unique_ptr(type T) { /* 生成 unique_ptr<T> 类型 */ }
comp auto make_unique(type T) { /* 生成该类型专用的工厂函数 */ }

unique_ptr<File> p;      // <> 调用：编译期 unique_ptr(^^File)，得到类型
unique_ptr<File> q = make_unique<File>("out.txt");
// make_unique<File> 是编译期调用 make_unique(^^File)，得到一个普通函数；
// ("out.txt") 是对该函数的运行时调用，参数原样转发给 File 的构造函数
```

工厂的两段调用体现了 `<>` 和 `()` 的分工：`<>` 里的类型参数在编译期消耗，
`()` 里的构造参数在运行时传递。这与 `Vector<T>` 只有编译期一段不同——
`make_unique<T>` 的生成结果是函数而不是类型。

**`<>` vs `()` 对比**：

| 语法 | 含义 | 参数处理 |
|------|------|----------|
| `X<A, B>` | 编译期调用 `X(^^A, ^^B)` | 自动加 `^^` 提升为 `Info` |
| `X(a, b)` | 普通函数调用 | 参数原样传递 |

**适用类型**：
- `Vector<T>`、`Optional<T>`、`unique_ptr<T>` - 类型生成
- `make_unique<T>(...)`、`make_shared<T>(...)` - 函数生成，再运行时调用
- `is<T>(value)`、`cast<T>(value)` - 类型查询/转换
- 所有需要类型作为参数的 comp 函数

## 非类型参数

`comp` 函数除了接受类型参数（`type T`），还可以接受编译期常量值参数（对应 C++26 的 non-type template parameters）：

```cpp
// 固定大小数组
comp type Array(type T, size_t N) {
    return [: define_aggregate("Array", {
        {make_array_type(^^T, N), "data"}
    }) :];
}

Array<int32_t, 10> arr;  // 编译期调用 Array(^^int32_t, 10)

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

Vector<float, Device::Gpu> gpu_vec;
Vector<float> cpu_vec;  // 默认是 Device::Cpu
```

**调用语法规则：**

`X<TypeArg, ValueArg>` 编译期调用 `X(^^TypeArg, ValueArg)`

- **类型参数自动加 `^^`**：转换为反射 `Info` 对象
- **非类型参数原样传递**：保持为编译期常量值

**支持的非类型参数类型：**
- 整数类型（`int32_t`, `size_t` 等）
- 枚举类型（`enum class Device`）
- 布尔类型（`bool`）
- 指针类型（编译期可求值的指针）

## `comp class` - 泛型类定义

`comp class` 是定义泛型类的语法，用于需要类型参数的复杂类定义。

### 基本语法

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

### 与 `comp type` 的区别

| 特性 | comp type | comp class |
|------|-----------|------------|
| 用途 | 简单聚合类型生成 | 复杂泛型类定义 |
| 语法 | 函数风格 | 类风格 |
| 成员函数 | 通过反射 API 注入 | 直接定义 |
| 类型参数使用 | 通过 `^^T` 引用 | 直接使用 `T` |
| 适用场景 | 编译期代码生成、简单数据结构 | 复杂内存管理、运算符重载、RAII |

两种写法最终都降低为公开的 `define_class` 描述和普通类定义：`comp type`
适合只生成布局，`comp class` 适合直接书写成员实现。二者没有不同的生命周期
或优化语义；需要资源管理时都必须生成或绑定明确的构造、析构、拷贝和移动操作。

### 何时使用 `comp class`

✅ **应该使用 comp class：**
- 需要自定义构造/析构逻辑
- 需要运算符重载
- 需要复杂的成员函数实现
- 需要 RAII 资源管理
- 需要类型参数在成员函数中自由使用

✅ **应该使用 comp type：**
- 只有数据成员的简单聚合
- 需要根据类型参数生成完全不同的结构布局
- 编译期批量代码生成

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
```

参数包展开使用 C++17 已有的折叠表达式。`{args...}` 只适用于所有参数类型相同
的情况；异质参数包必须用折叠表达式或递归展开。

## 裸 `comp { ... }` 块

保留顶层裸块，用于批量的编译期操作，编译器在翻译单元处理到此处时立即执行：

```cpp
comp {
    int32_t x = 10 + 20;
    println("Computed at compile time: {}", x);
}
```

## `comp` 块内部不需要重复标注 `comp`

已经身处 `comp { ... }` 块或者 `comp` 函数体内部的代码，本身就在编译期
上下文里执行——块/函数一级的 `comp` 已经声明过"这里是编译期"，块内部再
给每个局部变量、每个调用的函数、每个用到的类型重复加一次 `comp` 是多余
的重复标注。`comp` 只标在**边界**上（一个 `comp` 函数、一个 `comp` 类型、
一个裸 `comp` 块的起始处），进入边界之后就是普通语法：

```cpp
comp {
    // x、Local、helper 都不需要标 comp——已经在 comp 块内部，
    // 编译器知道这整块都在编译期求值
    int32_t x = 10 + 20;

    struct Local {
        int32_t a;
        int32_t b;
    };
    Local l{x, x * 2};

    auto helper = [](int32_t v) { return v * v; };
    println("{}", helper(l.a));
}
```

同理，`comp` 函数体内部调用的其他函数、定义的局部类型，只要不逃逸到运行时
上下文，都不需要单独标 `comp`——是否处于编译期由包住它的最近一层 `comp`
边界决定，不是逐个符号累加的属性。

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
```
