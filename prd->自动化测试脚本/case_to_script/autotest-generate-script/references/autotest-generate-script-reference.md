# AI 自动化测试脚本编写规则（Airtest + Poco + pytest）

本规则适用于任何基于 Airtest + Poco + pytest 的移动端 UI 自动化工程，覆盖 Android、iOS 及 Flutter 混合页面。

## 一、脚本生成前的必要输入

AI 不允许仅凭一句功能描述直接写脚本。生成自动化脚本前，必须读取 `testcase-workflow/script-context/` 中的上下文交接资料包。

读取顺序：

1. `handoff-summary.md`：交接摘要、范围、风险和读取顺序。
2. `project-structure.md`：项目关键路径和工程约定。
3. `automation-scope.md`：立即自动化、待补充、暂不自动化的用例范围。
4. `element-mapping.md`：业务元素到 Poco locator / 图片兜底的映射。
5. `image-review-list.md`：图片人工确认清单，定义阶段一代码引用名、候选图路径和建议正式名。
6. `implementation-plan.md`：PageObject 方法、断言方法、pytest 编排计划。
7. `user-confirmations.md`：用户确认、补充、假设授权。

缺失时处理：

| 输入 | 作用 | 缺失时处理 |
| --- | --- | --- |
| `handoff-summary.md` | 先看全局范围和风险 | 停止生成，要求补充上下文 |
| `project-structure.md` | 确定写入目录、fixture、平台变量、图片目录 | 停止生成 |
| `automation-scope.md` | 确定只生成哪些用例 | 停止生成 |
| `element-mapping.md` | 把业务元素映射为稳定定位 | 停止生成，不猜 locator |
| `image-review-list.md` | 确定图片阶段一引用名、候选图和正式名；无图片时也应写明“无” | 停止生成，要求重新执行 `/autotest-summary-case` |
| `implementation-plan.md` | 约束 PageObject、断言、pytest 编排 | 停止生成 |
| `user-confirmations.md` | 防止覆盖用户确认和假设授权 | 停止生成或只生成不依赖确认的部分 |

路径约定以 `project-structure.md` 为准。AI 生成代码前可以读取工程根目录 `CLAUDE.md` 和相关现有代码来确认编码风格、fixture、公共方法和写入目录；不得重新发现页面物料、重做元素映射或重判自动化范围。若 `CLAUDE.md` 与资料包冲突，以用户最新确认和 `CLAUDE.md` 的工程约定优先，并在输出自检中说明。

## 二、从需求到脚本的转换规则

AI 写脚本时必须沿着固定链路转换，不允许跳层。

```text
-> 已评审中文测试用例
-> 问题确认与用户授权
-> 自动化范围判断
-> 元素映射和实现计划
-> PageObject 页面动作和断言
-> pytest 测试用例编排
-> 自检、语法校验、收集校验
```

转换要求：

| 阶段 | AI 应做什么 | AI 不应做什么 |
| --- | --- | --- |
| 需求理解 | 读取页面入口、角色权限、前置状态、精确文案、成功和异常结果 | 不补造需求未写清的业务规则 |
| 用例理解 | 按文本用例理解用户在 UI 层怎么操作、预期怎么观察 | 不把“业务逻辑成立”当成 UI 校验 |
| 可行性判断 | 使用 `automation-scope.md` 的结论，只为「立即自动化」生成代码 | 不重新判断范围，不把待补充用例硬写成可运行用例 |
| 元素映射 | 使用 `element-mapping.md` 定位业务元素 | 不在测试函数中散落临时坐标和猜测 locator |
| 脚本生成 | 生成或复用 PageObject 方法，再写 pytest 用例 | 不把完整业务流程堆进测试函数 |
| 调试维护 | 根据日志、截图、UI 树和失败类型收敛修复 | 不做无关重构，不修改用户未授权图片资产 |

## 三、自动化范围门禁

只有 `automation-scope.md` 中明确列为「立即自动化」的用例才进入脚本生成。其他类别默认不生成；只有 `implementation-plan.md` 明确要求保留占位时，才生成 `pytest.skip` 用例。

