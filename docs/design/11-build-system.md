# 构建系统（Build System）

NCC 的构建系统使用 NCC 语言本身编写，提供声明式和命令式两种风格，参考了 CMake、xmake、Bazel 和 Rust 的设计。**内置多线程编译支持**。

## 核心理念

- **用 NCC 写构建脚本**：不需要学习新的 DSL（CMake）或新的语言（Python/Starlark）
- **声明式优先，命令式备用**：大部分场景用声明式 API，复杂逻辑用命令式
- **增量构建**：自动追踪依赖，只重新构建变化的部分
- **多线程编译**：自动并行编译，充分利用多核 CPU
- **跨平台**：构建脚本在所有平台上行为一致
- **编译期执行**：构建脚本在编译期运行，可以使用 `comp` 和反射

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

