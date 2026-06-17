---
name: autotest-review-script
description: Review Airtest+Poco automated test scripts generated from test cases before delivery or after failures. Use when the user asks to review generated scripts, analyze why an autotest script fails, validate PageObject/pytest layering, check whether operation steps and validation results match the source test cases, check image fallback assets, verify locator traceability to UI trees, or decide whether generated scripts are ready to run or deliver.
---

# 自动化脚本审查 Skill

## 目标

对 `/autotest-generate-script` 生成的 PageObject 和 pytest 脚本做交付前审查或失败后归因，拦住“能生成、但一跑就死”的问题。

默认只审查和输出报告，不修改脚本。只有用户明确要求“修改/修复/按审查结果改”时，才按最小必要范围改代码。

## 输入

优先读取以下材料：

1. 工程根目录 `CLAUDE.md`（如存在）：作为工程约定和交付红线基线。
2. `testcase-workflow/script-context/`：尤其是 `project-structure.md`、`automation-scope.md`、`element-mapping.md`、`image-review-list.md`、`implementation-plan.md`、`handoff-summary.md`。
3. 源测试用例：优先读取 `implementation-plan.md`、`automation-scope.md` 中的用例步骤/预期；如资料包不完整，再读取用户提供的 Excel、文本用例或 `script-clarify/input-inventory.md` 指向的原始用例。
4. 生成或修改的 PageObject 文件，通常在 `modules/`。
5. 生成或修改的 pytest 文件，通常在 `testcases/`。
6. 页面 UI 树、截图和 crops，通常在 `tools/pages/`。
7. 正式图片目录，通常在项目根目录 `images/`。
8. 用户提供的运行日志、pytest 输出、失败截图或当前 UI 树。

如果缺少脚本文件、上下文资料包、UI 树或日志中关键证据，必须说明缺什么，不要脑补结论。

如果缺少源测试用例或 `implementation-plan.md` 中没有操作步骤/预期结果映射，必须说明无法完成“测试用例一致性审查”，不能只按代码结构给出可交付结论。

## 输出

输出到对话中；如果用户要求保存报告，写入：

`testcase-workflow/script-review/review-report.md`

报告结论必须是以下四类之一：

| 结论 | 含义 |
| --- | --- |
| 可交付 | 静态审查通过，风险可接受 |
| 需修复后交付 | 有明确代码问题，修复后可继续 |
| 需补材料 | 缺 UI 树、正式图片、测试数据或用户确认，不能可靠判断 |
| 不建议自动化 | 用例本身不适合 UI 自动化或维护成本明显过高 |

问题按严重级别输出：

| 级别 | 含义 |
| --- | --- |
| P0 | 会导致脚本无法启动、无法收集、必现运行失败 |
| P1 | 主流程高概率失败或断言无效 |
| P2 | 稳定性/维护性风险，短期可运行但容易波动 |
| P3 | 命名、日志、可读性、轻微结构问题 |

每个问题必须包含：文件行号、证据、影响、建议修复。

## 审查流程

### 1. 确认审查范围

- 从用户指定文件、最近生成文件、`implementation-plan.md` 或 pytest 收集结果确定范围。
- 只审查当前任务相关文件，不扩大到无关模块。
- 如果工作区已有用户改动，不回滚、不覆盖；只指出与本次审查相关的风险。
- 先判断审查目标是“阶段一生成稿”还是“交付/可运行稿”。用户问“能不能跑/交付/上线回归”时按交付态审查；用户问“生成结果是否合理/图片如何确认”时按阶段一生成稿审查。
- 若存在 `CLAUDE.md`，必须用其中的工程约定作为审查基线；如果资料包或代码与 `CLAUDE.md` 冲突，按用户最新确认和 `CLAUDE.md` 判定风险。

### 2. pytest 层审查

pytest 用例只能做流程编排和断言入口，必须只调用 PageObject 公共业务方法。

禁止项：

- 调用 PageObject 私有方法，例如 `page._xxx()` 或 `._xxx(`。
- 出现 `base_action`、`baseutil`、`android_poco`、`ios_poco`、`poco(`。
- 出现 `IOSElements`、`AndroidElements`。
- 出现 Airtest `Template(`、图片文件名、坐标点击、坐标滑动。
- 直接写 locator、复杂滚动、底层输入细节。
- 调 `_handle_platform_popup*`。

如果命中，判为 P0/P1，并建议把步骤收敛到 PageObject 公共方法。

