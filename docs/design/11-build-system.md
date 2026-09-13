# 十一、构建系统

## 核心主旨：声明式配置 + 受限编译期构建脚本

NCC 的构建系统只有一个规范执行模型：`package.toml` 提供声明式配置，
可选的 `build.ncc` 在独立的受限构建上下文中执行一次。普通源文件、编译期
代码生成和外部工具都通过这个模型接入；不再存在第二种 `main()` 构建入口或
stdout 指令协议。

**规范文件与扩展名：**

- NCC 源文件统一使用 `.ncc`。
- 项目配置使用 `package.toml`。
- 构建脚本使用 `build.ncc`，必须通过 `import build` 使用受限 API。
- 构建脚本不生成可执行文件，不定义 `main()`；它返回结构化构建描述，由编译器直接消费。


## 外部输入与可重现性

普通 `comp` 只处理源码和编译期常量，不直接访问文件、网络、环境或外部进程。
这些操作只能在 `build.ncc` 的受限上下文中通过 `build` API 完成。所有影响结果
的输入必须显式登记，未登记的输入访问使构建失败。

```cpp
import build;

comp {
    auto schema = input_file("schema/user.json");
    auto protoc = tool("protoc", "25.1");

    custom_command({
        .inputs = {schema},
        .outputs = {out_dir() + "/user.ncc"},
        .program = protoc,
        .args = {"--ncc_out=" + out_dir(), schema},
    });
}
```

文件、目录 glob、环境变量、网络 URL 及校验和、外部工具版本和生成输出都会
进入构建图与缓存键。脚本只能写入 `out_dir()`；并行命令不得写入重叠输出。
相同的输入、工具、目标和配置必须产生相同的结构化构建描述。

**设计目标：**
1. **可并行**：依赖图分析 + 多核编译
2. **高性能**：增量构建 + 全局缓存
3. **小体积**：最小化生成的代码
4. **易修改**：声明式为主，代码生成为辅
5. **全功能**：覆盖从简单到复杂的所有场景

## 三层架构

```
第一层：零配置（80% 项目）
  - 只需 package.toml
  - 约定优于配置

第二层：编译期代码生成（15% 项目）
  - comp 函数 + 反射
  - 编译期生成代码

第三层：预构建钩子（5% 项目）
  - build.ncc 脚本
  - 调用外部工具
```

## 第一层：零配置（推荐）

### 约定优于配置

```toml
# package.toml
[package]
name = "myapp"
version = "1.0.0"

[dependencies]
http = "2.3"
```

**编译器自动：**
- 查找 `src/main.ncc` 或 `src/lib.ncc`
- 编译所有 `src/**/*.ncc`
- 链接依赖
- 生成二进制

```bash
$ ccc build
   Compiling http v2.3.5
   Compiling myapp v1.0.0
    Finished in 3.2s
```

## 第二层：编译期代码生成（comp）

### 使用 comp 函数替代构建脚本

大部分代码生成场景用 comp 函数即可，无需外部脚本：

```cpp
// src/models.ncc
import json;

// 编译期从 JSON schema 生成类型
comp {
    String schema = read_file("schema/user.json");
    auto types = parse_json_schema(schema);
    
    for (auto type_def : types) {
        generate_struct(type_def);
    }
}

// 生成的代码：
struct User {
    uint64_t id;
    String name;
    String email;
};
```

**优势：**
- ✅ 无需外部工具
- ✅ 类型安全（生成的代码立即参与类型检查）
- ✅ 增量构建（文件变化自动重新生成）

## 第三层：预构建钩子（build.ncc）

### 何时需要 build.ncc

只有以下场景需要 build.ncc：
1. 调用外部编译器（protoc, bindgen）
2. 编译 C/C++ 代码
3. 下载外部资源
4. 复杂的文件操作

### build.ncc 的简化设计

**build.ncc 在编译器内部作为 comp 块执行：**

```cpp
// build.ncc
import build;

comp {
    // 1. 调用外部工具（protoc）
    exec("protoc", {
        "--ncc_out=" + out_dir(),
        "schema/messages.proto",
    });
    
    // 2. 添加生成的文件
    add_source(out_dir() + "/messages.ncc");
    
    // 3. 链接 C 库
    if (target_os() == "linux") {
        link_lib("pthread");
    }
}
```

**关键特点：**
- ✅ 在隔离的构建上下文中执行，不共享主编译器可变状态
- ✅ 通过结构化 build API 返回输入、输出、依赖和链接声明
- ✅ 可以按依赖图并行执行，输出路径不得重叠

