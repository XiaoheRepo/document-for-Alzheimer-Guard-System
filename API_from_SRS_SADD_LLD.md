# 基于AI Agent的阿尔兹海默症患者协同寻回系统
## 接口文档（API Specification）

## 0. 文档信息

| 项目 | 内容 |
| :--- | :--- |
| 文档名称 | 接口文档（API Specification） |
| 版本 | V1.10 |
| 日期 | 2026-04-06 |
| 输入基线 | SRS_simplify.md（V1.0）、SADD_from_SRS_simplify.md（V1.0-R3）、LLD_from_SRS_SADD.md（V1.1） |
| 适用对象 | 后端研发、前端研发、测试、联调、答辩演示 |

说明：
1. 本文档为联调契约，接口字段、错误码、事件语义以本文档为准。
2. 演示版本采用单节点部署，不影响接口契约有效性。
3. 本次增补接口仅落在 SRS/SADD 已有的治理域与 AI 域，不新增其他业务领域。
4. 若与历史 `SRS.md` 描述存在冲突，以 `SRS_simplify.md` 为上位需求基线。

## 1. 全局接口规范

### 1.1 Base URL

| 场景 | 路径前缀 |
| :--- | :--- |
| 业务 API | /api/v1 |
| 匿名扫码入口 | /r/{resource_token} |
| 匿名短码页面 | /p/{short_code}/... |

ID 传输强制约定：
1. 文档中的 int64 为领域逻辑类型，不代表 JSON 数值类型。
2. 所有 ID 字段（task_id、patient_id、clue_id、user_id、invite_id、transfer_request_id、order_id 等）在 JSON 请求体与响应体中必须使用 string 传输。
3. 路径参数与查询参数中的 ID 也按十进制字符串处理，前端禁止按 JavaScript Number 参与精度敏感计算。
4. ID 传输值必须是纯数字字符串，不满足时返回 E_REQ_4005。

### 1.2 通用 Header

| Header | 是否必填 | 规则 |
| :--- | :---: | :--- |
| Authorization | 受保护接口必填 | Bearer JWT；匿名接口可不带 |
| X-Anonymous-Token | 匿名接口（非浏览器）可选 | APP/MINI_PROGRAM 等非浏览器端可用；与 entry_token 等效 |
| X-Request-Id | 写接口必填 | 长度 16-64，仅字母数字与- |
| X-Trace-Id | 全链路必填 | 长度 16-64，禁止空值，响应必须回写 |
| X-Action-Source | Agent 执行链路条件必填 | USER / AI_AGENT；默认 USER |
| X-Agent-Profile | Agent 执行链路条件必填 | RescueCommander / ClueInvestigator / GuardianCoordinator / MaterialOperator / AICaseCopilot / GovernanceSentinel / OutboxReliabilityAgent（见 9.2） |
| X-Execution-Mode | Agent 执行链路条件必填 | AUTO / CONFIRM_1 / CONFIRM_2 / CONFIRM_3 / MANUAL_ONLY |
| X-Confirm-Level | Agent 执行链路条件必填 | CONFIRM_1 / CONFIRM_2 / CONFIRM_3；与策略门禁匹配 |
| Content-Type | JSON 接口必填 | application/json |

### 1.3 网关保留头与防伪造

| 项 | 规则 |
| :--- | :--- |
| X-User-Id、X-User-Role | 仅网关注入；客户端同名头必须被清洗或拒绝 |
| 拒绝码 | E_REQ_4003 |
| 执行顺序 | 先清洗保留头，再做令牌解析，再注入内部头 |

### 1.4 通用响应结构

成功响应：

```json
{
  "code": "OK",
  "message": "success",
  "trace_id": "trc_20260405_001",
  "data": {}
}
```

失败响应：

```json
{
  "code": "E_TASK_4091",
  "message": "active task already exists",
  "trace_id": "trc_20260405_001",
  "data": null
}
```

### 1.5 分页响应结构（Offset 与 Cursor）

分页策略：
1. 普通查询接口可采用 Offset 模式（page_no/page_size）。
2. 追加写入型流水表（审计日志、通知流、事件流）必须采用 Cursor 优先，避免深度分页性能退化。
3. Cursor 为服务端生成的透明游标，客户端不得依赖其内部结构。
4. 事件版本锚点场景允许 `after_version` / `since_version` 作为游标变体；服务端必须等价转换为 Cursor 语义返回 `next_cursor`。

Offset 模式：

```json
{
    "code": "OK",
    "message": "success",
    "trace_id": "trc_xxx",
    "data": {
        "items": [
            {
                "id": "1001"
            }
        ],
        "page_no": 1,
        "page_size": 20,
        "total": 0,
        "has_next": false
    }
}
```

Cursor 模式：

```json
{
    "code": "OK",
    "message": "success",
    "trace_id": "trc_xxx",
    "data": {
        "items": [
            {
                "id": "1001"
            }
        ],
        "page_size": 20,
        "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNi0wNC0wNlQxMDowMDowMFoiLCJpZCI6IjgwMDEifQ==",
        "has_next": false
    }
}
```

### 1.6 幂等与时间窗

1. 所有写接口都必须支持 X-Request-Id 幂等。
2. request_time 与服务端偏差必须 <= 300s，超限返回 E_REQ_4221。
3. 同一个 request_id 重复提交返回首次处理结果，不重复副作用。

### 1.7 Trace-Id 生成与传播

1. 客户端必须传入 X-Trace-Id，缺失或空值请求直接拒绝。
2. 若客户端传入 X-Trace-Id 但格式非法，返回 E_REQ_4002。
3. 响应头必须回写同一 Trace-Id；服务端日志与事件消息必须带同一 trace_id。

### 1.8 限流响应头契约

1. 网关触发限流时必须返回 HTTP 429，并返回业务码（如 E_GOV_4291、E_AI_4292/E_AI_4293）。
2. HTTP 429 响应必须附带 Retry-After 与 X-RateLimit-Remaining，供客户端执行退避重试。
3. 推荐同时返回 X-RateLimit-Limit 与 X-RateLimit-Reset，便于客户端做配额可视化。

| Header | 是否必填 | 规则 | 示例 |
| :--- | :---: | :--- | :--- |
| Retry-After | 是 | 整数秒，>=1 | 30 |
| X-RateLimit-Remaining | 是 | 当前窗口剩余额度 | 0 |
| X-RateLimit-Limit | 否 | 当前窗口总额度 | 60 |
| X-RateLimit-Reset | 否 | 窗口重置时间（Unix 时间戳秒） | 1712400000 |

### 1.9 空间坐标系统一约定（高德接入）

1. 线索写接口入参 `coord_system` 允许枚举：`WGS84` / `GCJ-02` / `BD-09`。
2. 接入高德地图（AMap）时，客户端必须上传高德原始坐标并声明 `coord_system=GCJ-02`，禁止在客户端二次转换后再标记为 GCJ-02。
3. 网关必须在入库前完成坐标标准化：`GCJ-02` / `BD-09` -> `WGS84(EPSG:4326)`；转换失败返回 `E_CLUE_4007`。
4. 数据库存储与空间计算统一使用 `WGS84`，围栏判定、去重聚合、轨迹计算均以标准化结果为准。
5. 所有读接口返回的 `coord_system` 固定为 `WGS84`。

### 1.10 AI 模型接入约定（百炼/千问兼容）

1. AI 供应商实现可替换，当前推荐接入阿里云百炼（Qwen 系列）；接口契约保持供应商无关。
2. 写接口不要求客户端上传模型供应商信息；服务端按治理配置与模型白名单选择 `model_name`。
3. `token_usage` 至少返回 `prompt_tokens`、`completion_tokens`、`total_tokens`，可扩展 `model_name`、`billing_source`、`provider_request_id`。
4. 模型异常场景必须保持现有错误码语义：`E_AI_4292`、`E_AI_4293` 等不得因供应商替换而改变。
5. 降级场景建议返回 `fallback_response.mode`，取值 `RULE_BASED`/`SAFE_GUARD`/`TOOL_DEGRADED`。
6. AI 对话回复必须采用流式下发（SSE 首选、WebSocket 备选）；禁止将大模型全量生成暴露为长时间阻塞的同步 HTTP 响应。

### 1.11 AI Agent 受控执行契约

1. 当 `X-Action-Source=AI_AGENT` 时，`X-Agent-Profile`、`X-Execution-Mode`、`X-Confirm-Level` 必须同时出现。
2. 高风险写接口必须执行策略门禁（Policy Guard）：
    - 校验角色权限。
    - 校验数据归属（patient/task/clue/order 维度）。
    - 校验执行模式与确认等级。
3. 支持 `dry_run=true` 预检查模式（仅策略与参数校验，不产生副作用）；校验失败返回 `E_GOV_4226`。
4. 标记为 `MANUAL_ONLY` 的接口，若 `X-Action-Source=AI_AGENT` 必须拒绝并返回 `E_GOV_4231`。
5. 所有 Agent 写操作必须写审计：`action_source`、`agent_profile`、`execution_mode`、`confirm_level`、`trace_id`、`request_id`。

### 1.12 统一字段映射表（API 语义名 vs 持久化字段）

| API 字段 | 持久化字段 | 说明 |
| :--- | :--- | :--- |
| reported_by | rescue_task.created_by | 任务发起人 |
| start_time | rescue_task.created_at | 任务开始时间 |
| end_time | rescue_task.closed_at | 任务结束时间 |
| assignee_user_id | clue_record.assignee_user_id | 复核责任人 |
| operator_user_id | sys_log.operator_user_id | 审计操作人 ID |
| operator_username | sys_log.operator_username | 审计操作人快照 |
| confirmed_at（3.3.4 ACCEPT） | sys_user_patient.transfer_confirmed_at | 主监护转移确认时间 |
| confirmed_at（3.4.14） | tag_apply_record.closed_at | 家属签收完成时间（工单进入 COMPLETED） |

## 2. 错误码字典

### 2.1 通用与网关

| 错误码 | 含义 |
| :--- | :--- |
| E_REQ_4001 | 请求幂等键格式不合法 |
| E_REQ_4002 | TraceId 不合法 |
| E_REQ_4003 | 客户端伪造内部保留 Header |
| E_REQ_4005 | 通用 ID 参数格式不合法（必须为十进制字符串） |
| E_REQ_4150 | Content-Type 非 application/json |
| E_REQ_4221 | 请求时间偏差超限 |
| E_GOV_4004 | device_fingerprint 格式非法 |
| E_GOV_4011 | 鉴权失败或 Authorization 缺失 |
| E_GOV_4012 | ws_ticket 无效、过期或已使用 |
| E_GOV_4031 | 账号已被封禁，禁止访问 |
| E_GOV_4032 | 高危操作仅限 SUPERADMIN |
| E_GOV_4030 | 角色或授权不足 |
| E_GOV_4039 | 策略门禁拒绝 Agent 自动执行 |
| E_GOV_4038 | CAPTCHA 校验失败 |
| E_GOV_4291 | 网关限流（含匿名兜底等高频接口） |
| E_GOV_4292 | 网关冷却期未结束（需等待 Retry-After） |
| E_GOV_4097 | Agent 确认等级不足 |
| E_GOV_4226 | 预检查失败（dry_run 校验未通过） |
| E_GOV_4231 | 当前接口仅允许人工执行（MANUAL_ONLY） |
| E_GOV_5002 | 审计写入失败（关键操作回滚） |
| E_GOV_4046 | Outbox DEAD 事件不存在或不可见 |
| E_GOV_4096 | Outbox 事件当前状态不允许重放（非 DEAD 或分区闸门冲突） |

### 2.2 任务域

| 错误码 | 含义 |
| :--- | :--- |
| E_TASK_4001 | source 枚举不合法 |
| E_TASK_4002 | remark 长度非法 |
| E_TASK_4003 | 客户端传入 reported_by（禁止） |
| E_TASK_4004 | close_type 非法 |
| E_TASK_4005 | FALSE_ALARM 时 reason 非法 |
| E_TASK_4030 | 任务操作无授权 |
| E_TASK_4041 | 任务不存在 |
| E_TASK_4091 | 同患者已存在 ACTIVE 任务 |
| E_TASK_4093 | 任务状态冲突（终态不可重复流转） |
| E_TASK_4222 | 误报关闭条件不满足 |

### 2.3 线索域

| 错误码 | 含义 |
| :--- | :--- |
| E_CLUE_4001 | 纬度非法 |
| E_CLUE_4002 | 经度非法 |
| E_CLUE_4003 | description 非法 |
| E_CLUE_4004 | photo_url 非白名单 |
| E_CLUE_4005 | short_code 非法 |
| E_CLUE_4006 | pin_code 非法 |
| E_CLUE_4007 | 坐标系非法或转换失败 |
| E_CLUE_4008 | override 参数非法 |
| E_CLUE_4009 | override_reason 非法 |
| E_CLUE_4010 | reject_reason 非法 |
| E_CLUE_4012 | entry_token 无效或重放 |
| E_CLUE_4041 | tag_code/标签状态不可用 |
| E_CLUE_4042 | short_code 无效或不可公开 |
| E_CLUE_4043 | clue_id 不存在或不在可复核状态 |
| E_CLUE_5011 | override 重发事件失败 |
| E_CLUE_5012 | reject 关闭复核失败 |

### 2.4 档案与监护

| 错误码 | 含义 |
| :--- | :--- |
| E_PRO_4001 | patient_name 非法 |
| E_PRO_4002 | birthday 非法 |
| E_PRO_4003 | gender 非法 |
| E_PRO_4004 | chronic_diseases 非法 |
| E_PRO_4005 | 围栏参数非法 |
| E_PRO_4006 | relation_role 非法 |
| E_PRO_4007 | 邀请 reason 非法 |
| E_PRO_4008 | invitation action 非法 |
| E_PRO_4009 | transfer reason 非法 |
| E_PRO_4010 | transfer expire_in_seconds 非法 |
| E_PRO_4011 | transfer confirm action 非法 |
| E_PRO_4012 | transfer cancel_reason 非法 |
| E_PRO_4013 | REJECT 时 reject_reason 非法 |
| E_PRO_4014 | avatar_url 缺失、非法或尝试清空 |
| E_PRO_4030 | 患者授权不足 |
| E_PRO_4031 | 标签高危纠错权限不足（仅 ADMIN/SUPERADMIN） |
| E_PRO_4032 | 无主监护管理权限 |
| E_PRO_4033 | 非目标受方确认 |
| E_PRO_4034 | 非原发起方撤销 |
| E_PRO_4041 | patient_id 不存在 |
| E_PRO_4042 | invitee_user_id 不存在 |
| E_PRO_4043 | invite_id 不存在或状态非法 |
| E_PRO_4044 | target_user_id 不存在或非 ACTIVE 成员 |
| E_PRO_4045 | transfer_request_id 不存在 |
| E_PRO_4091 | 患者档案冲突（重复建档） |
| E_PRO_4094 | 重复邀请或已激活成员 |
| E_PRO_4095 | 同患者存在其他 PENDING_CONFIRM 转移 |
| E_PRO_4096 | 邀请状态冲突 |
| E_PRO_4097 | 转移请求状态冲突 |
| E_PRO_4098 | 转移请求不可撤销 |
| E_PRO_4099 | 受方 relation_status 非 ACTIVE |
| E_PRO_4092 | 标签状态机流转不合法 |
| E_PRO_4093 | 标签当前为终态或受限态，禁止普通流转 |
| E_PRO_4221 | 围栏开启时缺失中心点或半径 |

### 2.5 物资域

| 错误码 | 含义 |
| :--- | :--- |
| E_MAT_4001 | quantity 非法 |
| E_MAT_4002 | resource_token 格式非法 |
| E_MAT_4003 | cancel_reason 非法 |
| E_MAT_4004 | lost_reason 非法 |
| E_MAT_4005 | void_reason 非法 |
| E_MAT_4030 | 物资域权限不足 |
| E_MAT_4032 | resource_token 审计载荷非法 |
| E_MAT_4041 | order_id 不存在 |
| E_MAT_4044 | tag_code 不存在 |
| E_MAT_4091 | 工单不存在或不允许当前流转 |
| E_MAT_4092 | 工单状态非法，无法自动收敛 |
| E_MAT_4094 | 工单状态冲突，当前状态不允许取消 |
| E_MAT_4095 | 工单取消审核驳回或流转冲突 |
| E_MAT_4096 | 标签三方一致性校验失败 |
| E_MAT_4097 | resource_token patient_id 与路径不一致 |
| E_MAT_4098 | 标签状态冲突，不满足 LOST/VOID 前置状态 |
| E_MAT_4222 | 发货前置条件不满足 |
| E_MAT_4223 | resource_token 验签/解密失败 |
| E_MAT_4224 | 物流异常处置失败 |
| E_MAT_5003 | 资源链内部解析失败 |

### 2.6 AI 域

| 错误码 | 含义 |
| :--- | :--- |
| E_AI_4001 | session_id 或 prompt 非法 |
| E_AI_4002 | AI 会话创建参数非法（patient_id/task_id） |
| E_AI_4003 | memory_note 参数非法 |
| E_AI_4033 | 客户端非法传 patient_id/task_id 或无会话授权 |
| E_AI_4091 | 会话并发冲突（CAS 失败） |
| E_AI_4292 | 用户或患者频率限制触发 |
| E_AI_4293 | 用户或患者配额不足 |
| E_AI_4041 | ai_session 不存在 |

### 2.7 身份权限与治理域（认证与用户子能力）

| 错误码 | 含义 |
| :--- | :--- |
| E_AUTH_4001 | username 格式非法 |
| E_AUTH_4002 | password 格式非法 |
| E_AUTH_4091 | username 已存在 |
| E_USR_4001 | 新旧密码不能相同 |
| E_USR_4002 | 新密码不满足强度策略 |
| E_USR_4003 | status 枚举非法（仅支持 NORMAL/BANNED） |
| E_USR_4011 | old_password 校验失败 |
| E_USR_4041 | user_id 不存在 |

### 2.8 身份权限与治理域（通知子能力）

| 错误码 | 含义 |
| :--- | :--- |
| E_NOTI_4001 | 通知类型或状态筛选参数非法 |
| E_NOTI_4030 | 无通知访问权限 |
| E_NOTI_4041 | notification_id 不存在 |

## 3. HTTP 接口详细

字段类型说明：类型列标注 int64 时，其 JSON 线传类型一律为 string，详见 1.1 ID 传输强制约定。

领域口径说明（与 SADD 4.1 六域映射保持一致）：
1. 本文档领域边界固定为六域：AI 协同决策域、线索与时空研判域、患者档案与标识域、寻回任务执行域、物资运营域、身份权限与治理域。
2. 3.7/3.8/3.9 为同一“身份权限与治理域”的子能力分组，不构成新增领域。
3. 3.6 为查询编排层（跨域只读视图/BFF），用于联调检索便利，不计入领域数量。

接口分组与所属权威域映射：

| 接口分组 | 所属权威域 | 说明 |
| :--- | :--- | :--- |
| 3.1 | 寻回任务执行域 | 任务生命周期与状态收敛 |
| 3.2 | 线索与时空研判域 | 含匿名接入通道，不新增领域 |
| 3.3 | 患者档案与标识域 | 患者档案、监护关系与标签主数据 |
| 3.4 | 物资运营域 | 工单流转；标签流程与档案域协作 |
| 3.5 | AI 协同决策域 | 仅提供建议与会话能力，不直接改业务状态 |
| 3.6 | 查询编排层（非领域） | 跨域只读聚合视图 |
| 3.7-3.9 | 身份权限与治理域 | 认证、用户治理、通知子能力 |

## 3.1 寻回任务执行域（权威域）

### 3.1.1 POST /api/v1/rescue/tasks

用途：发起寻回任务。

权限：FAMILY / ADMIN / SUPERADMIN（且对 patient_id 有授权）。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| patient_id | int64 | 是 | 必须存在且无 ACTIVE 任务 |
| source | string | 是 | APP / MINI_PROGRAM / ADMIN_PORTAL |
| remark | string | 否 | <= 500 |

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| task_id | int64 | 新建任务 ID |
| status | string | ACTIVE |
| event_version | int64 | 初始版本 |

副作用事件：task.created、task.state.changed。

海报触发说明：task.created 事件由 ai-orchestrator 异步消费并生成海报，产出 ai.poster.generated 后由 task-service 回写 poster_url；海报生成失败不影响任务创建主流程。

错误码：E_TASK_4091、E_TASK_4030、E_TASK_4001、E_TASK_4002、E_REQ_4001。

请求示例：
```http
POST /api/v1/rescue/tasks HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "patient_id":  "1001",
    "source":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "task_id":  "1001",
                 "status":  "ACTIVE",
                 "event_version":  "1"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 409
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_TASK_4091",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 201 | OK | 请求成功 |
| 400 | E_TASK_4001、E_TASK_4002、E_REQ_4001 | 参数或格式不合法 |
| 403 | E_TASK_4030 | 权限不足或越权访问 |
| 409 | E_TASK_4091 | 状态冲突或重复提交 |
### 3.1.2 POST /api/v1/rescue/tasks/{task_id}/close

用途：关闭任务（RESOLVED / FALSE_ALARM）。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| close_type | string | 是 | RESOLVED / FALSE_ALARM |
| reason | string | 条件必填 | close_type=FALSE_ALARM 时必填，5-256 |

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| task_id | int64 | 任务 ID |
| status | string | RESOLVED / FALSE_ALARM |
| event_version | int64 | 最新版本 |

副作用事件：task.resolved 或 task.false_alarm，且同步发布 task.state.changed。

错误码：E_TASK_4041、E_TASK_4093、E_TASK_4004、E_TASK_4005、E_TASK_4222、E_REQ_4001。

请求示例：
```http
POST /api/v1/rescue/tasks/1001/close HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "close_type":  "demo",
    "reason":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "task_id":  "1001",
                 "status":  "RESOLVED",
                 "event_version":  "2"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 404
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_TASK_4041",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_TASK_4004、E_TASK_4005、E_REQ_4001 | 参数或格式不合法 |
| 422 | E_TASK_4222 | 业务前置条件不满足 |
| 404 | E_TASK_4041 | 资源不存在或不可见 |
| 409 | E_TASK_4093 | 状态冲突或重复提交 |
### 3.1.3 GET /api/v1/rescue/tasks/{task_id}/snapshot

用途：获取任务当前快照。

响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| task_id | int64 | 任务 ID |
| status | string | ACTIVE/RESOLVED/FALSE_ALARM |
| patient_id | int64 | 患者 ID |
| version | int64 | 状态版本 |
| event_time | string | 最后状态事件时间 |
| latest_trajectory | object | 最近轨迹摘要 |

错误码：E_TASK_4041、E_PRO_4030。

请求示例：
```http
GET /api/v1/rescue/tasks/1001/snapshot HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "task_id":  "1001",
                 "status":  "ACTIVE",
                 "patient_id":  "1001",
                 "version":  "1002",
                 "event_time":  "2026-04-06T10:00:00Z",
                 "latest_trajectory":  {
                                           "event_time":  "2026-04-06T10:00:00Z",
                                           "lat":  23.123456,
                                           "lng":  113.123456
                                       }
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 404
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_TASK_4041",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_TASK_4041 | 资源不存在或不可见 |
### 3.1.4 GET /api/v1/rescue/tasks/{task_id}/trajectory/latest

用途：拉取任务最新轨迹片段。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| limit | int32 | 否 | 1-200，默认 50 |
| since_event_time | string | 否 | ISO-8601 |

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| event_time | string | string | 轨迹时间点（ISO-8601） |
| lat | number | number | 纬度 |
| lng | number | number | 经度 |

错误码：E_TASK_4041、E_CLUE_4043、E_REQ_4001。

请求示例：
```http
GET /api/v1/rescue/tasks/1001/trajectory/latest?limit=1&since_event_time=2026-04-06T10%3A00%3A00Z HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "event_time":  "2026-04-06T10:00:00Z",
                                   "lat":  23.123456,
                                   "lng":  113.123456
                               }
                           ],
                 "has_more":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 404
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_TASK_4041",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 404 | E_TASK_4041、E_CLUE_4043 | 资源不存在或不可见 |
### 3.1.5 GET /api/v1/rescue/tasks

用途：任务列表分页查询。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| status | string | 否 | ACTIVE/RESOLVED/FALSE_ALARM |
| patient_id | int64 | 否 | 授权范围内过滤 |

