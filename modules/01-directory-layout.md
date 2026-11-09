# RAG 测评框架 v2：工程目录与依赖规则

- 设计标识：rag-evaluation-v2
- 审阅状态：Confirmed
- 内容修订：r001
- 架构依据：[总体架构 r001](00-system-architecture.md)。
- 确认范围：工程目录、模块到代码包的映射、公开导入边界、
  配置与产物位置、测试及依赖检查安排。
- 各模块字段、操作和内部实现：Pending。

## 1. 当前结构与本轮边界

当前项目包含：
- doc/：v2、数据集生成详设及前期评审资料。
- modules/：已确认的设计索引与总体架构。

当前未发现应用源码、依赖清单或测试工程。
下面的源码、配置、测试和运行目录均为拟定结构。

设计文档继续位于用户指定的 modules/。
其内部的 modules/<模块>/README.md 沿用已确认的设计包层级。

目录规划确认后，按选定候选创建实际需要的代码和文件。
目录树不要求一次性建立所有模块或空占位目录。

## 2. 拟定工程目录

```text
RAG-evaluation/
├── README.md
├── pyproject.toml
├── doc/                         原始需求、v2 和补充资料
├── modules/                     工程详细设计包
│   ├── index.md
│   ├── 00-system-architecture.md
│   ├── 01-directory-layout.md
│   ├── 02-cross-cutting-concerns.md
│   ├── modules/
│   │   └── <module-id>/README.md
│   ├── slices/                  候选选定并审阅后按需创建
│   ├── handoffs/                交接批准后按需创建
│   └── 99-requirement-traceability.md
├── src/
│   └── rag_eval/
│       ├── __init__.py
│       ├── __main__.py
│       ├── cli/
│       ├── artifact_store/
│       ├── review/
│       ├── dataset_registry/
│       ├── dataset_builder/
│       ├── adapters/
│       ├── evidence_mapper/
│       ├── evaluators/
│       ├── metric_planner/
│       ├── diagnostics/
│       ├── runner/
│       └── reporting/
├── tests/
│   ├── architecture/
│   ├── unit/
│   ├── contract/
│   ├── integration/
│   ├── e2e/
│   └── fixtures/
├── configs/
│   └── examples/
├── scripts/                     按需增加的开发、导出与发布工具
└── build/                       可再生成的构建输出，忽略提交
    └── schemas/                 按需导出的 JSON Schema
```

采用 src 布局，使测试和 CLI 面向已安装的包，
避免从项目根目录意外导入未打包的源码文件。

pyproject.toml 管理包元数据、Python 版本、依赖、
命令入口和开发工具配置。
具体依赖版本在首次实现时锁定。

rag-eval 与 python -m rag_eval 使用同一 CLI 入口。
__main__.py 仅转交入口，不重复实现命令行为。

## 3. 逻辑模块到代码包的映射

| 设计模块 ID | Python 包 |
|---|---|
| artifact-store | rag_eval.artifact_store |
| review | rag_eval.review |
| dataset-registry | rag_eval.dataset_registry |
| dataset-builder | rag_eval.dataset_builder |
| adapters | rag_eval.adapters |
| evidence-mapper | rag_eval.evidence_mapper |
| evaluators | rag_eval.evaluators |
| metric-planner | rag_eval.metric_planner |
| diagnostics | rag_eval.diagnostics |
| runner | rag_eval.runner |
| reporting | rag_eval.reporting |

逻辑模块的公开入口保持独立。
内部可以按实际复杂度拆分文件，不要求一个概念对应一个类。

每个已实现模块采用以下公开入口约定：

```text
<module>/
├── __init__.py
├── contracts.py
├── api.py
└── <内部实现文件或子包>
```

- contracts.py：公开数据模型、枚举、协议和模块错误定义。
- api.py：受支持的操作入口、实现构造入口或工厂。
- 内部文件：算法、持久化、解析、调用封装及具体集成。

contracts.py 与 api.py 是稳定导入门面。
内部实现可拆分并通过门面导出，不因此产生第二套契约。

模块内的具体文件名、私有类和辅助函数由实现决定。
公开入口迁移、数据含义或协议变化需要确认。

__init__.py 保持轻量。
导入包不得自动启动模型调用、建立数据库、扫描数据集、
创建运行目录或注册全部第三方插件。

## 4. 导入规则与具体集成位置

### 4.1 跨模块导入

