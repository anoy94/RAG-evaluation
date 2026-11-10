# Diagnostics：效果判定、重复与结论范围

- 状态：Confirmed；内容修订：r001；对应 README C7–C8。
- Diagnostics 解释逐对事实；统计汇总及展示归 Reporting。

## 1. 先验协议核对

assess_interventions 依次核对：
plan→slot→Run→实际 generation call→实际请求/上下文→Subject→Judgment。
检查原请求补丁、模型/config、工具/权限、容量、gold 全量保留、
输出来源、执行身份、评价基线、来源保证及全部未完成范围。
不能由 pass/fail 反向推断协议有效。

strict_valid：所有必需固定项和实际 patch 均核验，且没有已知冲突；
此状态只表示在记录及 provider 保证范围内成立，不认证日志真实性。
non_strict：明确有额外因素变化或已知协议违背；
unverified：必要观察缺失，无法确认。两者分开保存。
非 strict 并不删除运行或分数，可以描述观察转移，
但不能进入 strict 条件恢复统计。

每臂任务状态仅由绑定同一正式评价口径的 TaskSuccess 得到：
status=ok,value=pass/fail 分别映射 pass/fail；其他为 unknown。
Judgment 不存在、error、inconclusive、not_computable、not_applicable
分别保留原因。系统已知失败可由 Evaluator 按原规则产出任务 fail；
Diagnostics 不自行给空答案、超时或缺标签造分。
不同 package/rubric/gold 的历史分数必须重新评价双方，不能平均旧新口径。

## 2. 转移矩阵

以下只对 strict_valid 的任务转移使用正式 effect_code。
任一 unknown 为 insufficient_trial_evidence；non_strict/unverified 另列协议原因。

| Baseline | Treatment | effect_code 与允许结论 |
|---|---|---|
| fail | pass | recovered_under_context_replacement；该对在这套上下文替换条件下恢复 |
| fail | fail | failed_with_reviewed_context；在该证据条件下仍失败，检查生成、模板、题意、输入或参考 |
| pass | fail | degraded_under_context_replacement；发生干预退化，检查旁路一致性、gold、预算和随机性 |
| pass | pass | passed_both_conditions；两种条件均通过，不证明检索必要或所有组件正确 |

观察到 fail→pass 不能唯一归因 retrieval/filter：
实际证据可能原本存在，变化可能包含降噪、位置、长度及随机生成。
fail→fail 不排除检索和生成同时故障；通过 Oracle 也不免除其他违规。
no_context pass 不能证明训练泄漏；负前提试验失败也不自动证明原题真多跳。

hypothesis_updates 只追加：
supported_by_trial、not_supported_in_trial、unresolved_in_trial，
引用原 hypothesis 与具体 slot。原 Finding 不变，
没有“被验证为唯一根因”的升级状态。

## 3. 完整分母

先列 counts：source_observations、source_cases、
original_fail/pass/unknown、eligible/ineligible/eligibility_unknown、
selected_cases、planned_pairs、started_pairs、complete_pairs、
strict_valid_pairs、measured_pairs、protocol_invalid、protocol_unverified、
outcome_unknown、missing_judgments、not_started。
每个数字可追溯到互斥 disposition，重叠 selection channel 另列，不重复计总数。

正式逐对计数只在 strict_valid 且两臂任务可测的集合：
n_FP、n_FF、n_PF、n_PP。可测未知未从完整 planned 数中消失。
observed_recovery_fraction=n_FP/(n_FP+n_FF)，分母为零时 null。
它描述选中样本中“新 baseline 失败的有效配对”，
不是全基准恢复率，更不是某模块根因占比。
original_failure_selected 的 original_comparison
使用“被判原始失败、入选且有效可测的 original/Oracle 对”作分母，
必须与 paired_repeat 分开。

同时报告每个 case：
计划/有效重复数、四转移数、未知数、两臂 pass 次数、
配对差值 d_r=I(treatment pass)-I(baseline pass) 的均值；
有效数不足保留 partial，不用好的一次代表整个 case。
单次或小重复只做描述；不捏造稳定恢复阈值。
若声明 across_queries 描述量，先在 query 内对有效重复求平均，
再对 query 等权，附每题重复覆盖；不得把所有重复混成独立题。
同一 query 的多个 original slot 先按冻结规则合并，arm 分开。

只有失败定向选择的结果不能外推总体。
通过/独立随机渠道分开展示，重叠病例标识 shared；
混合渠道不做无依据人口加权。独立随机子集也需覆盖、独立组、
来源与缺失条件满足才能进入总体推断；首版默认仅描述。
类别可多标签，类别计数不是互斥根因分布，不强制归一成 100%。

## 4. Reporting 的统计责任

Diagnostics 输出完整 pair/case 明细、分子分母及局限，不导入统计库。
Reporting 首版扩展按描述量展示转移和逐题重复分布；
如需要样本方差，R>1 时报告 1/(R-1) 的配对差值样本方差，
R<2 为 unavailable，不能显示为 0。所有重复相同也不证明总体方差为零。

不复用 C2 A/B CI 冒充 Oracle 恢复 CI：
选择机制、条件分母及重复层次不同。
后续推断必须另定预注册 estimand、缺失规则与按 split_group 的成组重采样；
重复运行不作为独立重采样单位。
统计 CI、根因判断准确性和裁判校准误差是三件不同的事。
当前不提供根因概率、自动校准、Oracle 驱动发布 gate。

## 5. 手算验收

两个原始失败 query，各计划两个新配对：
Q1=[fail→pass, pass→pass]，Q2=[fail→fail, fail→unknown]。
假设协议均 strict_valid：
planned=4，measured=3，n_FP=1,n_FF=1,n_PF=0,n_PP=1，unknown=1。
observed_recovery_fraction=1/2；
按可测重复计算 d：Q1=[1,0] 均值 0.5，Q2=[0] 均值 0；
query 等权描述差=0.25，附 Q2 仅 1/2 可测的限制。
不能把 1/2 写成“全库一半失败来自检索”或报告无缺口的 4 对成功测量。

若 Q1 第二对实际模型改变，则该对 non_strict：
保留原始分数，严格 n_PP 减为 0，不按恢复好坏决定纳入。
若 Q2 unknown 后补有效判断，生成新的 envelope/assessment，
保留旧版本，不原地改计数。

计划验证四种转移、每一种 unknown、错评分 scope、
旧 gold/new gold、同一执行复用、失败定向与随机子集、
语义维度转移冒充整体成功，以及上述手算断言。
