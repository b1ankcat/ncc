# 四、反射：统一的一套 API

反射不区分"静态反射"和"动态反射"两套 API。原因：反射查询（`^^T`、
`members_of`、`name_of` 等）本身是 `comp`（consteval）操作，只要类型在
**当前上下文里是静态已知的**——包括具体类型、模板里的类型参数、tagged enum
的某个变体——查询就在编译期完成并展开成普通代码。之后这段普通代码在编译期
被 `comp` 块直接执行，或者留在函数体里在运行时被调用，用的都是**同一份**
函数，不是"编译期版本"和"运行时版本"两个重载。

## 核心能力

```cpp
struct User {
    uint64_t id;
    String name;
    String email;
};

comp {
    auto T = ^^User;

    println("Type: {}", name_of(T));
    println("Size: {} bytes", size_of(T));
    println("Align: {}", alignment_of(T));

    for (auto field : nonstatic_data_members_of(T)) {
        println("  {}: {} at offset {}",
            name_of(field),
            name_of(type_of(field)),
            offset_of(field));
    }
}
```

`Info` 是反射信息的统一载体（对应 C++26 `std::meta::info`，去掉 `std::meta::`
前缀作为内置类型），不需要文档里自造 `TypeInfo`/`FieldInfo`/`MethodInfo`
结构体——查字段用 `nonstatic_data_members_of`，查方法用 `members_of` 过滤
`is_function`，查基类用 `bases_of`，全部走同一套函数。

## 同一份代码，既能在编译期跑也能在运行时跑

```cpp
// 这个函数没有标成 comp，但只要调用处的 T 是静态已知类型
// （比如作为泛型参数传入），reflect_dump 内部的反射查询依然在编译期展开
comp fn reflect_dump<type T>(v: const T&) {
    for (auto field : nonstatic_data_members_of(^^T)) {
        println("{}: {}", name_of(field), v.[:field:]);
    }
}

// 编译期使用：在 comp 块里直接调用
comp {
    User u{1, "Alice", "alice@example.com"};
    reflect_dump(u);
}

// 运行时使用：main 里正常调用，字段遍历早已在实例化时展开成
// 具体的 println 调用序列，运行时执行的是展开后的普通代码
int main() {
    User u{1, "Alice", "alice@example.com"};
    reflect_dump(u);   // 同一个 reflect_dump，同一套反射 API
}
```

## 编译期代码生成示例

用 splice `[: ... :]` 把反射结果直接拼回代码，而不是拼字符串再注入：

```cpp
comp fn encode_json<type T>(v: const T&) -> String {
    String json = "{";
    bool first = true;
    for (auto field : nonstatic_data_members_of(^^T)) {
        if (!first) json += ",";
        first = false;
        json += "\"" + String(name_of(field)) + "\": " + encode_value(v.[:field:]);
    }
    json += "}";
    return json;
}
```

`v.[:field:]` 是反射的成员 splice 语法，直接访问反射得到的数据成员，取代
自造的字符串拼接 + 代码注入。这份 `encode_json` 同样不区分编译期/运行时。

## 多态对象的运行时类型查询

反射查询默认要求类型静态已知。唯一的例外是通过基类指针/引用拿到的多态
对象——它的具体派生类只有运行时才能确定。这**不需要一套独立的"动态反射"
API**，只需要在同一套反射 API 里增加一个桥接函数 `dynamic_type_of`：它接收
一个多态引用，返回和 `^^ConcreteType` 完全一样的 `Info`，之后就能像处理
静态已知类型一样，用 `name_of`/`members_of`/`nonstatic_data_members_of`
等**同一套函数**继续查询：

```cpp
class Shape {
public:
    virtual ~Shape() = default;
};

class Circle : public Shape { double radius; };
class Rect : public Shape { double w; double h; };

comp {
    // 只需要为参与多态分发的类型登记，登记后才能在运行时反解出 Info
    for (auto T : types_deriving_from(^^Shape)) {
        register_dynamic_type(T);
    }
}

void print_kind(const Shape& s) {
    Info T = dynamic_type_of(s);   // 唯一"运行时才能确定"的一步

    println("Concrete type: {}", name_of(T));   // 之后完全是同一套反射 API
    for (auto field : nonstatic_data_members_of(T)) {
        println("  field: {}", name_of(field));
    }
}
```

**实现原理：** `register_dynamic_type` 在编译期把每个具体子类的 `Info` 存进
一张挂在虚表旁边的表里；`dynamic_type_of` 在运行时通过虚函数取到这张表里
对应的 `Info`。用户完全不需要知道这张表的存在——从 `dynamic_type_of` 拿到
`Info` 之后，后续处理和处理一个编译期静态类型没有任何区别，是**同一套 API
的一个入口，不是另一套体系**。
