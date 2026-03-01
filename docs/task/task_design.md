# Aether 设计任务拆分（基于 `docs/requirements.md`）

## 0. 全局约束（强制，适用于所有 Task）

### 0.1 全局约束清单（来自 requirements §13.2）

| 约束ID | 约束内容 | 落地要求 |
| --- | --- | --- |
| GC-01 | `requirements.md` 不承载接口/DTO/OpenAPI 设计细节 | 相关内容必须写入 `docs/design.md` |
| GC-02 | 设计文档必须给出代码级接口定义 | 至少包含 Go 接口签名、DTO、枚举、字段校验、错误码表 |
| GC-03 | 设计文档必须定义接口边界映射 | 明确“上层模块 OpenAPI -> Aether 代码级接口”映射 |
| GC-04 | 设计文档必须定义任务状态机与幂等/重入规则 | 状态枚举、流转、冲突判定、重试语义必须可验证 |
| GC-05 | 设计文档必须定义资源类型注册与扩展机制 | 注册 schema、默认值策略、生命周期策略、新类型接入模板必须完整 |
| GC-06 | 设计文档必须定义依赖表达与删除策略 | 字段 schema、缺失处理、删除策略参数与行为必须明确 |
| GC-07 | 设计文档必须建立验收测试追踪矩阵 | FR/NFR/AC 与测试点一一映射 |
| GC-08 | 所有图示必须使用 Mermaid | 禁止图片、PlantUML 或其他图形语法 |
| GC-09 | 设计完成后必须达到可直接指导编码粒度 | 至少覆盖数据结构、状态流转、接口契约、异常码、并发/幂等、关键时序、边界条件 |
| GC-10 | values 渲染与 Helm 等效性必须单独设计 | 必须定义渲染规则、输出形态、一致性策略、`helm install/upgrade -f values.yaml` 等效保证 |

### 0.2 `docs/design.md` 章节预分配（供任务引用）

| 设计章节 | 章节名称 | 主责任务 |
| --- | --- | --- |
| §1 | 设计目标、范围、术语与决策基线 | T01 |
| §2 | 授权模型与工作空间/多集群作用域 | T02 |
| §3 | 统一接口契约、DTO、错误码 | T03 |
| §4 | 执行状态机、幂等与重入 | T04 |
| §5 | 资源类型注册与扩展机制 | T05 |
| §6 | 依赖表达与删除策略 | T06 |
| §7 | values 渲染与 Helm 等效执行 | T07 |
| §8 | 无状态执行与任务查询架构 | T08 |
| §9 | 上层集成边界与审计日志 | T09 |
| §10 | 验收追踪矩阵与设计完成性检查 | T10 |

## 1. 任务拆分与依赖

| Task ID | 任务名称 | 闭环目标 | 对应设计章节 | 主责需求 | 依赖 | 状态 |
| --- | --- | --- | --- | --- | --- | --- |
| T01 | 基线与边界建模 | 固化设计输入边界、术语、决策基线与非目标，防止后续设计越界 | §1 | RG-01~RG-05 | - | TODO |
| T02 | 权限与作用域模型设计 | 完成角色权限矩阵与 workspace/cluster/namespace 作用域规则 | §2 | FR-ACL-001~004, FR-WS-001~003 | T01 | TODO |
| T03 | 统一接口契约与错误码设计 | 产出 CUD/Query/Render 接口签名、DTO、校验与错误码 | §3 | FR-API-001~006, FR-CORE-003 | T01,T02 | TODO |
| T04 | 任务状态机与幂等重入设计 | 产出可执行状态机、幂等键/指纹/冲突判定与重入语义 | §4 | FR-EXEC-001~003, FR-Q-001~002 | T03 | TODO |
| T05 | 资源注册与扩展机制设计 | 六类资源统一注册 schema、生命周期参数与新增类型模板 | §5 | FR-RT-001~007, FR-CORE-001~002, NFR-002 | T01 | TODO |
| T06 | 依赖表达与删除策略设计 | 明确依赖 schema、展开规则、缺失处理与删除策略行为 | §6 | FR-DM-002~004 | T03,T05 | TODO |
| T07 | values 渲染与 Helm 等效性设计 | 定义渲染规则/输出形态/一致性与执行等效性保证机制 | §7 | FR-CORE-004, FR-EXEC-004, NFR-005 | T03,T05 | TODO |
| T08 | 无状态执行与查询架构设计 | 明确执行与查询事实来源、降级策略、无存储约束实现 | §8 | FR-EXEC-002, FR-DM-001, NFR-001,NFR-003 | T03,T04,T07 | TODO |
| T09 | 集成边界与可审计性设计 | 固化上层前置校验边界、调用映射与结构化日志规范 | §9 | FR-INT-001, NFR-004 | T02,T03 | TODO |
| T10 | 验收追踪矩阵与完成性收口 | 建立 FR/NFR/AC->测试点->设计章节全链路追踪并验收 | §10 | AC-001~006 | T01,T02,T03,T04,T05,T06,T07,T08,T09 | TODO |