### 3. 测试用例一致性审查

必须检查生成脚本的操作步骤和校验结果是否符合源测试用例。审查对象包括 pytest 编排、PageObject 公共方法、关键私有方法和断言方法。

逐条用例建立映射：

| 测试用例字段 | 脚本侧证据 |
| --- | --- |
| 用例 ID / 名称 | pytest 函数名、注释、marker、allure 标题或实现计划映射 |
| 入口路径 / 前置条件 | fixture、归位方法、入口公共方法、skip 条件 |
| 操作步骤 | pytest 调用顺序 + PageObject 公共方法内部业务动作 |
| 测试数据 | fixture 数据、方法参数、配置读取、专用测试数据 |
| 预期结果 / 可观测校验点 | PageObject 断言方法、返回值判定、pytest assert |

重点检查：

- 脚本是否覆盖源测试用例的每个核心操作步骤，顺序是否一致。
- 脚本是否漏掉入口路径、前置状态构造、关键点击/输入/选择/取消/确认动作。
- 脚本是否新增了测试用例未要求的破坏性动作、跨账号动作、清理动作或额外断言。
- 测试数据是否与用例要求一致；不能把“任意数据”硬编码成固定业务数据，不能用共享线上数据替代专用测试数据。
- 每条预期结果是否都有对应可观察校验；不能只校验“方法返回 True”或“点击成功”。
- 负向用例是否校验错误提示、按钮不可用、弹窗未关闭或数据未变化，而不是只执行到异常步骤。
- 排序、分页、权限、删除、重命名、筛选等用例，必须校验用例指定的结果，不得只校验页面未崩溃。
- 如果 `implementation-plan.md` 与源测试用例冲突，必须标记冲突；不能默认按代码或实现计划覆盖源用例。

判定规则：

- 脚本主流程与测试用例操作步骤不一致，或漏掉核心步骤：P1。
- 脚本断言没有覆盖测试用例主要预期结果：P1。
- 脚本只返回 True、只检查页面存在、只检查点击成功，但没有验证预期：P1/P2。
- 脚本使用了错误测试数据、错误账号/角色/平台，导致验证对象偏离用例：P1。
- 脚本增加了用例未要求且有副作用的动作：P1/P2。
- 用例本身步骤或预期不清，导致无法判断一致性：结论为「需补材料」，并列出需要补充的用例字段。

输出报告中必须单独给出“用例一致性”结论：一致 / 不一致 / 资料不足。若不一致，列出对应用例 ID、源用例要求、脚本实际行为、影响和修复建议。

### 4. PageObject 分层审查

PageObject 可以封装底层操作，但必须对外暴露稳定公共业务方法。

重点检查：

- 公共方法是否返回 `True` / `False`，是否有清晰日志和异常处理。
- 入口公共方法是否在每次跳转后验证目标页面关键元素，不允许只打印“已进入xxx”。
- 上一步失败后是否继续执行下一步；若继续，判为 P1。
- 页面跳转、弹窗关闭、列表刷新后是否重新获取 Poco 节点，避免旧节点缓存。
- 私有方法是否只封装单页动作，pytest 是否没有直接调用它。
- 数据变更类方法是否有前置数据、清理或幂等策略。

### 5. UI locator 和输入控件审查

所有 locator 必须能追溯到 `element-mapping.md` 或 UI 树。

输入框、文本域、可编辑控件必须重点检查：

- 不得根据“输入框”语义脑补 `UITextField`、`EditText` 或其他控件类型。
- 必须以 UI 树真实 `name` / `type` / `text` / `value` 为准。
- `StaticText` placeholder 只能作为提示文案或辅助校验，不能当作输入控件本体。
- 如果 UI 树真实节点是 `TextView`，代码和映射必须使用 `TextView` 或项目可用封装，不能写 `UITextField`。
- 输入前要确认控件出现，输入后要能反查 UI 状态；输入失败不能继续点保存/确定。

发现 locator 和 UI 树不一致，判为 P0/P1。

### 6. Airtest+Poco API 审查

禁止把 Selenium/Appium API 写进 Poco 代码。

重点扫描：

- `get_attribute(`：P0/P1。Poco `UIObjectProxy` 不是 Selenium 元素，需改为项目支持的 API，例如 `attr(...)`，或改用页面状态断言。
- `input_text()` 传入 `UIObjectProxy`：P1。项目封装通常要求传字符串 locator。
- `poco(name="xxx")[0].parent()` 等高风险链式寻址：P2。优先拆成稳定查询或复用项目封装。
- 直接 `set_text()` 用在 Flutter 混合页面：P1/P2。优先点击聚焦后使用项目输入封装或 Airtest 原生输入。
- 未设置循环上限的滚屏查找：P1/P2。

