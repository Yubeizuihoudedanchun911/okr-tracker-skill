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
- `~/.claude/okr/config.json` 已配置（`_setup-check` 会自动检查并引导）

## 工作流

### Step 1: 加载配置和目标

```bash
cat ~/.claude/okr/config.json
cat ~/.claude/okr/<year>-Q<n>.json
```

其中 `<year>-Q<n>` 取 `config.current_quarter` 的值。如果 OKR 文件不存在，提示用户先运行 `/okr-setup`。

### Step 2: 确定本次 review 的文档来源

按以下优先级确定文档 URL：

**优先级 1：用户本次指定**

用户消息中包含飞书 URL（如「帮我 review 这篇 <url>」）→ 使用该 URL，并将其加入 `doc_url_history`（保留最近 5 条）：

```bash
# 读取现有 history，添加新 URL，保留最新 5 条，写回
cat ~/.claude/okr/config.json | python3 -c "
import json,sys
c=json.load(sys.stdin)
h=c.get('doc_url_history',[])
new_url='<本次指定的 URL>'
if new_url not in h:
    h.insert(0,new_url)
c['doc_url_history']=h[:5]
print(json.dumps(c,ensure_ascii=False,indent=2))
" > /tmp/config_tmp.json && mv /tmp/config_tmp.json ~/.claude/okr/config.json
```

**优先级 2：parent 模式自动发现**

`config.doc_mode == "parent"` 时，从父级 wiki URL 提取 wiki token（URL 中 `/wiki/` 后的部分），调用 API 列出子页面：

```bash
lark-cli api GET /open-apis/wiki/v2/spaces/get_node \
  --params '{"token":"<wiki_token>","obj_type":"wiki"}'
```

从结果中按以下顺序匹配最新子页面：
1. 标题含 `YYYY-MM-DD` 格式（取最新日期）
2. 标题含 `W\d+`（如 W22，取最大周号）
3. 标题含 `第\d+周`（取最大周号）
4. 标题含 `周报`，取 `obj_create_time` 最新的

若 API 调用失败或无法解析子页面列表 → 告知用户本次无法自动发现，请提供本周文档 URL，使用优先级 1 流程继续。

**优先级 3：single 模式**

`config.doc_mode == "single"` 时：直接使用 `config.doc_url`。

### Step 3: 拉取文档内容

```bash
lark-cli docs +fetch --api-version v2 --doc "<doc_url>"
```

从响应中提取 `data.document.content`（HTML 格式字符串）。

### Step 4: 定位用户段落

在文档 HTML 内容中，按 `config.user_identifiers` 中的标识匹配：
1. 查找包含任一标识的标题标签（`<h1>`~`<h3>`、`<title>` 等）
2. 查找段落中 @ 提及用户的内容（如 `<p>@jijunling</p>`）
3. 确定用户段落起止范围：从匹配点开始，到下一个同级或更高级标题结束

未找到 → 报告错误：「在文档中未找到与 user_identifiers 匹配的用户段落，请检查配置中的标识列表，可添加更多写法（如中文名、拼音、@昵称）。」

### Step 5: 检测内容类型并处理图片

**5a. 检测用户段落中是否含 `<img>` 标签**

- **无图片** → 直接进入 Step 6（使用用户段落的文字内容做解析）
- **有图片** → 执行图片增强流程（5b~5e）

**5b. 提取用户段落内的图片 token**

从用户段落范围内的 `<img src="<token>">` 提取所有 `src` 值（仅段落范围内，不处理其他用户的图片）。

**5c. 并行下载图片**

```bash
mkdir -p /tmp/okr-review-imgs
cd /tmp/okr-review-imgs
# 对每个 token 并行执行（N 为序号，从 1 开始）：
lark-cli docs +media-download --token <token_1> --output img_1.png --overwrite &
lark-cli docs +media-download --token <token_2> --output img_2.png --overwrite &
# ... 以此类推
wait
```

**5d. 重建用户段落骨架**

将用户段落的 HTML 内容转换为带占位符的结构化文本：
- 保留文字内容和标题层级（`##`、`###` 等，对应 `<h2>`、`<h3>`）
- 将每个 `<img>` 标签替换为 `[IMAGE_1]`、`[IMAGE_2]`... 占位符（按出现顺序编号）

示例输出（假设文档结构为 `@jijunling` + 3 张图）：
```
@jijunling
[IMAGE_1]
业务容器约束：
[IMAGE_2]
营销密集治理优化：
[IMAGE_3]
```

**5e. 整体传入 Claude 分析**

使用 Read 工具依次加载所有图片文件（`/tmp/okr-review-imgs/img_1.png` 等），配合段落骨架和 OKR 目标，一次完成分析：

> 以下是用户本周周报的文档结构和图片内容。
>
> **文档骨架**（含图片占位符位置）：
> ```
> <段落骨架文本>
> ```
>
> 图片：[IMAGE_1] 对应 img_1.png，[IMAGE_2] 对应 img_2.png，...
>
> **本季度 OKR 目标**：
> ```json
> <objectives JSON>
> ```
>
> 请提取与每条 KR 相关的最新进展（当前数值如有请给出），输出格式：KR_id: 进展描述（当前值 X）

从分析结果提取结构化进展摘要，继续 Step 6。

**图片处理失败处理：**

| 场景 | 处理 |
|------|------|
| 下载失败（权限不足，`missing_scope`） | 提示：「请检查 `docs:document.media:download` scope，可运行 `lark-cli auth login --scope "docs:document.media:download"` 补授权」 |
| 部分图片识别内容为空 | 跳过该图，在进展摘要中备注「[IMAGE_N] 内容未识别」，继续分析其余内容 |
| 全部图片下载或识别失败 | 提示用户手动输入本周各 KR 进展，进入手动输入流程 |

### Step 6: 对比分析

对每条 Key Result 执行：

1. **内容匹配**：在周报进展摘要中搜索与该 KR 相关的描述（关键词匹配 + 语义理解）
2. **进度评估**：
   - 周报中有具体数值 → 更新 `current` 字段
   - 有进展描述但无数值 → 根据描述推断状态
   - 未提及 → 标记为潜在风险
3. **状态判定**：
   - 计算时间进度：`(当前日期 - 季度开始日期) / 季度总天数`
   - 有数值时计算目标进度（decrease 方向）：`(baseline - current) / (baseline - target)`
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
| KR | 目标 | 当前 | 状态 |
|----|------|------|------|
| KR1 <title> | <target><unit> | <current 或 —> | ✅/⚠️/🔴/🎯 |

### 未在周报中提及的 KR
- KR_id：<title>

### 建议
1. 【紧急】...（behind 状态 KR）
2. 【关注】...（at_risk 状态 KR）
3. 【提醒】...（未提及的 KR）
```

**状态图标**：`on_track` → ✅，`at_risk` → ⚠️，`behind` → 🔴，`completed` → 🎯

**建议生成规则**：
- `behind`：给出具体追赶建议，标记【紧急】
- `at_risk`：给出预防建议，标记【关注】
- 未提及的 KR：提醒更新进展，标记【提醒】
- 基于 OKR 的 `description` 和 `raw_input` 给出有上下文的建议，而非泛泛而谈

## 权限表

| 操作 | 所需 scope |
|------|-----------|
| 读取飞书文档 | `docx:document:readonly` |
| 下载文档图片 | `docs:document.media:download` |
