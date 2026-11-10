# Reporting：诊断与干预报告扩展

- 状态：Confirmed；文档、Contracts 及详设修订：r001；覆盖 Complete。
- profile=reporting.diagnostic.v1；C2 reporting 契约 r002 不变。
- 本扩展消费 Diagnostics 已提交结果，不能以报告逻辑创造新根因。

## Contracts

### P1. 请求与接口

create_diagnostic_reporting(config, artifact_store, dataset_registry, runner,
                            evaluators, diagnostics, local_export_policy=None)
返回 DiagnosticReporting；不增加依赖方向。
DiagnosticReportRequest：
finding_batch_refs、plan_batch_refs、assessment_batch_refs、
runner_snapshot_refs、optional_c2_analysis_ref、report_policy_ref、
display_view_refs、provenance_ref。
所有输入显式固定，多个不兼容数据/规则/评分 baseline 分开报告，
不通过字段同名混合。只报告已提供完整来源，不暗示全仓库汇总。

公开操作：
render_diagnostic_report(request_ref, access_context) -> DiagnosticReportBundleRef；
load_diagnostic_report(bundle_ref, access_context) -> DiagnosticReportReader；
export_diagnostic_bundle(bundle_ref, output_root, access_context) -> ExportReceipt。

bundle、refs、清单/回执模式沿用 C2 不可变产物规则，
contract_profile=reporting.diagnostic.v1，固定文件为：
diagnostic-report.md、diagnostic-report.json、findings.jsonl、
selection.jsonl、pairs.jsonl、case-summaries.jsonl、limitations.json。
完整明细不受展示样例配额裁剪；超输出限额明确报错，不删未知。
标准 metadata-only 展示和 explicit_display_views 沿用 C2 C17，
导出路径/权限/暂存与 manifest 最后提交沿用 C2 C18。
导出 manifest 使用扩展 bundle schema，不强行套 C2 file role 枚举。

### P2. 报告语义

同一语义树生成 Markdown/JSON，至少显示：
- 数据/Run/规则/mapper/Evaluator/模型/访问及来源保证版本。
- 全部症状、证据、支持/反证/缺口、竞争假设和 next actions。
- 具体 scope/阶段链，不把锚点缺失画成完整事实缺失。
- 选择母集/渠道/名额/重叠、eligible 与未选原因和预算缺口。
- original_comparison 与 paired_repeat 分开。
- strict_valid/non_strict/unverified 分开，真实与 fixture/external 来源分开。
- 全部四转移、unknown 原因、计划/有效 pair 分母、
  每题重复分布、跨 query 描述量及重复覆盖。
- context 长度/token/顺序/噪声的联动变化，已知/未知成本和未决调用。
- 复核建议及结论范围，不使用“100% 根因”或未经校准置信度。

具体公式按 [Diagnostics interpretation r001](../diagnostics/interpretation.md)；
Reporting 实现相同明细导出的可复核描述汇总。
方差仅 R>1 可测时按明确范围计算；不是 CI。
首版不提供干预恢复率 CI、总体根因比例或自动发布 gate。
optional_c2_analysis_ref 只能附原 C2 质量/门禁引用，
标明其范围，不因 Oracle pass 修改普通系统基线或原 GateDecision。
普通运行成本、干预成本、评分成本分账；重叠 scope 不直接相加，
未知调用不可按零费用统计。

缺报告源材料、撤权、损坏、版本冲突和导出失败
沿用 reporting 公开错误维度，新增 diagnostic_binding_mismatch。
协议无效但结果合法属于正文限制，不是渲染错误。
默认日志无 query/gold；不自动抓取远程资源或导出全部源文。

## 1. 验证与接缝

输入使用 Runner、Evaluator、Diagnostics 的真实公开 reader；
测试匹配每个 case/pair、精确分子分母、unknown 及模型漂移排除。
手算采用 interpretation 第 5 节，核验 1/2 的受限恢复分数和 0.25 的
带缺口 query 描述差，不能写成根因比例。
检查小数显示舍入不改明细、重复 query 不冒充新题、
selection 重叠不重复计数、类别多选不归一化及 fixture 标识。
Markdown/JSON 保持同源，原 C2 gate 和 ordinary 报告行为保持原 profile。
当前没有执行过应用测试或真实统计验收。
