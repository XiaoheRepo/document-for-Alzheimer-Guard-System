# 基于 AI Agent 的阿尔兹海默症 Alzheimer 患者协同寻回系统

## 接口文档（API Specification）

## 0. 文档信息

| 项目 | 内容 |
| :--- | :--- |
| 文档名称 | 接口文档（API Specification） |
| 版本 | V2.0 |
| 日期 | 2026-04-19 |
| 输入基线 | SRS V2.0（2026-04-04）、SADD V2.0（2026-04-12）、LLD V2.0（2026-04-19）、DBD V2.0（2026-06-17） |
| 适用对象 | 后端研发、前端研发（Android / H5 / Web Admin）、测试、联调、答辩演示 |

说明：

1. 本文档为联调契约，接口字段、错误码、事件语义以本文档为准。
2. 本文档严格对齐 SRS V2.0 功能需求编号（FR-\*）与 SADD V2.0 硬约束（HC-01 ~ HC-08），任何偏差均已在对应接口处显式标注。
3. 演示版本采用单节点部署，不影响接口契约有效性。
4. 事件总线选型为 Redis Streams（来自 SADD ADR-006），替代早期版本的 Kafka 方案。
5. 领域边界固定为六域：TASK / CLUE / PROFILE / MAT / AI / GOV，§3.6 为查询编排层（BFF），不计入独立领域。

---

## 1. 全局接口规范

### 1.1 Base URL

| 场景 | 路径前缀 |
| :--- | :--- |
| 业务 API | `/api/v1` |
| 匿名扫码入口 | `/r/{resource_token}` |
| 匿名短码页面 | `/p/{short_code}/...` |

**ID 传输强制约定**：

1. 文档中的 `int64` 为领域逻辑类型，不代表 JSON 数值类型。
2. 所有 ID 字段（`task_id`、`patient_id`、`clue_id`、`user_id`、`invite_id`、`transfer_request_id`、`order_id` 等）在 JSON 请求体与响应体中**必须使用 `string` 传输**。
3. 路径参数与查询参数中的 ID 也按十进制字符串处理，前端禁止按 JavaScript Number 参与精度敏感计算。
4. ID 传输值必须是纯数字字符串，不满足时返回 `E_REQ_4005`。

### 1.2 通用 Header

| Header | 是否必填 | 规则 |
| :--- | :---: | :--- |
| `Authorization` | 受保护接口必填 | `Bearer JWT`；匿名接口可不带 |
| `X-Anonymous-Token` | 匿名接口（非浏览器）可选 | APP / MINI_PROGRAM 等非浏览器端可用；与 `entry_token` 等效 |
| `X-Request-Id` | 写接口必填 | 长度 16-64，仅字母数字与 `-`（HC-03） |
| `X-Trace-Id` | 全链路必填 | 长度 16-64，禁止空值，响应必须回写（HC-04） |
| `X-Action-Source` | Agent 执行链路条件必填 | `USER` / `AI_AGENT`；默认 `USER` |
| `X-Agent-Profile` | Agent 执行链路条件必填 | Agent 能力包标识（见 §6.2） |
| `X-Execution-Mode` | Agent 执行链路条件必填 | `AUTO` / `CONFIRM_1` / `CONFIRM_2` / `CONFIRM_3` / `MANUAL_ONLY` |
| `X-Confirm-Level` | Agent 执行链路条件必填 | `CONFIRM_1` / `CONFIRM_2` / `CONFIRM_3`；与策略门禁匹配 |
| `Content-Type` | JSON 接口必填 | `application/json` |

### 1.3 网关保留头与防伪造

| 项 | 规则 |
| :--- | :--- |
| `X-User-Id`、`X-User-Role` | 仅网关注入；客户端同名头必须被清洗或拒绝 |
| 拒绝码 | `E_REQ_4003` |
| 执行顺序 | 先清洗保留头，再做令牌解析，再注入内部头 |

### 1.4 通用响应结构

成功响应：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_001",
  "data": {}
}
```

失败响应：

```json
{
  "code": "E_TASK_4091",
  "message": "任务进行中，请勿重复发起",
  "trace_id": "trc_20260419_001",
  "data": null
}
```

### 1.5 分页响应结构（Offset 与 Cursor）

**分页策略**：

1. 普通查询接口可采用 Offset 模式（`page_no` / `page_size`）。
2. 追加写入型流水表（审计日志、通知流、事件流）**必须**采用 Cursor 优先，避免深度分页性能退化。
3. Cursor 为服务端生成的透明游标，客户端不得依赖其内部结构。
4. 事件版本锚点场景允许 `after_version` / `since_version` 作为游标变体；服务端必须等价转换为 Cursor 语义返回 `next_cursor`。

**Offset 模式**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_xxx",
  "data": {
    "items": [],
    "page_no": 1,
    "page_size": 20,
    "total": 0,
    "has_next": false
  }
}
```

**Cursor 模式**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_xxx",
  "data": {
    "items": [],
    "page_size": 20,
    "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNi0wNC0xOVQxMDowMDowMFoiLCJpZCI6IjgwMDEifQ==",
    "has_next": false
  }
}
```

### 1.6 幂等与时间窗

1. 所有写接口都必须支持 `X-Request-Id` 幂等（HC-03）。
2. `request_time` 与服务端偏差必须 ≤ 300s，超限返回 `E_REQ_4221`。
3. 同一个 `request_id` 重复提交返回首次处理结果，不重复副作用。
4. 幂等缓存 `Redis Key = "idem:req:{X-Request-Id}"`，TTL = 24h。

### 1.7 Trace-Id 生成与传播

1. 客户端必须传入 `X-Trace-Id`，缺失或空值请求直接拒绝。
2. 若客户端传入 `X-Trace-Id` 但格式非法，返回 `E_REQ_4002`。
3. 响应头必须回写同一 Trace-Id；服务端日志与事件消息必须带同一 `trace_id`（HC-04）。

### 1.8 限流响应头契约

1. 网关触发限流时必须返回 HTTP 429，并返回业务码（如 `E_GOV_4291`、`E_AI_4292` / `E_AI_4293`）。
2. HTTP 429 响应必须附带 `Retry-After` 与 `X-RateLimit-Remaining`，供客户端执行退避重试。
3. 推荐同时返回 `X-RateLimit-Limit` 与 `X-RateLimit-Reset`，便于客户端做配额可视化。

| Header | 是否必填 | 规则 | 示例 |
| :--- | :---: | :--- | :--- |
| `Retry-After` | 是 | 整数秒，≥ 1 | `30` |
| `X-RateLimit-Remaining` | 是 | 当前窗口剩余额度 | `0` |
| `X-RateLimit-Limit` | 否 | 当前窗口总额度 | `60` |
| `X-RateLimit-Reset` | 否 | 窗口重置时间（Unix 时间戳秒） | `1712400000` |

### 1.9 空间坐标系统一约定（高德接入）

1. 线索写接口入参 `coord_system` 允许枚举：`WGS84` / `GCJ-02` / `BD-09`。
2. 接入高德地图（AMap）时，客户端必须上传高德原始坐标并声明 `coord_system=GCJ-02`，禁止在客户端二次转换后再标记为 `GCJ-02`。
3. 网关必须在入库前完成坐标标准化：`GCJ-02` / `BD-09` → `WGS84(EPSG:4326)`；转换失败返回 `E_CLUE_4007`。
4. 数据库存储与空间计算统一使用 `WGS84`，围栏判定、去重聚合、轨迹计算均以标准化结果为准。
5. 所有读接口返回的 `coord_system` 固定为 `WGS84`。

### 1.10 AI 模型接入约定（Spring AI Alibaba / 百炼 / 千问）

1. 后端采用 `spring-ai-alibaba-starter` 集成阿里云百炼（DashScope）通义千问大模型，通过 `ChatClient` + `FunctionCallback`（`@Tool`）实现 AI Agent 的 Tool-Use 编排。
2. AI Agent 可通过 Function Calling 调用标准域 API 完成家属侧核心操作（如发布任务、查询轨迹），所有写操作须经 Policy Guard 门禁（SADD §6.6）。
3. 写接口不要求客户端上传模型供应商信息；服务端按治理配置与模型白名单选择 `model_name`。
4. `token_usage` 至少返回 `prompt_tokens`、`completion_tokens`、`total_tokens`，可扩展 `model_name`、`billing_source`、`provider_request_id`。
5. 模型异常场景必须保持现有错误码语义：`E_AI_4292`、`E_AI_4293` 等不得因供应商替换而改变。
6. 降级场景建议返回 `fallback_response.mode`，取值 `RULE_BASED` / `SAFE_GUARD` / `TOOL_DEGRADED`。
7. AI 对话回复必须采用流式下发（SSE 首选、WebSocket 备选）；禁止将大模型全量生成暴露为长时间阻塞的同步 HTTP 响应（FR-AI-012）。

### 1.11 AI Agent 受控执行契约

1. 当 `X-Action-Source=AI_AGENT` 时，`X-Agent-Profile`、`X-Execution-Mode`、`X-Confirm-Level` 必须同时出现。
2. 高风险写接口必须执行策略门禁（Policy Guard）：
    - 校验角色权限。
    - 校验数据归属（patient / task / clue / order 维度）。
    - 校验执行模式与确认等级。
3. 支持 `dry_run=true` 预检查模式（仅策略与参数校验，不产生副作用）；校验失败返回 `E_GOV_4226`。
4. 标记为 `MANUAL_ONLY` 的接口，若 `X-Action-Source=AI_AGENT` 必须拒绝并返回 `E_GOV_4231`。
5. 所有 Agent 写操作必须写审计：`action_source`、`agent_profile`、`execution_mode`、`confirm_level`、`trace_id`、`request_id`。

### 1.12 统一字段映射表（API 语义名 vs 持久化字段）

| API 字段 | 持久化字段 | 说明 |
| :--- | :--- | :--- |
| `reported_by` | `rescue_task.created_by` | 任务发起人 |
| `start_time` | `rescue_task.created_at` | 任务开始时间 |
| `end_time` | `rescue_task.closed_at` | 任务结束时间 |
| `assignee_user_id` | `clue_record.assignee_user_id` | 复核责任人 |
| `operator_user_id` | `sys_log.operator_user_id` | 审计操作人 ID |
| `operator_username` | `sys_log.operator_username` | 审计操作人快照 |
| `confirmed_at`（§3.3 ACCEPT） | `guardian_transfer_request.confirmed_at` | 主监护转移确认时间 |
| `confirmed_at`（§3.4 签收） | `tag_apply_record.received_at` | 家属签收完成时间（工单进入 `RECEIVED`） |

---

## 2. 错误码字典

### 2.1 通用与网关

| 错误码 | HTTP | 含义 |
| :--- | :---: | :--- |
| `E_REQ_4001` | 400 | 请求幂等键格式不合法 |
| `E_REQ_4002` | 400 | Trace-Id 不合法 |
| `E_REQ_4003` | 400 | 客户端伪造内部保留 Header |
| `E_REQ_4005` | 400 | 通用 ID 参数格式不合法（必须为十进制字符串） |
| `E_REQ_4150` | 415 | Content-Type 非 `application/json` |
| `E_REQ_4221` | 422 | 请求时间偏差超限 |
| `E_GOV_4004` | 400 | `device_fingerprint` 格式非法 |
| `E_GOV_4011` | 401 | 鉴权失败或 Authorization 缺失 |
| `E_GOV_4012` | 401 | `ws_ticket` 无效、过期或已使用 |
| `E_GOV_4030` | 403 | 角色或授权不足 |
| `E_GOV_4031` | 403 | 账号已被封禁，禁止访问 |
| `E_GOV_4032` | 403 | 高危操作仅限 `SUPER_ADMIN` |
| `E_GOV_4038` | 403 | CAPTCHA 校验失败 |
| `E_GOV_4039` | 403 | 策略门禁拒绝 Agent 自动执行 |
| `E_GOV_4041` | 404 | 配置键不存在 |
| `E_GOV_4046` | 404 | Outbox DEAD 事件不存在或不可见 |
| `E_GOV_4091` | 409 | 用户名已存在 |
| `E_GOV_4092` | 409 | 邮箱已存在 |
| `E_GOV_4096` | 409 | Outbox 事件当前状态不允许重放（非 `DEAD` 或分区闸门冲突） |
| `E_GOV_4097` | 409 | Agent 确认等级不足 |
| `E_GOV_4098` | 409 | 同分区存在更早未修复 DEAD，需先处理 |
| `E_GOV_4101` | 410 | 邮箱验证链接已过期 |
| `E_GOV_4102` | 410 | 密码重置链接已过期 |
| `E_GOV_4226` | 422 | 预检查失败（`dry_run` 校验未通过） |
| `E_GOV_4231` | 423 | 当前接口仅允许人工执行（`MANUAL_ONLY`） |
| `E_GOV_4291` | 429 | 网关限流（含匿名兜底等高频接口） |
| `E_GOV_4292` | 429 | 网关冷却期未结束（需等待 `Retry-After`） |
| `E_GOV_5002` | 500 | 审计写入失败（关键操作回滚） |
| `E_SYS_5001` | 500 | Outbox 写入失败（数据库不可用） |
| `E_SYS_5002` | 500 | Redis 不可用（幂等缓存不可达） |
| `E_SYS_5031` | 503 | 依赖域查询超时（熔断降级） |

### 2.2 任务域（TASK）

| 错误码 | HTTP | 含义 |
| :--- | :---: | :--- |
| `E_TASK_4001` | 400 | `source` 枚举不合法 |
| `E_TASK_4002` | 400 | `remark` 长度非法 |
| `E_TASK_4003` | 400 | 客户端传入 `reported_by`（禁止） |
| `E_TASK_4004` | 400 | `close_type` 非法 |
| `E_TASK_4005` | 400 | `FALSE_ALARM` 时 `reason` 非法 |
| `E_TASK_4030` | 403 | 任务操作无授权 |
| `E_TASK_4031` | 403 | 无关闭权限（非发起者且非主监护人） |
| `E_TASK_4032` | 403 | 无患者监护授权 |
| `E_TASK_4041` | 404 | 任务不存在 |
| `E_TASK_4091` | 409 | 同患者已存在非终态任务（FR-TASK-001） |
| `E_TASK_4092` | 409 | 任务状态不可迁移（当前状态不允许目标操作） |
| `E_TASK_4093` | 409 | 长期维持任务不支持误报关闭 |
| `E_TASK_4094` | 409 | 并发冲突（CAS 失败，客户端可重试） |
| `E_TASK_4221` | 422 | 误报关闭条件不满足（缺少 `close_reason` 或长度不足） |

### 2.3 线索域（CLUE）

| 错误码 | HTTP | 含义 |
| :--- | :---: | :--- |
| `E_CLUE_4001` | 400 | 纬度非法 |
| `E_CLUE_4002` | 400 | 经度非法 |
| `E_CLUE_4003` | 400 | `description` 非法 |
| `E_CLUE_4004` | 400 | `photo_url` 非白名单 |
| `E_CLUE_4005` | 400 | `short_code` 非法 |
| `E_CLUE_4007` | 400 | 坐标系非法或转换失败 |
| `E_CLUE_4008` | 400 | `override` 参数非法 |
| `E_CLUE_4009` | 400 | `override_reason` 非法 |
| `E_CLUE_4010` | 400 | `reject_reason` 非法 |
| `E_CLUE_4012` | 409 | `entry_token` 无效、重放或已消费 |
| `E_CLUE_4013` | 403 | 凭据绑定校验失败（IP 或设备指纹不匹配） |
| `E_CLUE_4041` | 404 | `tag_code` / 标签状态不可用 |
| `E_CLUE_4042` | 404 | `short_code` 无效或不可公开 |
| `E_CLUE_4043` | 404 | `clue_id` 不存在或不在可复核状态 |
| `E_CLUE_4091` | 409 | 复核状态不可变（已 `OVERRIDDEN` / `REJECTED`） |
| `E_CLUE_4221` | 422 | 非可疑线索不可复核（`suspect_flag=false`） |
| `E_CLUE_5011` | 500 | `override` 重发事件失败 |
| `E_CLUE_5012` | 500 | `reject` 关闭复核失败 |

### 2.4 档案与监护域（PROFILE）

| 错误码 | HTTP | 含义 |
| :--- | :---: | :--- |
| `E_PRO_4001` | 400 | `patient_name` 非法 |
| `E_PRO_4002` | 400 | `birthday` 非法 |
| `E_PRO_4003` | 400 | `gender` 非法 |
| `E_PRO_4004` | 400 | `chronic_diseases` 非法 |
| `E_PRO_4005` | 400 | 围栏参数非法 |
| `E_PRO_4006` | 400 | `relation_role` 非法 |
| `E_PRO_4007` | 400 | 邀请 `reason` 非法 |
| `E_PRO_4008` | 400 | invitation `action` 非法 |
| `E_PRO_4009` | 400 | transfer `reason` 非法 |
| `E_PRO_4010` | 400 | transfer `expire_in_seconds` 非法 |
| `E_PRO_4011` | 400 | transfer confirm `action` 非法 |
| `E_PRO_4012` | 400 | transfer `cancel_reason` 非法 |
| `E_PRO_4013` | 400 | `REJECT` 时 `reject_reason` 非法 |
| `E_PRO_4014` | 400 | `avatar_url` 缺失、非法或尝试清空 |
| `E_PRO_4015` | 400 | `confirm` action 非法（需 `CONFIRM_MISSING` / `CONFIRM_SAFE`） |
| `E_PRO_4030` | 403 | 患者授权不足 |
| `E_PRO_4031` | 403 | 标签高危纠错权限不足（仅 `ADMIN` / `SUPER_ADMIN`） |
| `E_PRO_4032` | 403 | 无主监护管理权限 |
| `E_PRO_4033` | 403 | 非目标受方确认 |
| `E_PRO_4034` | 403 | 非原发起方撤销 |
| `E_PRO_4041` | 404 | `patient_id` 不存在 |
| `E_PRO_4042` | 404 | `invitee_user_id` 不存在 |
| `E_PRO_4043` | 404 | `invite_id` 不存在或状态非法 |
| `E_PRO_4044` | 422 | `target_user_id` 不存在或非 `ACTIVE` 成员 |
| `E_PRO_4045` | 404 | `transfer_request_id` 不存在 |
| `E_PRO_4091` | 409 | 患者档案冲突（重复建档） |
| `E_PRO_4092` | 409 | 标签状态机流转不合法 |
| `E_PRO_4093` | 409 | 标签当前为终态或受限态，禁止普通流转 |
| `E_PRO_4094` | 409 | 重复邀请或已激活成员 |
| `E_PRO_4095` | 409 | 同患者存在其他 `PENDING_CONFIRM` 转移 |
| `E_PRO_4096` | 409 | 邀请状态冲突 |
| `E_PRO_4097` | 409 | 转移请求状态冲突（已过期/已完成/已拒绝） |
| `E_PRO_4098` | 409 | 转移请求不可撤销 |
| `E_PRO_4099` | 409 | 受方 `relation_status` 非 `ACTIVE` |
| `E_PRO_4221` | 422 | 围栏开启时缺失中心点或半径 |

### 2.5 物资域（MAT）

| 错误码 | HTTP | 含义 |
| :--- | :---: | :--- |
| `E_MAT_4001` | 400 | `quantity` 非法 |
| `E_MAT_4002` | 400 | `resource_token` 格式非法 |
| `E_MAT_4003` | 400 | `cancel_reason` 非法 |
| `E_MAT_4004` | 400 | `lost_reason` 非法 |
| `E_MAT_4005` | 400 | `void_reason` 非法 |
| `E_MAT_4030` | 403 | 物资域权限不足 |
| `E_MAT_4032` | 403 | `resource_token` 审计载荷非法 |
| `E_MAT_4041` | 404 | `order_id` 不存在 |
| `E_MAT_4042` | 404 | 标签不存在 |
| `E_MAT_4044` | 404 | `tag_code` 不存在 |
| `E_MAT_4091` | 409 | 工单状态不允许当前流转 |
| `E_MAT_4092` | 409 | 工单状态非法，无法自动收敛 |
| `E_MAT_4094` | 409 | 工单状态冲突，当前状态不允许取消 |
| `E_MAT_4095` | 409 | 工单取消审核驳回或流转冲突 |
| `E_MAT_4096` | 409 | 标签三方一致性校验失败 |
| `E_MAT_4097` | 409 | `resource_token` 中 `patient_id` 与路径不一致 |
| `E_MAT_4098` | 409 | 标签状态冲突，不满足 `LOST` / `VOID` 前置状态 |
| `E_MAT_4221` | 422 | 标签数量与工单不匹配 |
| `E_MAT_4222` | 422 | 发货前置条件不满足（标签不可用） |
| `E_MAT_4223` | 422 | `resource_token` 验签 / 解密失败 |
| `E_MAT_4224` | 422 | 物流异常处置失败 / 库存不足 |
| `E_MAT_4225` | 422 | 批量发号超限（> 10000） |
| `E_MAT_5003` | 500 | 资源链内部解析失败 |

### 2.6 AI 域

| 错误码 | HTTP | 含义 |
| :--- | :---: | :--- |
| `E_AI_4001` | 400 | `session_id` 或 `prompt` 非法 |
| `E_AI_4002` | 400 | AI 会话创建参数非法（`patient_id` / `task_id`） |
| `E_AI_4003` | 400 | `memory_note` 参数非法 |
| `E_AI_4031` | 403 | 内容安全限制（Prompt 或回复触发安全过滤） |
| `E_AI_4033` | 403 | 客户端非法传 `patient_id` / `task_id` 或无会话授权 |
| `E_AI_4041` | 404 | `ai_session` 不存在 |
| `E_AI_4091` | 409 | 会话并发冲突（CAS 失败）或意图已过期 |
| `E_AI_4292` | 429 | 用户频率限制 / 配额已耗尽 |
| `E_AI_4293` | 429 | 患者频率限制 / 配额已耗尽 |
| `E_AI_5011` | 500 | Embedding 维度不匹配 |
| `E_AI_5021` | 500 | 上下文溢出截断后仍超限 |
| `E_AI_5031` | 503 | 模型服务超时（DashScope 响应超时） |

### 2.7 认证与用户域（GOV 子能力）

| 错误码 | HTTP | 含义 |
| :--- | :---: | :--- |
| `E_AUTH_4001` | 400 | `username` 格式非法 |
| `E_AUTH_4002` | 400 | `password` 格式非法 |
| `E_AUTH_4011` | 401 | 登录凭据错误 |
| `E_AUTH_4091` | 409 | `username` 已存在 |
| `E_USR_4001` | 400 | 新旧密码不能相同 |
| `E_USR_4002` | 400 | 新密码不满足强度策略 |
| `E_USR_4003` | 400 | `status` 枚举非法（仅支持 `ACTIVE` / `DISABLED`） |
| `E_USR_4004` | 400 | `role` 枚举非法（仅支持 `FAMILY` / `ADMIN` / `SUPER_ADMIN`） |
| `E_USR_4005` | 400 | `email` / `phone` / `nickname` 格式非法 |
| `E_USR_4011` | 401 | `old_password` 校验失败 |
| `E_USR_4032` | 403 | 普通管理员禁止操作非 `FAMILY` 账号 |
| `E_USR_4033` | 403 | 禁止操作 `SUPER_ADMIN` 账号（不可禁用 / 删除 / 降级） |
| `E_USR_4034` | 403 | 禁止操作自身账号（自禁 / 自删 / 自降级） |
| `E_USR_4035` | 403 | 角色变更仅 `SUPER_ADMIN` 可执行 |
| `E_USR_4041` | 404 | `user_id` 不存在 |
| `E_USR_4091` | 409 | 用户当前状态不允许该操作（如已 `DEACTIVATED` 再禁用） |
| `E_USR_4092` | 409 | 该用户仍持有主监护关系，禁止删除；请先转移主监护 |
| `E_USR_4093` | 409 | 该用户仍有未终态工单或任务，禁止删除 |
| `E_PRO_4035` | 403 | 管理员强制转移目标用户非 `FAMILY` 或非 `ACTIVE` |
| `E_PRO_4036` | 403 | 管理员只读访问受限（无全局患者查阅权限） |
| `E_PRO_4046` | 404 | `force-transfer` 目标 `target_user_id` 不存在 |

### 2.8 通知域（GOV 子能力）

| 错误码 | HTTP | 含义 |
| :--- | :---: | :--- |
| `E_NOTI_4001` | 400 | 通知类型或状态筛选参数非法 |
| `E_NOTI_4030` | 403 | 无通知访问权限 |
| `E_NOTI_4041` | 404 | `notification_id` 不存在 |

---

## 3. HTTP 接口详细定义

> 以下按领域拆分，每个域先给出接口概览表，再逐条展开完整契约。

### 3.1 寻回任务执行域（TASK）

> **底层设计基线**：LLD V2.0 §3（数据模型 `rescue_task`）、DBD V2.0 §4.1（`rescue_task` DDL）
> **需求覆盖**：FR-TASK-001 ~ FR-TASK-005

#### 3.1.0 接口概览

| # | 方法 | URI | 功能 | 鉴权 |
| :---: | :--- | :--- | :--- | :---: |
| 1 | POST | `/api/v1/rescue/tasks` | 发布寻回任务 | JWT |
| 2 | POST | `/api/v1/rescue/tasks/{task_id}/close` | 关闭任务 | JWT |
| 3 | GET | `/api/v1/rescue/tasks/{task_id}/snapshot` | 查询任务快照 | JWT |
| 4 | POST | `/api/v1/rescue/tasks/{task_id}/sustained` | 标记为长期维持 | JWT |
| 5 | GET | `/api/v1/rescue/tasks` | 任务列表查询 | JWT |

---

#### 3.1.1 发布寻回任务

**描述**：家属为指定患者发布一条寻回任务。同一患者不能同时存在非终态任务（FR-TASK-001），状态机初始态 `CREATED`（LLD §3.2.1）。

**HTTP 请求**：`POST /api/v1/rescue/tasks`

**安全性**：需鉴权（JWT），角色 `FAMILY`

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `Authorization` | string | 是 | `Bearer {token}` |
| `X-Request-Id` | string | 是 | 幂等键（HC-03） |
| `X-Trace-Id` | string | 是 | 链路追踪（HC-04） |
| `Content-Type` | string | 是 | `application/json` |

**Request Body (application/json)**：

```json
{
  "patient_id":  "string, 必填, 目标患者 ID",
  "source":      "string, 必填, 枚举: APP / ADMIN_PORTAL / AUTO_UPGRADE",
  "remark":      "string, 可选, 最大长度 500",
  "request_time": "string, 必填, ISO-8601, 客户端本地时间（偏差 ≤ 300s）"
}
```

> **禁止**：客户端传入 `reported_by`，由网关从 JWT 注入（`X-User-Id`）。

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_001",
  "data": {
    "task_id":   "8848",
    "task_no":   "TSK20260419001",
    "patient_id": "1001",
    "status":    "CREATED",
    "source":    "APP",
    "created_at": "2026-04-19T10:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_TASK_4091` | 409 | 同患者已存在非终态任务 |
