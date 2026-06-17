---
name: autotest-generate-script
description: 根据上下文交接资料包生成 Airtest+Poco+pytest 自动化脚本代码。当用户提到"生成脚本""写自动化脚本""用例转脚本""脚本生成""generate script""testcase to script"时触发。需要上下文交接资料包作为输入。不是问题确认，也不是总结上下文。
---

# 根据上下文交接资料包生成 Airtest+Poco+pytest 自动化脚本 — 执行协议

> 详细规则、框架排雷、编码规范、断言规则、失败处理规则等，全部放在脚本生成规则参考手册中。
> 脚本生成规则参考手册路径：`references/autotest-generate-script-reference.md`

## 适用范围

本 Skill 适用于任何基于 Airtest + Poco + pytest 的移动端 UI 自动化工程，覆盖 Android、iOS 及 Flutter 混合页面。

核心前提：

- UI 定位基于 Poco（name、text、resourceId、sibling 等）+ Airtest 图片识别兜底。
- 业务分层基于 PageObject 模式 + pytest 编排。
- 多平台差异基于项目约定的平台标识变量做显式分支。

## 职责边界

本 Skill 只负责生成脚本代码：

- 输入：`autotest-summary-case` 输出的上下文交接资料包（`testcase-workflow/script-context/`）。
- 输出：PageObject 代码文件 + pytest 用例文件。
- 不重新做测试用例文本规范审查（已在 `autotest-check-case` 完成）。
- 不重新做自动化落地可行性分析、范围判断和元素映射（已在 `autotest-summary-case` 完成）。
- 不擅自新增或修改正式图片目录中的图片。

## 核心原则

### 必须读取上下文交接资料包

本 Skill 不允许仅凭对话中的一句话或猜测写脚本。必须先读取 `testcase-workflow/script-context/` 中的交接资料包，按固定读取顺序获取所有必要信息。

如果 `testcase-workflow/script-context/` 不存在，提示用户先执行 `/autotest-summary-case`。

### 转换链路不允许跳层

AI 写脚本时必须沿着固定链路转换：

```text
-> 读取上下文交接资料包（自动化范围、元素映射、实现计划、用户确认）
-> 生成或复用 PageObject 方法
-> 编排 pytest 测试用例
-> 输出前自检和基础校验
```

转换要求：

| 阶段 | AI 应做什么 | AI 不应做什么 |
| --- | --- | --- |
| 读取上下文 | 按固定顺序读取交接资料包，严格按资料包中的内容生成脚本 | 不凭对话猜测用例、元素或方法 |
| 脚本生成 | 生成或复用 PageObject 方法，再写 pytest 用例 | 不把完整业务流程堆进测试函数 |
| 测试用例层 | 只调用 PageObject 公共方法、fixture、pytest marker 和 skip 条件 | 不直接写 Poco locator、图片名、坐标、`base_action`、`baseutil` 或 PageObject 私有方法 |
| 调试维护 | 根据日志、截图、UI 树和失败类型收敛修复 | 不做无关重构，不修改用户未授权图片资产 |

## 固定输入目录

上下文交接资料包路径：

`testcase-workflow/script-context/`

读取顺序：

1. `handoff-summary.md` — 交接摘要（先看全局概况）。
2. `project-structure.md` — 项目关键路径和工程约定。
3. `automation-scope.md` — 可立即自动化、待补充、暂不自动化的用例范围。
4. `element-mapping.md` — 页面元素映射表。
5. `image-review-list.md` — 图片人工确认清单，定义阶段一代码引用名、候选图路径和建议正式名。
6. `implementation-plan.md` — PageObject 方法、断言方法和 pytest 编排计划。
7. `user-confirmations.md` — 用户确认结论和假设授权。

兼容旧资料包：如果 `automation-scope.md` 不存在，可以读取旧版 `testcases-to-automate.md` 和 `testcases-pending.md`；如果 `implementation-plan.md` 不存在，可以读取旧版 `pageobject-methods.md` 和 `assertion-design.md`。读取旧资料包时，仍必须按本 Skill 的自检规则生成代码。

如果 `testcase-workflow/script-context/` 不存在或缺少标记文件 `.managed-by-autotest-summary-case`，必须提示用户先执行 `/autotest-summary-case`。

`image-review-list.md` 是必读文件。即使本次无图片兜底，也应由 `/autotest-summary-case` 生成并写明“无”；如果缺失，停止生成并提示用户重新执行 `/autotest-summary-case`。

如果工程根目录存在 `CLAUDE.md`，生成代码前必须直接读取。`project-structure.md` 是摘要，不能完全替代工程原始规则。若 `CLAUDE.md` 与资料包内容冲突，以用户最新确认和 `CLAUDE.md` 中的工程约定优先，并在输出自检中说明采用了哪条规则。

