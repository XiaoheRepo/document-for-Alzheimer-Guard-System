# 基于AI Agent的阿尔兹海默症患者协同寻回系统
## Android 前端开发手册

## 0. 文档信息

| 项目 | 内容 |
| :--- | :--- |
| 文档名称 | Android 前端开发手册 |
| 版本 | V1.1 |
| 日期 | 2026-04-07 |
| 输入基线 | SRS_simplify.md、SADD_from_SRS_simplify.md、LLD_from_SRS_SADD.md、API_from_SRS_SADD_LLD.md、backend_handbook.md |
| 适用对象 | Android 开发、测试、联调、架构评审 |

说明：
1. 本手册用于指导 Android 客户端从 0 到 1 实现与后端一致的业务系统。
2. 接口字段、错误码、事件语义以上游 API 文档为准；本手册定义客户端落地方式。
3. 本手册强调可执行标准，所有规则应最终转化为代码、测试或 CI 校验。

## 1. 目标与范围

### 1.1 目标

1. 保证 Android 客户端与后端契约严格一致，避免联调期“字段对了但语义错了”。
2. 保证关键场景稳定可用：登录、任务追踪、匿名线索上报、通知实时推送、AI 辅助。
3. 保证可维护性：模块清晰、状态可追踪、错误可定位、发布可回滚。

### 1.2 范围

1. 架构与项目结构。
2. 网络层与接口契约落地。
3. 业务模块实现规范。
4. 本地缓存、离线与重试。
5. WebSocket/SSE 实时链路与降级。
6. 测试、质量门禁、发布流程。

### 1.3 需求追踪矩阵（SRS -> Android）

| 需求/规则 | Android 责任 | 关键页面 | 关键接口 | 验收编号 |
| :--- | :--- | :--- | :--- | :--- |
| `FR-CLUE-008`（扫码动态路由） | 根据标签状态执行正确页面路由，不做本地猜测兜底 | `PUB-01`、`PUB-03`、`PUB-04` | `GET /r/{resource_token}` | `ACC-FE-001` |
| `FR-CLUE-003` + `BR-001`（匿名兜底准入） | 手动录入仅提交必要字段并执行冷却倒计时 | `PUB-02` | `POST /api/v1/public/clues/manual-entry` | `ACC-FE-002` |
| `FR-CLUE-005`（存疑线索闭环） | Android 侧消费复核结果回流并刷新线索终态，不出现悬挂态 | `CLUE-01` | `GET /api/v1/clues/{clue_id}` | `ACC-FE-003` |
| `FR-TASK-001`（唯一进行中任务） | 新建任务前后正确处理冲突并跳转已有任务 | `TASK-04`、`TASK-02` | `POST /api/v1/rescue/tasks` | `ACC-FE-004` |
| `FR-TASK-003/004`（任务关闭） | `FALSE_ALARM` 强制原因校验，终态不可重复提交 | `TASK-05` | `POST /api/v1/rescue/tasks/{task_id}/close` | `ACC-FE-005` |
| `FR-PRO-005/006` + `BR-006`（主监护转移） | 仅允许目标受方确认；成员移除后隐藏失效转移动作 | `GUA-01` | 监护转移相关接口 | `ACC-FE-006` |
| `FR-PRO-008`（围栏配置） | 围栏参数本地校验并对 `E_PRO_4221` 精准提示 | `PAT-03` | `PUT /api/v1/patients/{patient_id}/fence` | `ACC-FE-007` |
| `FR-TASK-005` + `HC-06`（通知触达） | 仅实现“应用推送 + 站内通知”双通道，不依赖短信 | `NOTI-01`、`TASK-02` | `GET /api/v1/notifications` | `ACC-FE-008` |
| `HC-03/HC-04`（幂等与追踪） | 写接口注入 `X-Request-Id`，全链路注入 `X-Trace-Id` | 全写接口页面 | 所有写接口 | `ACC-FE-009` |
| `HC-05`（WS 路由化下发） | 客户端执行去重、防乱序与补洞，不做全量广播假设 | `TASK-03`、`NOTI-01` | `POST /api/v1/ws/tickets`、轮询补洞接口 | `ACC-FE-010` |

说明：验收编号在第 17.4 节给出可执行测试口径。

## 2. 技术栈与版本基线

| 类别 | 选型 | 说明 |
| :--- | :--- | :--- |
| 语言 | Kotlin | 全量 Kotlin，禁止新增 Java 业务代码 |
| 最低系统 | minSdk 26 | 覆盖主流设备能力 |
| 目标系统 | targetSdk 35 | 紧跟平台能力与合规要求 |
| UI | Jetpack Compose + Material 3 | 统一声明式 UI |
| UI 第三方组件治理 | Whitelist + Adapter + Version Catalog | 成体系引入，避免 feature 层直接耦合外部库 |
| 架构 | Clean Architecture + MVI/UDF | 分层清晰，单向数据流 |
| DI | Hilt | 统一依赖注入 |
| 网络 | OkHttp + Retrofit + Kotlin Serialization | 可观测、可测试 |
| 并发 | Kotlin Coroutines + Flow | 生命周期安全 |
| 本地存储 | Room + DataStore | 结构化缓存与轻量配置 |
| 后台任务 | WorkManager | 可靠重试与离线补偿 |
| 日志与监控 | Timber + Crashlytics/等价方案 | 线上可诊断 |
| 测试 | JUnit5 + Turbine + MockWebServer + Compose UI Test | 分层测试覆盖 |

版本升级原则：
1. 只允许小步升级，每次升级必须附带回归测试结果。
2. 涉及网络协议或序列化升级时，必须跑契约测试。

## 3. 项目结构与模块职责

推荐多模块结构：

```text
android-app/
  app/                            # 壳工程、导航、全局初始化
  core/
    core-common/                  # 通用工具、Result、时间与ID工具
    core-network/                 # Retrofit/OkHttp、拦截器、错误解析
    core-database/                # Room DB、DAO、迁移脚本
    core-datastore/               # Token、设置、轻量状态
    core-ui/                      # Design System、主题、通用组件
    core-ui-adapter/              # 第三方 UI 组件适配层（唯一外部 UI 库接入点）
    core-realtime/                # WebSocket/SSE、重连与去重
    core-testing/                 # 测试桩、假数据、测试规则
  feature/
    feature-auth/                 # 注册/登录/登出/改密
    feature-task/                 # 搜救任务、状态流转、进度页
    feature-clue/                 # 匿名线索上报、人工录入
    feature-profile/              # 患者档案与标签状态
    feature-notification/         # 消息中心、已读管理
    feature-ai/                   # AI 评估与建议交互
  build-logic/                    # Convention Plugin、统一构建规则
```

模块规则：
1. feature 模块禁止直接依赖其他 feature。
2. feature 只能依赖 core 与 domain 接口。
3. 任何网络 DTO 不得直接暴露到 UI 层。
4. feature 模块禁止直接依赖第三方 UI 库，必须经 `core-ui` 或 `core-ui-adapter` 暴露能力。
5. 第三方 UI 库的主题映射必须落在 `core-ui-adapter`，避免多处维护。
6. 每个第三方能力都必须有降级方案（关闭开关或替代实现）。

## 4. 分层架构与状态管理

### 4.1 四层模型

1. data 层：RemoteDataSource、LocalDataSource、RepositoryImpl。
2. domain 层：UseCase、DomainModel、业务规则。
3. presentation 层：ViewModel、UiState、UiEvent、UiEffect。
4. ui 层：Compose 页面，只消费 UiState 与回传 UiEvent。

### 4.2 MVI 规范

1. `UiState` 必须为不可变 data class。
2. `UiEvent` 表示用户意图，`UiEffect` 表示一次性副作用（Toast/导航）。
3. ViewModel 仅通过 reducer 更新状态，禁止在多个协程中直接改同一状态字段。

示例：

```kotlin
data class LoginUiState(
    val username: String = "",
    val loading: Boolean = false,
    val error: String? = null,
    val loggedIn: Boolean = false
)

sealed interface LoginUiEvent {
    data class UsernameChanged(val value: String) : LoginUiEvent
    data object Submit : LoginUiEvent
}
```

