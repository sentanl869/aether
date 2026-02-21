# Aether 部署模块需求分析与拆解

## 背景与目标

Aether 是 AI Agent 运行平台的部署模块，负责在平台已纳管的 K8S 集群上部署平台内置组件与用户 Agent 应用。
平台通过 kubeconfig 纳管集群，用户上传 Agent 制品到平台后，通过平台完成 Agent 实例、组件实例与应用的部署。

平台内存在“工作空间”概念：一个工作空间可关联一个或多个 K8S 集群，并在每个关联集群中对应一个同名 namespace。

目标：

- 支持平台内置组件（如 milvus、mem0）的高可用部署与参数化配置。
- 支持用户 Agent 应用部署，并可关联已创建的组件实例。
- 支持用户使用自有 Agent 制品（镜像或 Helm Chart）时，直接选择已创建组件实例并一次完成应用创建。
- 支持对 K8S 上的 Agent 实例、Agent 应用进行增删改查（CRUD）。
- 保存应用实例与组件关系，支撑后续应用查看与运维。
- 基于工作空间提供多集群边界管理与权限隔离。

## 需求范围

### In Scope

- 纳管 K8S 集群的部署与运维能力（通过 kubeconfig）。
- 工作空间与集群绑定管理，以及同名 namespace 映射规则。
- 内置组件模板化部署（参数化 + 高可用配置模板）。
- 组件实例创建、更新、删除、查询。
- 组件运行时深度运维（扩缩容、升级/回滚、滚动重启等）。
- Agent 制品类型支持：镜像（Image）与 Helm Chart。
- Agent 实例独立部署与管理（CRUD）。
- Agent 应用部署（配置资源、制品来源、规模等）。
- Agent 应用与组件实例关联并联合部署。
- 应用实例信息、Agent 实例关系与组件关系的持久化。
- 镜像仓库凭证管理（用于推送/拉取镜像）。
- 工作空间隔离（资源边界、权限边界与跨边界关联限制）。
- 权限模型（超级管理员 / 用户）与工作空间级资源操作控制。
- 跨集群与跨 namespace 关联限制。
- Agent 实例、组件实例、Agent 应用的 CUD 采用生产者-消费者异步任务模型，Query 同步返回。

### Out of Scope

- 镜像仓库与 Chart 仓库的搭建及上传流程（由平台其他模块负责）。
- 服务网格等高级网络治理（当前未纳入设计）。

## 实现约束（技术栈）

- 后端实现语言必须使用 Go。
- Web/API 框架必须使用 Gin。
- 持久化数据库必须使用 PostgreSQL（Postgres）。

## 角色与使用场景

- 超级管理员：可操作全部工作空间、集群与资源；纳管集群，维护内置组件模板与 Agent 制品元数据。
- 用户：仅可操作其关联工作空间内资源；可创建组件实例、部署 Agent 应用并关联组件、查看应用状态。

## 关键概念

- 工作空间（Workspace）：资源管理边界。一个工作空间可关联多个 K8S 集群；每个关联集群中均对应一个同名 namespace。
- Agent 制品（Agent Artifact）：用于创建 Agent 实例的部署来源，类型包含镜像（Image）与 Helm Chart。
- 组件（Component）：平台内置中间件/基础能力，如 milvus、mem0。
- 组件实例（Component Instance）：根据模板与配置创建的具体部署。
- Agent 实例（Agent Instance）：根据 Agent 制品部署出的运行实例；可由镜像或 Helm Chart 创建。
- 应用（Application/Agent App）：M 个 Agent 实例 + N 个组件实例的组合体；组件实例可被多个应用复用；保存关系用于查询与运维。

## 需求拆解（功能）

### 1. 集群纳管、工作空间与目标环境

- R-ENV-001：使用 kubeconfig 纳管 K8S 集群。
- R-ENV-002：集群资源可用于部署组件与 Agent 应用。
- R-ENV-003：工作空间与集群建立关联关系后，系统需在每个关联集群中维护同名 namespace（不存在则创建）。
- R-ENV-004：创建组件实例、Agent 实例、Agent 应用时，需先选择工作空间和目标集群；命名空间由工作空间映射决定，不允许手工选择其他 namespace。
- R-ENV-005：镜像仓库凭证应可配置并下发至目标 namespace（用于拉取/推送）。
- R-ENV-006：工作空间与集群解绑采用“冻结 + 校验 + 可恢复窗口 + 回收”机制，不直接立即删除；默认可恢复窗口时长为 24 小时。
- R-ENV-007：解绑操作仅允许超级管理员执行，且必须进行二次确认。