| `E_TASK_4001` | 400 | `source` 非法枚举 |
| `E_TASK_4002` | 400 | `remark` 超限 |
| `E_TASK_4003` | 400 | 客户端传入 `reported_by` |
| `E_TASK_4032` | 403 | 无患者监护授权 |
| `E_REQ_4221` | 422 | `request_time` 偏差超限 |

**限流与幂等说明**：

- 幂等：`Redis Key = "idem:req:{X-Request-Id}"`，TTL = 24h。重复请求返回首次创建结果。
- 限流：按用户粒度，默认 5 req/min。

**Outbox 事件**：`task.created`（payload 见 §5.1.1）

---

#### 3.1.2 关闭任务

**描述**：将任务从当前状态迁移到终态 `CLOSED_FOUND` 或 `CLOSED_FALSE_ALARM`（FR-TASK-002）。

**HTTP 请求**：`POST /api/v1/rescue/tasks/{task_id}/close`

**安全性**：需鉴权（JWT），角色 `FAMILY`（发起者或主监护人）

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `Authorization` | string | 是 | `Bearer {token}` |
| `X-Request-Id` | string | 是 | 幂等键 |
| `X-Trace-Id` | string | 是 | 链路追踪 |

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `task_id` | string | 是 | 目标任务 ID |

**Request Body (application/json)**：

```json
{
  "close_type":    "string, 必填, 枚举: FOUND / FALSE_ALARM",
  "close_reason":  "string, FALSE_ALARM 时必填, 10-256 字符",
  "found_location": {
    "latitude":  "number, FOUND 时可选, -90 ~ 90",
    "longitude": "number, FOUND 时可选, -180 ~ 180",
    "coord_system": "string, 默认 WGS84"
  },
  "request_time":  "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_002",
  "data": {
    "task_id":    "8848",
    "task_no":    "TSK20260419001",
    "status":     "CLOSED_FOUND",
    "close_type": "FOUND",
    "closed_at":  "2026-04-19T18:30:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_TASK_4041` | 404 | 任务不存在 |
| `E_TASK_4031` | 403 | 非发起者且非主监护人 |
| `E_TASK_4092` | 409 | 当前状态不允许关闭（已为终态） |
| `E_TASK_4093` | 409 | 长期维持（`SUSTAINED`）状态不支持误报关闭 |
| `E_TASK_4004` | 400 | `close_type` 非法 |
| `E_TASK_4005` | 400 | `FALSE_ALARM` 时 `close_reason` 缺失或长度不足 |
| `E_TASK_4221` | 422 | 误报条件不满足 |

**限流与幂等说明**：

- 幂等：同一 `X-Request-Id`，重复返回首次关闭结果。
- 限流：按用户粒度，默认 10 req/min。

**状态迁移说明（LLD §3.2.1）**：

| 当前状态 | close_type | 目标状态 |
| :--- | :--- | :--- |
| `ACTIVE` | `FOUND` | `CLOSED_FOUND` |
| `ACTIVE` | `FALSE_ALARM` | `CLOSED_FALSE_ALARM` |
| `SUSTAINED` | `FOUND` | `CLOSED_FOUND` |
| `SUSTAINED` | `FALSE_ALARM` | ❌ 拒绝（`E_TASK_4093`） |

> **注意**：`CREATED` 态任务不可直接关闭，需先由系统推进到 `ACTIVE`（SRS §5.2.2）。

**Outbox 事件**：`task.closed.found` 或 `task.closed.false_alarm`（payload 见 §5.1.2）

---

#### 3.1.3 查询任务快照

**描述**：查询指定任务的最新快照，包括状态、关联患者信息（脱敏）、线索统计等聚合数据（FR-TASK-003）。

**HTTP 请求**：`GET /api/v1/rescue/tasks/{task_id}/snapshot`

**安全性**：需鉴权（JWT）

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `task_id` | string | 是 | 目标任务 ID |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_003",
  "data": {
    "task_id":     "8848",
    "task_no":     "TSK20260419001",
    "patient_id":  "1001",
    "status":      "MISSING",
    "source":      "APP",
    "reported_by": "2001",
    "remark":      "患者于下午3点外出后未归",
    "created_at":  "2026-04-19T10:00:00Z",
    "closed_at":   null,
    "close_type":  null,
    "close_reason": null,
    "sustained_at": null,
    "patient_snapshot": {
      "patient_name": "张**",
      "gender":       "MALE",
      "age":          78,
      "avatar_url":   "https://oss.example.com/avatar/1001.jpg",
      "short_code":   "A3B7K9",
      "appearance": {
        "height_cm":   170,
        "weight_kg":   65,
        "clothing":    "蓝色外套，灰色裤子",
        "features":    "左耳有助听器"
      }
    },
    "clue_summary": {
      "total_clue_count":    12,
      "valid_clue_count":    8,
      "suspect_clue_count":  2,
      "latest_clue_time":    "2026-04-19T15:30:00Z"
    },
    "trajectory_summary": {
      "point_count":       45,
      "latest_point_time": "2026-04-19T15:30:00Z",
      "bounding_box": {
        "min_lat": 39.90,
        "max_lat": 39.92,
        "min_lng": 116.38,
        "max_lng": 116.41
      }
    },
    "version": 5
  }
}
```

> **PII 脱敏**：`patient_name` 脱敏规则 `@Desensitize(NAME)`，仅首字 + `**`。

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_TASK_4041` | 404 | 任务不存在 |
| `E_TASK_4030` | 403 | 无授权查看 |

---

#### 3.1.4 标记为长期维持

**描述**：将 `ACTIVE` 态任务迁移到 `SUSTAINED`（FR-TASK-004），表示患者长期未归但仍持续关注。

**HTTP 请求**：`POST /api/v1/rescue/tasks/{task_id}/sustained`

**安全性**：需鉴权（JWT），角色 `FAMILY`（发起者或主监护人）

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `Authorization` | string | 是 | `Bearer {token}` |
| `X-Request-Id` | string | 是 | 幂等键 |
| `X-Trace-Id` | string | 是 | 链路追踪 |

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `task_id` | string | 是 | 目标任务 ID |

**Request Body (application/json)**：

```json
{
  "reason":       "string, 可选, 最大长度 500",
  "request_time": "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_004",
  "data": {
    "task_id":      "8848",
    "task_no":      "TSK20260419001",
    "status":       "SUSTAINED",
    "sustained_at": "2026-04-19T16:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_TASK_4041` | 404 | 任务不存在 |
| `E_TASK_4092` | 409 | 当前状态非 `ACTIVE`，不可迁移到 `SUSTAINED` |
| `E_TASK_4031` | 403 | 无权操作 |

**限流与幂等说明**：

- 幂等：同一 `X-Request-Id`，重复返回首次结果。
- 限流：按用户粒度，默认 10 req/min。

**状态迁移说明**：`ACTIVE → SUSTAINED`（仅此单向，不可回退）

**Outbox 事件**：`task.sustained`

---

#### 3.1.5 任务列表查询

**描述**：按条件分页查询当前用户的任务列表（含排序与筛选）。

**HTTP 请求**：`GET /api/v1/rescue/tasks`

**安全性**：需鉴权（JWT）

**Query Parameters**：

| 字段名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :---: | :--- | :--- |
| `patient_id` | string | 否 | — | 按患者筛选 |
| `status` | string | 否 | — | 枚举：`CREATED` / `ACTIVE` / `SUSTAINED` / `CLOSED_FOUND` / `CLOSED_FALSE_ALARM` |
| `source` | string | 否 | — | 枚举：`APP` / `ADMIN_PORTAL` / `AUTO_UPGRADE` |
| `page_no` | int | 否 | `1` | 页码 |
| `page_size` | int | 否 | `20` | 每页条数（≤ 100） |
| `sort_by` | string | 否 | `created_at` | 排序字段：`created_at` / `closed_at` |
| `sort_order` | string | 否 | `desc` | `asc` / `desc` |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_005",
  "data": {
    "items": [
      {
        "task_id":    "8848",
        "task_no":    "TSK20260419001",
        "patient_id": "1001",
        "patient_name": "张**",
        "status":     "ACTIVE",
        "source":     "APP",
        "reported_by": "2001",
        "created_at": "2026-04-19T10:00:00Z",
        "closed_at":  null
      }
    ],
    "page_no":   1,
    "page_size": 20,
    "total":     1,
    "has_next":  false
  }
}
```

> **PII 脱敏**：`patient_name` → `@Desensitize(NAME)`。

**异常响应**：通用参数校验错误（`E_REQ_4005` 等）。

---

### 3.2 线索与时空研判域（CLUE）

> **底层设计基线**：LLD V2.0 §4（数据模型 `clue_record` / `patient_trajectory`）、DBD V2.0 §4.2 ~ §4.3
> **需求覆盖**：FR-CLUE-001 ~ FR-CLUE-010

#### 3.2.0 接口概览

| # | 方法 | URI | 功能 | 鉴权 |
| :---: | :--- | :--- | :--- | :---: |
| 1 | GET | `/r/{resource_token}` | 二维码路由（扫码入口） | 匿名 |
| 2 | POST | `/api/v1/public/clues/manual-entry` | 手动短码录入 | 匿名 |
| 3 | POST | `/api/v1/clues/report` | 上报线索 | JWT / entry_token |
| 4 | POST | `/api/v1/clues/{clue_id}/override` | 复核覆写（可疑→有效） | JWT |
| 5 | POST | `/api/v1/clues/{clue_id}/reject` | 复核驳回（可疑→无效） | JWT |
| 6 | GET | `/api/v1/clues` | 线索列表查询 | JWT |
| 7 | GET | `/api/v1/rescue/tasks/{task_id}/trajectory/latest` | 轨迹查询 | JWT |

---

#### 3.2.1 二维码路由

**描述**：好心人扫描标签二维码后，网关验签 `resource_token`、签发一次性 `entry_token`（HttpOnly Cookie），然后 302 重定向至线索上报 H5 页面（FR-CLUE-002, LLD §4.3.2）。

**HTTP 请求**：`GET /r/{resource_token}`

**安全性**：匿名

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `resource_token` | string | 是 | 标签载荷密文（Base64URL 编码的 AES-256-GCM 加密 payload） |

**处理逻辑**：

1. 解密 `resource_token`，校验 `kid`、`tag_code`、`tag_type`。
2. 查询标签状态 `tag_asset`。
3. 通过 `short_code` 查询患者信息。
4. 签发 `entry_token`（JWT），存入 HttpOnly Cookie（`Secure; SameSite=Strict`）。TTL 由 `security.entry_token.ttl_seconds` 配置（默认 120s）。
5. 根据标签状态路由：
   - `BOUND` → 302 重定向至 `/p/{short_code}/clues/new`
   - `LOST` / `SUSPECTED_LOST` → 302 重定向至 `/p/{short_code}/emergency/report`
   - `UNBOUND` / `ALLOCATED` / `VOIDED` → 拦截页（不下发 `entry_token`）

**Response (HTTP 302)**：

```
HTTP/1.1 302 Found
Location: /p/A3B7K9/clues/new
Set-Cookie: entry_token=eyJ...; HttpOnly; Secure; SameSite=Strict; Max-Age=120; Path=/api/v1/clues
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_MAT_4223` | 422 | `resource_token` 验签 / 解密失败 |
| `E_CLUE_4041` | 404 | 标签不存在或状态不可用 |
| `E_CLUE_4042` | 404 | `short_code` 无效 |

**限流说明**：按 IP 粒度限流，默认 60 req/min/IP（HC-06）。

---

#### 3.2.2 手动短码录入

**描述**：好心人通过手动输入 6 位短码进入线索上报页面（FR-CLUE-003, LLD §4.3.3）。

**HTTP 请求**：`POST /api/v1/public/clues/manual-entry`

**安全性**：匿名（含设备指纹与 CAPTCHA）

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `X-Trace-Id` | string | 是 | 链路追踪 |

**Request Body (application/json)**：

```json
{
  "short_code":         "string, 必填, 6 位字母数字",
  "device_fingerprint": "string, 必填, 设备指纹（HC-06）",
  "captcha_token":      "string, 必填, 人机验证凭据"
}
```

**处理逻辑**（LLD §4.4.3 风控三层检查）：

1. **L1 频控**：同 IP 1 分钟 ≤ 10 次、同设备 1 小时 ≤ 20 次。
2. **L2 CAPTCHA**：校验 `captcha_token` 有效性。
3. **L3 冷却窗口**：同 `short_code` 冷却期内拒绝（配置键 `risk.manual_entry.cooldown.minutes`）。
4. 查询 `patient_profile`，确认 `short_code` 有效且患者为已建档激活状态。
5. 签发 `entry_token`，与扫码路径等效。

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_006",
  "data": {
    "redirect_url": "/p/A3B7K9/clues/new",
    "entry_token_set": true
  }
}
```

