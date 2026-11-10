# 质检、人工审定与正式发布

- 状态：Confirmed；内容修订：r001；用户授权代理连续完成。
- 上级契约：[README / C7–C8](README.md#contracts)。
- 正式质量等级及发布规则由 [Registry C7–C12](../dataset-registry/README.md#contracts)拥有。

## 1. 检查目录与适用性

ValidationPolicy 每项固定 check_id/version、实现/模型/模板/rubric、
applicability predicate、severity、输入依赖、通过条件和 required_for_tiers。
先做当前权限、schema/引用/来源与白名单运算，再做昂贵语义调用。
表内“检查通过”均需真实可消费的依据，不能从模型 confidence 推定。

| check_id | 检查与最小依赖 |
|---|---|
| schema_reference | 全部 DTO、引用、数量及 snapshot/source 绑定 |
| source_locator | quote/码点/页/区域与固定表示一致；歧义单列 |
| source_fact | 原文与事实的实体/期间/口径/否定/单位；不能只审 caption |
| query_alignment | 用户输入条件、蓝图与允许工具一致 |
| claim_support | 直接支持或合法推导闭合，所有必需答点均有路径 |
| arithmetic | 类型化规则、顺序、单位、精度与容差 |
| sufficiency | 至少一条声明路径完成任务；核验的组与范围明确 |
| necessity | 宣称多跳的必要事实作用与替代路径 |
| answerability | 当前授权完整范围、负例依据和工具条件 |
| ambiguity_conflict | 指代、实体、版本/口径冲突和裁决规则 |
| input_leakage | 不含内部参考、受限答案、标签和无业务理由的 source ID |
| duplicate_split | 精确/语义/蓝图关联及跨 split 检查 |
| access_time | 模拟用户权限、进程当前授权、有效期与出域许可 |
| presentation | 表达、长度、历史和必需引用的可用性 |

每题按真实任务确定适用；没有 numeric_rule 时 arithmetic 可不适用并给依据。
未决适用性不能当 pass；必需检查 fail/unknown/error/undetermined
均不能晋级 auto_validated。
advisory 不影响资格时仍保留在摘要。
单项修改 severity、阈值或适用规则形成新 policy，不能临时删失败检查。

## 2. 候选状态与派生资格

CandidateRevision 不可变；质量事件引用固定候选、依赖与检查选择，
状态派生为：
draft → generated → auto_validating → auto_validated；
任何阶段可进入 revise_required/rejected/quarantined。
auto_validated 可进入 review_pending；
符合当前完整人工绑定才为 human_reviewed。
发布资格满足时 release_ready，但还没有 DatasetVersion。

允许自动扩展从 auto_validated 进入 release_ready，
不伪造 human_reviewed。
自动验证通过与合成来源可产生 synthetic_extension 请求，
是否生效由 Registry 再检查。
human_core 要求逐题、原文和各必需范围审定，不从证据已审推整题已审。

fail 可导致 reject 或请求修复；unknown 导向 review_pending；
error 允许在冻结重试策略内再执行程序，不能记质量失败。
已知源文错误、越权、泄漏和跨 split 冲突默认 quarantined。
“作业 succeeded”只表示按协议产出检查/处置，不代表质量成功。

候选修订从 parent 建新内容，不回写旧事件。
修改 query → 对齐/解答/支持/可答性/泄漏/去重全部失效；
修改事实/参考 → 所有依赖蓝图、解答、检查及视图失效；
修改 checker → 对应验证及晋级资格失效，原提取可以复用。
人工改答案不能只保留旧通过检查。

## 3. 修复与质量批次处置

固定 reason 到修复路由：
格式 → 结构修复；
题面范围 → 保蓝图重写题面；
数值/来源 → 修参考/事实并重建依赖；
支持不足 → 新证据路径或新任务；
负例范围不足 → unknown 或新限定范围任务。
不执行模型返回的修复脚本，不无限换模型筛选“通过”。

SamplingReviewPlan 分 risk_targeted 与 probability_sample，
记录完整抽样框摘要、stratum、方法、seed、纳入概率、
未响应、审查量和覆盖范围。
定向样本不估总体错误率；随机样本也需按抽样设计解释区间。
本模块默认输出分层计数与审定结果，不实现复杂调查加权推断。
抽样通过仅提升实际全项审定题，未审题仍 synthetic_extension。

QualityBatchPolicy 在构建前明确错误容忍、最低人工覆盖、必查切片、
暂停/扩查/重建条件；无业务值则不作批次合格认证。
发现共享 OCR/模板/规则错误：
冻结相关 slice 晋级 → 追踪共同依赖 → 建完整影响集合 →
补审/修正后重新检查 → 新质量批次记录。
不只替换抽中的坏题以提高表面通过率。

## 4. 投影、导入与真实审定目标

发布前固定 PublicationSelection 与 BuildSnapshot，不按基线输赢选题。
ProjectionValidator 验证：
- 每题 core payload 与工作候选/解答一致。
- source revision、关系、claim/rule DAG、必需答点、充分路径闭合。
- C2 text profile 不包含不支持的 NumericRule、history、visual locator 语义。
- 原始模态与测评可用能力分别在兼容报告列明。
- 硬配额、split、去重及全部排除原因可追踪。

投影使用 Registry 自身 DTO/校验与 native_jsonl_v1：
corpus_snapshots、evidence_units、test_cases、members。
规范 JSONL 严格 UTF-8、固定对象身份排序；
history/operands 保留顺序，金额不经 float。
额外生成谱系通过成员 provenance_ref 或现有 rubric/assessment 引用，
不修改 Registry 核心 schema。

prepare_publication 第一次可以形成合法草稿，但因缺审定返回 blocked，
仍返回真实 projection_ref/draft_ref 和具体 blockers。
流程：
校验选择 → 保存全部核心对象/成员 JSONL →
Registry.import_dataset → 构造真实 review targets →
检查已有审定及必要时形成待办工作包 →
Registry.prepare_release（满足前置时）→ 保存 Builder 准备结果。
没有合法核心投影时不建假草稿、不建假 TestCase 填空；
仅返回 blocked 与候选问题清单。

为每个待审对象调用 Registry.build_review_target：
object_type=corpus_snapshot/evidence_unit/test_case，
object_key 来自新草稿，annotation_context_ref 使用固定范围。
不得从待检查 ReviewRecord 复制 target/review_view_ref。

人工先看原始文件或冻结页、必要标题/表头/脚注，
再填写当前目标检查结果；可隐藏生成器建议答案做首轮判读。
工作包含 target、rubric、response schema、必要授权资产引用、
待审项目及允许显示范围，不建设新的 ReviewItem/任务派发协议。

## 5. 自动检查到 ReviewRecord 的绑定

Builder 可以构造 method=automated 的记录，经现有 Review.import_records
校验保存；不能生成 method=human 或代替真实人工身份。
核心对象视图排除本次记录和推导等级，避免摘要循环。

synthetic_extension_v1 的最低检查映射：

| Registry check_id | Builder 必需依据 |
|---|---|
| candidate_consistency | query_alignment、schema_reference、presentation 及输入泄漏检查 |
| answerability_basis | 完整 AnswerabilityEvidence 和 answerability 检查，不是 search_no_hit |
| reference_consistency | 解答约束、所有必要答点及表达与规则一致 |
| evidence_consistency | claim_support、source_fact、source_locator、sufficiency 的适用完整范围 |
| numeric_rules | 存在规则时 arithmetic 通过；无规则按 rubric 合法不适用 |

映射表只定义如何引用依据，不保证模型判断正确。
每个所需条目都需消费当前 candidate、projection 和真实 target；
输入不一致或范围不足不得生成 approved。
ValidationResult.error/unknown 不能转成 ReviewRecord.check.pass；
失败/未决使用 reject/revise/needs_adjudication 并保留各项原因。

assessment_ref 使用 Registry 的 registry-review-assessment-v1：
scope_by_check、original_source_keys、examined_snapshot_scope、
resolves_record_keys、limitations。
各检查范围列清楚 claim、rule、set、EvidenceKey 和标注单元，
不能只列字符串 check_id；自动方法明确 common_mode_risk、
source_assurance 和 local_import_only。

human_core 严格沿用 Registry C10 的 13 个必要 check_id、
条件式不适用和逐题要求，不引入一套 Builder 替代人工标准。
source_reviewed、充分组、必需主张、数值规则分别核验，
完整题审定也不自动认证未审查替代来源。
新记录、反对意见和仲裁全部保留；只接收完整显式批次集合，
不能按最后时间/最高等级/多数票自动选择。

## 6. 二次准备、正式发布和 C2 对齐

人工通过既有 review import 提交真实文件后，
调用新的 prepare_publication，引用完整 ReviewSelection。
只改审定/成员元数据时 core 身份不变；
Registry view 是否仍一致由其所有者重新检查，不能想当然复用。
改变内容则新 core revision，旧审定失效。

ready 要求 Builder 硬配额/质量批次和 Registry ReleasePreparation 均 ready。
若基础输入合法但等级不足、分歧、硬配额缺口或未知负例，保持 blocked。
权限/损坏/存储故障返回操作错误，不能包装成普通阻塞然后降级发布。

publish_dataset：
重新授权并核验准备 → 核验当前批次 hold 状态/固定选择 →
Registry.publish → Registry.open_dataset(evaluation) 读取正式版本及成员 →
保存发布回执和 BuildPublicationRecord。
正式 visible 只由 Registry release 记录决定；
Builder publish journal 不能成为第二个发布权威。
任何失败不得抹去已经提交的 DatasetVersion。

发布记录包含 dataset_id/version、完整 DatasetVersionRef、
candidate→CaseKey/EvidenceKey 映射、配额/等级/来源/支持范围、
未发布候选、兼容报告和审定/生成版本。
C2 对齐验证使用其原公开读取与普通输入投影；
Builder 不导入 C2 Runner/Evaluator，测试组合入口负责跨候选接线。

unknown 可留复核池，不计确定负例和核心/合成有效配额；
必要时显式 basic_import_v1/unverified 发布须另给发布请求及用途，
不能为了完成目标自动降级。
一个版本可含多个 tier，但构建摘要和后续报告必须分层统计。

## 7. 工作包导出、增量更新和撤回

export_review_package 使用可信配置允许的 output_root，
包内默认是目标、rubric、响应模板和受控引用，不自动包含全部原文。
显式请求原文副本时逐项检查当前授权与出域/导出政策，
不给无权审定者产生含隐藏答案的文件。
路径拒绝绝对相对混用、..、设备名、符号链接/junction 和规范重名；
在允许根同卷独占暂存 → 流式核验全部字节 →
manifest 最后写入 → 原子提交目录。
已存在目标只在全字节相同后幂等返回，不覆盖异内容。
导出回执失败暴露提交阶段，重试核验再补回执。
撤权不能撤回已导出副本，不能作此保证。

新文档/事实/替代证据产生增量 BuildRequest，
DependencyImpactManifest 列直接/传递依赖、旧/新引用、失效检查和既有版本。
原版本不原地修 gold；C2 在共同新版本重算 A/B。
撤回使用 Registry.record_lifecycle，按明确事件保存原因及替代版本。
访问撤销/实际删除属 Store/外部治理能力，Builder 不越权清理；
拒权后的旧引用明确不可读，保留不含秘密的影响标记。

## 8. 构建摘要和验证计划

摘要固定记录源覆盖、事实/蓝图/候选、各 check 的适用/通过/失败/
未知/错误、候选自动通过率、逐题人工通过率、正式发布率、
配额/组数/变体、tier、未知可答性、修复次数、成本及未测能力。
分母分别声明，定向抽检不作为总体准确率。
build summary 属 Builder，不复用或篡改 Reporting 的评测报告/门禁类型。

| 场景 | 预期行为 |
|---|---|
| 自动全通过 | 只请求 synthetic_extension，不生成人工签核 |
| 抽检一题通过 | 只有该题具备实际审定范围，其他题不提升 |
| 原文不读只审生成答案 | 不满足 human_core 来源检查 |
| 检查只覆盖一个充分组 | 其他组不自动获得审定 |
| 审定旧 query 或旧规则 | 绑定失败，重新做真实 target |
| 自动记录 owner=dataset-builder | 不允许；必须使用真实 Registry target |
| 合法 draft 但无人工 | blocked，返回可审目标，不自动降低等级 |
| 业务要求为 hard cell 缺口 | 阻塞发布，不能仅 Registry ready 就放行 |
| manifest 提交后回执丢失 | 不重复发布/重合成，定位原正式版本 |
| 原 gold 错误或替代来源发现 | 新版本影响清单，原实验不改 |
| 工作包路径穿越/撤权 | 明确失败，无越界文件或凭旧回执读取 |
| 自动结果和人工异议并存 | 保留异议，明确仲裁前不能晋级 |

这些均为计划验证；本设计不认证真实人工真实性、合成正确率或生产门禁。
