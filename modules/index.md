# RAG 测评框架 v2：工程详细设计索引

- 设计标识：rag-evaluation-v2；根目录：C:/Users/32943/Desktop/RAG-evaluation。
- 设计包：modules/；模式：续建设计包；包/索引修订：r022。
- 索引状态：Confirmed。
- 单元修订独立：C1/C2 业务与计划验收保持原值；本轮新增 C3/r001 及共享导航。

2026-11-10 用户明确授权连续完成 Diagnostics 模块包及相关设计写入，
只在实质方案分歧或必要外部材料缺失时询问。
本轮 Confirmed 表示在该委托下完成设计与校验，不声称逐节人工复审。
设计就绪不等于代码、应用测试或真实业务验收通过。

## 1. 设计依据与范围

行为基线为 [RAG 测评设计 v2](../doc/RAG测评设计_v2.md)，
补充输入为 [数据集生成详设](../doc/RAG测评设计_v2_数据集生成详细设计.md)。
用户已选 C2 优先实施，C1 生成向 C2 对齐；Diagnostics 设计现在完成，
实际实施安排在数据和基础评分闭环之后。

核心约束：
Python 3.11+、单包/CLI、不可变产物；
gold 绑定源 revision 且不随系统 chunk 改动；
普通系统输入不含参考答案/审定/评分；
实际上下文与候选分离；
缺失、未知、测量故障和真实失败分开；
报告保留覆盖与三态门禁；
干预只产生范围受限的结论。

本包覆盖所有顶层模块的当前声明范围；
复杂原格式/视觉评分、在线模型裁判、通用回放、概率校准、
自动修复及服务化等远期扩展仍有明确延期。
不能用顶层 Complete 声称所有 v2 远期能力或代码已完成。

## 2. 状态与实施权限

审阅状态：Pending=未设计正文；In Progress=自身部分完成；
Confirmed=自身声明范围当前有效；Needs Review=依赖/假设变化需重审；
Superseded=已替代。
详设覆盖独立为 Not Started/Partial/Complete。
候选状态独立为 Design pending/Ready for OpenSpec/Needs Review/Superseded。
每份交接生命周期为 Current/Needs Review/Superseded。

包修订递增不自动使无关候选失效；以消费的确切单元修订和假设判断。
新快照保留旧批准正文，只更新旧快照生命周期及后继链接。
当前无绑定 OpenSpec 消费方；本轮不创建提案、tasks 或应用代码。

公开接口、数据含义与归属、依赖方向、持久化、权限、
关键状态机和验收行为变更须设计确认；
私有辅助组织和保持契约的重构由实施者决定。
用户本轮免逐步确认适用于当前设计交付，不是未来随意改契约许可。

## 3. 文档层级与模块计划

### 3.1 项目级文档

| 单元 | 文档修订 | 内容边界 |
|---|---|---|
| [总体架构](00-system-architecture.md) | r002 | 第 1–9 节规则正文 r001，进度 r002 |
| [工程目录/依赖](01-directory-layout.md) | r002 | 第 1–7 节规则正文 r001，进度 r002 |
| [横切规则](02-cross-cutting-concerns.md) | r002 | 第 1–10 节规则正文 r001，进度 r002 |
| [需求验收映射](99-requirement-traceability.md) | r004 | C2 R01–R32/r001；C1 D01–D26/r001；C3 I01–I24/r001；进度/导航独立 |

以上状态均 Confirmed。三份项目级设计本轮无需改变。
详细修订/稳定部分以当前交接为准，不能只按文档整体修订推断业务变化。

### 3.2 模块、契约及详设

每个模块的公共契约由提供方拥有；主 README 的 Contracts 是核心唯一正文。
新 profile 的唯一正文可位于该提供方的独立主题文件，不在消费者重复定义。

