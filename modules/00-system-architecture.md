# RAG 测评框架 v2：总体架构

- 设计标识：rag-evaluation-v2
- 架构审阅状态：Confirmed
- 架构内容修订：r001
- 确认范围：系统边界、模块职责与契约所有权、依赖方向、
  核心数据约束、主要流程及后续设计边界。
- 模块字段级契约与实现细节：Pending。

## 1. 依据、目标与边界

依据：
- [v2 设计](../doc/RAG测评设计_v2.md)，重点为第 3—8、11—18 节。
- [数据集生成详设](../doc/RAG测评设计_v2_数据集生成详细设计.md)。
- [设计包索引](index.md)。

框架提供三个主要结果：
1. 判断系统是否完成任务，并分别展示正确、完整、支持、
   引用、行为策略、成本与延迟。
2. 在固定数据、权限、评价及预算口径下比较配置。
3. 根据阶段产物与受控干预，形成有证据和结论边界的诊断。

主要参与者：
- 数据工程师与审定人员：构建、核验和发布基准。
- RAG 开发与算法工程师：接入系统、执行实验和分析变化。
- 质量负责人：配置验收条件，审阅报告与门禁结果。
- 外部系统：源文存储、被测 RAG、模型服务及已有追踪平台。

被测系统保留自己的解析、索引、检索及生成实现。
框架通过 Adapter 执行或导入真实运行，根据可获得的数据评价。

七阶段用于报告和诊断组织。循环检索、并行调用、多次生成
以真实 Span 表达；框架不接管被测系统内部业务工作流。

P0 采用有限并发与明确任务状态。
通用 DAG 引擎、任意阶段回放、独立服务集群、完整审定 UI
属于后续范围。

## 2. 运行与部署结构

首版采用 Python 3.11+ 单包和 CLI。
11 个模块是库内逻辑边界，不对应 11 个服务。

CLI 是组合入口，负责：
- 装配配置、凭据引用和具体插件。
- 选择数据构建、实验、重新评分或报告流程。
- 将模块结果转换为命令状态、机器可读原因和产物位置。

主要运行单元：
- 数据构建作业：由 dataset-builder 管理。
- 实验与评分作业：由 runner 管理。
- 对比报告作业：由 reporting 处理已提交产物。
- 人工审定：通过 review 的工作包导出、结果导入衔接。

数据构建作业不依赖实验 runner。
两者可复用内部限流或文件提交工具，但分别拥有任务语义、
状态与恢复规则，避免形成通用工作流平台依赖。

默认使用本地文件产物。
SQLite 可以作为任务或查询索引，对象存储可以承载大文件；
具体选型在目录、横切规则和选定候选中确认。
外部平台通过具体集成接入，不作为核心模块的强制依赖。

## 3. 模块职责与契约所有权

逻辑所有者负责定义、验证及演进契约。
物理存储位置或提交协调者不改变契约所有权。

以下链接是待编写的唯一契约位置；本轮确认归属，
不将尚未编写的契约标记为已确认。

