# 使用文档：只用 ChatGPT App（桌面版）启用隐藏模型

> 适用场景：**新电脑，只装了 ChatGPT App，不装 Codex CLI**。
> 原理、风险、完整 FAQ 见 [README.md](README.md)；代码隐私审查见 [PRIVACY.md](PRIVACY.md)。

## 为什么不装 CLI 也能用

ChatGPT App 内置的 Codex 和命令行版 Codex 共用同一个配置目录 `~/.codex`（CODEX_HOME）：

- App 内 Codex 的模型选择器读取的目录、`config.toml`、登录凭据都在这里
- 本工具改的就是这个目录下的文件，**对 App 内的 Codex 同样生效**
- 安装器只依赖 Python 3（macOS 系统自带），全程不需要 npm / Codex CLI

## 前提条件

| 项目        | 要求                                                                                         |
| ----------- | -------------------------------------------------------------------------------------------- |
| ChatGPT App | 已安装，能在 App 内使用 Codex 功能                                                           |
| 订阅        | Plus / Pro / Team（Free 会被服务端拒绝，无法绕过）                                           |
| Python 3    | 系统自带。首次运行若弹出「安装命令行开发者工具」对话框，点「安装」等它装完（一次性，几分钟） |

## 启用步骤

### 第 1 步：确认 App 已登录 Plus

打开 ChatGPT App -> 使用 Codex 功能 -> 确认当前登录的是 Plus / Pro / Team 账号。

### 第 2 步：用一次 Codex，生成模型缓存

在 App 的 Codex 里随便发一句话（比如"你好"）。这一步让 OpenAI 把模型目录下发到本地。

验证缓存已生成：

```bash
ls -la ~/.codex/models_cache.json
```

能看到文件即成功。看不到就再多聊几轮后重查；仍没有说明 App 版本较老或 Codex 功能未启用，先升级 App。

### 第 3 步：下载本工具

方式一（有 git）：

```bash
git clone https://github.com/zaoann8/codex-hidden-model-guide.git
cd codex-hidden-model-guide
```

方式二（不想装 git，用系统自带 curl 下载 ZIP）：

```bash
curl -L -o /tmp/guide.zip https://github.com/zaoann8/codex-hidden-model-guide/archive/refs/heads/main.zip
unzip /tmp/guide.zip -d /tmp
cd /tmp/codex-hidden-model-guide-main
```

> 两种方式首次运行时 macOS 都可能弹出「命令行开发者工具」安装框（因为要用到 git / python3）。点「安装」，装完继续。

### 第 4 步：运行安装器

```bash
chmod +x install.sh
./install.sh
```

安装器依次执行并打印 `==>` 进度：

1. 检查 `~/.codex/models_cache.json` 存在（不存在会报错，回到第 2 步）
2. 安装同步脚本到 `~/.codex/auto-model-cache.py`
3. 首次运行，生成 `~/.codex/models_auto_visible.json`（所有隐藏模型改为可见）
4. 注册每小时自动同步的定时任务（launchd）

### 第 5 步：改 config.toml（关键手动步骤）

安装器**不会**自动修改 config.toml。复制下面整段执行（自动备份 + 把配置行插入文件最顶部）：

```bash
cp ~/.codex/config.toml ~/.codex/config.toml.before-hidden-$(date +%Y%m%d-%H%M%S).bak
printf 'model_catalog_json = "%s"\n\n' "$HOME/.codex/models_auto_visible.json" \
  | cat - ~/.codex/config.toml > /tmp/codex-cfg-tmp \
  && mv /tmp/codex-cfg-tmp ~/.codex/config.toml
```

> 为什么插在**最顶部**：TOML 的顶层键必须位于任何 `[xxx]` 段之前，插在文件末尾会落入最后一个 section 导致不生效。

验证插入成功（第一行应显示 `model_catalog_json = ...`）：

```bash
head -2 ~/.codex/config.toml
```

### 第 6 步：彻底重启 ChatGPT App

**普通关窗口不算退出**。两种方式任选：

- App 内菜单：ChatGPT -> 退出 ChatGPT（⌘Q）
- 菜单栏 / 程序坞图标右键 -> 退出

然后重新打开 App。

### 第 7 步：验证

App 内新建 Codex 任务 -> 打开模型选择器 -> 应出现原本隐藏的模型，按次点选使用。

也可以先在终端确认目录里有哪些隐藏模型：

```bash
python3 -c "
import json
d = json.load(open('$HOME/.codex/models_auto_visible.json'))
for m in d['models']:
    print(m['slug'], m['visibility'])
"
```

## 日常使用

- **按次选用**：每次在模型选择器里点选隐藏模型。**不要**写进配置设为默认——隐藏路由随时可能被官方关闭，设为默认会让 Codex 整体不可用；按次选用则失效后随时切回正常模型。
- **自动同步**：launchd 每小时运行一次，官方新下发的隐藏模型自动变可见。日志在 `~/.codex/log/auto-model-cache.log`。
- **手动同步**（不想等定时任务时）：

```bash
python3 ~/.codex/auto-model-cache.py
```

## 常见问题

**选择器里看不到隐藏模型？**

1. App 彻底退出重启了吗？（⌘Q 或菜单退出，不是关窗口）
2. `head -2 ~/.codex/config.toml` 第一行是 `model_catalog_json = ...` 吗？
3. `ls ~/.codex/models_auto_visible.json` 存在吗？
4. 用上面的模型列表命令看目录里实际有什么——不同账号 / 时间下发的隐藏模型不同，教程里的 `gpt-5.6-sol-wm` 只是示例 slug，以自己机器上实际出现的为准。

**选中隐藏模型后报 `model is not supported`？**
服务端认为账号无权限。确认订阅是 Plus / Pro / Team；Free 账号本地配置无法绕过。

**报 401？**
登录过期，在 App 内重新登录。

**弹「命令行开发者工具」安装框？**
正常，点「安装」。这是 macOS 第一次使用 python3 / git 的标准流程，装一次以后不再弹。

**App 升级后隐藏模型消失了？**
App 大版本更新可能重写 `~/.codex/config.toml`。检查 `head -2`，若 `model_catalog_json` 行丢了，重新执行第 5 步的那段命令即可。模型目录本身由定时任务自动维护，无需重装。

## 回滚

```bash
# 恢复 config.toml 备份
BACKUP=$(ls -t ~/.codex/config.toml.before-hidden-*.bak 2>/dev/null | head -1)
[ -n "$BACKUP" ] && cp "$BACKUP" ~/.codex/config.toml

# 删除可见目录与定时任务
rm -f ~/.codex/models_auto_visible.json
launchctl bootout "gui/$(id -u)/com.ck.auto-model-cache" 2>/dev/null || true
```

彻底退出 App -> 重启，即完全还原。原有配置从未被修改过（除 `model_catalog_json` 一行），无其他残留。