| 模块及主文档 | 文档修订 | 当前核心契约及范围 | 审阅 / 详设 |
|---|---|---|---|
| [artifact-store](modules/artifact-store/README.md#contracts) | r004 | r002；本地字节/资产/授权/提交 | Confirmed / Complete |
| [review](modules/review/README.md#contracts) | r001 | r001；文件审定记录/绑定，完整队列延期 | Confirmed / Complete |
| [dataset-registry](modules/dataset-registry/README.md#contracts) | r004 | r002；核心对象/导入/审定衔接/正式版本发布 | Confirmed / Complete |
| [dataset-builder](modules/dataset-builder/README.md#contracts) | r001 | r001；生成/独立求解/质检/发布投影，五主题 r001 | Confirmed / Complete |
| [adapters](modules/adapters/README.md#contracts) | r003 | 核心 r001；C2 JSONL 导入；C3 独立扩展 r001 | Confirmed / Complete |
| [evidence-mapper](modules/evidence-mapper/README.md#contracts) | r003 | 核心 r001；保守文本定位；C3 独立扩展 r001 | Confirmed / Complete |
| [evaluators](modules/evaluators/README.md#contracts) | r003 | r001；确定规则/已有评分导入/事实与任务组合 | Confirmed / Complete |
| [metric-planner](modules/metric-planner/README.md#contracts) | r003 | r001；C2 离线依赖与完整槽位 | Confirmed / Complete |
| [diagnostics](modules/diagnostics/README.md#contracts) | r001 | r001；证据诊断/竞争假设/干预规划/受限解释 | Confirmed / Complete |
| [runner](modules/runner/README.md#contracts) | r004 | 核心 r002；C2 本地评分；C3 独立扩展 r001 | Confirmed / Complete |
| [reporting](modules/reporting/README.md#contracts) | r004 | 核心总体 r002；C1–C12/r001、C13–C18/r002；C3 扩展 r001 | Confirmed / Complete |

八份原模块 README 本轮仅更新导航、进度及新主题入口；
核心业务/契约/验收修订保持原值，Review/Builder 文件未改。
声明 Complete 均限各自范围，不表示实现、数据质量或所有远期插件已经交付。

Diagnostics 五文件为主 README、
[观察规则](modules/diagnostics/observations.md)、
[干预协议](modules/diagnostics/interventions.md)、
[结果解释](modules/diagnostics/interpretation.md)、
[运行产物/安全](modules/diagnostics/operations.md)，均 Confirmed/r001。

C3 提供方扩展：
[Adapter 文本生成/阶段观察](modules/adapters/diagnostic-extension.md#contracts)、
[Mapper 诊断阶段范围](modules/evidence-mapper/diagnostic-stages.md#contracts)、
[Runner 配对试验](modules/runner/diagnostic-experiments.md#contracts)、
[Reporting 诊断报告](modules/reporting/diagnostic-reporting.md#contracts)，
均 Confirmed/Complete/r001；不进入 C1/C2 的必需实施集合。

## 4. 依赖方向规划

依赖采用消费者→提供方，规则正文见 [架构第 4 节](00-system-architecture.md#4-依赖方向)。
模块包含层级不是依赖图；跨模块只使用 contracts/api。

| 消费者 | 提供方 |
|---|---|
| artifact-store | 无内部提供方 |
| review | artifact-store |
| dataset-registry | artifact-store、review |
| dataset-builder | artifact-store、review、dataset-registry |
| adapters | artifact-store |
| evidence-mapper | artifact-store、dataset-registry、adapters |
| evaluators | artifact-store、dataset-registry、adapters、evidence-mapper |
| metric-planner | dataset-registry、adapters、evaluators |
| diagnostics | artifact-store、dataset-registry、adapters、evidence-mapper、evaluators |
| runner | artifact-store、review、dataset-registry、adapters、evidence-mapper、evaluators、metric-planner、diagnostics |
| reporting | artifact-store、dataset-registry、runner、evaluators、diagnostics |

CLI 作为组合根投影 DTO，不被业务模块反向依赖。
Run/Span/GenerationCall 归 Adapter；作业/调度归 Runner；
Judgment 归 Evaluator；诊断/计划/效果解释归 Diagnostics；
统计/报告/gate 归 Reporting；正式 gold 归 Registry；ReviewRecord 归 Review。
Diagnostics 禁止导入 Runner/Reporting/Review/Builder/Planner，含类型导入。
C3 无新增依赖边，也不使 C2 必须注入 Diagnostics。

## 5. 候选推进范围

### C2：优先实施的离线评测闭环

结果：已有正式数据、A/B Run、有效标注 → 离线评分/覆盖/配对统计/
Markdown、JSON 报告与 pass/fail/inconclusive gate。

Implement：Store、最小 Review、Registry、Adapter JSONL、
文本 Mapper、确定规则/已有审定 Evaluator、Planner、
C2 Runner 及 Reporting、CLI/本地恢复与 E01–E24。
无已验证应用实现；若后来出现代码，必须核验同契约和真实测试才 Reuse。
Defer：Builder 生成、在线调用/裁判、Oracle/Diagnostics、
完整审定平台、视觉、Web/分布式与远程存储。
缺标签/上下文不补造分数；fixture 不证明真实质量；业务阈值需实际输入。
契约和验收基线与此前相同，不因 C3 设计暗增能力。

[C2 切片](slices/c2-offline-evaluation.md)文档 r005，
第 1–6 节业务/验收 r001，第 7 节进度 r005。
Ready for OpenSpec；当前 [C2-H003](handoffs/c2-r003.md)，包 r022。

### C1：完整数据集生成、审定与正式发布

结果：固定源/参考 → 事实/蓝图/分组采样 → 合成/独立求解 →
质检/文件审定 → 正式 DatasetVersion 和 C2 兼容报告。
curated_import 为其中离线子模式。

Implement：Builder 主契约及五主题；utf8_text_v1、
reference_extraction_jsonl_v1、recorded_model_response_v1、
chat_completions_json_v1；Store/Review/Registry 必要本地行为；
C1 CLI/G01–G32。
Reuse：C2 若先实施提供方，先核验代码/测试，不把设计算现成实现。
Defer：全格式 parser/OCR/视觉具体接入、embedding、动态工具、
Web 审定/身份签名、远程/分布式；更多 C2 数值/视觉 profile 单列扩展。
Builder 可验证乘除/比例，C2 NumericRule 仍只 identity/add/subtract；
无法无损投影则 blocked。人工核心、自动扩展和 replay 来源分开，
不泄漏权限题隐藏答案，不用抽检代替逐题核心审定。

[Builder 包](modules/dataset-builder/README.md)六文件均 r001；
[C1 切片](slices/c1-dataset-generation.md)文档 r002，
第 1–6 节业务/验收 r001、第 7 节进度 r002。
Ready for OpenSpec；当前 [C1-H002](handoffs/c1-r002.md)，包 r022。

### C3：证据诊断、严格文本 Oracle 与干预报告

结果：全预期离线观测 → 症状/观察/竞争假设 →
固定抽样/资格/干预计划 → 可选新配对文本生成 →
同口径评分/协议检查/受限结论 → 诊断报告。

Implement：Diagnostics 五文件、四提供方独立扩展、
[C3 CLI/集成/T01–T32](slices/c3-diagnostics.md)及 I01–I24 映射，均 r001。
依赖按实际代码证据兼容复用或实施必需核心，
不要求 Builder 或完整人工平台。
在线只执行 strict_context_oracle 的单次最终文本生成，
固定模板/模型/采样/工具/权限/预算，仅替换审定源 context；
无条件则 blocked，不自动改模型或截断证据。
外部结果可导入，但缺协议材料为 unverified。
新响应缺评分来源则效果 unknown，不隐式启动裁判。

Defer：其他干预的在线通用执行、多模态/工具链、诊断 LLM/概率、
自动修复、干预 CI/gate、分布式。fail→pass 不证明检索唯一根因。
Ready for OpenSpec；当前 [C3-H001](handoffs/c3-r001.md)，包 r022。
建议在 C1/C2 基础闭环实测后实施，不改变 C2 快速展示的优先级。

## 6. 运行输入与非阻塞边界

首个真实业务域、数据规模/格式/权限、原始实际 traces、
固定模型服务/协议/容量/默认值、凭据/出域、
新响应评分来源/校准、业务阈值/预算仍需运行前提供并核验。
这些缺失不妨碍设计表达 blocked/unknown/inconclusive，
也不允许用假数据或默认阈值代替真实质量依据。

Windows NTFS 锁/Job Object、HTTP 未决、依赖版本、安装包及性能
均需实施测试。当前没有代码/模型调用/应用验收证据；
精确说明见各交接。没有需要用户现在选择的阻塞设计分歧。

## 7. 交接与本轮检查

- Current：[C1-H002](handoffs/c1-r002.md)、[C2-H003](handoffs/c2-r003.md)、[C3-H001](handoffs/c3-r001.md)。
- Superseded：C1-H001、C2-H001/H002；旧正文保留，后继可追溯。
- C1/C2 业务与验证单元不变，刷新仅为共享文档/导航确切修订；
  C3 扩展 profile 与它们的实施范围隔离。
- C1 D01–D26/G01–G32、C2 R01–R32/E01–E24、C3 I01–I24/T01–T32 均为计划验证。
- 无绑定 OpenSpec 消费方；可由用户使用对应交接的标准提示词，
  本次没有调用 OpenSpec、提交 Git 或编写应用。
- 交付前核对内容回读、相对链接/锚点、修订、完整数量及依赖方向；
  这些文档检查不冒充应用验收。
- 后续变更按影响使相关交接 Needs Review，不因包修订增加自动失效无关候选。

## 8. 修订记录

| 包修订 | 日期 | 内容 | 影响 |
|---|---|---|---|
| r001 | 2026-11-09 | 确认初始设计包规划与索引 | 所有架构及模块正文保持 Pending；无既有交接需要失效 |
| r002 | 2026-11-09 | 确认总体架构 r001，同步契约所有权与依赖摘要 | 模块正文仍为 Pending；C1/C2 仍为 Design pending；无交接需要失效 |
| r003 | 2026-11-09 | 确认工程目录与依赖规则 r001 | 总体架构保持 r001；模块正文与候选状态保持原状；无交接需要失效 |
| r004 | 2026-11-09 | 确认横切规则 r001，完成项目级架构设计 | 模块正文仍为 Pending；C1/C2 仍为 Design pending；无交接需要失效 |
| r005 | 2026-11-09 | 记录 C2 优先选择，确认 Artifact Store 概述与公开契约 r001 | artifact-store 为 In Progress / Partial；其他模块仍 Pending；无交接需要失效 |
| r006 | 2026-11-09 | 确认 Artifact Store 本地实现与恢复，补全公开契约 r002 | 该模块 C2 范围为 Confirmed / Complete；其余模块 Pending；无已确认消费者或交接需要失效 |
| r007 | 2026-11-09 | 确认 Review 的 C2 最小审定记录设计 r001 | review 为 Confirmed / Complete，仅覆盖声明范围；其余模块状态不变；无交接需要失效 |
| r008 | 2026-11-09 | 确认 Dataset Registry 核心数据契约 r001 | dataset-registry 为 In Progress / Partial；其他模块状态不变；无交接需要失效 |
| r009 | 2026-11-09 | 确认 Dataset Registry 的 C2 导入、发布与恢复，公开契约推进到 r002 | dataset-registry 为 Confirmed / Complete，仅覆盖声明范围；其他模块状态不变；当前无已确认消费者或交接需要失效 |
| r010 | 2026-11-09 | 确认 Adapters 的 C2 运行记录与 JSONL 导入设计 r001 | adapters 为 Confirmed / Complete，仅覆盖声明范围；其他模块状态不变；无已确认消费者或交接需要失效 |
| r011 | 2026-11-09 | 确认 Evidence Mapper 的 C2 文本定位与严格缺失设计 r001 | evidence-mapper 为 Confirmed / Complete，仅覆盖声明范围；其他模块状态不变；无已确认消费者或交接需要失效 |
| r012 | 2026-11-09 | 确认 Evaluators 的 C2 指标、确定规则、已有审定评分导入与组合评价设计 r001 | evaluators 为 Confirmed / Complete，仅覆盖声明范围；C2 仍为 Design pending；其他模块状态不变；无已确认消费者或交接需要失效 |
| r013 | 2026-11-09 | 确认 Metric Planner 的 C2 观测、依赖解析与离线计划设计 r001 | metric-planner 为 Confirmed / Complete，仅覆盖声明范围；C2 仍为 Design pending；其他模块状态不变；无已确认消费者或交接需要失效 |
| r014 | 2026-11-09 | 确认 Runner 的 C2 实验准备、观测绑定与单次执行设计 r001 | runner 为 In Progress / Partial；预算、重试、缓存及恢复仍待确认；C2 仍为 Design pending；无已确认消费者或交接需要失效 |
| r015 | 2026-11-09 | 补全 Runner 的 C2 预算、失败控制、重试、评分复用与本地恢复，契约推进到 r002 | runner 为 Confirmed / Complete，仅覆盖声明范围；C2 仍为 Design pending；其他已确认模块不变；无已确认消费者或交接需要失效 |
| r016 | 2026-11-09 | 确认 Reporting 的 C2 聚合、覆盖、缺失范围与 A/B 配对统计设计 r001 | reporting 为 In Progress / Partial；报告输出和门禁仍待确认；C2 仍为 Design pending；无已确认消费者或交接需要失效 |
| r017 | 2026-11-09 | 补全 Reporting 的 C2 报告、样例、三态门禁与本地导出，公开契约推进到 r002 | reporting 为 Confirmed / Complete，仅覆盖声明范围；C2 仍为 Design pending；其他模块修订不变；无已确认消费者或交接需要失效 |
| r018 | 2026-11-09 | 确认 C2 的 CLI 命令、装配、端到端集成与计划验收 r001 | C2 仍为 Design pending，需求映射与交接待确认；模块修订不变；无已确认消费者或交接需要失效 |
| r019 | 2026-11-09 | 确认 C2 需求—设计—验收映射 r001，同步切片进度 | C2 仍为 Design pending，待就绪审查与交接；切片文档 r002，仅第 7 节状态更新；业务模块修订不变；无消费者或交接需要失效 |
| r020 | 2026-11-09 | 确认 C2 内容就绪检查和交接 C2-H001，清理过期进度说明 | C2 为 Ready for OpenSpec；交接 Current；仅进度及交接基线更新，业务单元与契约修订保持原值；C1 和延期模块仍 Pending；无已存在消费者或交接需要失效 |
| r021 | 2026-11-10 | 按用户委托完成 C1 生成模块主契约、五份主题、CLI/集成、D01–D26/G01–G32 与交接 C1-H001 | Builder Confirmed/Complete；C1 Ready for OpenSpec；C2 业务/验证单元保持原修订，刷新 C2-H002 Current、原 C2-H001 Superseded；Diagnostics 仍 Pending；无已绑定 OpenSpec 消费方 |
| r022 | 2026-11-10 | 按委托完成 Diagnostics 五文件、四提供方独立扩展、C3 切片/I01–I24/T01–T32 与 C3-H001；修复历史导航 | Diagnostics Confirmed/Complete；C3 Ready for OpenSpec；C1/C2 核心与验收不变，刷新 C1-H002/C2-H003 Current；旧交接 Superseded；无 OpenSpec 消费方 |
