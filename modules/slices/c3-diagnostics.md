# C3：诊断、严格文本 Oracle 与可审计报告

- 状态：Confirmed；文档/业务/验收修订：r001；覆盖 Complete。
- 候选：C3；Ready for OpenSpec；包 r022；交接 [C3-H001](../handoffs/c3-r001.md)。
- 2026-11-10 用户授权连续设计及写入；没有逐节人工复审。
- 实施顺序仍优先 C2，再 C1/真实数据基础，再 C3；本次不编写应用或 OpenSpec。

## 1. 范围与依赖处置

结果：从固定 C2 离线评分得到有证据/反证/缺口的诊断；
规划干预；具备条件时通过严格文本生成接口执行新配对；
同口径离线评分并解释效果，输出完整 JSON/Markdown 诊断报告。

Implement：
Diagnostics README C1–C10 与四主题 r001；
Adapter diagnostic-extension A1–A5/r001，
Mapper diagnostic-stages M1–M3/r001，
Runner diagnostic-experiments X1–X3、第 1–4 节/r001，
Reporting diagnostic-reporting P1–P2、第 1 节/r001；
本切片 CLI/集成/T01–T32。
首个在线接入仅 strict_text_http_v1；recorded_generation_v1 仅协议夹具。
新增扩展 profile 不改变 C1/C2 核心，不为 C2 隐式联网。

Reuse/Implement 依赖：
没有已验证应用代码。若 C1/C2 已实施，先读对应代码、契约与真实测试，
兼容则复用；否则按确切契约实施所需本地 Store/Registry/Review、
Adapter import、文本 Mapper、Evaluator、Planner、Runner 评分和
Reporting 导出/显示能力。C3 不包含 C1 Builder 生成。
实际 Run、Mapping、Judgment 通过各自公开 reader，不能直接解析私有目录。

Defer：
多模态/工具 Oracle、任意 stage replay、自动裁判/主张提取/校准、
诊断 LLM/概率、自动修复及改 gold、Web/队列/分布式、干预推断 CI/gate。
其他干预已定义规划/审计边界，但在线执行明确 unsupported。
真实模型 endpoint、不可变版本/默认/容量证据、权限/出域、预算及新评分来源
是运行输入；缺少则 blocked/unknown，不阻塞引擎设计。

## 2. CLI 契约

两种进程入口沿用 C2：rag-eval 与 python -m rag_eval，
同配置优先级、严格 UTF-8 JSON、单 JSON stdout/安全进度 stderr。
业务调用使用 --input <path>；引用参数输入文件只包含明确 ArtifactRef/对象键，
由可信入口导入 Store 并校验，不接受任意脚本或动态模块路径。

| 命令 | 输入与实际操作 | 返回主要引用 |
|---|---|---|
| diagnostic-stage import | Adapter A4 完整导入请求 | ObservationImportResult |
| diagnostic-map run | Mapper M1 请求 | MappingBatchResult |
| diagnose run | DiagnosticRequest；离线确定性诊断 | finding batch |
| diagnose plan | finding batch、固定 policy/spec/capability | selection/eligibility/plan batch |
| diagnostic-experiment prepare | Runner X1 请求；不产生网络调用 | ready/blocked manifest |
| diagnostic-experiment execute | 明确 prepared manifest；启动新真实调用 | job key/snapshot |
| diagnostic-job show | 安全 job key | 固定状态/未完成范围 |
| diagnostic-job resume | 原 manifest/checkpoint，不能改条件 | 新 snapshot |
| diagnostic-job cancel | job key/取消原因 | 请求与实际停止状态 |
| diagnostic-job reconcile | 已知 invocation/receipt，必要明确决策 | reconciled/unknown snapshot |
| diagnostic-score prepare | job/scoring template/完整审定来源 | 按 plan 的 C2 PreparationResult 完整批次 |
| diagnose evidence-export | 完整 job/评分快照到 DTO | TrialEvidenceEnvelopeRef |
| diagnose assess | 完整 envelope | assessment batch |
| diagnostic-report render | Reporting P1 请求 | DiagnosticReportBundleRef |
| diagnostic-report export | 显式 bundle 引用与 --output | ExportReceipt |

prepare、execute、diagnostic-score prepare 必需 --operation-key，按 Runner 的冻结键约束去重；
中断后使用返回的 job/operation handle，不重建作业绕过配额。
准备评分后，对每个 PreparationResult 使用原 C2 run / evaluate 和 job resume
完成评价；不发业务请求。
冻结 spec 是 diagnose plan 前 CLI 对 Adapter.freeze_generation 的本地组合步骤，
结果保存固定引用，不能临时读取当前模型默认。
外部试验先用 C2 runs import 导入，再显式构造 envelope，
无真实协议材料时评为 unverified。

