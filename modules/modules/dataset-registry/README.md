# Dataset Registry：数据、证据与版本注册

- 模块 ID：dataset-registry
- 文档状态：Confirmed
- 文档内容修订：r004
- 修订边界：本次仅进度摘要更新；业务单元及公开契约保持各自既有修订。
- 概述状态：Confirmed / r001
- 详设覆盖：Complete
- 声明范围：C2 所需的现有数据接收、版本固定与评价读取。
- 当前确认：核心契约、规范化导入、审定衔接、版本发布与恢复。
- 自动数据生成、审定 UI、远端后端及资产删除：后续范围。

## 1. 职责与边界

依据：
- [总体架构 r001](../../00-system-architecture.md)。
- [目录设计 r001](../../01-directory-layout.md)。
- [横切规则 r001](../../02-cross-cutting-concerns.md)。
- [Artifact Store 契约 r002](../artifact-store/README.md#contracts)。
- [Review 契约 r001](../review/README.md#contracts)。
- [v2 设计](../../../doc/RAG测评设计_v2.md)，第 4—6、17 节。

本模块拥有：
- CorpusSnapshot、EvidenceUnit、TestCase 和 DatasetVersion。
- 业务身份、revision、引用一致性和数据集发布规则。
- 数据内容与审定结果的版本绑定。
- 评价读取及普通系统输入投影。

本模块不生成问题、不运行被测系统、不执行模型裁判。
原始字节与 SourceArtifact 由 Artifact Store 管理。
审定记录由 Review 管理。
指标如何计分由 Evaluator 定义。

C2 支持先导入现有问题、参考和可用证据。
缺少某类标注不会自动补造，也不必阻止其他指标运行。
接收合法、内容可信和适合正式门禁分别判断。

## Contracts

契约状态：Confirmed。
契约修订：r002。
范围：C1—C13；核心对象、数据集发布、读取与生命周期协议。

### C1. 身份与格式

规范记录使用：
- schema_version：2.0。
- contract_profile：dataset-registry.c2.v1。

明确 profile 用于区分本契约与 v2 中的说明性展开示例。
已有文件通过显式映射配置导入；
不能仅凭 schema_version 相同就假设结构一致。

通用 ObjectKey 包含：
- object_id。
- revision。

CorpusSnapshotKey、EvidenceKey、CaseKey 是该结构的具名类型，
不能因为字段相同而相互替代。

revision 是不透明的非空字符串，不接受 latest。
同一对象身份和 revision 对应唯一规范内容。
内容变化必须使用新 revision，不能原地覆盖。

对象 ID 与 revision 在同一 Registry 访问域内唯一解释。
完整发布引用还需绑定具体数据集及其内容摘要，
不能仅按裸 case_id 查找“当前题目”。

缺失引用与有意不提供标注分别处理：
- 已声明引用但目标不存在：引用错误。
- 字段按契约允许为空：记录缺失，影响相应评价能力。
- 不支持的结构：明确拒绝或在导入阶段显式转换。

### C2. CorpusSnapshot

CorpusSnapshot 包含：

| 字段 | 约束 |
|---|---|
| snapshot_key | CorpusSnapshotKey |
| documents | 非空 DocumentRevision 列表 |
| access_policy_ref | 冻结访问策略的 ArtifactRef |
| effective_time | 可选，快照适用的业务时间说明 |

DocumentRevision 包含：
- document_id、document_revision。
- family_id：可选，已知文档族。
- original_source：原始文件的 SourceArtifactKey。
- representations：零个或多个参考表示。
- valid_from、valid_to：可选业务有效日期或带时区时间。

每个参考表示包含：
- representation_id、representation_revision。
- kind：canonical_text 或 external。
- source_artifact：对应 SourceArtifactKey。
- producer_id、producer_version。
- independent_reference：是否声明为独立参考表示。

canonical_text 必须能以严格 UTF-8 解码。
该表示及其固定 revision 用于字符位置和 quote 核验。

external 用于保留 PDF 页面、表格、图像等已有资产。
C2 不因此承诺完成对应模态的定位和评分。

original_source 与参考表示必须能够回到明确的源文版本。
不能把待测 parser 的可变输出默认为独立黄金参考。

independent_reference 是来源声明，
不单独证明参考内容正确或人工审定完成。

同一快照内不得出现相同 document_id、
document_revision 对应不同内容的情况。
本 profile 每个 document_id 选择一个明确 revision；
跨历史版本比较使用不同快照。

未掌握语料快照的外部问答可以先导入，
但其 TestCase 必须声明 corpus_binding=not_recorded，
不能伪造空快照或宣称全库可答性已经核验。

### C3. EvidenceUnit

EvidenceUnit 包含：

| 字段 | 约束 |
|---|---|
| evidence_key | EvidenceKey |
| snapshot_key | 明确的 CorpusSnapshotKey |
| document_id、document_revision | 必须存在于该快照 |
| fact_id | 该快照内的语义事实身份 |
| statement | 完整事实说明，包含必要限定 |
| qualifiers | 本节规定的事实限定 |
| modality | text、table、chart、image 或 other |
| locator | 本节规定的定位结构 |

qualifiers 包含：
- entity_ids：零个或多个明确实体 ID。
- period：可选，原业务期间字符串。
- conditions：条件说明列表。
- assertion_kind：actual、plan、forecast、definition 或 other。
- polarity：asserted 或 negated。
- quantity：可选，value 与 unit；value 为有限十进制字符串。

不能因数值相同而合并不同实体、期间、单位、
计划/实际、条件或否定关系。

同一 snapshot 内共享 fact_id 的证据，
必须具有相同的规范 statement 与 qualifiers。
不同来源可以支持同一事实，但需要明确确认身份等价。
自动聚类不能直接改写已发布 fact_id。

事实身份用于覆盖去重，不是来源可信度证明。

C2 接收两种 locator：

text_quote：
- kind：text_quote。
- representation_id、representation_revision。
- quote：非空原始摘录。
- prefix、suffix：可选消歧文本。
- section_path：可选章节路径。
- char_span：可选的 start、end。

char_span 使用固定参考文本的 Unicode code point，
采用半开区间 [start,end)。
提供时必须合法，并与 quote 对应。

没有 char_span 时，quote 仍需在指定参考表示中可核验。
多次出现的 quote 保留消歧范围；
无法唯一定位不能自动当作确定匹配。

external：
- kind：external。
- descriptor_format、descriptor_version。
- descriptor_ref：已有定位描述的 ArtifactRef。

external 用于保存尚未标准化的外部定位信息。
C2 文本 Mapper 对其返回不支持或不可计算，
不能从 opaque 描述推断 bbox、页面或表格精度。

源文 modality 与运行时实际输入通道分别记录。
图表源文被转换成文字，不表示生成器实际看到图像。

source_reviewed 等核验结论通过数据集审定绑定表达，
不由 EvidenceUnit 的存在、quote 命中或模型一致性推导。

### C4. TestCase

TestCase 包含：

| 字段 | 约束 |
|---|---|
| case_key | CaseKey |
| input | 本节规定的用户输入 |
| corpus_binding | recorded 或 not_recorded |
| snapshot_key | recorded 时必填，否则为 null |
| tags | 去重后的标签列表 |
| reference | 可为 null；提供时遵守 C5、C6 |

input 包含：
- query：非空字符串。
- history：按原顺序保存的消息列表。
- as_of：可选业务日期或带时区时间。
- access_scope：tenant_id、roles、policy_version。

C2 history 支持 user、assistant 两种角色及文本 content。
system 配置属于被测 SystemConfig。
工具消息和多模态历史需要后续明确协议，不能静默转成普通文本。

access_scope 是被测用户的业务权限条件，
不授予测评进程读取 Store 的权限。

reference 可以为空。
仅有 query 的用例可以参与运行导入、格式和计量评价，
不能因此产生正确性、完整性或黄金证据覆盖分数。

普通系统输入投影只允许：
query、history、as_of 和 access_scope。
reference、证据、审定、split 和标签不进入该投影。
具体 Adapter 如何传递 as_of 由接入协议明确。

split、split_group、质量等级及审定集合属于
DatasetVersion 的成员与标注信息，
不与 TestCase 核心内容混成一个可变对象。

v2 展开示例中的这些字段在导入时规范化到发布信息。
更改 split 或审定绑定必须产生新数据集版本，
不能改变旧实验使用的成员关系。

### C5. Reference、主张与充分组

Reference 包含：
- answerability：status 与 reason。
- expected_action：允许为空。
- reference_answers：零个或多个参考表达。
- claims：零个或多个 ReferenceClaim。
- required_answer_claim_ids。
- sufficient_evidence_sets。
- numeric_rules。
- task_policy：可为 null。

answerability.status：
answerable、unanswerable、ambiguous、conflicting、unknown。

expected_action：
answer、abstain、clarify、tool_required、policy_block 或 null。

可答性与预期行为分开。
unknown 不自动变成 unanswerable；
无权限也不等于语料中不存在答案。

answerability 与 reference_answers 是待评价基线的声明，
可信程度由对应审定记录和发布 profile 决定。

ReferenceClaim 包含：
- claim_id：本用例内唯一。
- text：主张内容。
- evidence_refs：零个或多个 EvidenceKey。
- derived_from：零个或多个本用例 claim_id。
- verification_rule_ids：零个或多个本用例规则 ID。

主张推导关系必须无环。
derived_from 非空时必须给出允许的验证规则。
同一主张不同时声明直接证据路径和派生路径；
可替代支持由独立充分组表达。

没有证据的参考主张允许接收，
但不能获得相应的证据支持认证。

required_answer_claim_ids 只列出回答必须包含的答点。
计算前提不因参与推导就自动成为答案必须复述的内容。

SufficientEvidenceSet 包含：
- set_id：本用例内唯一。
- evidence_refs：非空、无重复的 EvidenceKey 列表。
- allowed_rule_ids：允许使用的本用例规则 ID。

组内按 fact_id 聚合：
- 不同必要事实之间为 AND。
- 同一事实的已认可来源之间为 OR。
- 不同充分组之间为 OR。

该结构表示声明的充分路径，不自动证明充分性。
正式“已审定充分”结论来自对应审定绑定。

reference_answers 只提供可接受表达示例，
不能将字符串相似度直接解释成任务正确。

task_policy 提供：
- rubric：可选 ReviewRubricRef，固定任务要求正文。
- required_dimensions：业务要求的评价维度 ID。
- allowed_tools：允许工具 ID。
- allowed_numeric_operations：允许的数值操作。
- require_supported_citations：是否要求支持性引用。

维度 ID 不引入对 Evaluator 实现的反向依赖。
实际指标与维度的映射由评价 profile 定义。

缺少 task_policy 或 rubric 时，
不能声称已按完整业务要求判定 TaskSuccess。

### C6. 类型化数值规则

C2 的 NumericRule 仅支持 scalar_decimal_v1：
identity、add、subtract。

其他运算通过后续规则 profile 扩展，
不能执行模型生成的任意表达式或代码。

每条规则包含：
- rule_id。
- profile：scalar_decimal_v1。
- inputs：非空操作数列表。
- operation。
- operands：有序 operand_id 列表。
- output_claim_id。
- expected_value：十进制字符串。
- unit。
- absolute_tolerance、relative_tolerance：非负十进制字符串。

每个 input 包含：
- operand_id：规则内唯一。
- value：有限十进制字符串。
- unit。
- claim_id：可选，本用例输入主张。
- evidence_ref：可选，直接输入来源。

identity 要求一个操作数；
subtract 要求两个并保留顺序；
add 要求至少两个。

本 profile 的输入和输出使用相同单位。
单位转换必须在输入阶段有明确依据，
不隐式换算货币、万元、亿元或百分比。

校验规则时重新计算 expected_value，
本 profile 不实施隐式舍入。
预期结果与计算不一致时不能作为合法数值规则发布。

需要派生主张时，input.claim_id 与 derived_from 闭合。
直接来源存在 quantity 时，值、单位及必要限定必须一致。

没有输入证据的数值参考可以接收，
但保留参考来源限制，不能据此认证源文正确或证据充分。

判定数值的容差语义为：

abs(predicted - expected)
<= max(absolute_tolerance, relative_tolerance * abs(expected))

expected 为零时仍需有效绝对容差。

从系统答案中提取哪个数值、怎样核对输出单位，
由具体 Evaluator profile 定义。
不能在一段多数字回答中找到任意匹配数字就判正确。

### C7. 标注范围、审定视图与可信度

数据集发布信息必须分别记录：
- 来源：imported、curated、synthetic 或 online。
- 质量等级：human_core、synthetic_extension 或 unverified。
- 证据相关度标注范围：none、partial 或 complete。
- 实际审查的 snapshot、document 和 evidence 范围。
- 使用的 Review 批次、记录 revision 和发布 profile。

原始输入自报的等级保留为来源声明，
不能直接成为有效 human_verified 状态。

标注范围 complete 必须说明评价单元及核验范围。
它不等于穷举所有问法，也不由冻结或题目数量推导。

新发现但未标注的来源为 unjudged。
不能仅因不在已知 EvidenceKey 列表就判无关。

Registry 负责构造以下 ReviewTarget：
- owner：dataset-registry。
- object_type：corpus_snapshot、evidence_unit 或 test_case。
- object_id、object_revision：对应核心对象身份。
- review_view_version：registry-review-view-v1。
- review_view_ref：固定审定视图字节。

视图包含：
- 对象完整核心内容。
- 所引用的快照、证据、源资产和规则的固定依赖。
- 影响判断的权限、时间与任务条件。

视图排除：
- 指向本次审定记录的链接。
- 根据该审定推导的质量等级、核验状态和发布状态。
- 接收时间等不影响被审内容的操作记录。

对象、依赖或规则变化时重新构造视图，
旧审定不能通过复制其 review_view_ref 沿用。

构造视图由 Registry 负责；
Review 只验证绑定，不裁定内容正确。

发布 profile 对必需检查项、方法、来源、
分歧和覆盖范围的要求见 C10。
未满足要求时，不自动提升质量等级。

### C8. DatasetVersion 与成员关系

DatasetVersionKey 包含：
- dataset_id。
- version_id：明确的非空版本，不接受 latest。

DatasetVersionRef 包含：
- registry_id。
- store_id、access_domain。
- dataset_key：DatasetVersionKey。
- dataset_digest。
- manifest_ref：冻结清单的 ArtifactRef。

完整引用必须与 Registry 的正式发布记录一致。
只有 manifest 字节存在，不代表已经完成发布。

DatasetVersion 清单包含：
- schema_version：2.0。
- contract_profile：dataset-registry.c2.v1。
- dataset_key。
- registry_id、store_id、access_domain。
- snapshots_ref：CorpusSnapshot JSONL。
- evidence_units_ref：EvidenceUnit JSONL。
- test_cases_ref：TestCase JSONL。
- members_ref：DatasetMember JSONL。
- import_manifest_ref。
- release_request_ref。
- release_policy_ref。
- validation_report_ref。
- review_selection_ref。
- 对象数量、split 数量和有效质量等级数量。
- dataset_digest。

dataset_digest 是清单排除自身 dataset_digest 后，
按 rag-eval-json-v1 规范序列化所得字节的 SHA-256。
manifest_ref 不写入自身清单。

所有输入、规则、rubric、审定和标注依赖均使用固定引用。
操作时间及执行主体保存在独立回执，
不进入数据内容摘要。

DatasetMember 包含：
- case_key。
- split：train、dev、test 或 unspecified。
- split_group：非空分组 ID。
- origin：imported、curated、synthetic 或 online。
- declared_quality_tier：可为空，保留来源声明。
- requested_quality_tier。
- effective_quality_tier。
- review_bindings：实际采用的批次及记录 revision。
- annotation_scope：本节规定的相关度标注范围。
- reviewed_scope：实际审查的快照、文档及证据。
- provenance_ref：来源、生成及分组依据的冻结记录。
- limitations：稳定的限制代码列表。

每个已发布用例恰好对应一个成员。
同一版本不能同时包含一个 case_id 的多个 revision。
成员不得引用清单外的用例。

effective_quality_tier 由发布校验计算，
不直接接受外部输入。
任何请求等级不满足条件时，发布检查失败；
不静默降级后返回“发布成功”。

TestCase、EvidenceUnit 和 CorpusSnapshot 的核心内容
与成员元数据分别保存。
调整 split、等级、审定集合或标注范围形成新数据集版本。

annotation_scope 包含：
- level：none、partial 或 complete。
- unit_kind：C2 固定为 evidence_unit。
- universe：本用例明确声明的有限 EvidenceKey 集合。
- judgments：已标注单元及 relevant、irrelevant 标签。
- provenance_ref：标注方法与来源。
- reviewed_scope：已实际审查的单元。

universe 和 judgments 中的证据必须属于用例绑定快照。
同一证据不能出现相反标签。

none 要求 judgments 为空。
partial 表示声明范围内仍有未标注单元。
complete 要求 universe 非空且其中每个单元均有标签。

complete 仅表示该有限单元集合的标注覆盖，
不表示标签正确，也不代表全库或所有运行时 chunk 已穷举。
universe 外的新来源仍为 unjudged。

相关度标签、充分组成员和来源核验分别表达。
relevant 不自动等于足以支持完整答案；
充分组未列出的来源也不自动成为 irrelevant。

### C9. 导入包、映射与规范内容

DatasetImportBundle 通过不可变 ArtifactRef 提供，包含：
- bundle_format：native_jsonl_v1 或 mapped_jsonl_v1。
- source_files：记录种类与原始 ArtifactRef。
- mapping_profile_ref。
- supplement_ref：可为空，显式补充必需字段及引用。
- provenance_ref。
- import_limits。

规范记录种类为：
corpus_snapshots、evidence_units、test_cases、members。

test_cases 非空。
没有已知快照或证据时，对应集合可以为空，
但必须满足 C2—C5 的缺失语义。

所有原始文件使用严格 UTF-8 JSONL。
拒绝 BOM、重复 JSON 键、非法数值和空记录。
允许 LF、CRLF 和文件末尾换行。
逐行校验，不跳过坏行形成“完整导入”。

import_limits 包含：
- max_total_records：正整数，默认 100000。
- max_record_bytes：正整数，默认 1048576。
- max_nesting_depth：正整数，默认 64。

限制在读取与解析过程中生效。
字节产物同时受 Store 的大小限制约束。

映射配置包含：
- mapping_id、mapping_revision。
- input_format、input_format_version。
- 每类输入的字段映射和必需字段。
- 明确常量、枚举映射及引用查找表。
- 显式丢弃的字段路径及理由。
- 映射器实现版本。

C2 映射只允许：
- 按固定 JSON Pointer 读取字段。
- 字段重命名与明确常量。
- 有限枚举映射。
- 按显式查找表补齐对象 revision 和正式引用。
- 展开配置中明确指定的 evidence_units 数组。

不运行表达式、模板代码、插件脚本或模型。
不自动下载文件，也不按外部输入读取任意本地路径。

旧 v2 展开视图按以下规则规范化：
- 内嵌 EvidenceUnit 提取到唯一证据集合。
- split、质量声明和审定字段移入成员及来源记录。
- 简写 ID 通过查找表解析为完整身份。
- 缺少 statement、qualifiers、参考表示或 rubric 正式引用时，
  要求 supplement_ref 补齐，不能根据问题文本猜测。
- 不能支持的视觉定位可显式保存为 external descriptor，
  同时保留格式与能力限制。
- 原有 approved、source_reviewed、human_verified
  作为来源声明保留，不直接生效。

所有输入字段必须被映射、明确保留为来源声明，
或被配置明确列为丢弃。
未知字段不静默忽略。

同一原生集合内重复身份拒绝。
展开视图中重复出现的证据可以按明确配置去重，
但同身份的规范内容必须完全一致。

规范内容使用 rag-eval-json-v1。
顶层对象集合按完整身份排序后生成 JSONL。
对象内部数组保留契约规定的顺序；
history 和 operands 不重排。
业务文本与十进制字符串不进行隐式改写。

DatasetDraftRef 包含：
- registry_id、store_id、access_domain。
- draft_id。
- manifest_ref。

导入清单固定：
- 原始文件、映射及补充输入。
- 规范化集合与内容摘要。
- 导入器版本。
- 原始与规范记录数量。
- 字段迁移、去重、来源声明和限制报告。

draft_id 由上述清单排除自身 draft_id 后的规范内容计算。
接收主体和时间放入独立导入回执。

同一固定输入重复导入得到相同 draft_id。
导入成功不表示已发布或达到某种质量等级。

### C10. 发布策略与审定适用范围

ReleaseRequest 通过固定 ArtifactRef 提供，包含：
- dataset_key。
- draft_ref。
- requested_members：成员及其请求等级。
- split_policy_ref。
- release_policy_ref。
- review_selection_ref。
- change_description。
- predecessor_ref：可为空，只表示版本来源。

版本来源关系不自动将前版标为 superseded。

ReleasePolicy 固定：
- policy_id、policy_revision。
- 支持的等级 profile。
- 每种等级使用的完整 ReviewRubricRef。
- 允许的审定方法及声明主体。
- 必需检查项与条件。
- 所需审定数量。
- 标注、来源及分歧处理要求。

C2 提供以下等级 profile：

| profile | 有效等级 | 必要条件 |
|---|---|---|
| basic_import_v1 | unverified | 核心结构、引用、规则和发布一致性通过 |
| human_core_v1 | human_core | 本节规定的逐题人工审定完整通过 |
| synthetic_extension_v1 | synthetic_extension | 固定生成来源及规定自动验证记录完整通过 |

策略可以提高要求，不能删除 profile 的最低条件。

basic_import_v1：
- 允许缺少参考、证据或审定。
- 保留缺失和已知标签分歧。
- 不声明人工核验、证据充分或正式门禁资格。

human_core_v1：
- reference、snapshot、expected_action 和 task_policy.rubric 完整。
- answerability 不能为 unknown。
- 回答型用例具有明确必需答点。
- answerable 且预期作答时，至少一个充分组完成核验，
  并覆盖必需答点的直接或合法派生支持。
- 采用 method=human、版本绑定一致的审定记录。
- 每题完成必需项目，不能用批次抽检替代。
- 来源核验回到固定原始资产，
  不只审查待测 parser 输出或生成答案。

人工核心必需 check_id：
- source_correctness。
- locator_correctness。
- fact_qualifiers。
- query_validity。
- answerability。
- expected_action。
- required_claims。
- reference_correctness。
- numeric_rules。
- evidence_sufficiency。
- task_policy。
- access_time_conditions。
- relevance_annotations。

numeric_rules 仅在没有数值规则时允许 not_applicable。
relevance_annotations 仅在没有相关度标签时允许 not_applicable。
evidence_sufficiency 对无需作答支持的行为题可以不适用，
但必须由 rubric 规定条件并记录理由。
其他不适用情况需要新的 profile，不能任意绕过。

unanswerable 的审定必须记录授权快照、允许工具及实际核验范围。
不能用检索无命中或局部证据包无答案替代。

synthetic_extension_v1：
- origin=synthetic，生成器、配置和输入版本明确。
- 具有固定的验证 rubric 与逐题验证记录。
- 必需检查 candidate_consistency、answerability_basis、
  reference_consistency、evidence_consistency。
- 有数值规则时，numeric_rules 必须通过。
- 保留自动方法、模型及共模偏差限制。
- 不据此产生 human_verified 或人工 source_reviewed。
- 不要求 C2 自行运行生成器或新模型裁判；
  缺少已有验证记录时不能晋级。

ReviewSelection 固定：
- 完整 ReviewImportResult 列表。
- 拟采用的 batch、review_id、review_revision。
- 已知相关记录及排除理由。
- 明确的分歧解决记录。

Registry 从所提供批次中检查目标的相关记录，
不能只读取请求中选择的一条 approved。
该检查范围限于已声明批次，
不宣称已发现所有外部审定意见。

同一记录身份对应不同内容属于输入冲突。
同目标不同意见属于审定分歧。

已知存在适用的 reject、revise、needs_adjudication，
或必需检查相互矛盾时，阻止质量晋级。
不能通过文件顺序、最后写入或多数票自动解决。

分歧解决需要新的、明确绑定当前视图的人工审定记录，
在 assessment 中列出所解决的原记录及处理依据。
原始记录和分歧历史继续保留。

用于等级提升的 assessment_ref
采用 registry-review-assessment-v1，包含：
- scope_by_check：每项实际核验的对象、主张、规则、
  充分组及相关度标注单元。
- original_source_keys：实际检查的原始资产。
- examined_snapshot_scope：实际核验范围。
- resolves_record_keys：明确解决的已有记录。
- limitations。

Registry 验证 assessment 的格式、引用和覆盖。
只列出 check_id 而没有所需范围，
不能证明整题或全部充分组已审查。

source_reviewed、已审定充分组、
已审定必需答点及数值规则分别计算并保存。
human_core 不自动使所有未审查替代组获得充分认证。

method 和 reviewer_id 仍保留 Review 的来源声明语义。
C2 的身份保证为 local_import_only，
不宣称具有签名认证或不可伪造的多人独立签核。

### C11. split 与分组规则

SplitPolicy 固定：
- policy_id、policy_revision。
- 允许的 split。
- 分组维度和适用的切分协议。
- 明确的用例关联与来源分组依据。
- 去重方法、版本及检查结果。
- 尚未核验的维度。

同一 split_group 的所有成员必须处于同一 split。
已声明的同源变体、会话、对照和重复关系不得跨 split。

按文档族留出时，
共享已知文档族或跨文档依赖的用例需要合并关联组，
不能分别独立分配。

C2 检查已有明确关系，
不承诺完成语义近重复识别或实体解析。
未检查的维度必须保留 unknown。

缺少分组依据时，可以由映射配置显式将相关输入
放入同一个保守分组，并记录依据不足。
不能自动给每题独立分组，制造独立样本假象。

split=unspecified 可以基础发布，
不能据此宣称已形成独立测试集。

发布校验不自动移动用例、重分组或删题。
冲突通过修订成员配置和新的发布请求解决。
冻结版本不原地修改。

### C12. 审定视图与公开操作

registry-review-view-v1 使用规范 JSON，包含：
- 目标对象完整核心内容及内容引用。
- 固定依赖清单及其内容引用。
- 影响判断的权限、业务时间与任务条件。
- 本次请求审查的来源及相关度标注上下文。

依赖清单覆盖目标所引用的快照、文档、
参考表示、证据、主张规则和任务 rubric。
审定界面使用这些固定引用读取实际内容。

审定视图排除：
- 本次 ReviewRecord 及由其推导的状态。
- 有效质量等级和发布状态。
- 导入、发布主体及操作时间。
- 不参与内容审定的 split 分配。

原始质量声明可在来源记录中保留，
不能作为审定通过条件。

build_review_target 由真实对象和依赖重新构造视图，
不接受调用方直接指定一个旧 review_view_ref。
同一内容和审查上下文重复构造得到相同视图。
相关度标签或审查上下文改变时，视图也必须变化。

公开构造入口：

```text
open_local_registry(
    config,
    artifact_store,
    review_api,
    policy_provider
) -> DatasetRegistry
```

config 包含：
- root：本地 Registry 根目录。
- mode：open_existing 或 create_new，默认 open_existing。
- lock_timeout_ms：正整数，默认 5000。

root 解析为固定绝对路径。
Registry 绑定一个固定 Store。
policy_provider 使用 Artifact Store 已确认的
check(access_context, access_domain, operation) 语义。

公开业务操作：

```text
import_dataset(
    bundle_ref,
    access_context
) -> DatasetImportResult

build_review_target(
    draft_ref,
    object_type,
    object_key,
    annotation_context_ref,
    access_context
) -> ReviewTarget

prepare_release(
    draft_ref,
    release_request_ref,
    access_context
) -> ReleasePreparation

publish(
    preparation_ref,
    access_context
) -> DatasetPublishResult

open_dataset(
    version_ref,
    mode,
    access_context
) -> DatasetReader

project_system_input(
    test_case
) -> CaseInput

export_dataset(
    version_ref,
    access_context
) -> DatasetExportResult

record_lifecycle(
    version_ref,
    event_request_ref,
    access_context
) -> LifecycleEventRef
```

DatasetImportResult 返回 draft_ref 和独立 receipt_ref。

ReleasePreparation 返回：
- preparation_ref。
- status：ready 或 blocked。
- validation_report_ref。
- blockers 与 limitations 的安全摘要。

输入、引用或基础设施错误按 C13 失败。
合法输入但不满足请求等级或切分策略时，
返回 blocked，不生成成功发布结果。

准备产物固定 draft、请求、校验报告及候选清单。
ready 不代表已发布。
publish 只接受 ready 的固定准备产物，
并重新检查权限、引用、身份冲突及必要发布条件。

DatasetPublishResult 返回 version_ref 和 receipt_ref。
重复发布相同内容可以复用 version_ref，
每次调用保留独立操作回执。

open_dataset 的 mode 为 evaluation 或 audit。
evaluation 只读取正式发布且未撤回的版本。
audit 可读取已撤回版本的仍被授权内容，
同时返回撤回原因及复现限制。

DatasetReader 提供：
- manifest。
- lifecycle。
- members。
- 按明确 CaseKey、EvidenceKey、CorpusSnapshotKey 读取对象。
- 每个成员的标注状态、审定范围与限制。

不提供 latest 或裸 case_id 查找。
未知对象返回明确的引用错误。
Reader 不计算指标分数或正式门禁资格。

project_system_input 是纯投影：
只返回 query、history、as_of、access_scope。
不会返回 reference、tags、split、审定或发布信息。
Runner 负责与 Adapter 输入契约衔接；
Adapter 不因此新增对 Registry 的反向依赖。

export_dataset 返回：
- export_manifest_ref。
- manifest、test_cases、evidence_units、
  corpus_snapshots、members 及验证摘要的固定文件引用。

导出只来源于正式版本。
导出文件不成为可独立编辑的第二套权威数据。
展开参考视图必须校验与规范对象一致。

record_lifecycle 支持：
- superseded：显式指定同 dataset_id 的新正式版本。
- withdrawn：记录撤回原因和影响范围。

同一固定事件请求重复调用幂等。
生命周期记录不修改数据集内容摘要，
不执行 Store 删除或权限撤销。

### C13. 错误契约

公开错误遵守横切规则，包含安全的对象身份、
输入行号、阶段和原始提供方原因。

| reason_code | 含义 |
|---|---|
| dataset-registry.invalid_input | 格式、字段或操作参数非法 |
| dataset-registry.unsupported_schema | 不支持的 schema、映射或 profile |
| dataset-registry.limit_exceeded | 记录、行大小或解析深度超限 |
| dataset-registry.mapping_incomplete | 必需字段、引用映射或显式处置缺失 |
| dataset-registry.duplicate_object | 输入集合内不允许的重复身份 |
| dataset-registry.reference_invalid | 引用悬空、跨快照或关联不一致 |
| dataset-registry.revision_conflict | 已登记身份对应不同规范内容 |
| dataset-registry.review_binding_invalid | 拟采用审定与当前目标或 rubric 不符 |
| dataset-registry.not_published | 请求对象没有正式发布记录 |
| dataset-registry.withdrawn | 版本已撤回，不允许用于普通评价 |
| dataset-registry.access_denied | 当前读取或写入未获授权 |
| dataset-registry.integrity_mismatch | 内容、计数、摘要或正式记录不一致 |
| dataset-registry.write_conflict | 未取得本地提交协调权 |
| dataset-registry.storage_failure | 持久化失败，保留 Store 原始原因 |

发布检查报告中的阻塞项使用稳定代码：
- insufficient_review。
- review_disagreement。
- tier_requirements_not_met。
- split_conflict。
- grouping_scope_unknown。
- annotation_scope_invalid。

unknown 分组维度是否阻止发布由 SplitPolicy 明确。
引用损坏、权限错误不能被改写成业务检查不通过，
也不能转换为空数据、无证据或零分。

## 2. 核心校验规则

本轮确认以下不变量：
- 核心身份唯一，声明的 revision 与内容一致。
- 已声明的快照、源文、证据、主张和规则引用不得悬空。
- EvidenceUnit 与 TestCase 的快照关系一致。
- 同 fact_id 的事实限定一致，重复来源不增加必要事实数。
- 主张与规则依赖闭合且无环。
- text_quote 绑定固定参考表示，不绑定待测 chunk。
- 不把检索无命中变成 unanswerable 标签。
- 缺少证据、参考或审定时保留相应能力限制。
- 审定结果与冻结核心内容分别保存，避免自引用。
- 普通系统输入中不包含参考和标注信息。

完整发布检查见 C8—C13，提交与恢复流程见第 3 节。

## 3. 导入、版本发布与恢复

状态：Confirmed。
内容修订：r001。
范围：C2 本地文件后端的导入、审定衔接、
发布、生命周期记录、读取和恢复。

### 3.1 存储与权威记录

所有原始输入、核心对象、审定视图、
配置、清单、报告和回执通过 Artifact Store 保存。

Registry 只管理自身的身份、对象登记、
发布提交记录和生命周期记录：

```text
<root>/
├── registry.json
├── registry.lock
├── domains/<domain_digest>/
│   ├── objects/<typed_key_digest>.json
│   ├── drafts/<draft_id>.json
│   ├── releases/<dataset_key_digest>.json
│   └── lifecycle/<dataset_key_digest>/<event_sequence>.json
├── staging/<internal_operation_uuid>/
└── operations/<internal_operation_uuid>.json
```

registry.json 固定：
registry_id、store_id、storage_format_version=1、
canonicalization_version=rag-eval-json-v1。

create_new 不覆盖已有 Registry。
open_existing 不自动初始化或重建身份。

路径键由访问域、对象种类和完整身份的摘要生成。
外部 ID 不直接拼接为路径。
读取仍核对完整身份及内容引用。

objects 登记核心对象身份到不可变内容引用的绑定。
drafts 记录已完成导入。
releases 是数据集正式发布的权威可见性记录。
查询索引和缓存不是发布依据。

所有受管路径限制在固定 root 内。
拒绝内部链接、重解析点及跨卷暂存。

### 3.2 导入流程

```text
授权并校验输入包、映射和限额
→ 读取原始 JSONL
→ 显式映射及展开去重
→ 校验核心对象和完整引用
→ 核验参考文本、quote、span 及数值规则
→ 保存规范对象、集合和导入报告
→ 取得 Registry 写锁
→ 检查并登记对象身份
→ 提交完整 draft 记录
→ 保存导入回执
→ 返回 DatasetImportResult
```

完整引用校验包括：
- 对象存在性和类型。
- 同 Store、同 access_domain。
- SourceArtifact 及内容可读取。
- 快照、文档、证据和用例的一致性。
- 主张、规则、充分组依赖闭合且无环。
- 同 fact_id 的 statement 与 qualifiers 一致。

text_quote 必须在固定表示中存在。
提供 char_span 时须与该范围字节解码后的文本一致。
没有唯一定位时保留歧义限制，不自动选择首个命中。
quote 不存在属于非法证据引用，不能作为合法黄金证据登记。

external descriptor 只检查格式声明、引用和完整性，
不据此认证其视觉定位或内容正确。

校验失败不提交有效 draft。
已保存的 Store 字节可以暂时无引用。

核心对象登记逐条不可变提交。
发生中断时，可能留下已合法校验的身份绑定，
但这些对象不构成已完成 draft 或正式数据集。
重试复用相同绑定，不自动删除或改写。

### 3.3 审定衔接与发布准备

1. 从固定 draft 构造 ReviewTarget。
2. Review 导入已有记录并返回固定批次及回执。
3. 发布请求明确列出批次、记录和请求等级。
4. Registry 重新构造期望目标，检查 rubric 与内容绑定。
5. 根据 C10 检查方法、范围、必需项目和已知分歧。
6. 检查 split、分组、标注范围及所有引用。
7. 计算逐题有效等级、核验事实和限制。
8. 保存固定校验报告及准备产物。

不能将输入记录的 target 原样复制为期望目标。

校验报告固定：
- draft 与发布请求引用。
- 使用的策略和校验器版本。
- 所检查的批次及记录。
- 逐题等级、核验范围和限制。
- 阻塞项、缺失项及计数。
- split 和相关度标注检查结果。

报告不包含本次操作时间，
保证相同固定输入产生相同规范结果。
本地执行身份与时间保存在操作回执。

准备不调用模型、不运行被测系统，
也不自动补齐缺失审定。

### 3.4 状态与发布提交

生命周期含义：
- draft：导入操作已经开始，尚未形成完整接收结果。
- validated：核心校验完成，draft 已提交。
- reviewed：请求的审定要求已满足。
- frozen：正式发布记录已经提交。
- superseded：显式记录被新版本替代。
- withdrawn：明确撤回评价使用。

basic_import_v1 可以从 validated 发布为 frozen，
不需要伪造 reviewed。
frozen 不表示人工审定或门禁通过。

操作状态、数据质量与数据集生命周期分别保存。
失败的导入或发布 attempt 不成为数据集状态。

publish 流程：

```text
校验准备产物及 ready 状态
→ 重新授权并检查完整依赖
→ 取得 Registry 写锁
→ 检查对象登记和目标版本
→ 核验候选清单、集合、摘要及计数
→ 保存并核验最终 manifest
→ 提交正式 release 记录
→ 保存发布回执
→ 返回 DatasetPublishResult
```

正式 release 记录包含完整 DatasetVersionRef
及必要格式与记录摘要。
通过同文件系统暂存文件提交，
不得覆盖既有不同内容。

同 DatasetVersionKey：
- 引用及内容一致：验证后幂等复用。
- 内容不同：revision_conflict。

manifest 已保存在 Store，
但 release 记录尚未提交时，不对评价读者可见。

只修改 split、审定集合或 rubric，
也必须使用新 version_id。

### 3.5 并发与中断恢复

同一 root 使用进程内互斥及操作系统文件锁。
初始化、对象登记、draft 提交、发布、
生命周期写入和恢复遵守同一锁协议。

等待不超过 lock_timeout_ms。
不能按 PID、文件年龄或锁文件是否存在判断锁有效性。

固定锁顺序为：
Registry 锁 → Store 公开操作。
Store 和 Review 不回调 Registry，
不取得 Registry 锁。

耗时解析和规范化可在锁外完成；
提交时必须重新检查身份和权限。

恢复在取得 Registry 写锁后检查操作记录：
- 正式记录不存在：判为未提交，保留诊断。
- 正式记录完整且匹配：确认已提交。
- 正式记录损坏或不匹配：完整性故障，停止复用。
- 暂存归属或目标无法核验：保留现场。

不从残留暂存文件自动发布数据集。
通过原公开操作及同一固定输入重试。

若正式发布已经提交、回执保存失败：
- 返回包含提交阶段及安全版本身份的失败信息。
- 不声称目标未发布。
- 重试核验并复用已发布版本，生成新的操作回执。

导入回执失败按同样原则处理。
重试不覆盖原始文件、对象、版本或回执。

本轮仅清理能够确认归属的无效暂存文件。
不进行正式对象垃圾回收。

首个验收环境为 Windows 本地 NTFS。
网络盘、同步盘和断电持久性不在已验证范围内。

### 3.6 正式读取与导出

open_dataset 依次检查：
- 当前权限和 Registry、Store 身份。
- 正式发布记录。
- 完整 DatasetVersionRef。
- manifest、集合摘要、数量及引用关系。
- 当前生命周期和请求模式。

校验通过后才提供 DatasetReader。
读取中再次发生 I/O 或完整性错误时明确失败；
消费者不得把已读子集提交为完整数据集。

superseded 版本仍可显式读取。
withdrawn 版本拒绝普通评价读取，
audit 模式仍须满足当前权限。

导出文件从正式版本生成，
最终 export manifest 固定各文件摘要和来源版本。
中断导出不返回成功导出结果。
文件落地由可信组合入口执行，
不能把数据中的路径字段作为导出目标。

实际权限撤销、资产删除和内容损坏分别返回原因，
不因为版本曾经冻结就绕过当前授权。

### 3.7 更正与生命周期记录

更正答案、事实、证据、规则或其他核心内容：
新对象 revision → 新审定视图 → 必要复核 → 新数据集版本。

只调整成员信息：
复用未变核心对象 → 重新进行发布检查 → 新数据集版本。

superseded 必须显式引用同一 dataset_id 的新正式版本。
withdrawn 必须保存原因和受影响范围。

生命周期事件按版本串行追加，
包含单调序号、前序事件摘要及固定请求引用。
不修改已有事件或 manifest。

withdrawn 在 C2 中不恢复为 frozen。
修复通过新数据集版本重新发布。

Store 删除与访问撤销由相应维护能力负责；
Registry 只保存允许保留的状态和复现限制。

## 4. 验证计划

本节状态：Confirmed / r002。
以下是计划验收，尚未执行：

| 场景 | 应观察到的结果 |
|---|---|
| 只有 query，没有 reference 或快照 | 可以接收，参考与证据指标不可测 |
| 声明证据引用但证据不存在 | 引用错误 |
| answerable 但没有 EvidenceUnit | 不补造证据，不自动拒绝基础接收 |
| 同一事实来自两个源文 | 保留来源，覆盖按事实去重 |
| 相同数字但年份或计划/实际不同 | 不得共用同一事实身份 |
| 改变待测 chunk 粒度 | 黄金证据身份保持不变 |
| 重复 quote 无法消歧 | 保留定位限制 |
| 派生主张缺前提或关系成环 | 校验失败 |
| 计算结果未在原文逐字出现 | 可由合法输入与规则验证 |
| 计算前提未在答案中复述 | 不自动判必需答点缺失 |
| 导入旧版 approved 声明但无绑定记录 | 不自动成为 human_core |
| 改答案、证据或 rubric 后复用旧审定 | 绑定不一致 |
| 新替代证据尚未审查 | unjudged，不自动记无关 |
| 仅修改 split | 新数据集版本，旧版成员关系不变 |

以下为导入与发布的补充计划验收，尚未执行：

| 场景 | 应观察到的结果 |
|---|---|
| 相同输入包重复导入 | draft_id 相同，操作回执分别保留 |
| JSONL 中间一行非法 | 不形成成功 draft，不跳过坏行 |
| v2 展开视图重复引用同一证据 | 一致内容去重，不一致内容拒绝 |
| 映射缺少源文或 rubric revision | mapping_incomplete，不猜测 |
| 原文件自报 human_verified | 仅保留声明，不自动晋级 |
| 人工只核验充分组 S1 | 不自动认证其他充分组 |
| 原始资产未实际审查 | 不能满足核心集来源核验 |
| 自动扩展集完整验证 | 可按 synthetic profile 发布，保持方法限制 |
| 批次中存在相反审定 | 保留分歧，阻止未经解决的晋级 |
| 只在选择列表中排除 reject | 不能绕过已知适用分歧 |
| 修改相关度标注后沿用旧审定 | 视图变化，不能直接复用 |
| complete 只覆盖有限证据集合 | 集合外仍为 unjudged |
| 分组关系跨 dev/test | 按切分策略阻止发布 |
| 缺少分组依据 | 不自动生成逐题独立组 |
| 对象登记中途退出 | 没有有效半个 draft，重试复用合法绑定 |
| manifest 保存后、release 提交前退出 | 不可作为正式版本读取 |
| release 提交后、回执保存前退出 | 重试复用原版本，不覆盖 |
| 两进程提交同版本相同内容 | 成功结果绑定同一正式版本 |
| 两进程提交同版本不同内容 | 一次成功，另一次 revision_conflict |
| 正式集合数量或内容被改动 | integrity_mismatch |
| 版本被 superseded | 旧明确引用仍可读取 |
| 版本被 withdrawn | 普通评价拒绝，授权审计披露撤回 |
| 权限撤销后使用旧版本引用 | access_denied |
| 普通系统输入投影 | 不含参考、标签、split 或审定 |
| 导出中断或读取途中损坏 | 不宣称完整导出或完整消费 |

使用真实本地 Store、小型 JSONL、人工审定样例、
独立进程和可控提交故障点验证。
不需要模型服务、生产语料或数据库。

## 5. 依赖与设计进度

进度摘要：Confirmed / r004（包 r022）；业务单元修订仍按下表记录。

允许的业务依赖：
- [Artifact Store / Contracts r002](../artifact-store/README.md#contracts)。
- [Review / Contracts r001](../review/README.md#contracts)。

本模块提供的数据契约供 Adapter 对齐、Mapper、
Evaluator、Runner 和 Reporting 使用。
消费者只使用公开 contracts 和 api。

计划入口：
- rag_eval.dataset_registry.contracts。
- rag_eval.dataset_registry.api。

内部职责包括对象及引用校验、规范化导入、
审定视图与等级计算、本地发布和只读访问。
私有文件及类的拆分由实现决定。

使用 Artifact Store 的公开写入、核验读取及源资产接口，
以及 Review 的 load_records、check_binding。
不读取提供方私有目录或修改其记录。

本轮无网络请求或新增模型调用。
性能边界由显式记录限额、Store 大小限制及本地文件系统决定；
尚未进行规模或吞吐基准验证。

| 设计单元 | 审阅状态 | 内容修订 | 位置 |
|---|---|---|---|
| 概述与边界 | Confirmed | r001 | [第 1 节](#1-职责与边界) |
| 核心数据与公开操作契约 | Confirmed | r002 | [Contracts](#contracts) |
| 核心校验 | Confirmed | r002 | [第 2 节](#2-核心校验规则) |
| 导入、发布与恢复 | Confirmed | r001 | [第 3 节](#3-导入版本发布与恢复) |
| 契约验证计划 | Confirmed | r002 | [第 4 节](#4-验证计划) |
| 依赖与实现安排 | Confirmed | r002 | [第 5 节](#5-依赖与设计进度) |

Complete 仅覆盖本文件声明的 C2 范围。
应用实现与验收尚未执行。
C2 的当前就绪状态见[索引](../../index.md#5-候选推进范围)，
确切实现基线见[交接 C2-H003](../../handoffs/c2-r003.md)。

公开接口、数据含义、依赖方向、权限、持久化、
状态机和验收行为变化需确认。
私有实现可在已确认边界内决定。
