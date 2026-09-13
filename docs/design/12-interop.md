# 十二、C 互操作

NCC 与 C 语言完全兼容，可以无缝调用 C 库和暴露接口给 C 代码。

**设计原则：**
- **只与 C 互操作**：不直接调用 C++ 库（避免命名空间、模板等复杂性）
- **零成本抽象**：C 函数调用无额外运行时包装开销
- **显式 ABI**：C 函数声明必须放在 `extern "C"` 中；`import libc` 等内置绑定
  由编译器生成这些显式声明，不把未定义函数自动猜成 C ABI

## 调用 C 函数

### 直接声明 C 函数

```cpp
// 声明 C 标准库函数
extern "C" {
    c_int open(const char* path, c_int flags);
    void close(c_int fd);
    void* malloc(size_t size);
    void free(void* ptr);
    c_int printf(const char* fmt, ...);
}

// 使用
int main() {
    void* ptr = malloc(1024);
    printf("Allocated at %p\n", ptr);
    free(ptr);
}
```

**编译器行为：**
- 只把 `extern "C"` 声明视为 C ABI 外部符号
- 按目标平台的 C calling convention 生成调用
- 链接时按声明的 C 符号解析（从 libc 等）

### 使用 import 批量导入

```cpp
// 编译器内置的 C 标准库绑定
import libc;

int main() {
    int fd = open("/path/to/file", O_RDONLY);
    if (fd < 0) {
        perror("open failed");
        return 1;
    }
    
    char buffer[1024];
    ssize_t n = read(fd, buffer, sizeof(buffer));
    close(fd);
}
```

**内置的 C 绑定：**
- `import libc` - C 标准库（stdio.h, stdlib.h, string.h 等）
- `import posix` - POSIX API（unistd.h, fcntl.h, sys/socket.h 等）
- `import pthread` - POSIX 线程库

这些导入由编译器根据目标平台头文件和 ABI 生成显式的 `extern "C"` 声明；
导入不会改变 NCC 函数的默认 C++ ABI，也不会自动导入任意第三方头文件。

## C 类型映射

### 标量类型

```cpp
// C 类型 → NCC 类型（按目标 ABI 映射）
char          → char
signed char   → int8_t
unsigned char → uint8_t
short         → c_short
int           → c_int
long          → c_long
long long     → c_long_long
size_t        → size_t
ssize_t       → ssize_t

float         → float
double        → double

void*         → void*
```

`c_short`、`c_int`、`c_long` 和 `c_long_long` 是核心库提供的 ABI 精确类型；
它们的宽度和对齐由目标三元组决定。不能用机器位数替代 ABI 判断：Windows x64
的 `long` 仍为 32 位，Linux x86_64 的 `long` 通常为 64 位。需要固定宽度时，
C API 本身应使用 `int32_t`/`int64_t` 等固定宽度类型。

### 变参与回调

变参函数遵循 C 的默认参数提升规则，格式字符串不会触发额外的隐式类型检查：

```cpp
extern "C" {
    c_int printf(const char* format, ...);
    using Callback = void(*)(c_int value);
}
```

C 回调的函数指针必须显式声明在 C linkage 块中；NCC 闭包只有在无捕获且具有
兼容签名时才能转换为该回调类型。带捕获闭包需要通过不透明上下文指针和显式
销毁函数传递。

### 指针类型

```cpp
// C 字符串
const char* c_str = "Hello, C";

// NCC String → C 字符串
String ncc_str = "Hello, NCC";
const char* c_str = ncc_str.c_str();  // 零拷贝（内部 UTF-8）

// C 字符串 → NCC String
const char* c_str = getenv("PATH");
String ncc_str = String::from_c_str(c_str);  // 深拷贝
```

### 结构体类型

```cpp
// C 结构体定义
struct CStruct {
    int32_t x;
    int32_t y;
    char data[16];
};

// NCC 可以直接使用（布局兼容）
CStruct c_data;
c_data.x = 10;
c_data.y = 20;

// 传递给 C 函数
extern "C" void process_c_struct(CStruct* s);
process_c_struct(&c_data);
```

### 不透明指针（Opaque Pointer）

```cpp
// C 库常用不透明指针（如 FILE*）
void* file_ptr = fopen("file.txt", "r");
// ... 使用
fclose(file_ptr);

// 或者用 typedef 增强类型安全
using FILE = void;
FILE* file = fopen("file.txt", "r");
```

