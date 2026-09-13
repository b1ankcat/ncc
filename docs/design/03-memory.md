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
// 直接拥有裸资源的封装类型，明确实现所有权操作
class File {
public:
    explicit File(const char* path) : fd_(open(path, O_RDONLY)) {
        if (fd_ < 0) {
            throw IOException("failed to open file");
        }
    }

    ~File() noexcept {
        if (fd_ >= 0) close(fd_);
    }

    File(const File&) = delete;
    File& operator=(const File&) = delete;

    File(File&& other) noexcept : fd_(other.fd_) {
        other.fd_ = -1;
    }

    File& operator=(File&& other) noexcept {
        if (this != &other) {
            if (fd_ >= 0) close(fd_);
            fd_ = other.fd_;
            other.fd_ = -1;
        }
        return *this;
    }

private:
    int fd_ = -1;
};

File f("test.txt");  // 调用构造函数，RAII 自动管理文件描述符
File moved = move(f); // 转移描述符，f 变为空句柄
// 离开作用域时自动调用析构函数
```

整数成员本身可拷贝，因此不能靠编译器识别 `fd_` 是独占资源。这里显式删除
拷贝、实现移动，并在移动后清空源句柄；移动赋值先释放目标已有资源，自移动
保持对象不变。析构和移动操作不抛异常。本例的自动清理不报告 `close` 失败；
需要处理关闭错误时，应通过有错误报告契约的显式操作完成。

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

## 标准拷贝与移动语义：组合类型优先 Rule of Zero

拷贝、移动和特殊成员函数生成完全遵循 C++，不引入默认移动、`Copyable`
接口或裸资源所有权推断。Rule of Zero 是类型设计方式，不是额外的语言规则：
组合已经正确管理资源的成员，使组合类型无需自行声明析构、拷贝和移动操作。

- **组合类型**：隐式操作的可用性取决于基类、成员和 C++ 的特殊成员函数规则。
  包含 `unique_ptr` 的普通组合类型不可拷贝，满足标准条件时可以隐式移动。
- **直接管理裸资源的类型**：作者必须正确实现或删除相关操作。上面的 `File`
  明确处理了析构、拷贝构造、拷贝赋值、移动构造与移动赋值，即 Rule of Five。
  没有转移协议的封装也可以显式禁止移动。
- **用户声明特殊成员函数的影响**：例如，声明析构函数会阻止隐式移动操作的
  生成，但不会自动禁止浅拷贝。需要移动时必须按标准规则显式提供。
- **拷贝的含义由类型决定**：值成员按成员规则复制，裸指针复制地址，
  `shared_ptr` 共享所有权；容器的深拷贝由容器自己实现，不是编译器的默认规则。

```cpp
Vector<int32_t> v1;
v1.push(1);
v1.push(2);

auto v2 = v1;              // 拷贝构造：Vector 对此元素类型提供深拷贝
println("{}", v1.size());  // OK，v1 未受影响

auto v3 = move(v1);        // 显式 move：v1 的内部资源被转移给 v3
// 移动后的 v1 可析构和重新赋值；其他操作及具体状态以容器契约为准
```

```cpp
struct Handle {
    unique_ptr<Data> data;   // 含有独占资源成员
};

unique_ptr<Data> p = make_unique<Data>(...);
Handle h1{move(p)};         // unique_ptr 不可拷贝，聚合初始化也必须显式 move
// Handle h2 = h1;          // 编译错误：unique_ptr 成员使隐式拷贝操作被删除
auto h2 = move(h1);         // OK：move
```

### move 与返回值

`move(x)` 将表达式转换为右值引用，本身不搬运资源。最终调用哪个构造或赋值
操作由标准重载规则决定；没有可用移动操作时可能调用拷贝，也可能编译失败。
对 const 对象使用 `move()` 不会去掉 const，也通常不能调用接收非 const
右值引用的移动操作。

```cpp
File open_config() {
    File file("config.txt");
    return file;  // 可进行 NRVO；未消除时按 C++ 返回规则移动
}

