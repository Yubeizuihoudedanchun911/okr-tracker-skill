# OKR Tracker Skill 优化 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 重构 okr-tracker-skill，消除多轮打断授权、统一前置检查、支持多文档模式、增强图片周报处理。

**Architecture:** 新增内部文件 `shared/_setup-check.md` 承载所有环境前置检查逻辑；将 `shared/SKILL.md` 改名为 `shared/_shared.md` 保持内部文件命名约定；更新两个 user-facing skill 引用新文件并新增功能。内部文件加下划线前缀，不链接进 `~/.claude/skills/`。

**Tech Stack:** Markdown skill files for Claude Code CLI；lark-cli v1.x（`@larksuite/cli`）

---

## 文件变更一览

| 操作 | 路径 |
|------|------|
| 新建 | `shared/_setup-check.md` |
| 改名 + 修改 | `shared/SKILL.md` → `shared/_shared.md` |
| 修改 | `okr-setup/SKILL.md` |
| 修改 | `okr-review/SKILL.md` |
| 修改 | `README.md` |
| 修改 | `README.zh.md` |

---

### Task 1: 新建 `shared/_setup-check.md`

**Files:**
- Create: `shared/_setup-check.md`

- [ ] **Step 1: 创建文件，写入完整内容**

```markdown
---
name: okr-tracker-setup-check
description: "内部文件：OKR Tracker 环境前置检查。okr-setup 和 okr-review 在执行任何操作前必须先 Read 本文件并按顺序执行检查。正常运行时（已安装、已授权、config 已存在）所有步骤无感知通过。"
internal: true
---

# OKR Tracker 环境前置检查

**每次调用 okr-setup 或 okr-review 时，MUST 先执行本文件中的所有步骤。**

## 触发规则速查

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
lark-cli config show
```

- **成功**（输出 appId 等配置）→ 继续 Step 2
- **失败**（输出 "not configured"）→ 执行初始化：

  ```bash
  lark-cli config init --new
  ```

  此命令会阻塞并输出 `verification_url`。从输出中提取 URL，生成二维码展示：

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

**等待用户确认完成后**，使用输出中的 `device_code` 轮询：

```bash
lark-cli auth login --device-code "<device_code>"
```

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

将收集到的信息写入 `~/.claude/okr/config.json`，`current_quarter` 根据当前日期自动推断：

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

季度推断规则：1-3 月 → Q1，4-6 月 → Q2，7-9 月 → Q3，10-12 月 → Q4。

## Step 4: 增量 scope 检查（每次运行都执行）

```bash
lark-cli auth check --scope "docx:document:readonly docs:document.media:download okr:okr.period:readonly okr:okr.content:readonly" 2>&1
```

- **全部通过** → 前置检查完成，继续执行调用方 skill
- **有缺失 scope** → 从错误输出中提取缺失的 scope，仅补授权缺失部分：

  ```bash
  lark-cli auth login --scope "<缺失的 scope>" --no-wait --json
  ```

  生成二维码 → 等待用户扫码确认 → 使用 device_code 完成授权 → 重新执行 Step 4 验证。
```

- [ ] **Step 2: 验证文件结构正确**

```bash
head -5 /path/to/okr-tracker-skill/shared/_setup-check.md
```

预期输出：以 `---` 开头的 frontmatter，`name: okr-tracker-setup-check`。

- [ ] **Step 3: Commit**

```bash
git add shared/_setup-check.md
git commit -m "feat: add _setup-check.md with auto-prerequisite check flow"
```

---

### Task 2: 将 `shared/SKILL.md` 改名为 `shared/_shared.md` 并更新内容

**Files:**
- Modify/Rename: `shared/SKILL.md` → `shared/_shared.md`

- [ ] **Step 1: git mv 改名**

```bash
git mv shared/SKILL.md shared/_shared.md
```

- [ ] **Step 2: 更新文件内容**

用以下内容完整替换 `shared/_shared.md`：

```markdown
---
name: okr-tracker-shared
description: "内部文件：OKR Tracker 共享规则。数据格式定义、config.json schema、飞书文档解析规则、错误处理。okr-setup 和 okr-review 使用前必须先 Read 本文件。"
internal: true
---

