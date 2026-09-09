# hermes-portable 设计文档

日期：2026-09-09
状态：已与用户确认（方案 B）

## 目标

创建 `hermes-portable` 仓库，参照 [openclaw-portable](https://github.com/hllshiro/openclaw-portable) 的模式，
为 [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) 提供**完全离线**的便携版：
GitHub Actions 自动跟踪上游 release，构建 Windows / Linux x64 离线安装包并发布到 Releases。
目标机在无外网环境下解压 → 首次安装 → 直接使用。

**原则：不修改上游源码**，只做打包与外层脚本（与 openclaw-portable 一致）。

## 背景：hermes-agent 在线安装依赖分析

上游 Windows 安装器（`scripts/install.ps1`，12 个 stage）需要联网获取：

| 构件 | 来源 | 离线对策 |
|------|------|----------|
| uv | astral.sh / GitHub releases | 包内 `runtime/uv/` |
| Python 3.11（>=3.11,<3.14） | uv 管理的 python-build-standalone | 包内 `runtime/python/` |
| MinGit（~45MB） | git-for-windows releases | 包内 `runtime/git/` |
| Node.js ≥22 | nodejs.org | 包内 `runtime/node/` |
| ripgrep / ffmpeg | winget/choco/scoop | 包内 `runtime/tools/` |
| 源码仓库（~1.2 万文件） | GitHub | 包内 `app/` |
| Python 依赖 `uv sync --extra all --locked` | PyPI（uv.lock SHA256 哈希校验） | 包内 `wheels/` |
| node_modules（ui-tui、web workspace） | npm | CI 预装进 `app/` |
| Playwright Chromium | playwright CDN | 包内 `browsers/` |
| 运行时 lazy_deps 按需装的 provider 扩展包 | PyPI | `--extra all` wheel 已覆盖策展集，离线安装后 lazy_deps 命中已装包 |
| LLM API | 云端 provider | 用户配置内网 Ollama/vLLM（`provider: custom`，上游原生支持） |

Linux（`scripts/install.sh`）同构，差异见下文。

## 架构决策（已确认）

- **平台**：Windows x64 + Linux x64
- **构建方式**：GitHub Actions 自动构建（定时轮询上游 + 手动触发），不做本地构建脚本
- **依赖范围**：完整打包（`--extra all` 全部 wheel + node_modules + Playwright Chromium + 全部工具二进制）
- **打包策略**：方案 B —— wheel 缓存 + 首次启动离线安装。
  否决方案 A（直接打包 CI 产物 venv）：venv 不可重定位（pyvenv.cfg、Scripts/*.exe 内嵌绝对路径）。
  否决方案 C（Docker 镜像）：离线目标机不一定有 Docker，且与"解压即用"模式不符。
- **仓库归属**：GitHub 账号 `hllshiro`（gh CLI 已登录），仓库名 `hermes-portable`

## 离线包结构

Windows（Linux 同构，`.bat`→`.sh`、二进制换平台版本）：

```
hermes-portable/
├── hermes.bat            # 日常入口：设 PATH/HERMES_HOME → venv python 运行 cli.py
├── setup-offline.bat     # 首次安装（幂等，可重跑）
├── scripts/gateway.bat   # 启动消息网关
├── runtime/
│   ├── python/           # python-build-standalone 3.11 win-x64
│   ├── node/             # Node.js LTS win-x64（engines: >=22）
│   ├── git/              # MinGit
│   ├── uv/uv.exe
│   └── tools/            # rg.exe、ffmpeg.exe（含 DLL）
├── app/                  # hermes-agent 源码（上游 tag 原样，浅 .git，含预装 node_modules）
│   └── venv/             # setup-offline 生成（不入包）
├── data/                 # 默认 HERMES_HOME（配置、会话、记忆；不入包，首建）
├── wheels/               # uv.lock 导出的 --extra all 全部 wheel（带哈希）
├── browsers/             # Playwright Chromium（ms-playwright 缓存目录原样）
├── offline-uv.toml       # find-links 指向 wheels/，强制离线解析
├── VERSION               # 上游 tag + 构建时间 + 构建来源
└── README-PORTABLE.txt   # 快速上手（离线场景）
```

### 首次安装流程（setup-offline）

1. 设环境变量：`UV_OFFLINE=1`、`UV_NO_CONFIG=1`、`UV_PYTHON=<runtime/python>`、
   `UV_CONFIG_FILE=<offline-uv.toml>`、`VIRTUAL_ENV=<app/venv>`、
   `PLAYWRIGHT_BROWSERS_PATH=<browsers/>`
2. `uv sync --extra all --locked --offline`
   —— 保留上游 uv.lock 的 SHA256 哈希校验（供应链防线不降级），wheel 全部来自包内 `wheels/`
3. 写 `hermes` 启动 shim；创建 `data/`；从 `.env.example` 生成 `.env`（若不存在）
4. 冒烟自检：`venv python cli.py --version`；失败输出明确诊断并保留日志

### 启动器（hermes.bat / hermes.sh）

- 检测 `app/venv` 不存在 → 提示先运行 setup-offline
- `runtime/` 下全部目录前置 PATH（git、rg、ffmpeg、node 立即可见）
- `HERMES_HOME` 默认指向包内 `data/`（真正便携、不写系统目录），已有环境变量则尊重之
- 转发全部参数给 `venv python cli.py %*`

### Linux 差异

- ffmpeg / ripgrep 用静态构建（johnvansickle ffmpeg、ripgrep 官方 musl 静态包）
- git 依赖目标机系统自带（README 注明前置要求；离线 Linux 服务器普遍有 git）
- python-build-standalone 换 linux-x86_64（gnu）构建
- 产物 `.tar.gz`，脚本 `.sh` + `chmod +x`

## CI 工作流

仓库仅含：

```
hermes-portable/
├── README.md
├── .last-built-version
├── docs/superpowers/specs/…（本设计文档）
└── .github/workflows/
    ├── check-and-build.yml   # cron 每 2 小时 + workflow_dispatch
    ├── build.yml             # workflow_call 可复用（inputs: tag, version, node_version）
    └── manual-build.yml      # workflow_dispatch 指定任意上游 tag
```

### check-and-build.yml

- `GET /repos/NousResearch/hermes-agent/releases/latest` 取 tag；
  **回退**：若上游无 GitHub release（仅 tag），改查 `/tags` 取最新（实施时先探测确认）
- 与 `.last-built-version` 比对，新版则 bot 提交版本号并调用 build.yml
- 网络失败静默跳过（has_new=false）

### build.yml — Windows job（windows-latest）

1. checkout 上游 tag（`fetch-depth: 1`，保留浅 `.git`）
2. 装 Node（inputs.node_version，默认取上游 engines 下限 22）+ uv + Python 3.11
3. 下载运行时到 `runtime/`：
   - python-build-standalone 3.11（astral-sh/python-build-standalone release，win-x64 install_only）
   - Node LTS win-x64 zip（nodejs.org，与 runner 同大版本）
   - MinGit（git-for-windows 最新 release 的 MinGit-*-64-bit.zip）
   - uv.exe（astral-sh/uv release，与构建用 uv 同版本）
   - ripgrep（BurntSushi release win zip）、ffmpeg（gyan.dev essentials zip）
4. wheels：`uv export --extra all --locked --no-emit-project -o requirements.txt`
   → `pip download -r requirements.txt -d wheels/ --require-hashes`
5. node_modules：`npm install --workspace ui-tui --workspace web`（同上游 install.sh 范围），产物留在 `app/`
6. Playwright：`playwright install chromium` → 拷 `%LOCALAPPDATA%\ms-playwright` 到 `browsers/`
7. 生成启动器 / setup-offline / offline-uv.toml / VERSION / README-PORTABLE.txt
8. **包内冒烟测试**：干净 `HERMES_HOME` + `UV_OFFLINE=1` 跑完整 setup-offline，
   验证 `cli.py --version`；测试失败则 job 失败、不发布
9. `7z a -tzip hermes-portable-win-x64-v<版本>.zip -mx=7` → upload artifact（保留 5 天）

### build.yml — Linux job（ubuntu-latest）

同构；静态 ffmpeg/ripgrep；跳过 MinGit；`tar -czf hermes-portable-linux-x64-v<版本>.tar.gz`。
Linux 侧同样跑 setup-offline 冒烟测试。

### release job

- needs 两端构建；download artifacts
- 抓上游 release notes（无 release 时省略）
- `softprops/action-gh-release@v2` 发布：
  tag `v<版本>`，正文含平台下载表、上游 tag 链接、"自动化便携构建"声明

## 体积与降级预案

预估：wheels 400–700MB + Chromium ~170MB + ffmpeg ~100MB + Node/Python/git/源码 ~400MB，
压缩后应低于 GitHub 单资产 2GB 上限。
**若超限**：打包时排除非运行目录（`website/`、`evals/`、`mcp-research-data/`、
`datagen-config-examples/`、`tests*/`、`docs/`）——仅排除，不改代码；README 注明。

## 错误处理

- 构建期：任一下载/导出步骤 fail-fast，日志明确标注构件与 URL
- 冒烟测试失败 → 不发布 release（artifact 仍可查）
- 用户侧：setup-offline 每步校验 + 诊断输出；可重复运行（幂等）；
  hermes.bat 缺 venv 时引导而非崩溃
- lazy_deps 离线兜底：`--extra all` 已覆盖策展依赖集；超出集合的懒装请求会失败，
  README 说明此为离线环境固有限制

## 测试策略

1. **CI 冒烟**（每次构建必跑）：包内完整 setup-offline + `cli.py --version`，全程 `UV_OFFLINE=1`
2. **发布后人工验收**（首个版本）：在真实断网 Windows 机器解压 → setup-offline →
   `hermes model` 配置本地 Ollama → 完成一轮对话；Linux 同
3. workflow 语法：`actionlint`（若可用）或首次 push 后观察 Actions 运行

## 交付物

1. `hllshiro/hermes-portable` GitHub 仓库（本地创建于
   `C:\Users\18037\Documents\Default Project\hermes-portable`，git init 后推送）
2. README.md（中文，对齐 openclaw-portable 风格：工作原理 / 下载 / 使用说明 / 便携版特点 / 致谢）
3. 三个 workflow 文件
4. `.last-built-version`（初始为空或首建版本号）
5. 首次构建：推送后用 manual-build.yml 手动触发上游最新 tag（v2026.9.7）验证全链路