| 模块与契约位置 | 拥有的行为和记录 | 边界 |
|---|---|---|
| [artifact-store](modules/artifact-store/README.md#contracts) | 不可变字节、ArtifactRef、SourceArtifact、内容摘要与产物谱系 | 保存原文或输出不代表内容已审定；hash 不授予访问权 |
| [review](modules/review/README.md#contracts) | ReviewItem、ReviewRecord、分歧与仲裁记录、审定工作包 | 保存决定及其适用 revision；由业务对象所有者应用决定 |
| [dataset-registry](modules/dataset-registry/README.md#contracts) | CorpusSnapshot、EvidenceUnit、TestCase、DatasetVersion，版本校验与发布 | 正式数据和证据的语义权威；冻结内容的修改形成新版本 |
| [dataset-builder](modules/dataset-builder/README.md#contracts) | 构建配置、作业、锚点与事实候选、蓝图、题目候选、验证记录 | 负责生产候选；正式发布通过 registry 完成 |
| [adapters](modules/adapters/README.md#contracts) | 能力声明、SystemConfig、RunRecord、Span、实际上下文和接入协议 | 定义外部运行的标准表示；不读取 gold 决定普通调用 |
| [evidence-mapper](modules/evidence-mapper/README.md#contracts) | EvidenceMapping、定位方法、范围及不确定状态 | 判断证据在指定产物中的对应关系；不裁定完整任务正确性 |
| [evaluators](modules/evaluators/README.md#contracts) | MetricSpec、Judgment、主张提取及规则、模型、人工评分协议 | 明确评价依赖与口径；不擅自重跑被测系统 |
| [metric-planner](modules/metric-planner/README.md#contracts) | EvaluationPlan、依赖解析、适用性和可计算性检查 | 产生任务描述；不实现指标算法或执行外部调用 |
| [diagnostics](modules/diagnostics/README.md#contracts) | 症状、假设、干预计划、Intervention 与有限结论 | 提议和解释干预；干预调用由 runner 控制执行 |
| [runner](modules/runner/README.md#contracts) | ExperimentManifest、实验作业、执行顺序、预算、attempt 与恢复 | 协调执行和提交记录；不重新定义评分与运行记录语义 |
| [reporting](modules/reporting/README.md#contracts) | ComparisonReport、配对统计、切片、GatePolicy 与 GateDecision | 读取已提交记录形成结论；不通过补跑来改变比较样本 |

runner 可以持久化 Adapter、Mapper、Evaluator 返回的记录，
但必须使用其所有者的校验和版本定义。

公共契约归属提供方。
实现可以统一导出 JSON Schema 或类型入口，
该入口不能形成另一套可独立编辑的业务定义。

## 4. 依赖方向

下表中的方向是“消费者 → 提供方”。
包含层级、运行调用顺序和数据流不能据此混为一谈。

| 消费者 | 允许的内部提供方 |
|---|---|
| artifact-store | 无 |
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

具体约束：
- 业务模块不依赖 CLI。
- adapters 不依赖 runner；运行控制信息通过接入参数传入。
- evaluators 不依赖 runner 或 reporting。
- diagnostics 返回干预计划和解释结果，不反向调用 runner。
- review 使用带类型和 revision 的对象引用，
  不反向导入业务模块修改被审对象。
- registry 不依赖 builder，支持导入外部已审定数据。
- reporting 的数据契约依赖不授予其启动系统调用的能力。

本表定义允许边界，不要求每个实现都导入全部提供方。
候选实现范围仍需单独确认。

## 5. 数据、版本与访问约束

### 5.1 原始产物与正式数据

Artifact Store 管理字节、摘要及产物关系。
Registry 管理语料集合、证据和用例的业务含义与发布版本。

源文件 revision、语料 snapshot、数据集 version 分别标识。
地址迁移且内容与身份校验不变，可以只更新访问层；
源文内容改变必须产生新 revision，并检查受影响的 gold。

EvidenceUnit 是正式证据的唯一权威内容。
TestCase 中展开的证据属于固定 revision 的读取或导出视图，
不能形成另一套独立编辑的 gold。

gold 使用源文 revision、固定 quote、范围和消歧信息定位。
待测 chunk ID 只作为运行产物身份。

### 5.2 接收契约与发布规则

基础 TestCase 契约允许接收可答但缺少黄金证据的用例。
相应证据评价显示不可计算，是否满足门禁取决于评价 profile。

数据集生成详设中的严格发布检查用于构建产物的发布 profile：
- 人工核心集要求必要项目具备独立原文审定记录。
- 合成扩展集保留自动验证身份及实际审定范围。
- 声称充分的证据组必须有对应核验依据。
- Oracle 要求适用且已经审定的充分证据。

上述发布要求不能扩展成所有外部导入用例的统一接收门槛。
冻结、人工核验和证据标注完整性是不同属性。

### 5.3 状态、记录与更正

数据集生命周期、候选质量、构建作业、系统 Run、
证据映射和 Judgment 分别拥有状态机。

作业执行成功不代表候选质量通过；
系统返回成功不代表答案正确；
映射未解决不代表证据真实缺失。

源文件、运行输出、评分与人工仲裁保留独立记录。
重新评分生成新 Judgment，不覆盖原 Run 或原 Judgment。

冻结内容的语义修正产生新版本。
撤销访问或删除要求可以使旧内容不可用，
此时保留允许保存的审计标记并说明复现限制。

### 5.4 输入与权限隔离

普通 Adapter 输入采用显式白名单投影：
用户输入、历史、访问范围、业务配置和运行控制信息。

reference、gold 标识、审定结果和评分规则不进入普通系统调用。
Oracle 使用单独的显式干预路径。

原始请求、生产上下文、参考答案和报告分别受访问策略约束。
内容摘要相同不允许跨访问域绕过授权。
外部模型只接收符合出域策略的输入视图。

凭据通过环境或密钥管理引用解析。
manifest 和报告保存可审计的配置身份，不保存秘密值。

## 6. 主要流程

### 6.1 数据构建与发布

1. Store 保存源文及必要派生产物，Registry 固定 CorpusSnapshot。
2. Builder 从固定输入产生事实、蓝图、候选与验证结果。
3. Review 接收需要复核的对象 revision 和必要上下文。
4. 审定决定由 Builder 或 Registry 在其职责范围内校验应用。
5. Registry 检查发布 profile、引用、split、权限和审定范围。
6. 完整发布后才暴露冻结版本，失败的暂存产物不视为已发布。

生成 QA、参考答案和验证记录与待测知识库保持隔离。
新替代证据进入复核与新版本，不修改已执行实验的参考。

### 6.2 实验执行与评价

以下图示为产物流向，模块导入关系以第 4 节为准。

```mermaid
flowchart LR
    D["冻结数据集与配置"] --> R["Runner"]
    R --> A["Adapter 执行或导入"]
    A --> T["Run / Trace / 实际上下文"]
    T --> E["证据映射与评价"]
    E --> J["Judgment 与诊断产物"]
    J --> P["配对报告与门禁"]
```

执行顺序：
1. 校验版本、输入引用、权限、split、预算与指标配置。
2. Planner 根据 MetricSpec 和能力声明进行静态检查。
   必选指标缺条件时，计划不具备门禁资格，
   仍可按声明继续生成诊断产物。
3. Runner 冻结 manifest，按白名单构造系统输入。
4. 执行或导入真实 Run，记录 attempt、成本及实际阶段产物。
5. 根据实际到达的数据再次检查依赖；
   能力声明不能代替字段和产物校验。
6. Mapper 与 Evaluator 处理固定产物，保留所有异常状态。
7. Runner 收集诊断计划，按预算选择干预或生成复核工作项。
8. CLI 将已提交的实验引用交给 Reporting，
   形成配对、切片、覆盖、区间和门禁结果。

同一次链路评价只能使用该次真实执行产生的关联产物。
独立检索或生成消融另建实验，不能拼接为一条真实链路。

### 6.3 重新评分

重新评分读取已存在的 Run 和冻结依赖，
创建新的评价计划及 Judgment。

Evaluator 超时、模型版本变化或 rubric 调整，
不会隐式启动新的业务请求。
引用不同 gold 版本时，报告明确标识版本变化；
正式 A/B 使用一致的评价基线重算双方。

### 6.4 Oracle 与复核回流

Diagnostics 根据已有观察提出干预计划。
Runner 检查 Adapter 能力、权限、原生成配置、
充分证据及输入容量后执行。

严格上下文干预固定 query、history、模板、模型、
输出预算、采样及工具策略，只替换声明的 context。
变化的内容、顺序、长度和成本均被记录。

结果连接 baseline 和 intervention Run，
由相同评价口径判定效果。
fail→pass 只支持“在该次上下文替换条件下恢复”，
不能据此宣布检索是唯一根因。

复核可修正候选、标签或替代证据，但通过新 revision、
新数据集版本或新 Judgment 生效。

## 7. 失败、恢复与结论约束

- 计划缺少必需引用或版本不闭合时，在外部调用前失败。
- 导入记录缺字段时保留实际缺失，不补造 Trace 或上下文。
- 被测服务失败、超时与无效输出属于可观察的系统表现。
- Evaluator 故障属于测量失败，不能自动记作答案正确或错误。
- Mapper 无法确认存在或缺失时保留不确定状态。
- 非 ok Judgment 的 value 为 null，异常原因进入报告。
- 预算耗尽或取消保留已完成结果，并公开未完成范围。
- 重试保留 attempt 和全部已知成本；未知调用结果需要显式处理。
- 缓存绑定输入内容、访问域及完整依赖版本。
- 已提交结果可被恢复流程复用；重新执行业务请求必须可识别。

实际生成上下文必须对应实际发送内容及顺序。
只能取得摘要或脱敏视图时，记录限制并降低可测范围，
不能将重构 prompt 当作原请求。

报告展示适用数量、有效数量、未知和失败原因。
质量分、未知导致的识别范围、统计置信区间分别解释。

门禁为 pass / fail / inconclusive。
必选测量覆盖、标签、校准或样本不足不能自动通过；
已核验违反硬性权限约束可直接失败。

## 8. 关键选择与后续决策

| 选择 | 理由与代价 |
|---|---|
| 单包和 CLI | 便于运行可审计闭环；未来独立服务需保留相同契约边界 |
| 契约由提供方拥有 | 减少循环依赖和重复定义；统一 schema 导出需遵守所有权 |
| Run 表示归 adapters、作业归 runner | 接入层可独立标准化外部记录，执行器集中管理预算与恢复 |
| 评分与业务执行分开 | 支持改 judge 后重新评分；需要完整保存运行产物及依赖 |
| Builder 与实验 Runner 分开 | 保持候选质量和系统执行的不同语义；共享工具只复用内部能力 |
| 默认文件产物 | 提供可移植审计基础；并发索引和大对象能力按实际需要补充 |
| 严格接收与发布规则分层 | 保留外部数据兼容性，同时明确核心集的更高可信要求 |

后续确认位置：
- 目录与导入检查：01-directory-layout.md。
- 错误分类、提交、幂等、并发、重试、配置、
  审计、兼容及恢复规则：02-cross-cutting-concerns.md。
- 字段、操作、精确状态转换与模块验收：各模块 Contracts 及详细设计。
- 首个业务域、文档类型、Adapter、预算和业务阈值：候选范围确认。
- 第三方平台复用：取得具体契约与实现兼容证据后确认。

当前没有阻塞上述架构边界的未决问题。
业务阈值、部署规模及具体插件仍待确认，
因此本架构确认不使任何实现候选自动就绪。

## 9. 后续验收设计要求

后续验收应覆盖以下可区分正确与错误实现的案例：
- 改变 chunk 粒度后仍使用原 gold，重复来源不增加事实覆盖。
- 相同数字但年份、单位或否定不同，不能被模糊定位静默接受。
- 候选中有证据、实际生成输入中被截断，两阶段结果可区分。
- 缺少实际上下文时，支持度指标显示不可测。
- 服务超时与裁判超时形成不同的结果与门禁影响。
- 改 judge 可以重新评分，普通系统调用次数不增加。
- 新替代证据在共同新基线上重算双方，旧实验记录保持可审计。
- hash 相同但访问域不同，读取与缓存复用仍检查权限。
- 发布中断后不会暴露半成品，恢复不会静默覆盖已提交记录。
- Oracle 恢复仅产生受限干预结论。

以上是计划验收要求，尚未执行测试。
具体输入、断言和验收责任在选定模块与集成设计中确认。
