# Adapters：运行记录与离线导入

- 模块 ID：adapters
- 文档状态：Confirmed
- 文档内容修订：r001
- 概述状态：Confirmed / r001
- 详设覆盖：Complete
- 声明范围：C2 标准运行表示、文本链路产物及 JSONL 导入。
- 在线执行、阶段重放、Oracle 和厂商 SDK：后续范围。

## 1. 职责与边界

依据：
- [总体架构 r001](../../00-system-architecture.md)。
- [目录设计 r001](../../01-directory-layout.md)。
- [横切规则 r001](../../02-cross-cutting-concerns.md)。
- [Artifact Store 契约 r002](../artifact-store/README.md#contracts)。
- [v2 设计](../../../doc/RAG测评设计_v2.md)，第 7、12、16、17 节。

本模块拥有：
- AdapterCapabilities、SystemConfig、RunRecord 和 Span。
- 检索结果、生成请求、实际上下文与输出的标准表示。
- 外部运行记录的格式映射、引用核验和导入来源。
- 运行记录的可用性、完整性及计量声明。

本模块不负责：
- 读取黄金答案判断运行结果正确与否。
- 解析 DatasetVersion 或裁定用例绑定有效性。
- 选择 A/B 代表样本或某次最佳尝试。
- 调度实验、重试业务请求、计算指标及发布报告。

Runner 提供固定的用例关联信息并检查真实输入是否对齐。
Mapper 和 Evaluator 使用实际记录，决定对应评价能否计算。

导入成功表示记录按协议接收，
不表示被测系统成功、数据真实或评价条件完整。
日志真实性与采集完整性仍受来源保证限制。

## Contracts

契约状态：Confirmed。
契约修订：r001。
格式版本：2.0。
contract_profile：adapters.c2.v1。

### C1. 身份、修订与来源

RunKey 包含：
- namespace：明确的来源命名空间。
- run_id：一次来源运行的稳定记录身份。
- revision：该标准记录的不可变内容修订。

namespace 区分不同系统实例、日志生产者及数据来源。
不能只凭外部 run_id 相同跨来源合并记录。

有稳定来源运行 ID 时，
run_id 由 namespace 和来源运行 ID 的规范内容摘要生成。
原始 ID 保存在受控 provenance 中。

没有稳定 ID 时，
run_id 由原始文件引用及记录位置派生，
并记录 execution_identity_unverified。
这只能支持同一输入的重复导入去重，
不能证明不同日志文件中的记录属于独立执行。

revision 是标准 RunRecord 排除自身 revision 字段后，
按 rag-eval-json-v1 规范序列化所得内容的 SHA-256。

RunRef 包含：
- run_key。
- record_ref：完整 RunRecord 的 ArtifactRef。

同一 RunKey 对应唯一规范内容。
同一来源执行的新记录修订不构成一次新的业务执行。

source_execution_key 单独记录已知的真实来源执行身份。
Runner 合并多个批次时检查该身份：
- 相同执行、相同记录可复用。
- 相同执行的不同记录修订需要明确选择。
- 不得将重导入或修订计为独立样本。

不提供按裸 run_id 或 latest 查找的接口。
读取绑定具体 RunRef 和导入批次。

### C2. 用例关联与 SystemConfig

CaseAssociation 包含：
- case_id、case_revision。
- case_locator_ref：固定关联说明的 ArtifactRef。
- declared_input_ref：计划用例输入的 ArtifactRef。
- association_basis：explicit_id 或 explicit_position。
- provenance_ref。

case_locator_ref 由 Runner 提供，
用于绑定具体数据版本和用例身份。
其业务结构由所属模块解释；
Adapter 不解析 Registry 的私有数据或重新定义 CaseKey。

explicit_position 只能使用冻结文件中的明确记录位置。
不能按文件当前顺序、query 相似度或答案内容猜测关联。

这些字段表示导入时的关联声明。
真实输入、数据版本和访问条件是否一致，
由 Runner 调用相应提供方契约检查。

SystemConfig 包含：
- schema_version、contract_profile。
- config_id。
- revision：配置规范内容排除自身 revision 后的摘要。
- components：已知组件配置列表。
- source_config_ref：可为空。
- provenance_ref。
- limitations。

每个组件包含：
- stage：parse、chunk、embed、index、query_transform、
  retrieve、fuse、rerank、compress、generate、tool 或 other。
- component_id。
- version：可为空。
- version_kind：immutable、alias 或 not_recorded。
- parameters_ref：可为空，固定非秘密参数。
- observation_basis：logged、declared 或 not_recorded。

未知组件或版本明确保留为 not_recorded。
导入时的当前配置不能冒充历史运行配置。

模型别名可以作为来源事实保存，
但需要 alias 标记和对应复现限制。
不将其改写成猜测的具体模型版本。

RunRecord 分别记录：
- declared_system_config_ref。
- observed_system_config_ref：可为空。

两者不一致时保留差异。
Adapter 不用声明配置覆盖实际记录。

配置不保存凭据值。
声明配置不证明运行实际采用了全部配置。

### C3. 数据捕获与缺失语义

CaptureDescriptor 包含：
- availability：available、not_recorded、withheld
  或 not_applicable。
- content_ref：available 时必填，否则为 null。
- format、format_version：available 时必填。
- fidelity：original、redacted、summary 或 reconstructed。
- completeness：complete、partial 或 unknown。
- provenance_ref。
- reason：存在限制时必填。

没有内容时，fidelity 和 completeness 为 null。
available 的内容必须通过 Store 完整性及格式校验。

original 表示保留了来源记录中的原内容，
不单独证明日志采集真实或完整。

redacted、summary、reconstructed 必须保留变换身份和限制。
不能把重构 prompt 标记为实际生成请求。

实际空内容与没有记录分开：
- 完整检索返回零项：available、original、complete，
  内容为明确空列表。
- 日志没有检索字段：not_recorded。
- 内容因权限不提供：withheld。
- 阶段确实不适用：not_applicable，并说明依据。

没有某阶段的日志，不能据此认定该阶段未执行。

C2 解释本模块定义的原生 profile。
其他内容可作为 external descriptor 保存，
但不能自动进入需要原生语义的指标。

### C4. RunRecord

RunRecord 包含：
- schema_version、contract_profile。
- run_key。
- source_execution_key：可为空。
- case_association。
- declared_system_config_ref。
- observed_system_config_ref：可为空。
- observed_input_ref。
- attempt：正整数。
- repetition：正整数。
- attempt_basis：logged 或 assigned。
- status：succeeded、failed、timed_out 或 cancelled。
- output：CaptureDescriptor。
- spans：CaptureDescriptor。
- stage_results：StageResult 引用列表。
- generation_calls：GenerationCall 引用列表。
- final_output_origin。
- timing_ref。
- usage_ref。
- provenance_ref。
- limitations。

observed_input_ref 指向 adapters-observed-input-v1：
- query。
- history。
- as_of。
- access_scope。

每个字段分别记录：
- state：recorded、not_recorded 或 withheld。
- value：recorded 时提供。
- basis：logged、declared 或 transformed。
- provenance_ref。

已记录的 null 与未记录分开。
记录 history=[] 表示明确观察到空历史，
不能用它补齐未记录的历史。

query、history 和 scope 保留来源值。
不能从用例输入补写后将其标记为 logged。
C2 不支持的历史结构保留原始引用及限制，
不静默压平成文本。

attempt 是该记录被关联到的尝试序号。
assigned 只表示外部明确分配，
不证明更早的真实业务尝试不存在。

导入操作重试不增加业务 attempt 或 repetition。
内部模型重试由 Span 和 Usage 保留，
不能伪装成多个独立用例样本。

status 必须具有明确来源或固定协议依据。
不能仅因存在 answer 字段就推断 succeeded。
没有终态依据的记录不能构造上述四种终态 Run。

succeeded 表示来源系统完成其处理，
不表示答案正确、格式合规或任务成功。

失败 Run 可以保留部分输出。
成功 Run 也可以存在空答案或输出未披露。
这些情况由 Evaluator 解释。

输出原生 profile 为 adapters-output-v1：
- action：记录的行为，可为空。
- answer：字符串或 null；空字符串与 null 分开。
- citations：Citation 列表。

action 采用 answer、abstain、clarify、tool_required、
policy_block、other 或 null。
不能仅凭答案措辞猜测行为标签。
other 保留原始行为声明。

final_output_origin 包含：
- kind：generation_call、postprocessed、
  system_decision 或 unknown。
- generation_call_id：适用时提供。
- provenance_ref。

多个生成调用存在时，
不能默认取最后一个调用的上下文解释最终答案。

### C5. Span 与真实链路

Span 包含：
- span_id：Run 内唯一。
- stage。
- parent_span_id：可为空。
- links：带关系类型的 Span 引用。
- source_operation_id：可为空。
- status：succeeded、failed、timed_out、cancelled、
  unfinished 或 unknown。
- input_refs、output_refs。
- started_at、ended_at：可为空，提供时带时区。
- duration：可为空，遵守 C8。
- usage_refs。
- error_ref：可为空，受控错误明细。
- provenance_ref。

stage 使用 C2 的阶段枚举。
同一阶段可以出现多次，不强制一阶段一个 Span。

parent 关系必须无环。
links 的关系为 data_from、follows_from 或 related_to。
前两者的依赖关系必须无环；
related_to 不证明执行先后或数据传递。

原生引用必须在同一 Run 中闭合。
部分 Trace 缺少的外部父节点，
保留为明确的 external_parent 声明，
不能伪造本地 Span 或忽略缺口。

父子关系不单独证明真实数据流。
阶段输入输出和 GenerationCall 还需要明确引用关联。

循环、并行和多次生成按真实调用分别记录。
C2 不将不同独立运行的检索和生成结果拼接为一次 Run。

Span 的显示顺序不代表因果顺序。
时间戳相近、query 相同或内容相似均不足以建立链路。

### C6. 检索与上下文产物

StageResult 包含：
- result_id：Run 内唯一。
- stage：retrieve、fuse、rerank 或 compress。
- span_id：无 Trace 时可为空。
- generation_call_id：实际关联已知时提供。
- query_capture。
- candidates_capture。
- search_scope_ref：过滤及检索范围，可为空。
- provenance_ref。

原生候选列表 profile 为 adapters-candidates-v1。
每项包含：
- occurrence_id：本列表内唯一。
- runtime_item_id：来源 chunk 或候选 ID，可为空。
- rank：从 1 开始的正整数。
- score：有限十进制字符串或 null。
- score_type：score 非空时必填。
- scoring_config_ref：可为空。
- content：CaptureDescriptor。
- source_descriptor。
- provenance_ref。

source_descriptor 保存已知：
document_id、document_revision、SourceArtifactKey
及来源定位声明。

未知 revision 保留缺失，
不能补成 latest 或拿当前文档冒充历史来源。

rank 不重复。
来源 rank 从 0 开始时，须由映射明确转换并保留原值。
不能通过不同路线的裸 score 重排出“统一排名”。

检索、融合、重排和压缩各自保留结果集合。
压缩产物不覆盖原候选内容。

ContextRecord 使用 adapters-context-v1：
- context_id。
- generation_call_id。
- items：实际传入内容的有序列表。
- completeness。
- request_bindings。
- provenance_ref。

每个 context item 包含：
- occurrence_id：本上下文内唯一。
- channel：text、image 或 external。
- content_ref。
- source_candidate_refs：已知时关联 StageResult 及 occurrence_id。
- source_descriptor。
- provenance_ref。

文本 content_ref 保存实际传入的文本片段，
包括影响内容解释的引用包装。
不能用完整源 chunk 替换被截断后的实际片段。

保留重复项、实际顺序与截断结果。
不按候选排名自动重建上下文，
不因为内容重复而删除实际传入次数。

request_bindings 对文本提供：
- context occurrence_id。
- message_index、part_index。
- char_span：Unicode code point 半开区间。
- 对应 generation_call_id。

提供绑定时，范围文本必须与实际 context 内容一致。
没有可核验的请求关联时，
上下文只能保留为来源声明，
不能自动成为“已核验实际发送上下文”。

图像只在取得对应实际字节时记录 image 内容引用。
只有 URL、caption 或 OCR 时不能冒充原图输入。
C2 不自动下载资产或执行视觉语义核验。

### C7. 生成请求、调用与引用

GenerationCall 包含：
- generation_call_id：Run 内唯一。
- span_id：可为空。
- source_request_id：可为空。
- system_config_ref。
- request：CaptureDescriptor。
- context：CaptureDescriptor。
- response：CaptureDescriptor。
- timing_ref。
- usage_refs。
- provenance_ref。

每个上下文、请求和响应都绑定本次调用。
不能根据 query 相同在不同请求间替换内容。

原生请求 profile 为 adapters-generation-request-v1：
- messages：实际发送顺序。
- model_config_ref。
- output_budget_ref：可为空。
- tool_policy_ref：可为空。
- provenance_ref。

每个 message 保存：
- role：system、developer、user、assistant 或 tool。
- parts：有序内容列表。
- tool_call_id：适用时提供。

每个 part 包含：
- kind：text、image 或 external。
- content_ref。
- format、format_version。

text 按严格 UTF-8 解码。
非标准工具结构使用 external 保留，
不执行其中的调用或指令。

C2 只保存与解释已有记录，
不因此声明具有在线多模态或工具执行能力。

若只能取得最终上下文而缺少真实请求，
分别记录可用性和 request_binding 限制。
模板重新渲染得到的结果属于 reconstructed。

Citation 包含：
- citation_id。
- answer_span：可为空，提供时核对答案中的范围。
- raw_target：来源记录中的引用标识或地址。
- resolution_status：resolved、unresolved 或 not_recorded。
- target_item_ref：resolved 时提供本 Run 的具体产物关联。
- provenance_ref。

模型输出不存在的引用目标属于可观察表现。
可以保留为 unresolved Citation，
不能作为 Store 的合法 ArtifactRef 接受，
也不能因目标不存在而删除整条运行。

引用是否支持主张由 Evaluator 判定。
目标存在不表示引用正确；
引用 ID 与 gold ID 相同也不表示命中黄金证据。

### C8. 时间与计量

Measurement 包含：
- value：非负有限十进制字符串，未知时为 null。
- unit。
- basis：monotonic、reported、estimated 或 unknown。
- scope_id：测量边界与计时域。
- provenance_ref。
- reason：未知或受限时必填。

Timing 包含：
- started_at、ended_at：可为空，带时区。
- e2e：毫秒 Measurement。
- ttft：毫秒 Measurement。
- streaming：true、false 或 null。
- clock_description_ref：可为空。

E2E 使用来源记录的整次请求边界。
不使用导入耗时替代，也不累加并行 Span。

跨进程不直接相减单调时钟。
只有墙钟差值时保留 estimated 及其误差限制。
缺少时区需要显式映射规则，不能使用当前机器时区猜测。

仅在相同测量边界且可比较时检查 ttft <= e2e。
无法确认可比性时保留限制。

Usage 使用 adapters-usage-v1：
- entries：计量记录列表。
- provenance_ref。
- limitations。

每项包含：
- usage_id。
- provider_request_key：已知时提供。
- scope_id。
- scope_kind：request、run_total 或 stage_total。
- quantities：名称、值、单位及 observed/reported/estimated/unknown。
- pricing_ref：可为空。
- provenance_ref。

数量未知时为 null，不能默认为零。
token、请求、图像等计数必须为非负整数。
金额使用有限十进制字符串及明确币种。

同一个 scope 的总量和细项分别保留，
不能将二者直接相加。
同一 provider_request_key 的重复计量需要一致核验。

缺少请求关联时，不猜测多条记录是否重复计费。
有范围重叠或未知项时，
不能仅对已知子集求和并声称得到完整总成本。

来源运行成本与本次导入操作分别解释。
导入重试不代表再次调用被测系统，
也不能使来源调用成本重复增加。

### C9. 能力、映射与公开接口

AdapterCapabilities 包含：
- adapter_id、adapter_version。
- operations。
- supported_contract_profiles。
- supported_artifact_profiles。
- limitations。

C2 JSONL Adapter 的 operations 仅包含 import_run。
不声明 e2e_run、retrieve_only、generate_with_context
或 stage_replay。

支持读取 Span 或生成上下文，
表示实现可以处理这些格式，
不证明每条 Run 都具备相应数据。
实际可计算性由逐条产物决定。

C2 支持 external-rag-jsonl-v1：
每行是一条已结束来源运行，
可包含该运行的输入、输出、阶段及生成调用记录。

不跨行按相似 query 自动拼装 Trace。
拆散事件日志需先通过独立、明确的转换形成该协议。

RunImportRequest 通过固定 ArtifactRef 提供，包含：
- source_ref：原始 JSONL。
- mapping_config_ref。
- bindings_ref：逐条固定关联表。
- declared_system_config_ref。
- namespace。
- import_limits。

bindings_ref 为本模块定义的关联表：
来源记录键或明确行位置 → CaseAssociation、
attempt、repetition。

同一记录必须唯一关联。
未匹配、多重匹配或未明确处置的输入记录均报错。
不能只导入有答案或成功匹配的子集。

映射配置固定：
- mapping_id、mapping_revision。
- input_format、input_format_version。
- 字段 JSON Pointer。
- 明确枚举映射及单位转换。
- 常量及其声明依据。
- 引用查找表。
- 明确排除的字段及理由。
- mapper_version。

C2 允许复制、重命名、有限枚举映射、
引用查找及 s/ms/us 到毫秒的明确转换。
常量写入记录时标为 declared。
只有来源协议明确保证的固定值，
才能按协议依据解释为运行事实。

不执行脚本、模板、任意表达式或模型。
不自动打开输入中的路径或 URL。

原始输入中的 reference、gold 和 judge 字段
不能映射成实际检索结果、上下文或业务输出。
原始文件保留为受控来源，未使用字段明确处置。

import_limits：
- max_runs：正整数，默认 100000。
- max_record_bytes：正整数，默认 4194304。
- max_spans_per_run：正整数，默认 10000。
- max_items_per_collection：正整数，默认 10000。
- max_nesting_depth：正整数，默认 64。

解析期间执行限制。
字节产物还受 Store 的 max_blob_bytes 约束。

公开入口：

```text
create_jsonl_import_adapter(
    config,
    artifact_store
) -> ImportAdapter

describe() -> AdapterCapabilities

import_run(
    import_request_ref,
    access_context
) -> RunImportResult

load_runs(
    import_result,
    access_context
) -> RunBatchReader
```

config 固定 importer_id、importer_version
及所支持的格式 profile，不包含业务凭据。

import_run 的输入是一份 JSONL 导入请求，
返回完整批次结果；单条来源运行使用单行文件。

RunImportResult 包含：
- batch_ref：固定导入清单引用。
- receipt_ref：本次操作回执。

清单包含：
- 原始文件、映射、关联及配置引用。
- importer_version。
- 按 RunKey 排序的 RunRef 集合。
- 记录数量及状态数量。
- 映射报告和限制报告。
- batch_id。

batch_id 是清单排除自身 batch_id 后的规范摘要。
RunRecord 不引用包含自身的批次清单，避免自引用。

回执记录：
operation_id、批次引用、接收主体、UTC 接收时间、
Store 策略 revision 及 local_import_only 身份保证。

相同固定输入得到相同批次身份。
每次导入保留独立回执。

RunBatchReader 提供：
- manifest。
- RunRef 集合。
- 按完整 RunKey 读取 RunRecord 及其原生产物。

Reader 校验回执、批次、数量、内容和引用。
不自动选择最后一次或最佳一次运行。

### C10. 错误契约

| reason_code | 含义 |
|---|---|
| adapters.invalid_input | 参数、字段或 JSONL 不合法 |
| adapters.unsupported_schema | 不支持的版本或原生 profile |
| adapters.limit_exceeded | 行、记录、Span 或集合超限 |
| adapters.mapping_incomplete | 必需映射或字段处置缺失 |
| adapters.binding_invalid | 记录关联缺失、重复或冲突 |
| adapters.duplicate_run | 批次中重复来源运行 |
| adapters.unresolved_run_status | 无法确定支持的终态 |
| adapters.reference_invalid | 正式引用缺失、跨域或关联不一致 |
| adapters.trace_inconsistent | Span、调用及产物关系不一致 |
| adapters.integrity_mismatch | 内容、摘要、计数或回执不一致 |
| adapters.access_denied | 当前读写未授权 |
| adapters.storage_failure | Store 操作失败，保留原始原因 |

合法但缺少某类可选记录时，
保留 CaptureDescriptor 和 limitations，
不抛成系统运行失败。

来源系统的 failed、timed_out、cancelled 是 Run 状态，
不等于导入操作失败。
非法日志和存储异常不能伪造成失败 Run。

## 2. 导入、提交与恢复

### 2.1 导入顺序

```text
授权并核验请求及所有输入引用
→ 严格 UTF-8 JSONL 解析
→ 显式映射与逐条关联
→ 生成规范输入、配置、产物和 Run
→ 检查身份、状态、引用、Trace 与计量
→ 保存完整规范产物
→ 保存导入报告
→ 最后提交批次清单
→ 保存操作回执
→ 返回 RunImportResult
```

拒绝 BOM、重复 JSON 键、空记录和非法数值。
允许 LF、CRLF 及文件末尾换行。
逐行实施字节和解析深度限制。

批次中间一条非法时，整批不返回成功结果。
真实失败 Run 不作为坏行跳过。

批次内相同来源运行重复出现时拒绝，
不把重复日志当作多次独立执行。

同一运行的全部原生产物须能追溯到该行、
该来源执行及对应生成调用。
关联声明与实际记录不一致时保留或报错，
不能用其他运行的内容补齐。

### 2.2 持久化边界

全部产物通过 Artifact Store 保存。
本模块不建立另一套内容寻址库或全局可变 Run 表，
不直接写入 Runner 的实验目录。

已有规范产物但批次清单未提交：
- 不构成成功导入批次。
- 不允许把已保存子集声明为完整导入。

批次清单已提交但回执保存失败：
- 本次操作明确未完成。
- 返回可用的安全批次关联用于排查。
- 重试可复用批次并生成新回执。

正式实验是否接收该批次，
由 Runner 的实验提交规则决定。
存在导入批次不自动产生完整实验。

### 2.3 幂等、并发与中断

相同请求重试只进行本地校验和保存，
不调用业务系统或模型。

Store 负责不可变字节的并发提交。
本模块不并发追加共享 JSONL 文件。

相同固定请求产生相同规范对象和批次清单。
两个并发调用可以复用相同字节，
各自保留独立操作回执。

恢复通过原 import_run 请求重试：
- 已提交字节经完整核验后复用。
- 缺失产物重新生成。
- 不覆盖旧 Run 或旧回执。
- 不从临时文件自动宣布导入完成。

本地导入取消不改写来源 Run.status。
它只表示本次导入尚未完成。

### 2.4 消费与缺口

Runner 必须依据冻结关联表检查：
- 每条 Run 对应的正式数据版本和用例。
- 实际输入与声明输入的一致性。
- 系统配置、attempt、repetition 及选择策略。
- 预期记录集合中缺失的运行。
- 跨批次重复执行及冲突修订。

这些检查不能由 Adapter 的格式接收成功替代。

Planner 根据逐条实际 CaptureDescriptor 解析指标依赖。
能力声明、标签或配置名称不能替代真实产物。

读取途中发现损坏或引用失效时明确失败。
调用方不得将已读子集视为完整批次。

## 3. 安全与实现安排

唯一业务依赖：
[Artifact Store / Contracts r002](../artifact-store/README.md#contracts)。

使用 ArtifactRef、StoreAccessContext、
put_blob、open_verified 及必要的源资产读取接口。
不导入 Registry、Runner、Review 或 Evaluator。

所有导入引用位于同一 Store 和 access_domain。
读取、重复命中和重新导入均检查当前权限。

外部请求、源文、输出和错误作为数据处理。
不执行其中的工具调用、脚本或提示指令。
请求产物不包含认证头或秘密值。

日志只记录安全身份、行号、阶段、状态和原因，
不输出完整输入、上下文或答案。

计划入口：
- rag_eval.adapters.contracts。
- rag_eval.adapters.api。

内部职责包括：
- 格式与产物校验。
- 固定 JSONL 映射。
- 引用、调用及 Trace 一致性检查。
- 批次提交和读取。

私有文件和类的拆分由实现决定。
无需在线 SDK、数据库或网络访问。

运行规模由显式限额和 Store 大小限制约束。
本轮不承诺吞吐或时延目标，尚未进行性能基准验证。

## 4. 验证计划

以下为计划验收，尚未执行：

| 场景 | 应观察到的结果 |
|---|---|
| 只有答案和明确成功状态 | 可导入，缺少阶段数据的指标不可计算 |
| 有答案但没有终态依据 | unresolved_run_status |
| 来源系统真实失败 | 导入成功，Run 保留 failed |
| 导入中断 | 不把来源 Run 改为 cancelled |
| 日志没有 history | 保留 not_recorded，不补成空历史 |
| 用例输入被填入来源输入字段 | 保留 declared，不冒充 logged |
| 同输入重复导入 | 批次身份相同，不增加业务尝试 |
| 不同来源使用相同外部 ID | namespace 区分身份 |
| 同一来源执行出现多次 | 拒绝批次重复，跨批次交给 Runner 检查 |
| 同一次执行的新记录修订 | 不计为新的独立执行 |
| 检索结果为空与字段缺失 | 两种状态明确区分 |
| 原候选有证据、实际上下文被截断 | 保留两个不同产物 |
| 上下文中同一片段出现两次 | 顺序及次数保留 |
| 两个生成调用有不同上下文 | 分别绑定，不按 query 混合 |
| 请求绑定范围与上下文不一致 | trace_inconsistent |
| 只有重构 prompt | fidelity=reconstructed |
| 只有 OCR 或图片 URL | 不宣称取得实际图像字节 |
| 模型引用了不存在的来源 | 保留 unresolved Citation |
| 父 Span 成环或引用悬空 | 明确拒绝或保留允许的外部缺口 |
| 并行 Span | 不累加成 E2E |
| 缺少成本或 token | unknown，不记零 |
| 总量与细项同时存在 | 不重复相加 |
| 原始日志含 gold 字段 | 不映射为实际上下文或输出 |
| 批次中间一行非法 | 不返回成功的部分批次 |
| manifest 提交后回执失败 | 重试复用批次，形成新回执 |
| 两进程导入相同请求 | 相同批次，独立操作回执 |
| 读取半批后损坏 | 不作为完整批次提交 |
| 权限撤销后重读 | access_denied |

验证使用真实本地 Store、受控 JSONL、
明确预期的 Trace/上下文和提交故障点。

演示使用合成运行记录时标注演示身份，
不把它作为真实系统质量或性能证据。

## 5. 设计进度与后续边界

| 设计单元 | 审阅状态 | 内容修订 | 位置 |
|---|---|---|---|
| 概述与边界 | Confirmed | r001 | [第 1 节](#1-职责与边界) |
| 标准记录与公开接口 | Confirmed | r001 | [Contracts](#contracts) |
| 导入、提交与恢复 | Confirmed | r001 | [第 2 节](#2-导入提交与恢复) |
| 安全与实现安排 | Confirmed | r001 | [第 3 节](#3-安全与实现安排) |
| 验证计划 | Confirmed | r001 | [第 4 节](#4-验证计划) |

Complete 仅表示声明的 C2 本地导入范围设计完成。
应用实现和验收尚未执行。
C2 整体仍为 Design pending。

在线 run、retrieve_only、generate_with_context、
实时采集、阶段重放及复杂事件日志重建继续留在后续范围。
原生视觉评价协议也不属于本轮。

公开接口、数据含义、权限、持久化、
状态及验收行为变化需要确认。
私有实现可在已确认边界内决定。
