# 十、包管理系统

## 核心主旨：可重现的快速构建

NCC 的包管理系统定义以下目标：
1. **构建速度快**
2. **二进制体积小**
3. **依赖可重现**

**核心原则：**
1. **严格禁止循环依赖** - 促进清晰架构
2. **MVS 版本选择** - 可预测构建
3. **全局缓存 + 内容寻址** - 极速构建
4. **锁定文件保证可重现** - 生产环境安全

## 包管理能力
- ✅ **MVS 算法**：最小版本选择，可预测且稳定
- ✅ **全局缓存**：`~/.ccc/cache/`，所有项目共享已编译的包
- ✅ **快速依赖解析**：依赖图简单，无复杂的版本协商

- ✅ **语义化版本**：`^1.2.3` 表示兼容 1.x
- ✅ **Feature flags**：可选功能，减小二进制体积
- ✅ **锁定文件**：精确版本 + 内容哈希

- ✅ **扁平化依赖树**：避免重复下载相同版本的包
- ✅ **传递依赖精确锁定**：包括完整依赖树

- ✅ **精确版本锁定**：用于生产环境

## 包清单文件：package.toml

```toml
[package]
name = "myapp"
version = "1.0.0"
authors = ["Your Name <you@example.com>"]
edition = "2026"

[dependencies]
http = "2.3"        # 语义化版本：^2.3.0（兼容 2.x）
json = "1.5.0"      # 精确版本
log = { version = "0.4", features = ["json"] }  # 带 feature

[dev-dependencies]
test = "0.9"        # 只在测试时使用
mock = "1.0"

[build-dependencies]
codegen = "0.5"     # 构建脚本依赖

[target.'cfg(windows)'.dependencies]
windows_api = "0.48"

[target.'cfg(unix)'.dependencies]
libc = "0.2"

[profile.dev]
opt_level = 0
debug = true
incremental = true

[profile.release]
opt_level = 3
debug = false
lto = true
codegen_units = 1
```

## 语义化版本规则

```toml
[dependencies]
# 精确版本
json = "1.5.0"           # 只接受 1.5.0

# 兼容范围（必须显式写操作符）
http = "^2.3.0"          # >=2.3.0 <3.0.0

# 波浪号（补丁版本）
log = "~0.4.2"           # >=0.4.2 <0.5.0

# 通配符
util = "1.*"             # 接受 1.x.y
```

**语义化版本兼容性：** 精确版本只匹配自身；`^`、`~` 和通配符分别按其
明确范围匹配。解析器选择满足全部约束的最低可用版本；没有交集则报告冲突。

## 锁定文件：package-lock.toml

自动生成，记录精确的依赖树和内容哈希：

```toml
# package-lock.toml - 自动生成，不要手动编辑
version = 1

[[package]]
name = "http"
version = "2.3.5"
source = "registry+https://packages.ccc-lang.org"
checksum = "sha256:a3f8b9c4d2e1f3b5c8a9d7e2f1c4b6a8..."
dependencies = ["json"]

[[package]]
name = "json"
version = "1.5.0"
source = "registry+https://packages.ccc-lang.org"
checksum = "sha256:7d2b4e1a9c8f5b3d2e9f1c7a4b6d8e2a..."
dependencies = []

[[package]]
name = "log"
version = "0.4.2"
source = "registry+https://packages.ccc-lang.org"
checksum = "sha256:9e1c3f5b7a2d8c4f1e6b9d3a7c5e2f8b..."
features = ["json"]
dependencies = ["json"]

# 编译顺序（拓扑排序结果）
[build-order]
packages = ["json", "http", "log", "myapp"]
```

## 依赖解析流程

```
1. 读取 package.toml
2. 检查 package-lock.toml 是否存在
   - 存在且有效 → 直接使用锁定版本，跳到步骤7
   - 不存在或无效 → 继续
3. 递归解析所有依赖的 package.toml
4. 构建依赖图，检测循环依赖
5. MVS 算法选择版本
6. 生成 package-lock.toml
7. 按拓扑排序编译各包
8. 链接最终二进制
```

## MVS 算法（Minimal Version Selection）

选择满足所有约束的**最小**版本：

```
你的项目 package.toml:
  [dependencies]
  A = "1.2.0"
  B = "2.0.0"

A@1.2.0 的 package.toml:
  [dependencies]
  common = "1.0.0"

B@2.0.0 的 package.toml:
  [dependencies]
  common = "1.5.0"

MVS 算法：
  1. 收集所有对 common 的约束：["^1.0.0", "^1.5.0"]
  2. 求约束交集：[>=1.5.0, <2.0.0]
  3. 选择交集中的最低可用版本
  4. 验证并写入 package-lock.toml
```

**优势：**
- **可预测**：总是选择同一个版本，不依赖解析顺序
- **最小化**：避免不必要的版本升级
- **可解释**：约束交集和冲突来源会写入诊断；新增依赖可能改变传递依赖版本

## 版本冲突处理

```
场景：主版本冲突（不兼容）
  A 依赖 common@1.x
  B 依赖 common@2.x

错误信息：

  error: dependency conflict for 'common'
    Package 'A@1.0' requires common ^1.0
    Package 'B@2.0' requires common ^2.0
  
  note: semantic versioning treats major version changes as breaking
  
  help: update dependencies to use compatible versions:
    - Upgrade 'A' to v2.0+ (if available)
    - Or downgrade 'B' to v1.x
    - Or contact package maintainers
```

