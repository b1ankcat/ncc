# 四、反射：统一的一套 API

反射分为**类型反射**和**表达式反射**两个维度，分别使用 `TypeInfo` 和 `ExprInfo` 
句柄。`comp` 只在编译期执行：它读取静态类型信息和表达式结构，生成只读运行时
描述符、字段访问器和方法桥接函数。程序运行时通过 `dynamic_type_of` 获取描述符
并查询对象的类型、字段和方法；运行时不会执行 `comp`，也不会生成新类型。

**`^^` 运算符的重载语义**：
- 应用于类型 → `TypeInfo`：`^^int32_t`、`^^User`
- 应用于表达式 → `ExprInfo`：`^^(a + b)`、`^^(obj.field)`

## 类型反射（TypeInfo）

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

`TypeInfo` 是类型反射信息的统一载体，可以是编译期反射值或运行时类型描述符的
轻量句柄。`fields_of`、`methods_of`、`bases_of`、`name_of` 和 `type_of` 在
两种上下文中使用同一组名称；`nonstatic_data_members_of` 只用于编译期生成场景。

运行时通过 `get_field`、`set_field` 和 `invoke` 访问对象；函数检查类型、权限、
参数数量、参数类型和可写性，失败返回空值或抛出标准异常，不能产生未定义行为。

```cpp
// object 必须是已注册的多态类型；T 由调用处静态推导，函数取其动态类型
comp TypeInfo dynamic_type_of<type T>(const T& object);

Vector<TypeInfo> fields_of(TypeInfo type);
Vector<TypeInfo> methods_of(TypeInfo type);
Optional<Any> get_field(const void* object, TypeInfo field);
void set_field(void* object, TypeInfo field, Any value);
Optional<Any> invoke(const void* object, TypeInfo method, Vector<Any> arguments);
```

### `Any`：运行时值容器

```cpp
class Any {
public:
    Any();                             // 空值
    TypeInfo type() const;             // 持有值的类型；空值返回空 TypeInfo

    bool has_value() const;
    explicit operator bool() const;
};

// 取出值：类型不匹配时返回空 Optional，不抛异常、不复制失败的对象
comp Optional<T> cast<type T>(const Any& value);
```

`^^T` 得到编译期 `TypeInfo`，可用于 `splice` 和代码生成；`dynamic_type_of` 得到
运行时 `TypeInfo`，可用于查询和访问，但不能用于 `splice`、定义新类型或生成新方法。
多态类型必须在编译期注册；未注册类型返回空结果或抛出明确异常。

## 编译期生成运行时访问器

```cpp
comp void reflect_dump<type T>(const T& v) {
    for (auto field : nonstatic_data_members_of(^^T)) {
        println("{}: {}", name_of(field), v.[:field:]);
    }
}

// 编译期使用：在 comp 块里直接调用
comp {
    User u{1, "Alice", "alice@example.com"};
    reflect_dump(u);
}

// main 里调用编译期生成的普通运行时代码
int main() {
    User u{1, "Alice", "alice@example.com"};
    reflect_dump(u);
}
```

## 编译期代码生成示例

用 splice `[: ... :]` 把反射结果直接拼回代码，而不是拼字符串再注入：

```cpp
comp String encode_json<type T>(const T& v) {
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

`v.[:field:]` 是仅限编译期的成员 splice 语法，直接生成静态成员访问。运行时
动态对象应使用 `get_field` 或 `invoke`，不能把运行时 `Info` 作为 splice 操作数。

## Lambda 类型的反射

lambda 类型可以被反射，用于分析捕获列表和签名：

```cpp
// 反射 lambda 捕获列表
comp auto captures = captures_of(^^Lambda);  // 返回 Vector<TypeInfo>

for (auto capture : captures) {
    auto capture_type = type_of(capture);
    auto capture_mode = capture_mode_of(capture);  // CaptureMode::ByValue 或 ByReference
    println("Captured: {} as {}", name_of(capture_type), capture_mode);
}