## 工作流程（4 步，必须按顺序执行）

以下 4 步是强制流程，每一步必须完成后才能进入下一步。标注「参考」的步骤必须读取脚本生成规则参考手册（`references/autotest-generate-script-reference.md`）对应章节后才能继续。

### 步骤 1：读取上下文交接资料包

参考「一、脚本生成前的必要输入」章节。

按上述固定读取顺序，逐个读取 `testcase-workflow/script-context/` 下的文件。

读取完毕后确认以下信息齐全：

- 项目关键路径已确定。
- 可立即自动化的用例清单已获取。
- 每条立即自动化用例的页面元素映射已完成。
- PageObject 公共/私有方法设计已明确。
- 断言方法覆盖表已完成。
- pytest 编排计划已完成。
- 用户确认结论已获取。
- 图片兜底状态已明确。
- `image-review-list.md` 已存在；如果脚本会引用图片，必须给出阶段一代码引用名、候选图路径、建议正式名和 images 存在状态；如果本次无图片，必须明确写明无图片引用。

如果交接资料包中某项信息缺失，向用户说明缺什么，标记为「补充材料后生成」，不得自行补造，不得回头重新做范围判断或元素映射。

### 步骤 2：生成 PageObject 代码

参考「四、PageObject 编写规则」章节。

按交接资料包中的 `implementation-plan.md` 和 `element-mapping.md` 生成 PageObject 代码。

PageObject 规则：

- 放在项目的 PageObject 目录下，按业务模块组织。
- 按项目约定定义平台元素常量类（常见约定如文件顶部集中定义各平台的元素定位常量）。
- 新业务优先查找相似模块和公共方法，能复用必须复用。
- 入口方法不要默认当前页面正确，除非有明确前提。
- 单页动作封装为私有方法，对外暴露稳定公共方法。
- 公共方法必须 try...except Exception 包裹，返回 True / False。
- 多平台差异按项目约定显式处理（常见约定如用平台标识变量做 `if/elif` 分支）。
- 动态数据从配置读取，不硬编码账号、密码、组织、固定录音等环境数据。
- 业务文案和元素名集中管理。
- 页面跳转后必须重新获取 Poco 节点，避免旧节点缓存导致误判（Poco 缓存残影问题）。
- 写图片兜底时必须先查 `image-review-list.md`，不得绕过清单自行编图片名。该清单应已检查页面 `prompt.md` 弹药库、`tools/images/` 候选池和 `tools/pages/**/crops/` 页面裁图。物料有图时，阶段一代码引用名使用物料真实文件名；物料无图时，才使用语义化建议正式名提示补图。不得把建议正式名误当成已有物料名。
- 缺正式图片时不得擅自复制或新增图片；如果代码引用名尚未在正式图片目录存在，必须在自检清单和待补充清单中标注“未达交付态/真机运行前需补齐 images”。
- 输入框、文本域、可编辑控件的 locator 必须来自 `element-mapping.md` 中记录的 UI 树真实节点来源（`name` / `type` / `text` / `value`），不得根据业务语义脑补为 `UITextField`、`EditText`。如果资料包记录真实节点是 `TextView`，代码必须按 `TextView` 或项目可用封装定位。
- placeholder 文案（例如“请输入标签名称”）如果在 UI 树中是 `StaticText`，只能用于确认弹窗或辅助定位，不能当作输入控件本体。
- 写 Poco 属性读取时必须使用本项目支持的 API 或封装；禁止使用 Selenium 风格的 `get_attribute()`。如需读取属性，优先确认项目现有写法（例如 Poco `attr("enabled")`）或改用可观察 UI 状态断言。

### 步骤 3：生成 pytest 用例代码

参考「五、pytest 用例编写规则」章节。

只为 `automation-scope.md` 中明确列为「立即自动化」的用例生成 PageObject 和 pytest 代码。其他类别默认不生成；只有 `implementation-plan.md` 明确要求生成 `pytest.skip` 占位用例时，才生成 skip 用例。

按交接资料包中的 `automation-scope.md` 和 `implementation-plan.md` 生成 pytest 用例代码。

如果 `implementation-plan.md` 的 pytest 编排计划包含 `_xxx` 私有方法、`base_action`、`baseutil`、`poco`、`IOSElements`、locator、图片名或坐标，禁止照抄到 pytest。必须先把这些底层步骤收敛为 PageObject 公共业务方法；若公共方法不存在，就先补充 PageObject 公共方法，再让 pytest 只调用公共方法。

pytest 用例规则：