## 2. 任务详情

### T01 基线与边界建模

- 目标
  - 将 `requirements.md` 的目标、范围、非目标、术语、决策基线固化为设计输入，建立“可设计/不可设计”边界。
- 对应章节
  - `docs/design.md` §1
- 覆盖需求
  - RG-01 文档目标（requirements §1）
  - RG-02 业务背景与范围（requirements §2）
  - RG-03 术语与对象（requirements §3）
  - RG-04 生效决策基线（requirements §13.1 + 生效决策 D-001/D-003/D-005/D-007/D-010/D-011/D-012/D-013/D-015/D-016/D-017/D-018/D-019）
  - RG-05 设计文档强制约束总览（requirements §13.2）
- 输出物
  - 设计输入边界表（In Scope / Out of Scope / 上层职责）
  - 术语与对象字典（与 `requirements.md` 对齐）
  - 决策基线映射表（生效/废止及影响）
- DoD
  - 明确列出“不得下沉到 Aether”的职责：资源级业务约束、依赖展示语义、持久化状态维护。
  - 生效/废止决策不得混用，且每条生效决策有对应设计落点。
  - 输出文本与 `requirements.md` 语义一致，不新增未授权能力。
- 进度
  - TODO

### T02 权限与作用域模型设计

- 目标
  - 形成角色能力矩阵、授权判定规则、workspace->cluster->namespace 映射约束与校验流程。
- 对应章节
  - `docs/design.md` §2
- 覆盖需求
  - FR-ACL-001, FR-ACL-002, FR-ACL-003, FR-ACL-004
  - FR-WS-001, FR-WS-002, FR-WS-003
- 输出物
  - 角色-动作-资源矩阵（超级管理员/空间管理员/普通用户）
  - 作用域判定序列图（Mermaid）
  - `cluster_id` 必填校验与错误返回规则
- DoD
  - 同工作空间内“非创建者隔离”规则在 CUD/Query 均被明确。
  - 多集群场景下 namespace 同名约束有明确校验点。
  - `cluster_id` 缺失时错误码与错误消息模板明确可复用。
- 进度
  - TODO

### T03 统一接口契约与错误码设计

- 目标
  - 统一定义 Aether 代码级接口（CUD/Query/Render）与 DTO 字段、枚举、校验、错误码。
- 对应章节
  - `docs/design.md` §3
- 覆盖需求
  - FR-API-001, FR-API-002, FR-API-003, FR-API-004, FR-API-005, FR-API-006
  - FR-CORE-003
  - GC-02, GC-03
- 输出物
  - Go 接口签名清单（不对外 OpenAPI）
  - 请求/响应 DTO 与字段约束表
  - 统一错误码表（参数/权限/不存在/冲突/依赖缺失/渲染错误/查询上下文不可用/内部错误）
- DoD
  - CUD 返回固定 `ACCEPTED`，并包含 `request_id/task_id/resource_id/accepted_at/status`。
  - Query 仅支持 `task_id` 单任务查询，不引入列表检索能力。
  - 错误码与需求中的错误类别一一对应，无“兜底模糊错误”。
- 进度
  - TODO

### T04 任务状态机与幂等重入设计

- 目标
  - 定义部署任务状态机、状态流转边界、幂等命中/冲突规则、重入行为。
- 对应章节
  - `docs/design.md` §4
- 覆盖需求
  - FR-EXEC-001, FR-EXEC-003
  - FR-Q-001, FR-Q-002
  - requirements §13.3.1, §13.3.2
  - GC-04
- 输出物
  - 任务状态机图（Mermaid，含 `ACCEPTED/RUNNING/SUCCEEDED/FAILED/UNKNOWN`）
  - 幂等键与请求指纹计算规范
  - 幂等命中、指纹冲突、重入重试判定流程
- DoD
  - `UNKNOWN` 被定义为查询降级态而非持久终态。
  - 幂等键严格使用 `request_id + workspace_id + action`，不得引入额外未定义字段。
  - “同幂等键同指纹返回同 task_id/resource_id；同幂等键异指纹返回冲突”被明确。
- 进度
  - TODO

### T05 资源注册与扩展机制设计

