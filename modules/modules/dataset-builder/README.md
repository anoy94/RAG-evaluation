# Dataset Builder：数据集生成与可审计发布

- 模块 ID：dataset-builder。
- 文档状态：Confirmed；概述与契约修订：r001。
- 文档内容修订：r001；详设覆盖：Complete，仅限本包声明范围。
- 设计包修订：r021；候选：C1。
- 确认依据：2026-11-10 用户授权代理连续完成设计与写入；未逐节人工复审。
- 本包是工程设计，不表示应用实现、模型接入或真实数据验收已经完成。

## 1. 职责、依据与边界

依据为 [v2](../../../doc/RAG测评设计_v2.md)第 4–6、11、16–18 节、
[生成详设](../../../doc/RAG测评设计_v2_数据集生成详细设计.md)、
[总体架构](../../00-system-architecture.md)正文 r001、
[目录规则](../../01-directory-layout.md)正文 r001、
[横切规则](../../02-cross-cutting-concerns.md)正文 r001。
输入详设中的示例不是可直接执行的生产 schema；提供方契约决定正式结构。

Builder 拥有构建配置、源处理记录、锚点/事实候选、关系、蓝图、
采样计划、题目候选、求解/验证结果、构建作业和发布投影。
Store 拥有字节与 SourceArtifact，Registry 拥有正式核心对象及发布，
Review 拥有审定记录。Builder 不建立第二套正式 gold，不拥有评测 Judgment。

