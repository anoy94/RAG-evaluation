# C1：数据集生成、审定发布与 C2 对齐

- 候选：C1；审阅状态：Confirmed；内容修订：r002。
- 单元修订：第 1–6 节业务/验收 r001；第 7 节及导航 r002。
- 设计覆盖：Complete，仅限声明范围；就绪状态：Ready for OpenSpec。
- 2026-11-10 用户授权代理完成模块包和相关集成，无逐节人工复审。
- 本文定义待实现行为和验收，不表示真实生成/人工/发布已完成。

## 1. 结果、范围与依赖处置

成果：从固定源文件/参考提取物生成可审计题目，
完成合成、独立求解、规则与语义检查、分组去重、
人工文件审定或自动扩展发布，取得可由 C2 读取的 DatasetVersionRef。

C1 从原“已审定材料导入”子集扩展为完整生成侧设计，
保留 curated_import 作为最小可运行/离线验证路径。
generate 使用明确的模型 provider；发布不依赖某个模型必定生成正确题。

| 处置 | 精确范围 |
|---|---|
| Implement | Builder README 与 evidence/planning/synthesis/quality-release/execution 五份主题 r001；本文件的 CLI 和集成 |
| Implement 依赖 | Store Contracts r002、Review r001、Registry r002 在已声明本地范围内的公开操作；目前只有设计，没有已验证业务实现 |
| 依赖复用计划 | C2 若先实现这些提供方，先检验相同契约和真实测试再复用，不能凭模块名直接称兼容 |
| 第三方复用 | 严格 JSON/YAML、类型校验、HTTP、Windows 协调库；具体版本实施时锁定；统计推断不在 Builder 重写 |
| Defer | 原格式全栈 parser/OCR/视觉服务、embedding 去重服务、动态工具、Web 审定平台、签名认证、分布式执行、远端存储 |
| Defer 能力扩展 | C2 尚不支持的乘除数值 profile、视觉定位评价、多模态/工具历史；保留候选与兼容缺口，不伪造已可测 |

首版具体提取为 utf8_text_v1 和 reference_extraction_jsonl_v1；
模型为 recorded_model_response_v1 和 chat_completions_json_v1。
参考导入不证明原格式 parser 正确；模型重放不证明真实生成质量。
未配真实 provider 时应明确 blocked 或使用 curated_import/replay 演示，
不能显示“已调用生产生成器”。

C2 设计和原行为不变；新集成只经 Store/Review/Registry 公共契约。
C2-H003 为 Current；C2-H001/H002 均 Superseded。
本次 C3 设计只同步导航，C1/C2 原业务与验收单元保持原修订。

## 2. 装配与配置

沿用项目目录/横切规则的配置优先级、路径解释、未知键拒绝和安全日志。
Builder 采用单独 --build-runtime <file>，不向现有 C2 RuntimeConfig
塞入未定义字段，保留旧命令的输入与退出行为。

BuildRuntimeConfig 使用 profile=cli.builder-runtime.c1.v1，包含：
workspace、现有 Store/Registry 的公开配置、Builder local_config、
current_access_policy_ref、model/提取内置 provider 配置、
credential reference、egress policy、export root 白名单及 CLI 限额。
非秘密配置可用显式 --workspace 和规定的 RAG_EVAL_BUILD_ 白名单覆盖，
秘密不通过业务 --input 接受。
JSON 请求默认限 8 MiB、深度 64；所有者更严限额优先。

业务 request 一律 --input <UTF-8 JSON>，不接受嵌入脚本/任意模块路径。
大内容先用已有 artifact put/register-source 转为引用，
Builder 不读取模型/文件内容声称的路径和 URL。

装配 Store+当前授权 → Review → Registry →
Builder 的提取/模型 provider 注册表 → Builder。
不创建实验 Runner、Evaluator、Reporting 或 Diagnostics。
C1 集成测试在组合层另行接 C2，Builder 不反向导入它们。

## 3. CLI 命令和结果

新命令均在 rag-eval build 与 python -m rag_eval build，
不改变既有 workspace init --component store|registry|runner 或 job 命令。

| 命令 | 输入/控制参数 | 所有者调用及输出 |
|---|---|---|
| build init | --build-runtime；明确 --workspace | Builder create_new，只初始化 builder；依赖须已存在 |
| build plan | BuildRequest | plan_build → 固定计划及 blocker/能力限制 |
| build run | plan_ref；必需 --operation-key | execute_build → 明确快照及操作句柄 |
| build status | operation_key、operation_kind | get_build，只读 |
| build resume | 完整 OperationHandle | resume_build，原策略/预算/deadline |
| build cancel | OperationHandle、reason | cancel_build，取消请求回执 |
| build reconcile | handle、resolution_request_ref | reconcile_invocation，不静默重发网络请求 |
| build revise | change_request_ref | revise_candidates，新 revision 与影响清单 |
| build prepare-release | SelectionRequest；必需 --operation-key | prepare_publication → ready/blocked、projection/draft 引用 |
| build review-package | projection_ref、ReviewRequest | create_review_package → 真实 Registry 目标和工作包 |
| build review-export | package_ref；显式 --output-root | export_review_package，受控显式本地导出 |
| build publish | build_preparation_ref；必需 --operation-key | publish_dataset → Registry 正式 version_ref 及两级回执 |

