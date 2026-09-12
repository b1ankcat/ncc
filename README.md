# NCC - Modern C++ Simplified

NCC 是基于 C++26 的现代系统编程语言，通过"只删不加"的设计哲学简化语法，统一元编程机制。

## 核心特性

- **🎯 语法基线 = C++26**：只删不加，站在巨人肩膀上
- **⚡ `comp` 统一编译期计算**：替代 constexpr/consteval/template
- **🔍 内置反射**：统一的反射 API，编译期/运行时通用
- **🧵 现代并发**：cpu::parallel/gpu::parallel 统一数据并行 + Go 风格任务并发
- **📦 现代包管理**：类 Cargo 的依赖管理和全局缓存
- **🚀 快速编译**：C 后端 + 全局缓存，比传统 C++ 快数倍
- **🔗 C++ 互操作**：无缝调用 C++ 库，共享生态

## 快速示例

```cpp
// 定义数据结构
struct User {
    uint64_t id;
    String name;
    String email;
};

// 自动生成调试输出（通过反射）
comp fn debug_string<type T>(v: const T&) -> String {
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

> **"C++26 已经有的语法就直接用；必须要的能力只加 `comp`、tagged enum 和扩展 `import`；同一个问题合并成一种解法；C++ 已经用 Rule of Zero 解决的问题，不再叠加自造的检查机制。"**

### 核心简化

| 特性 | C++ | NCC |
|------|-----|-----|
| 代码组织 | 头文件 + namespace + 模块 | **只有模块** |
| 编译期计算 | constexpr/consteval/template | **`comp` 统一** |
| 类型转换 | 4 种 cast | **`cast<T>` 一个** |
| 泛型 | template<typename T> | **`comp fn f<type T>`** |
| 类型约束 | concept | **`comp bool` 函数** |

## 快速开始

```bash
# 安装（尚未发布，待实现）
curl -sSf https://install.ccc-lang.org | sh

# 创建新项目
ccc new myapp
cd myapp

# 构建并运行
ccc run
```

## 文档

完整设计文档位于 [docs/design/](docs/design/)：

- **[概览](docs/design/00-overview.md)** - 核心理念与设计原则
- **[模块系统](docs/design/01-modules.md)** - 删除头文件和 namespace
- **[类型系统](docs/design/02-types.md)** - 基础类型、字符串、Tagged enum
- **[内存管理](docs/design/03-memory.md)** - RAII、智能指针、Rule of Zero
- **[反射系统](docs/design/04-reflection.md)** - 统一的反射 API
- **[comp 系统](docs/design/05-comp.md)** - 编译期计算与泛型
- **[类型转换](docs/design/06-casting.md)** - `cast<T>` 统一转换
- **[接口系统](docs/design/07-interfaces.md)** - 抽象基类 + 虚函数
- **[编译器架构](docs/design/08-compiler.md)** - C99 后端和 LLVM 后端
- **[包管理](docs/design/09-packages.md)** - 依赖解析、全局缓存
- **[完整示例](docs/design/10-examples.md)** - HTTP 服务器、测试框架等
- **[语言对比](docs/design/11-comparison.md)** - vs C++/Rust/Zig
- **[总结](docs/design/12-summary.md)** - 核心价值与路线图

## 与其他语言的对比

| 特性 | C++ | Rust | Zig | NCC |
|------|-----|------|-----|-----|
| 零成本抽象 | ✓ | ✓ | ✓ | ✓ |
| 内存安全 | 否 | 是（严格） | 否 | 否（与 C++ 一致） |
| 学习曲线 | 陡峭 | 陡峭 | 中等 | 中等 |
| 编译速度 | 慢 | 慢 | 快 | 快 |
| 包管理 | 无标准 | Cargo | 内置 | 内置（类 Cargo） |
| 虚函数 | ✓ | Trait Objects | 手动 | ✓（保留） |

**选择指南**：
- 需要内存安全 → **Rust**
- 需要完整 C++ 生态 → **C++26**
- 需要现代语法 + C++ 互操作 → **NCC**

## 项目状态

🚧 **设计阶段** - 语言设计已完成，编译器实现进行中

### 路线图

- [ ] **Phase 1**: 核心编译器（6-12 个月）
  - Lexer、Parser、AST、HIR
  - 类型检查、`comp` 执行引擎
  - C99 代码生成
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
git clone https://github.com/ccc-lang/ccc.git
cd ccc

# 构建（待实现）
make build

# 运行测试
make test
```

## 许可证

MIT License - 详见 [LICENSE](LICENSE)

## 社区

- **GitHub**: https://github.com/ccc-lang/ccc（待创建）
- **文档**: https://docs.ccc-lang.org（待创建）
- **论坛**: https://discuss.ccc-lang.org（待创建）
- **Discord**: https://discord.gg/ccc-lang（待创建）

## 致谢

NCC 的设计受到以下语言的启发：
- **C++26**（语法基线）
- **Rust**（包管理、错误处理）
- **Zig**（comptime 概念）
- **Carbon**（C++ 继承者探索）