// 检查捕获的类型是否满足条件
comp bool is_gpu_safe(type Lambda) {
    for (auto capture : captures_of(^^Lambda)) {
        auto T = type_of(capture);
        if (!gpu_capture_safe(T, CaptureMode::ByValue)) {
            return false;
        }
    }
    return true;
}
```

**Lambda 反射 API：**

- `captures_of(^^Lambda)` → `Vector<TypeInfo>` - 返回所有捕获的变量
- `type_of(capture)` → `TypeInfo` - 捕获变量的类型
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

类生成只决定布局、成员签名及操作选择。生成的资源类调用核心库公开的
`StorageOps<T, Device>`，不再直接将 malloc/free 与元素构造/析构混为一体。
这些原语的生成器是核心库写好的 comp 函数，用户自定义容器同样可以使用。

```cpp
// comp 生成操作类型，随后生成的类可在运行时调用其中的普通函数
comp type StorageOps(type T, Device device = Device::Cpu);
```

Vector 类生成必须包含以下实际操作，生命周期的唯一规范见
[通用容器原语](03-memory.md#通用容器原语)：

| 生成部分 | 实现要求 |
| --- | --- |
| 数据成员 | 原始存储记录、已构造数量 size；capacity 属于存储记录 |
| 空构造 | 空存储、size 为零，不要求 T 可默认构造 |
| 大小构造 | 分配后逐个构造；记录成功数量，失败逆序销毁前缀并释放存储 |
| 拷贝构造/赋值 | 仅在 T 可拷贝构造时生成；复制新存储，成功后提交 |
| 移动构造/赋值 | 对可转移后端接管记录，清空源记录；赋值先清理目标旧资源 |
| reserve | 调用分配、move-or-copy 构造、失败清理和最终提交；不改变 size |
| push/emplace | 在未初始化尾槽构造，成功后更新 size；扩容时先处理尾元素的别名问题 |
| clear/析构 | 逆序析构活动对象，按需保留或释放原始存储 |
| GPU 迁移 | 验证元素设备能力，等待构造/传输完成后公开结果；禁止任意 T 的字节拷贝 |

`define_class` 接受 `.data_members`、`.constructors`、`.destructor` 与
`.methods` 中的公开描述，由相同的生成机制绑定上述实现。没有隐式为裸指针
补齐深拷贝的特例；生成器必须按 T 的能力提供或删除特殊成员函数。
普通用户容器可以通过相同描述生成同样的类，不依赖 Vector 的内置名字。

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

通过基类指针/引用拿到的多态对象，其具体类型只能在运行时确定。使用同一套
`TypeInfo` 查询函数读取描述符；运行时字段和方法访问使用 `get_field`、`set_field`
和 `invoke`。这不是第二套反射语法，也不会把运行时类型重新变成编译期类型：

```cpp
class Shape {
public:
    virtual ~Shape() = default;
};

class Circle : public Shape { double radius; };
class Rect : public Shape { double w; double h; };

comp {
    // 只需要为参与多态分发的类型登记，登记后才能在运行时反解出 TypeInfo
    for (auto T : types_deriving_from(^^Shape)) {
        register_dynamic_type(T);
    }
}

void print_kind(const Shape& s) {
    TypeInfo T = dynamic_type_of(s);   // 唯一"运行时才能确定"的一步

    println("Concrete type: {}", name_of(T));   // 之后完全是同一套反射 API
    for (auto field : fields_of(T)) {
        auto value = get_field(&s, field);
        println("  field: {}", name_of(field));
    }
}
```

**实现原理：** `register_dynamic_type` 在编译期把具体子类的描述符存进类型表；
`dynamic_type_of` 在运行时通过虚表关联信息取得描述符。描述符包含由编译期生成
的字段访问器和方法桥接函数，因此 `invoke` 无需动态生成代码。动态库卸载前必须
保证没有悬空 `TypeInfo` 句柄。

## 表达式反射（ExprInfo）

表达式反射提供**类型检查后的 AST 级访问**，保留源代码的语法结构，同时附加语义信息
（类型、重载决议结果）。这用于增强诊断、DSL 嵌入和元编程。

### 基本用法

```cpp
comp {
    int32_t a = 10, b = 20;
    
    ExprInfo expr = ^^(a + b * 2);
    
    println("Expression kind: {}", expr_kind(expr));     // ExprKind::BinaryOp
    println("Type: {}", name_of(type_of(expr)));         // "int32_t"
    println("Operator: {}", binary_operator(expr));      // Operator::Plus
    println("Stringified: {}", stringify(expr));         // "a + b * 2"
}
```

### ExprKind 枚举

```cpp
enum class ExprKind {
    // 基础表达式
    Literal,        // 字面量：42, "hello", true
    Variable,       // 变量引用：x, y
    