| 结论 | 条件 | 后续动作 |
| --- | --- | --- |
| 立即自动化 | 阻塞问题已确认，断言可观测，数据稳定，元素可定位，缺图不影响主路径 | 生成 PageObject 和 pytest 用例 |
| 补充材料后自动化 | 缺 UI 树、截图、页面映射、测试数据、断言文案或正式图片，且会影响脚本稳定运行 | 默认不生成；如实现计划明确要求占位，则生成 `pytest.skip` |
| 暂不自动化 | 需求不稳定、收益低、维护成本高、断言不可观察 | 保留为手工用例 |
| 需要研发支持 | 缺稳定控件标识、状态构造接口、测试数据接口或硬件控制能力 | 提出可测性诉求 |

正式图片门禁：

- `_handle_platform_popup*` 链路如果没有稳定 Poco locator 且缺正式图片，不得列入“立即自动化”。
- 如果有稳定 Poco locator，但缺正式图片兜底，可以生成代码，但必须在自检和待补充清单中列出。
- 不得用 `None` 作为交付版图片占位。
- 不得擅自把候选图复制到根目录 `images/`。

## 四、PageObject 编写规则

PageObject 目录承载业务页面对象。AI 必须优先复用现有模块风格，不要另起一套框架。

### 4.1 放置位置

PageObject 文件放在项目的页面对象目录下，按业务模块组织子目录。具体目录路径和类名格式在读取 `project-structure.md` 和现有代码时确定，不硬性规定。

### 4.2 方法职责

PageObject 负责页面动作、业务动作和页面断言：

```python
def ensure_xxx_page(self):
    """归位到目标页面"""
    pass

def open_xxx_detail(self):
    """打开详情"""
    pass

def rename_xxx(self, new_name):
    """重命名"""
    pass

def assert_xxx_not_in_list(self, xxx_title):
    """断言目标不在列表中"""
    pass
```

pytest 用例只调用 PageObject 暴露的公共方法，不直接写底层 Poco / Airtest 操作。

### 4.3 编码规则

- 新业务优先查找相似模块和公共方法，能复用必须复用。
- 页面入口方法不要默认当前页面正确，除非该业务已有明确前提。
- 单页动作可以封装为私有方法，对外暴露稳定公共方法。
- 公共业务方法必须捕获可预期失败并返回 `True` / `False`，避免把底层异常直接抛给用例层。
- 多平台差异必须显式处理，按项目约定使用分支逻辑。
- 动态数据从配置或测试数据读取，不在代码中硬编码账号、密码、组织、固定录音等环境数据。
- 业务文案和元素名应集中管理，避免在多个方法里重复散落魔法字符串。
- 页面跳转、返回、刷新后必须重新获取 Poco 节点，避免旧节点缓存导致误判。
- Flutter 等混合框架页面输入和点击要遵守 Poco 框架的稳定化规则，必要时使用项目的基础操作封装。

## 五、pytest 用例编写规则

测试用例目录承载 pytest 用例。测试函数只负责编排业务流程和断言结果。

推荐结构：

```python
@pytest.mark.xxx_module
@pytest.mark.regression
@pytest.mark.p0
def test_xxx_rename_success(app_driver, xxx_page):
    assert xxx_page.rename_xxx("自动化数据_rename_001")
```

用例层规则：

- 测试函数名称必须表达模块、页面、操作和条件。
- 每条用例尽量验证一个主要目标。
- 使用 fixture 获取页面对象，不在测试函数内手动初始化驱动。
- 用 `assert page.public_method()` 表达业务验证结果。
- 有数据变更的用例必须考虑清理或使用专用测试数据。
- P0、模块、回归范围用 pytest marker 管理。
- 多平台不一致时，用 marker、跳过规则或 PageObject 分支处理，不在用例中复制两套流程。
- 如果 `implementation-plan.md` 的 pytest 编排包含私有方法或底层对象调用，不得照抄；必须先补充或复用 PageObject 公共业务方法，再让 pytest 调用公共方法。