# OKR Tracker 共享规则

## 数据存储路径

所有数据存放在 `~/.claude/okr/` 目录下：

| 文件 | 用途 |
|------|------|
| `config.json` | 用户配置 |
| `<year>-Q<n>.json` | 季度 OKR 目标数据（如 `2026-Q2.json`） |

## config.json Schema

```json
{
  "user_name": "用户显示名",
  "user_identifiers": ["用户名", "@用户名", "拼音"],
  "doc_mode": "parent",
  "doc_url": "https://xxx.feishu.cn/wiki/parent-or-single-doc",
  "doc_url_history": [],
  "timezone": "Asia/Shanghai",
  "review_day": "manual",
  "current_quarter": "2026-Q2"
}
```

字段说明：

| 字段 | 说明 |
|------|------|
| `user_name` | 显示名称，用于报告标题 |
| `user_identifiers` | 飞书文档中用于定位用户段落的标识列表，支持中文名、拼音、@昵称等多种写法 |
| `doc_mode` | `"parent"`：一周一份子页面（doc_url 填父级 wiki）；`"single"`：单文档持续追加 |
| `doc_url` | parent 模式：父级 wiki URL；single 模式：固定文档 URL |
| `doc_url_history` | 最近 5 次 ad-hoc 指定的文档 URL，自动记录 |
| `timezone` | 用户时区 |
| `review_day` | 每周回顾日；`manual` 表示手动调用 |
| `current_quarter` | 当前追踪季度，格式 `YYYY-Q<n>` |

## OKR 目标文件 Schema（`<year>-Q<n>.json`）

```json
{
  "year": 2026,
  "quarter": 2,
  "created_at": "2026-04-01",
  "objectives": [
    {
      "id": "O1",
      "title": "目标标题",
      "description": "目标的完整描述，保留原始语义",
      "weight": 0.4,
      "key_results": [
        {
          "id": "KR1",
          "title": "关键结果标题",
          "description": "关键结果的完整描述",
          "metric": {
            "target": 0,
            "baseline": null,
            "unit": "次",
            "direction": "decrease"
          },
          "current": null,
          "status": "on_track"
        }
      ]
    }
  ],
  "raw_input": "用户录入时的原始文本"
}
```

字段规则：
- `id`：Objective 用 O1/O2/O3...，KeyResult 用 KR1/KR2/KR3...（全局唯一）
- `weight`：同一层级权重之和必须为 1.0
- `metric.direction`：`increase`（越大越好）或 `decrease`（越小越好）
- `metric.baseline`：起始基线值，可为 null
- `current`：最近一次 review 后的实际值，初始为 null
- `status`：`on_track` | `at_risk` | `behind` | `completed`
- `raw_input`：保留用户输入的原始文本

## 飞书文档解析规则

### 定位用户段落

从文档内容中按以下优先级匹配用户段落：

1. **标题匹配**：查找包含 `user_identifiers` 中任一标识的标题
2. **@ 标记匹配**：查找段落中 @ 提及用户的部分
3. **匹配范围**：从匹配到的标题/段落开始，到下一个同级或更高级标题结束

### 定位时间范围

从用户段落中提取最近一周的内容：

1. **日期标题模式**：匹配 `## YYYY-MM-DD ~ YYYY-MM-DD`、`### 第 N 周`、`### WN` 等
2. **日期段落模式**：匹配段落开头的日期标记
3. **兜底策略**：若无日期标记，取用户段落的最后 5 段内容作为最近进展

## Scope 权限表

| 操作 | 所需 scope |
|------|-----------|
| okr-setup（方式 A，粘贴文本） | 无 |
| okr-setup（方式 B，从飞书拉取） | `okr:okr.period:readonly okr:okr.content:readonly` |
| okr-review（文字文档） | `docx:document:readonly` |
| okr-review（含图片文档） | `docx:document:readonly docs:document.media:download` |
| 首次初始化（_setup-check Step 2） | 上述全部 |