### 2. 平台内置组件部署

- R-CMP-001：组件镜像存放在平台镜像仓库（平台部署后可用）。
- R-CMP-002：提供组件高可用部署模板，技术路线优先级为 Helm > Kustomize > 原生 YAML。
- R-CMP-003：组件实例创建时支持目标集群、资源规格、存储配置
  （SC + 容量）、规模、环境配置、Service 暴露策略
  （ClusterIP/NodePort）等参数，不包含 Ingress 相关配置。
- R-CMP-004：组件实例支持部署、更新、删除、查询能力。
- R-CMP-005：组件实例支持状态监控（部署成功/失败、运行态、事件摘要）。
- R-CMP-006：组件实例支持扩缩容、升级/回滚、滚动重启与健康检查管理。
- R-CMP-007：Service 命名策略：
  - 在“应用创建时直接关联组件实例”场景下，Service 名称由平台自动生成。
  - 在“独立创建组件实例”且用户显式选择自定义时，允许用户自定义 Service 名称（同 namespace 唯一）。

### 3. Agent 制品、Agent 实例与 Agent 应用部署

- R-AGT-001：平台需支持识别并使用两类 Agent 制品：镜像（Image）与 Helm Chart（上传流程由其他模块负责）。
- R-AGT-002：用户可基于 Agent 制品独立创建 Agent 实例；同一工作空间可同时存在镜像型与 Chart 型实例。
- R-AGT-003：Agent 实例部署支持目标集群、资源规格、存储配置（SC + 容量）、规模、必要环境变量。
- R-AGT-004：当 Agent 制品类型为 Helm Chart 时，需支持 Chart
  版本选择与 `values` 参数覆盖；部署参数优先级中，前端提交参数为
  最高优先级，效果等价于 `helm install -f values.yaml`。
- R-AGT-005：Chart 依赖校验需调用接口检查“工作空间关联镜像仓库”内依赖镜像是否存在；OCI Chart 源校验策略采用默认策略。
- R-APP-001：Agent 应用创建支持两种方式：
  - 选择已创建的一个或多个 Agent 实例，再关联已创建组件实例并创建应用。
  - 使用一个或多个 Agent 制品，在一次提交中完成 Agent 实例创建、组件关联与应用创建。
- R-APP-002：Application 与 Agent 实例关系为一对多（1:N）；同一 Agent 实例在同一时刻仅允许归属一个 Application。
- R-APP-003：组件关联限制：仅允许关联同一工作空间、同一集群（同一 namespace）下组件实例，不允许跨集群。
- R-APP-004：部署时自动生成 Deployment/StatefulSet、Service、
  ConfigMap/Secret（如需），并注入组件连接信息
  （敏感字段统一通过 Secret）。
- R-APP-005：Application 状态以应用自身原子任务结果为准；Query 查询不进行跨资源实时聚合计算。

### 4. 应用关系与持久化

- R-DATA-001：应用作为组合体，保存 M 个 Agent 实例 + N 个组件实例关系。
- R-DATA-002：应用创建后保存应用基本信息、Agent 实例列表与部署参数、关联组件列表及引用方式。
- R-DATA-003：应用查看时展示 Agent 集合与组件集合的独立状态、连接信息（Endpoint、Service 名称等）；不在查询阶段做聚合计算。

### 5. CRUD、权限与异步任务

- R-OPS-001：Agent 实例、Agent 应用、组件实例的 CUD 统一走异步任务模型；查询同步返回，且查询不做跨资源聚合。资源操作语义保持原子性。
- R-OPS-002：权限模型固定为“超级管理员 + 用户”。
- R-OPS-003：同一工作空间内用户可对该工作空间所有资源进行 CRUD。
- R-OPS-004：工作空间为资源与权限隔离边界（namespace 映射 + RBAC + 跨边界关联限制），当前版本不承诺网络层默认隔离。
- R-OPS-005：不允许跨 namespace 关联组件实例；同一应用不可跨 K8S 集群部署。
- R-OPS-006：应用删除默认不级联删除关联组件实例；支持用户显式选择是否级联删除。
- R-OPS-007：当用户选择级联删除时，范围包含：关联组件实例、关联 Agent 实例、PVC、Secret、ConfigMap（及与应用绑定的派生资源）。

## 领域模型与唯一性约束（设计输入）

