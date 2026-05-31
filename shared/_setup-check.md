---
name: okr-tracker-setup-check
description: "内部文件：OKR Tracker 环境前置检查。okr-setup 和 okr-review 在执行任何操作前必须先 Read 本文件并按顺序执行检查。正常运行时（已安装、已授权、config 已存在）所有步骤无感知通过。"
internal: true
---

# OKR Tracker 环境前置检查

**每次调用 okr-setup 或 okr-review 时，MUST 先执行本文件中的所有步骤。**

## 触发规则速查

速查表供快速定位：AI agent 应按 Step 0→1→2→3→4 顺序执行，各 Step 内部包含自己的触发条件判断，不需要提前跳过步骤。

| 条件 | 触发步骤 |
|------|---------|
| lark-cli 未安装 | Step 0（终止，等用户安装后重新调用） |
| lark-cli 未配置 | Step 1 |
| `~/.claude/okr/config.json` 不存在 | Step 2 + Step 3 |
| `~/.claude/okr/config.json` 已存在 | 仅 Step 4（轻量检查，通常无感知） |

## Step 0: 检查 lark-cli 是否安装

```bash
lark-cli --version
```

- **成功**（输出版本号）→ 继续 Step 1
- **失败**（command not found）→ 告知用户：

  > lark-cli 未安装，请先运行：
  > ```
  > npm install -g @larksuite/cli
  > ```
  > 安装完成后重新调用本 skill。

  **终止当前 skill，不继续执行。**

## Step 1: 检查 lark-cli config

```bash
lark-cli config show 2>&1; echo "EXIT:$?"
```

- **成功**（exit code 为 0 且输出 appId 等配置）→ 继续 Step 2
- **失败**（exit code 非 0 或输出包含 "not configured" / "error"）→ 后台启动初始化并捕获输出：

  ```bash
  lark-cli config init --new 2>&1 &
  sleep 3
  ```

  从输出中提取 `verification_url`（格式：`https://open.feishu.cn/page/cli?user_code=...`），生成二维码展示：

  ```bash
  lark-cli auth qrcode "<verification_url>" --ascii
  ```

  提示用户：「请用飞书 App 扫码或在浏览器打开以上链接完成初始配置，完成后告诉我。」

  **等待用户确认完成 → 继续 Step 2。**

## Step 2: 首次批量授权

**仅当 `~/.claude/okr/config.json` 不存在时执行本步骤。**

```bash
ls ~/.claude/okr/config.json 2>/dev/null || echo "NOT_FOUND"
```

输出 "NOT_FOUND" 时，一次性申请所有 scope：

```bash
lark-cli auth login --scope "docx:document:readonly docs:document.media:download okr:okr.period:readonly okr:okr.content:readonly" --no-wait --json
```

从 JSON 输出中提取 `verification_url`，生成二维码：

```bash
lark-cli auth qrcode "<verification_url>" --ascii
```

提示用户：「请用飞书 App 扫码完成授权（包含文档读取、图片下载、OKR 读取权限），完成后告诉我。」

**等待用户确认完成后**，使用输出中的 `device_code` 完成轮询：

```bash
lark-cli auth login --device-code "<device_code>"
```

如果 `--device-code` 命令报错（过期或失败），重新从 Step 2 开始（重新生成新的 device_code）。

→ 继续 Step 3。

## Step 3: 初始化 config.json

**仅当 `~/.claude/okr/config.json` 不存在时执行本步骤。**

```bash
mkdir -p ~/.claude/okr
```

依次询问用户以下信息（每个字段单独提问，有默认值的可直接回车跳过）：

1. **user_name**：你的显示名称（用于报告标题）
2. **user_identifiers**：你在飞书文档中出现的所有标识，用逗号分隔
   - 提示：通常包括中文名、拼音全名、飞书昵称、@昵称 等多种写法，多填几种可以避免识别漏掉
3. **doc_mode**：周报组织方式
   - `parent`：一周一个子页面（填父级 wiki URL）
   - `single`：在同一个文档中持续追加
4. **doc_url**：
   - `parent` 模式：填父级 wiki 页面 URL
   - `single` 模式：填周报文档 URL
5. **timezone**（默认 `Asia/Shanghai`）
6. **review_day**（默认 `manual`，表示手动调用；可填 `monday`~`sunday`）

将收集到的信息写入 `~/.claude/okr/config.json`，`current_quarter` 根据当前日期自动推断（1-3月→Q1，4-6月→Q2，7-9月→Q3，10-12月→Q4）：

```json
{
  "user_name": "<填写值>",
  "user_identifiers": ["<标识1>", "<标识2>"],
  "doc_mode": "<parent|single>",
  "doc_url": "<填写值>",
  "doc_url_history": [],
  "timezone": "Asia/Shanghai",
  "review_day": "manual",
  "current_quarter": "<YYYY-Q<n>>"
}
```

## Step 4: 增量 scope 检查（每次运行都执行）

```bash
lark-cli auth check --scope "docx:document:readonly docs:document.media:download okr:okr.period:readonly okr:okr.content:readonly" 2>&1
```

- **全部通过** → 前置检查完成，继续执行调用方 skill
- **有缺失 scope** → 从错误输出中提取缺失的 scope，仅补授权缺失部分：

  ```bash
  lark-cli auth login --scope "<缺失的 scope>" --no-wait --json
  ```

  生成二维码 → 等待用户扫码确认 → 使用 device_code 完成授权 → 重新执行 Step 4 验证。若连续 2 次补授权后仍有缺失 scope，停止重试，提示用户：
  「scope 授权验证失败，请检查飞书应用权限配置，或运行 `lark-cli auth status` 查看当前状态。」