响应结构：使用统一分页响应。

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| task_id | int64 | string | 任务 ID |
| patient_id | int64 | string | 患者 ID |
| patient_name_masked | string | string | 脱敏姓名 |
| status | string | string | ACTIVE/RESOLVED/FALSE_ALARM |
| source | string | string | APP/MINI_PROGRAM/ADMIN_PORTAL |
| latest_event_time | string | string | 最新状态事件时间（ISO-8601） |
| start_time | string | string | 任务开始时间（ISO-8601） |

字段映射说明：start_time 对应持久化字段 rescue_task.created_at。

请求示例：
```http
GET /api/v1/rescue/tasks?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "task_id":  "1001",
                                   "patient_id":  "1001",
                                   "patient_name_masked":  "王**",
                                   "status":  "ACTIVE",
                                   "source":  "APP",
                                   "latest_event_time":  "2026-04-06T11:00:00Z",
                                   "start_time":  "2026-04-06T10:00:00Z"
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 400
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_REQ_4001",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
### 3.1.6 GET /api/v1/rescue/tasks/{task_id}/events/poll

用途：弱网场景下的长轮询增量事件接口（WS 降级）。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| since_version | int64 | 是 | 客户端已确认版本 |
| timeout_ms | int32 | 否 | 1000-25000，默认 15000 |

返回语义：
1. 有增量时立即返回事件列表（aggregate_id、version、event_time）。
2. 无增量时超时返回空列表。

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| aggregate_id | int64 | string | 聚合 ID（任务 ID） |
| version | int64 | string | 事件版本号 |
| event_time | string | string | 事件时间（ISO-8601） |

请求示例：
```http
GET /api/v1/rescue/tasks/1001/events/poll?since_version=1001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "since_version":  "1001",
                 "latest_version":  "1002",
                 "items":  [
                               {
                                   "aggregate_id":  "1001",
                                   "version":  "1002",
                                   "event_time":  "2026-04-06T10:00:00Z"
                               }
                           ]
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 400
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_REQ_4001",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
### 3.1.7 POST /api/v1/rescue/tasks/{task_id}/force-close

用途：超级管理员强制关闭任务（紧急治理口）。

权限：仅 SUPERADMIN。

AI Agent 执行策略：
1. 本接口执行级别为 `MANUAL_ONLY`。
2. 若 `X-Action-Source=AI_AGENT`，网关必须拒绝并返回 `E_GOV_4231`。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| reason | string | 是 | 5-256，必须说明强制关闭原因 |

状态机约束：
1. 仅允许 ACTIVE -> RESOLVED。
2. RESOLVED/FALSE_ALARM 终态任务禁止重复关闭。

审计约束：
1. 强制关闭必须写入 sys_log（module=TASK，action=FORCE_CLOSE）。
2. 审计写入失败时整笔事务回滚。

错误码：E_GOV_4032、E_GOV_4231、E_TASK_4041、E_TASK_4093、E_GOV_5002、E_REQ_4001。

请求示例：
```http
POST /api/v1/rescue/tasks/1001/force-close HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "reason":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "task_id":  "1001",
                 "patient_id":  "1001",
                 "status":  "RESOLVED",
                 "source":  "APP",
                 "reported_by":  "2001",
                 "remark":  "demo",
                 "close_reason":  "SUPER_FORCE_CLOSE",
                 "start_time":  "2026-04-06T10:00:00Z",
                 "end_time":  "2026-04-06T11:46:00Z",
                 "ai_analysis_summary":  "疑似向东移动，建议沿主路网扩大排查",
                 "poster_url":  "https://cdn.example.com/posters/task_1001.png",
                 "version":  "13",
                 "event_time":  "2026-04-06T11:46:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

字段映射说明：reported_by/start_time/end_time 分别映射 rescue_task.created_by/created_at/closed_at。

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4032",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_GOV_4032 | 权限不足或越权访问 |
| 423 | E_GOV_4231 | 当前接口仅允许人工执行 |
| 404 | E_TASK_4041 | 资源不存在或不可见 |
| 409 | E_TASK_4093 | 状态冲突或重复提交 |
| 500 | E_GOV_5002 | 内部处理失败或审计回滚 |
### 3.1.8 GET /api/v1/rescue/tasks/{task_id}

用途：获取任务完整详情（用于前端首屏与刷新重建）。

权限：
1. FAMILY：需具备 patient_id 授权。
2. ADMIN/SUPERADMIN：按管理权限访问。

响应 data：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| task_id | int64 | string | 任务 ID |
| patient_id | int64 | string | 患者 ID |
| status | string | string | ACTIVE/RESOLVED/FALSE_ALARM |
| source | string | string | APP/MINI_PROGRAM/ADMIN_PORTAL |
| reported_by | int64 | string | 发起人用户 ID |
| remark | string | string | 任务备注 |
| close_reason | string | string | 关闭原因（可空） |
| start_time | string | string | 任务开始时间（ISO-8601） |
| end_time | string | string | 任务结束时间（ISO-8601，可空） |
| ai_analysis_summary | string | string | AI 研判摘要（可空） |
| poster_url | string | string | AI 海报链接（可空） |
| version | int64 | string | 状态版本 |
| event_time | string | string | 最新状态事件时间 |

字段映射说明（API vs 持久化）：
1. reported_by 为 API 领域语义名，持久化字段映射为 rescue_task.created_by。
2. start_time 为 API 领域语义名，持久化字段映射为 rescue_task.created_at。
3. end_time 为 API 领域语义名，持久化字段映射为 rescue_task.closed_at。

错误码：E_TASK_4041、E_PRO_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/rescue/tasks/1001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "task_id":  "1001",
                 "patient_id":  "1001",
                 "status":  "ACTIVE",
                 "source":  "APP",
                 "reported_by":  "2001",
                 "remark":  "demo",
                 "close_reason":  null,
                 "start_time":  "2026-04-06T10:00:00Z",
                 "end_time":  null,
                 "version":  "12",
                 "event_time":  "2026-04-06T10:05:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 404
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_TASK_4041",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_TASK_4041 | 资源不存在或不可见 |
### 3.1.9 GET /api/v1/rescue/tasks/{task_id}/events

用途：分页读取任务事件流（用于前端重建与排障）。

权限：FAMILY（需患者授权）/ ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| cursor | string | 否 | 透明游标；优先使用 |
| page_size | int32 | 否 | 1-200，默认 50 |
| after_version | int64 | 否 | 兼容字段（Deprecated）；语义等同 cursor 起点 |

分页规则：
1. 默认按 event_version ASC 返回，保证事件回放顺序稳定。
2. 存在 cursor 时按 Cursor 滚动翻页，响应 data 返回 next_cursor。
3. after_version 仅用于兼容旧客户端，服务端内部转换为 Cursor 处理。
4. 当仅传 after_version 时，服务端先将其转换为起始版本游标（version=after_version，id 为空哨兵），再按 Cursor 统一流程返回 next_cursor。

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| event_id | string | string | 事件 ID |
| event_type | string | string | 事件类型 |
| aggregate_id | int64 | string | 聚合 ID |
| version | int64 | string | 版本号 |
| event_time | string | string | 事件时间（ISO-8601） |

错误码：E_GOV_4011、E_TASK_4041、E_PRO_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/rescue/tasks/1001/events?cursor=eyJ2ZXJzaW9uIjoiMTAwMSIsImlkIjoiZXZ0XzEwMDEifQ%3D%3D&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "event_id":  "evt_20260406_0001",
                                   "event_type":  "task.state.changed",
                                   "aggregate_id":  "1001",
                                   "version":  "12",
                                   "event_time":  "2026-04-06T11:00:00Z"
                               }
                           ],
                 "page_size":  20,
                 "next_cursor":  "eyJ2ZXJzaW9uIjoiMTAyMSIsImlkIjoiZXZ0XzEwMjEifQ==",
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_TASK_4041 | 资源不存在或不可见 |
### 3.1.10 GET /api/v1/rescue/tasks/{task_id}/clues

用途：分页读取任务关联线索。

权限：FAMILY（需患者授权）/ ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| suspected_only | bool | 否 | true 仅返回待复核 |

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| clue_id | int64 | string | 线索 ID |
| patient_id | int64 | string | 关联患者 ID |
| is_valid | bool | boolean | 有效性判定 |
| suspect_reason | string | string | 可疑原因（可空） |
| reported_at | string | string | 上报时间（ISO-8601） |

错误码：E_GOV_4011、E_TASK_4041、E_PRO_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/rescue/tasks/1001/clues?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "clue_id":  "2001",
                                   "patient_id":  "1001",
                                   "is_valid":  true,
                                   "suspect_reason":  null,
                                   "reported_at":  "2026-04-06T11:02:00Z"
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_TASK_4041 | 资源不存在或不可见 |
### 3.1.11 GET /api/v1/admin/rescue/tasks

用途：管理端分页检索任务。

权限：ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| status | string | 否 | ACTIVE/RESOLVED/FALSE_ALARM |
| source | string | 否 | APP/MINI_PROGRAM/ADMIN_PORTAL |

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| task_id | int64 | string | 任务 ID |
| patient_id | int64 | string | 患者 ID |
| status | string | string | ACTIVE/RESOLVED/FALSE_ALARM |
| source | string | string | APP/MINI_PROGRAM/ADMIN_PORTAL |
| reported_by | int64 | string | 发起人用户 ID |
| event_time | string | string | 最新状态事件时间（ISO-8601） |

错误码：E_GOV_4011、E_GOV_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/rescue/tasks?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "task_id":  "1001",
                                   "patient_id":  "1001",
                                   "status":  "ACTIVE",
                                   "source":  "APP",
                                   "reported_by":  "2001",
                                   "event_time":  "2026-04-06T11:00:00Z"
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
### 3.1.12 GET /api/v1/admin/rescue/tasks/{task_id}

用途：管理端读取任务详情（治理视角）。

权限：ADMIN / SUPERADMIN。

错误码：E_GOV_4011、E_GOV_4030、E_TASK_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/rescue/tasks/1001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "task_id":  "1001",
                 "patient_id":  "1001",
                 "status":  "ACTIVE",
                 "source":  "ADMIN_PORTAL",
                 "reported_by":  "9001",
                 "remark":  "重点排查东侧地铁口",
                 "close_reason":  null,
                 "start_time":  "2026-04-06T10:00:00Z",
                 "end_time":  null,
                 "ai_analysis_summary":  "建议优先排查轨迹高频区域",
                 "poster_url":  "https://cdn.example.com/posters/task_1001.png",
                 "version":  "12",
                 "event_time":  "2026-04-06T10:05:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_TASK_4041 | 资源不存在或不可见 |
### 3.1.13 GET /api/v1/admin/rescue/tasks/{task_id}/audit

用途：读取任务状态流转与高危操作审计轨迹。

权限：ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| cursor | string | 否 | 透明游标；优先使用 |
| page_size | int32 | 否 | 1-200，默认 50 |

分页规则：
1. 默认按 created_at DESC, audit_id DESC 返回。
2. 存在 cursor 时按 Cursor 模式滚动翻页，响应 data 返回 next_cursor。

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| audit_id | int64 | string | 审计记录 ID |
| module | string | string | 模块 |
| action | string | string | 动作 |
| operator_user_id | int64 | string | 操作人 |
| result | string | string | SUCCESS/FAIL |
| created_at | string | string | 记录时间（ISO-8601） |

错误码：E_GOV_4011、E_GOV_4030、E_TASK_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/rescue/tasks/1001/audit?cursor=eyJjcmVhdGVkX2F0IjoiMjAyNi0wNC0wNlQxMDowMDowMFoiLCJpZCI6IjEwMDEifQ%3D%3D&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "audit_id":  "9001",
                                   "module":  "TASK",
                                   "action":  "FORCE_CLOSE",
                                   "operator_user_id":  "9001",
                                   "result":  "SUCCESS",
                                   "created_at":  "2026-04-06T11:03:00Z"
                               }
                           ],
                 "page_size":  20,
                 "next_cursor":  "eyJhdWRpdF9pZCI6IjkwMDEifQ==",
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_TASK_4041 | 资源不存在或不可见 |
### 3.1.14 GET /api/v1/rescue/tasks/{task_id}/clues/latest

用途：读取任务最新有效线索聚合。

权限：FAMILY（需患者授权）/ ADMIN / SUPERADMIN。

错误码：E_GOV_4011、E_TASK_4041、E_CLUE_4043、E_REQ_4005。

请求示例：
```http
GET /api/v1/rescue/tasks/1001/clues/latest HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "task_id":  "1001",
                 "accepted_channels":  [
                                           "IN_APP",
                                           "PUSH"
                                       ],
                 "retry_job_id":  "nrt_20260406_0001",
                 "processed_at":  "2026-04-06T10:10:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 404 | E_TASK_4041、E_CLUE_4043 | 资源不存在或不可见 |
### 3.1.15 GET /api/v1/rescue/tasks/{task_id}/alerts

用途：分页读取任务告警摘要。

权限：FAMILY（需患者授权）/ ADMIN / SUPERADMIN。

数据来源（毕设精简）：直接读取 `notification_inbox`（按 `related_task_id + type + level` 过滤）。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| level | string | 否 | INFO/WARN/CRITICAL |

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| alert_id | int64 | string | 告警 ID |
| level | string | string | INFO/WARN/CRITICAL |
| type | string | string | 告警类型 |
| title | string | string | 告警标题 |
| created_at | string | string | 产生时间（ISO-8601） |

错误码：E_GOV_4011、E_TASK_4041、E_PRO_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/rescue/tasks/1001/alerts?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "alert_id":  "7001",
                                   "level":  "WARN",
                                   "type":  "FENCE_ALERT",
                                   "title":  "围栏异常告警",
                                   "created_at":  "2026-04-06T11:04:00Z"
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_TASK_4041 | 资源不存在或不可见 |
### 3.1.16 POST /api/v1/admin/rescue/tasks/{task_id}/notify/retry

用途：对指定任务执行通知补偿重放。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| reason | string | 是 | 5-256 |
| channels | array<string> | 否 | IN_APP/PUSH（遵循 HC-06，不支持 SMS/EMAIL） |

通道约束：通知补偿仅支持应用推送（PUSH）与站内通知（IN_APP）。

审计约束：
1. 必须写 sys_log（module=TASK，action=NOTIFY_RETRY）。
2. 审计写入失败整笔回滚。

错误码：E_GOV_4011、E_GOV_4030、E_TASK_4041、E_REQ_4001、E_REQ_4005、E_GOV_5002。

请求示例：
```http
POST /api/v1/admin/rescue/tasks/1001/notify/retry HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "reason":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "task_id":  "1001",
                 "accepted_channels":  [
                                           "IN_APP",
                                           "PUSH"
                                       ],
                 "retry_job_id":  "nrt_20260406_0001",
                 "processed_at":  "2026-04-06T10:10:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001、E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_TASK_4041 | 资源不存在或不可见 |
| 500 | E_GOV_5002 | 内部处理失败或审计回滚 |
### 3.1.17 GET /api/v1/rescue/tasks/statistics

用途：读取任务域统计指标（运营看板）。

权限：ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| time_from | string | 否 | ISO-8601 |
| time_to | string | 否 | ISO-8601 |
| granularity | string | 否 | day/week/month，默认 day |

错误码：E_GOV_4011、E_GOV_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/rescue/tasks/statistics?time_from=2026-04-06T10%3A00%3A00Z&time_to=2026-04-06T10%3A00%3A00Z HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "time_from":  "2026-04-01T00:00:00Z",
                 "time_to":  "2026-04-06T23:59:59Z",
                 "granularity":  "day",
                 "total_tasks":  38,
                 "active_tasks":  7,
                 "resolved_tasks":  27,
                 "false_alarm_tasks":  4,
                 "resolution_rate":  0.7105
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
## 3.2 线索与时空研判域（含匿名接入通道）

### 3.2.0 匿名 Cookie 与跨域凭据约束

1. 浏览器端（H5）必须使用 HttpOnly Cookie 传递 entry_token，前端 JavaScript 不可直接读取。
2. 非浏览器端（APP/MINI_PROGRAM）允许通过 X-Anonymous-Token 传递等效匿名凭据。
3. 若同时携带 Cookie(entry_token) 与 X-Anonymous-Token 且值不一致，网关必须拒绝并返回 E_CLUE_4012。
4. 跨域调用匿名相关接口时，浏览器 Fetch 必须设置 credentials: include；Axios 必须设置 withCredentials: true。
5. 网关必须返回 Access-Control-Allow-Credentials: true 且使用明确 Origin，禁止配合通配符 *。
6. 若 Cookie 与 Header 两种匿名凭据均缺失或无效，返回 E_CLUE_4012。
7. 非浏览器端凭据来源：
    1. /api/v1/public/clues/manual-entry 返回的 manual_entry_token 可直接作为 X-Anonymous-Token。
    2. /r/{resource_token} 跳转响应可回写同值匿名令牌（见 3.2.1）。

### 3.2.1 GET /r/{resource_token}

用途：路人扫码入口，验签后动态路由。

行为：
1. 标签 BOUND -> 302 到 /p/{short_code}/clues/new 并下发 entry_token Cookie。
2. 标签 LOST -> 302 到 /p/{short_code}/emergency/report 并下发 entry_token Cookie。
3. 标签 UNBOUND/ALLOCATED/VOID -> 无效页。
4. 对 APP/MINI_PROGRAM 客户端，网关在响应头同步回写 X-Anonymous-Token（与 entry_token 等值）。

错误码：E_MAT_4002、E_MAT_4223、E_CLUE_4041。

请求示例：
```http
GET /r/rtk_demo_001 HTTP/1.1
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 302 Found
Location: /p/A1B2C3/clues/new
Set-Cookie: entry_token=token_demo_001; HttpOnly; Secure; SameSite=Strict; Max-Age=120
X-Anonymous-Token: token_demo_001
X-Trace-Id: trc_demo_20260406_001
```

响应示例（失败）：
```http
HTTP/1.1 400
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_MAT_4002",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 302 | - | 动态路由重定向成功 |
| 400 | E_MAT_4002 | 参数或格式不合法 |
| 404 | E_CLUE_4041 | 资源不存在或不可见 |
| 422 | E_MAT_4223 | 业务前置条件不满足 |
### 3.2.2 POST /api/v1/public/clues/manual-entry

用途：二维码污损时的手动兜底。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| short_code | string | 是 | 固定 6 位大写字母数字 |
| pin_code | string | 是 | 固定 6 位 |
| captcha_token | string | 是 | 必须通过验证码校验 |
| device_fingerprint | string | 是 | 16-128 |

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| manual_entry_token | string | 一次性令牌，TTL <= 120s；与 Set-Cookie 下发的 entry_token 值一致 |

令牌下发约束：
1. 验证通过后，网关必须同时通过 Set-Cookie 下发 entry_token。
2. Cookie 属性必须为 HttpOnly; Secure; SameSite=Strict; Max-Age<=120。
3. 浏览器前端必须依赖 Cookie 自动携带令牌，不得尝试用 JavaScript 写入 HttpOnly Cookie。
4. manual_entry_token 主要用于非浏览器调试链路，浏览器链路以 Cookie 为准。

风控限制：
1. IP <= 5 次/分钟。
2. 设备 <= 20 次/小时。
3. short_code 连续失败 >= 5 次触发 15 分钟冷却。

错误码：E_CLUE_4005、E_CLUE_4006、E_GOV_4038、E_GOV_4004、E_GOV_4291、E_GOV_4292。

请求示例：
```http
POST /api/v1/public/clues/manual-entry HTTP/1.1
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "pin_code":  "demo",
    "captcha_token":  "token_demo_001",
    "short_code":  "A1B2C3",
    "device_fingerprint":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 201
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "manual_entry_token":  "token_demo_001"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 400
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_CLUE_4005",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（限流）：
```http
HTTP/1.1 429
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
Retry-After: 30
X-RateLimit-Remaining: 0
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4291",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 201 | OK | 请求成功 |
| 400 | E_CLUE_4005、E_CLUE_4006、E_GOV_4004 | 参数或格式不合法 |
| 403 | E_GOV_4038 | 权限不足或越权访问 |
| 429 | E_GOV_4291、E_GOV_4292 | 频率或配额限制触发 |
### 3.2.3 GET /p/{short_code}/clues/new

用途：匿名线索上传页面（普通模式）。

访问约束：
1. 必须带 entry_token（HttpOnly Cookie）。
2. short_code 必须可公开且未失效。

错误码：E_CLUE_4005、E_CLUE_4012、E_CLUE_4042。

请求示例：
```http
GET /p/A1B2C3/clues/new HTTP/1.1
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 302 Found
Location: /public/clues/new?short_code=A1B2C3
X-Trace-Id: trc_demo_20260406_001
```

响应示例（失败）：
```http
HTTP/1.1 400
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_CLUE_4005",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 302 | - | 页面路由成功 |
| 400 | E_CLUE_4005 | 参数或格式不合法 |
| 401 | E_CLUE_4012 | 鉴权失败或凭据缺失/失效 |
| 404 | E_CLUE_4042 | 资源不存在或不可见 |
### 3.2.4 GET /p/{short_code}/emergency/report

用途：匿名紧急上报页面（LOST 模式）。

访问约束：同 /p/{short_code}/clues/new。

请求示例：
```http
GET /p/A1B2C3/emergency/report HTTP/1.1
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 302 Found
Location: /public/emergency/report?short_code=A1B2C3
X-Trace-Id: trc_demo_20260406_001
```

响应示例（失败）：
```http
HTTP/1.1 400
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_REQ_4001",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 302 | - | 页面路由成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
### 3.2.5 POST /api/v1/clues/report

用途：提交匿名线索。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| tag_code | string | 是 | 长度 8-100，且与匿名凭据绑定的 tag_code 一致 |
| coord_system | string | 是 | 枚举：WGS84 / GCJ-02 / BD-09 |
| location | object | 是 | 原始定位对象，包含 lat/lng（按 coord_system 声明） |
| description | string | 否 | <= 2000 |
| photo_url | string | 否 | 白名单域名 |

location 子字段：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| lat | number | 是 | 合法范围 |
| lng | number | 是 | 合法范围 |

坐标处理约束：
1. 高德（AMap）来源必须声明 `coord_system=GCJ-02`。
2. 网关将 `GCJ-02/BD-09` 标准化转换为 `WGS84` 后再进入业务服务与落库。
3. 任一坐标系声明非法或转换失败，返回 `E_CLUE_4007`。

鉴权约束：
1. 不接受 Body 传递 entry_token，网关只从 Cookie(entry_token) 或 X-Anonymous-Token 提取匿名凭据。
2. 网关完成凭据校验后，才进入 tag_code 一致性与业务校验。
3. source_type 由服务端按入口上下文推导：Cookie(entry_token)/X-Anonymous-Token 场景写入 SCAN，人工录入接口（3.2.2）写入 MANUAL；客户端不可覆盖。

隐私约束：
1. 匿名上报成功响应不得返回 patient_id、患者姓名等可识别信息。
2. 返回体仅提供线索受理回执字段（如 clue_id、status、reported_at）。

回执语义：`status=RECEIVED` 为瞬时受理回执态，不作为 `clue_record` 持久化状态列。

副作用事件：clue.reported.raw。

错误码：E_CLUE_4012、E_CLUE_4041、E_CLUE_4001、E_CLUE_4002、E_CLUE_4003、E_CLUE_4004、E_CLUE_4007。

请求示例：
```http
POST /api/v1/clues/report HTTP/1.1
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Cookie: entry_token=token_demo_001
Content-Type: application/json

{
    "coord_system":  "GCJ-02",
    "location":  {
                     "lat":  23.123456,
                     "lng":  113.123456
                 },
    "tag_code":  "TAG000001"
}
```

