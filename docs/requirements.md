# Aether 模块需求文档

## 1. 文档目标

本文档定义 Agent 平台中 Aether 部署模块的完整需求基线，覆盖：

- 六类资源的统一 CUD 与任务 Query（CUD 触发部署、Query 基于 `task_id` 同步读取当前部署任务状态与任务结果）
- 统一接口入参与返回值约束
- 基于调用参数渲染 Helm `values.yaml` 数据的能力，并保证部署执行与 `helm install/upgrade -f values.yaml` 语义等效
- 多角色、多工作空间、多集群场景下的资源作用域与权限边界
- 可扩展到新资源类型的通用部署抽象

## 2. 背景与范围

### 2.1 业务背景

Agent 平台需同时支持：

- 高代码 Agent 应用：DevBox 发布镜像或用户上传镜像/Helm Chart 创建
- 低代码 Agent 应用：平台内置 Dify/FastGPT 或用户上传 Helm Chart 的 Coze Studio/n8n

所有资源底层均通过 Helm 在 Kubernetes 部署。

### 2.2 Aether 责任范围（In Scope）

- 提供通用、复用的资源 CUD/Query 能力
- 统一资源部署动作管理（创建、更新、删除）与部署任务状态/结果查询接口
- 提供无状态部署执行能力（不承担资源/任务状态持久化）
- 基于工作空间/集群/命名空间的部署落地
- 为上层模块提供可复用接口函数（非对外 OpenAPI）
- 提供按调用参数渲染 Helm `values.yaml`（文件或数据）的能力，供部署执行与上层预览/审计复用

### 2.3 非目标（Out of Scope）

- 不负责低代码平台内部业务功能（如 Dify/FastGPT 的工作流细节）
- 不负责平台账号登录认证流程本身
- 不负责资源级业务约束前置校验（如单实例限制、资源来源组合限制等）
- 不负责部署状态、任务状态、历史快照的落库维护（由上层业务模块维护）
- 不负责资源运行态统一查询视图（由上层业务模块按需聚合）

## 3. 术语与对象

- 工作空间（Workspace）：资源管理与权限作用域的核心边界
- 纳管集群（Managed Cluster）：被平台接入的 Kubernetes 集群
- 命名空间（Namespace）：工作空间在关联集群中的同名 namespace
- 纳管镜像仓库（Managed Registry）：被平台接入的 Harbor 仓库
- 资源类型（Resource Type）：Aether 管理的部署对象类型
- 资源实例（Resource Instance）：某资源类型的一次实例化部署
- 部署执行（Deployment Execution）：一次 CUD 动作触发的实际 Helm/K8S 执行过程

## 4. 参与角色与权限要求

### 4.1 角色定义

- 超级管理员
- 空间管理员
- 普通用户

### 4.2 权限规则

#### FR-ACL-001 超级管理员权限

超级管理员可操作全部资源，至少包括：

- 纳管 K8S 集群 CRUD
- 纳管镜像仓库（Harbor）CRUD
- 工作空间 CRUD
- 工作空间关联集群 CRUD
- 工作空间关联镜像仓库 CRUD
- 工作空间网关实例 CRUD
- 工作空间空间管理员配置
- 空间管理员与普通用户全部能力

#### FR-ACL-002 空间管理员权限

空间管理员可在授权工作空间内：

- 开通低代码平台
- 关联与管理普通用户（CRUD）
- 执行普通用户全部能力

#### FR-ACL-003 普通用户权限

普通用户可在被关联工作空间内：

- 创建数据服务组件实例
- 使用 DevBox（CRUD）
- 创建记忆体实例
- 创建高代码应用
- 进入工作空间内低代码平台并操作

#### FR-ACL-004 工作空间内无细粒度隔离

同一工作空间内用户可操作该空间内资源，不做按“资源创建者”维度的二级隔离。

## 5. 资源类型与部署形态

### 5.1 资源类型清单

