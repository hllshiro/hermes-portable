# hermes-portable Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 创建 `hllshiro/hermes-portable` 仓库：GitHub Actions 自动跟踪 NousResearch/hermes-agent 上游 release，构建 Windows/Linux x64 完全离线便携包并发布到 Releases。

**Architecture:** 仓库只含 README + 3 个 workflow + `.last-built-version`（openclaw-portable 模式）。CI 在上游 tag 上组装离线包：源码 + 捆绑运行时（Python standalone / Node / MinGit / uv / ripgrep / ffmpeg）+ 全部 PyPI wheel（uv.lock 导出，带哈希）+ 预装 node_modules + Playwright Chromium + 启动器脚本。目标机首次运行 setup-offline 用 `uv sync --extra all --locked --offline` 从包内 wheels 重建 venv（保留上游 SHA256 哈希校验），之后 hermes.bat/.sh 直接启动。

**Tech Stack:** GitHub Actions（windows-latest / ubuntu-latest）、uv、pip download、npm workspaces、Playwright、7z/tar、gh CLI。

**Spec:** `docs/superpowers/specs/2026-09-09-hermes-portable-design.md`

## Global Constraints

- **不修改上游源码**：只打包与外层脚本；上游仓库 `NousResearch/hermes-agent`
- Python 固定 **3.11**（上游 `requires-python = ">=3.11,<3.14"`）
- Node CI 用 **22**（上游 engines `^22.22.0 || ^24.11.0 || >=26.0.0`）
- 依赖安装命令固定 `uv sync --extra all --locked`（哈希校验，供应链防线不降级）
- 根项目 PEP 517 构建后端 `setuptools==83.0.0` + `wheel` 必须额外下载到 wheels/（离线构建根项目用）
- npm 范围固定 `npm install --workspace ui-tui --workspace web`（同上游 install.sh）
- 产物命名：`hermes-portable-win-x64-v<版本>.zip`、`hermes-portable-linux-x64-v<版本>.tar.gz`
- 已核实事实：上游 `releases/latest` 可用（当前 v2026.9.7）；uv.lock 中 **0 个 git 源依赖**；入口 console script `hermes = hermes_cli.main:main`，`hermes --version` 可用；Playwright 为 Node 侧（`npx playwright install chromium`，同上游 install.sh）
- gh CLI 已登录账号 **hllshiro**；仓库 **public**
- 本机（Windows）无 python，YAML 校验用 `npx --yes js-yaml <file>`（node v22 可用）
- 工作目录：`C:\Users\18037\Documents\Default Project\hermes-portable`（git 已 init，spec 已提交）
- 上游参考克隆在 `C:\Users\18037\AppData\Local\Temp\opencode\hermes-agent`，openclaw-portable 在 `C:\Users\18037\AppData\Local\Temp\opencode\openclaw-portable`（对照用，只读）

---

### Task 1: 仓库脚手架（README / .gitignore / .last-built-version）

**Files:**
- Create: `README.md`
- Create: `.gitignore`
- Create: `.last-built-version`

**Interfaces:**
- Produces: `.last-built-version`（check-and-build.yml 读写此文件，内容为上游 tag，如 `v2026.9.7`）

- [ ] **Step 1: 写 README.md**

````markdown
# Hermes Agent Portable Build

自动从 [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) 上游 release 构建**完全离线**的便携版安装包。

## 工作原理

