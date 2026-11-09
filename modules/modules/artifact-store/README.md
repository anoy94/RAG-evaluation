# Artifact Store：产物存储

- 模块 ID：artifact-store
- 文档状态：Confirmed
- 文档内容修订：r002
- 概述状态：Confirmed / r001
- 详设覆盖：Complete
- 当前候选：C2，运行记录导入与离线评测闭环。

## 1. 职责与设计边界

依据：
- [总体架构 r001](../../00-system-architecture.md)。
- [目录与依赖规则 r001](../../01-directory-layout.md)。
- [横切规则 r001](../../02-cross-cutting-concerns.md)。
- [v2 设计](../../../doc/RAG测评设计_v2.md)，第 4、5、7、16 节。

本模块拥有：
- 不可变字节产物及完整性校验。
- ArtifactRef 的身份与读取协议。
- SourceArtifact 的元数据和派生关系。
- 本地产物的提交、重复写入、访问检查与恢复行为。

本模块隐藏内容寻址、存储路径、流式写入、提交和校验复杂度，
调用方只处理受支持的引用与操作。

本模块不解释 TestCase、Run、Judgment 的业务含义。
这些对象由各自所有者校验，再将序列化内容作为产物保存。

保存源文不代表源文已审定；
保存上下文不代表上下文足以作答；
保存评分不代表评分有效。

C2 使用本地文件后端。
对象存储、远端下载、自动解析、压缩分层和垃圾回收
属于后续扩展，不进入本轮实现范围。

## Contracts

契约状态：Confirmed。
契约修订：r002。
适用范围：C2 本地产物写入、读取、身份及谱系记录。

### C1. 存储身份与权限上下文

store_id 标识一个逻辑存储实例。
首次初始化后持久化，重新打开同一实例不生成新身份。

本地目录迁移且内容与身份保持不变时，可以保留 store_id。
独立副本不得在没有协调机制时以同一身份分叉写入。

access_domain 是不含敏感业务明文的访问域标识。
本轮禁止跨访问域物理去重，避免通过复用内容绕过访问控制。

StoreAccessContext 包含：
- principal_id：本次操作主体。
- readable_domains：允许读取的访问域。
- writable_domains：允许写入的访问域。
- policy_revision：本次使用的访问策略版本。
- operation_id：用于审计关联的操作标识。

上下文由可信组合入口依据运行配置构造，
不能从外部 Run、TestCase 或模型返回中直接接受权限声明。

StoreAccessContext 表示测评进程的存储权限，
与被测用例模拟用户的 access_scope 分开。

权限必须同时满足上下文与存储侧策略。
读取、写入、元数据查询和重复命中都执行检查。

本地实现依赖操作系统权限保护工作区，
不宣称隔离能够修改工作区文件的恶意本地程序。

### C2. ArtifactRef

ArtifactRef 表示不可变字节，不携带文件系统路径或临时 URL。

| 字段 | 约束 |
|---|---|
| schema_version | 本版为 2.0 |
| store_id | 固定存储实例身份 |
| access_domain | 内容所属访问域 |
| blob_id | sha256:<64 位小写十六进制摘要> |
| sha256 | 原始字节的 SHA-256 |
| size_bytes | 原始字节长度，非负整数 |

blob_id 与 sha256 必须一致。

字节产物的身份为：
(store_id, access_domain, blob_id)。

调用方提供的摘要和长度仅作为待核验声明，
不能替代存储重新计算。

同一实例、同一访问域、相同字节返回相同 ArtifactRef。
相同字节在不同访问域使用不同存储命名空间，
不产生跨域读取资格。

ArtifactRef 不承担媒体类型和业务文档 revision 的含义。
相同字节可以被不同的业务对象引用，
其业务身份由对象所有者管理。

BlobStat 的结构为：
- artifact_ref：正式 ArtifactRef。
- observed_size_bytes：实际文件长度，非负整数。
- integrity_check：固定为 metadata_only。

成功返回表示正式元数据合法、内容文件存在，
且实际长度与引用一致；尚未重新计算全部内容摘要。
长度不一致返回 integrity_mismatch。
完整摘要校验通过 open_verified 执行。