#### FR-RT-001 Aether 必须支持以下六类资源

- 数据服务组件：PostgreSQL、Redis、Milvus、Weaviate、pgvector、NebulaGraph、MongoDB
- 低代码平台：Dify、FastGPT、Coze Studio、n8n
- DevBox
- 网关（Higress）
- 高代码应用（openclaw、zeroclaw，以及用户自定义镜像/Chart）
- 记忆体（mem0）

### 5.2 资源形态说明

以下条目用于约定 Aether 需要承载的资源模板与来源能力，不定义资源级业务约束校验。
资源级业务约束（如单实例限制、资源来源组合限制）应由上层业务模块在下发资源操作前完成判断。

#### FR-RT-002 数据服务组件形态

- 数据服务组件模板由平台内置（Helm Chart）
- 高代码应用可绑定用户预先创建的数据服务组件
- 依赖数据服务的创建与展示策略由上层业务模块定义，Aether 仅按请求执行部署

#### FR-RT-003 低代码平台形态

- 平台内置 Dify/FastGPT 模板，且模板形态为 Helm Chart
- 支持超级管理员导入 Coze Studio、n8n 或外部 Dify/FastGPT，导入格式仅 Helm Chart
- 低代码平台依赖项由上层业务模块在请求中显式定义

#### FR-RT-004 DevBox 形态

- DevBox 基于平台内置高代码模板（Helm Chart）创建开发容器

#### FR-RT-005 网关形态

- 网关模板为平台内置 Helm Chart

#### FR-RT-006 高代码应用形态

- 支持三种来源创建：DevBox 发布镜像、用户上传镜像、用户上传 Helm Chart
- 镜像方式：1 个 Agent 实例 + 1..N 个预创建数据服务组件
- Helm Chart 方式：1..N 个 Agent 实例 + 1..N 个预创建数据服务组件
- Helm Chart 内部依赖的解释与展示边界由上层业务模块负责

#### FR-RT-007 记忆体形态

- 模板形态为平台内置 Helm Chart
- 一个记忆体实例由 1..N 记忆体业务实例 + 1..N 不同数据服务组件实例构成
- 记忆体依赖项由上层业务模块在请求中显式定义

## 6. 工作空间与多集群映射

### FR-WS-001 工作空间与集群命名空间映射

- 每个工作空间可关联 1..N 纳管集群
- 每个关联集群中必须创建与工作空间同名 namespace
- 若关联多个集群，必须在每个集群均创建同名 namespace

### FR-WS-002 工作空间与镜像仓库关联

- 每个工作空间可关联 0..N 纳管镜像仓库
- Aether 仅消费请求中的镜像/Chart 来源引用，不负责该引用关系持久化（由上层维护）

### FR-WS-003 部署位置必选

- 资源创建请求必须显式传入部署位置，部署位置即目标 K8S 集群
- 部署位置为必选参数，未传入时请求必须被拒绝并返回参数错误

## 7. Aether 通用能力需求

### 7.1 通用抽象与扩展机制

#### FR-CORE-001 统一部署抽象

Aether 必须提供统一资源部署抽象，至少包括：

- 资源类型注册机制（类型标识、模板来源、参数结构校验规则、默认值规则）
- 通用 Helm 生命周期执行器（install/upgrade/uninstall）
- 通用部署任务查询适配器（基于 `task_id` 返回当前部署任务状态与任务结果）

#### FR-CORE-002 可扩展性

新增资源类型时，不允许复制粘贴现有业务流程；应通过“类型注册 + 配置化规则 + 复用执行器”完成接入。

#### FR-CORE-003 统一资源标识

所有资源实例必须具有全局唯一 `resource_id`，并包含 `workspace_id`、`cluster_id`、`namespace`、`resource_type`；唯一性校验与冲突治理由上层模块负责。

#### FR-CORE-004 统一 values 渲染抽象

Aether 必须提供统一的 values 渲染能力，将调用方请求参数渲染为 Helm 可消费的 `values.yaml` 语义等价数据，并支持：

