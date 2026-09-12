# 六、类型转换：只有一个函数

## `cast<T>(value)`

只暴露一个转换函数，返回 `Optional<T>`，取代标准 C++ 的
`static_cast`/`dynamic_cast`/`reinterpret_cast`/`const_cast` 四个关键字——
数值收窄转换、多态向下转换、指针类型双关、去 `const`，本质都是"给我一个
新视角看这个值，看不看得通"，统一一个函数：

```cpp
// cast<T>(value) 返回 Optional<T>，<> 语法触发编译期调用 cast(^^T, value)
auto file = cast<File>(writer);

// 配合 if 初始化语句
if (auto f = cast<File>(writer)) {
    f->write("Hello");
}

// 提供默认值
auto f = cast<File>(writer).value_or(default_file);

// 强制转换（自己承担风险）：失败则终止
auto f = cast<File>(writer).value();

// 指针类型双关（取代 reinterpret_cast），检查即"按目标类型看是否合法"
auto bytes = cast<const uint8_t*>("Hello").value();

// 去掉/加上 const（取代 const_cast）
auto* mutable_p = cast<Point*>(const_ptr).value();
```

## `is<T>(value)`：`cast<T>` 的检查专用简写

检查"这个值现在是不是某个类型"是 `cast<T>` 的常见用法，`is<T>(value)` 是
`cast<T>(value).has_value()` 的简写，不是第二套机制，只是省掉手写
`.has_value()`：

```cpp
if (is<File>(writer)) {
    // writer 可以转换成 File，等价于 cast<File>(writer).has_value()
}
```

## `concept`/`requires`：用 `comp bool` 函数替代

`cast<T>` 解决的是"运行时一个值是不是某个类型"这个问题。泛型参数约束解决的
是"泛型函数的类型参数必须具备什么形状"——这个检查发生在**编译期**，针对的是
**类型参数本身**，不是某个运行时的值。

**`concept` 关键字被删除**，改用 `comp bool` 函数 + `<>` 调用语法统一表达：

```cpp
// concept 定义改用 comp bool 函数
comp bool Writable(type T) {
    return requires(T t) {
        { t.write(declval<uint8_t*>(), size_t{}) } -> same_as<size_t>;
    };
}

// 使用：<> 触发编译期调用 Writable(^^T)，返回 bool
comp fn process<type T>(w: T&) requires Writable<T> {
    w.write(cast<const uint8_t*>("Hello").value(), 5);
}
```

`Writable<T>` 本质是编译期调用 `Writable(^^T)`，返回 `bool` 用于 `requires` 子句。
`requires` 本身是 C++20 已有语法，保留不违反"只删不加"。

## 总结

- `cast<T>(value)` - 唯一的转换入口，返回 `Optional<T>`，取代
  `static_cast`/`dynamic_cast`/`reinterpret_cast`/`const_cast`
- `is<T>(value)` - `cast<T>(value).has_value()` 的简写，唯一的检查入口
- `comp bool` 函数 + `<>` - 泛型参数的结构性约束（编译期），取代 `concept` 关键字