响应示例（成功）：
```http
HTTP/1.1 201
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "clue_id":  "2001",
                 "status":  "RECEIVED",
                 "reported_at":  "2026-04-06T10:08:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_CLUE_4012",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 201 | OK | 请求成功 |
| 400 | E_CLUE_4001、E_CLUE_4002、E_CLUE_4003、E_CLUE_4004、E_CLUE_4007 | 参数或格式不合法 |
| 401 | E_CLUE_4012 | 鉴权失败或凭据缺失/失效 |
| 404 | E_CLUE_4041 | 资源不存在或不可见 |
### 3.2.6 POST /api/v1/clues/{clue_id}/override

用途：管理员强制回流可疑线索。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| override | bool | 是 | 仅允许 true |
| override_reason | string | 是 | 5-256 |

副作用事件：重发 clue.validated（override=true）。

错误码：E_CLUE_4043、E_CLUE_4008、E_CLUE_4009、E_GOV_4030、E_CLUE_5011。

请求示例：
```http
POST /api/v1/clues/2001/override HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "override_reason":  "demo",
    "override":  true
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "clue_id":  "2001",
                 "task_id":  "1001",
                 "patient_id":  "1001",
                 "tag_code":  "TAG000001",
                 "coord_system":  "WGS84",
                 "location":  {
                                  "lat":  31.2304,
                                  "lng":  121.4737
                              },
                 "description":  "在地铁口附近发现疑似目标",
                 "photo_url":  "https://cdn.example.com/clues/2001.jpg",
                 "is_valid":  true,
                 "suspect_reason":  "位置跳变异常，需人工复核",
                 "override":  true,
                 "override_reason":  "轨迹交叉验证后确认为有效线索",
                 "rejected_by":  null,
                 "reject_reason":  null,
                 "reported_at":  "2026-04-06T10:08:00Z",
                 "reviewed_at":  "2026-04-06T10:12:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 404
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_CLUE_4043",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_CLUE_4008、E_CLUE_4009 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_CLUE_4043 | 资源不存在或不可见 |
| 501 | E_CLUE_5011 | 业务处理失败 |
### 3.2.7 POST /api/v1/clues/{clue_id}/reject

用途：管理员驳回可疑线索并关闭复核工单。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| reject_reason | string | 是 | 5-256 |

副作用事件：clue.rejected。

错误码：E_CLUE_4043、E_CLUE_4010、E_GOV_4030、E_CLUE_5012。

请求示例：
```http
POST /api/v1/clues/2001/reject HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "reject_reason":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "clue_id":  "2001",
                 "task_id":  "1001",
                 "patient_id":  "1001",
                 "tag_code":  "TAG000001",
                 "coord_system":  "WGS84",
                 "location":  {
                                  "lat":  31.2304,
                                  "lng":  121.4737
                              },
                 "description":  "在地铁口附近发现疑似目标",
                 "photo_url":  "https://cdn.example.com/clues/2001.jpg",
                 "is_valid":  false,
                 "suspect_reason":  "位置跳变异常，需人工复核",
                 "override":  false,
                 "override_reason":  null,
                 "rejected_by":  "3001",
                 "reject_reason":  "轨迹信息冲突，证据不足",
                 "reported_at":  "2026-04-06T10:08:00Z",
                 "reviewed_at":  "2026-04-06T10:16:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 404
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_CLUE_4043",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 401 | E_CLUE_4010 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_CLUE_4043 | 资源不存在或不可见 |
| 501 | E_CLUE_5012 | 业务处理失败 |
### 3.2.8 GET /api/v1/clues/{clue_id}

用途：读取线索完整详情（管理端复核页、轨迹回放页）。

权限：ADMIN / SUPERADMIN，或具备关联患者授权的 FAMILY。

响应 data：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| clue_id | int64 | string | 线索 ID |
| task_id | int64 | string | 关联任务 ID（可空） |
| patient_id | int64 | string | 关联患者 ID（可空） |
| tag_code | string | string | 标签编码 |
| source_type | string | string | SCAN/MANUAL |
| coord_system | string | string | 标准化后固定返回 WGS84 |
| location | object | object | 标准化坐标 |
| description | string | string | 线索描述 |
| photo_url | string | string | 线索图片 |
| is_valid | bool | boolean | 是否有效 |
| suspect_reason | string | string | 可疑原因（可空） |
| override | bool | boolean | 是否人工强制回流 |
| override_reason | string | string | 强制回流原因（可空） |
| rejected_by | int64 | string | 驳回人（可空） |
| reject_reason | string | string | 驳回原因（可空） |
| reported_at | string | string | 上报时间（ISO-8601） |
| reviewed_at | string | string | 复核时间（ISO-8601，可空） |

错误码：E_CLUE_4043、E_GOV_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/clues/2001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "clue_id":  "2001",
                 "task_id":  "1001",
                 "patient_id":  "1001",
                 "tag_code":  "TAG000001",
                 "source_type":  "SCAN",
                 "coord_system":  "WGS84",
                 "location":  {
                                  "lat":  31.2304,
                                  "lng":  121.4737
                              },
                 "description":  "在地铁口附近发现疑似目标",
                 "photo_url":  "https://cdn.example.com/clues/2001.jpg",
                 "is_valid":  true,
                 "suspect_reason":  "位置跳变异常，需人工复核",
                 "override":  false,
                 "override_reason":  null,
                 "rejected_by":  null,
                 "reject_reason":  null,
                 "reported_at":  "2026-04-06T10:08:00Z",
                 "reviewed_at":  null
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 404
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_CLUE_4043",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_CLUE_4043 | 资源不存在或不可见 |
### 3.2.9 GET /api/v1/admin/clues/{clue_id}

用途：管理端读取线索复核详情。

权限：ADMIN / SUPERADMIN。

错误码：E_GOV_4011、E_GOV_4030、E_CLUE_4043、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/clues/2001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "clue_id":  "2001",
                 "task_id":  "1001",
                 "patient_id":  "1001",
                 "tag_code":  "TAG000001",
                 "source_type":  "SCAN",
                 "coord_system":  "WGS84",
                 "location":  {
                                  "lat":  31.2304,
                                  "lng":  121.4737
                              },
                 "description":  "在地铁口附近发现疑似目标",
                 "photo_url":  "https://cdn.example.com/clues/2001.jpg",
                 "is_valid":  true,
                 "suspect_reason":  "位置跳变异常，需人工复核",
                 "review_status":  "PENDING",
                 "assignee_user_id":  "3001",
                 "override":  false,
                 "override_reason":  null,
                 "rejected_by":  null,
                 "reject_reason":  null,
                 "reported_at":  "2026-04-06T10:08:00Z",
                 "reviewed_at":  null
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_CLUE_4043 | 资源不存在或不可见 |
### 3.2.10 GET /api/v1/admin/clues/review/queue

用途：管理端读取待复核队列。

权限：ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| assignee_user_id | int64 | 否 | 复核责任人 |

兼容说明：`assigned_to` 作为兼容别名保留（Deprecated），建议客户端统一使用 `assignee_user_id`。

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| clue_id | int64 | string | 线索 ID |
| task_id | int64 | string | 关联任务 ID |
| patient_id | int64 | string | 关联患者 ID |
| risk_score | number | number | 风险评分 |
| suspect_reason | string | string | 可疑原因摘要 |
| assignee_user_id | int64 | string | 复核责任人 |
| reported_at | string | string | 上报时间（ISO-8601） |

错误码：E_GOV_4011、E_GOV_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/clues/review/queue?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "clue_id":  "2001",
                                   "task_id":  "1001",
                                   "patient_id":  "1001",
                                   "risk_score":  0.92,
                                   "suspect_reason":  "位置跳变异常",
                                   "assignee_user_id":  "3001",
                                   "reported_at":  "2026-04-06T11:05:00Z"
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
### 3.2.11 GET /api/v1/clues/{clue_id}/timeline

用途：读取单条线索处理轨迹。

权限：FAMILY（需患者授权）/ ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| cursor | string | 否 | 透明游标；优先使用 |
| page_size | int32 | 否 | 1-200，默认 50 |

分页规则：
1. 默认按 created_at DESC, timeline_id DESC 返回。
2. 存在 cursor 时按 Cursor 模式滚动翻页，响应 data 返回 next_cursor。
3. timeline 记录来源于 sys_log（module='CLUE'），timeline_id 对应 sys_log.id。
4. cursor 基于 (created_at, id) 复合排序编码，其中 id 即 timeline_id。

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| timeline_id | int64 | string | 轨迹记录 ID |
| action | string | string | 处理动作 |
| operator_user_id | int64 | string | 操作人 ID |
| remark | string | string | 处理备注（可空） |
| created_at | string | string | 记录时间（ISO-8601） |

错误码：E_GOV_4011、E_CLUE_4043、E_PRO_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/clues/2001/timeline?cursor=eyJjcmVhdGVkX2F0IjoiMjAyNi0wNC0wNlQxMDowMDowMFoiLCJpZCI6IjIwMDEifQ%3D%3D&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "timeline_id":  "9101",
                                   "action":  "OVERRIDE",
                                   "operator_user_id":  "3001",
                                   "remark":  "人工确认为有效线索",
                                   "created_at":  "2026-04-06T11:06:00Z"
                               }
                           ],
                 "page_size":  20,
                 "next_cursor":  "eyJ0aW1lbGluZV9pZCI6IjkxMDEifQ==",
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_CLUE_4043 | 资源不存在或不可见 |
### 3.2.12 POST /api/v1/admin/clues/{clue_id}/assign

用途：分派线索复核责任人。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| assignee_user_id | int64 | 是 | 目标复核人用户 ID |
| reason | string | 否 | 0-256 |

错误码：E_GOV_4011、E_GOV_4030、E_CLUE_4043、E_REQ_4001、E_REQ_4005。

请求示例：
```http
POST /api/v1/admin/clues/2001/assign HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "assignee_user_id":  "1001"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "clue_id":  "2001",
                 "assignee_user_id":  "3001",
                 "assigned_at":  "2026-04-06T10:15:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001、E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_CLUE_4043 | 资源不存在或不可见 |
### 3.2.13 POST /api/v1/admin/clues/{clue_id}/request-evidence

用途：发起线索补证请求（预留接口）。

毕设精简说明：
1. 当前版本暂不开放本接口。
2. 不落地 `clue_evidence_request` 独立存储。
3. 相关尝试仅保留治理审计（sys_log）。

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 403 | E_GOV_4030 | 毕设版本暂不开放 |
### 3.2.14 GET /api/v1/admin/clues/statistics

用途：读取线索复核统计指标。

权限：ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| time_from | string | 否 | ISO-8601 |
| time_to | string | 否 | ISO-8601 |
| granularity | string | 否 | day/week/month，默认 day |

错误码：E_GOV_4011、E_GOV_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/clues/statistics?time_from=2026-04-06T10%3A00%3A00Z&time_to=2026-04-06T10%3A00%3A00Z HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "time_from":  "2026-04-01T00:00:00Z",
                 "time_to":  "2026-04-06T23:59:59Z",
                 "granularity":  "day",
                 "total_clues":  126,
                 "suspected_count":  19,
                 "overridden_count":  6,
                 "rejected_count":  11,
                 "avg_review_minutes":  42
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
## 3.3 患者档案与标识域（含监护关系）

### 3.3.1 POST /api/v1/patients/{patient_id}/guardians/invitations

用途：主监护邀请成员加入。

交互前置：
1. 前端若仅掌握手机号或账号名，必须先调用 3.3.7 用户查询接口换取 invitee_user_id。
2. 前端禁止猜测、拼接或缓存过期的 user_id。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| invitee_user_id | int64 | 是 | 由 3.3.7 查询获得；被邀请用户存在且非 ACTIVE 成员 |
| relation_role | string | 是 | 仅 GUARDIAN |
| reason | string | 否 | <= 256 |

错误码：E_PRO_4032、E_PRO_4041、E_PRO_4042、E_PRO_4094、E_PRO_4006、E_REQ_4001。

请求示例：
```http
POST /api/v1/patients/1001/guardians/invitations HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "invitee_user_id":  "1001",
    "relation_role":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 201
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "invite_id":  "3001"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4032",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 201 | OK | 请求成功 |
| 400 | E_PRO_4006、E_REQ_4001 | 参数或格式不合法 |
| 403 | E_PRO_4032 | 权限不足或越权访问 |
| 404 | E_PRO_4041、E_PRO_4042 | 资源不存在或不可见 |
| 409 | E_PRO_4094 | 状态冲突或重复提交 |
### 3.3.2 POST /api/v1/patients/{patient_id}/guardians/invitations/{invite_id}/accept

用途：受邀用户确认或拒绝。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| action | string | 是 | ACCEPT / REJECT |
| reject_reason | string | 条件必填 | action=REJECT 时必填，5-256 |

错误码：E_PRO_4041、E_PRO_4043、E_PRO_4096、E_PRO_4008、E_REQ_4001。

请求示例：
```http
POST /api/v1/patients/1001/guardians/invitations/3001/accept HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "action":  "ACCEPT"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "invite_id":  "3001",
                 "patient_id":  "1001",
                 "inviter_user_id":  "2001",
                 "invitee_user_id":  "2002",
                 "relation_role":  "GUARDIAN",
                 "status":  "ACCEPTED",
                 "reason":  "请协助共同监护",
                 "expire_at":  "2026-04-08T10:00:00Z",
                 "created_at":  "2026-04-06T10:00:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 404
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4041",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_PRO_4008、E_REQ_4001 | 参数或格式不合法 |
| 404 | E_PRO_4041、E_PRO_4043 | 资源不存在或不可见 |
| 409 | E_PRO_4096 | 状态冲突或重复提交 |
### 3.3.3 POST /api/v1/patients/{patient_id}/guardians/primary-transfer

用途：发起主监护权转移（PENDING_CONFIRM）。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| target_user_id | int64 | 是 | 必须是 ACTIVE 监护成员 |
| reason | string | 是 | 5-256 |
| expire_in_seconds | int32 | 否 | 300-86400，默认 1800 |

错误码：E_PRO_4032、E_PRO_4041、E_PRO_4044、E_PRO_4095、E_PRO_4009、E_PRO_4010、E_REQ_4001。

请求示例：
```http
POST /api/v1/patients/1001/guardians/primary-transfer HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "target_user_id":  "1001",
    "reason":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 201
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "transfer_request_id":  "trf_20260406_0001",
                 "patient_id":  "1001",
                 "from_user_id":  "2001",
                 "to_user_id":  "2002",
                 "transfer_state":  "PENDING_CONFIRM",
                 "requested_at":  "2026-04-06T10:00:00Z",
                 "expire_at":  "2026-04-06T10:30:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4032",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 201 | OK | 请求成功 |
| 400 | E_PRO_4009、E_REQ_4001 | 参数或格式不合法 |
| 401 | E_PRO_4010 | 鉴权失败或凭据缺失/失效 |
| 403 | E_PRO_4032 | 权限不足或越权访问 |
| 404 | E_PRO_4041、E_PRO_4044 | 资源不存在或不可见 |
| 409 | E_PRO_4095 | 状态冲突或重复提交 |
### 3.3.4 POST /api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/confirm

用途：受方确认或拒绝转移。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| action | string | 是 | ACCEPT / REJECT |
| reject_reason | string | 条件必填 | action=REJECT 时必填，5-256 |

错误码：E_PRO_4041、E_PRO_4045、E_PRO_4097、E_PRO_4099、E_PRO_4011、E_PRO_4013、E_REQ_4001。

请求示例：
```http
POST /api/v1/patients/1001/guardians/primary-transfer/4001/confirm HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "action":  "ACCEPT"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "transfer_request_id":  "4001",
                 "patient_id":  "1001",
                 "from_user_id":  "2001",
                 "to_user_id":  "2002",
                 "transfer_state":  "ACCEPTED",
                 "reason":  "临时出差，需家属接管",
                 "requested_at":  "2026-04-06T10:00:00Z",
                 "expire_at":  "2026-04-07T10:00:00Z",
                 "confirmed_at":  "2026-04-06T10:12:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 404
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4041",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 401 | E_PRO_4011、E_PRO_4013 | 鉴权失败或凭据缺失/失效 |
| 404 | E_PRO_4041、E_PRO_4045 | 资源不存在或不可见 |
| 409 | E_PRO_4097、E_PRO_4099 | 状态冲突或重复提交 |

落库语义：
1. `action=ACCEPT`：`transfer_state=ACCEPTED`，`confirmed_at` 映射 `sys_user_patient.transfer_confirmed_at`，并在同事务完成主监护角色切换。
2. `action=REJECT`：`transfer_state=REJECTED`，`confirmed_at` 固定返回 `null`，并写入 `sys_user_patient.transfer_rejected_at/transfer_reject_reason`。

### 3.3.5 POST /api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/cancel

用途：发起方撤销未确认转移。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| cancel_reason | string | 是 | 5-256 |

错误码：E_PRO_4041、E_PRO_4045、E_PRO_4098、E_PRO_4012、E_REQ_4001。

请求示例：
```http
POST /api/v1/patients/1001/guardians/primary-transfer/4001/cancel HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "cancel_reason":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "transfer_request_id":  "4001",
                 "patient_id":  "1001",
                 "transfer_state":  "CANCELLED",
                 "cancel_reason":  "demo",
                 "cancelled_at":  "2026-04-06T10:30:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 404
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4041",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 401 | E_PRO_4012 | 鉴权失败或凭据缺失/失效 |
| 404 | E_PRO_4041、E_PRO_4045 | 资源不存在或不可见 |
| 409 | E_PRO_4098 | 状态冲突或重复提交 |

落库语义：
1. `transfer_state/cancel_reason/cancelled_at` 分别映射 `sys_user_patient.transfer_state/transfer_cancel_reason/transfer_cancelled_at`。
2. `transfer_cancelled_by` 由鉴权主体补齐并同事务写入 `sys_user_patient.transfer_cancelled_by`。

### 3.3.6 DELETE /api/v1/patients/{patient_id}/guardians/{user_id}

用途：移除家庭成员。

处理约束：
1. 若目标成员存在 PENDING_CONFIRM 的主监护转移请求，必须同事务取消。
2. 仅 PRIMARY_GUARDIAN 或 SUPERADMIN 可执行。

错误码：E_PRO_4032、E_PRO_4041、E_PRO_4044、E_PRO_4099、E_REQ_4001。

请求示例：
```http
DELETE /api/v1/patients/1001/guardians/6001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "patient_id":  "1001",
                 "removed_user_id":  "6001",
                 "relation_status":  "REVOKED",
                 "removed_at":  "2026-04-06T10:35:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4032",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_PRO_4032 | 权限不足或越权访问 |
| 404 | E_PRO_4041、E_PRO_4044 | 资源不存在或不可见 |
| 409 | E_PRO_4099 | 状态冲突或重复提交 |

落库语义：`relation_status=REVOKED` 映射 `sys_user_patient.relation_status`；`removed_at` 由该行 `updated_at` 回传。

### 3.3.7 GET /api/v1/users/lookup

用途：邀请前置用户查询（按手机号或账号名换取 user_id）。

权限：受保护接口，需 Authorization。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| phone | string | 条件必填 | 手机号；phone 与 account 二选一且最多一个 |
| account | string | 条件必填 | 账号名；phone 与 account 二选一且最多一个 |

响应 data：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| user_id | int64 | string | 可用于邀请接口 |
| display_name_masked | string | string | 脱敏显示名 |
| phone_masked | string | string | 脱敏手机号 |
| account_masked | string | string | 脱敏账号 |

错误码：E_GOV_4011、E_GOV_4030、E_PRO_4042、E_REQ_4001、E_REQ_4005。

请求示例：
```http
GET /api/v1/users/lookup?phone=demo&account=demo HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "user_id":  "2002",
                 "display_name_masked":  "王*",
                 "phone_masked":  "138****1234",
                 "account_masked":  "fam***02"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001、E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_PRO_4042 | 资源不存在或不可见 |
### 3.3.8 POST /api/v1/patients

用途：创建患者档案（创世接口）。

权限：FAMILY / ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| patient_name | string | 是 | 1-64，脱敏展示名 |
| gender | string | 是 | MALE/FEMALE/UNKNOWN |
| birthday | string | 是 | ISO-8601 日期 |
| blood_type | string | 否 | A/B/AB/O/UNKNOWN |
| chronic_diseases | array<string> | 否 | 慢病标签列表 |
| allergy_notes | string | 否 | <= 500 |
| avatar_url | string | 是 | 白名单域名；必须为近期正面免冠照片 |

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| patient_id | string | 新建患者 ID |
| relation_role | string | 默认 PRIMARY_GUARDIAN |

业务规则：
1. 必须在同事务完成 patient_profile 创建与 sys_user_patient 关系创建。
2. 若调用方为 FAMILY，默认与新患者建立 PRIMARY_GUARDIAN 关系。
3. avatar_url 落库映射 patient_profile.photo_url；禁止空字符串与空白值。

副作用事件：profile.created（若调用方为 FAMILY，附带 guardian.relation.created）。

错误码：E_PRO_4001、E_PRO_4002、E_PRO_4003、E_PRO_4004、E_PRO_4014、E_PRO_4091、E_REQ_4001。

请求示例：
```http
POST /api/v1/patients HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "patient_name":  "demo",
    "gender":  "demo",
    "birthday":  "demo",
    "avatar_url":  "https://cdn.example.com/avatar/p1001.png"
}
```

响应示例（成功）：
```http
HTTP/1.1 201
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "patient_id":  "1001",
                 "relation_role":  "PRIMARY_GUARDIAN"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 400
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4001",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 201 | OK | 请求成功 |
| 400 | E_PRO_4001、E_PRO_4002、E_PRO_4003、E_PRO_4004、E_PRO_4014、E_REQ_4001 | 参数或格式不合法 |
| 409 | E_PRO_4091 | 状态冲突或重复提交 |
### 3.3.9 PUT /api/v1/patients/{patient_id}

用途：更新患者档案。

权限：PRIMARY_GUARDIAN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| patient_name | string | 否 | 1-64 |
| gender | string | 否 | MALE/FEMALE/UNKNOWN |
| birthday | string | 否 | ISO-8601 日期 |
| blood_type | string | 否 | A/B/AB/O/UNKNOWN |
| chronic_diseases | array<string> | 否 | 慢病标签列表 |
| allergy_notes | string | 否 | <= 500 |
| avatar_url | string | 否 | 白名单域名；不传则保持原值，禁止传 null/空字符串清空 |

业务规则：
1. 更新成功需提升 profile_version。
2. 触发 profile.updated 事件，供向量重建流水线消费。
3. 不允许将既有头像清空；仅允许替换为新的合法白名单地址。

错误码：E_PRO_4030、E_PRO_4041、E_PRO_4001、E_PRO_4002、E_PRO_4003、E_PRO_4004、E_PRO_4014、E_REQ_4001、E_REQ_4005。

请求示例：
```http
PUT /api/v1/patients/1001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{

}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "patient_id":  "1001",
                 "profile_version":  "9",
                 "updated_at":  "2026-04-06T10:40:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_PRO_4001、E_PRO_4002、E_PRO_4003、E_PRO_4004、E_PRO_4014、E_REQ_4001、E_REQ_4005 | 参数或格式不合法 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_PRO_4041 | 资源不存在或不可见 |
### 3.3.10 PUT /api/v1/patients/{patient_id}/fence

用途：更新患者围栏配置（fence.breached 判定前置能力）。

权限：PRIMARY_GUARDIAN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| fence_enabled | bool | 是 | true/false |
| fence_center | object | 条件必填 | fence_enabled=true 时必填，包含 lat/lng |
| fence_radius_m | int32 | 条件必填 | fence_enabled=true 时必填，范围 50-5000 |

fence_center 子字段：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| lat | number | 是 | 合法范围 |
| lng | number | 是 | 合法范围 |

业务规则：
1. 当 fence_enabled=true 时，fence_center 与 fence_radius_m 必须同时存在。
2. 当 fence_enabled=false 时，fence_center 与 fence_radius_m 必须置空。

错误码：E_PRO_4030、E_PRO_4041、E_PRO_4005、E_PRO_4221、E_REQ_4001、E_REQ_4005。

请求示例：
```http
PUT /api/v1/patients/1001/fence HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "fence_enabled":  true,
    "fence_radius_m":  500,
    "fence_center":  {
                         "lat":  23.123456,
                         "lng":  23.123456
                     }
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "patient_id":  "1001",
                 "fence_enabled":  true,
                 "fence_center":  {
                                      "lat":  31.2304,
                                      "lng":  121.4737
                                  },
                 "fence_radius_m":  500,
                 "updated_at":  "2026-04-06T10:42:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_PRO_4005、E_REQ_4001、E_REQ_4005 | 参数或格式不合法 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_PRO_4041 | 资源不存在或不可见 |
| 422 | E_PRO_4221 | 业务前置条件不满足 |
### 3.3.11 GET /api/v1/patients/{patient_id}

用途：读取患者完整档案。

权限：FAMILY（需患者授权）/ ADMIN / SUPERADMIN。

