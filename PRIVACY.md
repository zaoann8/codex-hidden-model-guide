# 隐私与安全审查报告

> 对本仓库全部可执行代码的逐文件审查结论。审查日期：2026-08-20。
> 本仓库共 4 个可执行/配置文件，无其他代码（无 GitHub Actions、无依赖包）。

## 审查范围

| 文件                                    | 角色                                 | 行数 |
| --------------------------------------- | ------------------------------------ | ---- |
| `scripts/auto-model-cache.py`           | 核心同步脚本（定时任务执行的就是它） | 299  |
| `install.sh`                            | macOS / Linux 安装器                 | bash |
| `install.ps1`                           | Windows 安装器（PowerShell）         | 89   |
| `launchd/com.ck.auto-model-cache.plist` | macOS 定时任务模板                   | 18   |

## 结论：干净，未发现隐私问题

### 1. 网络行为：全仓库只有一个出站端点

`auto-model-cache.py` 的 `fetch_official_catalog()`（约 150-201 行）是**唯一**发起网络请求的代码：

```
https://chatgpt.com/backend-api/codex/models?client_version=<版本号>
```

- 这是 **OpenAI 官方端点**，与 Codex CLI 自身使用的完全相同
- 启用了标准 TLS 证书校验（`ssl.create_default_context()`，187 行），不存在禁用证书验证的代码
- **没有**任何第三方服务器、遥测、统计上报、analytics、"phone home"逻辑
- `install.sh` / `install.ps1` / launchd plist 自身不发起任何网络请求

### 2. 凭据处理：token 只进官方端点的请求头，不落地、不进日志

脚本读取 `~/.codex/auth.json` 的 `access_token`（或 `OPENAI_API_KEY`），用途仅一处：上述官方请求的 `Authorization: Bearer` 头（180 行）。

逐条核查过全部 `log()` 调用与异常分支：

- 日志内容只包含：模型数量、模型 slug、错误码（如 `官方源 HTTP 401`）、来源描述
- HTTP 错误只记录状态码 `e.code`（197 行），**不记录响应正文**
- 网络错误只记录 `e.reason`（199 行），不含请求头
- token 不会被写入 `models_auto_visible.json`、日志或任何其他文件

### 3. 文件读写：只碰自己的文件，不碰敏感配置

| 文件                                | 读               | 写                                          |
| ----------------------------------- | ---------------- | ------------------------------------------- |
| `~/.codex/models_cache.json`        | ✅（回退数据源） | ❌ 从不写                                   |
| `~/.codex/auth.json`                | ✅（只取 token） | ❌ 从不写                                   |
| `~/.codex/config.toml`              | ❌               | ❌ 从不写（安装器只打印提示让你手动加一行） |
| `~/.codex/models_auto_visible.json` | ✅               | ✅（唯一的业务输出）                        |
| `~/.codex/log/auto-model-cache.log` | —                | ✅（自维护，1MB 轮转 / 7 天清理）           |

写入使用临时文件 + `os.replace` 原子替换（278-282 行），崩溃不会留下半成品。

### 4. 代码形态：全标准库，无动态代码加载

- 仅 import 标准库：`json / os / shutil / ssl / sys / time / urllib / datetime / fcntl / msvcrt`
- 无 `eval` / `exec` / `__import__` / 动态下载执行
- 无 base64/混淆字符串、无压缩 payload
- 脚本 299 行可完整人工审计，无外部依赖

### 5. 定时任务：明文可查，无隐藏参数

- **macOS launchd**：`/usr/bin/python3 <CODEX_ROOT>/auto-model-cache.py <CODEX_ROOT>`，每小时一次（`StartInterval 3600`）
- **Windows 计划任务**：`py -3` / `python` 执行同一脚本，每小时一次
- **Linux crontab**：同上

注册的任务可用系统工具随时查看与卸载：

```bash
# macOS
launchctl list | grep auto-model-cache
cat ~/Library/LaunchAgents/com.ck.auto-model-cache.plist

# Linux
crontab -l | grep auto-model-cache

# Windows(管理员 PowerShell)
Get-ScheduledTask -TaskName 'CodexAutoModelCache' | Select-Object -ExpandProperty Actions
```

## 次要观察（非泄露，供参考）

1. **日志含模型 slug**：即"你的账号能看到哪些模型"。仅存本地、7 天自动清理，不构成隐私风险。
2. **`models_auto_visible.json` 按默认权限（通常 644）写入**：内容是公开模型目录数据、不含任何凭据，多用户机器上他人可读也无害。介意可 `chmod 600 ~/.codex/models_auto_visible.json`。
3. **请求携带 `client_version` 与 `User-Agent: codex/<版本>`**：与 Codex CLI 自身发送的完全一致；这只向 OpenAI（他们本来就知道）暴露你的 Codex 版本。
4. **每小时定时任务意味着 token 每天最多 24 次发往 chatgpt.com**：与 Codex 本身的通信频率相比可忽略，且是同一端点。
5. **`install.ps1` 要求管理员权限**：仅因 `Register-ScheduledTask` 需要；安装器自身无特权操作。

## 自行验证方法

不想信这份报告，可以自己验证：

```bash
# 1. 全仓库搜索所有 URL(应只有 chatgpt.com 一个出站端点)
grep -rn "https\?://" scripts/ install.sh install.ps1

# 2. 确认无动态执行
grep -rn "eval\|exec\|__import__" scripts/

# 3. 跑脚本时监控网络连接
lsof -i -nP | grep python3   # 另一终端运行:python3 ~/.codex/auto-model-cache.py
# 只应看到与 chatgpt.com(含 CDN)的连接

# 4. 检查脚本实际读了哪些文件
# macOS 可用 fs_usage(需 sudo),或简单验证:临时移走 auth.json 后脚本应回退本地缓存
```

## 审查声明

本报告基于 2026-08-20 仓库快照（commit `aada819` 之前的代码状态）逐行人工审查。若后续提交修改了上述文件，请重新核实。