> 同时在 Response Header 中设置 `Set-Cookie: entry_token=eyJ...; HttpOnly; Secure; SameSite=Strict; Max-Age=120; Path=/api/v1/clues`

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_CLUE_4005` | 400 | `short_code` 格式非法 |
| `E_GOV_4004` | 400 | `device_fingerprint` 非法 |
| `E_GOV_4038` | 403 | CAPTCHA 校验失败 |
| `E_GOV_4291` | 429 | IP / 设备频控超限 |
| `E_GOV_4292` | 429 | `short_code` 冷却期未结束 |
| `E_CLUE_4042` | 404 | `short_code` 无效或患者非公开 |

**限流说明**：IP 限流 + 设备指纹限流 + 短码冷却（三层防护），具体阈值由 `sys_config` 下发（HC-05）。

---

#### 3.2.3 上报线索

**描述**：好心人或家属上报一条位置线索，系统执行 AI 初筛（反漂移）与去重聚合，生成 `clue_record` 并可能生成 `patient_trajectory` 轨迹点（FR-CLUE-001, LLD §4.3.1）。

**HTTP 请求**：`POST /api/v1/clues/report`

**安全性**：

- 已登录家属：JWT
- 匿名好心人：`entry_token`（Cookie）或 `X-Anonymous-Token`

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `Authorization` | string | 条件 | 已登录时 `Bearer {token}` |
| `X-Anonymous-Token` | string | 条件 | 非浏览器匿名端可用 |
| `X-Request-Id` | string | 是 | 幂等键 |
| `X-Trace-Id` | string | 是 | 链路追踪 |

**Request Body (application/json)**：

```json
{
  "patient_id":          "string, 必填",
  "task_id":             "string, 可选, 指定关联任务",
  "latitude":            "number, 必填, -90.0 ~ 90.0",
  "longitude":           "number, 必填, -180.0 ~ 180.0",
  "coord_system":        "string, 默认 GCJ-02, 枚举: WGS84 / GCJ-02 / BD-09",
  "description":         "string, 可选, 最大长度 1000",
  "photo_urls":          ["string, source_type=MANUAL 时必填, OSS 白名单 URL, 最多 9 张"],
  "device_fingerprint":  "string, 匿名必填（HC-06）",
  "request_time":        "string, 必填, ISO-8601",
  "tag_only":            "boolean, 可选, 默认 false, 仅发现标识未见到患者（FR-CLUE-004）"
}
```

**处理逻辑**（LLD §4.4.1）：

1. 坐标标准化：`GCJ-02` / `BD-09` → `WGS84`。
2. 反漂移检测（LLD §4.7）：若坐标漂移判定异常，`drift_flag = true`，线索自动标记为 `REJECTED`。
3. 时空去重聚合（LLD §4.4.1 Step 5-7）：500m + 30min 窗口内重复线索聚合，生成 `parent_clue_id`。
4. AI 初筛：疑似匹配度低的标记 `suspect_flag = true`（需人工复核）。
5. 有效线索写入 `patient_trajectory`。
6. Outbox 发布 `clue.reported.raw` → 后续异步产生 `clue.validated` / `track.updated` / `fence.breached`。

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_007",
  "data": {
    "clue_id":      "50001",
    "clue_no":      "CLU20260419001",
    "patient_id":   "1001",
    "task_id":      "8848",
    "status":       "VALID",
    "suspect_flag": false,
    "drift_flag":   false,
    "created_at":   "2026-04-19T15:30:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_CLUE_4001` | 400 | 纬度超限 |
| `E_CLUE_4002` | 400 | 经度超限 |
| `E_CLUE_4003` | 400 | `description` 超限 |
| `E_CLUE_4004` | 400 | `photo_urls` 非白名单 |
| `E_CLUE_4007` | 400 | 坐标系非法或转换失败 |
| `E_CLUE_4012` | 409 | `entry_token` 无效、重放或已消费 |
| `E_CLUE_4013` | 403 | 凭据绑定校验失败（IP / 设备指纹不匹配） |
| `E_GOV_4004` | 400 | `device_fingerprint` 格式非法（匿名时） |

**限流与幂等说明**：

- 幂等：`Redis Key = "idem:req:{X-Request-Id}"`，TTL = 24h。
- 限流：匿名 10 req/min/device，已登录 30 req/min/user。

**Outbox 事件**：`clue.reported.raw`（payload 见 §5.2.1）

---

#### 3.2.4 复核覆写（可疑 → 有效）

**描述**：家属对 AI 标记为可疑的线索进行人工复核，确认为有效（FR-CLUE-007, LLD §4.3.4）。

**HTTP 请求**：`POST /api/v1/clues/{clue_id}/override`

**安全性**：需鉴权（JWT），角色 `FAMILY`

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `Authorization` | string | 是 | `Bearer {token}` |
| `X-Request-Id` | string | 是 | 幂等键 |
| `X-Trace-Id` | string | 是 | 链路追踪 |

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `clue_id` | string | 是 | 目标线索 ID |

**Request Body (application/json)**：

```json
{
  "override_reason": "string, 必填, 5-256",
  "request_time":    "string, 必填, ISO-8601"
}
```

**处理逻辑**：

1. 校验线索存在且 `suspect_flag = true`、`status = VALID`。
2. 设置 `suspect_flag = false`，写入 `reviewed_at`、`reviewer_user_id`。
3. 补发 `clue.validated` 事件（写入轨迹链路），补写 `patient_trajectory`。

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_008",
  "data": {
    "clue_id":      "50002",
    "clue_no":      "CLU20260419002",
    "status":       "OVERRIDDEN",
    "suspect_flag": false,
    "reviewed_at":  "2026-04-19T16:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_CLUE_4043` | 404 | 线索不存在或非可复核状态 |
| `E_CLUE_4221` | 422 | 非可疑线索（`suspect_flag = false`） |
| `E_CLUE_4091` | 409 | 已复核过（`OVERRIDDEN` / `REJECTED`） |
| `E_CLUE_4008` | 400 | `override_reason` 非法 |

---

#### 3.2.5 复核驳回（可疑 → 无效）

**描述**：家属对可疑线索进行人工复核，确认为无效并驳回（FR-CLUE-007, LLD §4.3.5）。

**HTTP 请求**：`POST /api/v1/clues/{clue_id}/reject`

**安全性**：需鉴权（JWT），角色 `FAMILY`

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `clue_id` | string | 是 | 目标线索 ID |

**Request Body (application/json)**：

```json
{
  "reject_reason": "string, 必填, 5-256",
  "request_time":  "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_009",
  "data": {
    "clue_id":      "50002",
    "clue_no":      "CLU20260419002",
    "status":       "REJECTED",
    "reviewed_at":  "2026-04-19T16:05:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_CLUE_4043` | 404 | 线索不存在或非可复核状态 |
| `E_CLUE_4221` | 422 | 非可疑线索 |
| `E_CLUE_4091` | 409 | 已复核过 |
| `E_CLUE_4010` | 400 | `reject_reason` 非法 |

---

#### 3.2.6 线索列表查询

**描述**：分页查询指定任务或患者下的线索记录。

**HTTP 请求**：`GET /api/v1/clues`

**安全性**：需鉴权（JWT）

**Query Parameters**：

| 字段名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :---: | :--- | :--- |
| `task_id` | string | 否 | — | 按任务筛选 |
| `patient_id` | string | 否 | — | 按患者筛选 |
| `status` | string | 否 | — | 枚举：`VALID` / `OVERRIDDEN` / `REJECTED` |
| `suspect_flag` | boolean | 否 | — | 仅可疑 / 仅确认 |
| `page_no` | int | 否 | `1` | 页码 |
| `page_size` | int | 否 | `20` | 每页条数（≤ 100） |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_010",
  "data": {
    "items": [
      {
        "clue_id":      "50001",
        "clue_no":      "CLU20260419001",
        "task_id":      "8848",
        "patient_id":   "1001",
        "status":       "VALID",
        "suspect_flag": false,
        "drift_flag":   false,
        "latitude":     39.9100,
        "longitude":    116.3900,
        "coord_system": "WGS84",
        "description":  "在公园东门附近见到一位老人",
        "photo_urls":   ["https://oss.example.com/clue/50001_1.jpg"],
        "reporter_type": "ANONYMOUS",
        "created_at":   "2026-04-19T15:30:00Z"
      }
    ],
    "page_no":   1,
    "page_size": 20,
    "total":     1,
    "has_next":  false
  }
}
```

---

#### 3.2.7 轨迹查询

**描述**：查询指定任务关联的患者轨迹点列表（FR-CLUE-006, LLD §4 `patient_trajectory`）。

**HTTP 请求**：`GET /api/v1/rescue/tasks/{task_id}/trajectory/latest`

**安全性**：需鉴权（JWT）

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `task_id` | string | 是 | 目标任务 ID |

**Query Parameters**：

| 字段名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :---: | :--- | :--- |
| `since` | string | 否 | — | ISO-8601，查询此时间之后的轨迹点 |
| `after_version` | int | 否 | — | 增量拉取（大于此 version 的轨迹点） |
| `page_size` | int | 否 | `50` | 每页条数（≤ 200） |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_011",
  "data": {
    "items": [
      {
        "trajectory_id": "70001",
        "patient_id":    "1001",
        "task_id":       "8848",
        "clue_id":       "50001",
        "latitude":      39.9100,
        "longitude":     116.3900,
        "coord_system":  "WGS84",
        "recorded_at":   "2026-04-19T15:30:00Z",
        "source_type":   "CLUE",
        "version":       5
      }
    ],
    "page_size":   50,
    "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNi0wNC0xOVQxNTozMDowMFoiLCJpZCI6IjcwMDAxIn0=",
    "has_next":    false
  }
}
```

