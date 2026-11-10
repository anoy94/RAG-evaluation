# 合成与求解：输入隔离、可答性和数值规则

- 状态：Confirmed；内容修订：r001；用户授权代理连续完成。
- 上级契约：[README / C6、C9](README.md#contracts)。
- 所有模型输出都是待核验材料，不是可信裁判结论。

## 1. 模型端口及具体请求边界

ModelCapabilities 包含 provider_id/version、model_identity、
supported_roles、input_modalities、max_input/output、structured_output、
usage_basis、cancel/query_by_request/idempotency 能力及限制。
不以模型名称推断能力；缺少视觉能力拒绝 visual invocation。

ModelInvocation 固定：
invocation_id、stage、task_id、attempt_id、model_config_ref、
template_ref、schema_ref、ordered_message_views、asset_refs、
generation_parameters、max_output_tokens、input_content_digest、
egress_policy_ref。
message_view 为 role（system/user）和文本/字节资产引用；不接受任意工具定义。
模型调用控制端才解析秘密，秘密不进入 invocation 内容或缓存键明文。

ModelResponse 固定：
invocation_ref、provider_request_id（可空）、actual_model_identity、
raw_response_ref、normalized_payload_ref（可空）、parse_status、
finish_reason、usage（known/unknown）、timing、limitations。
截断、拒绝或非法 JSON 分别保留原因，不能按完整成功 payload 处理。
只保存必要业务结论和短依据，不索取隐藏思维链。

初始实现有两个明确 profile：
- recorded_model_response_v1：显式请求摘要到固定响应的映射，
  用于本地重放/夹具。无匹配即报错，不按近似 prompt 复用；
  replay 不代表当前发生了外部调用或独立求解。
- chat_completions_json_v1：由可信配置提供固定 HTTPS endpoint、
  model、credential_ref、timeout、传输大小和 egress 限制；
  只实现约定的非流式 POST /chat/completions 子集，
  输入 model/messages/max_tokens/temperature 及可选已声明的 seed、
  response_format，输出读取 choices[0].message.content、
  finish_reason、model、id、usage 的已声明字段。
  文本以 text content 发送，视觉仅在能力声明明确支持时将授权字节
  编成 image_url data URI，不发送模型可自行抓取的远程 URL。
  provider-specific 参数只能放经代码定义/版本化的白名单映射，
  不承诺任意“兼容”服务或模型都能接受相同字段。

HTTP adapter 禁止自动重定向、SDK 默认重试和自动 tools/function 执行。
非 JSON 外壳、多个选择的未声明处理、缺 content 或响应超限明确失败。
没有 response_format 能力时可使用纯文本 JSON 并做本地严格解析，
但必须在调用前声明该模式，不能失败后偷偷切换模型/提示。
具体 endpoint、模型、SDK/HTTP 库版本、供应商数据条款是实施/运行配置；
接入前用目标服务的官方协议和契约用例核验，本次未做真实请求。

## 2. 五类调用的输入投影

| 调用 | 允许输入 | 禁止输入 | 输出 |
|---|---|---|---|
| fact_extract | 固定锚点、来源定位、限定字段 schema | 既有未核实答案、基线结果 | FactCandidate 和疑点 |
| question_generate | 蓝图、生成所需证据、风格/会话规范 | 发布等级、系统成绩、测试标签 | query/history、约束自述、来源 ID |
| independent_solve | 实际 query/history、授权证据、允许规则、任务 rubric | 生成器答案、Solution、生成自评、提示目标数值 | 结构化 Solution |
| semantic_validate | 候选、固定来源、独立解、已声明参考和比较 rubric | 未记录外部检索、系统输赢 | 逐项 pass/fail/unknown、来源与差异 |
| variant_rewrite | 原用户可见输入、必须保持的约束、表达策略 | 当前目标答案、完整内部参考 | 新输入和约束对照 |

生成器可能看到计算前提，但求解器不接收 Blueprint target.expected_value
或生成器的草稿答案。SolverViewBuilder 按白名单从固定内容重建输入，
记录实际发送字节及全部 source refs。
如果当前 solve 调用实际用了答案作提示，标 answer_exposure，
不能以该结果满足独立求解要求。

每次调用为新消息上下文，不复用带隐藏历史的会话。
“独立”首先指输入和调用过程隔离；
不同 prompt/同模型或异构模型均不保证统计独立。
保留 model family、提取器共享关系和 common_mode_risk；
本地文件重放和模型共识不能认证源文事实。

Template 固定 template_id/revision、目的、变量 schema、
messages、输出 schema/profile、安全规则、最大渲染长度。
变量按数据边界编码，源文中的“忽略规则”不提升为控制指令。
指令注入检测失败产生 validation，不能执行源文指定的 URL/代码。

## 3. 问句约束、多轮与变体

生成结果重新独立抽取实体、期间、范围、单位、否定、量词、
plan/actual、as_of、允许工具及预期答点，并与蓝图逐字段比较。
模型自述只是一份候选解释；确定数字/枚举校验先行。
未知别名或语义差异进入复核，不按 embedding 高相似直接通过。

History 只含 Registry 支持的 user/assistant 文本消息，
保留真实可见顺序和 referent_slots。
若当前答案已在历史披露，改标 conversation_reuse 任务并重新定义目标，
不能当成原检索能力题。工具/多模态历史当前 profile 不支持，
不能转成普通文本绕过限制。

表达变体只能改变措辞、有限错字和风格；
数字、实体、否定或预期行为变化产生 contrast 或新任务。
每次变体重新对齐、求解、质检、去重并继承 group。
同题多问法不能增加独立来源组数量。

## 4. 数值验证与发布兼容

InternalNumericRule 的字段：
rule_id、profile=derivation_decimal_v1、operation、
inputs（id、value、unit、entity、period、claim_slot、source_ref、
precision_kind、lower/upper）、ordered_operands、
output_slot、output_unit、expected_value、rounding_mode、
decimal_precision、absolute_tolerance、relative_tolerance、
unit_conversion_refs、division_zero_policy、uncertainty_method。

先验证来源与语义，再计算：
- identity 一个操作数；add 至少两个且单位/口径可相加；
  subtract 两个保留顺序。
- multiply/divide 需要固定量纲规则，不能只拼接字符串；
  ratio 要求兼容量纲，输出无量纲；
  percent_change=(actual-base)/base，输出比例或百分数由 profile 明确。
- 除零不得返回 Infinity；结果 unknown 或任务定义的错误行为。
- 货币/比例/百分点不混同；换算依赖显式系数和来源。
- 输入为精确 Decimal 时用固定精度及显式舍入，超精度失败不截断。
- 估读输入传播区间：加减按端点，乘除按端点组合；
  分母区间跨零时不产生有限范围。
- 禁止从期望答案倒推扩大容差。

检验表达式沿用 max(abs_tol, rel_tol*abs(expected))；
这只检验数值，不证明来源、实体、期间正确。
reference_answers 从结构化 claims/规则产生或核对，不维护脱节自由文本。

投影到 C2 时，只接受其 scalar_decimal_v1：identity/add/subtract、
相同单位、无隐式舍入、有限 Decimal、操作数/主张依赖闭合。
规则需按 Registry 的实际类型重新序列化和校验，
不能把 InternalNumericRule 字段直接拷入 NumericRule。
非兼容运算形成 unsupported_publish_profile，
保留候选/规则与缺口，不删规则后冒充可评价任务。
这保留了完整生成侧推导设计，也保留了 C2 的真实能力边界。

## 5. 可答性、负例和策略行为

AnswerabilityEvidence={scope_ref, permitted_tools, access_view_ref,
search_plan_ref, search_runs_ref, examined_units, coverage,
positive_support_paths, negative_basis, reviewer_refs, limitations}。
scope 是当前用户可见授权快照，不是 Builder 可以读到的全部材料。

检查顺序：
1. 明确实体、期间、权限、工具和问题边界。
2. 词面/实体别名/数字年份检索，并扩展邻近章节、附表及版本链。
3. 固定本地 BM25 搜索器作初始具体实现；稠密检索为可选补充，
   没有相关索引不得伪造互补检索已完成。
4. 对命中候选运行支持与充分性检查，保存每次范围、预算、版本。
5. 找到充分路径则撤销原负标签，改题或重建 answerable 候选。
6. 无充分结论时 unknown，不能将 search_no_hit 改写 unanswerable。

仅在任务范围足够封闭且 negative_basis 成立时确定负例：
固定结构化字段明确 not_disclosed、显式全范围审查、
或受控语料删除所有相关来源后的独立 snapshot。
删除原始一跳不保证无答案，必须检查替代来源。
核心负例还需人工核验范围及允许工具。

ambiguous 对应 clarify；
无裁决规则的 conflicting 使用预定义 answer+冲突 rubric 或 clarify，
不能临时新造“说明冲突”action。
policy_block 独立于 answerability；不在无权模型/用户视图泄露受限答案。
tool_required 只声明必要工具，不在本模块启动动态外部工具；
动态资料未冻结时不发布成静态闭卷确定真值。

## 6. 分歧、修复与停止

独立解比较逐项核对实体、期间、必需答点、量值和证据路径。
同值不同年份仍为不同结论；不同文风不构成分歧。
同答案不同来源先审查替代路径，不按原生成来源排斥。
人工结论有真实适用绑定时优先解释，但仍保存自动结果和冲突处理。

schema repair 只处理结构问题，不可改业务数值；
semantic rewrite 形成新 Candidate revision，重新求解所有受影响阶段。
每种修复有明确次数上限，失败仍计预算。
多次挑中一个通过答案必须标 repair_count/selection_bias，
不宣称原始一次生成准确率提高。
达到预算/次数、无新来源或分歧不能裁决时停止到复核/隔离池。

## 7. 验证计划

| 场景 | 预期行为 |
|---|---|
| 求解请求夹带生成器答案 | 白名单拒绝，不能满足独立求解 |
| 同一模型两个 prompt | 记录输入隔离与共模限制，不声称独立真值 |
| provider 不支持视觉或 seed | 计划明确缺能力，不静默降级 |
| 模型要求执行工具/脚本 | 拒绝执行，原响应留档 |
| 单位为万元/亿元 | 显式换算依据缺失则不通过 |
| 百分比变化与百分点差 | 两种不同规则，不因数字接近视为相等 |
| divide 分母为零/跨零区间 | 保留 unknown，不出现 NaN/Infinity |
| 原始蓝图两跳，另一页有直接答案 | 新替代支持，不认定负例/必要多跳 |
| 负例全库找到答案 | 修改标签需新 revision，重跑下游 |
| 未找到答案且范围不穷尽 | unknown，排除确定拒答金标 |
| 内部乘法规则发布到 C2 | 明确阻塞，不静默删规则或输出 NumericCorrectness |
| 结构修复达到上限 | 停止且全部请求/成本可查 |

真实模型质量和供应商参数兼容需另以获授权服务与数据验证；
本设计的 replay 夹具不充当真实模型能力证据。
