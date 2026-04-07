# 后端开发手册（Backend Handbook）

## 0. 文档信息

| 项目 | 内容 |
| :--- | :--- |
| 文档名称 | 后端开发手册（Backend Handbook） |
| 文档版本 | V1.2 |
| 日期 | 2026-04-07 |
| 适用对象 | 后端研发、测试、运维、架构、技术管理 |
| 文档目标 | 任何研发成员按本文可开发并交付同构后端系统 |

### 0.1 文档定位

本手册是工程执行文档，不是概念说明文档。本文所有规则按三类关键词表达：

1. 必须：违反即视为实现不合规，禁止合并。
2. 应当：默认执行，偏离需设计评审与记录。
3. 可选：在满足前两类前提下按场景启用。

### 0.2 上位基线（权威顺序）

1. 需求基线：SRS_simplify.md。
2. 架构基线：SADD_from_SRS_simplify.md。
3. 详细设计基线：LLD_from_SRS_SADD.md。
4. 外部契约基线：API_from_SRS_SADD_LLD.md。
5. 数据落库基线：database_design_v5.md。

冲突处理规则：

1. 跨文档冲突时，优先满足 SRS_simplify.md 的需求语义。
2. 同层冲突时，优先满足最新版本号文档。
3. 接口字段与状态码争议时，以 API_from_SRS_SADD_LLD.md 为联调契约。
4. 字段能否落库争议时，以 database_design_v5.md 与 LLD_from_SRS_SADD.md 的一致交集为准。

### 0.3 架构硬约束（必须）

| 编号 | 约束 | 执行要求 |
| :--- | :--- | :--- |
| HC-01 | TASK 域是任务状态机唯一权威，AI 仅发布建议 | 禁止 AI 服务直接改 rescue_task.status |
| HC-02 | 核心状态变更必须本地事务 + Outbox 同提交 | 任何核心状态事件不得绕过 Outbox |
| HC-03 | 所有写接口必须支持 request_id 幂等 | 无幂等键写接口禁止上线 |
| HC-04 | 全链路必须透传 trace_id | 无 trace_id 的请求在网关拒绝 |
| HC-05 | WebSocket 必须路由后定向下发 | 禁止全量广播 |
| HC-06 | 通知不依赖短信 | 通知仅站内与应用推送 |

### 0.4 SRS 五段追踪矩阵（需求 -> 接口 -> 事件 -> 表 -> 测试）（必须）

| 需求 | 接口（主） | 事件 | 表 | 测试 |
| :--- | :--- | :--- | :--- | :--- |
| FR-CLUE-008（BOUND 路由） | GET /r/{resource_token}；GET /p/{short_code}/clues/new | 无（网关路由行为） | tag_asset、patient_profile | ACC-BE-001 |
| FR-CLUE-008（LOST 路由） | GET /r/{resource_token}；GET /p/{short_code}/emergency/report | 无（网关路由行为） | tag_asset、patient_profile | ACC-BE-002 |
| FR-CLUE-008 + BR-007（UNBOUND/ALLOCATED/VOID 拦截） | GET /r/{resource_token} | 无（禁止进入上报链路） | tag_asset | ACC-BE-003 |
| FR-CLUE-003 + BR-001（匿名兜底准入） | POST /api/v1/public/clues/manual-entry | 无（匿名准入，不产生日志外业务事件） | patient_profile、tag_asset、sys_log | ACC-BE-004 |
| FR-CLUE-005（存疑线索双分支闭环） | POST /api/v1/clues/{clue_id}/override；POST /api/v1/clues/{clue_id}/reject | clue.validated（override=true）、clue.rejected | clue_record、sys_outbox_log、consumed_event_log、sys_log | ACC-BE-005 |
| FR-PRO-005（主监护双阶段转移） | POST /api/v1/patients/{patient_id}/guardians/primary-transfer；POST /api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/confirm；POST /api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/cancel | 无独立 Kafka 事件（本期同事务状态更新） | sys_user_patient、guardian_invitation、sys_log | ACC-BE-006 |
| FR-PRO-006 + BR-006（成员移除后转移失效） | DELETE /api/v1/patients/{patient_id}/guardians/{user_id} | 无独立 Kafka 事件（同事务取消未决转移） | sys_user_patient、guardian_invitation、sys_log | ACC-BE-007 |
| FR-TASK-003/004（误报关闭门禁） | POST /api/v1/rescue/tasks/{task_id}/close | task.false_alarm、task.state.changed（仅合法关闭时） | rescue_task、sys_outbox_log、sys_log | ACC-BE-008 |
| FR-PRO-008（围栏配置语义） | PUT /api/v1/patients/{patient_id}/fence | fence.breached（运行时由判定链路触发） | patient_profile | ACC-BE-009 |
| FR-TASK-005 + HC-06（通知多通道触达） | POST /api/v1/rescue/tasks；POST /api/v1/rescue/tasks/{task_id}/close（触发通知） | task.created、task.resolved、task.false_alarm、fence.breached、track.updated | notification_inbox、sys_outbox_log、consumed_event_log | ACC-BE-010 |

---

## 1. 项目启动与本地开发环境

### 1.1 环境要求（必须）

1. JDK：21 LTS。
2. Maven：3.9+。
3. Docker / Docker Compose：用于本地依赖。
4. PostgreSQL：16（扩展 PostGIS、pgvector）。
5. Redis：7+。
6. Kafka：3.x。

### 1.2 本地依赖启动顺序（必须）

1. PostgreSQL（含扩展）。
2. Redis。
3. Kafka。
4. 网关与鉴权服务。
5. 领域服务（profile/task/clue/material/ai/governance）。
6. ws-gateway-service 与 notify-service。

### 1.3 本地启动检查项

1. /actuator/health 全部 UP。
2. Flyway 迁移成功，无 pending migration。
3. Kafka Topic 已创建并可写可读。
4. Redis 路由键与限流键写入正常。
5. API smoke case 通过：
   - 登录 -> 发起任务 -> 匿名上报 -> 管理复核 -> 关闭任务。

### 1.4 配置分层规范

1. application.yml：默认配置（不可含密钥）。
2. application-local.yml：本地覆盖。
3. 环境变量：敏感配置（DB 密码、JWT 密钥、第三方 AK/SK）。
4. sys_config：治理域动态配置，仅用于白名单键。

---

## 2. 项目结构目录与模块职责

### 2.1 统一仓库结构

```text
backend/
  docs/
    backend_handbook.md
    backend_development_handbook_outline.md
    architecture/
    api/
    db/
  scripts/
    local/
    ci/
  platform/
    starter-core/
    starter-web/
    starter-security/
    starter-observability/
  common/
    common-domain/
    common-infra/
    common-test/
  gateway/
    gateway-security/
    auth-service/
    risk-service/
  services/
    profile-service/
    task-service/
    clue-intake-service/
    clue-analysis-service/
    clue-trajectory-service/
    material-service/
    ai-orchestrator-service/
    ai-vectorizer-service/
    notify-service/
    ws-gateway-service/
    admin-review-service/
    outbox-dispatcher/
  integration-tests/
  pom.xml
  README.md
```

### 2.2 每个服务固定包结构（必须）

```text
service-x/
  src/main/java/.../
    interfaces/           # Controller、RequestVO、ResponseVO
    application/          # UseCase、CommandHandler、QueryHandler
    domain/               # Entity、ValueObject、DomainService、Repository接口
    infrastructure/       # DO、Mapper、Repository实现、MQ适配、外部SDK适配
    converter/            # MapStruct/Assembler
    config/               # Spring配置
  src/test/java/.../
    unit/
    integration/
    contract/
```

### 2.3 依赖方向（必须）

1. interfaces -> application -> domain。
2. infrastructure 依赖 domain（实现其接口）。
3. domain 不得依赖 interfaces/infrastructure。
4. common-test 可被所有服务测试依赖，不得反向依赖业务模块。

---

## 3. 技术栈与 SDK 清单

### 3.1 版本管理策略

1. 全部版本由 platform 模块 BOM 统一锁定。
2. 业务模块禁止自行指定核心依赖版本。
3. 升级必须走 ADR + 回归验证。

### 3.2 SDK 分组（必须接入）

