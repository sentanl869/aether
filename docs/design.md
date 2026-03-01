# Aether 设计文档

## §1 设计目标、范围、术语与决策基线（T01）

### 1.1 目标与需求追踪

本章节作为后续设计任务（§2~§10）的输入冻结层，仅定义“可设计/不可设计”边界与术语、决策基线，不提前下沉接口实现细节。

| 追踪ID | 来源 | 约束落点 |
| --- | --- | --- |
| RG-01 | requirements §1 文档目标 | 本文档仅覆盖 Aether 部署域能力，不扩展到上游制品构建流程。 |
| RG-02 | requirements §2 背景与范围 | 固化 In Scope / Out of Scope / 上层职责三段式边界。 |
| RG-03 | requirements §3 术语与对象 | 统一术语字典，禁止同义词混用。 |
| RG-04 | requirements §13.1 + 生效决策清单 | 仅使用 D-001/D-003/D-005/D-007/D-010/D-011/D-012/D-013/D-015/D-016/D-017/D-018/D-019 作为设计输入。 |
| RG-05 | requirements §13.2 | 后续章节必须满足 GC-01~GC-10 的交付约束。 |

### 1.2 设计输入边界（In Scope / Out of Scope / 上层职责）

| 边界分类 | 内容 |
| --- | --- |
| In Scope（Aether） | 统一 CUD/Query 执行能力；无状态部署执行；按工作空间/集群/命名空间落地；参数渲染 `values.yaml`；输出任务当前状态与结果。 |
| Out of Scope（Aether 不负责） | 低代码平台内部业务功能；登录认证流程；资源级业务约束前置校验；资源/任务状态与历史快照持久化；依赖展示语义模型。 |
| 上层职责（调用方必须完成） | 接收业务 OpenAPI；业务约束前置校验；Aether 入参转换；维护状态表与历史快照；决定依赖删除策略并显式传参。 |

不得下沉到 Aether 的职责（T01 DoD 强约束）：

1. 资源级业务约束校验（如单实例限制、来源组合限制）。
2. 依赖展示语义及其状态模型。
3. 资源/任务状态持久化维护。

### 1.3 术语与对象字典（与 requirements §3 对齐）

| 术语 | 规范定义 | 设计使用约束 |
| --- | --- | --- |
| Workspace | 资源管理与权限作用域的核心边界。 | 所有授权、幂等、隔离讨论默认以 `workspace_id` 为第一边界。 |
| Managed Cluster | 被平台接入的 Kubernetes 集群。 | 资源创建必须显式携带 `cluster_id`。 |
| Namespace | 工作空间在关联集群中的同名命名空间。 | 命名空间命名由工作空间映射决定，不在 Aether 内二次派生。 |
| Managed Registry | 被平台接入的 Harbor 仓库。 | 作为上层已纳管资产输入，Aether 不承担仓库纳管生命周期。 |
| Resource Type | Aether 管理的部署对象类型。 | 仅通过“类型注册 + 通用执行器”扩展，不复制流程。 |
| Resource Instance | 某资源类型的一次实例化部署。 | 作为执行目标实体，不承载业务域附加状态。 |
| Deployment Execution | CUD 触发的一次 Helm/K8S 执行过程。 | 仅表达当前执行事实，不承诺历史时间线。 |

### 1.4 决策基线映射（生效/废止隔离）

#### 1.4.1 生效决策与设计落点

| 决策ID | 状态 | 设计落点 | 约束摘要 |
| --- | --- | --- | --- |
| D-001 | 生效 | §1.1, §1.2 | Aether 聚焦部署消费，不承载制品构建流程。 |
| D-003 | 生效 | §1.3, §5（T05） | 资源扩展通过类型注册，执行链路复用通用执行器。 |
| D-005 | 生效 | §1.3, §2（T02） | 授权边界按工作空间统一，同空间不按创建者隔离。 |
| D-007 | 生效 | §1.3, §2（T02）, §3（T03） | 创建请求 `cluster_id` 必填。 |
| D-010 | 生效 | §1.2, §9（T09） | 资源级业务约束由上层前置校验。 |
| D-011 | 生效 | §1.2, §8（T08） | Aether 无状态执行，不维护资源/任务持久化。 |
| D-012 | 生效 | §1.1, §3（T03）, §4（T04） | Query 收敛为 `task_id` 单任务查询。 |
| D-013 | 生效 | §1.3, §4（T04） | 幂等键固定为 `request_id + workspace_id + action`。 |
| D-015 | 生效 | §1.2, §6（T06） | 依赖删除策略由上层显式传参，Aether 不内建隐式级联。 |
| D-016 | 生效 | §1.1, §4（T04）, §8（T08） | 幂等/重入采用“无存储 + 可观测事实判定”。 |
| D-017 | 生效 | §1.1, §3（T03）, §9（T09） | 接口定义与 OpenAPI 映射归属 `docs/design.md`。 |
| D-018 | 生效 | §1.1, §10（T10） | 图示仅 Mermaid；设计交付粒度需可直接指导编码。 |
| D-019 | 生效 | §1.1, §7（T07） | 必须支持 values 渲染并保证 Helm 执行语义等效。 |

