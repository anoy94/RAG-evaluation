# 需求—设计—验收映射

- 设计标识：rag-evaluation-v2。
- 审阅状态：Confirmed。
- 内容修订：r003。
- 单元修订：C2 需求、来源与验证计划 r001；C2 原进度/基线 r002；C2 当前交接进度 r003；C1 第 6–7 节 r001；共享导航 r003。
- 映射范围：C2 离线评测与 C1 数据集生成；各候选修订独立记录。
- 映射覆盖：Complete，仅指各候选已声明范围。
- C1 候选就绪状态：Ready for OpenSpec；当前交接 C1-H001。
- C2 候选就绪状态：Ready for OpenSpec；当前交接为 C2-H002。
- 验证执行状态：尚未实现和执行；本文件只确认计划对应关系。

## 1. 范围与来源

行为依据为 [RAG 测评设计 v2](../doc/RAG测评设计_v2.md)，
范围由 [索引中的 C2 选择](index.md#5-候选推进范围)界定。
C2 优先导入已有数据及运行；数据集自动生成后续对齐。

v2 为未声明不可变版本的本地 Markdown。
本次读取的原始字节 SHA-256：
e7427dd050fcef911a12a709e4fc5cf8ebd103ed631f17cf4bed52e71eb7f425。

本表 R01–R32 是工程追踪 ID，不改写 v2 的原始章节。
每项只有一个主设计所有者；
接口/流程列可以列出支持模块，但不转移契约所有权。

Confirmed 表示映射及对应设计已确认，
不表示实现、测试或真实业务验收通过。
本文件不将 C2 声称为完整 v2/P0 交付。
C1 的生成侧映射见第 6–7 节；不扩展本节 C2 的需求范围。

## 2. 消费的设计基线

项目级设计：
- [总体架构](00-system-architecture.md)：r001。
- [工程目录与依赖规则](01-directory-layout.md)：r001。
- [横切规则](02-cross-cutting-concerns.md)：r001。

模块契约：
- [Artifact Store](modules/artifact-store/README.md#contracts)：r002。
- [Review](modules/review/README.md#contracts)：r001。
- [Dataset Registry](modules/dataset-registry/README.md#contracts)：r002。
- [Adapters](modules/adapters/README.md#contracts)：r001。
- [Evidence Mapper](modules/evidence-mapper/README.md#contracts)：r001。
- [Evaluators](modules/evaluators/README.md#contracts)：r001。
- [Metric Planner](modules/metric-planner/README.md#contracts)：r001。
- [Runner](modules/runner/README.md#contracts)：r002。
- [Reporting](modules/reporting/README.md#contracts)：r002；
  C1–C12 的单元修订为 r001，C13–C18 为 r002。

集成及验收：
[C2 CLI 与离线评测集成](slices/c2-offline-evaluation.md)
第 1–6 节为 r001，第 7 节及基线摘要为 r004。
本节引用实现设计与契约修订；文档进度修订以
[交接 C2-H002](handoffs/c2-r002.md)列出的确切基线为准。
下表 E01–E24 引用该文档第 6 节的计划场景。
模块验证计划按实际场景名称引用，不伪造已存在的测试文件或结果。

## 3. C2 需求映射

| Candidate | Requirement | Primary Design Owner | Interface/Flow | Verification | Review State |
|---|---|---|---|---|---|
| C2 / R01 | 不可变字节、内容摘要、访问域及谱系；v2 §4.1–4.3、16.4 | Artifact Store | C1–C5；put_blob、register_source、open_verified；清单最后提交 | E18、E23；Store 第 4 节的损坏、同内容重用和提交中断场景 | Confirmed |
| C2 / R02 | 核心数据、证据和成员关系具有完整 revision；v2 §4.1–4.2、17.1、17.5 | Dataset Registry | C1–C8；来源、EvidenceUnit、TestCase、DatasetVersion 分别拥有身份 | E02、E12；Registry 第 4 节的悬空引用、同事实多来源及源 revision 校验 | Confirmed |
| C2 / R03 | 导入、审定衔接及正式发布不提升未经核验的质量等级；v2 §4.2、6.1、18.1 | Dataset Registry | C9–C12；import_dataset → prepare_release → publish；保留 unverified 和 blocked | E02、E12、E23；Registry 第 4 节的旧 approved 声明、部分审定和发布中断场景 | Confirmed |
| C2 / R04 | 审定绑定真实对象、视图和 rubric；v2 §11.1–11.3、17.5 | Review | C1–C5；import_records、check_binding；Runner 投影到 Evaluator 输入 | E12；Review 第 5 节与 Evaluator 第 5 节的旧视图、冲突材料和无 payload 场景 | Confirmed |
| C2 / R05 | split 和来源分组不能虚构独立性或跨集合泄漏；v2 §6.6、14.2 | Dataset Registry | C11；保存分组依据、跨 split 检查；Reporting 沿用分组 | E10；Registry 第 4 节的跨 split 关联和缺少分组依据场景 | Confirmed |
| C2 / R06 | 完整 JSONL 运行导入、能力声明及逐条固定关联；v2 §7.1–7.3、12.1 | Adapters | C1–C4、C9–C10；external-rag-jsonl-v1；不跳坏行、不按 query 猜关联 | E02、E03、E12、E23；Adapter 第 4 节的映射、重复身份及数量检查 | Confirmed |
| C2 / R07 | Span、阶段产物和实际生成输入保留真实身份及顺序；v2 §3.3、7.3、9.6.2 | Adapters | C3、C5–C7；检索结果与真实 generation context 分开，不重构缺失 prompt | E04；Mapper 第 5 节的实际请求绑定、多个调用及上下文截断场景 | Confirmed |
| C2 / R08 | gold 与待测 chunk 身份解耦，文本定位保留严格缺失状态；v2 §5.1–5.3、18.1 | Evidence Mapper | C1–C8；固定源 revision、quote、归一化、联合片段和 verified_absent 条件 | Registry 第 4 节的改 chunk；Mapper 第 5 节的数字/年份/否定差异、ambiguous、unresolved 和空上下文场景 | Confirmed |
| C2 / R09 | 新替代证据保持 unjudged，补标通过新版本生效；v2 §5.4、18.1 | Dataset Registry | C5、C7、C10；原版本不改写，重评分须使用明确新基线 | E12、E13；Registry 与 Mapper 验证计划中的未审查替代来源场景 | Confirmed |
| C2 / R10 | 指标实例固定单位、依赖、分母、方向及版本；v2 §8.1–8.2 | Evaluators | C1–C3、C7；同名不同规则/阶段使用不同实例，Reporting 检查兼容性 | E12；Evaluator 第 5 节及 Reporting 第 5 节的实例混合和版本错配场景 | Confirmed |
| C2 / R11 | 计划保留全部预期观测、根槽位和输入缺口；v2 §8.2、12.2 | Metric Planner | C1–C9；catalog_only → available_inputs；gate_input_status 不预测质量 | E03、E07；Planner 第 5 节的空 Run、共享依赖及 catalog_only 场景 | Confirmed |
| C2 / R12 | 实际数据版本、系统配置、输入及来源执行关联必须核验；v2 §7.3、12.2、14.1 | Runner | C1–C6；固定 Run 选择、Subject 和 Manifest；不自动选择最新/最高分 | E02、E12；Runner 第 5 节的 history/as_of 冲突、重复执行及配置差异场景 | Confirmed |
| C2 / R13 | 系统失败、测量失败和未完成槽位分别表达；v2 §8.3–8.4 | Evaluators | C6、C11、C13；真实任务失败可为 fail，测量故障保留 null；Runner 不伪造 Judgment | E03、E05、E06；Reporting 第 5 节的 completed 与有效覆盖分离场景 | Confirmed |
| C2 / R14 | 数值规则检查实体、期间、单位、运算及容差；v2 §6.4、9.6.1、17.5、18.1 | Evaluators | C8；消费 Registry C6 的类型化规则，按固定响应契约提取 | Evaluator 第 5 节与 Registry 第 4 节的白名单运算、单位和派生答案场景 | Confirmed |
| C2 / R15 | 正确、完整、实际上下文支持分别评分；v2 §9.6.1–9.6.2 | Evaluators | C4–C5、C9；完整输出主张、必要答点和实际上下文绑定；无事实主张不自动满分 | E04、E06、E12；Evaluator 第 5 节的额外错误、缺答点和纯拒答场景 | Confirmed |
| C2 / R16 | 证据按事实去重并按已审定充分组评价；v2 §5.4、9.5.2–9.5.3 | Evaluators | C10；消费可靠映射及语义审定，不把 quote 命中直接当支持 | E04；Evaluator 第 5 节的重复来源、必要事实、充分组及标注范围场景 | Confirmed |
| C2 / R17 | 任务成功满足原有必需维度，行为标签匹配不替代整体正确；v2 §2、9.6.1、9.6.4 | Evaluators | C7、C11；ExpectedActionMatch 与 TaskSuccess 分开，不用加权分抵消必需失败 | E05、E11、E12；Evaluator 第 5 节的拒答、缺权限/引用维度和任务失败场景 | Confirmed |
| C2 / R18 | query 宏平均、重复生成、覆盖与主张微平均口径明确；v2 §8.4、14.2 | Reporting | C3–C5；全部预期槽位、保守适用分母、完整题内均值和独立微平均 | E03、E06、E07；Reporting 第 5 节的零分、零适用、长短答案及重复槽位场景 | Confirmed |
| C2 / R19 | 缺失识别范围与抽样置信区间分别解释；v2 §8.4、14.2 | Reporting | C6、C9–C10、C15；合法缺失界不冒充 CI，未决适用性不造界 | E08、E10；Reporting 第 7.3 节的最坏情况有限基准和总体推断场景 | Confirmed |
| C2 / R20 | A/B 仅在兼容的同题共同集配对，保留单方集合及全部排除；v2 §8.4、14.1 | Reporting | C1–C2、C7；固定快照、B-A、direction、胜负平及 baseline=0 处理 | E02、E03、E08、E12；Reporting 第 5 节的非配对均值和不对称适用集合场景 | Confirmed |
| C2 / R21 | 独立组、重复、稀有事件及多重比较满足方法条件；v2 §14.2 | Reporting | C8–C10；精确二项或固定配对 cluster bootstrap；固定 seed、族及 m | E07、E10、E15；Reporting 第 5 节的单组、零错误、退化 bootstrap 和族成员缺失场景 | Confirmed |
| C2 / R22 | 来源构成、重要切片及探索性分析分别展示；v2 §6.1、14.1–14.3 | Reporting | C1、C11、C14–C16；固定切片，缺字段不猜测，重叠切片不加总 | E08、E09；Reporting 第 5、7.3 节的重叠切片、后验阈值和重要切片失败场景 | Confirmed |
| C2 / R23 | 时间和用量保留实际 scope、单位、来源及未知量；v2 §7.4、9.7、16.3 | Adapters | C8；Evaluator C7 的 E2E/TTFT/UsageQuantity 与 Reporting C11 消费明确口径 | E02、E05；Evaluator/Reporting 验证计划的未知计量、失败耗时、重复 scope 和经验 p99 场景 | Confirmed |
| C2 / R24 | 门禁具备业务阈值、覆盖、样本及非劣三态判定；v2 §14.4 | Reporting | C13–C15；必选规则、有限基准/总体推断、缺策略和不完整 profile | E08–E11、E24；Reporting 第 7.3 节的 margin、边界舍入、空切片及联合置信场景 | Confirmed |
| C2 / R25 | 审定/校准资格与硬性违规均需真实有效依据；v2 §11.3、14.4、16.4 | Reporting | C14–C15；校准不可核验则 inconclusive；具体硬约束的可靠违规可 fail | E11、E12、E24；Reporting 第 7.3 节的缺校准、正则命中及不完整审查场景 | Confirmed |
| C2 / R26 | 完整 Markdown/JSON 报告、确定性样例及受控展示；v2 §14.3、16.4 | Reporting | C16–C18；同源渲染、完整伴随明细、固定选例、DisplayView 和完整 bundle | E02、E14、E15、E19、E20；Reporting 第 7.3 节的未展示失败、转义和导出冲突场景 | Confirmed |
| C2 / R27 | 重新评分及精确复用不增加业务预测、不覆盖历史；v2 §4.2、12.3 | Runner | C8–C9、C14–C17；新评价基线产生新准备；显式 reuse_source_refs | E13、E14、E17；Runner 第 5 节的缓存依赖变化、冲突和来源计量场景 | Confirmed |
| C2 / R28 | 有限重试、预算、超时、取消及本地恢复保留操作连续性；v2 §12.3、16.3 | Runner | C12–C17；CallIntent、检查点、受控 worker、有限退避和原预算恢复 | E16、E17、E22；Runner 第 5 节的未知结果、outbox、锁竞争及 cleanup_unconfirmed 场景 | Confirmed |
| C2 / R29 | 输入投影、当前授权及日志不泄露 gold、上下文或秘密；v2 §4.1、16.4 | CLI | 切片第 2、4–5 节装配授权；Registry 白名单投影；各 Provider 逐次检查 | E18、E21、E23；Registry 普通输入投影和 Store 撤权验证 | Confirmed |
| C2 / R30 | CLI 可查询结果、区分退出码和操作/门禁状态；v2 §16.1 | CLI | 切片第 3–5 节；CommandResult、job status/resume/cancel、报告与显式导出 | E01、E09、E19、E21、E22、E24 | Confirmed |
| C2 / R31 | 非法输入、超限、部分提交和损坏不得静默转成成功子集；v2 §4.3、12.2、17.5 | CLI | 切片第 4–5 节转交所有者校验与提交；Store C4–C5；保留完整集合、提交阶段及安全引用 | E18–E20、E23；Store 与 Registry 的清单、回执及中断故障注入 | Confirmed |
| C2 / R32 | 单包安装、依赖边界和演示验收范围明确；v2 §3.2、16.1–16.2、18.1 | CLI | 目录设计第 4、6 节；切片第 1、5–6 节；使用真实本地 Provider 验收 | E01、E02；安装冒烟、依赖检查及端到端夹具，不将合成材料当业务质量证据 | Confirmed |

## 4. 延期与验证边界

以下是已选 C2 范围的边界汇总，不改变原需求，
不把未实施的完整 v2 能力标记为完成。

| 相关需求 | C2 处理 | 验证边界 |
|---|---|---|
| v2 §6、18.1：可信数据生产和真实业务核心集 | 实现已有数据接收、审定绑定与正式发布；自动生成和真实核心集建设后续推进 | 合成夹具只验证协议；真实来源核验、代表性和业务正确性需另行提供材料 |
| v2 §11、18.1：模型裁判和独立校准 | 导入已有评分并保留来源；不新增模型裁判或校准引擎 | 不能核验必需校准时相关门禁 inconclusive，不凭文件引用视为已校准 |
| v2 §9.6.3–9.6.4：引用、拒答和澄清专项 | 可消费明确 action 和已审定任务维度；完整自动评价套件延期 | 仅 action 相等不证明拒答正确；引用标号存在不证明引用有效 |
| v2 §9.1–9.4、10：解析、索引及视觉专项 | 当前限于已导入的可用文本链路产物 | 无视觉参考不生成 bbox 分；不能从候选检查外推全量索引或解析器质量 |
| v2 §12.1、13、15：在线执行、Oracle、监控和回流 | 仅 trace_audit；不重跑业务或执行干预 | 不宣称完成因果诊断、线上监控或普通调用的运行时 gold 隔离验收 |
| v2 §7.4、9.7、16.3：完整运行成本及总体性能 SLO | 展示明确 scope 的已知耗时、用量和覆盖 | 不补算跨来源完整成本；经验 p99 不证明总体 p99 SLO |
| v2 §14.2、16.4：高级推断、自动脱敏及发布治理 | 采用已确认统计方法和已有受控展示材料；外部发布延期 | 不提供缺失/抽样误差联合推断、来源真实性认证或已导出副本撤回保证 |

缺少真实业务阈值、人工标签或校准材料，
不阻塞未知、缺失、三态和导入协议的引擎实现；
但会阻止相应真实质量结论或正式门禁通过。

## 5. 验收证据与交接条件

计划验证分三层：
1. 模块契约和算法验证：使用各模块已确认验证表。
2. 真实本地集成验证：执行切片 E01–E24。
3. 业务质量验证：由实际数据、审定、采样及业务策略另行支持。

每条实际证据须记录：
- R 编号及对应 E 编号或模块场景名称。
- 实现/依赖版本、Windows 和文件系统环境。
- 固定输入、策略、随机 seed 和产物引用。
- 退出码、完整状态、数量、摘要及关键断言。
- 故障注入位置、未完成范围和限制。

当前不存在已执行证据；不能把“设计 Confirmed”
转换为“验收通过”，也不能用 mock 代替本地集成。
验收中发现公开契约或需求冲突时返回设计审阅，
不能只修改预期值以适配实现。

C2 内容就绪检查通过；[交接 C2-H002](handoffs/c2-r002.md)
为 Current，绑定包 r021。C2 为 Ready for OpenSpec。
R01–R32 及计划验证保持 r001；没有新增已执行证据。
当前无 OpenSpec 消费方；本次不生成 OpenSpec 变更或应用代码。

## 6. C1 数据集生成需求映射

本节内容修订 r001，依据 2026-11-10 用户授权代理连续完成设计。
C1 从已审定材料整理子集扩展至生成侧模块包，
范围及边界见 [C1 切片](slices/c1-dataset-generation.md)。
来源为 v2 第 4–6、10–11、16–18 节与生成详设第 1–18 节。
以下 G01–G32 均引用 C1 切片第 6 节；全部是计划证据，尚未执行。

| Candidate | Requirement | Primary Design Owner | Interface/Flow | Verification | Review State |
|---|---|---|---|---|---|
| C1 / D01 | 源身份、权限、revision 与全清单冻结；生成详设 §1、3、15 | Builder / evidence | SourceManifest → SourceBundle → Registry Snapshot 投影 | G02、G03、G26；字节/身份/数量闭合 | Confirmed |
| C1 / D02 | 保留提取 partial、独立参考与原文回查；§3 | Builder / evidence | ExtractionResult、ReferenceBundle、原文与参考分离 | G03、G06；缺页/坏提取不静默剔除 | Confirmed |
| C1 / D03 | 原子事实完整限定，零/未披露分开；§4、10 | Builder / evidence | FactCandidate、qualifiers、quantity/missing_kind | G04、G05；同值不同年度与单位 | Confirmed |
| C1 / D04 | 文本、表格和视觉定位稳定；§4、10 | Builder / evidence | text_quote、visual descriptor 与冻结渲染映射 | G05、G06；无 bbox 审查只声明 asset/page | Confirmed |
| C1 / D05 | 蓝图先于问法，目标/必要答点/前提分离；§5、8 | Builder / planning | Blueprint、target slots、推导 DAG | G02、G07；差额题不强求复述全部前提 | Confirmed |
| C1 / D06 | 真多跳、替代路径和充分组不混淆；§5 | Builder / planning | necessary fact roles、support alternatives、partial 范围 | G07、G13；直接差额与两跳分别处理 | Confirmed |
| C1 / D07 | 配额可行、有限缺口调度与来源多样性；§6 | Builder / planning | ScenarioPolicy、SelectionReceipt、硬/软与独立组数 | G09、G10、G19；缺口不可填假标签 | Confirmed |
| C1 / D08 | 先分组切分，再扩写变体；§6、12 | Builder / planning | ProvenanceGraph、SplitAssignment、关联合并 | G08、G10、G30；不拆大组制造独立性 | Confirmed |
| C1 / D09 | 精确、语义及蓝图去重，保留有意对照；§12 | Builder / planning | DedupPolicy、equivalent/distinct/unknown | G04、G08、G27；改条件后重新检查 | Confirmed |
| C1 / D10 | 合成约束对齐与多轮输入无标签泄漏；§7 | Builder / synthesis | QuestionCandidate、白名单 message views | G11、G26；实际输入可追 | Confirmed |
| C1 / D11 | 独立求解隔离生成答案，共模限制明确；§8、11 | Builder / synthesis | SolverViewBuilder、Solution、模型谱系 | G11、G12；replay 不当独立真值 | Confirmed |
| C1 / D12 | 数值白名单、单位、舍入与误差传播；§8 | Builder / synthesis | derivation_decimal_v1 与 Registry 无损投影 | G02、G14、G15；除零/乘除不冒充 C2 支持 | Confirmed |
| C1 / D13 | 可答/不可答/歧义/冲突/权限分别表达；§9 | Builder / synthesis | AnswerabilityEvidence、授权搜索范围和 negative_basis | G13、G26；search_no_hit 不是负例证明 | Confirmed |
| C1 / D14 | 自动质检适用性、fail/unknown/error 分开；§11、18 | Builder / quality | ValidationResult、固定必需检查与状态派生 | G16、G32；不适用不计通过 | Confirmed |
| C1 / D15 | 有限修复形成 revision，依赖按影响失效；§11、14 | Builder / execution | change requests、DependencyImpactManifest、原尝试账本 | G22、G27；不覆盖或重新计为一次通过 | Confirmed |
| C1 / D16 | 逐题原文人工核心，不以自动/抽检代替；§13 | Dataset Registry | C10 human_core_v1；Builder 真实 target 工作包 | G17–G19；13 必需项及范围核验 | Confirmed |
| C1 / D17 | 自动扩展有逐题可消费验证，保留来源等级；§11、13、15 | Builder / quality | automated ReviewRecord 投影 → Registry synthetic_extension_v1 | G16–G18；无自动 human 签核 | Confirmed |
| C1 / D18 | 定向/随机抽样及质量批次处置；§13 | Builder / quality | SamplingReviewPlan、QualityBatchPolicy、slice hold | G17、G20；不只替换发现的坏题 | Confirmed |
| C1 / D19 | 作业/候选/发布状态分开、完整计数；§14、18 | Builder / execution | BuildJob、TaskOutcome、BuildSnapshot | G16、G24、G32；全 reject 不等于程序失败 | Confirmed |
| C1 / D20 | 预算、调用意图、外部未决及恢复；§14 | Builder / execution | CallIntent、预算账本、reconcile_invocation | G21–G25；不重置预算/静默重发 | Confirmed |
| C1 / D21 | 当前授权、出域与安全日志；§1、9、14 | Builder / security | 逐次 Store 授权、最小模型视图、可信 provider 注册 | G11、G26、G29；无凭据/受限答案泄露 | Confirmed |
| C1 / D22 | 发布投影、全引用、等级与配额检查；§15、18 | Dataset Registry | import_dataset → build_review_target → prepare_release → publish | G02、G18、G19、G28；正式版本唯一权威 | Confirmed |
| C1 / D23 | 版本更正、替代证据、撤回与影响清单；§15 | Builder / quality | 新 BuildRequest/Projection → Registry lifecycle | G27、G28、G30；不改旧 gold/实验 | Confirmed |
| C1 / D24 | 标准数据与 C2 直接对齐，公开能力缺口；§16、18 | Builder / publication | Registry DTO/原生 JSONL，C2 文本/数值兼容报告 | G15、G31；完整成员读取与无 gold 输入投影 | Confirmed |
| C1 / D25 | CLI/文件审定和显式安全导出；§13–16 | CLI | C1 切片第 2–4 节命令与结果、受控工作包 | G01、G18、G29、G32；真实退出/单 JSON | Confirmed |
| C1 / D26 | 产能、未知、成本与验证限制可审计；§18 | Builder / summary | 分来源、候选、tier、stage、组与调用状态的摘要 | G09、G17、G25、G32；分母/未知费用明确 | Confirmed |

正式 SourceArtifact/原始字节归 Store；ReviewRecord 归 Review；
正式 Snapshot/Evidence/TestCase/DatasetVersion 归 Registry。
上表 Builder 工作所有权不转移这些正式记录的权威。

## 7. C1 就绪、限制和 C2 影响评估

C1 必需设计：Builder 六份模块文件 r001、C1 切片 r001、
本文件第 6–7 节 r001，以及现有 Store/Review/Registry 契约。
当前无阻塞工程决策，交接 [C1-H001](handoffs/c1-r001.md)为 Current，
C1 为 Ready for OpenSpec。真实数据、模型接入和人工质量尚未验证。

C1 全生成设计不等于所有原格式 parser 已集成，
不等于乘除/视觉定位已在 C2 实现评分。
未支持输入/发布 profile 明确 blocked，保留候选和缺口；
不能在兼容报告中标成已完成。

本文件 r003 新增 C1 映射及导航更新；
C2 的 R01–R32、来源、契约引用、延期和验收计划维持原修订。
C2 业务与验证部分仍有效；无关 C1 的 Pending 摘要由新映射替换。
为保持交接文件的确切文档修订一致，刷新为 C2-H002，原 C2-H001 为 Superseded。
该刷新只同步引用与进度，不改变 C2 业务契约或就绪结论。
此处是明确的影响评估，不将未检查的代码称为兼容。

本轮没有已执行应用验收、真实模型或人工业务正确性证据。
后续提案应把 D01–D26 与 G01–G32 转成可执行验收任务，
并区分本地重放、真实 HTTP 接入、真实来源/人工三个保证层级。
