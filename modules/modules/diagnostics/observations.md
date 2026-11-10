# Diagnostics：观察、假设与规则

- 状态：Confirmed；内容修订：r001；对应 README C2–C4。
- 本文是规则细节，实体及公共入口以 [Contracts](README.md#contracts)为唯一正文。

## 1. 观测归一与绑定

处理顺序：
1. 逐项授权、完整性和 schema 校验；核对正式数据版本、用例、Run、评价包。
2. 为每个预期 binding 建立结果槽位，包括 null Run。
3. 读取真实 association_report，解析明确阶段/调用及最终输出来源。
4. 读取指定完整 Judgment、MappingBatch 和 StageObservation 集合；逐项核验 Subject/scope。
5. 生成强类型观察；缺记录为 unknown，有矛盾的合法来源为 conflicting。
6. 执行固定规则，保存命中和未满足前提；生成竞争解释与可证伪后续动作。
7. 核对完整数量、依赖和覆盖，提交 Finding 及完整批次。

不因总体 TaskSuccess=fail 就把全部维度设为 fail；
一个确定必要维度失败可使 TaskSuccess 已知失败，其余仍 unknown。
状态 error 不等价于系统错误；缺少主张审定不等价于输出没有主张。
Judgment.value 的消费按 metric profile，不能把任意 ratio<1 当任务失败。
association unknown 阻止依赖真值的确定结论，但不抹除真实运行终态。

每条导出观察记录提供方对象摘要、字段路径、目标实例、完整审查范围、
producer/config/version 以及来源保证。Finding 可回查原始证据，不能只有自然语言总结。
机械观察与已有审定冲突时保留 conflicting，不采用“更像正确”的一个。

## 2. 阶段链和“在哪里丢失”

节点为同一 Run 的明确 StageResult、ContextRecord、GenerationCall，
以及有实际构建 membership 的独立阶段输出。
边只能来自可核验 input_refs/output_refs、source_candidate_refs、
data_from 及请求绑定；parent、follows_from、时序或相同文本不足以证明内容传递。

建立有向数据路径，但不枚举所有指数数量路径：
按边传播锚点已保留/已缺失/未知的有限状态；
限制节点/边数量，记录候选边界集合，不能任取一个作为唯一根因。
有环依赖为非法；业务循环必须展开为不同实际实例。

上游 matched、下游 verified_absent 仅支持：
“指定来源锚点在声明边界两端的保留状态不同”。
只有所有中间节点和具体传递边完整、没有旁路，才能把边界收窄。
直接边可标该变换区间发生锚点损失；仍不证明变换不合理或语义事实丢失。
旁路、并行候选、替代充分来源、合规过滤和未审定新表达均为竞争解释。

## 3. diagnostic_rules_v1

下表每行是独立规则；未满足前提保留缺口，不生成肯定断言。

| 规则 | 必需观察及限定 | 支持的候选/断言 | 反证与下一步 |
|---|---|---|---|
| O01 measurement_gap | 预期指标非 ok 或实际 capture 不足；非 not_applicable | measurement；说明缺上下文/标签/来源/能力等具体缺口 | 补足实际数据后重评分，不重跑业务 |
| O02 measurement_error | 真实 Judgment.status=error 或评价操作错误投影 | measurement；本次测量失败 | 核查评价实现及版本，不能给系统判错 |
| O03 reference_limitation | Registry 对应 scope 缺审定、gold partial、冲突或 unjudged | source_reference 与 measurement；参考不充分/未决 | 新 revision 复核；partial 不是标签一定错误 |
| O04 parser_anchor_loss | 审定源锚点在固定全量 parser 文本中 verified_absent；构建关联完整 | parsing；该解析输出未保留该字面锚点 | OCR/表示变换可能保留语义；查看原文和解析输出，不宣称事实一定丢失 |
| O05 chunk_index_boundary | parser/chunk 两端可核验、上游 matched、下游 verified_absent，实际构建一致 | chunk_index；该内容集合边界的锚点损失 | index inventory 不证明向量可达性；需要独立索引探针才能判断搜索问题 |
| O06 observed_candidate_absence | 指定 retrieve scope 满足严格锚点缺失 | retrieval_filter；在该次候选范围未保留指定锚点 | 不外推全库/索引；用 Evaluator fact_scope 进一步判断语义覆盖 |
| O07 linked_context_loss | 明确同 Run 上游 matched、实际生成 context verified_absent，数据路径核验 | rerank、compression_assembly 等最窄可观察边界候选 | 中间缺记录时保留多类；尚有充分替代来源则不声称任务证据不足 |
| O08 generation_use_candidate | 原始实际上下文 CompleteEvidenceSet=1，任务/相关维度已知 fail，调用与最终输出关系核验 | generation、source_reference；充分输入下仍失败的候选解释 | 检查任务理解、参考、格式、允许推导；不能把充分组声明直接当实际充分 |
| O09 operational_failure | 明确 Run failed/timed_out/cancelled | 对已知阶段标 operational_failure；未知阶段只标运行层症状 | 不用本地观测超时替代远端状态，不推测模型或检索故障 |
| O10 citation_candidate | 已有有效 RubricDimension 审定明确引用不存在/不支持/遗漏 | citation；对应具体引用维度失败 | raw 引用 unresolved 只证明无法解析，缺引用日志不能当引用错误 |
| O11 policy_candidate | 有固定 rubric、完整实际输入和有效违规 Judgment/审定 | policy；具体范围的已核验违规 | Store 可读不证明业务可读，预期 policy_block 不是实际违规 |
| O12 rewrite_routing_candidate | 固定实际 query_transform/route 与任务维度有效审定，存在明确冲突 | rewrite_routing；该变换/路由与目标约束冲突 | 文字不同不证明改写错误；只有低召回不能触发 |
| O13 no_observed_failure | 指定任务判断为 pass，其他观察照常登记 | no_observed_failure；当前口径未观察到失败 | 不证明所有环节正确；可进入通过/随机抽样对照 |

语义“事实未覆盖”只消费 Evaluator C10 对同一 scope 的确定性结果和细节。
Mapper.verified_absent 不单独转换成 cov(f)=0；
Mapper.matched 不单独转换成 sufficient_context=true。
事实覆盖、锚点定位、来源身份、任务正确、引用支持必须各自保留。

明确反证例：上游指定锚点 absent，但已审定的其他充分组完整，
不能报告“缺此锚点导致答案必错”。保留锚点观察，降低对应假设为
counterevidenced 或 unresolved，附替代路径，并允许任务仍失败于其他原因。

## 4. 规则合成与优先级

同一类别可有多个互不相同的 scope 假设；只在 claim_code、
关键参数、范围和依赖完全相同时合并，保留全部支持和反证。
跨阶段、跨请求及跨用例不合并。多原因不互斥，计数可能大于病例数。

诊断不得因测量缺口删除其他可靠观察；但受该缺口影响的细粒度结论必须降级。
排序先把会影响可解释性的 measurement/source_reference 检查呈现，
然后区分有语义审定支持、严格字面边界、仅候选缺口；同级按固定枚举/ID。
该顺序是检查成本和可解释性安排，不是“最可能根因”。

recommended_actions 仅为：
collect_artifact、verify_binding、review_reference、review_support、
inspect_stage_boundary、plan_intervention、repair_measurement。
每条指向具体 missing requirement 或可证伪假设。
不自动修改系统参数、重复调用、创建 ReviewItem 或写回 gold。

## 5. 可手算例子与验证

用例 Q 的两项必要事实为 f1、f2，充分组 S={f1,f2}。
实际 retrieve R1 两事实均有已审定支持，context C1 仅保留 f1；
Evaluator 给 R1 CompleteEvidenceSet=1、C1=0。
路径 R1 → rerank R2 → compress C3 → C1：
- R2 未记录：候选→上下文发生供给变化，但不能只归咎重排或压缩。
- R2 两事实完整、C3 对 f2 严格缺失且存在直接数据关联：
  边界缩小到 R2→C3，仍不能断言唯一根因。
- C3 换成同义摘要，Mapper unresolved：
  保留测量缺口，不能机械地把 f2 记 0。
- C1 包含另一已审定完整充分组 S2：
  “上下文不足”不成立，另查生成/任务/参考。
- 相同 query 的另一 Run 含 f2：不能补齐当前 Run。

计划验证以上分支、wrong call、parallel spans、相同数字跨年/单位、
未知来源、partial gold、非原始上下文和规则版本变化。
验证规则的逻辑边界不证明真实诊断召回率或准确率。
