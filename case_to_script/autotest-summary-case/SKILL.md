---
name: autotest-summary-case
description: 根据 autotest-check-case 的文本层用例检查结果、用户确认、页面物料和工程上下文，完成自动化落地可行性分析，并构建供自动化脚本生成使用的上下文交接资料包。当用户提到"构建自动化上下文""整理脚本生成上下文""交接到生成脚本""自动化上下文资料包""自动化落地分析""automation context build""script context handoff"时触发。不是测试用例文本规范检查，也不是生成脚本。
---

# 自动化落地分析与脚本生成上下文交接资料包 — 执行协议

## 职责边界

本 Skill 负责自动化落地可行性分析，并整理脚本生成交接资料包：

- 输入：测试用例 Excel 或提取的用例文本、页面映射表（tools/pages/page_ui_tree_mapping.yaml）、页面 UI 树/截图物料（tools/pages）、工程上下文、`autotest-check-case` 输出的文本层检查结果、用户确认/补充/假设授权。
- 输出：供 `autotest-generate-script` 读取的上下文交接资料包。
- 不重新做测试用例文本规范审查；文本问题沿用 `autotest-check-case` 的结论和用户确认。
- 必须做工程落地可行性分析：页面映射、UI 物料、元素定位、断言可观测性、测试数据、正式图片/候选图、PageObject/pytest 分层。
- 不生成脚本代码。
- 不替用户新增确认结论。

## 固定输出目录

资料包必须输出到当前工作目录下的固定目录：

`testcase-workflow/script-context/`

该目录由本 Skill 管理，目录内必须包含标记文件：

`.managed-by-autotest-summary-case`

每次执行本 Skill 时，必须先清空 `testcase-workflow/script-context/` 再重新生成资料包。清空前必须确认目录内存在 `.managed-by-autotest-summary-case`；如果目录不存在，直接创建；如果目录存在但没有该标记文件，不要清空，先向用户说明并等待确认。

## 核心原则

### 只继承确认，不补文本结论

本 Skill 不替用户新增第一阶段的文本确认结论。用户在问题确认阶段没有回答的问题，保持「待确认」状态，不在资料包中自行推断答案。

### 只分析和交接，不生成

本 Skill 输出自动化落地分析结果和交接资料包，不输出任何脚本代码。脚本生成由 `/autotest-generate-script` 完成。

### 缺图和未确认问题要进入范围判断

凡是影响元素定位、断言、测试数据、平台差异、正式图片兜底的问题，都必须影响用例分类。尤其是 `_handle_platform_popup*` 链路：

- 缺正式图片且没有稳定 Poco locator 的步骤，不得列入「立即自动化」。
- 有稳定 Poco locator 但缺正式图片兜底的步骤，可以列入「立即自动化」，但必须在 `handoff-summary.md`、`element-mapping.md` 和 `image-review-list.md` 标记风险、阶段一引用名、候选图路径和待补图状态。

## 工作流程（5 步，必须按顺序执行）

### 步骤 1：收集输入资料

收集以下材料。除工程规则文件外，工程材料必须按本次候选用例涉及的页面、模块、方法、图片前缀收敛读取，避免无关全量扫描：

- 测试用例：用户提供 Excel 文件路径或从文件读取的用例内容。如果已从 Excel 提取，直接使用提取结果。
- 页面映射表：读取映射表文件，并只抽取本次候选用例涉及的业务页面记录。
- 页面 UI 树/截图物料：从页面物料目录读取相关页面的 UI 树、截图、prompt 和必要 crops 信息（列出路径，不重复复制）。
- 工程上下文：先读取工程根目录 `CLAUDE.md`（如存在），再按相关模块读取 PageObject 目录结构、已有公共方法/相似实现、相关 pytest 文件、正式图片目录中相关图片清单；不得把无关代码和全量图片清单塞入资料包。
- 文本层检查结果和问题清单：从 `testcase-workflow/script-clarify/` 读取 `input-inventory.md`、`automation-readiness.md` 和 `clarify-issues.md`。
- 用户确认：用户在对话中对问题的确认、补充说明、假设授权。

如果 `testcase-workflow/script-clarify/` 不存在，提示用户先执行 `/autotest-check-case`。

如果 `automation-readiness.md` 存在，必须把其中的「文本层自动化候选初判」作为范围判断输入，但不得把它当作最终脚本生成范围：

