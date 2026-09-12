# 九、包管理系统

删除了 `#include`/`namespace`/`std::`，需要一个完整的包管理方案，统一模块和依赖管理。

## 包清单文件：`package.toml`

每个项目根目录包含 `package.toml`（借鉴 Cargo）：

```toml
[package]
name = "myapp"
version = "1.0.0"
authors = ["Your Name <you@example.com>"]
edition = "2026"

[dependencies]
http = "2.3.1"
json = "1.5.0"
log = "0.4"

[dev-dependencies]
test_framework = "0.9.0"
benchmark = "1.0"

[build-dependencies]
# 构建时需要的依赖（构建脚本、代码生成器等）
codegen = "0.5"
protobuf_compiler = "3.0"

[target.'cfg(windows)'.dependencies]
# Windows 特定依赖
windows_api = "0.48"

[target.'cfg(unix)'.dependencies]
# Unix 特定依赖
libc = "0.2"

[build]
# 编译选项
opt_level = 2              # 优化级别：0-3
debug = true               # 是否包含调试信息
lto = false                # 链接时优化
codegen_units = 16         # 代码生成单元数（并行编译）
incremental = true         # 增量编译
target = "x86_64-pc-windows-msvc"  # 目标三元组

# 编译器标志
[build.flags]
warnings = "all"           # 警告级别：none/default/all/pedantic
warnings_as_errors = false
overflow_checks = true     # 整数溢出检查
bounds_checks = true       # 数组边界检查

[profile.dev]
opt_level = 0
debug = true
incremental = true

[profile.release]
opt_level = 3
debug = false
lto = true
codegen_units = 1

[profile.bench]
opt_level = 3
debug = false
lto = true

# 构建脚本配置
[build-script]
# build.ccc 脚本的配置
env = { PROTOC = "/usr/bin/protoc" }
```

## 依赖类型

### 运行时依赖（dependencies）

```toml
[dependencies]
http = "2.3.1"              # 注册中心依赖
json = { version = "1.5.0", features = ["serde"] }  # 带特性
local = { path = "../mylib" }  # 本地路径
```

### 开发依赖（dev-dependencies）

只在 `ccc test`/`ccc bench` 时使用，不会被依赖此包的其他包引入：

```toml
[dev-dependencies]
test_framework = "0.9.0"
mock_http = "1.0"
```

### 构建依赖（build-dependencies）

用于构建脚本（`build.ccc`）的依赖，编译时执行：

```toml
[build-dependencies]
codegen = "0.5"          # 代码生成工具
protobuf = "3.0"         # Protocol Buffers 编译器
```

### 条件依赖（target-specific）

根据目标平台选择性编译：

```toml
[target.'cfg(windows)'.dependencies]
windows_api = "0.48"

[target.'cfg(unix)'.dependencies]
libc = "0.2"

[target.'cfg(target_arch = "wasm32")'.dependencies]
wasm_bindgen = "0.2"
```

## 模块导入语法

### 导入外部包（从 `package.toml`）

```cpp
// 导入外部依赖包
import http;
import json;

// 使用：模块导出直接进入当前作用域。
// 不使用包名命名空间；同名导出冲突的诊断与显式选择方式待确认。
Client client;
String data = parse("{\"key\": \"value\"}");
```

### 导入本地模块（相对路径）

```cpp
// 导入同一项目内的模块
import ./utils;
import ./models/user;

// 使用
validate(input);
User u = create("Alice");
```

### 核心库无需导入

```cpp
// 这些是语言内置的全局符号，不需要 import
String s = "Hello";
Vector<int32_t> v;
println("{}", s);
unique_ptr<Data> p(new Data(...));
```

## 依赖解析算法

```
1. 读取项目根目录的 package.toml
2. 递归解析所有依赖的 package.toml
3. 构建依赖图（有向无环图）
4. 检测循环依赖 → 编译错误，提示哪些包形成环
5. 版本统一（Minimal Version Selection，类似 Go modules）
6. 生成 package.lock 文件（锁定具体版本和 hash）
```

### 版本选择策略：最小版本选择（MVS）

```
你的项目依赖：
  A@1.2.0
  B@2.0.0

A@1.2.0 依赖：
  common@1.0.0

B@2.0.0 依赖：
  common@1.5.0

→ 选择 common@1.5.0（满足所有约束的最小版本）
```

**版本冲突处理**：

```
你依赖 A@1.0 和 B@2.0
A@1.0 依赖 common@1.0
B@2.0 依赖 common@2.0

→ 编译错误：无法同时满足 common@1.0 和 common@2.0
→ 提示：升级 A 到兼容 common@2.0 的版本，或降级 B
```

## 循环依赖检测

```cpp
// 包 A 依赖 B，B 依赖 A → 编译错误
Error: circular dependency detected:
  myapp → http → json → http

Solution: refactor common interfaces into a separate package
```

**解决方案**：重构公共接口到独立的基础包

```
Before:
  A ←→ B

After:
  A → Base ← B
```

## 幽灵依赖（Phantom Dependencies）