## 完整的构建流程

```
步骤 1：依赖解析
  - 读取 package.toml
  - MVS 算法选择版本
  - 生成 package-lock.toml
  - 下载依赖到全局缓存

步骤 2：按拓扑排序处理每个包
  For each package in dependency order:
    2a. 执行 build.ncc（如果存在）
        - 在隔离的构建上下文中运行一次
        - 调用外部工具（protoc, bindgen等）
        - 生成源文件到 out_dir()
        - 返回结构化构建描述（链接参数等）
    
    2b. 执行所有 comp 块
        - 泛型实例化（Vector<int32_t>）
        - 反射驱动的代码生成
        - 用户的 comp 代码生成逻辑
    
    2c. 类型检查
    
    2d. 编译为 MLIR
    
    2e. 生成目标文件（.o）并缓存

步骤 3：链接
  - 链接所有目标文件
  - 链接外部库
  - 生成最终二进制
```

## build API 详解

### 项目信息

```cpp
import build;

// 项目元数据（从 package.toml 读取）
String project_name();      // "myapp"
String project_version();   // "1.0.0"
String project_root();      // 项目根目录绝对路径

// 目标信息
String target_triple();     // "x86_64-pc-linux-gnu"
String target_os();         // "linux"
String target_arch();       // "x86_64"

// 构建配置
String profile();           // "dev" / "release"
String build_dir();         // "./target/release"
String out_dir();           // "./target/release/build/myapp_out"
```

### 代码生成

```cpp
// 执行外部命令
comp void exec(String cmd, Vector<String> args);

// 示例
comp {
    exec("protoc", {
        "--ncc_out=" + out_dir(),
        "schema/messages.proto",
    });
}

// 添加生成的源文件
comp void add_source(String path);

// 示例
comp {
    add_source(out_dir() + "/generated.ncc");
}
```

### 链接配置

```cpp
// 添加链接库
comp void link_lib(String name);
comp void link_lib(String name, LinkKind kind);

enum LinkKind {
    Static,   // 静态链接
    Dynamic,  // 动态链接
};

// 示例
comp {
    link_lib("pthread");
    link_lib("ssl", LinkKind::Dynamic);
}

// 添加库搜索路径
comp void link_search(String path);

// 示例
comp {
    link_search("/usr/local/lib");
    link_search(project_root() + "/vendor/lib");
}
```

### 编译标志

```cpp
// 添加编译标志
comp void add_compile_flag(String flag);

// 示例
comp {
    add_compile_flag("-DVERSION=\"" + project_version() + "\"");
    add_compile_flag("-DENABLE_FEATURE_X");
}
```

### 增量构建

```cpp
// 声明文件依赖（变化时重新运行 build.ncc）
comp void rerun_if_changed(String path);

// 声明环境变量依赖
comp void rerun_if_env_changed(String var);

// 示例
comp {
    rerun_if_changed("schema.proto");
    rerun_if_changed("codegen.py");
    rerun_if_env_changed("PROTOC_PATH");
}
```

### 环境变量

```cpp
// 获取环境变量
comp Optional<String> env(String key);

// 设置环境变量（子进程可见）
comp void set_env(String key, String value);

// 示例
comp {
    if (auto protoc = env("PROTOC")) {
        exec(protoc.value(), {"--version"});
    } else {
        throw RuntimeError("PROTOC not found");
    }
}
```

## build-dependencies 处理

### package.toml 配置

```toml
[package]
name = "myapp"
version = "1.0.0"

[dependencies]
http = "2.3"

[build-dependencies]
protobuf_compiler = "3.0"
codegen = "0.5"
```

### 编译顺序

```
1. 编译 build-dependencies
   protobuf_compiler → ~/.ccc/cache/build/protobuf_compiler-3.0.o
   codegen → ~/.ccc/cache/build/codegen-0.5.o

2. 执行 build.ncc
   - 可以 import build-dependencies
   - 调用其提供的 API

3. 编译主项目
   - 使用 runtime dependencies
```

### build.ncc 中使用 build-dependencies

```cpp
// build.ncc
import build;
import protobuf_compiler;  // build-dependency

comp {
    // 调用 build-dependency 的 API
    auto compiler = protobuf_compiler::Compiler();
    compiler.compile("schema.proto", out_dir() + "/generated");
    
    // 添加生成的文件
    add_source(out_dir() + "/generated/schema.ncc");
}
```

## 并行构建

