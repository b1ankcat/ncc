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

## Tagged Enum（携带数据的枚举）

唯二语法例外之一，语法对应 Rust 的 `enum`：

```cpp
enum Shape {
    Circle(double),              // 半径
    Rect(double, double),        // 宽、高
    Point,                       // 无数据
}

// 构造：标准聚合初始化语法
Shape s = Shape::Circle(5.0);
s = Shape::Rect(2.0, 4.0);          // 重新赋值成另一个变体，同一个 `=`

// 取值：`cast<T>`/`is<T>` 同一套转换 API，把"变体名"当成一个具体类型看待
if (auto c = cast<Shape::Circle>(s)) {
    println("radius = {}", c->radius);
}
if (is<Shape::Rect>(s)) {
    println("s is a Rect");
}

// 数组：固定大小用内置 Array<T, N>，动态数组用内置 Vector<T>
Array<int32_t, 10> fixed;   // 固定大小数组
Vector<int32_t> dyn;        // 动态数组

// 指针（无检查，完全由程序员控制）：标准后缀写法
Point* p;

// 引用：标准后缀写法，就是普通的 C++ 引用
Point& r;
```

**注意：** 没有 `mut` 关键字，所有变量默认可变。`T&`/`T*` 的语义与标准 C++
完全一致，没有额外的编译器检查（见"内存管理"章节）。