人工回复仍经既有 review import 导入，不新造 Builder-owner ReviewRecord。
发布后的数据导出使用 Registry.export_dataset 公开操作；
本轮不新增兼容性不明的导出格式。

BuilderCommandResult 使用 profile=cli.builder-result.c1.v1：
schema_version=2.0、command、command_invocation_id、command_status、
outcome、result_refs、operation_handle、error、limitations。
字段含义沿用既有命令结果规范，业务引用保留所有者完整 DTO；
不构造质量 gate，不把 Builder 质量状态转换为 C2 GateDecision。

JSON stdout 恰好一个结果和换行，进度/worker 输出到 stderr，
不输出完整源文、答案或凭据。
退出码：0 命令目标完成；2 输入/计划或发布 blocked；
3 运行/权限/存储/完整性失败；130 用户中断。
本组命令不产生 gate fail/inconclusive，因而不使用 4/5。
合法 build complete 但候选全 reject 可以返回 0，
必须显示 eligible=0；后续 prepare/publish 不能返回成功。
预算停止有可信 partial snapshot 时 run 可以 0 并突出 partial，
没有可信快照则为 3。
请求取消成功不等于远端已停止，展示真实当前 state。

## 4. S0–S10 集成与提交

| 阶段 | 输入 | 输出与完整性条件 |
|---|---|---|
| S0 冻结 | 明确源清单/权限/版本 | SourceBundle，全源处置闭合 |
| S1 提取 | 原始引用或固定提取批次 | ReferenceBundle、partial/failed 范围和资产 |
| S2 事实 | 固定锚点/人工更正/模型建议 | 限定完整 facts、来源和疑点，不冒充 gold |
| S3 蓝图 | 事实与核验关系 | 目标/推导/支持路径与必要性 |
| S4 切分采样 | 场景策略、provenance graph | 固定 group/split、批意图及配额可行性 |
| S5 合成 | scenario 与生成视图 | Candidate、约束、调用来源 |
| S6 求解 | 独立 SolverView | claims、规则、可答性、来源与分歧 |
| S7 质检 | 同一候选 revision | 各适用检查及晋级/修复/未知 |
| S8 去重 | 全相关候选及关联 | 去重/跨 split 冲突及新 group_version |
| S9 审定 | Registry 草稿真实目标 | 完整 Review 批次、实际范围及分歧处理 |
| S10 发布 | 固定选择、配额、审定 | Registry 准备/正式 DatasetVersionRef |

S9/S10 由显式 publication 流程进入，
build run 不自动等待人工无限运行，也不自动 publish。
先形成工作候选快照，再 prepare-release 得真实草稿/缺审定 blocker，
之后 review-package → 外部人工/自动记录 → review import →
新 prepare-release → 显式 publish。
自动扩展也必须绑定真实目标及对应逐题检查，
不能略过 Registry 所有者的等级计算。

每个 Provider 提交自己的不可变产物，跨阶段无文件系统事务。
下游失败保留已提交结果和安全恢复入口。
C2 对齐：
open_dataset(evaluation) → 读完全部成员/引用 →
project_system_input 不含 gold →
以同一正式 version 导入 A/B/评分并报告能力缺口。
评测不成功不能反过来静默修改题库。

## 5. 可手算基础夹具

小型原始材料含同一虚构实体：
2024 年计划 100 万元、实际 120 万元；
2023 年实际 120 万元；
固定表格另有“利润未披露”，与 0 区分。
各事实具有精确源 quote、期间、单位和 plan/actual 限定。

基础蓝图问“2024 年超额多少”，答案差额 20 万元，
前提为计划/实际，必需答点只有差额。
增补直接差额披露后存在单来源替代路径。
最小负例限定到完整字段表的未披露利润，
另有“自然全库检索无命中”的 unknown 对照。
变体、跨年/否定对照、会话和同源题均保留关联组。

夹具中人工记录为协议测试材料，标 fixture_only/local_import_only；
不能当成真实人审、代表性或生产发布证据。
真正模型接入验证另配获授权 endpoint 与输入，
不把 replay 的通过率写成模型生成准确率。

## 6. 端到端验收计划