| 分组 | 组件 | 用途 |
| :--- | :--- | :--- |
| Web | Spring Boot Web, Validation | REST 接口、参数校验 |
| Security | Spring Security, JWT | 鉴权、权限 |
| DB | PostgreSQL Driver, Flyway | 持久化与迁移 |
| GIS | PostGIS/JTS | 围栏、空间计算 |
| Vector | pgvector | RAG 向量召回 |
| Cache | Spring Data Redis, Lua | 幂等、限流、路由 |
| MQ | Spring Kafka | 事件总线 |
| Observe | Micrometer, OpenTelemetry, Logback JSON | 指标、追踪、日志 |
| Mapping | MapStruct | DTO/DO/VO 转换 |
| Test | JUnit5, Mockito, Testcontainers, RestAssured, ArchUnit | 质量门禁 |

### 3.3 AI 相关 SDK（必须）

1. 统一 AI Adapter 层，不得在业务层直接调用供应商 SDK。
2. 支持主模型与降级模型切换。
3. token_usage 必须统一输出键：
   - prompt_tokens
   - completion_tokens
   - total_tokens
   - model_name
   - billing_source

---

## 4. 分层架构与编码规范

### 4.1 控制器层规范

1. 只做协议适配、权限入口校验、参数校验、错误码映射。
2. 不得编写领域规则与跨聚合事务。
3. 每个写接口必须读取并透传 X-Request-Id、X-Trace-Id。

### 4.2 应用层规范

1. 一个用例对应一个 Application Service 或 Handler。
2. 应用层负责事务边界、调用顺序编排。
3. 禁止在应用层直接写 SQL。

### 4.3 领域层规范

1. 状态变更必须通过聚合根方法。
2. 聚合根方法返回领域事件集合。
3. 领域服务仅封装跨实体规则，不做 IO。

### 4.4 基础设施层规范

1. 只负责持久化、MQ、外部依赖适配。
2. Repository 实现必须使用领域接口返回领域对象。
3. 外部 SDK 结果必须转换为内部 DTO。

### 4.5 命名与后缀规范

1. Entity：*Entity。
2. ValueObject：*Value。
3. 持久化对象：*DO。
4. 请求对象：*Request。
5. 响应对象：*Response。
6. 命令对象：*Command。
7. 查询对象：*Query。
8. 事件对象：*Event。
9. 转换器：*Mapper 或 *Converter。

### 4.6 代码风格与静态检查（必须）

1. 仓库提交 Checkstyle 配置。
2. SonarLint 规则文件入库并在 IDE 启用。
3. PR 流水线执行：checkstyle + spotbugs + sonar gate。
4. 任一门禁失败，PR 禁止合并。

---

## 5. 内部数据结构与流转映射规范（重点）

### 5.1 对象职责定义

| 类型 | 作用域 | 用途 | 禁止事项 |
| :--- | :--- | :--- | :--- |
| Entity | domain | 业务行为与状态机 | 公开 setStatus 等破坏守卫的方法 |
| ValueObject | domain | 不可变值语义 | 可变字段 |
| DO | infrastructure | 表结构映射 | 出现在 Controller 入参/出参 |
| DTO | application/infrastructure | 内部传输 | 对外 API 直出 |
| VO | interfaces | API 输出契约 | 携带数据库技术字段 |
| Command | application | 写场景输入 | 直接拼 SQL |
| Query | application | 读场景输入 | 混入写语义 |
| EventPayload | domain/integration | 事件消息体 | 缺少版本、trace_id |

### 5.2 DTO -> Command/Query -> Aggregate -> DO -> DomainEvent

1. Controller 收到 RequestVO，转换为 Command/Query。
2. Application 执行命令，调用 Aggregate 行为方法。
3. Aggregate 返回新状态 + DomainEvent。
4. Infrastructure 保存 DO，并同事务写 Outbox。
5. Dispatcher 异步发布 EventPayload。

### 5.3 MapStruct 转换规则（必须）

1. 所有核心路径对象转换必须走 MapStruct。
2. 禁止在 Controller 手写字段逐个 copy（测试桩除外）。
3. MapStruct 必须启用 unmappedTargetPolicy=ERROR。

示例：

```java
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.ERROR)
public interface RescueTaskMapper {
    RescueTaskEntity toEntity(RescueTaskDO d);
    RescueTaskDO toDO(RescueTaskEntity e);
    RescueTaskDetailResponse toDetailVO(RescueTaskEntity e);
}
```

### 5.4 核心字段语义规则（必须）

1. ID 字段在 API 线传必须为 string。
2. reported_by/start_time/end_time 为 API 语义名，对应 DB created_by/created_at/closed_at。
3. assignee_user_id 为复核责任人标准字段；assigned_to 仅兼容保留，后端统一映射为 clue_record.assignee_user_id。
4. clue.source_type 必须映射 clue_record.source_type（SCAN/MANUAL），并与坐标标准化链路一致。
5. clue_record.review_status 仅可疑线索可用，非可疑必须为 null。
6. coord_system 入库固定 WGS84。

---

## 6. 核心实体设计与状态机

### 6.1 RescueTask

关键字段：

1. id, task_no, patient_id。
2. status: ACTIVE/RESOLVED/FALSE_ALARM。
3. event_version。
4. created_by, created_at, closed_at。

状态机（必须）：

| 当前状态 | 触发事件 | 下一状态 | 守卫 |
| :--- | :--- | :--- | :--- |
| - | task.create | ACTIVE | 同患者无 ACTIVE |
| ACTIVE | task.close.by_family | RESOLVED | 授权通过 |
| ACTIVE | task.close.false_alarm | FALSE_ALARM | 误报条件满足 |
| ACTIVE | task.close.force | RESOLVED | SUPERADMIN |
| RESOLVED/FALSE_ALARM | any | 原状态 | 终态不可变 |

实现要求：

1. 更新必须条件更新 where status='ACTIVE'。
2. 更新成功 event_version +1。
3. 必须发布 task.state.changed。

### 6.2 ClueRecord

关键字段：

1. risk_score, suspect_flag。
2. review_status（nullable）。
3. override/reject 审核字段。

强约束（必须）：

1. suspect_flag=false 时 review_status 必须为 null。
2. suspect_flag=true 时 review_status 取 PENDING/OVERRIDDEN/REJECTED。

### 6.3 线索补证能力（毕设精简）

当前毕设版本不启用 3.2.13 补证接口，不落独立 `clue_evidence_request` 表。

实现约束：

1. 相关操作仅保留审计记录，写入 `sys_log.detail`。
2. 若后续恢复该能力，再引入独立补证表与状态机。

### 6.4 TagApplyRecord

状态机主链：

1. PENDING -> PROCESSING -> SHIPPED -> COMPLETED。
2. PROCESSING -> CANCEL_PENDING -> CANCELLED/PROCESSING。
3. SHIPPED -> EXCEPTION -> PROCESSING/CANCELLED。

必填与闭环：

1. 发货需 tracking_number，可选 courier_name。
2. 终态写 closed_at。
3. 取消审核通过（CANCEL_PENDING -> CANCELLED）时，approved_at 与 closed_at 必须同事务写入。

### 6.5 TagAsset

状态机：

1. UNBOUND -> ALLOCATED -> BOUND。
2. BOUND -> LOST。
3. LOST/VOID 管理员高危纠错。

### 6.6 AiSession

规则：

1. 禁止全量覆盖写 messages。
2. 仅允许 jsonb 原子追加或 version CAS。
3. 长会话拆分 ai_session_message。
4. 会话归档接口必须更新 ai_session.status=ARCHIVED，并写 archived_at。

### 6.7 OutboxEvent 与 ConsumedEvent

Outbox phase：PENDING/DISPATCHING/SENT/RETRY/DEAD。

幂等表唯一键：consumer_name + topic + event_id + processed_at。

重放定位规则（必须）：

1. DEAD 重放定位必须使用 event_id + created_at 复合键，禁止仅凭 event_id 查询。
2. 重放成功必须回写 replay_reason/replay_token/last_intervention_*。

### 6.8 匿名扫码动态路由决策（必须）

标签状态 -> 路由行为：

| tag_asset.status | 路由结果 | 令牌策略 | 安全约束 |
| :--- | :--- | :--- | :--- |
| BOUND | 302 -> /p/{short_code}/clues/new | 下发 entry_token（Cookie） | 匿名普通模式 |
| LOST | 302 -> /p/{short_code}/emergency/report | 下发 entry_token（Cookie） | 匿名紧急模式 |
| UNBOUND/ALLOCATED/VOID | 无效页或拒绝 | 不得下发 entry_token | 禁止进入线索上报链路 |

实现约束：

