# 基于 AI Agent 的阿尔兹海默症患者协同寻回系统
# 数据库设计文档（重构版）

- 版本：V2.3
- 数据库：PostgreSQL 16
- 扩展：PostGIS、pgvector
- 文档日期：2026-04-06
- 适用范围：作为 API 规格落库基线与后续 DDL 实施依据

## 1. 设计目标与基线来源

本版数据库设计基于以下文档统一收敛：

- SRS_simplify.md（V1.0）：业务规则、验收口径、关键非功能与审计要求
- SADD：架构硬约束（HC-01~HC-06）与域边界
- LLD：数据模型、状态机守卫、Outbox 与幂等机制
- 原 V1.0 数据库设计：字段语义与历史实现习惯

本版重点修正：

1. 保持 HC-06：通知链路仅支持应用推送与站内通知，不引入短信通道。
2. 对齐 API 与存储映射：
   - patient_profile 承载 blood_type/chronic_diseases/allergy_notes（medical_history JSONB）
  - patient_profile.photo_url 作为患者近期正面免冠照片，必填（not null）
  - tag_apply_record 增加 delivery_address（并补 quantity、apply_note、courier_name、closed_at）
   - 增加 sys_config 支撑 3.8.8/3.8.11
3. 补齐消息中心落库：新增 notification_inbox 支撑 3.9.1/3.9.2/3.9.3。
4. 补齐线索复核工作流落库能力：clue_record 增加 review_status、assignee_user_id、assigned_at、reviewed_at、risk_score。
5. 新增 guardian_invitation 资源表，承接监护邀请生命周期；并为 transfer_request_id 增加全局唯一约束。
6. 毕设精简：不引入 clue_evidence_request、emergency_alert、logistics_tracking_event、export_job 四张独立表。
7. 保留并强化 Outbox + 消费幂等技术基线：sys_outbox_log + consumed_event_log。
8. 补齐 PostgreSQL 物理层治理：
  - vector_store 禁止按 patient_id 做 LIST 分区，默认单表或固定数量 HASH 分区。
  - ai_session 为长会话提供 ai_session_message 扩展表，规避 JSONB 大对象写放大。
  - sys_outbox_log 与 consumed_event_log 使用时间 RANGE 分区，按分区淘汰替代大批量 DELETE。

## 2. 全局硬约束（落库视角）

1. HC-01：TASK 域是任务状态机唯一权威，AI 不得直接改写任务状态。
2. HC-02：跨域状态变更必须本地事务提交 + Outbox 发布。
3. HC-03：写请求必须支持 request_id 幂等；数据库侧需有唯一约束/条件更新兜底。
4. HC-04：全链路 trace_id 可追踪；关键写操作必须审计可回放。
5. HC-05：WebSocket 集群精准下发，禁止广播；数据库存储仅承载持久化通知与审计证据。
6. HC-06：通知通道不依赖外部短信，数据库不设计 SMS 专用通道字段。
7. 空间坐标统一标准：数据库内所有空间坐标一律使用 `WGS84(EPSG:4326)`；上游 `GCJ-02/BD-09` 必须在网关标准化后入库。

## 3. 表清单与分类

本版共 17 张必选表 + 1 张扩展表（长会话场景建议启用）：

- 核心业务表（15）：
  - sys_user
  - patient_profile
  - sys_user_patient
  - guardian_invitation
  - rescue_task
  - clue_record
  - patient_trajectory
  - tag_asset
  - tag_apply_record
  - ai_session
  - patient_memory_note
  - vector_store
  - notification_inbox
  - sys_log
  - sys_config
- 技术调度表（1）：
  - sys_outbox_log
- 技术幂等表（1）：
  - consumed_event_log
- 扩展业务表（1，可选）：
  - ai_session_message

说明：

- 相比早期 11+1 基线，本版新增/显式化了 sys_config、consumed_event_log、notification_inbox、guardian_invitation、patient_memory_note。
- notification_inbox 为 API 3.9 消息中心接口所需持久化能力，不新增业务域，仅补齐治理域通知能力落库。
- guardian_invitation 承接 3.3 监护邀请生命周期（PENDING/ACCEPTED/REJECTED/EXPIRED/REVOKED），sys_user_patient 保持已生效关系快照。
- patient_memory_note 承接 3.5.5/3.5.6 记忆原始条目；向量化结果落 vector_store。
- 3.1.15 任务告警分页直接复用 notification_inbox（按 type/level/related_task_id 过滤）。
- 3.4.24 物流轨迹查询在毕设版本暂不开放，不落独立轨迹表。
- 3.8.6 数据导出不建任务表，导出请求与结果写入 sys_log.detail。
- 3.2.13 补证请求在毕设版本暂不开放，不落独立补证表。
- ai_session_message 用于长会话拆分存储，短会话可继续使用 ai_session.messages。

## 4. 数据表详细设计

### 4.1 sys_user（系统用户）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | bigint | PK | 用户主键 |
| username | varchar(64) | not null unique | 登录名 |
| password_hash | varchar(128) | not null | BCrypt 哈希 |
| display_name | varchar(64) | null | 显示名 |
| phone | varchar(20) | null | 手机号 |
| role | varchar(20) | not null default 'FAMILY' | FAMILY/ADMIN/SUPERADMIN |
| status | varchar(20) | not null default 'NORMAL' | NORMAL/BANNED |
| last_login_at | timestamptz | null | 最近登录时间 |
| last_login_ip | varchar(64) | null | 最近登录 IP |
| created_at | timestamptz | not null | 创建时间 |
| updated_at | timestamptz | not null | 更新时间 |

约束：

- check(role in ('FAMILY','ADMIN','SUPERADMIN'))
- check(status in ('NORMAL','BANNED'))

索引：

- ux_sys_user_username(username)
- idx_sys_user_phone(phone)
- idx_sys_user_role_status(role, status)

### 4.2 patient_profile（患者档案）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | bigint | PK | 患者主键 |
| profile_no | varchar(32) | unique | 业务编号 |
| name | varchar(64) | not null | 姓名 |
| gender | varchar(16) | not null | MALE/FEMALE/UNKNOWN |
| birthday | date | not null | 出生日期 |
| short_code | char(6) | unique not null | 外显短码 |
| pin_code_hash | varchar(128) | not null | 手动兜底 PIN 哈希 |
| pin_code_salt | varchar(64) | not null | PIN 盐值 |
| photo_url | varchar(1024) | not null | 患者近期正面免冠照片 |
| medical_history | jsonb | not null default '{}'::jsonb | 医疗扩展信息 |
| fence_enabled | boolean | not null default false | 围栏开关 |
| fence_center | geometry(Point,4326) | null | 围栏中心点 |
| fence_radius_m | int | null | 围栏半径（米） |
| lost_status | varchar(20) | not null default 'NORMAL' | NORMAL/MISSING |
| lost_status_event_time | timestamptz | not null | 状态锚点 |
| profile_version | bigint | not null default 1 | 档案版本 |
| created_at | timestamptz | not null | 创建时间 |
| updated_at | timestamptz | not null | 更新时间 |

