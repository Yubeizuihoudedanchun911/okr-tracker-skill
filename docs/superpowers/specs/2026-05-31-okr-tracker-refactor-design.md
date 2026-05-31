# OKR Tracker Skill 优化设计文档

**日期**：2026-05-31  
**背景**：首次端到端测试后发现的体验问题，本次优化目标是消除多轮打断授权、统一前置检查、支持多文档模式、增强图片周报处理能力。

---

## 一、问题归因

| 问题 | 根因 |
|------|------|
| lark-cli 未安装时无引导 | 没有前置检查步骤 |
| 3 轮独立授权扫码 | scope 按需申请，每次打断用户 |
| config.json 在 okr-setup 末尾顺带填写 | 没有独立的初始化引导 |
| user_identifiers 运行时才发现缺失 | 填写时没有验证 |
| shared/SKILL.md 被意外暴露给用户 | 内部文件与用户 skill 同目录 |
| 图片周报无法分析 | 只有文字解析逻辑，缺少图片处理流程 |
| 一周一份文档无法自动发现 | doc_url 只支持单一固定 URL |

---

## 二、文件结构变更

```
okr-tracker-skill/
├── shared/
│   ├── _shared.md          # 原 SKILL.md 改名（内部文件，加下划线前缀）
│   └── _setup-check.md     # 新增（内部文件）
├── okr-setup/
│   └── SKILL.md            # 修改：引用 _setup-check，精简 Step 8
└── okr-review/
    └── SKILL.md            # 修改：引用 _setup-check，新增双模式 + 图片处理
```

**命名约定**：下划线前缀（`_*.md`）表示内部文件，不对用户暴露，不链接进 `~/.claude/skills/`。

**安装链接调整**：

```bash
# 只链接用户可调用的 skill
ln -s $(pwd)/okr-setup  ~/.claude/skills/okr-setup
ln -s $(pwd)/okr-review ~/.claude/skills/okr-review
# shared/ 目录不再链接
```

两个 user-facing skill 通过相对路径 Read 内部文件：`../shared/_setup-check.md`、`../shared/_shared.md`。

---

## 三、`_setup-check.md` 设计

每次调用 `okr-setup` 或 `okr-review` 时，第一步 Read 此文件并按顺序执行。正常运行时（已安装、已授权、config 已存在）全部检查无感知通过。

### 执行流程

```
Step 0  检查 lark-cli 是否安装
        命令：lark-cli --version
        失败 → 提示：npm install -g @larksuite/cli
               终止当前 skill，等用户安装后重新调用

Step 1  检查 lark-cli config
        命令：lark-cli config show
        未配置 → 后台运行：lark-cli config init --new
                  从输出提取 verification_url
                  运行：lark-cli auth qrcode "<url>" --ascii
                  展示二维码，等待用户完成 → 继续

Step 2  首次批量授权（仅当 ~/.claude/okr/config.json 不存在时触发）
        一次性申请所有 scope：
          docx:document:readonly
          docs:document.media:download
          okr:okr.period:readonly
          okr:okr.content:readonly
        命令：lark-cli auth login --scope "<全部 scope>" --no-wait --json
        提取 verification_url → 生成二维码 → 等待用户完成

Step 3  检查 ~/.claude/okr/config.json
        不存在 → 引导填写：
          · user_name：显示名称
          · user_identifiers：飞书昵称、拼音、@昵称（提示多填几种写法）
          · doc_mode："parent"（一周一份子页面）或 "single"（单文档追加）
          · doc_url：父级 wiki URL 或固定文档 URL
          · timezone（默认 Asia/Shanghai）
          · review_day（默认 manual）
        写入 config.json，current_quarter 自动填当前季度
        创建 ~/.claude/okr/ 目录（如不存在）

Step 4  增量 scope 检查（每次运行都执行，轻量）
        命令：lark-cli auth check --scope "<所有 scope>"
        有缺失 → 仅补授权缺失部分，不重复扫码已有 scope
```

### 触发规则

| 条件 | 触发步骤 |
|------|---------|
| lark-cli 未安装 | Step 0（终止） |
| lark-cli 未配置 | Step 1 |
| config.json 不存在 | Step 2 + Step 3 |
| config.json 已存在 | 仅 Step 4（轻量检查） |

---

## 四、config.json Schema 变更

新增字段：