### 依赖包的并行处理

```
依赖图（拓扑排序）：
  json (无依赖)
  log (无依赖)
  http (依赖 json)
  myapp (依赖 http, log)

并行执行：
  [并行] 编译 json + log
  [等待 json]
  [串行] 编译 http
  [等待 http, log]
  [串行] 编译 myapp
```

### build.ncc 内部并行

```cpp
import build;
import thread;

comp {
    // 并行生成多个文件
    auto tasks = Vector<Thread::Handle>();
    
    for (auto proto_file : glob("schema/*.proto")) {
        tasks.push(Thread::spawn([=]{
            exec("protoc", {
                "--ncc_out=" + out_dir(),
                proto_file,
            });
        }));
    }
    
    // 等待所有任务完成
    for (auto& task : tasks) {
        task.join();
    }
}
```

## 增量构建

### 触发重新构建的条件

```
1. build.ncc 自身修改
2. build-dependencies 版本变化
3. rerun_if_changed() 声明的文件修改
4. rerun_if_env_changed() 声明的环境变量变化
```

### 编译器的增量检查

```text
增量检查流程（非 NCC 源代码）：
1. 检查 build.ncc 自身是否变化。
2. 检查 build-dependencies 是否变化。
3. 检查声明的文件输入是否变化。
4. 检查声明的环境变量是否变化。
以上任一项变化时重新执行构建脚本，否则继续检查其余缓存输入。
```

## 错误处理

### build.ncc 执行失败

```bash
$ ccc build

Running build.ncc...
error: build script failed

  × protobuf compiler not found
  ╰─▶ required by build.ncc:10

  help: install protobuf compiler:
    - Ubuntu: sudo apt install protobuf-compiler
    - macOS: brew install protobuf

Build failed. Fix the errors above and retry.
```

### comp 块中抛出异常

编译期求值期间未捕获的异常使编译失败，诊断包含异常信息和源码位置；
使用标准 `throw`，不引入宏式报错语法。运行时仍按普通异常规则传播。

```cpp
comp {
    if (!file_exists("required_file.txt")) {
        throw RuntimeError("required_file.txt not found");
    }
}

// 编译错误：
// error: uncaught exception during compile-time evaluation
//   required_file.txt not found
//   at build.ncc:5
```

## 实际示例

### 示例 1：Protocol Buffers

```cpp
// build.ncc
import build;

comp {
    // 查找所有 .proto 文件
    auto proto_files = glob("schema/*.proto");
    
    for (auto proto : proto_files) {
        // 调用 protoc
        exec("protoc", {
            "--ncc_out=" + out_dir(),
            "--proto_path=schema",
            proto,
        });
        
        // 声明依赖
        rerun_if_changed(proto);
    }
    
    // 添加生成的文件
    add_source(out_dir() + "/messages.ncc");
}
```

### 示例 2：FFI Bindings

```cpp
// build.ncc
import build;

comp {
    // 生成 C 库的绑定
    exec("bindgen", {
        "vendor/mylib.h",
        "--output", out_dir() + "/bindings.ncc",
    });
    
    add_source(out_dir() + "/bindings.ncc");
    
    // 链接 C 库
    link_search(project_root() + "/vendor/lib");
    link_lib("mylib", LinkKind::Static);
    
    rerun_if_changed("vendor/mylib.h");
}
```

### 示例 3：编译 C 代码

```cpp
// build.ncc
import build;

comp {
    // 编译 C 文件
    for (auto c_file : glob("vendor/*.c")) {
        exec("gcc", {
            "-c",
            "-O2",
            "-o", out_dir() + "/vendor.o",
            c_file,
        });
    }
    
    // 链接编译后的对象文件
    link_search(out_dir());
    link_lib("vendor", LinkKind::Static);
}
```

## 构建能力概览

| 特性 | NCC |
|------|-----|
| 配置文件 | package.toml |
| 预构建脚本 | build.ncc（受限编译期执行） |
| 代码生成 | comp 函数 |
| 并行编译 | ✅ |
| 全局缓存 | ✅ |

## 性能目标

- **冷编译**：< 5秒/万行代码
- **增量编译**：< 1秒（单文件修改）
- **并行度**：接近线性扩展（8核约 7-8倍）
- **缓存命中率**：> 90%

## 下一步

- 查看 [10-packages.md](10-packages.md) 了解包管理
- 查看 [12-interop.md](12-interop.md) 了解 C 互操作
- 查看 [13-performance.md](13-performance.md) 了解性能优化