错误码：E_GOV_4011、E_PRO_4030、E_PRO_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/patients/1001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "patient_id":  "1001",
                 "patient_name":  "王某某",
                 "gender":  "MALE",
                 "birthday":  "1948-02-15",
                 "blood_type":  "A",
                 "lost_status":  "NORMAL",
                 "fence_enabled":  true,
                 "fence_center":  {
                                      "lat":  31.2304,
                                      "lng":  121.4737
                                  },
                 "fence_radius_m":  500,
                 "medical_history":  {
                                         "chronic_diseases":  [
                                                                  "HYPERTENSION"
                                                              ],
                                         "allergy_notes":  "青霉素过敏"
                                     },
                 "profile_version":  "9",
                 "updated_at":  "2026-04-06T10:42:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_PRO_4041 | 资源不存在或不可见 |
### 3.3.12 GET /api/v1/patients/{patient_id}/guardians/invitations

用途：读取患者监护邀请记录。

权限：FAMILY（需主监护管理权限）/ ADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| status | string | 否 | PENDING/ACCEPTED/REJECTED/EXPIRED/REVOKED |

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| invite_id | int64 | string | 邀请 ID |
| invitee_user_id | int64 | string | 被邀请人 |
| relation_role | string | string | GUARDIAN |
| status | string | string | PENDING/ACCEPTED/REJECTED/EXPIRED/REVOKED |
| expire_at | string | string | 过期时间（ISO-8601） |
| created_at | string | string | 创建时间（ISO-8601） |

错误码：E_GOV_4011、E_PRO_4032、E_PRO_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/patients/1001/guardians/invitations?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "invite_id":  "3001",
                                   "invitee_user_id":  "2002",
                                   "relation_role":  "GUARDIAN",
                                   "status":  "PENDING",
                                   "expire_at":  "2026-04-08T10:00:00Z",
                                   "created_at":  "2026-04-06T10:00:00Z"
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_PRO_4032 | 权限不足或越权访问 |
| 404 | E_PRO_4041 | 资源不存在或不可见 |
### 3.3.13 GET /api/v1/patients/{patient_id}/guardians/transfers

用途：读取主监护转移申请列表。

权限：FAMILY（需主监护管理权限）/ ADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| transfer_state | string | 否 | NONE/PENDING_CONFIRM/ACCEPTED/REJECTED/CANCELLED/EXPIRED |

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| transfer_request_id | string | string | 转移请求号 |
| from_user_id | int64 | string | 发起人 |
| to_user_id | int64 | string | 受让人 |
| transfer_state | string | string | NONE/PENDING_CONFIRM/ACCEPTED/REJECTED/CANCELLED/EXPIRED |
| requested_at | string | string | 发起时间（ISO-8601） |
| expire_at | string | string | 过期时间（ISO-8601） |

字段语义：`transfer_state` 在 API 口径中始终返回非空；当不存在转移请求时返回 `NONE`。

错误码：E_GOV_4011、E_PRO_4032、E_PRO_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/patients/1001/guardians/transfers?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "transfer_request_id":  "4001",
                                   "from_user_id":  "2001",
                                   "to_user_id":  "2002",
                                   "transfer_state":  "PENDING_CONFIRM",
                                   "requested_at":  "2026-04-06T10:00:00Z",
                                   "expire_at":  "2026-04-07T10:00:00Z"
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_PRO_4032 | 权限不足或越权访问 |
| 404 | E_PRO_4041 | 资源不存在或不可见 |
### 3.3.14 GET /api/v1/patients/{patient_id}/fence

用途：读取患者围栏配置。

权限：FAMILY（需患者授权）/ ADMIN。

错误码：E_GOV_4011、E_PRO_4030、E_PRO_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/patients/1001/fence HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "patient_id":  "1001",
                 "fence_enabled":  true,
                 "fence_center":  {
                                      "lat":  31.2304,
                                      "lng":  121.4737
                                  },
                 "fence_radius_m":  500,
                 "updated_at":  "2026-04-06T10:42:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_PRO_4041 | 资源不存在或不可见 |
### 3.3.15 GET /api/v1/patients/{patient_id}/guardians/invitations/{invite_id}

用途：读取单条邀请详情。

权限：FAMILY（需主监护管理权限）/ ADMIN。

错误码：E_GOV_4011、E_PRO_4032、E_PRO_4041、E_PRO_4043、E_REQ_4005。

请求示例：
```http
GET /api/v1/patients/1001/guardians/invitations/3001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "invite_id":  "3001",
                 "patient_id":  "1001",
                 "inviter_user_id":  "2001",
                 "invitee_user_id":  "2002",
                 "relation_role":  "GUARDIAN",
                 "status":  "PENDING",
                 "reason":  "请协助共同监护",
                 "expire_at":  "2026-04-08T10:00:00Z",
                 "created_at":  "2026-04-06T10:00:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_PRO_4032 | 权限不足或越权访问 |
| 404 | E_PRO_4041、E_PRO_4043 | 资源不存在或不可见 |
### 3.3.16 GET /api/v1/patients/{patient_id}/guardians/transfers/{transfer_request_id}

用途：读取单条主监护转移申请详情。

权限：FAMILY（需主监护管理权限）/ ADMIN。

错误码：E_GOV_4011、E_PRO_4032、E_PRO_4041、E_PRO_4045、E_REQ_4005。

请求示例：
```http
GET /api/v1/patients/1001/guardians/transfers/4001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "transfer_request_id":  "4001",
                 "patient_id":  "1001",
                 "from_user_id":  "2001",
                 "to_user_id":  "2002",
                 "transfer_state":  "PENDING_CONFIRM",
                 "reason":  "临时出差，需家属接管",
                 "requested_at":  "2026-04-06T10:00:00Z",
                 "expire_at":  "2026-04-07T10:00:00Z",
                 "confirmed_at":  null
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_PRO_4032 | 权限不足或越权访问 |
| 404 | E_PRO_4041、E_PRO_4045 | 资源不存在或不可见 |
## 3.4 物资运营域（含标签协作流程）

### 3.4.1 POST /api/v1/material/orders

用途：创建标签申领工单。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| patient_id | int64 | 是 | 必须存在且有监护授权 |
| quantity | int32 | 是 | 1-20 |
| apply_note | string | 否 | <= 256 |
| delivery_address | string | 是 | 10-512 |

错误码：E_PRO_4030、E_PRO_4041、E_MAT_4001、E_MAT_4002、E_REQ_4001。

副作用事件：material.order.created。

请求示例：
```http
POST /api/v1/material/orders HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "patient_id":  "1001",
    "quantity":  1,
    "delivery_address":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 201
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "order_id":  "5001",
                 "status":  "PENDING"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 201 | OK | 请求成功 |
| 400 | E_MAT_4001、E_MAT_4002、E_REQ_4001 | 参数或格式不合法 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_PRO_4041 | 资源不存在或不可见 |
### 3.4.2 GET /api/v1/material/resources/{resource_token}

用途：内部资源链解析，返回患者与物资基础信息。

路径参数：resource_token（32-1024，Base64URL）。

错误码：E_MAT_4002、E_MAT_4223、E_MAT_5003。

请求示例：
```http
GET /api/v1/material/resources/rtk_demo_001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "patient_id":  "1001",
                 "patient_name_masked":  "王**",
                 "tag_code":  "TAG000001",
                 "tag_type":  "QR_CODE",
                 "order_id":  "5001",
                 "status":  "PROCESSING",
                 "resource_expires_at":  "2026-04-06T12:00:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 400
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_MAT_4002",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_MAT_4002 | 参数或格式不合法 |
| 422 | E_MAT_4223 | 业务前置条件不满足 |
| 500 | E_MAT_5003 | 内部处理失败或审计回滚 |
### 3.4.3 POST /api/v1/patients/{patient_id}/tags/bind

用途：在授权范围内完成标签绑定。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| tag_code | string | 是 | 8-100，字母数字 |
| resource_token | string | 是 | 32-1024，Base64URL |
| bind_source | string | 否 | APP / MINI_PROGRAM |
| scanned_at | string | 否 | ISO-8601 |

绑定校验：
1. order.tag_code == payload.tag_code == scanned_tag_code。
2. protected_payload.patient_id == path.patient_id。
3. 当前用户具备 patient_id 监护授权。
4. applicant_user_id 不一致可协同绑定但必须写高危审计。

副作用事件：tag.bound；若订单 SHIPPED 则触发 order.auto_confirm.on_bind。

事件收敛说明：material-service 消费 tag.bound 后，内部执行与 3.4.14（家属签收）等效的状态机收敛逻辑，将对应订单从 SHIPPED 推进到 COMPLETED；该流程为事件驱动内部动作，不新增外部接口。

错误码：E_PRO_4030、E_MAT_4223、E_MAT_4096、E_MAT_4097、E_MAT_4032、E_MAT_4092、E_REQ_4001。

请求示例：
```http
POST /api/v1/patients/1001/tags/bind HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "tag_code":  "TAG000001",
    "resource_token":  "token_demo_001"
}
```

响应示例（成功）：
```http
HTTP/1.1 201
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "order_id":  "5001",
                 "patient_id":  "1001",
                 "tag_code":  "TAG000001",
                 "status":  "BOUND",
                 "bound_at":  "2026-04-06T11:05:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_PRO_4030、E_MAT_4032 | 权限不足或越权访问 |
| 409 | E_MAT_4096、E_MAT_4097、E_MAT_4092 | 状态冲突或重复提交 |
| 422 | E_MAT_4223 | 业务前置条件不满足 |
### 3.4.4 POST /api/v1/material/orders/{order_id}/cancel

用途：家属发起订单取消。

权限：FAMILY（仅原申请人可操作）。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| cancel_reason | string | 是 | 5-256 |

状态机约束：
1. PENDING -> CANCELLED（同步收敛）。
2. PROCESSING -> CANCEL_PENDING（进入审核取消态）。
3. SHIPPED/CANCELLED 不允许取消。

副作用约束：状态变更与审计日志同事务提交，必要的跨域通知由既有基线 Topic 编排处理，不新增公开 Topic 名称。

错误码：E_MAT_4003、E_MAT_4030、E_MAT_4041、E_MAT_4094、E_REQ_4001、E_REQ_4005。

请求示例：
```http
POST /api/v1/material/orders/5001/cancel HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "cancel_reason":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "order_id":  "5001",
                 "status":  "CANCEL_PENDING",
                 "cancel_reason":  "demo",
                 "updated_at":  "2026-04-06T11:15:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 400
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_MAT_4003",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_MAT_4003、E_REQ_4001、E_REQ_4005 | 参数或格式不合法 |
| 403 | E_MAT_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4041 | 资源不存在或不可见 |
| 409 | E_MAT_4094 | 状态冲突或重复提交 |
### 3.4.5 POST /api/v1/patients/{patient_id}/tags/{tag_code}/lost

用途：家属主动挂失标签，触发紧急模式链路。

权限：FAMILY（需具备 patient_id 监护授权）。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| lost_reason | string | 否 | <= 256 |

状态机约束：
1. 仅允许 BOUND -> LOST。
2. 非 BOUND 状态提交必须拒绝。

副作用约束：标签状态更新后，扫码入口按基线规则切换到紧急模式页面。

错误码：E_PRO_4030、E_MAT_4004、E_MAT_4044、E_MAT_4098、E_REQ_4005。

请求示例：
```http
POST /api/v1/patients/1001/tags/TAG000001/lost HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{

}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "tag_code":  "TAG000001",
                 "patient_id":  "1001",
                 "status":  "LOST",
                 "lost_at":  "2026-04-06T11:18:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_MAT_4004、E_REQ_4005 | 参数或格式不合法 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4044 | 资源不存在或不可见 |
| 409 | E_MAT_4098 | 状态冲突或重复提交 |
### 3.4.6 POST /api/v1/admin/tags/{tag_code}/void

用途：管理员作废标签（物流异常、泄露风险等）。

落库语义：`status/void_reason/void_at` 分别映射 `tag_asset.status/tag_asset.void_reason/tag_asset.void_at`。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| void_reason | string | 是 | 5-256 |

状态机约束：
1. 允许 ALLOCATED/BOUND -> VOID。
2. 其他状态提交必须拒绝。

副作用约束：标签作废后必须阻断匿名上传入口，仅允许管理员纠错流程恢复。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| tag_code | string | 标签编码 |
| status | string | VOID |
| void_reason | string | 作废原因 |
| void_at | string | 作废时间（ISO-8601） |

错误码：E_GOV_4030、E_MAT_4005、E_MAT_4044、E_MAT_4098、E_REQ_4001。

请求示例：
```http
POST /api/v1/admin/tags/TAG000001/void HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "void_reason":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "tag_code":  "TAG000001",
                 "status":  "VOID",
                 "void_reason":  "demo",
                 "void_at":  "2026-04-06T11:25:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_MAT_4005、E_REQ_4001 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4044 | 资源不存在或不可见 |
| 409 | E_MAT_4098 | 状态冲突或重复提交 |
### 3.4.7 PUT /api/v1/admin/material/orders/{order_id}/approve

用途：管理员审核通过申领工单。

落库语义：`status/approved_at` 映射 `tag_apply_record.status/tag_apply_record.approved_at`。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| approve_note | string | 否 | <= 256 |

状态机约束：
1. 仅允许 PENDING -> PROCESSING。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| order_id | string | 工单 ID |
| status | string | PROCESSING |
| approved_at | string | 审核通过时间（ISO-8601） |

错误码：E_GOV_4030、E_MAT_4041、E_MAT_4091、E_REQ_4001。

请求示例：
```http
PUT /api/v1/admin/material/orders/5001/approve HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{

}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "order_id":  "5001",
                 "status":  "PROCESSING",
                 "approved_at":  "2026-04-06T11:26:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4041 | 资源不存在或不可见 |
| 409 | E_MAT_4091 | 状态冲突或重复提交 |
### 3.4.8 PUT /api/v1/admin/material/orders/{order_id}/cancel/approve

用途：管理员审核通过取消申请。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| approve_reason | string | 否 | <= 256 |

状态机约束：
1. 仅允许 CANCEL_PENDING -> CANCELLED。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| order_id | string | 工单 ID |
| status | string | CANCELLED |
| approved_at | string | 审核通过时间（ISO-8601） |

错误码：E_GOV_4030、E_MAT_4041、E_MAT_4094、E_REQ_4001。

落库语义：`status/approved_at` 映射 `tag_apply_record.status/tag_apply_record.approved_at`，且 `status=CANCELLED` 时同事务写入 `tag_apply_record.closed_at`。

请求示例：
```http
PUT /api/v1/admin/material/orders/5001/cancel/approve HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{

}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "order_id":  "5001",
                 "status":  "CANCELLED",
                 "approved_at":  "2026-04-06T11:27:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4041 | 资源不存在或不可见 |
| 409 | E_MAT_4094 | 状态冲突或重复提交 |
### 3.4.9 PUT /api/v1/admin/material/orders/{order_id}/cancel/reject

用途：管理员驳回取消申请。

落库语义：`status/reject_reason/rejected_at` 映射 `tag_apply_record.status/tag_apply_record.reject_reason/tag_apply_record.rejected_at`。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| reject_reason | string | 是 | 5-256，落库 tag_apply_record.reject_reason |

状态机约束：
1. 仅允许 CANCEL_PENDING -> PROCESSING。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| order_id | string | 工单 ID |
| status | string | PROCESSING |
| reject_reason | string | 驳回原因 |
| rejected_at | string | 驳回时间（ISO-8601） |

错误码：E_GOV_4030、E_MAT_4041、E_MAT_4095、E_REQ_4001。

请求示例：
```http
PUT /api/v1/admin/material/orders/5001/cancel/reject HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "reject_reason":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "order_id":  "5001",
                 "status":  "PROCESSING",
                 "reject_reason":  "demo",
                 "rejected_at":  "2026-04-06T11:28:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4041 | 资源不存在或不可见 |
| 409 | E_MAT_4095 | 状态冲突或重复提交 |
### 3.4.10 PUT /api/v1/admin/material/orders/{order_id}/ship

用途：管理员执行发货。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| tag_code | string | 是 | 必须存在且可锁定 |
| tracking_number | string | 是 | 6-64 |
| courier_name | string | 否 | <= 64 |

状态机约束：
1. 仅允许 PROCESSING -> SHIPPED。
2. 发货事务必须同时完成：tag_asset UNBOUND -> ALLOCATED、写入 resource_link。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| order_id | string | 工单 ID |
| status | string | SHIPPED |
| tag_code | string | 发货绑定标签 |
| tracking_number | string | 物流单号 |
| shipped_at | string | 发货时间（ISO-8601） |

错误码：E_GOV_4030、E_MAT_4041、E_MAT_4091、E_MAT_4222、E_REQ_4001。

请求示例：
```http
PUT /api/v1/admin/material/orders/5001/ship HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "tracking_number":  "demo",
    "tag_code":  "TAG000001"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "order_id":  "5001",
                 "status":  "SHIPPED",
                 "tag_code":  "TAG000001",
                 "tracking_number":  "SF1234567890",
                 "shipped_at":  "2026-04-06T11:29:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4041 | 资源不存在或不可见 |
| 409 | E_MAT_4091 | 状态冲突或重复提交 |
| 422 | E_MAT_4222 | 业务前置条件不满足 |
### 3.4.11 PUT /api/v1/admin/material/orders/{order_id}/logistics-exception

用途：管理员标记物流异常。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| exception_desc | string | 是 | 5-256 |
| evidence_urls | array<string> | 否 | 白名单域名 |

状态机约束：
1. 仅允许 SHIPPED -> EXCEPTION。
2. 同步执行作废处置（tag.void.by_admin -> VOID）。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| order_id | string | 工单 ID |
| status | string | EXCEPTION |
| exception_desc | string | 异常说明 |
| processed_at | string | 处理时间（ISO-8601） |

错误码：E_GOV_4030、E_MAT_4041、E_MAT_4091、E_MAT_4224、E_REQ_4001。

请求示例：
```http
PUT /api/v1/admin/material/orders/5001/logistics-exception HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "exception_desc":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "order_id":  "5001",
                 "status":  "EXCEPTION",
                 "exception_desc":  "demo",
                 "processed_at":  "2026-04-06T11:30:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4041 | 资源不存在或不可见 |
| 409 | E_MAT_4091 | 状态冲突或重复提交 |
| 422 | E_MAT_4224 | 业务前置条件不满足 |
### 3.4.12 PUT /api/v1/admin/material/orders/{order_id}/reship

用途：管理员执行异常补发。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| new_tag_code | string | 是 | 必须为 UNBOUND 可分配标签 |
| tracking_number | string | 是 | 6-64 |
| reship_reason | string | 是 | 5-256 |

状态机约束：
1. 仅允许 EXCEPTION -> PROCESSING。
2. 必须重新分配新标签并重建 resource_link。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| order_id | string | 工单 ID |
| status | string | PROCESSING |
| new_tag_code | string | 补发标签编码 |
| tracking_number | string | 新物流单号 |
| reshipped_at | string | 补发处理时间（ISO-8601） |

错误码：E_GOV_4030、E_MAT_4041、E_MAT_4091、E_REQ_4001。

请求示例：
```http
PUT /api/v1/admin/material/orders/5001/reship HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "reship_reason":  "demo",
    "tracking_number":  "demo",
    "new_tag_code":  "TAG000001"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "order_id":  "5001",
                 "status":  "PROCESSING",
                 "new_tag_code":  "TAG000001",
                 "tracking_number":  "SF1234567891",
                 "reshipped_at":  "2026-04-06T11:31:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4041 | 资源不存在或不可见 |
| 409 | E_MAT_4091 | 状态冲突或重复提交 |
### 3.4.13 PUT /api/v1/admin/material/orders/{order_id}/close-exception

用途：管理员异常关闭工单。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| exception_desc | string | 是 | 5-256 |

状态机约束：
1. 仅允许 EXCEPTION -> CANCELLED。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| order_id | string | 工单 ID |
| status | string | CANCELLED |
| closed_at | string | 异常关闭时间（ISO-8601） |

错误码：E_GOV_4030、E_MAT_4041、E_MAT_4091、E_REQ_4001。

请求示例：
```http
PUT /api/v1/admin/material/orders/5001/close-exception HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "exception_desc":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "order_id":  "5001",
                 "status":  "CANCELLED",
                 "closed_at":  "2026-04-06T11:32:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4041 | 资源不存在或不可见 |
| 409 | E_MAT_4091 | 状态冲突或重复提交 |
### 3.4.14 POST /api/v1/material/orders/{order_id}/confirm

用途：家属签收确认。

权限：FAMILY（工单关联家属）。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| confirm_note | string | 否 | <= 256 |

状态机约束：
1. 仅允许 SHIPPED -> COMPLETED。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| order_id | string | 工单 ID |
| status | string | COMPLETED |
| confirmed_at | string | 签收时间（ISO-8601） |

错误码：E_GOV_4030、E_MAT_4041、E_MAT_4092、E_REQ_4001。

落库语义：`status/confirmed_at` 映射 `tag_apply_record.status/tag_apply_record.closed_at`。

请求示例：
```http
POST /api/v1/material/orders/5001/confirm HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{

}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "order_id":  "5001",
                 "status":  "COMPLETED",
                 "confirmed_at":  "2026-04-06T11:33:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4041 | 资源不存在或不可见 |
| 409 | E_MAT_4092 | 状态冲突或重复提交 |
### 3.4.15 POST /api/v1/admin/tags/import

用途：管理员批量入库标签（TAG.IMPORT）。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| batch_no | string | 是 | 1-64 |
| tags | array<object> | 是 | 至少 1 条，单批建议 <= 500 |

tags 元素结构：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| tag_code | string | 是 | 8-100，唯一 |
| tag_type | string | 是 | QR_CODE/NFC |

结果语义：导入成功标签默认 status=UNBOUND。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| batch_no | string | 导入批次号 |
| total_count | number | 提交总量 |
| success_count | number | 成功导入数量 |
| failed_count | number | 失败数量 |

错误码：E_GOV_4030、E_PRO_4092、E_REQ_4001。

请求示例：
```http
POST /api/v1/admin/tags/import HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "tags":  [
                 "demo"
             ],
    "batch_no":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "batch_no":  "BATCH20260406",
                 "total_count":  100,
                 "success_count":  100,
                 "failed_count":  0
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 409 | E_PRO_4092 | 状态冲突或重复提交 |
### 3.4.16 POST /api/v1/admin/tags/{tag_code}/reset

用途：管理员重置标签（VOID/LOST -> UNBOUND）。

落库语义：`status/reset_at` 映射 `tag_asset.status/tag_asset.reset_at`。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| reason | string | 是 | 5-256 |

状态机约束：
1. 仅允许 VOID -> UNBOUND 或 LOST -> UNBOUND。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| tag_code | string | 标签编码 |
| status | string | UNBOUND |
| reset_at | string | 重置时间（ISO-8601） |

错误码：E_PRO_4031、E_MAT_4044、E_PRO_4093、E_REQ_4001。

请求示例：
```http
POST /api/v1/admin/tags/TAG000001/reset HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "reason":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "tag_code":  "TAG000001",
                 "status":  "UNBOUND",
                 "reset_at":  "2026-04-06T11:34:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4031",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_PRO_4031 | 权限不足或越权访问 |
| 404 | E_MAT_4044 | 资源不存在或不可见 |
| 409 | E_PRO_4093 | 状态冲突或重复提交 |
### 3.4.17 POST /api/v1/admin/tags/{tag_code}/recover

用途：管理员恢复标签（LOST -> BOUND）。

落库语义：`status/patient_id/recovered_at` 映射 `tag_asset.status/tag_asset.patient_id/tag_asset.recovered_at`。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| patient_id | int64 | 是 | 必须与实物核验归属一致 |
| reason | string | 是 | 5-256 |

状态机约束：
1. 仅允许 LOST -> BOUND。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| tag_code | string | 标签编码 |
| patient_id | string | 归属患者 ID |
| status | string | BOUND |
| recovered_at | string | 恢复时间（ISO-8601） |

错误码：E_PRO_4031、E_MAT_4044、E_PRO_4093、E_REQ_4001、E_REQ_4005。

