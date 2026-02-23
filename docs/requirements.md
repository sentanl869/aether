# Aether 部署模块需求分析与拆解

## 背景与目标

Aether 是 AI Agent 运行平台的部署模块，负责在平台已纳管的 K8S 集群上部署平台内置资源与用户 Agent 应用，并提供统一的生命周期管理能力。

平台通过 kubeconfig 纳管集群。超级管理员创建工作空间并关联集群、镜像仓库后，用户在工作空间边界内完成资源创建与应用发布。

平台同时支持两类 Agent 应用形态：

- 高代码 Agent 应用：
  - 由 DevBox 产出的已发布镜像创建的 Agent 实例，或用户自行上传镜像创建的 Agent 实例，
    与平台内提前创建的数据服务组件实例组合形成对外服务 Pod 集合。
  - 或用户上传 Helm Chart，部署一个或多个 Agent 实例，并可关联平台内数据服务组件；Chart 也可能自带数据服务组件。
- 低代码 Agent 应用：
  - 运行在 Dify/FastGPT/Coze Studio/n8n 等低代码平台中，由用户在低代码平台内部创建并对外提供服务。
  - Aether 负责低代码平台实例部署与生命周期，不解析低代码平台内部应用 DSL。

目标：

- 支持 5 类资源的统一 CRUD：数据服务组件、低代码平台、DevBox、网关、高代码应用。
- 在“都部署到 K8S Pod”的前提下建立统一抽象，保证新增资源类型时可复用同一部署行为。
- 明确共享资源与“应用内嵌资源”的可见性与管理边界，避免资源混管。
- 支持高代码应用发布时自动生成 Helm Chart、版本化入库并可下载导出。
- 基于工作空间实现多集群租户边界、权限边界和关联约束。

## 需求范围

### In Scope

- 纳管 K8S 集群（kubeconfig）及工作空间绑定管理。
- 工作空间与集群、镜像仓库关联；同名 namespace 映射与维护。
- 5 类资源的 CRUD、状态管理、异步任务执行与审计。
- 平台内置模板（数据服务组件、Dify/FastGPT、DevBox 框架、Higress）管理与部署。
- 低代码平台 Helm 导入能力（Coze Studio、n8n、外部 Dify/FastGPT）。
- 高代码应用三类创建来源：DevBox 发布镜像、用户上传镜像、用户上传 Helm Chart。
- 高代码应用与数据服务组件关系管理、级联删除策略、引用冲突处理。
- 高代码应用发布 Chart 的自动生成、版本记录、仓库存储、下载导出。
- 统一资源抽象与统一部署执行链路（可扩展到未来新资源类型）。

### Out of Scope

- 镜像仓库与 Chart 仓库基础设施搭建（由其他模块负责）。
- Dify/FastGPT/Coze Studio/n8n 内部工作流/Agent 编排能力实现。
- Ingress/服务网格等高级网络治理能力。

## 实现约束（技术栈）

- 后端实现语言必须使用 Go。
- Web/API 框架必须使用 Gin。
- 持久化数据库必须使用 PostgreSQL（Postgres）。
- 平台内置模板与部署包统一采用 Helm Chart 作为标准形态。

## 角色与租户边界

- 超级管理员：
  - 创建/管理工作空间。
  - 关联工作空间与纳管集群、镜像仓库。
  - 管理平台内置模板与导入低代码平台 Helm Chart。
  - 管理网关、低代码平台开通策略。
- 普通用户：
  - 仅能操作已关联工作空间内的资源。
  - 可创建数据服务组件、DevBox、高代码应用并执行发布流程。
  - 可使用已开通的低代码平台入口，但不可越权管理其他工作空间资源。

## 资源分层与关键概念

- 工作空间（Workspace）：资源与权限边界；每个关联集群中映射一个同名 namespace。
- 一级资源类型（L1 Resource Kind）：部署模块对外提供独立业务语义与管理边界的资源，固定 5 类：
  - 数据服务组件（Postgres、Redis、mem0、milvus、weaviate、pgvector、neo4j、NebulaGraph、MongoDB、MinIO）
  - 低代码平台（Dify、FastGPT、Coze Studio、n8n）
  - DevBox
  - 网关（Higress）
  - 高代码应用（openclaw、zeroclaw 等）
- 支撑资源（Supporting Resource）：服务于一级资源的领域对象，包括
  `AgentInstance`、`HighCodeArtifact`、`DevBoxPublishRecord`、
  `HighCodeReleaseChart` 及各类关系引用对象。部分支撑资源可提供独立 CRUD
  接口，但不计入“一级资源类型”口径。