medical_history JSONB 键契约：

- blood_type：A/B/AB/O/UNKNOWN
- chronic_diseases：array<string>
- allergy_notes：string（<=500）

约束：

- check(lost_status in ('NORMAL','MISSING'))
- check((fence_enabled=false and fence_center is null and fence_radius_m is null) or (fence_enabled=true and fence_center is not null and fence_radius_m between 50 and 5000))

lost_status 事件驱动转换规则（参见 LLD §6.1.1）：

| 当前状态 | 触发事件 | 下一状态 | 守卫 |
| :--- | :--- | :--- | :--- |
| NORMAL | task.created | MISSING | 事件版本更新且无主键冲突 |
| MISSING | task.resolved | NORMAL | 匹配同患者有效任务 |
| MISSING | task.false_alarm | NORMAL | 匹配同患者有效任务 |
| 任意 | 旧版本事件 | 原状态 | event_time <= lost_status_event_time 时丢弃 |

- profile-service 消费 task 域事件执行条件更新，必须使用 `lost_status_event_time` 防乱序覆盖。
- 应用层不提供独立写入 `lost_status` 的 API，该字段完全由事件驱动。

索引：

- ux_patient_profile_no(profile_no)
- ux_patient_short_code(short_code)
- idx_patient_lost_status(lost_status)
- gist_patient_fence_center(fence_center)
- gin_patient_medical_history(medical_history)

### 4.2.1 short_code 发号物理基线

序列真源：

- 必须使用数据库序列 `patient_short_code_seq` 作为唯一发号真源，禁止随机短码直写。
- 序列值仅用于内部发号，不对外暴露。

发号与灾备约束：

1. 服务节点按固定步长预取号段，本地消费；节点故障后剩余号段直接废弃。
2. 主备切换前后必须执行序列水位校准，确保新主序列值不小于最近已分配号段上界。
3. 可逆混淆必须是 `[0, 36^6-1]` 上双射，禁止截断、取模、大小写二次归一等破坏唯一性的变换。
4. 短码编码碰撞由 `ux_patient_short_code` 唯一约束兜底，冲突时自动重试下一个序列值，单请求最多重试 3 次。
5. 当 1 分钟窗口冲突率 >0.01% 或出现连续重试耗尽时，发号流程进入熔断并阻断建档写入，等待人工处置。

### 4.3 sys_user_patient（用户-患者关系）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | bigint | PK | 主键 |
| user_id | bigint | not null | 用户 |
| patient_id | bigint | not null | 患者 |
| relation_role | varchar(32) | not null | PRIMARY_GUARDIAN/GUARDIAN |
| relation_status | varchar(20) | not null | PENDING/ACTIVE/REVOKED |
| transfer_state | varchar(32) | not null default 'NONE' | NONE/PENDING_CONFIRM/ACCEPTED/REJECTED/CANCELLED/EXPIRED |
| transfer_request_id | varchar(64) | null | 转移请求号 |
| transfer_target_user_id | bigint | null | 受方 |
| transfer_requested_by | bigint | null | 发起人 |
| transfer_requested_at | timestamptz | null | 发起时间 |
| transfer_reason | varchar(256) | null | 发起原因 |
| transfer_cancelled_by | bigint | null | 撤销人 |
| transfer_cancelled_at | timestamptz | null | 撤销时间 |
| transfer_cancel_reason | varchar(256) | null | 撤销原因 |
| transfer_expire_at | timestamptz | null | 过期时间 |
| transfer_confirmed_at | timestamptz | null | 确认接受时间 |
| transfer_rejected_at | timestamptz | null | 拒绝时间 |
| transfer_reject_reason | varchar(256) | null | 拒绝原因 |
| created_at | timestamptz | not null | 创建时间 |
| updated_at | timestamptz | not null | 更新时间 |

约束：

- check(relation_status in ('PENDING','ACTIVE','REVOKED'))
- check(transfer_state in ('NONE','PENDING_CONFIRM','ACCEPTED','REJECTED','CANCELLED','EXPIRED'))
- check((transfer_state='ACCEPTED' and transfer_confirmed_at is not null) or (transfer_state<>'ACCEPTED' and transfer_confirmed_at is null))
- check((transfer_state='REJECTED' and transfer_rejected_at is not null and transfer_reject_reason is not null) or (transfer_state<>'REJECTED' and transfer_rejected_at is null and transfer_reject_reason is null))
- check((transfer_state='CANCELLED' and transfer_cancelled_by is not null and transfer_cancelled_at is not null and transfer_cancel_reason is not null) or (transfer_state<>'CANCELLED' and transfer_cancelled_by is null and transfer_cancelled_at is null and transfer_cancel_reason is null))
- uq_transfer_pending(patient_id) where transfer_state='PENDING_CONFIRM'
- uq_transfer_request_id(transfer_request_id) where transfer_request_id is not null
- uq_primary_guardian_per_patient(patient_id) where relation_role='PRIMARY_GUARDIAN' and relation_status='ACTIVE'

索引：

- idx_user_patient_user(user_id, relation_status)
- idx_user_patient_patient(patient_id, relation_status)
- idx_user_patient_transfer_request(transfer_request_id)

### 4.3.1 guardian_invitation（监护邀请）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | bigint | PK | 主键 |
| invite_id | varchar(64) | not null unique | 邀请号 |
| patient_id | bigint | not null | 邀请归属患者 |
| inviter_user_id | bigint | not null | 邀请发起人 |
| invitee_user_id | bigint | not null | 被邀请用户 |
| relation_role | varchar(32) | not null | PRIMARY_GUARDIAN/GUARDIAN |
| status | varchar(20) | not null default 'PENDING' | PENDING/ACCEPTED/REJECTED/EXPIRED/REVOKED |
| reason | varchar(256) | null | 发起原因 |
| reject_reason | varchar(256) | null | 拒绝/撤销原因 |
| expire_at | timestamptz | not null | 过期时间 |
| accepted_at | timestamptz | null | 接受时间 |
| rejected_at | timestamptz | null | 拒绝时间 |
| revoked_at | timestamptz | null | 撤销时间 |
| created_at | timestamptz | not null | 创建时间 |
| updated_at | timestamptz | not null | 更新时间 |

约束：

