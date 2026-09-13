# 十五、完整示例

## 定义数据结构

```cpp
export module app;

struct User {
    uint64_t id;
    String name;
    String email;
    uint32_t age;

    bool is_adult() const {
        return age >= 18;
    }
};
```

User 的所有字段都是可拷贝的（`uint64_t`/`String`/`uint32_t`），且没有声明
影响隐式拷贝生成的特殊成员函数，因此其隐式拷贝操作可用。这是组合类型遵循
Rule of Zero 的例子；String 自己负责其资源拷贝契约，不需要额外的 `comp` 声明。

## 自动生成代码

```cpp
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
```

## 使用

```cpp
import app;

int main() {
    User user{1, "Alice", "alice@example.com", 25};   // 聚合初始化

    println("{}", debug_string(user));  // 同一份反射代码，运行时正常调用

    if (user.is_adult()) {
        println("User is an adult");
    }
}
```

## HTTP 服务器示例

```cpp
export module server;

import http;
import json;

// 定义路由处理器
struct Handler {
    Response handle_user(const Request& req) {
        uint64_t user_id = req.params["id"].as_u64();
        
        // 从数据库查询
        Optional<User> user = db.find_user(user_id);
        
        if (!user) {
            return Response{
                .status = 404,
                .body = object({
                    {"error", "User not found"}
                }).to_string()
            };
        }
        
        // 序列化为 JSON
        return Response{
            .status = 200,
            .body = serialize_json(user.value())
        };
    }
};

// 自动生成 JSON 序列化
comp String serialize_json<type T>(const T& v) {
    String json = "{";
    bool first = true;
    for (auto field : nonstatic_data_members_of(^^T)) {
        if (!first) json += ",";
        first = false;
        json += format("\"{}\": ", name_of(field));
        json += to_json_value(v.[:field:]);
    }
    json += "}";
    return json;
}

int main() {
    Server server;
    Handler handler;
    
    // 注册路由
    server.get("/users/:id", [&](auto& req) {
        return handler.handle_user(req);
    });
    
    println("Server listening on http://localhost:8080");
    server.listen(8080);
}
```

## 反射驱动的测试框架

```cpp
export module test_framework;

// 发现与生成在同一个 comp 上下文内完成：
// comp 块之间不共享可变状态，跨块传递需要 comp 常量或函数返回值
comp Vector<Info> discover_tests() {
    Vector<Info> found;
    for (auto func : functions_of(^^current_module)) {
        if (name_of(func).starts_with("test_")) {
            found.push(func);
        }
    }
    return found;
}

// 运行所有测试：函数体由编译期展开生成
void run_all_tests() {
    comp {
        for (auto func : discover_tests()) {
            // 生成的是运行时代码：编译期只决定生成哪些语句，
            // 循环本身在编译期展开，println 和调用在运行时执行
            println("Running: {}", name_of(func));   // name_of 是编译期常量
            [:func:]();                              // 直接调用，不经过运行时 invoke
        }
    }
}

// 用户测试代码
void test_addition() {
    assert(1 + 1 == 2);
}

void test_string_concat() {
    String s = "Hello";
    s += " World";
    assert(s == "Hello World");
}

int main() {
    run_all_tests();  // 自动运行 test_addition 和 test_string_concat
}
```

## 多态与反射结合

```cpp
// 定义接口
class Shape {
public:
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

class Circle : public Shape {
public:
    double radius;
    
    double area() const override {
        return 3.14159 * radius * radius;
    }
};

class Rectangle : public Shape {
public:
    double width;
    double height;
    
    double area() const override {
        return width * height;
    }
};

// 注册多态类型（编译期）
comp {
    for (auto T : types_deriving_from(^^Shape)) {
        register_dynamic_type(T);
    }
}

// 运行时类型查询
void print_shape_info(const Shape& s) {
    Info type = dynamic_type_of(s);
    
    println("Shape type: {}", name_of(type));
    println("Area: {}", s.area());
    
    // 遍历字段（运行时）：运行时 Info 用 fields_of，
    // nonstatic_data_members_of 只用于编译期生成场景
    for (auto field : fields_of(type)) {
        println("  {}: {}", name_of(field), size_of(type_of(field)));
    }
}

int main() {
    Circle c{.radius = 5.0};
    Rectangle r{.width = 4.0, .height = 3.0};
    
    print_shape_info(c);  // 输出: Shape type: Circle, Area: 78.54
    print_shape_info(r);  // 输出: Shape type: Rectangle, Area: 12.0
}
```

## AI/ML 低精度计算示例

```cpp
import ml;

// 使用低精度类型加速推理
struct Model {
    Vector<bfloat16_t> weights;  // BF16 权重
    Vector<float8_e4m3_t> activations;  // FP8 激活
    
    Vector<float> forward(const Vector<float>& input) {
        // 逐元素转换为低精度：cast<T> 作用于单个值，
        // 容器的元素类型变换用显式的 map 适配器
        Vector<bfloat16_t> input_bf16 =
            input | map([](float v) { return cast<bfloat16_t>(v).value(); })
                  | collect<Vector<bfloat16_t>>();

        // 低精度矩阵乘法（硬件加速）
        Vector<bfloat16_t> output_bf16 = matmul(input_bf16, weights);

        // 转回 float32：bfloat16 → float 无损，转换不会失败
        return output_bf16 | map([](bfloat16_t v) { return cast<float>(v).value(); })
                           | collect<Vector<float>>();
    }
};

int main() {
    Model model = load_model("model.ckpt");
    
    Vector<float> input{1.0f, 2.0f, 3.0f};
    Vector<float> output = model.forward(input);
    
    println("Output: {}", output);
}
```