强制显式声明所有直接使用的包，避免隐式依赖传递依赖：

```cpp
// 你的代码直接使用了 json，但 json 是 http 的传递依赖
import json;  // 编译错误："json" not found in [dependencies]

// 必须在 package.toml 中显式声明
[dependencies]
json = "1.5.0"
```

## 编译缓存：全局共享 + 内容寻址

**目录结构**：

```
~/.ccc/cache/
  ├── modules/
  │   ├── http-2.3.1-<hash>.o       # 编译后的目标文件
  │   ├── json-1.5.0-<hash>.o
  │   └── ...
  ├── registry/
  │   └── index.json                # 包索引
  └── src/
      ├── http-2.3.1/               # 源码缓存
      └── json-1.5.0/
```

**缓存策略**：

1. **内容寻址**：`<hash>` 包含（包名、版本、编译选项、依赖树）
2. **全局共享**：同一个包的同一版本全局只编译一次
3. **增量编译**：只重新编译变化的模块
4. **分层缓存**：
   - Debug 和 Release 分开缓存
   - 不同编译选项（优化级别、目标架构）独立缓存

**空间节省示例**：

```
传统方式（每个项目独立编译）：
  project1/target/  → 500MB
  project2/target/  → 500MB
  project3/target/  → 500MB
  Total: 1.5GB

NCC 全局缓存：
  ~/.ccc/cache/modules/  → 500MB
  project1/.ccc/build/   → 10MB（只有本地代码）
  project2/.ccc/build/   → 15MB
  project3/.ccc/build/   → 8MB
  Total: 533MB（节省 64%）
```

**缓存清理**：

```bash
ccc cache --gc          # 清理 30 天未使用的缓存
ccc cache --clear       # 清空所有缓存
ccc cache --stats       # 查看缓存统计
```

## 包注册中心

### 官方注册中心

```toml
[registry]
default = "https://packages.ccc-lang.org"
```

### 自定义源（企业内网）

```toml
[registry]
default = "https://internal.company.com/ccc-packages"

[[registry.mirrors]]
url = "https://cdn.company.com/ccc-mirror"
```

### Git 依赖

```toml
[dependencies]
# head 可以是 tag、branch 或具体的 commit hash
mylib = { git = "https://github.com/user/mylib.git", head = "v1.0.0" }      # tag
another = { git = "https://github.com/user/another.git", head = "main" }     # branch
unstable = { git = "https://github.com/user/unstable.git", head = "a1b2c3d" } # commit

# 省略 head 默认为 "main"
simple = { git = "https://github.com/user/simple.git" }
```

**说明**：
- `head` 统一指定 Git 引用（tag/branch/commit）
- 编译时会解析为具体的 commit SHA，锁定到 `package.lock`
- 避免了 `tag`/`branch`/`rev`/`commit` 等多个字段的混乱

### 本地路径依赖（开发时）

```toml
[dependencies]
mylib = { path = "../mylib" }
```

**注意**：构建脚本的详细说明见 [11-build-system.md](11-build-system.md)。

## 编译配置（Profiles）

不同场景使用不同的编译配置：

### 内置配置

```bash
ccc build              # 使用 dev profile（快速编译）
ccc build --release    # 使用 release profile（优化）
ccc test               # 使用 dev profile + test
ccc bench              # 使用 bench profile（最大优化）
```

### 自定义配置

```toml
[profile.dev]
opt_level = 0              # 不优化
debug = true               # 完整调试信息
incremental = true         # 增量编译
overflow_checks = true     # 运行时溢出检查

[profile.release]
opt_level = 3              # 最大优化
debug = false              # 无调试信息
lto = true                 # 链接时优化
codegen_units = 1          # 单个编译单元（更好的优化）
strip = true               # 移除符号表

[profile.release-with-debug]
inherits = "release"       # 继承 release 配置
debug = true               # 但保留调试信息

[profile.size-optimized]
inherits = "release"
opt_level = "z"            # 优化代码大小
lto = true
strip = true
```

使用自定义配置：

```bash
ccc build --profile=release-with-debug
ccc build --profile=size-optimized
```

## 特性标志（Features）

包可以定义可选特性，让用户选择性编译：

### 定义特性

```toml
[package]
name = "http"
version = "2.0.0"

[features]
default = ["json"]         # 默认启用的特性
json = ["dep:serde"]       # json 特性依赖 serde 包
tls = ["dep:openssl"]      # tls 特性依赖 openssl 包
http2 = []                 # 无额外依赖的特性

[dependencies]
serde = { version = "1.0", optional = true }    # 可选依赖
openssl = { version = "0.10", optional = true }
```

### 使用特性

```toml
[dependencies]
# 使用默认特性
http = "2.0"

# 不使用默认特性
http = { version = "2.0", default-features = false }

# 显式指定特性
http = { version = "2.0", features = ["json", "tls"] }

# 不使用默认，但添加特定特性
http = { version = "2.0", default-features = false, features = ["http2"] }
```

### 代码中条件编译