- 以“YAML 数据”形式输出（用于接口返回、预览、审计、调试）
- 以“可执行输入文件”形式输出（用于 Helm 生命周期执行）
- 同一请求在同一模板版本下渲染结果必须保持语义确定性

### 7.2 接口函数契约

#### FR-API-001 Aether 仅提供代码级接口

Aether 不直接暴露外部 OpenAPI，仅提供可供其他模块调用的 Go 接口。

#### FR-API-002 统一 CUD 入参

CUD 接口必须复用统一请求结构，最小字段包括：

- `request_id`（幂等请求号）
- `operator`（操作者身份与角色信息）
- `workspace_id`
- `target`（resource_type、resource_id，可选）
- `cluster_id`（目标部署集群，必填）
- `spec`（资源声明配置，含 Helm values/依赖引用）
- `labels` / `annotations`（可选）

#### FR-API-003 统一 CUD 返回

CUD 接口必须同步返回受理结果，不阻塞部署完成，最小字段包括：

- `request_id`（回显）
- `task_id`（用于后续任务状态查询）
- `resource_id`（创建场景可先分配）
- `accepted_at`
- `status`（固定为 `ACCEPTED`）

#### FR-API-004 Query 接口能力

Query 接口必须同步返回，且仅支持按 `task_id` 执行单任务查询：

- 请求参数：`task_id`（必填）
- Query 返回值定义为“该 `task_id` 对应部署任务的当前状态与任务结果”，不承诺资源运行态视图与历史时间线聚合

#### FR-API-005 统一错误模型

接口错误必须采用统一错误码体系，至少区分：

- 参数错误
- 权限错误
- 资源不存在
- 资源冲突
- 依赖缺失
- 渲染错误（values 渲染失败、渲染结果不满足模板/Schema 约束）
- 任务查询上下文不可用（执行器/K8S/Helm 反馈不可达）
- 内部错误

#### FR-API-006 values 渲染接口能力

Aether 必须向上层调用方提供 values 渲染能力（代码级接口），用于将调用参数渲染为 `values.yaml`；该能力至少支持：

- 基于统一请求语义进行渲染（与 CUD 输入语义一致）
- 输出格式可选：`values.yaml` 数据或可执行输入文件
- 对同一请求的 CUD 执行，必须复用同一渲染规则，避免“预览渲染结果”与“实际执行输入”分叉

### 7.3 部署执行模型（无状态）

#### FR-EXEC-001 CUD 触发执行

Create/Update/Delete 均以“受理即触发执行”方式运行，CUD 接口不阻塞部署完成。

#### FR-EXEC-002 技术实现约束

- 执行与查询的事实来源为 Kubernetes/Helm
- Aether 不强制依赖 PostgreSQL 持久化资源/任务元数据
- Aether 不强制依赖 Redis 作为任务队列或分布式锁基础设施

#### FR-EXEC-003 幂等与重入

- 基于 `request_id + workspace_id + action` 提供幂等保护能力（仅使用已在接口契约中定义的字段）
- 可恢复错误的重试策略由调用方或上层编排模块负责，Aether 不要求内建持久重试队列
- 同一资源并发互斥不作为 Aether 强制内建能力，可由上层模块治理

#### FR-EXEC-004 Helm 命令语义等效约束

Aether 的 Create/Update 执行语义必须与以下命令保持等效：

- Create 等效于 `helm install <release> <chart> -f <rendered-values.yaml>`
- Update 等效于 `helm upgrade <release> <chart> -f <rendered-values.yaml>`

其中 `<rendered-values.yaml>` 必须来自 Aether 对同一请求参数的标准化渲染结果；Aether 不得在执行路径引入与渲染接口不同的第二套 values 组装逻辑。

### 7.4 查询与一致性

#### FR-Q-001 查询一致性要求