### C3. SourceArtifact

SourceArtifact 为原文件、渲染页、提取文本及其他派生产物
增加稳定元数据和谱系。

| 字段 | 约束 |
|---|---|
| schema_version | 本版为 2.0 |
| artifact_id | 逻辑资产 ID，不使用秘密值或敏感文件名 |
| revision | 明确的不可变资产修订，不接受 latest |
| content_ref | 指向本资产字节的 ArtifactRef |
| media_type | 显式媒体类型，作为声明保存 |
| parents | 零个或多个父 SourceArtifactKey |
| provenance | 本节规定的来源记录 |

SourceArtifactKey 包含：
store_id、access_domain、artifact_id、revision。

SourceArtifact 的身份由 SourceArtifactKey 唯一确定。
其访问域和存储实例取自 content_ref。

provenance 包含：
- producer_id：导入器、渲染器或提取器身份。
- producer_version：具体版本。
- config_digest：可选，相关配置摘要。
- external_record_id：可选，外部来源记录身份。

原始导入资产可以没有 parents。
派生产物必须关联已经提交的父资产，
禁止自引用和循环关系。

本轮 content_ref 与所有 parents 必须位于同一
store_id 和 access_domain。
跨域复制必须作为单独的授权导入行为处理。

SourceArtifact 的 revision 与源文业务 revision 分别解释：
前者标识资产记录，后者由 Dataset Registry 的语料契约管理，
并显式绑定所用资产。

同一资产身份下，内容、媒体声明、父关系或来源记录变化，
均不能覆盖旧记录。

media_type 不表示已经解析或验证其内容。
声明与内容是否一致，由使用该格式的消费者检查。

v2 示例中的简写资产 ID，由接入层解析为正式引用；
不能将简写 ID 当作已验证的文件位置。

### C4. 公开操作

本地实现的构造入口：

open_local_store(local_config, policy_provider) -> ArtifactStore

local_config 包含：
- root：存储根目录。
- mode：open_existing 或 create_new，默认 open_existing。
- max_blob_bytes：必填的正整数，限制单个字节产物大小。
- max_metadata_bytes：正整数，默认 1048576。
- lock_timeout_ms：正整数，默认 5000。

root 在构造时解析为固定的绝对路径。
普通读取不会自动创建存储。

create_new 只初始化新的存储。
已有合法存储返回 revision_conflict；
已有不完整或无法识别的数据时保留现场并拒绝初始化。
open_existing 要求存储身份与格式合法，不自动生成新身份。

policy_provider 由可信组合入口注入，提供：

check(access_context, access_domain, operation) -> None

operation 为 read 或 write。
拒绝时返回 access_denied。
检查上下文允许范围、存储侧当前策略及 policy_revision，
过期的权限上下文不能继续授权新操作。

put_blob 要求写权限。
register_source 要求写权限，以及校验所引用内容与父资产的读权限。
公开读取操作要求读权限。

权限检查不以文件存在性代替。
存储打开行为不授予任何访问域的数据权限。

以下为语义接口，具体 Python 类型遵守上述契约：

```text
put_blob(
    content_stream,
    access_domain,
    access_context,
    expected_sha256=None,
    expected_size_bytes=None
) -> ArtifactRef

stat_blob(
    artifact_ref,
    access_context
) -> BlobStat

open_verified(
    artifact_ref,
    access_context
) -> 上下文管理的只读、可定位二进制流

register_source(
    source_artifact,
    access_context
) -> SourceArtifact

get_source(
    source_key,
    access_context
) -> SourceArtifact
```

put_blob：
- 从当前位置开始读取输入流，调用方负责关闭输入。
- 按流计算摘要和长度，不要求将整个产物载入内存。
- 构造存储时必须设置正整数 max_blob_bytes。
- 超过大小限制或预期摘要、长度不符时失败。
- 成功返回表示产物已经完整提交。
- 重复写入只有在权限和现有内容完整性检查通过后才可复用。
- I/O 失败不能留下对正式读者可见的半成品。