#### 1.4.2 废止决策隔离规则

| 决策ID | 状态 | 替代关系 | 处理规则 |
| --- | --- | --- | --- |
| D-002 | 废止 | 被 D-011 取代 | 不得再用于定义任务持久化与列表查询能力。 |
| D-004 | 废止 | 被 D-015 取代 | 不得再引入依赖展示分类与强制级联删除语义。 |
| D-006 | 废止 | 被 D-011 取代 | 不得再将 PostgreSQL/Redis 作为 Aether 强依赖。 |
| D-008 | 废止 | 被 D-010 取代 | 不得再在 Aether 内固化“单实例”业务约束。 |
| D-009 | 废止 | 被 D-012 取代 | 不得再扩展分页/过滤/排序型 Query 能力。 |
| D-014 | 废止 | 被 D-017 取代 | 不得再将接口 DTO/OpenAPI 设计内容回填到 requirements。 |

基线判定规则：

1. 设计评审仅接受 requirements §13.1 生效清单中的决策条目。
2. 任何引用废止决策的设计内容都应判定为越界并回退。
3. 生效决策若存在历史表述冲突，以“最新替代链终点决策”解释为准。

### 1.5 边界判定关键时序（Mermaid）

```mermaid
sequenceDiagram
    participant Biz as 上层业务模块
    participant Gate as 边界判定
    participant Aether as Aether
    Biz->>Gate: 业务请求（含业务规则）
    Gate->>Gate: 校验资源级业务约束/依赖策略/审计策略
    alt 前置校验失败
        Gate-->>Biz: 拒绝下发（保留业务错误语义）
    else 前置校验通过
        Gate->>Gate: 转换为统一 Aether 入参
        Gate->>Aether: 调用 CUD/Query/Render
        Aether-->>Gate: 返回部署执行受理或查询事实
        Gate-->>Biz: 维护业务状态与历史快照
    end
```

### 1.6 数据结构与边界规则（设计输入层）

| 结构 | 字段 | 规则 |
| --- | --- | --- |
| `DesignBoundaryRow` | `category`, `responsibility`, `owner`, `forbidden_in_aether` | 用于标记职责归属；`owner` 只能是 `AETHER` 或 `UPSTREAM`。 |
| `GlossaryEntry` | `term`, `definition`, `source_section`, `aliases` | `aliases` 必须为空或仅含同义词映射，禁止改变术语边界。 |
| `DecisionBaselineEntry` | `decision_id`, `status`, `superseded_by`, `design_sections`, `enforced_rules` | `status=DEPRECATED` 的条目不得进入 `design_sections` 的实现约束集合。 |

边界条件与禁止操作：

1. 若需求变更引入新能力但未写入 requirements §13.1 生效决策，设计不得先行扩展。
2. 不允许将“上层已校验”替换为 “Aether 内部兜底业务校验”。
3. 不允许在无状态边界下声明“任务历史可追溯存储”能力。

## §2 授权模型与工作空间/多集群作用域（T02）

### 2.1 需求追踪与约束

| 追踪ID | 来源 | 设计落点 |
| --- | --- | --- |
| ACL-01 | FR-ACL-001 | 定义超级管理员对全部资源类型 CUD/QueryTask 的授权能力，并声明纳管资产管理动作由上层模块承载。 |
| ACL-02 | FR-ACL-002 | 定义空间管理员在授权工作空间内的全量工作空间级操作能力。 |
| ACL-03 | FR-ACL-003 | 定义普通用户可操作资源类型集合（数据服务、DevBox、高代码应用、记忆体）。 |
| ACL-04 | FR-ACL-004 | 明确同工作空间内不按创建者做二级隔离。 |
| WS-01 | FR-WS-001 | 固化 `workspace -> cluster -> namespace` 映射规则与校验点。 |
| WS-02 | FR-WS-002 | 定义工作空间仓库关联为“上层维护、Aether 消费引用”的边界。 |
| WS-03 | FR-WS-003 | 固化 `cluster_id` 必填规则及错误码/错误消息模板。 |

