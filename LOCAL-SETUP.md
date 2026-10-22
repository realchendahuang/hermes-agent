# 本机部署说明（私有 fork）

这是 **realchendahuang/hermes-agent**，取自上游 [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent)。
本文件记录这台 MacBook Air（air）上的实际部署方式、已验证配置和已知坑，供后续自己迭代时参考。

> 上游文档见 `README.md`；本文件只写"这台机器上真实发生了什么"。

---

## 1. 布局

| 项目 | 路径 | 说明 |
|---|---|---|
| 源码（你的 fork） | `~/code/hermes` | 工作目录，改代码就在这里 |
| 启动器 | `~/.local/bin/hermes` | shell shim → `~/code/hermes/.hermes/bin/hermes` |
| PM 依赖环境 | `~/.hermes/installs/<key>/` | Hermes 自己的包管理（PM）管的 generation venv |
| PM 工具链 | `~/.hermes/tools/` | Python 3.14 / Node 26 / ripgrep / ffmpeg / Chromium / cua-driver |
| 用户数据 | `~/.hermes/` | config、session、memory、skills、logs、gateway.sock |
| 密钥 | `~/.hermes/.env` | 权限 600，**不要提交到任何仓库** |
| 网关服务 | `~/Library/LaunchAgents/ai.hermes.gateway.plist` | launchd 托管，**登录自启 + 崩溃重启** |

**磁盘占用**（2026-10-22）：源码 1.6G + installs 962M + tools 950M ≈ **3.5G**。

**不要**在仓库里或别处自建 venv。依赖由 PM 管理，手工 venv 会干扰它的环境判定（已踩过）。

---

## 2. 运行方式（关键：PM 机制）

`~/.local/bin/hermes` 不是普通的 Python 入口，链路是：

```
~/.local/bin/hermes
  └─ ~/code/hermes/.hermes/bin/hermes        # 生成的 shim
       └─ ~/.hermes/tools/python-3.14.7.../bin/python3 -I   # store python，隔离模式
            └─ import hermes_bootstrap → activate_dependencies()
                 └─ site.addsitedir(PM generation 的 site-packages)
                      └─ hermes_cli.main
```

所以：

- **代码改完立即生效**（`sys.path` 直接指向 `~/code/hermes`，没有 editable 重装这一步）。
- 但 **gateway 是常驻进程**（默认空闲 600s 退出）。改了配置、换了模型、动了依赖，要重启它才看得到效果：

```bash
hermes gateway stop     # 停掉常驻 gateway
hermes -z "测试一下"     # 下次调用会自动拉起新的
```

---

## 3. 模型配置

### 当前生效（`~/.hermes/config.yaml`）

```yaml
model:
  provider: minimax-cn            # MiniMax 国内域（api.minimaxi.com）
  default: MiniMax-M3.1-Flash     # context 1M
fallback_model:
  provider: deepseek
  model: deepseek-flash           # 即 DeepSeek-V4.1-Flash
```

### 密钥（`~/.hermes/.env`）

```dotenv
MINIMAX_CN_API_KEY=sk-cp-...
DEEPSEEK_API_KEY=sk-...
```

两个 key 都迁自 **ZCode 的 provider 配置**（`~/.zcode/v2/provider_config.json`），原样可用。

### 已验证

| 链路 | 状态 | 验证方式 |
|---|---|---|
| MiniMax-M3.1-Flash（默认） | ✅ 对话 + **工具调用** | 让它执行 `echo` + `pwd`，正确返回 |
| DeepSeek-V4.1-Flash（fallback） | ✅ 对话 | `hermes --provider deepseek -m deepseek-flash -z "..."` |

### 坑

- **MiniMax 的 key 只对国内域生效**。`api.minimax.io`（global）用这个 key 返回 401，必须走 `minimax-cn`。
- `minimax-cn` 的协议是 **Anthropic Messages**（`api.minimaxi.com/anthropic`），不是 OpenAI 兼容层 —— 所以依赖 `anthropic` SDK（见 §6）。
- MiniMax M3.1 Flash 是 reasoning 模型，`max_tokens` 给小了会被 thinking 吃光、内容为空。日常用不需管（Hermes 不设上限）。
- 静态模型目录 `hermes_cli/models_catalog_static.py` 的 `_MINIMAX_MODELS` 还没有 `MiniMax-M3.1-Flash`（只有 M3/M2.7/…）。不影响使用（`hermes model` 会拉 live `/v1/models`），但想让它出现在静态菜单里就在那里加一行。

### 换模型

```bash
hermes model                                  # 交互式选择（会拉各家 live 模型列表）
hermes config set model.default <model-id>    # 直接改
hermes --provider <p> -m <model> -z "..."     # 单次覆盖
```

以后要接 OpenCode Go / CommandCode 的 key，同样加进 `~/.hermes/.env` 对应变量即可。

---

## 4. 常用命令

```bash
hermes                       # 经典 CLI 交互
hermes -z "问题"              # 单次问答
hermes --continue            # 恢复上一个会话
hermes doctor                # 体检（含 API 连通性）
hermes doctor --fix          # 自动修配置版本等
hermes config get/set <k>    # 读/写配置
hermes tools                 # 开关工具
hermes gateway start/stop/status
hermes logs                  # 看日志
hermes pm doctor             # 检查 PM 依赖/工具状态
hermes pm repair             # 依赖环境坏了时重建
```

`hermes --tui` 需要先构建 `ui-tui`（`npm install` + 构建），当前**未构建**，用经典 CLI。

---

## 5. 自己迭代