## 调用 C 库示例

### 示例 1：文件 I/O

```cpp
import libc;

void read_file(const String& path) {
    int fd = open(path.c_str(), O_RDONLY);
    if (fd < 0) {
        throw IOException("Failed to open file");
    }
    
    char buffer[4096];
    ssize_t n;
    while ((n = read(fd, buffer, sizeof(buffer))) > 0) {
        process_data(buffer, n);
    }
    
    close(fd);
}
```

### 示例 2：网络编程

```cpp
import posix;

void tcp_server(uint16_t port) {
    int sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        throw NetworkException("Failed to create socket");
    }
    
    sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(port);
    addr.sin_addr.s_addr = INADDR_ANY;
    
    if (bind(sock, cast<sockaddr*>(&addr).value(), sizeof(addr)) < 0) {
        close(sock);
        throw NetworkException("Failed to bind");
    }
    
    listen(sock, 128);
    
    while (true) {
        int client = accept(sock, nullptr, nullptr);
        if (client >= 0) {
            handle_client(client);
            close(client);
        }
    }
}
```

### 示例 3：调用第三方 C 库（SQLite）

```cpp
// 声明 SQLite API
using sqlite3 = void;
using sqlite3_stmt = void;

extern "C" {
    int32_t sqlite3_open(const char* filename, sqlite3** ppDb);
    int32_t sqlite3_close(sqlite3* db);
    int32_t sqlite3_prepare_v2(sqlite3* db, const char* sql, int32_t len,
                               sqlite3_stmt** stmt, const char** tail);
    int32_t sqlite3_step(sqlite3_stmt* stmt);
    int32_t sqlite3_finalize(sqlite3_stmt* stmt);
    const char* sqlite3_column_text(sqlite3_stmt* stmt, int32_t col);
}

// 使用
class Database {
    sqlite3* db_;
    
public:
    Database(const String& path) {
        if (sqlite3_open(path.c_str(), &db_) != 0) {
            throw DatabaseException("Failed to open database");
        }
    }
    
    ~Database() {
        sqlite3_close(db_);
    }

    // 此示例不提供连接所有权转移
    Database(const Database&) = delete;
    Database& operator=(const Database&) = delete;
    Database(Database&&) = delete;
    Database& operator=(Database&&) = delete;
    
    Vector<String> query(const String& sql) {
        sqlite3_stmt* stmt;
        sqlite3_prepare_v2(db_, sql.c_str(), -1, &stmt, nullptr);
        
        Vector<String> results;
        while (sqlite3_step(stmt) == 100) {  // SQLITE_ROW
            const char* text = sqlite3_column_text(stmt, 0);
            results.push(String::from_c_str(text));
        }
        
        sqlite3_finalize(stmt);
        return results;
    }
};
```

## 暴露 NCC 接口给 C

### 使用 export extern "C"

```cpp
// mylib.ncc
export module mylib;

// 不透明句柄（隐藏内部实现）
struct Context {
    Vector<uint8_t> buffer;
    HashMap<String, int32_t> cache;
};

// 导出 C API
export extern "C" {
    // 创建上下文
    void* mylib_create() {
        return new Context();
    }
    
    // 销毁上下文
    void mylib_destroy(void* ctx) {
        delete cast<Context*>(ctx).value();
    }
    
    // 处理数据
    int32_t mylib_process(void* ctx, const uint8_t* data, size_t len) {
        auto* c = cast<Context*>(ctx).value();
        
        // 使用 NCC 的内部实现
        c->buffer.resize(len);
        memcpy(c->buffer.data(), data, len);
        
        return process_internal(c->buffer);
    }
    
    // 查询结果
    int32_t mylib_get_result(void* ctx, const char* key) {
        auto* c = cast<Context*>(ctx).value();
        String k = String::from_c_str(key);
        
        if (auto val = c->cache.get(k)) {
            return *val;
        }
        return -1;
    }
}
```

### 生成 C 头文件

```bash
# 编译时自动生成 C 头文件
ccc build --generate-c-header

# 生成 mylib.h
```

**生成的头文件：**