- uq_guardian_invite_id(invite_id)
- uq_guardian_invite_pending(patient_id, invitee_user_id) where status='PENDING'
- check(status in ('PENDING','ACCEPTED','REJECTED','EXPIRED','REVOKED'))
- check((status='ACCEPTED' and accepted_at is not null) or (status<>'ACCEPTED' and accepted_at is null))
- check((status='REJECTED' and rejected_at is not null and reject_reason is not null) or (status<>'REJECTED' and rejected_at is null))
- check((status='REVOKED' and revoked_at is not null and reject_reason is not null) or (status<>'REVOKED' and revoked_at is null))
- check((reject_reason is null) or status in ('REJECTED','REVOKED'))

索引：

- idx_guardian_invite_patient_status(patient_id, status, created_at desc)
- idx_guardian_invite_invitee_status(invitee_user_id, status, created_at desc)
- idx_guardian_invite_expire(status, expire_at)

### 4.4 rescue_task（寻回任务）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | bigint | PK | 主键 |
| task_no | varchar(32) | unique | 任务编号 |
| patient_id | bigint | not null | 患者 |
| status | varchar(20) | not null | ACTIVE/RESOLVED/FALSE_ALARM |
| source | varchar(32) | not null | APP/MINI_PROGRAM/ADMIN_PORTAL |
| remark | varchar(500) | null | 发起备注 |
| ai_analysis_summary | text | null | AI 分析摘要 |
| poster_url | varchar(1024) | null | 海报地址 |
| close_reason | varchar(256) | null | 关闭原因 |
| event_version | bigint | not null default 0 | 事件版本 |
| created_by | bigint | not null | 发起人 |
| created_at | timestamptz | not null | 创建时间 |
| closed_at | timestamptz | null | 任务关闭时间（终态写入） |
| updated_at | timestamptz | not null | 更新时间 |

约束：

- check(status in ('ACTIVE','RESOLVED','FALSE_ALARM'))
- check((status='ACTIVE' and closed_at is null) or (status in ('RESOLVED','FALSE_ALARM') and closed_at is not null))
- uq_task_active_per_patient(patient_id) where status='ACTIVE'
- 状态更新必须条件更新（where id=? and status=?）

索引：

- idx_rescue_task_patient(patient_id, created_at desc)
- idx_rescue_task_status(status, created_at desc)

### 4.5 clue_record（线索记录）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | bigint | PK | 主键 |
| clue_no | varchar(32) | unique | 线索编号 |
| patient_id | bigint | not null | 患者 |
| task_id | bigint | null | 关联任务 |
| tag_code | varchar(100) | not null | 标签码 |
| source_type | varchar(20) | not null | SCAN/MANUAL |
| risk_score | numeric(5,4) | null | 风险分（0-1） |
| location | geometry(Point,4326) | not null | 坐标（WGS84） |
| coord_system | varchar(10) | not null default 'WGS84' | 标准化坐标系（恒为 WGS84；保留此列用于 DDL 自文档化与 CHECK 防御性约束，网关已在入库前完成 GCJ-02/BD-09 → WGS84 转换） |
| description | text | null | 现场描述 |
| photo_url | varchar(1024) | null | 照片地址 |
| is_valid | boolean | not null | 有效标记 |
| suspect_flag | boolean | not null | 可疑标记 |
| suspect_reason | varchar(256) | null | 可疑原因摘要 |
| review_status | varchar(20) | null default null | 仅可疑线索使用：PENDING/OVERRIDDEN/REJECTED；非可疑为 NULL |
| assignee_user_id | bigint | null | 复核指派人 |
| assigned_at | timestamptz | null | 指派时间 |
| reviewed_at | timestamptz | null | 完成复核时间 |
| override | boolean | not null default false | 管理员强制回流标记 |
| override_by | bigint | null | 强制回流操作人 |
| override_reason | varchar(256) | null | 强制回流原因 |
| reject_reason | varchar(256) | null | 驳回原因 |
| rejected_by | bigint | null | 驳回人 |
| created_at | timestamptz | not null | 创建时间 |
| updated_at | timestamptz | not null | 更新时间 |

约束：

- check(review_status is null or review_status in ('PENDING','OVERRIDDEN','REJECTED'))
- check(risk_score is null or (risk_score >= 0 and risk_score <= 1))
- check(coord_system='WGS84')
- check((assignee_user_id is null and assigned_at is null) or (assignee_user_id is not null and assigned_at is not null))
- check((suspect_flag=false and review_status is null) or (suspect_flag=true and review_status in ('PENDING','OVERRIDDEN','REJECTED')))
- check(((review_status in ('OVERRIDDEN','REJECTED')) and reviewed_at is not null) or ((review_status is null or review_status='PENDING') and reviewed_at is null))
- check((review_status='OVERRIDDEN' and override=true and override_reason is not null) or (review_status<>'OVERRIDDEN' and override=false and override_reason is null))
- check((review_status='REJECTED' and rejected_by is not null and reject_reason is not null) or (review_status<>'REJECTED' and rejected_by is null and reject_reason is null))

坐标入库规则：

1. `location` 必须为网关标准化后的 `WGS84` 坐标点。
2. 高德（AMap）来源坐标按 `GCJ-02` 处理，禁止直接以 `GCJ-02` 入库。
3. 若坐标转换失败，业务层应返回 `E_CLUE_4007` 并拒绝写入 `clue_record`。

索引：

- gist_clue_location(location)
- idx_clue_patient_created(patient_id, created_at desc)
- idx_clue_task_created(task_id, created_at desc)
- idx_clue_suspected(suspect_flag, is_valid)
- idx_clue_review_pending(review_status, created_at desc) where review_status='PENDING'
- idx_clue_assignee_pending(assignee_user_id, created_at desc) where review_status='PENDING'

### 4.5A 线索补证能力（毕设精简）

当前毕设版本不启用 3.2.13 补证请求能力，不创建 `clue_evidence_request` 独立表。

落库策略：

1. 补证相关动作仅写入 `sys_log`（module=CLUE，action=REQUEST_EVIDENCE）。
2. 扩展载荷（evidence_type、reason、deadline）写入 `sys_log.detail`。
3. 后续若恢复该能力，再单独引入补证表与状态机。

### 4.6 patient_trajectory（轨迹聚合）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | bigint | PK | 主键 |
| patient_id | bigint | not null | 患者 |
| task_id | bigint | null | 任务 |
| window_start | timestamptz | not null | 窗口起 |
| window_end | timestamptz | not null | 窗口止 |
| point_count | int | not null | 点数量 |
| geometry_type | varchar(32) | not null | LINESTRING/SPARSE_POINT/EMPTY_WINDOW |
| geometry_data | geometry(Geometry,4326) | null | 轨迹几何（WGS84） |
| created_at | timestamptz | not null | 创建时间 |

约束：

- check((geometry_type='EMPTY_WINDOW' and geometry_data is null) or (geometry_type in ('LINESTRING','SPARSE_POINT') and geometry_data is not null))
- geometry_data 非空时必须满足 SRID=4326。
- uq_traj_patient_window(patient_id, window_start, window_end)

索引：