## 5. 数据模型与映射规范

### 5.1 模型分层

| 层级 | 命名 | 说明 |
| :--- | :--- | :--- |
| 网络层 | `XxxDto` | 严格对应接口字段 |
| 领域层 | `Xxx` | 业务语义模型 |
| 展示层 | `XxxUiModel` | 适配 UI 展示 |
| 持久层 | `XxxEntity` | Room 表结构 |

### 5.2 强制规则

1. 所有 ID 字段一律使用 `String`，禁止转 `Long` 后再传回接口。
2. 时间字段统一解析为 `Instant` 或 `OffsetDateTime`，UI 层再格式化。
3. 枚举字段必须有 `UNKNOWN` 兜底，避免后端新增枚举导致崩溃。
4. DTO -> Domain 映射必须集中在 mapper 文件，禁止散落在 UI。

示例：

```kotlin
@Serializable
data class NotificationDto(
    val notification_id: String,
    val type: String,
    val title: String,
    val content: String,
    val read_status: String,
    val created_at: String
)

data class Notification(
    val id: String,
    val type: NotificationType,
    val title: String,
    val content: String,
    val read: Boolean,
    val createdAt: Instant
)
```

## 6. 网络层与接口契约落地

### 6.1 全局 Header 规则

1. 所有请求必须带 `X-Trace-Id`。
2. 所有写请求（POST/PUT/PATCH/DELETE）必须带 `X-Request-Id`。
3. 受保护接口必须带 `Authorization: Bearer <access_token>`。
4. Android 匿名链路统一使用 `X-Anonymous-Token`；客户端不使用 Cookie 凭据。

### 6.2 拦截器落地

```kotlin
class TraceAndIdempotencyInterceptor(
    private val traceIdProvider: () -> String,
    private val requestIdProvider: () -> String,
    private val tokenProvider: () -> String?
) : Interceptor {

    override fun intercept(chain: Interceptor.Chain): Response {
        val original = chain.request()
        val method = original.method.uppercase()
        val builder = original.newBuilder()
            .header("X-Trace-Id", traceIdProvider())

        tokenProvider()?.let { builder.header("Authorization", "Bearer $it") }

        if (method in setOf("POST", "PUT", "PATCH", "DELETE")) {
            builder.header("X-Request-Id", requestIdProvider())
        }

        return chain.proceed(builder.build())
    }
}
```

### 6.3 响应结构

统一按以下结构解析：

```json
{
  "code": "OK",
  "message": "success",
  "trace_id": "trc_xxx",
  "data": {}
}
```

客户端规则：
1. HTTP 非 2xx 一律进入错误分支。
2. HTTP 2xx 但 `code != OK` 视为业务失败。
3. `trace_id` 必须落日志，作为线上问题定位主键。

### 6.4 限流与退避

1. 当收到 429 时，优先读取 `Retry-After`。
2. 若缺失 `Retry-After`，采用指数退避 + 抖动。
3. 退避公式：`delay = min(base * 2^attempt + jitter, maxDelay)`。

### 6.5 契约一致性校验清单（联调前必过）

1. 所有 ID 字段按字符串处理与传输，禁止在网络层转数值再回传。
2. 写接口必须注入 `X-Request-Id`，长度与字符集满足契约约束。
3. 全请求注入 `X-Trace-Id`，并在错误日志中关联输出。
4. 遇到 `E_REQ_4003`（保留头伪造）时，客户端不得重试并提示版本或代理异常。
5. 429 场景必须优先读 `Retry-After`，无该头才走本地指数退避。
6. 401（`E_GOV_4011`）统一触发会话失效清理，不允许页面各自处理。
7. 匿名链路仅使用 `X-Anonymous-Token`，不得在 Body 透传敏感匿名凭据。

## 7. 鉴权与会话管理

覆盖接口：
1. `POST /api/v1/auth/register`
2. `POST /api/v1/auth/login`
3. `POST /api/v1/auth/logout`
4. `PUT /api/v1/users/me/password`

会话策略：
1. Access Token 使用短时有效期，过期自动刷新或引导重登。
2. Token 存储必须加密（EncryptedSharedPreferences 或等价安全存储）。
3. 401 (`E_GOV_4011`) 统一走会话失效处理，清空本地敏感缓存并跳转登录页。
4. 登出成功后，必须清理：Token、用户信息缓存、WebSocket 连接、通知未读状态缓存。

## 8. 匿名扫码与手动录入链路

端边界说明：
1. 本章仅定义 Android 原生客户端的匿名线索上报实现。
2. 仅覆盖 Android 原生页面与接口调用，不包含其他端实现。
3. 扫码后由 `GET /r/{resource_token}` 返回路由结果，客户端按结果进入 `PUB-03` 或 `PUB-04`。

### 8.1 关键接口

1. `GET /r/{resource_token}`
2. `POST /api/v1/public/clues/manual-entry`
3. `POST /api/v1/clues/report`

### 8.2 客户端强约束

1. Android 客户端不使用 `entry_token`（含 Body/Cookie）。
2. 匿名凭据统一使用 `X-Anonymous-Token`。
3. 若服务端返回 `E_CLUE_4012`，必须清除本地匿名令牌并提示重新扫码。
4. 线索上报成功后仅展示回执，不展示患者隐私信息。

### 8.3 风控处理

1. `E_GOV_4291` 或 `E_GOV_4292` 时读取 `Retry-After`，禁用提交按钮到期后恢复。
2. 人工录入连续失败时要给出冷却提示与倒计时。

### 8.4 扫码动态路由矩阵（FR-CLUE-008）

| 标签状态 | 路由结果 | 客户端动作 | 禁止行为 |
| :--- | :--- | :--- | :--- |
| `BOUND` | 普通匿名上报 | 跳转 `PUB-03`，展示普通上报文案 | 禁止进入紧急上报 |
| `LOST` | 紧急上报 | 跳转 `PUB-04`，默认聚焦紧急提交 | 禁止降级为普通上报 |
| `UNBOUND` | 拦截提示 | 展示“标签未绑定，暂不可上报” | 禁止展示患者信息 |
| `ALLOCATED` | 拦截提示 | 展示“标签待绑定，请联系监护人” | 禁止放行到上报页 |
| `VOID` | 拦截提示 | 展示“标签已作废”并引导重新确认标签 | 禁止进入任何上报链路 |

实现约束：路由结果以后端返回为准；客户端不得根据本地缓存推断标签状态。

## 9. 地图与坐标系统一规范

1. Android 高德定位原始坐标是 `GCJ-02`，上报必须声明 `coord_system=GCJ-02`。
2. 服务端读接口统一返回 `WGS84`。
3. UI 在地图展示时，如地图 SDK 需要 `GCJ-02`，只能在展示层转换，不得污染领域层数据。
4. 坐标字段保留 `Double` 精度，禁止提前四舍五入。

## 10. 核心业务模块实现规范

### 10.1 任务模块（feature-task）

1. 任务状态流转必须由服务端状态机驱动，客户端只做可操作性判断。
2. 同一任务页面只维护一个权威 `TaskDetailUiState`。
3. 实时事件进入后先比对 `version` 再更新 UI。

### 10.2 线索模块（feature-clue）

1. 线索表单提交前必须本地校验经纬度与描述长度。
2. 图片 URL 必须来自白名单上传通道。
3. 提交成功后清空本地草稿，失败保留草稿可重试。

### 10.3 通知模块（feature-notification）

覆盖接口：
1. `GET /api/v1/notifications`
2. `PUT /api/v1/notifications/{notification_id}/read`
3. `PUT /api/v1/notifications/read-all`

规则：
1. 列表优先使用 Cursor 翻页。
2. 本地读状态更新采用“先本地后回滚”策略，接口失败时回滚并提示。
3. 触达通道仅支持“应用推送 + 站内通知”，不实现短信分支。

### 10.4 AI 模块（feature-ai）

1. AI 请求超时必须可配置。
2. `E_AI_4292`、`E_AI_4293` 必须识别并展示“稍后重试/已降级”提示。
3. 若返回 `fallback_response.mode`，在 UI 上明确标注为降级结果。

### 10.5 关键状态机守卫矩阵（客户端可操作性）

