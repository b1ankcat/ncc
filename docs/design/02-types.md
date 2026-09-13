# 二、类型系统

## 基础类型

固定宽度整数类型等是核心库内置类型，不需要头文件、也不需要 `import`：

```cpp
// 整数
int8_t, int16_t, int32_t, int64_t
uint8_t, uint16_t, uint32_t, uint64_t
size_t

// 浮点
float, double

// 低精度整数（AI/ML 计算）
int4_t, uint4_t

// 低精度浮点（AI/ML 计算）
float16_t         // IEEE 754 half precision
bfloat16_t        // Brain Float 16 (Google)
tfloat32_t        // TensorFloat-32 (NVIDIA)
float8_e4m3_t     // 8-bit float, 4-bit exponent, 3-bit mantissa
float8_e5m2_t     // 8-bit float, 5-bit exponent, 2-bit mantissa
float4_nvfp4_t    // NVIDIA 4-bit float
float4_mxfp4_t    // Microsoft/AMD 4-bit float
float4_nf4_t      // NormalFloat 4-bit (QLoRA)

// 布尔
bool

// 字符与字符串（见下节）
char, char8_t, char16_t, char32_t
String
```

**低精度类型说明**：这些类型用于 AI/ML 加速和量化推理，直接内置到语言核心，
不需要外部库。编译器根据目标平台生成对应的硬件指令（AVX-512/AMX/CUDA/ROCm/
NPU）。软件实现作为 fallback。

## 字符串设计

### 字符类型

```cpp
char       // UTF-8 code unit (8-bit)，是默认字符类型
char8_t    // UTF-8 code unit，显式标注
char16_t   // UTF-16 code unit
char32_t   // Unicode code point
```

### String 类型

核心库内置的 `String` 类型设计目标：**UTF-8、小字符串免分配、零拷贝视图、高性能**。

`sizeof(String) == 24`，同一块存储有两种形态，靠最高位的判别位区分：

```
长形式（堆存储）：
  [0-7]   char* data          指向堆缓冲区
  [8-15]  size_t size         字节长度（不含终止符）
  [16-23] size_t capacity     容量；最高位为判别位 = 1

短形式（内联存储）：
  [0-22]  char buf[23]        UTF-8 数据 + NUL 终止符
  [23]    uint8_t control     低 7 位存长度；最高位为判别位 = 0
```

**核心特性**：

1. **默认 UTF-8**：所有字符串字面量、`String` 内部存储都是 UTF-8
2. **小字符串优化（SSO）**：**≤22 字节**内联存储，无堆分配
3. **始终 NUL 终止**：`c_str()` 因此是零拷贝的；容量计算为终止符多留一字节
4. **零拷贝视图**：`StringView` 不拥有数据，只是 `(const char*, size)` 指针对

SSO 容量是 22 而非 23：短形式的 23 字节缓冲区里必须留一个字节给 NUL 终止符。

**不采用写时复制（COW）**。COW 要求引用计数在多线程下原子化，使单线程用户也要为
跨线程共享的可能性付原子操作的代价，与"不为不用的功能付出代价"相悖；C++11 正是
因此禁止了 `std::basic_string` 的 COW。它还会与本文档的另外两条承诺冲突：
`operator[]` 不做检查（COW 下每个可写访问都要检查是否需要去共享）、`c_str()`
零拷贝（把指针交给 C 代码后共享状态失控）。避免拷贝请使用 `StringView` 或 `move()`
——显式且零开销，而非隐式地"也许省一次拷贝"。

**使用示例**：

```cpp
// 字面量
String s = "Hello, 世界";  // UTF-8 字面量

// 拼接
String greeting = "Hello, " + name;

// 子串（零拷贝视图）
StringView view = s.substr(0, 5);  // "Hello"

// 迭代字符（按 code point）
for (char32_t c : s.chars()) {
    println("{:x}", c);  // 打印 Unicode code point
}

// 迭代字节
for (char byte : s.bytes()) {
    println("{:02x}", byte);
}

// 格式化
String msg = format("User {} has {} points", name, score);
```

### 编码转换