- 统一运行抽象（Runtime Abstraction）：一级资源与支撑资源最终都映射为
  K8S Workload/Service/PVC/Secret 等对象集合；Pod 仅作为运行时观测对象，
  不作为控制面领域主实体。
- 资源模板（Template）：平台内置或管理员导入的 Helm Chart 模板与参数 schema。
- 资源实例（Instance）：模板或制品在指定工作空间/集群/namespace 的运行实体。
- 高代码制品（HighCode Artifact）：用于高代码应用部署的镜像或 Helm Chart 来源。
- 嵌入式数据服务（Embedded Data Service）：
  - 随低代码平台部署自动创建的数据服务实例。
  - 或随高代码 Helm Chart 自带创建的数据服务实例。
  - 仅归属其宿主资源，不进入“共享数据服务组件实例池”。

## 需求拆解（功能）

### 1. 工作空间、集群与镜像仓库

- R-ENV-001：平台通过 kubeconfig 纳管 K8S 集群。
- R-ENV-002：仅超级管理员可创建工作空间并关联一个或多个纳管集群、镜像仓库。
- R-ENV-003：一个工作空间在每个关联集群必须映射同名 namespace；若不存在则自动创建。
- R-ENV-004：用户只能操作与其关联工作空间内资源；禁止跨工作空间操作。
- R-ENV-005：创建任意资源实例时必须先选择工作空间与目标集群；namespace 由映射规则确定，禁止自定义其他 namespace。
- R-ENV-006：工作空间与镜像仓库关联关系必须可审计、可变更、可回滚。
- R-ENV-007：镜像仓库凭证可下发到工作空间 namespace，用于镜像与 Chart 推拉。
- R-ENV-008：同一应用内禁止跨集群部署与关联。
- R-ENV-009：工作空间与集群解绑采用“冻结 + 校验 + 24h 可恢复窗口 + 回收”机制。
- R-ENV-010：解绑仅允许超级管理员执行，必须二次确认并记录审计。

### 2. 统一资源抽象与可扩展部署行为

- R-ABS-001：5 类资源必须统一映射到“模板/制品 + 实例 + 关系 + 发布记录”抽象模型。
- R-ABS-002：5 类资源的 CUD 必须复用统一执行链路：参数校验 -> 模板渲染
  -> K8S Apply/Upgrade/Delete -> 状态回写 -> 审计落库。
- R-ABS-003：部署驱动采用可扩展适配器机制，默认 Helm 适配器；镜像型高代码应用通过平台生成 Chart 后复用 Helm 链路。
- R-ABS-004：新增资源类型时，不得重写任务队列、鉴权、审计、状态机主流程，只允许新增资源 schema 与适配器实现。
- R-ABS-005：统一资源标签规范：`workspace_id`、`cluster_id`、`resource_kind`、`resource_id`、`owner_kind`、`owner_id`。
- R-ABS-006：所有资源的可观测字段（状态、事件、最近任务）结构统一，便于通用列表与运维视图复用。
- R-ABS-007：领域模型采用“一级资源 + 支撑资源”分层；5 类一级资源口径用于权限矩阵、配额统计、审计报表与对外能力边界。
- R-ABS-008：控制面不得以 Pod 作为业务主实体；Pod/ReplicaSet/Job
  等仅作为运行时观测对象，并通过 `resource_kind/resource_id`
  回挂到领域实例。

### 3. 数据服务组件

- R-DSP-001：平台内置数据服务组件类型固定为：
  Postgres、Redis、mem0、milvus、weaviate、pgvector、neo4j、NebulaGraph、MongoDB、MinIO。
- R-DSP-002：上述组件模板全部由平台内置并以 Helm Chart 形式提供。
- R-DSP-003：共享数据服务组件实例支持独立创建、更新、删除、查询。
- R-DSP-004：高代码应用创建时可关联已创建的共享数据服务组件实例。
- R-DSP-005：组件实例部署参数支持：目标集群、资源规格、存储（SC + 容量）、副本规模、环境配置、Service 暴露策略（ClusterIP/NodePort）。
- R-DSP-006：组件实例支持扩缩容、升级/回滚、滚动重启与健康检查管理。
- R-DSP-007：Service 命名策略：
  - 应用创建时直接关联场景由平台自动生成。
  - 独立创建场景允许用户显式自定义（同 namespace 唯一）。
