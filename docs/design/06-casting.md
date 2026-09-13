# 六、类型转换：只有一个函数

## `cast<T>(value)`

只暴露一个转换函数 `cast<T>(value)`，取代标准 C++ 的四个 cast 关键字。
返回结果的形态由目标类型决定，避免把借用对象错误地复制成拥有值：

- 目标是值类型：返回 `Optional<T>`，执行定义明确的值转换；
- 目标是 `T*`：返回 `Optional<T*>`，成功时只返回指针，不转移所有权；
- 目标是 `T&`：返回 `Optional<reference_wrapper<T>>`，成功时借用原对象；
- 目标是基类/派生类指针或引用：仅对多态对象执行运行时类型检查。

```cpp
// 借用多态对象，不复制或转移 File
auto file = cast<File&>(writer);

// 配合 if 初始化语句
if (auto f = cast<File&>(writer)) {
    f->get().write("Hello");
}

// 指针目标只借用对象，失败返回空 Optional
auto f = cast<File*>(writer_ptr);

// 数值转换：溢出或不可表示时失败
auto small = cast<int8_t>(large_integer);

// 去掉 const 不由 cast 自动完成；必须从一开始取得可写指针
const Point* const_ptr = get_point();
// cast<Point*>(const_ptr) -> 失败
```

## `is<T>(value)`：`cast<T>` 的检查专用简写

检查"这个值现在是不是某个类型"是 `cast<T>` 的常见用法，`is<T>(value)` 是
`cast<T>(value).has_value()` 的简写，不是第二套机制，只是省掉手写
`.has_value()`：

```cpp
if (is<File&>(writer)) {
    // 只检查类型，不复制、不移动、不改变 writer
}
```

`is<T>(value)` 等价于 `cast<T>(value).has_value()`，并保证不会调用拷贝构造、
移动构造或析构对象。对多态引用/指针，检查的是运行时具体类型；对普通值类型，
检查的是是否存在定义的值转换。

### 数值转换规则

整数转换要求目标类型能够表示源值，否则失败。浮点转整数要求值为有限值且在
目标范围内，并按当前舍入模式取整；NaN、无穷大和越界值失败。整数转浮点和
浮点转浮点在目标精度不足时按 IEEE 754 舍入，但不因精度损失失败。布尔、枚举
和指针转换必须有明确的目标类型规则；不存在规则的转换直接在编译期拒绝。

`cast` 不检查裸指针指向对象的寿命、别名合法性或底层分配器一致性，也不自动
去除 `const`。这些仍遵循 C++ 指针语义；需要可写访问时，调用方必须持有可写
指针或引用。

### 指针转换规则

| 源 → 目标 | 结果 |
| --- | --- |
| 派生类 ↔ 基类指针/引用（多态） | 运行时检查，失败返回空 `Optional` |
| 任意 `T*` → `void*` | 总是成功 |
| `void*` → `T*` | 成功；正确性由调用方负责（C 互操作的不透明句柄） |
| 字节类型之间（`char`/`uint8_t`/`int8_t`/`std::byte`）的指针 | 成功；字节级重解释是明确允许的 |
| 其他不相关类型的指针互转 | **编译期拒绝** |
| 去除 `const` | **编译期拒绝** |

字节类型指针之间的转换单独允许，因为它是 C 互操作的必要操作（把
`const char*` 字面量交给接受 `const uint8_t*` 的接口）。其他类型双关
（如 `float*` → `uint32_t*`）一律拒绝——需要重解释位模式时使用核心库的
`bit_cast<T>(value)`，它要求两个类型大小相同且可平凡复制，语义明确且不产生
别名问题。

## `concept`/`requires`：用 `comp bool` 函数替代

`cast<T>` 解决的是"运行时一个值是不是某个类型"这个问题。泛型参数约束解决的
是"泛型函数的类型参数必须具备什么形状"——这个检查发生在**编译期**，针对的是
**类型参数本身**，不是某个运行时的值。

**`concept` 关键字被删除**，改用 `comp bool` 函数 + `<>` 调用语法统一表达：

```cpp
// concept 定义改用 comp bool 函数
comp bool Writable(type T) {
    return requires(T t) {
        { t.write(declval<const uint8_t*>(), size_t{}) } -> same_as<size_t>;
    };
}

// 使用：<> 触发编译期调用 Writable(^^T)，返回 bool
comp void process<type T>(T& w) requires Writable<T> {
    w.write(cast<const uint8_t*>("Hello").value(), 5);
}
```

`Writable<T>` 本质是编译期调用 `Writable(^^T)`，返回 `bool` 用于 `requires` 子句。
`requires` 本身是 C++20 已有语法，保留不违反"只删不加"。

## 总结

- `cast<T>(value)` - 唯一的转换入口；值、指针和引用目标分别返回对应的 Optional，取代
  `static_cast`/`dynamic_cast`/`reinterpret_cast`/`const_cast`
- `is<T>(value)` - `cast<T>(value).has_value()` 的简写，唯一的检查入口
- `bit_cast<T>(value)` - 同大小可平凡复制类型间的位模式重解释；不是 `cast` 的
  重载，因为它不做任何检查也不会失败，与 `cast` 的"可失败转换"语义不同
- `comp bool` 函数 + `<>` - 泛型参数的结构性约束（编译期），取代 `concept` 关键字

`cast` 与 `bit_cast` 的分工：前者转换**值**（`cast<int8_t>(300)` 失败，因为
300 不可表示），后者重解释**位**（`bit_cast<uint32_t>(1.0f)` 得到
`0x3F800000`）。两者都不去除 `const`，也都不检查指针寿命。
