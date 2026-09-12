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

## Lambda 类型的反射

lambda 类型可以被反射，用于分析捕获列表和签名：

```cpp
// 反射 lambda 捕获列表
comp auto captures = captures_of(^^Lambda);  // 返回 Span<Info>

for (auto capture : captures) {
    auto capture_type = type_of(capture);
    auto capture_mode = capture_mode_of(capture);  // CaptureMode::ByValue 或 ByReference
    println("Captured: {} as {}", name_of(capture_type), capture_mode);
}

// 检查捕获的类型是否满足条件
comp bool is_gpu_safe(type Lambda) {
    for (auto capture : captures_of(^^Lambda)) {
        auto T = type_of(capture);
        if (!is_gpu_accessible(T)) {
            return false;
        }
    }
    return true;
}
```

**Lambda 反射 API：**

- `captures_of(^^Lambda)` → `Span<Info>` - 返回所有捕获的变量
- `type_of(capture)` → `Info` - 捕获变量的类型
- `capture_mode_of(capture)` → `CaptureMode` - 捕获模式（值或引用）
- `name_of(capture)` → `String` - 捕获变量的原始名称

**CaptureMode 枚举：**

```cpp
enum class CaptureMode {
    ByValue,      // [x] 值捕获
    ByReference,  // [&x] 引用捕获
};
```

## 完整类定义：`define_class` API

`define_aggregate` 只能定义数据成员的聚合类型，不支持成员函数、构造/析构函数。
为了支持完整的泛型类定义，引入 `define_class` API：

```cpp
comp type Vector(type T, Device device = Device::Cpu) {
    if (device == Device::Cpu) {
        return [: define_class("Vector_Cpu", {
            // 数据成员
            .data_members = {
                data_member_spec(^^T*, {.name = "data_"}),
                data_member_spec(^^size_t, {.name = "size_"}),
                data_member_spec(^^size_t, {.name = "capacity_"}),
            },
            // 构造函数
            .constructors = {
                // 默认构造
                constructor_spec({}, [](auto& self) {
                    self.data_ = nullptr;
                    self.size_ = 0;
                    self.capacity_ = 0;
                }),
                // 带大小构造
                constructor_spec({^^size_t}, [](auto& self, size_t n) {
                    self.data_ = (T*)malloc(n * sizeof(T));
                    self.size_ = n;
                    self.capacity_ = n;
                }),
            },
            // 析构函数
            .destructor = [](auto& self) {
                if (self.data_) {
                    free(self.data_);
                }
            },
            // 成员函数
            .methods = {
                method_spec("size", {}, ^^size_t, [](const auto& self) {
                    return self.size_;
                }),
                method_spec("push", {^^T}, ^^void, [](auto& self, T value) {
                    // 扩容逻辑...
                    self.data_[self.size_++] = value;
                }),
                method_spec("to_device", {}, ^^Vector<T, Device::Gpu>, [](const auto& self) {
                    // 分配 GPU 内存并拷贝
                    Vector<T, Device::Gpu> result(self.size_);
                    gpu_memcpy(result.device_ptr_, self.data_, self.size_ * sizeof(T));
                    return result;
                }),
            },
        }) :];
    } else {
        return [: define_class("Vector_Gpu", {
            .data_members = {
                data_member_spec(^^void*, {.name = "device_ptr_"}),
                data_member_spec(^^size_t, {.name = "size_"}),
            },
            .constructors = {
                constructor_spec({^^size_t}, [](auto& self, size_t n) {
                    self.device_ptr_ = gpu_malloc(n * sizeof(T));
                    self.size_ = n;
                }),
            },
            .destructor = [](auto& self) {
                if (self.device_ptr_) {
                    gpu_free(self.device_ptr_);
                }
            },
            .methods = {
                method_spec("size", {}, ^^size_t, [](const auto& self) {
                    return self.size_;
                }),
                method_spec("to_host", {}, ^^Vector<T, Device::Cpu>, [](const auto& self) {
                    Vector<T, Device::Cpu> result(self.size_);
                    gpu_memcpy_to_host(result.data_, self.device_ptr_, self.size_ * sizeof(T));
                    return result;
                }),
            },
        }) :];
    }
}
```

**`define_class` 参数说明：**

- `.data_members` - 数据成员列表（同 `define_aggregate`）
- `.constructors` - 构造函数列表，每个包含参数类型和实现 lambda
- `.destructor` - 析构函数 lambda
- `.methods` - 成员函数列表，包含名称、参数、返回类型和实现

**关键机制：**

1. **延迟实例化**：`to_device()` 方法引用 `Vector<T, Device::Gpu>` 类型时，
   编译器不会立即实例化，而是记录依赖关系，在所有类型声明完成后再实例化。

2. **类型前向声明**：编译器为每个 `Vector<T, Device>` 组合生成前向声明，
   允许在方法定义中引用尚未完全实例化的类型。

3. **循环依赖检测**：如果类型之间存在真正的循环定义（不通过指针/引用），
   编译器报错。

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
