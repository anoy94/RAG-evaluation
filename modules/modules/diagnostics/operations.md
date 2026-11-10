# Diagnostics：产物、恢复、安全与验证

- 状态：Confirmed；修订：r001；对应 README C1、C9–C10。
- 首版为无业务网络调用的本地确定性模块。

## 1. 读取、配置及可复用边界

request/config/所有依赖均使用固定引用；可信 CLI 解析路径并存入 Store。
业务内容不提供任意文件路径、Python 模块路径或代码执行。
RulePack/转换器由安装包静态注册，版本固定。
限制配置在处理前核验；总量超限明确失败，不按“最重要”悄悄丢规则或病例。

精确复用键包括 store/access_domain、完整 request、
实际源/数据/gold revision、Run/call/stage/内容与顺序、
association、Judgment/Mapping/审定范围、完整源批次选择、
规则实现、provider profile、处理限额和 schema。
仅 query/fact_id/Run ID 不足。当前授权和完整性始终重核验。
diagnose/plan/assess 的重算不增加任何业务调用。

## 2. 持久化与可见性

所有内容通过 Store；本模块无共享可变数据库、全局结果缓存或追加式 JSONL。
逻辑产物为 request、observations、finding、selection、eligibility、
intervention_plan、trial_assessment、batch_manifest、receipt；
不是要求调用方自行管理文件布局。

提交流程：
1. 写内容对象及所依赖的固定引用。
2. 构造完整 binding/计划/slot 清单，核验每项唯一、计数及状态闭合。
3. 提交含 request/依赖/版本/所有输出/计数的 immutable batch_manifest。
4. 再提交 operation receipt（主体、UTC、策略 revision、完成阶段）。

manifest 未提交：子对象不能作为完整批次；
manifest 已提交而 receipt 失败：返回 storage_failure、
committed_manifest_ref 及阶段，不声称未产生结果。
相同输入重试复用精确内容并生成独立回执。
损坏的已提交结果报错，不用同 ID 重算覆盖。
并发提交由 Store 不可变写入协调；权限撤销后不因已缓存而继续读取。

外部试验执行状态由 Runner 管理；Diagnostics 不保有正在运行状态。
新 Judgment、gold 或协议证据产生新 envelope/assessment，
通过 derived_from 关联旧版本；历史结论不悄悄更新。

## 3. 权限、出域与显示

StoreAccessContext 仅授予测评进程存储权限；
业务 access_scope、allowed_tools、数据出域是另一层必要条件。
资格判定使用可信配置中冻结的授权事实/策略版本；
在每次生成前由 Runner/Adapter 重新核验实际权限。
不允许以管理员能读 gold 为由向无权限业务调用注入答案。

源文、上下文、模型返回均是数据；不得作为控制指令执行。
严格 Oracle 必须保存真实载荷用于核验，但日志只输出安全 ID、状态和原因。
报告可采用有版本的受控展示材料；脱敏视图不得冒充实际请求。
无权读取时错误不泄露 gold 文本、对象存在性或其他访问域位置。

本地能力不提供签名认证、多租户强隔离或日志不可伪造保证。
credential_ref 只交具体 provider，由可信组合入口注入，不进入诊断正文。
文本不包含外部链接自动抓取或资产下载行为。

## 4. 复核回流

Diagnostics 只生成 ReviewSuggestion：
target_owner、target_ref/revision、issue_code、受影响 scope、
evidence_refs、建议补查项及依赖的 Finding/Assessment。
这是本模块的建议 payload，不是 ReviewRecord 或 ReviewItem。

组合入口根据真实对象所有者调用 Registry/Evaluator 等公开 build_review_target，
再走现有 Review/发布协议；Diagnostics 不直接创建审定目标、
不依赖 Review、不改 source_reviewed 或判断值。
新的替代证据需 Registry 新版本；按共同新基线重评分相关对比。
新测量证据可只重评分/重诊断，不能隐式重跑业务。
未来自动化回流须另行设计权限与审批，不把建议当已修复。

## 5. 测试、部署与性能

私有实现可按观察归一、规则、计划、协议检查和持久化组织，
公开只用 contracts/api。单元测试应检验不同证据条件产生的不同结论，
不为每个 DTO getter 写镜像测试。
整体工作量与请求的已声明对象/字节/规则匹配数量有界；
不承诺未测量吞吐或准确率。

计划验证层次：
- 契约：身份、语义状态、错引用、完整数量、稳定排序与规则版本。
- 算法：阶段链、三值依赖、充分组、资格、转移与分母手算。
- 真实本地集成：Store/Registry/Adapter/Mapper/Evaluator/Runner/Reporting，
  Windows NTFS 中断、撤权、恢复及 CLI。
- 受控 HTTP 服务：检查实际载荷/顺序、超时、未决、无隐藏重试；
  是协议测试，不能证明真实模型效果。
- 实际业务：固定 provider/模型/数据/人工或有效裁判，
  再测诊断复核一致性、干预覆盖与稳定性。

端到端计划见 [T01–T32](../../slices/c3-diagnostics.md)。
发布为同一 Python 包，不增加 daemon、容器、Web 或数据库迁移；
核心旧 profile 必须继续可读。新 profile 未被旧版本支持时明确拒绝，
不能去掉字段后强行降级。当前应用和真实业务验证证据为零。