## 错误处理

| 场景 | 处理方式 |
|------|---------|
| `~/.claude/okr/` 不存在 | 提示用户先调用 `/okr-setup` |
| `config.json` 不存在 | `_setup-check` Step 3 会自动引导创建 |
| 当前季度 OKR 文件不存在 | 提示用户先运行 `/okr-setup` 创建本季度目标 |
| 飞书文档读取失败（权限） | `_setup-check` Step 4 会自动补授权 |
| 文档中未找到用户段落 | 报告未匹配，建议检查 `user_identifiers` 配置 |
| 文档中未找到时间段落 | 使用兜底策略，提示用户 |
```

- [ ] **Step 3: 验证改名和内容**

```bash
ls shared/
# 预期：_setup-check.md  _shared.md（无 SKILL.md）
```

- [ ] **Step 4: Commit**

```bash
git add shared/_shared.md
git commit -m "refactor: rename SKILL.md to _shared.md, update schema and scope table"
```

---

### Task 3: 更新 `okr-setup/SKILL.md`

**Files:**
- Modify: `okr-setup/SKILL.md`

- [ ] **Step 1: 用以下内容完整替换 `okr-setup/SKILL.md`**

```markdown
---
name: okr-setup
version: 2.0.0
description: "OKR 目标录入：在季度初创建或更新 OKR 目标。支持从飞书 OKR 系统拉取或手动粘贴文本录入。当用户说创建 OKR、录入目标、设置本季度 OKR 时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# OKR 目标录入

**CRITICAL — 开始前 MUST 依次 Read 以下两个内部文件并执行其中的检查：**
1. `../shared/_setup-check.md` — 环境前置检查（lark-cli 安装、授权、config 初始化）
2. `../shared/_shared.md` — 数据格式定义和存储路径约定

## 工作流

### Step 1: 确定季度

根据当前日期自动推断季度（1-3 月→Q1，4-6 月→Q2，7-9 月→Q3，10-12 月→Q4）。

询问用户确认，或允许用户指定其他季度。

### Step 2: 检查已有数据

检查 `~/.claude/okr/<year>-Q<n>.json` 是否已存在：

- **已存在**：展示现有目标摘要，询问用户：
  - A) 覆盖（重新录入全部目标）
  - B) 追加/修改单条 Objective
- **不存在**：继续新建流程

### Step 3: 获取 OKR 内容

提供两种录入方式，询问用户选择：

**方式 A：直接粘贴文本**

提示用户粘贴 OKR 内容（从飞书 OKR 系统复制、或自行编写）。

**方式 B：从飞书 OKR 系统拉取**

```bash
# 获取用户的 OKR 周期列表
lark-cli okr +cycle-list

# 获取指定周期的详细内容
lark-cli okr +cycle-detail --cycle-id "<cycle_id>"
```

### Step 4: 解析并结构化

将用户输入解析为结构化 JSON，遵循 `_shared.md` 中定义的 schema：

- 为每个 Objective 分配 id（O1, O2, ...）
- 为每个 Key Result 分配 id（KR1, KR2, ...，全局唯一）
- 保留原始描述到 `description` 字段
- 尝试提取量化指标到 `metric` 字段（`target`、`baseline`、`unit`、`direction`）；无法提取时 `metric` 设为 null
- 将用户的完整原始输入存入 `raw_input`
- 所有 `status` 初始化为 `on_track`，`current` 初始化为 null

### Step 5: 用户确认

以表格形式展示解析结果：

```
## 解析结果确认

| ID | 目标/KR | 权重 | 量化指标 |
|----|---------|------|---------|
| O1 | 提升系统稳定性 | 40% | — |
| └ KR1 | P0 故障数降至 0 | — | 基线: N/A → 目标: 0 次 (↓) |

是否确认？如需修改请指出。
```

### Step 6: 写入文件

确认后写入 `~/.claude/okr/<year>-Q<n>.json`。