Query 返回“当前部署任务状态与任务结果”，状态来源于当前执行上下文与 Helm/K8S 执行反馈；不承诺返回持久化中间态时间线。

#### FR-Q-002 部署任务状态与结果查询

必须提供任务查询接口函数（按 `task_id`）供上层轮询，并由上层自行维护任务历史快照。

## 8. 数据与状态管理需求

### FR-DM-001 无强制落库

Aether 不强制建设资源实例主表、任务表、任务事件表等持久化实体；任务状态与任务结果基于当前执行上下文与 Helm/K8S 执行反馈提供。

### FR-DM-002 依赖关系表达

资源依赖关系需通过请求 `spec` 显式表达，并在部署执行阶段正确展开：

- 高代码应用 -> 数据服务组件引用
- 低代码平台 -> 上层定义的依赖集合
- 记忆体 -> 上层定义的依赖集合
- Helm 高代码应用 -> 上层定义的依赖集合

### FR-DM-003 依赖展示责任边界

Aether 不维护依赖展示语义及其状态模型，相关策略由上层业务模块负责。

### FR-DM-004 依赖删除策略责任边界

- 依赖删除/保留策略由上层业务模块决定并在请求中显式声明
- Aether 不内置基于依赖展示分类的强制级联规则

## 9. 非功能需求

### NFR-001 技术栈约束

- 后端语言：Go
- Web/API 框架：Gin
- 模板/部署包标准：Helm Chart
- 任务状态/结果事实来源：部署执行上下文 + Kubernetes API / Helm 执行反馈

### NFR-002 可扩展性目标

新增资源类型不应修改通用部署编排核心逻辑，仅新增类型定义和规则。

### NFR-003 可靠性目标

- 部署执行失败可由调用方复用同一 `request_id + workspace_id + action` 触发幂等重试
- 任务查询结果应可反映当前部署任务状态与任务结果
- 对同一资源并发变更的治理由上层模块负责

### NFR-004 可审计性目标

所有 CUD 行为需输出结构化日志（含操作者、请求参数摘要、执行结果），持久审计留存由上层模块负责。

### NFR-005 values 渲染一致性目标

- 同一请求参数 + 同一模板版本在重复渲染时应输出语义等价的 `values.yaml`
- CUD 执行实际使用的 values 输入应可追溯到对应渲染结果（可通过摘要/指纹等方式）

## 10. 边界集成需求

### FR-INT-001 与上层模块集成方式

各业务模块（高代码应用、低代码平台、DevBox、网关等）负责：

- 接收自身 OpenAPI 请求
- 在调用 Aether 前完成资源级业务约束前置校验（如单实例限制、资源来源组合限制等）
- 将业务请求转换为 Aether 统一入参
- 按业务需要选择：先调用 values 渲染能力做预览/审计，再发起 CUD；或直接发起 CUD 由 Aether 内部渲染
- 调用 Aether 通用接口函数
- 轮询 Aether 任务查询接口并维护业务侧任务状态表/历史快照

## 11. 验收标准

### AC-001 统一接口

六类资源均可通过同一套 CUD/Query 接口完成生命周期操作，无需为每类资源单独实现一套引擎。

### AC-002 异步执行

CUD 请求返回 `ACCEPTED` 受理结果；上层可通过任务查询接口轮询当前部署任务状态与任务结果，并自行维护状态流转。

### AC-003 查询能力

任务查询仅支持按 `task_id` 单任务查询，并在工作空间维度满足权限规则。

### AC-004 依赖策略边界

依赖展示与管理策略由上层业务模块维护，Aether 仅执行部署与任务状态/结果查询，不提供依赖分类规则。

### AC-005 values 渲染能力

上层调用方可基于统一请求语义获取 `values.yaml` 渲染结果，且支持“YAML 数据”与“可执行输入文件”两种输出形态。

### AC-006 Helm 执行等效性

在相同 Chart 与相同渲染结果前提下，Aether Create/Update 的实际部署效果应与手工执行 `helm install/upgrade -f values.yaml` 等效。