- R-DSP-008：随低代码平台部署自动创建的数据服务实例必须标记为 `embedded_by_lowcode_platform`，不在共享组件列表中展示。
- R-DSP-009：随高代码 Helm Chart 自带创建的数据服务实例必须标记为 `embedded_by_highcode_chart`，不在共享组件列表中展示。
- R-DSP-010：嵌入式数据服务实例仅允许其宿主资源访问与运维，不允许被其他高代码应用或低代码平台复用绑定。
- R-DSP-011：共享组件与嵌入式组件必须在 API、控制台列表和权限校验上完全隔离展示。
- R-DSP-012：删除宿主资源时，嵌入式组件按宿主生命周期回收，不参与共享组件引用计数。

### 4. 低代码平台

- R-LCP-001：平台内置低代码平台模板包含 Dify 与 FastGPT，模板格式为 Helm Chart。
- R-LCP-002：超级管理员可在工作空间内开通内置低代码平台，部署到对应集群/namespace。
- R-LCP-003：超级管理员可导入 Coze Studio、n8n 或外部 Dify/FastGPT，导入格式仅支持 Helm Chart。
- R-LCP-004：低代码平台实例支持 CRUD、状态查询、版本升级与回滚。
- R-LCP-005：内置 Dify/FastGPT 部署时，其依赖数据服务组件随平台一起创建并标记为嵌入式。
- R-LCP-006：低代码平台依赖组件不可与用户手工创建的共享数据服务组件混合展示或统一运维。
- R-LCP-007：低代码平台实例需记录入口地址、管理员账号引用、依赖组件拓扑与版本信息。
- R-LCP-008：同一工作空间同一集群内，同一低代码平台类型默认仅允许一个活动实例（可配置是否允许多实例）。
- R-LCP-009：普通用户可查看其工作空间内低代码平台实例运行状态与访问入口，不可执行超管限定动作。
- R-LCP-010：Aether 不接管低代码平台内部应用对象 CRUD，仅保证平台实例层可用性。

### 5. DevBox

- R-DBX-001：DevBox 提供高代码 Agent 开发环境，内置框架模板以 Helm Chart 形式维护。
- R-DBX-002：用户可基于模板在目标工作空间/集群启动开发容器实例。
- R-DBX-003：DevBox 实例支持 CRUD（创建、配置更新、停止/删除、查询）。
- R-DBX-004：DevBox 需支持代码仓挂载、资源配额、环境变量与密钥引用配置。
- R-DBX-005：用户在 DevBox 内完成开发后，可在平台触发镜像发布流程。
- R-DBX-006：仅“已发布镜像”允许作为高代码应用创建来源；未发布镜像禁止入选。
- R-DBX-007：镜像发布需记录发布版本、镜像地址、digest、发布人、发布时间与关联 DevBox 实例。
- R-DBX-008：DevBox 模板与实例要支持版本演进与兼容性检查，避免旧模板参数直接破坏运行实例。
- R-DBX-009：DevBox 删除不自动删除其历史发布镜像与发布记录。

### 6. 网关（Higress）

- R-GTW-001：网关用于提供应用发布、AI 网关、MCP 网关三类能力。
- R-GTW-002：网关部署模板由平台内置并以 Helm Chart 形式提供。
- R-GTW-003：同一工作空间在同一集群（同 namespace）最多只能存在一个网关实例。
- R-GTW-004：网关实例支持 CRUD、状态查询、版本升级与回滚。
- R-GTW-005：高代码/低代码应用对外发布前，必须校验目标工作空间网关实例可用性。
- R-GTW-006：网关配置变更需支持灰度生效与回滚，并记录变更审计。

### 7. 高代码应用

- R-HCA-001：高代码应用支持三类创建渠道：
  - DevBox 已发布镜像。
  - 用户上传镜像。
  - 用户上传 Helm Chart。
- R-HCA-002：镜像渠道部署形态为“1 个 Agent 实例 + 0..N 个共享数据服务组件”。
- R-HCA-003：Helm Chart 渠道部署形态为“1..N 个 Agent 实例 + 0..N 个共享数据服务组件 + 0..N 个 Chart 自带嵌入式组件”。
- R-HCA-004：高代码应用创建支持两种流程：
  - 先创建 Agent 实例，再关联组件创建应用。
  - 一次提交中完成 Agent 实例创建、组件关联与应用创建。