- `Workspace`
  - 主标识：`workspace_id`
  - 唯一约束：`workspace_name` 全局唯一
  - 删除策略：软删
  - 备注：名称用于 namespace 映射
- `ManagedCluster`
  - 主标识：`cluster_id`
  - 唯一约束：`cluster_name` 全局唯一
  - 删除策略：软删
  - 备注：kubeconfig 独立存储
- `WorkspaceClusterBinding`
  - 主标识：`binding_id`
  - 唯一约束：`(workspace_id, cluster_id)` 唯一
  - 删除策略：软删
  - 备注：维护 `namespace_name`
- `AgentArtifact`
  - 主标识：`artifact_id`
  - 唯一约束：
    `(workspace_id, artifact_name, artifact_version, artifact_type)` 唯一
  - 删除策略：软删
  - 备注：`artifact_type` in `{image, helm_chart}`
- `ComponentTemplate`
  - 主标识：`template_id`
  - 唯一约束：`(component_name, template_version)` 唯一
  - 删除策略：软删
  - 备注：保存参数 schema 与渲染路线
- `ComponentInstance`
  - 主标识：`component_instance_id`
  - 唯一约束：`(workspace_id, cluster_id, component_instance_name)` 唯一
  - 删除策略：软删
  - 备注：Service 名称需满足
    `(cluster_id, namespace, service_name)` 唯一
- `AgentInstance`
  - 主标识：`agent_instance_id`
  - 唯一约束：`(workspace_id, cluster_id, agent_instance_name)` 唯一
  - 删除策略：软删
  - 备注：由 `artifact_id` 创建
- `Application`
  - 主标识：`application_id`
  - 唯一约束：`(workspace_id, cluster_id, application_name)` 唯一
  - 删除策略：软删
  - 备注：组合多个 Agent 实例
- `ApplicationAgentRef`
  - 主标识：`id`
  - 唯一约束：`(application_id, agent_instance_id)` 唯一
  - 删除策略：硬删
  - 备注：`agent_instance_id` 需全局唯一归属一个 Application
- `ApplicationComponentRef`
  - 主标识：`id`
  - 唯一约束：`(application_id, component_instance_id)` 唯一
  - 删除策略：硬删
  - 备注：组件可复用
- `AsyncTask`
  - 主标识：`task_id`
  - 唯一约束：`(idempotency_key)` 唯一
  - 删除策略：不删除（归档）
  - 备注：记录请求与执行结果
- `SecretVersion`
  - 主标识：`secret_version_id`
  - 唯一约束：`(workspace_id, namespace, secret_name, version)` 唯一
  - 删除策略：软删
  - 备注：仅允许平台控制面读取明文

补充规则：

- Application 创建时必须至少关联 1 个 Agent 实例。
- 删除 Application 且 `cascade=true` 时，若某组件仍被其他
  Application 引用（引用计数 > 1），本次操作返回 `409 Conflict`，
  不执行该组件删除。
- 删除 Application 且 `cascade=true` 时，删除该 Application 关联的 Agent 实例与独占引用组件实例及派生资源。
- 资源软删后保留审计字段：`deleted_at`、`deleted_by`、`delete_task_id`。

## 资源状态机（设计输入）

### 资源状态（ComponentInstance / AgentInstance / Application）

- `Creating`：已受理创建任务，资源尚未达到可用状态。
- `Running`：资源可用，健康检查通过。
- `Updating`：资源进行升级、配置变更或扩缩容。
- `Degraded`：资源部分可用，存在告警或部分副本异常。
- `Failed`：最近一次创建或更新失败且未恢复。
- `Deleting`：资源删除任务进行中。
- `Deleted`：资源已删除（逻辑删除完成）。
- `DeleteFailed`：删除任务失败，需重试或人工干预。

状态迁移规则：

- 创建成功：`Creating -> Running`
- 创建失败：`Creating -> Failed`
- 更新成功：`Updating -> Running`
- 更新失败：`Updating -> Degraded` 或 `Updating -> Failed`（根据可用副本判断）
- 删除成功：`Deleting -> Deleted`
- 删除失败：`Deleting -> DeleteFailed`
- 恢复操作成功：`Degraded/Failed/DeleteFailed -> Running`

Application 状态规则（非聚合）：

- Application 状态由 Application 自身任务结果驱动，不在 Query 阶段对成员资源做实时聚合。
- Application 创建/更新/删除任务失败时，Application 状态进入 `Failed` 或 `DeleteFailed`。
- 成员 Agent 实例与组件实例状态作为独立资源状态展示，不反向实时重算 Application 状态。

