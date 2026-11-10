# Diagnostics：证据诊断与受控干预

- 模块 ID：diagnostics；文档、概述及 Contracts 修订：r001。
- 状态：Confirmed；详设覆盖：Complete，仅限本包声明范围。
- 候选：C3；设计包：r022。
- 确认依据：2026-11-10 用户授权连续完成设计及写入；未逐节人工复审。
- 当前交付是设计，不是应用、真实干预或诊断准确性验收。

## 1. 职责、范围与关键选择

依据：[v2 §12–14](../../../doc/RAG测评设计_v2.md)、
[架构正文 r001](../../00-system-architecture.md)、
[目录正文 r001](../../01-directory-layout.md)和
[横切规则正文 r001](../../02-cross-cutting-concerns.md)。

本模块隐藏多阶段记录核对、观察规则、竞争假设、实验资格及受限解释。
公开结果遵循：
symptom → hypotheses → supporting_observations → interventions → observed_effect → conclusion_scope。

首版包括：
- 导入真实 Run、映射和 Judgment 的确定性诊断，不调用诊断 LLM。
- 所有类别的证据、未知及后续检查建议；有充分观察才能细分到具体环节。
- 首个严格 Oracle：同一最终文本生成调用的上下文替换、配对重复及审计。
- 外部已执行试验导入；不凭外部“Oracle”名称认定严格资格。
- 其他干预的类型化设计与导入解释；通用阶段执行不进入首版。

Diagnostics 不执行系统调用，不调度 Runner，不拥有 Run、Judgment、
gold 或 ReviewRecord，也不修改标签。Runner 执行并控制预算，Evaluator 评分，
Reporting 汇总与展示。无数据时形成明确缺口，不补造未观察到的阶段。