#### 10.5.1 任务关闭状态机

| 当前状态 | 操作 | 目标状态 | 是否允许 | 客户端守卫 |
| :--- | :--- | :--- | :---: | :--- |
| `ACTIVE` | 关闭为寻回 | `RESOLVED` | 是 | 允许提交，成功后刷新详情与时间线 |
| `ACTIVE` | 关闭为误报 | `FALSE_ALARM` | 是 | `reason` 长度 5-256，校验通过才可提交 |
| `RESOLVED` | 任意关闭 | - | 否 | 按钮置灰并提示“任务已终态” |
| `FALSE_ALARM` | 任意关闭 | - | 否 | 按钮置灰并提示“任务已终态” |

#### 10.5.2 主监护转移状态机（前端展示约束）

| 转移状态 | 可见动作（发起方） | 可见动作（目标受方） | 客户端约束 |
| :--- | :--- | :--- | :--- |
| `PENDING_CONFIRM` | 撤销 | 确认 / 拒绝 | 仅目标受方展示确认与拒绝按钮 |
| `CONFIRMED` | 无 | 无 | 展示完成态，不可重复操作 |
| `REJECTED` | 可重新发起 | 无 | 必须展示拒绝原因 |
| `CANCELLED` | 可重新发起 | 无 | 展示撤销人和撤销时间 |

#### 10.5.3 通知已读状态机

| 当前状态 | 用户动作 | 乐观更新 | 失败处理 |
| :--- | :--- | :---: | :--- |
| `UNREAD` | 标记已读 | 是 | 回滚为 `UNREAD` 并提示重试 |
| `READ` | 标记已读 | 否 | 保持 `READ`，不重复调用接口 |

## 11. 实时消息规范（WebSocket / SSE / 轮询）

### 11.1 WebSocket 建链流程

1. 调用 `POST /api/v1/ws/tickets` 获取短效一次性票据。
2. 用 `ws_endpoint` 建链：`/api/v1/ws/notifications?ticket={ws_ticket}`。
3. 票据默认 TTL 60 秒，过期后必须重新获取。

### 11.2 消息处理规则

1. 基于 `event_id` 去重。
2. 基于 `version` 防乱序，只接受更高版本。
3. 同版本冲突时按 `event_time` 取新。

### 11.3 断线重连

1. 网络抖动重连采用指数退避，最大间隔 30 秒。
2. 收到鉴权类关闭码（如 4401）不做无脑重连，先刷新会话。
3. 重连成功后立刻触发通知增量拉取补洞。

### 11.4 降级

1. WebSocket 不可用时，降级调用 `GET /api/v1/rescue/tasks/{task_id}/events/poll`。
2. 首屏消息中心始终支持 HTTP 拉取兜底。

### 11.5 事件一致性消费规范（对齐 Outbox 语义）

1. 事件应用顺序：先校验 `trace_id` 与 `event_id`，再做去重与版本比较。
2. 去重键建议：`event_id`；若服务端未给出，退化为 `type + aggregate_id + version`。
3. 版本守卫：仅当 `incoming.version > local.version` 才应用事件。
4. 空洞检测：若 `incoming.version > local.version + 1`，先触发快照拉取或增量补洞。
5. 恢复策略：App 从后台回前台时，必须先拉一次增量，后恢复实时流。
6. 事件应用失败时，禁止直接覆盖本地权威快照；应记录失败并触发兜底拉取。

建议实现伪代码：

```kotlin
fun handleEvent(localVersion: Long, event: TaskEvent): ConsumeAction {
    if (dedupStore.contains(event.eventId)) return ConsumeAction.Ignore
    if (event.version <= localVersion) return ConsumeAction.Ignore
    if (event.version > localVersion + 1) return ConsumeAction.FetchSnapshot
    return ConsumeAction.Apply
}
```

### 11.6 通知触达策略（无短信能力）

1. 紧急与高优事件必须双通道触达：应用推送 + 站内通知。
2. 推送不可达时，消息中心首屏必须可见未读站内消息。
3. 客户端不得显示“短信通知已发送”等误导文案。
4. 所有触达失败提示应给出可操作入口（查看消息中心或刷新）。

## 12. 本地存储、缓存与同步

### 12.1 存储分工

1. Room：通知、任务快照、草稿、事件游标。
2. DataStore：轻量设置、非敏感开关。
3. 安全存储：Token、敏感凭据。

### 12.2 缓存策略

1. 列表页采用 cache-aside，先读本地再并发拉远端。
2. 数据刷新必须带版本戳，避免旧数据覆盖新状态。
3. 用户登出或切换账号时必须清空账号域缓存。

### 12.3 离线写队列

1. 可重试写操作进入 WorkManager 队列（需业务允许）。
2. 每次重放保留原 `X-Request-Id`，确保服务端幂等。

## 13. 错误处理与用户反馈

### 13.1 错误分层

1. 网络错误：超时、断网、TLS。
2. 协议错误：反序列化失败、字段缺失。
3. 业务错误：`code != OK`。

### 13.2 错误码映射建议

| 错误码 | 客户端处理 |
| :--- | :--- |
| E_GOV_4011 | 会话失效，跳转登录 |
| E_GOV_4012 | ws_ticket 无效，重新申请 ticket |
| E_GOV_4291/E_GOV_4292 | 显示冷却倒计时，延迟重试 |
| E_REQ_4005 | 参数格式错误，记录埋点并阻断提交 |
| E_CLUE_4012 | 匿名凭据失效，提示重新扫码 |

## 14. UI 与交互规范（全量页面版）

### 14.1 设计与布局基线

1. 采用 8dp 间距系统，页面左右安全边距默认 16dp。
2. 列表页统一结构：顶部栏 + 筛选区 + 列表区 + 底部加载态。
3. 表单页统一结构：顶部栏 + 分组表单 + 主按钮 + 风险提示区。
4. 详情页统一结构：摘要卡 + 关键信息分区 + 时间线 + 操作区。
5. 地图页统一结构：地图层 + 浮层筛选 + 底部抽屉卡片。
6. 聊天页统一结构：会话头部 + 消息流 + 输入区 + 发送状态提示。
7. 最小点击区域 48dp，文本对比度不低于 WCAG AA。

### 14.2 导航结构与角色入口

1. 匿名域入口：扫码深链或短码手动录入后进入匿名线索上报链路。
2. 家庭域入口：登录成功后进入家庭工作台，底部导航包含任务、患者、通知、我的。
3. 管理与超管治理后台不在本手册范围。

### 14.3 全量页面范围

当前全量规范覆盖 Android 交付 29 个主页面（不含通用弹层、底部抽屉、系统对话框）。

| 功能域 | 页面数 |
| :--- | :---: |
| 匿名与鉴权 | 9 |
| 任务与线索 | 6 |
| 患者、监护与标签 | 6 |
| 物资运营 | 4 |
| AI 协同 | 4 |
| Android 交付合计 | 29 |

### 14.4 匿名与鉴权域页面（9 页）

| 页面 ID | 页面名称 | 关键接口 | 布局规范 | 交互规范 |
| :--- | :--- | :--- | :--- | :--- |
| PUB-01 | 扫码解析页 | GET /r/{resource_token} | 全屏状态页（解析中/成功跳转/失败） | 启动即请求；成功按路由结果映射到本地页面；失败显示重试与手动录入入口 |
| PUB-02 | 手动录入页 | POST /api/v1/public/clues/manual-entry | 顶部说明 + short_code/pin/captcha 表单 + 提交按钮 | 校验通过前禁用提交；429 按 Retry-After 倒计时；成功缓存匿名令牌并跳转上报页 |
| PUB-03 | 匿名线索上报页 | POST /api/v1/clues/report | 地图卡 + 定位信息 + 描述输入 + 照片区 + 提交按钮 | 提交时自动带 X-Anonymous-Token；成功跳回执页；E_CLUE_4012 触发重新扫码 |
| PUB-04 | 匿名紧急上报页 | POST /api/v1/clues/report | 红色风险提示条 + 一键定位 + 快速提交区 | 默认聚焦“立即上报”；失败保留表单；429 显示冷却提示 |
| PUB-05 | 匿名上报回执页 | POST /api/v1/clues/report（结果页） | 回执卡 + 时间 + 状态 + 继续上报入口 | 不展示患者隐私字段；支持复制 trace_id 反馈客服 |
| AUTH-01 | 登录页 | POST /api/v1/auth/login | Logo 区 + 用户名/密码 + 登录按钮 + 去注册入口 | 防重复提交；401 提示并保留输入；成功写会话并进入 HOME-01 |
| AUTH-02 | 注册页 | POST /api/v1/auth/register | 顶栏 + 注册表单 + 注册按钮 | 实时校验用户名与密码；E_AUTH_4091 就地提示“用户名已存在” |
| AUTH-03 | 修改密码页 | PUT /api/v1/users/me/password | 顶栏 + 旧/新/确认密码 + 提交按钮 | 新旧密码相同禁止提交；成功后强制重新登录 |
| NOTI-01 | 消息中心页 | GET/PUT /api/v1/notifications* | 顶栏 + 筛选条 + 通知列表 + 分页底部态 | 首屏先本地后远端；单条已读失败回滚；全部已读需二次确认 |