    // 一元运算
    UnaryOp,        // 一元运算：-x, !b, *p, &x
    
    // 二元运算
    BinaryOp,       // 二元运算：a + b, x > y, p && q
    
    // 调用
    Call,           // 函数调用：func(a, b)
    
    // 成员访问
    MemberAccess,   // 成员访问：obj.field, ptr->field
    Subscript,      // 下标：arr[i]
    
    // 其他
    Cast,           // 转换：cast<T>(value)
    Ternary,        // 三元：cond ? a : b
    Lambda,         // Lambda：[](int x) { return x + 1; }
};
```

### Operator 枚举

```cpp
enum class Operator {
    // 算术
    Plus, Minus, Multiply, Divide, Modulo,
    
    // 比较
    Equal, NotEqual, Less, LessEqual, Greater, GreaterEqual,
    
    // 逻辑
    LogicalAnd, LogicalOr, LogicalNot,
    
    // 位运算
    BitwiseAnd, BitwiseOr, BitwiseXor, BitwiseNot,
    LeftShift, RightShift,
    
    // 其他
    AddressOf, Dereference, UnaryPlus, UnaryMinus,
};
```

### 表达式查询 API

**通用查询**：

```cpp
ExprKind expr_kind(ExprInfo expr);        // 表达式种类
TypeInfo type_of(ExprInfo expr);          // 表达式的类型
String stringify(ExprInfo expr);          // 转换为源代码字符串（用于诊断）
```

**字面量** (`ExprKind::Literal`)：

```cpp
comp Optional<Any> literal_value(ExprInfo expr);  // 获取字面量的值
```

**变量** (`ExprKind::Variable`)：

```cpp
String variable_name(ExprInfo expr);              // 变量名
```

**一元运算** (`ExprKind::UnaryOp`)：

```cpp
Operator unary_operator(ExprInfo expr);           // 运算符
ExprInfo unary_operand(ExprInfo expr);            // 操作数
```

**二元运算** (`ExprKind::BinaryOp`)：

```cpp
Operator binary_operator(ExprInfo expr);          // 运算符
ExprInfo binary_lhs(ExprInfo expr);               // 左操作数
ExprInfo binary_rhs(ExprInfo expr);               // 右操作数
```

**调用** (`ExprKind::Call`)：

```cpp
ExprInfo callee(ExprInfo expr);                   // 被调用的函数表达式
Vector<ExprInfo> arguments(ExprInfo expr);        // 实参列表
Optional<FunctionInfo> resolved_callee(ExprInfo expr);  // 重载决议后的函数
```

**成员访问** (`ExprKind::MemberAccess`)：

```cpp
ExprInfo member_object(ExprInfo expr);            // 对象表达式
String member_name(ExprInfo expr);                // 成员名
Optional<FieldInfo> resolved_member(ExprInfo expr);  // 解析后的字段/方法
```

**下标** (`ExprKind::Subscript`)：

```cpp
ExprInfo subscript_array(ExprInfo expr);          // 数组表达式
ExprInfo subscript_index(ExprInfo expr);          // 索引表达式
```

**三元运算** (`ExprKind::Ternary`)：

```cpp
ExprInfo ternary_condition(ExprInfo expr);        // 条件
ExprInfo ternary_true_branch(ExprInfo expr);      // 真分支
ExprInfo ternary_false_branch(ExprInfo expr);     // 假分支
```

**转换** (`ExprKind::Cast`)：

```cpp
TypeInfo cast_target_type(ExprInfo expr);         // 目标类型
ExprInfo cast_operand(ExprInfo expr);             // 被转换的表达式
```

**编译期求值**（仅限编译期常量表达式）：

```cpp
comp Optional<Any> eval(ExprInfo expr);           // 求值表达式，返回结果
```

### 语义信息查询

表达式反射不仅提供 AST 结构，还附加类型检查后的语义信息：

```cpp
// 表达式属性
bool is_lvalue(ExprInfo expr);                    // 是否是左值
bool is_constexpr(ExprInfo expr);                 // 是否是编译期常量
bool is_noexcept(ExprInfo expr);                  // 是否不抛异常