### 异步任务状态（AsyncTask）

- 状态集合：`Pending`、`Running`、`Succeeded`、`Failed`、`Canceled`。
- 结果字段最小集合：`resource_type`、`resource_id`、`started_at`、`ended_at`、`failure_reason`、`retry_count`、`serialized_key`。

任务执行规则：

- CUD 请求必须携带 `Idempotency-Key`；同 key 的重复请求在 24 小时内返回相同 `task_id`。
- 串行化键为 `workspace_id:cluster_id:resource_type:resource_id_or_name`，同键任务必须串行。
- 重试策略：最多 5 次，指数退避（5s、15s、45s、135s、300s）。
- 超时策略：创建/更新默认 30 分钟，删除默认 15 分钟；超时后标记 `Failed`。
- 取消策略：`Pending` 任务可取消；`Running` 任务仅在执行器标记可中断时可取消，取消后置 `Canceled`。

## 接口契约要求（设计输入）

### 通用契约

- API 前缀统一为 `/api/v1`。
- CUD 接口必须支持 `Idempotency-Key` 请求头。
- 更新/删除接口必须使用并发控制字段 `resource_version`。
- Query 接口同步返回，支持分页（`page`、`page_size`）、过滤、排序；不执行跨资源状态聚合。
- 返回统一结构：`request_id`、`code`、`message`、`data`。

### 错误码最小集合

- `400`：参数非法或 schema 校验失败。
- `401`：未认证。
- `403`：无权限。
- `404`：资源不存在。
- `409`：并发冲突或引用冲突（如级联删除碰到共享组件）。
- `422`：业务规则校验失败（如跨集群关联、应用未关联 Agent 实例）。
- `429`：请求限流。
- `500`：系统内部错误。

### 资源接口分组

- `GET /agent-artifacts`、`GET /agent-artifacts/{id}`（制品查询，支持 Image/Helm Chart）
- `POST/GET/LIST/PATCH/DELETE /agent-instances`
- `POST/GET/LIST/PATCH/DELETE /component-instances`
- `POST/GET/LIST/PATCH/DELETE /applications`
- `GET /applications/{id}/relations`
- `GET /component-templates`、`GET /component-templates/{id}`
- `GET /tasks/{task_id}`、`GET /tasks/{task_id}/result`

## 权限动作矩阵（设计输入）

| 资源/动作 | 超级管理员 | 用户（已关联 workspace） | 用户（未关联 workspace） |
| --- | --- | --- | --- |
| 集群纳管与解绑 | 允许 | 拒绝 | 拒绝 |
| 工作空间与集群绑定关系查看 | 允许 | 允许（仅本 workspace） | 拒绝 |
| Agent 制品查询（Image/Chart） | 允许 | 允许（仅本 workspace） | 拒绝 |
| 组件模板维护 | 允许 | 拒绝 | 拒绝 |
| 组件实例 CRUD | 允许 | 允许（仅本 workspace） | 拒绝 |
| Agent 实例 CRUD | 允许 | 允许（仅本 workspace） | 拒绝 |
| Application CRUD | 允许 | 允许（仅本 workspace） | 拒绝 |
| 任务状态与结果查询 | 允许 | 允许（仅本 workspace） | 拒绝 |
| Secret 明文读取 | 拒绝（仅平台控制面服务账号可读） | 拒绝 | 拒绝 |
| Secret 引用（不含明文） | 允许 | 允许（仅本 workspace） | 拒绝 |

## Secret 生命周期与审计（设计输入）

- S-SEC-001：Secret 必须保存在工作空间对应 namespace 内。
- S-SEC-002：普通用户不可查看 Secret 明文，仅可在资源创建/更新时引用 `secret_ref`。
- S-SEC-003：Secret 采用版本化命名：`{secret_name}.v{n}`，`n` 为递增整数。
- S-SEC-004：更新 Secret 时写入新版本并切换“当前版本”指针，旧版本默认保留 10 个版本。
- S-SEC-005：必须支持版本回滚（指定历史版本为当前版本）。
- S-SEC-006：Secret 加密暂不引入额外 KMS 或数据库加密方案，使用 K8S Secret 默认机制。
- S-SEC-007：审计字段最小集合：`secret_name`、`version`、`action`、`actor`、`workspace_id`、`cluster_id`、`timestamp`、`request_id`、`result`。

## 非功能需求（量化）

### 性能与可用性

