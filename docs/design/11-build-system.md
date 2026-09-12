# 十一、构建系统

## 核心主旨：声明式配置 + 编译期代码生成

NCC 的构建系统使用 NCC 语言本身编写，提供声明式和编译期代码生成两种方式。

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
- ✅ 在编译器内部执行（统一机制）
- ✅ 直接修改编译器状态（无需 JSON 中间文件）
- ✅ 可以并行执行（按依赖顺序）

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
        - 在编译器内部作为 comp 块运行
        - 调用外部工具（protoc, bindgen等）
        - 生成源文件到 out_dir()
        - 修改编译器配置（链接参数等）
    
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
comp fn exec(cmd: String, args: Vector<String>);

// 示例
comp {
    exec("protoc", {
        "--ncc_out=" + out_dir(),
        "schema/messages.proto",
    });
}

// 添加生成的源文件
comp fn add_source(path: String);

// 示例
comp {
    add_source(out_dir() + "/generated.ncc");
}
```

### 链接配置

```cpp
// 添加链接库
comp fn link_lib(name: String);
comp fn link_lib(name: String, kind: LinkKind);

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
comp fn link_search(path: String);

// 示例
comp {
    link_search("/usr/local/lib");
    link_search(project_root() + "/vendor/lib");
}
```

### 编译标志

```cpp
// 添加编译标志
comp fn add_compile_flag(flag: String);

// 示例
comp {
    add_compile_flag("-DVERSION=\"" + project_version() + "\"");
    add_compile_flag("-DENABLE_FEATURE_X");
}
```

### 增量构建

```cpp
// 声明文件依赖（变化时重新运行 build.ncc）
comp fn rerun_if_changed(path: String);

// 声明环境变量依赖
comp fn rerun_if_env_changed(var: String);

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
comp fn env(key: String) -> Optional<String>;

// 设置环境变量（子进程可见）
comp fn set_env(key: String, value: String);

