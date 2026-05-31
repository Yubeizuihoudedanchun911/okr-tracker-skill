# OKR Tracker Skill

基于飞书 CLI 的 OKR 进度追踪工具，帮助你每周自动对比 OKR 目标与周报中的实际进展，识别落后项，给出优先级建议。

## 功能

- **`/okr-setup`**：季度初创建/录入 OKR 目标，支持从飞书 OKR 系统拉取或手动粘贴文本录入
- **`/okr-review`**：每周分析 OKR 进度，对比飞书周报内容，输出差距分析、进度表格和优化建议

## 前置条件

- [lark-cli](https://github.com/anthropics/cli) 已安装并完成 `lark-cli config init`
- Claude Code CLI

## 安装

```bash
git clone https://github.com/<your-username>/okr-tracker-skill.git
cd okr-tracker-skill

# 建立符号链接
ln -s $(pwd)/okr-setup ~/.claude/skills/okr-setup
ln -s $(pwd)/okr-review ~/.claude/skills/okr-review
ln -s $(pwd)/shared ~/.claude/skills/okr-tracker-shared
```

或在你的 `~/.claude/CLAUDE.md` 中添加引用：

```markdown
# OKR Tracker
- **okr-setup** (`/path/to/okr-tracker-skill/okr-setup/SKILL.md`) — 创建季度 OKR 目标
- **okr-review** (`/path/to/okr-tracker-skill/okr-review/SKILL.md`) — 每周 OKR 进度回顾
```

## 使用

### 1. 创建 OKR 目标（每季度初运行一次）

在 Claude Code 中输入：

```
/okr-setup
```

按提示录入本季度 OKR，支持两种方式：
- 直接粘贴文本（从飞书 OKR 系统复制）
- 提供飞书 OKR 文档 URL，自动拉取

### 2. 每周进度回顾

```
/okr-review
```

自动读取飞书周报文档，对比 OKR 目标，输出进度分析和建议。

### 3. 定时执行（可选）

参考 [examples/cron-setup.md](examples/cron-setup.md) 配置定时任务，支持：
- 系统 crontab
- macOS launchd
- Claude CronCreate（会话内临时）

## 数据存储

所有数据存放在 `~/.claude/okr/`：

| 文件 | 用途 |
|------|------|
| `config.json` | 用户配置（姓名、飞书文档 URL、时区等） |
| `<year>-Q<n>.json` | 季度 OKR 目标与进度数据 |

参考 [examples/sample-okr.json](examples/sample-okr.json) 查看数据格式示例。

## License

MIT
