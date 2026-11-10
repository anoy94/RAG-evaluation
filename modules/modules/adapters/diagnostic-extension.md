# Adapters：诊断阶段与受控文本生成扩展

- 状态：Confirmed；文档、Contracts 与详设修订：r001。
- 覆盖：Complete，仅 adapters.diagnostic.v1；核心 adapters.c2.v1/r001 不变。
- 依据：v2 §7、12–13；用户授权完成 Diagnostics 包所需的提供方设计。
- 本文件是 Adapters 的扩展契约唯一正文；不是 Diagnostics 对运行类型的复制。

## Contracts

### A1. 能力、输入和实际配置

扩展 AdapterCapabilities 的 supported_contract_profiles 可登记 adapters.diagnostic.v1。
新增 operation 为 freeze_generation、generate_with_context、reconcile_generation；
另可声明 import_diagnostic_observations。C2 JSONL Adapter 仍只有 import_run。
能力必须由具体实现验证，不能因可以导入生成记录就声明可在线执行。

FrozenGenerationSpec 由 Adapter.freeze_generation(request_ref, access_context) 产生：
- baseline_run_ref、generation_call_id、generation_request_ref。
- message/part 的实际字节及顺序；query/history/as_of/access_scope。
- context_slots：一个或多个明确、无重叠的 request_bindings 区间；
  替换可涉及同一文本 part 多个区间，按原位置降序改写。
- model_identity：provider、不可变模型 revision、部署/路由身份及已知后端指纹。
- model_parameters：temperature、top_p、seed、penalties 等全部实际参数；
  省略值只有在历史 provider 默认值及版本可确定时才可解析。
- output_budget、tokenizer_id/version、input_capacity、capacity_basis。
- template/system/developer、输出 schema、工具策略、缓存与其他请求参数。
- serialization_profile/version、source_guarantees、limitations。

返回 FrozenSpecResult={spec_ref, validation_ref, receipt_ref}。
真实数据缺失返回 validation decision=blocked/unknown 和具体缺口，不伪造 spec；
坏引用/矛盾返回 adapters.reference_invalid/target_inconsistent。
冻结只读取本地记录，不联网猜测历史默认或把模型 alias 解析为当前版本。

严格首版要求：单次最终文本生成；final_output_origin=generation_call，
最终输出与该响应逐字节一致；无后处理、后续决策或工具调用。
请求/上下文 original、complete、绑定已核验；无 image/external 通道。
tools 必须为显式禁用；没有 tool_policy 记录不自动等于禁用。
模型 alias、未知 system prompt、隐式服务端 RAG/模板、无法控制的额外指令均阻塞严格资格。
source_execution_key 与实际来源均保存；本地导入保证不升级为可信采集认证。

### A2. GenerationInvocation 与返回

GenerationInvocation：
schema_version=2.0、contract_profile、invocation_id、logical_trial_key、
pair_slot_id、arm（paired_baseline/treatment）、attempt、
spec_ref、ordered_context_refs、patch_manifest_ref、
case_association（沿用核心）、deadline_utc、timeout_ms、
resource_reservation_ref、egress_authorization_ref、provenance_ref。

Adapter 只接收实际请求、上下文及控制条件，不接收完整 TestCase、
reference_answers、Judgment、Finding、hypothesis 或预计算答案。
context refs 对 baseline 必须逐字等于原上下文，treatment 仅含批准的源证据。
patch_manifest 指定原区间与新内容；非 context 的请求字段必须与 spec 一致。
引用、权限、容量及固定条件在网络发送前再次校验，禁止静默截断和自动扩容。
token 预算必须采用目标服务可核验的计数规则；无法取得安全上界时 blocked，
字符数/近似 tokenizer 只能展示，不能声称保证容量。

公开操作：
```text
freeze_generation(request_ref, access_context) -> FrozenSpecResult
generate_with_context(invocation_ref, access_context) -> GenerationResult
reconcile_generation(request_ref, access_context) -> GenerationReconciliation
import_diagnostic_observations(request_ref, access_context) -> ObservationImportResult
load_diagnostic_observations(result_ref, access_context) -> ObservationReader
```

GenerationResult={receipt_ref, run_ref, raw_response_ref, audit_import_result}；
run_ref 在未形成可信终态时可 null。
RunRecord、Span、GenerationCall、ContextRecord、Usage 完全沿用核心所有权与字段，
Run 和原生产物仍用 adapters.c2.v1，扩展请求/回执才用 adapters.diagnostic.v1；
新增关系不塞入核心未定义字段，保存在回执。
audit_import_result 在形成合法 Run 时必填，类型为核心 RunImportResult。
Adapter 将真实请求/响应和终态转成固定 external-rag-jsonl-v1 单行快照，
保存显式映射/绑定，再通过核心 importer 形成该批次，run_ref 必须来自其中。
没有可信终态时该字段 null，不导入伪造 Run。
因此 C2 load_runs/trace_audit 可以直接消费真实批次，不需修改核心来源枚举。
本地捕获到的实际请求可标 logged，但保留采集实现和 local_import_only 来源保证。

GenerationReceipt 固定 invocation_ref、spec_ref、实际请求/patch、实际模型身份、
provider_request_key、响应/运行/计量引用、发送状态、原始响应持久化状态、
origin=live|recorded_fixture|external_import、来源限制。
send_state 为 not_sent、rejected_before_execution、completed、failed_known、
outcome_unknown。outcome_unknown 不伪造 timed_out Run。
本地读取超时不证明远端执行超时；只有明确 provider 终态才能形成对应 Run。
完整返回有非法模型答案仍是 succeeded，Evaluator 判任务格式失败；
HTTP 协议/响应解析失败不能随意补成“答案错误”。
凭据、原始错误秘密不写入回执；原始请求只保存安全业务载荷。