### 14.5 任务与线索域页面（6 页）

| 页面 ID | 页面名称 | 关键接口 | 布局规范 | 交互规范 |
| :--- | :--- | :--- | :--- | :--- |
| TASK-01 | 任务列表页 | GET /api/v1/rescue/tasks | 顶栏 + 状态筛选 + 任务卡片列表 | 支持下拉刷新和分页；切筛选重置游标；空态给“创建任务”入口 |
| TASK-02 | 任务详情页 | GET /api/v1/rescue/tasks/{task_id}, /snapshot, /events, /clues, /alerts | 摘要卡 + 状态时间线 + 线索列表 + 告警区 + 操作区 | 进入即并发拉取详情与快照；通过 version 防旧数据覆盖 |
| TASK-03 | 任务地图轨迹页 | GET /trajectory/latest, /events/poll, WS 推送 | 全屏地图 + 时间滑条 + 底部轨迹抽屉 | WS 实时更新轨迹；断线降级到 poll；地图点位按事件版本更新 |
| TASK-04 | 新建任务页 | POST /api/v1/rescue/tasks | 患者选择卡 + 来源与备注表单 + 提交按钮 | 本地校验后提交；E_TASK_4091 提示已有 ACTIVE 任务并跳详情 |
| TASK-05 | 任务关闭页 | POST /api/v1/rescue/tasks/{task_id}/close | 关闭类型选择 + 原因输入 + 风险提示 + 确认按钮 | FALSE_ALARM 需强校验 reason；提交中禁用返回与重复点击 |
| CLUE-01 | 线索详情与时间线页 | GET /api/v1/clues/{clue_id}, /timeline, /rescue/tasks/{task_id}/clues/latest | 线索信息卡 + 地图位置 + 处理时间线 + 关联任务入口 | 线索状态变化即时刷新；支持按 trace_id 查看关联操作 |

### 14.6 患者、监护与标签域页面（6 页）

| 页面 ID | 页面名称 | 关键接口 | 布局规范 | 交互规范 |
| :--- | :--- | :--- | :--- | :--- |
| HOME-01 | 家庭工作台 | GET /api/v1/patients/{patient_id}/profile, /notifications | 顶部患者切换 + 风险卡片 + 快捷入口宫格 + 消息摘要 | 首屏聚合任务、围栏、通知；点击卡片跳对应详情 |
| PAT-01 | 患者档案详情页 | GET /api/v1/patients/{patient_id}, /profile | 基础信息卡 + 健康信息 + 紧急联系人 + 围栏摘要 | 支持下拉刷新；无权限时显示申请授权引导 |
| PAT-02 | 患者档案编辑页 | POST /api/v1/patients, PUT /api/v1/patients/{patient_id} | 分组表单（基础/健康/紧急联系）+ 保存按钮 | 字段级校验；保存成功返回详情并刷新本地缓存 |
| PAT-03 | 围栏设置页 | GET/PUT /api/v1/patients/{patient_id}/fence | 地图选区 + 半径滑杆 + 地址检索 + 保存按钮 | 修改后先预览告警范围；E_PRO_4221 时提示围栏参数不合法 |
| GUA-01 | 监护管理页 | GET /guardians, /invitations, /transfers, DELETE /guardians/{user_id} | 成员列表 Tab + 邀请记录 Tab + 转移记录 Tab | 支持发起邀请、移除成员、查看转移状态；高风险操作二次确认 |
| TAG-01 | 标签管理页 | GET /patients/{patient_id}/tags, /tags/{tag_code}, /history, POST /tags/bind, /lost | 标签状态卡 + 历史时间线 + 操作按钮区 | BIND/LOST 按状态控制可见操作；状态变化后刷新详情与历史 |

### 14.7 物资运营域页面（4 页）

| 页面 ID | 页面名称 | 关键接口 | 布局规范 | 交互规范 |
| :--- | :--- | :--- | :--- | :--- |
| ORD-01 | 工单列表页 | GET /api/v1/material/orders | 顶栏 + 状态筛选 + 工单卡片列表 | 支持按状态与时间筛选；列表项展示物流与标签状态 |
| ORD-02 | 新建申领页 | POST /api/v1/material/orders | 地址信息 + 标签类型 + 数量 + 备注 + 提交按钮 | 提交后跳工单详情；失败保留输入可重试 |
| ORD-03 | 工单详情页 | GET /material/orders/{order_id}, POST /cancel, POST /confirm | 摘要卡 + 状态时间线 + 物流区 + 操作区 | FAMILY 仅可执行取消与签收；按钮可见性按状态机控制 |
| ORD-04 | 物流轨迹页 | GET /tracking, /resource-link | 物流节点列表 + 地图路线 + 异常提示条 | 节点按时间倒序；异常状态高亮并给联系客服入口 |

### 14.8 AI 协同域页面（4 页）

| 页面 ID | 页面名称 | 关键接口 | 布局规范 | 交互规范 |
| :--- | :--- | :--- | :--- | :--- |
| AI-01 | AI 会话列表页 | POST /api/v1/ai/sessions, GET /api/v1/ai/sessions | 顶栏 + 搜索 + 会话列表 + 新建按钮 | 新建会话成功后自动进入对话页；列表支持分页加载 |
| AI-02 | AI 对话页 | POST /api/v1/ai/sessions/{session_id}/messages, GET /messages | 消息流 + 输入框 + 发送按钮 + 降级提示条 | 流式消息分段渲染；E_AI_4292/4293 显示退避或降级提示 |
| AI-03 | 会话详情与配额页 | GET /api/v1/ai/sessions/{session_id}, /quota, POST /archive | 会话元信息卡 + 配额进度条 + 归档操作区 | 配额低于阈值高亮；归档需二次确认 |
| AI-04 | 记忆笔记页 | POST/GET /api/v1/patients/{patient_id}/memory-notes | 笔记列表 + 新增输入区 + 标签过滤 | 新增后乐观插入列表；失败回滚并提示重试 |

### 14.9 端边界声明（Android-only）

1. 本手册仅覆盖 Android 客户端页面与交互，不包含后台管理端页面。
2. 管理与超管治理能力由独立后台文档维护，Android 仅消费其结果回流状态。
3. 匿名线索上报仅描述 Android 原生页面，不包含 H5 页面交互约束。

### 14.10 通用交互与状态机规则

1. 所有页面必须具备 Loading、Empty、Error、Content 四态。
2. 所有写操作按钮必须具备 loading 态与防重入机制。
3. 429 错误必须读取 Retry-After 并展示倒计时，倒计时结束后才允许重试。
4. 401 错误统一执行会话失效流程：清理凭据、断开 WS、跳登录。
5. 分页页必须保存 `next_cursor`，切筛选时清空旧游标。
6. 乐观更新仅用于低风险状态（如通知已读）；失败必须回滚。
7. 地图相关页面必须标注当前坐标系，避免 GCJ-02/WGS84 混用。
8. WS 事件更新必须执行 event_id 去重与 version 防乱序。

### 14.11 跨页面关键流程