允许业务依赖只有：
[Store Contracts r002](../artifact-store/README.md#contracts)、
[Registry Contracts r002](../dataset-registry/README.md#contracts)、
[Adapter 核心 r001](../adapters/README.md#contracts)及
[诊断扩展 r001](../adapters/diagnostic-extension.md#contracts)、
[Mapper 核心 r001](../evidence-mapper/README.md#contracts)及
[阶段扩展 r001](../evidence-mapper/diagnostic-stages.md#contracts)、
[Evaluator Contracts r001](../evaluators/README.md#contracts)。
禁止导入 Runner、Reporting、Review、Builder 或 Planner，含 TYPE_CHECKING。
CLI/Runner 将其事实投影为本模块拥有的输入 DTO，保留来源引用与转换版本。

采用版本化规则而非自由文本自动根因排序：输出可复查、测试边界清楚；
代价是未覆盖的新模式只能请求补充观察。没有校准数据时不产生 confidence、
root_cause_probability 或“准确率”，不拿规则级别冒充统计置信度。

## Contracts

状态：Confirmed；修订：r001；schema_version=2.0；
contract_profile=diagnostics.c3.v1。以下为唯一公共契约正文；
主题文件解释算法，不另建同名实体。

### C1. 公共表示和身份

拒绝未知字段、重复 JSON 键、BOM、非有限数及 latest 引用。
ArtifactRef、RunRef、CaseKey、EvidenceKey、DatasetVersionRef、Judgment
直接使用提供方类型。字符位置为 Unicode code point 半开区间。
有限 Decimal 编码为字符串，计数为非负整数，操作时间为 UTC。

每个内容对象有 kind、content_id、producer_version、dependency_refs、limitations。
content_id 为 rag-eval-json-v1 规范内容排除自身 content_id 后的 SHA-256；
不含操作时间、操作者、回执、指向自身的 ArtifactRef。
dependency_refs 是具名、有序、无重复的实际依赖列表。
业务对象嵌套内容只引用已提交 ArtifactRef，禁止循环依赖。
同内容重算身份不变，规则/输入变化形成新对象，不覆盖历史。

BatchResult={manifest_ref, receipt_ref}。
子对象先提交、完整清单最后提交、操作回执独立；详见 operations 第 2 节。

### C2. DiagnosticRequest 与 ObservationBinding

DiagnosticRequest：
- dataset_version_ref、evaluator_package_ref、rule_pack_ref。
- ordered_bindings：无重复的 ObservationBinding 列表。
- context_policy_ref、limits_ref、provenance_ref。
- source_set_ref：显式冻结的完整输入来源集合，不搜索全库补齐。
- derived_from：可选旧批次，只用于谱系，不混用旧内容。

每个 ObservationBinding：
- observation_key：调用方冻结的槽位，不证明真实执行独立。
- case_key、arm_id、sample_slot_id、split_group_id。
- run_ref：可为 null；null 时有 missing_run_reason。
- association_report_ref：Evaluator 所有的真实输入关联报告。
- selected_stage_scopes、selected_generation_call_id：显式实例，缺失允许为空。
- judgment_refs、mapping_batch_refs、stage_observation_refs。
- expected_metric_instances：根症状的完整预期实例，不只传失败指标。
- provenance_ref、projection_version、source_manifest_refs。

Runner/CLI 提供槽位和来源；Diagnostics 重核对内容绑定。
已知错题、错 Run、错 scope、错 package 为 reference_invalid；
unknown 关联保留缺口，不生成参考依赖的确定质量结论。
多调用不自动取最后一个。run_ref=null 不伪造失败运行。
每个 binding 恰有一个 DiagnosticFinding，即使没有可测症状。

### C3. RulePack 与结构化观察

RulePack 包含 rule_pack_id、revision、规则列表、实现版本及 limits。
首版固定 diagnostic_rules_v1，规则不可执行任意脚本/表达式；
每条有 rule_id/version、适用目标、必需观察、否定条件、允许断言模板、
备选解释、可证伪后续动作及排序键。业务阈值不在本模块猜测。

DiagnosticObservation：
- observation_id、observation_key、kind、scope。
- state：observed、unknown、not_applicable、conflicting。
- value：kind 对应的强类型值；后三态为 null。
- evidence_refs：原始对象、具体字段/范围/实例/审查范围。
- basis：recorded、mechanically_verified 或 reviewed_assessment。
- assurance_limits、reason_code、dependency_refs。

kind 仅取 run_terminal、metric_result、anchor_presence、fact_support、
sufficient_context、stage_link、capture_quality、config_consistency、
policy_violation、reference_limitation、measurement_failure。
scope 固定 Run/调用/阶段实例、rank/输入范围；不能用显示顺序证明数据流。
recorded/reviewed 是证据来源，均不声明来源真实性认证。

### C4. DiagnosticFinding、症状和假设

Finding 包含 binding_ref、symptoms、observations、hypotheses、
recommended_actions、coverage、conclusion_scope、rule_pack_ref。
coverage 保存所有请求/已检查/未知/不适用的 scope 与原因，不用单个“覆盖率”掩盖范围。

Symptom：
symptom_id、metric_instance_id（可为空）、judgment_ref（可为空）、
kind、state、affected_scope、reason_code。
kind 为 task_failure、dimension_failure、operational_failure、
measurement_gap、measurement_error、reference_dispute、no_observed_failure。
状态为 established、undetermined、not_applicable。
Judgment.status=error 属 measurement_error，不属于 task_failure；
任务 pass 和非决定性测量缺口可同时保留。

Hypothesis：
hypothesis_id、categories、claim_code、claim_parameters、epistemic_state、
supporting_observation_ids、counter_observation_ids、missing_requirements、
competing_hypothesis_ids、rule_id/version、followup_action_ids、scope。
epistemic_state 为 supported_candidate、unresolved、counterevidenced；
不提供 confirmed_root_cause。空竞争列表也不证明唯一性。
categories 可多选：
source_reference、parsing、chunk_index、rewrite_routing、retrieval_filter、
rerank、compression_assembly、generation、citation、policy、measurement。
排序为测量/参考可解释性检查优先，再证据约束强弱、固定 category 顺序、ID；
是调查优先级，不是发生概率。无症状不生成“所有组件正确”。

### C5. InterventionPolicy、Selection 与资格

InterventionPolicy：
selection_policy（[interventions 第 1 节](interventions.md#1-抽样和预算)）、
allowed_kinds、set_selection、trial_mode、repetitions、pair_order_seed、evaluation_baseline、
generation_budget_policy_ref、capacity_policy_ref、provenance_ref。
trial_mode=original_comparison|paired_repeat；前者一个历史 original 与一个外部 Oracle，
repetitions 固定为 1，只导入审计；后者每个槽位新调用两臂。
repetitions 为正整数，expected_pair_slots 为 plan_id × 1..R，冻结后不增删。
选择先于观察新增试验结果冻结，禁止按恢复效果补选。

InterventionSelection 为完整来源母集逐项处置：
selected、not_selected、ineligible、eligibility_unknown；
保存选择渠道、随机优先键/seed、分层名额及原因。
渠道可重叠，最终去重，记录全部入选路径，不伪造联合纳入概率。

EligibilityReport：
binding_ref、kind、每项 requirement_id/state/evidence_refs/reason、
decision=eligible|ineligible|unknown、selected_gold_set_id、limitations。
缺能力/原始配置可为 ineligible；依赖未知为 unknown；
损坏或非法引用是操作错误，不包装为业务资格不足。

### C6. InterventionPlan

计划包含：
baseline_binding_ref、baseline_run_ref、generation_call_id、trial_mode、
kind、eligibility_report_ref、selection_ref、treatment_manifest_ref、
invariant_manifest_ref、evaluation_baseline、expected_pair_slots、
requested_resources、falsifiable_hypothesis、interpretation_limits。

kind：
strict_context_oracle、alternative_generation、fixed_candidate_order、
reviewed_rewrite、parser_substitution、evidence_position_noise、
premise_removal、no_context_control。
除 strict_context_oracle 外首版只规划和审计导入，不提供通用执行。
每类必须明确 changed_variables、fixed_variables、target_artifacts、
运行/评分条件及允许结论，不能把任意配置变化归为严格 Oracle。

TreatmentManifest 保存选定充分组、每个源 EvidenceKey 与固定文本、
gold 选择/排序规则、原/新 context、注入区间、格式化与 token 版本、
内容/长度/噪声/顺序差异、访问和出域判定依据。
只包含源证据及原格式所需来源标签，不把 reference_answers、
预计算答案、评分或诊断假设注入生成请求。
InvariantManifest 按 interventions 第 2 节固定全部非干预条件与可验证差异。
计划内容身份在执行之前冻结；不包含未来 Run 或判断结果引用。

### C7. TrialEvidenceEnvelope

供 Runner/CLI 投影，不导入其内部类型：
- plan_ref、execution_source_ref、converter_id/version。
- 全部预期 pair_slot 的 disposition；不只传成功配对。
- 每个槽位 baseline/treatment 的 RunRef（均可 null）及调用 ID。
- Adapter GenerationReceipt、实际请求/响应、来源执行身份。
- 每臂 Evaluator Subject、association report、Judgment 引用。
- attempts、未执行/结果未知/预算及取消原因的来源投影。
- protocol_check_refs、usage_refs、limitations。

槽位编号由计划定义，不等于业务 attempt。
历史 original 只允许单独 original_comparison，
不能复制成多个 baseline 重复。live 配对每次重新调用两臂。
同一来源执行不能同时充当两个臂或多个独立重复。
本 DTO 不提供密码学认证，来源声明保留 local_import_only。

### C8. InterventionAssessment 与效果

Assessment：
plan_ref、envelope_ref、protocol_validity、每项 invariant_check、
pair_results、counts、selection_scope、conclusion_scope、
hypothesis_updates、review_suggestions、dependency_refs。
protocol_validity=strict_valid|non_strict|unverified。
不合协议仍保存记录，不按严格试验合并，不自动重新执行。

每对保存 baseline_state、treatment_state（pass/fail/unknown）、
各 Judgment 状态及引用、transition、
semantic_scope、protocol_validity、effect_code、limitations。
只有兼容 TaskSuccess status=ok 才作任务 pass/fail；
其他指标需显式命名为 dimension_transition，不能冒称整体任务成功。
unknown 保留 not_computable/inconclusive/error/not_applicable 的细分原因。
解释与分母见 [interpretation](interpretation.md)，本模块不计算统计 CI 或门禁。

### C9. 公开入口

```text
create_diagnostics(artifact_store, dataset_registry, adapters, evidence_mapper,
                   evaluators) -> Diagnostics
diagnose(request_ref, access_context) -> BatchResult
plan_interventions(request_ref, access_context) -> BatchResult
assess_interventions(envelope_ref, access_context) -> BatchResult
load_batch(manifest_ref, access_context) -> DiagnosticBatchReader
```

plan_interventions 的请求为 findings_manifest_ref、policy_ref、
capability_snapshot_refs、frozen_generation_spec_refs；实际类型归对应提供方。
输出完整 selection、所有资格结果、合格计划及未规划原因。
capability 快照不能代替逐例实际产物检查。
assess 不重评分或调用 Runner；缺有效 Judgment 则 unknown。
reader 提供原始清单、全部对象及依赖读取，不按根因分数筛选。
上述操作均无业务网络请求；Runner 才调用在线 Adapter。

### C10. 错误与资源边界

reason_code：
diagnostics.invalid_input、unsupported_schema、reference_invalid、
target_inconsistent、access_denied、integrity_mismatch、limit_exceeded、
storage_failure、internal_error，遵守横切错误维度并保留安全提供方原因。
未记录、无有效标签、无适配器能力和协议未决是结果中的原因；
读取失败、坏哈希、矛盾绑定、超限是错误，不能降为“无证据”。

limits 必填 max_bindings、max_refs_per_binding、max_observations、
max_hypotheses、max_total_bytes、max_pair_slots，均正整数。
超限不截断病例/规则/竞争解释；整个操作明确失败，已提交子对象保留。
诊断是有界本地读取与规则匹配，不进行全库搜索或无界路径枚举。

## 2. 主题、实现安排与本地进度

| 单元 | 内容 | 状态 / 修订 |
|---|---|---|
| 本文 Contracts C1–C10 | 所有公共 DTO、操作与边界 | Confirmed / r001 |
| [observations.md](observations.md) | 观察归一、规则矩阵、否定约束及例子 | Confirmed / r001 |
| [interventions.md](interventions.md) | 选择、严格 Oracle、其他干预协议 | Confirmed / r001 |
| [interpretation.md](interpretation.md) | 转移、重复、分母及解释限制 | Confirmed / r001 |
| [operations.md](operations.md) | 持久化、恢复、权限、复核与验证 | Confirmed / r001 |

代码入口是 rag_eval.diagnostics.contracts/api；
规则匹配、资格判定、审计解释为模块私有实现。
不要求拆成独立服务、规则引擎或多个公共插件。

执行与展示接缝见
[Adapter 扩展](../adapters/diagnostic-extension.md)、
[Mapper 扩展](../evidence-mapper/diagnostic-stages.md)、
[Runner 扩展](../runner/diagnostic-experiments.md)、
[Reporting 扩展](../reporting/diagnostic-reporting.md)。
这些是提供方拥有的独立 r001 profile；C2 核心 profile 不被改写。

## 3. 完成与限制

模块设计在上述范围内完成。实施顺序为 C2 → C1/真实数据闭环 → C3；
先有实际观测和评分再诊断，有必要服务/权限/预算才在线干预。
真实阈值、模型固定版本和校准材料是运行输入，无须为设计虚构数据。

延后：诊断 LLM、概率校准、自动修复/改 gold、任意阶段回放、
多因素因果识别、在线监控及通用多模态 Oracle。
本模块不宣称覆盖这些能力或获得真实诊断准确性证据。

公开接口、数据含义/所有权、依赖方向、持久化、授权、
关键状态机及验收行为变更需设计确认；
私有组织和保持契约的重构由实施者决定。
