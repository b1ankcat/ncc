# NCC - Modern C++ Simplified

NCC 是基于 C++26 的现代系统编程语言，通过"只删不加"的设计哲学简化语法，统一元编程机制。

NCC 的哲学是：如果 c++ 只用一个语法来表示一个功能，没有历史兼容包袱，那么能做到多现代？

## 核心特性

- **🎯 语法基线 = C++26**：只删不加，站在巨人肩膀上
- **⚡ `comp` 统一编译期计算**：替代 constexpr/consteval/template
- **🔍 内置反射**：统一的 `TypeInfo` 和 `ExprInfo` 句柄，支持运行时类型、字段和方法查询/调用
- **🧵 结构化并发**：TaskScope 统一数据并行、任务和 Channel 生命周期
- **📦 包管理**：依赖解析、锁定文件和全局缓存
- **🚀 快速编译**：MLIR/LLVM 后端 + 增量与全局缓存
- **🔗 C 互操作**：显式 C ABI，支持 C 库和 C 包装层

## 快速示例

```cpp
// 定义数据结构
struct User {
    uint64_t id;
    String name;
    String email;
};

// 自动生成调试输出（通过反射）
comp String debug_string<type T>(const T& v) {
    String s = String(name_of(^^T)) + "{";
    bool first = true;
    for (auto field : nonstatic_data_members_of(^^T)) {
        if (!first) s += ", ";
        first = false;
        s += String(name_of(field)) + ": " + debug_string(v.[:field:]);
    }
    s += "}";
    return s;
}

// 使用
int main() {
    User user{1, "Alice", "alice@example.com"};
    println("{}", debug_string(user));
    // 输出: User{id: 1, name: Alice, email: alice@example.com}
}
```

## 设计哲学

> **"C++26 已经有的语法就直接用；必须要的能力只加 `comp`、tagged enum 和扩展 `import`；同一个问题合并成一种解法；拷贝与移动沿用 C++ 特殊成员函数规则，资源封装定义所有权操作，组合类型优先 Rule of Zero。"**

### 核心简化

| | C++ | NCC |
|------|-----|-----|
| 代码组织 | 头文件 + namespace + 模块 | **只有模块** |
| 编译期计算 | constexpr / consteval / template | **`comp` 统一** |
| 编译期断言 | static_assert（消息只能是字面量） | **`comp_assert` / `compile_error`，消息可格式化** |
| 文本替换 | 预处理器 | **删除，能力由 `comp` 承担** |
| 类型转换 | 4 种 cast | **`cast<T>` 一个，语义由目标类型决定** |
| 泛型 | `template<typename T>` | **`comp T f<type T>(T value)`** |
| 类型约束 | concept | **`comp bool` 函数** |
| 异步 | 协程 + 执行器 | **绿色线程；协程只用于惰性序列** |

## 快速开始

```bash
# 安装（尚未发布，待实现）
curl -sSf https://install.ncc-lang.org | sh

# 创建新项目
ncc new myapp
cd myapp

# 构建并运行
ncc run
```

## 文档

完整设计文档位于 [docs/design/](docs/design/)：

- **[概览](docs/design/00-overview.md)** - 核心理念与设计原则
- **[模块系统](docs/design/01-modules.md)** - 删除头文件和 namespace
- **[类型系统](docs/design/02-types.md)** - 基础类型、字符串、Tagged enum
- **[内存管理](docs/design/03-memory.md)** - RAII、智能指针、Rule of Zero
- **[反射系统](docs/design/04-reflection.md)** - 统一的反射 API
  - `TypeInfo` 和 `ExprInfo` 运行时类型句柄、字段/方法查询与动态调用
- **[comp 系统](docs/design/05-comp.md)** - 编译期计算与泛型
- **[类型转换](docs/design/06-casting.md)** - `cast<T>` 统一转换
- **[接口系统](docs/design/07-interfaces.md)** - 抽象基类 + 虚函数
- **[并发](docs/design/08-concurrency.md)** - TaskScope、Channel、数据并行
- **[GPU](docs/design/09-gpu.md)** - 设备视图与异构计算
- **[包管理](docs/design/10-packages.md)** - 依赖解析、全局缓存
- **[构建系统](docs/design/11-build-system.md)** - package.toml + build.ncc
- **[C 互操作](docs/design/12-interop.md)** - 显式 C ABI
- **[编译器架构](docs/design/13-compiler.md)** - MLIR/LLVM 后端
- **[性能](docs/design/14-performance.md)** - Benchmark 与优化
- **[完整示例](docs/design/15-examples.md)** - HTTP 服务器、测试框架等
- **[语言对比](docs/design/16-comparison.md)** - vs C++/Rust/Zig

## 与其他语言的对比

| | C++ | Rust | Zig | NCC |
|------|-----|------|-----|-----|
| 编译期内存安全 | 无 | 借用检查 | 无 | 无（与 C++ 一致） |
| 多态 | 虚表 | Trait Objects | 手动 vtable | 虚表（沿用 C++） |
| 元编程 | 模板 + constexpr | 宏 | comptime | `comp` + 反射 |
| 包管理 | 无标准 | Cargo | 内置 | 内置 |
| 语法来源 | 自身演进 | 自成体系 | 自成体系 | C++26 子集 |

**选择指南**：
- 需要编译期内存安全保证 → **Rust**
- 需要完整 C++ 生态与 C++ 库直接互操作 → **C++26**
- 需要 C++ 语义 + 统一元编程 + C ABI 互操作 → **NCC**

详细取舍见 [16-comparison.md](docs/design/16-comparison.md)。

## 项目状态

🚧 **设计阶段** - 规范持续定稿，编译器实现进行中；文档中的未实现 API 不代表已发布功能。

### 路线图

- [ ] **Phase 1**: 核心编译器（6-12 个月）
  - Lexer、Parser、AST、HIR
  - 类型检查、`comp` 执行引擎
  - MLIR 代码生成（CPU 目标）
- [ ] **Phase 2**: 核心库（3-6 个月）
  - String、Vector、Optional
  - 智能指针、println/format
- [ ] **Phase 3**: 包管理（3-6 个月）
  - package.toml、依赖解析
  - 全局缓存、注册中心
- [ ] **Phase 4**: 标准库扩展（持续）
- [ ] **Phase 5**: LLVM 后端优化（持续）

## 贡献

欢迎贡献！请阅读 [CONTRIBUTING.md](CONTRIBUTING.md)（待创建）了解详情。

```bash
# 克隆仓库
git clone https://github.com/ncc-lang/ncc.git
cd ncc

# 构建（待实现）
make build

# 运行测试
make test
```

## 许可证

MIT License - 详见 [LICENSE](LICENSE)

## 社区

- **GitHub**: https://github.com/ncc-lang/ncc（待创建）
- **文档**: https://docs.ncc-lang.org（待创建）
- **论坛**: https://discuss.ncc-lang.org（待创建）
- **Discord**: https://discord.gg/ncc-lang（待创建）
