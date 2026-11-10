# Runner：诊断试验执行扩展

- 状态：Confirmed；文档、Contracts 及详细设计修订：r001；覆盖 Complete。
- profile=runner.diagnostic.v1；runner.c2.v1/r002 保持不变。
- 依据：v2 §12–13；设计先完成，实施在数据/基础评分闭环之后。
- 本扩展拥有试验作业/调度/预算；不重新定义诊断、Run 或 Judgment。

## Contracts

### X1. 请求、冻结与状态

DiagnosticExperimentRequest：
dataset_version_ref、diagnostic_plan_batch_ref、selected_plan_ids、
adapter_binding_ref、execution_policy_ref、scoring_template_ref、
source_sets_ref、runtime_environment_ref、provenance_ref。
selected_plan_ids 有序无重复；每个计划必须来自完整已提交批次，
不得执行未合格计划或按结果追加计划。
在线只接收 trial_mode=paired_repeat；original_comparison 仅外部导入审计，
不把历史 original 复制成新生成或多个重复。

execution_policy 固定 profile=local_paired_oracle_v1、concurrency=1、
pair/arm 顺序、max_real_calls、每调用输入/输出限额、
deadline/timeout/cleanup_timeout、重试上限及分类、独立干预预算池、
外部未决处理、缓存模式、评分预算。
默认 generate 自动重试为 0，评分沿用 C2 有限恢复策略。
max_real_calls 包括失败、重试和可能已发送的请求；
无法确认的资源不释放成“未用额度”。不得根据答案质量增加次数。

DiagnosticExperimentManifest 固定请求、所有 plan/spec、
全部 pair_slot/arm、顺序、发送前核验结果及 blocked 原因、
期望评分槽位、预算、provider/规则/schema 版本。
身份为规范内容摘要；不含未来执行结果。
合法但不合格的计划形成 blocked，不静默降为 alternative；
schema/权限/完整性错误按公开错误契约失败。
准备不产生业务调用；只有 execute 明确启动后才可联网。

执行作业状态为 prepared、running、completed、partial、blocked、
cancelled、outcome_unknown；状态与评分/恢复效果分开。
completed 表示每个预期调用已形成终态或明确预执行阻断的处置且完整清单已提交，
不表示每个 pair 可测或系统通过。
预算耗尽/未开始形成 partial，用户取消为 cancelled；
存在未决外部请求时优先 outcome_unknown，保留 stop_reason=cancel/budget 等。
blocked 是首个调用前的能力/条件阻塞；不得含“已经执行但没记录”的请求。

### X2. 公开操作和结果

```text
create_diagnostic_runner(config, artifact_store, dataset_registry, adapters,
                        evidence_mapper, evaluators, review_api,
                        metric_planner, diagnostics) -> DiagnosticRunner
prepare_diagnostic_experiment(request_ref, operation_key, access_context) -> DiagnosticPreparationResult
execute_diagnostic_experiment(manifest_ref, operation_key, access_context) -> DiagnosticJobResult
get_diagnostic_job(job_key, access_context) -> DiagnosticJobResult
resume_diagnostic_job(request_ref, access_context) -> DiagnosticJobResult
cancel_diagnostic_job(request_ref, access_context) -> DiagnosticJobResult
reconcile_diagnostic_invocation(request_ref, access_context) -> DiagnosticJobResult
prepare_diagnostic_scoring(request_ref, operation_key, access_context) -> DiagnosticScoringBatchResult
export_trial_evidence(request_ref, access_context) -> TrialEvidenceEnvelopeRef
```

DiagnosticPreparationResult={status:ready|blocked,manifest_ref,validation_ref,receipt_ref}。
DiagnosticJobResult={job_key,state,snapshot_ref,receipt_ref,errors,uncompleted_scope}；
job_key 在首个外部调用前可查询，操作失败仍返回安全键/已提交阶段。
operation_key 是调用方提供的非敏感稳定键；逻辑键由 store/access_domain、
operation_kind、operation_key 组成，首次将其绑定输入 manifest/request 摘要。
同键同输入返回现有作业/快照，不重新生成；同键异输入为 operation_conflict。
新的真实重复实验须显式新 operation_key 并保留 derived_from 关系，
不能在 resume 内更换键或重置预算。prepare/scoring 采用同样规则，
内部 C2 operation_key 由该键与 plan_id 派生，冲突/锁语义沿用 C2。
TrialEvidenceEnvelopeRef={envelope_ref,receipt_ref}，DTO 归 Diagnostics C7。

resume 请求固定 job_key、expected_manifest_ref、expected_checkpoint_ref，
不能改变 policy/数据/计划。新增真实尝试须独立决策对象：
invocation_key、已知远端结果、决策者/授权来源、reason、
allow_additional_attempt、接受的重复执行/费用风险及预算上限。
unknown 时先 reconcile；不能因程序重启/重跑命令默认同意重发。
新增 attempt 保留旧未决记录，后续确认重复则去重源执行/费用。
已提交完成输出不能为了获得更好答案再调用。

### X3. 评分与装配边界

prepare_diagnostic_scoring 输入 job snapshot、scoring_template、
明确审定/映射来源。结果 DiagnosticScoringBatchResult={manifest_ref,receipt_ref}，
清单为每个选中 plan 保留恰一个 PreparationResult 或明确 blocked/operation_failed
及原因；子结果先提交，完整清单最后提交，不把已准备子集当全部完成。
它按每个 plan 生成新的 C2 trace_audit ExperimentRequest：
- 每个 trial plan 为独立固定 case selection，可包含多个 pair slot；
  arms 为 paired_baseline/treatment。
- 使用 Adapter GenerationResult.audit_import_result 的完整核心批次；
  若外部导入则沿用其真实核心 RunImportResult。