```cpp
// 使用 comp 检查特性
comp {
    if (has_feature("json")) {
        // 启用了 json 特性时才编译这段代码
    }
}

// 或者在函数级别
comp fn serialize_json<type T>(v: const T&) -> String
    requires has_feature("json")
{
    // 只有启用 json 特性才能调用此函数
}
```

## 工作空间（Workspace）

管理多个相关包：

### 定义工作空间

```toml
# 根目录 package.toml
[workspace]
members = [
    "server",
    "client",
    "common",
]

# 工作空间级别的依赖版本
[workspace.dependencies]
http = "2.3.1"
json = "1.5.0"
```

### 成员包使用工作空间依赖

```toml
# server/package.toml
[package]
name = "server"
version = "0.1.0"

[dependencies]
http = { workspace = true }    # 使用工作空间定义的版本
common = { path = "../common" }
```

### 工作空间结构

```
workspace/
├── package.toml          # 工作空间配置
├── server/
│   ├── package.toml
│   └── src/
├── client/
│   ├── package.toml
│   └── src/
└── common/
    ├── package.toml
    └── src/
```

### 工作空间命令

```bash
# 构建所有成员
ccc build --workspace

# 测试所有成员
ccc test --workspace

# 只构建特定成员
ccc build -p server
ccc build -p client

# 发布工作空间（按依赖顺序）
ccc publish --workspace
```

## 命令行工具

### 项目管理

```bash
# 创建新项目
ccc new myapp          # 创建可执行程序项目
ccc new --lib mylib    # 创建库项目

# 项目结构
myapp/
  ├── package.toml
  ├── src/
  │   └── main.ccm
  └── tests/
```

### 构建与运行

```bash
ccc build              # 构建项目
ccc build --release    # 发布构建（优化）
ccc run                # 构建并运行
ccc run --release      # 发布模式运行
ccc test               # 运行测试
ccc bench              # 运行基准测试
ccc clean              # 清理构建产物
```

### 依赖管理

```bash
ccc add http@2.3.1     # 添加依赖
ccc add json           # 添加最新版本
ccc remove http        # 移除依赖
ccc update             # 更新所有依赖到兼容的最新版本
ccc update http        # 更新指定依赖
ccc tree               # 显示依赖树
```

### 发布

```bash
ccc publish            # 发布到注册中心
ccc login              # 登录注册中心
ccc logout             # 登出
```

## 语义化版本

遵循 [SemVer 2.0](https://semver.org/)：

```
版本格式：MAJOR.MINOR.PATCH

1.2.3 → 1.2.4   # Patch：向后兼容的 bug 修复
1.2.3 → 1.3.0   # Minor：向后兼容的新功能
1.2.3 → 2.0.0   # Major：不兼容的 API 变更
```

**版本约束语法**：

```toml
[dependencies]
http = "2.3.1"         # 精确版本
json = "^1.5.0"        # 兼容版本：>= 1.5.0, < 2.0.0
log = "~0.4.2"         # 补丁版本：>= 0.4.2, < 0.5.0
fmt = ">=1.0, <2.0"    # 范围
```

## `package.lock` 文件

锁定确切的依赖版本和完整性校验：

```toml
# 自动生成，不要手动编辑
version = 1

[[package]]
name = "http"
version = "2.3.1"
checksum = "sha256:abc123..."
dependencies = ["json@1.5.0"]

[[package]]
name = "json"
version = "1.5.0"
checksum = "sha256:def456..."
dependencies = []
```

**作用**：
- 确保团队成员使用相同的依赖版本
- CI/CD 环境可重现构建
- 防止供应链攻击（checksum 验证）

## 模块可见性

```cpp
// math.ccm
export module math;

// 导出（公开 API）
export double square(double x) {
    return x * x;
}

// 内部实现（不导出）
double internal_helper(double x) {
    return x * 2;
}
```

## 完整示例

**项目结构**：

```
myapp/
  ├── package.toml
  ├── package.lock
  ├── src/
  │   ├── main.ccm
  │   ├── utils.ccm
  │   └── models/
  │       └── user.ccm
  └── tests/
      └── user_test.ccm
```

**package.toml**：

```toml
[package]
name = "myapp"
version = "0.1.0"

[dependencies]
http = "2.3.1"
json = "1.5.0"
```

**src/main.ccm**：

```cpp
// 导入外部包
import http;
import json;

// 导入本地模块
import ./utils;
import ./models/user;

int main() {
    // 使用外部包
    Client client;
    String response = client.get("https://api.example.com/users");
    
    // 解析 JSON
    auto data = parse(response);
    
    // 使用本地模块
    User user = from_json(data);
    println("User: {}", user.name);
    
    return 0;
}
```

**src/models/user.ccm**：

```cpp
export module models.user;

export struct User {
    uint64_t id;
    String name;
    String email;
};

export User from_json(const Value& v) {
    return User{
        .id = v["id"].as_u64(),
        .name = v["name"].as_string(),
        .email = v["email"].as_string()
    };
}
```