| 文本层初判 | 本 Skill 的处理 |
| --- | --- |
| 可进入自动化落地分析 | 可以继续做页面物料和工程可行性分析 |
| 需修订/确认后进入 | 只有相关问题已确认、已修订或用户授权假设后，才可进入落地分析；否则保持未进入立即自动化 |
| 建议保留手工或专项测试 | 没有用户新确认或用例修订前，不得提升为立即自动化 |
| 信息不足，无法判断 | 没有补充用例内容前，不得提升为立即自动化 |

兼容旧产物：如果旧版 `automation-readiness.md` 仍使用「自动化准备初判」且出现「补充材料后继续」或「不建议自动化」，在没有新材料、新确认或明确假设授权前，不得直接提升为「立即自动化」。

### 步骤 2：整理用户确认结论

将用户确认内容按问题逐项归并，区分三类：

| 类别 | 说明 | 处理 |
| --- | --- | --- |
| 已确认 | 用户明确给出答案的问题 | 写入确认结论 |
| 按假设继续 | 用户同意先按某个假设进入落地分析或脚本生成的问题 | 写明假设内容和假设条件 |
| 仍待确认 | 用户未回答的问题 | 继续标注待确认；若影响脚本行为，生成阶段必须 skip 或不生成 |

不要替用户新增确认结论。

### 步骤 3：自动化可行性判断

根据文本层初判、用户确认、页面物料和工程上下文，对每条测试用例得出以下四类结论之一。不得重新审查测试用例文本规范，但必须把第一阶段未确认的文本问题计入范围判断。

| 结论 | 条件 | 后续动作 |
| --- | --- | --- |
| 立即自动化 | 文本层无未解决阻塞；页面映射存在；关键 UI 物料存在；断言可观测；数据稳定；元素可定位；缺图问题不影响主路径或已有稳定 locator | 进入元素映射和实现计划 |
| 补充材料后自动化 | 缺 UI 树、截图、页面映射、测试数据、断言文案、正式图片或用户确认，且缺失会影响脚本稳定运行 | 写入 `automation-scope.md` 的待补充清单 |
| 暂不自动化 | 文本层建议保留手工且未被用户改判；需求不稳定；收益低；维护成本高；断言不可观察 | 不进入脚本生成，保留为手工用例 |
| 需要研发支持 | 缺稳定控件标识、状态构造接口、测试数据接口或硬件控制能力 | 提出可测性诉求 |

### 步骤 4：生成上下文交接资料包

使用固定目录 `testcase-workflow/script-context/`。

执行前按「固定输出目录」规则清空该目录。

创建 `.managed-by-autotest-summary-case` 标记文件。

必须生成以下文件；即使某类内容为空，也要生成对应文件并写明“无”或“不适用”：

- `handoff-summary.md`
- `project-structure.md`
- `automation-scope.md`
- `element-mapping.md`
- `image-review-list.md`
- `implementation-plan.md`
- `user-confirmations.md`

将最终资料放入以下文件：

#### handoff-summary.md

交接入口文件，保持简洁，包含：

- 本次平台和范围。
- 已确认、按假设、仍待确认的问题数量和关键结论。
- 可立即自动化、需补充材料后自动化、暂不自动化、需要研发支持的用例数量。
- 生成脚本时必须注意的风险。
- 缺正式图片清单摘要。
- 图片人工确认清单摘要：说明哪些图片来自物料、哪些缺物料、哪些已在正式图片目录。
- 下游读取顺序。

#### project-structure.md

项目关键路径和工程约定：

| 路径类型 | 实际路径 |
| --- | --- |
| 页面物料目录 | [探测到的路径] |
| 页面映射表文件 | [探测到的路径] |
| PageObject 目录 | [探测到的路径] |
| 测试用例目录 | [探测到的路径] |
| 开发期施工目录 | [探测到的路径] |
| 正式图片目录 | [探测到的路径] |
| 工程规则文件 | [CLAUDE.md 路径；不存在则写无] |

还必须记录本项目的关键编码约定，例如 fixture 获取方式、平台变量、图片目录规则、PageObject 风格。若工程根目录存在 `CLAUDE.md`，必须把其中与脚本生成有关的规则摘要写入本文件，至少覆盖：

- 图片三阶段工作流、正式图片目录、`tools/` 施工区和 `images/` 交付红线。
- PageObject 文件组织、公共/私有方法风格、返回值和异常处理约定。
- `_handle_platform_popup*` 调用规则和图片兜底约束。
- Airtest+Poco 输入、点击、属性读取、滚动、节点缓存等框架排雷规则。