1. entry_token 必须设置为 HttpOnly + Secure + SameSite=Strict，Max-Age <= 120。
2. APP/MINI_PROGRAM 调用 /r/{resource_token} 时，必须在响应头回写 X-Anonymous-Token，值与 entry_token 完全一致。
3. 当 Cookie(entry_token) 与 X-Anonymous-Token 同时存在但值不一致，必须拒绝 E_CLUE_4012。

### 6.9 监护邀请与主监护转移状态机（必须）

监护邀请→关系激活事务规则：

1. 邀请 ACCEPTED 后，必须在**同一事务**内创建或激活 `sys_user_patient` 行（`relation_role=GUARDIAN, relation_status=ACTIVE`）。
2. 若 `sys_user_patient` 已存在且 `relation_status=REVOKED`，同事务更新为 `ACTIVE` 并重置 `transfer_state=NONE`。
3. 若已存在 `relation_status=ACTIVE` 行，幂等返回成功。
4. 邀请 REJECTED 时不得创建或修改 `sys_user_patient`。

主监护转移状态机：

| 当前 transfer_state | 触发动作 | 下一状态 | 守卫 |
| :--- | :--- | :--- | :--- |
| NONE | transfer.request | PENDING_CONFIRM | 同患者仅允许一个 PENDING_CONFIRM |
| PENDING_CONFIRM | transfer.confirm(ACCEPT) | ACCEPTED | 仅目标受方 + relation_status=ACTIVE |
| PENDING_CONFIRM | transfer.confirm(REJECT) | REJECTED | reject_reason 必填 |
| PENDING_CONFIRM | transfer.cancel | CANCELLED | 仅原发起方或 SUPERADMIN |
| PENDING_CONFIRM | transfer.expire | EXPIRED | 定时任务触发 |
| ACCEPTED/REJECTED/CANCELLED/EXPIRED | any | 原状态 | 终态不可变 |

并发与竞态约束：

1. DELETE /api/v1/patients/{patient_id}/guardians/{user_id} 时，若目标成员存在 PENDING_CONFIRM 请求，必须同事务取消该请求。
2. 被移除成员发起的历史确认请求必须失效，统一返回状态冲突（E_PRO_4099）或受方身份失效（E_PRO_4011/E_PRO_4013）。
3. transfer_request_id 必须全局唯一，且转移发起/确认/拒绝/撤销字段必须完整落审计。
4. transfer_state=CANCELLED 时，transfer_cancelled_by/transfer_cancelled_at/transfer_cancel_reason 必须成组非空；非 CANCELLED 时三者必须为空。

### 6.10 存疑线索复核闭环状态机（必须）

| suspect_flag | review_status | 允许动作 | 结果事件 |
| :--- | :--- | :--- | :--- |
| false | NULL | 不入复核队列 | 无 |
| true | PENDING | override | clue.validated（override=true） |
| true | PENDING | reject | clue.rejected |
| true | OVERRIDDEN/REJECTED | 只读 | 终态，不可重复处置 |

字段守卫：

1. review_status=OVERRIDDEN 时，override=true 且 override_reason 必填。
2. review_status=REJECTED 时，rejected_by 与 reject_reason 必须成对非空。
3. review_status in (OVERRIDDEN, REJECTED) 时，reviewed_at 必须非空。
4. review_status!=OVERRIDDEN 时，override=false 且 override_reason 为空。
5. review_status!=REJECTED 时，rejected_by 与 reject_reason 必须同时为空。

### 6.11 围栏配置与抑制语义（必须）

1. fence_enabled=true 时，fence_center 与 fence_radius_m 必须同时存在，且半径范围 50-5000 米。
2. fence_enabled=false 时，fence_center 与 fence_radius_m 必须同时置空。
3. lost_status=NORMAL 时允许 fence.breached 触发告警；lost_status=MISSING 时必须抑制围栏告警风暴，但保留 track.updated 进展更新。
4. 围栏判定必须基于 task.state.changed 的 L1/L2 投影缓存，不得高频同步 RPC 拉取 task-service。

lost_status 状态驱动规则（参见 LLD §6.1.1）：

5. `lost_status` 完全由事件驱动，不提供独立写入 API：task.created → MISSING，task.resolved/task.false_alarm → NORMAL。
6. profile-service 消费 task 域事件执行条件更新，必须使用 `lost_status_event_time` 防乱序覆盖（event_time <= lost_status_event_time 时丢弃）。
7. 网关坐标转换在进入业务层之前完成（详见 §7.2），业务服务收到的坐标恒为 WGS84。

---

## 7. API 开发实现规范

### 7.1 通用协议（必须）

1. Base URL：/api/v1。
2. 写接口必须带 X-Request-Id。
3. 全链路必须带 X-Trace-Id。
4. Content-Type 必须 application/json。

### 7.2 Header 与网关职责

1. X-User-Id、X-User-Role 仅网关注入。
2. 客户端伪造内部头必须拒绝 E_REQ_4003。
3. 匿名凭据优先从 Cookie(entry_token) / X-Anonymous-Token 读取。
4. 当 X-Action-Source=AI_AGENT 时，必须要求同时传入 X-Agent-Profile、X-Execution-Mode、X-Confirm-Level。
5. 网关必须在进入业务服务前执行 Agent 策略门禁（权限 + 归属 + 确认等级）。
6. MANUAL_ONLY 接口被 Agent 调用时必须拒绝 E_GOV_4231。

能力包开关映射（`sys_config.scope=ai_policy`）：

| X-Agent-Profile | config_key |
| :--- | :--- |
| RescueCommander | agent.capability.rescue.enabled |
| ClueInvestigator | agent.capability.clue.enabled |
| GuardianCoordinator | agent.capability.guardian.enabled |
| MaterialOperator | agent.capability.material.enabled |
| AICaseCopilot | agent.capability.ai_case.enabled |
| GovernanceSentinel | agent.capability.governance.enabled |
| OutboxReliabilityAgent | agent.capability.outbox_reliability.enabled |

### 7.3 错误码与状态码

1. 业务码不可丢失，HTTP 状态与业务码双轨表达。
2. 新错误码必须先入错误码字典，再实现。
3. 接口示例与状态码矩阵必须一致。
4. Agent 相关错误码必须统一：E_GOV_4039、E_GOV_4097、E_GOV_4226、E_GOV_4231。

### 7.4 Ownership Check（必须）

任一 patient_id/task_id/clue_id/order_id 相关接口，Controller 必须先做数据归属校验。

伪代码：

```java
void assertOwnership(Long operatorId, Long patientId, Role role) {
    if (role.isAdmin()) return;
    boolean related = userPatientRepo.existsActiveRelation(operatorId, patientId);
    if (!related) throw BizException.of("E_PRO_4030");
}
```

### 7.5 分页规范

1. 普通列表：Offset（page_no/page_size/total/has_next）。
2. 流水型列表：Cursor（next_cursor/has_next）。

### 7.6 匿名凭据协同协议（必须）

1. 浏览器 H5 仅使用 HttpOnly Cookie(entry_token) 透传匿名凭据，前端 JavaScript 不得读取或拼接。
2. 非浏览器端允许使用 X-Anonymous-Token；若同时带 Cookie 与 Header，且值不一致，网关拒绝 E_CLUE_4012。
3. 跨域匿名请求必须显式开启 credentials（Fetch: include / Axios: withCredentials=true）。
4. 网关返回 Access-Control-Allow-Credentials=true 时必须使用明确 Origin，禁止与通配符 * 同时出现。

### 7.7 GET /r/{resource_token} 动态路由实现规范（必须）

1. 先验签 resource_token，再查询 tag_asset.status 与关联 short_code。
2. BOUND 路由到 /p/{short_code}/clues/new；LOST 路由到 /p/{short_code}/emergency/report。
3. UNBOUND/ALLOCATED/VOID 一律拦截，不得进入匿名上报页面。
4. 对 APP/MINI_PROGRAM 同步回写 X-Anonymous-Token（与 entry_token 等值）。
5. NFC 作为后续能力上线时，感应打开 URL 必须仍为 /r/{resource_token}，复用同一动态路由逻辑，不得新增独立匿名入口路由。

#### 7.7.1 resource_link 生成规则（确定版）

输入参数：
1. `PUBLIC_ENTRY_BASE_URL`（例如 `https://app.example.com`）。
2. `resource_token`（Base64URL，长度 32-1024）。

