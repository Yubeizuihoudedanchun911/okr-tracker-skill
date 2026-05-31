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