基线约束：本章节仅定义授权与作用域判定，不下沉登录认证与业务规则前置校验（D-005、D-010、D-017）。

### 2.2 授权输入模型与领域结构

| 结构 | 字段 | 规则 |
| --- | --- | --- |
| `OperatorContext` | `operator_id`, `roles[]`, `workspace_grants[]` | `roles` 取值：`SUPER_ADMIN`、`WORKSPACE_ADMIN`、`USER`。`workspace_grants` 由上层鉴权系统下发，Aether 不反查 IAM。 |
| `WorkspaceGrant` | `workspace_id`, `role`, `granted_at`, `expires_at` | 同一 `workspace_id` 可出现多角色并集；过期授权不得用于 CUD/Query。 |
| `AuthorizationRequest` | `action`, `workspace_id`, `cluster_id`, `resource_type`, `resource_id?`, `task_id?` | `action` 取值：`DEPLOY_CREATE`、`DEPLOY_UPDATE`、`DEPLOY_DELETE`、`QUERY_TASK`。 |
| `WorkspaceClusterBinding` | `workspace_id`, `cluster_id`, `namespace`, `registry_refs[]` | `namespace` 必须与该工作空间的命名空间键一致；`registry_refs` 只用于镜像引用合法性消费。 |
| `AuthorizationDecision` | `allowed`, `deny_code?`, `deny_message?`, `resolved_namespace?` | 拒绝时必须返回统一错误码；允许时输出 `resolved_namespace` 供执行链路复用。 |

字段约束：

1. CUD 请求的 `cluster_id` 必填（Create/Update/Delete 一致）。
2. `workspace_id` 必须来自上层请求路径或业务上下文，不允许由 Aether 推断。
3. `namespace` 不接受调用方自由传入，必须由 `WorkspaceClusterBinding` 解析得到。

### 2.3 角色-动作-资源矩阵

#### 2.3.1 Aether 资源操作矩阵

| 资源类型 | CUD 允许角色 | QueryTask 允许角色 | 说明 |
| --- | --- | --- | --- |
| `DATA_SERVICE` | `SUPER_ADMIN`、`WORKSPACE_ADMIN`、`USER` | 同 CUD | 对应 FR-ACL-003“创建数据服务组件实例”。 |
| `LOW_CODE_PLATFORM` | `SUPER_ADMIN`、`WORKSPACE_ADMIN` | 同 CUD | 普通用户在低代码平台内的业务操作不经 Aether CUD。 |
| `DEVBOX` | `SUPER_ADMIN`、`WORKSPACE_ADMIN`、`USER` | 同 CUD | 对应 FR-ACL-003“DevBox CRUD”。 |
| `GATEWAY` | `SUPER_ADMIN` | `SUPER_ADMIN` | 网关实例管理保留超级管理员权限。 |
| `HIGH_CODE_APP` | `SUPER_ADMIN`、`WORKSPACE_ADMIN`、`USER` | 同 CUD | 对应 FR-ACL-003“创建高代码应用”。 |
| `MEMORY` | `SUPER_ADMIN`、`WORKSPACE_ADMIN`、`USER` | 同 CUD | 对应 FR-ACL-003“创建记忆体实例”。 |

同工作空间内授权规则：

1. 不按资源创建者做隔离，判定维度仅为 `workspace_id + role + resource_type`（FR-ACL-004）。
2. QueryTask 与 CUD 使用同一资源权限矩阵（覆盖 TC-017）。

#### 2.3.2 超出 Aether 范围的管理动作

FR-ACL-001 中“纳管集群/仓库/工作空间 CRUD、空间管理员配置”等平台管理动作由上层模块实现；Aether 仅消费其产出（如 `WorkspaceClusterBinding`、`workspace_grants`），不提供对应 CUD 接口。

### 2.4 工作空间/多集群作用域规则