- idx_traj_patient_window(patient_id, window_start desc)
- gist_traj_geometry(geometry_data)

### 4.7 tag_asset（标签资产）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | bigint | PK | 主键 |
| tag_code | varchar(100) | unique not null | 标签码 |
| tag_type | varchar(20) | not null | QR_CODE/NFC |
| status | varchar(20) | not null | UNBOUND/ALLOCATED/BOUND/LOST/VOID |
| patient_id | bigint | null | 绑定患者 |
| apply_record_id | bigint | null | 关联工单 |
| import_batch_no | varchar(64) | null | 入库批次 |
| void_reason | varchar(256) | null | 作废原因 |
| lost_at | timestamptz | null | 挂失时间 |
| void_at | timestamptz | null | 作废时间 |
| reset_at | timestamptz | null | 管理员重置时间 |
| recovered_at | timestamptz | null | 管理员恢复时间 |
| created_at | timestamptz | not null | 创建时间 |
| updated_at | timestamptz | not null | 更新时间 |

约束：

- check(status in ('UNBOUND','ALLOCATED','BOUND','LOST','VOID'))
- check((status not in ('BOUND','LOST')) or patient_id is not null)
- check((status <> 'LOST') or lost_at is not null)
- check((status <> 'VOID') or (void_at is not null and void_reason is not null))
- check((reset_at is null) or status='UNBOUND')
- check((recovered_at is null) or (status='BOUND' and patient_id is not null))

索引：

- ux_tag_asset_code(tag_code)
- idx_tag_asset_status(status)
- idx_tag_asset_patient(patient_id, status)
- idx_tag_asset_batch(import_batch_no)

### 4.8 tag_apply_record（物资申领工单）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | bigint | PK | 主键 |
| order_no | varchar(32) | unique | 工单号 |
| patient_id | bigint | not null | 患者 |
| applicant_user_id | bigint | not null | 申请人 |
| quantity | int | not null | 数量（1-20） |
| apply_note | varchar(256) | null | 申领备注 |
| tag_code | varchar(100) | null | 发货时锁定 |
| status | varchar(20) | not null | PENDING/PROCESSING/CANCEL_PENDING/CANCELLED/SHIPPED/EXCEPTION/COMPLETED |
| delivery_address | varchar(512) | not null | 收货地址 |
| tracking_number | varchar(64) | null | 物流单号 |
| courier_name | varchar(64) | null | 物流公司 |
| resource_link | varchar(1024) | null | 资源链 |
| cancel_reason | varchar(256) | null | 取消原因 |
| approved_at | timestamptz | null | 审核通过时间 |
| reject_reason | varchar(256) | null | 驳回原因 |
| rejected_at | timestamptz | null | 驳回时间 |
| exception_desc | varchar(512) | null | 异常说明 |
| closed_at | timestamptz | null | 工单关闭时间（终态写入） |
| created_at | timestamptz | not null | 创建时间 |
| updated_at | timestamptz | not null | 更新时间 |

约束：

- check(status in ('PENDING','PROCESSING','CANCEL_PENDING','CANCELLED','SHIPPED','EXCEPTION','COMPLETED'))
- check((approved_at is null) or status in ('PROCESSING','CANCEL_PENDING','CANCELLED','SHIPPED','EXCEPTION','COMPLETED'))
- check((rejected_at is null and reject_reason is null) or (rejected_at is not null and reject_reason is not null))
- check((status in ('CANCELLED','COMPLETED') and closed_at is not null) or (status in ('PENDING','PROCESSING','CANCEL_PENDING','SHIPPED','EXCEPTION') and closed_at is null))

状态机流转（参见 LLD §6.2）：

- PENDING → PROCESSING（审核通过）。
- PROCESSING → CANCEL_PENDING（取消申请）| SHIPPED（发货）。
- CANCEL_PENDING → CANCELLED（取消审核通过）| PROCESSING（取消驳回）。
- SHIPPED → EXCEPTION（物流异常）| COMPLETED（家属签收 auto_confirm）。
- EXCEPTION → PROCESSING（补发）| CANCELLED（异常关闭）。
- 终态：CANCELLED、COMPLETED（closed_at 必须非空）。
- EXCEPTION 为**非终态**，可回流至 PROCESSING（补发）或转入 CANCELLED（异常关闭）。

补充说明：
1. `resource_token_expire_at` 与资源链 `status` 为令牌运行时属性，不作为 `tag_apply_record` 固定列。
2. 资源链状态由发码服务按令牌有效期实时计算并回传。

索引：

- idx_apply_patient_status(patient_id, status, created_at desc)
- idx_apply_applicant(applicant_user_id, created_at desc)
- idx_apply_status(status, created_at desc)

### 4.9 ai_session（AI 会话审计）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | bigint | PK | 主键 |
| session_id | varchar(64) | unique not null | 会话号 |
| user_id | bigint | not null | 用户 |
| patient_id | bigint | not null | 患者 |
| task_id | bigint | null | 任务 |
| messages | jsonb | not null | 会话消息数组 |
| request_tokens | int | not null default 0 | 请求 token |
| response_tokens | int | not null default 0 | 响应 token |
| token_usage | jsonb | not null default '{}'::jsonb | 细粒度账单键值（权威来源，多供应商统一） |
| token_used | int | not null default 0 | 派生聚合字段，持久化值 = request_tokens + response_tokens；写入时由服务端计算，禁止客户端直传 |
| model_name | varchar(64) | not null | 模型标识 |
| status | varchar(20) | not null default 'ACTIVE' | ACTIVE/ARCHIVED |
| archived_at | timestamptz | null | 归档时间 |
| version | bigint | not null default 0 | 乐观锁版本 |
| created_at | timestamptz | not null | 创建时间 |
| updated_at | timestamptz | not null | 更新时间 |

约束：

- messages 更新必须原子追加或 version CAS，禁止覆盖写。
- check(status in ('ACTIVE','ARCHIVED'))
- check((status='ACTIVE' and archived_at is null) or (status='ARCHIVED' and archived_at is not null))
- 单会话预计轮数 > 50 时，必须启用 ai_session_message 拆分存储，避免 JSONB 触发高频 TOAST 重写。

token_usage JSONB 兼容键契约（多供应商统一）：

- 必填键：prompt_tokens、completion_tokens、total_tokens、model_name、billing_source。
- 可选键：provider_request_id、model_price_tier、currency、estimated_cost。
- 百炼/千问接入时需保证以上键可稳定映射，便于审计与成本对账。

索引：

- ux_ai_session_session_id(session_id)
- idx_ai_session_user_created(user_id, created_at desc)
- idx_ai_session_patient_created(patient_id, created_at desc)

