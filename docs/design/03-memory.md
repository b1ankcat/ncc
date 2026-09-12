# 三、内存管理

## 对象构造：聚合初始化 + 构造函数

NCC 支持两种对象构造方式，与标准 C++ 完全一致：

### 聚合初始化（用户代码优先）

```cpp
struct Point {
    double x;
    double y;
};

Point p{1.0, 2.0};  // 聚合初始化
```

### 构造函数（RAII 资源获取）

核心库类型和用户类型都可以定义构造函数，用于资源获取和初始化逻辑：

```cpp
// 用户自定义构造函数
struct File {
    int fd;
    
    File(const char* path) : fd(open(path, O_RDONLY)) {
        if (fd < 0) {
            throw Error("failed to open file");
        }
    }
    
    ~File() {
        if (fd >= 0) close(fd);
    }
};

File f("test.txt");  // 调用构造函数，RAII 自动管理文件描述符
// 离开作用域时自动调用析构函数
```

**RAII（Resource Acquisition Is Initialization）**：构造函数获取资源，
析构函数释放资源。这是标准 C++ 的核心模式，栈上对象离开作用域时自动析构。

**核心库类型使用相同机制**：

```cpp
// Vector 内部有构造函数管理堆内存
Vector<int32_t> v;
v.push(1);
// 离开作用域自动释放内存

// String 同样
String s = "Hello";
// 离开作用域自动释放

// File（核心库）
File file("data.txt");
// 离开作用域自动关闭文件
```

**关键点**：核心库类型（`Vector`、`String`、`File`、`unique_ptr` 等）与
用户类型使用**完全相同的语法**，没有特殊例外。它们都是通过构造函数/析构函数
实现 RAII。

## 默认拷贝，move 显式化：标准 C++ 的 Rule of Zero

不引入任何自造的"默认 move"或 `Copyable` 接口。行为完全是原生 C++ 语义：
一个类型只要它的每个成员都能拷贝，它就自动能拷贝（编译器隐式生成拷贝构造/
赋值）；只要有一个成员禁止拷贝（比如内部含有 `unique_ptr`），它就自动隐式
禁止拷贝，只能 move。用户不需要写任何声明。

```cpp
Vector<int32_t> v1;
v1.push(1);
v1.push(2);

auto v2 = v1;              // 拷贝：Vector 本身可拷贝，赋值默认是深拷贝
println("{}", v1.size());  // OK，v1 未受影响

auto v3 = move(v1);        // 显式 move：v1 的内部资源被转移给 v3
// v1.size();               // v1 处于合法但不应再使用的"已移动"状态
```

```cpp
struct Handle {
    unique_ptr<Data> data;   // 含有独占资源成员
};

Handle h1{unique_ptr<Data>(new Data(...))};
// Handle h2 = h1;          // 编译错误：Handle 因为含有 unique_ptr 成员，
                            // 隐式禁用了拷贝——这是标准 C++ 的自动推导结果
auto h2 = move(h1);         // OK：move
```

一句话总结：**能拷贝就默认拷贝，不能拷贝的类型由编译器根据成员自动推导，
需要转移所有权时显式调用 `move()`——这就是标准 C++ 的行为，没有任何附加
规则。**

## `comp` 常量

`comp` 统一取代 `constexpr` / `consteval`，表达"编译期计算"。`const` 保留
其运行时不可变语义，和 C++ 完全一致：

```cpp
comp double PI = 3.14159;     // 编译期常量
const int32_t cfg = get_config();  // 运行时常量（值在运行时确定，但不可修改）

void process(const Writer& w) {    // const 引用：表达"不修改参数"
    // w.reset();  // 编译错误：const 对象不能调非 const 方法
}
```

`comp` 和 `const` 解决不同层次的问题：
- `comp`：编译期求值，运行时可能根本不存在这个变量（内联成字面量）
- `const`：运行时只读，值在运行时才确定，但禁止修改

宏、反射、泛型实例化、编译期代码生成等能力全部统一在 `comp` 里完成（见后续
章节），不再需要 `constexpr`/`consteval` 这套单独机制。

## RAII 和智能指针：沿用标准 C++ API

栈上自动析构是标准 C++ 已有的 RAII 语义；核心库类型可以有自己的真实构造函数
（如 `File` 直接在构造时打开文件，类似标准库 `ifstream`）：

```cpp
{
    File file("test.txt");
    // ...
} // file 自动关闭
```

**智能指针直接是标准 C++ 的 `unique_ptr` / `shared_ptr` / `weak_ptr`**，一个
符号都不改名。唯一删除的是 `make_unique`/`make_shared` 这套工厂函数——既然
构造函数本身已经够用，工厂函数就是多余的第二种写法：

```cpp
// 唯一所有权：必须保留——递归结构体、多态基类指针分发天然需要一层堆间接层，
// move 语义管不到堆布局本身
unique_ptr<Data> p1(new Data(...));

// 共享所有权：引用计数，标准 shared_ptr 语义（线程安全的原子计数）
shared_ptr<Data> p2(new Data(...));

// 弱引用：标准 weak_ptr，用来打破 shared_ptr 的循环引用，避免泄漏
weak_ptr<Data> p3 = p2;
```

## 引用 `T&` / 指针 `T*`：标准 C++ 语义，没有额外检查

`T&` 就是普通的 C++ 引用，`T*` 就是普通的裸指针，两者读写都不受限，编译器
**不做任何额外的借用检查**——安全性完全由程序员负责，和标准 C++ 完全一致，
没有比原生 C++ 多，也没有比原生 C++ 少：

```cpp
void move_point(Point& p, double dx, double dy) {
    p.x += dx;
    p.y += dy;
}

Point pt{0.0, 0.0};
move_point(pt, 1.0, 2.0);  // OK：引用传参，函数内部直接修改
```

引用和指针的行为与标准 C++ 完全一致，包括所有的风险（悬垂指针、空指针、
数据竞争等）。不尝试通过新的编译器检查解决这些问题——安全性由程序员和
RAII/智能指针保证。

## 内存管理总结

| 机制 | 语法 | 说明 |
|------|------|------|
| 栈分配 | `T obj{...};` | 作用域结束自动析构（RAII） |
| 聚合初始化 | `T{a, b}` | 按字段顺序初始化 |
| 构造函数 | `T(args)` | 资源获取（RAII） |
| 拷贝 | `auto x = y;` | 默认深拷贝（如果类型支持） |
| 移动 | `auto x = move(y);` | 显式转移所有权 |
| `T& x = ...;` | 引用 | 无额外检查，等同标准 C++ |
| `T* x = ...;` | 原始指针 | 无检查，程序员自己负责 |
| `unique_ptr<T>(new T(...))` | 唯一所有权 | 编译期保证只有一个所有者 |
| `shared_ptr<T>(new T(...))` | 共享所有权 | 引用计数；循环引用需用 `weak_ptr` 打破 |