请求示例：
```http
POST /api/v1/admin/tags/TAG000001/recover HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "patient_id":  "1001",
    "reason":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "tag_code":  "TAG000001",
                 "patient_id":  "1001",
                 "status":  "BOUND",
                 "recovered_at":  "2026-04-06T11:35:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4031",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001、E_REQ_4005 | 参数或格式不合法 |
| 403 | E_PRO_4031 | 权限不足或越权访问 |
| 404 | E_MAT_4044 | 资源不存在或不可见 |
| 409 | E_PRO_4093 | 状态冲突或重复提交 |
### 3.4.18 GET /api/v1/material/orders

用途：家属侧分页读取工单列表。

权限：FAMILY。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| status | string | 否 | PENDING/PROCESSING/CANCEL_PENDING/SHIPPED/EXCEPTION/COMPLETED/CANCELLED |
| patient_id | int64 | 否 | 按患者过滤 |

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| order_id | int64 | string | 工单 ID |
| patient_id | int64 | string | 患者 ID |
| quantity | int32 | number | 数量 |
| status | string | string | PENDING/PROCESSING/CANCEL_PENDING/SHIPPED/EXCEPTION/COMPLETED/CANCELLED |
| created_at | string | string | 创建时间（ISO-8601） |
| updated_at | string | string | 更新时间（ISO-8601） |

错误码：E_GOV_4011、E_MAT_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/material/orders?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "order_id":  "5001",
                                   "patient_id":  "1001",
                                   "quantity":  1,
                                   "status":  "PROCESSING",
                                   "created_at":  "2026-04-06T10:00:00Z",
                                   "updated_at":  "2026-04-06T11:00:00Z"
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_MAT_4030 | 权限不足或越权访问 |
### 3.4.19 GET /api/v1/material/orders/{order_id}

用途：家属侧读取工单详情。

权限：FAMILY（需患者授权）。

错误码：E_GOV_4011、E_MAT_4030、E_MAT_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/material/orders/5001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "order_id":  "5001",
                 "order_no":  "ORD202604060001",
                 "patient_id":  "1001",
                 "quantity":  1,
                 "status":  "PROCESSING",
                 "delivery_address":  "上海市浦东新区世纪大道100号",
                 "tracking_number":  "SF1234567890",
                 "resource_link":  "https://app.example.com/r/rtk_demo_001",
                 "created_at":  "2026-04-06T10:00:00Z",
                 "updated_at":  "2026-04-06T10:30:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_MAT_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4041 | 资源不存在或不可见 |
### 3.4.20 GET /api/v1/admin/material/orders/{order_id}

用途：管理端读取工单详情。

权限：ADMIN / SUPERADMIN。

错误码：E_GOV_4011、E_GOV_4030、E_MAT_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/material/orders/5001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "order_id":  "5001",
                 "order_no":  "ORD202604060001",
                 "patient_id":  "1001",
                 "applicant_user_id":  "2001",
                 "tag_code":  "TAG000001",
                 "quantity":  1,
                 "status":  "SHIPPED",
                 "delivery_address":  "上海市浦东新区世纪大道100号",
                 "tracking_number":  "SF1234567890",
                 "cancel_reason":  null,
                 "exception_desc":  null,
                 "created_at":  "2026-04-06T10:00:00Z",
                 "updated_at":  "2026-04-06T11:00:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4041 | 资源不存在或不可见 |
### 3.4.21 GET /api/v1/admin/material/orders/{order_id}/timeline

用途：管理端读取工单状态流转轨迹。

权限：ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| cursor | string | 否 | 透明游标；优先使用 |
| page_size | int32 | 否 | 1-200，默认 50 |

分页规则：
1. 默认按 created_at DESC, timeline_id DESC 返回。
2. 存在 cursor 时按 Cursor 模式滚动翻页，响应 data 返回 next_cursor。
3. 工单状态流转轨迹存储于 sys_log（module='MATERIAL_ORDER'），timeline_id 对应 sys_log.id。
4. cursor 基于 (created_at, id) 复合排序编码，其中 id 即 timeline_id。
5. `from_status/to_status` 必须来自 `sys_log.detail.from_status/sys_log.detail.to_status` 键契约。

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| timeline_id | int64 | string | 轨迹 ID |
| from_status | string | string | 变更前状态 |
| to_status | string | string | 变更后状态 |
| operator_user_id | int64 | string | 操作人 ID |
| remark | string | string | 备注（可空） |
| created_at | string | string | 记录时间（ISO-8601） |

错误码：E_GOV_4011、E_GOV_4030、E_MAT_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/material/orders/5001/timeline?cursor=eyJjcmVhdGVkX2F0IjoiMjAyNi0wNC0wNlQxMDowMDowMFoiLCJpZCI6IjUwMDEifQ%3D%3D&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "timeline_id":  "9201",
                                   "from_status":  "PENDING",
                                   "to_status":  "PROCESSING",
                                   "operator_user_id":  "3001",
                                   "remark":  "审核通过",
                                   "created_at":  "2026-04-06T11:07:00Z"
                               }
                           ],
                 "page_size":  20,
                 "next_cursor":  "eyJ0aW1lbGluZV9pZCI6IjkyMDEifQ==",
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4041 | 资源不存在或不可见 |
### 3.4.22 GET /api/v1/admin/tags

用途：管理端分页查询标签台账。

权限：ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| status | string | 否 | UNBOUND/ALLOCATED/BOUND/LOST/VOID |
| patient_id | int64 | 否 | 按患者过滤 |

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| tag_code | string | string | 标签编码 |
| tag_type | string | string | QR_CODE/NFC |
| status | string | string | UNBOUND/ALLOCATED/BOUND/LOST/VOID |
| patient_id | int64 | string | 患者 ID（可空） |
| updated_at | string | string | 更新时间（ISO-8601） |

错误码：E_GOV_4011、E_GOV_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/tags?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "tag_code":  "TAG000001",
                                   "tag_type":  "QR_CODE",
                                   "status":  "BOUND",
                                   "patient_id":  "1001",
                                   "updated_at":  "2026-04-06T11:08:00Z"
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
### 3.4.23 GET /api/v1/admin/tags/{tag_code}

用途：管理端读取标签详情。

权限：ADMIN / SUPERADMIN。

错误码：E_GOV_4011、E_GOV_4030、E_MAT_4044。

请求示例：
```http
GET /api/v1/admin/tags/TAG000001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "tag_code":  "TAG000001",
                 "tag_type":  "QR_CODE",
                 "status":  "BOUND",
                 "patient_id":  "1001",
                 "order_id":  "5001",
                 "batch_no":  "BATCH20260406",
                 "last_bound_at":  "2026-04-06T11:05:00Z",
                 "updated_at":  "2026-04-06T11:05:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4044 | 资源不存在或不可见 |
### 3.4.24 GET /api/v1/material/orders/{order_id}/tracking

用途：家属侧读取物流轨迹（预留接口）。

毕设精简说明：
1. 当前版本暂不展示物流轨迹。
2. 不落地 `logistics_tracking_event` 独立存储。

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 403 | E_MAT_4030 | 毕设版本暂不开放 |
### 3.4.25 GET /api/v1/material/orders/{order_id}/resource-link

用途：读取工单资源链状态。

resource_link 语义：
1. 统一入口 URL，格式为 `https://<domain>/r/{resource_token}`。
2. 当前用于二维码直达；NFC 后续上线时复用同一 URL 语义，不新增并行入口协议。

落库语义：`resource_link` 映射 `tag_apply_record.resource_link`；`resource_token_expire_at` 与 `status` 由令牌服务运行时计算回传，不作为持久化状态列。

权限：FAMILY（需患者授权）。

错误码：E_GOV_4011、E_MAT_4030、E_MAT_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/material/orders/5001/resource-link HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "order_id":  "5001",
                 "resource_link":  "https://app.example.com/r/rtk_demo_001",
                 "resource_token_expire_at":  "2026-04-13T10:00:00Z",
                 "status":  "ACTIVE"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_MAT_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4041 | 资源不存在或不可见 |
### 3.4.26 POST /api/v1/admin/tags/{tag_code}/allocate

用途：管理员手工分配标签。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| order_id | int64 | 是 | 目标工单 ID |
| reason | string | 否 | 0-256 |

错误码：E_GOV_4011、E_GOV_4030、E_MAT_4044、E_MAT_4098、E_REQ_4001、E_REQ_4005。

请求示例：
```http
POST /api/v1/admin/tags/TAG000001/allocate HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "order_id":  "1001"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "tag_code":  "TAG000001",
                 "order_id":  "5001",
                 "status":  "ALLOCATED",
                 "allocated_at":  "2026-04-06T10:40:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001、E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4044 | 资源不存在或不可见 |
| 409 | E_MAT_4098 | 状态冲突或重复提交 |
### 3.4.27 POST /api/v1/admin/tags/{tag_code}/release

用途：管理员释放已分配标签。

权限：ADMIN / SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| reason | string | 是 | 5-256 |

错误码：E_GOV_4011、E_GOV_4030、E_MAT_4044、E_MAT_4098、E_REQ_4001、E_REQ_4005。

请求示例：
```http
POST /api/v1/admin/tags/TAG000001/release HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "reason":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "tag_code":  "TAG000001",
                 "status":  "UNBOUND",
                 "released_at":  "2026-04-06T10:45:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001、E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4044 | 资源不存在或不可见 |
| 409 | E_MAT_4098 | 状态冲突或重复提交 |
### 3.4.28 GET /api/v1/patients/{patient_id}/tags/{tag_code}

用途：读取患者维度标签详情。

权限：FAMILY（需患者授权）/ ADMIN。

错误码：E_GOV_4011、E_PRO_4030、E_MAT_4044、E_REQ_4005。

请求示例：
```http
GET /api/v1/patients/1001/tags/TAG000001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "tag_code":  "TAG000001",
                 "patient_id":  "1001",
                 "status":  "BOUND",
                 "bind_time":  "2026-04-06T11:05:00Z",
                 "last_location":  {
                                       "lat":  31.2304,
                                       "lng":  121.4737,
                                       "reported_at":  "2026-04-06T11:10:00Z"
                                   },
                 "updated_at":  "2026-04-06T11:10:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4044 | 资源不存在或不可见 |
### 3.4.29 GET /api/v1/patients/{patient_id}/tags/{tag_code}/history

用途：读取标签历史流转轨迹。

权限：FAMILY（需患者授权）/ ADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |

轨迹来源约束：`from_status/to_status` 必须来自 `sys_log.detail.from_status/sys_log.detail.to_status` 键契约。

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| history_id | int64 | string | 历史记录 ID |
| from_status | string | string | 变更前状态 |
| to_status | string | string | 变更后状态 |
| operator_user_id | int64 | string | 操作人 |
| reason | string | string | 变更原因（可空） |
| created_at | string | string | 记录时间（ISO-8601） |

错误码：E_GOV_4011、E_PRO_4030、E_MAT_4044、E_REQ_4005。

请求示例：
```http
GET /api/v1/patients/1001/tags/TAG000001/history?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "history_id":  "9301",
                                   "from_status":  "ALLOCATED",
                                   "to_status":  "BOUND",
                                   "operator_user_id":  "2001",
                                   "reason":  "扫码绑定",
                                   "created_at":  "2026-04-06T11:10:00Z"
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_MAT_4044 | 资源不存在或不可见 |
## 3.5 AI 协同决策域

### 3.5.1 POST /api/v1/ai/sessions

用途：初始化并创建 AI 会话（进入 AI 协同页面第一步）。

权限：FAMILY（需具备 patient_id 监护授权）。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| patient_id | int64 | 是 | 患者 ID（JSON 线传 string） |
| task_id | int64 | 否 | 关联任务 ID（JSON 线传 string） |

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| session_id | string | 服务端生成的全局唯一会话 ID |

业务规则：
1. 必须校验当前用户对 patient_id 的监护授权。
2. 创建时在 ai_session 中落首条记录，messages 初始化为 []。
3. 会话 patient_id/task_id 由服务端固化，后续消息接口禁止客户端改写。

错误码：E_AI_4002、E_PRO_4030、E_AI_4292、E_AI_4293、E_REQ_4001、E_REQ_4005。

请求示例：
```http
POST /api/v1/ai/sessions HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "patient_id":  "1001"
}
```