```c
// mylib.h - 自动生成，不要手动编辑
#ifndef MYLIB_H
#define MYLIB_H

#include <stddef.h>
#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

// 创建上下文
void* mylib_create(void);

// 销毁上下文
void mylib_destroy(void* ctx);

// 处理数据
int32_t mylib_process(void* ctx, const uint8_t* data, size_t len);

// 查询结果
int32_t mylib_get_result(void* ctx, const char* key);

#ifdef __cplusplus
}
#endif

#endif // MYLIB_H
```

### C 代码使用 NCC 库

```c
// main.c
#include "mylib.h"
#include <stdio.h>

int main() {
    // 创建 NCC 库的上下文
    void* ctx = mylib_create();
    
    // 调用 NCC 库
    uint8_t data[] = {1, 2, 3, 4, 5};
    int32_t result = mylib_process(ctx, data, 5);
    printf("Process result: %d\n", result);
    
    // 查询结果
    int32_t value = mylib_get_result(ctx, "some_key");
    printf("Get result: %d\n", value);
    
    // 销毁上下文
    mylib_destroy(ctx);
    
    return 0;
}
```

## 类型转换辅助函数

### NCC 核心类型 ↔ C 类型

```cpp
// String ↔ C 字符串
class String {
public:
    // NCC String → C 字符串（零拷贝）
    const char* c_str() const;
    
    // C 字符串 → NCC String（深拷贝）
    static String from_c_str(const char* str);
    static String from_c_str(const char* str, size_t len);
};

// Vector ↔ C 数组
class Vector<type T> {
public:
    // 获取原始指针（与 C 数组兼容）
    T* data();
    const T* data() const;
    size_t size() const;
};

// 使用示例
Vector<int32_t> vec = {1, 2, 3, 4, 5};

// 传递给 C 函数
extern "C" void c_function(const int32_t* arr, size_t len);
c_function(vec.data(), vec.size());
```

## ABI 兼容性

### 函数调用约定

```cpp
// NCC 函数默认使用 C++ name mangling
void ncc_function(int32_t x);  // 符号：_Z12ncc_functioni

// export extern "C" 使用 C ABI
export extern "C" void c_function(int32_t x);  // 符号：c_function
```

### 结构体布局

```cpp
// ✅ 兼容：POD 类型
struct Point {
    float x, y;
};
// NCC 和 C 布局完全相同

// ✅ 兼容：简单结构体
struct User {
    uint64_t id;
    char name[64];
};

// ❌ 不兼容：NCC 核心类型
struct Data {
    String text;       // sizeof(String) = 24 (COW + SSO)
    Vector<int> nums;  // sizeof(Vector) = 24
};
// 不能直接传递给 C，需要通过 C API 包装
```

### 内存管理边界

```cpp
// ✅ 正确：在同一侧分配和释放
export extern "C" {
    char* allocate_string() {
        return strdup("Hello");  // C 分配
    }
    
    void free_string(char* str) {
        free(str);  // C 释放
    }
}

// ❌ 错误：跨边界分配/释放
export extern "C" {
    void* bad_alloc() {
        return new int32_t[100];  // NCC 分配（C++ new）
    }
}
// C 代码调用 free(bad_alloc()) → 未定义行为！

// ✓ 正确：使用统一的分配器
export extern "C" {
    void* safe_alloc(size_t size) {
        return malloc(size);  // C 分配器
    }
    
    void safe_free(void* ptr) {
        free(ptr);  // C 释放器
    }
}
```

## 链接 C 库

### package.toml 配置

```toml
[package]
name = "myapp"
version = "1.0.0"

# C 库通过 kind = "c-library" 声明，由 pkg-config 解析头文件与链接参数
[dependencies]
sqlite3 = { version = "3.0", kind = "c-library" }
openssl = { version = "1.1", kind = "c-library" }
```

声明 C 库只有两种途径，不提供第三种：

| 途径 | 适用场景 |
| --- | --- |
| `[dependencies]` 中 `kind = "c-library"` | 系统或第三方库能被 pkg-config 找到 |
| `build.ncc` 中 `link_lib()` / `link_search()` | 手工指定路径、随项目分发的 vendor 库、需要条件判断 |

`libc`、`libm`、`pthread` 这类始终存在的系统库由目标 ABI 隐含提供，不需要声明；
需要显式控制链接方式时在 `build.ncc` 中调用 `link_lib`。