stat_blob：
- 检查访问权限、存储身份及已提交元数据。
- BlobStat 包含正式 ArtifactRef 和可用性检查结果。
- 只做元数据查询不表示已验证完整字节内容。
- 缺失、未提交或引用不一致时返回相应错误。

open_verified：
- 在向调用方暴露内容前，检查完整摘要与长度。
- 使用同一个只读文件句柄完成校验和后续读取，
  校验后定位到字节起点。
- 校验失败不交付部分内容。
- 上下文退出时关闭流；调用方不能取得可写句柄。
- 文本解码及业务 schema 校验由消费者执行。
- 完整校验需要读取全部字节，成本为 O(产物大小)。
- 并发调用之间使用独立句柄，不共享读取位置。

register_source：
- 校验记录、content_ref、父引用和访问域。
- 要求字节和父资产已提交，禁止悬空谱系。
- 同身份、相同规范内容幂等返回。
- 同身份、不同规范内容返回冲突。
- 服务端审计时间独立记录，不改变记录的语义身份。

get_source：
- 返回指定身份已提交的资产记录。
- 校验资产元数据的完整性。
- 不隐式替换 revision，不搜索其他相似资产。
- 返回元数据不代替 open_verified 对字节的校验。

本轮不提供任意路径读取、自动 URL 下载、
原地更新和自动寻找最新版本的接口。

### C5. 提交与失败不变量

正式可见条件：
- 字节长度和摘要已经核验。
- 必要元数据已经提交。
- 相关引用满足本操作的前置要求。

字节写入与资产登记是两个操作：
前者成功、后者失败时，字节可以暂时未被资产引用。
未引用字节不自动成为有效源文或有效实验结果。

Store 不保证整个数据集或整个实验的跨对象事务。
上层所有者最后提交其发布清单，定义整体可见性。

已提交内容不得原地改写。
发现损坏时返回完整性错误，不用相同 ID 静默替换内容。

存储操作不自动重复读取已经消耗的输入流。
调用方需要重试时，重新提供同一输入并记录 attempt。

读取检查当前访问策略。
引用曾经合法，不保证撤销权限后仍可读取。

实际删除、访问撤销和内容损坏属于不同原因。
后续维护能力必须保留可区分的审计依据。

## 2. 错误契约

错误状态：Confirmed / r001。

错误遵守横切规则中的公共维度。
本模块公开以下 reason_code：

| reason_code | 含义 | 默认处理 |
|---|---|---|
| artifact-store.invalid_input | 字段、引用或参数非法 | 修正输入 |
| artifact-store.unsupported_schema | 不支持该契约版本 | 明确转换或升级 |
| artifact-store.access_denied | 操作无访问权限 | 不自动重试 |
| artifact-store.store_mismatch | 引用不属于本存储实例 | 检查配置或导入 |
| artifact-store.not_found | 授权可见范围内不存在已提交对象 | 检查引用 |
| artifact-store.integrity_mismatch | 字节或元数据完整性不符 | 停止使用并排查 |
| artifact-store.size_limit_exceeded | 超过配置的产物大小限制 | 调整输入或显式配置 |
| artifact-store.revision_conflict | 同身份对应不同内容 | 使用新 revision 或纠正重复提交 |
| artifact-store.invalid_lineage | 父引用非法、悬空或成环 | 修正谱系 |
| artifact-store.write_conflict | 当前无法取得提交协调权 | 按调用方策略处理 |
| artifact-store.io_error | 文件读写或持久化失败 | 依据明确原因决定重试 |

无权读取时不通过错误细节泄露对象是否存在、
真实路径、文件名或其他访问域信息。

write_conflict 与 io_error 不自动标记为必然可重试；
磁盘空间、权限和对象身份等问题须先得到解释。

消费者不能把读取错误转换成“空上下文”“无证据”或零分。
由上层按受影响阶段记录接入或测量失败。

## 3. 本地实现与恢复

状态：Confirmed。
内容修订：r001。
范围：C2 本地文件后端的身份、布局、提交、并发与恢复。

### 3.1 存储布局

root 对应工作区中的 artifacts 目录。
本地布局如下：