// 重载决议结果（调用表达式）
Optional<FunctionInfo> resolved_callee(ExprInfo expr);

// 成员解析结果（成员访问表达式）
Optional<FieldInfo> resolved_member(ExprInfo expr);
```

### 使用示例

**示例 1：增强的断言诊断**

```cpp
comp void check_impl(ExprInfo expr, SourceLocation loc = SourceLocation::current()) {
    auto result = eval(expr);
    
    if (result.has_value() && cast<bool>(*result).value_or(false)) {
        return;  // 检查通过
    }
    
    // 失败：生成详细诊断
    String message = format("Check failed at {}:{}\n  {}\n",
        loc.file(), loc.line(), stringify(expr));
    
    // 递归打印子表达式的值
    print_subexpr_values(expr, message);
    
    compile_error(message);
}

comp void print_subexpr_values(ExprInfo expr, String& out) {
    if (expr_kind(expr) == ExprKind::BinaryOp) {
        auto lhs = binary_lhs(expr);
        auto rhs = binary_rhs(expr);
        
        if (auto lhs_val = eval(lhs)) {
            out += format("    {} = {}\n", stringify(lhs), *lhs_val);
        }
        if (auto rhs_val = eval(rhs)) {
            out += format("    {} = {}\n", stringify(rhs), *rhs_val);
        }
    }
}

// 使用
comp {
    int32_t x = -5;
    check_impl(^^(x > 0));
    // 编译错误：
    // Check failed at test.ncc:42
    //   x > 0
    //     x = -5
    //     0 = 0
}
```

**示例 2：SQL DSL 生成**

```cpp
comp String generate_where_clause(ExprInfo filter) {
    if (expr_kind(filter) == ExprKind::BinaryOp) {
        auto op = binary_operator(filter);
        auto lhs = binary_lhs(filter);
        auto rhs = binary_rhs(filter);
        
        String op_str;
        if (op == Operator::Equal) op_str = "=";
        else if (op == Operator::Less) op_str = "<";
        else if (op == Operator::Greater) op_str = ">";
        else if (op == Operator::LogicalAnd) {
            return format("({} AND {})",
                generate_where_clause(lhs),
                generate_where_clause(rhs));
        }
        
        return format("{} {} {}",
            translate_expr(lhs),
            op_str,
            translate_expr(rhs));
    }
    
    if (expr_kind(filter) == ExprKind::MemberAccess) {
        return member_name(filter);
    }
    
    if (expr_kind(filter) == ExprKind::Literal) {
        auto val = literal_value(filter);
        return format("'{}'", *val);
    }
    
    compile_error("Unsupported expression in SQL filter");
}

// 使用
comp String sql = generate_where_clause(^^(user.age > 18 && user.active));
// 结果：(age > '18' AND active = 'true')
```

**示例 3：表达式复杂度分析**

```cpp
comp int32_t expr_complexity(ExprInfo expr) {
    if (expr_kind(expr) == ExprKind::Literal || 
        expr_kind(expr) == ExprKind::Variable) {
        return 1;
    }
    
    if (expr_kind(expr) == ExprKind::BinaryOp) {
        return 1 + expr_complexity(binary_lhs(expr)) 
                 + expr_complexity(binary_rhs(expr));
    }
    
    if (expr_kind(expr) == ExprKind::Call) {
        int32_t cost = 10;  // 函数调用基础成本
        for (auto arg : arguments(expr)) {
            cost += expr_complexity(arg);
        }
        return cost;
    }
    
    return 1;
}

comp {
    auto complexity = expr_complexity(^^(a + b * func(c, d)));
    println("Expression complexity: {}", complexity);  // 14
}
```

### 限制

1. **编译期上下文**：表达式反射只能在 `comp` 上下文中使用
2. **求值限制**：`eval()` 只能求值编译期常量表达式
3. **AST 稳定性**：表达式结构保证在语义等价的转换下稳定，但编译器优化可能简化 AST
4. **运行时不可用**：`ExprInfo` 不能在运行时创建或查询（不同于 `TypeInfo` 的 `dynamic_type_of`）
