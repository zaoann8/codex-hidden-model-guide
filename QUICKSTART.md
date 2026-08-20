# 快速上手：隐藏模型启用指南

> 精简版速查手册。完整原理、故障排查、回滚细节见 [README.md](README.md)。

## 这个工具解决什么

Codex 的模型目录由 OpenAI 服务器下发到本地 `~/.codex/models_cache.json`，其中部分模型标记为 `visibility: "hide"`。这些隐藏模型是 OpenAI 内部路由别名：

- 输出质量与对应公开模型一致（同一套权重）
- 走不同的计费路径，**不占 Plus 账号的 5 小时 / 每周用量窗口**
- 消费明细中不出现对应记录

本工具把目录复制一份、将 `hide` 改为 `list`，再让 Codex 指向改过的目录。本质是"把官方已下发、仅被隐藏的条目显示出来"，不伪造请求、不改认证。

| 组件                                         | 作用                                       |
| -------------------------------------------- | ------------------------------------------ |
| `models_auto_visible.json`                   | 改过可见性的模型目录                       |
| `config.toml` 中的 `model_catalog_json` 一行 | 让 Codex 读取自定义目录                    |
| `auto-model-cache.py` + 定时任务             | 每小时同步，官方新下发的隐藏模型自动变可见 |

## 新电脑四步启用（macOS / Linux）

前置：已有 ChatGPT **Plus / Pro / Team** 订阅（Free 账号会被服务端拒绝，本地配置无法绕过）。

### 1. 安装 Codex 并登录

```bash
npm i -g @openai/codex   # 未安装时
codex login              # 浏览器弹出，用 Plus 账号登录
```

### 2. 刷新模型缓存

```bash
codex
```

随便问一句话后退出。此步骤让官方把模型目录下发到本地。

### 3. 一键安装本工具

```bash
git clone https://github.com/zaoann8/codex-hidden-model-guide.git
cd codex-hidden-model-guide
chmod +x install.sh
./install.sh
```

安装器自动完成：生成可见目录 → 在 `config.toml` 追加 `model_catalog_json` 一行 → 注册每小时定时同步（macOS 为 launchd，Linux 为 crontab）。

### 4. 重启 Codex

**彻底退出**（包括菜单栏 / 系统托盘图标）后重新打开。新建任务 → 打开模型选择器 → 隐藏模型已出现，按次选用。

## 日常使用

```bash
# 按次指定隐藏模型（推荐，不依赖选择器）
codex --model <隐藏模型slug>

# 手动触发一次同步（不想等定时任务时）
python3 ~/.codex/auto-model-cache.py
```

查看当前全部隐藏模型：

```bash
python3 -c "
import json
d = json.load(open('$HOME/.codex/models_auto_visible.json'))
for m in d['models']:
    print(m['slug'], m['visibility'])
"
```

## 风险须知

1. **不要把隐藏模型设为默认**（`model = "..."`）。隐藏路由随时可能被官方关闭，设为默认会导致整个 Codex 不可用；按次选用则失效后随时切回正常模型，无影响。
2. "不消耗额度"基于社区持续观察，非官方承诺，OpenAI 可能在服务端随时掐断。
3. 目前无封号案例，本地仅改可见性一个字段，属于客户端配置行为；但是否违反服务条款由 OpenAI 认定，自担风险。
4. slug 不固定：教程示例中的 `gpt-5.6-sol-wm` 只是当时下发的模型名，实际以自己缓存里出现的为准。
5. Codex 客户端大版本升级后若模型目录字段结构变化，同步脚本会自动从官方源重新拉取适配。

## 回滚

删除 `config.toml` 中的 `model_catalog_json` 一行（或恢复备份 `config.toml.before-hidden-*.bak`），彻底重启 Codex 即可。完整回滚命令见 [README.md](README.md#回滚方法)。