- R-HCA-005：Application 与 AgentInstance 关系为 1:N；同一 AgentInstance 同时仅允许归属一个 Application。
- R-HCA-006：关联限制：仅允许同一工作空间、同一集群（同 namespace）的共享数据服务组件。
- R-HCA-007：Chart 型高代码应用需支持 Chart 版本选择、values 覆盖、依赖镜像校验（工作空间关联仓库）。
- R-HCA-008：部署时自动生成/管理 Deployment/StatefulSet、Service、ConfigMap/Secret，并注入组件连接信息。
- R-HCA-009：敏感配置统一通过 Secret 引用注入，普通用户不可读取明文。
- R-HCA-010：高代码应用支持 CRUD、状态查询、扩缩容、升级/回滚、滚动重启。
- R-HCA-011：删除应用默认不级联删除共享数据服务组件；支持显式 `cascade=true`。
- R-HCA-012：当 `cascade=true` 且共享组件仍被其他应用引用时返回 `409 Conflict` 并阻断该组件删除。
- R-HCA-013：`cascade=true` 时删除该应用关联的 Agent 实例与独占派生资源（PVC/Secret/ConfigMap 等）。
- R-HCA-014：Chart 自带嵌入式组件随该应用生命周期管理，不进入共享组件池。

### 8. 高代码应用发布与 Helm Chart 沉淀

- R-PKG-001：用户在平台发布高代码应用时，平台必须生成该应用对应 Helm Chart 包（标准化导出包）。
- R-PKG-002：生成的应用 Chart 必须版本化管理，并与应用发布记录一一关联。
- R-PKG-003：应用 Chart 版本需推送并保存到工作空间关联镜像仓库（OCI Artifact）。
- R-PKG-004：平台需提供应用 Chart 下载能力，支持用户在外部环境导入部署。
- R-PKG-005：应用 Chart 元数据最小集合：
  `application_id`、`release_version`、`chart_ref`、`source_type`、`source_version`、`digest`、`created_by`、`created_at`。
- R-PKG-006：镜像型来源与 Helm 来源都必须产出可复现的导出 Chart（包含渲染后必要 values/依赖锁定信息）。

### 9. 应用关系持久化与资源可见性

- R-DATA-001：高代码应用保存“Agent 实例集合 + 共享数据服务组件集合 + 嵌入式组件集合”关系。
- R-DATA-002：低代码平台保存“平台实例 + 嵌入式组件集合”关系。
- R-DATA-003：关系查询需区分 `shared` 与 `embedded` 资源类型，并标注 `owner_kind/owner_id`。
- R-DATA-004：查询接口不做跨资源实时聚合计算；应用状态由应用自身任务结果驱动。
- R-DATA-005：资源软删后保留审计字段：`deleted_at`、`deleted_by`、`delete_task_id`。
- R-DATA-006：嵌入式资源不得出现在共享资源选择器中。
- R-DATA-007：应用查看页必须展示 Agent 集合、组件集合、入口地址、发布 Chart 版本信息。
- R-DATA-008：关系变更（绑定/解绑/级联删除）必须记录审计轨迹。

### 10. CRUD、权限与异步任务

- R-OPS-001：数据服务组件、低代码平台、DevBox、网关、高代码应用 5 类资源的 CUD 统一走异步任务模型；Query 同步返回。
- R-OPS-002：所有 CUD 请求必须支持 `Idempotency-Key`；
  控制面以 `idempotency_scope` 作为 24h 去重唯一键，
  同 scope+同请求指纹返回同一 `task_id`，
  同 scope+异指纹返回 `409 IDEMPOTENCY_PAYLOAD_MISMATCH`。
- R-OPS-003：权限模型固定为“超级管理员 + 普通用户”，并按工作空间做资源边界控制。
- R-OPS-004：用户只能操作其已关联工作空间资源；跨工作空间、跨集群、跨 namespace 请求必须拒绝并审计。
- R-OPS-005：同资源串行化执行，防止并发更新破坏状态一致性。
- R-OPS-006：更新/删除必须使用 `resource_version` 并发控制。
- R-OPS-007：任务需支持重试、超时、取消与失败可诊断信息输出。
- R-OPS-008：任务执行结果需包含最小字段：`resource_kind`、`resource_id`、`started_at`、`ended_at`、`failure_reason`。
- R-OPS-009：删除宿主资源时需同时回收其嵌入式资源；回收失败需进入补偿流程并可重试。
- R-OPS-010：所有关键操作（创建、更新、删除、发布、导出）必须记录审计日志。

## 领域模型与唯一性约束（设计输入）

模型分层约束：

- 一级资源实体固定为：`DataServiceInstance`、`LowCodePlatformInstance`、`DevBoxInstance`、`GatewayInstance`、`HighCodeApplication`。
- 支撑资源实体包括：`AgentInstance`、`HighCodeArtifact`、`DevBoxPublishRecord`、`HighCodeReleaseChart`、关系引用实体、任务实体与密钥版本实体。
- 一级资源承担平台一级能力边界；支撑资源可独立存储或独立接口化，但不改变“5 类一级资源”口径。

