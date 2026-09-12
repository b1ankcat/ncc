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

```cpp
// 泛型类型定义（核心库 Vector 的简化示例）
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
comp fn max<type T>(a: T, b: T) -> T {
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

// 类型查询
comp bool is(type Target, type Source) { /* 编译期类型检查 */ }

is<File>(writer)         // <> 调用：编译期 is(^^File, ^^typeof(writer))
is(^^File, ^^Writer)     // () 直接调用，效果相同

// 智能指针
comp type unique_ptr(type T) { /* 生成 unique_ptr<T> 类型 */ }

unique_ptr<File> p;      // <> 调用：编译期 unique_ptr(^^File)，得到类型
unique_ptr<File>(new File(...));  // 先实例化类型，再调构造函数
```

**`<>` vs `()` 对比**：

| 语法 | 含义 | 参数处理 |
|------|------|----------|
| `X<A, B>` | 编译期调用 `X(^^A, ^^B)` | 自动加 `^^` 提升为 `Info` |
| `X(a, b)` | 普通函数调用 | 参数原样传递 |

**适用类型**：
- `Vector<T>`、`Optional<T>`、`unique_ptr<T>` - 类型生成
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
    
    // 构造函数
    CustomPtr(T* p = nullptr) : ptr_(p) {}
    
    // 析构函数
    ~CustomPtr() {
        if (ptr_) {
            delete ptr_;
        }
    }
    
    // 禁用拷贝
    CustomPtr(const CustomPtr&) = delete;
    CustomPtr& operator=(const CustomPtr&) = delete;
    
    // 移动构造
    CustomPtr(CustomPtr&& other) : ptr_(other.ptr_) {
        other.ptr_ = nullptr;
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

### 内存池分配器示例

```cpp
comp class MemoryPool<type T> {
    struct Block {
        T data;
        Block* next;
    };
    
    Block* free_list_;
    Vector<void*> allocated_chunks_;
    size_t chunk_size_;
    
    MemoryPool(size_t chunk_size = 1024) 
        : free_list_(nullptr), chunk_size_(chunk_size) 
    {
        allocate_chunk();
    }
    
    ~MemoryPool() {
        for (auto chunk : allocated_chunks_) {
            free(chunk);
        }
    }
    
    T* allocate() {
        if (!free_list_) {
            allocate_chunk();
        }
        Block* block = free_list_;
        free_list_ = block->next;
        return new (&block->data) T();
    }
    
    void deallocate(T* ptr) {
        ptr->~T();
        Block* block = reinterpret_cast<Block*>(ptr);
        block->next = free_list_;
        free_list_ = block;
    }
    
private:
    void allocate_chunk() {
        void* chunk = malloc(chunk_size_ * sizeof(Block));
        allocated_chunks_.push(chunk);
        
        Block* blocks = static_cast<Block*>(chunk);
        for (size_t i = 0; i < chunk_size_ - 1; ++i) {
            blocks[i].next = &blocks[i + 1];
        }
        blocks[chunk_size_ - 1].next = free_list_;
        free_list_ = blocks;
    }
};
```

### 与 `comp type` 的区别

| 特性 | comp type | comp class |
|------|-----------|------------|
| 用途 | 简单聚合类型生成 | 复杂泛型类定义 |
| 语法 | 函数风格 | 类风格 |
| 成员函数 | 通过反射 API 注入 | 直接定义 |
| 类型参数使用 | 通过 `^^T` 引用 | 直接使用 `T` |
| 适用场景 | 编译期代码生成、简单数据结构 | 复杂内存管理、运算符重载、RAII |

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

`Vector<T, Device>`, `String`, `HashMap<K, V>` 等核心容器是**编译器内置类型**，
使用与 `comp class` 相同的实现机制，但由编译器直接提供，享有特殊优化：

- 支持 `Device` 参数（CPU/GPU 内存）
- SIMD 向量化优化
- GPU 内核生成
- 与反射系统深度集成

用户自定义的 `comp class` 类型与核心容器类型地位平等，编译器对待方式一致。

## 变参泛型

```cpp
// 类型参数包（... 是 C++26 已有语法）
comp fn log<type... Args>(args: Args...) {
    // 用反射遍历参数包
    for (auto arg : {args...}) {
        println("{}", arg);
    }
}

log<int32_t, String, double>(42, "hello", 3.14);
```

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
    if (mode == "debug") {
        println("Debug mode enabled");
        enable_logging();
    } else if (mode == "release") {
        println("Release mode");
        enable_optimizations();
    }
}
```

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