响应示例（成功）：
```http
HTTP/1.1 201
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "session_id":  "ais_20260406_0001"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 400
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_AI_4002",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（限流）：
```http
HTTP/1.1 429
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
Retry-After: 30
X-RateLimit-Remaining: 0
```
```json
{
    "message":  "error",
    "code":  "E_AI_4292",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 201 | OK | 请求成功 |
| 400 | E_AI_4002、E_REQ_4001、E_REQ_4005 | 参数或格式不合法 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 429 | E_AI_4292、E_AI_4293 | 频率或配额限制触发 |
### 3.5.2 GET /api/v1/ai/sessions

用途：分页查询当前用户可访问的 AI 会话列表。

权限：FAMILY（需具备相关 patient_id 授权）。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| patient_id | int64 | 否 | 按患者过滤（JSON 线传 string） |

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| session_id | string | string | 会话 ID |
| patient_id | int64 | string | 患者 ID |
| task_id | int64 | string | 关联任务 ID（可空） |
| model_name | string | string | 模型标识 |
| token_used_total | int64 | string | 总消耗 token |
| round_count | int32 | number | 对话轮数 |
| status | string | string | ACTIVE/ARCHIVED（会话生命周期状态） |
| updated_at | string | string | 最后活跃时间（ISO-8601） |

语义说明：
1. `status` 表示会话生命周期状态（ACTIVE/ARCHIVED）。
2. `status` 与 3.5.3 的 `stream_status` 解耦，后者仅表示单次消息流式过程态。

错误码：E_PRO_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/ai/sessions?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "session_id":  "ais_20260406_0001",
                                   "patient_id":  "1001",
                                   "task_id":  "1001",
                                   "model_name":  "qwen-max-latest",
                                   "token_used_total":  "1820",
                                   "round_count":  6,
                                   "status":  "ACTIVE",
                                   "updated_at":  "2026-04-06T11:12:00Z"
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
### 3.5.3 POST /api/v1/ai/sessions/{session_id}/messages

用途：提交 AI 对话消息并创建流式回复任务（ACK-only）。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| prompt | string | 是 | 1-4000 |

请求头扩展：

| Header | 必填 | 规则 |
| :--- | :---: | :--- |
| Accept | 否 | `application/json` 或 `text/event-stream` |
| X-Stream-Transport | 否 | `SSE` / `WEBSOCKET`，默认 `SSE` |

强约束：
1. 禁止客户端传 patient_id / task_id。
2. 服务端按 session_id 反查上下文。
3. 先过 user_id 与 patient_id 双限流。
4. 调用前完成双账本配额预占。
5. messages 必须原子追加或 CAS 更新，禁止先查后改。
6. 接口不得阻塞等待模型全量完成：必须在 2 秒内返回 JSON ACK（202）或建立 SSE 首包（200）。
7. 回复正文必须经流式通道下发（SSE/WS）；`assistant_reply` 不得作为唯一返回载体。
8. 反向代理必须关闭响应缓冲并配置长连接超时（例如 `X-Accel-Buffering: no`），避免流式响应被超时截断。
9. `tool_intent.action` 必须命中 9.7 白名单；未知 action 或未授权 action 必须阻断并返回策略拒绝结果。

成功响应 data（JSON ACK 模式）：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| session_id | string | 会话 ID |
| message_id | string | 本次消息 ID |
| accepted | boolean | 是否已受理 |
| stream_transport | string | SSE/WEBSOCKET |
| stream_channel | string | 流式通道标识（SSE 路径或 WS topic） |
| stream_status | string | QUEUED/STREAMING |
| version | string | 会话版本号 |

字段语义：
1. `stream_status` 仅表示本次消息流式过程态。
2. 会话生命周期状态统一使用 `ai_session.status`（ACTIVE/ARCHIVED）。

流式完成事件（`event=done`）中返回：
- `token_usage.prompt_tokens/completion_tokens/total_tokens`
- `token_usage.model_name/billing_source/provider_request_id`（可选）
- `fallback_response.mode`（可空）
- `intent_suggestions`（可选，函数调用建议卡片）
- `action_id/result_code/executed_at`（当本次消息完成受控动作执行时必返）

错误码：E_AI_4001、E_AI_4033、E_AI_4041、E_AI_4292、E_AI_4293、E_AI_4091。

请求示例：
```http
POST /api/v1/ai/sessions/7001/messages HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json
Accept: application/json
X-Stream-Transport: SSE

{
    "prompt":  "demo"
}
```

响应示例（成功，JSON ACK）：
```http
HTTP/1.1 202
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "session_id":  "ais_20260406_0001",
                 "message_id":  "aim_20260406_0001",
                 "accepted":  true,
                 "stream_transport":  "SSE",
                 "stream_channel":  "/api/v1/ai/sessions/7001/messages/aim_20260406_0001/stream",
                 "stream_status":  "QUEUED",
                 "version":  "13"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（成功，SSE 流式）：
```http
POST /api/v1/ai/sessions/7001/messages HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json
Accept: text/event-stream
```
```text
event: ack
data: {"session_id":"ais_20260406_0001","message_id":"aim_20260406_0001","stream_status":"STREAMING"}

event: delta
data: {"index":1,"content":"已根据最新线索"}

event: tool_intent
data: {"intent_id":"int_20260407_0001","action":"propose_close","confirm_required":true}

event: done
data: {
  "token_usage":{"prompt_tokens":320,"completion_tokens":180,"total_tokens":500},
  "fallback_response":null,
    "action_id":"act_20260406_0001",
    "result_code":"OK",
    "executed_at":"2026-04-06T11:00:03Z",
  "version":"13"
}
```

响应示例（失败）：
```http
HTTP/1.1 400
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_AI_4001",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（限流）：
```http
HTTP/1.1 429
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
Retry-After: 30
X-RateLimit-Remaining: 0
```
```json
{
    "message":  "error",
    "code":  "E_AI_4293",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | SSE 流建立成功 |
| 202 | OK | 消息已受理（JSON ACK） |
| 400 | E_AI_4001 | 参数或格式不合法 |
| 403 | E_AI_4033 | 权限不足或越权访问 |
| 404 | E_AI_4041 | 资源不存在或不可见 |
| 409 | E_AI_4091 | 状态冲突或重复提交 |
| 429 | E_AI_4292、E_AI_4293 | 频率或配额限制触发 |

#### 3.5.3A AI 流式传输降级约定

1. 默认传输方案为 SSE（`Accept: text/event-stream`）。
2. 若客户端网络环境不支持 SSE，可切换 `X-Stream-Transport=WEBSOCKET`，并通过 3.6.7 获取 `ws_ticket` 建立 WS 连接。
3. WS 模式下建议订阅逻辑主题 `ai.session.{session_id}.message.{message_id}`，事件类型保持 `ack/delta/tool_intent/done/error` 一致。
4. 当 SSE/WS 均不可用时，允许短期退化到 `GET /api/v1/ai/sessions/{session_id}/messages` 轮询，但仅作为容灾，不得作为默认实现。
### 3.5.4 GET /api/v1/admin/ai/sessions

用途：管理端审计分页查询系统 AI 会话。

权限：ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| patient_id | int64 | 否 | 按患者过滤（JSON 线传 string） |
| user_id | int64 | 否 | 按会话归属用户过滤（JSON 线传 string） |

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| session_id | string | string | 会话 ID |
| user_id | int64 | string | 会话归属用户 |
| patient_id | int64 | string | 患者 ID |
| model_name | string | string | 模型标识 |
| token_used_total | int64 | string | 总消耗 token |
| round_count | int32 | number | 对话轮数 |
| status | string | string | ACTIVE/ARCHIVED（会话生命周期状态） |
| updated_at | string | string | 最后活跃时间（ISO-8601） |

语义说明：`status` 表示会话生命周期状态（ACTIVE/ARCHIVED），与 3.5.3 的 `stream_status`（单次消息流式过程态）解耦。

错误码：E_GOV_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/ai/sessions?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "session_id":  "ais_20260406_0001",
                                   "user_id":  "2001",
                                   "patient_id":  "1001",
                                   "model_name":  "qwen-max-latest",
                                   "token_used_total":  "1820",
                                   "round_count":  6,
                                   "status":  "ACTIVE",
                                   "updated_at":  "2026-04-06T11:12:00Z"
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
### 3.5.5 POST /api/v1/patients/{patient_id}/memory-notes

用途：家属补充患者非结构化记忆（习惯、常去地点、识别线索等）。

落库语义：原始结构化记忆写入 `patient_memory_note`，向量化异步写入 `vector_store`。

权限：FAMILY（需具备 patient_id 授权）。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| kind | string | 是 | HABIT/PLACE/PREFERENCE/SAFETY_CUE |
| content | string | 是 | 5-2000 |
| tags | array<string> | 否 | 语义标签 |

业务规则：
1. 写入后需触发 memory.appended 流水线事件。
2. 敏感信息必须脱敏后入库。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| note_id | string | 记忆条目 ID |
| patient_id | string | 患者 ID |
| kind | string | HABIT/PLACE/PREFERENCE/SAFETY_CUE |
| created_at | string | 创建时间（ISO-8601） |

错误码：E_AI_4003、E_PRO_4030、E_PRO_4041、E_REQ_4001、E_REQ_4005。

请求示例：
```http
POST /api/v1/patients/1001/memory-notes HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "kind":  "demo",
    "content":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 201
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "note_id":  "mn_20260406_0001",
                 "patient_id":  "1001",
                 "kind":  "HABIT",
                 "created_at":  "2026-04-06T11:36:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 400
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_AI_4003",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 201 | OK | 请求成功 |
| 400 | E_AI_4003、E_REQ_4001、E_REQ_4005 | 参数或格式不合法 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_PRO_4041 | 资源不存在或不可见 |
### 3.5.6 GET /api/v1/patients/{patient_id}/memory-notes

用途：分页读取患者记忆条目（用于 AI 上下文预览与维护）。

读取语义：本接口读取 `patient_memory_note` 原始条目，不直接读取向量库。

权限：FAMILY（需具备 patient_id 授权）/ ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| kind | string | 否 | HABIT/PLACE/PREFERENCE/SAFETY_CUE |

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| note_id | string | string | 记忆条目 ID |
| kind | string | string | HABIT/PLACE/PREFERENCE/SAFETY_CUE |
| content | string | string | 记忆内容（脱敏） |
| tags | array<string> | array | 语义标签 |
| created_at | string | string | 创建时间（ISO-8601） |

错误码：E_PRO_4030、E_PRO_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/patients/1001/memory-notes?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "note_id":  "mn_20260406_0001",
                                   "kind":  "HABIT",
                                   "content":  "常去小区南门广场散步",
                                   "tags":  [
                                                "WALKING"
                                            ],
                                   "created_at":  "2026-04-06T11:13:00Z"
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_PRO_4041 | 资源不存在或不可见 |
### 3.5.7 GET /api/v1/ai/sessions/{session_id}/messages

用途：分页读取会话消息（用户侧）。

权限：FAMILY / ADMIN（需会话授权）。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| before_time | string | 否 | ISO-8601 |

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| message_id | string | string | 消息 ID |
| role | string | string | user/assistant/system |
| content | string | string | 消息内容 |
| timestamp | string | string | 消息时间（ISO-8601） |
| token_used | int32 | number | 本条 token 消耗（可空） |

错误码：E_GOV_4011、E_AI_4033、E_AI_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/ai/sessions/7001/messages?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "message_id":  "aim_20260406_0001",
                                   "role":  "assistant",
                                   "content":  "建议优先排查东侧地铁口。",
                                   "timestamp":  "2026-04-06T11:14:00Z",
                                   "token_used":  500
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_AI_4033 | 权限不足或越权访问 |
| 404 | E_AI_4041 | 资源不存在或不可见 |
### 3.5.8 GET /api/v1/admin/ai/sessions/{session_id}

用途：管理端读取 AI 会话详情。

权限：ADMIN / SUPERADMIN。

错误码：E_GOV_4011、E_GOV_4030、E_AI_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/ai/sessions/7001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "session_id":  "ais_20260406_0001",
                 "user_id":  "2001",
                 "patient_id":  "1001",
                 "task_id":  "1001",
                 "model_name":  "qwen-max-latest",
                 "status":  "ACTIVE",
                 "round_count":  6,
                 "token_used_total":  "1820",
                 "created_at":  "2026-04-06T10:00:00Z",
                 "updated_at":  "2026-04-06T11:12:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_AI_4041 | 资源不存在或不可见 |
### 3.5.9 GET /api/v1/admin/ai/sessions/{session_id}/messages

用途：管理端分页读取会话消息审计视图。

权限：ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| include_deleted | bool | 否 | 是否包含逻辑删除消息 |

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| message_id | string | string | 消息 ID |
| role | string | string | user/assistant/system |
| content | string | string | 消息内容 |
| timestamp | string | string | 消息时间（ISO-8601） |
| is_deleted | bool | boolean | 是否逻辑删除 |
| deleted_at | string | string | 删除时间（ISO-8601，可空） |

错误码：E_GOV_4011、E_GOV_4030、E_AI_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/ai/sessions/7001/messages?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "message_id":  "aim_20260406_0001",
                                   "role":  "assistant",
                                   "content":  "建议优先排查东侧地铁口。",
                                   "timestamp":  "2026-04-06T11:14:00Z",
                                   "is_deleted":  false,
                                   "deleted_at":  null
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_AI_4041 | 资源不存在或不可见 |
### 3.5.10 GET /api/v1/ai/sessions/{session_id}/quota

用途：读取会话配额视图。

权限：FAMILY / ADMIN（需会话授权）。

错误码：E_GOV_4011、E_AI_4033、E_AI_4041、E_AI_4293、E_REQ_4005。

请求示例：
```http
GET /api/v1/ai/sessions/7001/quota HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "session_id":  "ais_20260406_0001",
                 "user_quota_daily":  "20000",
                 "user_used_daily":  "1820",
                 "patient_quota_daily":  "50000",
                 "patient_used_daily":  "7460",
                 "remaining_tokens":  "18180",
                 "reset_at":  "2026-04-07T00:00:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_AI_4033 | 权限不足或越权访问 |
| 404 | E_AI_4041 | 资源不存在或不可见 |
| 429 | E_AI_4293 | 频率或配额限制触发 |
### 3.5.11 POST /api/v1/ai/sessions/{session_id}/archive

用途：会话归档/失效处理。

协议说明：为规避 DELETE 请求体在代理/CDN/旧客户端链路中可能被丢弃的兼容性风险，该接口固定采用 POST 动作语义。

权限：FAMILY / ADMIN（需会话授权）。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| reason | string | 否 | 0-256 |

归档语义：归档动作必须更新 ai_session.status=ARCHIVED 并写 archived_at。

错误码：E_GOV_4011、E_AI_4033、E_AI_4041、E_AI_4091、E_REQ_4001、E_REQ_4005。

请求示例：
```http
POST /api/v1/ai/sessions/7001/archive HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{

}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "session_id":  "ais_20260406_0001",
                 "status":  "ARCHIVED",
                 "archived_at":  "2026-04-06T11:15:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001、E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_AI_4033 | 权限不足或越权访问 |
| 404 | E_AI_4041 | 资源不存在或不可见 |
| 409 | E_AI_4091 | 状态冲突或重复提交 |
## 3.6 查询编排层（跨域只读视图，非新增领域）

### 3.6.1 GET /api/v1/patients/{patient_id}/profile

用途：获取患者脱敏档案信息（家属端/管理端展示用）。

权限：
1. FAMILY：仅可访问具备监护授权的患者。
2. ADMIN/SUPERADMIN：可按管理权限访问。

响应 data：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| patient_id | int64 | string | 患者 ID |
| patient_name_masked | string | string | 脱敏姓名 |
| gender | string | string | MALE/FEMALE/UNKNOWN |
| age | int32 | number | 年龄 |
| blood_type | string | string | 血型（可空） |
| chronic_diseases | array<string> | array | 慢病标签列表 |
| allergy_notes | string | string | 过敏说明（脱敏后） |
| avatar_url | string | string | 脱敏头像地址 |
| lost_status | string | string | NORMAL/MISSING |

错误码：E_PRO_4030、E_PRO_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/patients/1001/profile HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "patient_id":  "1001",
                 "patient_name_masked":  "王**",
                 "gender":  "MALE",
                 "age":  78,
                 "blood_type":  "A",
                 "chronic_diseases":  [
                                          "HYPERTENSION"
                                      ],
                 "allergy_notes":  "青霉素过敏",
                 "avatar_url":  "https://cdn.example.com/avatar/p1001.png",
                 "lost_status":  "NORMAL"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_PRO_4041 | 资源不存在或不可见 |
### 3.6.2 GET /api/v1/admin/clues/suspected

用途：管理员获取待复核可疑线索列表（clue.suspected 工单池）。

权限：ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| task_id | int64 | 否 | 按任务过滤 |
| patient_id | int64 | 否 | 按患者过滤 |
| review_status | string | 否 | 默认 PENDING（仅 suspect_flag=true 的复核池记录），支持 PENDING/OVERRIDDEN/REJECTED |

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| clue_id | int64 | string | 线索 ID |
| task_id | int64 | string | 关联任务 ID |
| patient_id | int64 | string | 关联患者 ID |
| location | object | object | 标准化坐标点 |
| risk_score | number | number | 可疑评分 |
| suspect_reason | string | string | 可疑原因摘要 |
| reported_at | string | string | 上报时间（ISO-8601） |
| review_status | string | string | PENDING/OVERRIDDEN/REJECTED |

错误码：E_GOV_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/clues/suspected?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "clue_id":  "2001",
                                   "task_id":  "1001",
                                   "patient_id":  "1001",
                                   "location":  {
                                                    "lat":  31.2304,
                                                    "lng":  121.4737
                                                },
                                   "risk_score":  0.92,
                                   "suspect_reason":  "位置跳变异常",
                                   "reported_at":  "2026-04-06T11:05:00Z",
                                   "review_status":  "PENDING"
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
### 3.6.3 GET /api/v1/admin/material/orders

用途：管理员查询物资申领工单列表（含待处理工单）。

权限：ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| patient_id | int64 | 否 | 按患者过滤 |
| applicant_user_id | int64 | 否 | 按申请人过滤 |
| status | string | 否 | 逗号分隔状态；默认 PENDING,PROCESSING；支持 PENDING/PROCESSING/CANCEL_PENDING/SHIPPED/EXCEPTION/COMPLETED/CANCELLED |

兼容说明：`order_status` 作为兼容别名保留（Deprecated），建议统一使用 `status`。

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| order_id | int64 | string | 工单 ID |
| patient_id | int64 | string | 患者 ID |
| applicant_user_id | int64 | string | 申请人 ID |
| quantity | int32 | number | 申领数量 |
| status | string | string | PENDING/PROCESSING/CANCEL_PENDING/SHIPPED/EXCEPTION/COMPLETED/CANCELLED |
| apply_note | string | string | 申请备注 |
| created_at | string | string | 创建时间（ISO-8601） |

错误码：E_GOV_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/material/orders?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "order_id":  "5001",
                                   "patient_id":  "1001",
                                   "applicant_user_id":  "2001",
                                   "quantity":  1,
                                   "status":  "PROCESSING",
                                   "apply_note":  "尽快发货",
                                   "created_at":  "2026-04-06T10:00:00Z"
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
### 3.6.4 GET /api/v1/ai/sessions/{session_id}

用途：拉取 AI 会话历史上下文（进入 AI 协同页面首屏加载）。

权限：受保护接口，需会话访问授权。

响应 data：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| session_id | string | string | 会话 ID |
| patient_id | int64 | string | 患者 ID |
| task_id | int64 | string | 关联任务 ID（可空） |
| version | int64 | string | 会话版本（用于并发控制） |
| status | string | string | ACTIVE/ARCHIVED（会话生命周期状态） |
| archived_at | string | string | 归档时间（ISO-8601，可空） |
| messages | array<object> | array | 会话消息数组（按时间正序） |
| updated_at | string | string | 最新更新时间（ISO-8601） |

语义说明：
1. `status` 表示会话生命周期状态（ACTIVE/ARCHIVED）。
2. `status` 与 3.5.3 的 `stream_status` 解耦，后者仅表示单次消息流式过程态。

messages 元素结构：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| role | string | user/assistant/system |
| content | string | 消息内容 |
| timestamp | string | ISO-8601 |

错误码：E_AI_4001、E_AI_4033、E_AI_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/ai/sessions/7001 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "session_id":  "ais_20260406_0001",
                 "patient_id":  "1001",
                 "task_id":  "1001",
                 "version":  "13",
                 "status":  "ACTIVE",
                 "archived_at":  null,
                 "messages":  [
                                  {
                                      "role":  "user",
                                      "content":  "最近的线索有没有新变化？",
                                      "timestamp":  "2026-04-06T11:00:00Z"
                                  },
                                  {
                                      "role":  "assistant",
                                      "content":  "有新增定位点，建议向东侧路口排查。",
                                      "timestamp":  "2026-04-06T11:00:03Z"
                                  }
                              ],
                 "updated_at":  "2026-04-06T11:00:03Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 400
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_AI_4001",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_AI_4001、E_REQ_4005 | 参数或格式不合法 |
| 403 | E_AI_4033 | 权限不足或越权访问 |
| 404 | E_AI_4041 | 资源不存在或不可见 |
### 3.6.5 GET /api/v1/patients/{patient_id}/guardians

用途：查询患者当前监护关系列表（家属端“监护协同管理”页面渲染）。

权限：
1. FAMILY：需具备 patient_id 授权。
2. ADMIN/SUPERADMIN：按管理权限访问。

响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| patient_id | string | 患者 ID |
| guardians | array<object> | 成员列表 |

guardians 元素结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| user_id | int64 | string | 成员用户 ID |
| display_name_masked | string | string | 脱敏姓名 |
| relation_role | string | string | PRIMARY_GUARDIAN/GUARDIAN |
| relation_status | string | string | PENDING/ACTIVE/REVOKED |
| invited_at | string | string | 邀请时间（ISO-8601，可空） |

错误码：E_PRO_4030、E_PRO_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/patients/1001/guardians HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "patient_id":  "1001",
                 "guardians":  [
                                   {
                                       "user_id":  "2001",
                                       "display_name_masked":  "王*",
                                       "relation_role":  "PRIMARY_GUARDIAN",
                                       "relation_status":  "ACTIVE",
                                       "invited_at":  "2026-04-01T09:00:00Z"
                                   }
                               ]
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_PRO_4041 | 资源不存在或不可见 |
### 3.6.6 GET /api/v1/patients/{patient_id}/tags

用途：查询患者已绑定标签列表（含短码映射），用于家属端展示与挂失入口。

权限：
1. FAMILY：需具备 patient_id 授权。
2. ADMIN/SUPERADMIN：按管理权限访问。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| status | string | 否 | 默认 BOUND；支持 BOUND/LOST/VOID |

命名说明：
1. 查询参数 `status` 用于按标签状态过滤。
2. 响应字段 `tag_status` 与过滤语义同源，映射 `tag_asset.status`；保留 `tag_status` 命名用于避免与工单 `status` 语义混淆。
3. `lost_at` 映射 `tag_asset.lost_at`。

响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| patient_id | string | 患者 ID |
| tags | array<object> | 标签列表 |

tags 元素结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| tag_code | string | string | 标签编码 |
| tag_status | string | string | UNBOUND/ALLOCATED/BOUND/LOST/VOID |
| short_code | string | string | 对外短码（可公开时返回） |
| bound_at | string | string | 绑定时间（ISO-8601，可空） |
| lost_at | string | string | 挂失时间（ISO-8601，可空） |

错误码：E_PRO_4030、E_PRO_4041、E_REQ_4005。

请求示例：
```http
GET /api/v1/patients/1001/tags HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "patient_id":  "1001",
                 "tags":  [
                              {
                                  "tag_code":  "TAG000001",
                                  "tag_status":  "BOUND",
                                  "short_code":  "P8K2M1",
                                  "bound_at":  "2026-04-06T11:05:00Z",
                                  "lost_at":  null
                              }
                          ]
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_PRO_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 403 | E_PRO_4030 | 权限不足或越权访问 |
| 404 | E_PRO_4041 | 资源不存在或不可见 |
### 3.6.7 POST /api/v1/ws/tickets

用途：签发浏览器 WebSocket 握手用短效一次性票据。

权限：受保护接口，需 Authorization。

请求体：无。

响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| ws_ticket | string | 一次性票据，默认 TTL 60s |
| expires_in_seconds | int32 | 剩余有效期 |
| ws_endpoint | string | 建议连接地址模板：/api/v1/ws/notifications?ticket={ws_ticket} |

错误码：E_GOV_4011、E_GOV_4012、E_REQ_4001。

请求示例：
```http
POST /api/v1/ws/tickets HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "ws_ticket":  "wst_20260406_0001",
                 "expires_in_seconds":  60,
                 "ws_endpoint":  "/api/v1/ws/notifications?ticket=wst_20260406_0001"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 401 | E_GOV_4011、E_GOV_4012 | 鉴权失败或凭据缺失/失效 |
## 3.7 身份权限与治理域：身份认证与基础用户接口（Auth & User）

### 3.7.1 POST /api/v1/auth/register

用途：家属用户注册账号。

权限：匿名可用。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| username | string | 是 | 4-64，字母数字下划线 |
| password | string | 是 | 8-64，必须满足密码强度策略 |
| display_name | string | 否 | 1-64 |
| phone | string | 否 | 合法手机号格式 |

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| user_id | string | 新用户 ID |
| role | string | 默认 FAMILY |

错误码：E_AUTH_4001、E_AUTH_4002、E_AUTH_4091、E_REQ_4001。

请求示例：
```http
POST /api/v1/auth/register HTTP/1.1
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "password":  "demo",
    "username":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 201
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "user_id":  "7001",
                 "role":  "FAMILY"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 400
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_AUTH_4001",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 201 | OK | 请求成功 |
| 400 | E_AUTH_4001、E_AUTH_4002、E_REQ_4001 | 参数或格式不合法 |
| 409 | E_AUTH_4091 | 状态冲突或重复提交 |
### 3.7.2 POST /api/v1/auth/login

用途：账号密码登录并签发 JWT 令牌。

权限：匿名可用。

请求体：

```json
{
  "username": "string",
  "password": "string"
}
```

登录安全约束：
1. 密码明文仅允许在 TLS 链路上传输。
2. 登录成功后返回 JWT 与过期时间，网关写入在线会话缓存。

成功响应 data：

```json
{
  "token": "string",
  "expires_in": 7200,
  "user": {
    "user_id": "string",
    "role": "string"
  }
}
```

错误码：E_GOV_4011、E_GOV_4031、E_REQ_4001。

请求示例：
```http
POST /api/v1/auth/login HTTP/1.1
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{

}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "token":  "jwt_demo_token_001",
                 "expires_in":  7200,
                 "user":  {
                              "user_id":  "7001",
                              "role":  "FAMILY"
                          }
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4031 | 权限不足或越权访问 |
### 3.7.3 POST /api/v1/auth/logout

用途：注销登录并使当前 JWT 失效。

权限：受保护接口。

业务规则：
1. 网关将当前 token 的 jti 写入黑名单，TTL 至 token 自然过期。
2. 注销后当前 token 不得再次访问任意受保护接口。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| revoked_token_jti | string | 被回收的令牌标识 |
| logged_out_at | string | 注销时间（ISO-8601） |

错误码：E_GOV_4011、E_REQ_4001。

请求示例：
```http
POST /api/v1/auth/logout HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "revoked_token_jti":  "jti_20260406_0001",
                 "logged_out_at":  "2026-04-06T11:40:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
### 3.7.4 PUT /api/v1/users/me/password

用途：用户自行修改登录密码。

权限：受保护接口。

请求体：

```json
{
  "old_password": "string",
  "new_password": "string"
}
```

业务规则：
1. 新旧密码不能相同。
2. 后端更新 sys_user.password 时必须执行 BCrypt 重新哈希。
3. 更新成功后必须强制下线当前 Token（写入黑名单）。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| user_id | string | 当前用户 ID |
| password_updated_at | string | 密码更新时间（ISO-8601） |
| token_revoked | boolean | 当前令牌是否已失效 |

错误码：E_USR_4001、E_USR_4002、E_USR_4011、E_GOV_4011、E_REQ_4001。

请求示例：
```http
PUT /api/v1/users/me/password HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{

}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "user_id":  "2001",
                 "password_updated_at":  "2026-04-06T11:41:00Z",
                 "token_revoked":  true
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 400
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_USR_4001",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_USR_4001、E_USR_4002、E_REQ_4001 | 参数或格式不合法 |
| 401 | E_USR_4011、E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
## 3.8 身份权限与治理域：管理端用户治理接口（Admin User Management）

### 3.8.1 GET /api/v1/admin/users

用途：分页查询系统用户列表及账号状态。

权限：ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| page_no | int32 | 否 | >=1，默认 1 |
| page_size | int32 | 否 | 1-100，默认 20 |
| role | string | 否 | FAMILY/ADMIN/SUPERADMIN |
| status | string | 否 | NORMAL/BANNED |
| keyword | string | 否 | 用户名/昵称模糊检索 |

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| user_id | int64 | string | 用户 ID |
| username | string | string | 登录名 |
| role | string | string | 用户角色 |
| status | string | string | NORMAL/BANNED |
| last_login_at | string | string | 最近登录时间（ISO-8601，可空） |
| created_at | string | string | 创建时间（ISO-8601） |

错误码：E_GOV_4030、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/users?page_no=1&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "user_id":  "6001",
                                   "username":  "family_demo",
                                   "role":  "FAMILY",
                                   "status":  "NORMAL",
                                   "last_login_at":  "2026-04-06T10:20:00Z",
                                   "created_at":  "2026-03-20T09:00:00Z"
                               }
                           ],
                 "page_size":  20,
                 "total":  1,
                 "page_no":  1,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
### 3.8.2 PUT /api/v1/admin/users/{user_id}/status

用途：封禁/解封用户账号（BR-GOV-001）。

权限：ADMIN / SUPERADMIN。

请求体：

```json
{
  "status": "BANNED",
  "reason": "string"
}
```

业务规则：
1. status 仅允许 NORMAL/BANNED。
2. 当状态置为 BANNED 时，网关必须拒绝该用户后续所有受保护访问。
3. 状态切换成功后必须强制注销该用户已签发 JWT（写黑名单或会话失效）。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| user_id | string | 目标用户 ID |
| status | string | NORMAL/BANNED |
| updated_at | string | 更新时间（ISO-8601） |

错误码：E_USR_4003、E_USR_4041、E_GOV_4030、E_REQ_4001、E_REQ_4005。

请求示例：
```http
PUT /api/v1/admin/users/6001/status HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{

}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "user_id":  "6001",
                 "status":  "BANNED",
                 "updated_at":  "2026-04-06T11:42:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 400
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_USR_4003",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_USR_4003、E_REQ_4001、E_REQ_4005 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 404 | E_USR_4041 | 资源不存在或不可见 |
### 3.8.3 PUT /api/v1/admin/users/{user_id}/password:reset

用途：强制重置用户密码。

权限：仅 SUPERADMIN。

AI Agent 执行策略：
1. 本接口执行级别为 `MANUAL_ONLY`。
2. 若 `X-Action-Source=AI_AGENT`，网关必须拒绝并返回 `E_GOV_4231`。

请求体：

```json
{
  "new_password": "string",
  "reason": "string"
}
```

业务规则：
1. 仅当 X-User-Role=SUPERADMIN 才允许执行。
2. 重置密码属于高危操作，必须同事务写入 sys_log 审计记录。
3. 重置成功后，目标用户所有活跃 JWT 必须强制失效。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| user_id | string | 目标用户 ID |
| password_reset_at | string | 重置时间（ISO-8601） |
| force_relogin | boolean | 是否要求强制重新登录 |

错误码：E_GOV_4032、E_GOV_4231、E_USR_4002、E_USR_4041、E_REQ_4001、E_REQ_4005。

请求示例：
```http
PUT /api/v1/admin/users/6001/password:reset HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{

}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "user_id":  "6001",
                 "password_reset_at":  "2026-04-06T11:43:00Z",
                 "force_relogin":  true
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4032",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_USR_4002、E_REQ_4001、E_REQ_4005 | 参数或格式不合法 |
| 403 | E_GOV_4032 | 权限不足或越权访问 |
| 423 | E_GOV_4231 | 当前接口仅允许人工执行 |
| 404 | E_USR_4041 | 资源不存在或不可见 |
### 3.8.4 GET /api/v1/admin/logs

用途：管理员分页查询治理审计日志（HP-GOV-04）。

权限：ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| cursor | string | 否 | 透明游标；优先使用 |
| page_size | int32 | 否 | 1-200，默认 50 |
| page_no | int32 | 否 | 兼容字段（Deprecated）；仅浅分页建议 |
| module | string | 否 | 业务模块过滤 |
| action | string | 否 | 动作过滤 |
| user_id | int64 | 否 | 按操作者过滤 |
| trace_id | string | 否 | 链路过滤 |
| start_time | string | 否 | ISO-8601 |
| end_time | string | 否 | ISO-8601 |

分页规则：
1. 默认按 created_at DESC, log_id DESC 返回。
2. 存在 cursor 时按 Cursor 模式滚动翻页，响应 data 返回 next_cursor。
3. page_no 仅用于兼容旧客户端，不保证深度分页性能。

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| log_id | int64 | string | 审计日志 ID |
| module | string | string | 模块 |
| action | string | string | 动作 |
| operator_user_id | int64 | string | 操作人 |
| result | string | string | SUCCESS/FAIL |
| trace_id | string | string | 链路追踪 ID |
| created_at | string | string | 记录时间（ISO-8601） |

错误码：E_GOV_4030、E_GOV_4291、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/logs?cursor=eyJjcmVhdGVkX2F0IjoiMjAyNi0wNC0wNlQxMDowMDowMFoiLCJpZCI6IjEwMDAifQ%3D%3D&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "log_id":  "10001",
                                   "module":  "TASK",
                                   "action":  "FORCE_CLOSE",
                                   "operator_user_id":  "9001",
                                   "result":  "SUCCESS",
                                   "trace_id":  "trc_demo_20260406_001",
                                   "created_at":  "2026-04-06T11:15:00Z"
                               }
                           ],
                 "page_size":  20,
                 "next_cursor":  "eyJsb2dfaWQiOiIxMDAwMSJ9",
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 429
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
Retry-After: 30
X-RateLimit-Remaining: 0
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4291",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
| 429 | E_GOV_4291 | 网关限流，需按 Retry-After 退避 |
### 3.8.5 GET /api/v1/admin/dashboard/metrics

用途：获取运营治理看板聚合指标（HP-GOV-05）。

权限：ADMIN / SUPERADMIN。

数据来源：
1. 指标主数据来自可观测平台（如 Prometheus/时序后端），不直接全表扫描业务库。
2. `sys_log` 仅用于审计回查与口径校验，不作为实时聚合主链路。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| window | string | 否 | 1h/24h/7d/30d |

响应摘要：
1. 登录成功率与拒绝率。
2. 高危操作次数与失败率。
3. 关键接口 TP95/TP99 与错误码分布。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| window | string | 统计窗口 |
| login_success_rate | number | 登录成功率 |
| risk_operation_count | number | 高危操作次数 |
| tp95_ms | number | 接口 TP95（毫秒） |
| error_rate | number | 接口错误率 |

错误码：E_GOV_4030、E_REQ_4001。

请求示例：
```http
GET /api/v1/admin/dashboard/metrics HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "window":  "24h",
                 "login_success_rate":  0.983,
                 "risk_operation_count":  9,
                 "tp95_ms":  420,
                 "error_rate":  0.012
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4030",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_GOV_4030 | 权限不足或越权访问 |
### 3.8.6 POST /api/v1/admin/super/export-data

用途：超级管理员导出数据与审计报表（SUPER_EXPORT_DATA）。

权限：仅 SUPERADMIN。

落库语义（毕设精简）：不创建 `export_job` 独立表，导出请求与结果统一写入 `sys_log.detail`。

AI Agent 执行策略：
1. 本接口执行级别为 `MANUAL_ONLY`。
2. 若 `X-Action-Source=AI_AGENT`，网关必须拒绝并返回 `E_GOV_4231`。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| export_type | string | 是 | AUDIT_REPORT/OPS_METRICS |
| window_start | string | 是 | ISO-8601 |
| window_end | string | 是 | ISO-8601 |
| reason | string | 是 | 5-256 |

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| export_ref_id | string | 导出引用 ID（可取 X-Request-Id） |
| file_url | string | 导出文件地址（可空） |
| logged_at | string | 审计落库时间（ISO-8601） |

错误码：E_GOV_4032、E_REQ_4001、E_GOV_5002。

请求示例：
```http
POST /api/v1/admin/super/export-data HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "window_end":  "demo",
    "reason":  "demo",
    "window_start":  "demo",
    "export_type":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "export_ref_id":  "req_demo_20260406_001",
                 "file_url":  null,
                 "logged_at":  "2026-04-06T11:44:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4032",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_GOV_4032 | 权限不足或越权访问 |
| 500 | E_GOV_5002 | 内部处理失败或审计回滚 |
### 3.8.7 POST /api/v1/admin/super/logs/purge

用途：超级管理员执行过期审计日志清理（SUPER_PURGE_LOG）。

协议说明：为规避 DELETE 请求体在代理/CDN/旧客户端链路中可能被丢弃的兼容性风险，该接口固定采用 POST 动作语义。

