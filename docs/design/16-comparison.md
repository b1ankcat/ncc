# 十六、与其他语言的对比

## 特性对比表

| 特性 | C++ | Rust | Zig | NCC |
|------|-----|------|-----|-----|
| 零成本抽象 | ✓ | ✓ | ✓ | ✓ |
| 编译期计算 | 模板 + constexpr/consteval | 宏 + const fn | comptime | **`comp`（统一 constexpr/consteval）+ 反射，`template` 被 `comp` 函数彻底替代** |
| 反射 | C++26 静态反射（新） | 无（宏模拟） | comptime + @typeInfo | **统一的一套 API（编译期/运行时通用），无"静态/动态"之分** |
| 代码组织 | 头文件 + namespace + 模块（三套并存） | 模块系统 | 文件即模块 | **只有模块，无头文件、无 namespace** |
| 类型转换 | `static_cast`/`dynamic_cast`/`reinterpret_cast`/`const_cast` 多个 | `as`/`TryFrom` | `@as`/`@intCast` 等多个 | **只有 `cast<T>` + 简写 `is<T>`** |
| 所有权/拷贝语义 | 特殊成员函数，优先 Rule of Zero | Borrow Checker | 手动 | **沿用 C++ 特殊成员函数规则，资源封装显式定义所有权操作** |
| 虚函数/多态 | 虚表 + 抽象基类 | Trait Objects | 手动 vtable | **虚表保留，和 C++ 完全一致** |
| 内存安全 | 否 | 是（严格） | 否 | **与原生 C++ 完全一致，不多不少** |
| 学习曲线 | 陡峭 | 陡峭 | 中等 | **中等（C++26 子集）** |
| 编译速度 | 慢 | 慢 | 快 | **快（全局缓存）** |
| 语法基线 | 自身演进 | 自成体系 | 自成体系 | **C++26 子集，只删不加** |
| 泛型机制 | 模板 | Trait + 泛型 | comptime | **`comp` 函数 + `<>` 调用** |
| 包管理 | 无标准（CMake/vcpkg/Conan） | Cargo | Zig build | **内置（类 Cargo）** |
| 构造语法 | `()` / `{}` / `=` 多种 | 聚合初始化 | 聚合初始化 | **聚合初始化 + 构造函数（RAII）** |
| 默认可变性 | 可变 | 不可变 | 可变 | **可变（无 `mut` 关键字）** |
| 错误处理 | 异常 / `std::expected` | `Result<T, E>` | 错误联合 | **异常 + `Optional<T>`** |
| 字符串 | `std::string` (多种编码) | `String` (UTF-8) | `[]const u8` | **`String` (UTF-8 + SSO)** |
| 核心库特权 | 是（编译器魔法） | 部分（`std` 特殊） | 否 | **否（与用户代码一致）** |

## 详细对比

### vs C++

**删除的**：头文件、`namespace` 与 `std` 前缀（只保留模块）、`template`、
`concept`、`static_assert`、四种 cast、预处理器、`co_await` 与 awaiter 协议。

**新增的**：`comp`、tagged enum、扩展 `import`，以及内置包管理与一等反射。

**语义不变的**：特殊成员函数与拷贝消除规则、虚表与虚函数、RAII 与智能指针、
`T&`/`T*`、异常处理。这是 NCC 与 Rust/Zig 的根本差异——运行时语义就是 C++ 的，
变的只有书写方式和元编程机制。

### vs Rust

**共同点**：零成本抽象、强大的编译期计算、内置包管理。

**根本分歧在内存安全的实现方式**：Rust 用借用检查器在编译期证明安全，代价是
生命周期标注与所有权重构；NCC 不做这类检查，安全性由程序员和 RAII 保证，代价是
保留了 C++ 的全部内存错误可能。这不是程度差异而是路线差异，不存在折中版本。

其余差异：NCC 保留虚表（Rust 用 Trait Objects），拷贝与移动按 C++ 类型契约而非
默认移动语义，可与 C 生态直接互操作。

**选择**：需要编译期内存安全保证选 Rust；需要 C++ 语义与生态选 NCC。

### vs Zig

**共同点**：编译期计算是核心特性、语法简洁、无隐藏控制流。

**主要差异**：Zig 从零设计语法，NCC 是 C++26 的子集；Zig 手动实现 vtable 与
`defer`，NCC 保留虚函数与 RAII 自动析构。

Zig 的 `comptime` 比 `comp` 更自由：`type` 在 Zig 里是普通的一等值，可以存进变量、
放进数组；NCC 的 `type` 只能作 `comp` 函数的参数。代价是 Zig 的错误信息更依赖
实例化上下文。

**选择**：偏好极简语法与完全手动控制选 Zig；需要 RAII、虚表与 C++ 语义选 NCC。

## 代码示例对比

本节其他语言代码仅用于比较，不属于 NCC 源代码；NCC 示例以对应设计章节的最终契约为准。

### 泛型函数

**C++**:
```cpp
template<typename T>
T max(T a, T b) {
    return a > b ? a : b;
}
```

**Rust**:
```rust
fn max<T: PartialOrd>(a: T, b: T) -> T {
    if a > b { a } else { b }
}
```

**Zig**:
```zig
fn max(comptime T: type, a: T, b: T) T {
    return if (a > b) a else b;
}
```

**NCC**:
```cpp
comp T max<type T>(T a, T b) {
    return a > b ? a : b;
}
```

### 反射遍历字段

**C++26**:
```cpp
constexpr auto t = ^^User;
template for (constexpr auto field : nonstatic_data_members_of(t)) {
    std::println("{}", name_of(field));
}
```

**Rust** (proc macro):
```rust
#[derive(Debug)]
struct User { ... }
// 需要宏生成代码
```

**Zig**:
```zig
inline for (@typeInfo(User).Struct.fields) |field| {
    std.debug.print("{s}\n", .{field.name});
}
```

**NCC**:
```cpp
comp {
    auto t = ^^User;
    for (auto field : nonstatic_data_members_of(t)) {
        println("{}", name_of(field));
    }
}
```

### 智能指针

**C++**:
```cpp
std::unique_ptr<Data> p = std::make_unique<Data>(...);
```

**Rust**:
```rust
let p = Box::new(Data { ... });
```

**Zig**:
```zig
var p = try allocator.create(Data);
defer allocator.destroy(p);
```

**NCC**:
```cpp
unique_ptr<Data> p = make_unique<Data>(...);
```

## 性能

运行时性能应与 C++ 相当——降低到相同的 LLVM IR，虚表布局与异常机制也相同，
没有引入额外的抽象层。编译速度的目标高于 C++（无头文件重复解析、模块化编译、
全局缓存），但 `comp` 求值和单态化会抵消一部分收益，实际差距需要实现后测量。

具体的验收门槛见 [14-performance.md](14-performance.md#验收目标)。
