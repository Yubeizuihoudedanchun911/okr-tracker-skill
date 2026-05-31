---
name: okr-review
version: 1.0.0
description: "OKR 周度回顾：读取飞书周报文档，对比本季度 OKR 目标，分析进度偏差，识别落后项，给出优先级建议。当用户说 OKR 回顾、检查进度、本周 OKR 分析时使用。"
metadata:
  requires:
    bins: ["lark-cli"]
---

# OKR 周度回顾

**CRITICAL — 开始前 MUST 先用 Read 工具读取 [`../shared/SKILL.md`](../shared/SKILL.md)，其中包含数据格式定义、文档解析规则和错误处理**

## 前置条件

- `lark-cli` 已安装并完成 `config init`
- 已授权：`lark-cli auth login --scope "docx:document:readonly"`
- 已运行过 `/okr-setup` 创建本季度 OKR 目标
- `~/.claude/okr/config.json` 已配置

## 工作流

### Step 1: 加载配置和目标

```bash
# 读取配置
cat ~/.claude/okr/config.json
```

从 `config.json` 获取：
- `doc_url`：周报文档地址
- `user_identifiers`：用户标识列表
- `current_quarter`：当前季度

```bash
# 读取当前季度 OKR 目标
cat ~/.claude/okr/<year>-Q<n>.json
```

如果文件不存在，提示用户先运行 `/okr-setup`。

### Step 2: 读取飞书周报文档

```bash
lark-cli docs +fetch --api-version v2 --doc "<doc_url>"
```

> 如果文档过长，使用 `--scope` 参数限制读取范围（参考 lark-doc skill 的 fetch 用法）。

### Step 3: 定位用户段落

按 shared/SKILL.md 中的「飞书文档解析规则 > 定位用户段落」执行：

1. 在文档内容中搜索 `user_identifiers` 中的标识
2. 确定用户段落的起止范围（从匹配标题到下一个同级或更高级标题）
3. 如果未找到，报告错误并建议检查 `user_identifiers` 配置

### Step 4: 提取最近进展

按 shared/SKILL.md 中的「飞书文档解析规则 > 定位时间范围」执行：

1. 在用户段落中查找日期标题（`## YYYY-MM-DD ~ YYYY-MM-DD`、`### 第 N 周`、`### WN`）
2. 提取最近一周（或最近一次更新）的内容
3. 若无日期标记，取用户段落的最后 5 段内容

### Step 5: 对比分析

对每条 Key Result 执行：

1. **内容匹配**：在周报进展中搜索与该 KR 相关的描述（关键词匹配 + 语义理解）
2. **进度评估**：
   - 如果周报中提到了具体数值，更新 `current` 字段
   - 如果周报中描述了进展但无数值，根据描述推断状态
   - 如果周报中未提及该 KR，标记为潜在风险
3. **状态判定**：
   - 计算季度时间进度：`(当前日期 - 季度开始日期) / 季度总天数`
   - 计算目标进度：`current / target`（根据 direction 调整）
   - 判定规则：
     - 目标进度 >= 时间进度 → `on_track`
     - 目标进度 < 时间进度 且差距 < 20% → `at_risk`
     - 目标进度 < 时间进度 且差距 >= 20% → `behind`
     - 已达标 → `completed`
   - 无量化指标时，根据周报描述的语义做定性判断

### Step 6: 更新 OKR 文件

将分析结果写回 `~/.claude/okr/<year>-Q<n>.json`：
- 更新每条 KR 的 `current` 值
- 更新每条 KR 的 `status`

### Step 7: 输出分析报告

按以下格式输出到终端：

```
## OKR 周度回顾 — <year>-Q<n> 第 N 周

### 整体进度：X% 时间已过，目标完成约 Y%（状态描述）

### O1: <title>（权重 N%）
| KR | 目标 | 当前 | 进度 | 状态 |
|----|------|------|------|------|
| <KR title> | <target> | <current> | <icon> | <status> |

### 未在周报中提及的 KR
- <KR title>（上次更新：N 周前）

### 建议
1. 【优先级标签】具体建议内容
2. ...
```

**状态图标映射：**
- `on_track` → ✅
- `at_risk` → ⚠️
- `behind` → 🔴
- `completed` → 🎯

**建议生成规则：**
- `behind` 状态的 KR：给出具体追赶建议，标记【紧急】
- `at_risk` 状态的 KR：给出预防建议，标记【关注】
- 未提及的 KR：提醒更新进展，标记【提醒】
- 基于 OKR 的 `description` 和 `raw_input` 给出有上下文的建议，而非泛泛而谈

## 权限表

| 操作 | 所需 scope |
|------|-----------|
| 读取飞书文档 | `docx:document:readonly` |