允许业务依赖仅为：
[Artifact Store Contracts r002](../artifact-store/README.md#contracts)、
[Review Contracts r001](../review/README.md#contracts)、
[Dataset Registry Contracts r002](../dataset-registry/README.md#contracts)。
不导入实验 Runner、Adapter、Mapper、Evaluator、Reporting 或 Diagnostics。
模型与参考提取客户端位于本模块内部，并非被测系统 Adapter。

默认交付为本地文件、一个协调者与一个在途 worker，Windows NTFS、Python 3.11+。
设计覆盖原始引用冻结、参考表示接收、事实/蓝图、合成/独立求解、
负例/视觉证据、质检、分组切分、人工文件审定、正式发布和重启恢复。
不建设 Web 标注平台、图数据库、通用 DAG、分布式队列或自动运行被测系统。

核心产物流如下；箭头表示流程，不新增模块导入依赖。

```mermaid
flowchart LR
  S["冻结源/参考"] --> E["事实与证据候选"]
  E --> B["蓝图与分组采样"]
  B --> Q["合成与独立求解"]
  Q --> V["质检与复核"]
  V --> P["正式对象投影"]
  P --> R["Registry 审定与发布"]
  R --> C["C2 离线测评"]
```

## Contracts

契约状态：Confirmed；契约修订：r001。
Builder 记录使用 schema_version=2.0、contract_profile=dataset-builder.c1.v1。
字段详细规则由以下 C1–C14 与五份主题正文共同定义；消费者只从 contracts/api 导入。

### C1. 公共表示、身份与提交

所有 Builder DTO 拒绝未知字段、重复 JSON 键、BOM、NaN、Infinity。
枚举区分大小写；ID 是非空不透明字符串，不接受 latest；
日期采用明确业务日期或带时区时间，操作时间为 UTC。
Decimal 编码为有限十进制字符串，布尔值不作为数值。

ContentRef 是 Store 的 ArtifactRef 原类型，不复制定义。
正式 SnapshotKey、EvidenceKey、CaseKey、DatasetVersionRef 直接使用 Registry 类型。
BuilderKey={object_id, revision}；revision 为排除自身 revision 后规范内容的 SHA-256。
artifact 引用不嵌入自身正文。内容记录与操作回执分离。

工作对象共有字段：
key、kind、dependency_refs（有序具名引用）、producer（实现/规则版本）、
payload（对应 C4–C8 的强类型内容）、limitations（稳定代码）。
kind 仅取 source_bundle、reference_bundle、anchor、fact、relation、blueprint、
scenario、candidate、solution、validation、selection、projection、build_snapshot。
禁止任意自由 JSON 取代各 kind 的校验。

StageResultRef={manifest_ref, receipt_ref}。
manifest 固定请求、全部输入/输出、数量、排除原因、实现版本及内容摘要；
子产物先提交、manifest 最后提交，回执独立。
回执失败明确暴露已提交阶段，不能声称没有产生结果。

### C2. BuildConfig

固定配置至少包含：

| 字段 | 规则 |
|---|---|
| build_namespace、dataset_id | 非敏感命名空间；一个构建的逻辑归属 |
| source_manifest_ref、source_policy_ref | 明确原始源与业务授权、时间及文档族依据 |
| mode | generate 或 curated_import；前者可以调用模型，后者只接收固定候选/提取物 |
| target_profile | c2_text_compatible_v1 或 external_evidence_v1 |
| extraction_policy_ref | 固定提取 profile、表示/页/区域限额和质量要求 |
| scenario_policy_ref、split_policy_ref | 硬/软配额、关联分组、切分协议、seed 与限额 |
| synthesis_policy_ref、solver_policy_ref | 输入投影、模型、prompt/schema、样本数与独立求解要求 |
| validation_policy_ref、review_policy_ref | 必需检查、适用条件、修复规则、人工与自动 rubric |
| publication_policy_ref | Registry ReleasePolicy、目标等级、兼容要求及硬配额 |
| execution_policy_ref | C11 定义的预算、超时、停止与重试 |
| provider_bindings_ref | 内置具体 provider 配置；秘密仅保存引用 |
| seed、reuse_source_refs | 固定基础 seed；仅明确来源可复用，默认空 |

所有策略均有 policy_id、revision 和不可变内容引用。
plan 时核验存在性、版本、同域及组合合法性；不能先产生付费调用再报配置错误。
无真实业务配额/模型/预算时仍可用 curated_import 设计夹具验证流程，
不能把该模式称为已经实现了真实模型生成。

target_profile：
- c2_text_compatible_v1：可测证据使用 text_quote，数值规则只采用
  Registry scalar_decimal_v1 的 identity/add/subtract。
- external_evidence_v1：允许正式 external 描述符；必须输出 C2 不支持的
  视觉定位/证据指标清单，不能声称与文本 profile 同等可测。
- Builder 内部的复杂运算支持见 C6；不能偷偷写进上述旧 NumericRule。

### C3. 构建请求、计划与快照

BuildRequest={config_ref, optional_seed_bundle_ref, provenance_ref}。
seed bundle 声明 imported/curated/synthetic/online 来源和完整映射；
缺少来源不补造生成历史。原始线上 thread 和个人信息需先获授权。

BuildPlan 固定：
request_ref、全部策略/provider/源版本、阶段表 S0–S10、
已知任务、各批次扩展上限、预期输入缺口、能力兼容报告、
预算可执行性、plan_id、limitations。
动态采样只能在固定策略和上限内产生任务，禁止无界扩图。
计划只读既有引用和描述能力，不执行模型、提取或网络探测。

plan_status=executable 或 blocked；未知来源、未配模型、缺许可或
不支持的发布数值 profile 形成明确 blocker。结构/权限/损坏错误直接失败。
executable 不保证候选通过、质量等级或目标配额可达。

BuildSnapshot 固定 plan_ref、operation_handle、阶段/候选完整清单、
任务 outcome、已用/预留/未知预算、配额进度、分组版本、
validation/decision 选择、publication_refs、未完成/隔离集合。
snapshot_status=complete 或 partial，不能省略未开始或失败的工作。

### C4. 源、参考、事实与关系

SourceManifest、ReferenceBundle、Anchor、FactCandidate、RelationCandidate
的精确字段、不变量和提取协议见[证据构建](evidence.md)第 1–6 节。
每条事实保留实体、期间、单位、否定、条件及 plan/actual/forecast。
工作事实不是已发布 EvidenceUnit；正式投影不得改变这些限定。

模型 confidence 只存诊断，不能决定 source_reviewed 或质量等级。
原文、观测、推导分别记录；提取表示不冒充原文独立真值。

### C5. Blueprint、Scenario 与分组

字段及算法见[蓝图采样](planning.md)第 1–6 节。
Blueprint={key, task_type, constraints, premise_fact_refs, target_spec,
derivation_dag, sufficient_path_candidates, relationship_refs,
required_input_slots, allowed_rules, access_time_context, modality_requirements,
forbidden_assumptions, provenance_edges}。
Scenario={key, blueprint_ref, split_group, split, quota_cell_ids,
style, interaction, variant_policy, selection_receipt_ref}。

expected_action 只使用 Registry 的六种值（含 null），不新造 action。
切分先于问句变体；跨文档依赖、对照、会话和同蓝图产生关联组。
组只是相关性记录，不等于统计独立性认证。

### C6. 题目、独立求解与推导

QuestionCandidate={key, scenario_ref, input（Registry CaseInput）,
interpreted_constraints, declared_fact_refs, generation_call_ref,
base_candidate_ref, variant_kind, unresolved_items}。
不接受生成器写入 quality_tier、正式 split 或审定结论。
修改语义产生新任务 ID；只改表达则新 revision 或明确 variant。

Solution={key, candidate_ref, actual_solver_input_ref, interpreted_constraints,
answerability, expected_action, reference_claims, required_answer_claim_ids,
support_paths, derivation_rules, reference_answers, unresolved_items,
solver_call_ref, independence_limitations}。
所有声明引用当前固定证据，不能将通用知识填作来源。
详细投影、独立性和数值规则见[合成与求解](synthesis.md)。

Builder 的 derivation_decimal_v1 是内部验证协议：
identity/add/subtract/multiply/divide/ratio/percent_change，
显式有序操作数、维度/单位、舍入与误差传播，禁止执行代码。
它不是 Registry 的 NumericRule，不被 C2 Evaluator 直接解释。
无法无损投影的派生规则保留候选并阻塞该发布 profile；
需新增 Registry/Evaluator profile 后才能获得对应自动数值评分。
用户明确选择的“人工整体判断”须重建为独立任务 rubric/profile，
不能靠删除规则、补一个总体 pass 自动降级过关。

### C7. ValidationResult 与候选质量

ValidationResult={key, candidate_ref, check_id, check_version,
dependency_refs, applicability, status, severity, reason_code,
safe_reason, evidence_refs, checker_ref, call_ref, limitations}。
applicability=applicable/not_applicable/undetermined；
status 在 applicable 时为 pass/fail/unknown/error，其他情况为 null。
not_applicable 必须给规则依据，不能计入 pass。
severity=blocker/review/advisory。

fail 是内容检查不满足；error 是检查程序失败；unknown 是不足以判定。
候选质量状态、失效及修复规则见[质检与发布](quality-release.md)第 1–3 节。
release_ready 是 Builder 发布资格快照，不表示 Registry 已冻结或门禁通过。

### C8. 审定工作包与发布投影

PublicationSelection 固定完整候选集合、每题目标 tier、split/group、
相关全部 review 批次、排除原因、硬配额及 dedup 检查。
空集合不能伪装成功数据集；每个目标候选都有 selected/excluded/blocked 处置。
SelectionRequest={build_snapshot_ref, candidate_refs, requested_tier_by_candidate,
dataset_key, publication_policy_ref, split_assignment_ref, quality_batch_refs,
review_selection_ref, change_description, predecessor_ref（可空）}。
ReviewRequest={object_selection, rubric_refs, requested_checks,
reviewer_access_context_ref, display_policy_ref}，引用已提交投影中的真实对象。

PublicationProjection={key, build_snapshot_ref, selection_ref,
registry_import_bundle_ref, candidate_to_core_keys_ref,
unsupported_features, validation_summary_ref, limitations}。
Registry 核心 DTO 与 native_jsonl_v1 是唯一正式交换结构。

ReviewPackage={manifest_ref, targets, rubric_refs, source_views_ref,
response_schema_ref, pending_check_counts, access_limitations}。
targets 一律由 Registry.build_review_target 对真实草稿对象生成。
不把 owner=dataset-builder 写入 Review；不扩展 Review r001 的允许 owner。
验证结果转 automated ReviewRecord 必须绑定这些真实目标及全部检查范围。
人工方法只接收外部真实审定，不由 Builder 自动创建。

BuildReleasePreparation={preparation_ref, projection_ref, draft_ref,
registry_preparation_ref（可空）, status（ready/blocked）,
validation_report_ref, blockers, limitations}。
Registry ReleasePreparation 和 grade 才是正式发布依据；
详细流程见 quality-release.md 第 4–7 节。

### C9. 提取与模型 provider 端口

ReferenceExtractor.describe() -> ExtractionCapabilities。
ReferenceExtractor.extract(ExtractionRequest, CallControl) -> ExtractionResult。
ModelProvider.describe() -> ModelCapabilities。
ModelProvider.invoke(ModelInvocation, CallControl) -> ModelResponse。

端口、具体初始实现、错误和兼容要求见 evidence.md 第 2 节及 synthesis.md 第 1 节。
只允许代码注册的 provider ID；不执行配置给定的 Python import、命令或脚本。
能力声明必须与本次实际响应相符，视觉请求不得静默降为文本。
没有身份保证的“不同模型”只声明配置异构，不认证统计独立。

### C10. 公开业务接口

```text
create_builder(store, registry, review, extractor_registry, model_registry,
               authorization_provider, local_config) -> DatasetBuilder
plan_build(request_ref, access_context) -> BuildPlanRef
execute_build(plan_ref, operation_key, access_context) -> BuildAdvanceResult
get_build(operation_key, operation_kind, access_context) -> BuildOperationSnapshot
resume_build(operation_handle, access_context) -> BuildAdvanceResult
cancel_build(operation_handle, reason, access_context) -> CancellationReceipt
reconcile_invocation(operation_handle, resolution_request_ref, access_context)
    -> ReconciliationResult
revise_candidates(change_request_ref, access_context) -> CandidateRevisionResult
prepare_publication(selection_request_ref, operation_key, access_context)
    -> BuildReleasePreparation
create_review_package(projection_ref, review_request_ref, access_context)
    -> ReviewPackage
export_review_package(package_ref, output_root, access_context) -> ReviewExportReceipt
publish_dataset(build_preparation_ref, operation_key, access_context)
    -> BuildPublicationResult
load_build(snapshot_ref, access_context) -> BuildReader
```

所有 Ref 结果由 result_ref（或具名 manifest_ref）及 receipt_ref 组成；
BuildAdvanceResult={operation_handle, operation_snapshot_ref,
build_snapshot_ref（可空）, receipt_ref}。
BuildPublicationResult={version_ref（Registry 原类型）, build_publication_ref,
registry_receipt_ref, receipt_ref}。
ReconciliationResult={resolution_ref, operation_snapshot_ref, receipt_ref}；
ReviewExportReceipt={package_ref, output_manifest_ref, safe_target,
file_count, byte_count, completion_stage, receipt_ref}。
Reader 只读完整来源、候选/阶段/验证/选择/预算及限制，不能触发生成。
change request 明确 parent refs、修改字段、理由及新输入，不覆盖历史。

prepare_publication 只冻结投影、导入与发布检查，不调用生成模型或自动 publish。
create_review_package 不授予 read 权限，不建立分派平台，不冒称获得人工结果。
publish_dataset 只接受 ready 准备，重新核验固定依赖并调用 Registry.publish；
回执失败按已提交阶段恢复，不重跑合成。

### C11. 执行策略与预算

local_config={root, mode=open_existing|create_new, lock_timeout_ms=5000,
allowed_export_roots（默认空，绑定访问域）}。
root 与 Store/Registry/Runner 根不相同、不互相包含；不接受符号链接/重解析点。
普通构造、help、plan 不创建工作区。初始化冲突不覆盖。

ExecutionPolicy 明确：
deadline、task_timeout、cleanup_timeout、每阶段最大请求/修复次数、
candidate_limit、batch_limit、provider rate limits、各预算池和计价版本。
generate 模式必填请求数、输入/输出 token 及视觉像素/图像数上限。
未配成本估计或响应计量时成本为 unknown；不声称金额硬上限。
默认不自动网络重试；仅固定策略允许的明确可重试错误可重试。
外部结果未知停止该分支，人工/显式 reconcile 后才能新调用，禁止静默重发。

构建作业、候选质量和 Registry 生命周期分别保存。
执行、取消、缓存、远端未决、任务扩展及 NTFS 提交见[执行与恢复](execution.md)。

### C12. 错误与业务阻塞

公开错误均带 owner=dataset-builder、phase、reason_code、
category、safe_message、关联引用和原始 provider 原因，禁止敏感正文。

| reason_code 后缀 | 含义 |
|---|---|
| invalid_input / unsupported_schema | 配置、输入及版本非法 |
| capability_missing / unsupported_publish_profile | 缺阶段能力或无法无损转换到正式契约 |
| reference_invalid / integrity_mismatch | 身份、绑定、计数或字节错误 |
| access_denied / egress_denied | 当前存储或模型出域未获授权 |
| limit_exceeded / operation_conflict / operation_busy | 超限、同键异输入或活跃协调者冲突 |
| provider_failure / provider_result_unknown | 外部失败或可能执行但结果未知 |
| checkpoint_corrupt / resume_input_changed / clock_uncertain | 恢复不可信 |
| worker_timeout / cleanup_unconfirmed | worker 超时或无法证实清理完成 |
| storage_failure / publication_conflict | 持久化或发布身份冲突 |

上述后缀统一加 dataset-builder. 前缀。
合法但配额不可达、审定不足、分歧、未知负例、不支持特征
作为业务 blockers 保存，不与 I/O/权限失败混淆。
质量 reject/unknown 也不是程序异常；不能以 catch-all 置空再发布。

### C13. 安全与兼容不变量

逐次核验当前进程授权；模拟用户 access_scope、审定主体、模型出域许可分开。
源材料和模型输出作为数据，不具有控制指令权限。
模型只能收到已冻结的最小输入视图；动态工具和任意 URL 抓取默认禁用。
权限题不向无权用户或模型发送隐藏答案。
日志只含 ID、阶段、计数、安全原因，正文留在 Store。

现有 C2 核心契约、调用行为和交接保持不变。
Builder 新记录不能混入旧 TestCase/EvidenceUnit 未知字段；
额外来源、视觉描述、推导和验证数据通过现有 provenance_ref、
descriptor_ref、Review assessment 与冻结 rubric 关联。
不把 generated/synthetic_verified 等工作标签写成 Registry 的新质量枚举。

### C14. 验证、文件职责与进度

代码映射 rag_eval.dataset_builder；公开 contracts.py/api.py；
内部职责为 evidence、planning、synthesis、validation、publication、execution、
provider adapters；名称可调整，不创建五个必须公开的子服务。

| 设计单元 | 状态 / 修订 | 唯一正文 |
|---|---|---|
| 职责、公开接口与 DTO | Confirmed / r001 | 本 README 第 1 节、Contracts C1–C14 |
| 源与参考证据 | Confirmed / r001 | [evidence.md](evidence.md) |
| 蓝图、配额与分组 | Confirmed / r001 | [planning.md](planning.md) |
| 合成、独立求解与数值 | Confirmed / r001 | [synthesis.md](synthesis.md) |
| 验证、审定与正式发布 | Confirmed / r001 | [quality-release.md](quality-release.md) |
| 执行、预算与恢复 | Confirmed / r001 | [execution.md](execution.md) |
| C1 CLI、集成与验收 | Confirmed / r001 | [切片](../../slices/c1-dataset-generation.md) |

Complete 表示声明范围的工程设计完整，不表示全格式 parser 或模型已接入。
各主题验证表与切片 G01–G32 都是待执行计划。
真实业务源文、模型服务、人工和配额属于运行配置，不在本次设计中伪造。
需要更高保证的身份认证、更多规则/模态、远端后端时单独扩展契约。