| 规则ID | 规则 | 失败处理 |
| --- | --- | --- |
| SCOPE-01 | `workspace_id` 必须存在于 `workspace_grants`（或 `SUPER_ADMIN` 全局放行）。 | `AETHER_PERMISSION_DENIED` |
| SCOPE-02 | CUD 的 `cluster_id` 必填。 | `AETHER_INVALID_ARGUMENT` |
| SCOPE-03 | `cluster_id` 必须在 `WorkspaceClusterBinding` 中与 `workspace_id` 成对存在。 | `AETHER_PERMISSION_DENIED`（避免泄露绑定拓扑） |
| SCOPE-04 | `resolved_namespace` 必须等于该工作空间的规范命名空间键（默认同 `workspace_id`）。 | `AETHER_CONFLICT` |
| SCOPE-05 | 目标集群中必须已存在 `resolved_namespace`。 | `AETHER_RESOURCE_NOT_FOUND` |
| SCOPE-06 | `registry_refs` 仅作引用消费，不在 Aether 内落库。 | 非法引用由上层拒绝；Aether 不新增持久化实体 |

多集群同名 namespace 约束（FR-WS-001）：

1. 同一 `workspace_id` 绑定多个集群时，所有绑定记录的 `namespace` 必须一致。
2. 若任一集群缺失该 namespace，Aether 不自动创建，直接拒绝执行。
3. 绑定关系变更后，后续请求按最新绑定快照判定；Aether 不缓存长期拓扑状态。

### 2.5 接口边界映射（OpenAPI 3.0 RESTful -> Aether 代码接口）

Aether 本身不对外暴露 OpenAPI；本表用于固定上层 REST 接口与 Aether 鉴权输入的映射关系（GC-03）。

| 上层 OpenAPI 3.0 路径（RESTful） | Aether 动作 | 必要鉴权字段 | 备注 |
| --- | --- | --- | --- |
| `POST /api/v1/workspaces/{workspace_id}/resources/{resource_type}` | `DEPLOY_CREATE` | `operator`, `workspace_id`, `cluster_id`, `resource_type` | 缺失 `cluster_id` 按参数错误拒绝。 |
| `PUT /api/v1/workspaces/{workspace_id}/resources/{resource_type}/{resource_id}` | `DEPLOY_UPDATE` | 同上 + `resource_id` | 更新沿用创建同一作用域规则。 |
| `DELETE /api/v1/workspaces/{workspace_id}/resources/{resource_type}/{resource_id}` | `DEPLOY_DELETE` | 同上 + `resource_id` | 删除同样要求显式 `cluster_id`。 |
| `GET /api/v1/workspaces/{workspace_id}/tasks/{task_id}` | `QUERY_TASK` | `operator`, `workspace_id`, `task_id` | QueryTask 在工作空间维度校验权限。 |

错误码与错误消息模板（T02 DoD）：

| 场景 | 错误码 | 错误消息模板 |
| --- | --- | --- |
| CUD 缺失 `cluster_id` | `AETHER_INVALID_ARGUMENT` | `cluster_id is required (action={action}, workspace_id={workspace_id})` |
| 操作者不具备工作空间授权 | `AETHER_PERMISSION_DENIED` | `operator has no permission in workspace {workspace_id}` |
| 角色不允许操作目标资源类型 | `AETHER_PERMISSION_DENIED` | `role {role} cannot {action} on {resource_type} in workspace {workspace_id}` |
| 目标集群未绑定到工作空间 | `AETHER_PERMISSION_DENIED` | `cluster scope is not allowed for workspace {workspace_id}` |
| 工作空间 namespace 配置冲突 | `AETHER_CONFLICT` | `workspace namespace mapping conflict for workspace {workspace_id}` |
| 集群缺失目标 namespace | `AETHER_RESOURCE_NOT_FOUND` | `namespace {namespace} not found in cluster {cluster_id}` |

### 2.6 授权与作用域判定关键时序（Mermaid）

```mermaid
sequenceDiagram
    participant Caller as 上层模块
    participant Auth as Aether AuthZ
    participant Scope as ScopeResolver
    participant K8S as Cluster API
    participant Exec as Deployment Executor

    Caller->>Auth: CUD/QueryTask 请求（operator/workspace/cluster/resource）
    Auth->>Auth: 校验角色矩阵 + workspace_grants
    alt cluster_id 缺失（CUD）
        Auth-->>Caller: AETHER_INVALID_ARGUMENT
    else 工作空间或角色未授权
        Auth-->>Caller: AETHER_PERMISSION_DENIED
    else 鉴权通过
        Auth->>Scope: 查询 WorkspaceClusterBinding
        alt cluster 未绑定或 namespace 映射冲突
            Scope-->>Caller: AETHER_PERMISSION_DENIED or AETHER_CONFLICT
        else 绑定合法
            Scope->>K8S: 校验 namespace 存在
            alt namespace 不存在
                K8S-->>Caller: AETHER_RESOURCE_NOT_FOUND
            else 作用域确定
                Scope-->>Exec: resolved_namespace
                Exec-->>Caller: ACCEPTED / QueryTask Result
            end
        end
    end
```