- `Workspace`
  - 主标识：`workspace_id`
  - 唯一约束：`workspace_name` 全局唯一
  - 删除策略：软删
- `ManagedCluster`
  - 主标识：`cluster_id`
  - 唯一约束：`cluster_name` 全局唯一
  - 删除策略：软删
- `WorkspaceClusterBinding`
  - 主标识：`binding_id`
  - 唯一约束：`(workspace_id, cluster_id)` 唯一
  - 删除策略：软删
  - 备注：维护 `namespace_name`
- `WorkspaceRegistryBinding`
  - 主标识：`registry_binding_id`
  - 唯一约束：`(workspace_id, registry_id)` 唯一
  - 删除策略：软删
- `ResourceTemplate`
  - 主标识：`template_id`
  - 唯一约束：`(template_kind, template_name, template_version)` 唯一
  - 删除策略：软删
  - 备注：`template_kind` in `{data_service, lowcode_platform, devbox, gateway}`
- `HighCodeArtifact`
  - 主标识：`artifact_id`
  - 唯一约束：`(workspace_id, artifact_name, artifact_version, artifact_type)` 唯一
  - 删除策略：软删
  - 备注：`artifact_type` in `{devbox_published_image, uploaded_image, uploaded_helm_chart}`
- `DataServiceInstance`
  - 主标识：`data_service_instance_id`
  - 唯一约束：`(workspace_id, cluster_id, instance_name)` 唯一
  - 删除策略：软删
  - 备注：`visibility` in `{shared, embedded}`；`owner_kind/owner_id`
    可空或必填（embedded 必填）
- `LowCodePlatformInstance`
  - 主标识：`lowcode_platform_instance_id`
  - 唯一约束：`(workspace_id, cluster_id, platform_type, instance_name)` 唯一
  - 删除策略：软删
- `DevBoxInstance`
  - 主标识：`devbox_instance_id`
  - 唯一约束：`(workspace_id, cluster_id, instance_name)` 唯一
  - 删除策略：软删
- `DevBoxPublishRecord`
  - 主标识：`publish_id`
  - 唯一约束：`(devbox_instance_id, publish_version)` 唯一
  - 删除策略：硬删禁止（仅归档）
- `GatewayInstance`
  - 主标识：`gateway_instance_id`
  - 唯一约束：`(workspace_id, cluster_id)` 唯一
  - 删除策略：软删
- `HighCodeApplication`
  - 主标识：`application_id`
  - 唯一约束：`(workspace_id, cluster_id, application_name)` 唯一
  - 删除策略：软删
- `AgentInstance`
  - 主标识：`agent_instance_id`
  - 唯一约束：`(workspace_id, cluster_id, agent_instance_name)` 唯一
  - 删除策略：软删
  - 备注：由 `artifact_id` 创建
- `HighCodeApplicationAgentRef`
  - 主标识：`id`
  - 唯一约束：`(application_id, agent_instance_id)` 唯一
  - 删除策略：硬删
  - 备注：`agent_instance_id` 需唯一归属一个 `application_id`
- `HighCodeApplicationDataServiceRef`
  - 主标识：`id`
  - 唯一约束：`(application_id, data_service_instance_id)` 唯一
  - 删除策略：硬删
- `HighCodeReleaseChart`
  - 主标识：`release_chart_id`
  - 唯一约束：`(application_id, release_version)` 唯一
  - 删除策略：软删
  - 备注：记录 chart OCI 地址、digest、导出包元数据
- `AsyncTask`
  - 主标识：`task_id`
  - 唯一约束：`(idempotency_scope)` 唯一（24h 去重窗口内）
  - 删除策略：不删除（归档）
  - 备注：`idempotency_key` 保留原始请求头审计语义；`request_fingerprint` 用于同键异载荷冲突判定；`idempotency_expire_at` 用于 TTL 清理。
- `SecretVersion`
  - 主标识：`secret_version_id`
  - 唯一约束：`(workspace_id, namespace, secret_name, version)` 唯一
  - 删除策略：软删

补充规则：

- 高代码应用创建时必须至少关联 1 个 Agent 实例。
- `DataServiceInstance.visibility=embedded` 时，必须存在合法 `owner_kind/owner_id`。
- 删除高代码应用且 `cascade=true` 时，共享组件若被其他应用引用返回 `409 Conflict`。
- 删除低代码平台实例时，回收其 `embedded` 组件，不影响共享组件。
- 软删实体必须保留审计字段并支持按 `delete_task_id` 追踪任务。