File open_log() {
    return File("app.log");  // 同类型纯右值直接构造结果对象
}
```

从具名左值转移资源通常需要显式 `move()`；右值、符合条件的返回局部对象和
拷贝消除继续遵循标准 C++。不要为了“移动必须显式”给 `return file` 添加
`move()`，这会妨碍 NRVO。移动后源对象的状态由类型契约规定，语言不提供统一的
“移动后禁止使用”检查。

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
（如 `File` 直接在构造时打开文件）：

```cpp
{
    File file("test.txt");
    // ...
} // file 自动关闭
```

**智能指针直接是标准 C++ 的 `unique_ptr` / `shared_ptr` / `weak_ptr`**，并保留
核心库的 `make_unique` / `make_shared` 工厂函数。三个指针类型和两个工厂都是
`comp` 函数：`unique_ptr<T>` 是编译期调用 `unique_ptr(^^T)` 生成类型，
`make_unique<T>(args...)` 是编译期调用 `make_unique(^^T)` 生成该类型专用的
普通运行时函数，再以 `args...` 调用它。生成过程不改变标准所有权、拷贝、移动
或异常语义；直接构造函数 + `new` 也仍然有效。

```cpp
// 唯一所有权：必须保留——递归结构体、多态基类指针分发天然需要一层堆间接层，
// move 语义管不到堆布局本身
unique_ptr<Data> p1 = make_unique<Data>(...);

// 共享所有权：引用计数，标准 shared_ptr 语义（线程安全的原子计数）
shared_ptr<Data> p2 = make_shared<Data>(...);

// 弱引用：标准 weak_ptr，用来打破 shared_ptr 的循环引用，避免泄漏
weak_ptr<Data> p3 = p2;

// unique_ptr 可转为 shared_ptr（标准转换），反向不行
shared_ptr<Data> p4 = make_unique<Data>(...);

// 直接构造仍然有效，与工厂等价，只是不合并分配
unique_ptr<Data> p5(new Data(...));
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

## 通用容器原语

核心库提供写好的 **comp 生成函数** `StorageOps`，用户自定义容器与核心容器
使用同一份公开实现。它接受元素类型和设备参数，生成提供静态成员函数的操作类型：

```cpp
comp type StorageOps(type T, Device device = Device::Cpu);

// 在容器的泛型实例化中使用
using Ops = StorageOps<T, Device::Cpu>;
```

`StorageOps` 在编译期选择元素操作、对齐要求和分配后端，返回一个具体类型；
`Ops::allocate_raw(n)`、`Ops::construct_at(...)` 等是生成后的普通函数。
运行时长度、地址和构造参数传给这些普通函数，不传给编译期生成器。因此
“原语由 comp 实现”不表示运行时堆分配或用户构造函数在编译器里执行。
操作类型的生成使用公开的类生成 API；原始分配依赖公开的分配器接口，不能
通过识别 Vector 名字获得特殊生命周期行为。

### 存储与对象分离

每次分配得到一个 `Storage` 记录，保存地址、槽位数及匹配释放所需的后端信息。
这只是存储记录，不隐式复制或释放其指向的资源；容器通过 RAII 负责唯一释放。
`size` 是已构造元素数，`capacity` 是槽位数，始终满足 `size <= capacity`。

| 生成的操作 | 实际行为 |
| --- | --- |
| `Ops::allocate_raw(n)` | 分配 n 个满足 `alignof(T)` 的槽位，不构造 T；先检查字节数乘法溢出与后端容量限制；零容量不分配 |
| `Ops::slot(storage, i)` | 返回未初始化槽位地址，仅作构造目标；要求 i 小于 capacity，不解引用未构造对象 |
| `Ops::element(storage, i)` | 返回已构造元素的引用，调用方保证该槽位存在活动对象 |
| `Ops::construct_at(slot, args...)` | 保留参数的值类别，调用 T 的实际构造函数；成功后开始对象寿命 |
| `Ops::destroy_at(slot)` | 对活动 T 调用析构，结束对象寿命 |
| `Ops::destroy_range(storage, count)` | 逆序销毁前 count 个已构造元素，不处理未构造槽位 |
| `Ops::release_raw(storage)` | 使用原分配器、对齐与设备信息释放原始存储；不代替元素析构 |
| `Ops::move_or_copy_construct(dst, src)` | 移动构造不抛异常时移动；否则若可拷贝则拷贝；仅可移动时调用可能抛异常的移动构造 |

这些操作不新增语言语法。批量构造通过循环调用 `construct_at` 并记录成功数量
实现；异常时只销毁成功构造的对象。不得把 `malloc` 得到的存储直接当成已构造
的任意 T，也不得以 `free` 代替析构。平凡类型的析构循环可以优化掉；对象字节
搬运必须满足类型和对象寿命规则，不能对资源类型无条件 memcpy。

### Vector 生命周期

- 空 Vector 不构造任何 T；`reserve(n)` 只增加容量，不增加 size，不要求默认构造。
- `Vector<T>(n)` 与默认追加元素的 `resize(n)` 仅在 T 可默认构造时可用；第 k
  个构造失败时清理此前已构造的元素，构造中的 Vector 不会泄漏存储。