### 2.7 并发、幂等与边界条件

并发与幂等协同规则（与 §4 对齐）：

1. 授权/作用域判定必须发生在幂等键计算与 `task_id` 生成之前；鉴权失败不得占用幂等键。
2. 同一 `request_id + workspace_id + action` 的重试请求，若当前操作者在该工作空间仍有权限，则允许命中同一幂等结果（不要求“原操作者本人”）。
3. 任务执行中若角色被撤销，不影响已受理执行；后续 CUD/QueryTask 重新按最新授权快照判定。

边界条件与禁止操作：

1. 禁止通过伪造 `workspace_id` 访问其他工作空间任务或资源。
2. 禁止调用方显式覆盖 `resolved_namespace`（仅允许由绑定关系解析）。
3. 普通用户禁止对 `LOW_CODE_PLATFORM`、`GATEWAY` 发起 CUD/QueryTask。
4. 多集群绑定中出现 namespace 不一致时，所有涉及该工作空间的 CUD 必须失败，直至绑定修复。

## §3 统一接口契约、DTO、错误码（T03）

### 3.1 需求追踪与章节边界

| 追踪ID | 来源 | 设计落点 |
| --- | --- | --- |
| API-01 | FR-API-001 | Aether 仅定义代码级 Go 接口，不直接暴露对外 OpenAPI。 |
| API-02 | FR-API-002 | 统一 CUD 请求 DTO，固定最小字段与校验规则。 |
| API-03 | FR-API-003 | CUD 同步返回固定受理模型，`status=ACCEPTED`。 |
| API-04 | FR-API-004 | Query 仅支持 `task_id` 单任务查询，不提供列表检索。 |
| API-05 | FR-API-005 | 错误码类别一一映射，无“兜底模糊错误”。 |
| API-06 | FR-API-006 | RenderValues 复用 CUD 语义并提供数据/文件两种输出。 |
| CORE-03 | FR-CORE-003 | 资源标识在接口模型中统一携带 `resource_id/workspace_id/cluster_id/namespace/resource_type`。 |
| GC-02 | requirements §13.2 | 章节必须给出 Go 接口签名、DTO、枚举、字段校验、错误码。 |
| GC-03 | requirements §13.2 | 明确上层 OpenAPI 3.0 RESTful 到 Aether 代码接口映射。 |

章节边界：

1. 本节定义接口契约、DTO 与错误码，不定义执行状态机细节（状态机与重入细节见 §4）。
2. 本节定义“接口级并发/幂等约束”，不引入资源级并发锁实现（由上层治理，见 requirements NFR-003）。
3. 本节中 OpenAPI 仅作边界映射示意，Aether 仍保持“非对外 OpenAPI”定位。

### 3.2 代码级接口签名（Go）

```go
type DeploymentAPI interface {
    Create(ctx context.Context, req CUDRequest) (CUDAcceptedResponse, *AetherError)
    Update(ctx context.Context, req CUDRequest) (CUDAcceptedResponse, *AetherError)
    Delete(ctx context.Context, req CUDRequest) (CUDAcceptedResponse, *AetherError)
    QueryTask(ctx context.Context, req QueryTaskRequest) (QueryTaskResponse, *AetherError)
    RenderValues(ctx context.Context, req RenderValuesRequest) (RenderValuesResponse, *AetherError)
}
```

接口语义约束：

1. `Create/Update/Delete` 必须复用同一 `CUDRequest` 结构，动作语义由调用方法名确定。
2. `QueryTask` 的业务检索键仅允许 `task_id`；`workspace_id` 仅用于授权与作用域判定，不属于任务检索过滤条件。
3. `RenderValues` 仅支持 `DEPLOY_CREATE`、`DEPLOY_UPDATE` 两类渲染意图；不提供 `DEPLOY_DELETE` 渲染。

### 3.3 DTO、枚举与字段约束

#### 3.3.1 枚举定义

