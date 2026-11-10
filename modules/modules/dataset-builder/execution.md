# 执行与恢复：构建作业、预算、缓存和外部未决

- 状态：Confirmed；内容修订：r001；用户授权代理连续完成。
- 上级契约：[README / C3、C10–C12](README.md#contracts)。
- 本作业由 Builder 拥有，不使用实验 Runner 调度业务生成。

## 1. 作业与任务身份

操作种类 operation_kind=build/prepare_publication/publish。
OperationHandle={builder_id, store_id, access_domain, operation_kind,
operation_key, request_digest, operation_ref}。
调用方提供非敏感 operation_key，派发任何任务前先保存。
同作用域/种类/键、同请求可恢复；异请求 operation_conflict。
get_build 必须给 operation_kind，不按“最近一次构建”猜测。

BuildJob 状态 pending/running/succeeded/failed/timed_out/cancelled；
结果另记 complete/partial、stop_reason 和 quality_outcome。
quality_outcome 可为 candidates_available/waiting_review/no_eligible_candidates/
publication_blocked/published；不得从 succeeded 推导合格。

StageTask={task_id, stage, input_refs, config_digest, provider_ref,
schema/template/rule_versions, batch_or_slot_id, dependencies, limits}。
task_id 是上述逻辑请求规范摘要。
attempt_id=(operation_id,task_id,attempt_number) 表示实际执行。
真实多个合成样本使用不同 sample_slot；缓存/网络重试不增加独立样本数。
每项 TaskOutcome 保留 planned/started/succeeded/failed/timed_out/
cancelled/not_started/unresolved，附结果或原因。
所有已纳入计划的候选/批次/槽位必须有处置，不能丢失败数。

阶段 S0–S10 的输入输出见集成切片。
每批扩展先提交 BatchIntent 与任务列表，再请求 provider。
恢复时不能把同一批意图再次抽样生成另一组任务。
同输入模型重新采样要新 slot/profile，不能覆盖原响应。

## 2. 本地布局与提交协议

local root 绑定固定 builder_id、store_id 和 storage_format_version=1。
拟定布局：

```text
builder-root/
  builder.json
  root.lock
  operations/<kind>/<scoped-key-digest>/
    operation.json
    coordinator.lock
    events/<sequence>-<event-digest>.json
    requests/cancel-<id>.json
    handoff/<attempt-id-digest>/
  staging/<internal-uuid>/
```

业务内容、原始响应、预算账本、检查点和快照存 Store；
本地文件只保存固定引用发现、进程协调和 worker 交接，
不得成为第二套可编辑候选或正式版本权威。
所有外部 ID 先摘要成安全路径；拒绝内部 links/reparse points；
暂存与提交在同一 NTFS 卷，不把网络盘视为已验证运行环境。

初始化 create_new 只创建空新身份，已有残留/不完整根拒绝覆盖；
open_existing 不自动重建身份。
root.lock 用于创建操作/短元数据更新；
每操作 coordinator.lock 是 OS 排他锁，不用 PID/文件年龄判断有效。
当前 coordinator 活跃时另一执行/恢复请求 operation_busy。
查询与取消通过不可变请求/检查点，不夺走活跃协调权。

锁顺序固定 Builder root 短锁（不嵌套慢操作）；
Builder operation 锁 → Registry 公开操作 → Store 公开操作。
Review/Registry/Store 不回调 Builder；不同时持有两个 operation 锁。
Store 的锁语义仍由提供方负责。

每步提交：
校验当前授权与固定输入 → 记录 CallIntent/BatchIntent →
计算或 provider 调用 → 先存 raw/输出 →
验证并提交 StageResult manifest → 追加不可变预算/结果事件 →
提交 BuildSnapshot → 保存回执。
本地事件使用临时文件刷新、校验后条件提交；sequence 连续且父摘要相连。
残留 staging/worker 文件不是已完成业务结果。
进程终止持久性需集成验证；不承诺整机掉电零丢失。

## 3. 网络调用、worker 和未决结果

每操作一个协调者、一个在途 worker；worker 使用 Windows Job Object，
退出/取消时有界清理，所有子进程随受控作业管理。
协调者拥有提交权；worker 只写隔离 handoff，
输出经协调者重新授权、校验才进正式 Store。
worker timeout 不允许晚到结果覆盖已经提交的选择。

CallIntent 至少包含 invocation_ref、task/attempt、request_digest、
provider binding、egress policy、预算预留、deadline、
idempotency/query 能力。发送前完整提交。
CallOutcome 包含 accepted/failed_before_send/completed/remote_unknown、
原始结果引用、provider request ID、真实 usage、传输和超时事实。

区分：
- 请求尚未发出且可确认：可按固定策略重试。
- 明确可重试响应且协议允许：受原预算/deadline 和次数限制。
- 超时/连接断开/进程崩溃但可能发出：remote_unknown。
- 结束本地 worker 不能证明远端停止或没有计费。

remote_unknown 停止该分支的新调用，冻结保守资源预留，
产出 partial 快照及 unresolved reason。
不宣称外部调用 exactly-once，也不将它标成未执行后静默重发。

reconcile_invocation 接收固定 ResolutionRequest：
invocation_ref、resolution_kind、provider_evidence_ref、operator_reason、
acknowledged_unknown_cost、replacement_policy_ref（可空）。
resolution_kind：
- recovered_result：通过已声明查询能力取得同 provider request 的固定结果。
- confirmed_not_executed：可靠 provider 依据，允许按原剩余策略再派发。
- abandoned_unknown：明确放弃该分支；仍保留未知计量，不能回收成零。
若要新生成，另建新 slot/request，保留原 unresolved 和全部风险。
本地声称“没看到输出”不是 confirmed_not_executed 依据。
协调者不接受包含未知/变更输入的晚到结果作为旧任务完成。

退避/格式修复/内容重生成分开记账；
格式修复也可能产生额外真实模型请求，必须预留和记录。
HTTP 客户端/SDK 默认 retry 必须关闭，不能绕过调度账本。

## 4. 配额、计量和停止

预算池 extraction_vision、synthesis、solver、validation、repair、
human_review；每池单独 request/token/image/pixel/cost/human_limit。
每任务和每构建还有限次数、候选数、最大批数、deadline。
max_output_tokens 只限制响应配置，不证明 provider 一定按本地计费估算。

派发前按可证明上限预留，完成后以实际 usage 结算：
known_cost、reserved_upper、unresolved_exposure、unknown_usage 分开。
无可靠计价或输入 token 上界时，金额预算只可作为观测约束，
不得宣称硬金额保证；要求 hard_cost 时在 plan 阶段 blocked。
输入 token 估计、响应输出上限与文本字节限制各自记录，不能互相替代。

外部结果未知时保留预留上界；
没有合法上界则 hard_cost 模式不可继续。
供应商幂等返回同一计费请求按 request ID 去重计量，但保留本地 attempts。
replay/cache 本次新增调用数为零，历史来源费用另列，不算重新发生。

成本摘要分 generated/auto_validated/published 和 tier，
分母为零显示 null，未知成本存在则不显示“完整总成本”。
取消/超时/失败费用保留；人工定向/随机/全审工时分开。
不把外部人工工作时长与机器作业耗时混成 E2E 性能。

deadline 在恢复后不重置。
运行中使用单调时钟，持久化预算/UTC 锚和累计用时；
重启后无法保守核验剩余时间则 clock_uncertain，停止自动继续。
Ctrl+C 先停止派发、提交取消意图、有限清理、保存 partial；
取消回执仅表示请求已提交，直到 worker 确认结束才能说已停止。
cleanup_unconfirmed 保留未决范围，禁止同时启动替代 worker。

## 5. 精确复用和依赖失效

复用只从 BuildConfig.reuse_source_refs 和当前已提交检查点，
不搜索全局“最新”“最高通过率”结果。
键包含访问域、源/snapshot/表示、全部输入内容、stage、slot、
model/provider、template/schema、规则/normalizer、权限/时间、
允许工具/seed/采样参数及转换器版本。
当前授权与完整字节核验必须再次通过。

结果完整并可消费时才可复用：
提取成功不认证事实；验证输出复用不转成人工；
按质量筛选生成器多次输出须遵循固定 selection，不能挑最佳样本。
缓存 key 的完全相同不能证明生成分布有多次独立抽样。

DependencyImpactManifest 为具名输入/输出边表，
源修正追踪到事实、关系、蓝图、候选、解答、验证、审定视图及发布投影。
模型只用于某个 checker 更新时不重提取原文。
旧发布数据不就地 invalid 改写；输出修订/撤回建议及新版本关系。

## 6. 恢复与发布幂等

恢复首先：
取得原操作锁 → 验证 builder/store 身份 →
验证连续事件及最后完整检查点 →
重新读取当前所有固定引用及授权 →
核验请求摘要/策略/预算 →
逐 task 处理已提交、未开始、明确失败和 remote_unknown。
本地发现指针损坏时返回 checkpoint_corrupt，不能猜一个快照续跑。

如果 raw response 已存但 StageResult 未提交，
只在 intent、来源、数量和依赖全部匹配时重做确定性解析，
不再次发模型请求。
StageResult 已提交而检查点缺失时，可以根据完整 intent/outcome
补建检查点；不能仅凭某文件存在认定完成。

prepare_publication 和 publish 使用不同操作键与固定请求。
Registry 已正式发布但 Builder 回执未写时，
按准备中明确的 dataset/version 和正式记录重新核验，
复用相同版本并补回执；不新造版本重复发布。
同键换 selection/review 批次/profile 为 operation_conflict，
应新建请求；这不是“恢复”。

## 7. 验证计划

| 场景 | 预期行为 |
|---|---|
| 同键并发/异输入 | 分别 busy/conflict，不双派发 |
| 发出前崩溃 | 只有可靠未发送依据时重派发 |
| 请求已到达但无响应 | remote_unknown，保留费用与预留 |
| raw 已存而解析前退出 | 恢复解析原响应，不重调用 |
| SDK 打开隐藏 retry | 接入验收失败 |
| timeout 后 worker 晚回结果 | 不覆盖选定结果或再占一个槽位 |
| 恢复时 deadline 已耗尽 | partial，原上限不重置 |
| 当前授权撤销或 checkpoint 损坏 | 明确失败，不空输入重跑 |
| 改 checker/改源事实 | 分别局部/传递失效，不覆盖历史 |
| 同 cache 复用两次 | 新增模型调用数为零，不增独立样本 |
| Registry 发布完成但回执失败 | 恢复同 version，生成新回执 |
| 作业全部跑完但候选全 reject | 作业 succeeded，合格数 0，发布 blocked |

文件锁、Job Object、进程回收、NTFS 目录提交与安装包需在目标 Windows 验证。
真实费用、远端查询和取消能力依赖配置的 provider，不由本地演示证明。
