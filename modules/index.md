# RAG 测评框架 v2：工程详细设计索引

- 设计标识：rag-evaluation-v2
- 项目根目录：C:\Users\32943\Desktop\RAG-evaluation
- 设计包目录：modules/
- 模式：新建设计包
- 设计包修订：r020
- 本索引审阅状态：Confirmed
- 本索引内容修订：r020

本索引确认设计包规划。各架构文档、模块契约及详细设计的确认状态单独记录。

## 1. 设计依据与范围

主要依据：
[全链路 RAG 测评设计 v2](../doc/RAG测评设计_v2.md)。

补充输入：
[数据集生成详细设计](../doc/RAG测评设计_v2_数据集生成详细设计.md)。

v2 是本轮行为与边界的基线。已有数据集详设用于提取细节、核对一致性，
其内容进入本设计包时仍需按对应单元审阅。发现冲突时记录并提请决策。

设计覆盖 v2 的全部逻辑模块，推进顺序为：
总体架构 → 工程目录与依赖规则 → 横切规则 → 选定近期范围
→ 提供方契约 → 模块细节 → 集成流程 → 验收映射与交接。

基线包括：
- Python 3.11+、单包与 CLI、类型化校验及可审计文件产物。
- 黄金证据绑定源文 revision，与待测系统 chunk 身份解耦。
- 常规系统输入与参考答案、黄金证据、裁判结果隔离。
- 实际生成上下文与检索结果分别记录。
- 不可测、不确定、真实失败分别表达。
- 数据、运行、评分及对照具有独立版本和来源记录。
- 报告与门禁保留有效覆盖、缺失原因和三态结论。

## 2. 状态与实施权限

审阅状态：
- Pending：已规划，尚无批准正文。
- In Progress：自身范围内部分内容已确认，其余待确认。
- Confirmed：自身声明范围已确认且没有阻塞决策。
- Needs Review：已有内容因依赖或假设变化需要重新审阅。
- Superseded：已被替代或退役。

详设覆盖单独记录为 Not Started、Partial、Complete。
父级概述确认不代表子级细节确认；设计完成不代表代码完成。

候选就绪状态：
Design pending、Ready for OpenSpec、Needs Review、Superseded。
只有候选范围内全部必需设计通过检查，才可进入 Ready for OpenSpec。

公开接口、数据含义与归属、依赖方向、持久化契约、权限规则、
关键状态机及验收行为的变更需要确认。
实现者可在已确认边界内决定私有辅助函数、内部组织与契约保持的重构。

## 3. 文档层级与模块计划

### 3.1 项目级文档

| 设计单元 | 计划路径 | 审阅状态 | 内容修订 |
|---|---|---|---|
| 总体架构 | [00-system-architecture.md](00-system-architecture.md) | Confirmed | r002 |
| 工程目录与依赖规则 | [01-directory-layout.md](01-directory-layout.md) | Confirmed | r002 |
| 横切规则 | [02-cross-cutting-concerns.md](02-cross-cutting-concerns.md) | Confirmed | r002 |
| 需求与验收映射 | [99-requirement-traceability.md](99-requirement-traceability.md) | Confirmed（仅 C2） | r002 |

本表为文档修订；三份项目级设计正文保持 r001，
本次 r002 仅同步进度说明。需求映射单元保持 r001，文档为 r002。
各实现单元及契约修订以[交接 C2-H001](handoffs/c2-r001.md)为准。

总体架构先确认运行边界、模块职责、数据所有权、主要流程及依赖方向。
目录设计随后确定逻辑模块到代码包的映射。
横切规则统一权限、错误、版本、重试、幂等、恢复、审计与测试要求。

### 3.2 模块及契约归属规划

以下是包含层级。模块间依赖另见第 4 节。
每个模块主文档预留稳定的 Contracts 标题，作为公开契约的唯一正文位置。
消费者链接提供方定义及修订，不复制定义。