## 12. 待澄清项（需产品/架构进一步确认）

- 当前无

## 13. 设计与任务拆解补充基线（规范性附录）

本章节用于将现有需求条目转换为可直接拆分设计任务与开发任务的实现输入，约束级别与 FR/NFR/AC 等价。

### 13.1 当前生效决策基线（一页版）

- 适用日期：2026-03-01
- 生效决策（纳入设计输入）：D-001、D-003、D-005、D-007、D-010、D-011、D-012、D-013、D-015、D-016、D-017、D-018、D-019
- 废止决策（不得作为设计输入）：D-002、D-004、D-006、D-008、D-009、D-014

当前生效决策收敛后的实现边界如下：

- Aether 仅负责部署执行与任务状态/结果查询，不承担资源/任务状态持久化。
- CUD 同步受理并返回 `ACCEPTED + task_id`；Query 仅支持按 `task_id` 单任务查询。
- 幂等键仅允许使用 `request_id + workspace_id + action`。
- Aether 必须具备 `values.yaml` 标准化渲染能力，且 Create/Update 执行语义需等效 `helm install/upgrade -f values.yaml`。
- 资源级业务约束前置校验、依赖展示语义、依赖删除策略由上层业务模块负责并显式传参。
- 扩展新资源类型必须通过“类型注册 + 配置化规则 + 通用执行器复用”完成，禁止复制现有业务流程。
- 工作空间是核心授权边界；同空间内不按资源创建者做二级隔离。

### 13.2 设计文档强制约束（接口定义归属设计阶段）

`requirements.md` 仅定义“做什么”和边界，不承载函数接口定义、DTO 字段定义、OpenAPI 定义等设计产物；该类内容必须在设计文档中提供。

设计文档（`docs/design.md`）至少必须包含以下内容：

1. 代码级接口定义：Go 接口签名、请求/响应 DTO、枚举、字段校验规则、错误码表。
2. 接口边界映射：上层模块 OpenAPI 与 Aether 代码级接口之间的映射关系（Aether 自身仍不对外暴露 OpenAPI）。
3. 任务状态机与幂等/重入判定规则：状态枚举、状态流转、重试语义、冲突判定语义。
4. 资源类型注册与扩展机制：注册 schema、默认值策略、生命周期执行策略、新资源接入模板。
5. 依赖表达与删除策略：依赖字段 schema、解析/缺失处理、删除策略参数与行为。
6. 验收测试追踪矩阵：FR/NFR/AC 到测试点的一一映射关系。
7. 图示语法约束：凡需画图的内容（架构图、状态图、时序图、流程图）必须使用 Mermaid 语法；禁止使用图片、PlantUML 或其他图形语法。
8. 设计完成粒度约束：所有任务完成后，`docs/design.md` 必须达到“可直接指导编码”的粒度，至少覆盖：数据结构、状态流转、接口契约、异常码、并发/幂等、关键流程时序、边界条件。
9. values 渲染与执行等效性设计：必须定义参数到 `values.yaml` 的渲染规则、输出形态（数据/文件）、渲染一致性策略，以及与 `helm install/upgrade -f values.yaml` 的等效性保证机制。

任务拆解文档（`docs/task/task_design.md`）必须显式引用上述设计章节，并保证每个任务可追踪到对应需求条目与验收测试点。

### 13.3 任务状态机与幂等处理规则

#### 13.3.1 任务状态机

- CUD 受理态：`ACCEPTED`
- 执行中态：`RUNNING`
- 终态：`SUCCEEDED`、`FAILED`
- 查询降级态：`UNKNOWN`（仅表示查询时上下文不可达，不改变任务终态语义）

状态流转规则：

- `ACCEPTED -> RUNNING`
- `RUNNING -> SUCCEEDED | FAILED`
- `ACCEPTED -> FAILED`（受理后预检查失败）
- `* -> UNKNOWN`（查询时临时降级，不作为持久状态机终态）