生成步骤：
1. 校验 `PUBLIC_ENTRY_BASE_URL`：必须为 HTTPS，去除末尾 `/`。
2. 校验 `resource_token`：仅允许 Base64URL 字符集，长度在 32-1024。
3. 生成统一入口：`resource_link = PUBLIC_ENTRY_BASE_URL + "/r/" + resource_token`。
4. 校验生成结果长度不超过 1024（与存储字段约束一致）。
5. 与发货分配事务同提交：`tag_asset UNBOUND -> ALLOCATED` 与 `resource_link` 写入必须同事务。

运行时字段边界（必须）：
1. `resource_token_expire_at` 由令牌服务按 TTL 运行时计算并在查询接口返回。
2. 资源链接口返回的 `status` 属于令牌运行时状态，不作为 `tag_apply_record` 固定持久化列。
3. 持久化层仅保存 `resource_link` 字符串，令牌状态与过期时间以令牌服务为权威。

幂等与轮换规则：
1. 同一 `order_id + tag_code` 重复触发分配时，未发生风险轮换则返回同一 `resource_link`。
2. 标签作废重发、密钥轮换、泄露应急时，必须生成新 `resource_token` 并重建 `resource_link`。
3. NFC 上线后，NDEF URL 写入值必须等于该 `resource_link`，禁止生成第二套入口格式。

#### 7.7.2 可运行前置条件（上线前必过）

1. 环境变量 `PUBLIC_ENTRY_BASE_URL` 已在 dev/test/prod 分环境配置。
2. 网关已放行 `/r/{resource_token}`，并完成 HTTPS 与证书配置。
3. 通过 BOUND/LOST/UNBOUND/ALLOCATED/VOID 五态回归，路由行为与错误码符合约束。
4. 通过异常回归：非法 token、过期 token、密钥轮换后旧 token 失效。
5. 通过可观测性检查：`trace_id/request_id`、审计记录、错误码统计可追踪。

### 7.8 手动兜底入口实现规范（必须）

POST /api/v1/public/clues/manual-entry 必须满足：

1. 输入 short_code（6 位）+ pin_code（6 位）+ captcha_token + device_fingerprint。
2. 成功后同时返回 manual_entry_token 与 Set-Cookie(entry_token)，且两者值一致。
3. 频控阈值至少覆盖：IP 级、设备级、short_code 连续失败冷却。
4. 令牌 TTL <= 120 秒，超时或重放统一返回 E_CLUE_4012。

### 7.9 监护关系接口竞态规范（必须）

1. 发起转移：POST /api/v1/patients/{patient_id}/guardians/primary-transfer。
2. 受方确认/拒绝：POST /api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/confirm。
3. 发起方撤销：POST /api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/cancel。
4. 成员移除：DELETE /api/v1/patients/{patient_id}/guardians/{user_id}，必须同事务取消该成员未决转移请求。

### 7.10 围栏接口输入守卫（必须）

PUT /api/v1/patients/{patient_id}/fence：

1. fence_enabled=true 时，fence_center(lat/lng) 与 fence_radius_m 必填。
2. fence_enabled=false 时，fence_center 与 fence_radius_m 必须置空。
3. 坐标标准化为 WGS84 的转换职责归属**网关层**（gateway-security），业务服务收到的坐标恒为 WGS84，不得在业务层二次转换。非法坐标或转换失败由网关返回 `E_CLUE_4007` 拒绝。

---

## 8. API -> Domain -> DB 映射手册

### 8.1 核心映射表

| API 字段 | 领域字段 | DB 字段 |
| :--- | :--- | :--- |
| task_id | task.id | rescue_task.id |
| reported_by | task.reportedBy | rescue_task.created_by |
| start_time | task.startTime | rescue_task.created_at |
| end_time | task.endTime | rescue_task.closed_at |
| assignee_user_id | clue.assigneeUserId | clue_record.assignee_user_id |
| source_type | clue.sourceType | clue_record.source_type |
| suspect_reason | clue.suspectReason | clue_record.suspect_reason |
| review_status | clue.reviewStatus | clue_record.review_status |
| override | clue.override | clue_record.override |
| override_reason | clue.overrideReason | clue_record.override_reason |
| rejected_by | clue.rejectedBy | clue_record.rejected_by |
| reviewed_at | clue.reviewedAt | clue_record.reviewed_at |
| alert_id/level/type/title/created_at | task.alerts[] | notification_inbox(type in TASK_PROGRESS/FENCE_ALERT/TASK_CLOSED) |
| courier_name | order.courierName | tag_apply_record.courier_name |
| approved_at(cancel_approve) | order.approvedAt | tag_apply_record.approved_at + tag_apply_record.closed_at |
| resource_link | order.resourceLink | tag_apply_record.resource_link |
| resource_token_expire_at | order.resourceTokenExpireAt | 运行时字段（令牌服务，不落固定列） |
| status(resource_link) | order.resourceLinkStatus | 运行时字段（令牌服务，不落固定列） |
| note_id/kind/content/tags | memory.note | patient_memory_note.* |
| archived_at | aiSession.archivedAt | ai_session.archived_at |
| status(archive) | aiSession.status | ai_session.status |
| export_type/window_start/window_end/reason | export.request | sys_log.detail(export_payload) |
| export_result/file_url | export.result | sys_log.detail(export_result) |
| operator_user_id/operator_username | audit.operator | sys_log.operator_user_id/operator_username |
| invitation.status | guardianInvitation.status | guardian_invitation.status |
| transfer_request_id | guardian.transferRequestId | sys_user_patient.transfer_request_id |
| transfer_state | guardian.transferState | sys_user_patient.transfer_state |
| cancel_reason | guardian.transferCancelReason | sys_user_patient.transfer_cancel_reason |
| cancelled_at | guardian.transferCancelledAt | sys_user_patient.transfer_cancelled_at |
| reject_reason(confirm) | guardian.transferRejectReason | sys_user_patient.transfer_reject_reason |
| removed_at | guardian.removedAt | sys_user_patient.updated_at（回传别名） |
| fence_enabled/fence_center/fence_radius_m | profile.fence | patient_profile.fence_enabled/fence_center/fence_radius_m |
| replay.created_at | outboxReplay.createdAt | sys_outbox_log.created_at |
| medical_history.* | profile.medicalHistory | patient_profile.medical_history |
| avatar_url | profile.avatarUrl | patient_profile.photo_url |

状态枚举补充（必须）：

1. guardian_invitation.status：PENDING/ACCEPTED/REJECTED/EXPIRED/REVOKED。
2. ai_session.status：ACTIVE/ARCHIVED。

### 8.2 命名别名规则（必须）

1. API 对外语义可友好命名。
2. Domain 与 DB 保持语义稳定与审计友好。
3. 所有别名必须在映射表登记。

### 8.3 映射变更流程

1. 变更 API 字段。
2. 更新 MapStruct 映射。
3. 更新 DB 映射表。
4. 更新 DDL 注释。
5. 执行映射回归测试。

---

## 9. 数据库设计与迁移规范

### 9.1 建表与字段规范

1. 每表必须有 created_at、updated_at（技术表按需）。
2. 状态字段必须定义 check 约束。
3. 核心唯一约束必须体现业务不变量。

### 9.2 关键表清单（必须）

1. 核心业务：sys_user、patient_profile、sys_user_patient、guardian_invitation、rescue_task、clue_record、patient_trajectory、tag_asset、tag_apply_record、ai_session、patient_memory_note、vector_store、notification_inbox、sys_log、sys_config。
2. 技术表：sys_outbox_log、consumed_event_log。
3. 扩展表：ai_session_message。

### 9.3 分区策略（必须）

1. sys_outbox_log 按 created_at RANGE 分区。
2. consumed_event_log 按 processed_at RANGE 分区。
3. vector_store 默认单表或固定 HASH 分区，禁止 patient_id LIST 分区。

### 9.4 迁移脚本规范

命名：V{yyyymmddHHmm}__{feature}_{change}.sql。

规则：

1. 一次迁移只做一个主题。
2. DDL 必须带注释说明 API 映射关系。
3. 含数据修复时必须提供回滚脚本。

### 9.5 DDL 注释规范（必须）

示例：

```sql
COMMENT ON COLUMN rescue_task.created_by IS 'API.reported_by 映射字段';
COMMENT ON COLUMN rescue_task.created_at IS 'API.start_time 映射字段';
COMMENT ON COLUMN rescue_task.closed_at IS 'API.end_time 映射字段';
```

