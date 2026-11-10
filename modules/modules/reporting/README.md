# Reporting：聚合、配对统计、报告与三态门禁

- 模块 ID：reporting
- 文档状态：Confirmed
- 文档内容修订：r004
- 修订边界：本次仅进度摘要更新；业务单元及公开契约保持各自既有修订。
- 概述状态：Confirmed / r002
- 详设覆盖：Complete
- 已确认范围：C2 结果读取、覆盖、聚合、缺失范围、配对统计、
  Markdown/JSON 报告、样例展示、GatePolicy、GateDecision 与本地导出。
- 在线监控、自动诊断与复杂统计扩展：后续范围。

本 README 正文继续限定 C2；C3 新增的[诊断与干预报告](diagnostic-reporting.md)
为独立 Confirmed / Complete / r001 设计，不改变下述 C2 契约或实施范围。

## 1. 职责与边界

依据：
- [总体架构 r001](../../00-system-architecture.md)。
- [目录设计 r001](../../01-directory-layout.md)。
- [横切规则 r001](../../02-cross-cutting-concerns.md)。
- [Artifact Store 契约 r002](../artifact-store/README.md#contracts)。
- [Dataset Registry 契约 r002](../dataset-registry/README.md#contracts)。
- [Evaluators 契约 r001](../evaluators/README.md#contracts)。
- [Runner 契约 r002](../runner/README.md#contracts)。
- [v2 设计](../../../doc/RAG测评设计_v2.md)，第 8、9、14 节。

本模块拥有：
- AnalysisRequest、AnalysisPolicy 和 AnalysisResult。
- 指标聚合系列及比较口径。
- 完整结果槽位的覆盖统计。
- query 宏平均和独立列示的主张微平均。
- 缺失数据识别范围。
- A/B 共同可测集、差异、胜负及排除原因。
- 独立单位、分组重采样及统计区间。
- ComparisonReport、GatePolicy、GateDecision 及本地报告包。

本模块不负责：
- 补造或修改 Judgment。
- 重跑系统、映射或评价任务。
- 通过选择最高分执行结果修正比较。
- 认证审定人、gold 真值或统计独立性。
- 将测量覆盖、识别范围和统计区间混成一种分数。
- 从统计相关推断根因或单一因果效果。

本轮读取一份明确的 Runner 实验准备及执行结果快照。
该快照可以同时包含 A/B。

不自动寻找最新快照。
不合并同一操作的多份历史快照。
不从多次执行中选择最高分结果。

partial ExecutionResult 允许形成诊断分析，
但全部未完成槽位必须继续保留。

Diagnostics 在本轮不是必需依赖。
不要求占位实现，不调用受控干预。

## Contracts

契约状态：Confirmed。
契约修订：r002。
schema_version：2.0。
分析契约 profile：reporting.analysis.c2.v1，C1–C12 / r001。
报告及门禁 profile：reporting.report.c2.v1，C13–C18 / r002。

### C1. AnalysisRequest 与分析策略

AnalysisRequest 包含：
- schema_version、contract_profile。
- preparation_ref。
- execution_result_ref。
- analysis_policy_ref。
- analysis_implementation_ref。
- provenance_ref。
- derived_from_analysis_ref：可为空。

preparation_ref 和 execution_result_ref 必须属于同一实验。
请求固定一个明确执行结果快照，
不接受 latest 或按分数搜索。

AnalysisPolicy 包含：
- policy_id、policy_revision。
- series_specs。
- comparison_specs。
- slices。
- repetition_policy。
- statistics_policy。
- inference_intent。
- sampling_basis_ref。
- independence_basis_ref。
- limits。

inference_intent：
- descriptive。
- confirmatory_requested。

confirmatory_requested 只表示请求用途，
不自动证明满足正式推断或门禁条件。

策略与 Runner manifest.reporting_intent 中的固定引用一致时，
记录 manifest_bound。

否则记录 post_execution_analysis，
作为探索性分析保留。
不能通过分析时声明替代事前固定策略。

是否看过历史结果、是否多次调参及是否独立 holdout，
保留明确来源和限制。
本轮不提供预注册认证。

policy_revision 绑定完整策略内容。
修改系列、切片、区间方法或排除规则形成新策略。

### C2. 聚合系列与比较兼容性

SeriesSpec 包含：
- series_id。
- 完整 metric_id、version、profile。
- 明确 metric_instance_id。
- case_keys：固定作用范围。
- arms。
- target_semantics。
- value_projection。
- aggregation。
- compatibility_requirements。

本轮一个系列对应一个明确指标实例。
不自动合并多个 numeric_rule、维度或阶段。

不同实例需要汇总为 query 结果时，
必须有独立确认的上层指标或明确新 profile。
不能直接平均后沿用原指标名称。

target_semantics 固定：
- 阶段或输出角色。
- rank_limit。
- 计量 scope 及其边界。
- 单位、币种和计量 basis。
- 参考及评分方法。
- 其他影响分母和含义的参数。

运行内的 result_id、generation_call_id 可以不同，
但必须满足同一逻辑目标及固定映射。
不能把 retrieve 与 generation_context 混为同一系列。

value_projection：
- decimal_identity。
- pass_fail_binary：pass=1、fail=0。

pass_fail_binary 只适用于 MetricSpec 明确声明的类别。
其他类别只统计频数，不分配任意数值。

ComparisonSpec 包含：
- comparison_id。
- baseline_arm、candidate_arm。
- series_id。
- 固定比较用例范围。
- direction。
- tie_tolerance：非负 Decimal，默认 0。
- interval_method。
- comparison_family_id。
- inference_role：primary 或 exploratory。

比较前检查：
- 相同正式数据版本及参考基线。
- 相同评价包和指标语义。
- 相同实例或明确兼容的实例映射。
- 相同评价单位和分母。
- 相同来源保证、rubric 和实现版本。
- 相同阶段、预算及计量口径。
- arm 之间的已声明配置差异。

不通过显示名称相同认定可比较。

组件实验必须声明拟改变的变量。
多个组件联动时标记为整体方案比较，
不声称隔离了其中某个组件的效果。

不满足比较兼容性时：
- 保留各方独立聚合。
- Comparison 状态为 not_comparable。
- 输出具体差异。
- 不计算正式差异区间。

同一系列内部存在非法混合口径时，
分析请求失败，不能先合并再补一个限制说明。

### C3. 完整结果行

Reporting 通过 Runner Reader 读取：
- ExperimentManifest。
- 固定计划及全部根 ResultSlot。
- TaskOutcome 和 ResultSlotOutcome。
- 明确选定的 Judgment。
- 实际请求、复用、恢复及未完成记录。
- 真实 Run 身份与来源限制。

通过 Evaluator.load_judgment 核验评分内容。
不解析 Runner 或 Evaluator 私有目录。

每个根 ResultSlot 恰好生成一条 AnalysisRow：
- slot_id。
- case_key、arm_id、sample_slot_id。
- series_id。
- subject_ref、run_ref。
- source_execution_key：已知时保留。
- judgment_ref：可以为 null。
- Judgment 状态、值和精确计数。
- applicability_state。
- 完成来源：computed、recovered 或 cache_reused。
- 未完成及排除原因。
- split、split_group、origin 和有效质量等级。
- 依赖和来源限制。

辅助任务不作为新增根样本。
重复引用同一 Judgment 不增加独立采样数。

缺少 Judgment 时：
- row 保留。
- score=null。
- measurement_state=unfinished。
- 不构造替代 not_computable Judgment。

非法引用、内容损坏和跨对象评分不能降级为普通缺失。
这些情况导致明确分析错误。

一个槽位有多个当前选定结果时拒绝分析。
不能在 Reporting 中挑选较高分。

### C4. 有效覆盖与状态计数

每个 series × arm × slice 保存：

- N_total：预期根结果槽位数。
- N_not_applicable：有效 Judgment 明确不适用的数量。
- N_ok：Judgment.status=ok 的数量。
- N_not_computable。
- N_inconclusive。
- N_error。
- N_unfinished：没有选定 Judgment。
- N_applicability_undetermined。
- 原因计数和对应槽位身份。

状态计数满足：

N_total
= N_not_applicable
+ N_ok
+ N_not_computable
+ N_inconclusive
+ N_error
+ N_unfinished

业务 fail、数值 0 和已知系统失败，
只要有合法 ok Judgment，均计入 N_ok。

N_ok 表示测量有效，
不表示业务通过。

适用性不能因缺少结果而缩小。
本轮保守定义：

N_applicable = N_total - N_not_applicable

有效覆盖率：

measurement_coverage = N_ok / N_applicable

分母为零时 value=null，并标记 no_applicable_units。

没有 Judgment 的槽位，
不能仅凭计划中的推测不适用从分母删除。

存在适用性未决时：
- 继续包含在上述保守分母。
- 明确 denominator_basis=including_undetermined。
- 同时显示未决数量。
- 不将该分母称为已经完全核验的适用总体。

同时保存：
- confirmed_applicable_count。
- applicability_undetermined_count。
- slot_level_coverage。
- 后述 query_level_coverage。

缺失、错误及不确定原因分别展示。
主原因形成互斥计数；
全部关联原因另列，并声明可能重叠。

Runner execution=completed
不意味着 N_ok=N_total。
缓存命中也不额外增加 N_ok 或样本数量。

### C5. query 宏平均、重复与主张微平均

质量系列默认 query 等权。

repetition_policy 本轮固定：
- 保留全部预期 sample_slots。
- 每题先在题内聚合。
- 再对可测 query 进行宏平均。
- 不选择最佳、最后一次或首个成功输出。

同一 query：
1. 移除有有效依据的 not_applicable 槽位。
2. 剩余槽位全部为 ok 时，计算题内等权均值。
3. 存在缺失、错误或未决槽位时，
   query_score=null，并保留全部已知值。
4. 全部槽位确实不适用时，query 为 not_applicable。

第三项不能只对已知 repetition 求均值后冒充完整题目分数。

输出至少包含：
- N_query_total。
- N_query_not_applicable。
- N_query_ok。
- N_query_unknown。
- query_level_coverage。
- observed_macro_mean。
- observed_query_set_digest。

observed_macro_mean：
只对 query_score 已确定的 query 求均值，
并明确它是可测子集的描述性结果。

不称为全量任务总体的无条件均值。
N_query_ok=0 时均值为 null，不填零。

pass/fail 投影为二元值后，
按同样规则形成成功比例。
多次重复的题内均值可能为非二元值，
不能继续套用独立二项比例公式。

ratio 指标优先使用完整精确 numerator/denominator
恢复有理数进行聚合。
缺少精确计数时使用合法 Decimal 并保留依据。
不以已经展示舍入的数字进行阈值判断。

主张微平均另列：
- 只消费相同指标、相同分母含义的完整计数。
- micro = sum(numerator) / sum(denominator)。
- 显示纳入的输出和主张数。
- 不完整主张清单不进入完整分母。
- 缺失输出仍保留其覆盖缺口。

微平均表示已测完整输出中的主张统计。
它不替代 query 宏平均，
也不能用更多长答案提高 query 权重。

本轮不支持未声明的业务加权平均、
分层总体校正或多个规则实例的自动混合。

### C6. 缺失数据识别范围

识别范围与统计置信区间分别保存。

适用总体已确定、分数范围固定为 [0,1] 时：
- 已知值采用其精确值。
- 未知值采用 [0,1]。
- 有合法更紧部分范围时可以使用，
  但必须保留提供方依据和适用身份。

不通过未知填零生成最终质量分。

对固定 N 个二元任务：
- 已知成功 S。
- 已知失败 F。
- 未知 U。
- N=S+F+U。

成功率识别范围：

[S/N, (S+U)/N]

对一般有界分数，使用每个任务的合法 [L_i,U_i]：

[sum(L_i)/N, sum(U_i)/N]

重复输出先按固定题内规则计算范围，
再按 query 等权聚合。
不能因为重复次数不同改变 query 权重。

适用性本身仍未确定，
或指标没有合法有限值域时：
- 不输出貌似精确的总体识别范围。
- 返回 applicability_unknown 或 unbounded_domain。
- 保留已知计数及可测子集结果。

A/B 在相同固定 query 总体上，
若各 query 已有合法范围，则：

difference_lower = mean(L_B - U_A)
difference_upper = mean(U_B - L_A)

它表示缺失导致的差异识别范围。
不把它标为 95% CI，
也不与共同可测集的统计区间互相替代。

Evaluator 的非 ok value 仍为 null。
Reporting 不能把区间中点写回 Judgment。

### C7. A/B 配对集合与差异

配对键为：
- 固定 CaseKey。
- 同一 series。
- 相同预期 sample_slot 布局。

配对在全部根结果行建立后进行，
不能在读取阶段预先取共同可测交集。

一个 query 进入共同可测集需要：
- A/B 评价语义兼容。
- 双方 query_score 均已确定。
- 用于题内均值的实际适用槽位集合一致。
- 不存在已知重复执行或错误绑定。

双方适用集合不同，
记录 applicability_asymmetric，
不强行组成同口径质量对。

保存：
- N_pair_total。
- N_common_ok。
- A、B 各自可测集合。
- 共同可测 query 集合及摘要。
- 每个排除 query 的双方状态及原因。
- 共同集覆盖率。

对于共同可测 query：

d_i = score_B_i - score_A_i

输出：
- mean_A_on_common。
- mean_B_on_common。
- mean_difference。
- query_level_differences。
- wins、losses、ties。
- excluded_or_unknown。

mean_difference 使用同一共同集。
不能用双方各自不同可测集的均值相减，
再将结果称为配对差异。

质量比例和成功率差异同时显示：
- 原始比例差。
- 百分点差。

相对变化单独列示。
baseline=0 时相对变化为 null，
标记 baseline_zero，不输出 Infinity。

数据差始终保存 B-A。
direction 只影响“改善、退步”的解释及胜负：
- higher_better：正差为改善。
- lower_better：负差为改善。
- descriptive：不自动标胜负。

tie_tolerance 在分析前固定。
tolerance=0 时采用精确相等。
不得根据差异大小临时调整平局标准。

不同系统配置和历史运行环境的变化原样展示。
配对比较不自动构成随机对照或因果实验。

### C8. 独立单位与分组

query 是质量点估计的等权单位，
不一定是独立重采样单位。

默认使用 Registry 的 split_group 成组重采样。
同文档族、同会话和已声明关联题保留在同一组。

不得为了取得更窄区间，
将一个已声明 group 拆成独立 query。

重复生成：
- 先按 C5 形成题内均值。
- 全部 repetition 随该 query 一起进入组。
- 不把重复输出当成独立新题。

split_group 只是现有分组依据，
不自动证明所有跨组依赖已排除。

保存 independence_basis_ref 和以下限制：
- 来源执行身份未核验。
- 近重复关系未检查。
- 分组依据不足。
- 同组跨题依赖。
- 数据是否能代表声明的推断总体。

存在已知跨组关系但未合并，
不能继续声称独立组推断有效。

本轮不修改 Registry 分组。
需要修复时通过新的数据版本处理，
当前分析保留描述性结果和推断限制。

只有当固定数据证明各组为单个 query，
且独立采样假设得到明确声明和依据时，
才能使用独立 query 的二项区间。

一个保守大组不能拆开增加有效组数。
实际有效组只有一个时不计算抽样置信区间。

### C9. 统计区间方法

statistics_policy 包含：
- confidence_level：默认 0.95。
- method。
- inference_population。
- sampling_basis_ref。
- independence_basis_ref。
- min_independent_units：默认 30，至少为 2。
- bootstrap_resamples：默认 10000。
- seed。
- comparison_families。
- implementation_ref。

默认值是本轮操作策略，
不是对所有业务有效的样本量证明。
正式需求可以提高要求，
不能根据观察到的结果临时降低门槛。

本轮支持：

1. none_v1
   - 只给描述性结果。
   - 不生成抽样置信区间。

2. clopper_pearson_v1
   - 单方二元比例的精确二项区间。
   - 每个独立 query 恰好一个已知二元结果。
   - 使用成熟统计库的 exact 二项实现。
   - 不用于相关 query、重复均值或主张微平均。

3. marginal_cp_difference_v1
   - 在共同可测独立 query 上计算双方边际比例区间。
   - 为 A、B 分配各 alpha/2。
   - 差异区间为 [L_B-U_A, U_B-L_A]。
   - 使用联合界控制总覆盖，允许双方同题相关。
   - 结果保守，不声称等同高效配对比例区间。

4. paired_cluster_bootstrap_v1
   - 适用于 query 分数均值及 A/B 均值差。
   - 按完整 group 有放回重采样。
   - 一次抽取 G 个组，G 为当前有效组数。
   - 一个组被重复抽中时，其完整成员重复进入。
   - 每次重新计算 query 等权均值。
   - A/B 使用同一批抽中的组，保留配对关系。
   - 使用固定 percentile 区间和版本化分位数规则。

分组大小不同时，
不能先取每组均值再等权平均，
除非目标量已明确改为“组等权均值”。

默认目标仍是 query 等权。
每次重采样按抽中组内的实际 query 数重新计算。

随机实现使用固定版本的 NumPy PCG64。
每个 series、slice、comparison 和 method
由基础 seed 与稳定身份摘要派生子 seed。
结果不随循环顺序改变。

统计实现使用锁定版本的 SciPy、NumPy。
精确方法、bootstrap 方法、随机实现、
分位数及浮点转换版本写入结果。
不自行重写概率分布算法。

点估计保留 Decimal 或精确计数。
统计库需要浮点时明确记录转换；
拒绝溢出、NaN 或 Infinity。

统计区间针对其明确可测集合和抽样假设。
存在总体缺失时，
同时展示总体识别范围及可测集覆盖。

它不能修复：
- 错误参考或裁判偏差。
- 非随机缺失。
- 不代表目标总体的样本。
- 未处理的分组依赖。

### C10. 小样本、稀有事件和多重比较

IntervalEstimate 包含：
- status：ok、not_requested 或 inconclusive。
- interval_kind：sampling_confidence_interval。
- method、implementation_ref。
- lower、upper。
- confidence_level。
- effective_alpha。
- estimate_scope_digest。
- N_query、N_groups。
- sampling_unit。
- resampling_count、seed：适用时提供。
- assumptions。
- limitations。
- reason_code。

status 非 ok 时，lower、upper 为 null。
描述性点估计仍可以保存。

以下情况区间为 inconclusive：
- 有效独立单位少于固定要求。
- 独立性依据不足。
- bootstrap 分布退化。
- 重采样尾部支持不足。
- 统计计算无法合法完成。
- 方法与数据类型不匹配。

bootstrap 不采用失败后自动切换方法。
如果需要替代方法，必须在 policy 中预先声明。

对二元比例：
- 零次观测错误仍有非零风险上界。
- 全成功或全失败使用精确二项区间处理。
- 不能用全零错误的退化 bootstrap 输出 [0,0] 风险区间。

对分组相关的稀有事件：
本轮 bootstrap 不能提供可靠的罕见风险上界。
没有满足条件的方法时标记 rare_event_inference_unavailable，
保留频数，不伪装为精确二项独立样本。

对于 bootstrap，
每侧理论尾部重采样数：

bootstrap_resamples × effective_alpha / 2

本轮要求至少 20。
不足时标记 resampling_tail_insufficient。
这是数值稳定性检查，
不是统计有效性的充分证明。

comparison_families 在分析前固定：
- family_id。
- 全部 comparison 和 primary slice 成员。
- family_alpha。
- adjustment：none 或 bonferroni。

bonferroni：
- m 为预先固定的族成员数。
- 每项使用 alpha=family_alpha/m。
- 不因某项结果缺失而缩小 m。
- 方法内部需要分配两侧或两个边际时继续分配。

未调整的多个区间必须声明 pointwise，
不能称为同时具有 95% 总体置信保证。

探索性切片和多次配置搜索单独标记。
dev 选择结果不自动获得 holdout 结论。

本轮不提供 p-value、显著性星号、
自适应停止或复杂多重检验框架。
后续门禁不能从“区间不跨零”直接替代全部业务条件。

### C11. 切片、耗时与计量摘要

SliceSpec 包含：
- slice_id。
- 固定字段及允许值集合。
- selection_basis。
- inference_role。
- comparison_family_id：适用时提供。

本轮切片字段来自固定数据：
- split。
- effective_quality_tier。
- origin。
- 明确存在且类型合法的 TestCase 标签。

缺少字段进入 missing_value，
不能从答案内容猜测语言、难度或任务类型。

默认分别展示有效质量等级和来源。
合成、人工核心及其他来源不能无说明地混合成一个质量分。

总体汇总允许生成，
但必须同时显示构成及各组结果，
并明确是当前固定混合比例下的描述统计。

重叠切片分别计算，
不把多个切片的样本数相加作为总体数量。

耗时和 usage：
- 只汇总口径相同的合法指标。
- 成功、失败、超时运行分层保留。
- 不使用导入或评分耗时替代来源 E2E。
- 不将未完成请求当作零耗时。
- 币种、scope、计量 basis 和 pricing 不兼容时分开。

经验分位数采用 nearest-rank：
第 ceil(p×N) 个排序值。

p50、p95、p99 显示实际可测 N。
尾部支持以 N×(1-p) 记录。
小于 10 时标记 tail_support_low，
不把不稳定经验尾部值称为已验证 SLO。

本轮不给尾延迟分位数计算抽样区间。
均值区间也不能冒充 p99 区间。

UsageQuantity 仅按已明确的单一 scope 进行汇总。
不自动合并总量与细项，
不将未知计量填零。

跨来源的运行总成本、
CostPerSuccessfulTask 和索引摊销
需要完整 scope 与去重协议，留待后续专项确认。
本轮只展示已知量、覆盖和来源关系。

### C12. 结果、公开接口与错误

AnalysisResult 包含：
- schema_version、contract_profile。
- analysis_id。
- request_ref、policy_ref。
- preparation_ref、execution_result_ref。
- 固定版本及实现引用。
- AnalysisRow 清单。
- 系列及切片聚合。
- 覆盖和全部非 ok 原因。
- query 分数及集合摘要。
- 主张微平均。
- 缺失识别范围。
- ComparisonResult。
- IntervalEstimate。
- 兼容性和独立性检查。
- limitations。
- gate_evaluated：本轮固定为 false。

ComparisonResult 包含：
- comparison_id、series_id。
- arm 身份。
- compatibility_report。
- 单方与共同可测集合。
- 配对差异和胜负。
- 全部排除原因。
- 抽样区间。
- 缺失识别范围。
- 状态及限制。

analysis_id 是规范结果排除自身字段后的 SHA-256。
操作时间和主体放入独立回执。
相同固定输入及实现产生相同内容结果。

公开入口：

create_reporting(
    artifact_store,
    dataset_registry,
    runner,
    evaluators
) -> Reporting

analyze(
    request_ref,
    access_context
) -> AnalysisResultRef

load_analysis(
    result_ref,
    access_context
) -> AnalysisReader

AnalysisResultRef 包含：
- result_ref。
- receipt_ref。

analyze：
- 当前授权读取固定输入。
- 核验全部根结果槽位。
- 按固定策略计算。
- 保存明细和完整分析清单。
- 不启动新评分。
- 不生成 GateDecision。

load_analysis：
- 检查当前权限、内容摘要及结果数量。
- 返回完整只读结果。
- 不自动替换执行快照或策略。

公开错误：

| reason_code | 含义 |
|---|---|
| reporting.invalid_input | 请求或统计配置非法 |
| reporting.unsupported_schema | 不支持的契约或方法 |
| reporting.reference_mismatch | 实验、评分或版本关联错误 |
| reporting.result_set_incomplete | 根结果集合结构不完整 |
| reporting.duplicate_slot | 同一槽位存在重复当前结果 |
| reporting.series_incompatible | 系列内部口径非法混合 |
| reporting.limit_exceeded | 规模或计算限额超出 |
| reporting.access_denied | 当前读取或保存未授权 |
| reporting.integrity_mismatch | 引用、摘要或数量损坏 |
| reporting.storage_failure | 无法读取或提交分析结果 |
| reporting.internal_error | 分析程序故障 |

结构非法不能当作普通测量缺失。
统计方法合法但证据不足，
通过 IntervalEstimate.inconclusive 表达。

limits 默认：
- max_rows：100000。
- max_series：1000。
- max_slices：100。
- max_comparisons：1000。
- max_total_resample_group_draws：100000000。
- max_analysis_bytes：67108864。

预检重采样量和输出大小。
超限不减少 bootstrap 次数、
不删除切片、样本或缺失原因。
需要修改范围时形成新明确策略。

### C13. 报告与门禁请求

本次新增契约使用 schema_version=2.0，
contract_profile=reporting.report.c2.v1。
C1–C12 的分析语义和 reporting.analysis.c2.v1 保持不变。

GateRequest 包含：
- analysis_result_ref。
- gate_policy_ref：允许为 null，表示请求门禁但未配置策略。
- gate_implementation_ref。
- provenance_ref。

ReportRequest 包含：
- analysis_result_ref。
- gate_requested。
- gate_decision_ref：未请求时为 null；已请求时必须提供。
- report_policy_ref。
- renderer_implementation_ref。

GateDecision 必须绑定同一 AnalysisResult。
请求门禁但未提供策略时，先形成明确的 inconclusive 决定；
不能将未评价门禁显示为 pass。

ReportPolicy 包含：
- policy_id、policy_revision。
- 固定标题、目的、变更说明和展示语言。
- 系列、切片及章节的稳定展示顺序。
- decimal_places：默认 4；百分比默认 2 位小数。
- example_policy。
- display_mode：metadata_only 或 explicit_display_views。
- display_view_bindings：默认空。
- limits。

标题等配置同样作为不可信文本转义。
展示策略可以执行后固定，不改变统计集合或门禁阈值。
同一分析可以生成不同展示版本，保留各自策略身份。

门禁和报告不修改 AnalysisResult。
AnalysisResult.gate_evaluated 仍固定为 false。
门禁结论的唯一所有者是单独的 GateDecision。

### C14. GatePolicy 与证据资格

GatePolicy 包含：
- policy_id、policy_revision。
- purpose、decision_scope。
- confidence_claim：pointwise_only 或 simultaneous。
- required_rules、optional_rules。
- 业务阈值和风险容忍依据 threshold_source_ref。
- 允许的数据 split、effective_quality_tier 和 origin。
- 允许的评分来源及身份保证等级。
- reference_requirements、calibration_requirements。
- 最低 slot 覆盖、query 覆盖、query 数及独立组数。
- comparison_families。
- limits。

required_rules 必须非空。
合法结构但缺少业务阈值、来源、样本要求或完整必选清单，
记 gate_profile_incomplete，不能 pass。
非法数值、未知字段或矛盾规则属于 invalid_input。

不内置通用业务分数、覆盖率、样本量或非劣 margin。
统计方法已有的最低独立单位要求仍适用；
GatePolicy 可以提高要求，不能绕过方法条件。

每个规则绑定：
- rule_id、kind。
- 精确 series_id、arm 或 comparison_id、slice_id。
- statistic 及单位。
- estimand：finite_benchmark 或 sampling_population。
- operator、threshold，或非负 noninferiority_margin。
- threshold_source_ref。
- 该规则的证据资格要求。
- 对应的固定比较族及区间身份，适用时提供。

kind：
- absolute_bound：质量底线、风险上限或资源上限。
- noninferiority：B 相对 A 的非劣要求。
- hard_violation：任务原有硬性权限或安全约束。

重要切片逐条列为 required_rules。
未知切片字段、空切片和全不适用切片不能视为已满足。
optional_rules 只影响解释，不能抵消必选失败。

正式门禁要求分析策略及 GatePolicy
与 Runner manifest.reporting_intent 中的固定引用一致。
后验阈值、探索性切片和 dev 选择不能伪装为 holdout 验证。
本地冻结不能证明从未看过结果，相关来源和限制必须显示。

执行和数据身份、计划必选输入状态、实际审定范围、
指标口径、采样依据和独立性分别检查。
Planner 的必选输入缺口不能由报告层清除。

校准规则：
- 确定规则不冒称模型裁判，不适用的模型校准项给出依据。
- human 审定仍保留 local_import_only 和身份真实性限制。
- 只有 policy 明确接受该保证等级时，才能用于相应本地门禁。
- automated 或 unknown 来源不能自动当作已经校准。
- 本轮不生成或认证裁判校准报告。
- 需要校准但当前契约不能核验证据时，
  相关规则为 inconclusive，原因 calibration_evidence_unavailable。
- 一个文件引用、approved 标签或自报 confidence 不能解除该缺口。

证据资格不足影响对应规则。
不能让不合格系列的高分为其他规则提供通过依据。
本地门禁结论只覆盖声明数据、规则及来源保证，
不宣称已获生产发布授权。

### C15. 规则判定与三态组合

先核验输入结构、关联、当前权限及完整性。
这些失败返回操作错误，不生成可信业务决定。

再对每项规则形成 GateRuleResult：
- rule_id、required。
- status：pass、fail 或 inconclusive。
- evidence_eligibility 及逐项检查结果。
- numerator、denominator、覆盖、query 数和组数。
- 实际点估计。
- decision_range：区间端点及其明确类型，或 null。
- threshold、operator、unit、source_ref。
- 全部证据引用、reason_codes 和 limitations。

不得用展示舍入值判定阈值。

finite_benchmark：
- 只判断已冻结基准和运行槽位上的结果。
- 完整可测时使用精确统计值作为 [x,x]。
- 有缺失时，只允许使用 C6 已合法形成的识别范围。
- 适用性未决、缺失无合法界或覆盖低于业务要求时，
  该规则为 inconclusive。
- 允许最坏情况下仍满足约束的有限基准结论，
  但必须显示未知数及识别范围，不声称是总体置信保证。
- 资源均值缺少合法取值上界时，不能补造最坏情况上界。
- 资源经验分位数只支持完整固定测量集合上的约束；
  保留测量层级、样本数及 tail_support_low。
- 经验 p99 通过不能表述为已验证总体 p99 SLO。

sampling_population：
- 要求明确采样总体、来源依据、独立单位和样本要求。
- 当前方法只支持 C9 已声明可用的均值或二元比例区间。
- 要求目标范围内全部适用 query 及预期重复均完整可测，
  A/B 适用集合一致；适用性不能未决。
- 共同可测子集的 CI 不能单独支持全目标总体的门禁。
- 缺失识别范围不能冒充抽样 CI，
  也不将两者简单相加形成一个新置信区间。
- 区间 status 非 ok、选择偏差未控制或独立性依据不足，
  为 inconclusive。
- 本轮没有尾分位数总体推断；对应要求为 inconclusive。

对合法判定范围 [L,U]：
- 要求 x>=t：L>=t 为 pass；U<t 为 fail；其余 inconclusive。
- 要求 x<=t：U<=t 为 pass；L>t 为 fail；其余 inconclusive。

noninferiority：
- 原始差异始终为 B-A。
- higher_better：benefit_difference=B-A。
- lower_better：benefit_difference=A-B，区间端点同时反号并换位。
- descriptive 指标不能自动进行非劣判断。
- margin 为同单位绝对差，比例差使用百分点明确展示。
- benefit 区间下界 >= -margin 为 pass。
- benefit 区间上界 < -margin 为 fail。
- 其余为 inconclusive。
- estimand=finite_benchmark 使用合法完整差异或 C6 差异识别范围；
  estimand=sampling_population 使用满足条件的配对统计区间。

统计结论必须使用分析时已固定的比较族。
多项统计必选规则若要声明联合置信保证，
必须由分析策略预先纳入覆盖全部目标的 Bonferroni 族，
并保持固定 m。
本轮不在门禁阶段重新分配 alpha 或缩小比较族。
未满足联合声明要求时保留 pointwise 说明，
相应联合门禁为 inconclusive。

hard_violation：
- 规则绑定任务原有的具体权限/安全维度和完整审查范围。
- 消费对应、有效、同 Run 的 RubricDimension 等 Judgment，
  并回查该维度的固定规则及实际审定依据。
- 任一已核验违反该硬约束的结果即可 fail，
  即使其他槽位尚未完成。
- 普通 TaskSuccess=fail、正则命中或未知结果，
  不能单独证明权限或安全违规。
- 没有观察到违规，只能在全部必需范围均已有效检查时
  对有限基准给出 pass；其余 inconclusive。
- 零次观测违规不等于总体风险为零。

硬性违规短路仍要求本条证据的来源、审定、关联和完整性合格；
不能绕过该条本身的校准或审定缺口。
已核验硬性违规的 fail 不被其他统计证据不足抵消。
其余规则须先满足自身证据资格，再判断阈值。

GateDecision 包含：
- decision_id、analysis_result_ref、gate_policy_ref。
- implementation_ref、manifest_binding_status。
- decision_scope 和 inference_claim。
- 全部 GateRuleResult。
- overall_status。
- blocking_reasons、limitations。
- source_assurance。

组合规则：
1. 任一必选规则 fail，overall_status=fail。
2. 没有必选 fail，但存在必选 inconclusive，
   或 profile 不完整，overall_status=inconclusive。
3. 仅当非空必选清单全部 pass，overall_status=pass。

不使用加权总分抵消必选约束。
不请求门禁时没有 GateDecision，报告显示 not_requested，
该展示状态不属于三态结论。

decision_id 为规范决定排除自身字段后的 SHA-256。
操作主体和时间放在独立回执，不进入业务身份。

### C16. ComparisonReport 与 Markdown/JSON 一致性

ComparisonReport 包含：
- schema_version、contract_profile、report_id。
- analysis_result_ref、report_policy_ref。
- gate_requested、gate_decision_ref。
- renderer_implementation_ref。
- 目的、实际变更和适用范围。
- 数据、语料、Run、参考、评分、环境及统计版本。
- 来源构成、审定状态及身份保证。
- 全部配置系列和切片的结果或不可用原因。
- 完整覆盖、异常、共同集和排除摘要。
- 配对差异、胜负、区间与识别范围。
- 已知耗时、usage 及各自覆盖和口径。
- example_manifest_ref。
- limitations 和 deferred_sections。

report_id 为规范报告排除自身字段后的 SHA-256。
不将生成时间写入正文身份；操作时间在回执中保存。

Markdown 按以下顺序展示：
1. 目的、变更、范围、演示身份及完整版本。
2. 门禁总状态、逐项条件和阻塞原因。
3. 样本构成、审定来源、覆盖和全部异常。
4. E2E 与各质量维度，七阶段按已有系列组织。
5. A/B 共同集差异、胜负、统计区间及缺失范围。
6. 预设重要切片和探索性切片。
7. 耗时、已知计量、测量范围及未知计量。
8. 固定规则选取的失败、退步、未知及硬性违规样例。
9. gold、裁判、映射、采样与来源限制。

未接入的引用、拒答、澄清、模态、安全专项等，
标明 not_requested、unavailable 或 out_of_scope 及原因。
不为补齐章节产生新指标或虚构诊断。

来源类型分层与总体混合构成同时可见。
未知数量和覆盖在主表展示，不能只藏在附件。
系统失败和测量失败分别展示。
完整阶段诊断、假设及干预在本轮为 out_of_scope。

JSON 保存精确计数及未舍入 Decimal 字符串。
null 必须伴随状态或原因，不输出 NaN、Infinity 或虚假零值。
Markdown 百分比、百分点和原始单位明确区分。
全部结论来自同一 ComparisonReport 和 GateDecision，
不能在模板中重新计算阈值、平均数或统计区间。

汇总表只承担展示；
完整行和排除清单放入可校验的 JSON/JSONL 伴随文件。
行顺序按稳定身份固定，不能静默截掉异常结果。

### C17. 样例选择与受控展示

ExamplePolicy 固定：
- selection_version=hash_order_v1。
- seed。
- bucket_order：hard_violation、regression、task_failure、unknown。
- 各 bucket 配额：默认各 5。
- max_examples：默认 20，至少等于配额总和。
- 允许的系列、切片及精确判断条件。
- 每个样例允许展示的字段与 DisplayView 引用。

候选只来自当前固定分析：
- hard_violation：C15 已核验的具体违规。
- regression：按固定 direction 和 tie_tolerance 确定的配对退步。
- task_failure：有效 TaskSuccess 或明确任务维度失败。
- unknown：未完成、不可测、不确定或测量错误。

候选不按答案内容临时挑选。
每桶使用长度前缀编码的 seed、bucket、CaseKey、
arm/comparison 和 series 身份计算 SHA-256 排序。
摘要相同时按规范身份排序。
按固定桶序选取；同一 case/arm 比较及目标范围去重，
保留全部命中标签和跨桶去重数量。
不从其他桶补满剩余配额。

ExampleManifest 保存：
- 完整策略及分析引用。
- 每桶候选数、已选数、未展示数及去重数。
- 每个已选项的稳定身份、选择依据及双方状态。
- Judgment、主张、证据、实际上下文和源 revision 的授权引用。
- display_view_ref 或 unavailable 原因。
- limitations。

统计结论使用全量分析，不使用样例子集。
硬性违规即使未进入展示配额，也继续影响门禁。
样例页始终显示候选总数及“仅展示部分”的说明。

metadata_only 为默认：
只展示受策略允许的稳定展示标识、指标、状态、
简短 reason_code 及受控引用，不嵌入原 query、gold 或上下文。

explicit_display_views：
- 只消费显式指定的已有受控展示材料。
- DisplayView 绑定 source_ref、源 revision、作用范围、
  content_ref、变换/脱敏 revision 和来源记录。
- 检查绑定、当前权限及目标访问域。
- 材料中的身份或“已脱敏”声明不能自行授予导出权限。
- 本轮不实现自动敏感信息识别或脱敏引擎。
- 缺少合法展示材料时保留样例元数据及原因，
  不使用原文补位，不从可见摘要重建隐藏内容。
- 脱敏改变事实时，在展示中标明不能据此独立复核原结论。

原文中的指令始终作为数据处理。
转义 HTML、表格分隔符和 Markdown 控制字符；
代码块使用安全围栏，不生成可执行 HTML。
报告只产生受控相对文件链接，
不自动加载网络图片、远程 URL 或执行脚本。

### C18. 报告包、公开接口、导出与错误

ReportBundle 包含：
- schema_version、contract_profile、bundle_id。
- analysis_result_ref、gate_decision_ref、report_policy_ref。
- ComparisonReport、ExampleManifest 引用。
- file_entries。
- 总文件数、总字节数及版本。

清单中的数量和总量只统计 file_entries，不包含清单自身。
导出限额另计 manifest.json 的实际编码大小和一个文件，
不把自身大小写回清单形成递归。

每个 file_entry 包含：
- 固定相对路径。
- media_type。
- ArtifactRef、sha256、size_bytes。
- 文件角色。

固定输出角色：
- report.json：ComparisonReport。
- report.md：同一语义报告的 Markdown。
- rows.jsonl：完整规范结果行。
- comparisons.json：配对、区间及全部排除明细。
- examples.json：选择清单及允许展示的元数据。
- display/ 下显式允许的展示文件，适用时提供。

bundle_id 为规范清单排除自身字段后的 SHA-256。
清单不引用自身 ArtifactRef。
子文件不嵌入最终 bundle_id，
避免报告和清单之间形成摘要循环。

使用同一 Store、access_domain 保存，
不将摘要当作访问能力，不自动做跨域降密。
原材料保持原授权引用，不自动打包全部原文。

扩展公开入口：

decide_gate(request_ref, access_context) -> GateDecisionRef
render_report(request_ref, access_context) -> ReportBundleRef
load_report(bundle_ref, access_context) -> ReportReader
export_bundle(bundle_ref, output_root, access_context) -> ExportReceipt

GateDecisionRef 包含 result_ref、receipt_ref。
ReportBundleRef 包含 bundle_ref、receipt_ref。
均复用原 create_reporting 构造入口，不增加业务模块依赖。
构造入口新增可选关键字 local_export_policy=None，兼容原调用。
LocalExportPolicy 包含允许导出的绝对根目录及访问域映射，
由可信 CLI 或宿主配置构造，不能从被测材料接受授权声明。
未提供配置时不允许文件导出，仍可保存和读取 Store 报告包。

decide_gate 只读取固定分析及既有证据，不补跑评分。
render_report 只消费明确 GateDecision，不隐式重新决定门禁。
load_report 校验完整清单、数量、文件角色及引用，
每次读取继续检查当前权限和完整性。

export_bundle：
- output_root 来自显式可信调用参数，不从答案或源材料取值。
- 使用调用主体的操作系统权限及配置的允许导出目录。
- 重新检查报告域和展示材料的当前授权。
- 只导出清单列明的报告文件，不递归复制 Store。
- 相对路径拒绝绝对路径、盘符、..、设备名及重复规范路径。
- 拒绝导出路径中的符号链接或 junction。
- 在允许目录内创建独占暂存目录，逐文件流式校验并写入。
- manifest.json 最后写入，内容为 ReportBundle 清单。
- 完成后在同卷将暂存目录移为 report-<bundle_id>。
- 目标已存在时仅在完整清单和全部字节均相同后幂等返回。
- 不覆盖不同内容；残留暂存目录不表示导出成功。
- 不提供 ZIP、HTML 或外部发布功能。

ExportReceipt 保存安全目标位置、bundle_ref、
文件数、字节数、完成状态及独立操作时间，并经 Store 保存。
若目标已提交但保存回执失败，返回 export_failure 及安全提交阶段；
重试核验同内容目标后补记回执，不再次覆盖文件。
导出不改变门禁决定；已导出文件属于导出时快照，
后续 Store 撤权不能撤回已有副本。

新增操作错误：
- reporting.gate_binding_mismatch。
- reporting.report_binding_mismatch。
- reporting.export_target_invalid。
- reporting.export_conflict。
- reporting.export_failure。

沿用 C12 的 invalid_input、unsupported_schema、
access_denied、integrity_mismatch、limit_exceeded 和 storage_failure。

业务证据不足通过 GateDecision.inconclusive 表达；
权限失效、损坏及无法提交目标报告返回操作错误。
错误信息保留安全引用和已完成提交阶段，
不泄露原文或凭据。

报告限额默认：
- max_examples：100。
- max_display_codepoints_per_example：2000。
- max_bundle_files：200。
- max_bundle_bytes：67108864。

导出限额按全部文件总量计算。
预检超限明确失败，不静默删结果、删样例或截断展示材料。
改变范围需形成明确的新展示策略。

## 2. 计算与提交流程

读取请求和分析策略
→ 授权加载明确实验及执行快照
→ 核验全部槽位和 Judgment
→ 建立规范结果行
→ 检查系列及比较兼容性
→ 计算覆盖和原因
→ 形成题内及 query 聚合
→ 计算识别范围
→ 建立 A/B 共同可测集
→ 按固定独立单位计算区间
→ 保存明细
→ 最后提交 AnalysisResult 清单
→ 保存回执并返回

单项合法统计证据不足不阻止形成诊断分析。
输入损坏、非法关联或无法完整读取则明确失败。

## 3. 保存、重复处理与恢复

全部业务内容通过 Store 保存。
不修改 Runner、Registry 或 Evaluator 产物。

先保存规范行、query 结果和统计明细，
再核验数量及引用，
最后提交完整分析清单。

清单提交前的部分统计文件不代表完整分析成功。
清单已提交但回执失败时，
重复处理可以核验并复用相同内容，
生成新的操作回执。

统计 seed 和实现版本属于结果身份。
不能重复计算后选择更窄或更有利的区间。

本模块没有跨任务恢复调度器。
发生中断时重新读取固定输入并进行确定性计算。
已提交不可变明细可以精确复用。

缓存不绕过当前权限和完整性。
不扫描私有目录寻找“相似报告”。

## 4. 实现与安全安排

允许的本轮业务依赖：
- Artifact Store / Contracts r002。
- Dataset Registry / Contracts r002。
- Runner / Contracts r002。
- Evaluators / Contracts r001。

不直接导入 Adapters、Mapper、Review 或 Planner。
需要的对象由 Runner 公共 Reader 暴露，
沿用对象所有者定义，不复制核心契约。

不依赖 CLI。
Diagnostics 本轮不注入、不调用。

计划入口：
- rag_eval.reporting.contracts。
- rag_eval.reporting.api。

内部职责可以分为：
- 结果集合和系列校验。
- 覆盖与 query 聚合。
- 配对及独立分组。
- 统计库适配。
- 分析产物保存。

统计库版本在实现阶段锁定，
并记录方法及随机实现。
不能默认不同版本数值结果完全相同。

日志只保存安全身份、样本数量和原因。
原答案、主张和实际上下文保留授权引用，
不为统计复制到公开日志。

本轮没有网络、模型调用或数据库。
尚未验证大规模统计性能。

## 5. 验证计划

以下为计划验收，尚未执行：

| 场景 | 应观察到的结果 |
|---|---|
| 系统业务失败且已有合法 0 分 | 计入 N_ok 和质量分母 |
| Runner completed，但 Judgment 为 error | 计入 N_error，不计 N_ok |
| 没有 Judgment | 保留 unfinished 行和根槽位 |
| 适用性未决 | 保留保守覆盖分母及未决数量 |
| N_applicable=0 | 覆盖为 null，不输出满分 |
| 一题三次重复，其中一次缺失 | query_score 未知，不只平均两次 |
| 两题答案长度差异很大 | query 宏平均不按主张数量加权 |
| 主张微平均 | 单独列示完整计数和覆盖范围 |
| A、B 各自可测集合不同 | 显示单方集合和共同集 |
| 双方非配对均值不同 | 不冒充共同集配对差异 |
| 一方所有回答都没有事实主张 | Faithfulness 不自动高分或胜出 |
| 多个指标实例口径不同 | 不按显示名称直接合并 |
| baseline=0 | 相对变化为 null |
| 缺失任务可能改变总体成功率 | 显示识别范围 |
| 有识别范围和 95% CI | 分开命名和解释 |
| 同题重复或同会话多题 | 不作为新增独立采样 |
| 只有一个有效组 | 保留点估计，区间 inconclusive |
| 二元样本零次观测错误 | 精确区间保留非零风险上界 |
| 分组稀有事件不满足方法条件 | 不套用独立二项区间 |
| bootstrap 分布退化 | 不输出伪确定区间 |
| 改变计算循环顺序 | 固定子 seed 保持结果一致 |
| 多重比较中部分结果缺失 | 不缩小预设 family 的 m |
| 重叠切片 | 不相加成为总体样本数 |
| p99 尾部支持不足 | 保留样本量及 tail_support_low |
| usage 总量与明细重叠 | 不自动相加 |
| 旧执行快照有更高分 | 不自动替换请求指定快照 |
| Store 撤权或损坏 | 明确失败，不伪装成统计缺失 |
| 重采样规模超限 | 不静默减少次数或删样本 |
| 重复处理同一固定请求 | 同内容结果、独立操作回执 |

使用固定小型数据构造可手算结果，
验证计数、配对集合、题内聚合和识别范围。

精确区间与成熟统计库及公开公式交叉核验。
bootstrap 检查分组、配对及固定随机性，
不只断言某个浮点数恰好相等。

真实集成使用 Runner 的 completed 和 partial 快照，
包括缓存、恢复、错误 Judgment 与未完成任务。

应用测试、统计校验和集成验收尚未执行。

## 6. 设计进度与后续边界

进度摘要：Confirmed / r004（包 r022）；业务单元修订仍按下表记录。

| 设计单元 | 审阅状态 | 内容修订 | 位置 |
|---|---|---|---|
| 职责与边界 | Confirmed | r002 | 第 1 节 |
| 输入、系列和覆盖 | Confirmed | r001 | Contracts / C1–C4 |
| 聚合、缺失范围和配对 | Confirmed | r001 | Contracts / C5–C7 |
| 分组及统计区间 | Confirmed | r001 | Contracts / C8–C10 |
| 切片与计量摘要 | Confirmed | r001 | Contracts / C11 |
| 分析结果与接口 | Confirmed | r001 | Contracts / C12 |
| 处理、保存和安全 | Confirmed | r001 | 第 2–4 节 |
| 分析验证计划 | Confirmed | r001 | 第 5 节 |
| 报告请求、结构与展示 | Confirmed | r002 | Contracts / C13、C16 |
| GatePolicy 与证据资格 | Confirmed | r002 | Contracts / C14 |
| 规则判定与三态组合 | Confirmed | r002 | Contracts / C15 |
| 样例选择与受控展示 | Confirmed | r002 | Contracts / C17 |
| 报告包、接口与导出 | Confirmed | r002 | Contracts / C18 |
| 报告流程与新增验证 | Confirmed | r002 | 第 7 节 |

C3 扩展：[诊断与干预报告](diagnostic-reporting.md)，Confirmed / Complete / r001；
与本表的 C2 单元分开，不作为 C2 隐藏依赖。

本模块的已声明 C2 范围为 Confirmed / Complete。
C1–C12 保持 r001，C13–C18 为 r002。
文档修订为 r004；公开契约总体修订保持 r002。

后续范围：
- 自动裁判校准、元评测与来源真实性认证。
- 在线监控、Oracle、完整阶段诊断和干预解释。
- 缺失与抽样误差的联合推断。
- 总体尾分位数推断与复杂多重检验。
- 跨来源运行总成本及 CostPerSuccessfulTask。
- 自动脱敏、交互报告、远端发布和 Web 服务。

业务策略实例仍需提供具体阈值和证据要求。
缺少实例时可以诊断展示，不能宣称正式门禁已经通过。
这不阻塞 C2 门禁引擎的契约与缺失行为实现。

C2 的当前就绪状态见[索引](../../index.md#5-候选推进范围)，
确切实现基线见[交接 C2-H003](../../handoffs/c2-r003.md)。
应用实现与验收尚未执行。

公开接口、统计含义、样本与分母、
依赖方向、保存、权限和验收行为变化需要确认。
私有实现可在已确认边界内决定。

## 7. 报告、门禁与导出流程及验证

### 7.1 固定输入到最终报告

加载已提交 AnalysisResult
→ 核验分析、执行快照及版本关联
→ 按 GateRequest 核验策略和逐项证据资格
→ 保存全部规则结果及 GateDecision
→ 固定 ExamplePolicy 并选择样例
→ 核验受控展示材料
→ 生成 ComparisonReport 及机器明细
→ 从同一内容渲染 Markdown
→ 保存全部子文件并核验清单
→ 最后提交 ReportBundle
→ 保存操作回执
→ 按需执行独立导出

不请求门禁时跳过决定步骤，明确显示 not_requested。
有合法业务缺口时仍可生成带 inconclusive 的完整报告。
结构损坏或权限失败不能被转换成“缺少几个样例”。

### 7.2 提交、重复处理与恢复

GateDecision 和 ReportBundle 分别是不可变提交边界。
正文、明细和样例文件先提交，最终清单最后提交。
清单提交前的子文件不代表完整报告成功。

若门禁已提交但渲染失败：
- 保留决定及其引用。
- 重复渲染使用相同决定，不重算更有利的门禁。

若完整清单已提交但回执失败：
- 核验后复用同内容清单并补记新的回执。

固定输入、策略、随机实现及渲染版本产生相同业务内容。
当前时间和文件导出位置不参与业务摘要。
模板修复或脱敏材料变化形成新报告版本，
不修改旧报告和原分析。

导出中断只留下非正式暂存目录。
恢复从固定 bundle 重新验证和导出，
不扫描其他目录挑选“看起来完整”的结果。
并发同内容提交依赖 Store 的幂等及导出目标冲突校验。

### 7.3 计划验收

以下为新增计划，尚未执行：

| 场景 | 应观察到的结果 |
|---|---|
| 未请求门禁 | 显示 not_requested，不出现 pass |
| 请求门禁但缺少业务策略 | inconclusive / gate_profile_incomplete 或 gate_policy_missing |
| 必选清单为空或阈值来源不全 | 不能通过 |
| 舍入显示刚好等于阈值 | 使用未舍入值决定 |
| 100 题中 10 题缺失，已知成功 85 题 | 有限基准范围 [0.85,0.95]，不冒称 CI |
| 上例质量底线 0.84 且其他条件满足 | 仅可通过对应有限基准规则，保留 10 题未知 |
| 上例要求抽样总体质量结论 | 可测子集 CI 不能解除缺失，规则 inconclusive |
| 非劣 benefit 区间跨越 -margin | inconclusive |
| lower_better 非劣指标 | 同时反转差值方向和区间端点 |
| 重要切片失败，总体高分 | 门禁 fail |
| 必选规则通过但另一必选缺少校准 | inconclusive |
| 一个有效权限违规，其他样本缺失 | hard_violation fail，保留全部缺口 |
| 只有 TaskSuccess=fail 或正则命中 | 不自动标记权限违规 |
| 零次观察违规但审查范围不完整 | 不能 pass |
| 经验 p99 满足阈值 | 仅能声明固定集合经验约束满足 |
| p99 总体 SLO 要求 | 当前方法不足，inconclusive |
| 多重比较缺少预设联合控制 | 不伪造联合置信通过 |
| 更换后验阈值或排除规则 | 标出策略不绑定，正式门禁不能通过 |
| 样例超出配额 | 显示未展示数，完整失败继续进入门禁 |
| 更改读取遍历顺序 | 样例选择与报告内容保持一致 |
| 没有受控展示视图 | 保留样例元数据，不补入原文 |
| 展示材料包含 HTML、链接或围栏 | 作为转义数据，无远程资源执行 |
| Markdown 与 JSON | 相同原始值、范围、状态和门禁依据 |
| 中途提交失败 | 没有完整 bundle 就不报告渲染成功 |
| 相同目标已有不同内容 | export_conflict，不覆盖 |
| 导出含 ..、盘符或 junction | 拒绝越界或间接导出 |
| 撤权后加载或重复导出 | 不使用旧回执绕过当前权限 |
| 全量明细超过上限 | 明确失败，不删异常或缩小配额 |

使用可手算的固定数据覆盖阈值边界、三值逻辑、
缺失范围、非劣方向及样例去重。
真实集成读取 Runner 和 Evaluators 的已提交产物，
验证完整报告及导出中的故障注入。

本次只完成设计审阅；应用代码、测试和统计验收均未执行。