```json
{
  "user_name": "董梦圆",
  "user_identifiers": ["dongmengyuan", "jijunling", "董梦圆", "@董梦圆"],
  "doc_mode": "parent",
  "doc_url": "https://xxx.feishu.cn/wiki/parent-page",
  "doc_url_history": [],
  "timezone": "Asia/Shanghai",
  "review_day": "manual",
  "current_quarter": "2026-Q2"
}
```

| 字段 | 变更说明 |
|------|---------|
| `doc_mode` | 新增。`"parent"` 或 `"single"` |
| `doc_url` | 含义随 `doc_mode` 变化：parent 模式为父级 wiki URL；single 模式为固定文档 URL |
| `doc_url_history` | 新增。记录最近 5 次 ad-hoc 使用的 URL，供快速选择 |

---

## 五、`okr-review` 文档定位与图片处理流程

### 文档来源确定（新增前置步骤）

```
来源优先级：
1. 用户本次指定（「帮我 review 这篇 <url>」）→ 使用指定 URL，记入 doc_url_history
2. doc_mode = "parent" → 拉取父页面子页面列表，按标题/时间找最近一篇
3. doc_mode = "single" → 直接使用 config.doc_url
```

父页面子页面发现逻辑：优先匹配标题含 `YYYY-MM-DD`、`W\d+`、`第\d+周`、`周报` 的最新页面。

### 文档处理流程（含图片周报）

```
Step 1  拉取文档内容
        lark-cli docs +fetch --api-version v2 --doc "<url>"

Step 2  检测内容类型
        · 纯文字 → 走原有文字解析流程
        · 含 <img> 标签 → 走图片增强流程（下方）

Step 3  定位用户段落
        在文档文字内容中，按 user_identifiers 匹配标题或 @ 提及
        确定用户段落的起止范围（到下一个同级/更高级标题为止）
        未找到 → 报错，提示检查 user_identifiers 配置

Step 4  图片增强（仅当段落内含图片时触发）
        4a. 提取用户段落范围内的 <img> src token（不处理段落外图片）
        4b. 并行下载：lark-cli docs +media-download --token <src>
        4c. 重建用户段落骨架：
              保留文字、标题层级
              图片位置用 [IMAGE_1]、[IMAGE_2]... 占位
        4d. 整体传入 Claude：
              · 用户段落骨架（文字上下文 + 图片占位符）
              · 对应图片（按 IMAGE_N 顺序）
              · 当前季度 OKR 目标列表
              Prompt：提取与每条 KR 相关的最新进展数据
        4e. 输出结构化进展摘要，进入 Step 5

Step 5  对比分析（原有逻辑）
Step 6  更新 OKR 文件（原有逻辑）
Step 7  输出分析报告（原有逻辑）
```

### 图片处理失败处理

| 场景 | 处理 |
|------|------|
| 下载失败（权限不足） | 提示检查 `docs:document.media:download` scope，引导补授权 |
| 部分图片识别失败 | 跳过该图，备注「图片内容未识别」，继续分析其余内容 |
| 全部图片下载/识别失败 | 提示用户手动提供本周进展，进入手动输入流程 |

---

## 六、`_shared.md` Scope 表更新

| Skill | 所需 scope |
|-------|-----------|
| okr-setup（方式 A，粘贴文本） | 无 |
| okr-setup（方式 B，飞书拉取） | `okr:okr.period:readonly okr:okr.content:readonly` |
| okr-review（文字文档） | `docx:document:readonly` |
| okr-review（含图片） | `docx:document:readonly` `docs:document.media:download` |
| 首次初始化（_setup-check） | 上述全部 |

---

## 七、`okr-setup` 改动摘要

- 顶部新增：`MUST Read ../shared/_setup-check.md 并执行前置检查`
- 移除原有「lark-cli 鉴权」章节（迁移至 `_setup-check.md`）
- Step 8（更新 config）精简：若 config.json 已存在，只更新 `current_quarter`，不重新引导填写

---

## 八、实施顺序

1. 新建 `shared/_setup-check.md`
2. 将 `shared/SKILL.md` 改名为 `shared/_shared.md`，更新内容（scope 表、doc_mode 字段说明）
3. 更新 `okr-setup/SKILL.md`
4. 更新 `okr-review/SKILL.md`
5. 更新 `README.md` / `README.zh.md`（安装链接、字段说明）