- 目标
  - 完成六类资源的注册模型、模板来源、schema/defaults/lifecycle 参数与新增类型接入模板。
- 对应章节
  - `docs/design.md` §5
- 覆盖需求
  - FR-RT-001, FR-RT-002, FR-RT-003, FR-RT-004, FR-RT-005, FR-RT-006, FR-RT-007
  - FR-CORE-001, FR-CORE-002
  - NFR-002
  - requirements §13.4.1, §13.4.2
  - AC-001
- 输出物
  - 六类资源注册配置表（type/category/template/schema/defaults/lifecycle/capabilities/query-adapter）
  - 新资源接入模板与回归检查清单
  - “禁止复制粘贴流程”的扩展性约束说明
- DoD
  - 六类资源均可通过统一执行器进入 CUD/Query 流程。
  - 新资源接入步骤可独立执行，且不要求修改通用编排核心逻辑。
  - 每类资源的模板来源和依赖表达边界与需求一致。
- 进度
  - TODO

### T06 依赖表达与删除策略设计

- 目标
  - 完整定义 `spec.dependencies` schema、解析展开规则、缺失处理与删除策略参数。
- 对应章节
  - `docs/design.md` §6
- 覆盖需求
  - FR-DM-002, FR-DM-003, FR-DM-004
  - requirements §13.5.1, §13.5.2, §13.5.3
  - AC-004
- 输出物
  - 依赖字段 schema（字段含义、必填、约束、示例）
  - 依赖解析与缺失处理流程图（Mermaid）
  - 删除策略行为定义（`RETAIN/CASCADE_EXPLICIT/BLOCK`）
- DoD
  - 仅解析请求内显式依赖，不推断隐式依赖。
  - `required=true/false` 的失败或跳过行为定义完整，且有日志约束。
  - 不引入“依赖展示模型”或“基于展示分类的强制级联规则”。
- 进度
  - TODO

### T07 values 渲染与 Helm 等效性设计

- 目标
  - 统一参数到 `values.yaml` 的渲染规则，并保证 CUD 执行语义等效 Helm 标准命令。
- 对应章节
  - `docs/design.md` §7
- 覆盖需求
  - FR-CORE-004
  - FR-API-006
  - FR-EXEC-004
  - NFR-005
  - AC-005, AC-006
  - FR-API-005（渲染错误分支）
  - GC-10
- 输出物
  - 渲染输入语义与输出形态规范（YAML 数据/可执行输入文件）
  - 渲染一致性策略（确定性、摘要/指纹追溯）
  - `Create/Update` 与 `helm install/upgrade -f values.yaml` 等效性规则与验证方案
- DoD
  - 预览渲染与执行渲染复用同一规则链路，不允许双轨实现。
  - 同请求+同模板版本重复渲染语义等价。
  - 执行输入可追溯到渲染结果摘要，并定义异常分支错误码。
- 进度
  - TODO

### T08 无状态执行与查询架构设计

- 目标
  - 在“无持久化强依赖”前提下，定义执行上下文、Helm/K8S 事实读取与查询降级策略。
- 对应章节
  - `docs/design.md` §8
- 覆盖需求
  - FR-EXEC-002
  - FR-DM-001
  - NFR-001, NFR-003
  - FR-Q-001（上下文事实来源与一致性）
- 输出物
  - 无状态执行组件图（Mermaid）
  - 执行/查询事实来源说明（K8S API / Helm feedback）
  - 查询上下文不可达时的返回策略（错误码与 `UNKNOWN` 关系）
- DoD
  - 明确“不强制依赖 PostgreSQL/Redis”且不给出反向硬依赖。
  - 查询结果语义限定为“当前任务状态与结果”，不承诺历史时间线。
  - 可靠性目标只覆盖需求定义边界，不越界承担上层并发治理。
- 进度
  - TODO

### T09 集成边界与可审计性设计

- 目标
  - 约束上层模块与 Aether 的调用分工，并定义全链路结构化日志规范。
- 对应章节
  - `docs/design.md` §9
- 覆盖需求
  - FR-INT-001
  - NFR-004
  - GC-03（接口边界映射）
- 输出物
  - 上层 OpenAPI 到 Aether 接口映射表（职责与输入转换）
  - 前置校验责任清单（必须由上层完成的业务约束）
  - 结构化日志字段规范（操作者、请求摘要、执行结果、关联 ID）
- DoD
  - 设计明确上层可选“先 Render 后 CUD”与“直接 CUD 内部 Render”两条路径。
  - 前置业务约束不下沉到 Aether。
  - 审计日志字段可支撑问题追踪，不依赖 Aether 自身持久化。
- 进度
  - TODO

