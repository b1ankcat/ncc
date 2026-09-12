# 十二、总结

## NCC 的核心价值

1. **语法基线是 C++26** - 不发明新语法，只做删减；唯三例外是 `comp` 关键字
   （及其内部的 `type T` 参数声明）、tagged enum 和 `import` 关键字扩展
2. **一个问题一个解法，贯彻到底** - 智能指针原样保留标准三个、转换合并为一个
   函数、反射合并为一套 API、代码组织合并为一套模块机制、**泛型统一为 `comp`
   函数（删除 `template`）**
3. **`comp` 统一编译期能力** - 一个关键字取代 `constexpr`/`consteval`，并
   驱动反射与代码生成；**`template` 被 `comp` 函数 + `<>` 调用语法完全替代**
4. **`const` 保留运行时不可变语义** - `comp` 负责编译期求值，`const` 负责
   运行时只读，两者解决不同层次的问题，和 C++ 完全一致
5. **能力边界与原生 C++ 完全一致** - 不引入任何"看起来更安全但不完备"的
   额外编译器检查，`T&`/`T*`/拷贝语义都是标准 Rule of Zero 行为，没有隐藏
   规则；虚表保留（开放式多态必需）
6. **核心库与用户代码一致** - 核心库类型（`Vector`、`String`、`unique_ptr`）
   使用与用户代码相同的语法和机制，都通过构造函数/析构函数实现 RAII，没有特权
7. **零历史包袱** - 直接站在 C++26 的语法之上做减法，不重新发明轮子
8. **内置包管理** - 类 Cargo 的包管理系统，解决依赖、版本、构建问题

## 设计哲学

> **"C++26 已经有的语法就直接用；C++26 没有但必须要的能力，只加 `comp`、
> tagged enum 和扩展 `import` 这三处；同一个问题如果 C++ 给了好几种解法，
> 合并成一种（如 `template` 被 `comp` 函数彻底替代）；C++ 已经用 Rule of Zero
> 解决的问题，不再叠加一层自造的检查机制。"**

NCC 不是 Rust 的竞争者，也不是要重新发明一门新语言。它是 C++26 的一个受限、
自洽的子集：

- 如果你需要 Rust 级别的内存安全 → 用 Rust
- 如果你需要完整的 C++ 生态和全部语法 → 用标准 C++26
- **如果你需要 C++26 的反射与元编程能力，但希望每个问题只有一种解法、
  语法更收敛、有现代包管理 → 用 NCC**

## 适用场景

### ✅ 适合 NCC 的项目

- 系统编程（操作系统、驱动、嵌入式）
- 高性能计算（游戏引擎、图形渲染、科学计算）
- AI/ML 推理引擎（利用低精度类型硬件加速）
- 网络服务（高并发、低延迟）
- 工具链开发（编译器、解释器、构建工具）
- 需要 C++ 互操作的项目

### ❌ 不适合 NCC 的项目

- 需要严格内存安全保证的关键系统 → 用 Rust
- Web 前端 → 用 TypeScript/JavaScript
- 快速原型开发 → 用 Python/Go
- 简单脚本 → 用 Shell/Python

## 与 C++ 的互操作

### 调用 C++ 代码

```cpp
// C++ 库（legacy.h）
namespace legacy {
    int compute(int x);
}

// NCC 代码
extern "C++" {
    int legacy_compute(int x);  // 映射到 legacy::compute
}

int main() {
    int result = legacy_compute(42);
    println("{}", result);
}
```

### 从 C++ 调用 NCC

```cpp
// NCC 模块（导出为 C++ 兼容）
export module mylib;

export int my_function(int x) {
    return x * 2;
}

// C++ 代码
import mylib;  // C++26 模块导入

int main() {
    return my_function(21);
}
```

## 开发路线图

### Phase 1：核心编译器（6-12 个月）

- [ ] Lexer + Parser
- [ ] AST + HIR
- [ ] 类型检查
- [ ] `comp` 执行引擎
- [ ] 基础反射 API
- [ ] C99 代码生成
- [ ] 基础错误报告

### Phase 2：核心库（3-6 个月）

- [ ] `String` (UTF-8 + COW + SSO)
- [ ] `Vector<T>`、`Array<T, N>`
- [ ] `Optional<T>`（错误联合 `Result<T, E>` 的最终形式待确认）
- [ ] `unique_ptr`、`shared_ptr`、`weak_ptr`
- [ ] `println`、`format`
- [ ] 基础日志

### Phase 3：包管理（3-6 个月）

- [ ] `package.toml` 解析
- [ ] 依赖解析算法
- [ ] 全局缓存系统
- [ ] 注册中心协议
- [ ] `ccc` 命令行工具

### Phase 4：标准库扩展（持续）

- [ ] 文件 I/O
- [ ] 网络（TCP/UDP/HTTP）
- [ ] 并发（线程、异步）
- [ ] JSON/XML/YAML 解析
- [ ] 正则表达式
- [ ] 加密/哈希

### Phase 5：优化（持续）

- [ ] LLVM 后端
- [ ] 增量编译
- [ ] 并行编译
- [ ] LTO（链接时优化）
- [ ] 低精度类型硬件支持

## 贡献指南

### 参与开发

```bash
# 克隆仓库
git clone https://github.com/ccc-lang/ccc.git
cd ccc

# 构建编译器
make build

# 运行测试
make test

# 构建文档
make docs
```

### 代码风格

- 遵循项目现有代码风格
- 提交前运行 `make fmt`
- 所有公开 API 需要文档注释
- 新功能需要测试覆盖

### 提交 PR

1. Fork 仓库
2. 创建特性分支 (`git checkout -b feature/amazing-feature`)
3. 提交更改 (`git commit -m 'Add amazing feature'`)
4. 推送到分支 (`git push origin feature/amazing-feature`)
5. 开启 Pull Request

## 许可证

NCC 使用 [MIT License](LICENSE)。

## 社区

- **GitHub**: https://github.com/ccc-lang/ccc
- **论坛**: https://discuss.ccc-lang.org
- **Discord**: https://discord.gg/ccc-lang
- **文档**: https://docs.ccc-lang.org

## 致谢

NCC 的设计受到以下语言的启发：
- C++26（语法基线）
- Rust（包管理、错误处理）
- Zig（comptime 概念）
- Carbon（C++ 继承者探索）
