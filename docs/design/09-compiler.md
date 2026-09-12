# 八、编译器架构

## 第一阶段：C99 后端

```
Lexer
  ↓
Parser
  ↓
AST (Abstract Syntax Tree)
  ↓
HIR (High-level IR)
  ↓
Interface Resolution
  ↓
Type Checking
  ↓
Comp Execution (编译期求值 + 反射 + 泛型实例化)
  ↓
Code Generation (comp 函数生成具体类型/函数)
  ↓
RTTI Generation（仅参与多态分发的类型需要）
  ↓
C99 Code Generation
  ↓
clang/gcc
  ↓
Binary
```

**优势：**

1. **调试简单** - 生成的 C 代码可读
2. **跨平台免费** - clang 处理所有平台
3. **快速迭代** - 不用处理复杂的 LLVM API
4. **验证设计** - C 后端能验证语言设计的合理性

## 第二阶段：LLVM 后端（可选）

等语言稳定后，增加原生 LLVM 后端：

```
HIR
  ↓
MIR (Mid-level IR)
  ↓
LLVM IR
  ↓
LLVM
  ↓
Binary
```

保留 C 后端作为调试选项：

```bash
ccc build                  # 默认 LLVM 后端
ccc build --backend=c      # 生成 C 代码
ccc build --backend=llvm   # 直接生成 LLVM IR
ccc build --emit-c         # 输出 C 代码但不编译
ccc build --emit-llvm      # 输出 LLVM IR
```

## 编译流程细节

### Lexer（词法分析）

将源代码转换为 token 流：

```cpp
// 输入
Vector<int32_t> v{1, 2};

// Token 流
IDENTIFIER("Vector")
LESS
IDENTIFIER("int32_t")
GREATER
IDENTIFIER("v")
LBRACE
NUMBER(1)
COMMA
NUMBER(2)
RBRACE
SEMICOLON
```

### Parser（语法分析）

构建 AST：

```
VarDecl
  ├─ type: GenericType
  │   ├─ name: "Vector"
  │   └─ args: [int32_t]
  ├─ name: "v"
  └─ init: BraceInit
      ├─ 1
      └─ 2
```

### HIR（高层 IR）

简化 AST，展开语法糖：

```
- 聚合初始化 → 字段赋值序列
- for 循环 → while 循环
- 运算符重载 → 函数调用
```

### Comp Execution（编译期执行）

执行所有 `comp` 函数和块：

1. **泛型实例化**：`Vector<int32_t>` → 调用 `Vector(^^int32_t)` 生成具体类型
2. **反射查询**：`nonstatic_data_members_of(^^T)` → 返回字段列表
3. **代码生成**：`define_aggregate(...)` → 生成新类型定义

### Code Generation

将 HIR 转换为目标代码（C99 或 LLVM IR）：

```cpp
// HIR
VarDecl { type: Vector<int32_t>, name: "v" }

// C99 输出
struct Vector_int32_t {
    int32_t* data;
    size_t size;
    size_t capacity;
};
struct Vector_int32_t v;
Vector_int32_t_init(&v);
Vector_int32_t_push(&v, 1);
Vector_int32_t_push(&v, 2);
```

## 增量编译

### 模块级缓存

```
src/main.ccm    → .ccc/cache/main.o
src/utils.ccm   → .ccc/cache/utils.o
src/models.ccm  → .ccc/cache/models.o

只重新编译修改过的模块
```

### 依赖追踪

```
main.ccm imports:
  - utils
  - models

utils.ccm 修改 → 重新编译 utils.o 和 main.o
models.ccm 未修改 → 直接使用缓存的 models.o
```

## 并行编译

```
模块图：
  main
  ├─ utils
  └─ models
      └─ db

编译顺序：
1. 并行编译 db, utils（无依赖）
2. 编译 models（依赖 db）
3. 编译 main（依赖 utils, models）
```

## 错误报告

### 友好的错误信息

```
error[E0001]: type mismatch
  --> src/main.ccm:10:5
   |
10 |     Vector<String> v = get_numbers();
   |                        ^^^^^^^^^^^^^^ expected Vector<String>, found Vector<int32_t>
   |
help: consider converting the vector
   |
10 |     Vector<String> v = convert_to_strings(get_numbers());
   |                        ++++++++++++++++++++             +
```

### 编译期错误位置追踪

```
error: cannot call non-const method on const object
  --> src/main.ccm:15:7
   |
15 |     w.reset();
   |       ^^^^^ cannot call mutable method on const reference
   |
note: `w` is declared as const here
  --> src/main.ccm:14:25
   |
14 | void process(const Writer& w) {
   |              ^^^^^^^^^^^^^^^^
```
