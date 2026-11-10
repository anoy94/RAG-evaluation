# C2：CLI 契约与离线评测集成

- 候选：C2。
- 审阅状态：Confirmed。
- 内容修订：r005。
- 单元修订：第 1–6 节 r001；第 7 节及基线摘要 r005。
- 设计覆盖：Complete，仅限本文件声明范围。
- 候选就绪状态：Ready for OpenSpec。
- 运行边界：Windows、本地 NTFS、Python 3.11+、单机 CLI。
- 结果：导入已有数据和 A/B 记录，形成可追溯的 Markdown/JSON
  报告、覆盖、配对统计及三态门禁。
- 本文件确认工程设计及计划验收，不表示应用或测试已经完成。

## 1. 依据、范围与依赖

项目级依据：
- [总体架构 r001](../00-system-architecture.md)。
- [目录设计 r001](../01-directory-layout.md)。
- [横切规则 r001](../02-cross-cutting-concerns.md)。
- [v2 第 12、14、16、18 节](../../doc/RAG测评设计_v2.md)。

本切片消费下列确切设计：

| 模块 | 业务设计与契约修订 | 本切片使用范围 |
|---|---|---|
| [artifact-store](../modules/artifact-store/README.md#contracts) | r002 | 本地字节、源资产、授权、完整性及提交 |
| [review](../modules/review/README.md#contracts) | r001 | 已有审定导入、读取与 revision 绑定 |
| [dataset-registry](../modules/dataset-registry/README.md#contracts) | r002 | 导入、发布准备、正式版本、读取和分组 |
| [adapters](../modules/adapters/README.md#contracts) | r001 | external-rag-jsonl-v1 导入与固定运行读取 |
| [evidence-mapper](../modules/evidence-mapper/README.md#contracts) | r001 | 文本定位和严格缺失 |
| [evaluators](../modules/evaluators/README.md#contracts) | r001 | 确定规则、已有评分材料及组合指标 |
| [metric-planner](../modules/metric-planner/README.md#contracts) | r001 | catalog_only、available_inputs 和输入缺口 |
| [runner](../modules/runner/README.md#contracts) | r002 | 准备、执行、操作身份、预算、复用与恢复 |
| [reporting](../modules/reporting/README.md#contracts) | r002 | C1–C12/r001 分析，C13–C18/r002 报告及门禁 |

表中修订标识业务设计与契约；文档的进度更新不改变这些基线。
文档修订及各单元修订以[交接 C2-H003](../handoffs/c2-r003.md)为准。

Implement：
上述模块的已确认 C2 范围，以及本文件定义的 CLI 装配和命令行为。
设计引用是待实施基线，不计作已有可复用代码。

Reuse：
采用成熟 JSON/YAML、类型校验、统计、随机及 CLI 库。
具体库版本在实现时锁定，并验证 Windows 安装和打包。
没有已验证可复用的项目业务实现。

Defer：
dataset-builder、完整复核 UI、在线 Adapter、模型裁判、
视觉映射、Oracle、Diagnostics、远端存储和 Web 服务。
不注入占位 Diagnostics，不为延期模块创建空实现。
不通过替身声称已完成这些集成。

CLI 使用各提供方公开入口。
不读取其私有目录，不复制核心对象 schema，
业务模块不反向依赖 CLI。

## 2. CLI 入口、配置与文件边界

rag-eval 与 python -m rag_eval 使用同一入口。
帮助与版本命令不初始化工作区。
本轮不提供后台守护进程或网络 API。

公共参数：
- --config <runtime.yaml>：显式运行配置。
- --workspace <directory>：覆盖工作区位置。
- --format json|text：默认 text。
- 长操作的 --operation-key：调用方预先提供的非敏感键。

非秘密配置优先级、相对路径解析及 RAG_EVAL_ 白名单
遵循横切规则。不自动搜索父目录配置。
未知字段、重复键、非法枚举、NaN 和 Infinity 均拒绝。
配置使用安全 YAML；业务请求使用严格 UTF-8 JSON，无 BOM。

RuntimeConfig 包含：
- schema_version=2.0、profile=cli.runtime.c2.v1。
- workspace；默认启动目录下的 .rag-eval。
- store、registry、runner 的根目录及各自公开配置。
- access_policy_file。
- 固定内置 Provider ID、版本和支持的 profiles。
- Reporting 的 LocalExportPolicy。
- CLI 输入限额。

root 在启动时解析为绝对路径。
Provider 子目录不得彼此相同、互相包含或落入导出目录。
固定内置 Provider 选择不接受任意 import 路径或脚本。
普通命令均使用 open_existing，不自动重建缺失工作区。

workspace init --component store|registry|runner：
- 仅对明确指定组件使用其公开 create_new 构造模式。
- Store 先创建，Registry 和 Runner 再使用已有依赖初始化。
- 每次命令只初始化一个组件，不宣称三者原子初始化。
- 目标已存在或不完整时保留现场并返回冲突。
- 不提供覆盖初始化、自动清理或“修复”旧身份功能。

本地授权 Provider 由 CLI 依据操作系统主体和受保护的
access_policy_file 装配，调用 Store 已定义的 check 端口。
策略明确主体、访问域、read/write、policy_revision 及导出范围。
无匹配授权时拒绝，不使用默认全域允许。
每次检查核对当前策略；策略变化使旧上下文不能继续授权。
工作区依赖操作系统保护，不宣称服务级多租户隔离。

模拟用户 access_scope、审定人声明和进程权限分别处理。
不从数据、Run、审定文件或被测输出中接受存储授权。
凭据不进入配置快照正文、日志或报告。

--input <file> 表示命令表规定的请求 JSON。
文件中通过 ArtifactRef 或所有者 DTO 引用业务产物；
不接受 latest、裸 ID 或嵌套路径下载指令。
CLI 使用所有者校验器验证请求，再通过 Store 保存固定内容。

本地文件只有显式命令参数可打开。
外部文件中的路径或 URL 不作为读取入口。
--input 文件限额默认 8388608 字节、最大嵌套深度 64；
大批量记录通过 ArtifactRef 引用并流式导入。
Provider 的更严格限额继续生效，不静默截断。

## 3. 命令及输入输出

表中已有所有者请求类型的 --input 文件直接使用该类型，
不再包裹另一份业务 schema。
仅由多个引用字段组成的 CLI 请求另用 schema_version=2.0、
contract_profile=cli.input.c2.v1、command 和表中列明的字段；
未知字段拒绝。引用字段沿用其所有者的完整 DTO。

| 命令 | 输入与主要参数 | 调用及结果 |
|---|---|---|
| artifact put | --file、--access-domain；可选预期摘要/长度 | Store.put_blob；返回 ArtifactRef |
| artifact register-source | SourceArtifact | Store.register_source；返回正式源资产 |
| review import | source_ref、import_policy_ref | Review.import_records；返回完整导入结果 |
| dataset import | DatasetImportBundle | Registry.import_dataset；返回草稿及回执 |
| dataset review-target | draft_ref、object_type、object_key、annotation_context_ref | Registry.build_review_target；返回真实审定目标 |
| dataset prepare-release | draft_ref、release_request_ref | Registry.prepare_release；返回 ready/blocked 及校验报告 |
| dataset publish | preparation_ref | Registry.publish；返回正式 version_ref |
| dataset validate | version_ref | Registry.open_dataset(evaluation) 并读完正式成员及必需引用；返回核验摘要和原限制 |
| runs import | RunImportRequest | Adapter.import_run；返回完整批次及回执 |
| scoring-view | ScoringRequest | Evaluator.build_scoring_view；返回固定视图及回执 |
| plan | ExperimentRequest；必需 --operation-key | Runner.prepare_experiment；返回准备、蓝图、正式计划和操作定位 |
| run / evaluate | preparation_ref、reuse_source_refs；必需 --operation-key | Runner.execute；返回操作及明确执行快照 |
| job status | operation_key、operation_kind | Runner.get_operation；只读操作与已提交快照 |
| job resume | 完整 OperationHandle | Runner.resume_operation；不重置 deadline 或配额 |
| job cancel | OperationHandle、reason | Runner.cancel_operation；返回取消请求回执 |
| compare | AnalysisRequest | Reporting.analyze；返回 AnalysisResultRef |
| report | ReportWorkflowInput | 按需 decide_gate，随后 render_report；返回决定及完整报告包 |
| report export | bundle_ref；--output-root | Reporting.export_bundle；返回 ExportReceipt |

请求内引用的对象仍必须提前提交。
artifact put 不证明文件满足业务 schema；
register-source 不证明原文、gold 或审定正确。

dataset validate：
- 核验固定正式版本、全部成员数量、schema 和必要引用。
- 读取生命周期、原发布检查及实际审定范围。
- 不重新赋予质量等级，不改变发布策略或内容。
- 合法缺失仍显示限制；损坏、不可读及悬空引用则失败。

plan：
- 本轮只支持 trace_audit，包含本地导入和确定性映射准备。
- 不产生 Judgment，不调用被测系统或模型。
- 返回计划的 gate_input_status、依赖缺口和复现限制。
- no_known_input_gap 不等于正式门禁就绪。
- 合法的诊断准备可以成功，不能预测 N_ok 或系统分数。
- 固定请求的修改形成新实验修订。

run 和 evaluate 是同一离线执行入口的两个命令名。
不意味着本框架重新发起业务推理。
修改评价包时先构造引用原 Run 的新 ExperimentRequest，
重新 plan；不在既有准备清单中直接替换评价版本。
复用来源必须显式给定，默认空；不搜索最高分或最新评分。

scoring-view 需要已存在的 Subject。
可先形成保留缺口的诊断准备，再导出实际评分视图；
补入审定后形成新请求和新准备。
视图不等于标签，不自动生成审定内容。

compare 使用同一实验中的 A/B arms 和明确执行快照。
不直接将两个裸实验 ID 拼接为配对比较。
需要比较的来源运行先进入同一个固定 ExperimentRequest。

ReportWorkflowInput 包含：
- analysis_result_ref、report_policy_ref。
- gate_requested、gate_policy_ref。
- gate_implementation_ref、renderer_implementation_ref。
- provenance_ref。

gate_requested=false 时，gate_policy_ref 必须为 null。
gate_requested=true 时，缺少策略形成 inconclusive 决定。
CLI 先将 GateRequest 固定并提交，再调用 decide_gate；
取得明确 GateDecision 后固定 ReportRequest 并渲染。
不在模板中决定门禁，不为提高覆盖补跑评价。

report 默认只将报告保存到 Store。
文件导出必须另行显式执行 report export；
失败不会抹去已经提交的报告包。

## 4. 命令结果、退出码与中断

CLI 拥有 CommandResult：
- schema_version=2.0、contract_profile=cli.result.c2.v1。
- command、command_invocation_id。
- command_status：completed、blocked、failed 或 interrupted。
- outcome：提供方的正式状态及限制。
- result_refs：按 provider.result_role 为键的映射，
  值为对应所有者的完整引用 DTO，不降为裸 ID。
- operation_handle 或可供 job status 使用的操作定位。
- gate：requested、status、decision_ref；status 为
  pass、fail、inconclusive 或 null。
- error：失败时提供 reason_code、safe_message、stage、owner；
  正常完成时为 null，保留提供方的完整结构化原因。
- 已提交中间产物的安全引用。
- 独立操作时间及实现版本。

未请求 gate 时：
gate.requested=false、status=null、decision_ref=null。
正文可以显示 not_requested，不能产生 pass。

JSON 模式 stdout 恰好一个 CommandResult 和末尾换行。
进度、安全诊断及 Provider 子进程输出进入 stderr。
禁止将原 query、gold、上下文和凭据写入进度日志。
text 模式展示相同状态、引用、限制和下一恢复入口。

退出码沿用横切规则：
- 0：命令目标完成；请求门禁时须 gate=pass。
- 2：参数、结构、关联或执行前校验失败。
- 3：执行、权限、存储、完整性等故障，未完成命令目标。
- 4：已形成有效决定，gate=fail。
- 5：已形成有效决定，gate=inconclusive。
- 130：用户中断。

先判定命令目标是否完成，再应用 gate 退出码。
例如决定已提交但报告渲染失败，report 返回 3，
同时保留决定引用，不能返回 4/5 冒充报告已完成。
导出不重新请求门禁，成功返回 0 并保留原决定引用。

dataset prepare-release 返回 blocked 时：
command_status=blocked，退出 2，保留准备和校验报告引用。
计划合法但存在测量缺口时可返回 0，
outcome 必须保留 gate_input_status=blocked。

正常预算停止且形成 partial ExecutionResult 时，
run 目标完成，返回 0 并突出显示 partial 和未完成数。
这不等于质量通过；后续报告必须保留全部缺口。
Runner blocked/failed 或不能形成可信目标快照时，
按实际校验或执行错误返回 2/3。

Provider reason_code 原样保留，不统一改写成“运行失败”。
CLI 自有错误限于输入解析、配置、输出协议和装配边界：
cli.invalid_input、cli.invalid_config、cli.input_limit_exceeded、
cli.unsupported_component、cli.output_failure。
已存在组件的初始化冲突沿用其所有者错误。

第一次 Ctrl+C：
- 存在当前 Runner 操作时提交协作取消，停止后续阶段。
- 统计或渲染阶段停止本地计算，保留已提交引用；
  不取消历史上已完成的 Runner 操作。
- 采用 Runner 已确认的有限清理与 worker 终止规则。
- 能读取快照时返回其引用；不能确认结束时显示未决。
- 返回 130，不自动开始统计或渲染。

job cancel 成功只表示请求已经提交，不能显示“已停止”。
进程被强制终止或 stdout 管道断开时，
不承诺一定交付 CommandResult；
预先给定的 operation_key 仍可用于 status 和恢复。

## 5. 装配、集成流程与一致性

CLI 构造顺序：
Store 与授权 Provider
→ Review
→ Registry
→ JSONL Adapter
→ 文本 Mapper
→ Evaluators
→ Planner
→ Runner
→ Reporting。

仅按命令所需构造依赖，帮助或只读命令不创建运行目录。
CLI 不拥有另一套业务任务调度器、评分缓存或统计缓存。

完整主路径：

1. 明确初始化必要组件。
2. artifact put/register-source 提交明确文件、配置及源材料。
3. 导入数据草稿和已有审定；按发布策略准备并发布版本。
4. 导入完整 A/B 运行批次，保留失败及缺失观测。
5. 固定 ExperimentRequest、样本、评价包、
   AnalysisPolicy、GatePolicy 及 reporting_intent。
6. plan 产生全部 Subject、映射、材料绑定和 available_inputs 计划。
7. run/evaluate 形成 Judgment、操作检查点及执行快照。
8. compare 固定该准备和执行快照，形成完整分析。
9. report 形成决定、样例和 Markdown/JSON 报告包。
10. 按需 export，校验清单及全部字节。

涉及审定补齐时使用新的明确请求，
不修改已执行实验、原数据版本或 Judgment。
本地冻结策略不证明未查看既有来源结果。

CLI 不提供跨模块文件事务：
- 各 Provider 提交自己的不可变结果及回执。
- 后续阶段失败时，已经提交的结果继续有效。
- 下游只消费明确返回且通过当前读取校验的引用。
- 不通过目录存在性推断业务成功。

恢复边界：
- 准备/评分：使用 Runner 原 operation_key 和句柄。
- 导入/发布：按所有者幂等和冲突规则重新提交固定请求。
- 统计/渲染：重放固定请求，复用完全相同内容。
- 导出：只从已提交 bundle 重新核验并导出。
- 修改输入、策略、评价包或复用来源不属于原操作恢复。

代码位于 rag_eval.cli，公开进程入口由 pyproject.toml 注册。
解析、配置/授权装配、命令分派和结果序列化属于 CLI；
公开业务 DTO、算法和持久化仍归原模块。
私有文件拆分由实现决定。

## 6. 端到端验收设计

以下全部为计划，尚未执行。

验收使用安装后的包和真实本地 Store、Registry、
JSONL Adapter、文本 Mapper、Evaluator、Runner 与 Reporting。
不得只用 mock 证明 C2 主路径成立。
统计库、文件锁、Windows Job Object、目录提交与打包
必须在 Windows 本地 NTFS 上验证。

测试材料使用合成标识和固定内容；
审定测试记录验证绑定及消费协议，
不作为真实人工核验、裁判准确性或生产门禁证据。
报告必须显示演示身份、来源保证和适用范围。
人工正确性或业务外推需要另行提供真实证据。

基础夹具：
- 同一正式测试版本，4 个 query、2 个 arm、
  每题每 arm 一个固定 sample_slot。
- 明确 A/B 预期值、输出、参考和审定测试材料。
- 主二元系列按 query 顺序：A=[0,1,1,1]，B=[1,0,1,1]。
  双方均值均为 0.75，平均差为 0；1 胜、1 负、2 平。
- 不同文档族形成明确 group，不靠重复运行增加组数。
- 固定 CLI 请求、版本、随机 seed 和展示策略。

| 编号 | 场景 | 必须观察到的结果 |
|---|---|---|
| E01 | 安装后运行两种入口 | 相同命令行为、无导入副作用 |
| E02 | 正常主路径 | 8 个观测闭合，每个根指标实例保留 8 个槽位；A/B 结果可手算 |
| E03 | 删除 B 一条来源 Run | 预期槽位仍在；未完成/不可测原因可查，不缩小 N_total |
| E04 | 删除实际生成上下文 | 支持度不可测；不能以检索结果补位 |
| E05 | 来源 Run 为 timed_out | 保留真实系统失败；不能当测量失败或删去 |
| E06 | 评价器报错 | Judgment 为 error，value=null；与系统失败分开 |
| E07 | 一题两次预期生成，缺少一次 | query_score 未知；不只平均已知一次 |
| E08 | 缺失对可改变总体结论 | 同时展示识别范围和覆盖；总体统计门禁不能借共同集通过 |
| E09 | 明确有限基准门禁 | 固定规则分别产生 pass、fail、inconclusive；退出 0/4/5 |
| E10 | 少量组请求抽样推断 | 点估计保留，区间和相应门禁 inconclusive |
| E11 | 一个已核验硬性权限违规 | 其他未知不抵消 fail；普通任务失败不能冒充违规 |
| E12 | 错误 Run、旧 revision 或冲突审定 | 拒绝错配或保留未决，不自动选高分 |
| E13 | 只更换评价包 | 新准备/评分可追溯；来源 Run 不变，无新增业务调用 |
| E14 | 同输入重新分析及渲染 | 业务内容身份相同，操作回执分别记录 |
| E15 | 改变样本遍历顺序 | 固定样例、统计 seed 和结果不因遍历顺序改变 |
| E16 | worker 超时或中断后恢复 | 不覆盖已提交结果，不重置配额/deadline |
| E17 | 同 operation_key 换输入 | operation_conflict，不继续旧操作 |
| E18 | 当前权限撤销或引用损坏 | 明确失败，不用缓存/旧回执绕过 |
| E19 | 报告决定已提交，渲染失败 | 退出 3 并保留决定；重试不重新评分 |
| E20 | 导出中断、路径穿越或 junction | 不暴露完整假报告，不越界，不覆盖异内容 |
| E21 | JSON 命令输出 | stdout 单个合法结果，进度只在 stderr，无敏感正文 |
| E22 | Ctrl+C 与 job cancel | 请求状态与实际停止状态分开，可按键查回操作 |
| E23 | 非法行、超限或未知字段 | 全批次失败或明确错误，不静默删行/缩减任务 |
| E24 | 请求门禁但缺少业务策略 | 可生成诊断报告，gate=inconclusive，退出 5 |

E09 的 pass 仅验证声明测试基准和策略下的引擎行为；
不将小型演示转换为真实业务发布结论。
总体门禁验收须另配满足独立单位、完整测量、
来源资格及统计条件的固定夹具。

每项记录命令参数、退出码、JSON 结果、
产物摘要、数量及必要故障注入位置。
验收不只比较屏幕截图或检查文件是否存在。

## 7. 当前状态与后续交接

本文件第 1–6 节的 CLI、集成及验收设计保持 Confirmed / r001。
[C2 需求—设计—验收映射](../99-requirement-traceability.md)
已确认，C2 映射单元 r001、文档 r004；本节及基线摘要同步为 r005。
C2 内容就绪检查通过；[交接 C2-H003](../handoffs/c2-r003.md)
为 Current，绑定包 r022。候选为 Ready for OpenSpec。
本次仅刷新共享导航修订；CLI、集成与 E01–E24 不变，C1/C3 均不加入本切片。
该状态仅表示可以进入 OpenSpec 提案，不表示实现、验收或业务发布通过。

当前没有应用实现、执行过的验收记录或 OpenSpec 消费方。
业务数据、阈值及正式校准材料未被本文件补造。
这些缺少输入的情况必须能通过既定未知/阻塞行为表达。

公开命令、输入输出、退出码、授权、核心状态及验收行为
变化需要设计确认。
私有解析工具、类组织和契约保持的重构由实现决定。