#### 13.3.2 幂等与重入规则

- 幂等键：`request_id + workspace_id + action`
- 请求指纹：对 `operator + cluster_id + target + spec + labels + annotations` 做规范化哈希
- 无状态前提：Aether 不依赖资源表/任务表/幂等记录表进行幂等判定
- 事实来源：幂等与重入判定仅基于当前请求计算结果 + Helm/K8S 当前可观测事实（release 元数据、revision、资源注解）
- 命中规则：
  - 由幂等键生成稳定 `task_id` 与 release 标识；若查询到同幂等键且同请求指纹的在途/已完成执行事实，则返回同一 `task_id/resource_id`，不重复触发执行
  - 幂等键命中但请求指纹不一致：返回 `AETHER_CONFLICT`
- 重入规则：
  - 调用方可复用同一 `request_id + workspace_id + action` 发起重试
  - Aether 需先读取目标 release 当前事实状态；若已成功则直接返回成功结果，若未成功则继续执行到下一稳定状态

### 13.4 资源类型注册 schema 与新增资源接入模板

#### 13.4.1 注册 schema（示例）

```yaml
resource_type: HIGH_CODE_APP
category: HIGH_CODE
template:
  source_kind: BUILTIN_CHART # BUILTIN_CHART | IMAGE | EXTERNAL_CHART
  ref: charts/highcode
  version: ">=1.0.0"
spec_schema_ref: schemas/high_code_app.schema.json
defaults_ref: values/high_code_app.defaults.yaml
lifecycle:
  install_timeout_seconds: 900
  upgrade_timeout_seconds: 900
  uninstall_timeout_seconds: 600
  atomic: true
capabilities:
  supports_create: true
  supports_update: true
  supports_delete: true
  dependency_mode: EXPLICIT_ONLY
query:
  adapter: HELM_TASK_ADAPTER
```

#### 13.4.2 新增资源类型接入模板

1. 注册资源类型元数据：补齐 `resource_type/category/template/spec_schema_ref/defaults_ref`。
2. 定义参数 schema 与默认值：确保 `spec.values` 可验证、可补默认值。
3. 配置生命周期执行参数：install/upgrade/uninstall 超时、原子性、回滚策略。
4. 配置依赖展开规则：声明是否启用 `EXPLICIT_ONLY` 依赖解析。
5. 编写资源类型契约测试：覆盖 Create/Update/Delete/QueryTask 与错误码分支。
6. 编写扩展性回归测试：验证新增类型无需改动通用执行编排核心逻辑（NFR-002）。

### 13.5 依赖表达 schema 与执行/删除策略参数

#### 13.5.1 `spec.dependencies` schema（最小集）

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `dependency_id` | 是 | 依赖项唯一标识（请求内唯一） |
| `alias` | 是 | 绑定别名，供模板渲染引用 |
| `resource_type` | 是 | 依赖资源类型 |
| `resource_id` | 是 | 依赖资源实例标识 |
| `required` | 是 | `true` 缺失即失败；`false` 可按策略忽略 |
| `bind_path` | 是 | 注入到 `spec.values` 的目标路径 |

#### 13.5.2 依赖执行策略

- 展开原则：仅解析请求内显式声明的依赖项，不推断隐式依赖。
- 缺失处理：
  - `required=true` 且无法解析：返回 `AETHER_DEPENDENCY_MISSING`
  - `required=false` 且无法解析：允许跳过该绑定并记录结构化日志
- Aether 不维护“依赖展示模型”，仅负责请求内依赖展开与执行。

#### 13.5.3 依赖删除策略参数

- 枚举值：`RETAIN`、`CASCADE_EXPLICIT`、`BLOCK`。
- `RETAIN`：删除主资源时不处理依赖资源（默认）。
- `CASCADE_EXPLICIT`：仅删除 `delete_options.dependency_ids` 显式列出的依赖。
- `BLOCK`：若请求声明存在依赖关系，直接拒绝删除并返回冲突错误。
- Aether 不内置任何基于依赖“展示分类”的强制级联规则。