### build.ncc 中链接

```cpp
import build;

comp {
    // 链接系统库
    link_lib("pthread");
    link_lib("m");
    
    // 链接第三方库
    if (target_os() == "linux") {
        link_lib("dl");
    }
    
    // 添加库搜索路径
    link_search("/usr/local/lib");
    link_search(project_root() + "/vendor/lib");
}
```

## 最佳实践

### 1. 使用 RAII 包装 C 资源

```cpp
// ✓ 好：RAII 包装
class File {
    int fd_ = -1;
    
public:
    explicit File(const char* path) : fd_(open(path, O_RDONLY)) {
        if (fd_ < 0) {
            throw IOException("Failed to open file");
        }
    }
    
    ~File() noexcept {
        if (fd_ >= 0) close(fd_);
    }
    
    // 禁止拷贝
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
};

// ✗ 差：手动管理
int fd = open("file.txt", O_RDONLY);
// ... 使用
close(fd);  // 容易忘记
```

拷贝与移动按 C++ 规则由资源封装类型定义；裸句柄不会自动禁用拷贝。
移动后源 File 为空句柄，移动赋值释放目标已有句柄；清理期间的关闭错误处理
边界与 [内存管理](03-memory.md) 中的 File 示例一致。

### 2. 不透明句柄模式

```cpp
// NCC 库内部
struct Context {
    Vector<Data> internal_data;
    // ... 复杂的 NCC 类型
};

// 暴露给 C 的 API
export extern "C" {
    void* create_context() {
        return new Context();
    }
    
    void destroy_context(void* ctx) {
        delete cast<Context*>(ctx).value();
    }
    
    // C 代码只操作不透明指针，不访问内部结构
}
```

### 3. 错误处理转换

```cpp
// NCC 内部使用异常
int process_internal(const Vector<uint8_t>& data) {
    if (data.empty()) {
        throw InvalidArgumentException("Empty data");
    }
    // ...
}

// C API 转换为错误码
export extern "C" {
    int32_t process(const uint8_t* data, size_t len) {
        try {
            Vector<uint8_t> vec(data, data + len);
            return process_internal(vec);
        } catch (const InvalidArgumentException&) {
            return -1;  // EINVAL
        } catch (...) {
            return -2;  // 通用错误
        }
    }
}
```

## 与 C++ 互操作（不推荐）

NCC 设计上只与 C 互操作，**不直接调用 C++ 库**。如果必须使用 C++ 库：

1. **编写 C 包装层**：在 C++ 中编写 C API 包装
2. **通过 C ABI 调用**：NCC 通过 C 包装层间接使用 C++ 库

**示例：使用 Boost（通过 C 包装）**

```cpp
// boost_wrapper.cpp (C++ 代码)
#include <boost/asio.hpp>

extern "C" {
    void* boost_io_context_create() {
        return new boost::asio::io_context();
    }
    
    void boost_io_context_destroy(void* ctx) {
        delete static_cast<boost::asio::io_context*>(ctx);
    }
    
    void boost_io_context_run(void* ctx) {
        static_cast<boost::asio::io_context*>(ctx)->run();
    }
}
```

```cpp
// myapp.ncc (NCC 代码)
extern "C" {
    void* boost_io_context_create();
    void boost_io_context_destroy(void* ctx);
    void boost_io_context_run(void* ctx);
}

class IoContext {
    void* ctx_;
    
public:
    IoContext() : ctx_(boost_io_context_create()) {}
    ~IoContext() { boost_io_context_destroy(ctx_); }

    // 此示例不提供上下文所有权转移
    IoContext(const IoContext&) = delete;
    IoContext& operator=(const IoContext&) = delete;
    IoContext(IoContext&&) = delete;
    IoContext& operator=(IoContext&&) = delete;

    void run() { boost_io_context_run(ctx_); }
};
```

**不推荐理由：**
- 需要维护额外的 C 包装层
- 增加编译复杂度
- 失去类型安全（通过 void* 传递）

**推荐方案：**
- 优先使用原生 NCC 库
- 或者使用纯 C 的替代品

## 下一步

- 查看 [11-build-system.md](11-build-system.md) 了解如何链接 C 库
- 查看 [03-memory.md](03-memory.md) 了解内存管理
- 查看 [14-performance.md](14-performance.md) 了解性能优化