## 循环依赖检测与禁止

**NCC 严格禁止包级别的循环依赖**，原因：
1. 避免编译顺序歧义
2. 促进清晰的架构分层
3. 简化依赖图，提升构建性能

**错误报告示例：**

```
error: circular dependency detected

  myapp v1.0.0
    → http v2.3.5
      → auth v1.0.0
        → user v0.5.0
          → http v2.3.5  (cycle back)

note: dependency cycle forms a loop:
  http → auth → user → http

help: refactor common interfaces into a separate base package:
  
  Before (cyclic):
    http ←→ user
  
  After (acyclic):
    http → http-types ← user
```

## 全局缓存结构

```
~/.ccc/
  ├── cache/
  │   ├── registry/
  │   │   └── index.json                    # 包索引
  │   ├── src/
  │   │   ├── http-2.3.5/                   # 源码缓存
  │   │   └── json-1.5.0/
  │   └── build/
  │       ├── http-2.3.5-<hash>.o           # 编译缓存
  │       └── json-1.5.0-<hash>.o
  └── config.toml                            # 全局配置
```

**缓存键计算：**
```
cache_key = sha256(
    package_name,
    package_version,
    source_hash,
    compiler_version,
    optimization_level,
    target_triple,
    dependency_hashes,
    lockfile_hash,
    host_triple,
    enabled_features,
    build_script_inputs,
    external_tool_versions
)
```

**优势：**
1. **跨项目共享**：相同版本的包只编译一次
2. **内容寻址**：依赖变化自动重新编译
3. **增量构建**：只重新编译变化的部分

## Feature Flags（可选功能）

减小二进制体积，按需启用功能：

```toml
# 库的 package.toml
[package]
name = "json"
version = "1.5.0"

[features]
default = ["std"]
std = []              # 标准库支持
serde = ["std"]       # 序列化支持（依赖 std）
no_std = []           # 嵌入式环境

[dependencies]
serde_core = { version = "1.0", optional = true }
```

**使用方：**

```toml
[dependencies]
# 启用 serde feature
json = { version = "1.5", features = ["serde"] }

# 禁用默认 features
json = { version = "1.5", default-features = false, features = ["no_std"] }
```

**编译器行为：**
- 只编译启用的 feature 代码
- 未启用的 feature 通过条件编译排除
- 最终二进制只包含需要的功能

## 模块导入语法

### 导入外部包

```cpp
// 导入外部依赖包（从 package.toml）
import http;
import json;
import another_http as other_http;

// 使用导出的符号
Client client;
String data = parse("{\"key\": \"value\"}");
```

### 导入本地模块

```cpp
// 导入同一项目内的模块（相对路径）
import ./utils;
import ./models/user;

// 使用
validate(input);
User u = create("Alice");
```

### 核心库无需导入

```cpp
// 语言内置的全局符号，不需要 import
String s = "Hello";
Vector<int32_t> v;
println("{}", s);
unique_ptr<Data> p = make_unique<Data>(...);
```

## ccc 命令行工具

```bash
# 创建新项目
ccc new myapp
cd myapp

# 添加依赖
ccc add http@2.3
ccc add json --features serde

# 移除依赖
ccc remove json

# 更新依赖（重新运行 MVS）
ccc update

# 查看依赖树
ccc tree

# 构建项目
ccc build

# 构建并运行
ccc run

# 测试
ccc test

# 清理缓存
ccc clean

# 发布到注册中心
ccc publish
```

## 工作空间（Workspace）

支持多个包在同一仓库中，共享依赖版本：

```toml
# workspace.toml（仓库根目录）
[workspace]
members = [
    "packages/core",
    "packages/http",
    "packages/cli",
]

# 工作空间级别的依赖版本统一
[workspace.dependencies]
json = "1.5.0"
log = "0.4.2"
```

**优势：**
1. **版本统一**：所有成员包使用相同版本的共享依赖
2. **增量构建**：修改一个包只重新编译受影响的包
3. **单一 package-lock.toml**：整个工作空间共享锁定文件

## 性能目标

- **首次构建**（冷缓存）：< 30秒（中型项目，50个依赖）
- **增量构建**（热缓存）：< 3秒
- **依赖解析**：< 1秒

**优化策略：**
1. **并行下载**：同时下载多个包
2. **并行编译**：按拓扑排序并行编译无依赖关系的包
3. **全局缓存**：跨项目共享编译结果
4. **增量编译**：只重新编译变化的模块

## 与其他语言对比

| 特性 | Go | Rust | Node | **NCC** |
|------|----|----|------|---------|
| 版本算法 | MVS | 最新兼容 | 嵌套 | **MVS** |
| 循环依赖 | 允许 | 禁止 | 允许 | **禁止** |
| 缓存位置 | 全局 | 全局 | 本地 | **全局** |
| 锁定文件 | go.sum | Cargo.lock | package-lock | **package-lock.toml** |
| 首次构建 | 快 | 慢 | 快 | **快** |
| 二进制体积 | 大 | 小 | N/A | **小** |

## 下一步

- 查看 [11-build-system.md](11-build-system.md) 了解构建系统
- 查看 [01-modules.md](01-modules.md) 了解模块系统
