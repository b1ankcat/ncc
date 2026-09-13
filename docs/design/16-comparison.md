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
| 字符串 | `std::string` (多种编码) | `String` (UTF-8) | `[]const u8` | **`String` (UTF-8 + COW + SSO)** |
| 核心库特权 | 是（编译器魔法） | 部分（`std` 特殊） | 否 | **否（与用户代码一致）** |

## 详细对比

### vs C++

**NCC 的改进**：
- ✅ 删除头文件/namespace/std 前缀，只保留模块
- ✅ 统一泛型机制：`comp` 函数 + `<>` 替代 `template`
- ✅ 统一类型转换：`cast<T>` 替代四个 cast
- ✅ 统一编译期计算：`comp` 替代 constexpr/consteval
- ✅ 内置包管理（类 Cargo）
- ✅ 反射作为一等特性
- ✅ 更快的编译速度（全局缓存）

**保留的 C++ 特性**：
- ✅ C++ 特殊成员函数与拷贝消除规则，组合类型优先 Rule of Zero
- ✅ 虚表和虚函数
- ✅ RAII 和智能指针
- ✅ `T&`/`T*` 语义不变
- ✅ 异常处理

### vs Rust

**相似之处**：
- 零成本抽象
- 强大的编译期计算
- 现代包管理

**关键区别**：
- ❌ **无借用检查器**：NCC 不做编译时内存安全检查，与 C++ 一致
- ✅ **虚表保留**：支持开放式多态（Rust 需要 Trait Objects）
- ✅ **C++ 拷贝与移动**：按类型契约进行拷贝或移动，裸资源封装需显式处理所有权
- ✅ **更简单的学习曲线**：无生命周期标注
- ✅ **C++ 互操作性**：共享生态

**适用场景**：
- 需要内存安全 → 选 Rust
- 需要 C++ 生态 + 现代语法 → 选 NCC

### vs Zig

**相似之处**：
- 编译期计算作为核心特性
- 简洁的语法
- 快速编译
- 无隐藏控制流

**关键区别**：
- ✅ **语法基线**：NCC 基于 C++26，Zig 从零设计
- ✅ **虚表内置**：NCC 保留 C++ 虚函数，Zig 需手动实现
- ✅ **RAII**：NCC 自动析构，Zig 需 defer
- ✅ **反射 API 更统一**：编译期/运行时同一套
- ❌ **无 comptime 类型**：NCC 的 `type` 只能在 `comp` 函数参数中使用

**适用场景**：
- 需要极简语法 + 手动控制 → 选 Zig
- 需要 C++ 生态 + RAII → 选 NCC

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

## 性能对比

| 场景 | C++ | Rust | Zig | NCC |
|------|-----|------|-----|-----|
| 编译速度 | ⭐⭐ | ⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 运行时性能 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| 内存安全 | ⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐ (与 C++ 一致) |
| 开发速度 | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| 互操作性 | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