---

## 10. 缓存、并发控制与任务调度规范

### 10.1 缓存分层

1. L1：进程内本地缓存（短 TTL，热点只读投影）。
2. L2：Redis（跨节点共享）。

场景：

1. 围栏抑制状态：L1+L2。
2. 短码解析热点：L2 优先，L1 二级。
3. 活跃任务快照：L2。

### 10.2 Cache-Aside（必须）

1. 读流程：先查缓存，miss 查 DB 并回填缓存。
2. 写流程：先写 DB，再删缓存。
3. 删除失败必须重试或异步补偿。

### 10.3 防护策略

1. 穿透：空值缓存 + 参数白名单 + 布隆过滤。
2. 击穿：热点 key 互斥重建。
3. 雪崩：TTL 随机抖动 + 多级降级。

### 10.4 Key 命名规范（必须）

格式：{env}:{service}:{domain}:{entity}:{id}:{field}。

示例：

1. prod:task-service:task:snapshot:1001。
2. prod:profile-service:short-code:AB12CD。
3. prod:ws-gateway:route:user:2001。

### 10.5 并发控制

1. 默认优先 CAS（version 字段）。
2. 临界资源再使用分布式锁。
3. 锁释放必须 finally 保证。

### 10.6 分布式锁规范

1. 锁键必须含业务实体 ID。
2. 锁超时时间 > 业务最大执行时间。
3. 续期需可观测，超阈值告警。
4. 禁止跨资源长事务持锁。

### 10.7 定时任务规范

任务类型：

1. 过期转态（invitation/transfer/evidence request）。
2. Outbox DEAD 复核后重放。
3. 日志与技术表归档清理。
4. 向量失效清理。

规则：

1. 分布式任务必须防重执行。
2. 扫描任务必须游标分页，不得全表载入内存。
3. 单批大小和最大运行时必须配置化。

---

## 11. 事件驱动与一致性规范

### 11.1 Topic 清单（必须）

1. clue.reported.raw。
2. clue.validated。
3. clue.suspected。
4. clue.rejected。
5. track.updated。
6. fence.breached。
7. task.created。
8. task.state.changed。
9. task.resolved。
10. task.false_alarm。
11. ai.strategy.generated。
12. ai.poster.generated。
13. profile.created。
14. profile.updated。
15. profile.corrected。
16. profile.deleted.logical。
17. memory.appended。
18. memory.expired。
19. clue.vectorize.requested。
20. material.order.created。
21. tag.bound。

### 11.2 统一 Envelope（必须）

```json
{
  "event_id": "evt_xxx",
  "topic": "task.state.changed",
  "partition_key": "patient_1001",
  "aggregate_id": "task_8848",
  "event_time": "2026-04-05T10:21:00Z",
  "version": 12,
  "request_id": "req_xxx",
  "trace_id": "trc_xxx",
  "producer": "task-service",
  "payload": {}
}
```

### 11.3 Outbox 状态迁移

1. PENDING/RETRY -> DISPATCHING。
2. DISPATCHING -> SENT。
3. DISPATCHING -> RETRY。
4. RETRY -> DEAD（超阈值）。
5. DEAD -> RETRY（人工干预）。

### 11.4 消费幂等与防乱序

1. 消费前先查 uq_consumer_event。
2. 消费成功后同事务写幂等日志。
3. 投影更新仅接受 incoming.version > current.version。

### 11.5 核心事件消费动作矩阵（必须）

| 事件 | 生产方 | 消费方 | 消费动作（必须） |
| :--- | :--- | :--- | :--- |
| clue.suspected | clue-analysis-service | admin-review-service | 创建复核任务，设置 review_status=PENDING |
| clue.validated | clue-analysis-service/admin-review-service | task-service/ai-orchestrator-service | 推进任务进展并触发策略更新 |
| clue.rejected | admin-review-service | clue-analysis-service/governance | 关闭复核链路并记录拒绝审计 |
| task.state.changed | task-service | clue-analysis-service | 更新 L1/L2 状态投影，用于围栏抑制判定 |
| fence.breached | clue-analysis-service | task-service/notify-service | 触发告警事件并写通知落库 |
| task.false_alarm | task-service | profile-service/notify-service | 关闭任务但禁止进入长期经验沉淀 |

---

## 12. 安全与合规规范

### 12.1 鉴权与权限

1. 所有受保护接口必须 JWT。
2. 高危接口仅 SUPERADMIN。
3. 权限失败统一 403 + 业务码。
4. Agent 执行必须执行 A0-A4 分级：A4 操作永不允许自动执行。
5. A2/A3 写操作必须有确认等级门禁（CONFIRM_1/2/3）。

### 12.2 匿名链路

1. resource_token 验签后下发 entry_token。
2. entry_token 必须一次性、短 TTL、防重放。
3. 禁止 query 参数传 token。

### 12.3 敏感信息保护

1. 患者敏感信息不得出现在匿名接口响应。
2. 日志不得输出明文 pin_code、token、密码。
3. photo_url 必须白名单域名。

### 12.4 Data Ownership Check（必须）

1. 任一患者/任务/工单查询先做归属校验。
2. 越权访问统一拒绝，禁止“查不到即404掩盖”策略不一致。

### 12.5 审计要求

审计记录必须包含：

1. operator_user_id。
2. action/module/object_id。
3. result/risk_level。
4. request_id/trace_id。
5. before/after（高危动作）。
6. action_source（USER/AI_AGENT）。
7. agent_profile/execution_mode/confirm_level。
8. blocked_reason（门禁拦截时必填）。

### 12.6 通知触达与兜底矩阵（必须）

| 触发场景 | 一级通道 | 二级通道 | 禁止事项 |
| :--- | :--- | :--- | :--- |
| task.created/task.resolved/task.false_alarm | 应用推送 | 站内通知入箱 | 依赖短信 |
| fence.breached | 应用推送（高优先） | 站内通知 + 未读红点 | 仅 WebSocket 单通道 |
| WebSocket 路由缺失/离线 | 站内通知入箱 | 推送重试队列 | 丢弃告警 |

执行要求：

1. notify-service 必须保证幂等写入 notification_inbox。
2. 推送失败必须可重试并记录失败原因，禁止静默失败。

---

## 13. 长链接与流式数据规范（WebSocket / SSE）

### 13.1 WebSocket 握手

1. 浏览器使用 ws_ticket 或短效 token 握手。
2. 握手失败必须返回可识别业务错误码。
3. 建连后写路由表 user_id -> pod_id。

### 13.2 心跳与重连

1. 心跳周期固定（示例 30s）。
2. 连续 N 次超时断开并清理路由。
3. 客户端重连必须指数退避。

### 13.3 多节点定向下发

1. 业务事件消费者先查 Redis 路由。
2. 通过定向通道 ws.push.{pod_id} 下发。
3. 路由缺失降级站内通知。

### 13.4 SSE（AI 流式）

1. 流事件必须含 sequence_no。
2. 结束必须发送 DONE 事件。
3. 任何异常必须封装为结构化 error frame。
4. DONE 事件必须至少包含 token_usage 与 fallback_response.mode（可空）。
5. 当本次会话触发受控动作执行时，DONE 事件必须追加 `action_id`、`result_code`、`executed_at`。

---

## 14. 三方服务与 AI 大模型集成韧性规范

### 14.1 调用超时（必须）

每个外部调用必须配置：

1. connect timeout。
2. read timeout。
3. total timeout。

### 14.2 重试策略

1. 仅幂等请求可重试。
2. 重试采用指数退避 + 抖动。
3. 达到上限快速失败并打点。

### 14.3 熔断与降级

1. 达到错误阈值触发熔断。
2. AI 调用失败走 fallback_response。
3. 降级模式必须可审计：RULE_BASED/SAFE_GUARD/TOOL_DEGRADED。

### 14.4 配额与限流

1. 双账本：user_id 全局 + patient_id 全局。
2. 写请求先预占，再确认或回滚。
3. 额度不足返回 E_AI_4293。

### 14.5 埋点规范

外部依赖必须上报：

1. 调用次数。
2. 成功率。
3. TP95/TP99。
4. 错误分布（超时/5xx/业务拒绝）。

### 14.6 Agent 执行治理（必须）

