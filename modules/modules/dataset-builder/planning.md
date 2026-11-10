# 蓝图采样：覆盖、分组切分与去重

- 状态：Confirmed；内容修订：r001；用户授权代理连续完成。
- 上级契约：[README / C5](README.md#contracts)。
- 本文定义构建选择，不承担 Reporting 的统计推断。

## 1. 由事实形成可验证任务

输入为固定事实/关系/锚点集合，不使用被测系统答案挑蓝图。
Blueprint 的 constraints 显式包含实体、期间、条件、否定、
集合边界、业务时间、访问角色、答案类型与允许工具。
task_type=fact/definition/comparison/enumeration/calculation/aggregation/
compositional_reasoning/entity_chain/summary/behavior。
不能将“跨两份文档”直接标为真正多跳。

target_spec 包含必要答点的语义槽位、值域/类型、
输出要求和审核维度 ID；计算前提不自动成为必需答点。
derivation_dag 节点类型为 premise、operation、target；
每条边有目的、必要条件与允许规则。
构建时验证无环、每个 target 可由固定前提闭合、实体/单位/期间相容。
ReferenceClaim 的最终 claim_id 在求解投影中确定，蓝图使用局部 slot_id，
两者由显式映射关联，不能让模型任意添加必需答点。

关系候选只有经过规则/来源检查才可进入已验证推导。
暂未确定实体或输入读数的蓝图可放复核池，不计可用任务。

## 2. 充分路径、必要性与替代支持

sufficient_path_candidates 每项记录 path_id、
necessary_fact_slots、source_alternatives、rule_ids、
target_slots、validation_status、scope_ref。
组内不同必要事实 AND，同事实已确认来源 OR，组间 OR；
映射为 Registry sufficient_evidence_sets 时保留原语义。

必要性检查对事实角色进行，而非固定 evidence_id：
去掉一类必要事实后，检查在允许规则及已审查替代范围内
是否仍可唯一得到目标。记录 tested_scope、remaining_paths、
result=necessary/not_necessary/unknown 及依据。
模型去一跳仍答对只能提示潜在捷径，不能证明该前提不必要。

例：E_PLAN=100、E_ACTUAL=120，可由 subtract 得到差额 20。
若 E_SUMMARY 直接披露差额 20，则为另一单来源路径；
只在限制证据条件的专门任务中才能称原两事实都是必要。
自然全库任务保留替代路径，不故意把它标 irrelevant。
若宣称 entity_chain，还须证明中间实体依赖并非问题已直接暴露。

未穷举路径使用 partial 标注；
“至少一组可支持”与“全库相关度完全标注”分开。
缺少必要性依据时降为可证实的 aggregation/comparison 或待复核，
不能保留错误 difficulty 标签以填配额。

## 3. ScenarioPolicy 与覆盖计划

ScenarioPolicy 定义：
- target_count：正整数。
- hard_cells：cell_id、维度谓词、min_count、required_quality_tier、
  minimum_distinct_groups、priority、来源依据。
- soft_marginals：维度、目标比例/范围、业务权重。
- max_per_family、max_per_fact_signature、max_per_blueprint、
  max_variants_per_base、max_cross_family_edges。
- batch_size、max_batches、max_candidates、max_repairs_per_candidate。
- quota_basis：auto_validated 或 human_reviewed，分别统计不能混用。
- shortage_action：stop_with_gaps 或 block_publication；
  hard_cells 缺口永远不得被 soft 目标覆盖。

维度包括任务、源模态、表达、交互、可答性、action、语言/域、
时期/访问范围与来源难度。质量等级是发布目标，不由生成器选择。
只定义业务需要的边际及交叉格，不强制笛卡尔积全覆盖。
cell predicate 使用字段/枚举的白名单比较，不接受可执行表达式。

FeasibilityReport 保存每格：
候选来源/事实/蓝图数、可用独立组依据、上限、已知不适用、
未确定量、resource estimate、可能缺口。
没有图表来源时该格为 unavailable，不用文本题贴 chart 标签。
unknown 不计入 unanswerable 格，也不预支为 human_core。

## 4. 确定性、有限的缺口调度

先按 priority 和 hard 缺口，再按软目标缺口、来源多样性、
固定历史成本估计排序；稳定身份作为最终 tie-break。
可选成本估计只使用当前构建已允许的统计，不读取 test 系统得分。
每批选择的蓝图、selection_probability（可计算时）、候选池摘要、
seed、限制和排除原因写入 SelectionReceipt。

```text
freeze policy, source pool and initial grouping
for batch_index in range(max_batches):
    count eligible results by tier, cell and group
    if hard+soft goals reached: stop
    enumerate remaining feasible scenarios under per-source caps
    if none or budget cannot reserve: stop_with_gaps
    choose <= batch_size using frozen priority and derived RNG
    persist batch intent and scenario IDs before model calls
    evaluate results; count only actual eligible revisions
commit all selected/excluded/unknown outcomes and shortage report
```

seed 由基础 seed + 阶段/对象/批次身份摘要派生，
不依赖遍历顺序或 Python hash 随机盐。
同已提交快照可重放选择；外部模型重新调用不保证相同输出。
历史成功率只估算后续批量，不能让 1/低通过率导致无限生成。
人为删除失败后再次抽样仍计入原尝试和成本上限。

配额计数：
同蓝图十种问法可算十条表达样本，但 distinct_blueprints=1，
独立来源组最多按实际关系算一次。
输出 planned/generated/auto_validated/human_reviewed/released 五套数量，
不能以生成数量代表发布数量或独立样本量。
压力测试的均衡配额不冒称真实流量分布，外推需独立采样依据。

## 5. 先分组再扩写及新增关联处理

ProvenanceGraph 为结构化边表，不依赖图数据库。
节点是 document_family、seed_thread、base_task、blueprint、
conversation、contrast_pair；边有 type、reason、source_ref、status。
confirmed 的共享依赖、variant、重复和对照关系形成连通分量。
proposed 语义关联若未排除，采用保守合组或阻止独立 holdout 声明。

SplitAssignment 固定协议、seed、group_version、各 group 的成员、
split、理由与限制；split 使用 Registry train/dev/test/unspecified。
按权重贪心平衡组大小和 hard cell，不能为满足比例拆开组。
初次分配在问句变体产生前完成，所有变体继承 group/split。

生成后发现新跨组依赖：
- 尚未冻结：合并相关组，按固定重分策略产生新 group_version；
  所有相关候选重新做 split/dedup 检查，记录之前分配。
- 已发布：只生成影响清单和新发布版本，不修改旧成员。
- 形成大连通分量：限制新增跨族蓝图或保留专门跨文档子集；
  不能删真实关联边以制造独立性。
- 与已锁定 holdout 冲突：新候选隔离或新版本整体重分，
  不通过移动单题掩盖已发生的信息访问。

question_holdout 允许同一业务知识库中的源文用于检索；
corpus_holdout 另规定生成/开发/测试时可读的来源家族。
两者都禁止 QA/reference/审定解释进入被测索引或训练。
没有全训练数据可见性时仅记录已检查范围，不能认证“无训练污染”。

## 6. 精确、语义及蓝图去重

DedupPolicy 固定 normalizer/version、candidate generator、
人工/模型等价 rubric、阈值来源、已校准范围及保留策略。
三个层次：
1. query/history 精确签名：固定空白/大小写规则，保留年份、数字和否定。
2. 词面 MinHash/BM25 召回疑似近重复；可选 embedding 只能召回，
   未配置/校准不阻塞精确与蓝图检查，但保留 semantic_scope_incomplete。
3. fact signature + operation + constraints + base_task，
   将同事实不同问法归为 variants。

equivalence 决策为 equivalent/distinct/unknown，依据完整条件与目标，
不能用余弦分数直接删除。unknown 的跨 split 对阻止已完整去重声明。
“不是 2024”与“是 2024”、正/负对照、权限角色变化，
是有意 contrast 而非普通重复；保留 contrast_pair_id 并同组。

选择代表题按预先固定规则：完整性、明确来源、稳定身份；
不能按系统答对/答错选择。
被排除候选、原因及所有成本继续在构建清单；
人工改 query 后重做全相关去重，不能沿用旧签名。

## 7. 验证计划

| 场景 | 应观察到的结果 |
|---|---|
| 计划与实际差额蓝图 | 必需答点仅差额，前提单列 |
| 出现直接汇总表 | 新充分路径，不能强称必须两跳 |
| 缺单位/中间实体 | blueprint unknown 或 rejected |
| target 100、只有 3 来源组 | 保留上限与缺口，不用变体补独立题 |
| 改变输入遍历顺序 | 选择结果、seed 和代表题不变 |
| 同会话/变体/正负对照 | 全部继承 split_group |
| 跨文档图形成一大组 | 不任意拆边或拆组 |
| 冻结后发现跨 split 关联 | 新版本影响清单，旧版不修改 |
| 年份/否定只差一字 | 语义区分并保留 contrast |
| 无语义服务 | 显示未检查范围，不报告完全无重复 |
| 缺 hard cell 或 human 数 | 发布 blocker，不降成软目标 |
| 被测系统均答错 | 不自动删除题或生成高难真值标签 |

具体业务比例、近重复阈值和错误容忍均是运行策略输入；
设计不内置一组伪装适合所有领域的阈值。