如果 `CLAUDE.md` 与本资料包其他文件冲突，必须在 `handoff-summary.md` 标记冲突；下游生成时以用户最新确认和 `CLAUDE.md` 的工程规则优先。

#### automation-scope.md

合并原 `testcases-to-automate.md` 和 `testcases-pending.md`，包含两类表：

可立即自动化的用例清单：

| 用例ID | 用例名称 | 涉及页面 | 关键操作 | 可行性结论 | 优先级 | 数据/清理要求 |
| --- | --- | --- | --- | --- | --- | --- |

未进入立即自动化的用例清单：

| 用例ID | 用例名称 | 结论 | 缺失材料/原因 | 优先级 | 后续动作 |
| --- | --- | --- | --- | --- | --- |

#### element-mapping.md

页面元素映射表。对每条「立即自动化」的用例：

| 用例ID | 页面名 | 业务元素名 | 定位方式 | locator/图片名 | 正式图片状态 | 来源 |
| --- | --- | --- | --- | --- | --- | --- |

定位方式优先 Poco 稳定定位（name、resourceId、text、sibling）。不稳定定位标注需要图片兜底，并写明物料图片、阶段一代码引用名、建议正式图片命名和当前正式图片是否已存在。图片细节必须同步写入 `image-review-list.md`，不要只在 `element-mapping.md` 里用一句“候选有”概括。

输入框和可编辑控件必须单独校验：

- 必须从 UI 树真实节点读取 `name` / `type` / `text` / `value`，不得根据“输入框”语义脑补为 `UITextField`、`EditText` 或其他控件类型。
- 如果 UI 树中 placeholder（例如“请输入xxx”）是 `StaticText`，只能把它记录为提示文案，不能当作输入控件 locator。
- 如果真实可编辑节点是 `TextView`，`element-mapping.md` 必须写 `TextView`，不能写 `UITextField`。
- `locator/图片名` 列必须能区分“标签文案/placeholder/真实输入控件/字符数提示”，不能把多个元素合并成一个泛化输入框。
- 需要校验按钮可用态、输入框内容、属性值时，必须在 `implementation-plan.md` 标注项目支持的 Poco API 或现有封装；禁止写 Selenium 风格 API（例如 `get_attribute()`）。

#### image-review-list.md

图片人工确认清单。只列脚本实际会引用或 `_handle_platform_popup*` 链路可能用到的图片，不列全部 crops。如果本次没有任何脚本引用图片或 popup 图片链路，也必须生成本文件，并写明“本次无脚本引用图片”。

| 业务元素 | 使用场景 | popup链路 | 阶段一代码引用名 | 候选图路径 | prompt来源 | 建议正式图片名 | images中是否已存在 | 是否必须人工确认 | 当前状态 |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |

生成规则：

- 生成本清单前，必须先检查扫描器物料来源，不能直接编图片名。检查顺序：
  1. 页面 `prompt.md` 弹药库中列出的图片及语义说明。
  2. `tools/images/` 候选池。
  3. `tools/pages/**/crops/` 页面裁图。
- 如果扫描器物料中已有候选图，`阶段一代码引用名` 必须优先使用物料真实文件名，例如 `ios_icon_TL_tag_more.png`，方便用户从代码反查候选图。
- 如果物料没有图，`阶段一代码引用名` 才使用语义化建议正式名，例如 `ios_tinglan_tag_more_icon.png`，表示需要用户手动补图。
- `候选图路径` 必须写完整相对路径，例如 `tools/pages/.../crops/ios_icon_TL_tag_more.png`；不要让用户去 crops 目录盲找。
- `建议正式图片名` 只用于后续 promotion / 沉淀阶段，不等于阶段一代码引用名。
- `images中是否已存在` 必须检查正式图片目录，写 `是/否`。
- `当前状态` 使用：`物料待确认`、`缺物料需补图`、`已在images`、`已确认待promotion`、`无需图片`。
- 如果某步骤完全依赖稳定 Poco locator，且生成代码不会引用图片，可以不列；如果 `_handle_platform_popup*` 会传图片参数，必须列。

#### implementation-plan.md

合并原 `pageobject-methods.md` 和 `assertion-design.md`，包含：

PageObject 方法设计清单：

| 页面名 | PageObject 文件 | 类名 | 新增/复用方法名 | 方法职责 | 公共/私有 | 复用已有方法 |
| --- | --- | --- | --- | --- | --- | --- |

预期结果 -> 断言方法覆盖表：