长会话扩展表（建议）：ai_session_message

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | bigint | PK | 主键 |
| session_id | varchar(64) | not null | 会话号 |
| sequence_no | int | not null | 会话内递增序号 |
| role | varchar(16) | not null | system/user/assistant/tool |
| content | text | not null | 消息正文 |
| token_usage | jsonb | null | 单条消息消耗明细（可选） |
| created_at | timestamptz | not null | 创建时间 |

约束与索引：

- fk_ai_session_msg_session(session_id -> ai_session.session_id)
- uq_ai_session_msg_seq(session_id, sequence_no)
- idx_ai_session_msg_session_seq(session_id, sequence_no)
- idx_ai_session_msg_created(created_at)

启用策略：

- 短会话（<=50轮）可仅使用 ai_session.messages。
- 长会话（>50轮）改为逐条 INSERT ai_session_message，ai_session.messages 保留摘要或最近窗口。

### 4.10 vector_store（向量记忆库）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | bigint | PK | 主键 |
| patient_id | bigint | not null | 患者隔离键 |
| source_type | varchar(32) | not null | PROFILE/MEMORY/RESCUE_CASE |
| source_id | varchar(64) | not null | 来源记录 |
| source_version | bigint | not null | 版本锚点 |
| embedding | vector(1024) | not null | 向量 |
| content | text | not null | 原文片段 |
| valid | boolean | not null default true | 是否可召回 |
| superseded_at | timestamptz | null | 被替代时间 |
| deleted_at | timestamptz | null | 逻辑删除时间 |
| expired_at | timestamptz | null | 过期时间 |
| created_at | timestamptz | not null | 创建时间 |

约束：

- 检索必须先按 patient_id 过滤，再执行 ANN。

索引与分区：

- 默认方案：单表 + HNSW(embedding) + B-Tree(patient_id, valid, created_at)。
- 大规模方案：按 patient_id 固定数量 HASH 分区（建议 64 或 128），每个分区维护独立 HNSW 索引。
- 禁止方案：按单个 patient_id 做 LIST 分区（会导致 Catalog 膨胀与规划器性能退化）。
- 检索入口必须先按 patient_id 过滤，再执行向量相似度排序，禁止跨分区全量扫描。

### 4.10.1 Filter Pushdown 基准门禁

建表冻结前必须完成以下验证：

1. 使用 `EXPLAIN (ANALYZE, BUFFERS)` 验证查询计划满足 patient_id 过滤下推，且 HASH 分区模式下可观察到分区裁剪。
2. 查询模板必须固定包含 `WHERE patient_id=:pid AND valid=true`，禁止应用层“先 ANN 后过滤”。
3. 数据规模至少覆盖 100 万与 1000 万向量两档，且包含高密度患者与普通患者双场景。
4. TopK=10 条件下，高密度患者检索 TP95 < 120ms、TP99 < 200ms；recall@10 >= 0.90。
5. 若任一门槛不满足，禁止进入 DDL 冻结与上线阶段。

### 4.10A patient_memory_note（患者记忆原始条目）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | bigint | PK | 主键 |
| note_id | varchar(64) | unique not null | 记忆条目号 |
| patient_id | bigint | not null | 关联患者 |
| created_by | bigint | not null | 创建人 |
| kind | varchar(20) | not null | HABIT/PLACE/PREFERENCE/SAFETY_CUE |
| content | text | not null | 原始记忆内容 |
| tags | jsonb | null | 语义标签数组 |
| source_event_id | varchar(64) | null | 触发向量化的事件号 |
| created_at | timestamptz | not null | 创建时间 |
| updated_at | timestamptz | not null | 更新时间 |

约束：

- check(kind in ('HABIT','PLACE','PREFERENCE','SAFETY_CUE'))
- check(tags is null or jsonb_typeof(tags)='array')

索引：

- idx_memory_note_patient_created(patient_id, created_at desc)
- idx_memory_note_patient_kind(patient_id, kind, created_at desc)
- idx_memory_note_created_by(created_by, created_at desc)

### 4.10B 任务告警落库（毕设精简）

当前毕设版本不创建 `emergency_alert` 独立表，3.1.15 告警列表直接复用 `notification_inbox`。

实现约束：

1. 告警写入时 `notification_inbox.type` 使用 `FENCE_ALERT`/`TASK_PROGRESS`/`TASK_CLOSED`。
2. 告警等级写入 `notification_inbox.level`。
3. 任务与患者关联分别写入 `related_task_id`、`related_patient_id`。

### 4.10C 物流轨迹能力（毕设精简）

当前毕设版本不展示物流轨迹，不创建 `logistics_tracking_event` 独立表。

实现约束：

1. 物流状态仅保留在 `tag_apply_record` 的 `status/tracking_number/courier_name`。
2. API 3.4.24 在毕设版本暂不开放。

### 4.10D 数据导出落库（毕设精简）

当前毕设版本不创建 `export_job` 独立表，3.8.6 导出请求与结果统一记录到 `sys_log`。

实现约束：

1. 请求写审计：module=GOVERNANCE，action=EXPORT_DATA，result=SUCCESS/FAIL。
2. 导出参数（export_type/window_start/window_end/reason）写 `sys_log.detail.export_payload`。
3. 导出结果（file_url/failed_reason）写 `sys_log.detail.export_result`。

### 4.11 notification_inbox（站内通知中心）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| notification_id | bigint | PK | 通知主键 |
| user_id | bigint | not null | 接收用户 |
| type | varchar(32) | not null | TASK_PROGRESS/FENCE_ALERT/TASK_CLOSED/SYSTEM |
| title | varchar(128) | not null | 标题 |
| content | text | not null | 内容摘要 |
| level | varchar(16) | not null | INFO/WARN/CRITICAL |
| related_task_id | bigint | null | 关联任务 |
| related_patient_id | bigint | null | 关联患者 |
| read_status | varchar(16) | not null default 'UNREAD' | UNREAD/READ |
| read_at | timestamptz | null | 已读时间 |
| trace_id | varchar(64) | not null | 链路标识 |
| created_at | timestamptz | not null | 创建时间 |
| updated_at | timestamptz | not null | 更新时间 |

约束：

- check(read_status in ('UNREAD','READ'))
- check((read_status='UNREAD' and read_at is null) or (read_status='READ' and read_at is not null))

索引：

- idx_noti_user_created(user_id, created_at desc)
- idx_noti_user_read(user_id, read_status, created_at desc)
- idx_noti_user_type(user_id, type, created_at desc)
- idx_noti_related_task(related_task_id)
- idx_noti_related_patient(related_patient_id)

