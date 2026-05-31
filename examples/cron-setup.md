# OKR Tracker 定时任务配置指南

## 方案 1：系统 crontab（推荐长期使用）

```bash
# 编辑 crontab
crontab -e

# 每周五 17:00 执行 OKR 回顾
0 17 * * 5 claude -p "/okr-review" --allowedTools "Bash(lark-cli *),Read,Write"
```

## 方案 2：macOS launchd

创建文件 `~/Library/LaunchAgents/com.okr-tracker.weekly-review.plist`：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.okr-tracker.weekly-review</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/claude</string>
        <string>-p</string>
        <string>/okr-review</string>
        <string>--allowedTools</string>
        <string>Bash(lark-cli *),Read,Write</string>
    </array>
    <key>StartCalendarInterval</key>
    <dict>
        <key>Weekday</key>
        <integer>5</integer>
        <key>Hour</key>
        <integer>17</integer>
        <key>Minute</key>
        <integer>0</integer>
    </dict>
    <key>StandardOutPath</key>
    <string>/tmp/okr-review.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/okr-review.err</string>
</dict>
</plist>
```

加载：

```bash
launchctl load ~/Library/LaunchAgents/com.okr-tracker.weekly-review.plist
```

卸载：

```bash
launchctl unload ~/Library/LaunchAgents/com.okr-tracker.weekly-review.plist
```

## 方案 3：Claude CronCreate（会话内临时）

在 Claude Code 会话中设置，适合短期密集跟踪（最长 7 天有效）：

```
# 在 Claude 会话中执行
# 每周五 17:00 触发
CronCreate: cron="0 17 * * 5" prompt="/okr-review"
```

注意：CronCreate 仅在当前会话存活期间有效，会话结束后定时任务自动失效。适合季度末冲刺或特定时期的密集追踪。