允许的模块依赖以
[总体架构第 4 节](00-system-architecture.md#4-依赖方向)为准。

跨模块只使用提供方的：
- contracts：类型、协议、错误及校验入口。
- api：公开操作或实现构造入口。

消费者不得导入提供方私有文件或操作其内部表、缓存和目录。

例如，允许的消费者可以使用：
rag_eval.dataset_registry.contracts.TestCase。

同一消费者不能直接依赖 Registry 的私有存储实现，
也不能绕过 Registry 修改冻结数据文件。

TYPE_CHECKING 下的业务类型导入同样遵守模块依赖方向。
延迟导入、字符串导入和重新导出不能用于绕过边界。

### 4.2 组合与注入

CLI 根据配置构造具体实现，再通过公开协议传入业务模块。

runner 使用已注入的 Adapter、Store、Mapper、
Evaluator 等能力，避免业务流程自行搜索全局插件。

模块内部若需要真实变化点，再定义相应协议。
不为每个普通辅助函数创建抽象层。

### 4.3 第三方集成

具体系统接入位于 adapters 内部。
解析、模型裁判、存储等集成放在其行为所属模块内。

可选 SDK 在具体集成被选中时加载。
未安装可选依赖时，核心包仍可导入，
调用相应能力时返回明确的依赖缺失原因。

首次实现按需要声明可选依赖组。
不预先引入多个重叠的追踪、评测或实验平台。

### 4.4 导入约束检查

开发检查采用以下分工：
- import-linter：检查包依赖关系与禁止的反向依赖。
- architecture 测试：补充直接导入白名单、私有路径、
  TYPE_CHECKING 及不受支持的动态跨模块导入检查。
- 安装包冒烟检查：验证公开入口可导入且没有启动副作用。

检查配置随首次实现验证。
未覆盖的动态插件行为需在具体集成测试中说明，
不能仅凭静态导入检查宣称边界已经完全保证。

## 5. 配置与运行产物

### 5.1 配置位置

configs/examples/ 只保存可公开、可复查的配置示例，
使用合成标识和凭据引用。

真实语料路径、生产请求、内部端点和秘密值
不进入公共配置示例。

内置默认值随拥有该配置的包发布，
读取不依赖源码项目的当前工作目录。

配置优先级、相对路径基准和有效配置冻结规则
在横切规则中确认。

### 5.2 工作区

运行产物位于显式配置的工作区，不写入 src/。
工作区可以位于项目外；默认位置在横切规则中确认。

工作区内部规划如下，按实际功能创建：

```text
<workspace>/
├── artifacts/                   Store 管理的原始与派生产物
├── datasets/<dataset-version>/  Registry 发布的数据集
├── builds/<build-id>/           Builder 工作记录与任务产物
├── experiments/<experiment-id>/ 实验、运行与评价记录
├── reviews/<review-batch-id>/   受控审定工作包及结果
├── comparisons/<comparison-id>/ 配对结果和报告导出
├── staging/                     尚未提交的临时产物
├── cache/                       可重建且受访问域限制的缓存
└── logs/                        受控运行日志
```

数据集沿用 v2 的 manifest.json、
test_cases.jsonl、evidence_units.jsonl 等审计产物。

实验保留 v2 的 manifest、runs、spans、
evidence_mappings、judgments、interventions 等记录含义。
具体文件布局及多次评分的版本组织在所属模块中确认，
必须保留既有 Run 和 Judgment。

comparisons 保存固定输入版本下的报告。
若导出到实验目录，必须绑定明确的 comparison revision，
不能通过隐式“最新报告”改变历史实验的解释。

Store 的实际对象分片、索引和授权元数据布局
属于 artifact-store 详设。
消费者通过引用和公开接口访问，不拼接内部路径。

正式工作区与开发构建输出分别管理。
工作区可能含受限数据，不能因目录位于项目内而自动提交版本库。

### 5.3 生成物与迁移

JSON Schema 从契约所有者的类型定义导出，
按模块和 schema 版本区分。
build/schemas/ 中的内容可再生成，不手工维护第二份定义。

运行时校验器可以随安装包发布；
如需发布 schema 文件，由构建流程生成并检查一致性。

数据库迁移随持久化所有者维护。
只有选定 SQLite 等实际后端后，才增加对应迁移文件。

数据格式转换由契约所有者提供。
转换结果记录输入版本与转换版本；
语义变化形成新数据版本，不能用文件搬移掩盖。

## 6. 测试、工具与构建边界

| 位置 | 验证责任 |
|---|---|
| tests/architecture/ | 导入方向、公开入口和包导入副作用 |
| tests/unit/ | 重要算法与规则的边界行为 |
| tests/contract/ | 提供方公开操作、序列化、版本、错误与不变量 |
| tests/integration/ | 真实文件存储、模块衔接、提交及中断恢复 |
| tests/e2e/ | 选定候选的 CLI 输入到最终产物 |
| tests/fixtures/ | 小型、可解释、可公开的合成材料和审定样例 |

契约测试按提供方组织，消费者通过相同协议使用测试实现。
集成测试区分真实后端与替身，记录验证范围。

默认测试使用受控输入，不自动调用生产系统或付费模型。
需要真实外部调用的测试单独标识、显式启用并检查预算和权限。

fixtures 中的 gold、用户输入和待测知识材料分别组织，
用于验证参考隔离，不能因测试便利而混成一个输入对象。

scripts/ 只保留开发、构建和发布胶水。
业务规则仍通过包的公开入口调用。

开发工具建议采用 pytest、Ruff、mypy 和 import-linter；
具体版本在首次实现中锁定。

CI 应在安装包后运行检查，避免仅通过源码路径运行。
本轮只定义检查责任，尚未创建或执行应用测试。

## 7. 已确认边界与后续设计

本轮确认模块到代码包的映射及公开导入门面。
模块内部的契约保持重构由实现决定。

以下内容继续在对应设计中确认：
- 横切规则：错误格式、配置优先级、工作区默认位置、
  临时提交、幂等、并发、重试、恢复、审计与兼容。
- 模块详设：具体操作、字段、后端、事务、
  文件或表结构、版本转换及验收案例。
- 候选范围：需要实现的代码包和子功能。
- 具体集成：真实系统、SDK、模型和验证证据。

本目录树覆盖全局规划，不意味着全部模块已完成设计，
也不意味着选定候选必须实现所有目录。
