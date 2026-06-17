<div align="center">

# 测试用例生成器.skill

<p align="center">
  <sub>输入一份产品需求文档，输出 132 条结构化测试用例 Excel</sub>
  <br/>
  <sub>覆盖 12 种类型 · 待确认问题清单 · 一键出 Excel</sub>
</div>

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Agent Skills](https://img.shields.io/badge/Agent%20Skills-Standard-green)](https://agentskills.io)

<br>

**测试用例生成器帮你从 PRD 直接产出可执行的测试用例 Excel，覆盖正向流程、UI 展示、权限、边界、异常、网络状态、前后台状态、并发竞态等 12 种类型。不是生成自动化脚本，是生成手工测试用例。**

<sub>基于开放的 [Agent Skills](https://agentskills.io) 协议，可在 Claude Code、Codex、Cursor、OpenClaw、Hermes Agent 等 skills-compatible runtime 中运行。</sub>

<br>

[效果示例](#效果示例) · [安装](#安装) · [工作原理](#工作原理) · [仓库结构](#仓库结构)

<br>

---

## 效果示例

### 输入

一份「标签和搜索」功能的产品需求 PDF（6页），描述了听蓝 App 的标签创建/重命名/删除/移动、为录音添加标签、首页搜索等 5 个核心功能。

### 输出

132 条测试用例 Excel（7 个 Sheet）：

| Sheet | 内容 | 数量 |
|-------|------|------|
| 说明 | 项目名称、来源格式、日期 | 8 条 |
| 公共前置条件 | 登录账号、录音数据、标签数据等 | 7 条 |
| 页面路径约定 | 11 个页面/场景的完整入口路径 | 11 条 |
| 需求质量问题 | 🔴矛盾、🟡含糊、🔴缺失、🟡歧义等问题 | 15 条 |
| 需求覆盖矩阵 | 14 个功能点的覆盖状态和用例 ID 范围 | 18 条 |
| 待确认问题 | 阻塞级和假设级问题清单 | 15 条 |
| 测试用例 | 完整的 132 条测试用例 | 132 条 |

### 用例类型分布

| 类型 | 数量 | 说明 |
|------|------|------|
| UI 展示 | 38 | 每个页面元素、文案、格式、排序都有独立验证 |
| 正向流程 | 32 | 用户按主路径完成操作 |
| 输入边界 | 24 | 为空、最小值、最大值、超长、非法字符、空格 trim |
| 反向流程 | 9 | 取消、返回、关闭弹窗 |
| 数据一致性 | 8 | 修改后关联页面同步、删除后录音不受影响 |
| 网络状态 | 7 | 无网络、弱网、网络恢复 |
| 权限场景 | 4 | 固定标签不可操作、越权 |
| 异常流程 | 4 | 重名提示、空态 |
| 并发竞态 | 4 | 快速连续点击、多端同时操作 |
| 前后台状态 | 2 | 切后台、杀进程 |

### 单条用例示例

```
TC-20 标签删除-弹窗展示
- 优先级：P0
- 用例类型：UI展示
- 入口路径：听蓝首页 → Tab页 → 展开按钮 → 管理按钮 → 管理标签页 → 删除按钮
- 前置条件：管理标签页已打开，有自定义标签"项目A"
- 操作步骤：
  1. 进入管理标签页
  2. 点击自定义标签"项目A"的删除按钮
- 预期结果：
  1. 弹出双按钮提示弹窗
  2. 弹窗标题显示"提示"
  3. 弹窗文案显示"删除后，标签中的录音不会被删除，仅删除该标签分组，是否继续？"
  4. 左按钮显示"取消"（黑色字体）
  5. 右按钮显示"继续"（蓝色字体）
- 可观测校验点：弹窗标题、文案、按钮文案和颜色
- 自动化建议：适合自动化
```

---

## 三条铁律

1. **🔴 入口路径必须完整**：每条用例从起始页面写到当前操作页面，不得省略
2. **🔴 一条用例只验证一个验证点**：不同前置状态、不同操作分支、不同状态矩阵行 → 必须拆成独立用例
3. **🔴 禁止擅自简化内容**：步骤、预期结果必须完整，输出过长时向用户说明，不得砍内容

---

## 安装

### 方式一：一行命令（推荐，跨 runtime）

打开你的 agent（Claude Code、Codex、Cursor 等），告诉它：

```
帮我安装这个 skill：https://github.com/yourname/autotest-generate-cases
```

或者用通用 CLI 安装器：

```bash
npx skills add yourname/autotest-generate-cases
```

### 方式二：手动安装

| Runtime | 安装路径 |
|---------|---------|
| Claude Code | `~/.claude/skills/autotest-generate-cases/` |
| Codex CLI | `~/.codex/skills/autotest-generate-cases/` |
| Cursor | `~/.cursor/skills/autotest-generate-cases/` |

```bash
git clone https://github.com/yourname/autotest-generate-cases <上面对应的路径>
```

### 方式三：作为参考资料使用

把 `SKILL.md` 的内容粘贴进对话——它本质就是一份 markdown + YAML frontmatter。

---

### 使用

装好后，告诉 agent：

```
> 根据这个需求文档 /path/to/prd.pdf，生成测试用例
> 写用例
> 需求转用例
> test case
```

支持 PDF、Word (.docx)、Markdown (.md) 格式的需求文档。

---

## 工作原理

输入一份需求文档后，测试用例生成器执行 7 步强制流程：

| 步骤 | 说明 | 🔍 需读取参考手册 |
|------|------|-----------------|
| 1. 阅读需求文档 | PDF/Word 先提取文本，Markdown 直接读取 | ✅ PRD 输入格式适配 |
| 2. 提取信息 | 功能点、页面入口、角色权限、文案、状态规则 | - |
| 3. 拆分展开清单 | 为每个功能点列出要写哪些用例 | ✅ 用例拆分原则 |
| 4. 输出覆盖矩阵 | 功能点 × 覆盖用例数 × ID 范围 | - |
| 5. 标出阻塞/待确认问题 | 阻塞级和假设级问题清单 | - |
| 6. 逐条生成测试用例 | 每个功能点用例数 ≥ 8，总数在 功能点×8 ~ ×15 之间 | ✅ 数量预估规则 |
| 7. 生成 Excel | openpyxl 脚本生成 7 个 Sheet 的 xlsx 文件 | ✅ Excel 生成机制 |

最终产物是包含 7 个 Sheet 的 Excel 表格：
- 说明、公共前置条件、页面路径约定、需求质量问题、需求覆盖矩阵、待确认问题、测试用例

---

## 适用场景

| 场景 | 说明 |
|------|------|
| QA 工程师写手工测试用例 | 从 PRD 直接产出结构化用例，不写自动化脚本 |
| 测试团队快速覆盖新功能 | 132 条用例覆盖 12 种类型，不漏边界和异常 |
| 多端产品测试 | 支持 App（iOS/Android/HarmonyOS）、Web、API 等不同端的覆盖规则 |

**不适用的场景**：
- ❌ 不生成自动化测试脚本代码
- ❌ 不适合没有需求文档的纯技术任务

---

## 仓库结构

```
autotest-generate-cases/
├── SKILL.md                          # 执行协议（核心文件）
├── README.md                         # 本文件
├── LICENSE                           # MIT 许可证
├── references/
│   └── autotest-generate-cases-reference.md  # 参考手册（8 个章节）
└── examples/
    └── tags-and-search/              # 效果示例：标签和搜索功能
        ├── input.md                  # 提取的需求文本
        └── output-summary.md         # 产出概览（132 条用例的结构摘要）
```

---

## 许可证

MIT — 随便用，随便改，随便造。

---

<div align="center">

*输入 PRD，输出 Excel。不写脚本，只写用例。*

<br>

MIT License © 2026