1. 匿名上报流程：PUB-01 -> PUB-02 或 PUB-03/PUB-04 -> PUB-05。
2. 登录流程：AUTH-01 -> HOME-01 -> NOTI-01。
3. 任务闭环流程：TASK-04 -> TASK-02 -> TASK-03 -> TASK-05。
4. 监护管理流程：PAT-01 -> GUA-01 -> 邀请或转移 -> GUA-01 刷新。
5. 物资流程：ORD-02 -> ORD-03 -> ORD-04 -> ORD-03（签收/取消）。
6. AI 流程：AI-01 -> AI-02 -> AI-03 -> AI-01。

### 14.12 页面开发完成定义（UI DoD）

1. 页面布局符合本章定义，关键区块齐全。
2. 关键交互（提交、刷新、分页、回退）均有自动化或脚本化测试。
3. 错误码映射与反馈文案完整，不出现“未知错误”兜底泛提示。
4. 页面埋点齐全：进入、成功、失败、耗时、重试、退出。
5. 与后端联调通过，关键接口字段与状态机行为一致。

### 14.13 配色方案与主题令牌

视觉方向：可信医疗蓝 + 安全绿色 + 风险橙红，不使用高饱和紫色作为主色。

#### 14.13.1 基础色板（Light / Dark）

| 语义令牌 | Light | Dark | 用途 |
| :--- | :--- | :--- | :--- |
| primary | #0B6B8A | #67C7E6 | 品牌主按钮、选中态、关键链接 |
| onPrimary | #FFFFFF | #0D1B22 | 主色前景文字 |
| secondary | #2E7D32 | #7BD67F | 成功状态、完成态芯片 |
| onSecondary | #FFFFFF | #0E2210 | 次色前景文字 |
| tertiary | #B26A00 | #F2B84B | 提醒、待处理、弱警示 |
| onTertiary | #FFFFFF | #2A1B00 | 提醒色前景文字 |
| error | #B42318 | #FF8A80 | 错误、失败、阻断性操作 |
| onError | #FFFFFF | #2A0E0C | 错误色前景文字 |
| background | #F4F7F9 | #0F1418 | 页面背景 |
| onBackground | #18242D | #E7EEF3 | 页面主文字 |
| surface | #FFFFFF | #162028 | 卡片、弹窗、面板背景 |
| onSurface | #1A2A34 | #DCE7EE | 卡片主文字 |
| outline | #8CA1AE | #4D6472 | 分割线、描边 |
| surfaceVariant | #E8EEF2 | #22313C | 次级容器背景（筛选条、标签区） |

#### 14.13.2 状态色与业务语义映射

| 业务状态 | 背景色 | 前景色 | 使用位置 |
| :--- | :--- | :--- | :--- |
| ACTIVE | #E6F3FA | #0B6B8A | 任务进行中状态标签 |
| CLOSED_RESOLVED | #E8F5E9 | #2E7D32 | 任务闭环成功标签 |
| CLOSED_FALSE_ALARM | #FFF4E5 | #B26A00 | 误报闭环标签 |
| UNREAD | #E9F2FF | #1E4F9A | 通知未读标识 |
| READ | #EDF1F4 | #647886 | 通知已读标识 |
| RISK_CRITICAL | #FDECEC | #B42318 | 红色高危告警条 |
| RISK_WARN | #FFF4E5 | #B26A00 | 橙色预警条 |

#### 14.13.3 组件级配色规则

1. 主按钮使用 primary 背景 + onPrimary 文字。
2. 次按钮使用 surface 背景 + primary 描边 + primary 文字。
3. 危险按钮使用 error 背景，禁止与主按钮同屏并列同权重。
4. 页面背景固定 background，内容容器固定 surface，避免同层多底色造成噪音。
5. 列表分隔线使用 outline，弱化信息密度高页面的视觉负担。

#### 14.13.4 深色模式与可访问性

1. 深色模式必须使用独立 Dark 令牌，不做 Light 反色。
2. 正文文字与背景对比度 >= 4.5:1，标题与关键数字建议 >= 7:1。
3. 状态传达不得只依赖颜色，必须同时提供图标或文案（例如“高危”“已读”）。
4. 红绿组合场景需提供形状差异（实心点/描边点）以兼容色觉缺陷用户。

#### 14.13.5 动效与颜色联动规则

1. 状态变化动画时长 150ms-250ms，避免颜色闪烁超过 2 次。
2. 刷新与加载动效使用 primary 或 outline，不使用 error 颜色作为加载态。
3. 告警出现时采用背景淡入，不使用高频闪烁。

### 14.14 第三方 UI 组件成体系引入规范

#### 14.14.1 总体原则

1. 优先官方能力（Compose + Material 3），第三方仅用于官方缺失或成本显著更优场景。
2. 引入目标是“能力复用”，不是“UI 风格替换”，不得破坏本手册第 14.13 的配色体系。
3. 所有第三方 UI 组件必须经过白名单评审后接入，禁止 feature 模块私自新增依赖。
4. 必须可替换：任何第三方库都需通过适配接口暴露，禁止业务代码直接绑定实现细节。

#### 14.14.2 推荐白名单（V1）

| 能力域 | 推荐库 | 接入模块 | 使用边界 |
| :--- | :--- | :--- | :--- |
| 图片加载 | `io.coil-kt:coil-compose` | core-ui-adapter | 仅用于图片渲染与缓存，不处理业务鉴权 |
| 动画渲染 | `com.airbnb.android:lottie-compose` | core-ui-adapter | 仅用于状态动画与引导动画 |
| 图表可视化 | `com.patrykandpatrick.vico:compose-m3` | core-ui-adapter | 仅用于任务与通知趋势图等客户端统计展示 |
| 地图能力 | 高德 Android SDK（Map + Location） | core-ui-adapter | 仅用于地图展示与定位，不承载业务状态判断 |

说明：
1. 若新增库不在白名单，必须走 14.14.4 准入流程。
2. 白名单每季度复审一次，淘汰停更或高风险库。

#### 14.14.3 分层接入模型

依赖方向：
1. feature-* -> core-ui（稳定组件接口）
2. core-ui -> core-ui-adapter（可替换适配接口）
3. core-ui-adapter -> 第三方 UI 库（唯一实际依赖点）

强约束：
1. feature 禁止直接 import 第三方库命名空间。
2. `core-ui-adapter` 必须提供统一接口与默认降级实现。
3. 所有第三方组件外观需映射到 14.13 主题令牌。

#### 14.14.4 准入流程（RFC）

1. 需求提出：说明场景、官方能力缺口、预期收益。
2. 技术评审：评估维护活跃度、License、APK 体积、方法数、可访问性支持。
3. POC 验证：在 `core-ui-adapter` 完成最小可运行封装与主题映射。
4. 质量验证：通过 UI 测试、性能测试、崩溃与降级测试。
5. 发布准入：灰度启用并挂特性开关，观察崩溃率与渲染耗时。

准入阈值（建议）：
1. 冷启动增量 <= 80ms（中端机）。
2. APK 增量 <= 1.5MB（单库）。
3. 关键页面帧率下降不超过 5%。

#### 14.14.5 版本治理与升级策略

1. 所有第三方版本统一由 Version Catalog 管理，禁止子模块硬编码版本。
2. 升级节奏为“月度小升级 + 季度审计”，高危 CVE 必须紧急升级。
3. 每次升级至少验证：启动、列表滚动、主题切换、无障碍朗读、弱网重试。

示例（`libs.versions.toml`）：

```toml
[versions]
coil = "2.7.0"
lottie = "6.4.0"
vico = "1.15.0"

[libraries]
coil-compose = { module = "io.coil-kt:coil-compose", version.ref = "coil" }
lottie-compose = { module = "com.airbnb.android:lottie-compose", version.ref = "lottie" }
vico-compose = { module = "com.patrykandpatrick.vico:compose-m3", version.ref = "vico" }
```

#### 14.14.6 适配层模板（示意）

```kotlin
interface AppImage {
    @Composable
    fun Remote(url: String, contentDescription: String?)
}

class CoilAppImage : AppImage {
    @Composable
    override fun Remote(url: String, contentDescription: String?) {
        AsyncImage(model = url, contentDescription = contentDescription)
    }
}
```

