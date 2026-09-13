# NCC 语言设计文档索引

NCC 是基于 C++26 的现代系统编程语言，通过"只删不加"的设计哲学简化语法，统一元编程机制。

## 快速导航

- **新手入门**: 从 [00-overview.md](00-overview.md) 开始
- **核心特性**: 阅读 [05-comp.md](05-comp.md) 了解 `comp` 系统
- **实战示例**: 查看 [15-examples.md](15-examples.md)
- **语言对比**: 参考 [16-comparison.md](16-comparison.md)

## 文档结构

本索引按当前目录中的实际文件维护；不存在的草案编号不作为链接。

### 基础概念

- **[00-overview.md](00-overview.md)** - 核心理念与设计原则
  - 语法基线：C++26 只删不加
  - 唯三例外：`comp`、tagged enum、扩展 `import`
  - 主要删除项：头文件、`namespace`、`template`、`concept`、四种 cast、
    预处理器、`co_await` 与 awaiter 协议
  - attribute 保留及"必须可忽略"判据
  - 设计原则与哲学

- **[01-modules.md](01-modules.md)** - 模块系统
  - 删除头文件和 namespace
  - 模块定义与使用
  - 核心库（println、format、日志）

- **[02-types.md](02-types.md)** - 类型系统
  - 基础类型（包含 AI/ML 低精度类型）
  - 字符串设计（UTF-8 + SSO + NUL 终止）
  - UTF-8 错误策略和 64 位目标边界
  - Tagged enum（携带数据的枚举）
  - 先构造后交换的异常安全赋值
  - Optional 类型

### 核心机制

- **[03-memory.md](03-memory.md)** - 内存管理
  - 聚合初始化 + 构造函数（RAII）
  - C++ 特殊成员函数规则：资源封装定义所有权操作，组合类型优先 Rule of Zero
  - 智能指针（unique_ptr/shared_ptr/weak_ptr）与 make_unique/make_shared 工厂
  - 引用和指针（标准 C++ 语义）

- **[04-reflection.md](04-reflection.md)** - 反射系统
  - 统一的反射句柄（Info）
  - 编译期代码生成运行时元数据和访问器
  - 运行时类型、字段和方法查询/调用

- **[05-comp.md](05-comp.md)** - comp 编译期计算
  - `comp` 关键字（替代 constexpr/consteval）
  - 泛型定义（`comp` 函数 + `type` 参数）
  - `<>` vs `()` 调用区分
  - 批量处理与代码生成

- **[06-casting.md](06-casting.md)** - 类型转换
  - `cast<T>` 统一转换函数
  - `is<T>` 类型检查
  - 指针转换规则与 `cast<Bits<T>>` 位模式重解释
  - `comp bool` 函数替代 `concept`

- **[07-interfaces.md](07-interfaces.md)** - 接口系统
  - 抽象基类 + 纯虚函数
  - 接口与反射结合

- **[08-concurrency.md](08-concurrency.md)** - 并发与多线程
  - 结构化轻量级任务（M:N 调度），唯一入口 `TaskScope::spawn`
  - `Task<T>` 生命周期：析构即 detach，孤儿异常在作用域汇总
  - 通道：`Sender`/`Receiver` 引用计数，所有权决定关闭时机
  - 惰性序列：生成器协程（保留 `co_yield`，删除 `co_await`）
  - 阻塞的 C 调用与协作式调度的让出点
  - 异步句柄对照：`Task<T>` / `Completion` / `Completion<T>`
  - 并发原语（不增加额外借用或数据竞争检查）

- **[09-gpu.md](09-gpu.md)** - GPU 与异构计算
  - DeviceView 与设备可传输性
  - GPU 捕获的设备可传输性检查
  - CUDA/ROCm/OneAPI 目标

### 工具链

- **[10-packages.md](10-packages.md)** - 包管理系统
  - package.toml 配置
  - 依赖解析（约束交集 + 最低可用版本）
  - 全局缓存（内容寻址）
  - 循环依赖和幽灵依赖处理
  - ccc 命令行工具

- **[11-build-system.md](11-build-system.md)** - 构建系统
  - 构建脚本（build.ncc）
  - 声明式与命令式 API
  - 编译配置与 Profiles
  - 代码生成与外部工具集成

- **[12-interop.md](12-interop.md)** - C 互操作
  - 显式 `extern "C"` ABI、目标 ABI 类型映射与回调