1. 策略门禁顺序固定：权限校验 -> 数据归属校验 -> 执行模式校验 -> 确认等级校验。
2. 支持 `dry_run=true` 预检查模式，预检查不得产生副作用。
3. A4 操作（数据导出、日志清理、强制闭环）必须 `MANUAL_ONLY`。
4. Agent 执行链路必须可熔断：可按能力包与执行等级（A2/A3）独立关闭。
5. 连续失败超阈值时自动降级为只读建议模式。
6. Function Calling 的 `action` 必须命中白名单；未知 action、能力包未启用或确认等级不足时必须策略阻断。

Action 白名单映射（对齐 API 9.7 / LLD 10.9）：

| action | 目标接口（方法+路径） | 最低确认 | 执行约束 |
| :--- | :--- | :--- | :--- |
| propose_close | POST /api/v1/rescue/tasks/{task_id}/close | CONFIRM_1 | 可执行 |
| clue_override | POST /api/v1/clues/{clue_id}/override | CONFIRM_2 | 可执行 |
| clue_reject | POST /api/v1/clues/{clue_id}/reject | CONFIRM_2 | 可执行 |
| approve_material_order | PUT /api/v1/admin/material/orders/{order_id}/approve | CONFIRM_2 | 可执行 |
| archive_session | POST /api/v1/ai/sessions/{session_id}/archive | CONFIRM_1 | 可执行 |
| replay_outbox_dead | POST /api/v1/admin/super/outbox/dead/{event_id}/replay | CONFIRM_3 | 可执行 |
| request_evidence | POST /api/v1/admin/clues/{clue_id}/request-evidence | MANUAL_ONLY | 毕设版本暂不开放，仅可建议 |
| force_close_task | POST /api/v1/admin/super/rescue/tasks/{task_id}/force-close | MANUAL_ONLY | A4 动作，仅人工页面可执行 |

---

## 15. 可观测性与运维规范

### 15.1 日志字段规范（必须）

每条业务日志至少包含：

1. timestamp。
2. level。
3. service。
4. trace_id。
5. request_id。
6. user_id（可空）。
7. patient_id/task_id（按场景）。
8. code/message。
9. action_source。
10. agent_profile/execution_mode/confirm_level（Agent 链路）。
11. blocked_reason（策略阻断时）。

### 15.2 指标规范

1. API：QPS、TP95、TP99、错误率。
2. DB：慢 SQL、连接池使用率。
3. Redis：命中率、延迟、keyspace。
4. Kafka：lag、重试、死信数。
5. Outbox：phase 分布、DEAD 数。
6. AI：调用耗时、成功率、配额命中。
7. 运营治理看板指标主链路应来自可观测平台聚合，不得直接以业务库全表扫描作为实时指标来源。

### 15.3 告警规则

1. 核心写接口错误率超阈值告警。
2. Outbox DEAD 增长告警。
3. ws 路由缺失率告警。
4. AI 连续失败告警。

### 15.4 排障手册最小集合

1. 任务无法关闭。
2. 线索无法回流/驳回。
3. 工单无法自动收敛。
4. SSE 中途截断。
5. Outbox 卡在 DEAD。

---

## 16. 测试与质量门禁

### 16.1 测试分层

1. 单元测试：聚合守卫、状态机流转。
2. 集成测试：DB + Redis + Kafka 链路。
3. 契约测试：API 字段、状态码矩阵。
4. 事件测试：Outbox -> MQ -> Consumer。
5. 数据一致性测试：关键字段映射回归。
6. Agent 门禁测试：确认等级、MANUAL_ONLY、dry_run、策略阻断语义回归。

### 16.2 架构守护（ArchUnit，必须）

所有服务必须继承 common-test 基础规则。

示例：

```java
@AnalyzeClasses(packages = "com.company")
public class ArchitectureRulesTest extends BaseArchRules {
}

public abstract class BaseArchRules {
    @ArchTest
    static final ArchRule domainNotDependOnWeb =
        noClasses().that().resideInAPackage("..domain..")
            .should().dependOnClassesThat().resideInAPackage("..interfaces..");

    @ArchTest
    static final ArchRule controllerOnlyCallApplication =
        classes().that().resideInAPackage("..interfaces..")
            .should().onlyDependOnClassesThat()
            .resideInAnyPackage("..interfaces..", "..application..", "java..", "javax..", "org.springframework..");
}
```

### 16.3 静态检查门禁（必须）

CI 必跑：

1. mvn -q checkstyle:check。
2. mvn -q spotbugs:check。
3. mvn -q test（含 ArchUnit）。
4. sonar quality gate。

任一失败，禁止合并。

### 16.4 性能门禁（上线前必须）

1. 核心写接口 TP99 <= 200ms（示例基线，可按容量评审微调）。
2. 跨域一致性收敛 TP99 <= 3s。
3. 向量召回高密度患者 TP95 < 120ms，TP99 < 200ms，recall@10 >= 0.90。

### 16.5 SRS 关键验收测试模板（请求、事件、落库断言）（必须）

执行约定：

1. 所有请求必须带 X-Trace-Id；写请求必须带 X-Request-Id。
2. 事件断言优先使用 Kafka 探针；无探针时使用 sys_outbox_log + consumed_event_log 组合断言。
3. 落库断言统一在时间窗 [t0, t1] 内执行，避免历史数据干扰。

#### 16.5.0 用例索引

| 编号 | 验收场景 | 关联需求 |
| :--- | :--- | :--- |
| ACC-BE-001 | 扫码标签状态为 BOUND | FR-CLUE-008 |
| ACC-BE-002 | 扫码标签状态为 LOST | FR-CLUE-008 |
| ACC-BE-003 | 扫码标签状态为 UNBOUND/ALLOCATED/VOID | FR-CLUE-008、BR-007 |
| ACC-BE-004 | 手动兜底 short_code + pin_code + captcha | FR-CLUE-003 |
| ACC-BE-005 | 可疑线索 override 与 reject 分支 | FR-CLUE-005 |
| ACC-BE-006 | 发起主监护转移并由受方 ACCEPT/REJECT | FR-PRO-005 |
| ACC-BE-007 | 移除存在未决转移请求的成员 | FR-PRO-006、BR-006 |
| ACC-BE-008 | FALSE_ALARM 关闭任务且 reason 缺失 | FR-TASK-003/004 |
| ACC-BE-009 | 围栏开启/关闭输入组合校验 | FR-PRO-008 |
| ACC-BE-010 | WebSocket 离线时任务进展通知 | FR-TASK-005、HC-06 |

#### 16.5.1 ACC-BE-001 扫码标签状态为 BOUND

请求步骤：

1. 准备数据：tag_asset.status=BOUND，且 patient_profile.short_code 有效。
2. 发起 GET /r/{resource_token}。
3. 断言响应为 302，Location=/p/{short_code}/clues/new，且下发 entry_token。

事件断言：

1. 本请求不应发布 clue.reported.raw、clue.validated、clue.rejected。
2. sys_outbox_log 在 [t0,t1] 内不应新增上述 topic 记录。

落库断言（SQL 模板）：

1. SELECT status FROM tag_asset WHERE tag_code=:tag_code; 结果为 BOUND。
2. SELECT COUNT(1) FROM clue_record WHERE patient_id=:patient_id AND created_at BETWEEN :t0 AND :t1; 结果为 0。
3. SELECT COUNT(1) FROM notification_inbox WHERE related_patient_id=:patient_id AND created_at BETWEEN :t0 AND :t1; 结果为 0。

#### 16.5.2 ACC-BE-002 扫码标签状态为 LOST

请求步骤：

1. 准备数据：tag_asset.status=LOST，且 patient_profile.short_code 有效。
2. 发起 GET /r/{resource_token}。
3. 断言响应为 302，Location=/p/{short_code}/emergency/report，且下发 entry_token。

事件断言：

1. 本请求不应直接发布 clue.validated、task.state.changed。
2. sys_outbox_log 在 [t0,t1] 内不应新增由扫码路由直接触发的业务事件。

落库断言（SQL 模板）：

1. SELECT status FROM tag_asset WHERE tag_code=:tag_code; 结果为 LOST。
2. SELECT COUNT(1) FROM clue_record WHERE patient_id=:patient_id AND created_at BETWEEN :t0 AND :t1; 结果为 0。
3. SELECT COUNT(1) FROM notification_inbox WHERE related_patient_id=:patient_id AND created_at BETWEEN :t0 AND :t1; 结果为 0。

#### 16.5.3 ACC-BE-003 扫码标签状态为 UNBOUND/ALLOCATED/VOID

请求步骤：

