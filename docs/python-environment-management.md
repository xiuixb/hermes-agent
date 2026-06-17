# Hermes Agent Python 环境管理指南

> 本文档涵盖 Hermes Agent 的 Python 虚拟环境、依赖管理策略、锁文件工作流及常见问题排查。

---

## 一、方案概览

Hermes Agent 不使用 Conda，而是采用 **uv + venv** 的轻量级方案进行项目隔离。

| 组件 | 工具 | 说明 |
|------|------|------|
| **包管理器** | [uv](https://docs.astral.sh/uv/) | Rust 实现的高速 Python 包管理器 |
| **构建后端** | setuptools | 通过 `pyproject.toml` 配置 |
| **锁文件** | `uv.lock` | 记录所有传递依赖的精确版本 + SHA256 哈希 |
| **虚拟环境** | Python stdlib `venv` | 标准隔离机制，uv 创建并管理 |
| **Python 版本** | uv 自动管理 | 可自动下载并安装 Python 3.11 |

### 为什么不用 Conda

- **Conda 更重**：Conda 管理整个环境（包括 Python 本体和非 Python 库），对纯 Python 项目来说属于过度隔离
- **uv + venv 足够**：Python 的包级隔离已经能满足需求，且启动更快、占用更小
- **跨平台一致性**：uv 在 Linux / macOS / Windows 上的行为完全一致，Conda 则可能因为 channel 差异导致问题
- **CI 友好**：GitHub Actions 中 `uv sync` 远比 conda 环境初始化快

### 项目隔离原理

```
项目目录 (hermes-agent/)
├── .venv/              ← 虚拟环境（所有依赖安装在此）
│   ├── bin/hermes      ← 可执行入口
│   └── lib/python3.11/ ← 所有第三方包
├── pyproject.toml      ← 依赖声明
├── uv.lock             ← 锁文件（SHA256 验证）
└── src/                ← 项目源代码（editable install）
```

`uv sync` 或 `uv pip install -e "."` 之后，所有依赖都安装在 `.venv/` 内，与系统 Python 全局环境完全隔离。激活虚拟环境后运行 `hermes` 命令，实际执行的是 `.venv/bin/hermes`。

---

## 二、开发环境搭建

### 前置条件

| 条件 | 说明 |
|------|------|
| Python 3.11 ~ 3.13 | uv 会自动安装（上限 <3.14，因为部分 Rust 扩展包尚无 cp314 wheel） |
| uv | [安装指南](https://docs.astral.sh/uv/) |
| Git + git-lfs | 克隆仓库和大文件 |
| Node.js 20+（可选） | 浏览器工具和 WhatsApp 桥接需要 |

### 快速开始

```bash
# 1. 克隆仓库
git clone git@github.com:NousResearch/hermes-agent.git
cd hermes-agent

# 2. 一键开发环境搭建（推荐）
./setup-hermes.sh

# 3. 激活环境
source venv/bin/activate   # Linux/macOS
venv\Scripts\activate      # Windows

# 4. 运行
hermes
```

### 手动搭建（了解细节）

```bash
# 1. 创建虚拟环境（Python 3.11）
uv venv .venv --python 3.11

# 2. 激活环境
source .venv/bin/activate      # Linux/macOS
.venv\Scripts\activate         # Windows

# 3. 安装依赖（推荐：hash 验证的锁文件安装）
uv sync --extra all --locked

# 或者：PyPI 实时解析（不验证传递依赖哈希）
uv pip install -e ".[all,dev]"

# 4. 可选：安装浏览器工具
npm install
```

### 虚拟环境命名约定

项目中存在两种命名习惯：

| 名称 | 来源 | 说明 |
|------|------|------|
| `.venv` | README、AGENTS.md 推荐 | 隐藏目录，不会被 `ls` 直接列出 |
| `venv` | CONTRIBUTING.md、`setup-hermes.sh` | 非隐藏，方便查看 |

功能上完全等价，选择一种即可。`scripts/run_tests.sh` 会自动探测 `.venv` → `venv` → `~/.hermes/hermes-agent/venv` 的优先级链。

---

## 三、锁文件（uv.lock）工作流

### 锁文件的作用

`uv.lock` 记录了每个依赖包的精确版本和 SHA256 哈希值。当使用 `uv sync --locked` 安装时，uv 会验证每个包的哈希，确保与锁文件中记录的一致。这意味着：

- **可复现构建**：任何人在任何机器上安装，得到的都是完全相同的依赖集合
- **供应链安全**：即使 PyPI 上的某个传递依赖被恶意篡改，哈希不匹配会导致安装失败，而非静默安装恶意包

### 常用命令

```bash
# 根据 pyproject.toml 生成/更新锁文件
uv lock

# 使用锁文件安装（严格模式：不允许锁文件过期）
uv sync --locked --extra all

# 使用锁文件安装（冻结模式：连 pyproject.toml 也不检查）
uv sync --frozen

# CI 中校验锁文件是否与 pyproject.toml 一致
uv lock --check
```

### 何时需要重新生成锁文件

当你做了以下操作后需要运行 `uv lock`：

1. 修改了 `pyproject.toml` 中的依赖版本（bump pin）
2. 添加了新的核心依赖或 optional dependency
3. 移除了某个依赖

CI 中 `uv-lockfile-check` workflow 会自动检查锁文件是否同步。

---

## 四、依赖管理策略

### 精确锁定（Exact Pinning）

所有核心依赖都使用 `==X.Y.Z` 精确版本，不使用版本范围。这是 **2026-05-12** 之后强化的安全策略，直接原因是 [Mini Shai-Hulud 蠕虫](https://en.wikipedia.org/wiki/Mini_Shai-Hulud) 感染了 PyPI 上的 `mistralai==2.4.6`。

```toml
# 正确：精确锁定
"openai==2.24.0"

# 禁止：版本范围
"openai>=2.0.0,<3"     # 恶意版本可以落入此范围
```

### 依赖分层

| 层级 | 位置 | 策略 |
|------|------|------|
| **核心依赖** | `[project.dependencies]` | 每个 session 都会用到的包，精确锁定 |
| **可选依赖（extras）** | `[project.optional-dependencies]` | 按功能分组（`anthropic`、`vision`、`mcp` 等） |
| **懒加载依赖** | `tools/lazy_deps.py` | 首次使用时动态安装（大部分 provider 后端） |
| **dev 依赖** | `[dev]` extra | 测试、lint、类型检查工具 |

### `[all]` extra 策略

`[all]` 只包含**无法懒加载**的 extras — 即每个 session 都会用到的基础设施（CLI、终端、MCP、ACP 等）。所有可选后端（provider、搜索、TTS、图片生成、消息平台）都通过 `tools/lazy_deps.py` 在首次使用时安装。

这样设计的原因：
- 减小一次 PyPI 供应链攻击的爆炸半径
- 避免 `[matrix]` 等包含原生编译依赖（`python-olm` 需要 `make`）的 extra 在 Windows 上导致安装失败

### 添加新依赖的流程

```bash
# 1. 编辑 pyproject.toml，在合适的位置添加依赖
#    - 核心依赖 → [project.dependencies]
#    - 可选依赖 → [project.optional-dependencies] 下对应 extra
#    - 开发工具 → [dev] extra

# 2. 重新生成锁文件
uv lock

# 3. 同步安装
uv sync --extra all --extra dev

# 4. 运行测试确认一切正常
scripts/run_tests.sh
```

---

## 五、安装流程详解

`setup-hermes.sh` 实现了一套多层降级安装策略：

```
第一层：uv sync --extra all --locked
  ├── 成功 → hash 验证安装（最安全）
  └── 失败 ↓
第二层：uv pip install -e ".[all]"
  ├── 成功 → PyPI 实时解析（传递依赖无 hash 验证）
  └── 失败 ↓
第三层：uv pip install -e ".[<safe-extras>]"
  ├── 成功 → 过滤掉已知不可解析的 extras
  └── 失败 ↓
第四层：uv pip install -e "."
  └── 成功 → 仅核心依赖
```

设计理念：一个被污染的 PyPI 包不应该让整个安装静默降级为"核心依赖"。`_BROKEN_EXTRAS` 数组用于标记暂时无法解析的 extra。

---

## 六、常见问题排查

### uv sync 在 Windows 上报错（python-olm 等）

`[all]` extra 特意不包含 `[matrix]`（它依赖需要 `make` 的 `python-olm`）。如果手动指定了 `--all-extras`，Windows 上会尝试从源码编译 `python-olm` 并失败。

**解决**：使用 `--extra all` 而非 `--all-extras`。

### Python 版本不匹配

```bash
# 查看可用 Python 版本
uv python list

# 安装指定版本
uv python install 3.11

# 使用指定版本创建 venv
uv venv --python 3.11
```

如果 uv 自动选择了 3.14 并报错 "no cp314 wheel"，这是因为 `requires-python = ">=3.11,<3.14"` 上限生效了。显式指定 `--python 3.11` 即可。

### 虚拟环境损坏

```bash
# 删除并重建
rm -rf .venv
uv venv .venv --python 3.11
uv sync --extra all --locked
```

### PYTHONPATH 干扰

如果系统设置了 `PYTHONPATH` 环境变量，可能导致包导入混乱。安装脚本会自动清除，手动操作时请注意：

```bash
unset PYTHONPATH
unset PYTHONHOME
```

### 锁文件过期

当你修改了 `pyproject.toml` 但未更新锁文件时，`uv sync --locked` 会报错：

```bash
# 更新锁文件
uv lock

# 再次安装
uv sync --extra all --locked
```

### Termux / Android 环境

Termux 不使用 uv（因为 uv 在 Android 上支持有限），改用标准库 `venv` + `pip`：

```bash
python -m venv venv
source venv/bin/activate
pip install -e ".[termux]" -c constraints-termux.txt
```

`constraints-termux.txt` 为 Android 平台锁定了特定版本的包（如 `ipython`、`jedi` 等），确保在 Termux 上稳定安装。

---

## 七、平台差异

| 场景 | 包管理器 | 虚拟环境 | 说明 |
|------|----------|----------|------|
| **Linux/macOS 开发** | uv | `.venv` / `venv` | 标准流程 |
| **Windows 开发** | uv | `.venv` / `venv` | 避免 `--all-extras`（含 `[matrix]`） |
| **Termux / Android** | pip | stdlib `venv` | uv 支持有限，回退到 pip |
| **Docker** | uv | `/opt/hermes/.venv` | 分层缓存，`uv sync --frozen` |
| **Nix** | nix flake | nix 管理 | `nix develop` 进入开发环境 |
| **PyPI 安装** | pip/uv | 用户自行管理 | `pip install hermes-agent[all]` |

### Windows 特别说明

- `tzdata` 仅在 Windows 上安装（Linux/macOS 自带 IANA 时区数据库）
- `pywinpty` 仅在 Windows 上安装（伪终端支持），非 Windows 使用 `ptyprocess`
- Windows 的 MinGit 由安装脚本自动处理
- 数据目录为 `%LOCALAPPDATA%\hermes`（非 Windows 为 `~/.hermes`）

### Docker 环境

Docker 镜像采用多层缓存构建：

```dockerfile
# 第一层：只复制 pyproject.toml 和 uv.lock（缓存友好）
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-install-project --extra all ...

# 第二层：复制源码并 editable install
COPY . .
RUN uv pip install --no-cache-dir --no-deps -e "."
```

---

## 八、与 Conda 的对比

| 维度 | Conda | uv + venv（Hermes 方案） |
|------|-------|--------------------------|
| **安装速度** | 慢（需解析 conda-forge） | 快（Rust 实现，并行下载） |
| **包来源** | conda-forge / defaults | PyPI |
| **非 Python 依赖** | 支持（C 库、系统工具等） | 不支持（仅 Python 包） |
| **环境隔离** | 完整隔离（Python + 所有包） | 包级隔离 |
| **锁文件** | `environment.yml`（无哈希） | `uv.lock`（含 SHA256） |
| **Python 版本管理** | 内置 | uv 可自动下载 |
| **跨平台** | channel 差异可能导致问题 | 行为一致 |
| **磁盘占用** | 较大 | 较小 |

**结论**：对于 Hermes Agent 这样的纯 Python 项目，uv + venv 完全够用，且更轻量、更安全（哈希验证）、更快速。

---

## 九、参考

- [uv 官方文档](https://docs.astral.sh/uv/)
- [pyproject.toml](../pyproject.toml) — 依赖声明及策略注释
- [setup-hermes.sh](../setup-hermes.sh) — 开发环境一键搭建脚本
- [CONTRIBUTING.md](../CONTRIBUTING.md) — 贡献指南
- [uv.lock](../uv.lock) — 锁文件
- [constraints-termux.txt](../constraints-termux.txt) — Termux 约束文件