// 示例
comp {
    if (auto protoc = env("PROTOC")) {
        exec(protoc.value(), {"--version"});
    } else {
        panic!("PROTOC not found");
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

```rust
// 编译器检查是否需要重新运行 build.ncc
fn need_rebuild_script(package: &Package) -> bool {
    // 1. build.ncc 自身是否修改
    if file_changed("build.ncc") {
        return true;
    }
    
    // 2. build-dependencies 是否修改
    for dep in package.build_dependencies {
        if dependency_changed(dep) {
            return true;
        }
    }
    
    // 3. 用户声明的依赖是否修改
    for path in build_config.rebuild_if_changed {
        if file_changed(path) {
            return true;
        }
    }
    
    // 4. 环境变量是否修改
    for env_var in build_config.rerun_if_env_changed {
        if env_changed(env_var) {
            return true;
        }
    }
    
    false
}
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

```cpp
comp {
    if (!file_exists("required_file.txt")) {
        panic!("required_file.txt not found");
    }
}

// 编译错误：
// error: build script panicked
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

## 与其他语言对比

| 特性 | Rust | Go | NCC |
|------|------|-----|-----|
| 配置文件 | Cargo.toml | 无 | package.toml |
| 预构建脚本 | build.rs（独立编译） | go generate | build.ncc（编译期执行） |
| 代码生成 | 过程宏 + build.rs | go generate | comp 函数 |
| 并行编译 | ✅ | ✅ | ✅ |
| 全局缓存 | ✅ | ✅ | ✅ |

## 性能目标

- **冷编译**：< 5秒/万行代码
- **增量编译**：< 1秒（单文件修改）
- **并行度**：接近线性扩展（8核约 7-8倍）
- **缓存命中率**：> 90%

## 下一步

- 查看 [10-packages.md](10-packages.md) 了解包管理
- 查看 [12-interop.md](12-interop.md) 了解 C 互操作
- 查看 [13-performance.md](13-performance.md) 了解性能优化


## 构建脚本类型

### 1. 简单项目：`package.toml`

大多数项目只需要 `package.toml`，不需要构建脚本：

```toml
[package]
name = "myapp"
version = "1.0.0"

[dependencies]
http = "2.3.1"
```

编译器自动处理：
- 查找 `src/main.ccm` 或 `src/lib.ccm`
- 编译所有 `.ccm` 文件
- 链接依赖

### 2. 预构建钩子：`build.ccc`

需要代码生成、编译 C 代码、调用外部工具时，创建 `build.ccc`：

```cpp
// build.ccc - 在主项目编译前执行
import build;

int main() {
    // 代码生成
    compile("proto/messages.proto", "src/generated");
    
    // 编译 C 库
    compile_c("vendor/libfoo.c", "libfoo.a");
    
    // 设置链接选项
    link_lib("foo", LinkType::Static);
    link_search("./target");
    
    return 0;
}
```

### 3. 复杂项目：`build.ccc` 完整 API

使用完整的构建 API，类似 CMake/xmake：

```cpp
// build.ccc
import build;

int main() {
    // 定义目标
    auto exe = executable("myapp", {
        .sources = glob("src/**/*.ccm"),
        .dependencies = {"http", "json"},
        .defines = {"DEBUG"},
    });
    
    // 添加测试
    auto test = executable("myapp_test", {
        .sources = glob("tests/**/*.ccm"),
        .link_with = {exe},
    });
    
    // 自定义命令
    custom_command({
        .output = "src/generated.ccm",
        .command = "python codegen.py",
        .depends = {"schema.json"},
    });
    
    return 0;
}
```

## 构建 API 参考

### 核心模块：`build`

```cpp
import build;

// 项目信息
String project_name();
String project_version();
String build_dir();
String source_dir();

// 目标平台信息
String target_os();        // "windows", "linux", "macos", ...
String target_arch();      // "x86_64", "aarch64", "wasm32", ...
String target_triple();    // "x86_64-pc-windows-msvc"
bool is_cross_compile();

// 构建配置
String profile();          // "dev", "release", "bench", ...
bool is_debug();
bool is_release();
```

### 目标定义（Targets）

#### 可执行文件

```cpp
auto exe = executable("myapp", {
    .sources = {"src/main.ccm", "src/utils.ccm"},
    .dependencies = {"http@2.3.1"},
    .link_dirs = {"lib"},
    .link_libs = {"pthread"},
    .defines = {"VERSION=\"1.0.0\""},
    .features = {"json", "tls"},
});
```

#### 静态库

```cpp
auto lib = static_library("mylib", {
    .sources = glob("src/**/*.ccm"),
    // NCC 模块通过 export/import 发布；外部 C/C++ 头文件仅用于互操作。
    .export_module = "mylib",
});
```

#### 动态库

```cpp
auto dll = shared_library("mylib", {
    .sources = glob("src/**/*.ccm"),
    .version = "1.0.0",
    .soversion = "1",  // libmylib.so.1
});
```

### 文件查找

```cpp
// Glob 模式
Vector<String> sources = glob("src/**/*.ccm");
// 外部 C/C++ 头文件可由互操作目标显式提供；NCC 模块不使用头文件。

// 排除模式
Vector<String> files = glob("src/**/*.ccm", {
    .exclude = {"src/tests/**", "src/bench/**"}
});

// 递归查找
Vector<String> all_ccm = find_files(".", "*.ccm", {.recursive = true});
```

### 依赖管理

```cpp
// 添加依赖
add_dependency("http", "2.3.1");
add_dev_dependency("test_framework", "0.9.0");

// 条件依赖
if (target_os() == "windows") {
    add_dependency("windows_api", "0.48");
}

// 子目录
add_subdirectory("subproject");
```

### 编译选项

```cpp
// 全局编译选项
add_compile_flags({"-Wall", "-Wextra"});

// 目标特定选项
exe.add_compile_flags({"-O3", "-march=native"});

// 条件选项
if (is_debug()) {
    add_compile_flags({"-g", "-fsanitize=address"});
}

// 定义宏（通过编译期常量，不是 C 宏）
add_define("VERSION", "1.0.0");
add_define("DEBUG_MODE", is_debug());
```

### 链接选项

```cpp
// 链接库
link_lib("pthread", LinkType::Dynamic);
link_lib("mylib", LinkType::Static);

// 链接搜索路径
link_search("/usr/local/lib");
link_search("./vendor/lib");

// 链接标志
add_link_flags({"-Wl,-rpath,$ORIGIN"});

// 框架（macOS）
if (target_os() == "macos") {
    link_framework("Cocoa");
}
```

### 自定义命令

```cpp
// 代码生成
custom_command({
    .name = "generate_proto",
    .output = {"src/generated/messages.ccm"},
    .command = "protoc --ccc_out=src/generated proto/messages.proto",
    .depends = {"proto/messages.proto"},
    .working_dir = source_dir(),
});

// 资源复制
custom_command({
    .name = "copy_assets",
    .output = {"${BUILD_DIR}/assets"},
    .command = "cp -r assets ${BUILD_DIR}/",
    .depends = glob("assets/**/*"),
});

// 总是运行（无输出）
custom_target({
    .name = "format_check",
    .command = "ccc fmt --check",
    .always_run = true,
});
```

### 测试和基准

```cpp
// 添加测试
add_test("unit_tests", {
    .sources = glob("tests/**/*.ccm"),
    .link_with = {lib},
});

// 添加基准测试
add_benchmark("perf_test", {
    .sources = {"bench/main.ccm"},
    .link_with = {lib},
});

// 设置测试参数
set_test_args({"--verbose", "--color=always"});
```

## 多线程编译

构建系统内置多线程编译支持，自动并行编译源文件。

### 自动并行编译

```cpp
// build.ccc
import build;

int main() {
    auto exe = executable("myapp", {
        .sources = glob("src/**/*.ccm"),  // 1000 个源文件
    });
    
    // 编译器自动：
    // 1. 分析依赖关系
    // 2. 构建依赖图
    // 3. 并行编译无依赖的文件
    // 4. 使用 parallel 多线程编译
    
    return 0;
}
```

### 并行编译实现

```cpp
// 构建系统内部实现（用户无需关心）
comp fn compile_sources(sources: Vector<String>) {
    // 1. 构建依赖图
    auto dep_graph = build_dependency_graph(sources);
    
    // 2. 拓扑排序，找出可并行编译的批次
    Vector<Vector<String>> batches = topological_batches(dep_graph);
    
    // 3. 批次内并行编译
    for (auto& batch : batches) {
        parallel(batch.size(), [&](size_t i) {
            compile_file(batch[i]);
        });
    }
}
```

### 配置并行度

```toml
# package.toml
[build]
parallel = true              # 启用并行编译（默认）
num_jobs = 0                 # 0 = 自动（CPU 核心数），或指定数量
```

```bash
# 命令行控制
ccc build -j 8               # 使用 8 个线程
ccc build -j                 # 自动检测核心数
ccc build --no-parallel      # 禁用并行（调试用）
```

### 并行编译示例

```cpp
// build.ccc - 自定义并行编译策略
import build;
import thread;

int main() {
    Vector<String> sources = glob("src/**/*.ccm");
    
    // 方案 1：使用构建系统默认并行（推荐）
    auto exe = executable("myapp", {
        .sources = sources,
    });
    
    // 方案 2：手动并行编译（高级用法）
    Channel<CompileResult> results(sources.size());
    
    parallel(sources.size(), [&](size_t i) {
        CompileResult result = compile_file(sources[i]);
        results.send(result);
    });
    
    // 收集结果
    Vector<CompileResult> all_results;
    for (size_t i = 0; i < sources.size(); ++i) {
        all_results.push(results.recv());
    }
    
    // 检查错误
    for (auto& result : all_results) {
        if (!result.success) {
            println("Compilation failed: {}", result.error);
            return 1;
        }
    }
    
    return 0;
}
```

### 增量编译 + 并行

```cpp
// 构建系统自动结合增量编译和并行编译
comp fn incremental_parallel_build(sources: Vector<String>) {
    // 1. 检查哪些文件需要重新编译
    Vector<String> changed_files;
    for (auto& src : sources) {
        if (needs_recompile(src)) {
            changed_files.push(src);
        }
    }
    
    println("Recompiling {} / {} files", changed_files.size(), sources.size());
    
    // 2. 并行编译变化的文件
    parallel(changed_files.size(), [&](size_t i) {
        compile_file(changed_files[i]);
    });
}
```

### 构建缓存 + 并行

```cpp
// 全局缓存 + 并行编译
comp fn cached_parallel_build(sources: Vector<String>) {
    // 1. 检查全局缓存
    Vector<String> uncached_files;
    for (auto& src : sources) {
        String hash = compute_hash(src);
        if (!global_cache_has(hash)) {
            uncached_files.push(src);
        }
    }
    
    // 2. 并行编译未缓存的文件
    parallel(uncached_files.size(), [&](size_t i) {
        auto result = compile_file(uncached_files[i]);
        global_cache_put(compute_hash(uncached_files[i]), result);
    });
    
    // 3. 从缓存加载其他文件
    parallel(sources.size() - uncached_files.size(), [&](size_t i) {
        load_from_cache(sources[i]);
    });
}
```

### 并行链接

```cpp
// 大型项目：并行链接多个目标
int main() {
    auto lib1 = static_library("lib1", {.sources = glob("lib1/**/*.ccm")});
    auto lib2 = static_library("lib2", {.sources = glob("lib2/**/*.ccm")});
    auto lib3 = static_library("lib3", {.sources = glob("lib3/**/*.ccm")});
    
    // 构建系统自动并行链接（无依赖关系）
    // lib1、lib2、lib3 可以同时链接
    
    auto exe = executable("app", {
        .sources = glob("app/**/*.ccm"),
        .link_with = {lib1, lib2, lib3},
    });
    
    return 0;
}
```

### 性能监控

```cpp
// build.ccc
import build;

int main() {
    enable_profiling(true);  // 启用性能监控
    
    auto exe = executable("myapp", {
        .sources = glob("src/**/*.ccm"),
    });
    
    // 构建完成后输出统计
    // Compiled 1000 files in 12.3s
    // - Parallel efficiency: 87%
    // - Average per file: 12ms
    // - Peak memory: 2.1GB
    // - Cache hit rate: 65%
    
    return 0;
}
```

### 测试和基准

```cpp
// 添加测试
add_test("unit_tests", {
    .sources = glob("tests/**/*.ccm"),
    .link_with = {lib},
});

// 添加基准测试
add_benchmark("perf_test", {
    .sources = {"bench/main.ccm"},
    .link_with = {lib},
});

// 设置测试参数
set_test_args({"--verbose", "--color=always"});
```

### 安装规则

```cpp
// 安装可执行文件
install_target(exe, InstallDestination::Bin);

// 安装库
install_target(lib, InstallDestination::Lib);

// 安装外部 C/C++ 互操作头文件；NCC 模块通过 export/import 发布
install_files(glob("include/**/*.h"), "include/mylib");

// 自定义安装路径
install_file("config.toml", "/etc/myapp/config.toml");
```

## 声明式 vs 命令式

### 声明式风格（推荐）

类似 xmake/Bazel，声明"是什么"而不是"怎么做"：

```cpp
import build;

int main() {
    // 声明目标和依赖关系，构建系统自动推导构建顺序
    auto lib = static_library("core", {
        .sources = glob("core/**/*.ccm"),
    });
    
    auto exe = executable("app", {
        .sources = glob("app/**/*.ccm"),
        .link_with = {lib},
    });
    
    add_test("tests", {
        .sources = glob("tests/**/*.ccm"),
        .link_with = {lib},
    });
    
    return 0;
}
```

### 命令式风格（灵活）

类似 Rust build.rs，手动控制构建流程：

```cpp
import build;

int main() {
    // 1. 生成代码
    if (!exists("src/generated.ccm")) {
        run_command("python", {"codegen.py", "schema.json"});
    }
    
    // 2. 编译 C 库
    compile_c_files(glob("vendor/**/*.c"), "libvendor.a");
    
    // 3. 设置链接
    link_lib("vendor", LinkType::Static);
    link_search("./target");
    
    // 4. 通知构建系统
    rerun_if_changed("schema.json");
    
    return 0;
}
```

## 构建输出协议

构建脚本通过标准输出向编译器传递指令（类似 Cargo）：

```cpp
// 链接库
println("ccc:link-lib=static=foo");
println("ccc:link-lib=dynamic=bar");

// 链接搜索路径
println("ccc:link-search=native=/usr/local/lib");

// 重新运行条件
println("ccc:rerun-if-changed=build.ccc");
println("ccc:rerun-if-changed=schema.json");
println("ccc:rerun-if-env-changed=PROTOC_PATH");

// 设置编译期环境变量（传递给主项目）
println("ccc:rustc-env=GIT_HASH={}", git_hash());
println("ccc:rustc-env=BUILD_TIME={}", now());

// 警告和错误
println("ccc:warning=Using deprecated build option");
println("ccc:error=Missing required tool: protoc");
```

高层 API 自动生成这些输出：

```cpp
// 等价于 println("ccc:link-lib=static=foo");
link_lib("foo", LinkType::Static);

// 等价于 println("ccc:rerun-if-changed=schema.json");
rerun_if_changed("schema.json");
```

## 实际示例

### 示例 1：Protocol Buffers 项目

```cpp
// build.ccc
import build;
import protobuf;

int main() {
    // 编译所有 .proto 文件
    Vector<String> protos = glob("proto/**/*.proto");
    
    for (auto proto : protos) {
        String output = "src/generated/" + 
                        basename(proto).replace(".proto", ".ccm");
        
        custom_command({
            .output = {output},
            .command = format("protoc --ccc_out=src/generated {}", proto),
            .depends = {proto},
        });
        
        rerun_if_changed(proto);
    }
    
    // 主程序链接生成的代码
    auto exe = executable("server", {
        .sources = glob("src/**/*.ccm"),
        .include_dirs = {"src/generated"},
    });
    
    return 0;
}
```

### 示例 2：混合 C/C++ 项目

```cpp
// build.ccc
import build;
import cc;  // C/C++ 编译辅助库

int main() {
    // 编译 C 库
    auto c_lib = static_library("legacy", {
        .sources = glob("vendor/**/*.c"),
        .include_dirs = {"vendor/include"},
        .flags = {"-std=c99", "-Wall"},
    });
    
    // NCC 主程序
    auto exe = executable("app", {
        .sources = glob("src/**/*.ccm"),
        .link_with = {c_lib},
    });
    
    return 0;
}
```

### 示例 3：条件编译（跨平台）

```cpp
// build.ccc
import build;

int main() {
    Vector<String> sources = glob("src/**/*.ccm");
    Vector<String> deps;
    
    // 平台特定依赖
    comp {
        if (target_os() == "windows") {
            sources.push("src/platform/windows.ccm");
            deps.push("windows_api");
        } else if (target_os() == "linux") {
            sources.push("src/platform/linux.ccm");
            deps.push("libc");
        } else if (target_os() == "macos") {
            sources.push("src/platform/macos.ccm");
            link_framework("Cocoa");
        }
    }
    
    auto exe = executable("app", {
        .sources = sources,
        .dependencies = deps,
    });
    
    return 0;
}
```

### 示例 4：代码生成 + 测试

```cpp
// build.ccc
import build;

int main() {
    // 1. 代码生成
    custom_command({
        .output = {"src/tables.ccm"},
        .command = "python gen_tables.py data.csv",
        .depends = {"data.csv", "gen_tables.py"},
    });
    
    // 2. 主库
    auto lib = static_library("mylib", {
        .sources = glob("src/**/*.ccm"),
    });
    
    // 3. 可执行文件
    auto exe = executable("app", {
        .sources = {"app/main.ccm"},
        .link_with = {lib},
    });
    
    // 4. 单元测试
    add_test("unit_tests", {
        .sources = glob("tests/**/*.ccm"),
        .link_with = {lib},
    });
    
    // 5. 集成测试
    add_test("integration_tests", {
        .sources = glob("tests/integration/**/*.ccm"),
        .link_with = {exe},
        .test_data = glob("tests/fixtures/**"),
    });
    
    return 0;
}
```

## 构建系统对比

| 特性 | CMake | xmake | Bazel | Rust | NCC |
|------|-------|-------|-------|------|-----|
| 语言 | CMake DSL | Lua | Starlark | Rust | NCC |
| 风格 | 命令式 | 声明式 | 声明式 | 命令式 | 两者都支持 |
| 学习曲线 | 陡峭 | 中等 | 陡峭 | 简单 | 简单 |
| 跨平台 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 增量构建 | ✓ | ✓ | ✓ | ✓ | ✓ |
| 缓存 | 本地 | 本地 | 分布式 | 本地 | 全局 + 本地 |
| IDE 集成 | 强 | 中 | 弱 | 强 | 强（计划） |

## 构建脚本最佳实践

### 1. 优先使用声明式 API

```cpp
// ✓ 好：声明式
auto exe = executable("app", {
    .sources = glob("src/**/*.ccm"),
});

// ✗ 差：手动控制
for (auto file : glob("src/**/*.ccm")) {
    compile_file(file);
}
link_files(object_files, "app");
```

### 2. 使用 glob 而不是列举文件

```cpp
// ✓ 好：自动发现
.sources = glob("src/**/*.ccm")

// ✗ 差：手动维护
.sources = {"src/a.ccm", "src/b.ccm", "src/c.ccm"}
```

### 3. 条件逻辑用 comp

```cpp
// ✓ 好：编译期条件
comp {
    if (target_os() == "windows") {
        add_dependency("windows_api");
    }
}
```

### 4. 追踪所有输入

```cpp
// 确保文件改变时重新构建
rerun_if_changed("schema.json");
rerun_if_changed("codegen.py");
rerun_if_env_changed("PROTOC_PATH");
```

### 5. 使用工作空间管理多包

```toml
# 根目录 package.toml
[workspace]
members = ["server", "client", "common"]
```

## 命令行工具

```bash
# 构建
ccc build                      # 使用 dev profile
ccc build --release            # 使用 release profile
ccc build --profile=custom     # 自定义 profile

# 清理
ccc clean                      # 清理构建产物
ccc clean --release            # 清理 release 构建

# 测试
ccc test                       # 运行所有测试
ccc test pattern               # 运行匹配的测试
ccc test --release             # Release 模式测试

# 基准测试
ccc bench                      # 运行基准测试

# 安装
ccc install                    # 安装到默认位置
ccc install --prefix=/usr/local

# 查看构建图
ccc build --dry-run            # 不实际构建，显示构建图
ccc build --verbose            # 详细输出
```

## 下一步

- 查看 [10-packages.md](10-packages.md) 了解包管理
- 查看 [09-compiler.md](09-compiler.md) 了解编译器架构
- 查看 [12-examples.md](12-examples.md) 查看完整示例