| 枚举 | 取值 | 说明 |
| --- | --- | --- |
| `CUDAction` | `DEPLOY_CREATE`、`DEPLOY_UPDATE`、`DEPLOY_DELETE` | 用于幂等键中的 `action` 维度。 |
| `RenderIntent` | `DEPLOY_CREATE`、`DEPLOY_UPDATE` | Render 仅覆盖需要 values 输入的动作。 |
| `AcceptedStatus` | `ACCEPTED` | CUD 受理态固定枚举，不允许扩展。 |
| `TaskStatus` | `ACCEPTED`、`RUNNING`、`SUCCEEDED`、`FAILED`、`UNKNOWN` | Query 返回状态枚举（详细流转见 §4）。 |
| `RenderOutputFormat` | `YAML_DATA`、`EXECUTABLE_FILE` | 对应 FR-API-006 输出形态。 |

#### 3.3.2 请求 DTO

| DTO | 字段 | 类型 | 必填 | 约束 |
| --- | --- | --- | --- | --- |
| `CUDRequest` | `request_id` | `string` | 是 | 非空；用于幂等键组成。 |
| `CUDRequest` | `operator` | `OperatorContext` | 是 | 与 §2 授权模型一致；不可为空对象。 |
| `CUDRequest` | `workspace_id` | `string` | 是 | 非空；由上层传入，不允许 Aether 推断。 |
| `CUDRequest` | `target.resource_type` | `ResourceType` | 是 | 必须是已注册资源类型。 |
| `CUDRequest` | `target.resource_id` | `string` | 条件必填 | `Update/Delete` 必填；`Create` 可选。 |
| `CUDRequest` | `cluster_id` | `string` | 是 | 所有 CUD 必填；缺失返回 `AETHER_INVALID_ARGUMENT`。 |
| `CUDRequest` | `spec` | `map[string]any` | 是 | 非空；结构校验由类型注册 schema 执行。 |
| `CUDRequest` | `labels` | `map[string]string` | 否 | 键值需满足 K8S label 约束。 |
| `CUDRequest` | `annotations` | `map[string]string` | 否 | 键值需满足 K8S annotation 约束。 |
| `QueryTaskRequest` | `task_id` | `string` | 是 | 任务唯一查询键；禁止列表参数。 |
| `QueryTaskRequest` | `workspace_id` | `string` | 是 | 仅用于权限与作用域校验，不参与查询过滤。 |
| `QueryTaskRequest` | `operator` | `OperatorContext` | 是 | 与 CUD 相同授权输入。 |
| `RenderValuesRequest` | `intent` | `RenderIntent` | 是 | 仅允许 `DEPLOY_CREATE/DEPLOY_UPDATE`。 |
| `RenderValuesRequest` | `cud_request` | `CUDRequest` | 是 | 与目标 CUD 的语义字段完全一致。 |
| `RenderValuesRequest` | `output_format` | `RenderOutputFormat` | 是 | 决定返回 YAML 数据或文件内容。 |

#### 3.3.3 响应 DTO

| DTO | 字段 | 类型 | 必填 | 约束 |
| --- | --- | --- | --- | --- |
| `CUDAcceptedResponse` | `request_id` | `string` | 是 | 回显请求值。 |
| `CUDAcceptedResponse` | `task_id` | `string` | 是 | 后续 Query 唯一查询键。 |
| `CUDAcceptedResponse` | `resource_id` | `string` | 是 | `Create` 可新分配；`Update/Delete` 与目标一致。 |
| `CUDAcceptedResponse` | `accepted_at` | `string(date-time)` | 是 | RFC3339 UTC 时间戳。 |
| `CUDAcceptedResponse` | `status` | `AcceptedStatus` | 是 | 固定为 `ACCEPTED`。 |
| `CUDAcceptedResponse` | `resource_ref` | `ResourceRef` | 是 | 包含 `workspace_id/cluster_id/namespace/resource_type/resource_id`。 |
| `QueryTaskResponse` | `task_id` | `string` | 是 | 与请求一致。 |
| `QueryTaskResponse` | `resource_ref` | `ResourceRef` | 是 | 统一资源标识（FR-CORE-003）。 |
| `QueryTaskResponse` | `status` | `TaskStatus` | 是 | 当前任务状态。 |
| `QueryTaskResponse` | `result` | `TaskResult` | 是 | 当前任务结果快照。 |
| `QueryTaskResponse` | `observed_at` | `string(date-time)` | 是 | 状态观测时间。 |
| `RenderValuesResponse` | `request_id` | `string` | 是 | 回显渲染输入中的 `request_id`。 |
| `RenderValuesResponse` | `output_format` | `RenderOutputFormat` | 是 | 返回形态标记。 |
| `RenderValuesResponse` | `yaml_data` | `string` | 条件必填 | `output_format=YAML_DATA` 时必填。 |
| `RenderValuesResponse` | `file_content` | `[]byte` | 条件必填 | `output_format=EXECUTABLE_FILE` 时必填。 |
| `RenderValuesResponse` | `values_digest` | `string` | 是 | 渲染内容摘要，用于与 CUD 执行输入对账。 |
| `RenderValuesResponse` | `template_version` | `string` | 是 | 标识渲染使用的模板版本。 |