- 放在项目的测试用例目录中。
- 测试函数名表达模块、页面、操作和条件。
- 每条用例尽量验证一个主要目标。
- 使用 fixture 获取页面对象。
- 用 `assert page.public_method()` 表达验证结果。
- 用 pytest marker 管理 P0、模块、回归范围。
- 多平台不一致时在 PageObject 分支处理，不在用例中复制两套。
- 若资料包中标注某用例仍缺材料或存在影响行为的待确认项，默认不生成该用例；如 `implementation-plan.md` 明确要求保留占位，则必须使用 `pytest.skip`，不得用注释代替。

pytest 用例层硬性禁止出现：

- PageObject 私有方法调用，例如 `page._xxx()`。
- `base_action`、`baseutil`、`android_poco`、`ios_poco`、`poco(`。
- `IOSElements`、`AndroidElements`、locator 字符串。
- Airtest 图片名、坐标、滚动逻辑、locator 字符串。
- `_handle_platform_popup*` 调用。

### 步骤 4：输出前自检和基础校验

参考「十、AI 输出脚本前自检清单」章节。

逐项自检：

| 检查项 | 通过标准 |
| --- | --- |
| 需求来源 | 脚本能追溯到需求功能点或文本用例 |
| 用例清晰 | 每条 pytest 用例有明确标题、前置、动作和断言 |
| 数据明确 | 测试数据可准备、可复用或可清理 |
| 页面映射 | 涉及页面能在映射表或工程模块中找到 |
| 分层正确 | 业务动作在 PageObject 目录，流程编排在测试用例目录 |
| 测试层干净 | pytest 不直接调用私有方法、Poco、base_action、baseutil、图片名或坐标 |
| 定位可靠 | 优先使用稳定 Poco locator，不稳定步骤有图片兜底策略 |
| 图片合规 | 不擅自新增或修改正式图片目录，图片兜底链路不残留 None 占位 |
| 输入控件真实 | 输入框 locator 与 `element-mapping.md` 记录的 UI 树来源一致，未把 placeholder 或猜测 class 当作输入控件 |
| Poco API 合规 | 未使用 `get_attribute()` 等非本项目 Poco API |
| 平台明确 | 多平台差异已处理或明确跳过 |
| 失败可诊断 | 日志、返回值、断言信息能支持定位问题 |
| 回归可维护 | 用例可用 marker 管理，可纳入模块回归 |

pytest 测试文件必须执行静态扫描。以下任一命中都视为生成失败，必须重写测试层或补充 PageObject 公共方法后重新生成，不能交付：

- `page._` 或 `._` 形式的 PageObject 私有方法调用。
- `.base_action`、`.baseutil`、`android_poco`、`ios_poco`、`poco(`。
- `IOSElements`、`AndroidElements`。
- `Template(`、图片文件名、坐标型点击或滑动编排。
- `_handle_platform_popup`。

PageObject 文件也必须执行静态扫描。以下任一命中都视为生成失败，必须重写 PageObject 后再交付：

- `get_attribute(`。
- 生成或修改代码中出现 `UITextField`、`EditText`、`TextView` 等输入控件 locator，但 `element-mapping.md` 没有对应真实节点来源。
- 代码中出现 `*.png` 图片名，但既不在 `image-review-list.md`，也不能追溯到正式图片目录或候选物料。
- 用户要求“可直接运行/交付”时，代码中出现的 `*.png` 在正式图片目录不存在。
- `_handle_platform_popup*` 链路传入的图片文件名没有出现在 `image-review-list.md`，或状态不是 `已在images` / `物料待确认` / `缺物料需补图` 中的一种。

基础校验：

1. 对新增或修改的 Python 文件运行语法校验（如 `python -m py_compile`）。
2. 对相关测试文件运行收集校验（如 `pytest --collect-only`）。
3. 如果设备和环境可用，再按用户要求运行单条 smoke；如果不可用，必须说明未运行原因。

自检完成后，输出：

1. PageObject 代码文件（写入项目 PageObject 目录）。
2. pytest 用例文件（写入项目测试用例目录）。
3. 自检清单结果。
4. 基础校验结果。
5. 待补充材料清单（缺正式图片、缺测试数据、缺 UI 树等）。
6. 图片确认清单状态：说明代码引用的是物料名还是正式名，哪些图片还没有进入 `images/`。
7. 下一步建议：交付前可执行 `/autotest-review-script` 做独立脚本审查；如果生成期间命中过输入控件、图片兜底、Poco API 或分层风险，必须先审查再交付。

## 图片兜底铁律

参考「七、图片兜底规则」章节。

Airtest+Poco 工程采用“元素定位优先 + 图片识别兜底”模式。AI 写脚本时必须尊重开发期施工目录和正式图片目录的边界。

