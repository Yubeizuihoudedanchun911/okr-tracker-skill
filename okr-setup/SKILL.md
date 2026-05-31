---
name: okr-setup
version: 1.0.0
description: "OKR 目标录入：在季度初创建或更新 OKR 目标。支持从飞书 OKR 系统拉取或手动粘贴文本录入。当用户说创建 OKR、录入目标、设置本季度 OKR 时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# OKR 目标录入

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../shared/SKILL.md`](../shared/SKILL.md)，其中包含数据格式定义和存储路径约定**

## 前置条件

**CRITICAL — 执行任何 lark-cli 命令前，必须先按 shared/SKILL.md 中的「lark-cli 鉴权」章节完成鉴权检查（Step 0 → Step 1 → Step 2）。**

所需 scope（按录入方式）：
- 方式 A（粘贴文本）：无需额外 scope
- 方式 B（从飞书拉取）：`okr:okr.period:readonly okr:okr.content:readonly`

## 工作流

### Step 1: 初始化存储目录

检查 `~/.claude/okr/` 是否存在，不存在则创建：

```bash
mkdir -p ~/.claude/okr
```

### Step 2: 确定季度

根据当前日期自动推断季度：
- 1-3 月 → Q1
- 4-6 月 → Q2
- 7-9 月 → Q3
- 10-12 月 → Q4

询问用户确认，或允许用户指定其他季度。

### Step 3: 检查已有数据

检查 `~/.claude/okr/<year>-Q<n>.json` 是否已存在：
- **已存在**：展示现有目标摘要，询问用户选择：
  - A) 覆盖（重新录入全部目标）
  - B) 追加/修改单条 Objective
- **不存在**：继续新建流程

### Step 4: 获取 OKR 内容

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

### Step 5: 解析并结构化

将用户输入解析为结构化 JSON，遵循 shared/SKILL.md 中定义的 schema：

- 为每个 Objective 分配 id（O1, O2, ...）
- 为每个 Key Result 分配 id（KR1, KR2, ...）
- 保留原始描述到 `description` 字段
- 尝试提取量化指标到 `metric` 字段（无法提取时设为 null）
- 将用户的完整原始输入存入 `raw_input`
- 所有 `status` 初始化为 `on_track`
- 所有 `current` 初始化为 null

### Step 6: 用户确认

以表格形式展示解析结果：

```
## 解析结果确认

| ID | 目标/KR | 权重 | 量化指标 |
|----|---------|------|---------|
| O1 | 提升系统稳定性 | 40% | — |
| └ KR1 | P0 故障数降至 0 | — | 目标: 0 次 (↓) |
| └ KR2 | 可用性达到 99.95% | — | 目标: 99.95% (↑) |

是否确认？如需修改请指出。
```

### Step 7: 写入文件

确认后写入 `~/.claude/okr/<year>-Q<n>.json`。

### Step 8: 更新配置

读取或创建 `~/.claude/okr/config.json`：
- 更新 `current_quarter` 为当前季度
- 如果 config.json 不存在，引导用户填写：
  - `user_name`：你的名字（用于报告标题）
  - `user_identifiers`：你在周报文档中的标识（如 ["张三", "@张三"]）
  - `doc_url`：周报文档的飞书 URL
  - `timezone`：时区（默认 Asia/Shanghai）
  - `review_day`：每周回顾日（默认 friday）

## 权限表

| 操作 | 所需 scope |
|------|-----------|
| 拉取 OKR 周期 | `okr:okr.period:readonly` |
| 拉取 OKR 内容 | `okr:okr.content:readonly` |
