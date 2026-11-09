# Review：审定记录导入与版本绑定

- 模块 ID：review
- 文档状态：Confirmed
- 文档内容修订：r001
- 概述状态：Confirmed / r001
- 详设覆盖：Complete
- 声明范围：C2 所需的既有审定记录导入、校验读取及版本绑定。
- 完整人工队列、任务分派、审定 UI 和仲裁工作流：后续范围。

## 1. 职责与边界

依据：
- [总体架构 r001](../../00-system-architecture.md)。
- [目录设计 r001](../../01-directory-layout.md)。
- [横切规则 r001](../../02-cross-cutting-concerns.md)。
- [Artifact Store 契约 r002](../artifact-store/README.md#contracts)。
- [v2 设计](../../../doc/RAG测评设计_v2.md)，第 11、15、16 节。

本模块负责：
- 校验并保存已有审定记录及原始导入文件。
- 将记录绑定到明确的对象版本、内容视图和 rubric。
- 保留审定方法、声明的审定人、导入主体及时间。
- 检查消费者当前对象与记录适用对象是否一致。

本模块不负责：
- 判断参考答案、充分证据或人工评分在业务上是否正确。
- 将 approved 自动转换成 human_verified 或指标满分。
- 自动选择最后一条记录、少数服从多数或消除审定分歧。
- 修改数据集、Run、EvidenceMapping 或 Judgment。
- 调用模型生成审定结果。

业务对象所有者解释并应用审定结果。
Review 使用通用对象引用，不反向导入其他业务模块。

## Contracts

契约状态：Confirmed。
契约修订：r001。
格式版本：2.0。

### C1. 审定对象与规则引用

ReviewTarget 包含：

| 字段 | 含义 |
|---|---|
| owner | 对象所属模块 ID |
| object_type | 所有者定义的对象类型 |
| object_id | 逻辑对象身份 |
| object_revision | 明确修订，不接受 latest |
| review_view_version | 所有者定义的审定视图格式版本 |
| review_view_ref | 固定审定视图的 ArtifactRef |

owner 在 C2 中允许：
dataset-registry、adapters、evidence-mapper、evaluators。

review_view_ref 指向当时实际被审查的内容视图。
视图由对象所有者定义，必须覆盖本次判断依赖的内容及版本。

为避免循环引用，视图排除指向本次审定记录的链接，
以及由本次审定推导出的状态。
排除字段必须在所有者契约中明确，不能由导入器自行忽略。

Review 校验该视图字节和引用。
对象身份、revision 与视图是否正确对应，
仍由对象所有者核对。

ReviewRubricRef 包含：
- rubric_id。
- rubric_revision。
- content_ref：冻结 rubric 的 ArtifactRef。

rubric 的文字、检查项或解释规则变化，
均需新的 revision 或内容身份。
相同 rubric 名称不代表规则相同。

### C2. ReviewRecord

| 字段 | 约束 |
|---|---|
| schema_version | 固定为 2.0 |
| review_id | 非空逻辑审定 ID |
| review_revision | 明确的不可变记录修订 |
| target | ReviewTarget |
| rubric | ReviewRubricRef |
| method | human、automated 或 unknown |
| reviewer_id | 声明的审定主体 ID |
| performed_at | 声明的审定时间，带时区，导出为 UTC |
| decision | approved、revise、reject、needs_adjudication |
| checks | 非空、check_id 唯一的逐项结果 |
| reason | 简短结论说明 |
| assessment_ref | 可选，业务标签或评分明细的 ArtifactRef |

每个 check 包含：
- check_id：rubric 中的检查项身份。
- result：pass、fail、unknown 或 not_applicable。
- reason：简短依据；非 pass 结果必须说明原因。
- evidence_refs：用于该判断的零个或多个 ArtifactRef。

approved 要求 checks 中至少一项 pass，
且不包含 fail 或 unknown。
not_applicable 项必须有理由。
批准范围仅限已列出的检查项。

其他 decision 保留其逐项结果，
不会自动映射为某个任务分数。

ReviewRecord 不保存隐藏思维链或凭据。
涉及受限内容的详细说明放入授权产物，
记录正文只保留必要短说明与引用。

记录身份为：
(access_domain, review_id, review_revision)。

同身份、不同内容不得在同一审定集合中并存。
修改记录需新的 review_revision，
原记录和原导入批次保持不变。

assessment_ref 的业务结构由使用方定义并校验。
Review 只验证其存在性、访问域和完整性，
不能将未知结构当作有效评分。

method 与 reviewer_id 是来源声明，不是身份认证证明。
人工声明被导入，不意味着已经证实人工独立审定。

### C3. 导入策略

ReviewImportPolicy 通过不可变 ArtifactRef 提供，包含：
- schema_version：2.0。
- policy_id、policy_revision。
- allowed_target_owners：非空，取值限于 C1 的所有者。
- allowed_rubrics：非空的完整 ReviewRubricRef 列表。
- allowed_methods：允许的 human、automated、unknown 子集。
- allowed_reviewer_ids：可选；为空时不限制声明的审定人 ID。
- max_records：正整数，默认 10000。
- max_record_bytes：正整数，默认 1048576。

策略控制可接收范围，不认证声明的审定人身份。
允许 automated 或 unknown 记录，不授予人工审定等级。

同批次原始文件、策略、对象视图、rubric、
证据和 assessment 均须位于同一 Store 和 access_domain。

跨域材料需要先经过单独授权的导入，
不能通过审定文件中的引用扩大访问权限。

### C4. 导入批次及来源回执

ReviewBatchRef 包含：
- schema_version：2.0。
- batch_id。
- manifest_ref：冻结批次 manifest 的 ArtifactRef。

manifest 包含：
- batch_id。
- source_ref：原始 JSONL 文件。
- records_ref：规范化后的 ReviewRecord JSONL。
- record_count。
- import_policy_ref。
- importer_version。

batch_id 是 manifest 除 batch_id 外其余字段
规范序列化后的 SHA-256。
manifest_ref 本身不参与该摘要计算。

相同原始文件、策略和导入器版本产生相同批次身份。
改变文件、策略或导入器版本形成新批次。
每条记录的逻辑身份与批次身份分别解释。

ReviewImportResult 包含：
- batch_ref：ReviewBatchRef。
- receipt_ref：本次导入回执的 ArtifactRef。

回执记录：
- 唯一 import_operation_id。
- batch_ref。
- received_by：从可信 StoreAccessContext 取得的导入主体。
- received_at：本地导入 UTC 时间。
- access_policy_revision。
- identity_assurance：本版固定为 local_import_only。

重复导入可以复用同一批次，
但每次导入生成独立回执。
本地 received_at 不覆盖声明的 performed_at。

本版不提供签名核验或不可伪造的多人签核保证。

### C5. 公开操作

```text
import_records(
    source_ref,
    import_policy_ref,
    access_context
) -> ReviewImportResult

load_records(
    import_result,
    access_context
) -> 只读 ReviewRecord 迭代器

check_binding(
    record,
    expected_target,
    expected_rubric
) -> ReviewBindingCheck
```

import_records：
- 读取原始文件及策略前执行授权和完整性检查。
- 输入是 UTF-8 JSONL，每行一个 ReviewRecord。
- 拒绝 BOM、重复 JSON 键、空记录、未知字段和不支持的版本。
- 换行符允许 LF 或 CRLF；文件末尾换行不计为空记录。
- 按字节限制读取单行，避免先读取无限长行再检查大小。
- 空批次、超过限额及批次内重复记录身份均拒绝。
- 校验所有引用、同域约束和策略允许范围。
- 全部记录通过后，才提交批次 manifest。
- 不跳过坏行后将剩余记录伪装为完整导入。

load_records：
- 核验回执与批次引用相符。
- 核验 manifest、records_ref、数量及记录格式。
- 读取仍检查当前权限。
- 对同一已验证批次可重复读取，不修改原记录。
- 读取过程中发现错误时明确失败，
  调用方不能把已读子集当作完整批次提交。

check_binding 是纯检查，不读取 Store 或授予访问权。
expected_target 和 expected_rubric 必须由业务对象所有者
根据当前真实对象构造，不能直接复制待检查记录中的值。

ReviewBindingCheck 包含：
- status：matched、target_mismatch、target_changed、
  rubric_mismatch。
- mismatch_fields：不包含敏感内容的字段名列表。

检查顺序：
1. owner、object_type、object_id 不同：target_mismatch。
2. object_revision、review_view_version 或完整内容引用不同：
   target_changed。
3. rubric 身份、revision 或内容引用不同：rubric_mismatch。
4. 全部相同：matched。

matched 只表示版本绑定一致，
不表示审定正确、检查项完整、身份可信或门禁通过。

## 2. 导入、持久化与恢复

### 2.1 存储安排

所有原始文件、策略、规范化记录、manifest 和回执
通过 Artifact Store 保存，不建立另一套内容寻址存储。

Review 不操作 Store 的私有目录或锁。
不在业务模块间共享可变的全局记录表。

workspace/reviews 下可按需导出受控工作包，
导出文件不是新的权威记录。
读取与复现使用固定 ArtifactRef。

### 2.2 提交流程

```text
验证授权、源文件和策略
→ 逐行校验记录及引用
→ 生成完整规范化 records 产物
→ 保存 records_ref
→ 计算批次身份并保存 manifest
→ 保存本次导入回执
→ 返回 ReviewImportResult
```

任何一步失败都不返回成功的 ReviewImportResult。

已保存 records 但 manifest 未提交时，
这些字节不构成有效审定批次。

manifest 已提交但回执未保存时，
可以保留已知 batch_ref 用于排查；
调用方不得宣称本次导入已经成功完成。

恢复通过正常 import_records 重试：
- 相同输入重新得到相同批次身份。
- 已提交字节由 Store 校验并复用。
- 生成新的本次导入回执。
- 不覆盖原记录或原回执。

本轮没有外部调用，不存在远端重复执行问题。
Store 故障仍保留其原始 reason_code 和操作关联。

### 2.3 多批次与分歧

同一消费集合中：
- 相同记录身份和内容可以去重。
- 相同记录身份但内容不同属于记录冲突。
- 不同身份对同一对象作出不同判断，属于审定分歧。

前者是输入合法性问题，后者不能按输入损坏处理。

消费者必须保留所使用的批次、记录 revision 和筛选规则。
已知存在分歧时，不能通过文件顺序或最后写入自动选定结果。

C2 不自动执行多数投票或仲裁。
需要选择或修正时，使用明确的新记录与消费配置，
并披露已知分歧及处理依据。

## 3. 结果如何进入数据与评分

Dataset Registry 根据自己的发布 profile 核对：
- 对象及其内容视图。
- rubric 和必需审定项目。
- 方法、来源及实际审定范围。
- 分歧和不确定项。

单个 approved 记录不能直接提升整题或整批的质量等级。
抽检通过不能使未审定题目变为 human_verified。

人工评分通过 Evaluator 所有的导入契约转换成 Judgment。
转换由 runner 协调，并保留原 Review 引用与映射版本。

Evaluator 无需反向导入 Review 的实现或类型；
runner 调用双方公开接口，构造 Evaluator 定义的输入。
具体评分字段和转换规则在 Evaluator 设计中确认。

缺少有效记录、绑定不一致或判断分歧时，
不能用其他对象、旧 revision 或默认满分补齐。

Review 记录不传入普通被测系统调用。

## 4. 错误与安全

公开错误采用横切规则，并保留具体记录行号或安全身份：

| reason_code | 含义 |
|---|---|
| review.invalid_input | 格式、字段或参数不合法 |
| review.unsupported_schema | 不支持的记录或策略版本 |
| review.policy_rejected | 方法、rubric、所有者或声明主体不在允许范围 |
| review.record_limit_exceeded | 批次或单条记录超过配置限额 |
| review.duplicate_record | 批次内记录身份重复 |
| review.reference_invalid | 引用、回执或批次关系不一致 |
| review.access_denied | 当前操作或引用访问被拒绝 |
| review.storage_failure | 存储失败，保留 Artifact Store 原始原因 |
| review.integrity_mismatch | 批次身份、计数或内容校验不一致 |

版本绑定不一致通过 ReviewBindingCheck 返回，
不抛成基础设施异常。

审定分歧保留为业务事实，
不自动转换成错误或假造仲裁结果。

外部审定文本作为数据，不执行其中的代码、模板或指令。
默认日志不输出原始记录、对象视图或详细审定内容。

## 5. 验证计划

以下为计划验收，尚未执行：

| 场景 | 应观察到的结果 |
|---|---|
| 导入合法单条审定记录 | 返回固定批次和独立回执 |
| 同输入重复导入 | batch_id 一致，回执身份不同 |
| 批次中间一行非法 | 整批失败，无成功导入结果 |
| 相同记录身份重复出现 | duplicate_record |
| 同 ID、不同 revision | 作为不同记录保留 |
| 改对象答案但沿用旧审定 | target_changed |
| rubric 同名但内容变化 | rubric_mismatch |
| 引用无权限的材料 | 拒绝，不泄露材料内容 |
| approved 包含 fail 或 unknown | 拒绝矛盾记录 |
| automated 记录声明 approved | 保留 automated，不提升人工等级 |
| 两位审定人判断相反 | 两条记录保留，不自动取最后一条 |
| 记录只审查一个项目 | 不能代表其他项目通过 |
| records 保存后进程中断 | 重试复用字节，不产生有效半批次 |
| manifest 保存后回执失败 | 明确未完成，重试可形成新回执 |
| 修改批次内容、数量或回执关联 | 校验失败 |
| 读取半批后发生损坏 | 不提交仅含已读子集的消费结果 |

## 6. 依赖、代码与进度

唯一业务依赖：
[Artifact Store / Contracts r002](../artifact-store/README.md#contracts)。

使用的契约和操作：
ArtifactRef、StoreAccessContext、put_blob、open_verified。
不导入 Store 私有实现。

计划入口：
- rag_eval.review.contracts。
- rag_eval.review.api。

内部职责可分为记录验证、批次导入和版本绑定检查。
私有文件拆分由实现决定。

测试使用真实本地 Store、小型 JSONL 和可控故障点。
无需模型服务、生产数据、数据库或审定 UI。

| 设计单元 | 审阅状态 | 内容修订 |
|---|---|---|
| 概述与边界 | Confirmed | r001 |
| 公开契约 | Confirmed | r001 |
| 导入、持久化与恢复 | Confirmed | r001 |
| 消费边界 | Confirmed | r001 |
| 错误与安全 | Confirmed | r001 |
| 验证与依赖 | Confirmed | r001 |

Complete 仅覆盖本文件声明的 C2 最小审定能力。
应用实现、测试及完整人工审定工作流尚未完成。

公开接口、所有权、权限、持久化和验收行为的变化需确认；
私有实现可在已确认边界内决定。