## 资源状态机（设计输入）

### 资源状态（五类一级资源 + AgentInstance）

- `Creating`：已受理创建任务，资源尚未可用。
- `Running`：资源可用，健康检查通过。
- `Updating`：资源升级、配置变更或扩缩容进行中。
- `Degraded`：资源部分可用，存在告警或副本异常。
- `Failed`：最近一次创建或更新失败且未恢复。
- `Deleting`：资源删除进行中。
- `Deleted`：资源逻辑删除完成。
- `DeleteFailed`：删除失败，需重试或人工干预。

说明：上述状态为控制面领域状态，不与 K8S Pod Phase 一一对应。

DevBox 补充状态：

- `Stopped`：实例停止但未删除，可恢复启动。
- `Publishing`：镜像发布任务进行中。

高代码发布包补充状态（ReleaseChart）：

- `Packaging`：Chart 正在生成。
- `Pushed`：Chart 推送仓库成功。
- `ExportFailed`：生成或推送失败。

### 异步任务状态（AsyncTask）

- 状态集合：`Pending`、`Running`、`Succeeded`、`Failed`、`Canceled`。
- 结果字段最小集合：`resource_kind`、`resource_id`、`started_at`、`ended_at`、`failure_reason`、`retry_count`、`serialized_key`。

任务执行规则：

- CUD 请求必须携带 `Idempotency-Key`；服务端计算
  `idempotency_scope = sha256(actor_id + workspace_id + cluster_id + method + route_template + idempotency_key)`。
- `idempotency_scope` 命中且 `request_fingerprint` 一致：返回同一 `task_id`。
- `idempotency_scope` 命中但 `request_fingerprint` 不一致：返回 `409 IDEMPOTENCY_PAYLOAD_MISMATCH`。
- 去重窗口 24 小时，自首次写入起算；窗口到期后允许生成新任务。
- 串行化键：`workspace_id:cluster_id:resource_kind:resource_id_or_name`。
- 重试策略：最多 5 次，指数退避（5s、15s、45s、135s、300s）。
- 超时策略：创建/更新默认 30 分钟，删除默认 15 分钟，发布导出默认 20 分钟。
- 取消策略：`Pending` 可取消；`Running` 仅可中断任务可取消。

## 接口契约要求（设计输入）

### 通用契约

- API 前缀统一为 `/api/v1`。
- 资源作用域前缀统一为
  `{scope}=/api/v1/workspaces/{workspace_id}/clusters/{cluster_id}`。
- CUD 接口必须支持 `Idempotency-Key`。
- 更新/删除必须使用 `resource_version` 并发控制。
- Query 同步返回，支持分页、过滤、排序。
- 返回统一结构：`request_id`、`code`、`message`、`data`。

### 错误码最小集合

- `400`：参数非法或 schema 校验失败。
- `401`：未认证。
- `403`：无权限。
- `404`：资源不存在。
- `409`：并发冲突、引用冲突或 `IDEMPOTENCY_PAYLOAD_MISMATCH`（同作用域幂等键但请求载荷不一致）。
- `422`：业务规则校验失败（跨边界、单例冲突、发布前置条件不满足等）。
- `429`：请求限流。
- `500`：系统内部错误。

### 资源接口分组

- 设计约束（OpenAPI 3.0）：
  - 路径采用“资源集合 + 资源标识”建模，`LIST` 统一使用 `GET` 集合接口表达。
  - 同一路径通过不同 HTTP Method 区分语义，不使用 `POST/GET/LIST` 混写。
  - Path 段尽量使用单一单词的复数名词，避免连字符拼接命名。

- 数据服务组件实例（shared）：
  - `POST {scope}/dataservices`
  - `GET {scope}/dataservices`
  - `GET {scope}/dataservices/{dataservice_id}`
  - `PUT {scope}/dataservices/{dataservice_id}`
  - `DELETE {scope}/dataservices/{dataservice_id}`

- 低代码平台实例：
  - `POST {scope}/lowcodes`
  - `GET {scope}/lowcodes`
  - `GET {scope}/lowcodes/{lowcode_id}`
  - `PUT {scope}/lowcodes/{lowcode_id}`
  - `DELETE {scope}/lowcodes/{lowcode_id}`

