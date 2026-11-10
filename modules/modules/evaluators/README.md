# Evaluators：指标契约、确定性评价与审定评分导入

- 模块 ID：evaluators
- 文档状态：Confirmed
- 文档内容修订：r003
- 修订边界：本次仅进度摘要更新；业务单元及公开契约保持各自既有修订。
- 概述状态：Confirmed / r001
- 详设覆盖：Complete
- 声明范围：C2 单次评价、确定性指标、已有审定判断导入及组合评价。
- 在线模型裁判、自动主张提取、多模态评价和高级诊断：后续范围。

## 1. 职责与边界

依据：
- [总体架构 r001](../../00-system-architecture.md)。
- [目录设计 r001](../../01-directory-layout.md)。
- [横切规则 r001](../../02-cross-cutting-concerns.md)。
- [Artifact Store 契约 r002](../artifact-store/README.md#contracts)。
- [Dataset Registry 契约 r002](../dataset-registry/README.md#contracts)。
- [Adapters 契约 r001](../adapters/README.md#contracts)。
- [Evidence Mapper 契约 r001](../evidence-mapper/README.md#contracts)。
- [v2 设计](../../../doc/RAG测评设计_v2.md)，第 8、9、11、14 节。

本模块拥有：
- MetricSpec、MetricInstance 和 EvaluatorPackage。
- 评价对象、评分视图及已有判断的导入契约。
- 逐个评价对象的 Judgment。
- 确定性数值、行为、运行完成和计量评价。
- 基于已有审定材料的正确性、完整性与忠实度评价。
- 已审定充分证据组的事实覆盖与完整性计算。
- 固定任务 rubric 下的 TaskSuccess 组合。
- 评分依据、未知原因、版本和不可变保存。

本模块不负责：
- 生成参考答案、充分组或新的黄金证据。
- 选择最优 Run、attempt 或 repetition。
- 调用被测系统、模型裁判或外部工具。
- 调度评价 DAG、控制实验预算和重试。
- 跨用例聚合、配对统计、置信区间和发布门禁。
- 将人工声明提升为经过身份认证的人工审定。
- 根据 Store 可读权限认定业务访问合法。

本轮采用两类评分来源：
1. 固定版本的确定性规则。
2. 与真实对象、rubric 和内容视图绑定的已有审定判断。

缺少自由文本判断来源时，对应指标保留不可测状态。
不使用字符串相似度、任意数字搜索或 Mapper 命中代替语义判断。

## Contracts

契约状态：Confirmed。
契约修订：r001。
schema_version：2.0。
contract_profile：evaluators.c2.v1。

### C1. 评价包与版本身份

EvaluatorPackage 是不可变的评价配置产物，包含：
- package_id、package_revision。
- MetricSpec 列表。
- MetricInstance 列表。
- evaluator 实现身份与版本。
- 固定任务 rubric 及评分 rubric 引用。
- response_contracts：数值等确定性输出格式。
- dimension_bindings：任务维度到指标实例的映射。
- assessment_policy：审定来源、必需检查项及冲突处理规则。
- reference_policy：各指标要求的参考审定范围。
- normalizer、extractor、转换器及映射方法版本。
- limits。
- package_digest。

package_digest 为规范内容排除自身摘要字段后的 SHA-256。
操作时间、操作主体和回执不参与该摘要。

本轮内置指标 version 为 1.0.0。
每个指标仍必须声明具体 profile，不能仅凭显示名称识别口径。

以下变化产生新的指标版本或 profile：
- 评价单位、分母或标签含义变化。
- applicability、unknown 或失败处理变化。
- 允许的数值提取及归一化变化。
- rubric、评分来源或裁判方法变化。
- 主张拆分、支持判定或证据聚合语义变化。

实际依赖变化也必须形成不同的评价内容身份。
不能在同一个包身份下替换 rubric 或实现。

assessment_policy 明确：
- 允许的 human、automated 或 unknown 来源。
- 固定的来源方法及裁判版本声明要求。
- 必需通过的审定检查项 ID。
- 允许的 payload profile。
- 完整来源批次集合的选择规则。
- 仲裁材料的适用和替代关系规则。

正式评分默认要求 human 来源声明及完整必需检查项。
使用 automated 来源时采用单独的 imported_automated profile，
保存原裁判版本与校准限制，不调用该裁判重新评分。

unknown 方法只用于明确标记的诊断 profile。
不把 unknown 或 automated 记录计作人工审定。

human 是来源声明。
本轮不提供签名认证或独立审定保证。

reference_policy 默认要求实际消费的参考范围具有有效审定。
可定义 declared_reference 诊断 profile，
但必须与正式 reviewed_reference profile 分开报告。
不能在缺少审定时静默降级后沿用正式指标名称。

业务门禁阈值由 Reporting 的 GatePolicy 拥有。
本包只保存逐项判定需要的规则、
任务维度通过条件及其来源。

### C2. MetricSpec 与 MetricInstance

MetricSpec 包含：

| 字段 | 含义 |
|---|---|
| metric_id、version、profile | 指标语义身份 |
| evaluation_unit | query_run、numeric_rule、fact_scope 或 rubric_dimension |
| target_kind | run、final_output、generation_call 或 stage_candidates |
| requires | 类型化数据与前置指标依赖 |
| applicability | 固定的适用条件 |
| value_kind | decimal 或 category |
| value_domain | 数值范围或类别全集 |
| unit | ratio、binary、ms、token、currency 等 |
| direction | higher_better、lower_better 或 descriptive |
| denominator_rule | 分母及去重规则 |
| aggregation | 允许 Reporting 使用的聚合方式 |
| threshold_source | 逐项规则来源；没有则为 null |
| implementation_ref | 固定实现与版本 |
| judgment_source | deterministic 或 imported_assessment |
| unknown_policy | 本契约规定的非 ok 状态规则 |
| compatibility_key | 比较时必须一致的口径字段集合 |

requires 使用有限声明结构：
- all_of。
- any_of。
- 固定字段可用性检查。
- 固定引用、版本和作用域一致性检查。
- Registry 审定范围检查。
- Capture 的 fidelity、completeness 和绑定检查。
- 指定前置 MetricInstance 的 Judgment。

允许检查的字段与谓词由本 profile 枚举。
不接受任意表达式、代码、模板或动态插件路径。

MetricInstance 包含：
- instance_id。
- MetricSpec 的完整身份。
- 参数。
- 固定 target_selector。
- 适用 rubric、response_contract 和来源 policy。
- 必需前置实例。
- instance_digest。

例如：
- NumericCorrectness 对应一个明确 rule_id 和输出位置。
- EvidenceCoverage 对应一个明确阶段实例及 top-K。
- Faithfulness 对应一个明确生成调用的实际响应。
- RubricDimension 对应一个固定 dimension_id。

不同阶段、调用、数值规则和计量 scope 使用不同实例。
不得将它们按相同显示名称合并。

包内实例依赖必须无环。
TaskSuccess 只能消费显式登记的前置实例，
不得反向依赖自身或由自身推导出的状态。

适用性按冻结规则确定。
依赖缺失不能使适用指标变为 not_applicable。
无法判定适用性时保留 undetermined，
Reporting 将其作为覆盖缺口保留。

“已核验输出没有事实主张”可以使 Faithfulness 不适用。
该条件必须有完整输出及有效主张清单支持，
不能仅因没有抽取结果或系统回答失败而触发。

### C3. EvaluationSubject 与请求

EvaluationSubject 固定一次评价所对应的业务对象，包含：
- subject_id。
- observation_key：调用方预先冻结的观测槽位身份。
- dataset_version_ref、case_key。
- run_ref：可以为 null。
- package_ref。
- association_report_ref。
- 用例、运行、实际输入和配置的固定依赖引用。
- 已解析的输出及阶段/调用身份摘要。
- limitations。

observation_key 用于保留预期评价集合中缺失的运行。
它不能证明执行独立，也不能替代 source_execution_key。

run_ref 为 null 时：
- 保留该槽位及缺失原因。
- 不构造虚假失败 Run。
- 依赖运行的指标返回 not_computable。
- 不把“日志缺失”解释为“系统已知失败”。

同一真实执行的重复导入或 revision 不构成独立样本。
Judgment 保留 RunKey、source_execution_key、attempt 和 repetition，
供 Runner、Reporting 按冻结规则去重和配对。

完整数据版本、实际输入和系统配置的关联由 Runner 协调验证。
association_report_ref 采用 Evaluator 定义的输入结构，记录：
- 被核对的固定数据、用例和 Run 引用。
- query、history、as_of、access_scope 的核对状态。
- 数据集及配置绑定状态。
- matched、unknown、mismatch 和各自依据。

Evaluator 核对该报告覆盖的引用与真实依赖一致，
并拒绝已知关联冲突。
报告不能复制 Run 的声明后冒充独立核对。

实际输入关联未核验时：
- 可以计算不依赖参考的运行状态和明确 scope 计量。
- 依赖用例真值的正式质量指标保留不可测。
- 明确记录 association_unverified。

subject_id 是上述规范内容排除自身 ID 后的摘要。
不包含 Assessment、Judgment、操作时间或回执。

SubjectRequest 包含创建 Subject 所需的固定引用。
EvaluationRequest 包含：
- subject_ref。
- metric_instance_id。
- mapping_batch_refs。
- assessment_batch_refs。
- component_judgment_refs。

没有所需映射、审定或前置评分时使用空列表，
不以空结果冒充已经执行的检查。

每个 EvaluationRequest 只请求一个指标实例的一个 Judgment。
批量调度和结果集合完整性由 Runner 拥有。

### C4. 评分视图与审定绑定

ScoringRequest 固定：
- subject_ref。
- assessment_scope。
- 需要审查的指标实例。
- 对应 rubric_ref。

assessment_scope 可以是：
- 完整最终输出。
- 指定生成调用的响应及实际输入。
- 指定阶段的必要事实覆盖。
- 固定任务维度。

ScoringView 由 Evaluator 从真实对象构造，包含：
- 固定 Subject 与评价包。
- 选定 scope、指标实例及 rubric。
- 实际输出、生成请求或阶段产物。
- 捕获状态、完整性和 fidelity。
- 参考主张、任务要求和实际相关审定范围。
- Mapper 结果及其限制。
- 评分方法要求和依赖版本。

视图区分：
- reference_baseline：参考事实和任务要求。
- observed_inputs：真实发送给本次生成调用的输入。
- observed_output：本次实际输出。

参考资料可用于正确性评价，
不能因出现在评分视图中就被视为实际生成输入。

视图排除：
- 本次 Assessment 及指向它的链接。
- 本次及由其推导的 Judgment。
- 由本次审定产生的状态。
- 操作时间和回执。
- 指向视图自身的引用或摘要字段。

build_scoring_view 返回 ScoringViewDescriptor：
- owner：evaluators。
- object_type：evaluation_scoring_target。
- object_id：由 Store、访问域、observation_key 和 scope 身份确定。
- object_revision：评分视图内容摘要。
- review_view_version：evaluators-scoring-view-v1。
- review_view_ref。
- rubric_ref。

它是 Evaluator 所有的描述结构。
Runner 将其投影为 Review 所需的目标与 rubric；
Evaluator 不导入 Review 类型。

输出、上下文、参考、rubric 或关键方法版本变化，
都会改变视图 revision。
旧记录不能覆盖新视图。

绑定相同只证明审查对象一致，
不证明标签正确、主张清单完整或审定人身份可信。

### C5. 已有审定结果导入

AssessmentImportEnvelope 由 Runner 协调生成，
是 Evaluator 所有的输入 DTO，包含：
- schema_version、payload_profile。
- converter_id、converter_version。
- 固定 ScoringViewDescriptor 列表。
- 声明完整的来源批次集合。
- 按稳定顺序排列的 entries。
- 来源选择 policy_ref。
- 原始与规范化审定文件、批次清单及回执引用。

每个 entry 包含：
- 来源记录身份及其文件内定位。
- 原始来源引用。
- 对象和 rubric 绑定检查的投影结果。
- decision、方法及必需检查项的投影结果。
- assessment_payload_ref：可为空。
- 已声明仲裁或替代关系。
- 转换限制。

以上为 Review 事实到输入端口的显式投影，
不重新定义 ReviewRecord 的含义。
Runner 调用 Review 的公开接口完成读取与 check_binding，
然后构造本 DTO。

Evaluator 不解析 Review 私有文件，
不依赖 Review 或 Runner 的代码。
它校验自身视图、引用、转换版本、payload 和来源 policy，
保存投影来源，保证后续可以回查转换过程。

本地 DTO 和来源声明不提供密码学认证。
identity_assurance 保留 local_import_only。

assessment_payload 至少包含：
- payload_profile。
- scoring_view_ref、rubric_ref。
- assessment_scope。
- method_profile_ref。
- covered_metric_instances。
- coverage_attestation。
- 对应业务标签与简短依据。
- 实际使用的证据位置及允许规则。
- limitations。

按评分类型允许以下业务内容：

1. output_claims
   - claim_id。
   - 实际输出中的 code point 半开区间。
   - factual 或 not_verifiable 分类。
   - 必要的实体、期间、条件、单位和否定限定。
   - 分类依据。

2. required_claim_results
   - 固定 required_answer_claim_id。
   - satisfied、unsatisfied 或 unknown。
   - 对应实际输出位置与依据。

3. faithfulness_results
   - output_claim_id。
   - supported、contradicted、insufficient 或 unknown。
   - 本次实际生成输入的位置。
   - 联合证据和允许的推导规则。

4. fact_scope_results
   - 固定 fact_id 和目标 scope。
   - 已认可来源及各自覆盖判断。
   - Mapper 引用、实际位置和语义支持判断。
   - 替代来源检查范围及未标注内容。
   - 正向或否定结论的完整性依据。

5. rubric_dimension_results
   - 固定 dimension_id。
   - pass、fail 或 unknown。
   - 完整审查范围、依据与错误类型。

coverage_attestation 明确：
- 审查的实际内容范围。
- 主张清单是否覆盖完整输出。
- 是否检查额外错误主张。
- 所需上下文、来源及任务维度是否完整。
- complete、partial 或 unknown。

该声明仍属于审定材料。
不能把来源声明当作机械证明。

导入规则：
- 对象与 rubric 必须匹配当前真实视图。
- 正式消费的必需检查项必须明确通过。
- approved 表示评分材料获审定，不直接映射成分数 1。
- reject、revise 或 needs_adjudication 不直接映射成任务失败。
- 不同对象或旧 revision 的记录保留排除原因。
- 缺少有效 payload 时不能只根据 decision 生成质量分数。
- 不以最新时间、最高分或多数票自动选择结论。

同一作用域的有效材料存在标签冲突，
或有效记录对材料提出尚未解决的审定异议时，
相关指标为 inconclusive。

仲裁必须是新的不可变材料，
明确引用被处理的记录、争议范围和结论。
只有覆盖全部相关争议且满足固定 policy 的仲裁，
才能解除对应冲突。
旧记录继续保留。

来源集合只覆盖本次显式声明的批次。
不能声称已经检索或排除了系统中所有其他记录。
Runner 应在评分前冻结来源集合，
不能依据分数选择性删除低分或冲突记录。

AssessmentBatch 保存：
- 完整 envelope_ref。
- 被接受和排除的 entry 及原因。
- 未解决冲突。
- 来源、数量、转换版本与 batch_id。

导入结构非法时整个操作失败。
业务绑定不匹配或分歧可以作为批次中的明确结果保留，
不能静默跳过。

### C6. Judgment 与依赖报告

Judgment 包含：
- schema_version、contract_profile。
- judgment_id。
- subject_ref。
- metric_instance_id、metric_spec_ref。
- package_ref。
- resolved_target_scope。
- applicability_state。
- status、value。
- unit、direction。
- unit_counts。
- reason_code。
- dependency_report。
- dependency_refs。
- assessment_refs、mapping_refs。
- component_judgment_refs。
- implementation_ref。
- limitations。
- details_ref：可为空。

status 与 value：

| status | value | 含义 |
|---|---|---|
| ok | 合法 decimal 字符串或类别 | 当前依据足以确定结果 |
| not_applicable | null | 已证明不属于该指标的评价范围 |
| not_computable | null | 缺字段、接口、参考、计量或有效标签 |
| inconclusive | null | 标签冲突、映射歧义或支持判断未决 |
| error | null | 评价执行、解析实现或程序故障 |

TaskSuccess 和其他 pass/fail 指标：
- status=ok 时 value 为 pass 或 fail。
- 未知通过非 ok status 表达，value 为 null。
- 不额外使用数值 -1 或类别 unknown 代替状态。

ratio 指标保留精确 numerator、denominator。
二者为整数，denominator 必须大于零。
对外 decimal 展示最多保留 18 位小数，
使用 ROUND_HALF_EVEN，精确计数仍为依据。

数值正确性中的 Decimal 运算与容差比较不采用上述展示舍入。
TaskSuccess 的通过判断也不比较舍入后的显示值。

unit_counts 标明：
- 评价单位及其全集身份。
- numerator、denominator。
- complete、partial 或 unknown。

主张提取不完整时只保存 partial 计数，
不能将观察到的主张数量当作完整分母。

details 可以保存缺失数据导致的识别范围，
但必须标记 partial_identification_bounds。
它不是最终 value，也不是统计置信区间。

DependencyReport 对每项要求记录：
- requirement_id。
- satisfied、missing、uncertain 或 not_applicable。
- 实际引用和检查范围。
- reason_code。

依赖检查不产生质量分数。
“标签存在”不表示标签为通过。

评价器应按指标逻辑判断是否已有决定性结论。
不能因某个非决定性依赖缺失，
覆盖已经可靠成立的失败或成功结论。

judgment_id 是规范内容排除自身 ID 后的 SHA-256。
内容中不保存操作时间或指向自身的引用。
同一固定输入产生同一 Judgment 内容身份。

### C7. 本轮支持的指标

| metric_id / profile | 单次结果 | 核心条件 |
|---|---|---|
| OperationalCompletion / terminal_status_v1 | 0 或 1 | 有明确来源依据的 Run 终态 |
| ExpectedActionMatch / explicit_action_v1 | 0 或 1 | 已审定 expected_action 与明确实际 action |
| NumericCorrectness / scalar_decimal_v1 | 0 或 1 | 合法数值规则及固定输出提取契约 |
| AnswerCorrectness / full_output_imported_v1 | pass 或 fail | 覆盖完整输出及额外主张的有效审定 |
| AnswerCompleteness / required_claims_v1 | ratio | 全部必要答点及其满足标签 |
| Faithfulness / factual_claims_imported_v1 | ratio | 完整主张清单、实际生成输入及支持标签 |
| EvidenceCoverage / reviewed_sets_text_v1 | ratio | 已审定充分组及可靠事实覆盖 |
| CompleteEvidenceSet / reviewed_sets_text_v1 | 0 或 1 | 同上，按充分组逻辑计算 |
| RubricDimension / imported_dimension_v1 | pass 或 fail | 固定维度的完整有效审定 |
| TaskSuccess / required_dimensions_v1 | pass 或 fail | 固定任务要求及完整维度映射 |
| E2ELatency / explicit_scope_v1 | 毫秒值 | 明确 E2E 边界的实际计量 |
| TTFT / explicit_streaming_scope_v1 | 毫秒值 | 明确流式请求及首内容 token 计量 |
| UsageQuantity / explicit_scope_v1 | 数量或金额 | 明确 scope、单位与计量依据 |

质量指标默认按 query_run 宏聚合。
主张微平均由 Reporting 单独计算和展示。
numeric_rule、fact_scope 和 rubric_dimension 的结果
不能直接混入 query 宏平均。

OperationalCompletion：
- succeeded 为 1。
- failed、timed_out、cancelled 为 0。
- 表示所选运行记录是否完成处理，不表示业务正确。
- 作业取消但没有形成 Run 时不补造 cancelled Run。

ExpectedActionMatch：
- 比较完整标签是否相等。
- 实际 action=other 为已知不匹配。
- action=null 为 not_computable。
- 不从回答措辞推断 abstain、clarify 或 policy_block。
- 不单独证明拒答、澄清或权限阻断质量。

E2E、TTFT：
- 直接消费 Adapter 的明确计量，不使用导入耗时。
- 不累加并行 Span。
- 固定 measurement basis 及 scope；不同口径不能混合比较。
- 默认正式 profile 接受 monotonic、reported。
- estimated 使用单独 profile，并保留限制。
- 未知值不记零。
- streaming=false 时 TTFT 为 not_applicable。
- streaming 未知或首 token 边界不明时不可计算。
- Run 失败时已有耗时仍可保留，并按终态分层报告。

UsageQuantity：
- 每个实例选择明确 usage_id、scope 和 quantity。
- 数量保留非负整数约束。
- 金额保留 Decimal、币种和计价引用。
- reported、observed、estimated 不混作同一口径。
- 同 scope 重复数据必须一致核验。
- 多条无法区分或相互矛盾的数据返回 inconclusive。
- 不自动相加总量与明细。
- 不将已知子集之和称为完整运行成本。
- 没有完整总量时，不生成虚假的总成本。

本轮不提供通用成本补算或重叠 scope 自动合并。

### C8. 数值提取与判定

NumericCorrectness 的一个实例绑定：
- 一个固定 NumericRule。
- 一个明确 output_claim_id。
- 一个预先冻结的 response_contract。
- 一个明确输出目标。
- 输出单位要求。

数值规则直接消费 Registry 的 scalar_decimal_v1：
identity、add、subtract。
不执行任意表达式或代码。

评价前重新检查：
- 操作数身份、数量和顺序。
- 输入和输出单位。
- derived_from 与允许规则。
- expected_value 的重新计算结果。
- 对应 numeric_rules 审定范围。

参考规则矛盾属于参考或输入错误，
不能按错误参考给系统记零。

C2 支持两种输出契约：

1. scalar_with_unit_v1
   - 整个目标字符串为一个有限十进制数和明确单位。
   - 数字与单位分隔规则由契约固定。
   - 可允许固定 ASCII 首尾空白。
   - 不允许在任意自然语言中搜索数字。

2. json_scalar_with_unit_v1
   - 整个目标为合法 JSON。
   - 使用固定 JSON Pointer 选择数值和单位。
   - 数值表示类型及允许格式预先规定。
   - 拒绝重复键、布尔值、NaN 和 Infinity。
   - 不在多个候选值中选择最接近参考的数值。

输出契约必须是任务原有要求，
不能评分后临时施加给历史自然语言答案。

单位按固定标识精确核对。
本轮不隐式转换万元、亿元、百分数、货币或期间。

若已知任务要求固定格式，且取得完整原始输出：
- 空字符串、非法数字或不符合格式为已知失败，value=0。
- 缺少必填值或单位为已知失败。
- 明确输出单位错误为已知失败。

若实际任务允许任意自然语言，
但本轮提取器无法可靠定位数值：
- not_computable / extraction_contract_unavailable。
- 不把提取器能力不足当作系统错误。

输出 not_recorded、withheld 或内容不完整时，
不能据此认定格式失败。

提取合法数值后按以下规则判断：

abs(predicted - expected)
<= max(absolute_tolerance, relative_tolerance * abs(expected))

计算使用精确十进制及足够精度，
不通过 binary float 或隐式舍入实现。

NumericCorrectness 只证明该数值组件满足规则。
它不证明解释完整、引用有效、权限合规或整体任务成功。

### C9. 答案正确性、完整性与忠实度

AnswerCorrectness：

消费 full_output_imported_v1 的有效审定结果，
要求：
- 审查完整实际输出。
- 参考和任务规则版本固定。
- 明确检查输出中的额外错误主张。
- 保留实体、期间、单位、否定及条件。
- 结果包含错误类型和依据。

仅确认几个参考要点正确，
不能生成完整 AnswerCorrectness=pass。

不完整审查为 not_computable；
有效审查中的判断冲突为 inconclusive。

AnswerCompleteness：

分母为 required_answer_claim_ids 去重后的数量。
本轮等权，不额外引入 Registry 未定义的权重字段。

每个必要答点必须有：
- satisfied。
- unsatisfied。
- unknown，或明确缺少标签。

satisfied 数为 S，必要答点总数为 N：
- 全部已知时 value=S/N。
- 存在 unknown 时 value=null，status=inconclusive。
- 缺少所需标签时为 not_computable。
- 可以保存 [S/N, (S+U)/N] 的识别范围。

参考答点本身未完成所需审定时不可计算。
计算前提不自动加入必要答点分母。

必要答点明确为空且任务确实不要求实质答点时不适用。
无法确认是空还是缺失时不可计算。

Faithfulness：

评价单位为一个明确生成调用的实际响应。
必须有：
- 完整、原始响应。
- 完整输出主张清单及有效覆盖声明。
- 实际生成请求和上下文。
- 已核验的 request_bindings。
- 每个事实主张的支持标签。

若评价最终系统输出，
必须证明它与所选生成响应的关系。
postprocessed 或 unknown 输出不能自动继承某次调用的忠实度。

supported 需要实际输入中的支持位置。
可以使用多份实际证据联合和任务允许的推导规则。
不能用整库、其他 Run、参考答案或未发送源文补足支持。

事实主张标签：
- supported：实际输入充分支持。
- contradicted：实际输入与该主张矛盾。
- insufficient：已完成所需检查，但实际输入不足以支持。
- unknown：支持判断未决。

not_verifiable 仅用于定义上不作事实核验的内容。
必须有明确分类依据，
不能把难以判断的事实移出分母。

设全部应核验事实主张数为 N：
- value=supported/N。
- contradicted 和 insufficient 均保留在分母。
- 二者分别保存计数。
- unknown 不从分母删除。
- 技术或语义未知使最终 value 为 null。
- 主张清单不完整时不能计算正式比例。

经完整核验，N=0 时 not_applicable。
纯拒答不自动获得满分。

Faithfulness 不证明源文真实，
也不证明答案完整或响应用户意图。

### C10. 必要事实覆盖与充分组

EvidenceCoverage 和 CompleteEvidenceSet 只消费：
- 当前固定用例中实际已审定充分的证据组。
- 这些组内必要事实的固定来源。
- 明确阶段、调用及 top-K 范围。
- Mapper 的固定结果。
- 与相同实际范围绑定的有效事实支持审定。

human_core 不表示所有替代组都已审定充分。
没有合格充分组时为 not_computable，
不能返回 0 或 1。

组内按 fact_id 去重。
同一事实的重复 chunk 和重复引用不增加分数。
本轮事实等权。

事实覆盖值为 1、0 或 unknown。

覆盖为 1 要求：
- 至少一个 Registry 已认可来源。
- 来源、revision 与目标范围匹配。
- 指定锚点具有可靠映射。
- 有效审定确认实际内容完整提供该事实。
- 实体、期间、条件、单位和否定没有缺失。
- 联合片段均来自当前实际范围。

Mapper.matched 只满足定位条件。
support_assessment=not_assessed 不能直接换算成事实覆盖 1。

覆盖为 0 要求：
- 该事实全部已认可替代来源均被检查。
- 实际目标集合、内容及必要通道完整。
- 对应检查和搜索完整结束。
- 有有效审定确认整个指定范围没有提供该事实。
- 没有未决来源、未决映射或尚未判断的潜在替代内容。

Mapper.verified_absent 只证明指定锚点缺失，
不能单独使事实覆盖为 0。

以下情况保留 unknown：
- 只有锚点定位，没有完整事实支持判断。
- 某个合法替代来源尚未检查。
- 目标存在无法处理的改写。
- 来源身份或调用关联不明。
- 可能支持该事实的内容仍为 unjudged。
- 目标不完整或实际请求绑定未核验。

新来源即使看起来能够支持事实，
也不能在评分时直接加入已认可黄金路径。
保留 unjudged 及依据，后续通过 Registry 新版本处理。

本 profile 对语义改写采用保守处理。
已有审定不能直接改写 Mapper 状态；
缺少所需可靠定位时不输出本 profile 的确定覆盖分数。

对已审定充分组 S：
- F(S) 为不同必要 fact_id。
- cov(f) 为该事实的覆盖值。

全部已知时：

EvidenceCoverage
= max(S) [sum(f in F(S), cov(f)) / |F(S)|]

该指标表示已知充分组中覆盖最好一组的事实覆盖，
不表示完整语料召回或不存在其他充分路径。

存在未知时：
- 将未知分别取 0、1，计算下界和上界。
- 上下界相等时可以输出确定结果。
- 不相等时为 inconclusive，value=null。
- 保存精确计数与 partial_identification_bounds。

CompleteEvidenceSet 使用三值逻辑：
- 组内不同事实为 AND。
- 不同充分组为 OR。
- 任意一组全部为 1，结果为 1。
- 每个组都至少有一个确定 0，结果为 0。
- 其他情况为 unknown，Judgment 为 inconclusive。

某个已知完整组可以使结果为 1，
即使其他替代组仍有未决事实。

阶段范围分别计分：
- retrieve。
- fuse。
- rerank。
- compress。
- generation_context。

实例绑定明确 result_id 或 generation_call_id。
不得把候选命中复制到最终上下文。

top-K 只表示实际 rank 范围。
本轮不重新截断文本或估计 token 预算。
不能将该结果标成已经控制 token 预算的比较。

### C11. TaskSuccess 组合规则

TaskSuccess 必须具备：
- 固定 expected_action。
- 完整 task_policy 及 rubric。
- 对应任务要求的有效审定范围。
- 完整 required_dimensions。
- 固定 dimension_bindings。
- 包级必需约束。

包只能增加约束，不能删除 task_policy 的必需维度。

每个必需维度绑定：
- 唯一维度 ID。
- 一个或多个明确指标实例。
- all_of 或 any_of 组合。
- 通过条件。
- 与 rubric 的对应依据。

若 require_supported_citations=true，
维度映射必须覆盖任务定义的引用有效性和支持要求。
只检查“存在引用标号”不能满足该约束。

allowed_tools、allowed_numeric_operations、
格式、范围和权限要求也必须有对应验证路径。
没有实际观测或有效审定时保留未知，
不能默认合规。

RubricDimension 可以承接已有的完整审定，
但它必须具有该具体维度的固定规则和审查范围。
不能用一个无范围说明的总体 pass 替代缺失维度。

前置 Judgment 必须：
- 属于同一 Subject。
- 使用兼容的 package 与参考版本。
- 对应 dimension_bindings 指定的实例和目标范围。
- 保留真实 Run 和依赖来源。

跨 A/B、跨用例、跨 Run 或错误调用的 Judgment 拒绝使用。

组合顺序：

1. 校验任务基线和维度映射完整性。
2. 判断是否存在已知任务执行失败。
3. 计算各必需维度的 pass、fail、unknown。
4. 形成 TaskSuccess。

已知执行失败：
- 所选 Run 为 failed、timed_out 或 cancelled。
- 或完整实际输出违反任务原有的必需格式。
- 依据必须来自真实运行或有效指标。

在合法任务基线下，已知执行失败产生 fail。
部分答案即使数值正确，也不能覆盖该失败。

维度组合采用三值逻辑：
- 任意必需 all_of 维度确定失败，整体 fail。
- 全部必需维度确定通过，整体 pass。
- 其他情况整体未知。

缺少标签为 not_computable。
未解决语义分歧为 inconclusive。
必需评价程序失败为 error。

若已有可靠失败结论，
其他维度未知不阻止输出 fail，
但仍保存全部未决原因和覆盖缺口。

没有决定性结论时，多种非 ok 原因同时存在，
主状态优先为 error、inconclusive、not_computable，
完整原因均保留。

TaskSuccess 不使用加权平均抵消必需失败。
NumericCorrectness、ExpectedActionMatch 或 Faithfulness
均不能单独替代完整任务要求。

### C12. 公开接口

公开入口：

create_evaluators(
    artifact_store,
    dataset_registry,
    adapters,
    evidence_mapper
) -> Evaluators

describe_metrics(
    package_ref,
    access_context
) -> MetricCatalog

prepare_subject(
    subject_request_ref,
    access_context
) -> SubjectResult

build_scoring_view(
    scoring_request_ref,
    access_context
) -> ScoringViewResult

import_assessments(
    envelope_ref,
    access_context
) -> AssessmentBatchResult

check_dependencies(
    evaluation_request_ref,
    access_context
) -> DependencyReport

evaluate(
    evaluation_request_ref,
    access_context
) -> JudgmentResult

load_judgment(
    judgment_ref,
    access_context
) -> Judgment

MetricCatalog 返回固定指标规格、实例、依赖和兼容条件。
Planner 据此安排任务，
不能在描述阶段调用系统或模型。

check_dependencies：
- 读取真实当前依赖。
- 不产生分数，不修改源对象。
- 不调用其他评价器补齐前置结果。
- 不等于承诺后续实际计算一定成功。

evaluate：
- 重新检查实际依赖和权限。
- 不仅信任 Planner 的历史检查结果。
- 不自动选择其他 Run、来源批次或指标版本。
- 不递归调度前置指标。
- 返回一个固定请求对应的一个 Judgment。

SubjectResult、ScoringViewResult、
AssessmentBatchResult 和 JudgmentResult 均包含：
- 对应不可变内容的 ArtifactRef。
- 本次操作 receipt_ref。

回执记录操作主体、UTC 时间、访问策略 revision、
请求和结果引用及完成阶段。
操作信息不影响内容身份。

### C13. 错误与非 ok 原因

公开操作错误：

| reason_code | 含义 |
|---|---|
| evaluators.invalid_input | 请求、包或参数非法 |
| evaluators.unsupported_schema | 不支持的契约或 profile |
| evaluators.reference_invalid | 对象、作用域或版本引用不一致 |
| evaluators.assessment_invalid | 导入评分材料结构不合法 |
| evaluators.component_mismatch | 前置 Judgment 不属于指定实例或对象 |
| evaluators.limit_exceeded | 输入或处理限额超出 |
| evaluators.access_denied | 当前操作或依赖未授权 |
| evaluators.integrity_mismatch | 内容摘要、计数或引用损坏 |
| evaluators.storage_failure | Store 保存或读取失败 |
| evaluators.internal_error | 评价程序故障 |

业务及测量原因记录在 Judgment 中，包括：
- run_not_recorded。
- association_unverified。
- output_not_recorded。
- output_withheld。
- output_incomplete。
- expected_action_unreviewed。
- reference_scope_unreviewed。
- task_policy_missing。
- dimension_binding_incomplete。
- extraction_contract_unavailable。
- assessment_missing。
- assessment_conflict。
- claim_inventory_incomplete。
- actual_context_unverified。
- mapping_ambiguous。
- fact_support_unknown。
- sufficient_set_unreviewed。
- measurement_missing。
- measurement_scope_ambiguous。
- evaluator_timeout。
- evaluator_execution_failed。

错误处理边界：
- 请求或包本身非法：执行前拒绝，不生成虚假 Judgment。
- 合法请求中的评价程序超时或故障：
  在依赖可读且 Store 可写时保存 error Judgment。
- 按固定任务格式检查出非法系统输出：
  属于该规则下的已知业务失败。
- 解析实现崩溃或无法处理支持范围外的格式：
  属于评价故障或能力不足，不自动记系统失败。
- 权限、完整性或存储失败导致无法形成可信结果：
  操作明确失败，由 Runner 保留未完成槽位。

不能将基础设施异常、未读内容或截断输入转换为零分。
也不能在保存失败后返回虚构的成功结果引用。

## 2. 处理与集成流程

### 2.1 确定性评价

核验请求、包和权限
→ 读取固定 Subject
→ 解析指标实例及真实目标范围
→ 检查实际依赖和适用性
→ 按固定规则提取或读取值
→ 计算结果及完整原因
→ 校验 Judgment
→ 保存 Judgment
→ 保存回执并返回

不在评价过程中重新执行被测系统。

### 2.2 已有审定评分

Evaluator 构造真实评分视图
→ Runner 使用该视图协调 Review 读取与绑定检查
→ Runner 按固定转换版本生成 AssessmentImportEnvelope
→ Evaluator 校验并保存完整导入结果
→ Runner 请求对应指标实例
→ Evaluator 消费业务标签并生成 Judgment

Review 的来源记录、转换过程和最终业务分数分别保存。
来源记录不会被评分结果覆盖。

### 2.3 与 Planner、Runner、Reporting 的边界

Planner：
- 读取 MetricSpec 和实例依赖。
- 安排本来适用的任务。
- 不把声明 capability 当作真实依赖已经存在。

Runner：
- 冻结评价对象集合及 A/B 对齐规则。
- 选择预先规定的 attempt 和 repetition。
- 保留缺失 Run 和未完成评分。
- 调度 Mapper、Review 转换及指标依赖。
- 按冻结策略处理重试、取消和恢复。

Reporting：
- 统计 N_total、N_applicable、N_ok。
- 展示全部非 ok 原因与有效覆盖。
- 计算 query 宏平均和独立主张微平均。
- 展示双方可测集及共同可测对。
- 计算统计区间和门禁。

Evaluator 不因某条记录难以评分而将其从总体集合删除。

本轮没有实现的指标在计划阶段明确显示不支持。
不能用其他同名近似计算冒充已定义指标。

## 3. 持久化、缓存与恢复

所有包、请求、Subject、视图、AssessmentBatch、
Judgment 和回执通过 Artifact Store 保存。

业务源对象保持不变：
- 不修改 DatasetVersion 或参考标签。
- 不修改 RunRecord 或 Mapper 结果。
- 不覆盖原审定记录或旧 Judgment。

单次评价提交顺序：
1. 保存评分明细及所需依据。
2. 保存引用闭合的 Judgment。
3. 保存操作回执。
4. 返回结果。

Judgment 已提交但回执失败时明确报告阶段。
重试可以复用同一 Judgment，生成独立回执。

审定导入提交顺序：
1. 完整校验 envelope。
2. 保存接受、排除及冲突明细。
3. 校验来源数量与结果数量。
4. 最后提交批次清单。
5. 保存回执。

批次清单提交前的单条材料不代表完整导入成功。

评价缓存键至少覆盖：
- Store 和 access_domain。
- Subject 及实际数据、用例、Run。
- source_execution_key 和选定 attempt。
- 指标规格、实例、包和实现版本。
- rubric、response_contract。
- 实际输出、请求、上下文及捕获状态。
- Mapper、主张清单和全部选定 Assessment。
- 前置 Judgment。
- 来源选择及仲裁关系。
- 限额和影响结果的方法配置。

不能仅按 query、answer、case_id 或 metric_id 缓存。

每次读取缓存仍检查当前权限及完整性。
缓存不能绕过撤权。

重试不增加业务执行次数。
无法形成合法完整结果时保留未完成状态，
不以旧版本分数补齐。

新审定、新 rubric 或新实现形成新的评分。
同一 Run 可以重新评分，旧结果继续存在。

并发使用 Store 的不可变写入与协调机制。
本模块不维护共享可变评分表，不并发追加全局 JSONL。

## 4. 安全、限额与实现安排

允许的业务依赖：
- Artifact Store / Contracts r002。
- Dataset Registry / Contracts r002。
- Adapters / Contracts r001。
- Evidence Mapper / Contracts r001。

Review 仅作为集成流程参考：
[审定如何进入评分](../review/README.md#3-结果如何进入数据与评分)。

本模块不导入 Review、Runner、Reporting、Planner
或 Diagnostics 的代码与类型。

计划公开入口：
- rag_eval.evaluators.contracts。
- rag_eval.evaluators.api。

内部可以按实际需要划分：
- 包与实例校验。
- Subject 和评分视图构造。
- 审定输入转换校验及冲突解析。
- 确定性规则。
- 主张与事实聚合。
- 任务维度组合。
- 不可变结果保存。

不预设通用插件框架。
实现类、辅助函数和私有文件由实现阶段决定。

limits 默认：
- max_metric_instances：1000。
- max_assessment_entries：10000。
- max_claims_per_output：10000。
- max_fact_source_results：100000。
- max_single_text_codepoints：2000000。
- max_total_text_codepoints：20000000。
- max_decimal_digits：1000。
- max_decimal_scale：1000。

限额必须为有效非负或正值，具体约束随字段确定。
不截断主张、忽略冲突或减少来源检查后继续输出完整分数。
超限明确失败或产生评价 error，不能变为真实缺失。

Decimal 运算按输入位数及操作需求设置足够精度。
达到实现支持边界时明确报错，
不静默舍入后用于容差判定。

StoreAccessContext 控制测评进程访问。
业务 access_scope 与授权合规由任务规则及实际证据检查。
Store 可读不等于用户有权获得相关信息。

外部答案、rubric 和审定说明均作为数据处理。
不执行其中的指令、表达式、代码或工具调用。

默认日志只输出安全身份、指标、状态及原因。
完整答案、上下文和详细依据放入授权产物。
仅保存短说明与证据位置，不要求隐藏思维链。

本轮没有网络调用、模型部署或数据库迁移。
不承诺吞吐与时延目标；性能基准尚未执行。

## 5. 验证计划

以下为计划验收，尚未执行：

| 场景 | 应观察到的结果 |
|---|---|
| Run succeeded，但答案错误 | Completion=1，正确性和任务成功独立判定 |
| Run timed_out，部分答案数值正确 | 数值组件可单独通过；TaskSuccess=fail |
| 预期槽位没有 Run | 保留槽位，依赖运行的指标不可计算 |
| action 缺失但文本像拒答 | 不自动推断 abstain |
| 数值规则中的减法顺序不同 | 按固定顺序校验，不交换操作数 |
| expected_value 与重算矛盾 | 拒绝非法参考，不给系统记零 |
| 多数字自然语言中存在正确数值 | 不通过任意搜索判正确 |
| 已有任务要求固定 JSON，但实际格式非法 | 对应确定性规则记已知失败 |
| 任务允许自由文本，提取器不支持 | not_computable |
| 单位、期间、负号或数量级错误 | 不通过归一化掩盖 |
| 数值恰好位于容差边界 | 使用精确 Decimal 判断 |
| Review approved，但业务标签为 fail | 保留业务 fail，不转满分 |
| 来源绑定旧视图 | 排除并记录具体原因 |
| 同 scope 存在有效冲突标签 | inconclusive，不取最新或多数 |
| 有完整有效仲裁 | 仅解除被明确处理的争议 |
| 只审查参考要点，未检查额外错误主张 | 不生成完整正确性 pass |
| 推导前提未在答案中复述 | 不自动降低必要答点完整性 |
| 主张清单不完整 | 不输出正式忠实度比例 |
| contradicted 或 insufficient 主张 | 留在忠实度分母，分别计数 |
| 事实主张 unknown | 不从分母删除 |
| 完整核验没有事实主张 | Faithfulness 为 not_applicable |
| 支持仅存在于整库其他材料 | 不作为本次实际上下文支持 |
| 最终输出来自不明后处理 | 不自动继承某个生成调用忠实度 |
| Mapper matched，没有语义支持判断 | 事实覆盖保持 unknown |
| 指定锚点缺失，但有未决替代来源 | 不将事实覆盖记零 |
| 同事实存在多个重复 chunk | 只计一个必要事实 |
| 一组完整、另一组有未知 | CompleteEvidenceSet=1 |
| 所有组都有确定缺失事实 | CompleteEvidenceSet=0 |
| 部分组未审定充分 | 不进入正式充分组集合 |
| 候选命中，最终上下文没有保留 | 两个 scope 分别评价 |
| TaskSuccess 缺少权限或引用必需维度 | 不允许整体通过 |
| 必需维度已知失败，其余未知 | TaskSuccess=fail，同时保留缺口 |
| 仅 NumericCorrectness 通过 | 不生成完整任务成功 |
| 引用目标存在但未判断业务权限 | 不等同引用有效 |
| 非流式请求 | TTFT 不适用 |
| timing 或 usage 未知 | 不填零 |
| 总费用和请求明细重叠 | 不自动相加 |
| 同真实执行重复导入 | 不增加独立样本数或业务成本 |
| 包、裁判方法或提取规则变化 | 新结果身份，不复用旧评分 |
| 缓存存在但当前权限撤销 | 拒绝读取 |
| 超限或程序异常 | 不返回虚假零分或完整成功 |
| Judgment 提交后回执失败 | 重试复用内容，生成独立回执 |

验证通过公开接口，使用真实本地 Store，
接入已确认 Registry、Adapters 和 Mapper 契约。

审定数据使用明确可解释的固定材料。
合成演示材料必须标明演示身份。
评分替身只能验证接口，不作为真实语义质量证据。

应用测试、集成验收和性能基准尚未执行。

## 6. 设计进度与后续边界

进度摘要：Confirmed / r003（包 r022）；业务单元修订仍按下表记录。

| 设计单元 | 审阅状态 | 内容修订 | 位置 |
|---|---|---|---|
| 职责与边界 | Confirmed | r001 | 第 1 节 |
| 指标、对象与评分契约 | Confirmed | r001 | Contracts / C1–C6 |
| 确定性及审定指标 | Confirmed | r001 | Contracts / C7–C11 |
| 公开接口与错误 | Confirmed | r001 | Contracts / C12–C13 |
| 处理及集成边界 | Confirmed | r001 | 第 2 节 |
| 持久化、缓存与恢复 | Confirmed | r001 | 第 3 节 |
| 安全与实现安排 | Confirmed | r001 | 第 4 节 |
| 验证计划 | Confirmed | r001 | 第 5 节 |

Complete 仅指本文件声明的 C2 范围。
应用实现与验收尚未执行。
C2 的当前就绪状态见[索引](../../index.md#5-候选推进范围)，
确切实现基线见[交接 C2-H003](../../handoffs/c2-r003.md)。

后续范围：
- 在线模型裁判及自动主张提取。
- 裁判校准、元评测和自动分流。
- 完整 Citation 指标的自动解析与业务授权解析器。
- 传统 IR 指标的固定评价单元投影及 qrels 协议。
- 阶段保留率、token 预算和压缩专项指标。
- parser、ANN、多模态及安全专项评价。
- 通用计价补算、重叠 usage scope 合并。
- Oracle 与受控干预评价。

需要上述能力时单独确认扩展 profile。
不能沿用现有指标身份替换其语义。

公开接口、数据含义、依赖方向、持久化、
权限、关键状态与验收行为变化需要确认。
私有实现可在已确认边界内决定。