```bash
cd ~/code/hermes
git switch -c feat/你的改动      # 建议开分支
# ...改代码...
hermes -z "冒烟测试"
hermes gateway stop             # 改动涉及常驻进程时
```

- 测试：`scripts/run_tests.sh`（需要 `pytest`，PM 的 test extra 已装）
- 代码风格：`ruff` / `ty`（dev 依赖组已装）
- §2 说了：**改完不用重装**，直接从源码加载。

---

## 6. 与上游同步

`origin` = 你自己的 fork，`upstream` = NousResearch 官方。

```bash
cd ~/code/hermes
git fetch upstream
git merge upstream/main        # 或 rebase，看你的洁癖
git push origin main
```

建议把私有改动（文档、配置适配、自定义工具）保持在少量、集中在独立文件/独立模块，减少和上游的冲突面。

> `hermes update` 在 git 安装方式下走的是 `git pull`（对你的 fork）。上游同步请手动走上面三步。

---

## 7. 踩过的坑（复现记录）

| # | 现象 | 原因 | 处理 |
|---|---|---|---|
| 1 | `--version` 报 `OpenAI SDK: Not installed` | pyproject 里 **46 个 core 依赖全部带 `python_version >= '3.14'`**，`requires-python` 的 `>=3.11` 只是给 updater 兼容用的 | 必须用 **Python 3.14**。3.13 装出来是空壳 |
| 2 | `ImportError: The 'anthropic' package is required` | `minimax-cn` 走 Anthropic Messages 协议，但 `anthropic` 是 optional extra，不在 `all` 里 | 让 PM 按 `facts.json` 装（`hermes pm doctor` / 首次启动自动 sync） |
| 3 | 上面那个错误重复出现、重启也没用 | **gateway 是 7:21 启动的常驻进程**，一直用旧环境；后续 `hermes -z` 只是给它发消息 | `hermes gateway stop` 后重试。**换环境/改配置后务必重启 gateway** |
| 4 | `pm/extras.py:233` 报 "this process does not run from the install's dependency environment" | 进程启动时 `sys.path` 里没有 PM generation 的 site-packages；且手工 venv 干扰了判定 | 删掉手工 venv，用官方 `hermes_cli/_launchers.py` 发布启动器（`~/.local/bin/hermes`） |
| 5 | `hermes --tui` 不可用 | `ui-tui` 未构建 | 需要时再 `npm install` + 构建 |
| 6 | `hermes gateway install` 报 `launchctl bootstrap exit 5` | 新 API `launchctl bootstrap gui/501` 在本机报 EIO | 用旧 API 兜底：`launchctl load -w ~/Library/LaunchAgents/ai.hermes.gateway.plist`，之后 `launchctl list \| grep hermes` 能看到，`hermes gateway status` 也会认作 supervised |

---

## 8. Telegram 网关（常驻）

### 当前状态

| 项 | 值 |
|---|---|
| Bot 名称 | **MacBookair** |
| Username | `@MacBookair_Kim_bot`（`t.me/MacBookair_Kim_bot`） |
| Token | 在 `~/.hermes/.env` 的 `TELEGRAM_BOT_TOKEN`（**勿外传**） |
| 托管方式 | launchd LaunchAgent，`RunAtLoad` + `KeepAlive`，登录自启 & 崩溃重启 |
| 连接模式 | long polling（自动探测到你的 `127.0.0.1:7890` 代理并复用） |
| 已注册命令 | 60 条 bot 菜单命令 |

### 生效的配置

```yaml
# ~/.hermes/config.yaml
platforms:
  telegram:
    enabled: true
```

```dotenv
# ~/.hermes/.env
TELEGRAM_BOT_TOKEN=...
TELEGRAM_ALLOWED_USERS=<你的 Telegram 数字 user id>   # 不填则走 DM 配对
```

### 安全模型（重要）

Hermes **默认拒绝未知发送者**，两条路：

1. **白名单**（推荐）：`TELEGRAM_ALLOWED_USERS=123456789`（逗号分隔多个），立即放行
2. **DM 配对**：留空时，陌生人给 bot 发消息会收到一次性配对码，owner 执行
   `hermes pairing approve telegram <code>` 才放行

查看状态：`hermes pairing list`

### 管理命令

```bash
hermes gateway status                     # 是否被 launchd 托管、PID
hermes gateway start / stop / restart     # 控制服务
hermes gateway install / uninstall        # 装/卸 launchd 服务
hermes pairing list                       # 待批准 + 已批准用户
hermes pairing approve telegram <code>    # 批准配对
tail -f ~/.hermes/logs/gateway.log        # 实时日志
```

> ⚠️ 服务一旦由 launchd 托管，**不要**再用 `hermes gateway run` 前台起第二个实例 ——
> 两个进程轮询同一个 bot token 会互相抢 update，Telegram 会报 409 冲突。

---

## 9. 资源占用

| 项 | 实测（2026-10-22） |
|---|---|
| gateway 常驻（含 Telegram） | 主进程 ≈ 234 MB + 辅助进程 ≈ 14 MB，CPU ≈ 1.8% |
| 磁盘 | ≈ 3.5 G |
| 首次启动 | 需要 PM sync（约 1 分钟，之后是秒级） |

本机是 16G 内存的 MacBook Air。网关常驻约 250 MB，可接受；但**不要再叠**第二个常驻网关或桌面端。
如果要加 cron / 多平台（Discord、Slack）这类负载，按 `~/code/AGENTS.md` 的规矩应该搬到 mini。

---

_最后更新：2026-10-22_