### 7. 图片兜底审查

区分阶段一生成稿和交付/可运行稿：

- `tools/pages/**/crops/` 是候选图和扫描期物料，不是运行依赖。
- 正式运行图片必须在项目正式图片目录中真实存在，通常是 `images/`。
- 阶段一生成稿允许代码引用 `image-review-list.md` 中的物料真实文件名，前提是清单提供完整候选图路径，方便用户人工确认。
- 物料无图时允许代码引用语义化建议正式名作为待补图提示，前提是清单标注 `缺物料需补图`。
- 交付/可运行稿中，代码里出现的每个 `*.png` 都必须检查是否存在于正式图片目录。
- `_handle_platform_popup*` 链路只要传入图片文件名，就必须能在 `image-review-list.md` 中追溯；交付/可运行态还必须确认文件已在 `images/`。
- 只有建议正式图片名或 crops 候选图时，不能判为可交付；若没有稳定 Poco locator，应判为「需补材料」。
- 不得擅自复制、替换或新增正式图片，除非用户明确要求。

图片审查结论：

- 阶段一生成稿：图片名能追溯到 `image-review-list.md`，但未在 `images/`，判为 P2/P3 或「需补材料」，不要误判成生成错误。
- 交付/可运行稿：图片名不在 `images/`，判为 P0/P1。
- 运行日志出现 `File not exist: .../*.png`，判为 P0。

### 8. 断言和用例价值审查

每条用例必须能证明预期结果。

重点检查：

- 断言必须能追溯到源测试用例的预期结果或 `implementation-plan.md` 的“预期结果 -> 断言方法覆盖表”。
- 不能只断言“点击成功”或“方法返回 True”，必须覆盖用例预期。
- 创建、删除、重命名、排序、权限、列表刷新等必须有可观察结果。
- 负向用例必须断言错误提示、按钮不可用、弹窗未关闭或数据未变化。
- 破坏性操作必须使用专用测试数据，避免误删共享数据。
- 依赖固定录音、固定标签、跨账号状态时，必须写明前置和清理。

断言无效或无法观察，判为 P1/P2；预期不可自动化时判为「不建议自动化」或「需补材料」。

## 运行日志归因

拿到日志时，先判断失败发生在哪一层，不要只看最后一行。

常见签名：

| 日志特征 | 归因 |
| --- | --- |
| `ios-darwin ... info/launch failed` 但随后 `devicectl已启动IOS应用` | 环境 warning，通常不是脚本主因 |
| 已打印“已进入xxx”，随后找功能元素失败 | 入口可能已成功，后续页面状态或定位失败 |
| `File not exist: .../*.png` | 正式图片缺失，图片兜底门禁失败 |
| `Cannot find any visible node ... "UITextField"` | locator 与 UI 树不一致，常见为把真实 `TextView` 写成 `UITextField` |
| `'UIObjectProxy' object has no attribute 'get_attribute'` | 写入了 Selenium 风格 API |
| 找到弹窗标题后找不到输入框 | 输入控件定位错误或未使用聚焦后的 UI 树 |
| 点击后只 sleep，没有目标状态校验 | PageObject 状态断言不足 |

归因输出格式：

1. 直接失败点。
2. 上游诱因。
3. 是否是 case 步骤问题、skill 问题、模型生成问题、工程封装问题或环境问题。
4. 最小修复建议。

## 基础校验

审查时优先做只读检查；如果用户允许或任务需要，可运行：

- `python -m py_compile <PageObject> <pytest文件>`
- `pytest --collect-only <pytest文件>`
- `rg` 静态扫描禁用项和图片名。

如果运行校验会连接真机、修改数据、关闭应用或触发破坏性业务操作，必须先说明风险；没有用户要求时不要直接跑真机用例。

## 最终回答格式

先给结论，再给问题。

推荐格式：

```markdown
结论：需修复后交付

用例一致性：不一致

P0 [文件:行号] 问题标题
证据：...
影响：...
建议：...

P1 ...

残余风险：...
未运行校验：...
```

如果没有发现问题，明确说“未发现阻塞交付的问题”，并列出仍未验证的运行条件。