| 目录 | 定位 | 规则 |
| --- | --- | --- |
| 开发期施工目录 | 开发期施工区 | 存放扫描页面、候选截图、Poco 结构树、prompt，不是最终运行依赖 |
| 正式图片目录 | 正式图片资产 | 最终脚本运行依赖，新增或修改必须先经用户确认 |

强制规则：

- 优先使用 Poco 稳定定位（name、resourceId、text、sibling 等）。
- 不稳定定位可以接入 Airtest 图片识别兜底。
- 图片兜底链路的步骤，最终交付不得残留 None 占位。
- 图标图和文字图必须分开，不能混用。
- Android 和 iOS 即使语义相同，也应分别命名图片。
- 图片真实视觉语义优先于扫描器临时命名。
- AI 不得擅自往正式图片目录新增、复制、替换图片。
- 缺正式图片且无稳定 Poco locator 的步骤，不得生成“立即自动化”的可运行用例；应在上下文阶段列为待补充。只有 `implementation-plan.md` 明确要求保留占位时，才在 pytest 中 `skip`。
- 阶段一生成代码时，如果物料已有候选图，可以按 `image-review-list.md` 使用物料真实文件名，方便用户确认和溯源。
- 物料无图时，可以使用语义化建议正式名作为待补图提示，但必须标注未达交付态。
- 凡是写进 PageObject 的图片文件名，交付/真机运行前必须在正式图片目录真实存在；只有 `crops/` 候选图或只有建议正式图片名时，不得声称可交付或直接运行。

## Airtest+Poco 框架排雷

参考「框架排雷与防爆指南」章节。

以下排雷规则适用于所有 Airtest+Poco 工程，写脚本时必须遵守：

### Flutter 混合页面的“画布输入假象”

- 现象：Flutter 页面中直接调用 `poco().set_text()` 会修改底层属性，但不触发 UI 重绘。
- 铁律 A：调用底层 `input_text(element, content)` 时，`element` 必须传入字符串定位符，绝对禁止传入 `UIObjectProxy`。
- 铁律 B：如果 Android 端输入框失去特征只能靠下标定位，无法使用 `input_text`，则必须先 `.click()` 获取焦点，再用 Airtest 原生 `text("内容")` 注入硬件级输入事件。
- 铁律 C：输入框 locator 必须来自 `element-mapping.md` 记录的当前页面 UI 树真实可编辑节点。不要把 `StaticText` placeholder 当输入框，也不要把 iOS 输入控件默认写成 `UITextField`；若资料包记录真实节点为 `TextView`，应按 `TextView` 或项目可用封装处理。
- 铁律 D：Poco 的 `UIObjectProxy` 不是 Selenium 元素，禁止调用 `get_attribute()`；读取属性前必须确认项目 API，优先使用 Poco `attr(...)` 或通过页面状态变化做断言。

### Poco 的链式寻址崩溃与正则风险

- 避免 `poco(name="xxx")[0].parent()` 等容易触发 AST 解析崩溃的链式写法。
- 默认不用正则；遇到项目约定允许动态后缀匹配的场景，才使用 `textMatches` / `nameMatches`，并写明原因。
- 如果前置步骤已经通过搜索过滤到唯一目标，优先使用简单、可见的目标元素点击。

### iOS Flutter 的 Phantom Click

- 列表刷新或滚屏后，必须等待像素稳定。
- 边缘元素点击优先使用 `.focus([0.5, 0.5]).click()`，让 Poco 重新计算物理中心点。

### 智能滚屏雷达的小步快跑原则

- 循环滚屏探测必须采用小步幅滑动。
- 必须配套循环上限，循环内先 `exists()` 判定，再执行 `swipe`。

### Poco 对象缓存残影清理机制

- 页面发生路由跳转、返回、刷新后，必须重新实例化 Poco 节点，避免使用旧节点缓存。

## 输出格式

最终产物包含：

1. PageObject 代码文件（写入项目 PageObject 目录）。
2. pytest 用例文件（写入项目测试用例目录）。
3. 自检清单结果。
4. 基础校验结果。
5. 待补充材料清单。

## 与其他 Skill 的衔接

本 Skill 是脚本生成流程的终点：

```text
autotest-check-case -> autotest-summary-case -> autotest-generate-script
```

- 上游 Skill `autotest-summary-case` 输出的上下文交接资料包是本 Skill 的必读输入。
- 本 Skill 只生成脚本代码，不重新生成测试用例，不审查测试用例文本，不重新判断自动化范围或元素映射。
- 页面名称必须与页面映射表一致。

完整链路：

```text
autotest-check-prd -> autotest-summary-prd -> autotest-generate-cases
-> autotest-check-case -> autotest-summary-case -> autotest-generate-script
```