沿用 CommandResult、Error 及退出码：
0=本次命令要求的完整产物已提交；2=输入/前置 blocked；
3=执行/基础设施故障或 partial/outcome_unknown，130=用户中断。
CommandResult 成功时不伪造 gate=pass；本切片不请求新 gate。
诊断中 unknown、有效 TrialAssessment non_strict、全部 ineligible
可作为完整分析产物返回 0，并完整显示语义状态；
实验 prepare blocked 返回 2，execute 的未完成为 3 并附可恢复 job_key。
C2 独立门禁命令仍按原约定返回 4/5，不被新命令改写。
不得只靠退出 0 或页面出现“恢复”判断系统质量。

## 3. 装配与调用顺序

CLI 是组合根，依次构造配置/当前授权、Store、Registry、Review、
Adapters/Mapper/Evaluator/Planner、C2 Runner、Diagnostics、
可选 DiagnosticRunner 和 DiagnosticReporting。
离线 diagnose 不构造 HTTP 客户端或要求凭据。
Runner→Diagnostics、Reporting→Diagnostics 方向不反转。

S0 固定数据、原始母集、C2 实验及评分来源；无生产数据可用明确合成夹具。
S1 导入可用独立阶段观察；缺 parser/index 输出允许继续，保留 unknown。
S2 对实际阶段/调用做映射，C2 和诊断文本扩展分开 profile。
S3 读取 C2 全预期槽位/评分，投影 DiagnosticRequest，生成全部 Finding。
S4 冻结抽样/预算、spec/capabilities、充分组和权限，生成完整 selection/资格/计划。
S5 准备执行清单；任何不满足严格条件的计划不派发，不自动换模型/模板。
S6 明确 execute 才生成新 baseline/treatment，及时提交真实请求/响应/usage。
S7 将全部调用槽位（含 missing/unknown）投影新 C2 trace_audit；
同数据/任务/rubric/评分包评价；没有新审定则相关质量 unknown。
S8 导出完整 TrialEvidenceEnvelope；Diagnostics 独立核验协议并解释每对效果。
S9 Reporting 生成全明细/描述量/限制；可显式导出受控报告。
S10 建议复核通过真实对象所有者新 revision 处理，不修改旧记录或隐式重跑。

跨模块没有原子事务；各 owner 的不可变产物独立提交，
最后批次/作业/报告清单定义整体完成范围。
评分或报告失败恢复 S7–S9，不重复 S6。
网络请求结果未决保持 outcome_unknown，需要可证明 reconciliation 或明确新尝试决策。

## 4. 端到端固定夹具

本地文本源：D@1 有 f1“2024 营收 100 万元”、f2“2025 营收 120 万元”；
问题差额为 20 万元，identity/add/subtract 中采用 subtract；
源事实和充分组 S1 审定，允许规则/格式固定。
保留实际 quote、请求区间、两步必要前提与明确 expected_action。

阶段变化夹具：
R1 含 f1/f2，R2 含 f1/f2，C3/C1 只有 f1；所有 Run/调用/链路 ID 明确。
变体为缺 R2、未知来源、OCR 改写、S2 合法替代来源、错年数字、
多次生成的另一调用有 f2、完整/不完整 parser 和独立构建未绑定。
这些变体区分锚点、事实、充分、运行关联及范围。

试验夹具用两个原始失败 query、各两个 pair slot，
四转移及 unknown 的手算以 Diagnostics interpretation 第 5 节为准。
recorded_fixture 只检查闭环协议；报告显著标识，无真实干预结论。
受控 HTTP 服务器核对实际 payload/调用次数/AB顺序，不模拟真实人工准确率。
真实 provider 与真实评分必须另行提供并验证，不用夹具冒充生产能力。

## 5. 验收证据要求

每项记录 T 编号、实现/依赖/Windows/NTFS 版本、固定输入与策略、
命令和退出码、实际发送次数、产物/摘要/完整数量、断言及故障点。
核心路径使用真实本地提供方；不能用所有模块 mock 代替。
执行数量/网络/撤权/故障恢复必须观察实际副作用，
仅检查 plan 或截图不足。

依赖版本在实施中锁定并验证 Windows 安装、TLS/HTTP 行为、
code point 坐标、文件锁/Job Object/outbox 和目录提交。
当前测试数量为零，下面全部为计划案例。

