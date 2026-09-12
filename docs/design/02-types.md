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

核心库内置的 `String` 类型设计目标：**UTF-8、零拷贝视图、写时复制、高性能**。

```cpp
// String 内部表示（简化说明）
struct String {
    char* data;        // UTF-8 编码数据
    size_t len;        // 字节长度（不是字符数）
    size_t capacity;   // 分配容量
    size_t* refcount;  // 引用计数（用于写时复制）
};
```

**核心特性**：

1. **默认 UTF-8**：所有字符串字面量、`String` 内部存储都是 UTF-8
2. **写时复制（COW）**：赋值/传参只复制指针，修改时才真正复制数据
3. **小字符串优化（SSO）**：短字符串（≤23 字节）内联存储，无堆分配
4. **零拷贝视图**：`StringView` 不拥有数据，只是 `(const char*, len)` 指针对

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
    if (target_os == "windows") {
        // Windows UTF-16 转换在编译期决定是否包含
    }
}

// 或者运行时检测
Vector<wchar_t> wide = s.to_wide();  // Windows 上可用（核心库提供）
```

### 性能保证

- **小字符串（≤23 字节）**：零堆分配
- **中等字符串**：写时复制，赋值 O(1)
- **拼接**：`reserve()` 预分配避免多次重分配
- **视图**：`StringView` 零拷贝，适合传参

### 安全性

- **自动内存管理**：析构自动释放
- **边界检查**：`at(i)` 检查越界，`operator[]` 不检查（性能）
- **UTF-8 验证**：`from_bytes()` 可选验证 UTF-8 合法性

## 数组与动态数组

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
Optional<int32_t> maybe = get_value();

if (maybe.has_value()) {
    println("{}", maybe.value());
}

// 配合 if 初始化语句
if (auto v = get_value()) {
    println("{}", v.value());
}

// 提供默认值
int32_t x = maybe.value_or(0);
```

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
public:
    virtual String what() const = 0;
    virtual ~Exception() = default;
};

class RuntimeError : public Exception {};
class LogicError : public Exception {};

// I/O 异常
class IOException : public RuntimeError {};
class FileNotFound : public IOException {};
class PermissionDenied : public IOException {};
class ConnectionRefused : public IOException {};

// 解析异常
class ParseError : public RuntimeError {
    size_t position_;
public:
    ParseError(String msg, size_t pos);
    size_t position() const { return position_; }
};

// 逻辑错误
class InvalidArgument : public LogicError {};
class OutOfRange : public LogicError {};
class NullPointerError : public LogicError {};
```

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
        throw IOException::from_errno(errno, path);
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
    unique_ptr<Data> p(new Data());
    
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
        throw IoException::from_errno(errno, path);
    }
    return File(fd);
}

// 错误码到异常的映射
class IoException : public Exception {
public:
    static IoException from_errno(int err, const String& path) {
        String msg = format("{}: {}", path, strerror(err));
        
        switch (err) {
            case ENOENT: throw FileNotFound(msg);
            case EACCES: throw PermissionDenied(msg);
            default: throw IoException(msg);
        }
    }
};
```

## Tagged Enum（携带数据的枚举）

唯二语法例外之一，语法对应 Rust 的 `enum`：

```cpp
enum Shape {
    Circle(double),              // 半径
    Rect(double, double),        // 宽、高
    Point,                       // 无数据
};

// 构造：标准聚合初始化语法
Shape s = Shape::Circle(5.0);
s = Shape::Rect(2.0, 4.0);

// 取值：使用 match comp 函数
double area = match(s) {
    Circle(r) => 3.14 * r * r,
    Rect(w, h) => w * h,
    Point => 0.0,
};

// 或者使用 cast/is API
if (auto c = cast<Shape::Circle>(s)) {
    println("radius = {}", c->radius);
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

// 编译器生成的赋值运算符：
// 1. 调用当前变体（Circle）的析构函数
// 2. 复制构造新变体（Rect）
Shape& Shape::operator=(const Shape& other) {
    if (this != &other) {
        destroy_current_variant();  // 析构旧变体
        
        tag = other.tag;
        switch (tag) {
            case Tag::Circle:
                new (&data.circle) Circle(other.data.circle);
                break;
            case Tag::Rect:
                new (&data.rect) Rect(other.data.rect);
                break;
            case Tag::Point:
                new (&data.point) Point(other.data.point);
                break;
        }
    }
    return *this;
}
```

### match comp 函数（穷尽性检查）

`match` 是 `comp` 函数，提供穷尽性检查：

```cpp
// match 是表达式，可以返回值
double area = match(s) {
    Circle(r) => 3.14 * r * r,
    Rect(w, h) => w * h,
    Point => 0.0,
};

// match 也可以是语句（不返回值）
match(s) {
    Circle(r) => println("Circle: radius {}", r),
    Rect(w, h) => println("Rect: {}x{}", w, h),
    Point => println("Point"),
};

// 编译期穷尽性检查
double bad = match(s) {
    Circle(r) => 3.14 * r * r,
    Rect(w, h) => w * h,
    // 缺少 Point 变体
};
// 编译错误：
// error: non-exhaustive pattern match
// note: missing variant: Shape::Point

// 使用通配符 _ 捕获剩余变体
double approx = match(s) {
    Circle(r) => 3.14 * r * r,
    _ => 0.0,  // 匹配 Rect 和 Point
};
```

### 泛型 Tagged Enum

```cpp
// 泛型枚举示例
enum Option<type T> {
    Some(T),
    None,
};

// 使用
Option<int32_t> maybe = Option::Some(42);

match(maybe) {
    Some(value) => println("Value: {}", value),
    None => println("No value"),
};
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
    return match(*e) {
        Lit(n) => n,
        Add(left, right) => eval(left) + eval(right),
        Mul(left, right) => eval(left) * eval(right),
    };
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
