# 后端开发手册（Backend Handbook）

## 0. 文档信息

| 项目 | 内容 |
| :--- | :--- |
| 文档名称 | 后端开发手册（Backend Handbook） |
| 文档版本 | V1.1 |
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
| alert_id/level/type/title/created_at | task.alerts[] | notification_inbox(type in TASK_PROGRESS/FENCE_ALERT/TASK_CLOSED) |
| courier_name | order.courierName | tag_apply_record.courier_name |
| note_id/kind/content/tags | memory.note | patient_memory_note.* |
| archived_at | aiSession.archivedAt | ai_session.archived_at |
| status(archive) | aiSession.status | ai_session.status |
| export_type/window_start/window_end/reason | export.request | sys_log.detail(export_payload) |
| export_result/file_url | export.result | sys_log.detail(export_result) |
| operator_user_id/operator_username | audit.operator | sys_log.operator_user_id/operator_username |
| invitation.status | guardianInvitation.status | guardian_invitation.status |
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