规则：
1. 业务页面只依赖 `AppImage`、`AppChart`、`AppAnimation` 等抽象接口。
2. 第三方替换时仅改 `core-ui-adapter` 实现，不改 feature 业务页面。

#### 14.14.7 禁止事项

1. 禁止在 feature 模块直接引入第三方 UI 库。
2. 禁止直接使用第三方默认主题色覆盖全局主题。
3. 禁止无降级方案地引入关键页面核心组件。
4. 禁止未做 License 审核就上线。

#### 14.14.8 第三方 UI 引入完成定义（DoD）

1. 已通过 14.14.4 准入流程并留存评审记录。
2. 已完成 `core-ui-adapter` 封装与主题令牌映射。
3. 已提供降级开关与回退预案。
4. 已补齐 UI 自动化测试与性能基线对比结果。
5. 已更新手册白名单与版本记录。

### 14.15 字段级交互约束表（Android 交付 29 页）

适用约束：
1. 表内“典型错误码”必须与 `API_from_SRS_SADD_LLD.md` 对齐。
2. 所有写请求默认附带 `X-Request-Id` 与 `X-Trace-Id`；鉴权接口默认附带 `Authorization`（匿名接口除外）。
3. 所有 ID 入参与路径参数统一按“十进制字符串”处理，不允许前端用浮点数中转。
4. 空态/异常态必须映射到 Loading、Empty、Error、Content 四态，不允许仅用 Toast 兜底。
5. 本章不包含后台管理端（`ADM-*`）页面字段约束。

#### 14.15.1 匿名与鉴权域（9 页）

#### PUB-01 扫码解析页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `resource_token`（Path） | 是 | 长度 16-256；仅允许 URL 安全字符 | `E_CLUE_4042`、`E_CLUE_4041`、`E_MAT_4002` | 解析中显示骨架屏；无效 token 展示重试与手动录入入口 |
| `X-Trace-Id` | 是 | 长度 16-64，空值直接阻断 | `E_REQ_4002` | Header 异常进入 Error 态并提示“请更新客户端后重试” |

#### PUB-02 手动录入页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `short_code` | 是 | 固定 6 位数字 | `E_CLUE_4005` | 未输入显示表单空态；格式错误显示字段级报错 |
| `pin_code`、`captcha_token`、`device_fingerprint` | 是 | `pin_code` 4-8 位；`captcha_token` 非空；设备指纹格式合法 | `E_CLUE_4006`、`E_GOV_4038`、`E_GOV_4004`、`E_GOV_4291`、`E_GOV_4292` | 429 时锁定提交按钮并显示倒计时；失败保留输入 |

#### PUB-03 匿名线索上报页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `latitude`、`longitude`、`coord_system` | 是 | 经纬度范围合法；坐标系固定 `GCJ-02` 上报 | `E_CLUE_4001`、`E_CLUE_4002`、`E_CLUE_4007` | 定位失败显示“手动定位”入口并允许重试 |
| `description`、`photo_urls`、`X-Anonymous-Token` | 是 | 描述长度 1-500；图片 URL 白名单；匿名令牌必带 | `E_CLUE_4003`、`E_CLUE_4004`、`E_CLUE_4012`、`E_CLUE_4041` | 匿名令牌失效清理本地令牌并强制回扫码入口 |

#### PUB-04 匿名紧急上报页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `short_code`（路由上下文） | 是 | 6 位数字；与入口链路一致 | `E_CLUE_4005`、`E_CLUE_4042` | 无效短码直接拦截，不渲染紧急按钮 |
| `location`、`description`、`X-Anonymous-Token` | 是 | 至少包含有效定位；描述与图片规则同 PUB-03 | `E_CLUE_4012`、`E_CLUE_4001`、`E_CLUE_4003`、`E_GOV_4291` | 高压失败展示红色错误条与冷却倒计时 |

#### PUB-05 匿名上报回执页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `trace_id`（回执） | 是 | 非空；支持复制 | 沿用上报接口错误码：`E_CLUE_4012`、`E_CLUE_4001`、`E_CLUE_4041` | 回执缺失时展示“提交结果待确认”并提供返回入口 |
| `reported_at`、`receipt_status` | 是 | 时间格式可解析；状态枚举受控 | 同上 | 回执加载失败不回显隐私，仅给通用受理提示 |

#### AUTH-01 登录页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `username` | 是 | 4-32 字符；去首尾空格 | `E_AUTH_4001` | 空输入禁用登录按钮 |
| `password` | 是 | 8-64 字符；禁止纯空白 | `E_AUTH_4002`、`E_GOV_4011`、`E_GOV_4031` | 失败保留用户名；账号封禁显示不可恢复提示 |

#### AUTH-02 注册页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `username`、`password` | 是 | 与登录页一致 | `E_AUTH_4001`、`E_AUTH_4002` | 实时字段校验，错误就地提示 |
| `confirm_password` | 是 | 与 `password` 完全一致 | `E_AUTH_4091`、`E_REQ_4001` | 冲突时提示“用户名已存在”，不清空已填内容 |

#### AUTH-03 修改密码页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `old_password`、`new_password` | 是 | 新旧密码不得相同；新密码满足强度 | `E_USR_4001`、`E_USR_4002` | 不满足强度时给规则提示，不发请求 |
| `confirm_password` | 是 | 与 `new_password` 一致 | `E_USR_4011`、`E_GOV_4011`、`E_REQ_4001` | 修改成功强制退出并跳转登录页 |

#### NOTI-01 消息中心页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `cursor`、`page_size`、`type/status` 过滤 | 否 | `page_size` 建议 1-50；过滤值限定枚举 | `E_NOTI_4001`、`E_REQ_4005` | 无数据展示空态插画与返回工作台入口 |
| `notification_id`（标记已读） | 是 | 十进制字符串；重复点击去抖 | `E_NOTI_4030`、`E_NOTI_4041`、`E_GOV_4011`、`E_GOV_4291` | 乐观更新失败时回滚读状态并提示重试 |

#### 14.15.2 任务与线索域（6 页）

#### TASK-01 任务列表页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `status`（筛选） | 否 | 枚举：`ACTIVE/RESOLVED/FALSE_ALARM` | `E_REQ_4005` | 空列表展示“创建任务”主入口 |
| `cursor`、`page_size` | 否 | `page_size` 合法区间；切筛选重置游标 | `E_GOV_4011`、`E_GOV_4030` | 分页异常保留已加载列表并显示底部重试 |

#### TASK-02 任务详情页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `task_id`（Path） | 是 | 十进制字符串 | `E_TASK_4041`、`E_REQ_4005` | 不存在任务展示“返回任务列表” |
| `after_version`/`cursor`（事件、线索增量） | 否 | 非负整数；丢失时退化拉全量 | `E_GOV_4011`、`E_PRO_4030` | 增量失败时不清空详情，提示手动刷新 |

#### TASK-03 任务地图轨迹页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `task_id`、`after_version`（轮询） | 是/否 | `task_id` 必填；`after_version` >= 0 | `E_TASK_4041`、`E_REQ_4005` | 无轨迹点展示地图空态与最近更新时间 |
| `ws_ticket`（建链） | 是 | 单次使用；TTL 内消费 | `E_GOV_4012`、`E_GOV_4011` | 票据失效自动重取；连续失败降级轮询 |

#### TASK-04 新建任务页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `patient_id` | 是 | 十进制字符串；必须可见患者 | `E_TASK_4030`、`E_REQ_4001` | 未选患者时禁用提交 |
| `source`、`remark` | 是/否 | `source` 枚举合法；`remark` <= 500 | `E_TASK_4001`、`E_TASK_4002`、`E_TASK_4091` | 冲突时跳转已有 `ACTIVE` 任务详情 |

#### TASK-05 任务关闭页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `close_type` | 是 | 枚举：`RESOLVED/FALSE_ALARM` | `E_TASK_4004` | 非法值直接阻断提交 |
| `reason` | 条件必填 | 当 `FALSE_ALARM` 时长度 5-256 | `E_TASK_4005`、`E_TASK_4222`、`E_TASK_4093`、`E_TASK_4041` | 终态任务隐藏提交按钮并展示只读结果 |