权限：仅 SUPERADMIN。

AI Agent 执行策略：
1. 本接口执行级别为 `MANUAL_ONLY`。
2. 若 `X-Action-Source=AI_AGENT`，网关必须拒绝并返回 `E_GOV_4231`。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| before_time | string | 是 | ISO-8601 |
| reason | string | 是 | 5-256 |

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| purged_count | number | 清理记录数 |
| before_time | string | 清理阈值时间（ISO-8601） |
| purged_at | string | 执行时间（ISO-8601） |

错误码：E_GOV_4032、E_REQ_4001、E_GOV_5002。

请求示例：
```http
POST /api/v1/admin/super/logs/purge HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "before_time":  "2026-04-06T10:00:00Z",
    "reason":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "purged_count":  320,
                 "before_time":  "2026-04-06T10:00:00Z",
                 "purged_at":  "2026-04-06T11:45:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4032",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_GOV_4032 | 权限不足或越权访问 |
| 500 | E_GOV_5002 | 内部处理失败或审计回滚 |
### 3.8.8 PUT /api/v1/admin/super/config

用途：超级管理员修改全局阈值与治理策略（SUPER_CONFIG_CHANGE）。

持久化映射：写入 sys_config（按 config_key 更新 config_value），并同步写 sys_log 审计。

权限：仅 SUPERADMIN。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| config_key | string | 是 | 白名单键 |
| config_value | string | 是 | 与键类型匹配 |
| reason | string | 是 | 5-256 |

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| config_key | string | 已更新的配置键 |
| config_value | string | 已更新后的配置值 |
| scope | string | 配置作用域（public/ops/security/ai_policy） |
| updated_reason | string | 最近更新原因 |
| updated_at | string | 更新时间（ISO-8601） |

错误码：E_GOV_4032、E_REQ_4001、E_GOV_5002。

请求示例：
```http
PUT /api/v1/admin/super/config HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "config_key":  "demo",
    "config_value":  "demo",
    "reason":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "config_key":  "task.auto_close_minutes",
                 "config_value":  "30",
                 "scope":  "ops",
                 "updated_reason":  "demo",
                 "updated_at":  "2026-04-06T10:00:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4032",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_GOV_4032 | 权限不足或越权访问 |
| 500 | E_GOV_5002 | 内部处理失败或审计回滚 |
### 3.8.9 POST /api/v1/admin/super/rescue/tasks/{task_id}/force-close

用途：超级管理员强制关闭任务（SUPER_FORCE_CLOSE_TASK，管理端显式入口）。

权限：仅 SUPERADMIN。

AI Agent 执行策略：
1. 本接口执行级别为 `MANUAL_ONLY`。
2. 若 `X-Action-Source=AI_AGENT`，网关必须拒绝并返回 `E_GOV_4231`。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| reason | string | 是 | 5-256 |

约束说明：
1. 与 3.1.7 属同一治理动作；管理端入口用于后台操作闭环。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| task_id | string | 任务 ID |
| status | string | RESOLVED |
| closed_by | string | 操作人 ID |
| closed_at | string | 关闭时间（ISO-8601） |

错误码：E_GOV_4032、E_TASK_4041、E_TASK_4093、E_GOV_5002、E_REQ_4001。

请求示例：
```http
POST /api/v1/admin/super/rescue/tasks/1001/force-close HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "reason":  "demo"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "task_id":  "1001",
                 "status":  "RESOLVED",
                 "closed_by":  "9001",
                 "closed_at":  "2026-04-06T11:46:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4032",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 403 | E_GOV_4032 | 权限不足或越权访问 |
| 404 | E_TASK_4041 | 资源不存在或不可见 |
| 409 | E_TASK_4093 | 状态冲突或重复提交 |
| 500 | E_GOV_5002 | 内部处理失败或审计回滚 |
### 3.8.10 GET /api/v1/admin/metrics/security

用途：读取安全治理指标。

权限：ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| time_from | string | 否 | ISO-8601 |
| time_to | string | 否 | ISO-8601 |
| scope | string | 否 | summary/detail（detail 仅 SUPERADMIN） |

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| scope | string | 统计视角 |
| time_from | string | 开始时间（ISO-8601） |
| time_to | string | 结束时间（ISO-8601） |
| failed_login_count | number | 失败登录次数 |
| risk_operation_count | number | 高危操作次数 |
| banned_user_count | number | 封禁用户数 |
| captcha_trigger_count | number | CAPTCHA 触发次数 |

错误码：E_GOV_4011、E_GOV_4030、E_GOV_4032、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/metrics/security?time_from=2026-04-06T10%3A00%3A00Z&time_to=2026-04-06T10%3A00%3A00Z HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "scope":  "summary",
                 "time_from":  "2026-04-01T00:00:00Z",
                 "time_to":  "2026-04-06T23:59:59Z",
                 "failed_login_count":  14,
                 "risk_operation_count":  9,
                 "banned_user_count":  2,
                 "captcha_trigger_count":  21
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030、E_GOV_4032 | 权限不足或越权访问 |
### 3.8.11 GET /api/v1/admin/config

用途：读取全局配置快照。

读取映射：从 sys_config 读取按 scope 过滤后的配置快照。

权限：ADMIN / SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| scope | string | 否 | public/ops/security/ai_policy（security/ai_policy 仅 SUPERADMIN） |

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| scope | string | 返回结果所属作用域 |
| items | array<object> | 配置项列表 |
| items[].config_key | string | 配置键 |
| items[].config_value | string | 配置值 |
| items[].updated_at | string | 最近更新时间（ISO-8601） |

错误码：E_GOV_4011、E_GOV_4030、E_GOV_4032、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/config HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "scope":  "public",
                 "items":  [
                               {
                                   "config_key":  "risk.level.threshold",
                                   "config_value":  "70",
                                   "updated_at":  "2026-04-06T10:00:00Z"
                               }
                           ]
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4030、E_GOV_4032 | 权限不足或越权访问 |
### 3.8.12 GET /api/v1/admin/super/outbox/dead

用途：分页查询 Outbox DEAD 队列，用于故障诊断与人工干预前检查。

权限：仅 SUPERADMIN。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| cursor | string | 否 | 透明游标；优先使用 |
| page_size | int32 | 否 | 1-100，默认 20 |
| topic | string | 否 | 目标 Topic 前缀匹配 |
| partition_key | string | 否 | 分区键精确匹配 |
| from_time | string | 否 | ISO-8601 |
| to_time | string | 否 | ISO-8601 |

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| items | array<object> | DEAD 事件列表 |
| items[].event_id | string | 事件 ID |
| items[].created_at | string | 事件创建时间（ISO-8601） |
| items[].topic | string | 目标 Topic |
| items[].partition_key | string | 分区键 |
| items[].retry_count | number | 当前重试次数 |
| items[].last_error | string | 最近错误摘要 |
| items[].updated_at | string | 最近状态更新时间（ISO-8601） |
| items[].last_intervention_at | string | 最近人工干预时间（ISO-8601，可空） |
| items[].last_intervention_by | string | 最近人工干预人（可空） |
| page_size | number | 当前页大小 |
| next_cursor | string | 下一页游标（无则 null） |
| has_next | boolean | 是否有下一页 |

错误码：E_GOV_4011、E_GOV_4032、E_REQ_4005。

请求示例：
```http
GET /api/v1/admin/super/outbox/dead?page_size=20&topic=task.state.changed HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "event_id":  "evt_01HT_dead_0001",
                                   "created_at":  "2026-04-06T10:40:00Z",
                                   "topic":  "task.state.changed",
                                   "partition_key":  "patient_1001",
                                   "retry_count":  11,
                                   "last_error":  "broker timeout",
                                   "updated_at":  "2026-04-06T10:48:00Z",
                                   "last_intervention_at":  null,
                                   "last_intervention_by":  null
                               }
                           ],
                 "page_size":  20,
                 "next_cursor":  null,
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 403
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4032",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4032 | 权限不足或越权访问 |
### 3.8.13 POST /api/v1/admin/super/outbox/dead/{event_id}/replay

用途：对指定 DEAD 事件执行受控重放，恢复分区发送链路。

权限：仅 SUPERADMIN。

AI Agent 执行策略：
1. 本接口允许 Agent 受控执行，但执行模式最低要求 `CONFIRM_3`。
2. 若 `X-Action-Source=AI_AGENT` 且 `X-Confirm-Level` 低于 `CONFIRM_3`，返回 `E_GOV_4097`。
3. 建议先调用 `dry_run=true` 预检查；预检查失败返回 `E_GOV_4226`。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| created_at | string | 是 | DEAD 事件创建时间（ISO-8601，用于分区路由） |
| replay_reason | string | 是 | 5-256 |
| replay_mode | string | 否 | RETRY_NOW/RETRY_AT，默认 RETRY_NOW |
| next_retry_at | string | 条件必填 | replay_mode=RETRY_AT 时必填，ISO-8601 |

约束说明：
1. 仅允许 `phase=DEAD` 事件重放。
2. 同 partition_key 若存在更早 DEAD 未修复事件，必须拒绝越序重放。
3. 重放定位必须使用 `event_id + created_at`，以命中 Outbox 分区路由。
4. 成功重放后写入 `last_intervention_by/last_intervention_at/replay_reason/replay_token/replayed_at`（`replay_token` 映射自 X-Request-Id）。
5. 重放动作必须落审计并保留 trace_id。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| event_id | string | 事件 ID |
| previous_phase | string | DEAD |
| phase | string | RETRY |
| partition_key | string | 分区键 |
| next_retry_at | string | 下次重试时间（ISO-8601） |
| replay_reason | string | 重放原因 |
| replayed_by | string | 操作人 ID |
| replayed_at | string | 重放时间（ISO-8601） |

错误码：E_GOV_4011、E_GOV_4032、E_GOV_4046、E_GOV_4096、E_REQ_4001、E_GOV_5002。

请求示例：
```http
POST /api/v1/admin/super/outbox/dead/evt_01HT_dead_0001/replay HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{
    "created_at":  "2026-04-06T10:40:00Z",
    "replay_reason":  "broker 已恢复，执行人工重放",
    "replay_mode":  "RETRY_NOW"
}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "event_id":  "evt_01HT_dead_0001",
                 "previous_phase":  "DEAD",
                 "phase":  "RETRY",
                 "partition_key":  "patient_1001",
                 "next_retry_at":  "2026-04-06T11:55:00Z",
                 "replay_reason":  "broker 已恢复，执行人工重放",
                 "replayed_by":  "9001",
                 "replayed_at":  "2026-04-06T11:55:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 409
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4096",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_GOV_4032 | 权限不足或越权访问 |
| 404 | E_GOV_4046 | 资源不存在或不可见 |
| 409 | E_GOV_4096 | 状态冲突或不满足重放前置条件 |
| 500 | E_GOV_5002 | 内部处理失败或审计回滚 |
## 3.9 身份权限与治理域：通知与消息中心接口

### 3.9.1 GET /api/v1/notifications

用途：分页拉取历史通知（WebSocket 离线兜底与消息中心首屏）。

权限：受保护接口。

查询参数：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| cursor | string | 否 | 透明游标；优先使用 |
| page_size | int32 | 否 | 1-100，默认 20 |
| page_no | int32 | 否 | 兼容字段（Deprecated）；仅浅分页建议 |
| type | string | 否 | TASK_PROGRESS/FENCE_ALERT/TASK_CLOSED/SYSTEM |
| read_status | string | 否 | UNREAD/READ |
| since_time | string | 否 | ISO-8601 |

分页规则：
1. 默认按 created_at DESC, notification_id DESC 返回。
2. 存在 cursor 时按 Cursor 模式滚动翻页，响应 data 返回 next_cursor。
3. page_no 仅用于兼容旧客户端，不保证深度分页性能。

data.items 摘要结构：

| 字段 | 逻辑类型 | JSON 类型 | 说明 |
| :--- | :--- | :--- | :--- |
| notification_id | int64 | string | 通知 ID |
| type | string | string | 通知类型 |
| title | string | string | 标题 |
| content | string | string | 内容摘要 |
| level | string | string | INFO/WARN/CRITICAL |
| related_task_id | int64 | string | 关联任务（可空） |
| related_patient_id | int64 | string | 关联患者（可空） |
| read_status | string | string | UNREAD/READ |
| created_at | string | string | 创建时间（ISO-8601） |

错误码：E_GOV_4011、E_GOV_4291、E_NOTI_4001、E_REQ_4005。

请求示例：
```http
GET /api/v1/notifications?cursor=eyJjcmVhdGVkX2F0IjoiMjAyNi0wNC0wNlQxMDowMDowMFoiLCJpZCI6IjgwMDEifQ%3D%3D&page_size=20 HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "items":  [
                               {
                                   "notification_id":  "8001",
                                   "type":  "TASK_PROGRESS",
                                   "title":  "任务状态更新",
                                   "content":  "已收到新的有效线索。",
                                   "level":  "INFO",
                                   "related_task_id":  "1001",
                                   "related_patient_id":  "1001",
                                   "read_status":  "UNREAD",
                                   "created_at":  "2026-04-06T11:16:00Z"
                               }
                           ],
                 "page_size":  20,
                 "next_cursor":  "eyJub3RpZmljYXRpb25faWQiOiI4MDAxIn0=",
                 "has_next":  false
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 429
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
Retry-After: 30
X-RateLimit-Remaining: 0
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4291",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_NOTI_4001、E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 429 | E_GOV_4291 | 网关限流，需按 Retry-After 退避 |
### 3.9.2 PUT /api/v1/notifications/{notification_id}/read

用途：将单条通知标记为已读。

权限：受保护接口。

请求体：无。

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| notification_id | string | 通知 ID |
| read_status | string | READ |
| read_at | string | 已读时间（ISO-8601） |

错误码：E_GOV_4011、E_NOTI_4030、E_NOTI_4041、E_REQ_4001、E_REQ_4005。

请求示例：
```http
PUT /api/v1/notifications/8001/read HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "notification_id":  "8001",
                 "read_status":  "READ",
                 "read_at":  "2026-04-06T11:20:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001、E_REQ_4005 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
| 403 | E_NOTI_4030 | 权限不足或越权访问 |
| 404 | E_NOTI_4041 | 资源不存在或不可见 |
### 3.9.3 PUT /api/v1/notifications/read-all

用途：批量将当前用户通知标记为已读。

权限：受保护接口。

请求体：

| 字段 | 类型 | 必填 | 规则 |
| :--- | :--- | :---: | :--- |
| before_time | string | 否 | ISO-8601；缺省表示全部已读 |

成功响应 data：

| 字段 | 类型 | 说明 |
| :--- | :--- | :--- |
| affected_count | number | 本次更新条数 |
| before_time | string | 生效截止时间（ISO-8601） |
| read_at | string | 批量已读时间（ISO-8601） |

错误码：E_GOV_4011、E_REQ_4001。

请求示例：
```http
PUT /api/v1/notifications/read-all HTTP/1.1
Authorization: Bearer <access_token>
X-Trace-Id: trc_demo_20260406_001
X-Request-Id: req_demo_20260406_001
Content-Type: application/json