- DevBox 实例与发布记录：
  - `POST {scope}/devboxes`
  - `GET {scope}/devboxes`
  - `GET {scope}/devboxes/{devbox_id}`
  - `PUT {scope}/devboxes/{devbox_id}`
  - `DELETE {scope}/devboxes/{devbox_id}`
  - `POST {scope}/devboxes/{devbox_id}/publishes`
  - `GET {scope}/devboxes/{devbox_id}/publishes`
  - `GET {scope}/publishes/{publish_id}`

- 网关实例：
  - `POST {scope}/gateways`
  - `GET {scope}/gateways`
  - `GET {scope}/gateways/{gateway_id}`
  - `PUT {scope}/gateways/{gateway_id}`
  - `DELETE {scope}/gateways/{gateway_id}`

- 资源模板（按类型过滤）：
  - `GET {scope}/templates?kind=dataservice`
  - `GET {scope}/templates?kind=lowcode`
  - `GET {scope}/templates?kind=devbox`
  - `GET {scope}/templates?kind=gateway`
  - `GET {scope}/templates/{template_id}`

- 高代码制品：
  - `GET {scope}/artifacts`
  - `GET {scope}/artifacts/{artifact_id}`

- Agent 实例：
  - `POST {scope}/agents`
  - `GET {scope}/agents`
  - `GET {scope}/agents/{agent_id}`
  - `PUT {scope}/agents/{agent_id}`
  - `DELETE {scope}/agents/{agent_id}`

- 高代码应用与发布产物：
  - `POST {scope}/applications`
  - `GET {scope}/applications`
  - `GET {scope}/applications/{application_id}`
  - `PUT {scope}/applications/{application_id}`
  - `DELETE {scope}/applications/{application_id}`
  - `GET {scope}/applications/{application_id}/relations`
  - `POST {scope}/applications/{application_id}/releases`
  - `GET {scope}/applications/{application_id}/charts`
  - `GET {scope}/charts/{chart_id}/package`

- 异步任务：
  - `GET {scope}/tasks/{task_id}`
  - `GET {scope}/tasks/{task_id}/result`

## 权限动作矩阵（设计输入）

| 资源/动作 | 超级管理员 | 用户（已关联 workspace） | 用户（未关联 workspace） |
| --- | --- | --- | --- |
| 工作空间创建/删除 | 允许 | 拒绝 | 拒绝 |
| 集群纳管与解绑 | 允许 | 拒绝 | 拒绝 |
| 工作空间绑定集群/仓库 | 允许 | 拒绝 | 拒绝 |
| 数据服务组件实例 CRUD（共享） | 允许 | 允许（仅本 workspace） | 拒绝 |
| 低代码平台实例开通/变更/删除 | 允许 | 拒绝（默认） | 拒绝 |
| 低代码平台实例查询 | 允许 | 允许（仅本 workspace） | 拒绝 |
| DevBox 实例 CRUD | 允许 | 允许（仅本 workspace） | 拒绝 |
| DevBox 镜像发布 | 允许 | 允许（仅本 workspace） | 拒绝 |
| 网关实例 CRUD | 允许 | 拒绝（默认） | 拒绝 |
| 高代码制品查询 | 允许 | 允许（仅本 workspace） | 拒绝 |
| Agent 实例 CRUD | 允许 | 允许（仅本 workspace） | 拒绝 |
| 高代码应用 CRUD | 允许 | 允许（仅本 workspace） | 拒绝 |
| 高代码应用发布与 Chart 下载 | 允许 | 允许（仅本 workspace） | 拒绝 |
| 嵌入式组件独立运维 | 允许 | 拒绝（仅随宿主管理） | 拒绝 |
| Secret 明文读取 | 拒绝（仅平台控制面服务账号可读） | 拒绝 | 拒绝 |
| 任务状态与结果查询 | 允许 | 允许（仅本 workspace） | 拒绝 |

## Secret 生命周期与审计（设计输入）

- S-SEC-001：Secret 必须保存在工作空间 namespace。
- S-SEC-002：普通用户不可查看明文，仅可引用 `secret_ref`。
- S-SEC-003：Secret 采用版本化命名：`{secret_name}.v{n}`。
- S-SEC-004：更新 Secret 写入新版本并切换当前版本指针，默认保留 10 个历史版本。
- S-SEC-005：支持版本回滚。
- S-SEC-006：当前阶段使用 K8S Secret 默认机制，不引入额外 KMS。
- S-SEC-007：审计字段最小集合：
  `secret_name`、`version`、`action`、`actor`、`workspace_id`、`cluster_id`、`timestamp`、`request_id`、`result`。

## 非功能需求（量化）

### 性能与可用性