pytest 用例层禁止出现：

- PageObject 私有方法调用，例如 `page._xxx()`。
- `base_action`、`baseutil`、`android_poco`、`ios_poco`、`poco(`。
- `IOSElements`、`AndroidElements`、图片文件名、Airtest `Template`、坐标、复杂滚动逻辑。
- `_handle_platform_popup*` 调用。
- 未经封装的 locator 字符串。

错误示例：

```python
def test_tag_rename_leading_space(tinglan_tag_page):
    assert tinglan_tag_page._click_tag_rename_btn("测试标签A")
    assert tinglan_tag_page._input_tag_name("  工作")
```

正确示例：

```python
def test_tag_rename_leading_space(tinglan_tag_page):
    assert tinglan_tag_page.rename_tag_and_expect_normalized(
        old_name="测试标签A",
        raw_input="  工作",
        expected_name="工作",
    )
```

如果某条用例还缺 UI 树、测试数据、文案、正式图片或用户确认，默认不生成该用例；如 `implementation-plan.md` 明确要求保留占位，则必须在 pytest 中 `pytest.skip("缺少前置条件：...")`。

## 六、UI 树和页面映射规则

AI 编写脚本前，必须使用 `autotest-summary-case` 已建立好的业务页面到 UI 树关系，不得在本阶段重新做元素映射。

映射表职责：

- 统一业务页面名。
- 指向对应 UI 树、截图和 prompt。
- 指向对应 PageObject 文件和类名。
- 区分不同平台物料。
- 作为生成期物料，不作为最终运行依赖。

使用规则：

- 文本用例中的页面名称必须已在 `element-mapping.md` / `project-structure.md` 中完成映射。
- 若资料包显示页面缺 UI 树或截图，停止生成相关用例，不猜 Poco locator。
- 同一业务页面的不同平台 UI 树分开维护。
- 页面改版后，先更新 UI 树和映射表，再更新 PageObject。
- 输入框、文本域、可编辑控件必须以 `element-mapping.md` 记录的 UI 树真实 `name` / `type` / `text` / `value` 为准，不得根据“输入框”语义脑补为 `UITextField`、`EditText` 或其他控件类型。
- placeholder 文案如果在 UI 树中是 `StaticText`，只能作为提示文案或辅助校验，不能当作输入控件 locator。
- `element-mapping.md` 必须区分标签文案、placeholder、真实输入控件、字符数提示；例如真实节点是 `TextView` 时，不能写成 `UITextField`。

## 七、图片兜底规则

Airtest+Poco 工程采用“元素定位优先 + 图片识别兜底”。AI 写脚本时必须尊重开发期施工目录和正式图片目录的边界。

| 目录 | 定位 | 规则 |
| --- | --- | --- |
| 开发期施工目录 | 开发期施工区 | 存放扫描页面、候选截图、Poco 结构树、prompt，不是最终运行依赖 |
| 正式图片目录 | 正式图片资产 | 最终脚本运行依赖，新增或修改必须先经用户确认 |

强制规则：

- 优先使用 Poco 稳定定位（name、text、resourceId、sibling 等）。
- 不稳定定位可以接入 Airtest 图片识别兜底。
- 图片兜底链路的步骤，最终交付不得残留 `None` 占位。
- 图标图和文字图必须分开，不能混用。
- 不同平台即使语义相同，也应分别命名图片。
- 图片真实视觉语义优先于扫描器临时命名。
- AI 不得擅自往正式图片目录新增、复制、替换图片。
- 写图片兜底代码前必须先查 `image-review-list.md`，不得绕过清单自行编图片名。该清单应已检查页面 `prompt.md` 弹药库、`tools/images/` 候选池和 `tools/pages/**/crops/` 页面裁图。
- 阶段一生成代码时，如果物料已有候选图，可以按 `image-review-list.md` 使用物料真实文件名，方便用户确认和溯源。
- 物料无图时，可以使用语义化建议正式名作为待补图提示，但必须标注未达交付态。
- 缺正式图片且无稳定 Poco locator 的步骤，不得生成交付态可运行用例。
- 凡是写进 PageObject 的图片文件名，交付/真机运行前必须在正式图片目录真实存在；只有 `crops/` 候选图或只有建议正式图片名时，不得声称可交付或直接运行。

