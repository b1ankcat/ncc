# 一、模块系统（替代头文件与 `namespace`）

C++26 已经有 `export module` / `import` 这套模块机制（C++20 引入）。删除
`#include` 头文件机制和 `namespace` 机制，只保留模块——这是"只删不加"的直接
体现：模块语法本身不是新东西，只是不再需要头文件和命名空间来配合它。

## 定义模块

```cpp
// math.ncc —— 定义一个模块
export module math;

export double square(double x) {
    return x * x;
}
```

## 使用模块

```cpp
// main.ncc —— 使用模块
import math;

int main() {
    double r = square(3.0);
}
```

## 核心库是内置全局符号

以下符号像 `int32_t`、`bool` 一样是语言内置的一部分，直接使用，不挂在任何
`std` 或其他命名空间下，也不需要 `import`。**本表是核心库全局符号的唯一来源**，
其他章节不再各自列举。用户自己拆分的多文件项目之间用 `import 模块名;` 互相引用。

| 类别 | 符号 |
| --- | --- |
| 字符串 | `String`、`StringView` |
| 容器 | `Vector<T, Device>`、`Array<T, N>`、`HashMap<K, V>` |
| 可选值 | `Optional<T>`、`reference_wrapper<T>` |
| 智能指针 | `unique_ptr<T>`、`shared_ptr<T>`、`weak_ptr<T>`、`make_unique`、`make_shared` |
| 值容器 | `Any` |
| 输出与格式化 | `println`、`print`、`format`、`Logger`、`log_info`/`log_warn`/`log_error`、`LogLevel` |
| 反射：句柄与语法 | `Info`、`^^`、`[: :]` |
| 反射：查询 | `name_of`、`type_of`、`size_of`、`alignment_of`、`offset_of`、`fields_of`、`methods_of`、`bases_of`、`nonstatic_data_members_of`、`variants_of`、`captures_of`、`capture_mode_of`、`CaptureMode` |
| 反射：枚举作用域 | `types_of`、`functions_of`、`types_deriving_from`、`current_module` |
| 反射：谓词 | `is_scalar`、`is_pointer`、`is_aggregate`、`is_device_view`、`derives_from`、`satisfies`、`has_annotation` |
| 反射：运行时访问 | `dynamic_type_of`、`register_dynamic_type`、`get_field`、`set_field`、`invoke` |
| 反射：代码生成 | `define_aggregate`、`define_class`、`data_member_spec`、`make_array_type` |
| 类型操作 | `cast<T>`、`is<T>`、`Bits<T>`（仅作 `cast` 目标）、`move`、`declval`、`StorageOps` |
| 断言与诊断 | `assert`、`SourceLocation`、`comp_assert`、`compile_error`、`check` |
| 异常 | `Exception` 及其派生层次（见[类型系统](02-types.md#标准异常层次)） |
| 并发 | `TaskScope`、`Task<T>`、`Channel<T>`、`Sender<T>`、`Receiver<T>`、`Completion`、`Completion<T>`、`Mutex<T>`、`RwLock<T>`、`Semaphore`、`Atomic<T>`、`MemoryOrder`、`parallel`、`reduce`、`Schedule`、`blocking`、`Thread`、`Duration`、`AggregateException` |
| 惰性序列 | `Generator<T>`、管道适配器 `filter`、`map`、`take`、`for_each`、`collect<C>` |
| 设备 | `Device`、`DeviceView<T>`、`DeviceView2D<T>`、`GpuId` |
| 编译期查询 | `profile()`、`target_os()`、`target_arch()`、`target_triple()` |
| 文件 | `File` |

低精度数值类型（`float16_t`、`bfloat16_t`、`float8_e4m3_t` 等）见
[类型系统](02-types.md#基础类型)，同为内置类型。

导入模块不自动形成 namespace；同名导出使用已批准的 `import module as alias;`
扩展建立模块别名：

```cpp
import http as http_client;
import another_http as other_http;

http_client::Client a;
other_http::Client b;
```

`as` 只建立模块访问前缀，不改变导出符号原名。别名用于两个模块**模块名本身**
冲突的场合；导出符号同名的处理见[名称查找](#名称查找)。

### `::` 的三种用途

删除 `namespace` 后 `::` 仍然保留，但只有三种确定含义，按左操作数的种类区分：

| 形式 | 左操作数 | 含义 |
| --- | --- | --- |
| `mod::name` | 已导入的模块名或其别名 | 访问该模块的导出符号 |
| `Type::member` | 类型名 | 类的静态成员、嵌套类型、tagged enum 变体 |
| `Enum::Value` | 枚举类型名 | 枚举成员 |

三者不会歧义：**模块名不能与类型名同名**。具体规则：

1. **检查范围**：当前模块的顶层类型名（不含嵌套类型）
2. **检查时机**：双向检查
   - 导入模块时，检查模块名（或其别名）是否与当前已定义的类型名冲突
   - 定义类型时，检查类型名是否与已导入的模块名（或别名）冲突
3. **大小写敏感**：`http` 和 `HTTP` 是不同标识符，不冲突
4. **核心库保护**：[核心库符号表](#核心库是内置全局符号)中的所有名称不可用作模块名
   （包管理器在发布包时检查并拒绝）

```cpp
// ✓ 合法
struct Vector { int x; };  // 局部 Vector
import geometry;           // geometry 导出的 Vec2 不冲突

// ✗ 错误
struct Http { /* */ };
import Http;  // 错误：Http 已是当前模块的类型名

// ✗ 错误
import geometry;
struct geometry { /* */ };  // 错误：geometry 已是导入的模块名

// ✗ 错误（发布包时）
export module String;  // 错误：String 是核心库符号
```

`::` 不能嵌套用于模块（不存在 `a::b::c` 形式的子模块路径），模块名本身可以含 `.`，
如 `import net.http;` 后写 `net.http::Client`。没有全局作用域限定符 `::name`
的用法，因为不存在需要与之区分的嵌套命名空间。

```cpp
import net.http as http;

http::Client client;              // 模块导出符号
File::default_permissions();      // 类静态成员
Shape::Circle(5.0);               // tagged enum 变体
LogLevel::Debug;                  // 枚举成员
```

## 名称查找

删除 `namespace` 后，C++ 的 ADL（实参依赖查找）失去了载体——它的规则是"到实参
类型所属的 namespace 里找"。但 ADL 解决的问题依然存在：非成员运算符和泛型代码
里的依赖调用都需要在调用处找到定义在别处的自由函数。

**替代规则：把"关联 namespace"换成"关联模块"。** 未限定名的候选集由四个来源组成，
**按优先级从高到低查找**：

| 来源 | 内容 | 优先级 |
| --- | --- | --- |
| 当前作用域 | 局部声明、当前模块的所有声明（含未导出的） | 1（最高）|
| 显式导入 | 所有 `import` 模块的导出符号 | 2 |
| 实参关联模块 | 每个实参类型**定义所在模块**的导出符号 | 3 |
| 核心库 | 所有核心库全局符号（见[核心库符号表](#核心库是内置全局符号)）| 4（最低）|

优先级规则：有多个来源有候选时，**先按优先级过滤，只保留最高优先级层的候选**，
然后在该层内按 C++ 重载规则选择。核心库符号优先级最低，被任何用户定义或导入的
同名符号隐藏。

第三条是 ADL 的对应物，使非成员运算符能被找到：

```cpp
// geometry.ncc
export module geometry;

export struct Vec2 { double x; double y; };

// 非成员运算符：必须如此才能让 scalar * vec 与 vec * scalar 对称
export Vec2 operator+(Vec2 a, Vec2 b);
export Vec2 operator*(double k, Vec2 v);
```

```cpp
// main.ncc
import geometry;

Vec2 c = a + b;        // 实参类型 Vec2 定义在 geometry，故在其中查找 operator+
Vec2 d = 2.0 * a;      // 同理；double 是内置类型，只贡献核心库候选
```

泛型代码依赖同一规则，这是它存在的主要理由：

```cpp
comp void dump<type T>(const T& v) {
    println("{}", to_string(v));   // 到 T 定义所在的模块里找 to_string
}
```

`dump` 自己不知道 `to_string` 在哪，也不该 `import` 每个可能的类型所在的模块。
实例化时 `T` 已确定，其关联模块随之确定。

### 关联模块的确定

对每个实参，取其类型的**定义位置**所在模块。规则按类型种类递归应用：

- **类、结构体、枚举**：其定义所在模块
- **泛型实例**（`Vector<Vec2>`、`HashMap<K, V>`）：
  - 核心库泛型（`Vector`、`HashMap`、`Optional` 等）：不贡献自己的关联模块，只贡献类型参数的
  - 用户定义泛型（`comp class`）：贡献定义模块 + 所有类型参数的关联模块
  - 类型参数本身是泛型实例时，递归提取其关联模块
  - 最终关联模块集合自动去重
- **指针与引用**（`T*`、`T&`、`T&&`）：去掉修饰后的底层类型的关联模块
- **类型别名**（`using Point = Vec2`）：等于其底层类型的关联模块（别名透明）
- **内置类型**（`int32_t`、`double`、`bool`）：无关联模块

示例：

```cpp
import geometry;  // 导出 Vec2
import graph;     // 导出 comp class Graph<type T>

// 核心库泛型
Vector<Vec2> points;              // 关联模块：geometry
HashMap<String, Vec2> map;        // 关联模块：geometry（String 是核心库）
Vector<Vector<Vec2>> nested;      // 关联模块：geometry（递归提取）

// 用户定义泛型
Graph<Vec2> g;                    // 关联模块：graph + geometry
```

**实参关联查找的可见性边界**：关联模块必须在**当前模块的传递依赖闭包**内，即：
- 当前模块直接 `import` 的所有模块
- 以及它们递归 `import` 的所有模块

泛型函数的实例化发生在**调用点**所在的模块，查找使用该模块的依赖闭包：

```cpp
// geometry.ncc
export module geometry;
export struct Vec2 { double x, y; };
export Vec2 operator+(Vec2 a, Vec2 b);

// utils.ncc
export module utils;
// 注意：utils 不需要 import geometry

export comp void print_sum<type T>(T a, T b) {
    auto result = a + b;   // 依赖 T 的 operator+，但此处不查找
    println("{}", result);
}

// main.ncc
import geometry;  // ← 关键：实例化点所在模块导入了 geometry
import utils;

utils::print_sum(Vec2{1,2}, Vec2{3,4});  
// ✓ 实例化发生在 main.ncc，geometry 在其依赖闭包内，找到 operator+
```

这使泛型定义独立于具体类型模块，同时保持构建系统的依赖图可追踪：构建顺序由
`import` 声明确定，`.ncc.meta` 只需包含直接依赖的类型信息。

### comp 函数的名称查找

`comp` 函数（编译期函数）的调用**遵循相同的优先级系统和实参关联规则**：

```cpp
// math.ncc
export module math;
export comp int32_t square(int32_t x) { return x * x; }

// geometry.ncc
export module geometry;
import math;
export comp int32_t area(int32_t side) {
    return square(side);  // ✓ 通过显式导入找到（优先级2）
}

// utils.ncc
export module utils;
export struct Vec2 { double x, y; };
export comp String describe(Vec2 v) { 
    return format("({}, {})", v.x, v.y); 
}

// main.ncc
import geometry;
import utils;

comp void print_desc<type T>(const T& value) {
    println("{}", describe(value));  // 查找 describe
}

comp {
    print_desc(Vec2{1, 2});  
    // ✓ 实例化在 main.ncc，实参 Vec2 关联 utils，main 导入了 utils
}
```

`comp` 函数可以调用运行时函数（生成调用代码），运行时代码可以调用 `comp` 函数
（编译期求值）。`import` 同时为编译期和运行时建立依赖，无需单独的"编译期导入"。

反射通过 `^^T` 捕获类型的完整定义（从 `.ncc.meta` 读取），类型信息随类型本身
传递，不受当前模块是否导入该类型定义模块的限制。

### 歧义与优先级

**重载决议采用两阶段过程**：

1. **收集候选**：从所有四个优先级层收集符合名称的候选函数
2. **优先级过滤**：只保留**最高优先级层**的候选，丢弃所有低优先级的
3. **重载决议**：在保留的候选中，按 C++ 重载规则选择最佳匹配
4. **歧义检查**：如果有多个候选同等好，报编译错误

这意味着低优先级的候选即使是更好的匹配，也会被高优先级的候选隐藏：

```cpp
void process(double x);           // 优先级1（当前作用域）
import math;                      // 导出 process(int32_t)，优先级2

process(42);                      // 调用 process(double)
// 虽然 process(int32_t) 是精确匹配，但被优先级1隐藏
```

这与 C++ 的"局部声明隐藏外层"一致，但与 C++ 的 ADL 不同（ADL 候选与普通候选
合并，无优先级）。NCC 的优先级系统使核心库符号可被替换，同时保持当前作用域
的直觉优先性。

同一优先级层内有多个候选同等匹配时**报歧义**，要求显式限定：

```cpp
import http;      // 导出 parse(String)
import json;      // 也导出 parse(String)

// parse("...");           // 错误：候选来自 http 与 json，歧义
auto a = http::parse("...");   // 显式限定消解
```

**运算符查找遵循统一的优先级系统**。运算符的特殊性体现在语法层面，而非查找规则：

1. **不能写显式限定**：`mod::operator+(a, b)` 不是合法语法，运算符必须以中缀形式调用
2. **歧义无法消解**：如果多个候选同等匹配，编译器报错，且无法用限定名消解

这意味着当前模块可以定义任何类型的运算符（包括核心库类型），按优先级1获胜；
两个模块为同一组实参类型定义了相同签名的运算符且处于同一优先级层时，只能由
这些类型的定义方协调。

```cpp
// 当前模块的运算符（优先级1）
String operator+(String a, String b) {
    return custom_concat(a, b);  // 覆盖核心库的 operator+
}

void test() {
    String s = "a" + "b";  // 使用当前模块的版本
}
```

### 与显式限定的关系

`mod::name` 绕过上述查找，直接在指定模块中解析。它是歧义的消解手段，也用于
可读性（明确标示符号来源）。规则细节：

1. **核心库符号不可限定**：它们没有模块前缀。一旦被用户符号隐藏就不可访问——这是
   设计意图，核心库符号可被替换。需要同时使用核心库版本和用户版本时，应给用户符号
   起不同的名字（如 `CustomString` 而非 `String`）。

2. **当前模块可以自限定**：`geometry::Vec2` 在 `geometry.ncc` 内部访问自己的导出符号，
   主要用于可读性（当前作用域已是最高优先级）。不能访问非导出符号。

3. **限定完全绕过查找**：`mod::name(args)` 只在 `mod` 解析 `name`，不触发实参关联查找，
   不考虑其他模块或优先级。这是明确语义："我要这个模块的符号，不管别处有什么"。

4. **嵌套限定左结合**：`mod::Type::Member` 解析为 `(mod::Type)::Member`——第一个 `::` 
   是模块限定，后续 `::` 是成员访问。

5. **重载决议范围**：`geometry::distance(a, b)` 只考虑 `geometry` 导出的所有 `distance` 
   重载，不考虑其他模块的候选。

```cpp
import geometry;
geometry::Vec2 v;              // 限定访问
geometry::Vec2::Polar p;       // 嵌套：先 geometry::Vec2，再 ::Polar

import user;  // 导出 String
user::String s1;               // 用户的 String
// String s2;                  // 被 user::String 隐藏（优先级 2 > 4）
// 解决方案：给用户类型起不同的名字，如 UserString
```

未限定查找失败或有歧义时，编译器的诊断必须列出所有候选及其所属模块。

## 内置模块

`gpu`、`benchmark`、`build`、`libc`、`posix` 等由编译器提供的模块**需要显式
`import`**，与第三方包写法一致；它们不是核心库的全局符号。区别在于无需在
`package.toml` 中声明依赖：

```cpp
import gpu;         // 内置模块，不需要写进 package.toml
import http;        // 第三方包，必须在 package.toml 声明

gpu::available();
```

只有 `String`、`Vector`、`Optional`、`println` 这类核心库符号才是无需 `import`
的全局符号。

## 格式化：`println` / `format`

`println`/`format` 共用格式化引擎：占位符支持位置参数、具名参数、宽度/精度/对齐/进制，
语法就是 `std::format` 的花括号规格（去掉 `std::` 前缀，作为内置函数）：

```cpp
println("{}", 42);                                 // 位置参数
println("{0} {1} {0}", "a", "b");                  // 位置索引，可重复引用
println("{name} is {age}", "name"_a = "Alice", "age"_a = 25); // 具名参数
println("{:>10.2f}", 3.14159);                      // 对齐 / 精度
```

`println` 内部对标准输出做同步保护，多线程并发调用不会出现交叉写乱序的
字符——这是核心库实现细节，用户不需要自己加锁。`format(...)` 返回
`String`，格式规则与 `println` 完全一致，只是不直接写输出流。

## 内置日志库

核心库内置一个日志类型 `Logger`，不需要 `import`：

```cpp
log_info("user {} logged in", user.id);
log_warn("cache miss for key {}", key);
log_error("failed to open {}: {}", path, err);

Logger logger{.min_level = LogLevel::Debug, .sink = File("app.log")};
logger.info("custom logger instance");
```

日志级别 `LogLevel::{Trace, Debug, Info, Warn, Error, Fatal}`；格式字符串
与 `println`/`format` 共用同一套格式化引擎，没有第二套格式规则。写入是
线程安全的（按 sink 做同步保护），多线程并发调用不会交叉写坏一行日志。
默认的全局 `log_*` 函数写向一个默认 `Logger` 实例；需要自定义时用聚合
初始化创建自己的 `Logger`，指向不同 sink（文件、`stderr`、自定义 `Writer`）。

## comp class 的导出与实例化

### 导出规则

```cpp
export module containers;

// ✓ 导出 comp class（完整定义）
export comp class CustomPtr<type T> {
    T* ptr_;

public:
    explicit CustomPtr(T* p = nullptr) noexcept : ptr_(p) {}
    ~CustomPtr() noexcept { delete ptr_; }

    CustomPtr(const CustomPtr&) = delete;
    CustomPtr& operator=(const CustomPtr&) = delete;

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

    T& operator*() { return *ptr_; }
    T* operator->() { return ptr_; }
};

// ✗ 不能只导出声明
export comp class CustomPtr<type T>;  // 编译错误：comp class 必须包含完整定义
```

`CustomPtr` 仅示范拥有单个对象的指针封装，要求对象析构不抛异常。裸指针
不会自动禁止拷贝，因此此处显式定义所有权操作；comp 不改变特殊成员函数规则。

**完整定义必须可见的原因：**

- `comp class` 是编译期代码生成机制
- 编译器需要在使用处看到完整的类体才能实例化
- 模块元数据必须包含实例化所需的完整实现

### 实例化模型

```cpp
// 模块 A：定义 comp class
export module containers;

export comp class CustomPtr<type T> {
    T* ptr_;
    // ... 完整实现
};

// 模块 B：使用 comp class
import containers;
import user;

int main() {
    // 编译器在此处实例化 CustomPtr<User>
    CustomPtr<User> p(new User{1, "Alice"});
}
```

**实例化流程：**

1. **延迟实例化**：编译器在首次使用 `CustomPtr<User>` 时生成代码
2. **符号去重**：链接器自动合并重复的实例化符号（使用 weak symbols）
3. **增量缓存**：编译器缓存已实例化的类型，加速增量编译

### .ncc.meta 元数据文件

每个模块编译后生成 `.ncc.meta` 文件，包含：

- `comp class` 的完整 AST
- 导出符号的类型签名
- 依赖的其他模块列表

**示例：**

```bash
$ ccc build containers.ncc
# 生成：
#   containers.o       - 目标文件（不含 comp class 实例化代码）
#   containers.ncc.meta - 元数据文件（包含完整 AST）
```

其他模块导入时，编译器从 `.ncc.meta` 读取完整定义并按需实例化。

### 编译和链接流程

```bash
# 阶段 1：编译各模块（生成 .o 和 .ncc.meta）
ccc build containers.ncc   # → containers.o + containers.ncc.meta
ccc build user.ncc         # → user.o + user.ncc.meta
ccc build main.ncc         # → main.o（包含实例化的 CustomPtr<User>）

# 阶段 2：链接
ccc link main.o containers.o user.o → main.exe
# 链接器自动去重重复的 CustomPtr<User> 符号
```

**缓存优化：**

```bash
target/.ccc-cache/
  ├── containers.CustomPtr_User.o      # CustomPtr<User> 的缓存
  ├── containers.CustomPtr_Product.o   # CustomPtr<Product> 的缓存
  └── ...
```

项目内的编译缓存位于 `target/.ccc-cache/`，跨项目共享的包缓存位于
`~/.ccc/cache/`（见 [包管理](10-packages.md#全局缓存结构)）。

编译器为每个 `<comp_class, 类型参数>` 组合缓存生成的代码，
如果定义未改变，直接使用缓存，加速增量编译。