### T10 验收追踪矩阵与完成性收口

- 目标
  - 形成 FR/NFR/AC -> 设计章节 -> 测试点（TC）闭环，并验证设计达到可编码粒度。
- 对应章节
  - `docs/design.md` §10
- 覆盖需求
  - AC-001, AC-002, AC-003, AC-004, AC-005, AC-006
  - requirements §13.6（TC-001~TC-022）
  - GC-07, GC-08, GC-09
- 输出物
  - 验收追踪矩阵（需求 ID、章节、测试点、验证证据）
  - Mermaid 合规性检查清单
  - 设计完成度门禁清单（数据结构/状态/契约/错误/并发幂等/时序/边界）
- DoD
  - AC 条目全部映射到具体设计章节与至少一个 TC。
  - `TC-001~TC-022` 全部有对应设计支撑点，不存在“无设计依据测试”。
  - 通过“可直接指导编码”检查，不留关键空白章节。
- 进度
  - TODO

## 3. 无遗漏覆盖校验

### 3.1 需求条目 -> 主责任务映射（唯一 Owner）

| 需求ID | 主责任务 |
| --- | --- |
| FR-ACL-001 | T02 |
| FR-ACL-002 | T02 |
| FR-ACL-003 | T02 |
| FR-ACL-004 | T02 |
| FR-RT-001 | T05 |
| FR-RT-002 | T05 |
| FR-RT-003 | T05 |
| FR-RT-004 | T05 |
| FR-RT-005 | T05 |
| FR-RT-006 | T05 |
| FR-RT-007 | T05 |
| FR-WS-001 | T02 |
| FR-WS-002 | T02 |
| FR-WS-003 | T02 |
| FR-CORE-001 | T05 |
| FR-CORE-002 | T05 |
| FR-CORE-003 | T03 |
| FR-CORE-004 | T07 |
| FR-API-001 | T03 |
| FR-API-002 | T03 |
| FR-API-003 | T03 |
| FR-API-004 | T03 |
| FR-API-005 | T03 |
| FR-API-006 | T03 |
| FR-EXEC-001 | T04 |
| FR-EXEC-002 | T08 |
| FR-EXEC-003 | T04 |
| FR-EXEC-004 | T07 |
| FR-Q-001 | T04 |
| FR-Q-002 | T04 |
| FR-DM-001 | T08 |
| FR-DM-002 | T06 |
| FR-DM-003 | T06 |
| FR-DM-004 | T06 |
| NFR-001 | T08 |
| NFR-002 | T05 |
| NFR-003 | T08 |
| NFR-004 | T09 |
| NFR-005 | T07 |
| FR-INT-001 | T09 |
| AC-001 | T10 |
| AC-002 | T10 |
| AC-003 | T10 |
| AC-004 | T10 |
| AC-005 | T10 |
| AC-006 | T10 |
| requirements §13.1 | T01 |
| requirements §13.2 | 全局约束（GC-01~GC-10）+ T10 |
| requirements §13.3 | T04 |
| requirements §13.4 | T05 |
| requirements §13.5 | T06 |
| requirements §13.6 | T10 |

### 3.2 验收测试点（TC） -> 主责任务映射

| TC-ID | 主责任务 |
| --- | --- |
| TC-001 | T03 |
| TC-002 | T04 |
| TC-003 | T04 |
| TC-004 | T02 |
| TC-005 | T02 |
| TC-006 | T04 |
| TC-007 | T04 |
| TC-008 | T06 |
| TC-009 | T06 |
| TC-010 | T06 |
| TC-011 | T08 |
| TC-012 | T05 |
| TC-013 | T05 |
| TC-014 | T09 |
| TC-015 | T08 |
| TC-016 | T09 |
| TC-017 | T02 |
| TC-018 | T06 |
| TC-019 | T07 |
| TC-020 | T07 |
| TC-021 | T03 |
| TC-022 | T07 |

### 3.3 No-Gap 自检结论（对应 gap-splitter checklist）

- Gap Completeness：通过。所有 FR/NFR/AC 与 §13 附录条目均映射到唯一主责任务。
- Requirement Traceability：通过。每个任务均标注了需求 ID 与对应设计章节。
- Design Scope Coverage：通过。`docs/design.md` 预分配章节已覆盖接口、状态机、扩展、依赖、渲染、架构、集成、验收。
- DoD Testability：通过。每个任务 DoD 均可通过文档证据审查，不含模糊表述。
- Dependency Soundness：通过。依赖链无环，核心基线任务（T01/T03）先行。
- Status Planning Integrity：通过。全部任务状态为 `TODO`，未错误标记为 `DONE`。