### 4.12 sys_log（治理审计日志）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | bigint | PK | 主键 |
| module | varchar(64) | not null | 模块 |
| action | varchar(64) | not null | 动作 |
| action_id | varchar(64) | null | Agent 执行动作 ID（执行回执） |
| result_code | varchar(64) | null | 执行结果码（OK/业务码） |
| executed_at | timestamptz | null | 动作执行完成时间 |
| operator_user_id | bigint | null | 操作人 ID |
| operator_username | varchar(64) | not null | 操作账号快照 |
| object_id | varchar(64) | null | 对象 ID |
| result | varchar(20) | not null | SUCCESS/FAIL |
| risk_level | varchar(20) | null | LOW/MEDIUM/HIGH/CRITICAL |
| detail | jsonb | null | 扩展信息 |
| action_source | varchar(20) | not null default 'USER' | USER/AI_AGENT |
| agent_profile | varchar(64) | null | Agent 能力包标识 |
| execution_mode | varchar(20) | null | AUTO/CONFIRM_1/CONFIRM_2/CONFIRM_3/MANUAL_ONLY |
| confirm_level | varchar(20) | null | CONFIRM_1/CONFIRM_2/CONFIRM_3 |
| blocked_reason | varchar(128) | null | 门禁阻断原因 |
| request_id | varchar(64) | null | 幂等键 |
| trace_id | varchar(64) | not null | 链路标识 |
| created_at | timestamptz | not null | 创建时间 |

约束：

- check(result in ('SUCCESS','FAIL'))
- check(action_source in ('USER','AI_AGENT'))
- check(execution_mode is null or execution_mode in ('AUTO','CONFIRM_1','CONFIRM_2','CONFIRM_3','MANUAL_ONLY'))
- check(confirm_level is null or confirm_level in ('CONFIRM_1','CONFIRM_2','CONFIRM_3'))

索引：

- idx_log_module_action_time(module, action, created_at desc)
- idx_log_operator_time(operator_user_id, created_at desc)
- idx_log_trace(trace_id)
- idx_log_action_id(action_id)
- idx_log_action_source_time(action_source, created_at desc)
- idx_log_agent_profile_time(agent_profile, created_at desc)
- idx_log_created_at(created_at desc)

执行回执固化规则：
1. 当 `action_source='AI_AGENT'` 且动作执行完成时，必须落库 `action_id/result_code/executed_at`。
2. 当命中策略阻断（如 `POLICY_BLOCK`）时，`result_code` 必须落库；`executed_at` 可空。

状态轨迹 detail 键契约：
1. 当 `module='MATERIAL_ORDER'` 或 `module='TAG_ASSET'` 且 action 表示状态流转时，`detail` 必须包含 `from_status`、`to_status`。
2. `detail.remark` 可选；时间线记录时间统一取 `sys_log.created_at`。

### 4.13 sys_config（治理配置）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| config_key | varchar(128) | PK | 白名单配置键 |
| config_value | text | not null | 配置值 |
| scope | varchar(32) | not null default 'public' | public/ops/security/ai_policy |
| updated_by | bigint | not null | 最近更新操作人 |
| updated_reason | varchar(256) | not null | 最近更新原因 |
| created_at | timestamptz | not null | 创建时间 |
| updated_at | timestamptz | not null | 更新时间 |

索引：

- idx_sys_config_scope(scope)

AI Agent 策略键约定：

| config_key | 说明 |
| :--- | :--- |
| agent.capability.rescue.enabled | RescueCommander 能力包开关 |
| agent.capability.clue.enabled | ClueInvestigator 能力包开关 |
| agent.capability.guardian.enabled | GuardianCoordinator 能力包开关 |
| agent.capability.material.enabled | MaterialOperator 能力包开关 |
| agent.capability.ai_case.enabled | AICaseCopilot 能力包开关 |
| agent.capability.governance.enabled | GovernanceSentinel 能力包开关 |
| agent.capability.outbox_reliability.enabled | OutboxReliabilityAgent 能力包开关 |
| agent.execution.max_level | 允许执行上限（A0/A1/A2/A3） |
| agent.confirmation.policy | 确认级别策略映射 |
| agent.manual_only.actions | 人工专属接口白名单 |

### 4.14 sys_outbox_log（Outbox 调度）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| event_id | varchar(64) | not null，PK(联合) | 事件 ID |
| topic | varchar(128) | not null | 目标 Topic |
| aggregate_id | varchar(64) | not null | 聚合 ID |
| partition_key | varchar(64) | not null | 分区键 |
| payload | jsonb | not null | 事件载荷 |
| request_id | varchar(64) | not null | 幂等键 |
| trace_id | varchar(64) | not null | 链路标识 |
| phase | varchar(20) | not null | PENDING/DISPATCHING/SENT/RETRY/DEAD |
| retry_count | int | not null default 0 | 重试次数 |
| next_retry_at | timestamptz | null | 下次重试 |
| lease_owner | varchar(64) | null | 抢占实例 |
| lease_until | timestamptz | null | 租约到期 |
| sent_at | timestamptz | null | 成功时间 |
| last_error | varchar(512) | null | 最近错误 |
| last_intervention_by | bigint | null | 最近人工干预操作人 |
| last_intervention_at | timestamptz | null | 最近人工干预时间 |
| replay_reason | varchar(256) | null | 最近一次重放原因 |
| replay_token | varchar(64) | null | 最近一次重放幂等键 |
| replayed_at | timestamptz | null | 最近一次重放时间 |
| created_at | timestamptz | not null，PK(联合)，分区键 | 创建时间 |
| updated_at | timestamptz | not null | 更新时间 |

主键与唯一约束（分区表）：

- pk_outbox(event_id, created_at)
- 分区表上的唯一约束必须包含分区键 created_at。
- check(phase in ('PENDING','DISPATCHING','SENT','RETRY','DEAD'))

索引：

- idx_outbox_phase_retry(phase, next_retry_at, created_at)
- idx_outbox_partition_phase(partition_key, phase, created_at)
- idx_outbox_event_id_created(event_id, created_at desc)
- idx_outbox_dead_queue(phase, updated_at desc) where phase='DEAD'

分区与清理策略：

- 按 created_at 做 RANGE 分区（建议按周或按天）。
- 保留最近 N 个活跃分区用于在线读写。
- 历史清理优先采用 DROP PARTITION（或 detach 后异步归档），避免大批量 DELETE 造成死元组膨胀。
- 以 event_id 定位 Outbox 记录时，业务查询必须同时带 created_at（或分区桶）以触发分区裁剪。

### 4.15 consumed_event_log（消费幂等日志）

| 字段 | 类型 | 约束 | 说明 |
| :--- | :--- | :--- | :--- |
| id | bigint | not null，PK(联合) | 主键流水号 |
| consumer_name | varchar(64) | not null | 消费者服务名 |
| topic | varchar(128) | not null | Topic |
| event_id | varchar(64) | not null | 事件 ID |
| partition | int | not null | 分区 |
| offset | bigint | not null | 位点 |
| processed_at | timestamptz | not null，PK(联合)，分区键 | 处理时间 |
| trace_id | varchar(64) | not null | 链路标识 |

唯一约束：