### 3.4 字段校验、失败分支与状态约束

| 校验规则ID | 规则 | 失败错误码 |
| --- | --- | --- |
| VAL-01 | CUD 缺失 `request_id/workspace_id/cluster_id/spec/target.resource_type` 任一必填字段 | `AETHER_INVALID_ARGUMENT` |
| VAL-02 | `Update/Delete` 缺失 `target.resource_id` | `AETHER_INVALID_ARGUMENT` |
| VAL-03 | `QueryTaskRequest` 含分页、过滤、排序类参数 | `AETHER_INVALID_ARGUMENT` |
| VAL-04 | `QueryTaskRequest.task_id` 为空 | `AETHER_INVALID_ARGUMENT` |
| VAL-05 | `RenderValuesRequest.intent=DEPLOY_DELETE` | `AETHER_INVALID_ARGUMENT` |
| VAL-06 | `target.resource_type` 未注册 | `AETHER_RESOURCE_NOT_FOUND` |
| VAL-07 | 请求通过语法校验但权限不足/作用域不合法 | `AETHER_PERMISSION_DENIED` |

接口状态约束：

1. CUD 成功受理时 `status` 必须且仅能为 `ACCEPTED`。
2. Query 返回状态必须属于 `TaskStatus` 枚举集合；`UNKNOWN` 仅表示查询降级态。
3. Render 不产生 `task_id`，其结果仅作为 CUD 输入等价视图。

### 3.5 统一错误响应与错误码体系

统一错误 DTO：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `code` | `string` | Aether 统一错误码。 |
| `message` | `string` | 面向调用方的稳定错误消息模板。 |
| `details` | `map[string]any` | 字段级错误详情（可选）。 |
| `retriable` | `bool` | 是否建议上层重试。 |
| `trace_id` | `string` | 调用链跟踪标识。 |

错误码映射（与 FR-API-005 一一对应）：

| 错误码 | 类别 | 典型触发场景 | `retriable` | 上层 OpenAPI 建议 HTTP 状态 |
| --- | --- | --- | --- | --- |
| `AETHER_INVALID_ARGUMENT` | 参数错误 | 缺失必填字段、枚举非法、Query 出现列表参数 | `false` | `400` |
| `AETHER_PERMISSION_DENIED` | 权限错误 | workspace/角色/集群作用域不满足 | `false` | `403` |
| `AETHER_RESOURCE_NOT_FOUND` | 资源不存在 | `task_id` 不存在、类型未注册、namespace 不存在 | `false` | `404` |
| `AETHER_CONFLICT` | 资源冲突 | 幂等键命中但请求指纹冲突、namespace 映射冲突 | `false` | `409` |
| `AETHER_DEPENDENCY_MISSING` | 依赖缺失 | `spec.dependencies` 中 required 依赖未找到 | `false` | `422` |
| `AETHER_RENDER_ERROR` | 渲染错误 | values 渲染失败、渲染结果不满足模板/schema | `false` | `422` |
| `AETHER_QUERY_CONTEXT_UNAVAILABLE` | 查询上下文不可用 | Query 时 Helm/K8S 暂不可达 | `true` | `503` |
| `AETHER_INTERNAL` | 内部错误 | 非预期运行时异常 | `true` | `500` |

禁止规则：

1. 禁止定义“`UNKNOWN_ERROR`/`AETHER_ERROR`”等模糊兜底业务错误码。
2. 所有失败分支必须映射到上表 8 类之一；无法归类时只能落 `AETHER_INTERNAL`。

### 3.6 OpenAPI 3.0 RESTful 到代码接口映射

说明：以下路径由上层模块对外提供，Aether 仅提供代码接口实现（FR-API-001，GC-03）。