```text
<root>/
├── store.json
├── store.lock
├── domains/
│   └── <domain_key>/
│       ├── blobs/sha256/<前两位>/<完整摘要>/
│       │   ├── payload.bin
│       │   └── manifest.json
│       └── sources/<前两位>/<source_key_digest>.json
├── staging/
│   └── <内部生成的操作 UUID>/
└── audit/
    └── <内部生成的记录 UUID>.json
```

domain_key 为 access_domain 的 UTF-8 字节摘要。
source_key_digest 为完整 SourceArtifactKey 规范序列化后的摘要。

摘要负责形成安全、固定长度的路径。
读取记录后仍需检查完整身份，不能仅凭路径摘要接受对象。

调用方提供的 artifact_id、revision、operation_id
均不直接拼接为文件名。

所有受管路径必须落在固定 root 内。
拒绝内部符号链接、重解析点及跨文件系统暂存配置；
不跟随资产记录中的 URL 或文件路径。

本地目录是私有持久化格式。
消费者只能通过公开接口访问。

### 3.2 存储身份与格式

store.json 保存：
- storage_format_version：本版为 1。
- store_id：初始化时生成的唯一身份。
- created_at：UTC 时间。
- hash_algorithm：sha256。
- canonicalization_version：rag-eval-json-v1。

身份文件提交后不原地修改。
打开时检查必需字段、版本及摘要算法。

缺少身份文件但已有业务数据或身份暂存文件时，
不能自动生成另一个 store_id。
返回明确的 I/O 或格式错误，保留恢复依据。

不支持的存储格式返回 unsupported_schema。
本轮不提供自动迁移和静默重建。

### 3.3 元数据规范化

rag-eval-json-v1 规则：
- UTF-8，无 BOM。
- 对象键按 Unicode code point 排序。
- 使用固定紧凑分隔符，不写入额外空白。
- 拒绝重复对象键、NaN、Infinity 和非法字段类型。
- 保留字符串本身，不做 Unicode 或业务文本归一化。
- 可选字段按类型定义填充默认值后再序列化。

SourceArtifact 的 parents 按完整父身份排序，
重复父引用返回 invalid_lineage。

业务金额等类型化数值由其所有者序列化；
Store 不改写传入的 payload 字节。

资产记录文件包含：
- storage_format_version。
- source：规范化 SourceArtifact。
- source_sha256：source 规范字节的摘要。

读取时重新校验 source_sha256 和完整资产身份。
这是损坏检测，不提供抵抗恶意本地文件篡改的数字签名保证。

### 3.4 写入协调

同一规范 root：
- 进程内使用共享互斥，防止多个 Store 对象并发写入。
- 跨进程使用操作系统文件锁。
- 所有初始化、写入、恢复和暂存清理遵守同一锁协议。

store.lock 为固定锁文件，不通过删除锁文件释放锁。
PID、文件年龄和文件存在性不作为锁是否有效的判据。

首个实现目标为 Windows 本地 NTFS。
平台封装使用操作系统支持的文件锁；
进程退出后由操作系统释放。

获取锁的总等待时间受 lock_timeout_ms 限制。
超时返回 write_conflict，不改变目标内容。

C2 中 put_blob 在读取输入到完成提交期间持有写锁，
避免恢复流程清理活跃暂存文件。
register_source 同样串行提交。

读取不取得写锁。
正式对象提交后不再修改，因此独立读取句柄可以并行使用。

网络盘、同步盘及其他文件系统不默认视为具备相同能力；
支持范围需通过对应集成验证后声明。

### 3.5 字节写入与提交

put_blob 流程：

```text
校验参数并检查写权限
→ 取得存储写锁
→ 建立内部操作 UUID 和暂存目录
→ 写入调用意图
→ 流式写入 payload，计算摘要及长度
→ 校验大小上限和 expected 值
→ 得到正式内容身份
→ 检查正式目标是否已存在
→ 提交完整对象，或验证并复用既有对象
→ 保存操作结果
→ 释放写锁
```

调用意图只保存安全关联信息，不保存凭据。

目标不存在时：
1. 在暂存对象目录写入 payload.bin。
2. 写入包含 storage_format_version 和 ArtifactRef 的 manifest.json。
3. 刷新并关闭写句柄。
4. 再次确认权限和目标状态。
5. 将完整对象目录重命名到正式位置。
6. 校验正式元数据，记录提交结果后返回。