### Step 7: 更新 config

读取 `~/.claude/okr/config.json`：

- **已存在**：仅更新 `current_quarter` 为当前季度，其他字段不变
- **不存在**：_setup-check Step 3 已在本次运行前完成引导，此处不应再次出现

## 权限表

| 操作 | 所需 scope |
|------|-----------|
| 方式 A（粘贴文本） | 无（_setup-check 已完成批量授权） |
| 方式 B（飞书拉取） | `okr:okr.period:readonly okr:okr.content:readonly` |
```

- [ ] **Step 2: 确认文件内容正确**

```bash
grep -n "CRITICAL\|_setup-check\|Step 7" okr-setup/SKILL.md
```

预期输出：第 8-9 行含 `_setup-check.md` 引用，Step 7 中无「引导填写」逻辑。

- [ ] **Step 3: Commit**

```bash
git add okr-setup/SKILL.md
git commit -m "feat: update okr-setup to use _setup-check, simplify config step"
```

---

### Task 4: 更新 `okr-review/SKILL.md`

**Files:**
- Modify: `okr-review/SKILL.md`

- [ ] **Step 1: 用以下内容完整替换 `okr-review/SKILL.md`**

```markdown
---
name: okr-review
version: 2.0.0
description: "OKR 周度回顾：读取飞书周报文档，对比本季度 OKR 目标，分析进度偏差，识别落后项，给出优先级建议。当用户说 OKR 回顾、检查进度、本周 OKR 分析时使用。支持文字文档和图片截图式周报。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# OKR 周度回顾

**CRITICAL — 开始前 MUST 依次 Read 以下两个内部文件并执行其中的检查：**
1. `../shared/_setup-check.md` — 环境前置检查（lark-cli 安装、授权、config 初始化）
2. `../shared/_shared.md` — 数据格式定义、文档解析规则、错误处理

## 前置条件

- 已运行过 `/okr-setup` 创建本季度 OKR 目标
- `~/.claude/okr/config.json` 已配置（_setup-check 会自动检查并引导）

## 工作流

### Step 1: 加载配置和目标

```bash
cat ~/.claude/okr/config.json
cat ~/.claude/okr/<year>-Q<n>.json  # current_quarter 对应的文件
```

如果 OKR 文件不存在，提示用户先运行 `/okr-setup`。

### Step 2: 确定本次 review 的文档来源

按以下优先级确定文档 URL：

**优先级 1：用户本次指定**
用户消息中包含飞书 URL（如「帮我 review 这篇 <url>」）→ 使用该 URL，并将其加入 `config.doc_url_history`（保留最近 5 条，用 `jq` 更新 config.json）。

**优先级 2：parent 模式自动发现**
`config.doc_mode == "parent"` 时，从父级 wiki URL 提取 wiki token（URL 中 `/wiki/` 后的部分），调用 API 列出子页面：

```bash
# 列出 wiki 节点的子页面
lark-cli api GET /open-apis/wiki/v2/spaces/get_node \
  --params '{"token":"<wiki_token>","obj_type":"wiki"}'
```

若 API 返回子节点列表，按以下顺序匹配最新子页面：
1. 标题含 `YYYY-MM-DD` 格式（取最新日期）
2. 标题含 `W\d+`（如 W22，取最大周号）
3. 标题含 `第\d+周`（取最大周号）
4. 标题含 `周报`，取 `obj_create_time` 最新的

若 API 调用失败或无法解析子页面列表 → 告知用户本次无法自动发现，请提供本周文档 URL（记入 `doc_url_history` 并继续）。

选定子页面 URL 后继续 Step 3。

**优先级 3：single 模式**
`config.doc_mode == "single"` 时：直接使用 `config.doc_url`。

### Step 3: 拉取文档内容

```bash
lark-cli docs +fetch --api-version v2 --doc "<doc_url>"
```

从响应中提取 `data.document.content`（HTML 格式）。

### Step 4: 定位用户段落