{

}
```

响应示例（成功）：
```http
HTTP/1.1 200
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "success",
    "code":  "OK",
    "data":  {
                 "affected_count":  16,
                 "before_time":  "2026-04-06T11:20:00Z",
                 "read_at":  "2026-04-06T11:20:00Z"
             },
    "trace_id":  "trc_demo_20260406_001"
}
```

响应示例（失败）：
```http
HTTP/1.1 401
Content-Type: application/json
X-Trace-Id: trc_demo_20260406_001
```
```json
{
    "message":  "error",
    "code":  "E_GOV_4011",
    "data":  null,
    "trace_id":  "trc_demo_20260406_001"
}
```

状态码矩阵：
| HTTP 状态 | 业务码 | 典型场景 |
| :---: | :--- | :--- |
| 200 | OK | 请求成功 |
| 400 | E_REQ_4001 | 参数或格式不合法 |
| 401 | E_GOV_4011 | 鉴权失败或凭据缺失/失效 |
## 4. WebSocket、SSE 与长轮询契约

### 4.1 WebSocket 握手与连接

用途：任务进展、告警、状态更新推送。

握手端点：GET /api/v1/ws/notifications。

鉴权方案：
1. 浏览器端默认采用 Ticket 模式：先调用 3.6.7 获取 ws_ticket，再通过 Query 参数 ticket 建链。
2. 非浏览器调试可采用子协议模式：Sec-WebSocket-Protocol 传递 Bearer Token。
3. 同时存在两种凭据时，以 ws_ticket 优先。

失败语义：
1. Upgrade 前鉴权失败返回 HTTP 401（E_GOV_4011 或 E_GOV_4012）。
2. 连接建立后令牌失效，服务端以 4401 关闭连接并记录审计。

连接约定：
1. 客户端连接后注册 user_id -> pod_id 路由。
2. 事件经 ws.push.{pod_id} 定向下发，禁止全量广播。
3. 服务端每 30s 发送心跳；连续 2 个心跳周期未响应则主动断开并触发重连建议。

### 4.2 WebSocket 消息 Envelope

```json
{
  "event_id": "evt_01H...",
  "topic": "task.state.changed",
  "aggregate_id": "task_8848",
  "version": 12,
  "event_time": "2026-04-05T10:21:00Z",
  "trace_id": "trc_...",
  "payload": {}
}
```

客户端防乱序：
1. 主锚点为 version，只接收更大版本。
2. version 相同冲突时以 event_time 决定最终覆盖。
3. 旧版本或旧时间事件必须丢弃并计数。

### 4.3 长轮询降级

接口：GET /api/v1/rescue/tasks/{task_id}/events/poll。

用途：WS 不可用时拉取增量事件。

### 4.4 AI 流式（SSE）传输基线

适用范围：`POST /api/v1/ai/sessions/{session_id}/messages` 的流式响应。

响应头基线：
1. `Content-Type: text/event-stream`
2. `Cache-Control: no-cache`
3. `Connection: keep-alive`
4. `X-Accel-Buffering: no`（反向代理禁缓冲）

连接与保活：
1. 服务端每 15s 发送一次心跳注释（`: ping`）。
2. 客户端连续 2 次重连失败后，可切换到 WS 备选链路。
3. 流式传输失败不得导致本次用户消息丢失；客户端可通过 3.5.7 回补历史消息。

## 5. 事件总线契约（Kafka）

### 5.1 统一事件结构

```json
{
  "event_id": "evt_01H...",
  "topic": "task.state.changed",
  "partition_key": "patient_1001",
  "aggregate_id": "task_8848",
  "event_time": "2026-04-05T10:21:00Z",
  "version": 12,
  "request_id": "req_...",
  "trace_id": "trc_...",
  "producer": "task-service",
  "payload": {}
}
```

### 5.2 Topic 清单

| Topic | 分区键 | 生产方 | 消费方 |
| :--- | :--- | :--- | :--- |
| clue.reported.raw | tag_code | clue-intake-service | clue-analysis-service |
| clue.vectorize.requested | patient_id | clue-intake-service | ai-vectorizer-service |
| clue.validated | patient_id | clue-analysis-service | task-service、ai-orchestrator-service、clue-trajectory-service |
| clue.suspected | patient_id | clue-analysis-service | admin-review-service、task-service |
| clue.rejected | patient_id | admin-review-service | clue-analysis-service、governance |
| track.updated | patient_id | clue-analysis/trajectory | task-service、ai-orchestrator-service |
| fence.breached | patient_id | clue-analysis-service | task-service、notify-service |
| task.created | patient_id | task-service | profile-service、notify-service、ai-orchestrator-service |
| task.state.changed | patient_id | task-service | clue-analysis-service |
| task.resolved | patient_id | task-service | profile-service、clue-trajectory-service、notify-service、ai-orchestrator-service |
| task.false_alarm | patient_id | task-service | profile-service、clue-trajectory-service、notify-service |
| ai.strategy.generated | task_id | ai-orchestrator-service | task-service |
| ai.poster.generated | task_id | ai-orchestrator-service | task-service |
| profile.created | patient_id | profile-service | ai-vectorizer-service |
| profile.updated | patient_id | profile-service | ai-vectorizer-service |
| profile.corrected | patient_id | profile-service | ai-vectorizer-service |
| profile.deleted.logical | patient_id | profile-service | ai-vectorizer-service |
| memory.appended | patient_id | ai-orchestrator-service | ai-vectorizer-service |
| memory.expired | patient_id | ai-orchestrator-service | ai-vectorizer-service |
| material.order.created | order_id | material-service | admin handlers |
| tag.bound | patient_id | profile-service | material-service、notify-service |

Topic 闭环说明：material-service 消费 tag.bound 后，触发 order.auto_confirm.on_bind 内部收敛动作，语义上等效执行 3.4.14 的签收状态推进。

## 6. 联调执行建议

1. 前端先对接 3.7 认证接口（register/login）与 3.1 任务域接口，完成账号到业务闭环。
2. 管理端并行接入 3.2.6 和 3.2.7（override/reject）、3.8 用户治理接口与 3.3 监护关系接口。
3. 所有写接口统一注入 X-Request-Id 和 X-Trace-Id，建立联调日志追踪。
4. WebSocket 与 events/poll 必须共享同一消息 Envelope 解析器。
5. 联调阶段禁止私自新增字段，新增需求走版本变更流程。

## 7. 版本管理

1. 当前版本：V1.10。
2. 破坏性变更（字段删除、类型变更、错误码语义变更）必须升级主版本。
3. 非破坏性变更（新增可选字段、新增错误码）升级次版本并补充变更记录。

## 8. 落地接口覆盖目录（同域，不新增领域）

说明：
1. 本节用于将 SRS 用例、状态机动作和运营治理查询展开为工程可落地的接口覆盖目录。
2. 仅在现有域内补齐，不新增业务领域对象。
3. 目录按实施优先级分层：P0（核心闭环）、P1（运营可观测）、P2（治理增强）。

### 8.1 规格对齐规则（与第 3 章一致）

1. 目录条目进入开发前，必须补齐与第 3 章同等结构：用途、权限、请求参数表、响应 data 字段表、错误码。
2. 写接口（POST/PUT/DELETE）必须补充状态机约束与审计约束，并强制要求 X-Request-Id（见 1.2、1.6）。
3. 列表接口必须遵循 1.5 分页结构：普通列表使用 Offset（items/page_no/page_size/total/has_next），追加流水使用 Cursor（items/page_size/next_cursor/has_next，入参 cursor 优先）。
4. 全部 ID 字段遵循 1.1 规则，以 string 线传；路径与查询参数中的 ID 非法统一返回 E_REQ_4005。

### 8.2 核心验收目录（主题对齐版，44 条，非全量索引）

说明：
1. 8.2 仅作为高优先级验收检查单（P0-P2 核心闭环）。
2. 第 3 章方法+路径全量覆盖见 8.4 统计视图。

| 编号 | 方法 | 路径 | 归属域 | 请求形态 | 角色 | 错误码最小集 | 优先级 | 规格状态 | 对齐章节 |
| :--- | :---: | :--- | :--- | :--- | :--- | :--- | :---: | :--- | :--- |
| EX-001 | GET | /api/v1/rescue/tasks/{task_id} | 任务域 | DETAIL | FAMILY/ADMIN/SUPERADMIN | G_READ + E_TASK_4041 | P1 | 已落地 | 3.1.8 |
| EX-002 | GET | /api/v1/rescue/tasks/{task_id}/events | 任务域 | LIST | FAMILY/ADMIN/SUPERADMIN | G_READ + E_TASK_4041 | P1 | 已落地 | 3.1.9 |
| EX-003 | GET | /api/v1/rescue/tasks/{task_id}/clues | 任务域 | LIST | FAMILY/ADMIN/SUPERADMIN | G_READ + E_TASK_4041 | P1 | 已落地 | 3.1.10 |
| EX-004 | GET | /api/v1/rescue/tasks/{task_id}/alerts | 任务域 | LIST | FAMILY/ADMIN/SUPERADMIN | G_READ + E_TASK_4041 | P2 | 已落地 | 3.1.15 |
| EX-005 | GET | /api/v1/admin/rescue/tasks | 任务域 | LIST | ADMIN/SUPERADMIN | G_READ + E_TASK_4041 | P1 | 已落地 | 3.1.11 |
| EX-006 | GET | /api/v1/admin/rescue/tasks/{task_id} | 任务域 | DETAIL | ADMIN/SUPERADMIN | G_READ + E_TASK_4041 | P1 | 已落地 | 3.1.12 |
| EX-007 | GET | /api/v1/admin/rescue/tasks/{task_id}/audit | 任务域 | LIST | ADMIN/SUPERADMIN | G_READ + E_TASK_4041 | P1 | 已落地 | 3.1.13 |
| EX-008 | POST | /api/v1/admin/rescue/tasks/{task_id}/notify/retry | 任务域 | COMMAND | ADMIN/SUPERADMIN | G_WRITE + E_TASK_4041 + E_GOV_5002 | P2 | 已落地 | 3.1.16 |
| EX-009 | GET | /api/v1/rescue/tasks/statistics | 任务域 | LIST | ADMIN/SUPERADMIN | G_READ + E_TASK_4041 | P2 | 已落地 | 3.1.17 |
| EX-010 | GET | /api/v1/clues/{clue_id} | 线索域 | DETAIL | FAMILY/ADMIN/SUPERADMIN | G_READ + E_CLUE_4043 | P1 | 已落地 | 3.2.8 |
| EX-011 | GET | /api/v1/clues/{clue_id}/timeline | 线索域 | LIST | FAMILY/ADMIN/SUPERADMIN | G_READ + E_CLUE_4043 | P2 | 已落地 | 3.2.11 |
| EX-012 | GET | /api/v1/rescue/tasks/{task_id}/clues/latest | 线索域 | DETAIL | FAMILY/ADMIN/SUPERADMIN | G_READ + E_CLUE_4043 | P1 | 已落地 | 3.1.14 |
| EX-013 | GET | /api/v1/admin/clues/{clue_id} | 线索域 | DETAIL | ADMIN/SUPERADMIN | G_READ + E_CLUE_4043 | P1 | 已落地 | 3.2.9 |
| EX-014 | GET | /api/v1/admin/clues/review/queue | 线索域 | LIST | ADMIN/SUPERADMIN | G_READ + E_CLUE_4043 | P1 | 已落地 | 3.2.10 |
| EX-015 | POST | /api/v1/admin/clues/{clue_id}/assign | 线索域 | COMMAND | ADMIN/SUPERADMIN | G_WRITE + E_CLUE_4043 | P2 | 已落地 | 3.2.12 |
| EX-016 | POST | /api/v1/admin/clues/{clue_id}/request-evidence | 线索域 | COMMAND | ADMIN/SUPERADMIN | G_WRITE + E_CLUE_4043 | P2 | 暂不开放（毕设精简） | 3.2.13 |
| EX-017 | GET | /api/v1/admin/clues/statistics | 线索域 | LIST | ADMIN/SUPERADMIN | G_READ + E_CLUE_4043 | P2 | 已落地 | 3.2.14 |
| EX-018 | GET | /api/v1/patients/{patient_id} | 档案域 | DETAIL | FAMILY/ADMIN/SUPERADMIN | G_READ + E_PRO_4041 | P1 | 已落地 | 3.3.11 |
| EX-019 | PUT | /api/v1/patients/{patient_id} | 档案域 | COMMAND | FAMILY/ADMIN/SUPERADMIN | G_WRITE + E_PRO_4041 | P1 | 已落地 | 3.3.9 |
| EX-020 | GET | /api/v1/patients/{patient_id}/guardians/invitations | 档案域 | LIST | FAMILY/ADMIN | G_READ + E_PRO_4041 | P1 | 已落地 | 3.3.12 |
| EX-021 | GET | /api/v1/patients/{patient_id}/guardians/invitations/{invite_id} | 档案域 | DETAIL | FAMILY/ADMIN | G_READ + E_PRO_4043 | P2 | 已落地 | 3.3.15 |
| EX-022 | GET | /api/v1/patients/{patient_id}/guardians/transfers | 档案域 | LIST | FAMILY/ADMIN | G_READ + E_PRO_4041 | P1 | 已落地 | 3.3.13 |
| EX-023 | GET | /api/v1/patients/{patient_id}/guardians/transfers/{transfer_request_id} | 档案域 | DETAIL | FAMILY/ADMIN | G_READ + E_PRO_4045 | P2 | 已落地 | 3.3.16 |
| EX-024 | PUT | /api/v1/patients/{patient_id}/fence | 档案域 | COMMAND | FAMILY/ADMIN | G_WRITE + E_PRO_4041 + E_PRO_4221 | P1 | 已落地 | 3.3.10 |
| EX-025 | GET | /api/v1/patients/{patient_id}/fence | 档案域 | DETAIL | FAMILY/ADMIN | G_READ + E_PRO_4041 | P1 | 已落地 | 3.3.14 |
| EX-026 | GET | /api/v1/material/orders | 物资域 | LIST | FAMILY | G_READ + E_MAT_4041 | P1 | 已落地 | 3.4.18 |
| EX-027 | GET | /api/v1/material/orders/{order_id} | 物资域 | DETAIL | FAMILY | G_READ + E_MAT_4041 | P1 | 已落地 | 3.4.19 |
| EX-028 | GET | /api/v1/material/orders/{order_id}/tracking | 物资域 | LIST | FAMILY | G_READ + E_MAT_4041 | P2 | 暂不开放（毕设精简） | 3.4.24 |
| EX-029 | GET | /api/v1/material/orders/{order_id}/resource-link | 物资域 | DETAIL | FAMILY | G_READ + E_MAT_4041 | P2 | 已落地 | 3.4.25 |
| EX-030 | GET | /api/v1/admin/material/orders/{order_id} | 物资域 | DETAIL | ADMIN/SUPERADMIN | G_READ + E_MAT_4041 | P1 | 已落地 | 3.4.20 |
| EX-031 | GET | /api/v1/admin/material/orders/{order_id}/timeline | 物资域 | LIST | ADMIN/SUPERADMIN | G_READ + E_MAT_4041 | P1 | 已落地 | 3.4.21 |
| EX-032 | GET | /api/v1/admin/tags | 物资域 | LIST | ADMIN/SUPERADMIN | G_READ + E_MAT_4044 | P1 | 已落地 | 3.4.22 |
| EX-033 | GET | /api/v1/admin/tags/{tag_code} | 物资域 | DETAIL | ADMIN/SUPERADMIN | G_READ + E_MAT_4044 | P1 | 已落地 | 3.4.23 |
| EX-034 | POST | /api/v1/admin/tags/{tag_code}/allocate | 物资域 | COMMAND | ADMIN/SUPERADMIN | G_WRITE + E_MAT_4044 + E_MAT_4098 | P2 | 已落地 | 3.4.26 |
| EX-035 | POST | /api/v1/admin/tags/{tag_code}/release | 物资域 | COMMAND | ADMIN/SUPERADMIN | G_WRITE + E_MAT_4044 + E_MAT_4098 | P2 | 已落地 | 3.4.27 |
| EX-036 | GET | /api/v1/patients/{patient_id}/tags/{tag_code} | 物资域 | DETAIL | FAMILY/ADMIN | G_READ + E_MAT_4044 | P2 | 已落地 | 3.4.28 |
| EX-037 | GET | /api/v1/patients/{patient_id}/tags/{tag_code}/history | 物资域 | LIST | FAMILY/ADMIN | G_READ + E_MAT_4044 | P2 | 已落地 | 3.4.29 |
| EX-038 | GET | /api/v1/ai/sessions/{session_id}/messages | AI 域 | LIST | FAMILY/ADMIN | G_READ + E_AI_4041 | P1 | 已落地 | 3.5.7 |
| EX-039 | GET | /api/v1/ai/sessions/{session_id}/quota | AI 域 | DETAIL | FAMILY/ADMIN | G_READ + E_AI_4041 + E_AI_4293 | P2 | 已落地 | 3.5.10 |
| EX-040 | POST | /api/v1/ai/sessions/{session_id}/archive | AI 域 | COMMAND | FAMILY/ADMIN | G_WRITE + E_AI_4041 + E_AI_4091 | P2 | 已落地 | 3.5.11 |
| EX-041 | GET | /api/v1/admin/ai/sessions/{session_id} | AI 域 | DETAIL | ADMIN/SUPERADMIN | G_READ + E_AI_4041 | P1 | 已落地 | 3.5.8 |
| EX-042 | GET | /api/v1/admin/ai/sessions/{session_id}/messages | AI 域 | LIST | ADMIN/SUPERADMIN | G_READ + E_AI_4041 | P1 | 已落地 | 3.5.9 |
| EX-043 | GET | /api/v1/admin/metrics/security | 治理域 | LIST | ADMIN/SUPERADMIN | G_READ + E_GOV_4032 | P2 | 已落地 | 3.8.10 |
| EX-044 | GET | /api/v1/admin/config | 治理域 | DETAIL | ADMIN/SUPERADMIN | G_READ + E_GOV_4032 | P2 | 已落地 | 3.8.11 |

### 8.3 覆盖补齐目录（治理域扩展：认证与通知能力，新增 18 条）

| 编号 | 方法 | 路径 | 归属域 | 请求形态 | 角色 | 错误码最小集 | 优先级 | 规格状态 | 对齐章节 |
| :--- | :---: | :--- | :--- | :--- | :--- | :--- | :---: | :--- | :--- |
| EX-045 | POST | /api/v1/auth/register | 治理域（认证能力） | COMMAND | ANONYMOUS | E_AUTH_4001 + E_AUTH_4091 | P0 | 已落地 | 3.7.1 |
| EX-046 | POST | /api/v1/auth/login | 治理域（认证能力） | COMMAND | ANONYMOUS | E_GOV_4011 + E_GOV_4031 | P0 | 已落地 | 3.7.2 |
| EX-047 | POST | /api/v1/auth/logout | 治理域（认证能力） | COMMAND | FAMILY/ADMIN/SUPERADMIN | G_WRITE + E_GOV_4011 | P0 | 已落地 | 3.7.3 |
| EX-048 | PUT | /api/v1/users/me/password | 治理域（认证能力） | COMMAND | FAMILY/ADMIN/SUPERADMIN | G_WRITE + E_USR_4011 + E_USR_4002 | P1 | 已落地 | 3.7.4 |
| EX-049 | GET | /api/v1/admin/users | 治理域 | LIST | ADMIN/SUPERADMIN | G_READ + E_GOV_4030 | P1 | 已落地 | 3.8.1 |
| EX-050 | PUT | /api/v1/admin/users/{user_id}/status | 治理域 | COMMAND | ADMIN/SUPERADMIN | G_WRITE + E_USR_4041 + E_USR_4003 | P1 | 已落地 | 3.8.2 |
| EX-051 | PUT | /api/v1/admin/users/{user_id}/password:reset | 治理域 | COMMAND | SUPERADMIN | G_WRITE + E_GOV_4032 + E_USR_4041 | P1 | 已落地 | 3.8.3 |
| EX-052 | GET | /api/v1/admin/logs | 治理域 | LIST | ADMIN/SUPERADMIN | G_READ + E_GOV_4030 + E_GOV_4291 | P1 | 已落地 | 3.8.4 |
| EX-053 | GET | /api/v1/admin/dashboard/metrics | 治理域 | LIST | ADMIN/SUPERADMIN | G_READ + E_GOV_4030 | P2 | 已落地 | 3.8.5 |
| EX-054 | POST | /api/v1/admin/super/export-data | 治理域 | COMMAND | SUPERADMIN | G_WRITE + E_GOV_4032 + E_GOV_5002 | P2 | 已落地 | 3.8.6 |
| EX-055 | POST | /api/v1/admin/super/logs/purge | 治理域 | COMMAND | SUPERADMIN | G_WRITE + E_GOV_4032 + E_GOV_5002 | P1 | 已落地 | 3.8.7 |
| EX-056 | PUT | /api/v1/admin/super/config | 治理域 | COMMAND | SUPERADMIN | G_WRITE + E_GOV_4032 + E_GOV_5002 | P2 | 已落地 | 3.8.8 |
| EX-057 | POST | /api/v1/admin/super/rescue/tasks/{task_id}/force-close | 治理域 | COMMAND | SUPERADMIN | G_WRITE + E_GOV_4032 + E_TASK_4093 | P2 | 已落地 | 3.8.9 |
| EX-058 | GET | /api/v1/notifications | 治理域（通知能力） | LIST | FAMILY/ADMIN/SUPERADMIN | G_READ + E_GOV_4011 + E_GOV_4291 | P1 | 已落地 | 3.9.1 |
| EX-059 | PUT | /api/v1/notifications/{notification_id}/read | 治理域（通知能力） | COMMAND | FAMILY/ADMIN/SUPERADMIN | G_WRITE + E_NOTI_4041 | P1 | 已落地 | 3.9.2 |
| EX-060 | PUT | /api/v1/notifications/read-all | 治理域（通知能力） | COMMAND | FAMILY/ADMIN/SUPERADMIN | G_WRITE + E_GOV_4011 | P1 | 已落地 | 3.9.3 |
| EX-061 | GET | /api/v1/admin/super/outbox/dead | 治理域 | LIST | SUPERADMIN | G_READ + E_GOV_4032 + E_REQ_4005 | P1 | 已落地 | 3.8.12 |
| EX-062 | POST | /api/v1/admin/super/outbox/dead/{event_id}/replay | 治理域 | COMMAND | SUPERADMIN | G_WRITE + E_GOV_4032 + E_GOV_4096 | P1 | 已落地 | 3.8.13 |

### 8.4 第 3 章全量覆盖统计（方法+路径 114 条）

| 分组 | 覆盖章节 | 方法+路径数 | 说明 |
| :--- | :--- | :---: | :--- |
| 任务域 | 3.1.1 ~ 3.1.17 | 17 | 含任务状态、事件流、审计与统计 |
| 线索域 | 3.2.1 ~ 3.2.14 | 14 | 含匿名入口、复核流、轨迹时间线 |
| 档案域 | 3.3.1 ~ 3.3.16 | 16 | 含监护关系、患者档案与围栏 |
| 物资域 | 3.4.1 ~ 3.4.29 | 29 | 含工单、标签、资源链与管理视图 |
| AI 域 | 3.5.1 ~ 3.5.11 | 11 | 含会话、消息、记忆、归档与配额 |
| 查询补充域 | 3.6.1 ~ 3.6.7 | 7 | 含疑似任务/线索与管理查询 |
| 治理域（认证能力） | 3.7.1 ~ 3.7.4 | 4 | 注册、登录、登出、改密 |
| 治理域（基础治理） | 3.8.1 ~ 3.8.13 | 13 | 用户治理、日志、超管操作、配置、Outbox 干预 |
| 治理域（通知能力） | 3.9.1 ~ 3.9.3 | 3 | 消息中心与读状态管理 |
| 合计 | 3.1 ~ 3.9 | 114 | 仅统计“方法+路径”显式接口 |

### 8.5 数量与推进建议

1. 当前第 3 章显式主接口为 114 条（`/api/v1` 方法+路径去重口径）。
2. 目录层当前为 62 条（8.2 核心 44 + 8.3 补齐 18），已覆盖治理域（认证/基础治理/通知能力）关键验收项。 
3. 全量口径以第 3 章与 8.4 统计视图为准，后续按发布节奏将高频接口滚动纳入目录验收项。

## 9. Agent 能力清单（安全扩围版）

### 9.1 扩围目标与红线

1. 目标是在不突破 HC-01（AI 不直接改业务状态）的前提下，最大化开放 Agent 的可执行能力。
2. 默认放开 A0/A1（读与建议），按确认等级逐步放开 A2/A3（受控写）。
3. A4 动作长期保持 `MANUAL_ONLY`，仅允许 Agent 预检查与建议，不允许代执行。
4. 所有 Agent 写请求必须先通过 `dry_run=true` 预检查，再进入确认流程。
5. 任何执行都必须具备 `X-Action-Source`、`X-Agent-Profile`、`X-Execution-Mode`、`X-Confirm-Level`，并落审计字段。

### 9.2 能力包白名单（X-Agent-Profile）

| 能力包 | 职责范围 | 默认最高级别 |
| :--- | :--- | :--- |
| RescueCommander | 任务态势汇总、闭环建议、通知重试编排 | A3 |
| ClueInvestigator | 线索聚合、复核建议、补证建议、轨迹关联分析 | A3 |
| GuardianCoordinator | 监护关系流程建议与受控编排（邀请、转移、撤销） | A2 |
| MaterialOperator | 物资工单与标签链路建议、异常处置编排 | A3 |
| AICaseCopilot | 会话管理、策略建议、记忆沉淀与摘要 | A2 |
| GovernanceSentinel | 用户治理、审计检索、治理风险预警 | A3 |
| OutboxReliabilityAgent | DEAD 队列诊断、重放预检与执行编排 | A3 |

能力包开关配置键（`sys_config.scope=ai_policy`）：

| 能力包（X-Agent-Profile） | config_key |
| :--- | :--- |
| RescueCommander | agent.capability.rescue.enabled |
| ClueInvestigator | agent.capability.clue.enabled |
| GuardianCoordinator | agent.capability.guardian.enabled |
| MaterialOperator | agent.capability.material.enabled |
| AICaseCopilot | agent.capability.ai_case.enabled |
| GovernanceSentinel | agent.capability.governance.enabled |
| OutboxReliabilityAgent | agent.capability.outbox_reliability.enabled |

Agent 策略配置键（`sys_config.scope=ai_policy`）：

| config_key | 说明 |
| :--- | :--- |
| agent.execution.max_level | 允许执行上限（A0/A1/A2/A3） |
| agent.confirmation.policy | 确认级别策略映射 |
| agent.manual_only.actions | 人工专属接口白名单 |

说明：上述策略键与能力包开关键共同构成 `ai_policy` 作用域下的完整 Agent 治理键集。

### 9.3 执行等级与确认策略

| 等级 | 执行语义 | 最低确认策略 | 典型用途 |
| :--- | :--- | :--- | :--- |
| A0 | 自动观测 | AUTO | 列表/详情读取、指标聚合、异常检测 |
| A1 | 智能助理 | AUTO + USER_REVIEW | 建议卡片、草稿、参数预填 |
| A2 | 受控执行 | CONFIRM_1 | 常规业务写操作 |
| A3 | 高风险受控执行 | CONFIRM_2 或 CONFIRM_3 | 管理治理写操作、状态推进 |
| A4 | 人工专属 | MANUAL_ONLY | 不可逆或高合规敏感动作 |

### 9.4 可执行能力清单（扩围后）

#### 9.4.1 A0-A1 自动观测与智能助理（默认 AUTO）

| 能力组 | 代表接口 | 说明 |
| :--- | :--- | :--- |
| 任务态势观测 | 3.1.3/3.1.5/3.1.8/3.1.9/3.1.10/3.1.15/3.1.17 | 任务详情、事件、告警、统计只读分析 |
| 线索态势观测 | 3.2.8/3.2.9/3.2.10/3.2.11/3.2.14 | 线索详情、复核队列、时间线、统计 |
| 监护与档案观测 | 3.3.11/3.3.12/3.3.13/3.3.14/3.3.15/3.3.16 | 监护关系与围栏配置读取 |
| 物资链路观测 | 3.4.18~3.4.23/3.4.25/3.4.28/3.4.29 | 工单、资源链、标签详情与历史读取 |
| AI 会话观测 | 3.5.2/3.5.4/3.5.7/3.5.8/3.5.9/3.5.10 | 会话、消息、配额只读与辅助分析 |
| 查询补充能力 | 3.6.1~3.6.6 | 聚合视图与检索入口 |
| 治理观测能力 | 3.8.1/3.8.4/3.8.5/3.8.10/3.8.11/3.8.12 | 用户、日志、指标、配置、DEAD 队列读取 |
| 通知观测能力 | 3.9.1 | 通知中心只读聚合 |

#### 9.4.2 A2 受控执行（CONFIRM_1，建议先 dry_run）

| 能力组 | 代表接口 | 最低确认 |
| :--- | :--- | :--- |
| 任务创建与常规闭环 | 3.1.1、3.1.2、3.1.16 | CONFIRM_1 |
| 线索入站与协同 | 3.2.5、3.2.12 | CONFIRM_1 |
| 监护流程编排 | 3.3.1、3.3.2、3.3.3、3.3.4、3.3.5、3.3.6 | CONFIRM_1 |
| 档案更新与围栏配置 | 3.3.8、3.3.9、3.3.10 | CONFIRM_1 |
| 物资家属侧流程 | 3.4.1、3.4.3、3.4.4、3.4.5、3.4.14 | CONFIRM_1 |
| 标签常规调度 | 3.4.26、3.4.27 | CONFIRM_1 |
| AI 会话与记忆管理 | 3.5.1、3.5.3、3.5.5、3.5.11 | CONFIRM_1 |
| 通知读状态 | 3.9.2、3.9.3 | CONFIRM_1 |

#### 9.4.3 A3 高风险受控执行（CONFIRM_2/3）

| 能力组 | 代表接口 | 最低确认 |
| :--- | :--- | :--- |
| 线索治理处置 | 3.2.6、3.2.7 | CONFIRM_2 |
| 物资管理关键流转 | 3.4.7、3.4.8、3.4.9、3.4.10、3.4.11、3.4.12、3.4.13 | CONFIRM_2 |
| 标签高风险治理 | 3.4.6、3.4.15、3.4.16、3.4.17 | CONFIRM_2 |
| 用户治理动作 | 3.8.2 | CONFIRM_2 |
| 超管配置变更 | 3.8.8 | CONFIRM_3 |
| Outbox 死信重放 | 3.8.13 | CONFIRM_3 |

#### 9.4.4 A4 人工专属（MANUAL_ONLY）

| 动作 | 接口 | 规则 |
| :--- | :--- | :--- |
| 数据导出 | 3.8.6 | Agent 禁止代执行，命中返回 `E_GOV_4231` |
| 日志清理 | 3.8.7 | Agent 禁止代执行，命中返回 `E_GOV_4231` |
| 强制关单（管理端） | 3.8.9 | Agent 禁止代执行，命中返回 `E_GOV_4231` |
| 强制关单（任务域紧急口） | 3.1.7 | 建议按 A4 管理，仅保留人工页面执行 |

### 9.5 默认禁代执行清单（即使扩围也不放开）

1. 凭据与身份敏感链路：3.7.1、3.7.2、3.7.4、3.8.3。
2. 全量数据外发与破坏性清理：3.8.6、3.8.7。
3. 不可逆紧急治理：3.8.9、3.1.7。
4. 任意未声明 `X-Execution-Mode` 与 `X-Confirm-Level` 的写请求。

### 9.6 扩围实施顺序（建议）

1. 第一阶段：放开全部 A0/A1 + A2（任务、档案、AI、通知、线索协同）并强制 `dry_run`。
2. 第二阶段：放开 A3 中 `CONFIRM_2` 动作（线索治理、物资关键流转、标签治理）。
3. 第三阶段：放开 `CONFIRM_3` 动作（3.8.8、3.8.13），仍保持 A4 永不代执行。
4. 任意阶段出现连续策略阻断或执行失败率异常，按能力包一键降级到 A1 只建议模式。

### 9.7 Function Calling Action 白名单与接口映射

| action | 目标接口（方法+路径） | 最低确认 | 执行约束 |
| :--- | :--- | :--- | :--- |
| propose_close | POST /api/v1/rescue/tasks/{task_id}/close（3.1.2） | CONFIRM_1 | 可执行 |
| clue_override | POST /api/v1/clues/{clue_id}/override（3.2.6） | CONFIRM_2 | 可执行 |
| clue_reject | POST /api/v1/clues/{clue_id}/reject（3.2.7） | CONFIRM_2 | 可执行 |
| approve_material_order | PUT /api/v1/admin/material/orders/{order_id}/approve（3.4.7） | CONFIRM_2 | 可执行 |
| archive_session | POST /api/v1/ai/sessions/{session_id}/archive（3.5.11） | CONFIRM_1 | 可执行 |
| replay_outbox_dead | POST /api/v1/admin/super/outbox/dead/{event_id}/replay（3.8.13） | CONFIRM_3 | 可执行 |
| request_evidence | POST /api/v1/admin/clues/{clue_id}/request-evidence（3.2.13） | MANUAL_ONLY | 本版本暂不开放，仅可建议 |
| force_close_task | POST /api/v1/admin/super/rescue/tasks/{task_id}/force-close（3.8.9） | MANUAL_ONLY | A4 动作，仅人工页面可执行 |

处理规则：
1. `action` 不在白名单、能力包未启用、或确认等级不足时，服务端必须策略阻断（`POLICY_BLOCK`）且不得触发领域写入。
2. `MANUAL_ONLY` 动作仅允许生成建议卡片，不允许 Agent 代执行。
3. 执行成功时，必须在 3.5.3 `event=done` 中回传 `action_id/result_code/executed_at`。
