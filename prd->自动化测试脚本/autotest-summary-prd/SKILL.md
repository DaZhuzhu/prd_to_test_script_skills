---
name: autotest-summary-prd
description: 根据原始需求、需求评审输出和用户确认回复整理测试用例生成前的定稿需求资料包。当用户提到"定稿需求""整理确认后的需求""交接到生成测试用例""测试用例输入资料包""finalize requirements"时触发。不是需求评审，也不是生成测试用例。
---

# 定稿需求资料包 — 执行协议

## 职责边界

本 Skill 只负责整理交接资料包：

- 输入：原始需求文档、需求评审输出、用户确认/补充/假设授权。
- 输出：供 `autotest-generate-cases` 读取的定稿需求资料包。
- 不重新做需求质量审查。
- 不生成测试用例。

## 固定输出目录

资料包必须输出到当前工作目录下的固定目录：

`testcase-workflow/finalized-requirements/`

该目录由本 Skill 管理，目录内必须包含标记文件：

`.managed-by-autotest-summary-prd`

每次执行本 Skill 时，必须先清空 `testcase-workflow/finalized-requirements/` 再重新生成资料包。清空前必须确认目录内存在 `.managed-by-autotest-summary-prd`；如果目录不存在，直接创建；如果目录存在但没有该标记文件，不要清空，先向用户说明并等待确认。

## 工作流程（4步，必须按顺序执行）

1. **收集输入资料**
   - 原始需求文档或提取后的需求文本。
   - `autotest-check-prd` 输出的需求质量问题清单和待确认问题清单。
   - 用户对问题的确认、补充说明、假设授权。

2. **整理用户确认结论**
   - 将用户确认内容按问题逐项归并。
   - 区分已确认、按假设继续、仍待确认但不阻塞的内容。
   - 不要替用户新增确认结论。

3. **生成定稿需求资料包**
   - 使用固定目录 `testcase-workflow/finalized-requirements/`。
   - 执行前按「固定输出目录」规则清空该目录。
   - 创建 `.managed-by-autotest-summary-prd` 标记文件。
   - 将最终资料放入以下文件：
     - `original-requirement.md`
     - `requirement-review.md`
     - `user-confirmations.md`
     - `finalized-requirement.md`
     - `handoff-summary.md`

4. **输出交接说明**
   - 输出资料包路径。
   - 说明 `autotest-generate-cases` 下一步应读取该资料包。

## 文件职责

### original-requirement.md

保存原始需求，或从 PDF/Word 提取后的需求文本。

### requirement-review.md

保存需求质量问题清单和待确认问题清单。

### user-confirmations.md

保存用户对需求评审问题的确认、补充、假设授权。

### finalized-requirement.md

保存用于生成测试用例的定稿需求内容。

### handoff-summary.md

保存交接摘要，包括：

- 已确认问题。
- 按假设继续的问题。
- 仍待确认但不阻塞的问题。
- 生成测试用例时需要注意的风险。
