# Metric Planner：适用性、依赖解析与离线评价计划

- 模块 ID：metric-planner
- 文档状态：Confirmed
- 文档内容修订：r003
- 修订边界：本次仅进度摘要更新；业务单元及公开契约保持各自既有修订。
- 概述状态：Confirmed / r001
- 详设覆盖：Complete
- 声明范围：C2 已导入数据和运行记录的评价计划。
- 在线执行计划、通用工作流及自适应指标选择：后续范围。

## 1. 职责与边界

依据：
- [总体架构 r001](../../00-system-architecture.md)。
- [目录设计 r001](../../01-directory-layout.md)。
- [横切规则 r001](../../02-cross-cutting-concerns.md)。
- [Dataset Registry 契约 r002](../dataset-registry/README.md#contracts)。
- [Adapters 契约 r001](../adapters/README.md#contracts)。
- [Evaluators 契约 r001](../evaluators/README.md#contracts)。
- [v2 设计](../../../doc/RAG测评设计_v2.md)，第 8、12、14 节。

本模块拥有：
- PlanRequest、PlanningSnapshot 和 EvaluationPlan。
- 指标根请求及前置指标的依赖展开。
- 预期观测与结果槽位的完整性检查。
- 计划阶段的依赖缺口与适用性信息。
- 确定性任务顺序和请求模板。
- 计划内容身份、编码及校验。

本模块不负责：
- 重新定义指标算法、参考可信度或评分状态。
- 选择最佳 Run、attempt 或 repetition。
- 调用被测系统、Mapper、Review 或模型。
- 生成 Assessment 或 Judgment。
- 执行任务、重试、取消及预算控制。
- 计算 A/B 差异、统计区间或门禁结论。
- 直接读取或写入 Artifact Store。

C2 的准备顺序为：
数据与 Run 导入
→ 固定观测集合及关联
→ 准备 Subject、可用映射和已有审定材料
→ Planner 构造计划
→ Runner 执行评价
→ Reporting 形成报告与门禁。

准备阶段也可以先使用计划蓝图获取所需指标实例，
再补充实际依赖检查，形成正式执行计划。

Planner 采用纯计划构造：
接收已经通过提供方公开契约核验的固定输入，
返回可序列化的计划值。

输入加载、当前授权、持久化及操作回执归 Runner 协调。
这样保持已确认的依赖方向，
不为计划模块增加 Store、Mapper 或 Review 依赖。

## Contracts

契约状态：Confirmed。
契约修订：r001。
schema_version：2.0。
contract_profile：metric-planner.c2.v1。

### C1. PlanRequest

PlanRequest 包含：
- schema_version、contract_profile。
- planning_scope_key。
- inspection_level。
- snapshot。
- observation_layout。
- observation_bindings。
- metric_requests。
- policy。
- planner_id、planner_version。
- derived_from_plan_id：可为空。

planning_scope_key 是调用方预先固定的计划作用域身份。
它不能通过尚未生成的 manifest 内容引用构造，
避免循环引用。

inspection_level：
- catalog_only：展开指标与观测，生成蓝图。
- available_inputs：消费逐条实际依赖检查，生成执行计划。

catalog_only 不得被标记为已完成实际输入检查。
它可以用于准备材料和展示预计任务，
不能直接作为正式执行计划或门禁依据。

available_inputs 不代表所有指标都可计算。
存在合法的数据缺失时，仍可以形成带缺口的诊断执行计划。

所有引用必须固定到明确版本。
不接受 latest、浮动模型别名或裸 case_id 查找。

输入在构造期间必须保持不可变。
Planner 不读取环境变量、当前时间或工作目录，
不根据外部隐式配置改变计划。

### C2. PlanningSnapshot

PlanningSnapshot 包含：
- dataset_version_ref。
- Registry 提供的固定成员及选择结果。
- EvaluatorPackage 引用及 MetricCatalog。
- 本次涉及的 EvaluationSubject 及其固定引用。
- AdapterCapabilities。
- 实际 DependencyObservation 集合。
- 已有前置 Judgment 的固定描述：可为空。
- 来源、读取限制及完整性核验摘要。

共享对象沿用提供方定义：
- 数据成员、CaseKey、split、group 和审定范围归 Registry。
- Run、CaptureDescriptor 和能力声明归 Adapters。
- Subject、MetricSpec、实例、依赖报告和 Judgment 归 Evaluators。

本模块只定义这些对象的组合关系，
不建立第二套数据、运行或评分 schema。

调用方通过提供方公开接口取得输入。
Planner 校验交入对象之间的一致性，
不宣称纯内存校验能够认证外部来源或当前授权。

snapshot 不要求嵌入原始答案、源文或完整上下文。
优先传递固定引用、规范对象及已核验的依赖检查结果。
敏感内容仍由提供方和 Store 的访问规则保护。

input_snapshot_digest 覆盖影响计划的全部规范输入：
- 数据版本和成员选择。
- Subject、Run、配置及关联身份。
- 指标包与实例。
- 映射和审定批次选择。
- 实际依赖检查。
- 已有前置结果。
- 能力、限制和计划 policy。

不将本次规划操作时间写入内容身份。

### C3. 观测布局与结果槽位

observation_layout 包含：
- selected_cases：有序、无重复的 CaseKey。
- arms：有序、无重复的 arm_id。
- sample_slots：有序、无重复的采样槽位身份。
- selection_basis：固定选择依据。

最小 C2 示例：
- arms 为 A、B。
- sample_slots 只有 r1。
- 每题预期两个观测槽位。

预期观测全集为：

selected_cases × arms × sample_slots

observation_bindings 必须为每个组合提供唯一绑定：
- case_key。
- arm_id。
- sample_slot_id。
- observation_key。
- subject_ref。
- EvaluationSubject。

每个 Subject 必须与其观测、CaseKey、
数据版本和评价包一致。

没有运行记录的槽位仍应提供 run_ref=null 的 Subject。
不能通过省略绑定缩小观测全集。

缺少绑定或重复绑定属于计划输入错误。
有绑定但 Run 缺失属于评价依赖缺口。
两者分别处理。

sample_slot_id 表示预先定义的评价位置，
不重新编号来源 Run 的 attempt 或 repetition。
真实运行选择由 Runner 的固定策略完成。

Planner 不执行：
- 仅保留成功 Run。
- 仅保留答案非空记录。
- 仅保留有上下文或审定材料的记录。
- 在 A/B 中预先取共同可测交集。
- 根据分数选择最佳 repetition。

Registry 的 split、group、tier 和审定限制原样保留。
筛选子集必须有明确身份和依据，
不能称为完整基准集。

每个根指标在其请求范围内，
为每个观测生成一个 ResultSlot：
- slot_id。
- observation_key。
- root_metric_instance_id。
- requested_role：required 或 optional。
- task_id。
- 评价单位和固定目标范围。
- 规划时适用性及依据。
- 依赖缺口。

ResultSlot 表示预期结果位置，
不是评分成功记录。

辅助指标也可以产生 Judgment，
但不因依赖展开而自动增加根指标样本数。
主张或数值规则分母继续由 Evaluator 定义。

### C4. 指标根请求与依赖展开

metric_requests 每项包含：
- metric_instance_id。
- role：required 或 optional。
- case_keys：固定、非空的作用范围。
- selection_basis。

case_keys 必须是 selected_cases 的子集。
同一个根请求对所选 A/B arms 使用相同用例范围。

作用范围按预先选择和任务配置固定，
不能根据 Run 成败、实际答案或评分结果缩小。

请求必须指向同一固定 MetricCatalog 中的明确实例。
不根据相似名称选择其他实现。

不支持的 metric_id、version、profile 或 instance_id：
- 返回明确 unsupported_metric 错误。
- 列出不支持的完整身份。
- 不用近似指标补齐。
- 不伪造 not_applicable Judgment。

依赖展开只消费 Evaluator 已定义的：
- MetricInstance 前置实例。
- MetricSpec.requires。
- 任务维度映射及其固定组合。

规划器不重新实现数值提取、事实支持和任务成功算法。

对每个观测：
1. 加入其作用范围内的根实例。
2. 递归加入全部前置实例。
3. 同一 Subject、包和实例只创建一个任务节点。
4. 合并节点的根请求来源。
5. 检查引用闭合和依赖无环。

required 根的前置任务标记为 required_support。
optional 根的前置任务标记为 optional_support。
同一辅助节点同时服务两者时保留全部来源，
必需程度提升为 required_support。

节点可以同时是根指标和辅助指标。
只执行一次，分别记录其角色。

依赖图仅表达固定评价任务。
不允许任意用户代码节点、
网络调用节点或跨样本聚合节点。

### C5. DependencyObservation 与缺口处理

DependencyObservation 将以下内容绑定：
- 一个明确 EvaluationRequest。
- 对应 Evaluator DependencyReport。
- 该请求中的固定 Subject、指标实例及依赖引用。
- 本次检查覆盖的内容身份。

Runner 调用 Evaluator.check_dependencies 取得报告。
Planner 不根据 AdapterCapabilities 自行补造检查结果。

available_inputs 计划要求每个任务都有对应实际检查。
检查请求中的未来前置 Judgment 可以尚未存在，
但必须与计划中的前置实例唯一对应。

依赖区分为：
- available：实际固定依赖已经满足。
- pending_component：由本计划的前置评价任务提供。
- missing_input：缺少现有数据、参考、映射、计量或标签。
- uncertain_input：现有依据存在歧义或冲突。
- unchecked：尚未完成实际检查。

仅当提供方声明能唯一对应到前置指标时，
才能使用 pending_component。
无法解析的依赖不能凭名称猜测。

catalog_only 可以包含 unchecked。
available_inputs 不能用 unchecked 冒充检查完成；
无法取得检查结果时，执行计划准备明确失败。

pending_component 不表示前置任务将得到 ok。
它只表示 Runner 将按计划尝试取得结果。

missing_input 和 uncertain_input：
- 保留对应任务和 ResultSlot。
- 记录 requirement_id、原因及固定作用范围。
- 不自动删除指标或观测。
- 不改成零分。
- 不自动调用模型或创建新标签。

缺少映射、审定或 Subject 的准备工作由 Runner 协调。
Planner 只列出依赖需求和负责模块，
不复制 Mapper、Review 的请求契约。

准备工作改变了实际输入时，
使用新的 snapshot 构造新计划。
不原地删除旧计划中的缺口记录。

能力声明只作为格式兼容与接入信息：
- 声明支持上下文不证明每条 Run 有上下文。
- import_run 能力足以接入已有运行。
- 不要求在线 e2e_run 能力才能评价已导入的真实链路。
- 不从 operations 推断尚未记录的产物。

### C6. 适用性与评价状态的边界

计划中的 applicability 为：
- applicable。
- not_applicable。
- undetermined。

适用条件和含义由 MetricSpec 拥有。
Planner 不另建指标名称到适用规则的硬编码表。

没有可消费的明确适用性依据时，
使用 undetermined。
依赖检查全部完成也不自动等于 applicable。

已经存在、且输入绑定完全一致的有效 Judgment，
可以为适用性提供明确依据。
Planner 仍保留该 ResultSlot，
由 Runner 决定是否按恢复规则复用或重新评价。

不能因为以下情况设置 not_applicable：
- Run 失败或超时。
- 答案、上下文或标签缺失。
- 评分程序曾经失败。
- 没有抽取出主张。
- 某个来源映射未命中。

例如：
- 非流式 TTFT 的不适用规则归 Evaluator。
- 无事实主张的 Faithfulness 需要完整核验。
- Planner 无法确认时保留 undetermined，
  等实际 Judgment 给出结论。

Planner 不生成 Judgment.status。
计划缺口、任务执行状态和评分状态是三个不同层次。

### C7. PlanTask 与执行依赖

PlanTask 包含：
- task_id。
- observation_key、subject_ref。
- metric_instance_id、instance_digest。
- role 及服务的根请求。
- fixed_input_refs。
- component_bindings。
- depends_on。
- evaluation_request_template。
- preflight_findings。

evaluation_request_template 使用 Evaluator 请求字段：
- subject_ref。
- metric_instance_id。
- mapping_batch_refs。
- assessment_batch_refs。

前置结果通过 component_bindings 描述：
- 来源 task_id 或已存在的 Judgment 引用。
- 预期 metric_instance_id。
- 预期 Subject 和目标范围。

任务 ID 不能作为 component_judgment_refs 使用。
Runner 必须在前置任务结束后，
取得实际 Judgment 引用并实例化 EvaluationRequest。

depends_on 表示：
前置任务已经结束并明确记录结果或失败，
不要求其业务分数通过，也不要求 status=ok。

不能因为前置分数为 0，
就按执行失败跳过后续评价。

前置评价产生 error Judgment 时，
后续组合指标仍可以消费该状态。
前置操作未能提交 Judgment 时，
Runner 保留该缺口，并按 Evaluator 契约请求后续判定。

这样可保留：
- TaskSuccess 的已知失败短路。
- CompleteEvidenceSet 的三值逻辑。
- 已有决定性结论与其他未知原因。

Planner 不自行推导这些分数。

task_id 由以下规范内容确定：
- planner profile。
- planning_scope_key。
- observation_key 和 Subject。
- 指标实例身份。
- 固定输入。
- 前置任务或已有结果绑定。

task_id 不包含执行 attempt、操作时间或调度状态。
它是计划节点身份，
不单独构成可复用 Judgment 的完整缓存键。

实际 EvaluationRequest 的所有依赖，
仍由 Evaluator 在执行和复用时校验。

### C8. EvaluationPlan

EvaluationPlan 包含：
- schema_version、contract_profile。
- plan_id。
- planner_id、planner_version。
- planning_scope_key。
- inspection_level。
- input_snapshot_digest。
- derived_from_plan_id。
- dataset_version_ref、package_ref。
- 观测布局及绑定摘要。
- 根指标请求。
- ResultSlot 集合。
- PlanTask 集合。
- 依赖边及确定性执行顺序。
- 输入缺口和限制。
- 重复执行来源提示。
- preflight_summary。
- plan_policy。

preflight_summary 包含：
- 预期观测数及各 arm 数量。
- 各根实例的预期结果槽位数。
- 根任务与辅助任务数量。
- 已检查、待前置结果、缺失及不确定依赖数。
- required 范围中的输入缺口。
- applicability 的三态计数。
- inspection_complete。
- gate_input_status。

gate_input_status：
- blocked：存在已知必选输入缺口或未完成必要检查。
- no_known_input_gap：当前没有已知必选输入缺口。

no_known_input_gap 不表示门禁通过，
也不证明样本量、统计条件或业务阈值满足。

catalog_only 始终不能提供正式门禁输入保证。
已知必选输入不足时只能形成诊断结果，
补齐后需要新计划和后续 Reporting 判定。

preflight_summary 不包含预测 N_ok 或预测平均分。
不能把可用字段数称为实际指标有效覆盖率。

计划内容采用规范 JSON：
- UTF-8，无 BOM。
- 对象键采用固定规范顺序。
- ID 保持字符串。
- 拒绝重复键、未知字段、NaN 和 Infinity。
- 计数为整数，Decimal 保持所属契约的字符串表示。
- 不把无序集合的迭代顺序写入内容身份。

plan_id 为规范内容排除自身 plan_id 后的 SHA-256。
计划不嵌入指向自身的 ArtifactRef。

Runner 使用 Planner 编码接口保存计划，
在自身 manifest 中记录 ArtifactRef。
Planner 不直接操作 Store。

### C9. 确定性排序与一致性

观测顺序保留：
- selected_cases 顺序。
- arms 顺序。
- sample_slots 顺序。

根结果槽位保留 metric_requests 的声明顺序。
依赖任务的稳定排序以：
- 观测序号。
- metric_instance_id。
- task_id。
作为拓扑排序中的候选优先级。

执行顺序必须满足全部前置依赖。
排序规则不能选择“更可能成功”的任务。

对相同规范输入和 Planner 版本：
- plan_id 相同。
- 节点、边、槽位及顺序相同。
- 缺口摘要相同。

并发度、worker 分配、实际开始时间及重试
由 Runner 拥有，不写入 PlanTask 的可变状态。

规划检测以下冲突：
- 数据版本或访问域不一致。
- Subject 与观测不一致。
- 一个观测有多个已选 Run。
- 同一槽位有重复绑定。
- 指标依赖不存在或成环。
- 前置 Judgment 属于其他 Subject、包或实例。
- 同一逻辑任务出现不一致的固定输入。

已知重复 source_execution_key 保留提示与来源关系。
不把重复导入当作独立采样。

同一 Subject 被绑定为不同 A/B 观测属于关联错误。
来源执行身份未知时保留限制，
Planner 不认证统计独立性。

### C10. 公开接口

公开入口：

build_plan(
    request: PlanRequest
) -> EvaluationPlan

validate_plan(
    plan: EvaluationPlan,
    request: PlanRequest
) -> PlanValidation

encode_plan(
    plan: EvaluationPlan
) -> bytes

decode_plan(
    content: bytes,
    limits: PlanLimits
) -> EvaluationPlan

上述操作均不执行文件、网络或模型 I/O。

build_plan：
- 校验请求及所有对象组合。
- 展开依赖。
- 构造完整槽位及任务。
- 检查限额、排序、身份和摘要。
- 返回不可变计划值。

validate_plan：
- 检查 schema、身份、数量及引用关系。
- 对照当前明确交入的固定 request。
- 检查快照、任务、边和槽位是否一致。
- 不通过旧计划证明当前 Store 权限仍然有效。

PlanValidation：
- status：valid、input_changed 或 invalid。
- mismatch_fields。
- 安全错误及限制。

encode_plan 不接受内容身份与正文不一致的对象。
decode_plan 执行大小、深度、未知字段及摘要检查。

decode_plan 只证明文件符合计划格式，
不证明其依赖仍可读或来源可信。
恢复执行前仍需 Runner 重新核验依赖，
并调用 validate_plan。

### C11. 错误与限额

公开错误：

| reason_code | 含义 |
|---|---|
| metric-planner.invalid_input | 请求、布局、策略或字段非法 |
| metric-planner.unsupported_schema | 不支持的计划版本 |
| metric-planner.unsupported_metric | 请求的指标实现不存在 |
| metric-planner.reference_mismatch | 固定对象或版本组合不一致 |
| metric-planner.observation_conflict | 观测绑定重复、缺少或冲突 |
| metric-planner.dependency_invalid | 依赖不存在或无法唯一解析 |
| metric-planner.dependency_cycle | 评价依赖成环 |
| metric-planner.inspection_incomplete | 执行计划缺少实际依赖检查 |
| metric-planner.limit_exceeded | 计划规模超限 |
| metric-planner.integrity_mismatch | 计划摘要、数量或内容不一致 |
| metric-planner.internal_error | 计划构造程序故障 |

Planner 不捕获提供方访问失败后补成空数据。
输入加载失败由调用方按原所属模块保留错误。

合法的数据缺失作为计划缺口表达，
不等同于非法引用或计划构造失败。

PlanLimits 默认：
- max_observations：10000。
- max_metric_instances：1000。
- max_tasks：100000。
- max_dependency_edges：1000000。
- max_plan_bytes：67108864。
- max_json_depth：64。

展开前检查明显的规模乘积，
展开过程中继续检查实际节点和边数。
编码和解码均检查字节上限。

超限不截断用例、不减少根指标、
不丢弃辅助任务或省略缺口。
调用方需要明确改变范围后重建计划。

## 2. 计划构造流程

### 2.1 两阶段规划

第一阶段，蓝图：

固定成员、观测布局和指标包
→ 校验根指标
→ 展开前置实例
→ 生成完整 ResultSlot 和任务模板
→ 返回 catalog_only 计划

Runner 可以使用蓝图准备：
- Subject。
- 所需实际映射。
- 已有审定材料导入。
- 每个静态 EvaluationRequest 的依赖检查。

上述准备不得自动增加模型或业务系统调用。

第二阶段，执行计划：

取得实际 DependencyObservation
→ 使用固定观测和指标请求重建
→ 区分前置结果等待与外部输入缺口
→ 形成 available_inputs 计划
→ Runner 保存并执行

第二阶段可以引用第一阶段 plan_id，
但必须形成新的计划身份。
不能原地把蓝图状态改成“已检查”。

### 2.2 依赖展开伪代码

for observation in frozen_observations:
    roots = roots_selected_for(observation.case_key)

    for root in roots:
        visit(root, observation)

visit(instance, observation):
    校验实例存在
    若处于当前递归栈，报告 dependency_cycle
    若相同节点已经展开，合并根来源并返回

    标记进入递归栈
    对固定前置实例递归 visit
    创建节点和组件绑定
    标记退出递归栈

检查所有槽位均指向存在的任务
检查全部边和数量
执行稳定拓扑排序
保存缺口摘要
计算 plan_id

已有 Judgment 只在绑定完整且来源选择明确时作为输入。
Planner 不从多个候选结果中选择最高分或最新结果。

### 2.3 执行时再次检查

计划反映构造时的固定输入，
不替代实际执行检查。

Runner 每次派发：
- 检查计划和任务身份。
- 检查当前访问权限及内容完整性。
- 解析实际前置结果。
- 构造合法 EvaluationRequest。
- 调用 Evaluator。

Evaluator 重新检查指标实际依赖。
计划中的 no_known_input_gap 不能覆盖执行时新发现的问题。

出现新的数据版本、来源批次或指标包时，
重建计划并保留与旧计划的关联。

## 3. 保存、恢复与结果完整性

本模块没有自己的数据库、工作目录或任务状态表。

Runner 保存：
- 编码后的 PlanRequest 和 EvaluationPlan。
- 本次操作回执。
- 任务 attempt、执行状态及结果引用。
- 未开始、失败、取消和未完成范围。

PlanTask 不包含可变执行状态，
避免计划内容随着执行进度改变。

恢复时：
1. 通过 Runner 授权读取计划。
2. decode_plan 校验内容。
3. 重新核验固定依赖。
4. validate_plan 对照明确输入。
5. Runner 按任务记录复用或续做。
6. Evaluator 校验实际评分复用条件。

PlanTask 已存在不代表任务已执行。
某个 Judgment 存在也不代表全部计划完成。

每个根 ResultSlot 最终必须对应：
- 一个明确选定的 Judgment。
- 或一个明确的未完成记录。

Evaluator 操作失败但没有 Judgment 时，
不能通过删除 ResultSlot 宣称完成。

辅助任务的结果不能冒充根任务结果。
一个评分被多个根任务引用，也不能增加样本数。

A/B 的共同可测集在评分后由 Reporting 计算。
Planner 始终保留原始双方观测和缺失槽位。

## 4. 实现与安全安排

允许的业务依赖：
- Dataset Registry / Contracts r002。
- Adapters / Contracts r001。
- Evaluators / Contracts r001。

不依赖 Artifact Store、Mapper、Review、
Runner、Reporting 或 Diagnostics 的代码。

输入中的嵌套产物引用沿用提供方公开 DTO。
本模块不复制 ArtifactRef 定义，也不直接读取其内容。

计划入口：
- rag_eval.metric_planner.contracts。
- rag_eval.metric_planner.api。

内部可以按需要划分：
- 请求和快照组合校验。
- 观测与槽位构造。
- 依赖展开与排序。
- 缺口归类。
- 规范编码及计划校验。

不引入通用 DAG 引擎或第三方工作流服务。
私有图结构、函数和文件组织由实现决定。

构造复杂度主要受任务节点和依赖边控制。
不进行检索、主张抽取或答案比较。
本轮没有已验证的吞吐与内存目标。

计划只保存必要身份、引用和安全原因。
不将答案、上下文、秘密值或详细审定说明复制到日志。

CLI、Web、数据库迁移和外部部署在本模块中不适用。
原因是本模块仅构造和编码离线计划。

## 5. 验证计划

以下为计划验收，尚未执行：

| 场景 | 应观察到的结果 |
|---|---|
| A/B 各一条完整 Run | 每题保留两个观测及各根结果槽位 |
| B 缺少某题 Run | 保留 null Run Subject，不取共同交集 |
| 成功和失败 Run 混合 | 均进入原定观测集合 |
| 同槽位出现两个 Run | 报关联冲突，不选最佳 |
| 一个根指标只适用于固定用例范围 | 仅按冻结范围构造，双方范围一致 |
| 同一辅助指标被多个根依赖 | 一个任务，多条根来源 |
| 辅助指标本身也是根 | 执行一次，保留双重角色 |
| 指标依赖不存在或成环 | 构造失败，明确原因 |
| 请求未知指标 profile | 明确不支持，不替换近似指标 |
| Adapter 声明支持上下文，实际未记录 | 保留输入缺口 |
| Adapter 只有 import_run 能力 | 可评价实际已导入的完整记录 |
| 缺少实际依赖检查 | 蓝图可形成，执行计划不能冒充检查完成 |
| 前置 Judgment 尚未计算 | pending_component，不假定其将通过 |
| 标签缺失或映射歧义 | 保留任务、槽位和不同缺口原因 |
| Run 失败但任务可确定失败 | 后续 Evaluator 仍有机会给出真实 fail |
| 前置 Judgment 为 error | 作为评分状态传递，不按业务失败跳过 |
| 仅因答案为空 | 不自动设置 not_applicable |
| 已核验的不适用结果 | 保留槽位及明确依据 |
| 同一真实执行重复导入 | 保留来源关系，不增加独立采样声明 |
| 前置结果来自其他 Subject | 拒绝绑定 |
| 相同规范输入重复构造 | 计划、顺序和摘要一致 |
| 输入集合迭代顺序不同 | 规范结果保持稳定 |
| 改变包、来源批次或固定观测 | 新计划身份 |
| Planner 输出任务 ID | Runner 不将其当 ArtifactRef 使用 |
| 编码内容被修改或截断 | 解码或摘要检查失败 |
| 计划规模超限 | 明确失败，不截断样本 |
| 执行中取消或预算停止 | Runner 保留未完成槽位 |
| no_known_input_gap | 不自动生成 gate=pass |

单元验证使用固定的规范输入。
集成验证使用真实 Registry、Adapters 和 Evaluator 的读取与检查结果。

计划测试不使用评分替身证明真实质量。
图和排序测试只验证计划语义。
应用测试与集成验收尚未执行。

## 6. 设计进度与后续边界

进度摘要：Confirmed / r003（包 r022）；业务单元修订仍按下表记录。

| 设计单元 | 审阅状态 | 内容修订 | 位置 |
|---|---|---|---|
| 职责与边界 | Confirmed | r001 | 第 1 节 |
| 计划、观测和依赖契约 | Confirmed | r001 | Contracts / C1–C9 |
| 接口、错误和限额 | Confirmed | r001 | Contracts / C10–C11 |
| 构造与集成流程 | Confirmed | r001 | 第 2 节 |
| 保存、恢复及结果完整性 | Confirmed | r001 | 第 3 节 |
| 实现与安全安排 | Confirmed | r001 | 第 4 节 |
| 验证计划 | Confirmed | r001 | 第 5 节 |

Complete 仅覆盖本文件声明的 C2 离线范围。
应用实现与验收尚未执行。
C2 的当前就绪状态见[索引](../../index.md#5-候选推进范围)，
确切实现基线见[交接 C2-H003](../../handoffs/c2-r003.md)。

后续范围：
- 在线系统执行和能力驱动调用计划。
- Oracle、受控干预及自适应诊断。
- 在线模型裁判和预算分流。
- 通用工作流、分布式调度和动态扩图。
- 根据已观察结果调整采样或选择指标的实验协议。

上述能力需单独确认。
不得在当前 C2 计划中隐式引入。

公开接口、任务身份、观测全集、依赖含义、
保存边界和验收行为变化需要确认。
私有实现可在已确认边界内决定。
