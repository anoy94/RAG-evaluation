# Diagnostics：干预选择与严格 Oracle

- 状态：Confirmed；修订：r001；对应 README C5–C7。
- 本文定义行为；公共 DTO 仍以 [Contracts](README.md#contracts)为准。

## 1. 抽样和预算

冻结的母集必须含全部原始观测、任务 Judgment 及缺失记录，
不得只传入失败日志却称全基准抽样。
selection_policy 固定 failure_priority、pass_control、independent_random 三个渠道的
整数名额、分层字段、seed、算法版本、总 trial/pair 上限和未达额处置。
只允许 CaseKey、arm、split、tier、document_family、原始 pass/fail/unknown 作为分层键；
字段缺失进入 explicit_unknown 层，不依赖新增试验结果。
未知标签保留 unknown 层，不能归为失败或从母集消失。

先核验每例资格，保存 eligible/ineligible/unknown 及原因；
在 eligible 母集内按渠道筛选，每层对
[hash_order_v1,seed,channel,CaseKey,arm,slot] 的 rag-eval-json-v1 规范数组
计算 SHA-256 优先键，键相同按规范身份排序。三个渠道独立排序，可重叠；
合并时去重并登记全部渠道，再按预设总上限和固定渠道顺序截取。
不把各渠道概率简单相加，不支持无依据的逆概率加权。
为降低 query 重复偏重，默认每个 case/arm 只选一个显式 original slot；
其他 original repetition 保留在来源账本，不视为新题。

名额不足报告实际数和缺口，不临时放宽审定、权限或模型固定条件。
预算在选择前固定每例重复次数 R、最大输入/输出、最大真实调用数、
deadline、重试额度和独立干预费用池。配对每例需 2R 次新生成；
预算不足则整例不派发或保留未完成槽位，不删掉失败臂。
费用未知时只接受可强制的请求/token/时间预算，
金额标观测约束，不能承诺货币硬上限。

## 2. 严格上下文干预资格

strict_context_oracle 必须同时满足：
1. 正式用例预期 answer、answerable，目标充分组在当前 source/task scope 已审定充分；
   不将 human_core 当全部组均审定。不对权限拒绝/澄清题强行注入答案。
2. 固定充分组所有证据可读取，来源、期间、单位、条件和允许推导完整；
   当前业务权限、工具规则及出域规则允许这些证据进入该调用。
3. 原始 generation request/context original、complete，
   request_bindings 可核验，实际配置满足 Adapter FrozenGenerationSpec。
4. 本例单次最终文本生成，无后处理、后续工具、服务端隐藏 RAG 或未控模板。
5. Adapter 声明并实际支持所需模型和操作；所有非 context 参数可固定。
6. 新上下文精确容量可核验且不超限；无法核验不靠估计放行。
7. 比较两臂采用同一正式参考、任务 rubric、EvaluatorPackage/metric profile、
   评分来源 policy 和 calibration policy。旧分数不兼容时对两臂重评分。
8. 新上下文确实只来自选中源证据；无参考答案、解题推导结论或标签泄漏。

ineligible 与 unknown 均不得执行严格计划；可以生成补材料建议。
如明确选择 alternative_generation，必须新建计划并公开变更因素，
不能自动 fallback 后继续沿用 strict 名称。

固定条件包括 query、history、as_of、access_scope、全部非 context messages、
system/developer/template、模型不可变 revision/部署、输出预算/格式、
采样参数/seed、工具策略、序列化、provider defaults 和缓存策略。
全部 repetition 保留原请求的采样参数及 seed；原调用未设置且协议不支持 seed 时，
两臂均不新增 seed。pair_order_seed 只用于试验顺序，不传给生成模型。
另换生成 seed 必须作为新的显式条件计划，不能冒充原采样条件重复。
有相同 seed 也不承诺服务端确定性；已知模型/环境变化使相关严格对比失效。

InvariantManifest 每字段保存原值引用、比较规则和预期变化许可：
- patch 外字节相同，其他 JSON 字段规范值相同。
- patch 内容、顺序、长度/token、噪声可变化。
- 请求 ID/时间/计量允许变化但须保存，不当作生成配置。
- 不可观测的后端差异列为保证限制；已知冲突不能藏入 limitation 仍判 strict_valid。
服务协议能保证的范围与导入来源真实性必须分别报告。

## 3. gold 物化与补丁

set_selection 为 explicit_set_id，或按事前固定 set_id 字典序选首个合格组；
不得试多个组再选恢复最好的一组。比较多组必须预注册为多个条件。
组内默认按显式 EvidenceKey 顺序；去除完全相同 EvidenceKey，
保留不同来源和同一事实的组内实际项，不用重复证据提高覆盖。

直接读取 Registry EvidenceUnit 指向的固定参考文本和锚点范围，
保留必要限定。只使用在充分性审定范围内的源证据内容；
若充分性依赖 quote 外背景，需纳入该已审定源范围，否则资格 unknown。
物化记录 evidence→源范围→输出片段→patch range 的完整映射。
格式包装、分隔符、引用来源标签沿原 context_slot 的固定布局；
无法从原模板确定安全替换边界时 blocked，不移动 system/user 指令。

多个 slot 的证据分配、排序和分隔符在计划中显式冻结，
不按试验结果调整。baseline 臂用同一补丁机制写回原始文本，
核验最终请求与原始请求完全一致，防止旁路实现差异。
treatment 保留 patch 外字节，新增 context 的 request_bindings 由真实请求重建。
不从 query/reference 中自动补“答案提示”，不静默截断 gold。
token/长度/噪声/顺序变化作为组合干预共同报告，
严格 Oracle 本身不隔离“纯召回量”单因素。

计划校验物化完整性，执行后再核验实际请求确实包含所有物化证据；
供应方截断、插入额外模板或替换模型，使 protocol non_strict/unverified，
不能因任务恰好恢复而忽略。

## 4. 配对执行的解释边界

original_comparison：历史 original 与一次新 Oracle 对比，
报告时间/服务变化，不能当作随机交错的新配对试验。
paired_repeat：每个 pair_slot 重新执行 baseline 与 treatment，
冻结 AB/BA 平衡顺序，分配通过固定 seed 打散；单 worker 顺序调用。
原始失败选择条件固定为 original 的任务状态；
新 baseline pass 时依然保留，不用新失败筛选替换配对样本。

Evaluator 需要新的实际响应/评分视图，旧人工判断不能复制到新响应。
无新合法评分来源时生成有效 Run，但效果 unknown；不得调用隐式裁判。
已执行基线是否能重放必须由 Adapter 能力与实际 FrozenSpec 判定，
不是由“同一个 model 名称”推断。

## 5. 其他干预协议

这些类型可规划、导入及受限解释；首版执行器明确 unsupported。

| 类型 | 改变项与必须固定项 | 可检验内容/限制 |
|---|---|---|
| alternative_generation | 明示改变模型/模板/预算；query/数据/评价基线固定 | 整体生成条件变化，不能当严格上下文效果 |
| fixed_candidate_order | 固定候选 ID/内容/预算，仅排序变 | 排序/位置效应；有截断时公开实际留存变化 |
| reviewed_rewrite | 固定原任务和下游配置，换已审定 rewrite | 该改写条件效果；必须核对任务语义未漂移 |
| parser_substitution | 固定源 revision/权限与下游构建条件，替换解析 | 必须记录重建索引；存在交互时是整体方案比较 |
| evidence_position_noise | 固定必要证据内容/可核验预算，显式位置/噪声因素 | 位置与噪声分条件；联动时不能单独归因 |
| premise_removal | 去除固定必要前提，其他输入固定 | 检查对该前提的依赖；替代路径/先验知识需保留 |
| no_context_control | 明确空 context，其他请求固定 | 无上下文条件表现；成功不证明训练污染或检索无用 |

每类记录 falsifiable_hypothesis、changed/fixed、真实输入、
副作用、协议核对、同口径评价和结论范围。
不允许字符串 kind 外加无校验自由 payload 冒充完整试验协议。
未来要执行这些类型，需给提供方/Runner 新增明确行为和验收，
不以通用 stage_replay 作为当前模块完成门槛。

## 6. 验证

覆盖资格每一项的单独缺失、权限/工具题、相同来源不同 revision、
充分组背景缺失、多个合格组选择、请求多区间补丁、
gold 超容量、参数/模型漂移、原始失败在新 baseline 中通过、
两渠道重叠、配额不足、重复执行身份及完整未完成槽位。
测试断言实际发送载荷，不能只断言 plan JSON 看似正确。
