# 十、完整示例

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

User 的所有字段都是可拷贝的（`uint64_t`/`String`/`uint32_t`），所以 User
本身自动可拷贝——这是标准 C++ 的 Rule of Zero 自动推导出的结果，不需要任何
`comp` 声明。

## 自动生成代码

```cpp
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
comp fn serialize_json<type T>(v: const T&) -> String {
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

// 测试注册表（编译期构建）
comp {
    Vector<Info> test_functions;
    
    // 自动发现所有 test_ 开头的函数
    for (auto func : functions_of(^^current_module)) {
        if (name_of(func).starts_with("test_")) {
            test_functions.push(func);
        }
    }
}

// 运行所有测试
void run_all_tests() {
    comp {
        for (auto func : test_functions) {
            println("Running: {}", name_of(func));
            // 调用测试函数
            [: invoke(func) :];
        }
    }
}

// 用户测试代码
void test_addition() {
    assert(1 + 1 == 2);
}

void test_string_concat() {
    String s = "Hello" + " World";
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
    
    // 遍历字段（运行时）
    for (auto field : nonstatic_data_members_of(type)) {
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
        // 转换输入为低精度
        Vector<bfloat16_t> input_bf16 = cast<Vector<bfloat16_t>>(input).value();
        
        // 低精度矩阵乘法（硬件加速）
        Vector<bfloat16_t> output_bf16 = matmul(input_bf16, weights);
        
        // 转回 float32
        return cast<Vector<float>>(output_bf16).value();
    }
};

int main() {
    Model model = load_model("model.ckpt");
    
    Vector<float> input{1.0f, 2.0f, 3.0f};
    Vector<float> output = model.forward(input);
    
    println("Output: {}", output);
}
```