- `emplace(args...)` 直接构造尾元素；`push(const T&)` 拷贝构造，`push(T&&)`
  按 C++ 重载规则构造。成功后才递增 size，不以赋值写入未初始化槽位。
- `clear()` 逆序析构活动元素、将 size 置零并保留容量；Vector 析构再释放存储。
- Vector 拷贝逐元素拷贝构造新存储，不共享所有权；仅在 T 可拷贝构造时提供。
  拷贝赋值先构造临时容器，成功后交换存储记录，不交换单个 T。
- 对同一可转移分配后端，Vector 移动接管存储记录并清空源记录，不逐元素移动。
  Vector 的移动不要求 T 可移动；扩容才要求 T 可移动构造或可拷贝构造。
- 析构与释放不抛异常。需要可能抛异常析构的 T 不满足此容器契约，实例化时报错；
  已声明 `noexcept` 的析构实际抛出时按 C++ 终止规则处理。
- 扩容成功后旧元素引用、指针和视图失效；容量未变的尾部添加不使已有元素失效。

### 扩容的实际流程

以下为生成的 CPU Vector 方法体示意：`storage_`、`size_` 是容器成员，`Ops`
是上述 comp 生成的具体操作类型；没有运行时 comp 调用。

```cpp
void reserve(size_t requested) {
    if (requested <= storage_.capacity) return;

    auto next = Ops::allocate_raw(requested);
    size_t constructed = 0;
    try {
        for (; constructed < size_; ++constructed) {
            Ops::move_or_copy_construct(
                Ops::slot(next, constructed),
                Ops::element(storage_, constructed));
        }
    } catch (...) {
        Ops::destroy_range(next, constructed);
        Ops::release_raw(next);
        throw;
    }

    Ops::destroy_range(storage_, size_);
    Ops::release_raw(storage_);
    storage_ = next;  // 转移记录；此后 next 不再承担释放责任
}
```

| 情况 | 失败后的保证 |
| --- | --- |
| 分配失败或容量溢出 | 抛出内存不足或长度异常，原容器未被修改 |
| T 移动不抛异常 | 分配后迁移不会因 T 的移动失败，提交前无其他可抛操作 |
| T 移动可能抛异常但可拷贝 | 迁移选择拷贝；失败只清理新存储，旧内容保持不变（前提是 T 的拷贝不修改源值） |
| T 仅可移动且移动可能抛异常 | 清理新存储，保留旧存储和 size；部分旧元素可能已移动，元素状态取决于 T 的异常契约，不能保证原内容不变 |

`push/emplace` 扩容时先在新存储尾槽构造追加元素，再迁移旧元素，最后提交。
失败清理时分别销毁尾槽与成功迁移的前缀，不能把两者误当连续的完整区间。
这同时避免 `push(v[0])` 在迁移旧数据后读取失效的源引用。显式移动追加参数
（例如 `push(move(v[0]))`）和可能修改实参的用户构造函数仍可改变源值；不承诺
回滚用户副作用。移动可能抛异常的仅移动 T 同样不能获得普遍的强异常保证。

GPU 存储使用相同的“原始存储/活动对象”区分，但只支持目标后端能构造、析构
和传输的类型；不自动把 CPU 文件、锁或任意 C++ 对象迁移到 GPU。具体边界见
[GPU 章节](09-gpu.md)。

## 内存管理总结

| 机制 | 语法 | 说明 |
|------|------|------|
| 栈分配 | `T obj{...};` | 作用域结束自动析构（RAII） |
| 聚合初始化 | `T{a, b}` | 按字段顺序初始化 |
| 构造函数 | `T(args)` | 资源获取（RAII） |
| 拷贝 | `auto x = y;` | 按类型的拷贝构造规则，可能深拷贝、共享或被禁止 |
| 移动 | `auto x = move(y);` | 以右值参与重载选择；是否转移资源取决于类型 |
| 引用 | `T& x = ...;` | 无额外检查，等同标准 C++ |
| 原始指针 | `T* x = ...;` | 无检查，程序员自己负责 |
| 唯一所有权 | `make_unique<T>(...)` | 不可拷贝，只能 `move`；等价写法 `unique_ptr<T>(new T(...))` |
| 共享所有权 | `make_shared<T>(...)` | 原子引用计数；循环引用需用 `weak_ptr` 打破 |
| 弱引用 | `weak_ptr<T> w = sp;` | 不计入所有权，使用前须提升为 `shared_ptr` |