1. 分别准备 tag_asset.status=UNBOUND、ALLOCATED、VOID 三组数据。
2. 对每组数据发起 GET /r/{resource_token}。
3. 断言返回拦截页或业务拒绝，不下发 entry_token。

事件断言：

1. 不应发布任何 clue.* 或 task.* 事件。
2. sys_outbox_log 在 [t0,t1] 内新增事件数为 0。

落库断言（SQL 模板）：

1. SELECT status FROM tag_asset WHERE tag_code=:tag_code; 状态不变。
2. SELECT COUNT(1) FROM clue_record WHERE tag_code=:tag_code AND created_at BETWEEN :t0 AND :t1; 结果为 0。
3. SELECT COUNT(1) FROM notification_inbox WHERE related_patient_id=:patient_id AND created_at BETWEEN :t0 AND :t1; 结果为 0。

#### 16.5.4 ACC-BE-004 手动兜底 short_code + pin_code + captcha

请求步骤：

1. 准备合法 short_code、pin_code、captcha_token、device_fingerprint。
2. 发起 POST /api/v1/public/clues/manual-entry。
3. 断言 HTTP 201，返回 manual_entry_token，且与 Set-Cookie(entry_token) 一致。

事件断言：

1. 兜底准入阶段不应发布 clue.reported.raw、clue.validated、clue.rejected。
2. sys_outbox_log 在 [t0,t1] 内不应新增 clue.* 事件。

落库断言（SQL 模板）：

1. SELECT short_code, pin_code_hash FROM patient_profile WHERE short_code=:short_code; 记录存在且 pin_code_hash 非空。
2. SELECT COUNT(1) FROM clue_record WHERE patient_id=:patient_id AND created_at BETWEEN :t0 AND :t1; 结果为 0。
3. SELECT COUNT(1) FROM tag_asset WHERE patient_id=:patient_id; 与基线数量一致（准入不改标签状态）。

#### 16.5.5 ACC-BE-005 可疑线索 override 与 reject 分支

请求步骤：

1. 准备两条 suspect_flag=true 且 review_status=PENDING 的线索（clue_a、clue_b）。
2. 发起 POST /api/v1/clues/{clue_a}/override（override=true，override_reason 有效）。
3. 发起 POST /api/v1/clues/{clue_b}/reject（reject_reason 有效）。

事件断言：

1. clue_a 必须发布 clue.validated，且 payload.override=true。
2. clue_b 必须发布 clue.rejected。
3. consumed_event_log 应可观察到下游消费记录（允许最终一致延迟）。

落库断言（SQL 模板）：

1. SELECT review_status, override, override_reason, reviewed_at FROM clue_record WHERE id=:clue_a; 结果为 OVERRIDDEN/true/非空/非空。
2. SELECT review_status, rejected_by, reject_reason, reviewed_at FROM clue_record WHERE id=:clue_b; 结果为 REJECTED/非空/非空/非空。
3. SELECT COUNT(1) FROM sys_outbox_log WHERE topic IN ('clue.validated','clue.rejected') AND created_at BETWEEN :t0 AND :t1; 结果 >= 2。

#### 16.5.6 ACC-BE-006 主监护转移 ACCEPT/REJECT

请求步骤：

1. 发起 POST /api/v1/patients/{patient_id}/guardians/primary-transfer，获得 transfer_request_id。
2. 由受方发起 POST /api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/confirm（action=ACCEPT）。
3. 使用新请求重复步骤 1，并执行 action=REJECT（reject_reason 有效）。

事件断言：

1. 转移链路不强依赖独立 Kafka 事件；断言不产生与 task/clue 领域无关的误事件。
2. sys_outbox_log 在 [t0,t1] 内不应出现与本转移请求无关的 task.false_alarm/clue.rejected 事件。

落库断言（SQL 模板）：

1. ACCEPT 后：SELECT transfer_state, transfer_confirmed_at FROM sys_user_patient WHERE patient_id=:patient_id AND transfer_request_id=:req_accept; 结果为 ACCEPTED/非空。
2. REJECT 后：SELECT transfer_state, transfer_rejected_at, transfer_reject_reason FROM sys_user_patient WHERE patient_id=:patient_id AND transfer_request_id=:req_reject; 结果为 REJECTED/非空/非空。
3. ACCEPT 后主监护切换：校验同 patient_id 仅一个 relation_role=PRIMARY_GUARDIAN 且 relation_status=ACTIVE。

#### 16.5.7 ACC-BE-007 移除存在未决转移请求的成员

请求步骤：

1. 先创建 transfer_state=PENDING_CONFIRM 的转移请求，目标成员为 user_x。
2. 发起 DELETE /api/v1/patients/{patient_id}/guardians/{user_x}。
3. 继续调用旧 transfer_request_id 的 confirm，断言返回 E_PRO_4099 或 E_PRO_4011/E_PRO_4013。

事件断言：

1. 不应出现旧请求被确认成功的后续事件。
2. sys_outbox_log 不应出现与该旧请求关联的成功确认事件。

落库断言（SQL 模板）：

1. SELECT relation_status FROM sys_user_patient WHERE patient_id=:patient_id AND user_id=:user_x; 结果为 REVOKED。
2. SELECT transfer_state FROM sys_user_patient WHERE patient_id=:patient_id AND transfer_request_id=:req_id; 结果为 CANCELLED（或至少非 PENDING_CONFIRM）。
3. SELECT transfer_cancelled_at FROM sys_user_patient WHERE patient_id=:patient_id AND transfer_request_id=:req_id; 结果非空。

#### 16.5.8 ACC-BE-008 FALSE_ALARM 关闭任务且 reason 缺失

请求步骤：

1. 准备 status=ACTIVE 的任务。
2. 发起 POST /api/v1/rescue/tasks/{task_id}/close，close_type=FALSE_ALARM 且不传 reason。
3. 断言返回业务错误码（E_TASK_4005 或同语义错误）。

事件断言：

1. 不应发布 task.false_alarm。
2. 不应发布 task.state.changed。

落库断言（SQL 模板）：

1. SELECT status, closed_at, close_reason FROM rescue_task WHERE id=:task_id; 结果为 ACTIVE/null/原值。
2. SELECT COUNT(1) FROM sys_outbox_log WHERE aggregate_id=:task_aggregate_id AND topic IN ('task.false_alarm','task.state.changed') AND created_at BETWEEN :t0 AND :t1; 结果为 0。
3. SELECT COUNT(1) FROM notification_inbox WHERE related_task_id=:task_id AND created_at BETWEEN :t0 AND :t1; 结果为 0。

#### 16.5.9 ACC-BE-009 围栏开启/关闭输入组合校验

请求步骤：

1. 发起 PUT /api/v1/patients/{patient_id}/fence：fence_enabled=true，携带 fence_center + fence_radius_m。
2. 发起 PUT /api/v1/patients/{patient_id}/fence：fence_enabled=false，显式置空 fence_center + fence_radius_m。
3. 发起非法组合请求（fence_enabled=true 但缺 fence_center 或 fence_radius_m），断言被拒绝。

事件断言：

1. 围栏配置写入不应直接发布 task.false_alarm、clue.rejected。
2. fence.breached 仅允许在后续运行时判定链路中出现。

落库断言（SQL 模板）：

1. 步骤1后：SELECT fence_enabled, fence_center, fence_radius_m FROM patient_profile WHERE id=:patient_id; 结果为 true/非空/范围内。
2. 步骤2后：SELECT fence_enabled, fence_center, fence_radius_m FROM patient_profile WHERE id=:patient_id; 结果为 false/null/null。
3. 步骤3后：patient_profile 记录保持步骤2结果不变。

#### 16.5.10 ACC-BE-010 WebSocket 离线时任务进展通知兜底

请求步骤：

1. 模拟接收用户离线（清理 ws 路由，确保无可用 pod_id）。
2. 触发任务进展写请求（例如 POST /api/v1/rescue/tasks 或合法关闭接口）。
3. 等待 notify-service 消费完成。

事件断言：

1. 触发链路必须发布对应 task.* 或 fence.breached/track.updated 事件。
2. consumed_event_log 中 notify-service 对应 topic 应存在消费记录。

落库断言（SQL 模板）：

1. SELECT type, read_status, related_task_id, related_patient_id, trace_id FROM notification_inbox WHERE user_id=:user_id AND created_at BETWEEN :t0 AND :t1 ORDER BY created_at DESC LIMIT 1; 结果应存在，且 read_status=UNREAD。
2. 结果 type 必须属于 TASK_PROGRESS/FENCE_ALERT/TASK_CLOSED。
3. trace_id 必须与触发请求 trace_id 一致。