#### CLUE-01 线索详情与时间线页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `clue_id`（Path） | 是 | 十进制字符串 | `E_CLUE_4043`、`E_REQ_4005` | 线索不存在显示“回任务详情”入口 |
| `cursor`（时间线）、`task_id`（关联） | 否 | 游标透传；任务 ID 合法 | `E_TASK_4041`、`E_GOV_4030` | 时间线空态显示“暂无处理记录” |

#### 14.15.3 患者、监护与标签域（6 页）

#### HOME-01 家庭工作台

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `patient_id`（当前上下文） | 是 | 十进制字符串；切换后清理旧页面缓存 | `E_PRO_4041`、`E_REQ_4005` | 无患者时展示“新建患者档案”空态 |
| 聚合查询时间窗参数 | 否 | 枚举窗口（今日/7天/30天） | `E_GOV_4011`、`E_PRO_4030` | 拉取失败保留上次快照并显示重试 |

#### PAT-01 患者档案详情页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `patient_id` | 是 | 十进制字符串 | `E_PRO_4041`、`E_REQ_4005` | 不存在时显示不可见或已删除提示 |
| `include_profile`（扩展信息） | 否 | 布尔值 | `E_GOV_4011`、`E_PRO_4030` | 权限不足展示申请授权入口 |

#### PAT-02 患者档案编辑页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `patient_name`、`birthday`、`gender` | 是 | 非空；生日可解析；性别枚举合法 | `E_PRO_4001`、`E_PRO_4002`、`E_PRO_4003` | 字段非法就地标红，不触发提交 |
| `chronic_diseases`、`avatar_url` | 是 | 文本格式合法；头像 URL 必填且可访问 | `E_PRO_4004`、`E_PRO_4014`、`E_PRO_4091`、`E_PRO_4030` | 保存失败保留草稿并支持重试 |

#### PAT-03 围栏设置页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `enabled`、`center_lat`、`center_lng`、`radius_m` | 条件必填 | 开启围栏时中心点与半径必填；半径范围受限 | `E_PRO_4005`、`E_PRO_4221` | 参数不全直接提示“开启围栏需完整中心与半径” |
| `address_label`、`notify_switch` | 否 | 长度与布尔值合法 | `E_PRO_4030`、`E_PRO_4041`、`E_REQ_4001`、`E_REQ_4005` | 保存失败回退至上次已保存配置 |

#### GUA-01 监护管理页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| 邀请参数：`invitee_user_id`、`relation_role`、`reason` | 是 | 目标用户与关系枚举合法；原因长度合法 | `E_PRO_4006`、`E_PRO_4007`、`E_PRO_4042`、`E_PRO_4094` | 无成员时展示邀请引导空态 |
| 转移/确认参数：`target_user_id`、`action`、`reject_reason/cancel_reason` | 条件必填 | 仅目标受方可确认；拒绝/撤销需原因 | `E_PRO_4032`、`E_PRO_4033`、`E_PRO_4034`、`E_PRO_4044`、`E_PRO_4045`、`E_PRO_4095`、`E_PRO_4097`、`E_PRO_4098`、`E_PRO_4099`、`E_PRO_4011`、`E_PRO_4012`、`E_PRO_4013` | 非授权操作者隐藏按钮并展示只读状态 |

#### TAG-01 标签管理页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `tag_code`、`patient_id` | 是 | 标签编码非空；患者 ID 合法 | `E_MAT_4044`、`E_PRO_4041`、`E_REQ_4005` | 无标签时展示“去申领/绑定”空态 |
| 操作参数：`resource_token`、`lost_reason`、`void_reason` | 条件必填 | `LOST/VOID` 动作需原因；资源令牌格式合法 | `E_MAT_4002`、`E_MAT_4004`、`E_MAT_4005`、`E_MAT_4098`、`E_MAT_4223`、`E_PRO_4092`、`E_PRO_4093`、`E_PRO_4031` | 状态冲突时刷新标签详情并锁定非法动作 |

#### 14.15.4 物资运营域（4 页）

#### ORD-01 工单列表页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `status`（筛选） | 否 | 枚举值合法 | `E_REQ_4005` | 空列表显示“新建申领”按钮 |
| `cursor/page_size` 或 `page_no/page_size` | 否 | 分页参数范围合法 | `E_GOV_4011`、`E_MAT_4030`、`E_MAT_4041` | 查询失败保留旧数据并提供重试 |

#### ORD-02 新建申领页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `patient_id`、`quantity` | 是 | 患者 ID 合法；数量 > 0 且不超上限 | `E_MAT_4001`、`E_PRO_4041`、`E_PRO_4030` | 提交失败保留表单数据 |
| `resource_token`、地址字段 | 条件必填 | 资源令牌格式合法；地址必填项非空 | `E_MAT_4002`、`E_REQ_4001` | 地址缺失时锁定提交并聚焦错误项 |

#### ORD-03 工单详情页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `order_id`（Path） | 是 | 十进制字符串 | `E_MAT_4041`、`E_REQ_4005` | 工单不存在时展示返回列表入口 |
| 操作参数：`cancel_reason`、`confirm_action` | 条件必填 | 取消原因长度合法；仅允许状态机可执行动作 | `E_MAT_4003`、`E_MAT_4091`、`E_MAT_4094`、`E_MAT_4030`、`E_REQ_4001` | 状态冲突后强制刷新详情并重绘按钮 |

#### ORD-04 物流轨迹页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `order_id` | 是 | 十进制字符串 | `E_MAT_4041`、`E_REQ_4005` | 无轨迹时展示“物流暂未同步”空态 |
| `tracking_cursor`、`resource_link` | 否 | 游标透传；资源链接签名合法 | `E_MAT_4223`、`E_MAT_5003`、`E_GOV_4030` | 链接失效时提示刷新并回到工单详情 |

#### 14.15.5 AI 协同域（4 页）

#### AI-01 AI 会话列表页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| 新建会话参数：`patient_id`、`task_id`（可选） | 是/否 | `patient_id` 必填；`task_id` 合法且可访问 | `E_AI_4002`、`E_PRO_4030`、`E_AI_4292`、`E_AI_4293` | 会话创建失败保留输入并给冷却提示 |
| 列表参数：`cursor`、`page_size` | 否 | 分页参数合法 | `E_GOV_4011`、`E_REQ_4005` | 无会话时展示“发起第一次提问”空态 |

#### AI-02 AI 对话页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `session_id`、`prompt` | 是 | `prompt` 非空且长度受限；`session_id` 合法 | `E_AI_4001`、`E_AI_4041`、`E_AI_4033` | 会话不存在时引导返回会话列表 |
| `stream`、`X-Request-Id` | 是 | 流式标志固定；请求幂等键合法 | `E_AI_4091`、`E_AI_4292`、`E_AI_4293`、`E_REQ_4001` | 429 按 `Retry-After` 显示倒计时并允许重试 |

#### AI-03 会话详情与配额页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `session_id`（详情/配额） | 是 | 十进制字符串 | `E_AI_4041`、`E_REQ_4005` | 无详情时展示“会话已归档或删除” |
| `archive` 操作参数 | 否 | 二次确认后提交；避免重复点击 | `E_AI_4091`、`E_AI_4293`、`E_GOV_4011` | 归档失败保留当前页面并提示重试 |

#### AI-04 记忆笔记页

| 字段 | 必填 | 入参校验 | 典型错误码 | 空态/异常态 |
| :--- | :---: | :--- | :--- | :--- |
| `patient_id` | 是 | 十进制字符串 | `E_PRO_4041`、`E_PRO_4030`、`E_REQ_4005` | 无笔记显示“添加首条记忆笔记”空态 |
| `note_content`、`tags` | 是/否 | 内容非空且长度受限；标签数量受限 | `E_AI_4003`、`E_REQ_4001` | 写入失败回滚乐观插入并提示重试 |

#### 14.15.6 端边界补充（剥离 Web/H5 页面）

1. 本章仅覆盖 Android 交付 29 页字段约束。
2. 管理与超管治理页面（`ADM-*`）字段约束由独立后台文档维护。
3. 匿名线索上报字段仅描述 Android 原生页面，不包含 H5 页面。

