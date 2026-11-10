# Evidence Mapper：文本证据定位与范围核验

- 模块 ID：evidence-mapper
- 文档状态：Confirmed
- 文档内容修订：r002
- 修订边界：本次仅进度摘要更新；业务单元及公开契约保持各自既有修订。
- 概述状态：Confirmed / r001
- 详设覆盖：Complete
- 声明范围：C2 文本证据到候选及生成上下文的映射。
- 语义改写识别、视觉定位、全量索引与解析产物协议：后续范围。

## 1. 职责与边界

依据：
- [总体架构 r001](../../00-system-architecture.md)。
- [目录设计 r001](../../01-directory-layout.md)。
- [横切规则 r001](../../02-cross-cutting-concerns.md)。
- [Artifact Store 契约 r002](../artifact-store/README.md#contracts)。
- [Dataset Registry 契约 r002](../dataset-registry/README.md#contracts)。
- [Adapters 契约 r001](../adapters/README.md#contracts)。
- [v2 设计](../../../doc/RAG测评设计_v2.md)，第 5、8、9 节。

本模块拥有：
- MappingRequest、MappingConfig 和 EvidenceMapping。
- 固定参考文本中的锚点解析。
- 来源身份核对、文本位置及跨片段覆盖。
- 匹配状态、检查范围、依据和不确定原因。
- 映射结果的不可变保存与重复读取。

本模块不负责：
- 生成或修改 EvidenceUnit、fact_id、参考答案和充分组。
- 把新发现来源直接提升为已认可替代证据。
- 判断完整答案正确、主张蕴含或上下文业务充分。
- 计算事实覆盖率、检索分数及门禁。
- 自动执行人工仲裁或调用模型裁判。

Mapper 回答：
指定源证据的文本锚点，是否能在指定运行产物中可靠定位。

Evaluator 决定：
这些内容是否提供了所需事实，是否满足条件和充分组。

quote 命中不自动证明完整事实得到支持。
指定来源未出现也不证明不存在其他支持来源。

## Contracts

契约状态：Confirmed。
契约修订：r001。
schema_version：2.0。
contract_profile：evidence-mapper.c2.v1。

### C1. 请求与固定范围

MappingRequest 包含：
- dataset_version_ref：正式 DatasetVersionRef。
- case_key：CaseKey。
- run_ref：RunRef。
- evidence_keys：明确、无重复的 EvidenceKey 列表。
- target_selector。
- config_ref：MappingConfig 的 ArtifactRef。

EvidenceKey 必须存在于所选正式版本，
并与用例绑定快照一致。
不能通过裸 evidence_id 搜索“最新证据”。

evidence_keys 可以为空。
此时返回明确的空映射批次及 no_selected_evidence，
不能生成零分或伪造无答案结论。

target_selector 支持：

stage_candidates：
- stage：retrieve、fuse、rerank 或 compress。
- result_id：明确阶段实例，可为空。
- rank_limit：可为空；提供时为正整数。

generation_context：
- generation_call_id：明确调用，可为空。

实例未指定时：
- 只有一个兼容实例：可选定，并保存解析结果。
- 没有记录：返回 unresolved / stage_not_recorded。
- 存在多个实例：返回 ambiguous / stage_instance_ambiguous。

显式指定不存在的实例属于引用错误。
不能自动取最后一次或最成功的一次调用。

rank_limit 表示只检查 rank <= K 的候选。
结果绑定该范围，不外推到完整候选集或索引。
无法解释的 rank 缺口影响范围完整性，
不能自动补齐或重排。

Mapper 核对 Run 中显式 case_id、case_revision。
完整数据版本、实际输入及系统配置对齐由 Runner 负责。
映射结果固定请求的版本组合，
不单独证明 Run 与该数据版本绑定正确。

### C2. MappingConfig

MappingConfig 包含：
- config_id、config_revision。
- mapper_id、mapper_version。
- normalizer_profile：exact_v1 或 ascii_whitespace_v1。
- source_identity_policy。
- limits。

source_identity_policy：
- source_artifact_required：要求明确资产身份及其固定谱系。
- document_revision_allowed：允许来源明确声明
  document_id、document_revision。

默认采用 document_revision_allowed，
并在结果中保留 declared_document_revision 保证级别。
它不等于经过签名认证的来源证明。

只有相同文本但缺少来源身份时，
可以生成诊断候选，不能自动成为可靠来源映射。

limits 包含：
- max_evidence_units：正整数，默认 1000。
- max_target_items：正整数，默认 10000。
- max_text_codepoints：单个文本上限，默认 2000000。
- max_total_codepoints：本次处理总量上限，默认 20000000。
- max_occurrences_per_anchor：正整数，默认 1000。

不隐式缩小 top-K、截断文本或忽略多余匹配以满足限额。
超限按错误契约返回，
不能以“已经没有更多候选”结束搜索。

同一固定请求、实现版本及配置产生相同规范结果。
处理时间和操作主体放入独立回执。

### C3. 参考锚点解析

C2 支持 Registry 的 text_quote locator。
参考表示必须：
- 属于 EvidenceUnit 指定的快照和文档 revision。
- 使用指定 representation_id、representation_revision。
- 能以严格 UTF-8 解码。
- 通过 Store 完整性核验。

字符位置使用 Unicode code point 半开区间。
不使用 UTF-16 code unit、字节偏移或 grapheme cluster
替代该坐标。

存在 char_span 时：
- 核对范围合法。
- 核对原始范围与 quote 一致。
- 使用该明确位置作为参考锚点。

没有 char_span 时：
- 在固定参考表示中枚举 quote 位置。
- 使用明确可核验的 prefix、suffix 消歧。
- 仍有多个来源位置时保留 ambiguous。

C2 将 prefix、suffix 用于相邻文本核对。
无法按该规则核验时，不猜测它们对应的章节范围。
section_path 作为描述保留，
不凭章节名称虚构字符区间。

重复出现在源文不同位置的同一 quote，
不能自动取第一次出现。

同一个明确参考锚点被运行产物重复引用，
属于多个有效出现位置，不因此成为参考歧义。

external locator、图像区域或未标准化结构，
返回 not_applicable / unsupported_locator。
不能从 descriptor 推断页面、bbox 或表格精度。

源文 modality=chart 但 locator 指向固定文本表示时，
可以执行文本映射；
结果不表示完成图像或图表视觉核验。

### C4. 文本归一化与坐标恢复

exact_v1 不改变任何字符。

ascii_whitespace_v1 只将以下字符组成的连续非空序列
替换为一个 U+0020：
- U+0020 空格。
- U+0009 TAB。
- U+000A LF。
- U+000D CR。

不删除空白，不去除首尾空白。
其他 Unicode 字符保持不变。

不执行：
- 大小写折叠。
- 全半角转换。
- Unicode 兼容归一化。
- 标点、负号或比较符删除。
- 数字、中文数词、日期或单位转换。
- OCR 字符纠错。
- 词语替换或语义相似匹配。

因此，“1 000”不会被改成“1000”，
“未完成”不会与“完成”匹配，
“万元”不会与“亿元”匹配。

先尝试原始精确匹配。
配置允许时，再尝试固定空白归一化匹配。

每个归一化字符保存到原始字符范围的映射。
结果同时保存：
- 原始参考范围。
- 原始目标范围。
- 必要的归一化范围。
- 使用的 normalizer 版本。

归一化命中须回到原始文本重新核验。
不得只保存归一化字符串的位置，
导致无法回查实际输入。

### C5. 来源身份与目标可见性

来源身份分为：
- source_artifact：匹配固定 SourceArtifactKey
  或可核验的固定父资产谱系。
- declared_document_revision：符合配置的明确文档及 revision 声明。
- text_only：只有文本相似或相同。
- unknown：没有足够来源信息。

来源声明与所选快照不一致时，
不能用相同 quote 强行认定为同一黄金来源。

不同 revision 中出现同一数字或同一句话，
仍保留为不同来源身份。

目标读取使用 Adapter 的 CaptureDescriptor。
可靠映射要求内容 available、原生格式可解释，
并保留 fidelity、completeness 和来源限制。

not_recorded、withheld 或只取得 summary、reconstructed 时，
返回 unresolved，并保留具体原因。
可以保存已有文本候选，不能冒充实际完整产物。

generation_context 额外核对：
- 对应 GenerationCall。
- 原始生成请求和上下文记录。
- request_bindings 的调用身份、消息及 part。
- 字符范围与 context 内容一致。
- 实际项目顺序。

缺少真实请求或无法核验绑定时，
不能确认映射针对实际发送上下文。
结果为 unresolved / request_binding_unverified，
诊断候选仍可保留。

完整性分别记录：
- 目标集合是否完整。
- 每个目标内容是否完整。
- 请求绑定是否核验。
- 搜索是否完成。
- 来源身份覆盖是否完整。

一个 complete 标记不能替代其他条件。

### C6. 单片段与跨片段映射

匹配方法：
- exact：原始文本精确匹配。
- normalized：固定空白归一化后匹配。
- joint_literal：多个实际片段共同覆盖同一参考锚点。
- candidate_only：保留待判断候选，不确认可靠映射。

单片段匹配需要同时确定：
- 参考锚点。
- 目标文本中的位置。
- 配置要求的来源身份。
- 与该参考位置的一致性。

短 quote 或源文重复摘录，
还需要足够位置上下文。
仅有相同数字不能证明对应实体、期间或计划/实际一致。

C2 可将目标完整文本与相同来源的固定参考表示比对，
推导其连续源区间。
只有区间唯一且内容核验通过时，
才能用于来源位置判断。

无法唯一对齐的候选保留歧义，
不按最高相似度自动接受。

跨片段覆盖要求：
- 所有片段属于同一快照、文档 revision 和参考表示。
- 每个片段都有经过核验的源区间。
- 使用实际目标内容与其实际顺序。
- 所选参考锚点的全部必要字符区间均被覆盖。
- 归一化差异符合 C4。

以区间并集检查覆盖：
- 重叠不重复增加覆盖。
- 未覆盖字符不能从参考源文自动补入。
- 不跨文档或不同 revision 拼接。
- 不把两个不相邻片段之间的原文视为实际输入。

joint_literal 保存全部实际片段位置。
它表示参考锚点被这些片段共同保留，
不声称运行时存在一段虚构的连续文本。

无论使用哪种方法，
语义关系、条件完整性与事实充分性仍由 Evaluator 检查。

### C7. EvidenceMapping 与状态

每个 EvidenceMapping 对应：
一个 EvidenceKey × 一个明确目标范围。

包含：
- schema_version、contract_profile。
- mapping_id。
- dataset_version_ref、case_key、evidence_key。
- fact_id：仅用于消费者关联，不在本模块计分。
- run_ref。
- resolved_target_scope。
- status。
- reason_code。
- match_method。
- identity_basis。
- reference_anchor。
- matches：已确认位置。
- candidates：待判断位置。
- inspection_summary。
- reference_review_scope。
- support_assessment：C2 固定为 not_assessed。
- config_ref、mapper_version、normalizer_version。
- dependency_refs。
- limitations。

mapping_id 是上述规范内容排除自身 mapping_id 后的 SHA-256。
不包含操作时间或指向自身的引用。

每个已确认位置包含：
- 目标产物和调用/阶段实例。
- item occurrence_id。
- 原始字符范围。
- 对应参考范围。
- 来源身份与核验方法。
- joint_literal 的片段分组关系。

inspection_summary 包含：
- 检查的 item 数量及范围。
- 目标集合和内容完整性。
- 请求绑定核验状态。
- 搜索完成情况。
- 未知来源、外部通道及不可解释内容范围。

状态定义：

| status | 含义 |
|---|---|
| matched | 至少一个位置或联合片段可靠映射到指定参考锚点 |
| verified_absent | 在满足 C8 的完整可核验范围内，确认指定来源锚点未保留 |
| ambiguous | 存在多个无法排除的参考位置、目标位置或阶段实例解释 |
| unresolved | 当前证据、可见性或方法不能确认命中或缺失 |
| not_applicable | 当前文本 profile 不适用于该 locator 或目标类型 |

matched 不要求枚举所有可能替代来源，
但必须说明实际确认的位置和检查范围。

partial 目标中确认的实际出现可以是 matched；
它只支持已观察到的出现，不能证明其余内容缺失。

部分片段命中但不能覆盖完整锚点时，
状态为 unresolved，并保留 partial_anchor_coverage。

结果不包含任意构造的置信度概率。
方法类别和核验条件不能伪装成已校准置信度。

### C8. verified_absent 的严格条件

verified_absent 针对指定来源锚点和指定目标范围，
不表示语义事实在全库或所有可能表达中不存在。

必须全部满足：
1. 参考锚点合法且已唯一确定。
2. 来源事实存在对应的有效审定及范围记录。
3. 目标集合和内容完整、原始且可读取。
4. 生成上下文的实际请求绑定已经核验。
5. 本次选定范围内的搜索完整结束。
6. 没有未检查的外部通道、未知内容或来源身份缺口。
7. 目标为空，或全部目标内容都能核验为固定参考表示的字面片段。
8. 指定来源锚点既未单片段命中，也未被联合覆盖。

第 7 项用于限制否定结论：
存在无法解释的改写、压缩或新表达时，
搜索无命中只能得到 unresolved。

source_reviewed 等状态从 Registry 的固定审定结果读取。
不能根据 quote 存在或 independent_reference=true 自行补造。

任一条件不满足时：
- 原因明确且存在竞争解释：ambiguous。
- 其他情况：unresolved。

相同事实的新来源仍可能属于 unjudged。
本条否定结论不能直接使 fact_id 的覆盖变成 0。
Evaluator 还需处理已认可替代来源、
未知映射及未标注来源。

没有检索结果不表示用例 unanswerable。
候选中缺失不证明索引或语料中缺失。
最终上下文中缺失不证明检索阶段缺失。

### C9. 公开接口与结果批次

公开入口：

```text
create_text_mapper(
    artifact_store,
    dataset_registry,
    adapters
) -> EvidenceMapper

map_evidence(
    request_ref,
    access_context
) -> MappingBatchResult

load_mappings(
    batch_result,
    access_context
) -> MappingBatchReader
```

request_ref 指向固定 MappingRequest。
消费者使用提供方公开契约，
不访问私有文件、缓存或工作目录。

MappingBatchResult 包含：
- batch_ref：固定批次清单。
- receipt_ref：本次处理回执。

清单固定：
- request_ref。
- 实际解析的目标范围。
- 所有依赖内容引用。
- config_ref、实现与 normalizer 版本。
- 按 EvidenceKey 排序的结果引用。
- 请求数量、结果数量和状态计数。
- 空请求时的 no_selected_evidence。
- batch_id。

batch_id 为清单排除自身 batch_id 后的规范摘要。

每个选定 EvidenceKey 必须有一个结果，
或者整个操作明确失败。
不能跳过难以匹配的证据后返回完整成功。

回执记录操作主体、UTC 时间、批次引用、
访问策略 revision 和处理完成状态。
操作信息不影响映射内容身份。

MappingBatchReader 提供完整清单及结果读取，
不把状态转换为分数，不选择充分组。

### C10. 错误契约

| reason_code | 含义 |
|---|---|
| evidence-mapper.invalid_input | 参数、格式或配置非法 |
| evidence-mapper.unsupported_schema | 不支持的请求或配置版本 |
| evidence-mapper.reference_invalid | 证据、版本、实例或关联引用不一致 |
| evidence-mapper.reference_locator_invalid | 参考范围或 quote 与固定文本矛盾 |
| evidence-mapper.target_inconsistent | 目标内容、调用或字符绑定矛盾 |
| evidence-mapper.limit_exceeded | 处理大小或匹配数量超限 |
| evidence-mapper.access_denied | 当前读取或保存未授权 |
| evidence-mapper.integrity_mismatch | 内容、结果数量或批次校验不一致 |
| evidence-mapper.storage_failure | Store 失败，保留原始原因 |
| evidence-mapper.internal_error | 映射程序故障 |

业务不确定原因保存在 EvidenceMapping.reason_code，
包括：
- stage_not_recorded。
- stage_instance_ambiguous。
- reference_anchor_ambiguous。
- source_identity_unknown。
- source_revision_mismatch。
- target_incomplete。
- target_fidelity_limited。
- request_binding_unverified。
- partial_anchor_coverage。
- paraphrase_not_supported。
- reference_review_missing。
- unsupported_locator。
- unsupported_target_channel。

输入矛盾、读取失败、程序异常和资源超限
不能转成 verified_absent。
这些错误属于接入或测量失败，
不构成被测系统质量失败。

## 2. 判定与处理流程

### 2.1 处理顺序

```text
核验权限、请求、正式数据版本和 Run 引用
→ 解析明确阶段或生成调用
→ 核验目标原生产物与请求绑定
→ 读取固定证据、源资产及参考表示
→ 确定唯一参考锚点
→ 核对来源身份并寻找原始精确匹配
→ 按配置执行保守空白归一化
→ 检查单片段与同源联合覆盖
→ 检查否定结论所需的完整性和方法条件
→ 生成位置、状态、依据及限制
→ 保存完整结果集合和批次清单
→ 保存回执并返回
```

不读取其他实验的相似答案或阶段产物补齐当前请求。

### 2.2 判定优先级

```text
输入或依赖非法：
    操作失败

locator 或目标通道不适用：
    not_applicable

阶段实例或参考锚点无法消歧：
    ambiguous

缺少目标、实际请求或必要可见性：
    unresolved

存在满足条件的完整锚点映射：
    matched

所有严格缺失条件满足：
    verified_absent

存在竞争解释：
    ambiguous

其他：
    unresolved
```

不因某一片段匹配失败就提前认定整体缺失。
需要检查所选范围内其他片段及合法联合覆盖。

多次出现同一已确定来源锚点时保留所有已核验位置，
不会将重复来源转换成多个必要事实。

### 2.3 正向与否定结论的不同要求

正向定位只需可靠证明一个实际出现。
否定结论需要范围完整、搜索完成及表达方式可核验。

因此：
- 不完整候选中的一个可靠命中可以确认。
- 不完整候选中的零命中不能确认缺失。
- 无法处理的改写不作为漏检证明。
- 仅在其他实验中命中不能补成当前实验命中。

## 3. 持久化、缓存与恢复

所有请求、配置、结果、位置依据及清单通过 Store 保存。
不修改 EvidenceUnit 或 RunRecord。

提交顺序：
1. 保存已验证的单条映射。
2. 保存完整结果集合。
3. 核验引用、数量和状态计数。
4. 最后提交批次清单。
5. 保存本次操作回执。

单条结果存在但批次清单未提交，
不代表完整映射批次成功。

批次清单已提交、回执失败时明确报告阶段。
正常重试复用相同批次，生成独立回执。

请求及批次内容身份可作为精确复用键，
必须覆盖：
- Store 与 access_domain。
- 数据集、快照、证据和参考表示内容。
- Run、阶段或生成调用。
- 目标内容、顺序及 rank 限制。
- 捕获 fidelity、completeness 和绑定记录。
- 审定状态、范围及依赖。
- 配置、实现、normalizer 和处理限额。

不能只按 quote、query、chunk ID 或 fact_id 缓存。

读取缓存前仍检查当前权限及依赖可用性。
缓存不能绕过撤权，也不能掩盖已发现的损坏。

并发提交由 Store 的不可变写入协调。
本模块不追加共享 JSONL 或维护全局可变结果表。

重新映射不调用被测系统。
修改 mapper 或 normalizer 产生新映射结果，
旧结果继续保留。

本轮不自动应用 Review 记录改写映射状态。
需要人工结论时，由后续评价或复核协议
固定其对象、内容视图及规则，不覆盖本次机械定位结果。

## 4. 安全与实现安排

允许的业务依赖：
- Artifact Store / Contracts r002。
- Dataset Registry / Contracts r002。
- Adapters / Contracts r001。

不依赖 Runner、Review、Evaluator 或 Reporting。
数据和运行内容只通过提供方公开接口读取。

StoreAccessContext 控制测评进程的读取权限。
用例业务 access_scope 不授予 Store 权限，
也不由 Mapper 重新解释为业务授权结果。

目标中出现无权业务内容是否构成违规，
由相应 Evaluator 判断；
Mapper 的命中不代表业务授权通过。

外部文本作为数据处理。
本轮无模型、网络、工具调用或自动资产下载。
默认日志只输出安全身份、阶段、状态和原因。

计划入口：
- rag_eval.evidence_mapper.contracts。
- rag_eval.evidence_mapper.api。

内部职责可分为：
- 请求与依赖解析。
- 参考锚点及来源核对。
- 归一化与坐标恢复。
- 单片段、区间联合覆盖和严格缺失检查。
- 不可变批次保存与读取。

私有类、索引结构及搜索算法由实现决定。
实现必须遵守固定匹配语义和限额，
不能因性能优化漏掉候选后仍宣称完整搜索。

本轮不承诺吞吐或时延目标。
尚未执行性能基准。

## 5. 验证计划

以下为计划验收，尚未执行：

| 场景 | 应观察到的结果 |
|---|---|
| 相同源文 revision、唯一 quote 精确出现 | matched，位置可回查 |
| 相同内容来自错误 revision | 不冒充原黄金来源 |
| 只有文本相同但没有来源身份 | 诊断候选，不自动确认可靠映射 |
| 原文多个相同 quote，无可用消歧 | ambiguous |
| 明确锚点在目标中重复出现 | 保留多个位置，仍为 matched |
| CRLF、TAB 和连续空格差异 | 仅按配置归一化并恢复原位置 |
| 1 000 与 1000 | 不因删除空白错误命中 |
| 年份、负号、单位或否定不同 | 不自动判等 |
| 中文及补充平面字符 | 使用 code point 范围，位置无偏移 |
| 相邻 chunk 各保留锚点一部分 | 合法同源区间可 joint_literal |
| 两个片段中间缺少关键字符 | 不从源文补齐，不确认完整覆盖 |
| 两片段来自不同文档或 revision | 不联合为同一来源锚点 |
| 候选中命中、最终上下文被截断 | 两个范围输出不同结果 |
| 不完整候选中存在可靠命中 | 确认该出现并保留范围限制 |
| 不完整候选没有命中 | unresolved |
| 原始完整空上下文，严格条件齐全 | 指定范围 verified_absent |
| 目标存在无法解释的语义改写 | unresolved，不记真实缺失 |
| 没有 source_reviewed 依据 | 不输出已核验缺失结论 |
| 有上下文但缺少实际请求绑定 | unresolved |
| 多个生成调用但未指定实例 | ambiguous，不自动取最后一次 |
| 外部图片 locator | not_applicable，不虚构视觉匹配 |
| 新替代来源未标注 | 保留 unjudged，不改写 gold |
| quote 命中但条件或推导尚未核验 | support_assessment=not_assessed |
| 超过匹配或处理限额 | limit_exceeded，不返回缺失 |
| Store 内容损坏或读取拒绝 | 明确错误，不补空内容 |
| 相同固定输入重复处理 | 相同批次，独立回执 |
| normalizer、顺序或审定范围变化 | 形成不同结果身份 |
| 清单提交前中断 | 不产生成功完整批次 |
| 清单提交后回执失败 | 重试复用批次 |
| 重新映射 | 不增加业务系统调用 |

验证使用真实本地 Store、固定小型参考文本、
明确可解释的候选及生成请求。

测试同时检查位置与状态，
不能只检查是否找到相同字符串。
不需要生产材料、模型服务或数据库。

## 6. 设计进度与后续边界

进度摘要：Confirmed / r002（包 r020）；业务单元修订仍按下表记录。

| 设计单元 | 审阅状态 | 内容修订 | 位置 |
|---|---|---|---|
| 概述与边界 | Confirmed | r001 | [第 1 节](#1-职责与边界) |
| 请求、结果与公开契约 | Confirmed | r001 | [Contracts](#contracts) |
| 判定与处理流程 | Confirmed | r001 | [第 2 节](#2-判定与处理流程) |
| 持久化、缓存与恢复 | Confirmed | r001 | [第 3 节](#3-持久化缓存与恢复) |
| 安全与实现安排 | Confirmed | r001 | [第 4 节](#4-安全与实现安排) |
| 验证计划 | Confirmed | r001 | [第 5 节](#5-验证计划) |

Complete 仅覆盖本文件声明的 C2 文本范围。
应用实现与验收尚未执行。
C2 的当前就绪状态见[索引](../../index.md#5-候选推进范围)，
确切实现基线见[交接 C2-H001](../../handoffs/c2-r001.md)。

语义改写、视觉映射、全量 parser/index 产物、
人工映射仲裁应用和高级模糊匹配继续留在后续范围。

公开接口、匹配语义、坐标、权限、持久化、
状态和验收行为变化需要确认。
私有实现可在已确认边界内决定。
