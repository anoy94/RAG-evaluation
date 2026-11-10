# Runner：离线实验准备与评价执行

- 模块 ID：runner
- 文档状态：Confirmed
- 文档内容修订：r003
- 修订边界：本次仅进度摘要更新；业务单元及公开契约保持各自既有修订。
- 概述状态：Confirmed / r002
- 详设覆盖：Complete
- 已确认范围：C2 实验准备、评价执行、预算与失败控制、重试、缓存及本地恢复。
- 并发范围：每个操作一个协调者、一个在途本地 worker；不同操作按各自身份协调。
- 在线系统调用、Oracle 和高级诊断：后续范围。

## 1. 职责与边界

依据：
- [总体架构 r001](../../00-system-architecture.md)。
- [目录设计 r001](../../01-directory-layout.md)。
- [横切规则 r001](../../02-cross-cutting-concerns.md)。
- [Artifact Store 契约 r002](../artifact-store/README.md#contracts)。
- [Review 契约 r001](../review/README.md#contracts)。
- [Dataset Registry 契约 r002](../dataset-registry/README.md#contracts)。
- [Adapters 契约 r001](../adapters/README.md#contracts)。
- [Evidence Mapper 契约 r001](../evidence-mapper/README.md#contracts)。
- [Evaluators 契约 r001](../evaluators/README.md#contracts)。
- [Metric Planner 契约 r001](../metric-planner/README.md#contracts)。
- [v2 设计](../../../doc/RAG测评设计_v2.md)，第 7、8、12、14 节。

本模块拥有：
- ExperimentRequest 和 ExperimentManifest。
- 数据、来源批次、A/B arm 与预期观测的组合关系。
- 明确的 Run 选择和实际输入关联检查。
- Provider 调用顺序及跨模块输入转换。
- EvaluationPlan 的保存和执行。
- 评价执行身份、任务操作记录及结果集合。
- 根结果槽位的完整性。
- 重试、预算、缓存与本地恢复协议。

本模块不负责：
- 重新定义数据、运行、映射或评分对象。
- 实现指标算法或改写 Judgment。
- 自动修正参考、映射歧义或审定分歧。
- 选择最高分答案或运行。
- 计算统计区间和发布门禁。
- 将操作完成解释成系统质量通过。

本轮只调用已有本地提供方：
- 导入运行和审定文件。
- 读取正式数据版本。
- 进行文本证据映射。
- 转换已有审定材料。
- 执行确定规则和已有判断的离线评分。

不调用被测业务系统、模型、网络服务或外部工具。

Diagnostics 不属于本次必需实现范围。
构造 Runner 时不要求注入其占位实现。

Runner 的公开契约覆盖本文件声明的 C2 本地范围。
完整 C2 实现交接仍需完成 Reporting、CLI、
集成验收、需求映射及交接审阅。

## Contracts

契约状态：Confirmed。
契约修订：r002。
schema_version：2.0。
contract_profile：runner.c2.v1。

本次契约覆盖实验准备、单次与可恢复执行、
有限重试、明确来源的评分复用及本地协调。
不提供分布式调度、多 worker 并行或自动搜索全库评分缓存。

### C1. ExperimentRequest 与实验身份

ExperimentRequest 是不可变产物，包含：
- schema_version、contract_profile。
- experiment_key：非敏感逻辑实验名称。
- dataset_version_ref。
- case_selection。
- arms。
- sample_slots。
- run_sources。
- run_selection。
- evaluator_package_ref。
- metric_requests。
- mapping_preparation。
- assessment_preparation。
- execution_policy。
- reporting_intent。
- runtime_environment_ref。
- provenance_ref。
- derived_from_experiment：可为空。

所有数据、配置、来源及规则使用明确引用。
不根据当前机器状态寻找 latest 或默认数据集。

case_selection 包含：
- 有序、无重复的 CaseKey 列表。
- 所依据的正式数据版本。
- split、group、tier 等选择依据。
- selection_basis_ref。

选择必须在本次评分前固定。
不根据答案是否正确、上下文是否完整或标签是否存在筛选。

本地固定选择不构成预注册证明。
已有历史结果是否被查看过作为来源限制保留，
不宣称已实现事前盲选。

arms 是有序列表，每项包含：
- arm_id。
- 显示名称。
- declared_system_config_ref。
- 允许的来源 namespace。
- 来源及复现限制。

sample_slots 是有序、无重复的预期采样位置。
它不重新定义 Adapter 的 attempt 或 repetition。

planning_scope_key 为：
Store 身份、访问域及 ExperimentRequest 规范内容的摘要。

它不包含尚未生成的准备清单或执行结果引用。
后续 plan、Subject 和观测身份可以引用该键，
避免 manifest 与自身依赖循环。

同一 experiment_key 可以有多个不可变实验修订。
不同请求不覆盖旧定义。

本轮 mode 固定为 trace_audit。
导入的是完整链路记录时可以评价其实际阶段，
但不能因此宣称本框架在线执行过该链路。

### C2. 来源导入与 Run 选择

run_sources 每项采用以下一种形式：
- existing_batch：完整 RunImportResult。
- import_request：明确的 Adapter RunImportRequest 引用。

不得同时提供两种形式。

existing_batch 通过 Adapter.load_runs 校验读取。
import_request 通过 Adapter.import_run 完整导入后读取。

Runner 不解析 Adapter 私有文件，
不复制 JSONL 字段映射规则。

导入文件包含非法行、关联冲突或未明确处置的记录时，
按 Adapter 契约失败。
不能只导入成功或有答案的部分。

run_selection 采用 explicit_run_table_v1：
为每个预期观测明确给出：
- case_key。
- arm_id。
- sample_slot_id。
- selected_run_ref：可以为 null。
- selection_basis_ref。
- 被检查来源范围。
- 未选择候选及原因。

预期观测全集为：

selected_cases × arms × sample_slots

每个组合必须恰有一条选择记录。

selected_run_ref=null 表示没有选定可用运行。
必须说明：
- source_record_missing。
- 明确未执行。
- 其他可核验的缺失原因。

不能将 null 自动解释为系统超时或失败。

本轮不自动选择：
- 最新 revision。
- 最大 attempt。
- 最后一次运行。
- 最高分或最长答案。
- 第一个成功记录。

Run 选择可以引用冻结文件中的明确来源位置，
但必须有确定的映射依据。
不能按当前文件顺序、query 相似度或答案内容猜测。

完整来源批次内的记录分别登记：
- selected。
- not_selected。
- outside_selected_cases。
- duplicate_import。
- alternate_revision。

每种非 selected 状态均保留原因。
全部来源记录必须得到明确处置。

来源执行去重：
- 相同 RunRef 的重复导入不增加运行数量。
- 相同 source_execution_key 不构成独立采样。
- 同一执行的不同 revision 需要明确选择。
- 同一已知执行不得占用多个独立观测槽位。
- 未知执行身份保留 execution_identity_unverified。

A/B 同时选择同一真实执行不能证明系统差异。
已知重复绑定属于准备冲突；
未知来源身份形成复现与独立性限制，
交由后续报告明确呈现。

未选择的真实运行及计量仍保留，
不能因不参与质量评分而删除历史成本。

### C3. 用例定位与实际关联

Runner 拥有 runner-case-locator-v1：
- dataset_version_ref。
- case_key。
- projected_input_ref。
- projection_version。
- locator_digest。

projected_input_ref 来自 Registry.project_system_input，
只含 query、history、as_of 和 access_scope。

locator 用于导入关联与审计，
不是发送给被测系统的普通输入。

CaseAssociation、declared_input_ref 和来源位置
只构成导入时的声明。
Runner 仍检查真实运行所记录的输入。

关联检查顺序：
1. 数据版本和 CaseKey 存在于固定选择中。
2. Run 的 case_id、case_revision 与选择一致。
3. case_locator_ref 指向正确数据版本及用例。
4. declared_input_ref 与白名单输入投影一致。
5. observed_input 的各字段与实际用例条件比较。
6. 声明配置和实际配置的差异被保留。
7. Run 来源符合所选 arm 的固定约束。

query、history 等按提供方规范类型比较。
不额外执行：
- 大小写折叠。
- 去除首尾空白。
- 历史消息压平。
- 时间条件猜测。
- 权限范围扩张。

明确允许的类型等价转换必须有固定版本与依据。
不能通过临时归一化掩盖真实输入差异。

逐字段状态：
- matched：值一致且有符合规则的实际记录依据。
- unknown：未记录、withheld、无法解释或仅有不足的声明。
- mismatch：存在明确矛盾。

declared 值恰好与用例相同，
不能自动提升为独立观察到的 matched。
协议保证及其依据可以被固定记录，
但不能复制期望输入后标记为 logged。

已知 mismatch：
- 阻止该实验准备完成。
- 不将 Run 强行绑定到所选用例。
- 不生成虚假系统质量零分。

unknown：
- 保留观测和 Run。
- 构造带限制的 Subject。
- 由 Evaluator 决定哪些质量指标不可测。
- 不阻止运行状态和明确计量等独立指标。

检查结果投影为 Evaluator 所有的 association_report。
Runner 不建立另一套与其矛盾的评分资格规则。

### C4. 准备步骤与模块间转换

准备步骤按固定顺序执行：

1. 校验请求、权限、版本和选择范围。
2. 完成或读取完整 Run 导入批次。
3. 校验全部观测及来源记录处置。
4. 核对实际输入、配置和用例关联。
5. 通过 Evaluator.prepare_subject 创建全部 Subject。
6. 构造 catalog_only 蓝图。
7. 准备所需实际映射和已有审定材料。
8. 对全部评价任务调用 check_dependencies。
9. 构造 available_inputs 计划。
10. 保存完整准备清单。

所有预期观测先有 Subject，
再交给 Planner 构造观测绑定。
没有 Run 的观测使用 run_ref=null 的 Subject。

mapping_preparation 包含：
- 明确的 MappingRequest 引用。
- 或已经存在的 MappingBatchResult。
- 与观测、指标实例及 scope 的关联。

Mapper 请求和结果沿用其提供方契约。
不能根据答案内容选择更有利的 top-K、
阶段实例或生成调用。

assessment_preparation 包含：
- 明确的 ScoringRequest 引用。
- 已有 ReviewImportResult，或本地审定文件及导入 policy。
- 来源批次选择规则。
- 固定转换器身份与版本。
- 与观测、指标实例及 scope 的关联。

转换流程：
- Evaluator 根据真实对象构造 ScoringViewDescriptor。
- Runner 投影为 ReviewTarget 和 ReviewRubricRef。
- Review 读取并检查来源绑定。
- Runner 构造 AssessmentImportEnvelope。
- Evaluator 校验 payload、冲突及业务适用范围。

预期 target 和 rubric 必须来自真实当前视图，
不能直接复制待检查 ReviewRecord 的值。

转换必须保留：
- 原始文件与规范记录。
- 批次、记录身份、revision 和文件内定位。
- 导入回执。
- check_binding 结果。
- decision、method 和必需检查项。
- assessment_payload_ref。
- 排除及分歧。
- 转换版本。

Runner 不将 approved 转为任务分数 1，
也不将 reject 转为系统任务失败。

所有选定来源批次完整读完后才提交转换。
中途读取失败不能提交已读子集冒充完整来源集合。

没有有效评分材料时允许形成空 assessment 选择，
保留相关依赖缺口。
不能自动创建裁判、人工标签或默认满分。

没有 Run 时不伪造 Mapper 请求。
对应依赖在 Subject 和 DependencyReport 中明确缺失。

准备操作只进行本地读取、导入和确定性计算。
不产生新的业务预测或模型裁判调用。

### C5. ExperimentManifest

ExperimentManifest 固定一份可解释的准备结果，包含：
- schema_version、contract_profile。
- experiment_key。
- experiment_revision。
- planning_scope_key。
- request_ref。
- mode。
- store_id、access_domain。
- dataset_version_ref。
- 固定用例选择及来源。
- arms 和 sample_slots。
- 全部预期观测。
- 完整 Run 选择及未选记录处置。
- 用例关联报告。
- Subject 引用集合。
- MappingBatchResult 集合。
- AssessmentBatchResult 集合。
- catalog_only 蓝图引用。
- available_inputs EvaluationPlan 引用。
- 各提供方实现及契约版本。
- execution_policy。
- reporting_intent。
- runtime_environment_ref。
- provenance 和 limitations。

experiment_revision 为规范内容排除自身修订字段后的摘要。
清单不嵌入自身 ArtifactRef 或执行结果，
不包含由本次评分推导的分数和状态。

完整 EvaluationPlan 使用 Planner.encode_plan 保存。
Runner 不直接重写计划 JSON 或其中的 task_id。

清单提交条件：
- 全部必需引用已提交并可读取。
- 来源批次和观测数量闭合。
- 每个观测具有唯一 Subject。
- 计划与当前固定输入通过 validate_plan。
- 不存在已知关联冲突或非法引用。
- 缺失和不确定依赖已被明确登记。

有效数据缺失不等于清单非法。
可提交带测量缺口的诊断实验。

准备成功只表示可以按计划尝试离线评价。
不表示：
- 所有指标可计算。
- 系统质量通过。
- 样本量和统计条件充分。
- 实验已经执行。
- 门禁可以通过。

Planner 的 gate_input_status 原样保留。
存在必选输入缺口时只能形成诊断依据；
不得在 Runner 中改写为门禁就绪。

reporting_intent 保存：
- requested：是否要求后续报告。
- 固定的报告、比较及门禁 policy 引用。
- 未提供或尚未核验的配置限制。

这些 policy 的业务含义归 Reporting。
Runner 只核验引用存在性与完整性，
不自行认证统计方法或门禁资格。

Reporting 设计尚未确认时，
不得声称已经形成可执行的正式门禁闭环。

### C6. PreparationResult

PreparationResult 包含：
- preparation_ref。
- status：ready 或 blocked。
- experiment_manifest_ref：blocked 时可以为 null。
- validation_report_ref。
- blockers。
- limitations。
- receipt_ref。

ready：
准备清单完整提交，可以进入本轮单次执行流程。

blocked：
请求能够被解释，但存在明确关联或配置冲突，
尚不能提交可执行实验清单。

非法 schema、损坏引用、权限或存储失败
采用公开错误返回，不伪造 blocked 内容。

准备校验报告保存：
- 每项检查及结果。
- 完整观测数和来源记录处置数。
- 关联冲突。
- 输入缺口。
- 计划检查摘要。
- 已完成和未完成的准备步骤。

准备部分完成后失败：
- 已提交 Provider 产物保留其合法身份。
- 不宣称整个实验已经 ready。
- 不覆盖或删除合法原始导入批次。

中间产物的发现、核验及复用遵守 C13、C14 和 C16。
只有正式提交且通过当前授权和完整性检查的结果才能复用。

### C7. 执行身份与任务操作记录

execution_policy 使用 C12 定义的明确 profile。
single_pass_v1 保持单次调用、关闭自动重试和评分缓存。
recoverable_local_v1 支持受预算约束的本地恢复、
明确允许的操作重试及显式来源的评分复用。

两个 profile 均固定 concurrency=1，
不调用业务系统或外部裁判，
不按评分值改变任务集合或顺序。

开始评价前保存 ExecutionIntent：
- execution_id：唯一操作身份。
- experiment_manifest_ref。
- plan_ref。
- execution_policy。
- 当前运行环境引用。
- 操作主体、UTC 开始时间及访问策略 revision。

ExecutionIntent 成功提交后才能派发评分。
它不修改 ExperimentManifest。

execution_id 与来源 Run、业务 attempt 分开。
重新执行离线评分不会增加业务执行次数。

每个任务的 TaskOperationRecord 包含：
- execution_id、task_id。
- task_input_revision：实际 EvaluationRequest 的规范摘要。
- evaluation_attempt：同一任务和输入修订内的正整数调用序号。
- invocation_id：本次真实 Provider 调用身份。
- continuation_id：本次推进或恢复的操作身份。
- call_intent_ref。
- 实际 EvaluationRequest 引用。
- 输入和前置结果引用。
- 操作开始、结束时间。
- operation_state。
- JudgmentResult：可为空。
- 结构化错误：可为空。
- evaluator 原始完成状态和限制。

operation_state：
- completed：形成合法、已提交的 Judgment。
- operation_failed：调用结束但没有形成合法 Judgment。

completed 不要求 Judgment.status=ok，
也不要求其业务 value 为通过。

error Judgment 是有效评分结果，
因此操作状态仍为 completed。

没有提交 Judgment 的程序、权限或存储失败
才属于 operation_failed。
Runner 不能自己生成替代 Judgment。

未开始任务不创建虚假的执行 attempt。
它们由最终 TaskOutcome 明确记录。

### C8. 前置结果与实际请求实例化

执行顺序采用固定计划的拓扑顺序。
Runner 不重新排序以优先展示更高分结果。

前置依赖满足的执行条件是：
前置任务已经结束，并明确保存结果或失败。

不要求：
- 前置 Judgment.status=ok。
- 前置指标 value 大于零。
- 前置任务业务通过。

Runner 对 component_bindings：
- 读取指定任务的真实 Judgment 引用。
- 或读取计划固定的已有 Judgment。
- 核对 Subject、实例、scope 和包身份。
- 将实际引用放入 component_judgment_refs。

任务 ID 不是 ArtifactRef，
不能作为评分结果引用传给 Evaluator。

前置操作没有形成 Judgment：
- 保留其失败记录。
- 后续请求只传入实际存在的合法组件。
- 将缺少组件作为真实依赖缺口保留。
- 仍允许 Evaluator 按自身逻辑形成结论。

例如：
- TaskSuccess 可以依据已知系统超时返回 fail。
- 某个必要维度已经失败时，不被其他未知维度覆盖。
- 充分组指标可以使用三值逻辑得到确定结论。

Runner 不自行计算这些分数。

组件引用损坏或指向其他对象时，
不能作为“缺少标签”继续使用。
应停止该次非法请求并保存操作错误。

每次派发重新检查：
- 当前权限和固定输入完整性。
- 计划身份及任务对应关系。
- 实际 EvaluationRequest 与模板一致性。

Evaluator 仍独立检查真实依赖。
计划阶段曾经通过，不替代运行时检查。

### C9. TaskOutcome 与 ExecutionResult

TaskOutcome 为每个计划任务提供唯一结论：
- task_id。
- state：judgment_available、operation_failed 或 not_started。
- TaskOperationRecord 引用：适用时提供。
- Judgment 引用：只有真实存在时提供。
- reason_code。
- limitations。

ExecutionResult 包含：
- schema_version、contract_profile。
- execution_id。
- execution_intent_ref。
- experiment_manifest_ref。
- plan_ref。
- status：completed 或 partial。
- 全部 TaskOutcome。
- 全部根 ResultSlotOutcome。
- 数量和状态摘要。
- 开始、结束时间。
- 错误及未完成范围。
- 来源 usage 及本次离线操作的计量关系。
- result_id。

根 ResultSlotOutcome 包含：
- 原 plan 中的 slot_id。
- observation_key。
- root_metric_instance_id。
- judgment_ref：可以为 null。
- 未完成原因。
- 关联 TaskOutcome。

不得省略缺少答案、上下文、标签或评分的槽位。

completed：
全部计划任务均形成合法 Judgment。

即使存在：
- not_computable。
- inconclusive。
- error。
- 业务 fail。
执行也可以 completed。

partial：
至少一个任务没有形成 Judgment。
必须保留全部未完成范围。

作业完成度与指标有效覆盖率分别统计。
Runner 不将 completed 数量直接称为 N_ok。

结果数量必须满足：
- TaskOutcome 数等于计划任务数。
- 根槽位结果数等于原 ResultSlot 数。
- 每个任务及槽位恰好一次。
- 根槽位不能引用辅助指标冒充结果。

result_id 为规范内容排除自身字段后的摘要。
执行时间和操作身份属于本次执行事实，
因此不同 execution 可以有不同结果身份。

ExecutionResult 最后提交。
没有结果清单不代表已经形成完整实验结果。

如果 Store 故障导致结果清单无法提交，
公开操作明确失败。
不能返回不存在的结果引用。

### C10. 公开接口

公开构造入口：

create_runner(
    artifact_store,
    review,
    dataset_registry,
    adapters,
    evidence_mapper,
    evaluators,
    metric_planner
) -> Runner

公开操作：

prepare_experiment(
    request_ref,
    access_context
) -> PreparationResult

execute_once(
    preparation_ref,
    access_context
) -> ExecutionResultRef

load_experiment(
    preparation_ref,
    access_context
) -> ExperimentReader

load_execution(
    result_ref,
    access_context
) -> ExecutionReader

ExecutionResultRef 包含：
- result_ref。
- receipt_ref。

prepare_experiment：
- 校验并协调本地导入与准备。
- 不执行指标评分。
- 不产生被测业务调用。

execute_once：
- 只接受 ready 的准备产物。
- 只接受 available_inputs 计划。
- 重新核验计划和固定依赖。
- 按本轮单次策略执行。
- 不自动生成报告或发布门禁。

load_experiment 返回：
- 请求与准备清单。
- 数据、arm、观测、Run 和 Subject。
- 固定映射、审定及计划引用。
- 准备限制和依赖。

load_execution 返回：
- 固定 ExecutionResult。
- 任务及根槽位结果。
- 实际请求、Judgment 和错误的读取关联。
- 完整度及未完成范围。

Reader 在读取时检查当前权限和完整性。
不能只依赖历史回执或磁盘文件存在。

Reporting 后续消费这些公开只读结果，
不能通过 Runner Reader 启动新的评分或系统调用。

### C11. 错误与限制

公开错误：

| reason_code | 含义 |
|---|---|
| runner.invalid_input | 请求、策略或布局非法 |
| runner.unsupported_schema | 不支持的版本或 profile |
| runner.source_conflict | 来源、修订或独立执行选择冲突 |
| runner.association_mismatch | 用例、输入或配置存在明确矛盾 |
| runner.preparation_not_ready | 准备尚未形成可执行清单 |
| runner.plan_invalid | 计划与实际固定输入不一致 |
| runner.component_invalid | 前置结果属于错误对象或实例 |
| runner.provider_operation_failed | 提供方操作失败，保留原错误 |
| runner.access_denied | 当前读取或写入未授权 |
| runner.integrity_mismatch | 内容、数量或引用损坏 |
| runner.storage_failure | 无法读取或提交要求的产物 |
| runner.internal_error | 编排程序故障 |

提供方错误保留：
- 原 owner、reason_code 和 category。
- 安全对象身份及调用阶段。
- 原始重试条件声明。

是否重试由 C12 的冻结策略及 C15 的判定规则决定。
提供方的重试提示不直接授权重试。

全局权限失效或完整性损坏时停止新派发。
若仍能合法保存，提交 partial 结果；
无法保存时明确返回操作失败。

Provider 的单项评分故障可以继续其他合法任务。
不能将其错误改写为系统质量失败。

规模限制沿用各提供方：
- 来源文件：Adapter 和 Review 的导入限额。
- 观测与任务：Planner 限额。
- 文本与主张：Mapper 和 Evaluator 限额。
- 字节产物：Store 限额。

Runner 在准备阶段预检能够确定的规模。
不得通过截断用例、减少指标或忽略来源满足限额。

预算、恢复、评分复用及本地协调接口见 C12–C17。
未声明的策略值和超出本轮范围的并发模式在执行前拒绝。

### C12. 执行策略、预算与停止条件

execution_policy 是实验请求中的冻结配置，包含：
- profile：single_pass_v1 或 recoverable_local_v1。
- concurrency：本轮只允许 1。
- preparation_deadline_ms。
- execution_deadline_ms。
- local_step_timeout_ms。
- cleanup_grace_ms。
- max_local_invocations。
- max_attempts_per_task_input。
- automatic_retries。
- retryable_reason_codes。
- retry_unknown_local_outcome。
- backoff_initial_ms、backoff_max_ms。
- jitter_seed。
- cache_policy。
- failure_policy。

默认值：
- profile：single_pass_v1。
- preparation_deadline_ms：600000。
- execution_deadline_ms：600000。
- local_step_timeout_ms：60000。
- cleanup_grace_ms：5000。
- max_local_invocations：100000。
- max_attempts_per_task_input：1。
- automatic_retries：false。
- retryable_reason_codes：空列表。
- retry_unknown_local_outcome：false。
- backoff_initial_ms：250。
- backoff_max_ms：5000。
- cache_policy.mode：disabled。
- failure_policy：continue_independent_tasks。

所有时间和数量必须为正整数。
开启 automatic_retries 时，
max_attempts_per_task_input 必须明确，且本轮不超过 3。

single_pass_v1：
- 每个计划任务最多调用一次。
- 不自动重试已知失败或未知调用结果。
- 不消费跨 execution 评分缓存。
- 可以发现已经提交但尚未返回的结果。
- 不通过 resume 增加该任务的调用次数。

recoverable_local_v1：
- 允许在冻结次数、错误范围和预算内重试。
- 是否重试未知结果必须单独明确。
- 允许 explicit_candidates_v1 评分复用。
- 仍不改变业务 Run、来源 attempt 或 repetition。

预算分别覆盖：
- preparation：导入、映射、评分材料转换及依赖检查。
- execution：指标评价和执行结果提交。

一个阶段的剩余配额不能自动转移给另一个阶段。
来源系统的历史 token、费用和耗时不消费本地调用配额，
但必须继续保留在来源计量中。

每次 Provider 调用前：
1. 检查取消和 deadline。
2. 检查调用总配额。
3. 检查该任务输入修订的 attempt 上限。
4. 持久化调用意图并占用一次调用配额。
5. 才允许派发 worker。

调用失败、超时或结果未知也占用已派发次数。
不能因没有取得答案而退还调用次数。

max_local_invocations 限制调用次数，
不表示货币、CPU 时间或内存硬上限。

源 usage 未知仍为未知。
不能因为本框架没有新增模型调用，
就把历史系统成本填成零。

首次开始阶段时固定：
- started_at。
- expires_at。
- 总调用配额。
- 原始策略身份。

单次运行内部使用单调时钟计算耗时。
绝对 expires_at 用于进程重启后的期限检查。

恢复时不重新计算完整 deadline，
不重置调用次数、attempt 或退避进度。

检测到明显时钟回退或无法可靠解释剩余期限时，
停止派发并记录 clock_uncertain，
不能因此获得更长运行时间。

单次 worker 的有效 timeout 为：
local_step_timeout 与阶段剩余时间的较小值。

准备工作或评分缺少预算时：
- 停止新派发。
- 保留所有已提交产物。
- 保留未完成步骤及完整结果槽位。
- 不删除样本，不降低指标要求。
- 不改变审定规则。

没有剩余预算的恢复只进行允许的核验和结果发现，
不派发新的 Provider 调用。
增加预算需要新的明确请求与操作，
旧操作记录不被改写。

### C13. 操作身份、调用意图与检查点

Runner 定义 OperationHandle：
- store_id、access_domain。
- operation_key：调用方预先固定的非敏感幂等键。
- operation_id。
- operation_kind：prepare 或 execute。
- binding_digest。
- root_identity。

operation_key 不直接作为文件名或路径。
本地路径使用完整作用域身份的摘要。

binding_digest 绑定：
- prepare：固定 ExperimentRequest。
- execute：固定 PreparationResult、执行策略及复用来源集合。

相同作用域、相同 operation_key、相同绑定：
返回或继续同一个操作。

相同 key 对应不同绑定：
返回 operation_conflict。
不能覆盖原操作或自动解释为新实验。

operation_id 与以下身份分别解释：
- experiment_key 和 experiment_revision。
- execution_id。
- PlanTask.task_id。
- 来源 Run 和 source_execution_key。
- Provider 的导入操作。
- 真实业务 attempt。

每次推进操作生成 continuation_id。
它不重置 operation_id、execution_id 或预算。

每次真实 Provider 调用先提交 CallIntent：
- operation_id、continuation_id。
- step_id 或 task_id。
- task_input_revision。
- invocation_id。
- evaluation_attempt。
- 固定请求引用。
- 前置结果引用。
- 预算占用。
- timeout 和期限。
- UTC 调用意图时间。
- 上一个检查点引用。

没有合法 CallIntent 不得启动 worker。

CheckpointEvent 包含：
- operation_id。
- sequence：连续正整数。
- previous_event_ref。
- event_type。
- 调用、任务及请求身份。
- 已提交 Provider 结果引用。
- 调用结果和错误。
- 预算累计值。
- 已确定的任务结果选择。
- 内容摘要。

事件类型包括：
- operation_started。
- call_intended。
- call_finished。
- call_outcome_unknown。
- result_recovered。
- result_reused。
- retry_scheduled。
- dispatch_stopped。
- result_snapshot_committed。

事件通过 Store 保存为不可变产物。
sequence 与 previous_event_ref 构成顺序链。

只有协调者提交检查点和任务结果选择。
worker 不并发追加共享日志或修改计划。

检查点不能只保存“完成到第几题”。
必须能够还原：
- 哪个固定请求被调用。
- 哪些结果已经提交。
- 哪些调用结果仍未知。
- 已消耗多少配额。
- 哪个 Judgment 被选中。
- 哪些根槽位尚未完成。

call_intended 后没有可靠完成记录：
恢复为 outcome_unknown，
不直接视为尚未执行，也不假定已经成功。

### C14. 评分缓存与精确复用

cache_policy.mode：
- disabled。
- explicit_candidates_v1。

disabled：
不搜索或使用其他 execution 的评分。
同一操作的已提交结果发现属于恢复，
不计作跨作业缓存。

explicit_candidates_v1：
调用方在执行开始前提供固定的复用来源集合。
来源是明确的 ExecutionResult 引用，
不接受目录扫描、latest 或按 query 搜索。

复用来源冻结在 ExecutionIntent 中，
不进入引用自身结果的 ExperimentManifest。

来源集合不得按业务分数排序或事后删选。
候选按固定完整身份顺序处理。

只对最终实际 EvaluationRequest 检查复用，
不能仅凭 PlanTask.task_id 判断相同。

精确复用必须一致：
- Store、access_domain。
- Subject。
- 数据版本、CaseKey、Run 和实际输入。
- 评价包、MetricSpec、MetricInstance。
- scope 和调用、阶段、top-K。
- mapping_batch_refs。
- assessment_batch_refs 及其完整来源选择。
- component_judgment_refs。
- rubric、提取器、规则及实现版本。
- 影响结果的配置、限额和来源保证。

所有集合按所属契约的规范规则比较。
不能通过忽略引用差异提升缓存命中率。

候选 Judgment 通过 Evaluator.load_judgment 核验。
Runner 不重写其身份、依据、Subject 或版本。

默认允许复用：
- ok。
- not_applicable。
- not_computable。
- inconclusive。

这些状态只在完整请求及依赖确实相同时复用。
新增标签、修复上下文或改变前置结果，
都会使实际请求不同，不能继续复用旧结论。

error Judgment 默认不进入跨 execution 缓存。
明确启动新的 execution 可以重新尝试评价；
不能在同一任务中循环重试到出现通过分数。

多个候选的 judgment_id 相同：
按同一内容去重，保留全部来源关系。

相同实际请求出现不同 Judgment 内容：
记录 cache_conflict。
不选择较高分、较新时间或有利状态，
也不通过立即重算掩盖该冲突。

正式消费前仍重新检查：
- 当前权限。
- 内容完整性。
- 固定依赖可用性。
- 绑定与实现兼容性。

发现撤权、损坏或非法绑定时明确报错，
不能当作普通 cache miss 后绕开检查。

缓存命中生成 CacheReuseRecord：
- 当前 operation、execution 和 task。
- 实际 EvaluationRequest。
- 原 Judgment 和来源 execution。
- 精确匹配依据。
- 当前核验记录。

缓存命中不创建虚假的 evaluation_attempt，
不占用 Provider 调用次数。
本地核验仍受 deadline 和输入限额约束。

TaskOutcome 的 origin 为：
- computed。
- recovered。
- cache_reused。

对应的 completion_record_ref 指向：
TaskOperationRecord、恢复记录或 CacheReuseRecord。

原评分及原 usage 谱系继续保留。
本次新增评分调用为零，
不表示来源 Run 或原评分历史成本为零。

### C15. 重试、超时、取消与任务结果选择

#### C15.1 重试条件

自动重试需要同时满足：
- recoverable_local_v1。
- automatic_retries=true。
- 当前错误在明确 allowlist 中。
- 提供方或 worker 边界表明具备重试条件。
- 次数、调用配额和 deadline 均有余量。
- 当前没有取消。
- 固定请求和实际依赖没有变化。
- 前一 worker 已确认结束。

以下情况不自动重试：
- 业务分数为零或 fail。
- not_applicable。
- not_computable。
- inconclusive。
- 已提交的 error Judgment。
- 输入、schema、版本或关联错误。
- 审定分歧。
- 权限拒绝和内容损坏。
- 数据或实现不受支持。
- 已知预算耗尽。

可重试项必须使用完整 owner.reason_code。
不能将所有 storage_failure 或 internal_error 一概列为临时故障。

已知 operation_failed 先检查是否存在可恢复的提交结果，
再判断是否需要新调用。

outcome_unknown 只有在以下条件成立时才可重试：
- retry_unknown_local_outcome=true。
- 操作属于本轮明确允许的本地导入、映射或评价。
- 已确认旧 worker 不再运行。
- 本地 Provider 操作具有不可变或幂等提交语义。
- 所有原意图及未知原因继续保留。
- 仍受原调用和 attempt 配额约束。

不将上述许可推广到未来的业务系统或远端模型调用。

退避采用有上限的指数退避和固定 seed 抖动。
等待时间也消耗阶段 deadline。
恢复沿用原 attempt 和退避进度，不重新从第一次开始。

#### C15.2 本地 worker 与超时

本轮执行环境为 Windows 本地 NTFS。
每个操作最多一个在途 worker。

耗时 Provider 调用放入独立进程。
线程等待超时不被解释成计算已经停止。

worker 只运行内置、明确列举的本地步骤：
- 运行或审定导入。
- 文本映射。
- 评分视图及审定材料处理。
- 依赖检查。
- 单项评价。

不接受外部脚本路径、任意模块导入或用户代码。

worker 通过固定配置重新构造所需提供方，
版本、Store 和访问域必须与协调者一致。
它不取得 Runner 作业协调锁。

Windows worker 置于受控 Job Object，
禁止脱离作业；协调者退出时关闭作业并终止其 worker。
启动采用先关联作业、后允许执行的顺序，
避免产生未被跟踪的子进程。

如果环境不能提供所需隔离和终止语义，
recoverable_local_v1 在启动前拒绝，
不降级后继续声称 timeout 可强制执行。

超时或取消时：
1. 停止新派发。
2. 通知本地步骤停止。
3. 终止仍未结束的 worker。
4. 等待确认进程结束。
5. 核验已提交结果或交接记录。
6. 保存超时、取消和未知结果。
7. 按冻结规则决定是否还能重试。

cleanup_grace_ms 限制清理等待。
无法确认 worker 结束时记录 cleanup_unconfirmed，
不启动替代 worker，不允许并发恢复同一操作。

达到 timeout 后才取得的结果作为迟到产物保留。
是否超过期限依据协调者观察到的完成边界确定，
不根据分数决定接受或舍弃。

超时不授权 Runner 构造 error Judgment。
有无评分结果由 Evaluator 的实际提交决定；
没有合法结果时保存操作失败或未知记录。

CPU、内存和磁盘使用不在本轮承诺硬上限。
文本、记录和产物大小继续由提供方限额约束。

#### C15.3 取消

cancel_operation 提交独立的取消请求：
- operation_id。
- 当前主体和访问策略。
- UTC 时间。
- reason。

取消请求不直接覆盖任务状态。
协调者在派发前、退避期间和等待 worker 时检查。

取消后：
- 不派发新调用。
- 不增加重试。
- 保留已提交结果。
- 结束在途 worker 后保存完整未完成范围。

取消是操作停止原因，
不构造来源 Run.status=cancelled，
也不等于系统任务质量失败。

#### C15.4 结果选择与后继重算

同一 task_input_revision：
- 第一个符合期限和绑定条件的合法 Judgment 被选中。
- 不根据业务 value 继续尝试。
- 后续重复内容只增加来源记录。
- 冲突内容明确报错，不覆盖已有选择。

自动重试在该任务向后继释放最终结果前完成。
重试次数耗尽后，可以将明确缺少组件的状态传给后继。

恢复 partial 执行时，
若此前失败任务得到真实组件，
可能改变后继实际 EvaluationRequest。

此时：
- 为受影响任务计算新的 task_input_revision。
- 只使实际输入变化的后继结果不再作为当前选择。
- 按依赖闭包检查并重新评价。
- 保留旧请求、旧 Judgment 和替代原因。
- 不重置整个操作的总调用预算。

相同输入修订的 error Judgment 不因 resume 自动重试。
输入确实变化后的重新评价属于新输入修订，
必须明确记录变化来源。

数据版本、来源审定、包、rubric 或 Run 的变化
不属于当前操作内的恢复。
它们需要新的实验请求或准备修订。

### C16. 本地保存、发现与恢复

#### C16.1 本地根目录

create_runner 增加 local_config，包含：
- root：Runner 自己的本地协调目录。
- mode：open_existing 或 create_new。
- lock_timeout_ms：默认 5000。
- max_journal_record_bytes：默认 1048576。

root 解析为固定绝对路径，
绑定固定 Store 和本地格式版本。

Runner 只操作自身 root，
不扫描或修改 Store、Registry、Adapter 的私有目录。

业务产物和检查点正文保存在 Store。
本地目录用于：
- 按 operation_key 查找操作。
- 发现不可变事件引用。
- worker 结果交接。
- 当前进程协调和取消请求。

本地布局：

root/
  runner.json
  root.lock
  operations/
    <scoped-key-digest>/
      operation.json
      operation.lock
      events/
        <sequence>/
          manifest.json
      candidates/
      worker-outbox/
      control/
  indexes/

目录名使用摘要或内部生成身份。
不拼接外部路径片段，不跟随越出 root 的链接。

indexes 是可重建索引。
它不决定实验、Judgment 或结果是否正式存在。

#### C16.2 协调与提交

进程内对同一规范 operation 使用共享互斥。
跨进程使用操作系统文件锁，
在推进或恢复期间持有 operation.lock。

不根据 PID、文件年龄或文件存在性判断锁已失效。
不通过删除锁文件解锁。

锁顺序：
root 初始化锁
→ operation 锁
→ Provider 自身锁。

root 初始化锁不在耗时评价期间长期持有。
worker 不反向取得 operation 锁，避免锁顺序循环。

同一操作同时被推进或恢复：
后一个调用在限时内等待，
超时返回 operation_busy。

不同 operation 可以独立推进；
Store 的写入仍服从自身协调规则。
本轮不提供一个操作内部的多 worker 调度。

CheckpointEvent 提交：
1. 构造规范事件字节及预期摘要、长度。
2. 在 Runner candidates 中保存待提交引用和关联。
3. 通过 Store.put_blob 提交事件正文。
4. 核验正式内容。
5. 在同一文件系统暂存完整本地事件目录。
6. 刷新并关闭文件。
7. 原子提交到 events/<sequence>。
8. 更新可重建索引。

本地事件只保存固定引用和校验信息。
正式事件正文仍从 Store 授权读取。

Store 成功而本地事件目录未提交时，
恢复可以使用 candidates 中已知的完整引用
调用公开接口核验，而不扫描 Store 私有文件。

只有正式事件链中的预算和结果选择生效。
不能根据孤立缓存文件推断任务完成。

#### C16.3 worker 结果交接

worker 调用 Provider 后，
通过 Runner 自有 outbox 写入：
- invocation_id。
- 固定请求身份。
- Provider 返回结果或明确错误。
- 已知结果引用。
- schema、摘要和完整性信息。

outbox 使用暂存文件加原子提交。
协调者验证后转换为正式检查点。

worker 已提交结果，但在写入 outbox 前退出：
可能无法取得结果引用。

此时：
- 保存 outcome_unknown。
- 不宣称已发现全部已提交产物。
- 不扫描提供方私有目录。
- 按 C15 的未知结果策略决定是否重新调用。
- 保留可能存在但尚未关联的产物限制。

重做本地确定性评价不会创建业务 Run，
也不构成外部调用 exactly-once 保证。

#### C16.4 恢复顺序

恢复时：

1. 授权读取操作绑定。
2. 取得 operation 锁。
3. 确认不存在仍在运行的旧 worker。
4. 核验事件链及已知候选。
5. 处理合法 outbox。
6. 通过公开 Reader 核验 Provider 结果。
7. 还原预算、调用次数及结果选择。
8. 重新核验准备清单、计划和当前权限。
9. 决定复用、继续、停止或报错。
10. 提交新的检查点和结果快照。

链断裂、摘要错误或同序号内容冲突：
返回 checkpoint_corrupt，保留现场。
不能跳过坏事件继续计数。

已知当前输入变化：
返回 resume_input_changed。
不在恢复时自动改绑新版本。

读取旧数据集时仍遵守 Registry 生命周期规则。
已撤回数据只能用于明确的授权审计，
不能恢复成正式可发布门禁实验。

准备步骤恢复只复用：
请求、版本、转换器和依赖一致的完整结果。

不完整导入批次、部分读取来源、
未提交清单及非法中间文件不提升为成功产物。

#### C16.5 结果快照

每次结束推进生成不可变结果快照。

ExecutionResult 增加：
- continued_from_result_ref：可为空。
- checkpoint_ref。
- stop_reason。
- deadline 和预算消耗摘要。
- 全部 attempt、复用与结果未知记录。
- 当前选定结果的替代关系。

status 仍为 completed 或 partial，
沿用 C9 的完整度定义。

stop_reason 可以是：
- all_tasks_accounted。
- deadline_exhausted。
- invocation_budget_exhausted。
- cancelled。
- provider_failure。
- cleanup_unconfirmed。
- integrity_failure。
- access_revoked。

旧快照保持不变。
load_execution(result_ref) 始终读取明确指定的快照，
不把旧引用自动重定向到最新结果。

completed 快照再次 resume：
核验后返回已完成结果与新的操作回执，
不因为存在低分或 error Judgment 重新评分。

partial 快照的继续受原策略和剩余预算约束。
不能仅因用户调用 resume 就增加原配额。

结果已经提交但响应或回执失败：
优先发现并核验该结果，
再生成独立回执，不重新执行任务。

若正式结果提交与本地发现记录之间的引用也丢失，
按已知提交不确定性报告，
不能承诺在任意文件丢失后完整自动恢复。

本轮保证目标是本地进程中断后的可核验恢复。
整机断电持久性、磁盘损坏修复和分布式故障恢复
不在声明范围，实际提交语义仍需目标环境验证。

### C17. 扩展公开接口与状态读取

create_runner 的 local_config 为明确构造参数。
其余 Provider 参数保持 C10 的归属和依赖方向。

新增操作：

execute(
    preparation_ref,
    operation_key,
    reuse_source_refs,
    access_context
) -> OperationAdvanceResult

resume_operation(
    operation_handle,
    access_context
) -> OperationAdvanceResult

get_operation(
    operation_key,
    operation_kind,
    access_context
) -> OperationSnapshot

cancel_operation(
    operation_handle,
    reason,
    access_context
) -> CancellationReceipt

prepare_experiment 增加可选 operation_key 参数。
未提供时生成并在 Provider 调用前记录操作身份。
需要在没有成功响应的情况下确定恢复入口时，
调用方应预先提供 operation_key。

execute_once 保持原单次入口：
- 采用 single_pass_v1。
- 自动生成新的操作键。
- 不自动消费复用来源。
- 记录可回查的 OperationHandle。
- 不在同一任务输入下增加调用次数。

OperationAdvanceResult 包含：
- operation_handle。
- operation_snapshot_ref。
- preparation_ref：准备操作形成结果时提供。
- execution_result_ref：执行操作形成快照时提供。
- receipt_ref。

OperationSnapshot 包含：
- operation_handle。
- phase：preparing、evaluating 或 finalizing。
- state：created、running、stopping、
  completed、partial、blocked 或 failed。
- 当前检查点引用。
- 明确的结果快照引用。
- stop_reason。
- 预算及未完成摘要。

这些状态属于编排操作，
不能转换为 Run.status 或 Judgment.status。

get_operation 不执行任务、不自动恢复，
不返回“最高分结果”或跨操作 latest。

cancel_operation 只提交取消请求。
只有协调者确认在途调用结束，
才能把操作更新为相应停止快照。

新增错误：
- runner.operation_conflict。
- runner.operation_busy。
- runner.checkpoint_corrupt。
- runner.resume_input_changed。
- runner.cache_conflict。
- runner.deadline_exhausted。
- runner.invocation_budget_exhausted。
- runner.worker_timeout。
- runner.worker_interrupted。
- runner.cleanup_unconfirmed。
- runner.clock_uncertain。
- runner.unsupported_runtime。

其中正常预算或取消停止优先通过 partial 快照表达。
无法形成可信快照或操作本身非法时，
才采用公开操作错误返回。

本模块没有已经部署的 r001 数据需要迁移。
实现时应按 r002 契约生成并校验完整 schema。
后续若需要读取旧格式，必须声明支持范围，
不能忽略新增必需状态后声称兼容。

## 2. 单次处理流程

### 2.1 正常准备

读取并固定请求
→ 核验正式数据版本与选择
→ 完整导入或读取来源批次
→ 检查全部 Run 选择及记录处置
→ 检查实际输入关联
→ 保存全部 Subject
→ 构造指标蓝图
→ 准备映射与已有审定输入
→ 执行真实依赖检查
→ 构造 available_inputs 计划
→ 保存计划和引用闭合的 ExperimentManifest
→ 提交 ready PreparationResult
→ 保存回执

Manifest 提交前已有 Provider 结果，
不表示整个实验已经 ready。

### 2.2 正常执行

读取 ready 准备清单
→ 当前授权、完整性和计划复核
→ 保存 ExecutionIntent
→ 按顺序实例化 EvaluationRequest
→ 调用 Evaluator 并保存任务操作结果
→ 处理后续任务
→ 核对全部任务与根槽位
→ 最后提交 ExecutionResult
→ 保存回执

所有派发都引用固定对象。
不存在“观察到低分后换一条 Run 再继续”的分支。

### 2.3 重新评分

修改 evaluator、rubric、参考或来源材料时：
- 创建新的请求或准备修订。
- 保存 derived_from_experiment。
- 保留旧 Run、旧计划和旧 Judgment。
- 明确说明评价基线变化。

正式 A/B 使用相同评价基线。
单独重算一方后不能无说明地与另一方旧评分比较。

相同请求再次 execute_once：
- 创建新的 execution_id。
- 不增加业务 Run。
- 按 single_pass_v1 执行，不自动复用跨 execution 评分。
- 不将重复评分作为新增独立样本。

可恢复执行及显式评分复用使用 C17 的扩展接口，
并遵守 C12–C16。

## 3. 预算、失败与恢复流程

### 3.1 正常推进

固定操作绑定
→ 取得协调权
→ 还原检查点及预算
→ 检查取消、deadline 和调用配额
→ 实例化真实请求
→ 检查精确缓存或已提交结果
→ 必要时保存 CallIntent
→ 启动受控 worker
→ 核验 Provider 结果
→ 提交检查点
→ 释放前置结果给后继
→ 提交完整或 partial 结果快照

缓存命中、真实调用和恢复发现分别记录。
任何路径都不能省略原定根结果槽位。

### 3.2 失败分流

合法 Judgment：
- 作为真实结果保存。
- 无论业务分数和 status 如何，操作可以完成。

没有 Judgment 的已知调用失败：
- 先发现已有提交。
- 再按冻结条件重试。
- 不具备条件则保留操作失败。

结果未知：
- 保留调用意图和未知原因。
- 确认旧 worker 结束。
- 只有明确允许时才重做本地操作。

权限或完整性失败：
- 停止新派发。
- 不通过缓存、换版本或跳过引用规避。
- 在仍能合法保存时提交 partial 快照。

预算或取消停止：
- 不启动新调用。
- 处理在途 worker。
- 提交已知结果和未完成范围。

### 3.3 依赖变化后的恢复

某个失败前置任务在恢复中得到结果时：
- 重新实例化其后继实际请求。
- 比较 task_input_revision。
- 输入未变则继续使用原选择。
- 输入变化则记录替代关系，
  重新检查受影响后继的缓存或评分。

不会重跑无关任务。
不会删除旧结果。
不会把新增评分作为新增业务采样。

### 3.4 提交与恢复的限制

恢复依赖：
- Runner 本地协调记录。
- 已知固定引用。
- 当前授权。
- Provider 的公开读取与不可变提交能力。

不通过读取私有目录绕过模块边界。
无法发现的已提交产物明确保留为未知，
不伪装成已知未执行。

检查点、Provider 结果、最终快照和回执
分别具有提交边界。
不宣称它们构成跨文件、跨模块原子事务。

## 4. 安全与实现安排

本轮允许依赖：
- Artifact Store / Contracts r002。
- Review / Contracts r001。
- Dataset Registry / Contracts r002。
- Adapters / Contracts r001。
- Evidence Mapper / Contracts r001。
- Evaluators / Contracts r001。
- Metric Planner / Contracts r001。

不反向导入 CLI 或 Reporting。
Diagnostics 在本轮不注入、不调用。

计划入口：
- rag_eval.runner.contracts。
- rag_eval.runner.api。

内部职责可以分为：
- 请求与观测校验。
- 来源读取及 Run 选择。
- 实际输入关联。
- Provider 转换及准备。
- 计划执行。
- 任务与结果集合校验。
- 不可变产物提交。

私有文件和类由实现阶段决定。

全部产物通过 Store 保存，
不直接修改提供方文件、索引或私有目录。
本轮不建立共享可变 JSONL 结果表。

Runner 自有协调目录只保存操作定位、固定引用、
事件发现信息、worker 交接和取消请求。
它不成为业务数据或评分的第二套权威正文。

路径和锁遵守 C16。
worker 构造配置使用内置步骤白名单，
不能从外部产物接受可执行路径。

当前访问上下文由可信入口传递。
恢复使用当前授权重新读取，
不依赖旧 worker 留下的权限声明。

StoreAccessContext 来自可信组合入口。
不能从 Run、TestCase 或审定材料接受权限声明。

业务 access_scope 与测评进程权限分别核验。
能够读取参考不表示被测系统用户有权获得该内容。

普通系统输入投影始终排除：
reference、gold、审定材料、评分规则和实验结果。

本轮没有在线发送行为，
但仍保持这一输入边界，
避免后续接入时改变数据含义。

日志只保存必要身份、阶段、状态及安全原因。
原答案、上下文和审定说明通过授权引用回查。

依赖实现及环境版本记录在准备与执行产物中。
本轮未验证性能目标或生产部署行为。

## 5. 验证计划

以下为计划验收，尚未执行：

| 场景 | 应观察到的结果 |
|---|---|
| 数据版本、A/B 和运行均完整 | 形成全部观测及 ready 准备清单 |
| 某方缺少 Run | 保留 null Run Subject 和根结果槽位 |
| 来源文件有非法记录 | 导入失败，不提交有效子集冒充完整批次 |
| 一个观测存在两个候选且未选择 | 准备冲突，不自动选最新或成功项 |
| 相同执行被重复导入 | 不增加独立执行数量 |
| 同一执行占用多个独立采样槽位 | 阻止准备完成 |
| 来源执行身份未知 | 保留独立性限制 |
| query 相同但 history 或 as_of 不同 | 保留真实输入冲突 |
| observed_input 未记录 | unknown，不从用例补写 logged 输入 |
| 声明配置与实际配置不一致 | 保留并处理差异，不覆盖来源事实 |
| 缺上下文、参考或标签 | 保留诊断计划和依赖缺口 |
| Review approved，但业务标签 fail | 不转满分 |
| Review 对象或 rubric 过期 | 保留排除原因，不迁移到新视图 |
| 审定来源读取到一半失败 | 不提交已读子集 |
| 只有 catalog_only 计划 | 不允许 execute_once |
| 前置指标 value=0 | 后续任务仍执行 |
| 前置产生 error Judgment | 操作 completed，后续可消费该状态 |
| 前置操作失败且无 Judgment | 后续保留缺口，不伪造组件 |
| TaskSuccess 有真实已知失败 | 不被其他未知维度阻止形成 fail |
| 所有任务形成非 ok Judgment | execution 仍可 completed，质量覆盖另算 |
| 一个任务没有 Judgment | partial，保留全部槽位 |
| 辅助指标被多个根依赖 | 不重复计独立样本 |
| 执行后只重算一方评价基线 | 不无说明地进行正式配对比较 |
| 保存结果清单失败 | 明确操作失败，不返回虚假结果引用 |
| 当前权限失效 | 停止新派发，不使用缓存或旧回执绕过 |
| 历史来源成本与本次评分耗时 | 分开记录，不重复增加业务成本 |

验证使用真实本地 Store 和已确认提供方。
端到端场景在后续集成设计中另行确认。

补充恢复与执行控制验收：

| 场景 | 应观察到的结果 |
|---|---|
| 相同 operation_key 和绑定重复调用 | 继续同一操作，不创建重复执行 |
| 相同 key 绑定不同请求 | operation_conflict |
| 两个进程同时恢复同一操作 | 一个持锁，另一个限时等待或 operation_busy |
| 协调者退出，worker 尚在运行 | 受控作业终止 worker，不产生孤立继续计算 |
| 单次调用超过 timeout | 停止 worker，保留真实提交或未知结果 |
| 无法确认 worker 已结束 | cleanup_unconfirmed，不启动替代 worker |
| 恢复时原 deadline 已耗尽 | 不重置时间，不派发新调用 |
| 恢复时调用配额已耗尽 | 不因进程重启退还次数 |
| 时钟回退导致期限无法判断 | clock_uncertain，不扩大可执行时间 |
| 业务 fail 或数值为零 | 不触发重试 |
| 缺标签、歧义或审定冲突 | 不通过重试制造分数 |
| 已提交 error Judgment | 同一输入不自动循环重试 |
| 明确临时操作故障且策略允许 | 有限退避和有限次数重试 |
| 调用意图存在但结果未记录 | outcome_unknown，不当作未执行 |
| 未允许未知结果重试 | 保留未知，不再次调用 |
| Store 事件已提交，本地指针未提交 | 通过已知候选引用核验恢复 |
| Provider 返回后协调者退出 | 从合法 outbox 恢复结果 |
| Provider 提交后、outbox 前退出 | 保留未知，不承诺发现隐藏结果 |
| 最终结果提交后回执失败 | 发现并复用结果，生成独立回执 |
| 事件链断裂或同序号冲突 | checkpoint_corrupt，保留现场 |
| 输入、包、来源或版本变化 | resume_input_changed，不自动改绑 |
| 缓存请求只有 query 相同 | 不命中 |
| 缓存缺少相同实际组件引用 | 不命中 |
| 缓存内容撤权或损坏 | 明确失败，不当普通 miss 绕过 |
| 同请求缓存出现冲突结果 | cache_conflict，不取最高分 |
| 缓存命中 | 不产生虚假 evaluation_attempt |
| 前置结果补齐改变下游请求 | 只重算受影响后继，保留旧结果 |
| completed 快照包含低分 | resume 不为提高分数再次调用 |
| 取消发生在退避或在途调用中 | 停止新派发并记录完整未完成范围 |
| partial 后重复恢复 | 原策略、预算和 attempt 连续 |
| 不同 execution 重复评分 | 不增加业务 Run 和独立样本 |

使用临时本地目录、真实 Store 和规范小型材料，
通过进程中断、提交边界故障注入及多进程竞争验证。

时钟、worker 启停和提交边界提供私有可替换测试接缝。
不需要生产模型或业务网络服务。

Windows Job Object、文件锁、刷新和原子提交行为
必须在目标 Windows 本地 NTFS 环境验证。
这些是计划验收，尚未执行。

应用测试、故障注入、性能基准和集成验收尚未执行。

## 6. 设计进度

进度摘要：Confirmed / r003（包 r020）；业务单元修订仍按下表记录。

| 设计单元 | 审阅状态 | 内容修订 | 位置 |
|---|---|---|---|
| 职责与边界 | Confirmed | r002 | 第 1 节 |
| 实验、来源及准备契约 | Confirmed | r002 | Contracts / C1–C6 |
| 执行与结果契约 | Confirmed | r002 | Contracts / C7–C11 |
| 预算、操作与评分复用契约 | Confirmed | r002 | Contracts / C12–C14 |
| 重试、超时及取消 | Confirmed | r002 | Contracts / C15 |
| 本地保存与恢复 | Confirmed | r002 | Contracts / C16 |
| 扩展接口与状态 | Confirmed | r002 | Contracts / C17 |
| 基础处理流程 | Confirmed | r002 | 第 2 节 |
| 失败与恢复流程 | Confirmed | r002 | 第 3 节 |
| 安全与实现安排 | Confirmed | r002 | 第 4 节 |
| 验证计划 | Confirmed | r002 | 第 5 节 |

Complete 只覆盖本文件声明的 C2 本地范围。
应用实现与验收尚未执行。
C2 的当前就绪状态见[索引](../../index.md#5-候选推进范围)，
确切实现基线见[交接 C2-H001](../../handoffs/c2-r001.md)。

后续范围：
- 在线业务系统和模型裁判调用。
- Oracle 及高级诊断。
- 多 worker 并行和分布式调度。
- 全局评分缓存搜索。
- 远端结果未知下的幂等与计费协调。
- CPU、内存硬配额和整机断电持久性。
- 远端存储与其他文件系统支持。

这些范围不作为当前 C2 实现的隐藏依赖。

公开接口、观测和运行选择、任务身份、
依赖方向、持久化、权限、关键状态及验收行为变化需要确认。

私有辅助函数、内部类和契约保持的组织方式，
由实现阶段决定。