暂存位置和正式位置必须位于同一文件系统。
不采用跨卷复制后删除模拟原子提交。

目标已存在时：
- 验证正式 manifest、身份、长度和完整内容摘要。
- 一致则幂等返回既有引用。
- 损坏则返回 integrity_mismatch。
- 不删除、不覆盖、不自动修补既有正式对象。

读者只认可正式位置中结构完整的对象。
临时目录和缺少合法 manifest 的目录不可读取。

进程在提交后、返回前退出时，
对象可能已经提交。
调用方重试通过内容身份校验并复用，不再覆盖。

输入流异常、空间不足或摘要不符时，
保存失败原因，正式位置保持不变。

### 3.6 资产登记与读取

register_source 在写锁内：
1. 校验 schema、完整身份和同域要求。
2. 校验 content_ref，并完整检查内容字节。
3. 读取并核验直接父资产记录。
4. 拒绝自引用、重复父引用和未提交父资产。
5. 规范化 SourceArtifact 并计算记录摘要。
6. 检查相同 SourceArtifactKey 是否已有正式记录。
7. 相同内容幂等返回，不同内容返回 revision_conflict。
8. 新记录通过暂存文件提交到正式位置。

父资产必须已经提交且不可变。
新资产只引用既有节点，正常写入流程由此保持无环。
资产身份相同的重复登记不新增节点或边。

get_source 重新验证记录摘要、schema 和请求身份。
不把索引缓存当作唯一权威数据。

stat_blob 检查正式 manifest、身份、文件存在性与长度。
open_verified 使用同一只读句柄完整计算摘要，
通过后将位置归零，再交给调用方。

Store 不主动修改已打开的正式内容。
操作开始及提交前检查策略；
已交付给调用方的字节无法通过后续撤权追回，
后续读取必须重新授权。

### 3.7 中断恢复与暂存清理

恢复只在成功取得同一存储写锁后进行。
此时没有遵守协议的活跃写入者。

恢复检查每个本模块拥有的暂存操作：
- 正式目标存在且完整：记录已提交结果。
- 正式目标不存在：记录未提交或失败结果。
- 正式目标存在但不完整：记录完整性故障，停止复用该对象。
- 操作身份或路径无法核验：保留现场，不自动删除。

恢复不会将残留暂存目录自动提升为正式对象。
调用方重新提供输入，通过正常接口完成重试。

本轮只清理能够确认归属的暂存文件，
不清理正式对象，也不实施未引用对象垃圾回收。

清理前重新核验绝对路径位于本操作的暂存目录。
清理不得越过 root，不跟随链接。
清理失败记录原因，不伪装为恢复完成。

对象已经提交但审计结果尚未保存时，
暂存调用意图保留恢复依据。
审计失败可以导致本次操作返回 io_error，
错误须说明目标可能已提交并关联已知身份。

恢复依据对象的正式状态判断结果，
不能把调用方没有收到成功响应解释为对象不存在。

### 3.8 一致性与持久性保证

C2 的目标保证：
- 进程异常退出下，正式读者不会接收部分写入内容。
- 重试不会覆盖已提交对象。
- 同源身份冲突可检测。
- 恢复不会误删遵守协议的活跃写入。

目录重命名、文件刷新及操作系统锁的实际行为，
需要在目标环境通过故障注入和多进程测试验证。

本版不承诺整机断电后所有已返回提交均持久保存，
也不宣称存储硬件或文件系统故障能够自动修复。
更强的断电持久性需要单独验证和设计。

读取发现缺失或损坏时保留错误，
上层据此标记产物不可用和复现限制。

### 3.9 代码与测试接缝

计划的关键内部职责：
- local_store：实现公开存储操作。
- layout：安全路径及身份映射。
- canonical：带版本的元数据规范化。
- coordination：进程内互斥和操作系统锁。
- commit：暂存、正式提交及故障阶段记录。
- recovery：恢复检查与受控暂存清理。

这些是职责分工，私有文件和辅助类可以合并或拆分。

