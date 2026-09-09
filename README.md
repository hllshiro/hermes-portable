# Hermes Agent Portable Build

自动从 [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) 上游 release 构建**完全离线**的便携版安装包。

## 工作原理

1. **每 2 小时**自动检查上游 release
2. 发现新版本后自动构建 Windows / Linux 离线便携包（内含 Python、Node.js、Git、uv、ripgrep、ffmpeg、全部依赖的 uv 离线缓存、Playwright Chromium）
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

- **完全离线安装**：首次 `setup-offline` 从包内 uv 缓存以 `uv sync --locked --offline` 重建环境，依赖版本严格锁定于上游 uv.lock
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