> 轨迹接口采用 **Cursor 模式**分页（§1.5），支持 `after_version` 增量拉取。

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_TASK_4041` | 404 | 任务不存在 |
| `E_TASK_4030` | 403 | 无授权查看 |

---

### 3.3 患者档案与监护域（PROFILE）

> **底层设计基线**：LLD V2.0 §5（数据模型 `patient_profile` / `guardian_relation` / `guardian_invitation` / `guardian_transfer_request`）、DBD V2.0 §4.4 ~ §4.7
> **需求覆盖**：FR-PRO-001 ~ FR-PRO-010、FR-TASK-003（外观快照）

#### 3.3.0 接口概览

| # | 方法 | URI | 功能 | 鉴权 |
| :---: | :--- | :--- | :--- | :---: |
| 1 | POST | `/api/v1/patients` | 创建患者档案 | JWT |
| 2 | PUT | `/api/v1/patients/{patient_id}/profile` | 更新患者基础档案 | JWT |
| 3 | PUT | `/api/v1/patients/{patient_id}/appearance` | 更新外观特征 | JWT |
| 4 | PUT | `/api/v1/patients/{patient_id}/fence` | 设置电子围栏 | JWT |
| 5 | POST | `/api/v1/patients/{patient_id}/missing-pending/confirm` | 确认走失 / 确认安全 | JWT |
| 6 | POST | `/api/v1/patients/{patient_id}/guardians/invitations` | 发起监护邀请 | JWT |
| 7 | POST | `/api/v1/patients/{patient_id}/guardians/invitations/{invite_id}/respond` | 响应监护邀请 | JWT |
| 8 | POST | `/api/v1/patients/{patient_id}/guardians/primary-transfer` | 发起主监护转移 | JWT |
| 9 | POST | `/api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/respond` | 响应主监护转移 | JWT |
| 10 | POST | `/api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/cancel` | 撤销主监护转移 | JWT |
| 11 | DELETE | `/api/v1/patients/{patient_id}/guardians/{user_id}` | 移除监护成员 | JWT |
| 12 | GET | `/api/v1/patients/{patient_id}` | 查询患者档案 | JWT |
| 13 | GET | `/api/v1/patients` | 患者列表查询 | JWT |
| 14 | DELETE | `/api/v1/patients/{patient_id}` | 逻辑删除患者档案 | JWT |
| 15 | GET | `/api/v1/admin/patients` | 管理员全局患者档案列表 | JWT(ADMIN) |
| 16 | GET | `/api/v1/admin/patients/{patient_id}` | 管理员查看患者档案详情 | JWT(ADMIN) |
| 17 | POST | `/api/v1/admin/patients/{patient_id}/guardians/force-transfer` | 管理员强制转移主监护权 | JWT(SUPER_ADMIN) |

---

#### 3.3.1 创建患者档案 

**描述**：当前用户为患者创建档案，自动成为主监护人（FR-PRO-001, LLD §5.3.1）。

**HTTP 请求**：`POST /api/v1/patients`

**安全性**：需鉴权（JWT），角色 `FAMILY`

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `Authorization` | string | 是 | `Bearer {token}` |
| `X-Request-Id` | string | 是 | 幂等键 |
| `X-Trace-Id` | string | 是 | 链路追踪 |

**Request Body (application/json)**：

```json
{
  "patient_name":     "string, 必填, 2-64",
  "gender":           "string, 必填, 枚举: MALE / FEMALE / UNKNOWN",
  "birthday":         "string, 必填, yyyy-MM-dd",
  "id_card_hash":     "string, 可选, SHA-256 哈希",
  "chronic_diseases": "string, 可选, 最大长度 500",
  "medication":       "string, 可选, 最大长度 500",
  "allergy":          "string, 可选, 最大长度 500",
  "emergency_contact_phone": "string, 可选, 手机号格式",
  "avatar_url":       "string, 必填, OSS URL",
  "long_text_profile": "string, 可选, 最大长度 5000, 用于 AI 向量化",
  "appearance": {
    "height_cm":  "integer, 可选, 50-250",
    "weight_kg":  "integer, 可选, 10-300",
    "clothing":   "string, 可选, 最大长度 500",
    "features":   "string, 可选, 最大长度 500"
  },
  "fence": {
    "enabled":     "boolean, 默认 false",
    "center_lat":  "number, 围栏启用时必填, -90 ~ 90",
    "center_lng":  "number, 围栏启用时必填, -180 ~ 180",
    "radius_m":    "integer, 围栏启用时必填, 100-50000",
    "coord_system": "string, 默认 WGS84"
  },
  "request_time": "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_012",
  "data": {
    "patient_id":  "1001",
    "profile_no":  "PRO20260419001",
    "short_code":  "A3B7K9",
    "status":      "NORMAL",
    "created_at":  "2026-04-19T09:00:00Z",
    "guardian": {
      "user_id":         "2001",
      "relation_role":   "PRIMARY_GUARDIAN",
      "relation_status": "ACTIVE"
    }
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_PRO_4001` | 400 | `patient_name` 非法 |
| `E_PRO_4002` | 400 | `birthday` 非法 |
| `E_PRO_4003` | 400 | `gender` 非法 |
| `E_PRO_4014` | 400 | `avatar_url` 缺失或非法 |
| `E_PRO_4221` | 422 | 围栏启用但缺中心点 / 半径 |
| `E_PRO_4091` | 409 | 重复建档 |

**Outbox 事件**：`profile.created`（payload 见 §5.3.1）

---

#### 3.3.2 更新患者基础档案

**描述**：更新患者的基础信息（FR-PRO-001, LLD §5.3.7）。

**HTTP 请求**：`PUT /api/v1/patients/{patient_id}/profile`

**安全性**：需鉴权（JWT），角色 `FAMILY`（需监护权限）

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `patient_id` | string | 是 | 目标患者 ID |

**Request Body (application/json)**：

```json
{
  "patient_name":     "string, 可选, 2-64",
  "gender":           "string, 可选",
  "birthday":         "string, 可选, yyyy-MM-dd",
  "chronic_diseases": "string, 可选",
  "medication":       "string, 可选",
  "allergy":          "string, 可选",
  "emergency_contact_phone": "string, 可选",
  "avatar_url":       "string, 可选（传入时不可为空字符串）",
  "long_text_profile": "string, 可选, 最大长度 5000",
  "request_time":     "string, 必填, ISO-8601"
}
```

> 仅传入的字段会被更新，未传入字段保持原值。

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_013",
  "data": {
    "patient_id": "1001",
    "profile_no": "PRO20260419001",
    "updated_at": "2026-04-19T11:00:00Z",
    "version":    3
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_PRO_4041` | 404 | 患者不存在 |
| `E_PRO_4030` | 403 | 无监护权限 |
| `E_PRO_4014` | 400 | `avatar_url` 传入空字符串 |

**Outbox 事件**：`profile.updated`（触发 AI 向量化重建）

---

#### 3.3.3 更新外观特征

**描述**：单独更新患者的外观快照（FR-TASK-003 快照投影, LLD §5.3.6）。该接口在任务进行中由家属实时更新穿着等信息。

**HTTP 请求**：`PUT /api/v1/patients/{patient_id}/appearance`

**安全性**：需鉴权（JWT），角色 `FAMILY`

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `patient_id` | string | 是 | 目标患者 ID |

**Request Body (application/json)**：

```json
{
  "height_cm":  "integer, 可选, 50-250",
  "weight_kg":  "integer, 可选, 10-300",
  "clothing":   "string, 可选, 最大长度 500",
  "features":   "string, 可选, 最大长度 500",
  "request_time": "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_014",
  "data": {
    "patient_id": "1001",
    "updated_at": "2026-04-19T11:30:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_PRO_4041` | 404 | 患者不存在 |
| `E_PRO_4030` | 403 | 无监护权限 |

**Outbox 事件**：`profile.appearance.updated`（任务快照投影更新）

---

#### 3.3.4 设置电子围栏

**描述**：为患者设置 / 更新 / 关闭围栏配置（FR-PRO-010, LLD §5.3.5）。

**HTTP 请求**：`PUT /api/v1/patients/{patient_id}/fence`

**安全性**：需鉴权（JWT），角色 `FAMILY`

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `patient_id` | string | 是 | 目标患者 ID |

**Request Body (application/json)**：

```json
{
  "enabled":      "boolean, 必填",
  "center_lat":   "number, 启用时必填, -90 ~ 90",
  "center_lng":   "number, 启用时必填, -180 ~ 180",
  "radius_m":     "integer, 启用时必填, 100-50000",
  "coord_system": "string, 默认 WGS84",
  "request_time": "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_015",
  "data": {
    "patient_id":  "1001",
    "fence_enabled": true,
    "updated_at":  "2026-04-19T12:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_PRO_4041` | 404 | 患者不存在 |
| `E_PRO_4030` | 403 | 无监护权限 |
| `E_PRO_4005` | 400 | 围栏参数非法 |
| `E_PRO_4221` | 422 | 启用时缺中心点或半径 |

---

#### 3.3.5 确认走失 / 确认安全

**描述**：当围栏告警触发 `MISSING_PENDING` 状态后，家属确认走失或确认安全（LLD §5.2.1 状态机、§5.3.4）。

**HTTP 请求**：`POST /api/v1/patients/{patient_id}/missing-pending/confirm`

**安全性**：需鉴权（JWT），角色 `FAMILY`

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `patient_id` | string | 是 | 目标患者 ID |

**Request Body (application/json)**：

```json
{
  "action":       "string, 必填, 枚举: CONFIRM_MISSING / CONFIRM_SAFE",
  "request_time": "string, 必填, ISO-8601"
}
```

**处理逻辑**：

- `CONFIRM_MISSING`：患者状态 `MISSING_PENDING → MISSING`，自动发布 `AUTO_UPGRADE` 任务（Outbox `task.auto_upgrade_requested`）。
- `CONFIRM_SAFE`：患者状态 `MISSING_PENDING → NORMAL`，事件 `patient.confirmed_safe`。

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_016",
  "data": {
    "patient_id":  "1001",
    "new_status":  "MISSING",
    "task_auto_created": true,
    "task_id":     "8849"
  }
}
```

> 当 `action=CONFIRM_SAFE` 时，`task_auto_created=false`，`task_id=null`。

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_PRO_4041` | 404 | 患者不存在 |
| `E_PRO_4030` | 403 | 无监护权限 |
| `E_PRO_4015` | 400 | `action` 非法 |
| `E_PRO_4092` | 409 | 患者当前状态非 `MISSING_PENDING` |

---

#### 3.3.6 发起监护邀请

**描述**：主监护人邀请其他已注册用户成为共同监护人（FR-PRO-006, LLD §5.3.2）。

**HTTP 请求**：`POST /api/v1/patients/{patient_id}/guardians/invitations`

**安全性**：需鉴权（JWT），角色 `FAMILY`（主监护人）

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `patient_id` | string | 是 | 目标患者 ID |

**Request Body (application/json)**：

```json
{
  "invitee_user_id":  "string, 必填",
  "relation_role":    "string, 必填, 枚举: GUARDIAN",
  "reason":           "string, 可选, 最大长度 500",
  "request_time":     "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_017",
  "data": {
    "invite_id":   "3001",
    "patient_id":  "1001",
    "invitee_user_id": "2002",
    "status":      "PENDING",
    "expires_at":  "2026-04-26T09:00:00Z",
    "created_at":  "2026-04-19T09:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_PRO_4041` | 404 | 患者不存在 |
| `E_PRO_4032` | 403 | 非主监护人 |
| `E_PRO_4042` | 404 | `invitee_user_id` 不存在 |
| `E_PRO_4094` | 409 | 已是成员或存在待处理邀请 |
| `E_PRO_4006` | 400 | `relation_role` 非法 |

---

#### 3.3.7 响应监护邀请

**描述**：被邀请人接受或拒绝监护邀请（FR-PRO-006, LLD §5.3.2）。

**HTTP 请求**：`POST /api/v1/patients/{patient_id}/guardians/invitations/{invite_id}/respond`

**安全性**：需鉴权（JWT），被邀请人本人

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `patient_id` | string | 是 | 目标患者 ID |
| `invite_id` | string | 是 | 邀请 ID |

**Request Body (application/json)**：

```json
{
  "action":         "string, 必填, 枚举: ACCEPT / REJECT",
  "reject_reason":  "string, REJECT 时必填, 5-256",
  "request_time":   "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_018",
  "data": {
    "invite_id": "3001",
    "status":    "ACCEPTED",
    "responded_at": "2026-04-19T10:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_PRO_4043` | 404 | 邀请不存在或已处理 |
| `E_PRO_4096` | 409 | 邀请状态非 `PENDING` |
| `E_PRO_4008` | 400 | `action` 非法 |
| `E_PRO_4013` | 400 | `REJECT` 时 `reject_reason` 非法 |

---

#### 3.3.8 发起主监护转移

**描述**：主监护人发起将主监护权转移给另一名 `ACTIVE` 成员（FR-PRO-007, LLD §5.3.3）。

**HTTP 请求**：`POST /api/v1/patients/{patient_id}/guardians/primary-transfer`

**安全性**：需鉴权（JWT），角色 `FAMILY`（主监护人）

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `patient_id` | string | 是 | 目标患者 ID |

**Request Body (application/json)**：

```json
{
  "target_user_id":      "string, 必填, 受让人 user_id",
  "reason":              "string, 可选, 最大长度 500",
  "expire_in_seconds":   "integer, 可选, 默认 604800 (7天), 最大 2592000 (30天)",
  "request_time":        "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_019",
  "data": {
    "transfer_request_id": "4001",
    "patient_id":   "1001",
    "from_user_id": "2001",
    "to_user_id":   "2002",
    "status":       "PENDING_CONFIRM",
    "expires_at":   "2026-04-26T09:00:00Z",
    "created_at":   "2026-04-19T09:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_PRO_4041` | 404 | 患者不存在 |
| `E_PRO_4032` | 403 | 非主监护人 |
| `E_PRO_4044` | 422 | 受让人不存在或非 `ACTIVE` 成员 |
| `E_PRO_4095` | 409 | 同患者已存在 `PENDING_CONFIRM` 转移 |
| `E_PRO_4010` | 400 | `expire_in_seconds` 非法 |

---

#### 3.3.9 响应主监护转移

**描述**：受让人确认接受或拒绝主监护权转移（FR-PRO-007, LLD §5.3.3）。

**HTTP 请求**：`POST /api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/respond`

**安全性**：需鉴权（JWT），受让人本人

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `patient_id` | string | 是 | 目标患者 ID |
| `transfer_request_id` | string | 是 | 转移请求 ID |

**Request Body (application/json)**：

```json
{
  "action":        "string, 必填, 枚举: ACCEPT / REJECT",
  "reject_reason": "string, REJECT 时必填, 5-256",
  "request_time":  "string, 必填, ISO-8601"
}
```

**处理逻辑**（`ACCEPT` 时）：

1. 原主监护人 `relation_role` → `GUARDIAN`。
2. 受让人 `relation_role` → `PRIMARY_GUARDIAN`。
3. 转移请求状态 → `COMPLETED`。
4. 同事务 Outbox 发布 `guardian.primary.transferred`。

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_020",
  "data": {
    "transfer_request_id": "4001",
    "status":       "COMPLETED",
    "confirmed_at": "2026-04-19T10:30:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_PRO_4045` | 404 | 转移请求不存在 |
| `E_PRO_4033` | 403 | 非受让人本人 |
| `E_PRO_4097` | 409 | 请求已过期 / 已完成 / 已拒绝 |
| `E_PRO_4011` | 400 | `action` 非法 |
| `E_PRO_4013` | 400 | `REJECT` 时 `reject_reason` 非法 |

---

#### 3.3.10 撤销主监护转移

**描述**：发起人在受让人响应前撤销转移请求（LLD §5.3.3）。

**HTTP 请求**：`POST /api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/cancel`

**安全性**：需鉴权（JWT），发起人本人

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `patient_id` | string | 是 | 目标患者 ID |
| `transfer_request_id` | string | 是 | 转移请求 ID |

**Request Body (application/json)**：

```json
{
  "cancel_reason": "string, 可选, 最大长度 500",
  "request_time":  "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_021",
  "data": {
    "transfer_request_id": "4001",
    "status":       "REVOKED",
    "revoked_at": "2026-04-19T09:30:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_PRO_4045` | 404 | 转移请求不存在 |
| `E_PRO_4034` | 403 | 非发起人本人 |
| `E_PRO_4098` | 409 | 请求已不可撤销（已完成 / 已拒绝 / 已过期） |

---

#### 3.3.11 移除监护成员

**描述**：主监护人将指定家属从患者的监护成员列表中移除（FR-PRO-006, BR-006）。移除时，该成员所有 `PENDING_CONFIRM` 状态的转移请求必须同事务失效。

**HTTP 请求**：`DELETE /api/v1/patients/{patient_id}/guardians/{user_id}`

**安全性**：需鉴权（JWT），角色 `PRIMARY_GUARDIAN`

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `patient_id` | string | 是 | 目标患者 ID |
| `user_id` | string | 是 | 待移除的监护成员 ID |

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `Authorization` | string | 是 | `Bearer {token}` |
| `X-Request-Id` | string | 是 | 幂等键（HC-03） |
| `X-Trace-Id` | string | 是 | 链路追踪（HC-04） |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_022",
  "data": {
    "patient_id": "1001",
    "removed_user_id": "2003",
    "invalidated_transfer_requests": 1
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_PRO_4046` | 404 | 监护关系不存在 |
| `E_PRO_4035` | 403 | 非主监护人 |
| `E_PRO_4099` | 409 | 不可移除自身（主监护人） |

---

#### 3.3.12 查询患者档案

**描述**：查询单个患者的完整档案信息（含监护人列表、围栏配置等）。

**HTTP 请求**：`GET /api/v1/patients/{patient_id}`

**安全性**：需鉴权（JWT）

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `patient_id` | string | 是 | 目标患者 ID |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_022",
  "data": {
    "patient_id":  "1001",
    "profile_no":  "PRO20260419001",
    "patient_name": "张**",
    "gender":      "MALE",
    "birthday":    "1948-05-15",
    "age":         78,
    "avatar_url":  "https://oss.example.com/avatar/1001.jpg",
    "short_code":  "A3B7K9",
    "status":      "NORMAL",
    "chronic_diseases": "高血压，糖尿病",
    "medication":  "降压药",
    "allergy":     "青霉素",
    "emergency_contact_phone": "138****5678",
    "appearance": {
      "height_cm":  170,
      "weight_kg":  65,
      "clothing":   "蓝色外套",
      "features":   "左耳助听器"
    },
    "fence": {
      "enabled":    true,
      "center_lat": 39.9100,
      "center_lng": 116.3900,
      "radius_m":   1000,
      "coord_system": "WGS84"
    },
    "guardians": [
      {
        "user_id":         "2001",
        "nickname":        "张家属",
        "relation_role":   "PRIMARY_GUARDIAN",
        "relation_status": "ACTIVE",
        "joined_at":       "2026-04-19T09:00:00Z"
      },
      {
        "user_id":         "2002",
        "nickname":        "李家属",
        "relation_role":   "GUARDIAN",
        "relation_status": "ACTIVE",
        "joined_at":       "2026-04-19T10:00:00Z"
      }
    ],
    "created_at": "2026-04-19T09:00:00Z",
    "updated_at": "2026-04-19T11:00:00Z",
    "version":    3
  }
}
```

> **PII 脱敏**：`patient_name` → `@Desensitize(NAME)`，`emergency_contact_phone` → `@Desensitize(PHONE)`。

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_PRO_4041` | 404 | 患者不存在 |
| `E_PRO_4030` | 403 | 无授权查看 |

---

#### 3.3.13 患者列表查询

**描述**：查询当前用户名下的患者列表。

**HTTP 请求**：`GET /api/v1/patients`

**安全性**：需鉴权（JWT）

**Query Parameters**：

| 字段名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :---: | :--- | :--- |
| `status` | string | 否 | — | 枚举：`NORMAL` / `MISSING_PENDING` / `MISSING` |
| `page_no` | int | 否 | `1` | 页码 |
| `page_size` | int | 否 | `20` | 每页条数（≤ 100） |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_023",
  "data": {
    "items": [
      {
        "patient_id":   "1001",
        "profile_no":   "PRO20260419001",
        "patient_name": "张**",
        "gender":       "MALE",
        "age":          78,
        "avatar_url":   "https://oss.example.com/avatar/1001.jpg",
        "short_code":   "A3B7K9",
        "status":       "NORMAL",
        "fence_enabled": true,
        "relation_role": "PRIMARY_GUARDIAN"
      }
    ],
    "page_no":   1,
    "page_size": 20,
    "total":     1,
    "has_next":  false
  }
}
```

---

#### 3.3.14 逻辑删除患者档案

**描述**：主监护人逻辑删除患者档案。触发级联清理：AI 会话、向量、记忆笔记物理删除（FR-PRO-009, LLD §5.4.5）。

**HTTP 请求**：`DELETE /api/v1/patients/{patient_id}`

**安全性**：需鉴权（JWT），角色 `FAMILY`（主监护人）

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `Authorization` | string | 是 | `Bearer {token}` |
| `X-Request-Id` | string | 是 | 幂等键 |
| `X-Trace-Id` | string | 是 | 链路追踪 |

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `patient_id` | string | 是 | 目标患者 ID |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_024",
  "data": {
    "patient_id": "1001",
    "deleted_at": "2026-04-19T20:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_PRO_4041` | 404 | 患者不存在 |
| `E_PRO_4032` | 403 | 非主监护人 |
| `E_PRO_4092` | 409 | 患者有非终态任务，禁止删除 |

**Outbox 事件**：`profile.deleted.logical`（触发 AI 域物理清理向量、会话、记忆）

---

#### 3.3.15 管理员全局患者档案列表

**描述**：管理员跨监护关系查询全局患者档案，用于运营治理与协查（FR-PRO-011，LLD V2.0 §5.3.8）。**仅只读**，不得变更档案。

**HTTP 请求**：`GET /api/v1/admin/patients`

**安全性**：需鉴权（JWT），角色 `ADMIN` 或 `SUPER_ADMIN`

**Query Parameters**：

| 字段名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :---: | :--- | :--- |
| `keyword` | string | 否 | — | 模糊匹配 `patient_name` / `short_code` / `profile_no` |
| `status` | string | 否 | — | `NORMAL` / `MISSING_PENDING` / `MISSING` |
| `gender` | string | 否 | — | `MALE` / `FEMALE` / `UNKNOWN` |
| `primary_guardian_user_id` | string | 否 | — | 按主监护人过滤 |
| `cursor` | string | 否 | — | 分页游标 |
| `page_size` | int | 否 | `20` | 每页（≤ 100） |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_060",
  "data": {
    "items": [
      {
        "patient_id":   "1001",
        "profile_no":   "PRO20260419001",
        "short_code":   "A3B7K9",
        "patient_name": "李**",
        "gender":       "FEMALE",
        "age":          78,
        "status":       "NORMAL",
        "primary_guardian": {
          "user_id":  "2001",
          "nickname": "张**",
          "phone":    "138****5678"
        },
        "guardian_count": 3,
        "active_task_id": null,
        "created_at":     "2026-04-01T08:00:00Z"
      }
    ],
    "page_size":   20,
    "next_cursor": "eyJpZCI6MTAwMX0=",
    "has_next":    true
  }
}
```

**处理逻辑与合规要求**：

1. 响应字段默认脱敏：`patient_name` / 监护人 `nickname` / `phone` / `email` 均走 `@Desensitize`（HC-07）。
2. `admin` 与 `super_admin` 同表可见；不区分监护关系。
3. 查询行为写入 `sys_log`，`risk_level=MEDIUM`，`action=admin.patient.list`。
4. 禁止导出原始明文（导出接口独立，另行评审）。

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_AUTH_4031` | 403 | 非 `ADMIN` / `SUPER_ADMIN` |
| `E_REQ_4002` | 400 | `status` / `gender` 枚举非法 |

---

#### 3.3.16 管理员查看患者档案详情

**描述**：管理员查看单个患者档案详情（只读）。字段与 `GET /api/v1/patients/{patient_id}` 一致，但不走监护关系校验；响应中追加 `guardian_list`（所有 `ACTIVE` 监护成员）。

**HTTP 请求**：`GET /api/v1/admin/patients/{patient_id}`

**安全性**：需鉴权（JWT），角色 `ADMIN` 或 `SUPER_ADMIN`

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_061",
  "data": {
    "patient_id":   "1001",
    "profile_no":   "PRO20260419001",
    "short_code":   "A3B7K9",
    "patient_name": "李**",
    "gender":       "FEMALE",
    "birthday":     "1947-08-12",
    "status":       "NORMAL",
    "avatar_url":   "https://oss.example.com/avatar/p1001.jpg",
    "chronic_diseases": "…",
    "medication":       "…",
    "allergy":          "…",
    "appearance": { "height_cm": 160, "weight_kg": 52, "clothing": "…", "features": "…" },
    "fence": { "enabled": true, "center_lat": 30.12, "center_lng": 120.34, "radius_m": 500 },
    "guardian_list": [
      { "user_id": "2001", "relation_role": "PRIMARY_GUARDIAN", "nickname": "张**", "relation_status": "ACTIVE" },
      { "user_id": "2002", "relation_role": "GUARDIAN",         "nickname": "张**", "relation_status": "ACTIVE" }
    ],
    "active_task_id":   null,
    "last_track_at":    "2026-04-19T09:00:00Z",
    "profile_version":  7,
    "created_at":       "2026-04-01T08:00:00Z",
    "updated_at":       "2026-04-15T10:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_PRO_4041` | 404 | 患者不存在 |
| `E_AUTH_4031` | 403 | 非 `ADMIN` / `SUPER_ADMIN` |

**审计**：`sys_log` 记录 `action=admin.patient.read`，`risk_level=MEDIUM`，`object_id=patient_id`。

---

#### 3.3.17 管理员强制转移主监护权

**描述**：当原主监护人账号异常（失联 / 禁用 / 注销）且新监护人因无法取得原主确认而无法走 §3.3.8 / §3.3.9 双阶段流程时，由 **超级管理员** 行政强制转移主监护权（FR-PRO-012，LLD V2.0 §5.3.9，§5.4.3b）。

**HTTP 请求**：`POST /api/v1/admin/patients/{patient_id}/guardians/force-transfer`

**安全性**：需鉴权（JWT），角色仅 `SUPER_ADMIN`；`CONFIRM_3` 高确认等级（BDD §7 / §12）。

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `Authorization` | string | 是 | `Bearer {token}` |
| `X-Request-Id` | string | 是 | 幂等键 |
| `X-Trace-Id` | string | 是 | 链路追踪 |
| `X-Confirm-Level` | string | 是 | 必须为 `CONFIRM_3` |

**Request Body (application/json)**：

```json
{
  "target_user_id": "string, 必填, 新主监护人 user_id, 必须为 FAMILY+ACTIVE+该患者现有 ACTIVE 监护成员",
  "reason":         "string, 必填, 20-256, 行政理由（人工审批单号等）",
  "evidence_url":   "string, 可选, OSS 审批材料 URL"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_062",
  "data": {
    "patient_id":     "1001",
    "previous_primary_user_id": "2001",
    "new_primary_user_id":      "2002",
    "transferred_at": "2026-04-19T21:00:00Z",
    "audit_log_id":   "log_2026041921000001"
  }
}
```

**处理逻辑**：

1. 校验 `target_user_id` 为该患者当前 `guardian_relation.relation_status=ACTIVE` 成员；若非成员必须先通过邀请流程。
2. 事务内：
   - 原 `PRIMARY_GUARDIAN` → `GUARDIAN`；
   - 目标成员 → `PRIMARY_GUARDIAN`；
   - `patient_profile.profile_version += 1`（HC-01 乐观锁）。
3. 写 `sys_log` `action=admin.patient.force_transfer_primary`，`risk_level=CRITICAL`，`confirm_level=CONFIRM_3`。
4. 发布事件 `patient.primary_guardian.force_transferred`（payload 含 `previous_primary_user_id` / `new_primary_user_id` / `operator_user_id` / `reason` / `evidence_url`），消费方：通知域 → 通知原主监护 + 新主监护 + 其他 `ACTIVE` 成员。
5. 被禁用 / 注销的原主监护即使重新激活，也不会自动恢复主监护角色。

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_PRO_4041` | 404 | 患者不存在 |
| `E_PRO_4046` | 404 | `target_user_id` 不存在 |
| `E_PRO_4044` | 422 | `target_user_id` 非当前 `ACTIVE` 监护成员 |
| `E_PRO_4035` | 403 | `target_user_id` 非 `FAMILY` 或非 `ACTIVE` |
| `E_AUTH_4031` | 403 | 非 `SUPER_ADMIN` 或缺少 `CONFIRM_3` |

**Outbox 事件**：`patient.primary_guardian.force_transferred`。

---

### 3.4 标签与物资运营域（MAT）

> **底层设计基线**：LLD V2.0 §6（数据模型 `tag_asset` / `tag_apply_record`）、DBD V2.0 §4.8 ~ §4.9
> **需求覆盖**：FR-MAT-001 ~ FR-MAT-006

#### 3.4.0 接口概览

| # | 方法 | URI | 功能 | 鉴权 |
| :---: | :--- | :--- | :--- | :---: |
| 1 | POST | `/api/v1/material/orders` | 创建物资申领工单 | JWT |
| 2 | POST | `/api/v1/material/orders/{order_id}/approve` | 审批工单 | JWT(ADMIN) |
| 3 | POST | `/api/v1/material/orders/{order_id}/ship` | 发货 | JWT(ADMIN) |
| 4 | POST | `/api/v1/material/orders/{order_id}/receive` | 家属签收 | JWT |
| 5 | POST | `/api/v1/material/orders/{order_id}/cancel` | 取消工单 | JWT |
| 6 | POST | `/api/v1/tags/{tag_code}/bind` | 标签绑定 | JWT |
| 7 | POST | `/api/v1/tags/{tag_code}/loss/confirm` | 确认标签遗失 | JWT |
| 8 | POST | `/api/v1/tags/batch-generate` | 批量发号 | JWT(ADMIN) |
| 9 | GET | `/api/v1/tags/batch-generate/jobs/{job_id}` | 查询发号任务 | JWT(ADMIN) |
| 10 | GET | `/api/v1/tags/inventory/summary` | 库存摘要 | JWT(ADMIN) |
| 11 | GET | `/api/v1/material/orders` | 工单列表查询 | JWT |
| 12 | POST | `/api/v1/material/orders/{order_id}/resolve-exception` | 物流异常处置（补发/作废） | JWT(ADMIN) |

---

#### 3.4.1 创建物资申领工单

**描述**：家属为患者申请标签物资（FR-MAT-001, LLD §6.3.1）。

**HTTP 请求**：`POST /api/v1/material/orders`

**安全性**：需鉴权（JWT），角色 `FAMILY`

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `Authorization` | string | 是 | `Bearer {token}` |
| `X-Request-Id` | string | 是 | 幂等键 |
| `X-Trace-Id` | string | 是 | 链路追踪 |

**Request Body (application/json)**：

```json
{
  "patient_id":    "string, 必填",
  "tag_type":      "string, 必填, 枚举: QR_CODE / NFC",
  "quantity":      "integer, 必填, 1-10",
  "shipping_address": {
    "province":  "string, 必填",
    "city":      "string, 必填",
    "district":  "string, 必填",
    "detail":    "string, 必填, 最大长度 500",
    "receiver":  "string, 必填, 最大长度 64",
    "phone":     "string, 必填, 手机号格式"
  },
  "remark":        "string, 可选, 最大长度 500",
  "request_time":  "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_025",
  "data": {
    "order_id":   "6001",
    "order_no":   "ORD20260419001",
    "patient_id": "1001",
    "status":     "PENDING_AUDIT",
    "tag_type":   "QR_CODE",
    "quantity":   2,
    "created_at": "2026-04-19T09:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_MAT_4001` | 400 | `quantity` 非法 |
| `E_PRO_4041` | 404 | 患者不存在 |
| `E_PRO_4030` | 403 | 无监护权限 |

**工单状态机（LLD §6.2.2）**：

```
PENDING_AUDIT → PENDING_SHIP → SHIPPED → RECEIVED
      ↓               ↓          ↓
  CANCELLED       REJECTED    CANCELLED
```

---

#### 3.4.2 审批工单

**描述**：管理员审批物资申领工单（FR-MAT-001, LLD §6.3.2）。

**HTTP 请求**：`POST /api/v1/material/orders/{order_id}/approve`

**安全性**：需鉴权（JWT），角色 `ADMIN` / `SUPER_ADMIN`

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `order_id` | string | 是 | 目标工单 ID |

**Request Body (application/json)**：

```json
{
  "action":         "string, 必填, 枚举: APPROVE / REJECT",
  "reject_reason":  "string, REJECT 时必填, 5-256",
  "request_time":   "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_026",
  "data": {
    "order_id":    "6001",
    "order_no":    "ORD20260419001",
    "status":      "PENDING_SHIP",
    "reviewed_at": "2026-04-19T10:00:00Z",
    "reviewer_user_id": "9001"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_MAT_4041` | 404 | 工单不存在 |
| `E_MAT_4091` | 409 | 工单状态非 `PENDING_AUDIT` |
| `E_MAT_4030` | 403 | 非管理员 |

---

#### 3.4.3 发货

**描述**：管理员对已审批工单执行发货操作，分配标签并加密生成 `resource_token`（FR-MAT-002, LLD §6.3.3）。

**HTTP 请求**：`POST /api/v1/material/orders/{order_id}/ship`

**安全性**：需鉴权（JWT），角色 `ADMIN` / `SUPER_ADMIN`

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `order_id` | string | 是 | 目标工单 ID |

**Request Body (application/json)**：

```json
{
  "tag_codes":       ["string, 必填, 分配的标签编码列表"],
  "logistics_no":    "string, 可选, 物流单号",
  "logistics_company": "string, 可选, 物流公司",
  "request_time":    "string, 必填, ISO-8601"
}
```

**处理逻辑**（LLD §6.4.3）：

1. 校验 `tag_codes` 数量与工单 `quantity` 一致。
2. 校验每个标签状态为 `UNBOUND`。
3. 标签状态流转 `UNBOUND → ALLOCATED`。
4. 为每个标签生成 `resource_token`（AES-256-GCM 加密）。
5. 工单状态 → `SHIPPED`。
6. Outbox 发布 `material.order.shipped`。

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_027",
  "data": {
    "order_id":   "6001",
    "order_no":   "ORD20260419001",
    "status":     "SHIPPED",
    "shipped_at": "2026-04-19T14:00:00Z",
    "tags": [
      {
        "tag_code":       "TAG20260419001",
        "resource_token": "Base64URL密文（仅内部使用，不面向前端展示）",
        "status":         "ALLOCATED"
      }
    ]
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_MAT_4041` | 404 | 工单不存在 |
| `E_MAT_4091` | 409 | 工单状态非 `PENDING_SHIP` |
| `E_MAT_4221` | 422 | 标签数量与工单不匹配 |
| `E_MAT_4222` | 422 | 标签不可用（非 `UNBOUND`） |
| `E_MAT_4044` | 404 | 标签编码不存在 |

---

#### 3.4.4 家属签收

**描述**：家属确认收到物资并签收（LLD §6.3.3 扩展）。

**HTTP 请求**：`POST /api/v1/material/orders/{order_id}/receive`

**安全性**：需鉴权（JWT），角色 `FAMILY`（申领人）

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `order_id` | string | 是 | 目标工单 ID |

**Request Body (application/json)**：

```json
{
  "request_time": "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_028",
  "data": {
    "order_id":    "6001",
    "order_no":    "ORD20260419001",
    "status":      "RECEIVED",
    "received_at": "2026-04-19T16:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_MAT_4041` | 404 | 工单不存在 |
| `E_MAT_4091` | 409 | 工单状态非 `SHIPPED` |
| `E_MAT_4030` | 403 | 非申领人 |

---

#### 3.4.5 取消工单

**描述**：家属取消未发货的工单，或管理员取消任意未终态工单（LLD §6.4.2）。

**HTTP 请求**：`POST /api/v1/material/orders/{order_id}/cancel`

**安全性**：需鉴权（JWT）

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `order_id` | string | 是 | 目标工单 ID |

**Request Body (application/json)**：

```json
{
  "cancel_reason": "string, 必填, 5-256",
  "request_time":  "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_029",
  "data": {
    "order_id":     "6001",
    "order_no":     "ORD20260419001",
    "status":       "CANCELLED",
    "cancelled_at": "2026-04-19T09:30:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_MAT_4041` | 404 | 工单不存在 |
| `E_MAT_4094` | 409 | 当前状态不允许取消（已终态） |
| `E_MAT_4003` | 400 | `cancel_reason` 非法 |

---

#### 3.4.6 标签绑定

**描述**：家属收到标签后，扫码绑定至患者（FR-MAT-003, LLD §6.3.4）。

**HTTP 请求**：`POST /api/v1/tags/{tag_code}/bind`

**安全性**：需鉴权（JWT），角色 `FAMILY`

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `tag_code` | string | 是 | 标签编码 |

**Request Body (application/json)**：

```json
{
  "patient_id":   "string, 必填",
  "request_time": "string, 必填, ISO-8601"
}
```

**处理逻辑**（LLD §6.4.4 三方一致性校验）：

1. 校验标签状态为 `ALLOCATED`。
2. 三方一致性：`resource_token.patient_id` == `tag_asset.patient_id` == 请求 `patient_id`。
3. 标签状态 → `BOUND`。
4. Outbox 发布 `tag.bound`。

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_030",
  "data": {
    "tag_code":   "TAG20260419001",
    "patient_id": "1001",
    "status":     "BOUND",
    "bound_at":   "2026-04-19T17:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_MAT_4044` | 404 | 标签不存在 |
| `E_MAT_4096` | 409 | 三方一致性校验失败 |
| `E_MAT_4091` | 409 | 标签状态不允许绑定 |

---

#### 3.4.7 确认标签遗失

**描述**：家属或管理员确认标签遗失，触发关联状态联动（LLD §5.2.3、§6.3.5）。

**HTTP 请求**：`POST /api/v1/tags/{tag_code}/loss/confirm`

**安全性**：需鉴权（JWT），角色 `FAMILY` 或 `ADMIN`

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `tag_code` | string | 是 | 标签编码 |

**Request Body (application/json)**：

```json
{
  "lost_reason":  "string, 必填, 5-256",
  "request_time": "string, 必填, ISO-8601"
}
```

**处理逻辑**：

1. 标签状态 `SUSPECTED_LOST → LOST`。
2. 使 `resource_token` 失效（按 `kid` 撤销）。
3. Outbox 发布 `tag.loss.confirmed`。
4. 通知主监护人（`tag.suspected_lost` 通知路由）。

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_031",
  "data": {
    "tag_code":  "TAG20260419001",
    "status":    "LOST",
    "lost_at":   "2026-04-19T18:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_MAT_4044` | 404 | 标签不存在 |
| `E_MAT_4098` | 409 | 标签状态不满足 `LOST` 前置条件（非 `SUSPECTED_LOST`） |
| `E_MAT_4004` | 400 | `lost_reason` 非法 |

---

#### 3.4.8 批量发号

**描述**：管理员触发批量生成标签编码（异步任务），系统按序列号规则生成标签（FR-MAT-005, LLD §6.3.6）。

**HTTP 请求**：`POST /api/v1/tags/batch-generate`

**安全性**：需鉴权（JWT），角色 `ADMIN` / `SUPER_ADMIN`

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `Authorization` | string | 是 | `Bearer {token}` |
| `X-Request-Id` | string | 是 | 幂等键 |
| `X-Trace-Id` | string | 是 | 链路追踪 |

**Request Body (application/json)**：

```json
{
  "tag_type":     "string, 必填, 枚举: QR_CODE / NFC",
  "quantity":     "integer, 必填, 1-10000",
  "batch_key_id": "string, 可选, 加密批次密钥 ID（默认使用当前激活密钥）",
  "request_time": "string, 必填, ISO-8601"
}
```

**Response (HTTP 202)**：

```json
{
  "code": "ok",
  "message": "任务已提交",
  "trace_id": "trc_20260419_032",
  "data": {
    "job_id":     "JOB20260419001",
    "status":     "RUNNING",
    "quantity":   1000,
    "created_at": "2026-04-19T08:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_MAT_4225` | 422 | 数量超限（> 10000） |
| `E_MAT_4030` | 403 | 非管理员 |

---

#### 3.4.9 查询发号任务

**描述**：查询批量发号任务的执行状态（FR-MAT-005, LLD §6.3.7）。

**HTTP 请求**：`GET /api/v1/tags/batch-generate/jobs/{job_id}`

**安全性**：需鉴权（JWT），角色 `ADMIN` / `SUPER_ADMIN`

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `job_id` | string | 是 | 发号任务 ID |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_033",
  "data": {
    "job_id":         "JOB20260419001",
    "status":         "COMPLETED",
    "total_count":    1000,
    "success_count":  1000,
    "fail_count":     0,
    "tag_type":       "QR_CODE",
    "created_at":     "2026-04-19T08:00:00Z",
    "completed_at":   "2026-04-19T08:05:00Z"
  }
}
```

---

#### 3.4.10 库存摘要

**描述**：查询标签库存概况（FR-MAT-006, LLD §6.3.8）。

**HTTP 请求**：`GET /api/v1/tags/inventory/summary`

**安全性**：需鉴权（JWT），角色 `ADMIN` / `SUPER_ADMIN`

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_034",
  "data": {
    "summary": [
      {
        "tag_type":       "QR_CODE",
        "total":          5000,
        "unbound":        3200,
        "allocated":      800,
        "bound":          900,
        "lost":           50,
        "voided":         50
      },
      {
        "tag_type":       "NFC",
        "total":          2000,
        "unbound":        1500,
        "allocated":      200,
        "bound":          280,
        "lost":           10,
        "voided":         10
      }
    ],
    "updated_at": "2026-04-19T08:00:00Z"
  }
}
```

---

#### 3.4.11 工单列表查询

**描述**：分页查询物资工单列表。

**HTTP 请求**：`GET /api/v1/material/orders`

**安全性**：需鉴权（JWT）

**Query Parameters**：

| 字段名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :---: | :--- | :--- |
| `patient_id` | string | 否 | — | 按患者筛选 |
| `status` | string | 否 | — | 枚举：`PENDING_AUDIT` / `PENDING_SHIP` / `SHIPPED` / `RECEIVED` / `EXCEPTION` / `VOIDED` / `REJECTED` / `CANCELLED` |
| `page_no` | int | 否 | `1` | 页码 |
| `page_size` | int | 否 | `20` | 每页条数（≤ 100） |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_035",
  "data": {
    "items": [
      {
        "order_id":   "6001",
        "order_no":   "ORD20260419001",
        "patient_id": "1001",
        "tag_type":   "QR_CODE",
        "quantity":   2,
        "status":     "SHIPPED",
        "created_at": "2026-04-19T09:00:00Z",
        "shipped_at": "2026-04-19T14:00:00Z"
      }
    ],
    "page_no":   1,
    "page_size": 20,
    "total":     1,
    "has_next":  false
  }
}
```

---

#### 3.4.12 物流异常处置

**描述**：管理员对处于 `EXCEPTION` 状态的工单执行补发（重新发货）或直接作废处置（SRS AC-07，FR-MAT-004；LLD §6.3.8）。

**HTTP 请求**：`POST /api/v1/material/orders/{order_id}/resolve-exception`

**安全性**：需鉴权（JWT），角色 `ADMIN` / `SUPER_ADMIN`

**Path Parameters**：

| 字段名 | 类型 | 描述 |
| :--- | :--- | :--- |
| `order_id` | string | 工单 ID |

**Request Body (application/json)**：

```json
{
  "action":       "string, 必填, RESHIP | VOID",
  "reason":       "string, 必填, 10-200 字",
  "tracking_no":  "string, action=RESHIP 时必填, 6-32",
  "carrier":      "string, action=RESHIP 时必填, SF/YTO/ZTO/JD/EMS/OTHER"
}
```

**状态机跃迁**：

| `action` | 前置状态 | 目标状态 |
| :--- | :--- | :--- |
| `RESHIP` | `EXCEPTION` | `SHIPPED`（补发新物流） |
| `VOID` | `EXCEPTION` | `VOIDED` |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_090",
  "data": {
    "order_id":   "6001",
    "status":     "SHIPPED",
    "resolved_at":"2026-04-22T10:00:00Z",
    "resolved_by":"admin_user_id"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :--- | :--- |
| `E_MAT_4091` | 409 | 工单状态不为 `EXCEPTION` |
| `E_MAT_4224` | 422 | `RESHIP` 时补发物流单号重复 / 库存不足 |
| `E_MAT_4226` | 422 | `action` 枚举值非法或 `reason` 为空 |
| `E_AUTH_4031` | 403 | 非 ADMIN 角色 |

**Outbox 事件**：
- `RESHIP`：`order.exception.reshipped`
- `VOID`：`order.exception.voided`

---

### 3.5 AI 协同支持域（AI）

> **底层设计基线**：LLD V2.0 §7（数据模型 `ai_session` / `patient_memory_note` / `vector_store` / `ai_quota_ledger`）、DBD V2.0 §4.10 ~ §4.13
> **需求覆盖**：FR-AI-001 ~ FR-AI-015
> **模型接入**：Spring AI Alibaba + DashScope 通义千问（§1.10）

#### 3.5.0 接口概览

| # | 方法 | URI | 功能 | 鉴权 |
| :---: | :--- | :--- | :--- | :---: |
| 1 | POST | `/api/v1/ai/sessions` | 创建 AI 会话 | JWT |
| 2 | POST | `/api/v1/ai/sessions/{session_id}/messages` | 发送消息（SSE 流式） | JWT |
| 3 | POST | `/api/v1/ai/sessions/{session_id}/intents/{intent_id}/confirm` | 确认 Agent 意图 | JWT |
| 4 | POST | `/api/v1/ai/sessions/{session_id}/feedback` | 会话反馈 | JWT |
| 5 | POST | `/api/v1/ai/poster` | AI 生成寻人海报 | JWT |

---

#### 3.5.1 创建 AI 会话

**描述**：为指定患者与任务创建一个 AI 对话会话。同一患者同一任务仅能有一个活跃会话（FR-AI-001, LLD §7.3.3）。

**HTTP 请求**：`POST /api/v1/ai/sessions`

**安全性**：需鉴权（JWT），角色 `FAMILY`

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `Authorization` | string | 是 | `Bearer {token}` |
| `X-Request-Id` | string | 是 | 幂等键 |
| `X-Trace-Id` | string | 是 | 链路追踪 |

**Request Body (application/json)**：

```json
{
  "patient_id":   "string, 必填",
  "task_id":      "string, 必填",
  "request_time": "string, 必填, ISO-8601"
}
```

> **禁止**：客户端直接传入 `patient_id` / `task_id` 绕过会话归属校验。服务端从 JWT 提取 `user_id`，校验该用户对 `patient_id` 的监护权限，且 `task_id` 属于该患者。

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_036",
  "data": {
    "session_id":  "sess_001",
    "patient_id":  "1001",
    "task_id":     "8848",
    "status":      "ACTIVE",
    "created_at":  "2026-04-19T10:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_AI_4002` | 400 | 参数非法 |
| `E_AI_4033` | 403 | 无会话归属权限 |
| `E_AI_4091` | 409 | 已存在活跃会话 |

---

#### 3.5.2 发送消息（SSE 流式）

**描述**：家属向 AI 会话发送消息，AI 通过 Server-Sent Events (SSE) 流式返回推理结果。这是 AI 域的核心交互接口（FR-AI-001, FR-AI-012, LLD §7.3.1）。

**HTTP 请求**：`POST /api/v1/ai/sessions/{session_id}/messages`

**安全性**：需鉴权（JWT），角色 `FAMILY`

**Content-Type**：请求 `application/json`，响应 `text/event-stream`

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `session_id` | string | 是 | 目标会话 ID |

**Request Body (application/json)**：

```json
{
  "prompt":       "string, 必填, 1-2000",
  "request_time": "string, 必填, ISO-8601"
}
```

**Response (HTTP 200, text/event-stream)**：

SSE 事件流包含以下 5 种事件类型：

**① `event: token`** — AI 逐 token 流式输出

```
event: token
data: {"content": "根据", "index": 0}

event: token
data: {"content": "最新线索分析", "index": 1}
```

**② `event: tool_call`** — AI Agent 发起 Function Calling

```
event: tool_call
data: {
  "intent_id": "int_001",
  "action": "CREATE_TASK",
  "description": "AI 建议为患者发布寻回任务",
  "parameters": {
    "patient_id": "1001",
    "source": "AUTO_UPGRADE",
    "remark": "围栏告警升级"
  },
  "execution_level": "A2",
  "requires_confirm": true,
  "expires_at": "2026-04-19T10:10:00Z"
}
```

> `execution_level` 对应 LLD §7.2 Function Calling 白名单：
> - `A0`：纯读取，自动执行
> - `A1`：低风险写入，自动执行
> - `A2`：高风险写入，需用户确认
> - `A3`：行政级操作，需二次确认
> - `A4`：禁止自动执行

**③ `event: usage`** — Token 消耗统计

```
event: usage
data: {
  "prompt_tokens": 1200,
  "completion_tokens": 450,
  "total_tokens": 1650,
  "model_name": "qwen-max",
  "billing_source": "DashScope"
}
```

**④ `event: done`** — 流结束信号

```
event: done
data: {"finish_reason": "stop"}
```

降级场景：

```
event: done
data: {
  "finish_reason": "fallback",
  "fallback_response": {
    "mode": "RULE_BASED",
    "content": "建议在朝阳公园西门至北门区域搜寻"
  }
}
```

**⑤ `event: error`** — 错误事件

```
event: error
data: {"code": "E_AI_5031", "message": "AI 服务暂时不可用"}
```

**异常响应**（非流式错误，直接返回 JSON）：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_AI_4001` | 400 | `prompt` 非法 |
| `E_AI_4041` | 404 | 会话不存在 |
| `E_AI_4292` | 429 | 用户配额耗尽 |
| `E_AI_4293` | 429 | 患者配额耗尽 |
| `E_AI_4031` | 403 | 内容安全限制 |
| `E_AI_5021` | 500 | 上下文溢出 |
| `E_AI_5031` | 503 | 模型超时 |

**限流与幂等说明**：

- 限流：用户级 + 患者级双台账配额（LLD §7.7.3）。走失态患者豁免扣减（SADD §6.3）。
- 幂等：SSE 流式接口不支持严格幂等（重复请求产生新推理）。`X-Request-Id` 仅用于审计。

**配额说明**：

| 配额维度 | 配置键 | 默认值 |
| :--- | :--- | :--- |
| 用户月度 Token | `ai.quota.user.monthly_limit` | 由 `sys_config` 下发 |
| 患者月度 Token | `ai.quota.patient.monthly_limit` | 由 `sys_config` 下发 |
| 预占超时 | `ai.quota.pending.timeout.seconds` | 300 |

---

#### 3.5.3 确认 Agent 意图

**描述**：当 AI Agent 通过 `tool_call` 事件提出需用户确认的操作意图时，用户通过此接口确认或拒绝（FR-AI-007, LLD §7.3.2）。

**HTTP 请求**：`POST /api/v1/ai/sessions/{session_id}/intents/{intent_id}/confirm`

**安全性**：需鉴权（JWT），角色 `FAMILY`

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `session_id` | string | 是 | 目标会话 ID |
| `intent_id` | string | 是 | 意图 ID（来自 `tool_call` 事件） |

**Request Body (application/json)**：

```json
{
  "action":       "string, 必填, 枚举: APPROVE / REJECT",
  "request_time": "string, 必填, ISO-8601"
}
```

**处理逻辑**（LLD §7.4.1 意图确认子流程）：

1. 校验 `intent_id` 存在于 Redis 缓存且未过期。
2. `APPROVE`：执行 Function Calling 对应的域 API，注入 `X-Action-Source=AI_AGENT` 等 Agent Headers。
3. `REJECT`：标记意图为用户拒绝，AI 收到反馈继续对话。
4. 执行结果通过审计回执落库 `sys_log`（`action_source=AI_AGENT`）。

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_037",
  "data": {
    "intent_id":        "int_001",
    "action":           "APPROVE",
    "execution_result": {
      "success":   true,
      "action":    "CREATE_TASK",
      "result_id": "8849",
      "message":   "任务已成功创建"
    }
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_AI_4041` | 404 | 会话不存在 |
| `E_AI_4091` | 409 | 意图已过期或已处理 |
| `E_GOV_4039` | 403 | Policy Guard 拦截 |

---

#### 3.5.4 会话反馈

**描述**：家属对 AI 会话质量进行反馈评价（FR-AI-014, LLD §7.3.4）。

**HTTP 请求**：`POST /api/v1/ai/sessions/{session_id}/feedback`

**安全性**：需鉴权（JWT），角色 `FAMILY`

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `session_id` | string | 是 | 目标会话 ID |

**Request Body (application/json)**：

```json
{
  "rating":       "integer, 必填, 1-5",
  "comment":      "string, 可选, 最大长度 1000",
  "request_time": "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_038",
  "data": {
    "session_id":  "sess_001",
    "feedback_at": "2026-04-19T18:00:00Z"
  }
}
```

---

#### 3.5.5 AI 生成寻人海报

**描述**：AI 根据患者档案与任务信息生成寻人海报（FR-AI-013, LLD §7.3.5）。

**HTTP 请求**：`POST /api/v1/ai/poster`

**安全性**：需鉴权（JWT），角色 `FAMILY`

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `Authorization` | string | 是 | `Bearer {token}` |
| `X-Request-Id` | string | 是 | 幂等键 |
| `X-Trace-Id` | string | 是 | 链路追踪 |

**Request Body (application/json)**：

```json
{
  "patient_id":   "string, 必填",
  "task_id":      "string, 必填",
  "template_id":  "string, 可选, 海报模板（默认 TPL_MISSING_V1）",
  "request_time": "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_039",
  "data": {
    "poster_url":   "https://oss.example.com/posters/task_8848.png",
    "template_id":  "TPL_MISSING_V1",
    "generated_at": "2026-04-19T10:30:00Z",
    "token_usage": {
      "prompt_tokens":     800,
      "completion_tokens": 200,
      "total_tokens":      1000,
      "model_name":        "qwen-vl-max"
    }
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_AI_4002` | 400 | 参数非法 |
| `E_AI_4033` | 403 | 无患者或任务授权 |
| `E_AI_4292` | 429 | 用户配额耗尽 |
| `E_AI_5031` | 503 | 模型超时 |

**Outbox 事件**：`ai.poster.generated`（payload 见 §5.4.4）

---

### 3.6 通用治理域（GOV）

> **底层设计基线**：LLD V2.0 §8（数据模型 `sys_user` / `sys_log` / `sys_outbox_log` / `consumed_event_log` / `sys_config` / `notification_inbox`）、DBD V2.0 §4.14 ~ §4.19
> **需求覆盖**：FR-GOV-001 ~ FR-GOV-010

#### 3.6.0 接口概览

| # | 方法 | URI | 功能 | 鉴权 |
| :---: | :--- | :--- | :--- | :---: |
| 1 | POST | `/api/v1/auth/register` | 用户注册 | 匿名 |
| 2 | POST | `/api/v1/auth/login` | 用户登录 | 匿名 |
| 3 | POST | `/api/v1/auth/token/refresh` | 刷新令牌 | Refresh Token |
| 4 | POST | `/api/v1/auth/password-reset/request` | 请求密码重置 | 匿名 |
| 5 | POST | `/api/v1/auth/password-reset/confirm` | 确认密码重置 | 匿名(Token) |
| 6 | GET | `/api/v1/users/me` | 获取当前用户信息 | JWT |
| 7 | PUT | `/api/v1/users/me/password` | 修改密码 | JWT |
| 8 | PUT | `/api/v1/admin/configs/{config_key}` | 修改系统配置 | JWT(ADMIN) |
| 9 | GET | `/api/v1/admin/configs` | 查询配置列表 | JWT(ADMIN) |
| 10 | GET | `/api/v1/admin/logs` | 审计日志查询 | JWT(ADMIN) |
| 11 | GET | `/api/v1/notifications/inbox` | 通知收件箱 | JWT |
| 12 | POST | `/api/v1/notifications/{notification_id}/read` | 标记通知已读 | JWT |
| 13 | POST | `/api/v1/admin/super/outbox/dead/{event_id}/replay` | DEAD 事件重放 | JWT(SUPER_ADMIN) |
| 14 | GET | `/api/v1/admin/super/outbox/dead` | DEAD 事件列表 | JWT(SUPER_ADMIN) |
| 15 | GET | `/api/v1/admin/users` | 用户列表查询 | JWT(ADMIN) |
| 16 | GET | `/api/v1/admin/users/{user_id}` | 查看用户详情 | JWT(ADMIN) |
| 17 | PUT | `/api/v1/admin/users/{user_id}` | 修改用户资料 / 角色 | JWT(ADMIN) |
| 18 | POST | `/api/v1/admin/users/{user_id}/disable` | 禁用用户 | JWT(ADMIN) |
| 19 | POST | `/api/v1/admin/users/{user_id}/enable` | 启用用户 | JWT(ADMIN) |
| 20 | DELETE | `/api/v1/admin/users/{user_id}` | 逻辑删除用户 | JWT(ADMIN) |
| 21 | GET | `/api/v1/admin/logs/export` | 审计日志导出（CSV/JSON） | JWT(SUPER_ADMIN) |

---

#### 3.6.1 用户注册

**描述**：新用户注册账号，注册后发送邮箱验证链接（FR-GOV-001, FR-GOV-002, LLD §8.3.1）。

**HTTP 请求**：`POST /api/v1/auth/register`

**安全性**：匿名

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `X-Request-Id` | string | 是 | 幂等键 |
| `X-Trace-Id` | string | 是 | 链路追踪 |

**Request Body (application/json)**：

```json
{
  "username": "string, 必填, 4-64, 仅字母数字下划线",
  "email":    "string, 必填, 邮箱格式, 最大 128",
  "password": "string, 必填, 8-128, 至少含大小写字母+数字",
  "nickname": "string, 可选, 最大 64"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_040",
  "data": {
    "user_id": "2001",
    "username": "zhangsan",
    "email_verification_sent": true
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_AUTH_4001` | 400 | `username` 格式非法 |
| `E_AUTH_4002` | 400 | `password` 格式非法 |
| `E_GOV_4091` | 409 | 用户名已存在 |
| `E_GOV_4092` | 409 | 邮箱已存在 |

**处理逻辑**：密码 BCrypt 哈希后存储；发送邮箱验证链接（TTL = 30 分钟）。

---

#### 3.6.2 用户登录

**描述**：用户通过用户名和密码登录，返回 JWT 令牌对（FR-GOV-001, LLD §8.3.2）。

**HTTP 请求**：`POST /api/v1/auth/login`

**安全性**：匿名

**Request Body (application/json)**：

```json
{
  "username": "string, 必填",
  "password": "string, 必填"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_041",
  "data": {
    "access_token":  "eyJhbGciOiJSUzI1NiJ9...",
    "refresh_token": "eyJhbGciOiJSUzI1NiJ9...",
    "token_type":    "Bearer",
    "expires_in":    3600,
    "user": {
      "user_id":  "2001",
      "username": "zhangsan",
      "nickname": "张三",
      "role":     "FAMILY",
      "avatar_url": "https://oss.example.com/avatar/u2001.jpg"
    }
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_AUTH_4011` | 401 | 凭据错误 |
| `E_GOV_4031` | 403 | 账号已被封禁 |

---

#### 3.6.3 刷新令牌

**描述**：使用 Refresh Token 获取新的 Access Token。

**HTTP 请求**：`POST /api/v1/auth/token/refresh`

**Request Body (application/json)**：

```json
{
  "refresh_token": "string, 必填"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_042",
  "data": {
    "access_token":  "eyJhbGciOiJSUzI1NiJ9...",
    "refresh_token": "eyJhbGciOiJSUzI1NiJ9...",
    "token_type":    "Bearer",
    "expires_in":    3600
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_GOV_4011` | 401 | Refresh Token 无效或已过期 |

---

#### 3.6.4 请求密码重置

**描述**：用户通过邮箱请求密码重置链接（FR-GOV-002, AC-15, LLD §8.3.3）。

**HTTP 请求**：`POST /api/v1/auth/password-reset/request`

**安全性**：匿名

**Request Body (application/json)**：

```json
{
  "email": "string, 必填, 邮箱格式"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "如果该邮箱已注册，将收到重置链接",
  "trace_id": "trc_20260419_043",
  "data": null
}
```

> 无论邮箱是否存在均返回相同消息，防止用户枚举。

**处理逻辑**：生成重置 Token（TTL = 30 分钟），仅通过邮件渠道发送（HC-08）。

---

#### 3.6.5 确认密码重置

**描述**：用户通过邮箱链接中的 Token 设置新密码。

**HTTP 请求**：`POST /api/v1/auth/password-reset/confirm`

**安全性**：匿名（含重置 Token）

**Request Body (application/json)**：

```json
{
  "token":        "string, 必填, 重置 Token",
  "new_password": "string, 必填, 8-128, 至少含大小写字母+数字"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "密码已重置",
  "trace_id": "trc_20260419_044",
  "data": null
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_GOV_4102` | 410 | 重置链接已过期 |
| `E_AUTH_4002` | 400 | 新密码不符合强度要求 |

---

#### 3.6.6 获取当前用户信息

**描述**：获取当前登录用户的个人信息。

**HTTP 请求**：`GET /api/v1/users/me`

**安全性**：需鉴权（JWT）

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_045",
  "data": {
    "user_id":    "2001",
    "username":   "zhangsan",
    "nickname":   "张三",
    "email":      "zha****@example.com",
    "phone":      "138****5678",
    "role":       "FAMILY",
    "status":     "ACTIVE",
    "avatar_url": "https://oss.example.com/avatar/u2001.jpg",
    "email_verified": true,
    "created_at": "2026-04-01T08:00:00Z"
  }
}
```

> **PII 脱敏**：`email` → `@Desensitize(EMAIL)`，`phone` → `@Desensitize(PHONE)`。

---

#### 3.6.7 修改密码

**描述**：已登录用户修改自己的密码。

**HTTP 请求**：`PUT /api/v1/users/me/password`

**安全性**：需鉴权（JWT）

**Request Body (application/json)**：

```json
{
  "old_password": "string, 必填",
  "new_password": "string, 必填, 8-128",
  "request_time": "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "密码已更新",
  "trace_id": "trc_20260419_046",
  "data": null
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_USR_4011` | 401 | 旧密码校验失败 |
| `E_USR_4001` | 400 | 新旧密码相同 |
| `E_USR_4002` | 400 | 新密码不满足强度策略 |

---

#### 3.6.8 修改系统配置

**描述**：管理员修改系统配置项（FR-GOV-008, LLD §8.3.4）。

**HTTP 请求**：`PUT /api/v1/admin/configs/{config_key}`

**安全性**：需鉴权（JWT），角色 `ADMIN` / `SUPER_ADMIN`

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `config_key` | string | 是 | 配置键 |

**Request Body (application/json)**：

```json
{
  "config_value": "string, 必填",
  "reason":       "string, 必填, 5-256",
  "request_time": "string, 必填, ISO-8601"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_047",
  "data": {
    "config_key":   "ai.quota.user.monthly_limit",
    "config_value": "100000",
    "updated_at":   "2026-04-19T10:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_GOV_4041` | 404 | 配置键不存在 |
| `E_GOV_4030` | 403 | 权限不足 |

---

#### 3.6.9 查询配置列表

**描述**：管理员查询系统配置列表。

**HTTP 请求**：`GET /api/v1/admin/configs`

**安全性**：需鉴权（JWT），角色 `ADMIN` / `SUPER_ADMIN`

**Query Parameters**：

| 字段名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :---: | :--- | :--- |
| `scope` | string | 否 | — | 枚举：`public` / `ops` / `security` / `ai_policy` |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_048",
  "data": {
    "items": [
      {
        "config_key":   "ai.quota.user.monthly_limit",
        "config_value": "100000",
        "scope":        "ai_policy",
        "description":  "用户月度 AI Token 配额",
        "updated_at":   "2026-04-19T10:00:00Z"
      }
    ]
  }
}
```

---

#### 3.6.10 审计日志查询

**描述**：管理员查询系统审计日志，支持多维度筛选（FR-GOV-006, FR-GOV-007, LLD §8.3.5）。

**HTTP 请求**：`GET /api/v1/admin/logs`

**安全性**：需鉴权（JWT），角色 `ADMIN` / `SUPER_ADMIN`

**Query Parameters**：

| 字段名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :---: | :--- | :--- |
| `module` | string | 否 | — | 枚举：`TASK` / `CLUE` / `PROFILE` / `MAT` / `AI` / `GOVERNANCE` / `NOTIFICATION` |
| `action` | string | 否 | — | 动作标识 |
| `action_source` | string | 否 | — | 枚举：`USER` / `AI_AGENT` |
| `operator_user_id` | string | 否 | — | 操作人 |
| `date_from` | string | 否 | — | ISO-8601 |
| `date_to` | string | 否 | — | ISO-8601 |
| `risk_level` | string | 否 | — | 枚举：`LOW` / `MEDIUM` / `HIGH` / `CRITICAL` |
| `cursor` | string | 否 | — | 分页游标 |
| `page_size` | int | 否 | `50` | 每页条数（≤ 200） |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_049",
  "data": {
    "items": [
      {
        "id":                "90001",
        "module":            "TASK",
        "action":            "CLOSE_TASK",
        "action_source":     "USER",
        "operator_user_id":  "2001",
        "operator_username": "zhangsan",
        "object_id":         "8848",
        "result":            "SUCCESS",
        "risk_level":        "MEDIUM",
        "ip":                "192.168.1.100",
        "trace_id":          "trc_20260419_002",
        "created_at":        "2026-04-19T18:30:00Z"
      }
    ],
    "page_size":   50,
    "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNi0wNC0xOVQxODozMDowMFoiLCJpZCI6IjkwMDAxIn0=",
    "has_next":    true
  }
}
```

> 审计日志采用 **Cursor 模式**分页（§1.5），适合追加写入型流水表。

---

#### 3.6.11 通知收件箱

**描述**：查询当前用户的通知收件箱（FR-GOV-010, LLD §8.3.6）。

**HTTP 请求**：`GET /api/v1/notifications/inbox`

**安全性**：需鉴权（JWT）

**Query Parameters**：

| 字段名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :---: | :--- | :--- |
| `read_status` | string | 否 | — | 枚举：`UNREAD` / `READ` |
| `type` | string | 否 | — | 枚举：`TASK_PROGRESS` / `FENCE_ALERT` / `TASK_CLOSED` / `MISSING_PENDING_ALERT` / `TAG_SUSPECTED_LOST` / `TRANSFER_REQUEST` / `SYSTEM` |
| `cursor` | string | 否 | — | 分页游标 |
| `page_size` | int | 否 | `20` | 每页条数（≤ 100） |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_050",
  "data": {
    "items": [
      {
        "notification_id":    "9001",
        "type":               "TASK_PROGRESS",
        "title":              "新线索已验证",
        "content":            "患者张**收到一条新的有效线索",
        "level":              "INFO",
        "channel":            "WEBSOCKET",
        "related_task_id":    "8848",
        "related_patient_id": "1001",
        "read_status":        "UNREAD",
        "created_at":         "2026-04-19T15:30:00Z"
      }
    ],
    "page_size":   20,
    "next_cursor": "eyJjcmVhdGVkX2F0IjoiMjAyNi0wNC0xOVQxNTozMDowMFoiLCJpZCI6IjkwMDEifQ==",
    "has_next":    false
  }
}
```

---

#### 3.6.12 标记通知已读

**描述**：标记单条通知为已读。

**HTTP 请求**：`POST /api/v1/notifications/{notification_id}/read`

**安全性**：需鉴权（JWT）

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `notification_id` | string | 是 | 通知 ID |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_051",
  "data": {
    "notification_id": "9001",
    "read_status":     "READ",
    "read_at":         "2026-04-19T16:00:00Z"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_NOTI_4041` | 404 | 通知不存在 |
| `E_NOTI_4030` | 403 | 非通知接收人 |

---

#### 3.6.13 DEAD 事件受控重放

**描述**：超级管理员对 Outbox 中的 DEAD 事件执行受控重放（SADD §5.3, LLD §8.3.7）。

**HTTP 请求**：`POST /api/v1/admin/super/outbox/dead/{event_id}/replay`

**安全性**：需鉴权（JWT），角色 `SUPER_ADMIN`，确认等级 `CONFIRM_3`

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `event_id` | string | 是 | DEAD 事件 ID |

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `Authorization` | string | 是 | `Bearer {token}` |
| `X-Request-Id` | string | 是 | 幂等键 |
| `X-Trace-Id` | string | 是 | 链路追踪 |

**Request Body (application/json)**：

```json
{
  "reason":       "string, 必填, 5-256",
  "replay_token": "string, 必填, 幂等键",
  "request_time": "string, 必填, ISO-8601"
}
```

**前置校验**：

1. 事件当前 `phase` 必须为 `DEAD`。
2. 同 `partition_key` 无更早未修复 `DEAD`（分区闸门，LLD §9.5）。

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_052",
  "data": {
    "event_id":     "evt_01H_dead_001",
    "phase":        "RETRY",
    "replayed_at":  "2026-04-19T20:00:00Z",
    "replay_token": "replay_001"
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_GOV_4046` | 404 | 事件不存在 |
| `E_GOV_4096` | 409 | 事件非 `DEAD` 状态 |
| `E_GOV_4098` | 409 | 同分区存在更早未修复 `DEAD` |
| `E_GOV_4032` | 403 | 非 `SUPER_ADMIN` |

---

#### 3.6.14 DEAD 事件列表

**描述**：查询 Outbox 中的 DEAD 事件列表（供运维排查）。

**HTTP 请求**：`GET /api/v1/admin/super/outbox/dead`

**安全性**：需鉴权（JWT），角色 `SUPER_ADMIN`

**Query Parameters**：

| 字段名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :---: | :--- | :--- |
| `topic` | string | 否 | — | 按 Topic 筛选 |
| `partition_key` | string | 否 | — | 按分区键筛选 |
| `cursor` | string | 否 | — | 分页游标 |
| `page_size` | int | 否 | `20` | 每页条数（≤ 100） |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_053",
  "data": {
    "items": [
      {
        "event_id":      "evt_01H_dead_001",
        "topic":         "task.created",
        "aggregate_id":  "8848",
        "partition_key": "patient_1001",
        "phase":         "DEAD",
        "retry_count":   11,
        "last_error":    "Redis connection refused",
        "created_at":    "2026-04-19T10:00:00Z",
        "updated_at":    "2026-04-19T19:00:00Z"
      }
    ],
    "page_size":   20,
    "next_cursor": null,
    "has_next":    false
  }
}
```

---

#### 3.6.15 用户列表查询

**描述**：管理员分页查询用户列表（FR-GOV-011，LLD V2.0 §8.3.8）。

**HTTP 请求**：`GET /api/v1/admin/users`

**安全性**：需鉴权（JWT），角色 `ADMIN` 或 `SUPER_ADMIN`

**Query Parameters**：

| 字段名 | 类型 | 必填 | 默认值 | 描述 |
| :--- | :--- | :---: | :--- | :--- |
| `keyword` | string | 否 | — | 模糊匹配 `username` / `nickname` / `email` / `phone` |
| `role` | string | 否 | — | `FAMILY` / `ADMIN` / `SUPER_ADMIN`（多值英文逗号） |
| `status` | string | 否 | — | `ACTIVE` / `DISABLED` / `DEACTIVATED` |
| `cursor` | string | 否 | — | 分页游标 |
| `page_size` | int | 否 | `20` | ≤ 100 |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_070",
  "data": {
    "items": [
      {
        "user_id":        "2001",
        "username":       "zhangsan",
        "nickname":       "张三",
        "email":          "zha****@example.com",
        "phone":          "138****5678",
        "role":           "FAMILY",
        "status":         "ACTIVE",
        "email_verified": true,
        "last_login_at":  "2026-04-19T10:00:00Z",
        "created_at":     "2026-04-01T08:00:00Z"
      }
    ],
    "page_size":   20,
    "next_cursor": "eyJpZCI6MjAwMX0=",
    "has_next":    true
  }
}
```

**授权矩阵**（查询结果过滤）：

| 当前角色 | 可见 `role` 集合 |
| :--- | :--- |
| `ADMIN` | `FAMILY` 唯一 |
| `SUPER_ADMIN` | `FAMILY` / `ADMIN` / `SUPER_ADMIN` 全部 |

> **服务端强制过滤**：`ADMIN` 传入 `role=ADMIN|SUPER_ADMIN` 不报错但结果不含；前端按钮级同步隐藏（WAHB P-14）。

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_AUTH_4031` | 403 | 非 `ADMIN` / `SUPER_ADMIN` |
| `E_REQ_4002` | 400 | `role` / `status` 枚举非法 |

---

#### 3.6.16 查看用户详情

**描述**：管理员查看单个用户详情。

**HTTP 请求**：`GET /api/v1/admin/users/{user_id}`

**安全性**：需鉴权（JWT），角色 `ADMIN` 或 `SUPER_ADMIN`

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_071",
  "data": {
    "user_id":        "2001",
    "username":       "zhangsan",
    "nickname":       "张三",
    "email":          "zha****@example.com",
    "phone":          "138****5678",
    "role":           "FAMILY",
    "status":         "ACTIVE",
    "email_verified": true,
    "last_login_at":  "2026-04-19T10:00:00Z",
    "last_login_ip":  "1.2.3.**",
    "deactivated_at": null,
    "created_at":     "2026-04-01T08:00:00Z",
    "updated_at":     "2026-04-15T10:00:00Z",
    "stats": {
      "primary_guardian_patient_count": 1,
      "guardian_patient_count":         2,
      "pending_material_order_count":   0
    }
  }
}
```

**授权约束**：

| 当前角色 | 可查看目标 |
| :--- | :--- |
| `ADMIN` | 仅目标 `role=FAMILY` |
| `SUPER_ADMIN` | 任意 |

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_USR_4041` | 404 | 用户不存在 |
| `E_USR_4032` | 403 | `ADMIN` 访问非 `FAMILY` 账号 |
| `E_AUTH_4031` | 403 | 非管理员 |

---

#### 3.6.17 修改用户信息

**描述**：管理员修改目标用户资料或角色（FR-GOV-012）。

**HTTP 请求**：`PUT /api/v1/admin/users/{user_id}`

**安全性**：需鉴权（JWT），角色 `ADMIN` 或 `SUPER_ADMIN`

**Request Body (application/json)**：

```json
{
  "nickname": "string, 可选, 最大 64",
  "email":    "string, 可选, 邮箱格式, 最大 128",
  "phone":    "string, 可选, 手机号",
  "role":     "string, 可选, FAMILY / ADMIN / SUPER_ADMIN, 仅 SUPER_ADMIN 可传"
}
```

**授权矩阵**：

| 当前角色 | 可操作目标 | 可改字段 | 特殊禁令 |
| :--- | :--- | :--- | :--- |
| `ADMIN` | 目标 `role=FAMILY` | `nickname` / `email` / `phone` | 不得传 `role` |
| `SUPER_ADMIN` | 任意 | 全部 | 不得将 `SUPER_ADMIN` 降级；不得修改自身角色 |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_072",
  "data": {
    "user_id":    "2001",
    "updated_at": "2026-04-19T21:30:00Z"
  }
}
```

**处理逻辑**：

1. 校验授权矩阵；
2. 若修改 `email` 则 `email_verified` 置 `false` 并触发二次验证邮件；
3. 若修改 `role` 发布事件 `user.role.changed`，通知目标用户并失效其所有现行 JWT（强制重登）；
4. 写 `sys_log`，`action=admin.user.update`，`risk_level=HIGH`（角色变更为 `CRITICAL`）。

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_USR_4041` | 404 | 目标不存在 |
| `E_USR_4032` | 403 | `ADMIN` 操作非 `FAMILY` |
| `E_USR_4033` | 403 | 试图降级 `SUPER_ADMIN` |
| `E_USR_4034` | 403 | 试图修改自身角色 |
| `E_USR_4035` | 403 | 非 `SUPER_ADMIN` 传入 `role` |
| `E_USR_4004` | 400 | `role` 枚举非法 |
| `E_USR_4005` | 400 | `email` / `phone` / `nickname` 格式非法 |

---

#### 3.6.18 禁用用户

**描述**：管理员禁用目标用户账号，禁用后该账号所有 JWT 失效，无法登录（FR-GOV-013）。

**HTTP 请求**：`POST /api/v1/admin/users/{user_id}/disable`

**安全性**：需鉴权（JWT），角色 `ADMIN` 或 `SUPER_ADMIN`

**Request Body (application/json)**：

```json
{
  "reason": "string, 必填, 10-256, 行政理由"
}
```

**授权矩阵**：

| 当前角色 | 可操作目标 |
| :--- | :--- |
| `ADMIN` | `role=FAMILY` 且非自身 |
| `SUPER_ADMIN` | `role=FAMILY` / `role=ADMIN` 且非自身；**`SUPER_ADMIN` 永不可禁用** |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_073",
  "data": {
    "user_id":     "2001",
    "status":      "DISABLED",
    "disabled_at": "2026-04-19T22:00:00Z"
  }
}
```

**处理逻辑**：

1. 若目标已持有任一患者的 `PRIMARY_GUARDIAN` 关系，**不阻断禁用**，但事件 `user.disabled` payload 中列出 `primary_patient_ids`，运营需跟进 §3.3.17 强制转移；
2. 吊销所有 JWT（Redis 黑名单，TTL = refresh token 剩余）；
3. 写 `sys_log`，`action=admin.user.disable`，`risk_level=HIGH`，`confirm_level=CONFIRM_2`。

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_USR_4041` | 404 | 用户不存在 |
| `E_USR_4032` | 403 | `ADMIN` 操作非 `FAMILY` |
| `E_USR_4033` | 403 | 操作 `SUPER_ADMIN` |
| `E_USR_4034` | 403 | 操作自身 |
| `E_USR_4091` | 409 | 用户已 `DISABLED` |

**Outbox 事件**：`user.disabled`。

---

#### 3.6.19 启用用户

**描述**：将 `DISABLED` 用户恢复为 `ACTIVE`。

**HTTP 请求**：`POST /api/v1/admin/users/{user_id}/enable`

**安全性**：授权矩阵同 §3.6.18。

**Request Body (application/json)**：

```json
{
  "reason": "string, 可选, 最大 256"
}
```

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_074",
  "data": { "user_id": "2001", "status": "ACTIVE", "enabled_at": "2026-04-19T22:30:00Z" }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_USR_4041` | 404 | 用户不存在 |
| `E_USR_4091` | 409 | 用户当前非 `DISABLED` |
| `E_USR_4032` / `E_USR_4033` / `E_USR_4034` | 403 | 授权不满足 |

**Outbox 事件**：`user.enabled`。

---

#### 3.6.20 逻辑删除用户

**描述**：将用户标记为 `DEACTIVATED`（注销）。与禁用不同，注销是终态不可逆（FR-GOV-014）。

**HTTP 请求**：`DELETE /api/v1/admin/users/{user_id}`

**安全性**：`CONFIRM_3` 高确认等级。

**Headers 扩展**：`X-Confirm-Level: CONFIRM_3` 必填。

**Request Body (application/json)**：

```json
{ "reason": "string, 必填, 20-256" }
```

**授权矩阵**：

| 当前角色 | 可操作目标 |
| :--- | :--- |
| `ADMIN` | `role=FAMILY` 且非自身 |
| `SUPER_ADMIN` | `role=FAMILY` / `role=ADMIN` 且非自身；**`SUPER_ADMIN` 永不可删除** |

**前置检查**：

1. 若目标当前为任意患者的 `PRIMARY_GUARDIAN`：返回 `E_USR_4092`；必须先走 §3.3.17 强制转移；
2. 若目标持有 `PENDING_AUDIT` / `PENDING_SHIP` 物资工单或 `ACTIVE` / `SUSTAINED` 寻回任务：返回 `E_USR_4093`；
3. 校验通过后事务：
   - `sys_user.status = DEACTIVATED`；`deactivated_at = now()`；`username`/`email`/`phone` 追加后缀 `#DEL_{timestamp}` 释放唯一约束；
   - `guardian_relation.relation_status = REVOKED`（非主监护的）；
   - 吊销所有 JWT；
   - 写 `sys_log` `risk_level=CRITICAL`，`confirm_level=CONFIRM_3`。

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_075",
  "data": { "user_id": "2001", "status": "DEACTIVATED", "deactivated_at": "2026-04-19T22:45:00Z" }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_USR_4041` | 404 | 用户不存在 |
| `E_USR_4032` / `E_USR_4033` / `E_USR_4034` | 403 | 授权不满足 |
| `E_USR_4092` | 409 | 仍持有主监护关系 |
| `E_USR_4093` | 409 | 仍有未终态任务 / 工单 |
| `E_AUTH_4031` | 403 | 缺少 `CONFIRM_3` |

**Outbox 事件**：`user.deactivated`。

---

#### 3.6.21 审计日志导出

**描述**：超级管理员按过滤条件导出结构化审计日志（CSV 或 JSON，单次上限 10,000 条），满足 FR-GOV-007 合规数据导出要求。

**HTTP 请求**：`GET /api/v1/admin/logs/export`

**安全性**：需鉴权（JWT），角色 `SUPER_ADMIN`

**Query Parameters**（全部可选，至少提供一个时间限制）：

| 字段名 | 类型 | 必填 | 默认 | 描述 |
| :--- | :--- | :---: | :--- | :--- |
| `start_at` | string(ISO8601) | **是** | — | 起始时间（最早不超过当前时间 180 天） |
| `end_at` | string(ISO8601) | **是** | — | 结束时间；与 `start_at` 间隔不超过 31 天 |
| `operator_id` | string | 否 | — | 按操作人过滤 |
| `action` | string | 否 | — | 按动作类型过滤 |
| `resource_type` | string | 否 | — | 按资源类型过滤 |
| `format` | string | 否 | `json` | 响应格式：`json` / `csv` |

**Response**：

- `format=json`：HTTP 200，`Content-Type: application/json`，响应体结构与 §3.6.10 相同（`items[]` 无分页）
- `format=csv`：HTTP 200，`Content-Type: text/csv; charset=utf-8`，`Content-Disposition: attachment; filename="audit_logs_{start_at}_{end_at}.csv"`
- 超过 10,000 条时返回 `E_GOV_4094`（422），提示缩短时间范围

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :--- | :--- |
| `E_AUTH_4031` | 403 | 非 SUPER_ADMIN |
| `E_GOV_4094` | 422 | 结果超 10,000 条 |
| `E_GOV_4095` | 422 | 时间范围超 31 天或超出 180 天归档窗口 |

---

### 3.7 查询编排层（BFF）

> **底层设计基线**：SADD V2.0 §4.1 BFF 层；非独立领域，仅做跨域聚合查询编排。
> 此层接口聚合多个域的数据，为前端提供"一次请求获取全貌"的能力。

#### 3.7.0 接口概览

| # | 方法 | URI | 功能 | 鉴权 |
| :---: | :--- | :--- | :--- | :---: |
| 1 | GET | `/api/v1/dashboard` | 首页仪表盘 | JWT |
| 2 | GET | `/api/v1/rescue/tasks/{task_id}/full` | 任务全景聚合 | JWT |

---

#### 3.7.1 首页仪表盘

**描述**：聚合当前用户名下的患者摘要、活跃任务数、未读通知数等。

**HTTP 请求**：`GET /api/v1/dashboard`

**安全性**：需鉴权（JWT）

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_054",
  "data": {
    "patient_count":         2,
    "active_task_count":     1,
    "unread_notification_count": 5,
    "patients": [
      {
        "patient_id":   "1001",
        "patient_name": "张**",
        "status":       "MISSING",
        "avatar_url":   "https://oss.example.com/avatar/1001.jpg",
        "active_task": {
          "task_id":    "8848",
          "task_no":    "TSK20260419001",
          "status":     "ACTIVE",
          "created_at": "2026-04-19T10:00:00Z",
          "clue_count": 12
        }
      }
    ]
  }
}
```

---

#### 3.7.2 任务全景聚合

**描述**：一次请求聚合任务快照、患者档案、最新线索列表、轨迹摘要、AI 会话摘要。

**HTTP 请求**：`GET /api/v1/rescue/tasks/{task_id}/full`

**安全性**：需鉴权（JWT）

**Path Variables**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `task_id` | string | 是 | 目标任务 ID |

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_055",
  "data": {
    "task": {
      "task_id": "8848",
      "task_no": "TSK20260419001",
      "status":  "ACTIVE",
      "source":  "APP",
      "created_at": "2026-04-19T10:00:00Z"
    },
    "patient": {
      "patient_id":   "1001",
      "patient_name": "张**",
      "gender":       "MALE",
      "age":          78,
      "avatar_url":   "https://oss.example.com/avatar/1001.jpg",
      "short_code":   "A3B7K9",
      "appearance": {
        "clothing": "蓝色外套",
        "features": "左耳助听器"
      }
    },
    "recent_clues": [
      {
        "clue_id":    "50001",
        "clue_no":    "CLU20260419001",
        "status":     "VALID",
        "latitude":   39.9100,
        "longitude":  116.3900,
        "created_at": "2026-04-19T15:30:00Z"
      }
    ],
    "trajectory_summary": {
      "point_count":       45,
      "latest_point_time": "2026-04-19T15:30:00Z"
    },
    "ai_session": {
      "session_id": "sess_001",
      "status":     "ACTIVE",
      "message_count": 8
    }
  }
}
```

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_TASK_4041` | 404 | 任务不存在 |
| `E_TASK_4030` | 403 | 无授权查看 |

---

## 4. WebSocket 实时推送契约

> **底层设计基线**：LLD V2.0 §11（WebSocket 路由与通知设计）、SADD V2.0 HC-08（四通道通知）

### 4.1 连接建立

**WebSocket 端点**：`wss://{host}/ws?ticket={ws_ticket}`

#### 4.1.1 获取 WebSocket Ticket

**描述**：客户端通过已鉴权的 HTTP 请求获取一次性 WebSocket Ticket，用于后续 WebSocket 连接建立（LLD §11.1）。

**HTTP 请求**：`POST /api/v1/ws/ticket`

**安全性**：需鉴权（JWT）

**Headers**：

| 字段名 | 类型 | 必填 | 描述 |
| :--- | :--- | :---: | :--- |
| `Authorization` | string | 是 | `Bearer {token}` |
| `X-Request-Id` | string | 是 | 幂等键（HC-03） |
| `X-Trace-Id` | string | 是 | 链路追踪（HC-04） |

**Request Body**：无

**Response (HTTP 200)**：

```json
{
  "code": "ok",
  "message": "success",
  "trace_id": "trc_20260419_050",
  "data": {
    "ws_ticket": "wst_xxx_onetime",
    "expires_in": 30
  }
}
```

> Ticket 一次性消费，TTL = 30s。

**异常响应**：

| 错误码 | HTTP | 触发条件 |
| :--- | :---: | :--- |
| `E_GOV_4010` | 401 | 未登录或 JWT 无效 |
| `E_GOV_4012` | 401 | `ws_ticket` 生成频率超限 |
| `E_GOV_4291` | 429 | 请求频率过高 |

### 4.2 心跳机制

| 参数 | 值 |
| :--- | :--- |
| 心跳基线 | 30s |
| 抖动窗口 | 0 ~ 8s 随机（避免同频写放大，LLD §11.1） |
| 续期阈值 | 路由键剩余 TTL < 60s 时触发续期 |

客户端发送：`{"type": "ping"}`
服务端回复：`{"type": "pong", "server_time": "2026-04-19T10:00:00Z"}`

### 4.3 下行消息格式

所有下行消息遵循统一结构：

```json
{
  "type":         "string, 消息类型",
  "aggregate_id": "string, 聚合 ID（如 task_id、patient_id）",
  "version":      "integer, 事件版本（客户端防乱序锚点）",
  "event_time":   "string, ISO-8601, 事件时间",
  "trace_id":     "string, 链路追踪",
  "payload":      {}
}
```

**客户端防乱序规则**（LLD §11.4）：

1. 主判定锚点：`version`（仅接受更大 `version`）。
2. `version` 相同时以 `event_time` 为最终锚点。
3. `version` 回退必须丢弃并记录乱序计数。

### 4.4 消息类型定义

| type | 触发事件 | 目标用户 | 说明 |
| :--- | :--- | :--- | :--- |
| `task.created` | 任务发布 | 家属 | 新任务已创建 |
| `task.state.changed` | 状态迁移 | 家属 | 含 `ACTIVE` / `SUSTAINED` / `CLOSED_FOUND` / `CLOSED_FALSE_ALARM` |
| `clue.validated` | 线索验证通过 | 家属 | 受通知节流控制（默认 300s 窗口） |
| `track.updated` | 轨迹更新 | 家属 | 受节流控制，仅在线推送 |
| `fence.breached` | 围栏突破 | 家属 | `level=CRITICAL`，受节流控制 |
| `patient.missing_pending` | 走失待确认 | 家属 | `level=CRITICAL` |
| `patient.confirmed_safe` | 确认安全 | 家属 | — |
| `tag.suspected_lost` | 标签疑似遗失 | 主监护人 | — |
| `guardian.transfer.requested` | 监护转移请求 | 受让人 | — |
| `notification.generic` | 站内通知 | 目标用户 | 兜底通知类型 |

### 4.5 降级策略

| 场景 | 降级行为 |
| :--- | :--- |
| WebSocket 路由缺失 | 降级至 JPush + 站内通知 |
| WebSocket 连接断开 | 降级至 JPush + 站内通知 |
| JPush 不可达 | 仅写站内通知 + 审计日志 |
| 路由存储短暂不可用 | 推送/站内通知兜底 + 可观测降级告警 |

### 4.6 通知节流

> 对应 LLD §8.4.2 通知路由规则

| 事件类型 | 节流窗口 | 配置键 |
| :--- | :--- | :--- |
| `clue.validated` | 默认 300s | `notification.clue_throttle.interval_seconds` |
| `track.updated` | 默认 300s | 同上 |
| `fence.breached` | 默认 300s | 同上 |

节流窗口内新事件**仅写站内通知**，不触发 WebSocket / JPush。

---

## 5. 事件总线契约（Redis Streams）

> **底层设计基线**：SADD V2.0 ADR-006（Redis Streams）、LLD V2.0 §9（Outbox 模型）
> **传输层**：Redis Streams，所有事件通过 Outbox 模式写入，Polling Dispatcher 投递。

### 5.1 TASK 域事件

#### 5.1.1 task.created

```json
{
  "event_type":  "task.created",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T10:00:00Z",
  "payload": {
    "task_id":    "8848",
    "task_no":    "TSK20260419001",
    "patient_id": "1001",
    "source":     "APP",
    "reported_by": "2001",
    "status":     "CREATED"
  }
}
```

消费方：CLUE 域（状态投影）、GOV 域（通知分发）、AI 域（会话关联）

#### 5.1.2 task.closed.found / task.closed.false_alarm

```json
{
  "event_type":  "task.closed.found",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T18:30:00Z",
  "payload": {
    "task_id":    "8848",
    "patient_id": "1001",
    "close_type": "FOUND",
    "closed_by":  "2001"
  }
}
```

消费方：AI 域（清除走失豁免、会话归档、记忆沉淀/阻断）、GOV 域（通知分发）、CLUE 域（状态投影更新）

#### 5.1.3 task.sustained

```json
{
  "event_type":  "task.sustained",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T16:00:00Z",
  "payload": {
    "task_id":     "8848",
    "patient_id":  "1001",
    "sustained_by": "2001"
  }
}
```

#### 5.1.4 task.state.changed

```json
{
  "event_type":  "task.state.changed",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T10:01:00Z",
  "payload": {
    "task_id":    "8848",
    "patient_id": "1001",
    "old_status": "CREATED",
    "new_status": "ACTIVE",
    "version":    2
  }
}
```

消费方：CLUE 域（L2 Redis 投影缓存 `clue:taskstate:l2:{patient_id}`）

### 5.2 CLUE 域事件

#### 5.2.1 clue.reported.raw

```json
{
  "event_type":  "clue.reported.raw",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T15:30:00Z",
  "payload": {
    "clue_id":    "50001",
    "patient_id": "1001",
    "task_id":    "8848",
    "latitude":   39.9100,
    "longitude":  116.3900,
    "reporter_type": "ANONYMOUS"
  }
}
```

#### 5.2.2 clue.validated

```json
{
  "event_type":  "clue.validated",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T15:30:05Z",
  "payload": {
    "clue_id":    "50001",
    "patient_id": "1001",
    "task_id":    "8848",
    "latitude":   39.9100,
    "longitude":  116.3900
  }
}
```

消费方：GOV 域（通知）、轨迹服务（写入 `patient_trajectory`）

#### 5.2.3 track.updated

```json
{
  "event_type":  "track.updated",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T15:30:06Z",
  "payload": {
    "trajectory_id": "70001",
    "patient_id":    "1001",
    "task_id":       "8848",
    "latitude":      39.9100,
    "longitude":     116.3900,
    "version":       5
  }
}
```

#### 5.2.4 fence.breached

```json
{
  "event_type":  "fence.breached",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T15:35:00Z",
  "payload": {
    "patient_id": "1001",
    "fence_center_lat": 39.9100,
    "fence_center_lng": 116.3900,
    "fence_radius_m":   1000,
    "breach_lat":       39.9250,
    "breach_lng":       116.4100,
    "distance_m":       1500
  }
}
```

消费方：PROFILE 域（`NORMAL → MISSING_PENDING`）、GOV 域（`CRITICAL` 通知）

### 5.3 PROFILE 域事件

#### 5.3.1 profile.created

```json
{
  "event_type":  "profile.created",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T09:00:00Z",
  "payload": {
    "patient_id":      "1001",
    "profile_no":      "PRO20260419001",
    "profile_version": 1
  }
}
```

消费方：AI 域（触发向量化）

#### 5.3.2 profile.updated

```json
{
  "event_type":  "profile.updated",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T11:00:00Z",
  "payload": {
    "patient_id":      "1001",
    "profile_version": 3,
    "changed_fields":  ["long_text_profile", "chronic_diseases"]
  }
}
```

消费方：AI 域（触发向量重建）

#### 5.3.3 profile.deleted.logical

```json
{
  "event_type":  "profile.deleted.logical",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T20:00:00Z",
  "payload": {
    "patient_id": "1001"
  }
}
```

消费方：AI 域（物理删除向量、归档会话与记忆笔记）

#### 5.3.4 patient.missing_pending

```json
{
  "event_type":  "patient.missing_pending",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T15:36:00Z",
  "payload": {
    "patient_id":  "1001",
    "trigger":     "FENCE_BREACH",
    "trigger_clue_id": "50001"
  }
}
```

消费方：GOV 域（`CRITICAL` 通知推送）

#### 5.3.5 patient.confirmed_safe

```json
{
  "event_type":  "patient.confirmed_safe",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T15:40:00Z",
  "payload": {
    "patient_id":   "1001",
    "confirmed_by": "2001"
  }
}
```

### 5.4 AI 域事件

#### 5.4.1 memory.appended

```json
{
  "event_type":  "memory.appended",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T18:00:00Z",
  "payload": {
    "note_id":    "note_001",
    "patient_id": "1001",
    "kind":       "RESCUE_CASE",
    "source_event_id": "evt_task_closed_found_xxx"
  }
}
```

消费方：ai-vectorizer-service（记忆向量化）

#### 5.4.2 memory.expired

```json
{
  "event_type":  "memory.expired",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T19:00:00Z",
  "payload": {
    "note_id":    "note_001",
    "patient_id": "1001",
    "reason":     "PROFILE_DELETED"
  }
}
```

消费方：ai-vectorizer-service（清理 Embedding）

#### 5.4.3 ai.strategy.generated

```json
{
  "event_type":  "ai.strategy.generated",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T10:08:00Z",
  "payload": {
    "task_id":       "8848",
    "patient_id":    "1001",
    "session_id":    "sess_001",
    "strategy_type": "SEARCH_AREA_RECOMMENDATION",
    "summary":       "建议在朝阳公园西门至北门区域搜寻",
    "confidence":    0.82
  }
}
```

#### 5.4.4 ai.poster.generated

```json
{
  "event_type":  "ai.poster.generated",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T10:09:00Z",
  "payload": {
    "task_id":     "8848",
    "patient_id":  "1001",
    "poster_url":  "https://oss.example.com/posters/task_8848.png",
    "template_id": "TPL_MISSING_V1"
  }
}
```

### 5.5 MAT 域事件

#### 5.5.1 material.order.created

```json
{
  "event_type":  "material.order.created",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T09:00:00Z",
  "payload": {
    "order_id":   "6001",
    "order_no":   "ORD20260419001",
    "patient_id": "1001",
    "applicant_user_id": "2001",
    "quantity":   2
  }
}
```

#### 5.5.2 material.order.approved

```json
{
  "event_type":  "material.order.approved",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T10:00:00Z",
  "payload": {
    "order_id":   "6001",
    "order_no":   "ORD20260419001",
    "patient_id": "1001",
    "reviewer_user_id": "9001"
  }
}
```

#### 5.5.3 material.order.shipped

```json
{
  "event_type":  "material.order.shipped",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T14:00:00Z",
  "payload": {
    "order_id":   "6001",
    "order_no":   "ORD20260419001",
    "patient_id": "1001",
    "tag_codes":  ["TAG20260419001", "TAG20260419002"]
  }
}
```

#### 5.5.4 tag.allocated

```json
{
  "event_type":  "tag.allocated",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T14:00:00Z",
  "payload": {
    "tag_code":   "TAG20260419001",
    "order_id":   "6001",
    "patient_id": "1001",
    "resource_token_generated": true
  }
}
```

#### 5.5.5 tag.bound

```json
{
  "event_type":  "tag.bound",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T17:00:00Z",
  "payload": {
    "tag_code":   "TAG20260419001",
    "patient_id": "1001",
    "bound_by":   "2001"
  }
}
```

#### 5.5.6 tag.loss.confirmed

```json
{
  "event_type":  "tag.loss.confirmed",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T18:00:00Z",
  "payload": {
    "tag_code":   "TAG20260419001",
    "patient_id": "1001",
    "lost_by":    "2001"
  }
}
```

消费方：GOV 域（通知 `tag.suspected_lost`）

### 5.6 GOV 域事件

#### 5.6.1 notification.sent

```json
{
  "event_type":  "notification.sent",
  "event_id":    "evt_01H...",
  "trace_id":    "trc_xxx",
  "occurred_at": "2026-04-19T10:05:00Z",
  "payload": {
    "notification_id": "9001",
    "user_id":         "2001",
    "channel":         "WEBSOCKET",
    "type":            "TASK_PROGRESS",
    "source_event":    "task.created",
    "result":          "DELIVERED"
  }
}
```

---

## 6. AI Agent 能力清单与 Function Calling 白名单

> **底层设计基线**：LLD V2.0 §7.2（Function Calling 白名单 + Policy Guard）

### 6.1 执行等级定义

| 等级 | 含义 | 用户交互 |
| :--- | :--- | :--- |
| `A0` | 纯读取（查询） | 自动执行，无需确认 |
| `A1` | 低风险写入 | 自动执行（需审计） |
| `A2` | 高风险写入 | 需用户一次确认（`tool_call` → `confirm`） |
| `A3` | 行政级操作 | 需用户二次确认 |
| `A4` | 禁止自动执行 | Agent 仅展示建议，完全由用户手动操作 |

### 6.2 Function Calling 白名单

**家属侧可调用操作**：

| 函数名 | 执行等级 | 对应 API | 说明 |
| :--- | :---: | :--- | :--- |
| `query_task_snapshot` | A0 | `GET /api/v1/rescue/tasks/{id}/snapshot` | 查询任务快照 |
| `query_trajectory` | A0 | `GET /api/v1/rescue/tasks/{id}/trajectory/latest` | 查询轨迹 |
| `query_clues` | A0 | `GET /api/v1/clues` | 查询线索列表 |
| `query_patient_profile` | A0 | `GET /api/v1/patients/{id}` | 查询患者档案 |
| `generate_poster` | A1 | `POST /api/v1/ai/poster` | 输出 JSON 文案，经敏感词过滤 |
| `create_task` | A2 | `POST /api/v1/rescue/tasks` | 发布任务 |
| `submit_material_order` | A2 | `POST /api/v1/material/orders` | AI 辅助填写收货地址，家属确认 |
| `report_tag_lost` | A2 | `POST /api/v1/tags/{tag_code}/loss/confirm` | 列出绑定标签供家属勾选 |
| `update_fence_config` | A2 | `PUT /api/v1/patients/{id}/fence` | 严禁 AI 直接关闭围栏 |
| `update_daily_appearance` | A2 | `PUT /api/v1/patients/{id}/appearance` | 当日着装为最高视觉锚点 |
| `update_patient_profile` | A2 | `PUT /api/v1/patients/{id}/profile` | 长期 RAG 素材，需家属确认 |
| `sustained_task` | A2 | `POST /api/v1/rescue/tasks/{id}/sustained` | 标记长期维持 |
| `propose_close_found` | A3 | `POST /api/v1/rescue/tasks/{id}/close` | 极高风险，必须物理点击确认 |
| `confirm_missing` | A3 | `POST /api/v1/patients/{id}/missing-pending/confirm` | 确认走失 |

**管理侧可调用操作**：

| 函数名 | 执行等级 | 对应 API | 说明 |
| :--- | :---: | :--- | :--- |
| `approve_material_order` | A2 | `POST /api/v1/material/orders/{id}/approve` | 审核通过 |
| `replay_outbox_dead` | A3 | `POST /api/v1/admin/super/outbox/dead/{id}/replay` | DEAD 事件受控重放 |

### 6.3 Policy Guard 校验链

```
请求 → 白名单检查 → 角色权限检查 → 数据归属检查 → 执行模式检查 → 确认等级检查 → 放行/拒绝
```

拒绝时返回 `E_GOV_4039`（403），并写审计日志（`blocked_reason`）。

---

## 7. 接口覆盖矩阵（LLD → API 映射）

| LLD 章节 | SRS 编号 | API 端点 | 本文档章节 | 优先级 |
| :--- | :--- | :--- | :---: | :---: |
| §3 TASK | FR-TASK-001 | `POST /api/v1/rescue/tasks` | §3.1.1 | P0 |
| §3 TASK | FR-TASK-002 | `POST /api/v1/rescue/tasks/{id}/close` | §3.1.2 | P0 |
| §3 TASK | FR-TASK-003 | `GET /api/v1/rescue/tasks/{id}/snapshot` | §3.1.3 | P0 |
| §3 TASK | FR-TASK-004 | `POST /api/v1/rescue/tasks/{id}/sustained` | §3.1.4 | P0 |
| §3 TASK | — | `GET /api/v1/rescue/tasks` | §3.1.5 | P1 |
| §4 CLUE | FR-CLUE-002 | `GET /r/{resource_token}` | §3.2.1 | P0 |
| §4 CLUE | FR-CLUE-003 | `POST /api/v1/public/clues/manual-entry` | §3.2.2 | P0 |
| §4 CLUE | FR-CLUE-001 | `POST /api/v1/clues/report` | §3.2.3 | P0 |
| §4 CLUE | FR-CLUE-007 | `POST /api/v1/clues/{id}/override` | §3.2.4 | P0 |
| §4 CLUE | FR-CLUE-007 | `POST /api/v1/clues/{id}/reject` | §3.2.5 | P0 |
| §4 CLUE | — | `GET /api/v1/clues` | §3.2.6 | P1 |
| §4 CLUE | FR-CLUE-006 | `GET /api/v1/rescue/tasks/{id}/trajectory/latest` | §3.2.7 | P0 |
| §5 PROFILE | FR-PRO-001 | `POST /api/v1/patients` | §3.3.1 | P0 |
| §5 PROFILE | FR-PRO-001 | `PUT /api/v1/patients/{id}/profile` | §3.3.2 | P0 |
| §5 PROFILE | FR-TASK-003 | `PUT /api/v1/patients/{id}/appearance` | §3.3.3 | P0 |
| §5 PROFILE | FR-PRO-010 | `PUT /api/v1/patients/{id}/fence` | §3.3.4 | P0 |
| §5 PROFILE | §5.2.1 | `POST /api/v1/patients/{id}/missing-pending/confirm` | §3.3.5 | P0 |
| §5 PROFILE | FR-PRO-006 | `POST /api/v1/patients/{id}/guardians/invitations` | §3.3.6 | P0 |
| §5 PROFILE | FR-PRO-006 | `POST .../invitations/{id}/respond` | §3.3.7 | P0 |
| §5 PROFILE | FR-PRO-007 | `POST /api/v1/patients/{id}/guardians/primary-transfer` | §3.3.8 | P0 |
| §5 PROFILE | FR-PRO-007 | `POST .../primary-transfer/{id}/respond` | §3.3.9 | P0 |
| §5 PROFILE | FR-PRO-007 | `POST .../primary-transfer/{id}/cancel` | §3.3.10 | P0 |
| §5 PROFILE | FR-PRO-006 | `DELETE /api/v1/patients/{id}/guardians/{user_id}` | §3.3.11 | P0 |
| §5 PROFILE | — | `GET /api/v1/patients/{id}` | §3.3.12 | P0 |
| §5 PROFILE | — | `GET /api/v1/patients` | §3.3.13 | P1 |
| §5 PROFILE | FR-PRO-009 | `DELETE /api/v1/patients/{id}` | §3.3.14 | P0 |
| §6 MAT | FR-MAT-001 | `POST /api/v1/material/orders` | §3.4.1 | P0 |
| §6 MAT | FR-MAT-001 | `POST .../orders/{id}/approve` | §3.4.2 | P0 |
| §6 MAT | FR-MAT-002 | `POST .../orders/{id}/ship` | §3.4.3 | P0 |
| §6 MAT | — | `POST .../orders/{id}/receive` | §3.4.4 | P0 |
| §6 MAT | — | `POST .../orders/{id}/cancel` | §3.4.5 | P1 |
| §6 MAT | FR-MAT-003 | `POST /api/v1/tags/{code}/bind` | §3.4.6 | P0 |
| §6 MAT | §5.2.3 | `POST /api/v1/tags/{code}/loss/confirm` | §3.4.7 | P0 |
| §6 MAT | FR-MAT-005 | `POST /api/v1/tags/batch-generate` | §3.4.8 | P0 |
| §6 MAT | FR-MAT-005 | `GET .../batch-generate/jobs/{id}` | §3.4.9 | P0 |
| §6 MAT | FR-MAT-006 | `GET /api/v1/tags/inventory/summary` | §3.4.10 | P0 |
| §6 MAT | — | `GET /api/v1/material/orders` | §3.4.11 | P1 |
| §7 AI | FR-AI-001 | `POST /api/v1/ai/sessions` | §3.5.1 | P0 |
| §7 AI | FR-AI-001 | `POST .../sessions/{id}/messages` | §3.5.2 | P0 |
| §7 AI | FR-AI-007 | `POST .../intents/{id}/confirm` | §3.5.3 | P0 |
| §7 AI | FR-AI-014 | `POST .../sessions/{id}/feedback` | §3.5.4 | P2 |
| §7 AI | FR-AI-013 | `POST /api/v1/ai/poster` | §3.5.5 | P1 |
| §8 GOV | FR-GOV-001 | `POST /api/v1/auth/register` | §3.6.1 | P0 |
| §8 GOV | FR-GOV-001 | `POST /api/v1/auth/login` | §3.6.2 | P0 |
| §8 GOV | — | `POST /api/v1/auth/token/refresh` | §3.6.3 | P0 |
| §8 GOV | FR-GOV-002 | `POST /api/v1/auth/password-reset/request` | §3.6.4 | P0 |
| §8 GOV | FR-GOV-002 | `POST /api/v1/auth/password-reset/confirm` | §3.6.5 | P0 |
| §8 GOV | — | `GET /api/v1/users/me` | §3.6.6 | P0 |
| §8 GOV | — | `PUT /api/v1/users/me/password` | §3.6.7 | P1 |
| §8 GOV | FR-GOV-008 | `PUT /api/v1/admin/configs/{key}` | §3.6.8 | P0 |
| §8 GOV | — | `GET /api/v1/admin/configs` | §3.6.9 | P1 |
| §8 GOV | FR-GOV-006 | `GET /api/v1/admin/logs` | §3.6.10 | P1 |
| §8 GOV | FR-GOV-010 | `GET /api/v1/notifications/inbox` | §3.6.11 | P0 |
| §8 GOV | — | `POST .../notifications/{id}/read` | §3.6.12 | P1 |
| §8 GOV | §5.3 | `POST .../outbox/dead/{id}/replay` | §3.6.13 | P1 |
| §8 GOV | — | `GET .../outbox/dead` | §3.6.14 | P1 |
| BFF | — | `GET /api/v1/dashboard` | §3.7.1 | P0 |
| BFF | — | `GET .../tasks/{id}/full` | §3.7.2 | P0 |
| WS | — | `POST /api/v1/ws/ticket` | §4.1 | P0 |

---

## 8. 一致性自检报告

| 检查项 | 覆盖状态 | 不符合项 |
| :--- | :---: | :--- |
| 路径符合 RESTful 规范（无动词） | ⚠️ | 大多数接口为名词资源 + HTTP Method；部分 non-CRUD 操作采用 `POST + 动作子资源` 模式（如 `/close`、`/confirm`、`/approve`、`/bind` 等），属本项目设计决策 |
| 非安全请求已配置幂等 Headers（HC-03） | ✅ | 所有 POST / PUT / PATCH / DELETE 写接口均要求 `X-Request-Id` |
| 暴露的 DTO 字段是否包含未脱敏 PII（HC-07） | ✅ | `patient_name` / `email` / `phone` / `emergency_contact_phone` 均已标注脱敏规则 |
| 分页与列表查询接口规范一致性（HC-05） | ✅ | Offset 模式统一 `page_no` / `page_size` / `total` / `has_next`；Cursor 模式统一 `next_cursor` / `has_next` |
| 所有字段类型与 DBD 彻底对齐 | ✅ | 任务状态 `CREATED`/`ACTIVE`/`SUSTAINED`/`CLOSED_FOUND`/`CLOSED_FALSE_ALARM`、患者走失状态 `NORMAL`/`MISSING_PENDING`/`MISSING`、标签状态 `UNBOUND`/`ALLOCATED`/`BOUND`/`SUSPECTED_LOST`/`LOST`/`VOIDED`、工单状态 `PENDING_AUDIT`/`PENDING_SHIP`/`SHIPPED`/`RECEIVED`/`EXCEPTION`/`VOIDED`、监护角色 `PRIMARY_GUARDIAN`/`GUARDIAN` 均与 DBD CHECK 一致 |
| 事件总线选型与 SADD 一致（Redis Streams） | ✅ | 全文无 Kafka 引用 |
| 事件命名与 LLD V2.0 一致 | ✅ | `task.closed.found` / `task.closed.false_alarm`、`clue.reported.raw` 等与 LLD §2.1 对齐 |
| ID 传输统一为 string（JSON Body） | ✅ | §1.1 强制约定 + HTTP 响应与事件 payload 中 ID 均为字符串 |
| 全链路追踪 `X-Trace-Id` 覆盖（HC-04） | ✅ | 请求 / 响应 / 事件 payload 均含 `trace_id` |
| AI Agent 受控执行契约完整性 | ✅ | §1.11 + §6 + `E_GOV_4039` / `E_GOV_4231` |
| 匿名接口 `device_fingerprint` 风控（HC-06） | ✅ | §3.2.2 / §3.2.3 强制要求 |
| 通知四通道对齐（HC-08） | ✅ | §4 WebSocket + JPush + Email + NoOp SMS |
| 业务单号（`task_no` / `clue_no` / `profile_no` / `order_no`）统一使用 | ✅ | 所有域响应均包含业务单号 |
| 错误码段不重叠 | ✅ | 通用 `E_REQ` / `E_GOV` / `E_SYS`，域级 `E_TASK` / `E_CLUE` / `E_PRO` / `E_MAT` / `E_AI` / `E_AUTH` / `E_USR` / `E_NOTI` |
| 坐标系统一约定（WGS84 存储） | ✅ | §1.9 + 所有坐标字段 `coord_system` 标注 |
| `tag_only` 字段支持（FR-CLUE-004） | ✅ | §3.2.3 线索上报请求体已包含 `tag_only` 布尔字段 |

---