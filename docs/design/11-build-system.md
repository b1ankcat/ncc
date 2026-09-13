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

### 声明式 API 是规范形式

`build.ncc` 只有一套语义：脚本执行的结果是一份**结构化构建描述**，编译器读取
该描述后才真正执行外部命令。因此存在两层写法，但不是两套机制：

| 层次 | 函数 | 说明 |
| --- | --- | --- |
| 规范形式 | `input_file`、`tool`、`custom_command`、`embed_as_constant` | 显式登记输入、工具与输出，直接构成构建图节点 |
| 便捷封装 | `exec`、`add_source`、`link_lib`、`rerun_if_changed` | 在规范形式之上的简写，展开为等价的构建图节点 |

便捷封装不绕过登记：`exec(...)` 展开为一个 `custom_command`，其 `program` 必须
来自 `tool(...)`；`add_source(p)` 要求 `p` 在 `out_dir()` 内并声明为某个命令的
输出。因此 `exec` 不是"立即执行子进程"，而是"向构建图追加一条命令"。
命令之间的并行由编译器按构建图调度，脚本自身不启动线程。

## 三层架构

按需要的能力分三层，各层可叠加：

| 层 | 机制 | 适用 |
| --- | --- | --- |
| 一 | 只有 `package.toml` | 约定优于配置，无需代码生成 |
| 二 | `comp` 函数 + 反射 | 从已登记的输入生成代码，不调用外部工具 |
| 三 | `build.ncc` | 需要外部工具、编译 C 代码或下载资源 |

## 第一层：零配置

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

大部分代码生成场景用 comp 函数即可，无需外部工具。但普通 `comp` 不能读文件：
外部输入必须由 `build.ncc` 登记，再作为编译期常量交给普通 `comp` 使用。

```cpp
// build.ncc —— 登记外部输入，内容进入构建图与缓存键
import build;

comp {
    embed_as_constant("USER_SCHEMA", input_file("schema/user.json"));
}
```

```cpp
// src/models.ncc —— 普通 comp 只消费已登记的编译期常量
import json;

comp {
    auto types = parse_json_schema(USER_SCHEMA);

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

相比调用外部生成器，这条路径不依赖外部工具，且生成的代码立即参与类型检查——
拼错字段名在编译期就会报错，而不是等生成的文件被编译时。`input_file` 已登记该
输入，schema 变化会触发重新生成。

`read_file`、`file_exists`、`glob`、`env` 等访问外部状态的函数只存在于
`import build` 提供的受限 API 中，在普通源文件的 `comp` 块里不可见。

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

脚本在隔离的构建上下文中执行，不共享主编译器的可变状态；它的产出是一份结构化
构建描述（输入、输出、依赖与链接声明），由编译器消费后再实际执行命令。因此
输出路径不得重叠——重叠会在构建图校验阶段被拒绝，而不是留到运行时竞争同一文件。

## 完整的构建流程

```
步骤 1：依赖解析
  - 读取 package.toml
  - 求约束交集，选择最低可用版本
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

### 输入登记（规范形式）

```cpp
// 登记单个文件输入；缺失时抛出 FileNotFound
comp String input_file(String path);

// 同上，但缺失时返回空 Optional（同样登记"不存在"这一事实）
comp Optional<String> try_input_file(String path);

// 登记 glob 模式及其匹配结果；新增/删除匹配文件都会触发重新构建
comp Vector<String> input_glob(String pattern);

// 登记外部工具及其版本要求；返回可作为 program 使用的句柄
comp Tool tool(String name, String version_requirement);

// 登记环境变量读取
comp Optional<String> env(String key);

// 登记网络输入，必须提供校验和
comp String input_url(String url, String sha256);

// 把已登记输入的内容作为编译期常量暴露给普通源文件的 comp 块
comp void embed_as_constant(String name, String content);

// 用显式路径登记工具（例如来自环境变量），而非在 PATH 中查找
comp Tool tool_at(String path, String version_requirement);
```

### 路径与列表辅助

```cpp
comp String stem_of(String path);        // "schema/user.proto" → "user"
comp String extension_of(String path);   // → "proto"
comp String join_path(String a, String b);

comp Vector<String> concat(Vector<String> a, Vector<String> b);
```

这些是构建脚本常用的纯函数，不访问文件系统，因此不需要登记。

### 命令与代码生成

```cpp
// 登记一条命令：inputs/outputs 构成构建图的边
comp void custom_command(CommandSpec spec);

struct CommandSpec {
    Vector<String> inputs;
    Vector<String> outputs;    // 必须位于 out_dir() 内，且不与其他命令重叠
    Tool program;
    Vector<String> args;
};

// 便捷封装：展开为一条 custom_command
comp void exec(Tool program, Vector<String> args);

// 添加生成的源文件；path 必须是某条命令声明过的输出
comp void add_source(String path);

// 示例
comp {
    auto generated = out_dir() + "/generated.ncc";
    exec(tool("protoc", "25.1"),
         {"--ncc_out=" + out_dir(), input_file("schema/messages.proto")});
    add_source(generated);
}
```

### 链接配置

```cpp
// 添加链接库
comp void link_lib(String name);
comp void link_lib(String name, LinkKind kind);

enum class LinkKind {
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

`input_file`、`input_glob`、`env` 等登记函数已经把对应输入写进构建图，因此
**不需要**额外声明重新构建条件。只有一种情况需要显式声明：脚本读取了某个文件
但没有把它作为命令输入，例如仅用于决定分支的配置文件。

```cpp
// 声明额外的文件依赖（不作为任何命令的输入，但影响构建描述）
comp void rerun_if_changed(String path);