```cpp
// UTF-8 ↔ UTF-16 ↔ UTF-32
String utf8 = "Hello";
Vector<char16_t> utf16 = utf8.to_utf16();
Vector<char32_t> utf32 = utf8.to_utf32();

String back = String::from_utf16(utf16);
```

### 兼容性

```cpp
// C 字符串互操作
const char* c_str = s.c_str();  // 以 null 结尾
String from_c = String::from_c_str(c_str);

// 平台字符串（使用 comp 条件编译）
comp {
    if (target_os() == "windows") {
        // Windows UTF-16 转换在编译期决定是否包含
    }
}

// 或者运行时检测
Vector<wchar_t> wide = s.to_wide();  // Windows 上可用（核心库提供）
```

### 性能保证

- **小字符串（≤22 字节）**：零堆分配，拷贝为 24 字节的按位复制
- **长字符串**：拷贝为 O(n)，分配新缓冲区；避免拷贝用 `StringView` 或 `move()`
- **移动**：O(1)，接管缓冲区并将源置为空的短形式
- **拼接**：`reserve()` 预分配避免多次重分配
- **视图**：`StringView` 零拷贝，适合传参

### `StringView`

```cpp
class StringView {
public:
    const char* data() const;   // 不保证 NUL 终止
    size_t size() const;
    // 没有 c_str()
};
```

`StringView` **不提供 `c_str()`**：它可能指向某个 `String` 的中间位置，该位置之后
没有终止符。需要传给 C API 时先构造 `String`。视图不拥有数据，来源 `String` 销毁
或重分配后即失效。

### 安全性

- **自动内存管理**：析构自动释放
- **边界检查**：`at(i)` 检查越界，`operator[]` 不检查（性能）
- **UTF-8 验证**：`from_bytes()` 可选验证 UTF-8 合法性

## 数组与动态数组

NCC 当前仅支持 64 位目标；32 位目标不在语言、ABI 和核心库支持范围内。

String 默认严格拒绝非法 UTF-8；显式替换模式使用 U+FFFD。StringView 不拥有
数据，来源 String 销毁或重分配后失效。低精度类型的宽度、对齐、舍入和累加精度
由类型定义；不支持的硬件使用软件 fallback 或在编译期拒绝。

```cpp
// 固定大小数组：内置 Array<T, N>（栈分配）
Array<int32_t, 10> fixed;
fixed[0] = 42;

// 动态数组：内置 Vector<T>（堆分配）
Vector<int32_t> dyn;
dyn.push(1);
dyn.push(2);

// 字符串就是特化的动态数组
String s = "Hello";  // 内部是 UTF-8 字节序列
```

## Optional 类型

内置的 `Optional<T>`（语义等价 `std::optional`，去掉 `std::` 前缀）：

```cpp
comp class Optional<type T> {
public:
    bool has_value() const;
    explicit operator bool() const;   // 用于 if / while 条件

    T& value();                       // 空值时抛出 LogicError
    const T& value() const;
    T value_or(T fallback) const;

    T& operator*();                   // 前置条件：has_value()，不检查
    const T& operator*() const;
    T* operator->();
    const T* operator->() const;
};
```

```cpp
Optional<int32_t> maybe = get_value();

if (maybe.has_value()) {
    println("{}", maybe.value());
}

// 配合 if 初始化语句：explicit operator bool 在条件中生效
if (auto v = get_value()) {
    println("{}", *v);
}

// 提供默认值
int32_t x = maybe.value_or(0);
```

`value()` 检查并在空值时抛出 `LogicError`；`operator*` 和 `operator->` 不检查，
要求调用方已确认 `has_value()`。`Optional<reference_wrapper<T>>`（`cast<T&>` 的
返回类型）用 `->` 取得 `reference_wrapper`，再用 `.get()` 取得被引用对象。

**Optional 不是错误处理机制**，而是表示"值可能不存在"的类型：

```cpp
// ✅ 使用 Optional：查找操作（找不到不是错误）
Optional<User> find_user(uint64_t id);

if (auto user = find_user(123)) {
    println("Found: {}", user->name);
} else {
    println("User not found");  // 正常情况，不是错误
}

// ✅ 使用 Optional：可选配置
struct Config {
    String host;
    Optional<uint16_t> port;  // 未设置时使用默认值
};

// ❌ 不要用 Optional 处理错误：
// 文件打开失败应该抛出异常，而不是返回 Optional<File>
```