- pk_consumed_event(id, processed_at)
- uq_consumer_event(consumer_name, topic, event_id, processed_at)
- 分区表上的唯一约束必须包含分区键 processed_at。

索引：

- idx_consumed_consumer_event(consumer_name, topic, event_id, processed_at desc)
- idx_consumed_processed_at(processed_at)

分区与清理策略：

- 按 processed_at 做 RANGE 分区（建议按周或按天）。
- 幂等保留窗口到期后，按分区直接 DROP，避免频繁 VACUUM 抖动。

## 5. 外键关系

建议外键策略：默认 ON DELETE NO ACTION / ON UPDATE NO ACTION。

| 外键字段 | 引用 | 说明 |
| :--- | :--- | :--- |
| sys_user_patient.user_id | sys_user.id | 用户-患者关系 |
| sys_user_patient.patient_id | patient_profile.id | 用户-患者关系 |
| sys_user_patient.transfer_target_user_id | sys_user.id | 主监护转移受方 |
| guardian_invitation.patient_id | patient_profile.id | 监护邀请归属患者 |
| guardian_invitation.inviter_user_id | sys_user.id | 邀请发起人 |
| guardian_invitation.invitee_user_id | sys_user.id | 被邀请用户 |
| rescue_task.patient_id | patient_profile.id | 任务归属患者 |
| rescue_task.created_by | sys_user.id | 任务发起人 |
| clue_record.patient_id | patient_profile.id | 线索归属患者 |
| clue_record.task_id | rescue_task.id | 线索关联任务 |
| clue_record.assignee_user_id | sys_user.id | 线索复核指派人 |
| patient_trajectory.patient_id | patient_profile.id | 轨迹归属患者 |
| patient_trajectory.task_id | rescue_task.id | 轨迹关联任务 |
| tag_asset.patient_id | patient_profile.id | 标签绑定患者 |
| tag_asset.apply_record_id | tag_apply_record.id | 标签关联工单 |
| tag_apply_record.patient_id | patient_profile.id | 工单归属患者 |
| tag_apply_record.applicant_user_id | sys_user.id | 工单申请人 |
| ai_session.user_id | sys_user.id | 会话发起用户 |
| ai_session.patient_id | patient_profile.id | 会话关联患者 |
| ai_session.task_id | rescue_task.id | 会话关联任务 |
| ai_session_message.session_id（可选） | ai_session.session_id | 长会话明细归属 |
| patient_memory_note.patient_id | patient_profile.id | 记忆条目归属患者 |
| patient_memory_note.created_by | sys_user.id | 记忆条目创建人 |
| vector_store.patient_id | patient_profile.id | 向量归属患者 |
| notification_inbox.user_id | sys_user.id | 通知接收用户 |
| notification_inbox.related_task_id | rescue_task.id | 关联任务 |
| notification_inbox.related_patient_id | patient_profile.id | 关联患者 |
| sys_log.operator_user_id | sys_user.id | 操作人（可空） |
| sys_config.updated_by | sys_user.id | 配置更新人 |
| sys_outbox_log.last_intervention_by | sys_user.id | 最近 DEAD 干预操作人（可空） |

## 6. 枚举值定义

| 枚举类别 | 值 |
| :--- | :--- |
| user_role | FAMILY/ADMIN/SUPERADMIN |
| user_status | NORMAL/BANNED |
| relation_role | PRIMARY_GUARDIAN/GUARDIAN |
| relation_status | PENDING/ACTIVE/REVOKED |
| invitation_status | PENDING/ACCEPTED/REJECTED/EXPIRED/REVOKED |
| transfer_state | NONE/PENDING_CONFIRM/ACCEPTED/REJECTED/CANCELLED/EXPIRED |
| lost_status | NORMAL/MISSING |
| task_status | ACTIVE/RESOLVED/FALSE_ALARM |
| clue_source_type | SCAN/MANUAL |
| clue_review_status | PENDING/OVERRIDDEN/REJECTED |
| tag_type | QR_CODE/NFC |
| tag_asset_status | UNBOUND/ALLOCATED/BOUND/LOST/VOID |
| apply_status | PENDING/PROCESSING/CANCEL_PENDING/CANCELLED/SHIPPED/EXCEPTION/COMPLETED |
| ai_session_status | ACTIVE/ARCHIVED |
| memory_note_kind | HABIT/PLACE/PREFERENCE/SAFETY_CUE |
| notification_type | TASK_PROGRESS/FENCE_ALERT/TASK_CLOSED/SYSTEM |
| notification_level | INFO/WARN/CRITICAL |
| notification_read_status | UNREAD/READ |
| log_result | SUCCESS/FAIL |
| config_scope | public/ops/security/ai_policy |
| action_source | USER/AI_AGENT |
| execution_mode | AUTO/CONFIRM_1/CONFIRM_2/CONFIRM_3/MANUAL_ONLY |
| confirm_level | CONFIRM_1/CONFIRM_2/CONFIRM_3 |
| outbox_phase | PENDING/DISPATCHING/SENT/RETRY/DEAD |

## 7. 与 API 的关键映射（防断层）

