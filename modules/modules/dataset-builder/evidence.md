# 证据构建：源文件、参考表示与事实

- 状态：Confirmed；内容修订：r001；范围：Dataset Builder C1。
- 确认依据：用户授权代理连续完成；验证计划尚未执行。
- 上级契约：[README / C4、C9](README.md#contracts)。
- 正式数据语义只引用 [Registry C2–C3](../dataset-registry/README.md#contracts)。

## 1. 输入冻结与来源闭合

SourceManifest 包含 manifest_id、source_policy_ref、documents。
每个 SourceDocument 记录 document_id、document_revision、original_source_key、
content_ref、sha256、media_type、language、family_id、family_basis_ref、
published_at、valid_from/valid_to、business_tags、access_domain。
不知道的期间/文档族为 null，并记 missing_basis；不猜测当前年份或独立家族。
同 document_id 的多个版本必须显式选入不同 snapshot，不能任意取最新。

原文件先经 Store.put_blob/register_source 保存；
Builder 接受固定引用，不扫描输入内容中的本地路径或 URL。
SourcePolicy 规定本次 snapshot 的选入范围、业务角色可见性、
提取/模型出域/审定权限、保留策略及动态依赖限制。
snapshot 的 access_policy_ref 是固定业务条件，不能覆盖当前存储授权。

冻结顺序：完整枚举 → 逐项授权/摘要 → 显式选择 revision →
固定清单与来源保证 → 提交 source bundle。
缺失文件、坏摘要、重复冲突身份导致整个冻结失败；
文档可读但不能提取则保留到提取处置表，不能从源清单删除。

输出 SourceBundle={source_manifest_ref, selected_documents_ref,
snapshot_projection_ref, source_count, limitations}。
snapshot_projection 是待交 Registry 校验的对象，不自称正式登记。

## 2. 提取协议和具体接入

ExtractionRequest={request_id, source_key, source_ref, media_type,
extraction_profile_ref, rendition_policy_ref, allowed_scope, limits}。
CallControl 包含 task/attempt 身份、取消/deadline、受信访问上下文和资源预留，
不从文件内容接收授权声明。
ExtractionCapabilities 明确 media_types、原始视觉输入、定位坐标、
批处理、支持的取消/计量、producer/version 和限制。

ExtractionResult={request_ref, producer, status, covered_units,
failed_units, structures_ref, assets_ref, observations_ref, diagnostics_ref,
usage_ref, source_binding}；status=succeeded/partial/failed。
covered 与 failed 必须覆盖请求枚举的全部页/节点；
总页数未知时标 coverage_unverified，不能声称全页成功。
failed 是没有可用观测；partial 表示存在明确缺口；
I/O、拒权和响应损坏是操作错误，不转成普通 failed 文档。

初始具体实现：
- utf8_text_v1：直接接收严格 UTF-8 文本，保留原文换行与码点；
  只派生固定标题/段落边界，不改原文作为“清洗后真值”。
- reference_extraction_jsonl_v1：导入 PDF/DOCX/PPT/表格/OCR/视觉
  工具的固定提取结果，显式 source/rendition 绑定、页/shape/单元格和失败清单。
  逐行限额、坏行整批失败；不调用外部 parser，不冒称完成其准确性验证。
- 后续 native parser/OCR/视觉客户端实现同一端口；各自需媒体类型能力与
  固定版本、字体/渲染环境、许可及目标样例契约验收，不能只注册名字即宣称支持。

参考导入协议每行：
source_key、unit_id、parent_unit_id、unit_kind、page_or_slide、
raw_text、reading_order、asset_refs、table_ref、locator_ref、
extraction_status、producer、uncertainties。
unit_kind=document/page/section/paragraph/table/cell/image/chart/note。
同 source+unit_id 唯一；顺序稳定；必要父级存在且无环。
完整批次 manifest 列文档数量、每文档页/节点集合、失败及未核验范围。

引用图像均为 Store 字节；禁止模型 URL 回读未授权网页。
内置文本/导入路径足以验证 C1 核心流程；
复杂原格式的自动解析属于具体接入能力，未提供时明确 capability_missing。

## 3. ReferenceBundle 与定位坐标

ReferenceBundle={key, source_manifest_ref, document_units_ref,
canonical_representations_ref, rendition_manifest_ref, diagnostics_ref,
coverage_ref, provenance_ref}。

每个表示固定 representation_id/revision、source_key、producer、
kind（canonical_text/external）、content_ref、source_mapping_ref、
independence_declaration 与 uncertainties。
canonical_text 使用原始字符或明确版本化的拼接规则；
拼接标题/页分隔符时给出每个全局区间到源节点/页的映射。
不能把归一化检索文本写回 quote。

Anchor={key, representation_ref, source_key, included_ranges,
structural_context_refs, required_context_refs, extraction_coverage,
uncertainties}。
range 为同一固定表示的 code point 半开区间；非连续区间分别保存。
prefix/suffix 仅用于消歧，不吞并缺失表头/否定条件。

同一 source 的新提取器输出是新表示；已确认 reference 不因 parser 更新自动替换。
manual correction 请求引用 parent、旧/新范围、理由和操作主体，
产生新 reference revision，并计算所有依赖失效集合。
操作身份不等于人工核心审定，正式来源正确性仍经 Registry 目标审查。

## 4. 事实、数量及语义身份

FactCandidate.payload：
subject_ref、predicate、object（entity/text/quantity 互斥类型）、
statement、qualifiers、source_assertion、support_anchor_refs、
source_spans、identity_resolution_ref、status、uncertainties。
qualifiers 与 Registry C3 的实体/期间/条件/assertion_kind/polarity 一致。
数量额外记录 raw_literal、normalized_decimal、unit、
precision_kind（exact/disclosed_rounded/estimated）、bounds 和 conversion_ref。
“—”、空白、未披露、缺测、not_applicable、零分别编码，不能统一为零。

事实抽取先规则后可选模型建议；模型结果必须回指原锚点。
所有事实的状态为 candidate/auto_checked/needs_review/rejected，
不直接产出 source_reviewed。
文本命中只证明位置；语义支持需来源检查。
语义合并需 qualifiers 与规范 statement 一致并有 resolution 依据；
没有明确同义依据时保持两个 fact_id，用关系记录可能等价。

自动实体链接候选不据 embedding 分数直接合并。
年份、单位、地区、计划/实际、否定、条件任一不同均分开。
同事实的多份来源允许共享 fact_id，但来源身份、locator 和核验记录各自保留。

RelationCandidate={key, relation_type, endpoints, direction, qualifiers,
source_refs, validation_ref, status}。
relation_type 为 same_entity、same_metric_different_period、plan_vs_actual、
definition_applies_to、references、prerequisite、alternative_support、
contradicts、supersedes。
状态 proposed/validated/needs_review/rejected；
相似检索只产生 proposed，不能直接生成必要推理边。
不同来源冲突优先检查有效期/口径，无法裁决保留 conflicting。

## 5. 表格、图表与扫描件

表格记录 table_id、source/page、完整行列头路径、merged_cells、
单位/脚注作用范围、续表关系、每格 raw/value/missing_kind、formula_ref。
不依赖待测系统的行列编号；表格重排产生新表示映射。
公式和值分别保存，没有固定计算环境时不擅自执行公式。

视觉 Descriptor 使用 builder.visual-locator.v1：
source_key、rendition_source_key、asset_ref、page_or_slide、
coordinate_space（pixel 或 normalized）、width/height、rotation、
bbox（可空）、region_role、provenance_ref、verification_scope。
bbox 为 [x0,y0,x1,y1]，必须在对应尺寸内且严格正面积；
裁剪记录 crop_to_page_transform，坐标不能跨渲染版本复用。
无可靠区域时只存 asset/page，不能补造框。
descriptor 经 Registry external locator 引用，不扩展其核心字段。

图表观测包含 axes、scale（含 log/truncated）、legend、series、
period、labels、units、direct_values、estimated_intervals、footnotes。
值、实体、期间均回指图上位置；caption 是解释通道，不替代图像。
OCR/视觉读数冲突保留 uncertainty，不投票制造核心事实。
小图、图例、公式和箭头按内容作用判断，不用统一像素阈值删除。

三层分开：原始资产 → 直接观测 → 类型化推导；业务解释标 interpreted。
source_modality 与 vision_dependency_assessment 分开：
图表来源的事实存在合法文本替代时，不强称必须视觉理解。
扫描不可读可以成为压力输入，gold 仍需另一独立可确认依据；
原文和所有备份都无法核验时不得发布确定答案核心题。

## 6. 正式对象投影及失败处理

EvidenceUnit 使用 Registry DTO：
snapshot_key、source document/revision、fact_id、statement、qualifiers、
modality、locator；extraction 来源放入现有 provenance/源资产谱系，
不能塞入未声明 verification_status 字段。
text_quote 必须在固定 canonical_text 中核验；
非连续事实用多个 EvidenceUnit/支持路径，不拼伪连续 quote。
external 描述符会限制 C2 文本定位指标，能力差异进入兼容报告。

候选参考结构和分组依据先留 Builder，只有 Registry 接收后的
正式 EvidenceUnit 是发布权威。raw_source vs reference 的独立性
保留声明和审定，不由 independent_reference=true 自动认证。

提取严重失败隔离相关锚点，不让其他页替它取得通过状态。
适用集合可在显式选题策略下限制可用页，但构建摘要必须显示全源覆盖缺口。
若请求的是全源完整覆盖，partial 阻塞目标，不能静默改成局部完成。

## 7. 验证计划

| 场景 | 必须观察到的结果 |
|---|---|
| 相同内容跨授权域 | 逐域授权，无缓存绕过 |
| PDF 导入少一页但声称完整 | 覆盖不一致，整批错误或明确 partial |
| canonical text 含 CRLF、中文和组合字符 | 定位使用真实码点，quote 不被归一化改写 |
| 同数值但年度/单位/否定不同 | 不能合并 fact_id |
| 单元格未披露与 0 | 保持不同状态 |
| 表头在前页 | 锚点依赖带入前页表头，不裸读数值 |
| 图例很小、bbox 无证据 | 保留图例；只发布 asset/page 定位 |
| OCR 与图像值冲突 | needs_review，不自动 source_reviewed |
| 更新 parser | 原 gold 仍绑定原表示，新版本需依赖重审 |
| import 含路径/脚本/远程图片 | 作为数据拒绝或保留，不执行 |
| visual external 进入 C2 文本 Mapper | 明确不可测，不生成伪定位分 |

使用真实 Store、小型原始文本及冻结提取夹具。
没有原 PDF/OCR 服务时只验证导入契约，不能宣称完成对应 parser 的真实集成。