- NFR-001：CUD 提交接口（仅提交任务）P95 <= 300ms，P99 <= 800ms。
- NFR-002：Query 接口（非聚合）P95 <= 800ms，P99 <= 2s。
- NFR-003：任务从 `Pending` 到 `Running` 排队时间 P95 <= 5s。
- NFR-004：常规创建/更新任务完成时间 P95 <= 10 分钟；删除 P95 <= 5 分钟；发布导出 P95 <= 8 分钟。
- NFR-005：控制面月度可用性 >= 99.9%。

### 容量

- NFR-006：支持 >= 100 个工作空间，>= 50 个纳管集群。
- NFR-007：支持 >= 5000 个共享数据服务实例。
- NFR-008：支持 >= 2000 个低代码平台实例、>= 5000 个 DevBox 实例、>= 5000 个网关实例。
- NFR-009：支持 >= 5000 个高代码应用实例，>= 10000 个 Agent 实例。
- NFR-010：支持日均 >= 10000 个异步任务提交。
- NFR-011：支持日均 >= 2000 次高代码应用发布 Chart 产物生成与存储。

### 安全与审计

- NFR-012：敏感信息不得明文持久化。
- NFR-013：审计日志保留期 >= 180 天，任务执行日志保留期 >= 30 天。
- NFR-014：跨工作空间/跨集群/跨 namespace 关联请求必须拒绝并审计。
- NFR-015：发布制品元数据（来源、版本、digest、操作者）必须可追溯。

### 可扩展性与运维

- NFR-016：新增资源类型应在不修改任务队列、鉴权主流程前提下完成接入。
- NFR-017：模板参数必须具备 schema 校验能力，非法参数在提交阶段失败。
- NFR-018：任务失败需提供可诊断失败原因（错误码、错误信息、失败步骤）。
- NFR-019：状态更新延迟 <= 30s。

## 验收标准与追踪矩阵

| 需求 ID | 验收条件 | 测试类型 | 责任方 |
| --- | --- | --- | --- |
| R-ENV-003 | 绑定 workspace 与 cluster 后，自动创建/复用同名 namespace | 集成测试 | 后端 |
| R-ABS-002 | 5 类资源 CUD 均复用统一执行链路并产出统一状态结构 | 集成测试 | 后端 |
| R-DSP-008 | 低代码平台嵌入式组件不出现在共享组件列表 | 集成测试 + E2E | 后端 + 前端 |
| R-LCP-003 | 超级管理员可通过 Helm 导入 Coze/n8n/外部 Dify/FastGPT | 集成测试 | 后端 |
| R-DBX-006 | 未发布镜像不能创建高代码应用 | 单元测试 + 集成测试 | 后端 |
| R-GTW-003 | 每 workspace+cluster 仅允许一个网关实例 | 单元测试 + 集成测试 | 后端 |
| R-HCA-003 | Chart 渠道可部署多 Agent 并处理 Chart 自带组件 | 集成测试 + E2E | 后端 |
| R-HCA-012 | `cascade=true` 且共享组件被复用时返回 `409` | 集成测试 | 后端 |
| R-PKG-001 | 发布高代码应用后自动生成可用 Helm Chart 包 | 集成测试 + E2E | 后端 |
| R-PKG-003 | 发布 Chart 成功推送到工作空间关联仓库并可追溯版本 | 集成测试 | 后端 |
| R-PKG-004 | 用户可下载发布 Chart 并在外部环境导入成功 | E2E | 前端 + 后端 |
| R-OPS-001 | 5 类资源 CUD 异步、Query 同步符合契约 | E2E | 前端 + 后端 |
| NFR-001~005 | 性能与可用性满足量化指标 | 压测 + 运行验收 | SRE + 后端 |

## 交付物与接口（面向设计）

- OpenAPI 3.0 草案（含 5 类资源、发布制品接口、错误码、幂等与并发控制）。
- 统一资源抽象模型与 ERD（含 owner/visibility 语义、唯一性约束）。
- 统一状态机定义（资源状态机、任务状态机、发布包状态机、解绑状态机）。
- 异步任务与补偿设计说明（串行化键、重试、超时、取消、回收补偿）。
- 模板与制品适配设计（内置 Helm、导入 Helm、镜像转发布 Chart）。
- 权限矩阵与鉴权流程说明（角色、workspace 边界、嵌入式资源边界）。
- NFR 验收报告模板与需求追踪矩阵。

## 需求管理

- 需求持续演进，不设置冻结点；每次变更必须同步更新 `docs/requirements.md` 与 `docs/decision.md`。