---

## 17. 手把手开发流程（新人可执行）

### 17.1 新需求开发 12 步

| 步骤 | 输入 | 输出 | 必查项 |
| :--- | :--- | :--- | :--- |
| 1 | 需求单 | 需求理解记录 | 是否落在六域边界内 |
| 2 | 需求理解 | 领域归属结论 | 是否影响状态机 |
| 3 | 领域归属 | API 草案 | 错误码是否复用现有 |
| 4 | API 草案 | Command/Query 定义 | ID 是否 string 线传 |
| 5 | Command/Query | Entity 行为设计 | 守卫是否完整 |
| 6 | 实体设计 | DDL 变更脚本 | 是否有映射注释 |
| 7 | DDL | Repository/Mapper 实现 | MapStruct 是否无未映射字段 |
| 8 | 代码实现 | Outbox/Event 实现 | 事件 envelope 是否完整 |
| 9 | 代码实现 | 单元测试 | 状态机与边界条件覆盖 |
| 10 | 单测通过 | 集成与契约测试 | API/DB 映射回归通过 |
| 11 | 测试通过 | 代码评审单 | ArchUnit/静态检查绿灯 |
| 12 | 评审通过 | 发布申请 | 灰度与回滚方案齐全 |

### 17.2 代码评审清单

1. 是否新增或破坏 HC-01~HC-06。
2. 写接口是否具备幂等与 trace_id。
3. 是否存在跨层依赖违规。
4. 是否有字段映射遗漏。
5. 是否有状态机非法流转。
6. 是否补齐事件与幂等处理。
7. 是否补齐审计日志。

### 17.3 联调清单

1. 前后端接口字段完全一致。
2. 状态码矩阵与响应示例一致。
3. 匿名链路 token 来源一致（非 body）。
4. WebSocket 与 events/poll 使用同一 envelope 解析器。

### 17.4 文档编写优先级

1. 先完成第 17 章（流程可执行）。
2. 再完成第 19 章（模板可复用）。
3. 再补策略章节。

---

## 18. 发布、灰度、回滚与应急

### 18.1 发布前检查

1. 所有门禁通过。
2. 迁移脚本在预发完成演练。
3. 指标与告警配置已生效。
4. 回滚脚本与负责人明确。

### 18.2 灰度策略

1. 先 5% 流量灰度。
2. 观察 30 分钟关键指标。
3. 无异常再扩容到 25% -> 50% -> 100%。

### 18.3 回滚策略

1. 代码回滚：快速回退上一个稳定版本。
2. 配置回滚：恢复 sys_config 快照。
3. 数据回滚：仅允许前向补偿，禁止回滚已发布事件。

### 18.4 生产应急分工

1. 值班后端：故障止血与回滚。
2. DBA：迁移与数据修复支持。
3. SRE：流量切换与资源处置。
4. 架构负责人：根因复盘与制度修正。

---

## 19. 附录模板

### 19.1 Entity 模板

```java
public class RescueTaskEntity {
    private Long id;
    private Long patientId;
    private TaskStatus status;
    private Long eventVersion;

    public List<DomainEvent> closeByFamily(Long operatorId, String reason) {
        if (this.status != TaskStatus.ACTIVE) {
            throw BizException.of("E_TASK_4093");
        }
        this.status = TaskStatus.RESOLVED;
        this.eventVersion += 1;
        return List.of(new TaskResolvedEvent(this.id, this.patientId, this.eventVersion, operatorId, reason));
    }
}
```

### 19.2 Command 模板

```java
public record CloseTaskCommand(
    Long taskId,
    Long operatorUserId,
    String closeType,
    String reason,
    String requestId,
    String traceId
) {}
```

### 19.3 Controller 模板

```java
@RestController
@RequestMapping("/api/v1/rescue/tasks")
public class RescueTaskController {

    @PostMapping("/{taskId}/close")
    public ApiResponse<CloseTaskResponse> close(
            @PathVariable String taskId,
            @RequestHeader("X-Request-Id") String requestId,
            @RequestHeader("X-Trace-Id") String traceId,
            @RequestBody @Valid CloseTaskRequest request,
            AuthPrincipal principal) {

        CloseTaskCommand cmd = new CloseTaskCommand(
                Long.valueOf(taskId), principal.userId(), request.closeType(), request.reason(), requestId, traceId);
        return ApiResponse.ok(appService.closeTask(cmd));
    }
}
```

### 19.4 MapStruct 模板

```java
@Mapper(componentModel = "spring", unmappedTargetPolicy = ReportingPolicy.ERROR)
public interface ClueMapper {
    ClueRecordEntity toEntity(ClueRecordDO d);
    ClueRecordDO toDO(ClueRecordEntity e);
    ClueDetailResponse toDetailVO(ClueRecordEntity e);
}
```

### 19.5 ArchUnit 模板

```java
@AnalyzeClasses(packages = "com.company")
public class ServiceArchTest extends BaseArchRules {}
```

### 19.6 Checkstyle 配置模板（片段）

```xml
<module name="Checker">
  <module name="TreeWalker">
    <module name="AvoidStarImport"/>
    <module name="FinalClass"/>
    <module name="UnusedImports"/>
    <module name="MethodLength">
      <property name="max" value="80"/>
    </module>
  </module>
</module>
```

### 19.7 DDL 模板（含映射注释）

```sql
CREATE TABLE rescue_task (
  id BIGINT PRIMARY KEY,
  patient_id BIGINT NOT NULL,
  status VARCHAR(20) NOT NULL,
  created_by BIGINT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL,
  closed_at TIMESTAMPTZ NULL,
  updated_at TIMESTAMPTZ NOT NULL
);

COMMENT ON COLUMN rescue_task.created_by IS 'API.reported_by';
COMMENT ON COLUMN rescue_task.created_at IS 'API.start_time';
COMMENT ON COLUMN rescue_task.closed_at IS 'API.end_time';
```

### 19.8 测试用例模板

```markdown
- 用例编号：TASK-CLOSE-001
- 前置条件：任务状态 ACTIVE
- 输入：close_type=RESOLVED, reason="已寻回"
- 期望：
  - HTTP 200
  - status=RESOLVED
  - event_version +1
  - Outbox 生成 task.resolved + task.state.changed
```

### 19.9 上线检查模板

```markdown
- [ ] DB migration 在预发执行成功
- [ ] 核心接口契约测试通过
- [ ] ArchUnit/Checkstyle/Sonar 全绿
- [ ] 告警规则已发布
- [ ] 回滚脚本可执行
```

---

## 20. 文档到代码（Doc as Code）落地计划

### 20.1 规则代码化映射

| 文档规则 | 代码载体 |
| :--- | :--- |
| 分层依赖约束 | common-test ArchUnit |
| 命名与风格 | Checkstyle/Sonar 规则文件 |
| 字段映射约束 | MapStruct + 编译期 ERROR |
| 接口契约一致性 | 契约测试 + OpenAPI 校验 |
| 状态机守卫 | 聚合单元测试 |

### 20.2 CI 流水线（必须）

流水线阶段：

1. compile。
2. checkstyle。
3. unit-test（含 ArchUnit）。
4. integration-test。
5. contract-test。
6. quality-gate。
7. package。

规则：任一阶段失败直接阻断合并。

### 20.3 规则变更流程

1. 提案：架构负责人或模块负责人发起。
2. 评审：至少后端、测试、运维三方通过。
3. 发布：先在一个服务灰度试运行。
4. 全量：所有服务统一升级。

### 20.4 一致性巡检机制

1. 周检：文档规则与代码规则差异巡检。
2. 里程碑复核：发布前全链路检查。
3. 复盘：事故后 48 小时内补齐规则并更新手册。

---

## 21. 开发完成定义（Definition of Done）

一个需求仅在满足以下条件时可标记完成：

1. API、领域、DB 三层字段映射完整且可追溯。
2. 核心状态机守卫有单测覆盖。
3. Outbox 与幂等链路经集成测试验证。
4. 安全与归属校验通过。
5. 观测与告警已配置。
6. 发布与回滚方案齐备。
7. 文档、模板、代码三者一致。

本手册即为后端系统唯一工程执行规范。除非通过规则变更流程，本手册中的“必须”条款不得绕过。