## 断言：`assert`

预处理器整体删除（见[概览](00-overview.md)），因此 `assert` 不是宏。它是一个
**普通运行时函数**——条件的真假只有运行时才知道——但源码位置由编译期默认实参提供：

```cpp
void assert(bool condition, SourceLocation location = SourceLocation::current());
```

`SourceLocation::current()` 是 `comp` 函数，作为默认实参在**调用处**求值，因此
拿到的是调用者的位置而非 `assert` 自身的位置。这是 C++20 已有的机制，不需要宏。

```cpp
void withdraw(Account& account, int64_t amount) {
    assert(amount > 0);
    assert(account.balance >= amount);
    // ...
}

// 失败时的诊断：
// assertion failed at bank.ncc:12:5
```

### 需要表达式文本时反射表达式

上面的形式拿不到表达式文本——参数已经求值成 `bool`，源文本不在其中。需要更丰富的
诊断时把表达式**反射**后传入，用已有的 `^^` 与 splice，不引入新的参数种类：

```cpp
// 接受表达式的反射句柄，在编译期取其文本与结构，生成运行时检查
comp void check(Info expression);
```

```cpp
check(^^(account.balance >= amount));

// 生成的诊断包含表达式文本与两侧的值：
// assertion failed: account.balance >= amount
//   at bank.ncc:12:5
//   left  = 50
//   right = 120
```

`check` 是 `comp` 函数，但它**生成**的是运行时代码：编译期从 `Info` 取出表达式文本
与左右操作数，splice 回去构成运行时比较，失败时报告两侧的值。这与 `comp` 的既有
规则一致——`comp` 在编译期执行并产出运行时代码，而不是把运行时判断提前。

> `^^` 作用于表达式而非类型，要求反射能提供表达式级的 `Info`。当前
> [反射章](04-reflection.md)只规定了类型、字段、方法与捕获的反射；表达式反射的
> 具体能力边界待反射规范补充，因此 `check` 标为**待确认**。`assert` 不依赖该能力，
> 可独立成立。

报告子表达式的值是相对 C 宏的实际优势。两者都由独立的 `assertions` 构建配置项控制，
**与优化级别解耦**：release 构建默认关闭断言，但可以显式开启，不必为了保留断言而
放弃优化。

### 编译期断言：删除 `static_assert` 关键字

`static_assert` 是 C++ 的关键字，但它解决的问题——"编译期条件不满足时报错"——
已经由 `comp` 完全覆盖，因此**删除该关键字**，改用核心库的 `comp` 函数：

```cpp
comp void comp_assert(bool condition, String message);

// 无条件报错，用于不可达的分支
[[noreturn]] comp void compile_error(String format, auto... args);
```

```cpp
comp type Vector(type T) {
    comp_assert(size_of(^^T) > 0, "元素类型不能是不完整类型");
    // ...
}

comp bool check_capture(type T) {
    if (!is_device_transferable(T)) {
        compile_error("不是设备可传输类型：{}", name_of(^^T));
    }
    return true;
}
```

**为什么删除关键字而非保留**：