- NFR-001：CUD 提交接口（仅提交任务）P95 响应时间 <= 300ms，P99 <= 800ms。
- NFR-002：Query 接口（非聚合查询）P95 响应时间 <= 800ms，P99 <= 2s（不含大规模导出）。
- NFR-003：任务从 `Pending` 到 `Running` 的排队等待时间 P95 <= 5s。
- NFR-004：常规创建/更新任务完成时间 P95 <= 10 分钟；删除任务完成时间 P95 <= 5 分钟。
- NFR-005：控制面月度可用性 >= 99.9%。

### 容量

- NFR-006：支持 >= 100 个工作空间。
- NFR-007：支持 >= 50 个纳管集群。
- NFR-008：支持 >= 5000 个组件实例、>= 5000 个 Agent 实例、>= 5000 个应用实例。
- NFR-009：支持日均 >= 10000 个异步任务提交。
- NFR-010：支持至少 30% 的 Agent 实例来自 Helm Chart 制品，并保持与镜像实例同等运维可观测能力。

### 安全与审计

- NFR-011：敏感信息（密码、Token、Key）不得明文存储。
- NFR-012：审计日志保留期 >= 180 天，任务执行日志保留期 >= 30 天。
- NFR-013：所有跨工作空间、跨集群、跨 namespace 关联请求必须被拒绝并记录审计。
- NFR-014：Agent Chart 制品元数据需记录来源与版本，支持审计追溯。

### 可观测性与运维

- NFR-015：任务、资源状态、关键事件需可查询；状态更新延迟 <= 30s。
- NFR-016：失败任务需提供可诊断失败原因（错误码、错误信息、失败步骤）。
- NFR-017：模板参数必须具备 schema 校验能力，非法参数在提交阶段返回错误。

## 验收标准与追踪矩阵

| 需求 ID | 验收条件 | 测试类型 | 责任方 |
| --- | --- | --- | --- |
| R-ENV-003 | 绑定 workspace 与 cluster 后，自动创建/复用同名 ns | 集成测试 | 后端 |
| R-ENV-006 | 解绑流程必须经历冻结、校验、恢复窗口、回收四阶段并可审计 | 集成测试 + E2E | 后端 + 运维 |
| R-AGT-001 | Agent 制品类型同时支持 Image 与 Helm Chart，并可被实例创建流程识别 | 集成测试 | 后端 |
| R-AGT-004 | Chart 型实例可按“前端参数最高优先级”应用 `values` | 集成测试 + E2E | 后端 |
| R-AGT-005 | Chart 依赖可通过接口校验工作空间关联镜像仓库中依赖镜像是否存在 | 集成测试 | 后端 |
| R-CMP-006 | 组件实例支持扩缩容、升级/回滚、滚动重启并反馈任务状态 | 集成测试 | 后端 |
| R-APP-001 | 应用创建两种模式均可成功提交并返回 `task_id` | E2E | 前端 + 后端 |
| R-APP-002 | 同一应用可关联多个 Agent 实例，且单个 Agent 实例不可被多个应用同时占用 | 集成测试 | 后端 |
| R-APP-003 | 跨 cluster 或跨 namespace 关联组件请求返回 `422` | 单元测试 + 集成测试 | 后端 |
| R-OPS-001 | CUD 异步、Query 同步且不聚合行为符合接口契约 | E2E | 前端 + 后端 |
| R-OPS-006 | 删除应用默认不级联组件实例 | 集成测试 | 后端 |
| R-OPS-007 | 级联删除遇到共享组件时返回 `409` 并不删除共享组件 | 集成测试 | 后端 |
| S-SEC-004 | Secret 更新后新版本生效，旧版本可追溯 | 集成测试 | 后端 |
| NFR-001~005 | 性能和可用性满足量化指标 | 压测 + 运行验收 | SRE + 后端 |

## 交付物与接口（面向设计）

- OpenAPI 3.0 草案（包含错误码、并发控制、幂等约束）。
- 领域模型与 ERD（含唯一性约束、索引建议、软删策略）。
- 状态机定义（资源状态机、任务状态机、解绑流程状态机）。
- 异步任务设计说明（串行化键、重试、超时、取消、补偿）。
- 权限矩阵与鉴权流程说明（角色、workspace 边界、审计点）。
- Agent 制品适配设计（Image 与 Helm Chart 的统一抽象与运行时差异处理）。
- NFR 与容量验收报告模板。

## 需求管理

- 需求持续演进，不设置冻结点；变更需同步更新本文件与 `docs/decision.md`。
