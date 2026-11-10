# Evidence Mapper：诊断文本阶段范围扩展

- 状态：Confirmed；文档、Contracts 及详设修订：r001；覆盖 Complete。
- profile=evidence-mapper.diagnostic.v1；核心 evidence-mapper.c2.v1/r001 不变。
- 输入仅为 Adapter 已导入的固定文本阶段观察；不新增 parser 或索引引擎。

## Contracts

### M1. 请求与范围

DiagnosticMappingRequest 包含核心 MappingRequest 的
dataset_version_ref、case_key、run_ref、evidence_keys、config_ref，
以及 diagnostic_observation_ref、scope_id、scope_selection。
scope_selection=complete_declared_scope 或 explicit_item_ids（有序无重复列表）。

引用 [Adapter A4](../adapters/diagnostic-extension.md#contracts) 的唯一真实观察。
公开操作 map_diagnostic_scope(request_ref, access_context) -> MappingBatchResult，
由 create_text_mapper 返回的扩展实现提供；既有 map_evidence 不接收新 target_selector。
输出 EvidenceMapping 使用原字段与状态，contract_profile 为扩展 profile；
resolved_target_scope 增加 observation_ref、scope_id、实际 items 和构建关联状态。
身份、持久化、错误和坐标恢复完整沿用核心，不复制其算法。

### M2. 能力和否定条件

只消费 adapters-diagnostic-text-scope-v1 的严格 UTF-8 文本，
先核对数据快照、SourceArtifact 谱系、文档 revision、捕获完整性、
预期 item 数量、生产者的全量声明依据与 Run 的 membership。
explicit_item_ids 只能对该子集出结论，不外推全量。

matched 表示指定锚点在观察的真实位置保留，
不表示语义充分，也不证明该构建被当前 Run 使用。
verified_absent 除核心 C8 的所有条件外还要求：
- full_document_output 或清楚限定且可核验的 declared_inventory。
- 完整处理所声明范围，集合和内容皆 original、complete。
- 参考源资产与目标生产链一致，锚点已审定且唯一。
- 全部内容为可解释的字面片段，无未知/OCR改写/语义压缩/外部通道。
- 对“当前 Run 的阶段缺失”还要求实际构建 membership 已核验；
  否则只能声明该独立组件输出的锚点缺失。
这里没有生成请求，因此核心 request_binding 条件为
not_applicable / non_generation_scope，有明确目标依据；不得伪造 true。

源证据来自独立参考不意味着 parser 输出必定逐字一致。
OCR 改写、公式、表格重组、未能唯一映回固定参考文本时 unresolved，
即使简单搜索没有命中，也不认定“事实解析丢失”。
本扩展仍不做视觉、向量索引可达性或语义支持判断。

### M3. 验证与实现

依赖为核心已有 Store/Registry/Adapter，以及 Adapter 扩展 A4；
不新增其他模块方向。只从 contracts/api 读取观察。
核心与扩展分别声明 profile 支持，无扩展实现时明确 capability_unavailable，
不能用 C2 map_evidence 假装检查过 parser 全量。

计划验证：全量字面 parser 有/无锚点、缺一页、仅 top-K、
未绑定构建、来源 revision 错误、OCR 同义改写、集合计数不闭合、
局部正向命中与严格否定条件区别、缓存跨域撤权。
验收案例见 [C3 切片](../../slices/c3-diagnostics.md) T06–T09。
当前只有设计，没有已执行定位或 parser 质量证据。