1. **每 2 小时**自动检查上游 release
2. 发现新版本后自动构建 Windows / Linux 离线便携包（内含 Python、Node.js、Git、uv、ripgrep、ffmpeg、全部依赖 wheel、Playwright Chromium）
3. 构建产物自动发布到 [Releases](https://github.com/hllshiro/hermes-portable/releases)
4. 也支持手动触发，指定任意上游 tag

## 下载

前往 [Releases](https://github.com/hllshiro/hermes-portable/releases) 页面下载最新版本。

---

## 使用说明

### Windows

1. 下载 `hermes-portable-win-x64-v*.zip`
2. 解压到任意目录（路径避免特殊字符）
3. 双击或在命令行运行：

```cmd
:: 首次安装（完全离线，约 1-3 分钟）
setup-offline.bat

:: 启动交互式 CLI
hermes.bat

:: 启动消息网关（Telegram / Discord 等）
scripts\gateway.bat
```

### Linux

前置要求：系统自带 git（其余运行时均已捆绑）。

```bash
tar -xzf hermes-portable-linux-x64-v*.tar.gz
cd hermes-portable

# 首次安装（完全离线）
./setup-offline.sh

# 启动
./hermes.sh
./scripts/gateway.sh
```

### 离线环境配置模型

离线环境无法访问云端 API，请指向内网模型服务（Ollama / vLLM / llama.cpp 等 OpenAI 兼容端点）：

```bash
hermes model
# 选择 Custom endpoint，填入如 http://localhost:11434/v1，模型名与上下文长度按实际填写
```

或直接编辑 `data/config.yaml`（`HERMES_HOME` 默认为包内 `data/` 目录）：

```yaml
model:
  default: qwen3.5:27b
  provider: custom
  base_url: http://localhost:11434/v1
```

---

## 便携版特点

- **完全离线安装**：首次 `setup-offline` 从包内 wheel 缓存重建环境，保留上游 uv.lock 的 SHA256 哈希校验
- 无需安装 Python / Node.js / Git / uv / ripgrep / ffmpeg
- 内含 Playwright Chromium（浏览器工具离线可用）
- `HERMES_HOME` 默认指向包内 `data/`，解压即用、可整目录迁移（迁移后重跑一次 `setup-offline`）
- 不修改上游源码，venv 在目标机生成（无路径重定位问题）

## 已知限制

- 超出上游 `all` extra 策展集的运行时懒安装依赖（lazy_deps）在离线环境不可用
- `hermes update` 需要联网，离线环境不适用（请下载新版便携包）
- Linux 浏览器工具需要目标机具备常规 Chromium 运行库（glibc 发行版一般满足）

## 致谢

本项目不修改 Hermes Agent 源码，仅提供构建打包服务。Hermes Agent 由 [NousResearch](https://github.com/NousResearch/hermes-agent) 团队开发，MIT 许可。
````

- [ ] **Step 2: 写 .gitignore**

```
# 构建产物（本地试验时生成）
hermes-portable/
*.zip
*.tar.gz
node_modules/
```

- [ ] **Step 3: 写 .last-built-version（空文件占位）**

内容为空（首次 check 时任何上游 tag 都视为新版本）。用命令创建：

```powershell
New-Item -ItemType File -Path .last-built-version -Force
```

- [ ] **Step 4: 校验与提交**

```powershell
git add README.md .gitignore .last-built-version
git commit -m "feat: repo scaffold (README, gitignore, version record)"
```

---

### Task 2: build.yml（可复用构建工作流，Win + Linux + release）

**Files:**
- Create: `.github/workflows/build.yml`

**Interfaces:**
- Consumes: inputs `tag`（上游 tag，如 `v2026.9.7`）、`version`（去 v 版本号）、`node_version`（默认 '22'）、`python_version`（默认 '3.11'）
- Produces: Release 资产 `hermes-portable-win-x64-v<version>.zip`、`hermes-portable-linux-x64-v<version>.tar.gz`；包内脚本 `hermes.bat/.sh`、`setup-offline.bat/.sh`、`scripts/gateway.bat/.sh`、`offline-uv.toml`、`VERSION`、`README-PORTABLE.txt`（check/manual 工作流以 `workflow_call` 调用本文件）

- [ ] **Step 1: 写 .github/workflows/build.yml（完整内容如下）**

```yaml
name: Build Portable Packages

on:
  workflow_call:
    inputs:
      tag:
        required: true
        type: string
      version:
        required: true
        type: string
      node_version:
        required: false
        type: string
        default: '22'
      python_version:
        required: false
        type: string
        default: '3.11'

permissions:
  contents: write

jobs:
  build-windows:
    name: Build Windows
    runs-on: windows-latest
    steps:
      - name: Checkout upstream source
        uses: actions/checkout@v4
        with:
          repository: NousResearch/hermes-agent
          ref: ${{ inputs.tag }}
          path: source
          fetch-depth: 1

      - name: Setup uv (with Python ${{ inputs.python_version }})
        uses: astral-sh/setup-uv@v5
        with:
          python-version: ${{ inputs.python_version }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node_version }}

      - name: Create portable skeleton
        shell: bash
        run: |
          mkdir -p hermes-portable/runtime/{python,node,git,uv,tools}
          mkdir -p hermes-portable/{wheels,browsers,scripts,data}
          cp -r source hermes-portable/app
          echo "Skeleton ready:"; ls hermes-portable

      - name: Bundle Python runtime
        shell: bash
        run: |
          uv python install ${{ inputs.python_version }}
          PYDIR=$(uv python dir)
          SRC=$(ls -d "$PYDIR"/cpython-${{ inputs.python_version }}*windows*x86_64* | head -1)
          echo "Copying Python from: $SRC"
          cp -r "$SRC"/* hermes-portable/runtime/python/
          hermes-portable/runtime/python/python.exe --version

      - name: Bundle Node.js runtime
        shell: bash
        run: |
          NODE_VER=$(node -v | sed 's/v//')
          echo "Downloading Node.js v$NODE_VER..."
          curl -sL -o node.zip "https://nodejs.org/dist/v${NODE_VER}/node-v${NODE_VER}-win-x64.zip"
          unzip -q node.zip -d node-tmp
          mv node-tmp/node-v${NODE_VER}-win-x64/* hermes-portable/runtime/node/
          rm -rf node.zip node-tmp
          hermes-portable/runtime/node/node.exe --version

      - name: Bundle MinGit
        shell: bash
        run: |
          ASSET=$(curl -s https://api.github.com/repos/git-for-windows/git/releases/latest \
            | grep browser_download_url | grep -i mingit | grep 64-bit | grep -v busybox | head -1 | sed 's/.*: "//;s/".*//')
          echo "MinGit asset: $ASSET"
          [ -n "$ASSET" ] || { echo "ERROR: MinGit asset not found"; exit 1; }
          curl -sL -o mingit.zip "$ASSET"
          unzip -q mingit.zip -d hermes-portable/runtime/git
          rm mingit.zip
          hermes-portable/runtime/git/cmd/git.exe --version

      - name: Bundle uv, ripgrep, ffmpeg
        shell: bash
        run: |
          cp "$(which uv)" hermes-portable/runtime/uv/uv.exe
          RG_URL=$(curl -s https://api.github.com/repos/BurntSushi/ripgrep/releases/latest \
            | grep browser_download_url | grep x86_64-pc-windows-msvc.zip | head -1 | sed 's/.*: "//;s/".*//')
          echo "ripgrep asset: $RG_URL"
          [ -n "$RG_URL" ] || { echo "ERROR: ripgrep asset not found"; exit 1; }
          curl -sL -o rg.zip "$RG_URL"
          unzip -q rg.zip -d rg-tmp
          find rg-tmp -name rg.exe -exec cp {} hermes-portable/runtime/tools/ \;
          rm -rf rg.zip rg-tmp
          curl -sL -o ffmpeg.zip "https://www.gyan.dev/ffmpeg/builds/ffmpeg-release-essentials.zip"
          unzip -q ffmpeg.zip -d ff-tmp
          find ff-tmp -name ffmpeg.exe -exec cp {} hermes-portable/runtime/tools/ \;
          find ff-tmp -name ffprobe.exe -exec cp {} hermes-portable/runtime/tools/ \;
          rm -rf ffmpeg.zip ff-tmp
          hermes-portable/runtime/uv/uv.exe --version
          hermes-portable/runtime/tools/rg.exe --version | head -1
          hermes-portable/runtime/tools/ffmpeg.exe -version | head -1

      - name: Export locked requirements and download wheels
        shell: bash
        working-directory: source
        run: |
          uv export --extra all --locked --no-emit-project --no-annotate -o ../requirements-export.txt
          python -m pip download -r ../requirements-export.txt -d ../hermes-portable/wheels
          # Root project PEP 517 build backend (needed to build hermes-agent itself offline)
          python -m pip download "setuptools==83.0.0" wheel -d ../hermes-portable/wheels
          echo "Wheel count: $(ls ../hermes-portable/wheels | wc -l)"

      - name: Install Node dependencies into app
        shell: bash
        working-directory: hermes-portable/app
        run: |
          npm install --workspace ui-tui --workspace web --silent || {
            echo "::warning::scoped npm install failed, falling back to root install"
            npm install --silent
          }

      - name: Bundle Playwright Chromium
        shell: bash
        working-directory: hermes-portable/app
        run: |
          npx playwright install chromium || echo "::warning::playwright chromium download failed; browser tool will need a system browser"
          if [ -d "$LOCALAPPDATA/ms-playwright" ]; then
            cp -r "$LOCALAPPDATA/ms-playwright"/* ../browsers/
            echo "Bundled browsers:"; ls ../browsers
          fi

      - name: Generate launcher scripts (Windows)
        shell: pwsh
        run: |
          @"
          @echo off
          setlocal
          set "ROOT=%~dp0"
          if not exist "%ROOT%app\venv\Scripts\hermes.exe" (
            echo [Hermes] Offline install not found. Run setup-offline.bat first.
            exit /b 1
          )
          set "PATH=%ROOT%runtime\python;%ROOT%runtime\node;%ROOT%runtime\git\cmd;%ROOT%runtime\uv;%ROOT%runtime\tools;%PATH%"
          set "PLAYWRIGHT_BROWSERS_PATH=%ROOT%browsers"
          if not defined HERMES_HOME set "HERMES_HOME=%ROOT%data"
          "%ROOT%app\venv\Scripts\hermes.exe" %*
          "@ | Out-File "hermes-portable\hermes.bat" -Encoding ASCII

          @"
          @echo off
          setlocal
          set "ROOT=%~dp0"
          echo [Hermes] Offline install starting (1-3 min)...
          set "PATH=%ROOT%runtime\uv;%ROOT%runtime\python;%ROOT%runtime\node;%ROOT%runtime\git\cmd;%ROOT%runtime\tools;%PATH%"
          set "UV_OFFLINE=1"
          set "UV_PYTHON=%ROOT%runtime\python\python.exe"
          set "UV_FIND_LINKS=%ROOT%wheels"
          set "UV_CONFIG_FILE=%ROOT%offline-uv.toml"
          set "UV_PROJECT_ENVIRONMENT=%ROOT%app\venv"
          set "PLAYWRIGHT_BROWSERS_PATH=%ROOT%browsers"
          cd /d "%ROOT%app"
          uv sync --extra all --locked --offline
          if errorlevel 1 (
            echo [Hermes] Offline install FAILED. See output above.
            exit /b 1
          )
          if not exist "%ROOT%data" mkdir "%ROOT%data"
          if not exist "%ROOT%app\.env" if exist "%ROOT%app\.env.example" copy "%ROOT%app\.env.example" "%ROOT%app\.env" >nul
          echo [Hermes] Verifying...
          "%ROOT%app\venv\Scripts\hermes.exe" --version
          if errorlevel 1 (
            echo [Hermes] Verification FAILED.
            exit /b 1
          )
          echo [Hermes] Offline install complete. Run hermes.bat to start.
          "@ | Out-File "hermes-portable\setup-offline.bat" -Encoding ASCII

          @"
          @echo off
          set "SCRIPT_DIR=%~dp0"
          call "%SCRIPT_DIR%..\hermes.bat" gateway %*
          "@ | Out-File "hermes-portable\scripts\gateway.bat" -Encoding ASCII

      - name: Write offline config, VERSION, portable README
        shell: bash
        run: |
          cat > hermes-portable/offline-uv.toml << 'EOF'
          # hermes-portable: force fully-offline resolution.
          # Wheels come from the bundled wheels/ directory (UV_FIND_LINKS,
          # absolute path set by setup-offline / launcher scripts).
          offline = true
          EOF
          cat > hermes-portable/VERSION << EOF
          upstream_tag=${{ inputs.tag }}
          version=${{ inputs.version }}
          built=$(date -u +%Y-%m-%dT%H:%M:%SZ)
          built_by=hermes-portable CI (https://github.com/hllshiro/hermes-portable)
          EOF
          cat > hermes-portable/README-PORTABLE.txt << 'EOF'
          Hermes Agent Portable (offline build)
          =====================================
          Windows: run setup-offline.bat first (once), then hermes.bat.
          Gateway: scripts\gateway.bat
          Model config for offline networks: hermes.bat model -> Custom endpoint
          Data/config lives in .\data\ (HERMES_HOME).
          See the repository README for details:
          https://github.com/hllshiro/hermes-portable
          EOF

      - name: Smoke test offline install (empty uv cache)
        shell: pwsh
        run: |
          $env:UV_CACHE_DIR = Join-Path $env:RUNNER_TEMP "uv-cache-smoke"
          New-Item -ItemType Directory -Force -Path $env:UV_CACHE_DIR | Out-Null
          cmd /c "hermes-portable\setup-offline.bat"
          if ($LASTEXITCODE -ne 0) { throw "setup-offline.bat failed with exit code $LASTEXITCODE" }
          cmd /c "hermes-portable\hermes.bat --version"
          if ($LASTEXITCODE -ne 0) { throw "hermes.bat --version failed with exit code $LASTEXITCODE" }
          Write-Host "Smoke test PASSED"

      - name: Clean generated state before packaging
        shell: bash
        run: |
          rm -rf hermes-portable/app/venv hermes-portable/data
          rm -f hermes-portable/app/.env
          find hermes-portable -type d -name __pycache__ -prune -exec rm -rf {} +
          mkdir -p hermes-portable/data

      - name: Create zip
        shell: bash
        run: |
          ZIP="hermes-portable-win-x64-v${{ inputs.version }}.zip"
          7z a -tzip "$ZIP" hermes-portable -mx=7
          ls -lh "$ZIP"

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: windows
          path: hermes-portable-win-x64-*.zip
          retention-days: 5

  build-linux:
    name: Build Linux
    runs-on: ubuntu-latest
    steps:
      - name: Checkout upstream source
        uses: actions/checkout@v4
        with:
          repository: NousResearch/hermes-agent
          ref: ${{ inputs.tag }}
          path: source
          fetch-depth: 1

      - name: Setup uv (with Python ${{ inputs.python_version }})
        uses: astral-sh/setup-uv@v5
        with:
          python-version: ${{ inputs.python_version }}

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node_version }}

      - name: Assemble portable package
        shell: bash
        run: |
          DIR="hermes-portable"
          mkdir -p "$DIR/runtime"/{python,node,uv,tools} "$DIR"/{wheels,browsers,scripts,data}
          cp -r source "$DIR/app"

          # Python runtime (uv-managed standalone build, relocatable)
          uv python install ${{ inputs.python_version }}
          PYDIR=$(uv python dir)
          SRC=$(ls -d "$PYDIR"/cpython-${{ inputs.python_version }}*x86_64*linux*gnu* | head -1)
          echo "Copying Python from: $SRC"
          cp -rL "$SRC"/* "$DIR/runtime/python/"
          "$DIR/runtime/python/bin/python3" --version

          # Node.js runtime
          NODE_VER=$(node -v | sed 's/v//')
          curl -sL -o node.tar.gz "https://nodejs.org/dist/v${NODE_VER}/node-v${NODE_VER}-linux-x64.tar.gz"
          tar -xzf node.tar.gz -C "$DIR/runtime/node" --strip-components=1
          rm node.tar.gz
          "$DIR/runtime/node/bin/node" --version

          # uv
          cp "$(which uv)" "$DIR/runtime/uv/uv"

          # ripgrep (static musl build)
          RG_URL=$(curl -s https://api.github.com/repos/BurntSushi/ripgrep/releases/latest \
            | grep browser_download_url | grep x86_64-unknown-linux-musl.tar.gz | head -1 | sed 's/.*: "//;s/".*//')
          echo "ripgrep asset: $RG_URL"
          [ -n "$RG_URL" ] || { echo "ERROR: ripgrep asset not found"; exit 1; }
          curl -sL -o rg.tar.gz "$RG_URL"
          mkdir -p rg-tmp
          tar -xzf rg.tar.gz -C rg-tmp --strip-components=1
          cp rg-tmp/rg "$DIR/runtime/tools/"
          rm -rf rg.tar.gz rg-tmp

          # ffmpeg (johnvansickle static build)
          curl -sL -o ff.tar.xz "https://johnvansickle.com/ffmpeg/releases/ffmpeg-release-amd64-static.tar.xz"
          mkdir -p ff-tmp && tar -xJf ff.tar.xz -C ff-tmp --strip-components=1
          cp ff-tmp/ffmpeg ff-tmp/ffprobe "$DIR/runtime/tools/"
          rm -rf ff.tar.xz ff-tmp

          "$DIR/runtime/uv/uv" --version
          "$DIR/runtime/tools/rg" --version | head -1
          "$DIR/runtime/tools/ffmpeg" -version | head -1

      - name: Export locked requirements and download wheels
        shell: bash
        working-directory: source
        run: |
          uv export --extra all --locked --no-emit-project --no-annotate -o ../requirements-export.txt
          python -m pip download -r ../requirements-export.txt -d ../hermes-portable/wheels
          python -m pip download "setuptools==83.0.0" wheel -d ../hermes-portable/wheels
          echo "Wheel count: $(ls ../hermes-portable/wheels | wc -l)"

      - name: Install Node dependencies into app
        shell: bash
        working-directory: hermes-portable/app
        run: |
          npm install --workspace ui-tui --workspace web --silent || {
            echo "::warning::scoped npm install failed, falling back to root install"
            npm install --silent
          }

      - name: Bundle Playwright Chromium
        shell: bash
        working-directory: hermes-portable/app
        run: |
          npx playwright install chromium || echo "::warning::playwright chromium download failed; browser tool will need a system browser"
          if [ -d "$HOME/.cache/ms-playwright" ]; then
            cp -r "$HOME/.cache/ms-playwright"/* ../browsers/
            echo "Bundled browsers:"; ls ../browsers
          fi

      - name: Generate launcher scripts (Linux)
        shell: bash
        run: |
          cat > hermes-portable/hermes.sh << 'EOF'
          #!/bin/bash
          DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
          if [ ! -x "$DIR/app/venv/bin/hermes" ]; then
            echo "[Hermes] Offline install not found. Run ./setup-offline.sh first."
            exit 1
          fi
          export PATH="$DIR/runtime/python/bin:$DIR/runtime/node/bin:$DIR/runtime/uv:$DIR/runtime/tools:$PATH"
          export PLAYWRIGHT_BROWSERS_PATH="$DIR/browsers"
          export HERMES_HOME="${HERMES_HOME:-$DIR/data}"
          exec "$DIR/app/venv/bin/hermes" "$@"
          EOF
          chmod +x hermes-portable/hermes.sh

          cat > hermes-portable/setup-offline.sh << 'EOF'
          #!/bin/bash
          set -e
          DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
          echo "[Hermes] Offline install starting (1-3 min)..."
          export PATH="$DIR/runtime/uv:$DIR/runtime/python/bin:$DIR/runtime/node/bin:$DIR/runtime/tools:$PATH"
          export UV_OFFLINE=1
          export UV_PYTHON="$DIR/runtime/python/bin/python3"
          export UV_FIND_LINKS="$DIR/wheels"
          export UV_CONFIG_FILE="$DIR/offline-uv.toml"
          export UV_PROJECT_ENVIRONMENT="$DIR/app/venv"
          export PLAYWRIGHT_BROWSERS_PATH="$DIR/browsers"
          cd "$DIR/app"
          uv sync --extra all --locked --offline
          mkdir -p "$DIR/data"
          [ -f "$DIR/app/.env" ] || { [ -f "$DIR/app/.env.example" ] && cp "$DIR/app/.env.example" "$DIR/app/.env"; }
          echo "[Hermes] Verifying..."
          "$DIR/app/venv/bin/hermes" --version
          echo "[Hermes] Offline install complete. Run ./hermes.sh to start."
          EOF
          chmod +x hermes-portable/setup-offline.sh

          cat > hermes-portable/scripts/gateway.sh << 'EOF'
          #!/bin/bash
          DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
          exec "$DIR/hermes.sh" gateway "$@"
          EOF
          chmod +x hermes-portable/scripts/gateway.sh

      - name: Write offline config, VERSION, portable README
        shell: bash
        run: |
          cat > hermes-portable/offline-uv.toml << 'EOF'
          # hermes-portable: force fully-offline resolution.
          # Wheels come from the bundled wheels/ directory (UV_FIND_LINKS,
          # absolute path set by setup-offline / launcher scripts).
          offline = true
          EOF
          cat > hermes-portable/VERSION << EOF
          upstream_tag=${{ inputs.tag }}
          version=${{ inputs.version }}
          built=$(date -u +%Y-%m-%dT%H:%M:%SZ)
          built_by=hermes-portable CI (https://github.com/hllshiro/hermes-portable)
          EOF
          cat > hermes-portable/README-PORTABLE.txt << 'EOF'
          Hermes Agent Portable (offline build)
          =====================================
          Linux: run ./setup-offline.sh first (once), then ./hermes.sh
          Gateway: ./scripts/gateway.sh
          Requires: system git (all other runtimes bundled)
          Model config for offline networks: ./hermes.sh model -> Custom endpoint
          Data/config lives in ./data/ (HERMES_HOME).
          See the repository README for details:
          https://github.com/hllshiro/hermes-portable
          EOF

      - name: Smoke test offline install (empty uv cache)
        shell: bash
        run: |
          export UV_CACHE_DIR="$RUNNER_TEMP/uv-cache-smoke"
          mkdir -p "$UV_CACHE_DIR"
          bash hermes-portable/setup-offline.sh
          ./hermes-portable/hermes.sh --version
          echo "Smoke test PASSED"

      - name: Clean generated state before packaging
        shell: bash
        run: |
          rm -rf hermes-portable/app/venv hermes-portable/data
          rm -f hermes-portable/app/.env
          find hermes-portable -type d -name __pycache__ -prune -exec rm -rf {} +
          mkdir -p hermes-portable/data

      - name: Create tarball
        shell: bash
        run: |
          TARBALL="hermes-portable-linux-x64-v${{ inputs.version }}.tar.gz"
          tar -czf "$TARBALL" hermes-portable
          ls -lh "$TARBALL"

      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: linux
          path: hermes-portable-linux-x64-*.tar.gz
          retention-days: 5

  release:
    name: Publish release
    needs: [build-windows, build-linux]
    runs-on: ubuntu-latest
    steps:
      - name: Download all artifacts
        uses: actions/download-artifact@v4
        with:
          path: artifacts

      - name: Fetch upstream release body
        id: upstream
        run: |
          BODY=$(curl -s https://api.github.com/repos/NousResearch/hermes-agent/releases/tags/${{ inputs.tag }} \
            | python3 -c "import sys,json; print(json.load(sys.stdin).get('body',''))" 2>/dev/null || echo "")
          echo "$BODY" > upstream_body.txt
          echo "Upstream body fetched ($(wc -c < upstream_body.txt) bytes)."

      - name: Create Release
        uses: softprops/action-gh-release@v2
        with:
          tag_name: "v${{ inputs.version }}"
          name: "Hermes Agent Portable v${{ inputs.version }}"
          body: |
            > 基于上游 [NousResearch/hermes-agent ${{ inputs.tag }}](https://github.com/NousResearch/hermes-agent/releases/tag/${{ inputs.tag }}) 的**完全离线**便携构建。

            ## 下载 / Downloads

            | 平台 Platform | 文件 File |
            |----------|------|
            | Windows x64 | `hermes-portable-win-x64-v${{ inputs.version }}.zip` |
            | Linux x64 | `hermes-portable-linux-x64-v${{ inputs.version }}.tar.gz` |

            ## 使用 / Usage

            - **Windows**: 解压 → `setup-offline.bat`（首次，离线安装）→ `hermes.bat`
            - **Linux**: 解压 → `./setup-offline.sh` → `./hermes.sh`（需系统 git）
            - 离线环境请配置本地模型端点：`hermes model` → Custom endpoint（Ollama / vLLM 等）

            详见 [README](https://github.com/hllshiro/hermes-portable#使用说明)。

            ---
            ⚠️ 自动化便携构建，不修改上游源码。上游发布说明：https://github.com/NousResearch/hermes-agent/releases/tag/${{ inputs.tag }}
          files: |
            artifacts/**/*.zip
            artifacts/**/*.tar.gz
          draft: false
          prerelease: false
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```

- [ ] **Step 2: YAML 语法校验**

```powershell
npx --yes js-yaml .github/workflows/build.yml | Out-Null; if ($LASTEXITCODE -eq 0) { "build.yml OK" }
```

Expected: 输出 `build.yml OK`，无解析错误。

- [ ] **Step 3: 提交**

```powershell
git add .github/workflows/build.yml
git commit -m "feat: reusable build workflow (windows + linux + release)"
```

---

### Task 3: check-and-build.yml + manual-build.yml

**Files:**
- Create: `.github/workflows/check-and-build.yml`
- Create: `.github/workflows/manual-build.yml`

**Interfaces:**
- Consumes: `./.github/workflows/build.yml`（workflow_call，inputs: tag, version）；`.last-built-version`（Task 1）
- Produces: 定时/手动触发构建的入口

- [ ] **Step 1: 写 .github/workflows/check-and-build.yml**

```yaml
name: Check upstream release

on:
  schedule:
    - cron: '0 */2 * * *'  # Every 2 hours
  workflow_dispatch:

permissions:
  contents: write

jobs:
  check:
    name: Check for new release
    runs-on: ubuntu-latest
    outputs:
      has_new: ${{ steps.check.outputs.has_new }}
      tag: ${{ steps.check.outputs.tag }}
      version: ${{ steps.check.outputs.version }}
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Check upstream release
        id: check
        run: |
          LAST_BUILT=$(cat .last-built-version 2>/dev/null || echo "")
          echo "Last built: ${LAST_BUILT:-none}"

          LATEST_TAG=$(curl -s https://api.github.com/repos/NousResearch/hermes-agent/releases/latest \
            | grep '"tag_name"' | head -1 | sed 's/.*: "//;s/".*//')
          echo "Upstream latest: $LATEST_TAG"

          if [ -z "$LATEST_TAG" ]; then
            echo "Failed to fetch upstream release"
            echo "has_new=false" >> $GITHUB_OUTPUT
            exit 0
          fi

          if [ "$LATEST_TAG" = "$LAST_BUILT" ]; then
            echo "Already built, skipping."
            echo "has_new=false" >> $GITHUB_OUTPUT
          else
            echo "New version detected!"
            echo "has_new=true" >> $GITHUB_OUTPUT
          fi

          VERSION="${LATEST_TAG#v}"
          echo "tag=$LATEST_TAG" >> $GITHUB_OUTPUT
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "Result: $LATEST_TAG (version: $VERSION)"

      - name: Update version record
        if: steps.check.outputs.has_new == 'true'
        run: |
          echo "${{ steps.check.outputs.tag }}" > .last-built-version
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add .last-built-version
          git commit -m "build: ${{ steps.check.outputs.tag }}" || echo "No changes"
          git push

  build:
    name: Build portable packages
    needs: check
    if: needs.check.outputs.has_new == 'true'
    uses: ./.github/workflows/build.yml
    with:
      tag: ${{ needs.check.outputs.tag }}
      version: ${{ needs.check.outputs.version }}
    secrets: inherit
```

- [ ] **Step 2: 写 .github/workflows/manual-build.yml**

```yaml
name: Manual Build

on:
  workflow_dispatch:
    inputs:
      upstream_tag:
        description: 'Upstream tag (e.g. v2026.9.7). Leave empty to use latest release.'
        required: false
        default: ''
      node_version:
        description: 'Node.js major version'
        required: false
        default: '22'

permissions:
  contents: write

jobs:
  resolve:
    name: Resolve version
    runs-on: ubuntu-latest
    outputs:
      tag: ${{ steps.resolve.outputs.tag }}
      version: ${{ steps.resolve.outputs.version }}
    steps:
      - name: Resolve target tag and generate version
        id: resolve
        run: |
          TAG="${{ github.event.inputs.upstream_tag }}"

          if [ -z "$TAG" ]; then
            TAG=$(curl -s https://api.github.com/repos/NousResearch/hermes-agent/releases/latest \
              | grep '"tag_name"' | head -1 | sed 's/.*: "//;s/".*//')
          fi

          if [ -z "$TAG" ]; then
            echo "ERROR: Could not resolve tag"
            exit 1
          fi

          TIMESTAMP=$(date -u +%Y%m%d%H%M%S)
          TAG_VERSION="${TAG#v}"
          VERSION="${TAG_VERSION}-build.${TIMESTAMP}"

          echo "tag=$TAG" >> $GITHUB_OUTPUT
          echo "version=$VERSION" >> $GITHUB_OUTPUT
          echo "Building: $TAG (version: $VERSION)"

  build:
    name: Build portable packages
    needs: resolve
    uses: ./.github/workflows/build.yml
    with:
      tag: ${{ needs.resolve.outputs.tag }}
      version: ${{ needs.resolve.outputs.version }}
      node_version: ${{ github.event.inputs.node_version }}
    secrets: inherit
```

- [ ] **Step 3: YAML 语法校验**

```powershell
npx --yes js-yaml .github/workflows/check-and-build.yml | Out-Null; if ($LASTEXITCODE -eq 0) { "check OK" }
npx --yes js-yaml .github/workflows/manual-build.yml | Out-Null; if ($LASTEXITCODE -eq 0) { "manual OK" }
```

Expected: `check OK`、`manual OK`。

- [ ] **Step 4: 提交**

```powershell
git add .github/workflows/check-and-build.yml .github/workflows/manual-build.yml
git commit -m "feat: scheduled upstream check + manual build workflows"
```

---

### Task 4: 创建 GitHub 仓库并推送

**Files:**
- 无新文件（远端操作）

**Interfaces:**
- Consumes: Task 1–3 的全部提交；gh CLI（已登录 hllshiro）
- Produces: `https://github.com/hllshiro/hermes-portable`（public，含全部 workflow）

- [ ] **Step 1: 创建远端仓库并推送**

```powershell
gh repo create hermes-portable --public --source . --push --description "Offline portable builds of NousResearch/hermes-agent (Windows/Linux x64), auto-tracked from upstream releases"
```

Expected: 输出仓库 URL，push 成功。若提示已存在，改用：

```powershell
git remote add origin https://github.com/hllshiro/hermes-portable.git
git push -u origin master
```

- [ ] **Step 2: 验证远端内容**

```powershell
gh api repos/hllshiro/hermes-portable/contents/.github/workflows --jq ".[].name"
```

Expected: 列出 `build.yml`、`check-and-build.yml`、`manual-build.yml`。

---

### Task 5: 触发首次构建、监控迭代、验证发布

**Files:**
- Modify（仅在 CI 失败需修复时）: `.github/workflows/build.yml`

**Interfaces:**
- Consumes: manual-build.yml（workflow_dispatch）；上游 tag `v2026.9.7`（当前 latest，执行时以实际 latest 为准）
- Produces: 首个 Release（含 win zip + linux tar.gz 两个资产）

- [ ] **Step 1: 触发手动构建**

```powershell
gh workflow run manual-build.yml --repo hllshiro/hermes-portable -f upstream_tag=v2026.9.7
Start-Sleep -Seconds 10
gh run list --repo hllshiro/hermes-portable --workflow manual-build.yml --limit 1
```

Expected: 出现新 run（queued/in_progress）。

- [ ] **Step 2: 监控构建直至完成**

```powershell
$runId = (gh run list --repo hllshiro/hermes-portable --workflow manual-build.yml --limit 1 --json databaseId --jq ".[0].databaseId")
gh run watch $runId --repo hllshiro/hermes-portable --exit-status
```

Expected: 全部 job 成功。构建时间预计 20–40 分钟（wheel 下载 + Chromium + 双平台冒烟测试）。

- [ ] **Step 3: 失败则迭代修复**

失败时定位日志：

```powershell
gh run view $runId --repo hllshiro/hermes-portable --log-failed | Select-Object -First 100
```

常见故障与预案（逐个排除，每次修复后 commit + push + 重新触发 Step 1）：
- **`uv export` 参数不兼容**：改用 `uv export --extra all --locked --no-emit-project` 去掉 `--no-annotate`，或查 `uv export --help` 调整
- **pip download 哈希失败 / 仅 sdist 包**：对失败包单独 `pip download <pkg>==<ver> -d wheels --no-deps`，或在 export 时加 `--no-hashes` 并在 VERSION 中注明（最后手段，优先保哈希）
- **Python runtime glob 未命中**（`cpython-3.11*…` 目录名格式变化）：在 CI 日志中打印 `ls $(uv python dir)` 修正 glob
- **MinGit 资产名变化**：调整 grep 过滤条件（保持排除 busybox）
- **ffmpeg/gyan.dev 或 johnvansickle 下载失败**：重试；必要时换 BtbN/FFmpeg-Builds GitHub release
- **冒烟测试 `uv sync --offline` 解析失败**：确认 `UV_FIND_LINKS` 为绝对路径、wheels 目录完整；查缺哪个包补哪个
- **包体超 2GB**：在两个 build job 的打包步骤前排除非运行目录：`rm -rf hermes-portable/app/{website,evals,mcp-research-data,datagen-config-examples,tests,tests-js,docs}`（README 注明）
- **npm workspace 失败**：已有 root install 兜底；若 Electron 等无关 postinstall 拖慢/失败，加 `--omit=dev`

- [ ] **Step 4: 验证 Release 资产**

```powershell
gh release view --repo hllshiro/hermes-portable --json tagName,assets --jq "{tag: .tagName, assets: [.assets[].name]}"
```

Expected: tag `v2026.9.7`，资产含 `hermes-portable-win-x64-v2026.9.7.zip` 与 `hermes-portable-linux-x64-v2026.9.7.tar.gz`，各自体积 < 2GB。

- [ ] **Step 5: 验证 check-and-build 定时链路（可选但推荐）**

```powershell
gh workflow run check-and-build.yml --repo hllshiro/hermes-portable
```

首次运行会因 `.last-built-version` 为空判定"新版本"并再次构建 v2026.9.7——属预期；确认 bot 提交了 `.last-built-version` 更新：

```powershell
gh api repos/hllshiro/hermes-portable/contents/.last-built-version --jq ".content" | ForEach-Object { [Text.Encoding]::UTF8.GetString([Convert]::FromBase64String($_)) }
```

Expected: 输出 `v2026.9.7`。第二次触发应输出 "Already built, skipping."。

- [ ] **Step 6: 最终提交（如有修复）**

```powershell
git status
git log --oneline -5
```

Expected: 工作区干净，所有修复已提交并推送。