## 八、断言和结果校验规则

断言必须来自 UI 层可观察结果、稳定数据结果或明确日志结果，不能只写业务逻辑推断。

要求：

1. 每条测试用例的每一条预期结果都必须映射为 PageObject 中的一个或多个断言方法。
2. 如果某条预期结果无法自动化，不要省略，必须在 pytest 中 `pytest.skip`，或在 PageObject 中返回明确失败原因，并说明缺少的 UI 树/测试数据/控件属性。
3. 输出前必须给出“预期结果 -> 断言方法”的覆盖表或引用 `implementation-plan.md` 中的覆盖表。
4. 测试函数不得只调用一个宽泛方法，必须能追溯覆盖了哪些预期。
5. 删除、排序、列表字段、跳转类预期必须有独立断言。

删除类和破坏性操作要求：

- 必须覆盖确认、取消、成功和重复操作。
- 成功删除不能只判断当前屏幕不可见；必要时使用列表刷新、返回重进或 Poco 滚屏扫描确认。
- 必须使用专用测试数据，避免误删共享数据。

输入框操作要求：

1. 先确认输入框真的出现。
2. 输入前清空旧值。
3. 输入后反查校验。
4. 输入失败时不要继续点确定。
5. 输入控件 locator 必须来自 `element-mapping.md` 记录的 UI 树真实可编辑节点；不要把 `StaticText` placeholder 当输入框。
6. 不要把 iOS 输入控件默认写成 `UITextField`，也不要把 Android 输入控件默认写成 `EditText`；必须以 `element-mapping.md` 记录的 UI 树真实 `type/name` 为准。

## 九、失败分析和修复规则

脚本失败后，AI 必须先归因，再修复。

排查顺序：

1. 终端错误摘要。
2. Captured log setup / call / teardown。
3. 项目日志。
4. 失败截图。
5. 当前页面 Poco UI 树。
6. 测试数据状态。
7. 需求或文案是否变化。

修复约束：

- 只改当前失败相关的最小必要范围。
- 优先改 PageObject，不批量改测试函数。
- 不借失败修复做无关重构。
- 不覆盖用户已有改动。
- 不擅自修改正式图片资产。

## 十、AI 输出脚本前自检清单

AI 提交脚本或脚本方案前，必须逐项自检。

| 检查项 | 通过标准 |
| --- | --- |
| 需求来源 | 脚本能追溯到需求功能点或文本用例 |
| 用例清晰 | 每条 pytest 用例有明确标题、前置、动作和断言 |
| 数据明确 | 测试数据可准备、可复用或可清理 |
| 页面映射 | 涉及页面能在映射表或工程模块中找到 |
| 分层正确 | 业务动作在 PageObject 目录，流程编排在测试用例目录 |
| 测试层干净 | pytest 不直接调用私有方法、Poco、base_action、baseutil、图片名或坐标 |
| 定位可靠 | 优先使用稳定 Poco locator，不稳定步骤有图片兜底策略 |
| 图片合规 | 不擅自新增或修改正式图片目录，兜底链路不残留 `None` |
| 图片清单可追溯 | 代码中的图片名能在 `image-review-list.md` 追溯到候选图路径或正式图片 |
| 输入控件真实 | 输入框 locator 与 `element-mapping.md` 记录的 UI 树来源一致，未把 placeholder 或猜测 class 当作输入控件 |
| Poco API 合规 | 未使用 `get_attribute()` 等非本项目 Poco API |
| 平台明确 | 多平台差异已处理或明确跳过 |
| 失败可诊断 | 日志、返回值、断言信息能支持定位问题 |
| 回归可维护 | 用例可用 marker 管理，可纳入模块回归 |