| 用例ID | 预期结果 | 断言方法名 | 断言逻辑 | 无法自动化时处理 |
| --- | --- | --- | --- | --- |

pytest 编排计划：

| 用例ID | pytest 函数名 | 调用的 PageObject 公共方法 | marker | skip 条件 |
| --- | --- | --- | --- | --- |

`调用的 PageObject 公共方法` 列只能填写已有公共方法或本资料包中明确要求新增的公共方法。禁止在该列或 pytest 编排说明中出现：

- PageObject 私有方法（例如 `_enter_xxx`、`_click_xxx`、`_input_xxx`）。
- `base_action`、`baseutil`、`poco`、`android_poco`、`ios_poco`。
- `IOSElements` / `AndroidElements`、locator 字符串、图片文件名、坐标、Airtest `Template`。

如果某条用例需要特殊动作（例如输入带首位空格后保存、清空已有输入、校验输入框归一化），必须先在 PageObject 方法设计清单中新增公共业务方法，再让 pytest 编排计划只调用该公共方法。例如：新增 `rename_tag_and_expect_normalized(old_name, raw_input, expected_name)`，pytest 只写 `assert page.rename_tag_and_expect_normalized(...)`。

生成 `implementation-plan.md` 后必须自检 pytest 编排计划；如果发现 `_xxx` 私有方法、底层对象、locator、图片名或坐标，必须重写资料包，不得把违规编排交给 `autotest-generate-script`。

生成 `element-mapping.md` 后还必须自检输入控件映射：凡是写入 `UITextField`、`EditText`、`TextView`、placeholder 文案或 class/type 定位的元素，必须能在对应 UI 树中找到同名或同类型真实节点；找不到时必须改为「补充材料后自动化」或写明待确认，不得把猜测 locator 交给 `autotest-generate-script`。

生成 `image-review-list.md` 后必须自检图片映射：凡是 `element-mapping.md` 或 `implementation-plan.md` 提到图片兜底、图片候选、建议正式名的业务元素，都必须在 `image-review-list.md` 有一行可追溯记录；缺少候选图路径时不得写“候选有”。

#### user-confirmations.md

用户对脚本生成前问题的确认、补充、假设授权。按问题逐项列出：

| 问题ID | 原问题 | 用户确认结论 | 类别（已确认/按假设/仍待确认） | 对脚本生成的影响 |
| --- | --- | --- | --- | --- |

### 步骤 5：输出自检和交接说明

输出前必须自检；任一项不通过，都必须重写资料包后再交付：

- `script-context/` 中存在 `.managed-by-autotest-summary-case`。
- `handoff-summary.md`、`project-structure.md`、`automation-scope.md`、`element-mapping.md`、`image-review-list.md`、`implementation-plan.md`、`user-confirmations.md` 都已生成。
- `automation-scope.md` 没有把第一阶段仍为「需修订/确认后进入」「建议保留手工或专项测试」「信息不足，无法判断」且未获新确认/修订的用例列为「立即自动化」。
- 每条「立即自动化」用例都有元素映射和实现计划；无法映射的用例必须降级为「补充材料后自动化」「暂不自动化」或「需要研发支持」。
- `element-mapping.md` 或 `implementation-plan.md` 中凡是出现图片兜底、候选图、建议正式名、阶段一引用名，`image-review-list.md` 必须有对应可追溯记录。
- `implementation-plan.md` 的 pytest 编排计划不直接出现 PageObject 私有方法、底层 Poco/base_action/baseutil、locator、图片名或坐标。
- `user-confirmations.md` 覆盖所有进入范围判断的待确认/假设问题，未确认项必须说明对脚本生成的影响。

- 输出资料包路径：`testcase-workflow/script-context/`。
- 说明 `/autotest-generate-script` 下一步应读取该资料包。
- 提醒用户：如果还有未确认的阻塞问题，建议先确认后再生成脚本。

## 与其他 Skill 的衔接

上游：

- `/autotest-check-case` 输出的 `input-inventory.md`、`automation-readiness.md` 和 `clarify-issues.md` 是本 Skill 的必读输入。
- 用户在对话中对问题的确认是本 Skill 的关键输入。

下游：

- 本 Skill 输出的资料包（`testcase-workflow/script-context/`）供 `/autotest-generate-script` 读取。
- 下游 Skill 不再直接读取 `testcase-workflow/script-clarify/`。

完整链路：

```text
autotest-check-prd -> autotest-summary-prd -> autotest-generate-cases
-> autotest-check-case -> autotest-summary-case -> autotest-generate-script
```
