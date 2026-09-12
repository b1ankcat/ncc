# 一、模块系统（替代头文件与 `namespace`）

C++26 已经有 `export module` / `import` 这套模块机制（C++20 引入）。删除
`#include` 头文件机制和 `namespace` 机制，只保留模块——这是"只删不加"的直接
体现：模块语法本身不是新东西，只是不再需要头文件和命名空间来配合它。

## 定义模块

```cpp
// math.ccm —— 定义一个模块
export module math;

export double square(double x) {
    return x * x;
}
```

## 使用模块

```cpp
// main.ccm —— 使用模块
import math;

int main() {
    double r = square(3.0);
}
```

## 核心库是内置全局符号

`String`、`Vector<T>`、`Optional<T>`、`unique_ptr<T>`、`shared_ptr<T>`、
`weak_ptr<T>`、`println`、反射用的 `Info`/`^^`/`[: :]` 等，全部像 `int32_t`、
`bool` 一样，是语言内置的一部分，直接使用，不挂在任何 `std` 或其他命名空间下，
也不需要额外 `import`。用户自己拆分的多文件项目之间用 `import 模块名;` 互相引用。

## 格式化：`println` / `format`

`println`/`format` 与 C++20 `std::format`（Rust `format!`/`fmt` 同级）能力
对齐：占位符支持位置参数、具名参数、宽度/精度/对齐/进制等标准格式规格，
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

核心库内置一个日志类型 `Logger`，能力对齐社区常见日志库（如 spdlog、
tracing），不需要 `import`：

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