- **[13-compiler.md](13-compiler.md)** - 编译器架构
  - MLIR/LLVM 后端
  - 增量编译与并行编译
  - 错误报告

- **[14-performance.md](14-performance.md)** - 性能基准与优化
  - 可重复 benchmark 目标
  - CPU/GPU 优化与回归检测

### 参考资料

- **[15-examples.md](15-examples.md)** - 完整示例
  - 数据结构与反射
  - HTTP 服务器
  - 测试框架
  - 多态与反射结合
  - AI/ML 低精度计算

- **[16-comparison.md](16-comparison.md)** - 语言对比
  - vs C++：改进与保留
  - vs Rust：内存安全取舍
  - vs Zig：语法基线差异
  - 代码示例对比

## 核心特性速查

本索引只重复总纲中已经批准的规则。包名、构建 API 的成员访问形式，以及并发后端
选择仍属于未定稿的库设计；相关章节中的示例标有“待确认”，不能视为新增语言语法。

### `comp` 系统

```cpp
// 编译期函数
comp int32_t square(int32_t x) {
    return x * x;
}

// 泛型类型
comp type Vector(type T) { /* ... */ }
Vector<int32_t> v;  // <> 触发编译期调用 Vector(^^int32_t)

// 泛型函数
comp T max<type T>(T a, T b) {
    return a > b ? a : b;
}
```

### 反射

```cpp
comp {
    auto T = ^^User;
    for (auto field : nonstatic_data_members_of(T)) {
        println("{}: {}", name_of(field), size_of(type_of(field)));
    }
}
```

### 类型转换

```cpp
auto file = cast<File&>(writer);  // 借用，返回 Optional<reference_wrapper<File>>
if (is<File&>(writer)) { /* ... */ }
```

### 包管理

```toml
[package]
name = "myapp"
version = "1.0.0"

[dependencies]
http = "2.3.1"
json = "1.5.0"
```

```cpp
import http;
import json;

// 包导出的符号如何避免名称冲突：待确认。
Client client;
parse("{...}");
```

## 设计原则

1. **语法基线 = C++26**：只删不加
2. **一个问题一个解法**：统一机制，避免多种写法
3. **编译期优先**：能在编译期做的不拖到运行时
4. **零成本抽象**：不为不用的功能付出代价
5. **核心库与用户代码一致**：无特权，无魔法
6. **标准 C++ 语义**：`T&`/`T*`/特殊成员函数完全一致

## 快速开始

```bash
# 安装 ccc
curl -sSf https://install.ccc-lang.org | sh

# 创建新项目
ccc new myapp
cd myapp

# 构建并运行
ccc run
```

## 学习路径

### 1. 基础（1-2 天）
- 阅读 00-overview.md 和 01-modules.md
- 了解基本语法和模块系统
- 学习 02-types.md 中的类型系统

### 2. 核心（3-5 天）
- 深入 05-comp.md 理解编译期计算
- 学习 04-reflection.md 反射系统
- 掌握 03-memory.md 内存管理

### 3. 实战（1 周）
- 阅读 15-examples.md 完整示例
- 学习 10-packages.md 包管理
- 实现一个小项目

### 4. 高级（持续）
- 研究 13-compiler.md 编译器架构
- 对比 16-comparison.md 与其他语言
- 参与社区贡献

## 常见问题

### NCC 和 C++ 有什么区别？

NCC 是 C++26 的简化子集，删除了头文件、namespace、template 等冗余特性，用 `comp` 统一编译期计算，并内置包管理。

### NCC 和 Rust 有什么区别？

NCC 没有借用检查器，与 C++ 一样依靠程序员保证内存安全。适合需要 C++ 互操作性和虚表多态的项目。

### 为什么不用 Rust？

如果需要编译期内存安全保证，应该用 Rust。NCC 适合需要 C++ 生态和更简单学习曲线的场景。

### 能和 C++ 代码互操作吗？

NCC 直接支持显式 C ABI；C++ 库通过 `extern "C"` 包装层接入，详见
[12-interop.md](12-interop.md)。

## 社区与支持

- **GitHub**: https://github.com/ccc-lang/ccc
- **文档**: https://docs.ccc-lang.org
- **论坛**: https://discuss.ccc-lang.org
- **Discord**: https://discord.gg/ccc-lang

## 许可证

MIT License - 详见 [LICENSE](../../LICENSE)
