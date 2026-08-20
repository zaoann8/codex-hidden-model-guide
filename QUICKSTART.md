# Codex 隐藏模型启用速查（详细版）

> 精简原理与完整 FAQ 见 [README.md](README.md)。本文按步骤给出可直接复制的命令与预期结果。

## 一、这是什么、有什么用

Codex 的模型目录由 OpenAI 服务器下发到本地 `~/.codex/models_cache.json`，其中部分模型标记为 `visibility: "hide"`。这些隐藏模型是 OpenAI 内部路由别名：

- 输出质量与对应公开模型一致（同一套权重）
- 走不同的计费路径，**不占 Plus / Pro / Team 账号的 5 小时 / 每周用量窗口**
- 消费明细中不出现对应记录

本工具做的事：复制一份目录、把 `hide` 改成 `list`、让 Codex 读取改过的目录。本质是"把官方已下发、仅被隐藏的条目显示出来"，不伪造请求、不修改认证，属于本地展示配置。

| 组件                       | 位置                   | 作用                                                     |
| -------------------------- | ---------------------- | -------------------------------------------------------- |
| `models_auto_visible.json` | `~/.codex/`            | 所有隐藏模型改为可见的模型目录                           |
| `model_catalog_json` 一行  | `~/.codex/config.toml` | 让 Codex 读取自定义目录而非官方缓存                      |
| `auto-model-cache.py`      | `~/.codex/`            | 每小时同步：直连官方端点拉最新目录，新隐藏模型自动变可见 |
| 定时任务                   | launchd / cron         | 每小时运行一次同步脚本                                   |

**前置条件**：ChatGPT Plus / Pro / Team 付费订阅。Free 账号会被服务端直接拒绝，本地配置无法绕过。

## 二、必须装 CLI 吗？

**不必须。** Codex 桌面版（ChatGPT App 内的 Codex）、IDE 插件、CLI 三者共用同一个配置根目录 `~/.codex`（CODEX_HOME），`config.toml` 里的 `model_catalog_json` 对它们都生效。本工具的安装器只依赖系统自带 python3。

| 场景           | 桌面版 / IDE 插件   | CLI                                   |
| -------------- | ------------------- | ------------------------------------- |
| 登录 Plus 账号 | App 内直接登录      | `codex login`                         |
| 生成模型缓存   | App 里正常用一次    | 终端跑一次 `codex`                    |
| 安装本工具     | 同一个 `install.sh` | 同一个 `install.sh`                   |
| 选用隐藏模型   | 模型选择器里点选    | 选择器点选，或 `codex --model <slug>` |
| 命令行探针验证 | 不支持              | 支持（见第四节）                      |

两点差异需要知道：

1. **登录态**：同步脚本优先用 `auth.json` 里的 access_token 直连官方端点拉取最新目录；取不到 token 时自动回退读本地缓存，功能不缺失，只是感知官方变更慢一些。
2. **命令行按次指定模型**（`codex --model`）只有 CLI 有。只用桌面版的话，在模型选择器里按次点选即可，效果相同。

## 三、启用步骤

### 路线 A：只用桌面版 / IDE 插件（不装 CLI）

**第 1 步：确认已登录 Plus 账号**

打开 ChatGPT App 的 Codex 功能，确认右上角账号是 Plus / Pro / Team。未登录就先在 App 内登录。

**第 2 步：用一次 Codex 生成缓存**

随便发一句话。完成后检查缓存文件存在：

```bash
ls -la ~/.codex/models_cache.json
```

看到文件即成功。看不到说明 Codex 还没向服务器要过模型目录，多用几轮再查。

**第 3 步：一键安装**

```bash
git clone https://github.com/zaoann8/codex-hidden-model-guide.git
cd codex-hidden-model-guide
chmod +x install.sh
./install.sh
```

安装器依次执行：检查缓存存在 -> 安装同步脚本到 `~/.codex/auto-model-cache.py` -> 首次运行生成 `models_auto_visible.json` -> 注册每小时定时任务。每步都有 `==>` 输出，报错会明确说缺什么。

**第 4 步：改 config.toml（手动加一行）**

安装器**不会**自动改 config.toml。编辑 `~/.codex/config.toml`，在**顶层**（任何 `[section]` 之前）加一行：

```toml
model_catalog_json = "/Users/你的用户名/.codex/models_auto_visible.json"
```

注意：TOML 不支持变量，必须写完整绝对路径。改之前建议备份：

```bash
cp ~/.codex/config.toml ~/.codex/config.toml.before-hidden-$(date +%Y%m%d-%H%M%S).bak
```

**第 5 步：彻底重启 Codex**

从菜单栏 / 系统托盘**彻底退出** Codex（不是点窗口关闭按钮），重新打开。新建任务，打开模型选择器，隐藏模型已出现。

### 路线 B：CLI 用户

```bash
# 1. 安装并登录
npm i -g @openai/codex
codex login                # 浏览器弹出，用 Plus 账号登录

# 2. 刷新模型缓存
codex                      # 随便问一句话后退出

# 3. 安装本工具（同路线 A 第 3 步）
git clone https://github.com/zaoann8/codex-hidden-model-guide.git
cd codex-hidden-model-guide
chmod +x install.sh
./install.sh

# 4. config.toml 加 model_catalog_json 一行（同路线 A 第 4 步）

# 5. 彻底退出 Codex 后重开（同路线 A 第 5 步）
```

