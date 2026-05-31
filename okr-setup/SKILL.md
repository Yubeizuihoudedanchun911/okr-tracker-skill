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
- **不存在**：`_setup-check` Step 3 在本次运行前已完成引导，此处不应再次出现

## 权限表

| 操作 | 所需 scope |
|------|-----------|
| 方式 A（粘贴文本） | 无（_setup-check 已完成批量授权） |
| 方式 B（飞书拉取） | `okr:okr.period:readonly okr:okr.content:readonly` |