### 13.6 验收用例矩阵（FR/NFR/AC -> 测试点）

| TC-ID | 覆盖需求 | 测试场景 | 期望结果 |
| --- | --- | --- | --- |
| TC-001 | FR-API-002、FR-WS-003 | Create 缺少 `cluster_id` | 返回 `AETHER_INVALID_ARGUMENT` |
| TC-002 | FR-API-003、FR-EXEC-001、AC-002 | 提交合法 Create | 同步返回 `ACCEPTED + task_id`，不阻塞部署完成 |
| TC-003 | FR-API-004、FR-Q-002、AC-003 | 通过 `task_id` 查询任务 | 返回当前任务状态与结果，仅支持单任务查询 |
| TC-004 | FR-API-005、FR-ACL-001/002/003 | 非授权角色调用 CUD | 返回 `AETHER_PERMISSION_DENIED` |
| TC-005 | FR-ACL-004 | 同空间不同用户操作同一资源 | 在工作空间维度允许操作，不按创建者隔离 |
| TC-006 | FR-EXEC-003、NFR-003 | 同幂等键重复提交且请求指纹一致 | 返回同一 `task_id/resource_id`，不重复执行 |
| TC-007 | FR-EXEC-003、NFR-003 | 同幂等键重复提交但请求指纹不同 | 返回 `AETHER_CONFLICT` |
| TC-008 | FR-DM-002、FR-RT-006/007 | 请求包含 `required=true` 且缺失依赖 | 返回 `AETHER_DEPENDENCY_MISSING` |
| TC-009 | FR-DM-004、AC-004 | Delete 使用 `RETAIN` | 主资源删除后依赖不被 Aether 强制处理 |
| TC-010 | FR-DM-004 | Delete 使用 `CASCADE_EXPLICIT` 并传依赖列表 | 仅删除显式列出的依赖项 |
| TC-011 | FR-Q-001、FR-API-005 | Query 时 K8S/Helm 不可达 | 返回 `AETHER_QUERY_CONTEXT_UNAVAILABLE` 或 `UNKNOWN` 状态 |
| TC-012 | FR-CORE-001、AC-001 | 六类资源分别执行 CUD + QueryTask | 通过同一套接口完成生命周期操作 |
| TC-013 | FR-CORE-002、NFR-002 | 新增资源类型接入 | 仅新增注册与规则，无需改通用执行核心 |
| TC-014 | FR-INT-001 | 上层模块传入已完成前置校验的请求 | Aether 正常受理，不重复执行业务约束校验 |
| TC-015 | NFR-001 | 技术栈合规检查 | 代码实现满足 Go + Gin + Helm 约束 |
| TC-016 | NFR-004 | CUD 全链路日志检查 | 日志包含操作者、请求摘要、执行结果 |
| TC-017 | AC-001、AC-003 | 六类资源的 QueryTask 权限校验 | 均按工作空间权限边界生效 |
| TC-018 | AC-004、FR-DM-003 | 查询返回中验证依赖展示职责 | 不输出依赖展示语义模型，仅返回任务状态/结果 |
| TC-019 | FR-CORE-004、FR-API-006、AC-005 | 对同一请求执行 values 渲染（数据输出 + 文件输出） | 两种输出语义等价且均可被 Helm 消费 |
| TC-020 | FR-EXEC-004、AC-006 | 使用同一 Chart 与渲染结果对比 Aether Create/Update 与手工 Helm 命令 | 部署结果语义等效 |
| TC-021 | FR-API-005 | values 渲染失败或不满足模板/Schema 约束 | 返回“渲染错误”类别错误码 |
| TC-022 | NFR-005 | 同一请求 + 同一模板版本重复渲染 | 输出语义保持一致，且执行输入可追溯到渲染结果摘要 |