- explicit_run_table_v1 保留全部 pair slots，缺 Run=null 并保留真实原因。
- 使用同一数据/EvaluatorPackage/rubric、query/history/as_of/access_scope；
  association checks 重新基于实际运行，不复制计划预期值冒充 logged。
- 若某例/调用不满足质量关联条件，依赖真值的指标不可测。

然后复用 C2 prepare_experiment/execution 评分方法；
扩展 Runner 可调用同模块内部实现，不形成 Runner→Reporting 依赖。
新响应的审定目标从 Evaluator 真实 ScoringView 构造，
再调用 Review；不复用原答案的 ReviewRecord。
可在没有自由文本新标签时完成确定性规则，其他效果 unknown。
评分缺口不触发补生成或隐式裁判。

export_trial_evidence 先冻结指定 job/评分快照及完整来源集合，
将执行/预算/slot 事实投影为 Diagnostics C7 输入，
保留 converter identity、来源和每项处置。Diagnostics 独立核验评分对象。
导出 DTO 不自动运行 assess；CLI 显式调用。
本作业的主 manifest 不引用未来 envelope/assessment，避免摘要循环。

## 1. 执行顺序、权限和容量

每个 slot 按冻结顺序：
当前授权/出域/能力/原始 spec/模型身份/容量核验
→ 持久化 CallIntent 和预算预留
→ worker 调 Adapter.generate_with_context
→ Adapter 尽早持久化原始响应与 GenerationReceipt
→ worker 持久化 outbox
→ 协调者提交调用结果/预算结算
→ 下一臂/下一对。
Runner 在发送前重新运行必要资格检查，不依赖过期 eligible 快照。

gold 获取只用于显式 intervention 调用；ordinary C2 导入/评分路径不接收它。
只发送 TreatmentManifest 白名单源证据，
业务 access_scope 不因试验主体可读取 gold 而扩张。
超容量、缺模型固定、无法控制工具均在发送前阻塞，调用计数为零。
若调用过程中 provider 改模型/截断，保留运行与 protocol mismatch，
停止按策略派发后续依赖调用，不以恢复成功放行。

预算预留采用 provider 可核验 token/请求上界；
费用缺价格或远端强制机制时不能宣称货币硬上限。
known usage 与 outcome_unknown 计量分别记录，所有失败尝试保留。
两臂不能共用响应缓存；每次 repetition 是新真实调用，
固定 provider 缓存策略及其限制，不能用 cached output 伪造重复。

## 2. 本地持久化和恢复

沿用 C2 C16 的 Windows 本地 NTFS 锁、不可变事件/快照、
单协调者/单 worker、outbox、提交顺序和安全目录规则，
使用独立 diagnostic job 命名空间，不复用评价 execution_id。
调用身份至少包含访问域、job/manifest、plan、pair、arm、attempt；
同一 intent 不因恢复生成新 ID。

故障处理：
- intent 未提交：不允许发送。
- intent 已提交、明确 not_sent：按原预算继续。
- 可能发送但无合法 receipt：outcome_unknown，停止自动重发，
  reconcile 查询实际 provider；不支持查询则保留未知，等待明确决策。
- Adapter receipt 已提交/outbox 未写：只有已知确切 receipt_ref 或
  Adapter 可证明的 reconciliation 才能接收；未保存引用且无法查询则
  outcome_unknown，不承诺发现所有已提交结果，不全库扫描挑选对象。
- outbox 已写、协调快照未提交：校验后接收同一结果，不重调。
- 结果已提交：评分失败或报告失败只恢复下游。
- Store manifest 已提交/操作回执失败：返回已提交引用，重试只补回执。
- 撤权/内容损坏/事件链断裂：停止并保留现场，不当 ordinary cache miss。
- cancel：停止新派发，有限清理；本地结束不表示远端停止计费。

reconcile request 包含 invocation_ref、known_receipt_ref（可空）；
若已知引用则先验证并核对 invocation，否则调用 Adapter 的公开 reconciliation。
无本地/远端可核验证据时保持 unknown，不允许读取 Adapter 私有目录。
确定未发现本地回执也不能证明远端没执行。
Job Object、锁和目录提交仍须在实际 Windows NTFS 验证。

## 3. 外部结果导入和其他边界

外部试验先通过核心 Adapter.import_run 完整导入，
再按显式表构造计划/slot/调用对应的 envelope；
不按同 query 或“oracle”文件名推断关系。
没有实际请求/spec/权限/评价一致性即 unverified；
存在额外模型/预算变化为 non_strict，仍可描述原始表现。

当前在线只执行 strict_context_oracle。
其他 Intervention.kind 返回 runner.intervention_kind_unsupported，
不启动通用 replay、重新建索引或调用 Builder。
不提供分布式调度、任意 SDK 代码、LLM 裁判或自动修复。

错误新增 runner.diagnostic_plan_invalid、generation_spec_unavailable、
intervention_kind_unsupported、capacity_unverified、invocation_outcome_unknown、
diagnostic_budget_exhausted；保留 C2 的引用、权限、锁、恢复和存储错误维度。
outcome_unknown 是有记录的作业状态，不自动变为可重试错误。

## 4. 计划验证

真实本地模块衔接、受控 HTTP 服务观测实际载荷/请求次数；
故障注入覆盖 intent、发送、Adapter 提交、outbox、快照及回执边界。
测试计数包括 failed/unknown，不把 retries 当独立 repetition。
新人工材料缺失、评分 error、报告渲染失败均不增加业务调用。
真实 provider 在必要 endpoint/凭据/授权/预算具备后单独验收；
协议 fixture 不能证明模型恢复或业务根因准确。