## 四、验证是否生效

**看隐藏模型列表**（最直接）：

```bash
python3 -c "
import json
d = json.load(open('$HOME/.codex/models_auto_visible.json'))
for m in d['models']:
    print(m['slug'], m['visibility'])
"
```

预期：目标隐藏模型为 `list`，原可见模型保持 `list` 不变。

**CLI 探针**（确认服务端接受你的账号，把 slug 换成实际的）：

```bash
codex --ask-for-approval never exec \
  --ignore-user-config --ephemeral --json --skip-git-repo-check \
  -C "$TMPDIR" --sandbox read-only \
  --model <隐藏模型slug> \
  -c 'model_reasoning_effort="low"' \
  -c 'approval_policy="never"' \
  -c 'web_search="disabled"' \
  -c 'features.shell_tool=false' \
  -c 'features.multi_agent=false' \
  -c 'project_doc_max_bytes=0' \
  'Reply exactly HIDDEN_OK. Do not call any tool.'
```

| 输出                     | 含义                             |
| ------------------------ | -------------------------------- |
| `HIDDEN_OK` + 退出码 0   | 服务端接受该账号，可以正常使用   |
| `model is not supported` | 账号无权限（大概率是 Free 账号） |
| 401                      | 登录过期，重新登录               |
| 额度超限                 | 不能据此判断隐藏模型是否可用     |

**桌面版验证**：模型选择器选中隐藏模型，发一句消息，正常回复即生效。

## 五、日常使用

**按次选用（推荐）**：每次在模型选择器里选中隐藏模型，或 CLI 用 `codex --model <slug>`。不要写进 `model = "..."` 设为默认——隐藏路由随时可能被官方关闭，设为默认会让整个 Codex 挂掉；按次选用则失效后随时切回正常模型。

**手动触发同步**（不想等定时任务时）：

```bash
python3 ~/.codex/auto-model-cache.py
```

同步逻辑：优先直连官方端点 `/backend-api/codex/models` 拉最新目录（实时感知官方新增/移除模型），官方源不可用时回退本地缓存。只把 `hide` 改成 `list`，其他字段一律保留；内容无变化时零写入。

**查看同步日志**：`~/.codex/log/auto-model-cache.log`（超 1MB 自动轮转，超 7 天自动清理）。

## 六、常见问题排查

**选择器里看不到隐藏模型？**

1. Codex 彻底退出重启了吗？（菜单栏/托盘图标也要退）
2. `models_auto_visible.json` 存在吗？`ls ~/.codex/models_auto_visible.json`
3. config.toml 的 `model_catalog_json` 路径写对了吗？必须是完整绝对路径，且在顶层
4. 缓存里本来就没有隐藏模型？用第四节的列表命令看实际有什么

**报 `model is not supported`？**
服务端认为账号无权限。确认订阅状态是 Plus / Pro / Team；Free 账号无法绕过。

**报 401？**
登录过期，CLI 重新 `codex login`，桌面版在 App 内重新登录。

**找不到教程里的 `gpt-5.6-sol-wm`？**
slug 不固定。那是教程编写时下发的模型名，不同账号/时间下发的隐藏模型不同，以自己缓存里实际出现的为准（用第四节的列表命令查看）。

**CLI 升级后失效？**
新版本可能改变目录字段结构。同步脚本会自动从官方源重新拉取适配；手动跑一次 `python3 ~/.codex/auto-model-cache.py` 即可。

**想只暴露指定模型、不显示 auto-review 等其他隐藏项？**
见 [README.md FAQ Q8](README.md#q8-严格模式只暴露指定模型不显示-auto-review)。

## 七、风险须知

1. **"不消耗额度"是社区持续观察的结论**，非官方承诺，OpenAI 可能在服务端随时掐断该路由。能用就多用，别依赖它做关键路径。
2. 目前无封号案例：本地仅改 `visibility` 一个字段，不改认证、不伪造请求。但是否违反服务条款由 OpenAI 认定，自担风险。
3. 想加深推理可加 `model_reasoning_effort = "max"`；想开快速模式可加 `service_tier = "fast"` + `[features] fast_mode = true`。均为可选项，不影响本工具效果。

## 八、回滚

最小回滚：删除 `config.toml` 中的 `model_catalog_json` 一行，彻底重启 Codex。原有 `model` 默认值从未被改动，不受影响。

完整回滚：

```bash
# 恢复 config.toml 备份（如备份过）
BACKUP=$(ls -t ~/.codex/config.toml.before-hidden-*.bak 2>/dev/null | head -1)
[ -n "$BACKUP" ] && cp "$BACKUP" ~/.codex/config.toml

# 删除可见目录与定时任务
rm -f ~/.codex/models_auto_visible.json
launchctl bootout "gui/$(id -u)/com.ck.auto-model-cache" 2>/dev/null || true   # macOS
crontab -l | grep -v "auto-model-cache.py" | crontab -                         # Linux
```

彻底退出 Codex -> 重启，即完全还原。