测试通过隔离目录、可控输入流、故障注入点
及独立进程验证，不使用生产材料。

模块不依赖 Dataset Registry、Runner 或其他业务模块。

## 4. 验证计划

状态：Confirmed / r002。
以下为计划验收，尚未执行。

| 场景 | 应观察到的结果 |
|---|---|
| 同域重复保存相同字节 | 返回相同引用，不改写正式内容 |
| 同域保存不同字节 | 返回不同内容身份 |
| 不同域保存相同字节 | 分别检查授权，不能跨域读取 |
| 伪造摘要、长度或 blob_id | 拒绝不一致引用或写入 |
| 写入超过大小限制 | 不产生可见半成品 |
| 元数据查询后内容被损坏 | open_verified 拒绝交付内容 |
| 中文、换行和二进制零字节 | 原始字节可完整还原 |
| 同源资产身份对应不同父关系 | revision_conflict |
| 父资产缺失或跨域 | invalid_lineage 或 access_denied |
| 内容已经写入但资产登记失败 | 不出现有效 SourceArtifact |
| 权限撤销后使用旧引用 | access_denied |
| Store 读取失败 | 上层保留原因，不补造空产物 |

| 场景 | 应观察到的结果 |
|---|---|
| 两进程保存同域相同字节 | 最终只有一个正式内容身份，两次成功结果一致 |
| 两进程登记同身份不同资产 | 一次成功，另一次 revision_conflict |
| 活跃写入时启动恢复 | 等待写锁或 write_conflict，不删除活跃暂存 |
| payload 写入期间终止进程 | 正式对象不可见，恢复后可以正常重试 |
| manifest 完成但目录尚未提交时退出 | 暂存内容不作为正式对象读取 |
| 正式提交后、返回前退出 | 重试验证并复用既有对象 |
| 审计结果写入失败 | 保留提交阶段和对象身份，不谎报目标不存在 |
| 正式 payload 或资产元数据损坏 | 读取返回 integrity_mismatch |
| 缺少 store.json 但存在数据 | 拒绝生成新 store_id |
| 内部路径被替换为链接或重解析点 | 拒绝越界访问和清理 |
| 写锁持有进程退出 | 后续进程可以重新取得操作系统锁 |
| 只调用 stat_blob | 不宣称已通过完整内容摘要校验 |

上述测试尚未执行。
Windows 本地 NTFS 是首次实现验收环境；
其他文件系统和断电持久性不在本轮已验证范围内。

## 5. 依赖与实现安排

本模块没有其他业务模块依赖。
允许使用标准库及按需要确认的类型校验、文件协调依赖。

提供方契约：
本文件 Contracts / r002。

消费者包括数据注册、接入、证据映射、评分、
执行及报告模块，均只能通过公开契约和 api 使用本模块。

计划代码入口：
- rag_eval.artifact_store.contracts
- rag_eval.artifact_store.api

本地布局与关键实现职责见第 3 节。
私有辅助函数和契约保持的内部拆分由实现决定。

测试接缝：
- 可控的输入流与只读输出流。
- 隔离的真实本地工作区。
- 明确的 StoreAccessContext。
- 可注入的提交故障点及操作协调行为。

外部网络集成：N/A，本轮仅使用本地产物。
业务模型调用：N/A，本模块不调用模型。

## 6. 本地进度

| 设计单元 | 审阅状态 | 内容修订 | 位置 |
|---|---|---|---|
| 概述与边界 | Confirmed | r001 | 第 1 节 |
| 公开契约 | Confirmed | r002 | Contracts |
| 错误 | Confirmed | r001 | 第 2 节 |
| 本地实现与恢复 | Confirmed | r001 | 第 3 节 |
| 契约验证计划 | Confirmed | r002 | 第 4 节 |
| 依赖与实现安排 | Confirmed | r002 | 第 5 节 |

Complete 仅表示本文件声明的 C2 本地存储范围设计完成。
应用实现和验收尚未执行；远端存储、垃圾回收等仍在后续范围。
C2 整体仍为 Design pending。

公开接口、数据含义、存储与授权语义、验收行为的变更需确认。
私有实现可在上述边界内决定。