- **消息可格式化**。`static_assert` 只接受字符串字面量，拼不进类型名；
  `compile_error` 与 `println` / `format` 共用[同一套格式化引擎](01-modules.md#格式化println-format)，
  可以报告具体是哪个类型、哪个字段不满足条件。这是实践中最需要的能力。
- **条件由 `comp` 求值，规则统一**。`comp_assert` 的条件就是普通的 `comp` 表达式，
  不需要为 `static_assert` 单独规定"什么算常量表达式"。
- **符合"一个问题一个解法"**。编译期计算的入口只有 `comp`，报错也不该例外。

在 `comp` 上下文中，`throw` 同样使编译失败并产生诊断（见[概览](00-overview.md)）。
三者的分工：

| 形式 | 用途 |
| --- | --- |
| `comp_assert(cond, msg)` | 检查编译期条件，不满足则报错 |
| `compile_error(fmt, ...)` | 无条件报错，用于本不该到达的分支 |
| `comp` 中 `throw` | 编译期求值过程中的异常，诊断含异常信息与位置 |

运行时断言用 [`assert`](#断言assert)，那是普通函数；编译期用这里的 `comp` 函数。

## 错误处理机制

**NCC 使用 C++ 标准异常机制处理错误**，完全保留 `try`/`catch`/`throw` 语法。

### 异常语法

```cpp
// 标准 C++ 异常语法
try {
    File f("/path/to/file");
    String content = f.read_all();
    Data data = parse(content);
} catch (const IOException& e) {
    println("IO error: {}", e.what());
} catch (const ParseError& e) {
    println("Parse error at position {}: {}", e.position(), e.what());
} catch (const Exception& e) {
    println("Error: {}", e.what());
}

// 抛出异常
if (fd == -1) {
    throw IOException(format("Failed to open file: {}", path));
}
```

### 标准异常层次

```cpp
// 内置异常类型（对应 C++ 标准异常，去掉 std:: 前缀）
class Exception {  // 基类
    String message_;
public:
    explicit Exception(String message);
    virtual String what() const;   // 默认返回构造时的 message_
    virtual ~Exception() = default;
};

class RuntimeError : public Exception {
public:
    using Exception::Exception;     // 继承 (String) 构造函数
};

class LogicError : public Exception {
public:
    using Exception::Exception;
};

// I/O 异常
class IOException : public RuntimeError { public: using RuntimeError::RuntimeError; };
class FileNotFound : public IOException { public: using IOException::IOException; };
class PermissionDenied : public IOException { public: using IOException::IOException; };
class ConnectionRefused : public IOException { public: using IOException::IOException; };
class NetworkException : public IOException { public: using IOException::IOException; };

// 解析异常
class ParseError : public RuntimeError {
    size_t position_;
public:
    ParseError(String message, size_t position);
    size_t position() const { return position_; }
};

// 并发与设备异常
class ChannelClosed : public RuntimeError { public: using RuntimeError::RuntimeError; };
class GpuUnavailable : public RuntimeError { public: using RuntimeError::RuntimeError; };

// 逻辑错误
class InvalidArgument : public LogicError { public: using LogicError::LogicError; };
class OutOfRange : public LogicError { public: using LogicError::LogicError; };
class NullPointerError : public LogicError { public: using LogicError::LogicError; };
```

`Exception` 提供接受 `String` 的构造函数并实现 `what()`，派生类通过
`using Base::Base` 继承该构造函数，因此空的派生类体也是可用的具体类型；
`what()` 是虚函数，需要拼接额外上下文的类型（如 `ParseError`）可以覆盖它。
核心库不定义其他异常基类；用户自定义异常应从 `RuntimeError` 或 `LogicError`
派生，以便统一被 `catch (const Exception&)` 捕获。

### 何时使用异常 vs Optional

**使用异常：**
- 操作失败（文件打开失败、网络错误、解析失败）
- 前置条件违反（数组越界、空指针解引用）
- 资源耗尽（内存不足、文件描述符用尽）
- 构造函数失败（构造函数无法返回错误码）

**使用 Optional：**
- 查找操作（找不到不是错误，是正常情况）
- 可选配置（未设置时使用默认值）
- 函数可能无返回值（正常的业务逻辑）

```cpp
// ✅ 异常：文件打开失败是错误
File open_file(const String& path) {
    int fd = ::open(path.c_str(), O_RDONLY);
    if (fd == -1) {
        throw_errno(errno, path);
    }
    return File(fd);
}

// ✅ Optional：查找不到不是错误
Optional<User> find_user(uint64_t id) {
    if (auto it = users.find(id); it != users.end()) {
        return it->second;
    }
    return {};
}
```

### RAII 与异常安全

NCC 完全依赖 RAII 实现异常安全，资源通过析构函数自动释放：

```cpp
void process_file(const String& path) {
    File file(path);  // 构造时打开，析构时自动关闭
    
    // 如果下面的代码抛异常，file 自动析构（关闭文件）
    String content = file.read_all();
    Data data = parse(content);
    save(data);
    
    // 无需手动关闭文件
}

// 智能指针自动管理内存
void process() {
    unique_ptr<Data> p = make_unique<Data>(...);
    
    // 如果抛异常，p 自动析构（释放内存）
    risky_operation();
}
```

### 与 C 互操作

包装 C 函数，将错误码转换为异常：

```cpp
// C 函数：int open(const char* path, int flags);
// 返回 -1 表示失败，errno 包含错误码

File open_file(const String& path) {
    int fd = ::open(path.c_str(), O_RDONLY);
    if (fd == -1) {
        throw_errno(errno, path);   // 不返回
    }
    return File(fd);
}

// 错误码到异常的映射：核心库提供的辅助函数
[[noreturn]] void throw_errno(int err, const String& path) {
    String message = format("{}: {}", path, strerror(err));

    switch (err) {
        case ENOENT: throw FileNotFound(message);
        case EACCES: throw PermissionDenied(message);
        case ECONNREFUSED: throw ConnectionRefused(message);
        default: throw IOException(message);
    }
}
```

映射写成 `[[noreturn]]` 的自由函数，而不是返回异常对象的静态成员：具体抛出的
类型由 errno 决定，按值返回基类会丢失派生类型。

## Tagged Enum（携带数据的枚举）

三类语法例外之一：枚举变体可以携带数据，匹配使用普通调用形式的 `comp` 函数。

```cpp
enum Shape {
    Circle(double),              // 半径
    Rect(double, double),        // 宽、高
    Point,                       // 无数据
};

// 构造载荷值，再构造相应的枚举值
Shape s = Shape::Circle(5.0);
s = Shape::Rect(2.0, 4.0);

// 取值：使用 match comp 函数
double area = match(
    s,
    [](const Shape::Circle& c) {
        auto [r] = c;
        return 3.14 * r * r;
    },
    [](const Shape::Rect& rect) {
        auto [w, h] = rect;
        return w * h;
    },
    [](const Shape::Point&) { return 0.0; }
);

// 或者使用 cast/is API
if (auto c = cast<Shape::Circle&>(s)) {
    const auto& [radius] = c->get();
    println("radius = {}", radius);
}
if (is<Shape::Rect>(s)) {
    println("s is a Rect");
}
```

### Tagged Enum 的内存布局

编译器生成的内存布局（用户不可见）：

```cpp
// enum Shape { Circle(double), Rect(double, double), Point };
// 编译器生成：

struct Shape {
    enum class Tag : uint8_t {
        Circle = 0,
        Rect = 1,
        Point = 2,
    };
    
    Tag tag;
    alignas(8) union {
        struct { double radius; } circle;
        struct { double width; double height; } rect;
        struct {} point;  // 空
    } data;
    
    // 编译器自动生成构造/析构/赋值
};

// 内存布局：
// sizeof(Shape) = 24 字节
//   [0]     tag (uint8_t)
//   [1-7]   padding
//   [8-23]  union data (16 字节，最大变体的大小)
```

### Tagged Enum 的生命周期

```cpp
Shape s = Shape::Circle(5.0);
s = Shape::Rect(2.0, 4.0);  // 重新赋值

// 编译器生成的赋值运算符：先构造临时值，成功后再替换当前值。
// 新变体构造失败时，s 保持原来的 Circle，不会留下无效 tag。
Shape& Shape::operator=(const Shape& other) {
    if (this != &other) {
        Shape temporary(other);       // 可能抛出；当前对象尚未改变
        swap(temporary);               // 交换 tag 和活动载荷
    }
    return *this;
}
```

编译器为每个 Tagged enum 生成 `swap`：交换 tag 后，仅对实际活动的载荷调用
不抛异常的移动构造或交换操作；临时对象析构原来的载荷。若载荷不提供不抛
异常的交换/移动，赋值操作按 C++ 重载规则被删除或降级为其可用的异常保证，
不会伪造强保证。析构函数遵循 C++ 的 `noexcept` 规则；仅可移动载荷使枚举
自动成为仅可移动类型，不可默认构造载荷只限制对应变体构造路径。自赋值和
自移动遵循 C++ 特殊成员函数规则。

### match comp 函数（穷尽性检查）

`match(value, handler...)` 是普通调用形式的 **`comp` 函数**，不是关键字或
特殊表达式。处理函数使用普通 lambda 或其他可调用对象；函数实现通过公开的
comp/反射能力检查全部变体并生成分发代码，用户可实现同等能力的函数。

**调用契约：**

- 每个变体都有可命名的载荷类型，如 `Shape::Circle`。载荷按声明顺序支持
  标准结构化绑定；匿名载荷不自动获得 `radius`、`width` 等业务字段名。
- 处理函数组成重载集合，按 C++ 调用规则为每个变体选择唯一的可调用处理函数。
  任一变体缺少处理函数或调用有歧义，均在编译期报错。
- 只调用当前活动变体对应的处理函数一次。枚举表达式求值一次，载荷保留其
  const 和值类别传递；接收引用的处理函数不会因为匹配而复制载荷。
- 所有可达处理调用的返回类型必须一致（包含引用限定），全部为 `void` 也合法。
  处理函数的异常按普通函数调用规则传播。
- 普通泛型 lambda 可以作为兜底处理函数，不引入专用通配符或分支语法。

下例使用运行时枚举：穷尽检查和分发代码生成在编译期完成，选中的处理函数
在运行时执行。这里只确定调用契约；通用的 comp 求值阶段与反射展开规则
[尚未定稿](00-overview.md#文档结构)，但不给 `match` 单独设置阶段例外。

```cpp
// 普通函数调用，也可以只执行操作、不返回值
match(
    s,
    [](const Shape::Circle& c) {
        const auto& [r] = c;
        println("Circle: radius {}", r);
    },
    [](const Shape::Rect& rect) {
        const auto& [w, h] = rect;
        println("Rect: {}x{}", w, h);
    },
    [](const Shape::Point&) { println("Point"); }
);

// 编译期穷尽性检查
double bad = match(
    s,
    [](const Shape::Circle& c) {
        auto [r] = c;
        return 3.14 * r * r;
    },
    [](const Shape::Rect& rect) {
        auto [w, h] = rect;
        return w * h;
    }
    // 缺少 Point 变体
);
// 编译错误：
// error: non-exhaustive match call
// note: missing variant: Shape::Point

// 使用普通泛型 lambda 处理其余变体
double approx = match(
    s,
    [](const Shape::Circle& c) {
        auto [r] = c;
        return 3.14 * r * r;
    },
    [](const auto&) { return 0.0; }  // 匹配 Rect 和 Point
);
```

### 泛型 Tagged Enum

```cpp
// 泛型枚举示例
enum Option<type T> {
    Some(T),
    None,
};

// 使用
Option<int32_t> maybe = Option<int32_t>::Some(42);

match(
    maybe,
    [](const Option<int32_t>::Some& some) {
        const auto& [value] = some;
        println("Value: {}", value);
    },
    [](const Option<int32_t>::None&) { println("No value"); }
);
```

### 嵌套 Tagged Enum

```cpp
enum Expr {
    Lit(int32_t),
    Add(Expr*, Expr*),  // 递归
    Mul(Expr*, Expr*),
};

// 递归访问
int32_t eval(Expr* e) {
    return match(
        *e,
        [](const Expr::Lit& lit) {
            auto [n] = lit;
            return n;
        },
        [](const Expr::Add& add) {
            auto [left, right] = add;
            return eval(left) + eval(right);
        },
        [](const Expr::Mul& mul) {
            auto [left, right] = mul;
            return eval(left) * eval(right);
        }
    );
}
```

### 与 comp 系统集成

```cpp
// 编译期反射 tagged enum
comp {
    auto T = ^^Shape;
    
    // 查询所有变体
    for (auto variant : variants_of(T)) {
        println("Variant: {}", name_of(variant));
        
        // 查询变体的字段
        for (auto field : fields_of(variant)) {
            println("  {}: {}", name_of(field), name_of(type_of(field)));
        }
    }
}
```

## 数组与容器

```cpp

// 指针（无检查，完全由程序员控制）：标准后缀写法
Point* p;

// 引用：标准后缀写法，就是普通的 C++ 引用
Point& r;
```

**注意：** 没有 `mut` 关键字，所有变量默认可变。`T&`/`T*` 的语义与标准 C++
完全一致，没有额外的编译器检查（见"内存管理"章节）。