// 示例：codegen.py 由 python 间接调用，不出现在 inputs 里
comp {
    rerun_if_changed("codegen.py");
}
```

没有 `rerun_if_env_changed`：环境变量只能通过 `env()` 读取，而 `env()` 本身就
完成了登记。

### 环境变量

```cpp
// 读取环境变量（登记进缓存键）
comp Optional<String> env(String key);

// 为某条命令设置环境变量
comp void command_env(String key, String value);
```

`command_env` 作用于随后登记的命令，并进入这些命令的缓存键。构建脚本不能修改
编译器自身进程的环境，因此没有全局 `set_env`——否则命令的执行结果会依赖脚本的
执行顺序，破坏可重现性。

```cpp
comp {
    // 允许用 PROTOC 覆盖工具路径，缺失时回退到 PATH 查找
    auto protoc = env("PROTOC").has_value()
        ? tool_at(env("PROTOC").value(), "25.1")
        : tool("protoc", "25.1");

    exec(protoc, {"--version"});
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

### build.ncc 声明的命令并行

构建脚本自身单线程执行，只负责登记命令；相互独立的命令由编译器按构建图并行
调度。脚本里不需要（也不能）创建线程：

```cpp
import build;

comp {
    auto protoc = tool("protoc", "25.1");

    // 每个 proto 文件登记为一条独立命令，输出互不重叠
    for (auto proto_file : input_glob("schema/*.proto")) {
        auto generated = out_dir() + "/" + stem_of(proto_file) + ".ncc";

        custom_command({
            .inputs = {proto_file},
            .outputs = {generated},
            .program = protoc,
            .args = {"--ncc_out=" + out_dir(), "--proto_path=schema", proto_file},
        });

        add_source(generated);
    }
}
```

编译器发现这些命令没有相互依赖，会并行执行它们。输出路径重叠的命令会在构建图
校验阶段被拒绝，而不是留到运行时竞争同一文件。

## 增量构建

### 触发重新构建的条件

```
1. build.ncc 自身修改
2. build-dependencies 版本变化
3. input_file() / input_glob() 登记的文件或匹配结果变化
4. env() 读取的环境变量变化
5. tool() 解析到的外部工具版本或路径变化
6. rerun_if_changed() 显式声明的文件修改
```

### 编译器的增量检查

```text
增量检查流程（非 NCC 源代码）：
1. 检查 build.ncc 自身是否变化。
2. 检查 build-dependencies 是否变化。
3. 检查登记的文件输入与 glob 匹配结果是否变化。
4. 检查登记的环境变量是否变化。
5. 检查登记的外部工具版本与路径是否变化。
以上任一项变化时重新执行构建脚本，得到新的结构化构建描述；
描述未变时沿用上次的构建图，再按各命令自身的缓存键决定是否重跑。
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
// build.ncc：input_file 登记输入，文件缺失时抛出
import build;

comp {
    if (!try_input_file("required_file.txt")) {
        throw RuntimeError("required_file.txt not found");
    }
}

// 编译错误：
// error: uncaught exception during compile-time evaluation
//   required_file.txt not found
//   at build.ncc:5
```

`input_file` 在文件缺失时直接抛出 `FileNotFound`；`try_input_file` 返回
`Optional`，用于需要自定义诊断或可选输入的场景。两者都会把该路径（含"不存在"
这一事实）登记进构建图，文件之后被创建会触发重新构建。

## 实际示例

### 示例 1：Protocol Buffers

```cpp
// build.ncc
import build;

comp {
    auto protoc = tool("protoc", "25.1");

    // input_glob 登记模式本身：新增或删除 .proto 文件都会触发重新构建
    for (auto proto : input_glob("schema/*.proto")) {
        auto generated = out_dir() + "/" + stem_of(proto) + ".ncc";

        custom_command({
            .inputs = {proto},
            .outputs = {generated},
            .program = protoc,
            .args = {"--ncc_out=" + out_dir(), "--proto_path=schema", proto},
        });

        add_source(generated);
    }
}
```

`input_glob` 同时登记匹配模式和匹配结果，因此不需要额外的 `rerun_if_changed`。
每个 proto 文件对应一条独立命令和一个独立输出，编译器可并行执行。

### 示例 2：FFI Bindings

```cpp
// build.ncc
import build;

comp {
    auto header = input_file("vendor/mylib.h");   // 登记输入
    auto bindings = out_dir() + "/bindings.ncc";

    // 生成 C 库的绑定
    custom_command({
        .inputs = {header},
        .outputs = {bindings},
        .program = tool("bindgen", "0.69"),
        .args = {header, "--output", bindings},
    });

    add_source(bindings);

    // 链接 C 库
    link_search(project_root() + "/vendor/lib");
    link_lib("mylib", LinkKind::Static);
}
```

### 示例 3：编译 C 代码

```cpp
// build.ncc
import build;

comp {
    auto cc = tool("gcc", "13");
    Vector<String> objects;

    // 每个 .c 文件编译到各自的 .o：所有命令共用一个输出会被构建图拒绝
    for (auto c_file : input_glob("vendor/*.c")) {
        auto object = out_dir() + "/" + stem_of(c_file) + ".o";
        objects.push(object);

        custom_command({
            .inputs = {c_file},
            .outputs = {object},
            .program = cc,
            .args = {"-c", "-O2", "-o", object, c_file},
        });
    }

    // 打包为静态库后链接
    auto archive = out_dir() + "/libvendor.a";
    custom_command({
        .inputs = objects,
        .outputs = {archive},
        .program = tool("ar", "2.41"),
        .args = concat({"rcs", archive}, objects),
    });

    link_search(out_dir());
    link_lib("vendor", LinkKind::Static);
}
```

构建相关的性能目标见 [14-performance.md](14-performance.md#编译速度)。