在文档内容中，按 `config.user_identifiers` 中的标识匹配：
1. 查找包含任一标识的标题标签（`<h1>`~`<h3>`）
2. 查找段落中 @ 提及用户的内容
3. 确定用户段落起止范围：从匹配点开始，到下一个同级或更高级标题结束

未找到 → 报告错误，建议检查 `user_identifiers` 配置，提示可加入更多标识写法。

### Step 5: 检测内容类型并处理图片

**5a. 检测用户段落中是否含 `<img>` 标签**

- **无图片** → 直接进入 Step 6（文字解析流程）
- **有图片** → 执行图片增强流程（5b~5e）

**5b. 提取用户段落内的图片 token**

从 `<img src="<token>">` 提取所有 `src` 值（仅段落范围内的，不处理其他用户的图片）。

**5c. 并行下载图片**

```bash
# 在临时目录中执行（lark-cli 要求相对路径）
mkdir -p /tmp/okr-review-imgs
cd /tmp/okr-review-imgs
# 对每个 token 并行执行：
lark-cli docs +media-download --token <token> --output img_<N>.png --overwrite &
# 等待所有下载完成
wait
```

**5d. 重建用户段落骨架**

将用户段落的 HTML 内容转换为带占位符的结构化文本：
- 保留文字内容和标题层级（`##`、`###` 等）
- 将每个 `<img>` 标签替换为 `[IMAGE_1]`、`[IMAGE_2]`... 占位符（按出现顺序编号）

示例输出：
```
@jijunling
[IMAGE_1]
业务容器约束：
[IMAGE_2]
营销密集治理优化：
[IMAGE_3]
```

**5e. 整体传入 Claude 分析**

使用 Read 工具加载所有图片，配合段落骨架和 OKR 目标，一次性完成分析：

> 系统提示（传给自己）：
> 以下是用户本周周报的文档结构和图片内容。
> 文档骨架（含图片占位符位置）：
> ```
> <段落骨架>
> ```
> 图片：[IMAGE_1] = <img1>，[IMAGE_2] = <img2>，...
>
> 本季度 OKR 目标：
> ```json
> <objectives 列表>
> ```
>
> 请提取与每条 KR 相关的最新进展，输出格式：
> - KR_id：描述（当前值 X，若有）

从分析结果中提取结构化进展摘要，继续 Step 6。

**图片处理失败处理：**

| 场景 | 处理 |
|------|------|
| 下载失败（权限不足） | 提示检查 `docs:document.media:download` scope，建议运行 `lark-cli auth login --scope "docs:document.media:download"` |
| 部分图片识别失败 | 跳过该图，备注「图片内容未识别」，继续分析其余内容 |
| 全部图片下载/识别失败 | 提示用户手动提供本周进展，进入手动输入流程 |

### Step 6: 对比分析

对每条 Key Result 执行：

1. **内容匹配**：在周报进展摘要中搜索与该 KR 相关的描述
2. **进度评估**：
   - 周报中有具体数值 → 更新 `current` 字段
   - 有进展描述但无数值 → 根据描述推断状态
   - 未提及 → 标记为潜在风险
3. **状态判定**：
   - 计算时间进度：`(当前日期 - 季度开始日期) / 季度总天数`
   - 计算目标进度（有数值时）：`(baseline - current) / (baseline - target)`（decrease 方向）
   - `目标进度 >= 时间进度` → `on_track`
   - `目标进度 < 时间进度 且差距 < 20%` → `at_risk`
   - `目标进度 < 时间进度 且差距 >= 20%` → `behind`
   - 已达标 → `completed`
   - 无量化指标时，根据语义做定性判断

### Step 7: 更新 OKR 文件

将分析结果写回 `~/.claude/okr/<year>-Q<n>.json`：
- 更新每条 KR 的 `current` 值（有数据时）
- 更新每条 KR 的 `status`

### Step 8: 输出分析报告