## 6. T01–T32 验收场景

| ID | 场景 | 必须断言 |
|---|---|---|
| T01 | 安装后双 CLI/依赖规则 | 入口等价；Diagnostics 不导入 Runner/Reporting/Review/Builder |
| T02 | C2/C1 回归 | 原 profile/命令/契约有效，无新 HTTP/Diagnostics 隐藏依赖 |
| T03 | 评分错误与任务失败 | error→measurement_error；已知任务 fail 保留，缺 Run 不造失败 |
| T04 | 充分/映射区别 | matched 不当事实充分；verified_absent 不单独当 cov=0 |
| T05 | 多调用/平行 Run | 显式 call；无跨 Run 拼接/默认最后调用 |
| T06 | 全量/partial parser | 仅合格完整字面范围允许否定，partial 不认漏解析 |
| T07 | 构建未关联/索引清单 | 组件观察不冒充当前 Run，不推断向量可达性 |
| T08 | 阶段边界丢失 | 完整链定位最窄区间；缺 R2 保留竞争环节 |
| T09 | 改写/未知来源/同值跨年 | 保留 unresolved/来源冲突，不判确定漏检 |
| T10 | 替代充分组/partial gold | 合法 S2 阻止“不足”；unjudged 不当 irrelevant |
| T11 | 引用/策略/路由 | 只有对应有效审定才能支持具体失败；不从关键词/Store 权限推断 |
| T12 | 选择三渠道/重复 query | 固定 seed 可重现、重叠去重、不混总体根因比例 |
| T13 | 严格资格逐项缺失 | 缺实际请求/模型固定/工具规则/审定/容量均不派发 |
| T14 | 权限/出域/拒答题 | 无权 gold 不发送，policy_block 题不注入答案 |
| T15 | 充分组物化 | 所有审定背景/证据/顺序保留，无参考答案/标签泄漏 |
| T16 | 中文/emoji/多区间 patch | patch 外字节一致，baseline 原文重写等于实际原请求 |
| T17 | 容量/gold截断/模型漂移 | 发送前超限零调用；实际漂移 non_strict/unverified |
| T18 | AB/BA/重复基线 | 每对新调用，原失败的新 baseline pass 仍保留 |
| T19 | 未支持干预/HTTP字段 | 明确拒绝，无 fallback、隐式工具或任意脚本 |
| T20 | 四转移与全部 unknown | 受限 effect_code，不产唯一根因概率 |
| T21 | 重复/分母手算 | planned=4、measured=3、恢复1/2、带缺口query差0.25 |
| T22 | 评分视图/旧标签 | 新输出使用新 target；缺新审定则 unknown，错 gold/package 拒绝 |
| T23 | outcome_unknown | 发送后断连/崩溃不静默重发，无可信终态不造 timed_out Run |
| T24 | worker/outbox/checkpoint | 中断恢复保留调用身份；已提交输出不重发 |
| T25 | 预算/取消/重试 | 失败与未决计入额度，未完成全保留，不以质量触发重试 |
| T26 | 评分/报告失败 | 恢复只续下游；业务请求次数不增加 |
| T27 | 撤权/损坏/跨域缓存 | 读取/计划/执行/报告/导出均重查，不转空内容 |
| T28 | 清单/回执/并发提交 | 无半批成功，manifest后失败明确阶段，同内容幂等不覆盖异内容 |
| T29 | JSON/Markdown/导出 | 同源完整明细，无敏感日志/路径越界/远程内容执行 |
| T30 | fixture/外部导入 | fixture标签必保留；无请求的外部“Oracle”只能 unverified |
| T31 | 新证据/复核回流 | 建议不改 gold/原 Judgment；新共同基线产生新结果 |
| T32 | 真实服务接缝 | 实际 endpoint/固定模型/授权预算与载荷核验；无材料则 not_run，不伪报通过 |

## 7. 就绪与后续权限

需求映射为 [I01–I24](../99-requirement-traceability.md#8-c3-diagnostics-需求映射)；
交接 C3-H001 逐文件/部分绑定。内容就绪通过只表示可提案，
不表示 C1/C2 已实现或 T01–T32/真实模型质量验收通过。
当前无绑定 OpenSpec 消费方，不创建提案/任务/应用代码。

公开接口、数据含义、所有权、依赖方向、持久化、授权、
关键状态机和验收行为变更需设计确认；
私有辅助组织与保持契约的重构由实施者决定。