| 上层 REST 路径 | 方法 | Aether 接口 | 关键 DTO | 关键返回 |
| --- | --- | --- | --- | --- |
| `/api/v1/workspaces/{workspace_id}/resources/{resource_type}` | `POST` | `Create` | `CUDRequest` | `CUDAcceptedResponse` |
| `/api/v1/workspaces/{workspace_id}/resources/{resource_type}/{resource_id}` | `PUT` | `Update` | `CUDRequest` | `CUDAcceptedResponse` |
| `/api/v1/workspaces/{workspace_id}/resources/{resource_type}/{resource_id}` | `DELETE` | `Delete` | `CUDRequest` | `CUDAcceptedResponse` |
| `/api/v1/workspaces/{workspace_id}/tasks/{task_id}` | `GET` | `QueryTask` | `QueryTaskRequest` | `QueryTaskResponse` |
| `/api/v1/workspaces/{workspace_id}/resources/{resource_type}/render-values` | `POST` | `RenderValues` (`intent=DEPLOY_CREATE`) | `RenderValuesRequest` | `RenderValuesResponse` |
| `/api/v1/workspaces/{workspace_id}/resources/{resource_type}/{resource_id}/render-values` | `POST` | `RenderValues` (`intent=DEPLOY_UPDATE`) | `RenderValuesRequest` | `RenderValuesResponse` |

### 3.7 并发、幂等与一致性约束（接口层）

1. 幂等键固定为 `request_id + workspace_id + action`，`action` 由 `Create/Update/Delete` 方法确定（与 §4 保持一致）。
2. 同幂等键且请求指纹一致时，CUD 必须返回同一 `task_id/resource_id`（不重复触发执行）；指纹冲突返回 `AETHER_CONFLICT`。
3. Query 不提供批量、列表、分页、过滤、排序参数；任何此类扩展均判定为越界。
4. 同一资源的并发治理策略由上层负责，Aether 不在接口层声明强制互斥锁。
5. `RenderValuesResponse.values_digest` 必须可用于与后续 CUD 实际执行输入进行等价对账（渲染一致性细节见 §7）。

### 3.8 关键时序（Mermaid）

```mermaid
sequenceDiagram
    participant Client as 上层模块(OpenAPI)
    participant Adapter as API适配层
    participant Aether as Aether DeploymentAPI
    participant Auth as AuthZ/Scope
    participant Exec as 执行器

    Client->>Adapter: CUD 请求
    Adapter->>Aether: Create/Update/Delete(CUDRequest)
    Aether->>Auth: 字段校验 + 鉴权/作用域判定
    alt 参数或权限失败
        Auth-->>Adapter: AETHER_INVALID_ARGUMENT / AETHER_PERMISSION_DENIED
        Adapter-->>Client: 4xx + AetherError
    else 通过
        Aether->>Exec: 幂等判定并触发执行
        Exec-->>Aether: task_id/resource_id
        Aether-->>Adapter: CUDAcceptedResponse(status=ACCEPTED)
        Adapter-->>Client: 202 Accepted
        Client->>Adapter: GET /tasks/{task_id}
        Adapter->>Aether: QueryTask(task_id, workspace_id)
        Aether-->>Adapter: QueryTaskResponse(当前状态与结果)
        Adapter-->>Client: 200 OK
    end
```

### 3.9 边界条件与禁止操作

1. 禁止在 Aether 侧新增“任务列表检索接口”或“资源运行态聚合视图接口”。
2. 禁止在 CUD 受理响应中返回除 `ACCEPTED` 以外的状态值。
3. 禁止 Render 接口接受与 CUD 语义不一致的独立参数模型（避免两套输入语义分叉）。
4. `resource_id` 全局唯一性冲突治理由上层负责；Aether 不承担跨业务域唯一性仲裁。
5. `workspace_id` 必须贯穿 CUD/Query/Render 请求，禁止由 `task_id` 反推并隐式覆盖调用方作用域。

## §4 执行状态机、幂等与重入（T04）

待 T04 补充。

## §5 资源类型注册与扩展机制（T05）

待 T05 补充。

## §6 依赖表达与删除策略（T06）

待 T06 补充。

## §7 values 渲染与 Helm 等效执行（T07）

待 T07 补充。

## §8 无状态执行与任务查询架构（T08）

待 T08 补充。

## §9 上层集成边界与审计日志（T09）

待 T09 补充。

## §10 验收追踪矩阵与设计完成性检查（T10）

待 T10 补充。