### A3. 首个具体在线接入

内置实现 strict_text_http_v1：显式配置 HTTPS POST、JSON、非流式单文本响应。
配置固定 endpoint（含已批准路径）、provider 协议版本、
credential_ref、不可变模型 identity、白名单请求字段和 JSON Pointer 响应字段、
tokenizer/capacity 证据、provider 默认值表及版本、超时和响应大小上限。
具体地址/凭据/版本/费用表是运行输入，不在设计里臆造。

仅支持明确的 messages role/content 文本序列、temperature/top_p/seed、
max output tokens、penalty、stop、response format 等白名单字段。
映射到 provider 的字段名需在静态注册的协议 profile 中定义；不执行任意模板/代码。
原始请求存在不支持的影响生成参数时 blocked，不能丢参数后称同条件。
只支持单文本 choice；n>1、流式、多模态、服务端工具/RAG 均 unsupported。
缺少 action/citation 的服务不猜测；Run.output.action=null。
已知任务需要引用语义时必须使用有效审定，不能因为该 HTTP 子集而删任务维度。

关闭 SDK/HTTP 自动业务重试、自动重定向、自动 fallback 和动态工具；
TLS 校验开启，endpoint 与出域批准一致。401/403 不重试，
429 仅在有明确“未执行”协议依据时允许 Runner 按冻结策略重试。
连接失败也不能仅凭异常名字认定请求没发出。
本实现默认不支持远端查询、取消或 exactly-once；reconcile 返回 unknown，
除非固定 provider 协议明确提供查询/幂等保证，并保存其证据。
reconcile 请求固定 invocation_ref、known_receipt_ref（可空）、provider_binding_ref；
已知回执先核验其 invocation 绑定，缺回执且无可核验查询则 unknown。
Reconciliation 保存 disposition=not_sent|completed|failed_known|unknown、
对应 receipt/Run（适用时）、查询依据与来源限制，不执行新生成。
本地/远端查询失败保持 unknown 或具体操作错误，绝不据此认定未执行。
响应 model/fingerprint 与固定身份冲突时记录 protocol mismatch，
不能静默把它变成合法严格试验。

recorded_generation_v1 是无网络契约夹具：每个预定 invocation 显式绑定固定响应，
记录 origin=recorded_fixture，不能当成真实独立重复或真实模型恢复效果。
真实 provider 接入验收必须在实际 endpoint 上另跑，不能用 fixture 替代。

### A4. 诊断阶段观察导入

DiagnosticStageObservation 是 Adapter 拥有的外部观测记录：
observation_key、run_ref、span_id（可空）、artifact_role、
capture、source_snapshot_key、source_assets、producer_config_ref、
build_execution_key（可空）、membership_ref、coverage_attestation_ref、
input_refs/output_refs、provenance_ref、limitations。
artifact_role 为 parser_output、chunk_inventory、index_inventory、
query_transform_output、assembly_input 或 assembly_output。
不扩充核心 StageResult 的 stage 枚举范围。

原生文本 profile adapters-diagnostic-text-scope-v1：
有序 items（occurrence_id、text_ref、source_descriptor、input/output links）、
scope_id、scope_kind、selected_document_revisions、completeness、
exhaustiveness_basis_ref。来源必须能固定到真实 SourceArtifact 谱系或明确文档 revision。
scope_kind 为 full_document_output、declared_inventory 或 selected_subset。
complete 只是生产者完整声明，需有预期清单、计数、快照与成员关联可核验；
导入成功不认证真实性，也不证明 parser/index 正确。
index_inventory 只能表示已导出的内容集合，不凭 chunk 数证明向量可检索。
全量 parser/index 不在每 query 伪造执行；多 query 可引用同一构建输出，
membership_ref 证明该 Run 实际使用的构建版本，否则仅属独立组件观察。

import request 固定完整源文件、显式映射/绑定及限额；每行按唯一键关联，
不按 query 相似度或当前时间猜测；全部来源条目必须有处置结果。
可按核心已知 RunRef 引用构建观察，但无实际使用关联时不得补造 stage_link。
规范对象先写、完整导入 manifest 最后写、回执独立，沿用 Store 提交规则。

### A5. 错误与验证

新增原因：adapters.generation_not_reproducible、capacity_unverified、
unsupported_generation_profile、egress_denied、invocation_conflict、
provider_protocol_error、outcome_unknown；仍保留核心完整性/授权/存储错误。
outcome_unknown 同时是已保存的发送状态，不等于可重试建议。

计划验证：非 context 字段变一个字符被拒；固定模型冲突；
多区间位置改写含中文/emoji；工具/alias/未知默认阻塞；
真实空 context 与缺记录分开；超容量发送次数为零；
请求已发而本地崩溃不静默重发；来源冒充全量 parser；
fixture 永远显式标识。没有执行过真实服务验证。

## 1. 所有权、依赖与范围

仅依赖 Store，不能导入 Registry、Diagnostics、Evaluator 或 Runner。
诊断关系、充分性和干预资格由消费者判断；
Adapter 只核验收到的固定请求、能力、观测和执行事实。
HTTP 客户端、parser、锁和序列化版本在实现时锁定。
核心 C2 JSONL 导入完全不需要构造本扩展或联网。