| 模块 ID 与职责 | 主文档及契约位置 | 核心契约归属（状态见主文档） | 审阅 / 覆盖 |
|---|---|---|---|
| artifact-store：不可变产物、内容寻址与谱系 | [modules/artifact-store/README.md](modules/artifact-store/README.md#contracts) | ArtifactRef、SourceArtifact、产物读写协议 | Confirmed / Complete |
| review：复核工作项与审定记录 | [modules/review/README.md](modules/review/README.md#contracts) | ReviewRecord、版本绑定与批次导入；ReviewItem 后续设计 | Confirmed / Complete |
| dataset-registry：数据与证据版本发布 | [modules/dataset-registry/README.md](modules/dataset-registry/README.md#contracts) | 核心对象、DatasetVersion、导入、审定衔接与发布协议 | Confirmed / Complete |
| dataset-builder：候选构建与质量验证 | [modules/dataset-builder/README.md](modules/dataset-builder/README.md#contracts) | BuildConfig、BuildJob、Blueprint、候选与验证记录 | Pending / Not Started |
| adapters：系统能力与运行接入 | [modules/adapters/README.md](modules/adapters/README.md#contracts) | 标准运行、阶段与生成产物、计量及 JSONL 导入协议 | Confirmed / Complete |
| evidence-mapper：跨阶段证据定位 | [modules/evidence-mapper/README.md](modules/evidence-mapper/README.md#contracts) | 文本锚点、来源、联合覆盖及严格缺失判定 | Confirmed / Complete |
| evaluators：评分行为与裁判版本 | [modules/evaluators/README.md](modules/evaluators/README.md#contracts) | MetricSpec、Judgment、评分导入与 Evaluator 协议 | Confirmed / Complete |
| metric-planner：依赖与适用性解析 | [modules/metric-planner/README.md](modules/metric-planner/README.md#contracts) | EvaluationPlan、依赖检查与执行任务描述 | Confirmed / Complete |
| diagnostics：症状、假设与受控干预 | [modules/diagnostics/README.md](modules/diagnostics/README.md#contracts) | DiagnosticFinding、Intervention、Oracle 协议 | Pending / Not Started |
| runner：实验执行、导入与恢复 | [modules/runner/README.md](modules/runner/README.md#contracts) | ExperimentManifest、实验作业与执行控制协议 | Confirmed / Complete |
| reporting：统计、报告与门禁 | [modules/reporting/README.md](modules/reporting/README.md#contracts) | AnalysisResult、ComparisonReport、GatePolicy、GateDecision | Confirmed / Complete |

artifact-store 的 C2 本地范围已确认，文档修订 r003，
公开契约 r002。
review 的 C2 最小审定记录范围已确认，文档与契约修订 r001；
完整队列和 ReviewItem 仍在后续范围。
dataset-registry 的 C2 范围已确认，文档 r003、公开契约 r002；
覆盖导入、审定衔接、版本发布、读取与恢复。
adapters 的 C2 本地导入范围已确认，文档 r002、公开契约 r001；
在线执行、实时采集及 Oracle 接入仍在后续范围。
evidence-mapper 的 C2 文本范围已确认，文档 r002、契约 r001；
事实支持计分归 Evaluator，语义改写和视觉映射留在后续范围。
evaluators 的 C2 确定性评价与已有审定评分范围已确认，
文档 r002、契约 r001；覆盖逐项指标、评分视图、审定导入、
事实覆盖及 TaskSuccess 组合。
在线裁判、自动主张提取、传统 IR 投影和专项评价留在后续范围。
metric-planner 的 C2 离线计划范围已确认，文档 r002、契约 r001；
覆盖固定观测、结果槽位、指标依赖、输入缺口和确定性计划。
计划保存与执行归 Runner，在线计划和动态工作流留在后续范围。
runner 的 C2 本地范围已确认，文档 r003、契约 r002；
覆盖实验准备、评价执行、预算与失败控制、
有限重试、显式评分复用和本地进程恢复。
每个操作采用一个协调者和一个在途 worker；
在线调用、多 worker 并行及分布式调度留在后续范围。
reporting 的 C2 聚合、覆盖、配对统计、Markdown/JSON 报告、
样例、三态门禁与本地导出范围已确认，文档 r003、契约 r002；
C1–C12 保持 r001，新增 C13–C18 为 r002。
自动校准、完整诊断、总体尾分位数与跨来源成本仍在后续范围。
上述八个模块本次只同步进度摘要；业务设计、公开契约和已确认单元修订保持原值。
review 文档与契约保持 r001；其余未设计模块内容修订为“—”。
Complete 表示该模块声明范围的设计覆盖，不表示实现完成。
公开字段、方法和状态转换在各自设计中确认；
本表汇总总体架构确认的所有权。

较大模块按以下主题分批审阅：
- dataset-builder：基础证据、蓝图与采样、合成与求解、质检、审定与发布衔接。
- runner：计划与执行、预算及失败处理、缓存与恢复。
- reporting：聚合与配对统计、报告解释、门禁规则。

初期以模块 README 承载批准内容和本地进度。
需要拆分主题文件时，先确认拆分计划及关联更新。

## 4. 依赖方向规划

依赖方向已由[总体架构第 4 节](00-system-architecture.md#4-依赖方向)确认；
下表为消费者使用提供方契约的摘要。
它不是运行调用时序，也不表示候选必须实现全部依赖模块。

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

依赖中的契约位置统一指向第 3.2 节对应提供方的 Contracts 标题。
CLI 作为组合入口调用业务模块，业务模块不反向依赖 CLI。

架构设计需区分契约引用、运行调用和存储访问。
运行控制与记录类型分离：adapters 拥有标准运行记录，
runner 拥有实验调度与作业状态，避免双方互相依赖。

ReviewItem 的目标使用类型与 revision 引用标识；
review 不通过反向导入各业务模块来修改被审对象。
审定结果由对象所有者校验并应用到新版本。

## 5. 候选推进范围

用户已选定 C2 优先推进，以尽快形成可展示的离线测评结果。
C1 及数据集自动生成后续对齐测评框架确认的契约。
C1 为 Design pending；C2 为 Ready for OpenSpec，当前交接 C2-H001。

### C1：可信数据版本闭环，后续候选

结果：从冻结源材料和已审定题目，形成引用闭合、
证据可回查、切分可检查的不可变数据集版本。

拟 Implement：
- artifact-store：本地产物写入、摘要校验及读取。
- review：人工审定文件的导入与合法性校验。
- dataset-registry：用例与证据校验、revision、发布状态与冻结导出。
- dataset-builder：冻结输入及已审定材料的导入整理子集。
- CLI：上述流程的必要命令入口。

Reuse：目前没有已验证可复用的项目实现。
已有详设属于设计输入，不计作实现复用。

Defer：自动合成、复杂解析与视觉提取、在线评测、
系统调用、评分报告及审定 UI。

### C2：导入运行记录后的离线评测闭环，当前选定

结果：
从已有数据集、A/B 运行记录和可用标注出发，
形成 Markdown 报告和 JSON 结果，包括配对差异、
失败样例、指标覆盖、缺失原因及 pass/fail/inconclusive 门禁。

Implement：
- artifact-store：本地不可变产物、引用、授权与完整性校验。
- review：已有审定记录的契约、校验读取及 revision 绑定。
- dataset-registry：已有数据导入、版本冻结、引用与 split 校验；
  不要求先完成数据生成或完整审定平台。
- adapters：标准记录和一种明确映射配置下的 import_run。
- evidence-mapper：文本 quote 定位、范围和不确定状态。
- evaluators：确定规则及已有审定判断导入；
  自由文本主张判断缺少有效评分来源时保留不可测状态。
- metric-planner：适用性、依赖与可计算性解析。
- runner：计划、导入、评分、attempt、缓存与恢复。
- reporting：配对统计、切片、覆盖、样例、报告与门禁。
- CLI：上述流程的必要入口与机器可读结果。

Reuse：
当前没有已验证可复用的项目实现。
已有文档属于设计依据；第三方库复用另行检查兼容证据。

Defer：
- dataset-builder 的自动生成、独立求解与生成质检流水线。
- 完整人工队列、审定 UI 及多人审批平台。
- 在线系统调用与实时追踪集成。
- 新增模型裁判、视觉映射、Oracle 与高级诊断。
- Web 应用、通用 DAG、远端对象存储和大规模并行调度。

边界：
- C2 通过真实本地存储和标准文件导入验收。
- 评分替身仅验证协议，不作为真实质量判断。
- 使用合成材料展示时明确标注演示身份。
- 缺少实际上下文、审定主张或规则时，不补造对应分数。
- 小样本可以展示描述性结果；正式门禁仍需满足
  预设样本、有效覆盖和统计条件。
- 被延后的完整模块不作为必需依赖偷渡进入实现范围；
  上述必要契约和最小读写能力仍需确认并实现。

必要设计集合：
三份项目级设计，以及上述 Implement 模块在 C2 范围内的
公开契约、关键失败与恢复流程、CLI 和集成验收。
提供方契约先于消费者内部设计确认。

推进顺序：
Artifact Store → 最小审定记录与数据契约 → Adapter 运行契约
→ Mapper / Evaluator → Planner / Runner → Reporting
→ 集成流程、需求映射与交接。

当前状态：
Ready for OpenSpec；C2 所需业务模块、CLI、端到端集成和需求验收映射已确认，
内容就绪检查通过，当前交接为 [C2-H001](handoffs/c2-r001.md) / Current。
就绪仅限交接声明范围，不表示应用实现、测试或业务发布通过。

[C2：CLI 契约与离线评测集成](slices/c2-offline-evaluation.md)
已确认，文档修订 r003，设计覆盖 Complete，仅限声明范围。
CLI、集成及验收正文第 1–6 节保持 r001，第 7 节及基线摘要为 r003。
该文档固定 CLI 命令、输入输出、退出码、集成及计划验收；
不表示应用已实现或验收已执行。
替身仅证明其接口范围，不能作为生产集成完成的证据。

## 6. 待决策与边界

以下问题在相关设计单元或候选选择时确认：
- 首个业务域、源文件类型、数据规模及权限环境。
- 首个被测系统及接入能力。
- 本地文件之外是否确需 SQLite 或对象存储。
- 模型服务、裁判版本、人工校准样本与调用预算。
- 业务指标阈值、最低有效覆盖与门禁策略。
- 已有数据集详设中各状态机、字段和发布规则的兼容性。

工程目录、公开导入边界与依赖检查责任已由
[目录设计](01-directory-layout.md)确认；具体工具版本在首次实现时锁定。

上述问题不妨碍规划评审；若某项阻塞具体设计，
在该设计单元中明确阻塞点及受影响范围。

## 7. 交接与验证记录

- 当前交接：[C2-H001](handoffs/c2-r001.md)，交接修订 r001，包 r020，生命周期 Current。
- C2 为 Ready for OpenSpec；C1 保持 Design pending。
- 当前没有绑定的 OpenSpec 消费方。
- C2 需求映射单元保持 r001，文档 r002；C1 映射仍 Pending。
- 本包记录设计及计划验收，尚无应用实现或已执行验收证据。
- 下一步由用户调用交接中的标准 $openspec-propose 提示词；本设计流程不执行提案或实现。
- 当前交接绑定确切文件、单元与修订；相关变更须按影响更新状态并重新审阅。

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