| API 章节 | 关键入参/出参 | 落库表 | 落库字段 |
| :--- | :--- | :--- | :--- |
| 3.1 任务快照/列表 | reported_by/start_time/end_time | rescue_task | created_by/created_at/closed_at |
| 3.1.15 告警列表 | alert_id/level/type/title/created_at | notification_inbox | type/level/title/content/related_task_id/created_at |
| 3.1.16 notify/retry | channels（仅 IN_APP/PUSH） | notification_inbox / sys_log | type、level、trace_id、审计记录 |
| 3.2.8/3.2.9 线索详情 | source_type/suspect_reason/review_status | clue_record | source_type/suspect_reason/review_status |
| 3.2.6/3.2.7 线索复核处置 | review_status/reviewed_at/override/override_reason/reject_reason | clue_record | review_status/reviewed_at/override/override_reason/rejected_by/reject_reason |
| 3.2.10/3.2.12 线索复核队列与分配 | review_status/assignee_user_id/assigned_at/risk_score | clue_record | review_status/assignee_user_id/assigned_at/risk_score |
| 3.2.13 发起补证请求 | - | - | 毕设版本暂不开放 |
| 3.3.1/3.3.12 监护邀请创建与查询 | invite_id/status/expire_at | guardian_invitation | invite_id/status/expire_at/invitee_user_id |
| 3.3.2 监护邀请确认 | status/reject_reason | guardian_invitation | status/accepted_at/rejected_at/reject_reason |
| 3.3.3 主监护转移发起 | transfer_request_id/transfer_state/requested_at/expire_at | sys_user_patient | transfer_request_id/transfer_state/transfer_requested_at/transfer_expire_at |
| 3.3.4 主监护转移确认 | transfer_state/confirmed_at/reject_reason | sys_user_patient | transfer_state/transfer_confirmed_at/transfer_rejected_at/transfer_reject_reason |
| 3.3.5 主监护转移撤销 | transfer_state/cancel_reason/cancelled_at | sys_user_patient | transfer_state/transfer_cancelled_by/transfer_cancel_reason/transfer_cancelled_at |
| 3.3.6 移除家庭成员 | relation_status/removed_at | sys_user_patient | relation_status/updated_at（removed_at 由 updated_at 回传） |
| 3.3.8/3.3.9 患者建档与更新 | blood_type/chronic_diseases/allergy_notes | patient_profile | medical_history(JSONB) |
| 3.3.8/3.3.9 患者建档与更新 | avatar_url | patient_profile | photo_url（not null，禁止清空） |
| 3.3.10 围栏配置 | fence_enabled/fence_center/fence_radius_m | patient_profile | fence_enabled/fence_center/fence_radius_m |
| 3.4.1 创建工单 | quantity/apply_note/delivery_address | tag_apply_record | quantity/apply_note/delivery_address |
| 3.4.6 标签作废 | status/void_reason/void_at | tag_asset | status/void_reason/void_at |
| 3.4.7 工单审核通过 | status/approved_at | tag_apply_record | status/approved_at |
| 3.4.8 取消审核通过 | status/approved_at | tag_apply_record | status/approved_at/closed_at |
| 3.4.9 取消驳回 | status/reject_reason/rejected_at | tag_apply_record | status/reject_reason/rejected_at |
| 3.4.10 工单发货 | tracking_number/courier_name | tag_apply_record | tracking_number/courier_name |
| 3.4.13 异常关闭 | status/closed_at | tag_apply_record | status/closed_at |
| 3.4.14 家属签收 | status/confirmed_at | tag_apply_record | status/closed_at |
| 3.4.15 批量入库标签 | tag_code/tag_type/batch_no | tag_asset | tag_code/tag_type/import_batch_no |
| 3.4.16 标签重置 | status/reset_at | tag_asset | status/reset_at |
| 3.4.17 标签恢复 | status/patient_id/recovered_at | tag_asset | status/patient_id/recovered_at |
| 3.4.21 工单时间线 | from_status/to_status/created_at | sys_log | detail.from_status/detail.to_status/created_at |
| 3.4.24 物流轨迹查询 | - | - | 毕设版本暂不开放 |
| 3.4.25 资源链读取 | resource_link/resource_token_expire_at/status | tag_apply_record + token 服务 | resource_link（落库）/token.expire_at（运行时）/status（运行时计算，不持久化） |
| 3.4.29 标签历史 | from_status/to_status/created_at | sys_log | detail.from_status/detail.to_status/created_at |
| 3.5.5/3.5.6 记忆写入与分页 | note_id/kind/content/tags/created_at | patient_memory_note | note_id/kind/content/tags/created_at |
| 3.5.3 AI 流式过程态 | stream_status | - | 过程态字段，仅传输层可观测，不直接落库 |
| 3.5.11 会话归档 | status/archived_at | ai_session | status/archived_at |
| 3.6.6 患者标签列表 | status/tag_status/lost_at | tag_asset | status/lost_at |
| 3.8.4 审计日志查询 | module/action/user_id/trace_id/cursor | sys_log | module/action/operator_user_id/trace_id/created_at |
| 3.8.6 数据导出 | export_type/window_start/window_end/reason/file_url | sys_log | detail.export_payload/detail.export_result |
| 3.8.8 修改配置 | config_key/config_value/reason | sys_config | config_key/config_value/updated_reason |
| 1.11 Agent 受控执行契约 | X-Action-Source/X-Agent-Profile/X-Execution-Mode/X-Confirm-Level | sys_log | action_source/agent_profile/execution_mode/confirm_level |
| 3.5.3 AI done 执行回执 | action_id/result_code/executed_at | sys_log | action_id/result_code/executed_at |
| 1.11 Agent 策略阻断 | blocked_reason | sys_log | blocked_reason |
| 1.11 Agent 能力包开关 | capability/policy 配置键 | sys_config | config_key/config_value/scope=ai_policy |
| 3.8.11 读取配置 | scope | sys_config | scope |
| 3.8.12 查询 DEAD 队列 | topic/partition_key/cursor/page_size | sys_outbox_log | phase='DEAD'、last_error、last_intervention_at、updated_at |
| 3.8.13 DEAD 重放 | event_id+created_at/replay_reason/next_retry_at/replayed_at | sys_outbox_log / sys_log | phase( DEAD->RETRY )、replay_reason、replay_token、replayed_at、last_intervention_* |
| 3.9.1 通知列表 | 通知分页与筛选 | notification_inbox | user_id/type/read_status/created_at |
| 3.9.2 单条已读 | notification_id | notification_inbox | read_status/read_at |
| 3.9.3 全部已读 | before_time | notification_inbox | 批量更新 read_status/read_at |
| 5.2 事件总线契约 | topic + trace + request | sys_outbox_log | event_id+created_at 复合主键路由，topic/trace_id/request_id/payload |

## 8. 分区、归档与运维策略

1. sys_log 建议按月分区；超保留期分区归档后删除。
2. sys_outbox_log 必须按 created_at RANGE 分区；主键采用 (event_id, created_at)，并要求查询携带 created_at 路由到目标分区。
3. consumed_event_log 必须按 processed_at RANGE 分区；主键采用 (id, processed_at)，幂等唯一键采用 (consumer_name, topic, event_id, processed_at)。
4. vector_store 默认单表；超大规模采用固定数量 HASH 分区，禁止 patient_id LIST 分区。
5. ai_session 长会话启用 ai_session_message 逐条写入，避免 JSONB 大对象 TOAST 写放大。
6. notification_inbox 除 user_id 维度外，必须维护 related_task_id、related_patient_id 反向索引。
7. 全部高危写操作必须记录 trace_id + request_id 到审计证据。
8. Agent 触发写操作必须记录 action_source/agent_profile/execution_mode/confirm_level/blocked_reason。
9. short_code 发号依赖的 `patient_short_code_seq` 必须纳入主备切换 Runbook，切换后先做序列水位校准再恢复建档流量。
10. `sys_outbox_log` 的 DEAD 干预必须保留 replay_reason/replay_token/last_intervention_*，用于复盘与追责。

## 9. 结论

本版数据库设计已对齐 SRS/SADD/LLD 与当前 API 规范，核心特点：

- 架构硬约束不被突破（尤其 HC-06 通知基线）。
- API 关键字段均有明确落库映射。
- Outbox 与消费幂等机制闭环可落地。
- 通知中心、配置中心、医疗扩展与记忆条目形成完整存储支撑（毕设精简方案）。

该文档可直接作为后续 DDL、迁移脚本与接口联调的数据库基线。