```
## OKR 周度回顾 — <year>-Q<n> 第 N 周

### 整体进度：X% 时间已过，目标完成约 Y%（状态描述）

### O1: <title>（权重 N%）
| KR | 目标 | 当前 | 进度 | 状态 |
|----|------|------|------|------|
| KR1 <title> | <target> | <current> | — | ✅/⚠️/🔴/🎯 |

### 未在周报中提及的 KR
- <KR title>

### 建议
1. 【紧急】...
2. 【关注】...
3. 【提醒】...
```

**状态图标**：`on_track` → ✅，`at_risk` → ⚠️，`behind` → 🔴，`completed` → 🎯

**建议生成规则**：
- `behind`：给出具体追赶建议，标记【紧急】
- `at_risk`：给出预防建议，标记【关注】
- 未提及的 KR：提醒更新进展，标记【提醒】
- 基于 OKR 的 `description` 和 `raw_input` 给出有上下文的建议

## 权限表

| 操作 | 所需 scope |
|------|-----------|
| 读取飞书文档 | `docx:document:readonly` |
| 下载文档图片 | `docs:document.media:download` |
```

- [ ] **Step 2: 确认关键段落存在**

```bash
grep -n "_setup-check\|Step 2.*来源\|5e\|IMAGE_" okr-review/SKILL.md | head -20
```

预期：包含 `_setup-check.md` 引用、Step 2 文档来源确定、5e 整体传入 Claude、`[IMAGE_N]` 占位符逻辑。

- [ ] **Step 3: Commit**

```bash
git add okr-review/SKILL.md
git commit -m "feat: update okr-review with doc-mode support and image processing flow"
```

---

### Task 5: 更新 README

**Files:**
- Modify: `README.md`
- Modify: `README.zh.md`

- [ ] **Step 1: 更新 `README.zh.md` 安装章节和配置说明**

将「安装」章节中 `shared` 的链接行删除：

```bash
# 找到并删除 shared 链接行
# 修改前：
# ln -s $(pwd)/shared ~/.claude/skills/okr-tracker-shared
# 修改后：删除此行
```

在「数据存储」章节新增 `config.json` 字段说明，列出 `doc_mode`、`doc_url_history` 两个新字段。

- [ ] **Step 2: 同步更新 `README.md`（英文版）**

与中文版保持一致：移除 shared 链接，新增字段说明。

- [ ] **Step 3: Commit**

```bash
git add README.md README.zh.md
git commit -m "docs: update README for new install instructions and config schema"
```

---

### Task 6: 迁移现有 config.json 到新 schema

**Files:**
- Modify: `~/.claude/okr/config.json`（用户本地文件）

- [ ] **Step 1: 读取现有 config，补充新字段**

```bash
cat ~/.claude/okr/config.json
```

当前内容缺少 `doc_mode` 和 `doc_url_history`。询问用户：

> 你的周报是一周一份子页面（parent）还是在同一份文档持续追加（single）？

根据回答，用 `jq` 或直接写入更新 config.json，添加：
```json
"doc_mode": "<parent|single>",
"doc_url_history": []
```

- [ ] **Step 2: 验证 config 完整性**

```bash
cat ~/.claude/okr/config.json | python3 -c "
import json,sys
c = json.load(sys.stdin)
required = ['user_name','user_identifiers','doc_mode','doc_url','doc_url_history','timezone','review_day','current_quarter']
missing = [k for k in required if k not in c]
print('Missing:', missing if missing else 'None')
"
```

预期：`Missing: None`

- [ ] **Step 3: Commit（如果 config 在 git 追踪范围内则跳过）**

config.json 是用户本地文件，不在 git 中，无需 commit。

---

## 验收标准

全部任务完成后，验证端到端流程：

1. **首次使用模拟**：`rm ~/.claude/okr/config.json && /okr-setup` → 应自动触发 _setup-check 的 Step 2（批量授权）+ Step 3（config 引导）
2. **再次运行**：`/okr-review` → _setup-check 仅执行 Step 4（轻量 scope 检查，无感知），直接进入文档拉取
3. **图片周报**：包含图片的文档应先定位用户段落，再下载段落内图片，最终输出完整 OKR 分析报告