### 14.16 字段级约束完成定义（DoD）

1. Android 交付 29 页均有“字段、校验、错误码、空态/异常态”四要素。
2. Android 交付范围 29 页必须全部实现并通过测试。
3. 任一页面新增写交互时，必须同步补齐本章对应表项。
4. 页面实现、测试用例、埋点事件中的字段命名必须与本章一致。
5. 联调发现错误码偏差时，先修正实现，再更新本章并记录变更。

## 15. 安全与隐私规范

1. 禁止在日志打印 Token、手机号、姓名、精确定位。
2. Debug 包可启用网络日志，Release 包必须关闭敏感日志。
3. 剪贴板、截图、分享链路不得暴露隐私字段。
4. 本地数据库可按能力启用加密（如 SQLCipher）。
5. 崩溃上报需脱敏后上传。

## 16. AI 交互与流式输出规范

1. 支持普通请求与流式请求双通道。
2. 流式输出按片段累积，UI 层做节流刷新，避免高频重组造成卡顿。
3. 任何 AI 输出都不是“系统真相”，高风险建议必须附免责声明。
4. 发生降级时，在会话中保留“降级原因 + 可重试入口”。

### 16.1 AI 可实现指标（移动端）

| 指标 | 目标值 | 采集口径 |
| :--- | :--- | :--- |
| 首 token 时延（P95） | <= 4s | 用户点击发送到首片段渲染 |
| 完整响应时延（P95） | <= 12s | 用户点击发送到最终完成 |
| 会话成功率 | >= 99% | `code=OK` 且可渲染完成 |
| 降级可用率 | >= 99.5% | 主路径失败后仍返回可解释结果 |
| 429 恢复成功率 | >= 95% | 按 `Retry-After` 重试后成功 |

### 16.2 降级模式前端映射

| `fallback_response.mode` | 前端文案 | 交互动作 |
| :--- | :--- | :--- |
| `RULE_BASED` | 已切换规则引擎结果 | 显示“重试”按钮 |
| `SAFE_GUARD` | 当前仅返回安全建议 | 显示免责声明，不给高风险快捷操作 |
| `TOOL_DEGRADED` | 工具能力受限，结果可能不完整 | 提示稍后重试并保留上下文 |
| 未知值 | 已切换降级结果 | 统一降级文案 + 上报埋点 |

## 17. 测试与质量门禁

### 17.1 测试分层

1. Unit Test：UseCase、Mapper、Reducer。
2. Integration Test：Repository + MockWebServer。
3. UI Test：关键流程（登录、提报线索、通知已读、任务详情刷新）。
4. Contract Test：对关键接口 schema 与错误码进行回归。

### 17.2 覆盖目标

1. core 与 domain 层行覆盖率 >= 80%。
2. feature 核心 UseCase 覆盖率 >= 85%。
3. 至少包含 1 条 WebSocket 断线重连自动化用例。

### 17.3 必测清单

1. ID 全字符串传输回归。
2. 写请求 `X-Request-Id` 注入回归。
3. 429 + `Retry-After` 退避回归。
4. 匿名凭据失效链路回归。
5. 通知乱序事件去重与防覆盖回归。

### 17.4 需求验收用例矩阵（P0）

| 验收编号 | 用例说明 | 前置条件 | 预期结果 |
| :--- | :--- | :--- | :--- |
| `ACC-FE-001` | 扫码路由到普通或紧急页 | 准备 `BOUND` 与 `LOST` 标签二维码 | `BOUND->PUB-03`、`LOST->PUB-04` |
| `ACC-FE-002` | 手动录入限流冷却 | 连续触发匿名录入直到 429 | 按 `Retry-After` 倒计时解禁提交 |
| `ACC-FE-003` | 存疑线索闭环可见性 | 存在一条 `SUSPECTED` 线索并完成复核 | 页面状态从存疑进入终态且可追溯 |
| `ACC-FE-004` | 重复创建任务冲突处理 | 患者已有 `ACTIVE` 任务 | 返回冲突后跳转或引导到已存在任务 |
| `ACC-FE-005` | 误报关闭原因必填 | 任务为 `ACTIVE` | 空原因不可提交；合法原因可关闭为 `FALSE_ALARM` |
| `ACC-FE-006` | 主监护转移按钮权限 | 存在 `PENDING_CONFIRM` 转移请求 | 仅目标受方可见确认/拒绝按钮 |
| `ACC-FE-007` | 围栏参数非法提示 | 围栏开启但缺失中心点或半径 | 精准提示 `E_PRO_4221` 文案 |
| `ACC-FE-008` | 通知双通道兜底 | 模拟推送不可达 | 站内消息中心仍能看到未读通知 |
| `ACC-FE-009` | 头部注入一致性 | 执行任意写接口 | 请求携带 `X-Request-Id` + `X-Trace-Id` |
| `ACC-FE-010` | WS 乱序与补洞 | 注入乱序/跨版本事件 | 旧版本事件被忽略，空洞触发补洞拉取 |

## 18. 可观测性与线上诊断

1. 每个网络请求日志携带 `trace_id` 与接口路径。
2. WebSocket 生命周期事件（connect/open/close/retry）必须打点。
3. 关键业务埋点：登录成功率、线索提交成功率、通知到达时延、AI 响应时延。
4. 线上问题排查顺序：trace_id -> 请求日志 -> 业务码 -> 客户端状态机快照。

## 19. CI/CD 与发布规范

### 19.1 CI 必过项

1. `./gradlew clean test`
2. `./gradlew detekt ktlintCheck`
3. `./gradlew assembleDebug`
4. 关键模块 instrumentation test（可按夜间任务执行）

### 19.2 发布策略

1. 内测 -> 灰度 -> 全量三阶段。
2. 灰度监控 24 小时，关注崩溃率、ANR、接口失败率。
3. 发现高危问题可通过远程开关关闭 AI 或实时推送入口。

### 19.3 回滚策略

1. 版本回滚与配置回滚分离。
2. 不可逆数据迁移必须提供降级兼容读取。

## 20. 新人开发落地流程（7 天）

1. Day 1：拉代码、跑通环境、阅读本手册与 API 文档全局规范。
2. Day 2：完成一个只读列表页面（含分页、错误态）。
3. Day 3：完成一个写接口流程（含 `X-Request-Id` 与重试）。
4. Day 4：接入通知中心（HTTP 拉取 + 本地缓存）。
5. Day 5：接入 WebSocket ticket 建链与断线重连。
6. Day 6：补齐单测与集成测试。
7. Day 7：通过 CR、联调、灰度发布演练。

## 21. 开发完成定义（Definition of Done）

一个需求“完成”必须同时满足：

1. 代码完成并通过 Code Review。
2. 与后端联调通过，业务码处理完整。
3. 单元测试、集成测试、关键 UI 流程测试通过。
4. 日志与埋点已补齐，可定位线上问题。
5. 文档更新：接口变更、状态机变更、错误码变更同步到手册。
6. 灰度观测项已配置，具备回滚方案。

---

附录 A：Repository 模板

```kotlin
class NotificationRepositoryImpl(
    private val api: NotificationApi,
    private val dao: NotificationDao,
    private val io: CoroutineDispatcher
) : NotificationRepository {

    override fun streamNotifications(): Flow<List<Notification>> {
        return dao.observeAll().map { entities -> entities.map { it.toDomain() } }
    }

    override suspend fun refresh(cursor: String?): Result<Unit> = withContext(io) {
        runCatching {
            val resp = api.getNotifications(cursor = cursor, pageSize = 20)
            require(resp.code == "OK") { resp.code }
            dao.upsert(resp.data.items.map { it.toEntity() })
        }
    }
}
```

附录 B：WebSocket 事件处理模板

```kotlin
fun shouldApplyEvent(currentVersion: Long, incomingVersion: Long): Boolean {
    return incomingVersion > currentVersion
}
```

附录 C：提交前自检清单

1. 是否所有写接口都自动注入 `X-Request-Id`。
2. 是否所有请求都注入 `X-Trace-Id`。
3. 是否把所有 ID 按字符串处理。
4. 是否处理了 401/403/429 的用户反馈与重试。
5. 是否补齐测试与埋点。
