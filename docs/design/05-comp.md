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
- 项目内：`target/.ccc-cache/`
- 跨项目：`~/.ccc/cache/`（见 [包管理](10-packages.md#全局缓存结构)）

增量编译时，只有依赖的 comp 定义改变时才重新实例化。
```