以下全部尚未执行。安装后的包、真实本地 Store/Registry/Review/Builder
是主路径；替身用于精确故障位置，不能替代整个真实集成。

| 编号 | 场景 | 必须观察到的结果 |
|---|---|---|
| G01 | 安装后两入口、help、初始化 | 行为相同，无 import 自动创建/调用 |
| G02 | 基础文本到正式数据 | 来源/题/证据/规则/成员闭合，20 可手算 |
| G03 | 原文件损坏、缺页或提取 partial | 明确错误/范围，数量不静默缩减 |
| G04 | 120 同值不同年份、否定或单位 | 不合并事实或误过约束 |
| G05 | 表格未披露、0 和合并表头 | 缺失类型与单位/表头完整 |
| G06 | 视觉区域无审查、OCR 与图冲突 | 保留疑点/asset 定位，不能伪 bbox gold |
| G07 | 直接披露差额替代原两跳 | 记录另一充分路径，不强认两跳必要 |
| G08 | 变体、对照和会话跨 split | 同组或隔离，不通过拆组制造独立性 |
| G09 | 只有几个来源却请求大配额 | 硬缺口可查，不用多问法充独立样本 |
| G10 | 输入遍历顺序变化 | 固定选择、子 seed、代表题不变 |
| G11 | 求解视图夹带生成器答案 | 拒绝独立性声明，实际输入可追 |
| G12 | 外部模型未配/不支持视觉 | plan blocked 或显式离线模式，无静默降级 |
| G13 | 无命中 vs 封闭负例 vs 发现全库答案 | unknown/核验负例/修订可答分别表达 |
| G14 | 除零、单位换算、百分比与百分点 | 合法精确运算或未知/阻塞，不执行代码 |
| G15 | 乘除规则投影到 C2 | unsupported_publish_profile，不删规则过关 |
| G16 | 质检 fail/unknown/error/不适用 | 状态和适用分母分开，不记默认通过 |
| G17 | 自动通过池与抽检少量人工 | 只提升实际全审题，未审不成核心 |
| G18 | Registry 真实 target、旧视图/冲突审定 | 版本绑定检查，不能复制旧 target |
| G19 | 缺人工或硬配额 | prepare-release blocked，合法草稿仍可审 |
| G20 | 共享 OCR/模板错误 | hold 整个受影响 slice，不只替换坏题 |
| G21 | 付费请求发出后响应丢失 | remote_unknown，恢复不自动重发，费用未知保留 |
| G22 | raw 已存、提交/回执中断 | 复用确定结果并补回执，不重复调用 |
| G23 | 同键并发、同键改输入 | busy/conflict，无双协调者 |
| G24 | 超时/Ctrl+C/清理不能确认 | partial 和未决可查，不宣称远端已停止 |
| G25 | 恢复 deadline/预算、未知费用 | 不重置上限，不能用 unknown=0 继续 |
| G26 | 撤权、出域拒绝或缓存旧权限 | 当前授权失败，无正文泄露或缓存绕过 |
| G27 | 改 query/事实/checker | 依赖正确失效，旧审定不跨内容复用 |
| G28 | Registry 发布后 Builder 回执失败 | 恢复同一正式 version，不重复发布 |
| G29 | 工作包导出路径穿越/junction | 拒绝、无越界/异内容覆盖 |
| G30 | 新 gold/替代来源/撤回 | 新版本影响清单；旧实验和内容不回写 |
| G31 | 新版本进入 C2 | 全部成员可读、普通输入无参考、原评测契约不变 |
| G32 | 全 reject/预算停止及 JSON 输出 | 真实 outcome/退出码、完整计数、单个 JSON、无 gate 假通过 |

每个实际记录包含 C1 需求 ID、G 编号、固定输入/策略/provider 版本、
命令/退出码、摘要/数量、故障点、Windows/NTFS 环境及限制。
真实 HTTP 子集、Job Object、文件锁、目录提交和打包需要目标环境证据。
外部模型有效性、真实人工/业务质量、视觉 parser 准确性另行验证。

## 7. 状态、设计充分性及实施权限

本节与导航修订 r002；第 1–6 节行为和 G01–G32 不变。

Builder 各部分和必需提供方契约已确认；
C1 需求映射及 [C1-H002](../handoffs/c1-r002.md)精确绑定设计，
因此 C1 为 Ready for OpenSpec，仅表示可以生成提案。
真实源数据、endpoint/凭据、业务配额、人工预算和阈值是运行输入，
缺失时按 blocked/unknown 处理，不阻塞此模块的工程设计。

公开接口、数据含义/归属、依赖、持久化、权限、关键状态及验收变更
须开发者确认；契约保持的私有函数/类/文件组织由实施者决定。
按用户指示，本轮设计直接完成并落盘；这不授权未来实现者随意改契约。