pytest 测试文件必须静态扫描，命中以下任一模式都视为生成失败，必须重写测试层或补充 PageObject 公共方法后重新生成：

- `page._` 或 `._` 私有方法调用。
- `.base_action`、`.baseutil`、`android_poco`、`ios_poco`、`poco(`。
- `IOSElements`、`AndroidElements`。
- `Template(`、图片文件名、坐标型点击或滑动编排。
- `_handle_platform_popup`。

PageObject 文件也必须静态扫描，命中以下任一模式都视为生成失败，必须重写 PageObject 后重新生成：

- `get_attribute(`。
- 生成或修改代码中出现 `UITextField`、`EditText`、`TextView` 等输入控件 locator，但 `element-mapping.md` 没有对应真实节点来源。
- 代码中出现 `*.png` 图片名，但既不在 `image-review-list.md`，也不能追溯到正式图片目录或候选物料。
- 用户要求“可直接运行/交付”时，代码中出现的 `*.png` 在正式图片目录不存在。
- `_handle_platform_popup*` 链路传入的图片文件名没有出现在 `image-review-list.md`，或状态不是 `已在images` / `物料待确认` / `缺物料需补图` 中的一种。

基础校验：

1. 对新增或修改的 Python 文件运行 `python -m py_compile`。
2. 对相关测试文件运行 `pytest --collect-only`。
3. 有设备和明确要求时，再运行单条 smoke。
4. 如果无法运行校验，必须说明原因。

## 框架排雷与防爆指南

以下排雷规则适用于所有 Airtest+Poco 工程，写脚本时必须遵守。

### Flutter 混合页面的画布输入假象

- Flutter 页面中直接调用 `poco().set_text()` 可能修改底层属性但不触发 UI 重绘。
- 调用底层 `input_text(element, content)` 时，`element` 必须传入字符串定位符，禁止传入 `UIObjectProxy`。
- 如果 Android 输入框只能靠下标定位，无法使用 `input_text`，先点击获取焦点，再用 Airtest 原生 `text("内容")` 输入。
- 输入框 locator 必须来自 `element-mapping.md` 记录的当前页面 UI 树真实可编辑节点。不要把 `StaticText` placeholder 当输入框，也不要把 iOS 输入控件默认写成 `UITextField`；若资料包记录真实节点为 `TextView`，应按 `TextView` 或项目可用封装处理。
- Poco 的 `UIObjectProxy` 不是 Selenium 元素，禁止调用 `get_attribute()`；读取属性前必须确认项目 API，优先使用 Poco `attr(...)` 或通过页面状态变化做断言。

### Poco 链式寻址崩溃与正则风险

- 避免 `poco(name="xxx")[0].parent()` 等容易触发 AST 解析崩溃的链式写法。
- 默认不用正则；只有项目已有约定且控件带动态后缀时，才使用 `textMatches` / `nameMatches`。
- 如果前置搜索已经过滤到唯一目标，优先直接点击视野内目标。

### iOS Flutter 的 Phantom Click

- 列表刷新或滚屏后，等待像素稳定。
- 边缘元素点击优先使用 `.focus([0.5, 0.5]).click()`。

### 智能滚屏雷达的小步快跑原则

- 循环滚屏探测必须采用小步幅滑动。
- 必须配套循环上限，循环内先 `exists()` 判定，再执行 `swipe`。

### Poco 对象缓存残影清理机制

- 页面发生路由跳转、返回、刷新后，必须重新实例化 Poco 节点。

## 十一、交付标准

一批 AI 生成的自动化脚本只有同时满足下面条件，才算可以进入回归候选。

```text
需求有来源
用例有评审
页面有映射
数据有准备
动作在 PageObject
断言在 UI 层可观察
pytest 只编排流程
图片兜底合规
语法校验通过
pytest 收集通过
失败能被归因
```

最终稳定后，再为用例增加合适 marker，纳入模块回归、P0 回归或全量回归集合。
