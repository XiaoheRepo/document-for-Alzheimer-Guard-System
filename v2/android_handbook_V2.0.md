# 基于 AI Agent 的阿尔兹海默症（Alzheimer）患者协同寻回系统

## 家属端 Android 开发手册（AHB）V2.0

## 0. 文档信息

| 项目 | 内容 |
| :--- | :--- |
| 文档名称 | 家属端 Android 开发手册（Android Handbook） |
| 版本 | V2.0 |
| 发布日期 | 2026-04-22 |
| 输入基线 | SRS V2.0（[v2/SRS.md](SRS.md)）、SADD V2.0（[v2/SADD_V2.0.md](SADD_V2.0.md)）、LLD V2.0（[v2/LLD_V2.0.md](LLD_V2.0.md)）、DBD V2.0（[v2/DBD.md](DBD.md)）、API V2.0（[v2/API_V2.0.md](API_V2.0.md)）、BDD V2.0（[v2/backend_handbook_v2.md](backend_handbook_v2.md)）、CHANGELOG V2.1（[v2/CHANGELOG_V2.1.md](CHANGELOG_V2.1.md)）|
| 适用对象 | Android 研发、客户端架构评审、联调、测试、灰度与发布 |
| 文档定位 | Android **家属端**（角色 `FAMILY`）从 0 到 1 的落地基线；对齐 V2 全部硬约束与 API 契约 |
| 系统名称 | 「**码上回家**」 |
| 品牌主色 | `#F97316`（primary）/ `#0EA5E9`（link / secondary）|

说明：

1. 本手册是 V2 基线的**工程落地**产物，接口字段、状态机、错误码、事件语义以 [API_V2.0.md](API_V2.0.md) 为准，本手册只定义 Android 端实现方式与交付标准。
2. 家属端仅承载 `FAMILY` 角色视图。`ADMIN` / `SUPER_ADMIN` 的全部页面（用户治理、强制转移、DEAD 重放、全局审计、全局患者视图等）由 Web 管理端承载，见 [web_admin_handbook_V2.0.md](web_admin_handbook_V2.0.md)，Android **不消费**、**不呈现**、**不路由**这些能力。
3. 匿名路人上报链路由 **H5 线索端**承载，Android **不承载**浏览器匿名场景；Android 仅在"原生扫码 → 普通/紧急上报"场景下复用公开接口（`GET /r/{resource_token}`、`POST /api/v1/public/clues/manual-entry`、`POST /api/v1/clues/report`），详见 §20。
4. 本手册所有规则最终必须沉淀为代码、测试或 CI 校验；任一条"建议"在工程实施时应转化为硬性门禁。
5. 与基线冲突必须回退以基线为准；发现基线疑义时先起 RFC，不得在手册中私改。

## 1. 引言与范围

### 1.1 目标

1. 保证 Android 客户端与 V2 后端契约**逐字一致**，杜绝"字段对了但语义错了"。
2. 保证家属核心场景**端到端可用**：注册/登录 → 建档 → 物资申领 → 标签绑定 → 围栏配置 → 疑似走失确认 → 发布寻回任务 → 实时轨迹与线索 → 关闭任务 → 审计闭环。
3. 构建 **GUI + LUI 双模交互**（LUI = Language UI，自然语言对话）：GUI 覆盖全量功能入口，LUI 使家属在紧急场景下可用自然语言发起寻人、查询轨迹等高频动作；两模共享同一套后端 API，结果完全等价（对齐 SRS §4.1 FR-AI-001 ~ FR-AI-015、API §1.11 Agent 受控执行契约）。
4. 面向阿尔兹海默症家属群体，**默认支持"大字易读模式"**作为无障碍增强（最小正文 20sp、按钮 56dp、触控区 48dp、对比度 WCAG AA），在系统设置中一键切换。
5. 保证可维护性：模块清晰、状态可追踪、错误可定位、发布可回滚。

### 1.2 范围

范围内：

1. Android `FAMILY` 角色全量页面与交互实现。
2. 与 V2 后端的网络层、鉴权、WebSocket、SSE、推送、幂等、限流退避等联调落地。
3. 本地缓存、离线写队列、重试幂等、冲突回滚。
4. 主题（Light / Dark）、国际化（zh-CN / en-US）、无障碍（TalkBack / 大字易读模式）。
5. 单元测试、集成测试、UI 测试、契约测试、性能测试、灰度与回滚。

范围外：

1. Web 管理端、H5 线索端实现（由各自手册覆盖）。
2. 后端内部实现（由 BDD 覆盖）。
3. 治理后台能力（`/api/v1/admin/**`）在 Android 上的承载。
4. 医疗诊断能力、硬件生产制造能力。

### 1.3 名词与缩写

| 名词 | 说明 |
| :--- | :--- |
| AHB | Android Handbook，本文档 |
| GUI | Graphical User Interface，传统图形界面 |
| LUI | Language User Interface，自然语言交互界面（SRS 术语表） |
| 大字易读模式 | Android 家属端无障碍增强布局模式（单列、20sp 起、56dp 按钮、48dp 触控、WCAG AA 对比度） |
| HC | Hard Constraint，硬约束（源于 SADD §2） |
| FR | Functional Requirement，功能需求（源于 SRS §4） |
| BR | Business Rule，业务规则（源于 SRS §5） |
| BFF | Backend For Frontend，查询编排层（API §3.7） |
| PII | Personally Identifiable Information，个人身份信息 |
| CAS | Compare-And-Swap，乐观并发版本控制 |

### 1.4 文档约定

1. 接口路径、状态枚举、错误码、Header 名、类名、包名使用英文 `代码样式`。
2. 页面 ID 使用 `MH-*` 前缀（Mobile Handheld），便于与 Web `P-*`、H5 `H-*` 区分。
3. 所有 ID 字段（`task_id` / `patient_id` / `clue_id` / `user_id` / `order_id` / `tag_code` / `invite_id` / `transfer_request_id` / `notification_id` / `event_id` 等）在 Kotlin 层一律使用 `String` 承载；**禁止** `Long` / `Int` 中转（对齐 API §1.1 ID 传输强制约定 + HC-ID-String）。
4. 时间字段统一解析为 `java.time.Instant` / `OffsetDateTime`，UI 层再按 `Locale` 做本地化格式化。
5. 表格中"是/否"列的"—"表示"不适用"；"必填条件"表示"条件必填"。

### 1.5 需求追踪锚点

| 基线 | 家属端 Android 强相关章节 |
| :--- | :--- |
| SRS §4.1 FR-AI-001 ~ 015 | AI 双模交互（§22 AI 域）、LUI 执行确认（§22.3） |
| SRS §4.2 FR-CLUE-001 ~ 008 | 扫码动态路由（§20）、线索域（§19） |
| SRS §4.3 FR-TASK-001 ~ 005 | 任务域（§19）、状态机守卫（§9.5） |
| SRS §4.4 FR-PRO-001 ~ 014 | 档案/围栏/监护/标签（§17 §18） |
| SRS §4.5 FR-MAT-001 ~ 007 | 物资域（§21） |
| SRS §4.6 FR-GOV-001 ~ 014 | 鉴权/通知/我的（§16 §22） |
| SADD §2 HC-01 ~ HC-08 | 全局硬约束（§2） |
| SADD §4 领域边界 | 多模块划分（§4） |
| API V2.0 §1 全局规范 | 网络层（§10）、WS（§11） |
| API V2.0 §2 错误码字典 | 错误码映射（§24） |
| API V2.0 §6 Agent 白名单 | LUI Tool Call 映射（§22.3） |
| CHANGELOG V2.1 | 家属端**不受影响**，仅边界标注（§15.6） |

---

## 2. 硬约束（HC）在 Android 家属端的落地

本章对齐 SADD V2.0 §2 硬约束与 API V2.0 §1 全局规范，给出 Android 的逐条映射。任一 HC 违反即视为联调阻断问题。

### 2.1 HC 落地矩阵

| 编号 | 约束项 | Android 家属端落地方式 |
| :--- | :--- | :--- |
| HC-Stack | 技术栈锁定 | Kotlin `1.9.x`（JDK 17） + AGP `8.5.x` + Jetpack Compose BOM `2024.09.x`（Material 3） + Hilt + Retrofit + OkHttp + Kotlinx Serialization + Coroutines / Flow + Room + DataStore + WorkManager + Navigation-Compose + Coil；`minSdk=26`，`targetSdk=35`。**禁止**引入 Flutter / React Native / XML View 作为主方案，**禁止**新增 Java 业务代码。 |
| HC-Brand | 品牌与视觉 | 系统名固定「**码上回家**」；Compose Material 3 `lightColorScheme.primary = #F97316`、`secondary = #0EA5E9`、`tertiary = #0EA5E9`；`darkColorScheme` 主色保持 `#F97316`，`surface = #1F1F1F`、`background = #141414`；启动图、AppBar、Tab 指示条、主要 CTA 按钮全部以主色承载。详见 §5。 |
| HC-I18n | 中英双语 | `res/values/strings.xml`（默认 zh-CN） + `res/values-en/strings.xml`；全部可见字符串走 `stringResource()`，严禁硬编码；用户可在「我的 → 设置 → 语言」手动切换（通过 `AppCompatDelegate.setApplicationLocales`），日期与数字走 `java.text.NumberFormat` / `DateTimeFormatter.ofLocalizedDateTime`。详见 §7。 |
| HC-Theme | 亮 / 暗双主题 | `MaterialTheme(colorScheme = if (darkTheme) DarkColorScheme else LightColorScheme)`；默认跟随系统；用户可在设置中强制 Light / Dark；主题切换不重启 Activity。详见 §5.3。 |
| HC-01 | 六域隔离 | Feature 模块按 `task / clue / profile / mat / ai / notification / auth / me` 划分，**禁止** feature 跨域直接依赖对方内部类；跨域协作走 `core:domain` 定义的 UseCase 接口或 `core:eventbus`（本地事件总线）。详见 §4。 |
| HC-02 | 状态权威来自后端 | Android 禁止前端自行推算 `rescue_task.status` / `clue_record.review_state` / `tag.state` / `material_order.state` / `guardian_transfer_request.state`；收到 WS `state.changed` 事件后以服务端快照为准重渲染，前端仅维护 `UiState` 投影。详见 §9.4。 |
| HC-03 | 幂等键 `X-Request-Id` | 所有 `POST/PUT/PATCH/DELETE` 请求由 `RequestIdInterceptor` 自动注入 UUID v4（长度 16–64，`[A-Za-z0-9-]`）；失败重试必须**复用**同一 `X-Request-Id`；离线写队列入队时固化 `X-Request-Id`，重放时不重新生成。详见 §10.3。 |
| HC-04 | 全链路追踪 `X-Trace-Id` | 所有请求由 `TraceIdInterceptor` 注入；跨链路（如从推送跳转任务详情）透传上游 `trace_id`；响应回写的 `trace_id` 落结构化日志；错误 Toast 展示 `trace_id` 前 8 位供客服复述。详见 §10.3 §27。 |
| HC-05 | 无硬编码阈值 + 定向推送 | 速度阈值、围栏半径上限、轮询间隔、限流退避上限等全部来自 `GET /api/v1/admin/configs`（首启拉取，异常缓存默认值）；WebSocket 仅订阅当前登录用户与当前关注任务频道，**禁止**全量广播假设；未关注任务必须经 `POST /api/v1/ws/ticket` 重新协商后再订阅。详见 §11。 |
| HC-06 | 无短信 | App 内**严禁**出现任何"发送短信 / SMS 验证码"相关 UI、文案、按钮；密码重置走 `POST /api/v1/auth/password-reset/request`（邮件链接）；通知触达仅走"FCM / 华为 / 小米 等系统推送 + 站内消息中心"双通道。详见 §16.4。 |
| HC-07 | PII 脱敏 + 不落盘明文 | 本地 `EncryptedSharedPreferences` 存 Token；姓名、手机号、精确定位**禁止**进入明文日志与 Crashlytics；截图 / 分享链路对 PII 打码；剪贴板复制 `trace_id` 时不得附带 PII；启用 `FLAG_SECURE` 于档案编辑与监护转移页面。详见 §25。 |
| HC-08 | 网关保留头禁伪造 | Android 客户端**严禁**主动发送 `X-User-Id` / `X-User-Role`，由网关注入；若出现任一保留头，测试必须失败；伪造触发 `E_REQ_4003`，客户端视为"包升级或代理异常"并阻断重试。详见 §10.3.5。 |
| HC-Auth 角色 | 家属端仅 `FAMILY` | 登录成功后，若响应 `user.role != FAMILY`，客户端立即登出并提示"该账号不支持家属端登录，请使用 Web 管理端"；**禁止** Android 在任何菜单、入口、深链中暴露 `ADMIN` / `SUPER_ADMIN` 的治理页面。详见 §16.1。 |
| HC-ID-String | ID 字符串 | 所有 ID 字段在 DTO、Domain、Entity、UI 层一律 `String`；`@Serializable data class TaskDto(val task_id: String, …)`；**禁止** `task_id.toLong()` 后再传回接口；Compose 导航参数 `navArgument("task_id") { type = NavType.StringType }`。详见 §5.2。 |
| HC-Coord | 坐标系统一 | 高德定位原始坐标为 `GCJ-02`，上报时 `coord_system="GCJ-02"`；服务端返回固定 `WGS84`；如地图 SDK 需反向转换，只在**展示层**进行，不污染 Domain / Entity。详见 §11.6。 |
| HC-Realtime | 实时能力 | WebSocket 用于任务详情页 / 地图页 / 通知中心的实时态；推送用于 App 在后台时的深链唤起；离线期间用 WorkManager 做通知增量拉取补洞；断线按 `2^n + jitter` 退避，上限 30s。详见 §11。 |
| HC-A11y | 无障碍 | 所有可点击元素必须有 `contentDescription`；支持 TalkBack 连读；色彩对比度 ≥ WCAG AA（正文 4.5:1、标题 / 数字 ≥ 7:1）；**大字易读模式**为家属端强制特性（默认跟随系统「最大字体缩放」，也可手动开关）。详见 §6.4。 |

### 2.2 全局 Header 契约（与 API V2.0 §1.2 逐字对齐）

| Header | 注入位置 | 规则 |
| :--- | :--- | :--- |
| `Authorization: Bearer <access_token>` | `AuthInterceptor` | 登录后从 `AuthTokenStore`（`EncryptedSharedPreferences`）读取；401 触发 `/auth/token/refresh`，刷新失败跳登录 |
| `X-Request-Id` | `RequestIdInterceptor`（仅写方法） | UUID v4，长度 16–64，`^[A-Za-z0-9-]{16,64}$`；重试复用 |
| `X-Trace-Id` | `TraceIdInterceptor`（所有请求） | 新请求 UUID v4；跨链路透传；长度 16–64 |
| `Content-Type: application/json; charset=utf-8` | Retrofit 默认 | 文件上传单独处理（`multipart/form-data`） |
| `X-Anonymous-Token` | `PublicApi` 专用 | 仅原生扫码链路（§20）使用；登录态接口**禁止**携带 |
| `X-User-Id` / `X-User-Role` | **严禁** | 客户端发送即被网关拒绝（`E_REQ_4003`） |
| `X-Action-Source` / `X-Agent-Profile` / `X-Execution-Mode` / `X-Confirm-Level` | LUI 链路 | 由 AI 网关代为处理；Android 客户端**不主动**设置，仅在调用 `POST /api/v1/ai/sessions/{id}/intents/{intent_id}/confirm` 时透传后端回传的 `intent_id` |

### 2.3 统一响应外壳处理规则（对齐 API §1.4）

Android 端 `ApiResponseAdapter`（Retrofit CallAdapter）统一规则：

1. HTTP 2xx 且 `code == "ok"` → 返回 `data`，并把 `trace_id` 放入 `Result.Success.trace`。
2. HTTP 2xx 且 `code` 以 `E_` 开头 → 包装为 `DomainException(code, message, trace)`，按 §24 错误码表映射到 UI。
3. HTTP 401 / `E_AUTH_4011` / `E_GOV_4011` → `AuthRepository.refresh()` 尝试一次；失败则清空会话、断开 WS、跳 `AuthLoginScreen`。
4. HTTP 403 / `E_GOV_4030` / `E_GOV_4031` → 展示只读提示或封禁提示，**不**重试。
5. HTTP 429 / `E_GOV_4291` / `E_GOV_4292` / `E_AI_4292` / `E_AI_4293` → 读取 `Retry-After` + `X-RateLimit-Reset` 做指数退避，UI 显示倒计时；缺 `Retry-After` 走本地 `min(base * 2^n + jitter, 30s)`。
6. 网络错误 / 超时 → `notification.error` 附 `trace_id` 前 8 位；同一请求幂等键最多自动重试 2 次（仅幂等读接口）。

### 2.4 联调前必过清单（HC-Check）

| # | 检查项 | 校验手段 |
| :---: | :--- | :--- |
| 1 | 所有 ID 按 `String` 传输 | Contract Test：对 10 个关键写接口断言 `task_id is String` |
| 2 | 所有写接口携带 `X-Request-Id` 且长度合法 | OkHttp `EventListener` 埋点 → CI Gate |
| 3 | 所有请求携带 `X-Trace-Id` 且响应回写一致 | Integration Test：对比请求/响应 `trace_id` |
| 4 | 遇 `E_REQ_4003` 不重试，提示"版本或代理异常" | UI Test：MockWebServer 构造该响应 |
| 5 | 401 统一走 `AuthSession.invalidate()` | UI Test |
| 6 | 429 优先读 `Retry-After` | Unit Test |
| 7 | `X-User-Id` / `X-User-Role` 客户端**零发送** | OkHttp Interceptor 断言 |
| 8 | 匿名链路仅用 `X-Anonymous-Token`，登录态不带 | Contract Test |

---

## 3. 技术栈与依赖版本矩阵

### 3.1 基础栈（锁版本，禁止替换）

| 类别 | 选型 | 版本 | 备注 |
| :--- | :--- | :--- | :--- |
| 语言 | Kotlin | `1.9.25` | JDK 17 目标，`-Xjsr305=strict` |
| 构建 | Android Gradle Plugin | `8.5.2` | Gradle `8.7`；Kotlin DSL 全量 |
| Compose BOM | `androidx.compose:compose-bom` | `2024.09.03` | Material 3 `1.3.x` |
| 最低 / 目标系统 | minSdk / targetSdk / compileSdk | `26` / `35` / `35` | 覆盖主流设备与最新政策 |
| DI | Hilt | `2.52` | `hilt-navigation-compose 1.2.0` |
| 网络 | OkHttp | `4.12.0` | `logging-interceptor` 仅 Debug |
| 网络 | Retrofit | `2.11.0` | `retrofit2-kotlinx-serialization-converter 1.0.0` |
| 序列化 | Kotlinx Serialization | `1.7.3` | 严格匹配 API DTO（`snake_case`） |
| 协程 | Kotlin Coroutines | `1.9.0` | `Flow` / `StateFlow` / `SharedFlow` |
| 导航 | Navigation-Compose | `2.8.1` | 深链接（`applink`）与类型安全参数 |
| 本地数据库 | Room | `2.6.1` | KSP 驱动，支持迁移 |
| 键值存储 | DataStore | `1.1.1` | Preferences 与 Proto 双形态 |
| 安全存储 | `androidx.security:security-crypto-ktx` | `1.1.0-alpha06` | Token / 敏感标识加密 |
| 后台任务 | WorkManager | `2.9.1` | 离线写队列与通知补洞 |
| 图片加载 | Coil | `2.7.0` | `coil-compose`；白名单 URL |
| WebSocket | OkHttp WebSocket | 同 OkHttp | 自研 `MhWebSocketClient`（§11） |
| SSE（AI 流式） | OkHttp-EventSource | `okhttp-sse 4.12.0` | 用于 `POST /api/v1/ai/sessions/{id}/messages` |
| 推送 | FCM（Android 基线） | `firebase-bom 33.3.0` | 国内渠道同步接入华为 / 小米 / OPPO / vivo |
| 地图定位 | 高德 Android SDK | `Map 10.x` + `Location 6.x` | 对齐 API §1.9 高德接入约定 |
| 扫码 | ZXing + CameraX | `zxing-core 3.5.3` / `camera-* 1.3.4` | 原生扫码链路 |
| 日志 | Timber | `5.0.1` | Release 包关闭敏感日志 |
| 崩溃上报 | Firebase Crashlytics | 同 FCM | 上传前脱敏（§25） |
| Lint | ktlint + detekt + Android Lint | `ktlint 1.3.1` / `detekt 1.23.7` | CI 硬门禁 |
| 单元测试 | JUnit5 + MockK + Turbine | `5.10.x` / `1.13.12` / `1.2.0` | 协程与 Flow 友好 |
| 集成测试 | MockWebServer | 同 OkHttp | Contract / Error-Code / Retry 回归 |
| UI 测试 | Compose UI Test + Espresso | BOM 与 `3.6.x` | 关键流程回归 |
| 截图测试 | Paparazzi（可选） | `1.3.4` | 主题与大字模式快照 |

### 3.2 版本升级原则

1. 每次升级必须附带回归测试结果（单测 + 契约 + 关键 UI 流程）。
2. Compose BOM 升级视为"大版本"，需跑全量快照测试 + 性能基线对比（冷启动 / 首屏 TTI / 帧率）。
3. 高危 CVE 在 72h 内紧急升级；其余月度升级 + 季度审计。
4. 依赖版本统一走 `libs.versions.toml`（Version Catalog），**禁止** feature 模块硬编码版本。

### 3.3 显式禁止依赖

| 禁止项 | 理由 |
| :--- | :--- |
| React Native / Flutter / Weex（作为主方案） | 违反 HC-Stack |
| 任何 `XML View` 新增业务界面 | Compose 单向数据流更契合 MVI；XML 仅允许出现在 `MainActivity` 容器 |
| 明文存储 JWT / 手机号 / 姓名的库 | 违反 HC-07 |
| 未接入 `core-ui-adapter` 的第三方 UI 库 | 违反"第三方白名单 + 适配层"策略（§4.3） |
| 自实现 BigInt / Long → Number 序列化 | ID 已是 `String`（HC-ID-String） |

### 3.4 构建脚本约定（节选 `libs.versions.toml`）

```toml
[versions]
kotlin = "1.9.25"
agp = "8.5.2"
compose-bom = "2024.09.03"
hilt = "2.52"
retrofit = "2.11.0"
okhttp = "4.12.0"
serialization = "1.7.3"
coroutines = "1.9.0"
room = "2.6.1"
navigation = "2.8.1"
workmanager = "2.9.1"
coil = "2.7.0"
timber = "5.0.1"
amap-map = "10.0.800"
amap-location = "6.4.9"

[libraries]
compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "compose-bom" }
compose-material3 = { group = "androidx.compose.material3", name = "material3" }
retrofit-core = { group = "com.squareup.retrofit2", name = "retrofit", version.ref = "retrofit" }
retrofit-serialization = { group = "com.jakewharton.retrofit", name = "retrofit2-kotlinx-serialization-converter", version = "1.0.0" }
okhttp-core = { group = "com.squareup.okhttp3", name = "okhttp", version.ref = "okhttp" }
okhttp-sse = { group = "com.squareup.okhttp3", name = "okhttp-sse", version.ref = "okhttp" }
okhttp-logging = { group = "com.squareup.okhttp3", name = "logging-interceptor", version.ref = "okhttp" }
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-compiler", version.ref = "hilt" }
hilt-nav-compose = { group = "androidx.hilt", name = "hilt-navigation-compose", version = "1.2.0" }
kotlinx-serialization-json = { group = "org.jetbrains.kotlinx", name = "kotlinx-serialization-json", version.ref = "serialization" }
kotlinx-coroutines-android = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-android", version.ref = "coroutines" }
room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }
workmanager = { group = "androidx.work", name = "work-runtime-ktx", version.ref = "workmanager" }
datastore-preferences = { group = "androidx.datastore", name = "datastore-preferences", version = "1.1.1" }
security-crypto = { group = "androidx.security", name = "security-crypto-ktx", version = "1.1.0-alpha06" }
coil-compose = { group = "io.coil-kt", name = "coil-compose", version.ref = "coil" }
amap-map = { group = "com.amap.api", name = "3dmap", version.ref = "amap-map" }
amap-location = { group = "com.amap.api", name = "location", version.ref = "amap-location" }
timber = { group = "com.jakewharton.timber", name = "timber", version.ref = "timber" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
android-library = { id = "com.android.library", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version.ref = "kotlin" }
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
ksp = { id = "com.google.devtools.ksp", version = "1.9.25-1.0.20" }
```

### 3.5 签名与环境矩阵

| 环境 | applicationIdSuffix | minify | `BuildConfig.API_BASE` | 日志 |
| :--- | :--- | :--- | :--- | :--- |
| `debug` | `.debug` | 关闭 | `https://dev-api.mshj.example.com` | Timber + HTTP Body |
| `staging` | `.staging` | 开启（保留 map） | `https://stg-api.mshj.example.com` | Timber（Info+）+ HTTP 摘要 |
| `release` | 无 | 开启 | `https://api.mshj.example.com` | 仅 Crashlytics |

---

## 4. 工程目录结构

### 4.1 多模块划分

```text
mshj-android/
├─ app/                                 # 壳工程：Application、MainActivity、NavGraph 总装
├─ build-logic/                         # Convention Plugin（统一 compileSdk / compose 配置）
│  └─ convention/
├─ core/
│  ├─ core-common/                      # Result / DomainException / Clock / Ids / Validators
│  ├─ core-designsystem/                # Material3 Theme / ColorScheme / Typography / Shape / Motion
│  │                                    #   + MhAppBar / MhButton / MhLoading / MhEmpty / MhError
│  │                                    #   + AccessibilityMode（大字易读模式）
│  ├─ core-ui-adapter/                  # 第三方 UI 唯一接入点（Coil / Lottie / 图表 / 高德）
│  ├─ core-network/                     # Retrofit / OkHttp / 拦截器 / ApiResponseAdapter
│  ├─ core-database/                    # Room DB、DAO、迁移脚本
│  ├─ core-datastore/                   # EncryptedSharedPreferences + DataStore
│  ├─ core-realtime/                    # WebSocket / SSE / 重连 / 去重 / 事件总线
│  ├─ core-permission/                  # 运行时权限 + 按钮级权限（HC-Auth）
│  ├─ core-location/                    # 高德定位 / 前台服务 / 坐标系声明
│  ├─ core-analytics/                   # 埋点 / Crashlytics 脱敏 / trace 关联
│  ├─ core-i18n/                        # Locale 切换与 `stringResource` 扩展
│  └─ core-testing/                     # 测试桩、假数据、测试规则
├─ domain/                              # 纯 Kotlin 模块（无 Android 依赖）
│  ├─ task/ clue/ profile/ mat/ ai/ notification/ auth/
│  └─ 每个子包：UseCase + DomainModel + Repository 接口
├─ data/                                # 每个子模块对应 RepositoryImpl + RemoteDataSource + LocalDataSource
│  └─ task/ clue/ profile/ mat/ ai/ notification/ auth/
├─ feature/                             # UI + ViewModel；禁止相互依赖
│  ├─ feature-auth/                     # MH-AUTH-*
│  ├─ feature-home/                     # MH-HOME-01
│  ├─ feature-patient/                  # MH-PAT-*
│  ├─ feature-guardian/                 # MH-GUA-*
│  ├─ feature-tag/                      # MH-TAG-*
│  ├─ feature-task/                     # MH-TASK-*
│  ├─ feature-clue/                     # MH-CLUE-*
│  ├─ feature-public/                   # MH-PUB-*（原生扫码链路）
│  ├─ feature-material/                 # MH-ORD-*
│  ├─ feature-notification/             # MH-NOTI-*
│  ├─ feature-ai/                       # MH-AI-*（GUI + LUI）
│  └─ feature-me/                       # MH-ME-*（我的 / 设置）
└─ baselineprofile/                     # Baseline Profile 生成（冷启动优化）
```

### 4.2 依赖方向约束（架构守护）

```
feature-* ───▶ domain/*                   （UseCase / Repository 接口）
feature-* ───▶ core-designsystem           （主题与公共组件）
feature-* ───▶ core-common / core-i18n     （工具与文案）
feature-* ───▶ core-permission / core-ui-adapter（按需）

data/* ─────▶ domain/*                    （实现 Repository 接口）
data/* ─────▶ core-network / core-database / core-datastore / core-realtime

app ────────▶ feature-* / data/* / core/*（仅 `app` 可装配全部）
```

**强制规则**：

1. **`feature-X` 禁止依赖 `feature-Y`**；跨域协作只能通过 `domain` UseCase 或 `core-realtime` 本地事件总线。
2. **`domain` 禁止依赖 `android.*`**；纯 Kotlin 单元测试覆盖。
3. **`feature` 禁止依赖 `data`**；必须通过 Hilt 注入 `Repository` 接口（在 `app` 层绑定实现）。
4. **`feature` 禁止直接 `import` 第三方库**（OkHttp / Coil / AMap / Retrofit 的 DTO 不能跨越 `data` 边界）。
5. 循环依赖由 `:build-logic:convention:ArchitectureCheckPlugin` 在 CI 阻断。

### 4.3 第三方 UI 适配层（`core-ui-adapter`）

遵循"**业务页面只依赖抽象，第三方替换只改 adapter**"：

```kotlin
// core-ui-adapter/src/main/kotlin/.../MhImage.kt
interface MhImage {
    @Composable fun Remote(url: String, contentDescription: String?, modifier: Modifier = Modifier)
}

// 实现：Coil
class CoilMhImage @Inject constructor() : MhImage {
    @Composable override fun Remote(url: String, contentDescription: String?, modifier: Modifier) {
        AsyncImage(model = url, contentDescription = contentDescription, modifier = modifier)
    }
}
```

第三方白名单（V2 基线）：

| 能力域 | 库 | 接入模块 | 使用边界 |
| :--- | :--- | :--- | :--- |
| 图片加载 | Coil | `core-ui-adapter` | 仅渲染与缓存，不处理鉴权 |
| 动画 | Lottie（可选） | `core-ui-adapter` | 仅引导动画 / 状态动画 |
| 图表 | Vico（可选） | `core-ui-adapter` | 仅客户端侧统计图（如配额进度） |
| 地图 | 高德 Android SDK | `core-location` + `core-ui-adapter` | 仅展示与定位，业务状态不依赖 |
| 扫码 | ZXing + CameraX | `feature-public` 私有 | 仅扫码解析，不做业务判定 |

**禁止**：`feature` 模块直接 `import com.amap.*` / `import coil.*`。

### 4.4 包命名规范

| 层 | 包名前缀 | 示例 |
| :--- | :--- | :--- |
| Feature UI | `com.mshj.feature.<domain>.ui` | `com.mshj.feature.task.ui.detail.TaskDetailScreen` |
| Feature ViewModel | `com.mshj.feature.<domain>.presentation` | `TaskDetailViewModel` |
| Domain | `com.mshj.domain.<domain>` | `GetTaskSnapshotUseCase` |
| Data | `com.mshj.data.<domain>` | `TaskRepositoryImpl` |
| Core | `com.mshj.core.<ability>` | `com.mshj.core.network.interceptor.TraceIdInterceptor` |

### 4.5 Convention Plugin（节选）

```kotlin
// build-logic/convention/src/main/kotlin/AndroidLibraryConventionPlugin.kt
class AndroidLibraryConventionPlugin : Plugin<Project> {
    override fun apply(target: Project) = with(target) {
        pluginManager.apply("com.android.library")
        pluginManager.apply("org.jetbrains.kotlin.android")
        extensions.configure<LibraryExtension> {
            compileSdk = 35
            defaultConfig { minSdk = 26 }
            compileOptions {
                sourceCompatibility = JavaVersion.VERSION_17
                targetCompatibility = JavaVersion.VERSION_17
            }
            kotlinOptions { jvmTarget = "17"; freeCompilerArgs += "-Xjsr305=strict" }
        }
    }
}
```

---

## 5. 主题与设计令牌（Material 3）

### 5.1 品牌与语义色（与 Web / H5 端口径一致）

基线：Web Admin Handbook V2.0 §4、H5 Handbook（同期对齐）。主色 `#F97316`（品牌橙，紧急温暖），辅色 `#0EA5E9`（天蓝，理性链接）。

| 语义令牌 | Light | Dark | 用途 |
| :--- | :--- | :--- | :--- |
| `primary` | `#F97316` | `#F97316` | 主按钮、主 CTA、Tab 指示条、任务进行中高亮 |
| `onPrimary` | `#FFFFFF` | `#1A0F04` | 主色前景文字 |
| `primaryContainer` | `#FFEDD5` | `#7C2D12` | 主色次级容器（芯片、选中态） |
| `onPrimaryContainer` | `#7C2D12` | `#FFEDD5` | 主色容器前景 |
| `secondary` | `#0EA5E9` | `#38BDF8` | 链接、次级操作、信息提示 |
| `onSecondary` | `#FFFFFF` | `#082F49` | 次色前景 |
| `tertiary` | `#0EA5E9` | `#38BDF8` | 与 secondary 同族，预留给地图浮层 / 告警前图标 |
| `error` | `#EF4444` | `#F87171` | 错误、失败、不可逆 CTA 描边 |
| `onError` | `#FFFFFF` | `#450A0A` | 错误前景 |
| `background` | `#F5F7FA` | `#141414` | 页面背景 |
| `onBackground` | `#0F172A` | `#E5E7EB` | 页面主文字 |
| `surface` | `#FFFFFF` | `#1F1F1F` | 卡片、BottomSheet |
| `onSurface` | `#0F172A` | `#E5E7EB` | 卡片文字 |
| `surfaceVariant` | `#F1F5F9` | `#262626` | 次级容器（筛选条 / 表单组背景） |
| `onSurfaceVariant` | `#475569` | `#CBD5E1` | 次级容器文字 / 图标 |
| `outline` | `#CBD5E1` | `#525252` | 分割线、描边 |
| `outlineVariant` | `#E2E8F0` | `#3F3F46` | 弱分割线 |

### 5.2 状态语义色（业务到视觉）

| 业务状态 | 背景（Light） | 前景（Light） | 背景（Dark） | 前景（Dark） | 使用位置 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `ACTIVE`（任务进行中） | `#FFEDD5` | `#C2410C` | `#7C2D12` | `#FDBA74` | 任务状态芯片 / 时间线首节点 |
| `CREATED` | `#FEF3C7` | `#B45309` | `#78350F` | `#FCD34D` | 新建任务 |
| `RESOLVED` | `#DCFCE7` | `#15803D` | `#14532D` | `#86EFAC` | 任务寻回 |
| `FALSE_ALARM` | `#E0E7FF` | `#4338CA` | `#1E1B4B` | `#A5B4FC` | 任务误报 |
| `MISSING` | `#FEE2E2` | `#B91C1C` | `#450A0A` | `#FCA5A5` | 走失状态 |
| `MISSING_PENDING` | `#FFEDD5` | `#C2410C` | `#7C2D12` | `#FDBA74` | 疑似走失待确认 |
| `UNREAD` | `#E0F2FE` | `#0369A1` | `#082F49` | `#7DD3FC` | 未读通知 |
| `READ` | `#F1F5F9` | `#64748B` | `#262626` | `#94A3B8` | 已读通知 |
| `RISK_CRITICAL` | `#FEE2E2` | `#B91C1C` | `#450A0A` | `#FCA5A5` | 红色高危告警条 |
| `RISK_WARN` | `#FFEDD5` | `#C2410C` | `#7C2D12` | `#FDBA74` | 橙色预警条 |

### 5.3 Compose 主题骨架

```kotlin
// core-designsystem/src/main/kotlin/com/mshj/core/designsystem/theme/Color.kt
val MhOrange         = Color(0xFFF97316)
val MhOrangeDark     = Color(0xFFEA580C)
val MhOrangeContainer     = Color(0xFFFFEDD5)
val MhOrangeContainerDark = Color(0xFF7C2D12)
val MhSky            = Color(0xFF0EA5E9)
val MhSkyLight       = Color(0xFF38BDF8)
// ……其余令牌见 §5.1

val MhLightColors = lightColorScheme(
    primary = MhOrange, onPrimary = Color.White,
    primaryContainer = MhOrangeContainer, onPrimaryContainer = Color(0xFF7C2D12),
    secondary = MhSky, onSecondary = Color.White,
    tertiary  = MhSky,
    error = Color(0xFFEF4444), onError = Color.White,
    background = Color(0xFFF5F7FA), onBackground = Color(0xFF0F172A),
    surface = Color.White, onSurface = Color(0xFF0F172A),
    surfaceVariant = Color(0xFFF1F5F9), onSurfaceVariant = Color(0xFF475569),
    outline = Color(0xFFCBD5E1), outlineVariant = Color(0xFFE2E8F0),
)

val MhDarkColors = darkColorScheme(
    primary = MhOrange, onPrimary = Color(0xFF1A0F04),
    primaryContainer = MhOrangeContainerDark, onPrimaryContainer = Color(0xFFFFEDD5),
    secondary = MhSkyLight, onSecondary = Color(0xFF082F49),
    tertiary  = MhSkyLight,
    error = Color(0xFFF87171), onError = Color(0xFF450A0A),
    background = Color(0xFF141414), onBackground = Color(0xFFE5E7EB),
    surface = Color(0xFF1F1F1F), onSurface = Color(0xFFE5E7EB),
    surfaceVariant = Color(0xFF262626), onSurfaceVariant = Color(0xFFCBD5E1),
    outline = Color(0xFF525252), outlineVariant = Color(0xFF3F3F46),
)
```

```kotlin
// core-designsystem/src/main/kotlin/com/mshj/core/designsystem/theme/Theme.kt
@Composable
fun MhTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    accessibilityLarge: Boolean = LocalAccessibility.current.largeMode,
    content: @Composable () -> Unit,
) {
    val colorScheme = if (darkTheme) MhDarkColors else MhLightColors
    MaterialTheme(
        colorScheme = colorScheme,
        typography  = if (accessibilityLarge) MhLargeTypography else MhTypography,
        shapes      = MhShapes,
        content     = content,
    )
}
```

### 5.4 主题切换策略

1. `AppPrefsStore.themeMode: 'SYSTEM' | 'LIGHT' | 'DARK'`（DataStore 持久化）。
2. `SYSTEM` 模式监听 `isSystemInDarkTheme()`；切换不重启 Activity。
3. **Dark 模式**主色保持 `#F97316`，仅调背景 / 表面 / 文字，避免品牌色漂移。
4. Compose Preview 必须同时提供 `@Preview(name = "Light")` + `@Preview(name = "Dark")` + `@Preview(fontScale = 1.5f, name = "Large")`。
5. 主题与大字易读模式是**正交**维度：4 组合（Light/Dark × Normal/Large）必须在 Paparazzi 截图测试全覆盖。

### 5.5 Shape 与 Elevation

| 令牌 | 值 | 适用 |
| :--- | :--- | :--- |
| `MhShapes.small` | `RoundedCornerShape(8.dp)` | 芯片、输入框 |
| `MhShapes.medium` | `RoundedCornerShape(12.dp)` | 卡片默认 |
| `MhShapes.large` | `RoundedCornerShape(16.dp)` | BottomSheet、弹窗 |
| `MhShapes.extraLarge` | `RoundedCornerShape(24.dp)` | 启动图、Hero Banner |
| Elevation 0 | `0.dp` | 背景容器、分隔面板 |
| Elevation 1 | `1.dp` | 常规卡片（Light 下主要阴影） |
| Elevation 3 | `3.dp` | 浮层（BottomSheet / Dialog） |
| Elevation 6 | `6.dp` | AI 对话的 Tool Intent 确认卡片 |

### 5.6 颜色使用硬约束

1. 主按钮 = `primary` 背景 + `onPrimary` 文字；次按钮 = `surface` 背景 + `primary` 描边 + `primary` 文字；危险按钮 = `error` 背景。
2. **同屏禁止**并列两个"主按钮"同权重。
3. 状态传达**不得只依赖颜色**：必须同时提供图标或文案（如"已读"文案 + 圆点色）。
4. 红绿并存场景必须提供形状差异（实心 vs 描边），兼容色觉缺陷。
5. ECharts / Vico 图表主轴 `#F97316`、辅轴 `#0EA5E9`、网格线 Light `#E2E8F0` / Dark `#3F3F46`。

---

## 6. 排版、图标、动效与无障碍

### 6.1 字体族

```kotlin
val MhFontFamily = FontFamily(
    Font(R.font.harmonyos_sans_regular, FontWeight.Normal),
    Font(R.font.harmonyos_sans_medium,  FontWeight.Medium),
    Font(R.font.harmonyos_sans_bold,    FontWeight.Bold),
)
```

说明：优先使用系统自带字体（HarmonyOS Sans / PingFang SC / Roboto），不内嵌商业字体；内嵌中文字体**禁止**（APK 体积）。

### 6.2 Typography 阶梯（Normal vs Large）

| 角色 | Normal（sp） | Large（sp，×1.25） | 用途 |
| :--- | :---: | :---: | :--- |
| `displayLarge` | 40 | 52 | 仅启动图 / 品牌页 |
| `headlineLarge` | 28 | 36 | 页面主标题（任务详情顶部） |
| `headlineMedium` | 24 | 30 | 分段标题（卡片组标题） |
| `titleLarge` | 20 | 26 | 卡片标题、AppBar 标题 |
| `titleMedium` | 16 | 20 | 列表项主文字 |
| `bodyLarge` | 16 | 20 | 正文 |
| `bodyMedium` | 14 | 18 | 次级正文（描述、备注） |
| `labelLarge` | 14 | 18 | 按钮文字（最小可点击字号） |
| `labelMedium` | 12 | 16 | 芯片、标签 |

**大字易读模式（Accessibility Large Mode）强制约束**：

1. 正文最小 `20sp`、强调 `24/28sp`。
2. 按钮最小高度 `56dp`、最小宽度 `120dp`；图标按钮最小尺寸 `48dp × 48dp`。
3. 最小点击区 `48dp × 48dp`（覆盖 Material 3 默认 `48dp`）。
4. 单列布局：`LazyColumn` + `padding(horizontal = 16.dp)`；禁止 2 列及以上密集网格。
5. 色彩对比度 ≥ `7:1`（高于 WCAG AAA 下限）。
6. 图标 ≥ `32dp`；危险图标额外加描边。

### 6.3 图标

1. 图标统一来自 `androidx.compose.material.icons`（Material Symbols Rounded 风格）。
2. 禁止 feature 直接引用 `androidx.compose.material.icons.filled.*`，统一通过 `MhIcons` 对象重导出，便于整体更换图标集。
3. 自定义图标存放 `core-designsystem/src/main/res/drawable/`，命名 `ic_mh_<domain>_<name>.xml`；禁止栅格 PNG。

### 6.4 无障碍（A11y）

| 项 | 要求 |
| :--- | :--- |
| `contentDescription` | 所有可点击元素、图片、图标按钮必须提供；装饰性图片显式设 `null` |
| TalkBack | 列表项语义合并（`Modifier.semantics(mergeDescendants = true)`）；任务详情时间线按时间顺序朗读 |
| 焦点顺序 | 使用 `Modifier.focusProperties { next = focusRequester }` 保证 Tab 键 / 切换按钮按视觉顺序 |
| 语言朗读 | 与 `AppCompatDelegate.setApplicationLocales` 同步，TalkBack 读出正确语言 |
| 触控区 | 全部 ≥ 48dp（Material 3 默认）；大字易读模式下提升至 56dp |
| 动态字号 | `fontScale` 必须支持至少到 `2.0f` 不破版（关键：避免 `fixed text size`）|
| 色觉 | 不单独依赖颜色传达信息；附加图标 / 文案冗余 |
| 隐藏辅助 | 纯装饰性动画 `Modifier.semantics { invisibleToUser() }` |

`AccessibilityMode` 数据流：

```kotlin
data class AccessibilityState(val largeMode: Boolean, val highContrast: Boolean)

val LocalAccessibility = compositionLocalOf { AccessibilityState(false, false) }

@Composable
fun ProvideAccessibility(state: AccessibilityState, content: @Composable () -> Unit) {
    CompositionLocalProvider(LocalAccessibility provides state, content = content)
}
```

**设置页切换**：`我的 → 设置 → 无障碍`，开关"大字易读模式"→ 同步写 DataStore → 所有页面立即重组。

### 6.5 动效（Motion）

| 场景 | 曲线 | 时长 |
| :--- | :--- | :--- |
| 页面过渡 | `FastOutSlowInEasing` | 250ms |
| 卡片展开 / 折叠 | `EaseOutCubic` | 200ms |
| Bottom Sheet | `spring(stiffness = Spring.StiffnessMedium)` | — |
| 状态颜色切换 | `tween(150)` | 150ms |
| 告警红条入场 | `fadeIn + expandVertically` | 200ms |
| Loading 骨架屏 | `shimmer`（周期 1200ms） | 循环 |

**禁令**：

1. 告警出现**禁止**使用高频闪烁（频率 > 3Hz，光敏性癫痫风险）。
2. 加载态**禁止**使用 `error` 色。
3. 页面过渡**禁止**超过 300ms。

### 6.6 触控与手势

1. 主要操作按钮放置在视觉"右下"或"底部居中"，便于单手操作。
2. 关键破坏性动作（关闭任务、删除患者、确认走失）必须二次确认；确认弹窗主按钮为 `error` 风格。
3. 滑动手势仅用于"列表项标记已读 / 删除"，与 `IconButton` 并行提供，避免误触。

---

## 7. 国际化（i18n）

### 7.1 方案概览

1. 默认语言 `zh-CN`；次级语言 `en-US`；后续可扩展。
2. 全部可见文案走 `androidx.compose.ui.res.stringResource(R.string.xxx)` 或 `LocalContext.current.getString()`；**严禁**硬编码中英文。
3. 语言切换通过 `AppCompatDelegate.setApplicationLocales(LocaleListCompat.forLanguageTags("zh-CN"))`；与系统级"按应用语言设置"兼容。
4. 日期 / 时间走 `DateTimeFormatter.ofLocalizedDateTime(FormatStyle.SHORT).withLocale(locale)`；数字走 `NumberFormat.getInstance(locale)`。
5. TalkBack 朗读语言与 `AppCompatDelegate` 同步。

### 7.2 目录结构

```text
core-i18n/src/main/res/
├─ values/
│  ├─ strings_common.xml          # 通用（确定/取消/重试/加载中…）
│  ├─ strings_menu.xml            # 导航 / 菜单
│  ├─ strings_auth.xml            # 登录 / 注册 / 改密 / 重置
│  ├─ strings_home.xml            # 家庭工作台
│  ├─ strings_patient.xml         # 患者档案 / 围栏
│  ├─ strings_guardian.xml        # 监护管理 / 邀请 / 转移
│  ├─ strings_tag.xml             # 标签
│  ├─ strings_task.xml            # 寻回任务
│  ├─ strings_clue.xml            # 线索
│  ├─ strings_public.xml          # 原生扫码链路
│  ├─ strings_material.xml        # 物资工单
│  ├─ strings_notification.xml    # 消息中心
│  ├─ strings_ai.xml              # AI 对话（GUI + LUI）
│  ├─ strings_me.xml              # 我的 / 设置
│  ├─ strings_error.xml           # 错误码 → 文案（对齐 API §2）
│  └─ strings_enum.xml            # 枚举翻译（TaskState / TagState …）
└─ values-en/
   └─ …同上
```

### 7.3 命名规范

| 类型 | 规范 | 示例 |
| :--- | :--- | :--- |
| 页面标题 | `screen.<domain>.<page>.title` | `screen.task.detail.title` |
| 按钮 | `btn.<domain>.<action>` | `btn.task.close` |
| 错误码 | `error.<code>` | `error.E_TASK_4091` |
| 枚举 | `enum.<type>.<value>` | `enum.task_state.ACTIVE` |
| 表单字段 | `form.<domain>.<field>` | `form.patient.birthday.label` |
| 表单提示 | `form.<domain>.<field>.placeholder` | `form.patient.birthday.placeholder` |
| 字段校验 | `form.<domain>.<field>.error.<rule>` | `form.patient.birthday.error.required` |

### 7.4 示例（中英对照）

```xml
<!-- values/strings_task.xml -->
<resources>
    <string name="screen.task.detail.title">任务详情</string>
    <string name="btn.task.close">关闭任务</string>
    <string name="btn.task.close.false_alarm">标记为误报</string>
    <string name="task.close.reason.hint">请输入误报原因（5–256 字）</string>
    <string name="enum.task_state.CREATED">已创建</string>
    <string name="enum.task_state.ACTIVE">进行中</string>
    <string name="enum.task_state.RESOLVED">已寻回</string>
    <string name="enum.task_state.FALSE_ALARM">已误报</string>
</resources>
```

```xml
<!-- values-en/strings_task.xml -->
<resources>
    <string name="screen.task.detail.title">Task Detail</string>
    <string name="btn.task.close">Close Task</string>
    <string name="btn.task.close.false_alarm">Mark as False Alarm</string>
    <string name="task.close.reason.hint">Reason (5–256 chars)</string>
    <string name="enum.task_state.CREATED">Created</string>
    <string name="enum.task_state.ACTIVE">Active</string>
    <string name="enum.task_state.RESOLVED">Resolved</string>
    <string name="enum.task_state.FALSE_ALARM">False Alarm</string>
</resources>
```

### 7.5 错误码文案（节选，对齐 API §2）

| code | zh-CN | en-US |
| :--- | :--- | :--- |
| `E_AUTH_4001` | 用户名格式不合法（4–32 位） | Invalid username format (4–32 chars) |
| `E_AUTH_4002` | 密码格式不合法（8–64 位） | Invalid password format (8–64 chars) |
| `E_AUTH_4011` | 用户名或密码错误 | Incorrect username or password |
| `E_AUTH_4091` | 用户名已被占用 | Username already exists |
| `E_GOV_4011` | 登录已过期，请重新登录 | Session expired, please log in again |
| `E_GOV_4030` | 当前账号权限不足 | Insufficient permission |
| `E_GOV_4031` | 账号已被封禁 | Account has been banned |
| `E_GOV_4291` | 请求过于频繁，请 {0}s 后重试 | Too many requests, retry in {0}s |
| `E_TASK_4091` | 该患者当前已有进行中任务 | An active task already exists for this patient |
| `E_TASK_4221` | 误报关闭需填写原因（5–256 字） | False-alarm close requires a reason (5–256 chars) |
| `E_PRO_4221` | 开启围栏需填写完整的中心点与半径 | Fence requires both center point and radius |
| `E_CLUE_4012` | 匿名凭据已失效，请重新扫码 | Anonymous credential invalid, please rescan |
| `E_AI_4091` | 操作对象状态已变更，请重新发起 | State changed, please retry |
| `E_AI_4292` | AI 用量已达上限，请稍后再试 | AI quota exhausted, retry later |
| `E_AI_4293` | 该患者 AI 用量已达上限 | Patient AI quota exhausted |
| 完整错误码映射 | 详见 §24 | — |

### 7.6 RTL 与占位符

1. 所有字符串使用 `{0}` `{1}` 占位符，走 `String.format(locale, ...)`；**禁止**字符串拼接。
2. RTL 语言（阿拉伯语等）未启用，但布局使用 `Modifier.paddingFromBaseline` 与 `start/end` 方向，预留 RTL 适配空间。
3. 单复数走 `PluralRules`（`plurals.xml`）：

```xml
<plurals name="task.clue.count">
    <item quantity="one">%d clue</item>
    <item quantity="other">%d clues</item>
</plurals>
```

### 7.7 i18n 完成定义（DoD）

1. `strings_*.xml` 两语必须**键齐全**（CI 脚本 `scripts/i18n-parity-check.py` 阻断不齐）。
2. 页面新增字符串时同步提交两语；缺失 `en-US` 时 CI 告警而非失败，但上线前必须补齐。
3. 错误码文案表（§24）与 `strings_error.xml` 一一对应。
4. 关键页面（登录、任务详情、AI 对话）在 Paparazzi 截图测试对 zh / en 双语覆盖。

---

## 8. 导航与应用外壳

### 8.1 导航框架

使用 `Navigation-Compose 2.8.x`；单 Activity（`MainActivity`）+ Compose NavHost；导航参数采用类型安全的 `Route` 数据类（`@Serializable`）。

```kotlin
@Serializable sealed interface MhRoute {
    @Serializable data object Splash : MhRoute
    @Serializable data object AuthLogin : MhRoute
    @Serializable data object AuthRegister : MhRoute
    @Serializable data object AuthPasswordReset : MhRoute

    @Serializable data object Home : MhRoute                       // 工作台（Tab 根）
    @Serializable data object MapTab : MhRoute                     // 地图（Tab 根）
    @Serializable data object NotificationTab : MhRoute            // 消息中心（Tab 根）
    @Serializable data object MeTab : MhRoute                      // 我的（Tab 根）

    @Serializable data class  PatientDetail(val patientId: String) : MhRoute
    @Serializable data class  PatientEdit(val patientId: String?)  : MhRoute   // null = 新建
    @Serializable data class  PatientFence(val patientId: String)  : MhRoute
    @Serializable data class  GuardianHub(val patientId: String)   : MhRoute
    @Serializable data class  TagManager(val patientId: String)    : MhRoute

    @Serializable data class  TaskDetail(val taskId: String)       : MhRoute
    @Serializable data class  TaskMap(val taskId: String)          : MhRoute
    @Serializable data class  TaskCreate(val patientId: String)    : MhRoute
    @Serializable data class  TaskClose(val taskId: String)        : MhRoute
    @Serializable data class  ClueDetail(val clueId: String)       : MhRoute

    @Serializable data object OrderList : MhRoute
    @Serializable data class  OrderDetail(val orderId: String)     : MhRoute
    @Serializable data object OrderCreate : MhRoute

    @Serializable data object AiSessionList : MhRoute
    @Serializable data class  AiChat(val sessionId: String)        : MhRoute
    @Serializable data class  AiMemoryNotes(val patientId: String) : MhRoute

    @Serializable data class  PublicScan(val resourceToken: String): MhRoute
    @Serializable data class  PublicReport(val mode: String, val anonymousToken: String): MhRoute
    @Serializable data object PublicManualEntry : MhRoute
}
```

### 8.2 底部导航（Tab）

四 Tab 分别为**首页 / 地图 / 通知 / 我的**；Tab 标签必须走 i18n；未读数走实时订阅（§11）。

```
┌────────────────────────────────────────────┐
│ [首页 Home]  [地图 Map]  [通知 ●]  [我的]      │
└────────────────────────────────────────────┘
```

| Tab | 图标 | 根路由 | 徽标 |
| :--- | :--- | :--- | :--- |
| 首页 | `Home` | `Home` | 无 |
| 地图 | `Map` | `MapTab`（默认展示主患者位置） | 无 |
| 通知 | `Notifications` | `NotificationTab` | 未读数（> 99 显示 `99+`） |
| 我的 | `Person` | `MeTab` | 有提醒时显小红点 |

**大字易读模式**下：Tab 图标 `32dp`、文字 `16sp`；不做 Badge 外文字省略，未读数独立一行。

### 8.3 深链接（Deep Link）

推送跳转 / 扫码 / 邮件回跳统一走深链：

| Scheme | 示例 | 处理 |
| :--- | :--- | :--- |
| `mshj://task/{task_id}` | `mshj://task/8848` | 推送点击 → `TaskDetail` |
| `mshj://notification` | `mshj://notification` | 推送点击 → `NotificationTab` |
| `mshj://ai/session/{session_id}` | `mshj://ai/session/sess_xx` | LUI 续聊 |
| `https://r.mshj.example.com/{token}` | 扫码落地 | App Links 拦截 → `PublicScan` |
| `mshj://auth/reset?token=...` | 邮件链接 | → `AuthPasswordReset` |

**约束**：

1. 深链跳转前必须执行"登录态校验 + 角色校验"：未登录则重定向 `AuthLogin` 并在登录后回跳。
2. 非 `FAMILY` 角色（`ADMIN` / `SUPER_ADMIN`）点击深链时拦截并提示"请前往 Web 管理端"。
3. `task_id` / `session_id` 参数通过 Room 或接口反查校验归属；归属不匹配（`E_TASK_4030` / `E_AI_4033`）时显示 403 页。

### 8.4 应用外壳（AppShell）

```
┌─────────────────────────────────────┐
│ StatusBar                            │
├─────────────────────────────────────┤
│ TopAppBar（页面标题 / 返回 / 动作）    │
├─────────────────────────────────────┤
│                                     │
│ NavHost Content                     │
│                                     │
├─────────────────────────────────────┤
│ BottomNavigationBar（4 Tab，仅根页） │
└─────────────────────────────────────┘
```

二级页面隐藏底部 Tab；Bottom Sheet / Dialog 跟随 Material 3 默认行为；全屏地图页面通过 `WindowInsets` 进入沉浸模式。

### 8.5 NavGraph 装配（示意）

```kotlin
@Composable
fun MhNavHost(navController: NavHostController, startDestination: MhRoute) {
    NavHost(navController, startDestination = startDestination) {
        composable<MhRoute.Splash>      { SplashScreen(onReady = { navController.navigate(MhRoute.Home) { popUpTo(0) } }) }
        composable<MhRoute.AuthLogin>   { AuthLoginScreen(navController) }
        // ……
        composable<MhRoute.TaskDetail>  { backStack ->
            val args = backStack.toRoute<MhRoute.TaskDetail>()
            TaskDetailScreen(taskId = args.taskId, navController)
        }
    }
}
```

### 8.6 返回栈规则

1. `Splash → Home` 后 `popUpTo(0)` 清栈。
2. Tab 之间切换保留状态（`saveState = true, restoreState = true`）。
3. 二级页回退默认弹出上一级；任务关闭成功后需 `popBackStack(MhRoute.TaskDetail, inclusive = false)` 回到详情并触发刷新。
4. AI 对话页 `AiChat` 支持后台最小化，不走 `Activity.finish`；回前台读取本地持久化的 `session_id`，继续原 SSE 连接（基线未提供单会话 GET，上下文由服务端通过 session_id 自动关联）。

---

## 9. 分层、MVI 与状态管理

### 9.1 分层模型（Clean Architecture + UDF）

```
UI (Compose Screens) ─▶ ViewModel (MVI) ─▶ UseCase ─▶ Repository (interface)
                                                          ▲
                                                          │
                              Data (RepositoryImpl ─▶ Remote/Local/Realtime)
```

| 层 | 职责 | 数据类型 |
| :--- | :--- | :--- |
| Network DTO | 严格对齐后端字段（snake_case） | `@Serializable data class TaskDto(val task_id: String, …)` |
| Domain Model | 业务语义（camelCase），类型安全枚举 | `data class Task(val id: String, val status: TaskState, …)` |
| UiState | 展示适配，不可变 | `data class TaskDetailUiState(val task: Task?, val loading: Boolean, …)` |
| Entity | Room 存储 | `@Entity data class TaskEntity(val taskId: String, …)` |

**映射规则**：

1. DTO ↔ Domain 映射集中在 `data/<domain>/mapper/` 目录，单向函数（`TaskDto.toDomain()`）。
2. Domain → UiState 映射发生在 ViewModel 或 UseCase；**禁止** UI 直接持有 DTO。
3. 枚举兜底：所有服务端枚举映射必须有 `UNKNOWN` 分支：

```kotlin
enum class TaskState { CREATED, ACTIVE, RESOLVED, FALSE_ALARM, SUSTAINED, UNKNOWN }
fun String.toTaskState() = runCatching { TaskState.valueOf(this) }.getOrDefault(TaskState.UNKNOWN)
```

### 9.2 MVI（UiState / UiIntent / UiEffect）

```kotlin
data class TaskDetailUiState(
    val loading: Boolean = true,
    val task: Task? = null,
    val timeline: List<TaskEvent> = emptyList(),
    val clues: List<Clue> = emptyList(),
    val alerts: List<Alert> = emptyList(),
    val error: DomainException? = null,
    val version: Long = 0L,               // CAS / event-version 守卫
    val canClose: Boolean = false,        // 客户端可操作性投影
    val canMarkSustained: Boolean = false,
)

sealed interface TaskDetailIntent {
    data object LoadInitial : TaskDetailIntent
    data object Refresh : TaskDetailIntent
    data class  ApplyEvent(val event: TaskEvent) : TaskDetailIntent
    data object ClickClose : TaskDetailIntent
    data object ClickSustained : TaskDetailIntent
}

sealed interface TaskDetailEffect {
    data class  NavigateTo(val route: MhRoute) : TaskDetailEffect
    data class  ShowToast(val msgRes: Int, val args: List<Any> = emptyList()) : TaskDetailEffect
    data class  ShowError(val exception: DomainException) : TaskDetailEffect
}
```

**硬约束**：

1. `UiState` 必须为 `data class` + 不可变；
2. `ViewModel` 只能通过 `reduce(state, intent): state` 的纯函数更新；禁止多协程并发改同一字段；
3. `UiEffect` 只用于一次性副作用（导航 / Toast / Snackbar），走 `Channel` + `receiveAsFlow`，订阅方用 `collectLatest` 消费；
4. 任何 WS 事件先进入 `Intent.ApplyEvent`，经 `reduce` 更新状态；**禁止**在 UI 层直接消费 WS。

### 9.3 ViewModel 模板

```kotlin
@HiltViewModel
class TaskDetailViewModel @Inject constructor(
    savedState: SavedStateHandle,
    private val observeTask: ObserveTaskDetailUseCase,
    private val refreshTask: RefreshTaskDetailUseCase,
    private val closeTask:   CloseTaskUseCase,
) : ViewModel() {

    private val taskId: String = checkNotNull(savedState["task_id"])
    private val _state = MutableStateFlow(TaskDetailUiState())
    val state: StateFlow<TaskDetailUiState> = _state.asStateFlow()

    private val _effect = Channel<TaskDetailEffect>(Channel.BUFFERED)
    val effect: Flow<TaskDetailEffect> = _effect.receiveAsFlow()

    init { dispatch(TaskDetailIntent.LoadInitial) }

    fun dispatch(intent: TaskDetailIntent) = viewModelScope.launch {
        _state.update { reduce(it, intent) }
        when (intent) {
            TaskDetailIntent.LoadInitial -> observe()
            TaskDetailIntent.Refresh     -> refresh()
            TaskDetailIntent.ClickClose  -> _effect.send(TaskDetailEffect.NavigateTo(MhRoute.TaskClose(taskId)))
            is TaskDetailIntent.ApplyEvent -> Unit
            TaskDetailIntent.ClickSustained -> markSustained()
        }
    }

    private fun observe() = viewModelScope.launch {
        observeTask(taskId).collect { snap ->
            _state.update { it.copy(loading = false, task = snap.task, timeline = snap.events, /*…*/, version = snap.version) }
        }
    }
    // ……
}
```

### 9.4 状态权威（HC-02）

1. `TaskState` / `TagState` / `ClueReviewState` / `OrderState` / `GuardianTransferState` / `InvitationState` 的**当前值**完全来自服务端响应或 WS 事件。
2. ViewModel 只保存"服务端快照 + 本地派生投影"；`canClose` / `canMarkSustained` 等"可操作性"字段是基于服务端 `status + version + user.role` 的**纯函数派生**（§9.5）。
3. 乐观更新**仅**允许用于：通知已读（单条）、AI 记忆笔记本地草稿；失败必须回滚。

### 9.5 客户端状态机守卫矩阵（可操作性投影）

#### 9.5.1 任务关闭（对齐 API §3.1.2 + SRS FR-TASK-003/004）

| 当前状态 | 操作 | 是否允许 | 客户端守卫 |
| :--- | :--- | :---: | :--- |
| `CREATED` / `ACTIVE` | 关闭为 `RESOLVED` | 是 | 按钮启用；成功后刷新详情与时间线 |
| `CREATED` / `ACTIVE` | 关闭为 `FALSE_ALARM` | 条件允许 | `reason` 必填 5–256 字；校验通过才可提交 |
| `SUSTAINED`（长期维持） | 关闭为 `RESOLVED` | 是 | 按钮启用 |
| `SUSTAINED` | 关闭为 `FALSE_ALARM` | 否 | 按钮置灰，提示 `E_TASK_4093` 文案 |
| `RESOLVED` / `FALSE_ALARM` | 任意关闭 | 否 | 按钮隐藏；展示终态 |

#### 9.5.2 主监护转移（对齐 API §3.3.8 ~ §3.3.10 + SRS FR-PRO-005/006 + BR-006）

| 转移状态 | 发起方可见动作 | 目标受方可见动作 | 约束 |
| :--- | :--- | :--- | :--- |
| `PENDING_CONFIRM` | 撤销（`E_PRO_4034` 兜底） | 确认 / 拒绝（带原因） | 仅目标受方可确认；非目标方隐藏按钮 |
| `ACCEPTED` | 无 | 无 | 展示完成态 |
| `REJECTED` | 可重新发起 | 无 | 必须展示拒绝原因 |
| `CANCELLED` | 可重新发起 | 无 | 展示撤销人 / 时间 |
| `EXPIRED` | 可重新发起 | 无 | 展示超时失效 |

#### 9.5.3 标签状态（对齐 DBD `tag.state` + API §3.4.6/§3.4.7）

| 当前状态 | 绑定 | 挂失 | 作废 | 展示 |
| :--- | :---: | :---: | :---: | :--- |
| `UNBOUND` | 是 | 否 | 否 | 「未绑定」|
| `ALLOCATED` | 是 | 否 | 否 | 「已分配，待绑定」 |
| `BOUND` | 否 | 是 | 是 | 「已绑定」+ 二维码入口 |
| `LOST` | 否 | 否 | 是 | 「挂失中」|
| `VOID` | 否 | 否 | 否 | 「已作废」终态，仅只读 |

#### 9.5.4 物资工单（对齐 API §3.4.1 ~ §3.4.5）

家属仅可执行 `CREATE` / `CANCEL` / `RECEIVE`；`APPROVE` / `SHIP` / `RESOLVE_EXCEPTION` 为 Web 管理端专属。

| 状态 | 家属动作 |
| :--- | :--- |
| `SUBMITTED` | 取消 |
| `APPROVED` / `SHIPPING` | 仅查看，不可操作 |
| `SHIPPED` | 确认签收（`RECEIVE`） |
| `RECEIVED` / `CANCELLED` / `REJECTED` | 终态，只读 |
| `EXCEPTION` | 联系客服（跳 AI 助手或电话） |

### 9.6 事件一致性（对齐 Outbox 语义）

1. 去重键：优先 `event_id`；缺失时 `type + aggregate_id + version`。
2. 版本守卫：仅当 `incoming.version > local.version` 才 `apply`。
3. 空洞检测：若 `incoming.version > local.version + 1` → 触发 `fetchSnapshot()`（全量拉取）或增量 `after_version=local.version`。
4. 应用失败：不覆盖本地快照，记录失败 + 触发兜底拉取。
5. 前后台切换：从后台回前台必须 `fetchIncrementalSince(local.version)`。

```kotlin
fun handleEvent(local: Long, e: TaskEvent): ConsumeAction = when {
    dedupStore.contains(e.eventId) -> ConsumeAction.Ignore
    e.version <= local             -> ConsumeAction.Ignore
    e.version > local + 1          -> ConsumeAction.FetchSnapshot
    else                            -> ConsumeAction.Apply
}
```

---

## 10. 网络层（Retrofit + OkHttp + 拦截器）

### 10.1 OkHttp Client 构建

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {

    @Provides @Singleton fun okHttpClient(
        auth: AuthInterceptor,
        trace: TraceIdInterceptor,
        requestId: RequestIdInterceptor,
        anonymous: AnonymousTokenInterceptor,
        logger: HttpLoggingInterceptor,
    ): OkHttpClient = OkHttpClient.Builder()
        .connectTimeout(15, TimeUnit.SECONDS)
        .readTimeout(30, TimeUnit.SECONDS)
        .writeTimeout(30, TimeUnit.SECONDS)
        .callTimeout(60, TimeUnit.SECONDS)
        .addInterceptor(trace)             // 第一道：注入 X-Trace-Id
        .addInterceptor(requestId)         // 第二道：写方法注入 X-Request-Id
        .addInterceptor(anonymous)         // 第三道：公开接口注入 X-Anonymous-Token
        .addInterceptor(auth)              // 第四道：JWT 注入 + 401 刷新
        .addInterceptor(ReservedHeaderGuardInterceptor())  // HC-08 保护
        .apply { if (BuildConfig.DEBUG) addNetworkInterceptor(logger) }
        .certificatePinner(MhCertificatePinner)   // §25
        .build()

    @Provides @Singleton fun retrofit(client: OkHttpClient, json: Json): Retrofit = Retrofit.Builder()
        .baseUrl(BuildConfig.API_BASE)
        .client(client)
        .addCallAdapterFactory(ApiResponseCallAdapterFactory(json))
        .addConverterFactory(json.asConverterFactory("application/json".toMediaType()))
        .build()

    @Provides @Singleton fun json(): Json = Json {
        ignoreUnknownKeys = true
        encodeDefaults = true
        explicitNulls = false
        classDiscriminator = "event_type"
    }
}
```

### 10.2 拦截器逐条

#### 10.2.1 `TraceIdInterceptor`

```kotlin
class TraceIdInterceptor @Inject constructor(private val traceStore: TraceIdStore) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val req = chain.request()
        val upstream = traceStore.peekUpstreamOrNull()   // 深链 / 推送透传
        val traceId = req.header("X-Trace-Id") ?: upstream ?: "trc_" + Uuid.v4().replace("-", "")
        return chain.proceed(req.newBuilder().header("X-Trace-Id", traceId).build())
            .also { resp -> Timber.tag("NET").d("trace=%s %s %d", traceId, req.url.encodedPath, resp.code) }
    }
}
```

#### 10.2.2 `RequestIdInterceptor`（HC-03）

```kotlin
class RequestIdInterceptor : Interceptor {
    private val writeMethods = setOf("POST", "PUT", "PATCH", "DELETE")
    override fun intercept(chain: Interceptor.Chain): Response {
        val req = chain.request()
        if (req.method !in writeMethods) return chain.proceed(req)
        // 允许上游透传（离线重放复用）
        val existing = req.header("X-Request-Id")
        val rid = existing ?: ("req_" + Uuid.v4().replace("-", ""))
        require(rid.matches(Regex("^[A-Za-z0-9-]{16,64}$"))) { "Illegal X-Request-Id: $rid" }
        return chain.proceed(req.newBuilder().header("X-Request-Id", rid).build())
    }
}
```

#### 10.2.3 `AuthInterceptor`

```kotlin
class AuthInterceptor @Inject constructor(
    private val tokenStore: AuthTokenStore,
    private val refresher: Lazy<TokenRefresher>,
) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val req = chain.request()
        // 公开接口不注入 JWT
        if (req.url.encodedPath.startsWith("/api/v1/public") ||
            req.url.encodedPath.startsWith("/r/") ||
            req.url.encodedPath.startsWith("/api/v1/auth/")) return chain.proceed(req)

        val token = tokenStore.access()
        val authed = if (token != null) req.newBuilder().header("Authorization", "Bearer $token").build() else req
        val resp = chain.proceed(authed)
        return if (resp.code == 401) {
            val peek = resp.peekBody(Long.MAX_VALUE).string()
            if (peek.contains("E_GOV_4011") || peek.contains("E_AUTH_4011")) {
                resp.close()
                runBlocking { refresher.get().refreshOnce() }?.let { newTok ->
                    chain.proceed(req.newBuilder().header("Authorization", "Bearer $newTok").build())
                } ?: resp
            } else resp
        } else resp
    }
}
```

#### 10.2.4 `AnonymousTokenInterceptor`

```kotlin
class AnonymousTokenInterceptor @Inject constructor(private val store: AnonymousTokenStore) : Interceptor {
    override fun intercept(chain: Interceptor.Chain): Response {
        val req = chain.request()
        val path = req.url.encodedPath
        val needAnon = path.startsWith("/api/v1/public") || path.startsWith("/api/v1/clues/report")
        return if (needAnon && store.token() != null)
            chain.proceed(req.newBuilder().header("X-Anonymous-Token", store.token()!!).build())
        else chain.proceed(req)
    }
}
```

#### 10.2.5 `ReservedHeaderGuardInterceptor`（HC-08）

```kotlin
class ReservedHeaderGuardInterceptor : Interceptor {
    private val forbidden = setOf("X-User-Id", "X-User-Role")
    override fun intercept(chain: Interceptor.Chain): Response {
        val req = chain.request()
        forbidden.forEach { check(req.header(it) == null) { "Forbidden reserved header: $it" } }
        return chain.proceed(req)
    }
}
```

### 10.3 统一响应适配（`ApiResponseCallAdapter`）

后端统一外壳 `{ code, message, trace_id, data }`（API §1.4）。客户端 `CallAdapter` 统一解包：

```kotlin
sealed interface ApiResult<out T> {
    data class Success<T>(val data: T, val trace: String) : ApiResult<T>
    data class Failure(val exception: DomainException) : ApiResult<Nothing>
}

data class DomainException(
    val code: String,
    override val message: String,
    val trace: String,
    val http: Int,
    val retryAfterSeconds: Int? = null,
    val cause: Throwable? = null,
) : RuntimeException(message)
```

**解包规则（与 §2.3 一致）**：

1. HTTP 2xx + `code == "ok"` → `Success(data, trace)`。
2. HTTP 2xx + `code` 以 `E_` 开头 → `Failure(DomainException(code, message, trace, 200))`。
3. HTTP 401 → `AuthInterceptor` 先尝试刷新；失败上抛 `DomainException(E_GOV_4011, …, 401)`，`AuthSessionManager` 订阅并执行 `invalidate()`。
4. HTTP 403 → `DomainException(code, …, 403)`。
5. HTTP 429 → 读取 `Retry-After` / `X-RateLimit-Reset` 填入 `retryAfterSeconds`；UI 展示倒计时。
6. HTTP 5xx / 网络异常 / 反序列化异常 → `DomainException(code="E_NET_UNKNOWN", http = resp.code, cause = …)`。

### 10.4 Repository 模板

```kotlin
class TaskRepositoryImpl @Inject constructor(
    private val api: TaskApi,
    private val dao: TaskDao,
    @IoDispatcher private val io: CoroutineDispatcher,
) : TaskRepository {

    override fun observeTaskDetail(taskId: String): Flow<TaskSnapshot> = dao.observe(taskId)
        .map { it.toDomain() }
        .onStart { refreshTaskDetail(taskId) }        // 静默并发拉远端
        .flowOn(io)

    override suspend fun refreshTaskDetail(taskId: String) = withContext(io) {
        when (val r = api.getTaskFull(taskId)) {
            is ApiResult.Success -> dao.upsert(r.data.toEntity())
            is ApiResult.Failure -> if (r.exception.code !in retryableCodes) throw r.exception
        }
    }
}
```

### 10.5 Retrofit API 接口（以 Task 为例）

```kotlin
interface TaskApi {

    @POST("api/v1/rescue/tasks")
    suspend fun create(@Body body: CreateTaskReq): ApiResult<CreateTaskResp>

    @POST("api/v1/rescue/tasks/{task_id}/close")
    suspend fun close(
        @Path("task_id") taskId: String,
        @Body body: CloseTaskReq,
    ): ApiResult<CloseTaskResp>

    @GET("api/v1/rescue/tasks/{task_id}/snapshot")
    suspend fun snapshot(@Path("task_id") taskId: String): ApiResult<TaskSnapshotDto>

    @POST("api/v1/rescue/tasks/{task_id}/sustained")
    suspend fun markSustained(
        @Path("task_id") taskId: String,
        @Body body: SustainedReq,
    ): ApiResult<SustainedResp>

    @GET("api/v1/rescue/tasks")
    suspend fun list(
        @Query("status") status: String? = null,
        @Query("cursor") cursor: String? = null,
        @Query("page_size") pageSize: Int = 20,
    ): ApiResult<CursorPage<TaskDto>>

    @GET("api/v1/rescue/tasks/{task_id}/full")     // BFF
    suspend fun getTaskFull(@Path("task_id") taskId: String): ApiResult<TaskFullDto>

    @GET("api/v1/rescue/tasks/{task_id}/trajectory/latest")
    suspend fun trajectory(
        @Path("task_id") taskId: String,
        @Query("after_version") afterVersion: Long? = null,
    ): ApiResult<TrajectoryDto>
}
```

### 10.6 限流与退避

| 响应 | 客户端行为 |
| :--- | :--- |
| 429 + `Retry-After: N` | 禁用触发按钮 `N` 秒 + 倒计时；期间禁止自动重试；到期后允许用户手动重试 |
| 429 + 无 `Retry-After` | 本地指数退避 `delay = min(500ms * 2^n + random(0…300ms), 30s)` |
| 5xx（idempotent 读） | 最多自动重试 2 次，每次退避 1s / 2s |
| 5xx（写） | **不**自动重试；使用方通过 UI 决策；离线写队列保留 `X-Request-Id` |
| 网络错误 | 首次立即重试 1 次；再失败 → 展示"网络异常，请检查连接"+ `trace_id` 前 8 位 |

### 10.7 契约一致性联调前必过

| # | 检查 | 工具 |
| :---: | :--- | :--- |
| 1 | 所有 ID 字段按 String 序列化 | MockWebServer + 断言 |
| 2 | 写请求必带 `X-Request-Id` 且符合正则 | `EventListener` 断言 |
| 3 | 所有请求必带 `X-Trace-Id` | 同上 |
| 4 | 禁止发送 `X-User-Id` / `X-User-Role` | `ReservedHeaderGuardInterceptor` + 测试 |
| 5 | 401 自动刷新仅一次；二次 401 必清会话跳登录 | `TokenRefresher` 单测 |
| 6 | 429 按 Retry-After 执行 | Unit Test |
| 7 | 公开接口**不**带 `Authorization` | 测试断言 |
| 8 | 登录态接口**不**带 `X-Anonymous-Token` | 测试断言 |

---

## 11. 实时通道：WebSocket / SSE / 推送

### 11.1 通道选型与用途（对齐 SADD HC-05 + API §4）

| 通道 | 用途 | 触发 |
| :--- | :--- | :--- |
| **WebSocket**（`/ws/v1/stream`） | 定向事件流：任务状态 / 线索到达 / 监护转移 / 物资工单 / 通知 / 标签状态变更 | 登录后应用前台 |
| **SSE**（`/api/v1/ai/sessions/{id}/messages`） | AI 流式响应（delta / tool_intent / final） | AI 对话页打开会话 |
| **FCM / 厂商推送** | 系统级离线通知；深链回跳 | 后台 / 被杀进程 |
| **WorkManager 补洞** | 前后台切换 / 长连接断开后增量补齐 | 回前台 / 重连成功 |

### 11.2 WebSocket 鉴权与建连

```kotlin
class MhWebSocketClient @Inject constructor(
    private val client: OkHttpClient,
    private val ticketApi: WsTicketApi,
    private val dispatcher: RealtimeEventDispatcher,
    private val stateStore: LocalVersionStore,
) {
    private val backoff = ExponentialBackoff(baseMs = 1_000L, maxMs = 30_000L)

    suspend fun connect() {
        while (true) {
            runCatching {
                val ticket = ticketApi.issue(WsTicketReq(capabilities = listOf("task", "clue", "notify", "guardian", "tag", "material"))).unwrap()
                val req = Request.Builder()
                    .url(BuildConfig.WS_BASE + "/ws/v1/stream?ticket=${ticket.ticket}")
                    .header("Sec-WebSocket-Protocol", "mshj.v1")
                    .build()
                awaitSocket(req)       // 挂起直到 onClosed / onFailure
            }.onFailure { Timber.w(it, "[WS] disconnected") }
            delay(backoff.next())
        }
    }

    private suspend fun awaitSocket(req: Request) = suspendCancellableCoroutine<Unit> { cont ->
        val socket = client.newWebSocket(req, object : WebSocketListener() {
            override fun onOpen(ws: WebSocket, resp: Response) { backoff.reset(); sendResume(ws) }
            override fun onMessage(ws: WebSocket, text: String) { dispatcher.onMessage(text) }
            override fun onClosed(ws: WebSocket, code: Int, reason: String) { cont.resume(Unit) {} }
            override fun onFailure(ws: WebSocket, t: Throwable, resp: Response?) { cont.resume(Unit) {} }
        })
        cont.invokeOnCancellation { socket.close(1000, "cancel") }
    }

    private fun sendResume(ws: WebSocket) {
        val resume = ResumeFrame(
            last_event_ids = mapOf(
                "task" to stateStore.lastEventId("task"),
                "clue" to stateStore.lastEventId("clue"),
                "notify" to stateStore.lastEventId("notify"),
            ),
            after_versions = mapOf(
                "task" to stateStore.lastVersion("task"),
                "clue" to stateStore.lastVersion("clue"),
            ),
        )
        ws.send(Json.encodeToString(resume))
    }
}
```

**约束**：

1. `ticket` 单次使用、1 分钟过期；建连失败（401 / 403）必须回到 `AuthInterceptor` 刷新路径。
2. 心跳：服务端下发 `{"type":"ping"}` 每 25s，客户端在 30s 内无任何帧 → 主动 `close(4000, "idle")` 触发重连。
3. 订阅仅当前登录用户 `user_id` 的事件；**严禁**客户端上送他人 `user_id` 作为过滤参数。
4. 订阅能力清单由 `capabilities` 声明，避免订阅不需要的分域事件。

### 11.3 事件分发（`RealtimeEventDispatcher`）

```kotlin
@Singleton
class RealtimeEventDispatcher @Inject constructor(
    private val json: Json,
    private val sinks: Map<String, @JvmSuppressWildcards MutableSharedFlow<RealtimeEvent>>,
    private val dedup: EventDedupStore,
    private val versionStore: LocalVersionStore,
) {
    fun onMessage(text: String) {
        val env = json.decodeFromString<EventEnvelope>(text)
        if (!dedup.tryAdd(env.event_id)) return
        val local = versionStore.lastVersion(env.domain)
        when {
            env.version <= local        -> return
            env.version > local + 1     -> triggerSnapshot(env.domain, local)
            else                        -> {
                sinks[env.domain]?.tryEmit(env.toRealtimeEvent())
                versionStore.update(env.domain, env.version, env.event_id)
            }
        }
    }
}
```

| 领域 key | Sink 消费方 | 事件类型 |
| :--- | :--- | :--- |
| `task` | `TaskDetailVM` / `HomeVM` | `task.created` / `task.progress` / `task.closed` / `task.sustained` |
| `clue` | `TaskDetailVM` / `ClueDetailVM` | `clue.arrived` / `clue.reviewed` / `clue.rejected` |
| `notify` | `NotificationTabVM` / AppShell 未读数 | `notify.new` / `notify.read` |
| `guardian` | `GuardianHubVM` | `guardian.invited` / `guardian.confirmed` / `guardian.transfer.*` |
| `tag` | `TagManagerVM` | `tag.bound` / `tag.lost` / `tag.void` |
| `material` | `OrderListVM` / `OrderDetailVM` | `order.state.changed` |

### 11.4 SSE（AI 对话流）

```kotlin
class AiSseClient @Inject constructor(private val factory: EventSources.Factory) {
    fun stream(sessionId: String, cursor: String?): Flow<AiSseEvent> = callbackFlow {
        val req = Request.Builder()
            .url(BuildConfig.API_BASE + "api/v1/ai/sessions/$sessionId/messages?stream=true" + (cursor?.let { "&cursor=$it" } ?: ""))
            .header("Accept", "text/event-stream")
            .build()
        val source = factory.newEventSource(req, object : EventSourceListener() {
            override fun onEvent(es: EventSource, id: String?, type: String?, data: String) {
                trySend(AiSseEvent.parse(type, data, id))
            }
            override fun onClosed(es: EventSource) { close() }
            override fun onFailure(es: EventSource, t: Throwable?, r: Response?) { close(t) }
        })
        awaitClose { source.cancel() }
    }
}
```

事件类型：`message.delta` / `message.tool_intent` / `message.final` / `message.error`（对齐 API §3.7）；断线重连携带 `Last-Event-ID`。

### 11.5 推送（FCM + 厂商通道）

| 用途 | 通道 | 负载字段 | 深链 |
| :--- | :--- | :--- | :--- |
| 新任务通知 | FCM data message | `{type:"task.created", task_id, trace_id}` | `mshj://task/{task_id}` |
| 新线索到达 | 同上 | `{type:"clue.arrived", task_id, clue_id}` | `mshj://task/{task_id}` |
| 监护邀请 | 同上 | `{type:"guardian.invited", invitation_id}` | `mshj://guardian/{invitation_id}` |
| 工单状态 | 同上 | `{type:"order.state.changed", order_id}` | `mshj://order/{order_id}` |
| 系统消息 | 同上 | `{type:"notify.new"}` | `mshj://notification` |

**客户端处理**：

1. 应用在前台：**忽略**系统通知栏，直接通过 WS 更新 UI；避免重复提示。
2. 应用在后台：走 Notification Channel（`mshj_task` / `mshj_clue` / `mshj_system`）；点击进入深链。
3. Token 注册 / 注销调用 `POST /api/v1/users/me/push-tokens`（API §3.8.5.1）与 `DELETE /api/v1/users/me/push-tokens/{push_token_id}`（API §3.8.5.2）；请求体 `platform=ANDROID_FCM|ANDROID_HMS|ANDROID_MIPUSH` 按设备指纹识别，`device_id` 复用 `Settings.Secure.ANDROID_ID`。
4. 华为 / 小米 / vivo / OPPO 通过 `push-unification` 库分发；不足时回退 FCM；**禁止**回退短信（HC-Channel）。

### 11.6 WorkManager 补洞

| 任务 | 触发 | 行为 |
| :--- | :--- | :--- |
| `RealtimeReconcileWorker` | 网络恢复 / 回前台 / 每 15 min 心跳 | 对 `task` / `clue` / `notify` 调用 `GET …?after_version=<local>` 全量拉齐 |
| `OutboxReplayWorker` | 网络恢复 / 工单 Pending 存在 | 复用原 `X-Request-Id` 重放未成功的写请求（§12.5） |
| `AiMessageReconcileWorker` | 恢复前台 + 存在进行中 SSE | 由 `MhSseClient` 自行重连；对已中断会话调用 `GET /api/v1/ai/sessions/{id}/messages?after_cursor=last_seq` (API §3.8.1.4) 拉取漏接消息；若需主动中止则调 `POST /api/v1/ai/sessions/{id}/messages/{message_id}/cancel`（API §3.8.1.5） |

### 11.7 回前台 / 断网恢复的统一编排

```
onAppForeground()
  ├─ if (!ws.isConnected) ws.connect()
  ├─ scheduleOneTime(RealtimeReconcileWorker)
  ├─ scheduleOneTime(OutboxReplayWorker)  // 若 outbox 非空
  └─ activeSessionId?.let { scheduleOneTime(AiMessageReconcileWorker(it)) }
```

---

## 12. 本地存储

### 12.1 存储分层

| 介质 | 用途 | 示例 |
| :--- | :--- | :--- |
| **Room** | 实体缓存 + 离线读取 + Outbox 队列 | 患者 / 任务 / 线索 / 通知 / 版本号 / 去重表 |
| **DataStore Preferences** | 用户配置、轻量状态 | 语言、主题、大字模式、最后患者 ID、推送 token 缓存 |
| **EncryptedSharedPreferences** | 敏感凭据（access / refresh token） | JWT、匿名 token |
| **File Cache**（`context.cacheDir`） | 图片 / 海报 / 地图瓦片 | Coil DiskCache |
| **File MediaStore** | 扫码拍照原图（分区存储） | `Environment.DIRECTORY_PICTURES/MH` |

### 12.2 Room 架构

```kotlin
@Database(
    entities = [
        PatientEntity::class, TaskEntity::class, ClueEntity::class, TagEntity::class,
        GuardianEntity::class, OrderEntity::class, NotificationEntity::class,
        AiSessionEntity::class, AiMessageEntity::class,
        OutboxEntity::class, EventDedupEntity::class, VersionEntity::class,
    ],
    version = 1,
    exportSchema = true,
)
@TypeConverters(MhTypeConverters::class)
abstract class MhDatabase : RoomDatabase() {
    abstract fun patient(): PatientDao
    abstract fun task(): TaskDao
    abstract fun clue(): ClueDao
    abstract fun tag(): TagDao
    abstract fun guardian(): GuardianDao
    abstract fun order(): OrderDao
    abstract fun notification(): NotificationDao
    abstract fun aiSession(): AiSessionDao
    abstract fun aiMessage(): AiMessageDao
    abstract fun outbox(): OutboxDao
    abstract fun dedup(): EventDedupDao
    abstract fun version(): VersionDao
}
```

### 12.3 关键 Entity 字段（对齐 DBD）

```kotlin
@Entity(tableName = "patient", primaryKeys = ["patient_id"])
data class PatientEntity(
    val patient_id: String,
    val name: String,
    val gender: String,                      // MALE / FEMALE / UNKNOWN
    val birthday: String?,                   // yyyy-MM-dd
    val id_card_masked: String?,
    val avatar_url: String?,
    val appearance_json: String?,            // 服装描述 JSON
    val fence_center_lng: Double?,
    val fence_center_lat: Double?,
    val fence_radius_m: Int?,
    val fence_enabled: Boolean,
    val primary_guardian_user_id: String?,
    val missing_pending: Boolean,
    val updated_at: String,
    val version: Long,
)

@Entity(tableName = "task", primaryKeys = ["task_id"])
data class TaskEntity(
    val task_id: String,
    val patient_id: String,
    val state: String,                       // CREATED / ACTIVE / RESOLVED / FALSE_ALARM / SUSTAINED
    val created_by: String,
    val created_at: String,
    val closed_at: String?,
    val closed_reason: String?,
    val sustained_mode: Boolean,
    val last_known_lng: Double?,
    val last_known_lat: Double?,
    val last_event_at: String?,
    val version: Long,
)

@Entity(tableName = "clue", primaryKeys = ["clue_id"],
    foreignKeys = [ForeignKey(entity = TaskEntity::class, parentColumns = ["task_id"], childColumns = ["task_id"], onDelete = ForeignKey.CASCADE)])
data class ClueEntity(
    val clue_id: String,
    val task_id: String,
    val source: String,                      // FAMILY / PUBLIC_SCAN / PUBLIC_MANUAL
    val review_state: String,                // PENDING / APPROVED / REJECTED
    val photo_url: String?,
    val lng: Double?, val lat: Double?,
    val reported_at: String,
    val reporter_anon_id: String?,
    val note: String?,
    val version: Long,
)

@Entity(tableName = "outbox", primaryKeys = ["request_id"])
data class OutboxEntity(
    val request_id: String,                  // 就是 X-Request-Id
    val endpoint: String,                    // e.g. POST /api/v1/clues/report
    val method: String,
    val body_json: String,
    val created_at: Long,                    // millis
    val retry_count: Int,
    val last_error: String?,
    val status: String,                      // PENDING / SUCCESS / GIVE_UP
)

@Entity(tableName = "event_dedup", primaryKeys = ["event_id"])
data class EventDedupEntity(val event_id: String, val seen_at: Long)

@Entity(tableName = "version", primaryKeys = ["domain"])
data class VersionEntity(
    val domain: String,                      // "task"/"clue"/"notify"/...
    val last_event_id: String?,
    val last_version: Long,
    val last_synced_at: Long,
)
```

### 12.4 迁移策略

1. `exportSchema = true`，将 JSON 纳入 Git，用于 CI 迁移校验。
2. 客户端版本号内含 Schema 版本；大版本升级提供 `fallbackToDestructiveMigration()` 仅用于加密敏感数据未命中的全新安装。
3. **禁用** 生产默认的 `fallbackToDestructiveMigration`；必须编写 `Migration` 类。
4. Migration 测试走 `MigrationTestHelper`，CI 必跑。

### 12.5 Outbox 写请求队列

适用场景：发线索（`POST /api/v1/clues/report`）、创建任务（`POST /api/v1/rescue/tasks`）、接收工单（`POST /api/v1/material/orders/{order_id}/receive`）、AI 意图确认（`POST /api/v1/ai/sessions/{session_id}/intents/{intent_id}/confirm`）、通知标记已读（循环单条）。

```kotlin
suspend fun submitClue(req: CreateClueReq): Result<ClueCreatedResp> {
    val requestId = "req_" + Uuid.v4().replace("-", "")
    outbox.insert(OutboxEntity(requestId, "POST /api/v1/clues/report", "POST", json.encodeToString(req), now(), 0, null, "PENDING"))
    return try {
        val resp = api.submit(req, requestId).unwrap()
        outbox.markSuccess(requestId)
        Result.success(resp)
    } catch (e: DomainException) {
        if (e.http in setOf(0, 408, 500, 502, 503, 504)) {
            WorkManager.getInstance(ctx).enqueueUniqueWork("outbox-replay", KEEP, OneTimeWorkRequestBuilder<OutboxReplayWorker>().build())
            Result.failure(e)
        } else {
            outbox.markGiveUp(requestId, e.code)
            Result.failure(e)
        }
    }
}
```

**幂等保证**：复用原 `request_id` 重放；服务端 1h 内同 `X-Request-Id` 视为幂等（API §1.5）。

### 12.6 DataStore Preferences 定义

```kotlin
object AppPrefsKeys {
    val LANGUAGE        = stringPreferencesKey("language")            // zh-CN / en-US
    val THEME           = stringPreferencesKey("theme")               // SYSTEM / LIGHT / DARK
    val LARGE_MODE      = booleanPreferencesKey("large_mode")         // 大字易读模式
    val LAST_PATIENT_ID = stringPreferencesKey("last_patient_id")
    val REALTIME_ON     = booleanPreferencesKey("realtime_on")        // 远程 kill switch
    val AI_ON           = booleanPreferencesKey("ai_on")
}
```

### 12.7 加密存储（Token）

```kotlin
@Singleton
class AuthTokenStore @Inject constructor(@ApplicationContext ctx: Context) {
    private val prefs: SharedPreferences = EncryptedSharedPreferences.create(
        ctx, "mh_auth",
        MasterKey.Builder(ctx).setKeyScheme(MasterKey.KeyScheme.AES256_GCM).build(),
        EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
        EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM,
    )
    fun access()  = prefs.getString("access", null)
    fun refresh() = prefs.getString("refresh", null)
    fun save(access: String, refresh: String) = prefs.edit { putString("access", access); putString("refresh", refresh) }
    fun clear()   = prefs.edit { clear() }
}
```

### 12.8 清理与登出策略

| 触发 | 清理内容 |
| :--- | :--- |
| 主动登出 | `AuthTokenStore.clear()` + `MhDatabase.clearAllTables()` + 删除 Coil DiskCache |
| 切换账号 | 同上 |
| 被禁用 / 密码变更 / 全局登出 | 同上 + 跳 `AuthLogin` |
| 卸载 / 清数据 | Android 系统处理 |

---

## 13. 权限与隐私合规

### 13.1 运行时权限清单

| 权限 | 用途 | 申请时机 | 拒绝后降级 |
| :--- | :--- | :--- | :--- |
| `ACCESS_FINE_LOCATION` / `ACCESS_COARSE_LOCATION` | 地图、附近线索、任务轨迹 | 打开地图 / 任务详情前 | 地图以上海坐标兜底；任务可继续但展示"未授权精确定位" |
| `ACCESS_BACKGROUND_LOCATION`（API 29+） | 家庭围栏越界后台告警 | 开启围栏时**单独**引导 | 仅前台告警 |
| `CAMERA` | 扫码、拍照上传线索 | 点扫码 / 拍照时 | 提示去系统相册选图 |
| `READ_MEDIA_IMAGES`（API 33+）/ `READ_EXTERNAL_STORAGE` | 相册选图（线索 / 头像） | 点相册时 | 提示去相机拍摄 |
| `POST_NOTIFICATIONS`（API 33+） | 系统通知 | 首次进入首页 | 通知中心内站内提示 |
| `RECORD_AUDIO` | AI 语音输入（可选） | 点麦克风时 | 回退到文字输入 |
| `READ_PHONE_STATE`（可选） | 崩溃归因 | **不申请**，保持合规 | — |

### 13.2 权限申请 UX 流程

```
 触发功能（如打开地图）
        │
        ▼
┌────────────────────────────┐
│ 已授权？                    │──是─▶ 直接使用
└────────────────────────────┘
        │ 否
        ▼
┌────────────────────────────┐
│ 是否首次或 shouldShowRationale │
└────────────────────────────┘
   │首次                │已拒绝且 rationale
   ▼                    ▼
展示说明 Dialog        引导到系统设置
（为什么需要、不给会怎样）  页面，提示完成后回 App
   │
   ▼
调用 ActivityResultContracts.RequestPermission
   │
   ▼
同意 ─▶ 执行功能；拒绝 ─▶ 展示降级 UI + 关闭/去设置按钮
```

统一封装：

```kotlin
@Composable
fun rememberPermissionFlow(perm: String, rationaleRes: Int, onGranted: () -> Unit): () -> Unit {
    val activity = LocalContext.current as ComponentActivity
    var showRationale by remember { mutableStateOf(false) }
    val launcher = rememberLauncherForActivityResult(ActivityResultContracts.RequestPermission()) { granted ->
        if (granted) onGranted() else showRationale = true
    }
    if (showRationale) RationaleDialog(rationaleRes, onConfirm = { showRationale = false; launcher.launch(perm) }, onSettings = { showRationale = false; activity.openAppSettings() }, onDismiss = { showRationale = false })
    return {
        when {
            activity.has(perm)           -> onGranted()
            activity.shouldShowRationale(perm) -> showRationale = true
            else                         -> launcher.launch(perm)
        }
    }
}
```

### 13.3 隐私政策同意（首启）

1. `SplashScreen` 首启**必须**弹出《隐私政策》+《用户协议》弹窗（可滚动阅读）+【同意 / 不同意】；不同意 → `finish()`。
2. 同意前**禁止**初始化任何三方 SDK（高德、FCM、Crashlytics、统计），集中在 `MhApplication.afterConsent()` 调用。
3. 用户可在"我的 → 隐私设置"撤回同意；撤回后清空本地缓存并回到登录页。
4. 《隐私政策》内容引用企业备案版本；本地打包 `assets/privacy_zh.html` / `assets/privacy_en.html`。

### 13.4 分场景降级策略

| 场景 | 完整权限行为 | 降级行为 |
| :--- | :--- | :--- |
| 发线索（拍照 + 定位） | 直接拍照 + 自动记录 GPS | 相册选图 + 手动在地图上标点；若无相册权限提示去设置 |
| 围栏告警 | 后台 + 前台提醒 | 仅前台（需打开 App） |
| 扫码 | 直接开扫 | 提示"请在系统设置开启相机权限" + 跳转设置按钮 |
| 推送通知 | 系统通知栏 + 站内 | 仅站内通知中心 + 角标 |

### 13.5 权限级 UI 约束（HC-Auth 客户端投影）

角色判定来自 `GET /api/v1/users/me` → `role ∈ {FAMILY, ADMIN, SUPER_ADMIN}`；家属端只允许 `FAMILY`。

| 入口 | 显示条件 |
| :--- | :--- |
| 首页"一键求助" | `role == FAMILY` 且当前患者有进行中任务 → 跳详情；无任务 → 跳 TaskCreate |
| 监护管理 → 转移主监护 | `primary_guardian_user_id == me.user_id` |
| 标签挂失 | 该标签 `state == BOUND` 且 `owner_guardian_user_id in guardians(patient)` |
| 物资工单取消 | `order.state == SUBMITTED` 且 `creator_user_id == me.user_id` |
| 物资工单签收 | `order.state == SHIPPED` 且 `receiver_user_id == me.user_id` |
| AI 写操作确认 | 任何 `tool_intent.action ∈ {CLOSE_TASK, UPDATE_PATIENT, …}` 必须展示"待确认"卡片 |

**非 FAMILY 角色**（用户从 Web 迁移而来）登录家属端时：弹窗"您当前为管理员账号，请使用 Web 管理端登录"后跳回登录页。

### 13.6 `AndroidManifest.xml` 必声明权限（节选）

```xml
<uses-permission android:name="android.permission.INTERNET"/>
<uses-permission android:name="android.permission.ACCESS_NETWORK_STATE"/>
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
<uses-permission android:name="android.permission.ACCESS_COARSE_LOCATION"/>
<uses-permission android:name="android.permission.ACCESS_BACKGROUND_LOCATION"/>
<uses-permission android:name="android.permission.CAMERA"/>
<uses-permission android:name="android.permission.READ_MEDIA_IMAGES"/>
<uses-permission android:name="android.permission.POST_NOTIFICATIONS"/>
<uses-permission android:name="android.permission.RECORD_AUDIO" android:required="false"/>
<uses-permission android:name="android.permission.VIBRATE"/>
<uses-permission android:name="android.permission.WAKE_LOCK"/>
<uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION"/>
```

**严禁声明**：`SEND_SMS` / `READ_SMS` / `READ_CONTACTS` / `READ_CALL_LOG`（HC-Channel + 合规）。

---

## 14. 公共 UI 组件库（`core-designsystem`）

### 14.1 组件清单

| 组件 | 用途 | 关键 API |
| :--- | :--- | :--- |
| `MhScaffold` | 统一页面骨架（TopBar + Content + Snackbar + 加载罩） | `title`, `navigation`, `actions`, `content` |
| `MhAppBar` | Top app bar | `title`, `navIcon`, `actions`, `largeText` |
| `MhBottomNavBar` | 四 Tab 底部导航 | `current`, `onSelect`, `badges: Map<Tab,Int>` |
| `MhPrimaryButton` / `MhSecondaryButton` / `MhTextButton` | 按钮三档 | 自动大字模式；最小高度 48dp / 大字 56dp |
| `MhIconButton` | 带 `contentDescription` 强制的 IconButton | 编译期 lint 阻断缺省 |
| `MhTextField` | 表单输入 | 统一错误文案 / 字数统计 / 大字模式字号 |
| `MhDateTimeField` | 日期时间选择 | Locale 感知 |
| `MhLoading` / `MhEmpty` / `MhError` | 三态页 | `MhError` 展示 `trace_id` 前 8 位 |
| `MhDialog` / `MhBottomSheet` | 对话框 / 底部抽屉 | 标题 / 内容 / 双按钮 |
| `MhStatusChip` | 任务 / 标签 / 工单状态标签 | 颜色由 `MhStatusToken` 驱动 |
| `MhAvatar` | 头像（Coil） | 带兜底 `Person` 图标 |
| `MhPermissionButton` | 按钮级权限守卫（隐藏 / 置灰 / 正常） | `visibleWhen`, `enabledWhen`, `disabledReason` |
| `MhTraceBadge` | 调试模式展示 `X-Trace-Id` | `BuildConfig.DEBUG` 才挂载 |
| `MhTimeline` | 任务 / 工单时间线 | 支持大字 |
| `MhMap` | 地图包装（高德） | `markers`, `trajectory`, `fence`, `onClick` |
| `MhCountdownButton` | 限流倒计时按钮 | 自动根据 `retryAfterSeconds` 禁用 |
| `MhVersionGuard` | 乐观并发守卫弹窗 | `E_AI_4091` / `E_PRO_4091` 统一提示"已更新，点此刷新" |

### 14.2 关键组件示例

#### 14.2.1 `MhStatusChip`

```kotlin
@Composable
fun MhStatusChip(state: TaskState) {
    val (labelRes, color) = when (state) {
        TaskState.CREATED      -> R.string.`enum.task_state.CREATED`   to MaterialTheme.colorScheme.tertiary
        TaskState.ACTIVE       -> R.string.`enum.task_state.ACTIVE`    to MaterialTheme.colorScheme.primary
        TaskState.SUSTAINED    -> R.string.`enum.task_state.SUSTAINED` to MaterialTheme.colorScheme.secondary
        TaskState.RESOLVED     -> R.string.`enum.task_state.RESOLVED`  to MhStatusOk
        TaskState.FALSE_ALARM  -> R.string.`enum.task_state.FALSE_ALARM` to MhStatusWarn
        TaskState.UNKNOWN      -> R.string.`enum.task_state.UNKNOWN`   to MaterialTheme.colorScheme.outline
    }
    AssistChip(
        onClick = {},
        label = { Text(stringResource(labelRes)) },
        colors = AssistChipDefaults.assistChipColors(containerColor = color.copy(alpha = 0.15f), labelColor = color),
        modifier = Modifier.semantics { contentDescription = "status: " },
    )
}
```

#### 14.2.2 `MhPermissionButton`

```kotlin
@Composable
fun MhPermissionButton(
    label: String,
    visibleWhen: Boolean = true,
    enabledWhen: Boolean = true,
    disabledReason: String? = null,
    onClick: () -> Unit,
) {
    if (!visibleWhen) return
    val ctx = LocalContext.current
    MhPrimaryButton(
        onClick = {
            if (enabledWhen) onClick()
            else disabledReason?.let { Toast.makeText(ctx, it, Toast.LENGTH_SHORT).show() }
        },
        enabled = true,          // 交由内部 onClick 拦截，保持可触达以便提示
        label = label,
        alpha = if (enabledWhen) 1f else 0.4f,
    )
}
```

#### 14.2.3 `MhCountdownButton`

```kotlin
@Composable
fun MhCountdownButton(
    label: String,
    retryAfterSeconds: Int?,
    onClick: () -> Unit,
) {
    var left by remember(retryAfterSeconds) { mutableStateOf(retryAfterSeconds ?: 0) }
    LaunchedEffect(retryAfterSeconds) {
        while (left > 0) { delay(1_000); left -= 1 }
    }
    MhPrimaryButton(
        onClick = { if (left <= 0) onClick() },
        label = if (left > 0) "$label (${left}s)" else label,
        enabled = left <= 0,
    )
}
```

#### 14.2.4 `MhVersionGuard`

```kotlin
@Composable
fun MhVersionGuardDialog(visible: Boolean, onRefresh: () -> Unit, onDismiss: () -> Unit) {
    if (!visible) return
    MhDialog(
        title = stringResource(R.string.`dialog.version_guard.title`),   // 内容已更新
        message = stringResource(R.string.`dialog.version_guard.message`), // 为了操作准确性，请刷新后重试
        confirmText = stringResource(R.string.`btn.refresh`),
        onConfirm = { onDismiss(); onRefresh() },
        onDismiss = onDismiss,
    )
}
```

触发条件：消费 `DomainException` 时 `code ∈ { E_AI_4091, E_PRO_4091, E_TASK_4094 }`。

#### 14.2.5 `MhTraceBadge`

```kotlin
@Composable
fun MhTraceBadge(trace: String?) {
    if (!BuildConfig.DEBUG || trace.isNullOrBlank()) return
    Surface(
        modifier = Modifier.padding(6.dp),
        shape = RoundedCornerShape(8.dp),
        color = MaterialTheme.colorScheme.tertiaryContainer.copy(alpha = 0.8f),
    ) { Text("trace=${trace.take(8)}…", style = MaterialTheme.typography.labelSmall, modifier = Modifier.padding(6.dp)) }
}
```

### 14.3 组件级编码规范

1. **无障碍强制**：所有可点击元素必须有 `contentDescription`；`MhIconButton` 使用 `@RequireContentDescription` 编译期注解（自研 detekt 规则）阻断缺省。
2. **大字适配**：组件内部使用 `MhTypography.current` 读取当前字号阶梯；**禁止**硬编码 `sp`。
3. **颜色取用**：**禁止**写死 `Color(0xFFxxxxxx)`；必须 `MaterialTheme.colorScheme.*` 或 `MhStatusToken.*`。
4. **尺寸取用**：**禁止**硬编码 `dp`；统一走 `MhSpacing` / `MhDimens`。
5. **Preview 双态**：Light / Dark × 中文 / 英文 × 标准 / 大字 共 8 套 Preview。
6. **截图测试**：每个组件至少一个 Paparazzi 截图测试；CI 必跑。

### 14.4 主题感知入口

```kotlin
@Composable
fun MhRoot(content: @Composable () -> Unit) {
    val prefs by appPrefsFlow.collectAsStateWithLifecycle()
    MhTheme(
        darkTheme = when (prefs.theme) { "DARK" -> true; "LIGHT" -> false; else -> isSystemInDarkTheme() },
        largeText = prefs.largeMode,
    ) {
        MhTypographyProvider(largeText = prefs.largeMode, content = content)
    }
}
```

---

## 15. 页面总清单

> 页面编号规则：`MH-<域>-<序号>`；域代码与 §4.2 一致。所有路由须在 §8.1 `MhRoute` 中声明；所有 API 必与 `v2/API_V2.0.md` 逐字对齐。**非 FAMILY 专属接口（`/admin/**`、force-transfer、DEAD replay 等）不在本手册范围**。

### 15.1 全量页面矩阵

| 编号 | 页面名 | 路由 | 所属模块 | 运行时权限 | 关联 API（主） | 需求锚点（SRS / LLD） |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| MH-SPL-01 | 启动页 + 隐私同意 | `Splash` | `feature-splash` | — | — | NFR-SEC-004 |
| MH-AUTH-01 | 登录 | `AuthLogin` | `feature-auth` | — | `POST /api/v1/auth/login` | FR-GOV-001 |
| MH-AUTH-02 | 注册 | `AuthRegister` | `feature-auth` | — | `POST /api/v1/auth/register` | FR-GOV-001 |
| MH-AUTH-03 | 找回密码（发送邮件） | `AuthPasswordReset` | `feature-auth` | — | `POST /api/v1/auth/password-reset/request` | FR-GOV-002（HC-06 邮件通道） |
| MH-AUTH-04 | 重置密码（邮件回跳） | `AuthPasswordResetConfirm` | `feature-auth` | — | `POST /api/v1/auth/password-reset/confirm` | FR-GOV-002 |
| MH-AUTH-05 | 修改密码 | `MeChangePwd` | `feature-me` | — | `PUT /api/v1/users/me/password` | FR-GOV-003 |
| MH-HOME-01 | 家庭工作台 | `Home` | `feature-home` | 通知、（可选）定位 | 前端聚合：`GET /api/v1/patients` + `GET /api/v1/rescue/tasks?state=ACTIVE,SUSTAINED` + `GET /api/v1/notifications/inbox?page_size=3`（基线未提供 `/home/summary` BFF） | FR-PRO-008、FR-TASK-005 |
| MH-PAT-01 | 患者档案详情 | `PatientDetail` | `feature-patient` | — | `GET /api/v1/patients/{patient_id}`（响应内嵌 `guardians[] / appearance / fence / tags[]`） | FR-PRO-001/003 |
| MH-PAT-02 | 患者档案新建 / 编辑 | `PatientEdit` | `feature-patient` | 相机 / 相册 | `POST /api/v1/patients`、`PUT /api/v1/patients/{patient_id}/profile`、`PUT /api/v1/patients/{patient_id}/appearance` | FR-PRO-001/002 |
| MH-PAT-03 | 围栏设置 | `PatientFence` | `feature-patient` | 精确定位 | `PUT /api/v1/patients/{patient_id}/fence` | FR-PRO-003 |
| MH-PAT-04 | 误记未寻回确认 | `PatientMissingPending` | `feature-patient` | — | `POST /api/v1/patients/{patient_id}/missing-pending/confirm`（`action=CONFIRM_MISSING` 发起任务 / `action=CONFIRM_SAFE` 撤销） | FR-PRO-004 |
| MH-PAT-05 | 档案注销 | `PatientDelete` | `feature-patient` | — | `DELETE /api/v1/patients/{patient_id}` | FR-PRO-011 |
| MH-GUA-01 | 监护管理中心 | `GuardianHub` | `feature-guardian` | — | `GET /api/v1/patients/{patient_id}`（读取 `guardians[]`、`pending_invitations[]`、`active_primary_transfer`；基线未单列邀请列表端点，以 patient 聚合返回为准） | FR-PRO-005 |
| MH-GUA-02 | 发出监护邀请 | `GuardianInvite` | `feature-guardian` | — | `POST /api/v1/patients/{patient_id}/guardians/invitations` | FR-PRO-005 |
| MH-GUA-03 | 接受 / 拒绝 / 取消邀请 | `GuardianInviteRespond` | `feature-guardian` | — | `POST /api/v1/patients/{patient_id}/guardians/invitations/{invitation_id}/respond`（body `{action: "ACCEPT"｜"REJECT"｜"CANCEL"}`） | FR-PRO-005 |
| MH-GUA-04 | 主监护转移发起 | `GuardianTransferInit` | `feature-guardian` | — | `POST /api/v1/patients/{patient_id}/guardians/primary-transfer` | FR-PRO-006 |
| MH-GUA-05 | 主监护转移响应 | `GuardianTransferRespond` | `feature-guardian` | — | `POST /api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/respond`（body `{action: "CONFIRM"｜"REJECT"}`）；发起方撤回走 `POST /api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/cancel` | FR-PRO-006 |
| MH-GUA-06 | 移除监护 | `GuardianRemove` | `feature-guardian` | — | `DELETE /api/v1/patients/{patient_id}/guardians/{user_id}` | FR-PRO-007 |
| MH-TAG-01 | 标签列表 | `TagManager` | `feature-tag` | — | 从 `GET /api/v1/patients/{patient_id}` 响应 `tags[]` 字段读取（基线未单列 `/patients/{id}/tags`） | FR-MAT-004 |
| MH-TAG-02 | 标签绑定 | `TagBind` | `feature-tag` | 相机（扫码） | `POST /api/v1/tags/{tag_code}/bind` | FR-MAT-004 |
| MH-TAG-03 | 标签挂失确认 | `TagLossConfirm` | `feature-tag` | — | `POST /api/v1/tags/{tag_code}/loss/confirm` | FR-MAT-005 |
| MH-TASK-00 | 任务列表 | `TaskList` | `feature-task` | — | `GET /api/v1/rescue/tasks` | FR-TASK-005 |
| MH-TASK-01 | 创建任务 | `TaskCreate` | `feature-task` | 精确定位 | `POST /api/v1/rescue/tasks` | FR-TASK-001 |
| MH-TASK-02 | 任务详情 | `TaskDetail` | `feature-task` | — | `GET /api/v1/rescue/tasks/{task_id}/full`（BFF 聚合）；局部刷新走 `GET /api/v1/rescue/tasks/{task_id}/snapshot` | FR-TASK-002 |
| MH-TASK-03 | 任务关闭 | `TaskClose` | `feature-task` | — | `POST /api/v1/rescue/tasks/{task_id}/close` | FR-TASK-003/004 |
| MH-TASK-04 | 任务地图 / 轨迹 | `TaskMap` | `feature-task` | 精确定位 | `GET /api/v1/rescue/tasks/{task_id}/trajectory/latest`（支持 `?after_version=` 增量） | FR-TASK-005 |
| MH-TASK-05 | 长期维持标记 | `TaskSustained` | `feature-task` | — | `POST /api/v1/rescue/tasks/{task_id}/sustained` | FR-TASK-006 |
| MH-CLUE-01 | 线索列表 / 详情 | `ClueList`/`ClueDetail` | `feature-clue` | — | `GET /api/v1/clues?task_id=...&clue_id=...`（基线无单条 GET，列表按 `clue_id` 过滤获取详情） | FR-CLUE-004 |
| MH-CLUE-02 | 家属提交线索 | `ClueFamilySubmit` | `feature-clue` | 相机、相册、精确定位 | `POST /api/v1/clues/report`（登录态携带 JWT，自动识别为家属身份） | FR-CLUE-001 |
| MH-PUB-01 | 二维码扫码落地 | `PublicScan` | `feature-public` | 相机 | `GET /r/{resource_token}` | FR-CLUE-002 |
| MH-PUB-02 | 手动输入 TagCode | `PublicManualEntry` | `feature-public` | — | `POST /api/v1/public/clues/manual-entry` | FR-CLUE-003 |
| MH-PUB-03 | 匿名上报表单 | `PublicReport` | `feature-public` | 相机、相册、粗略定位 | `POST /api/v1/clues/report` | FR-CLUE-001 |
| MH-PUB-04 | 紧急联系人提示 | `PublicEmergency` | `feature-public` | — | `GET /r/{resource_token}`（`emergency_contact`） | FR-CLUE-004 |
| MH-PUB-05 | 上报成功回执 | `PublicReceipt` | `feature-public` | — | — | FR-CLUE-001 |
| MH-ORD-01 | 物资工单列表 | `OrderList` | `feature-material` | — | `GET /api/v1/material/orders` | FR-MAT-001 |
| MH-ORD-02 | 工单详情 | `OrderDetail` | `feature-material` | — | `GET /api/v1/material/orders?order_id=...`（基线列表接口按 ID 过滤；不提供单条 GET） | FR-MAT-001 |
| MH-ORD-03 | 发起工单 | `OrderCreate` | `feature-material` | — | `POST /api/v1/material/orders` | FR-MAT-002 |
| MH-ORD-04 | 确认签收 / 取消 | `OrderReceive`（详情内动作） | `feature-material` | — | `POST /api/v1/material/orders/{order_id}/receive`、`POST /api/v1/material/orders/{order_id}/cancel` | FR-MAT-003 |
| MH-NOTI-01 | 消息中心 | `NotificationTab` | `feature-notification` | 通知 | `GET /api/v1/notifications/inbox`、`POST /api/v1/notifications/{notification_id}/read`（基线无批量已读端点，前端循环调用单条） | FR-GOV-008 |
| MH-AI-01 | AI 对话入口 / 历史会话列表 | `AiChatLauncher` | `feature-ai` | — | `GET /api/v1/ai/sessions`（API §3.8.1.1，历史列表）、`POST /api/v1/ai/sessions`（新建/续接）、`DELETE /api/v1/ai/sessions/{id}`（§3.8.1.3，归档） | FR-AI-001 |
| MH-AI-02 | AI 对话（LUI 主页） | `AiChat` | `feature-ai` | 麦克风（可选） | `POST /api/v1/ai/sessions`、`POST /api/v1/ai/sessions/{session_id}/messages`（SSE）、`GET /api/v1/ai/sessions/{id}/messages`（§3.8.1.4 续传）、`POST /api/v1/ai/sessions/{id}/messages/{mid}/cancel`（§3.8.1.5 取消） | FR-AI-001/002 |
| MH-AI-03 | 工具意图确认卡 | `AiIntentConfirm`（Chat 内子组件） | `feature-ai` | — | `POST /api/v1/ai/sessions/{session_id}/intents/{intent_id}/confirm` | FR-AI-003（HC-01） |
| MH-AI-05 | AI 海报生成 | `AiPoster` | `feature-ai` | 相册（保存） | `POST /api/v1/ai/poster` | FR-AI-004 |
| MH-AI-06 | AI 反馈 | `AiFeedback`（Chat 内） | `feature-ai` | — | `POST /api/v1/ai/sessions/{session_id}/feedback` | FR-AI-005 |
| MH-ME-01 | 我的主页 | `MeTab` | `feature-me` | — | `GET /api/v1/users/me` | FR-GOV-003 |
| MH-ME-02 | 资料查看（只读） | `MeProfile` | `feature-me` | — | `GET /api/v1/users/me`（基线未开放 `PUT /api/v1/users/me`；个人资料修改 V2.x 待补） | FR-GOV-003 |
| MH-ME-03 | 语言 / 主题 / 大字 | `MeSettings` | `feature-me` | — | — | NFR-UX-002 |
| MH-ME-04 | 隐私中心 | `MePrivacy` | `feature-me` | — | — | NFR-SEC-004 |
| MH-ME-05 | 关于 / 版本 / 强升 | `MeAbout` | `feature-me` | — | `GET /api/v1/meta/version?platform=ANDROID`（API §3.8.6.1）；返回 `min_compatible_version` / `force_upgrade` 驱动 `MhVersionGuardDialog` | NFR-OPS-001 |
| MH-ME-06 | 推送令牌注册（系统页 / 后台） | — | `feature-me` | 通知 | `POST /api/v1/users/me/push-tokens`（API §3.8.5.1）注册；`DELETE /api/v1/users/me/push-tokens/{id}`（§3.8.5.2）注销；退出账号链式调用 `POST /api/v1/auth/logout`（§3.8.7.1）带 `push_token_id` | NFR-NOTI-001 |
| MH-ERR-01 | 网络错误 / 空态 / 403 / 404 | `ErrorPage` | `feature-common` | — | — | NFR-UX-001 |

> **说明**：已删除 `MH-AI-04 AI 记忆笔记`（基线未提供 `/ai/memory-notes` 端点），改由会话内上下文承载；如 V2.x 开放再独立建页。

### 15.2 不在家属端范围（排除说明）

以下 API 归 Web 管理端，**不得**在 App 出现入口，作排除记录防止误实现：

| API | 归属 |
| :--- | :--- |
| `/api/v1/admin/**`（用户审核、强制转移、任务强闭、标签作废、线索强审） | Web 管理端 |
| `POST /api/v1/material/orders/{id}/approve` / `/ship` / `/resolve-exception` | Web 管理端 |
| `DEAD letter replay`（诊断） | Web 管理端 |
| `GET /api/v1/admin/patients` / `admin/users` | Web 管理端 |
| `POST /api/v1/public/clues/manual-entry`**（非登录态）** | H5 公众端（本 App `MH-PUB-02` 仅作可选入口，登录态下复用） |

### 15.3 模块依赖关系

```
feature-splash ──▶ feature-auth ──▶ feature-home
                                       │
          ┌────────────┬───────────────┼──────────────┬──────────┬──────────┬──────────┐
          ▼            ▼               ▼              ▼          ▼          ▼          ▼
   feature-patient  feature-task  feature-guardian  feature-tag  feature-clue  feature-ai  feature-material
          │            │               │              │          │          │          │
          └────────────┴───────────────┴──────────────┴──────────┴──────────┴──────────┘
                                        │
                                        ▼
                           feature-notification ─▶ feature-me
                                        │
                                        ▼
                                  feature-public （扫码 / 匿名）
```

---

## 16. 鉴权域详细规格（Auth）

### 16.1 MH-SPL-01 启动页 + 隐私同意

- **路由**：`Splash`
- **业务目标**：冷启动执行自检、等待隐私同意、初始化 SDK、路由到登录或首页。
- **交互线框**：

```
┌──────────────────────────┐
│        码上回家           │
│                          │
│        [  Logo  ]        │
│                          │
│         加载中…           │
└──────────────────────────┘
(首启时弹《隐私政策》+《用户协议》全屏对话)
```

- **主要逻辑**：

  1. 读取 `DataStore.consentGranted`；未同意 → 弹隐私政策全屏；同意后写回。
  2. 同意后初始化：高德地图、FCM、Crashlytics、`MhWebSocketClient`（登录后连接）。
  3. `AuthTokenStore.access()` 有值 → 跳 `Home` 并 `popUpTo(0)`；无 → 跳 `AuthLogin`。
  4. 非 `FAMILY` 角色由 `GET /api/v1/users/me` 返回后拦截（§13.5）。

- **API**：无（`GET /api/v1/users/me` 在首启后静默执行）
- **i18n keys**：`splash.title`, `privacy.dialog.title`, `privacy.dialog.agree`, `privacy.dialog.disagree`
- **大字模式**：Logo + 加载文字字号自动放大。

### 16.2 MH-AUTH-01 登录

- **路由**：`AuthLogin`
- **业务目标**：用户名 + 密码登录家属端；获取 JWT 并持久化。
- **交互线框**：

```
┌──────────────────────────┐
│ ← 返回                    │
│                          │
│   码上回家（家属端）       │
│                          │
│  用户名 [_______________] │
│  密  码 [_______________] │
│  □ 记住我                 │
│                          │
│  [  登  录  （主色）  ]   │
│                          │
│  注册 | 忘记密码           │
│                          │
│  语言：中文 / English     │
└──────────────────────────┘
```

- **字段校验**：

  | 字段 | 规则 | 错误 key |
  | :--- | :--- | :--- |
  | username | 4–32 位，字母/数字/下划线 | `E_AUTH_4001` |
  | password | 8–64 位 | `E_AUTH_4002` |

- **接口**：`POST /api/v1/auth/login`

  ```json
  // 请求
  { "username": "alice", "password": "secret123" }
  // 响应
  { "code": "ok", "data": { "access_token": "…", "refresh_token": "…", "expires_in": 3600, "user": { "user_id": "usr_…", "role": "FAMILY" } } }
  ```

- **交互动作**：

  | 触发 | 动作 | 成功 | 失败 |
  | :--- | :--- | :--- | :--- |
  | 点【登录】 | 校验 → 调用接口 | 跳 `Home`；保存 token | `E_AUTH_4011` 密码错、`E_GOV_4031` 封禁；错误走 Snackbar |
  | 点【注册】 | 跳 `AuthRegister` | — | — |
  | 点【忘记密码】 | 跳 `AuthPasswordReset` | — | — |
  | 点语言切换 | `AppCompatDelegate.setApplicationLocales` | — | — |

- **非 FAMILY 处理**：响应 `user.role != "FAMILY"` → 清 token，提示"请使用 Web 管理端"。

- **i18n keys**：`screen.auth.login.title`, `form.auth.username.label`, `form.auth.password.label`, `btn.auth.login`, `btn.auth.register`, `btn.auth.forgot`

- **无障碍**：
  - 用户名 `contentDescription`："用户名输入框，支持 4 到 32 位字母数字"
  - 密码框 `contentDescription`："密码输入框，不会被屏幕阅读器朗读内容"

### 16.3 MH-AUTH-02 注册

- **路由**：`AuthRegister`
- **业务目标**：家属新用户注册账号；强制同意协议。
- **字段**：username、password、confirm_password、email、display_name、同意勾选项。
- **接口**：`POST /api/v1/auth/register`

  ```json
  // 请求
  { "username": "alice", "password": "secret123", "email": "a@x.com", "display_name": "Alice" }
  // 响应
  { "code": "ok", "data": { "user_id": "usr_…", "created_at": "2025-01-01T00:00:00Z" } }
  ```

- **失败**：`E_AUTH_4091` 用户名占用、`E_AUTH_4092` 邮箱占用、`E_AUTH_4003` 邮箱格式错。
- **成功**：跳 `AuthLogin` 并自动填充用户名。
- **i18n keys**：`screen.auth.register.title`, `form.auth.email.label`, `form.auth.display_name.label`, `btn.auth.register.submit`

### 16.4 MH-AUTH-03 找回密码（发送邮件）

- **路由**：`AuthPasswordReset`
- **业务目标**：输入邮箱 → 服务端下发重置链接。
- **接口**：`POST /api/v1/auth/password-reset/request`

  ```json
  // 请求
  { "email": "a@x.com" }
  // 响应
  { "code": "ok", "data": { "sent_at": "2025-01-01T00:00:00Z" } }
  ```

- **风控**：60s 倒计时按钮（`MhCountdownButton`）防频繁请求；`E_GOV_4291` 按 `Retry-After` 倒计时。
- **成功提示**：「若邮箱已注册，我们已向其发送重置链接。」（不暴露邮箱是否存在）

### 16.5 MH-AUTH-04 重置密码（邮件回跳）

- **路由**：`AuthPasswordResetConfirm?token=...`（深链 `mshj://auth/reset`）
- **业务目标**：通过邮件 token 重置密码。
- **接口**：`POST /api/v1/auth/password-reset/confirm`

  ```json
  { "token": "prt_…", "new_password": "new_secret_123" }
  ```

- **失败**：`E_AUTH_4012` token 无效 / 过期。
- **成功**：跳回 `AuthLogin` 并 Snackbar 提示密码已更新。

### 16.6 MH-AUTH-05 修改密码（登录态）

- **路由**：`MeChangePwd`
- **接口**：`PUT /api/v1/users/me/password`

  ```json
  { "old_password": "old", "new_password": "new_123" }
  ```

- **失败**：`E_AUTH_4013` 旧密码错。
- **成功**：本地清 refresh token，强制重新登录。

### 16.7 Token 刷新（`POST /api/v1/auth/token/refresh`）

- 由 `AuthInterceptor` 自动触发（§10.2.3）；客户端无 UI。
- **刷新失败**（HTTP 401 / `E_GOV_4011`）→ 清会话、清本地库、跳 `AuthLogin`，提示"登录已过期，请重新登录"。
- **并发守卫**：使用 `Mutex` 保证同一时刻仅一次刷新；其余请求等结果后重试。

---

## 17. 家庭工作台 + 患者域

### 17.1 MH-HOME-01 家庭工作台

- **路由**：`Home`（底部 Tab 1）
- **业务目标**：一屏聚合"进行中任务 / 当前患者概览 / 快捷动作 / 最新通知摘要"。

- **交互线框**：

```
┌──────────────────────────────────┐
│ TopBar: 码上回家 / 个人头像（导航） │
├──────────────────────────────────┤
│ ┌─当前患者卡片─────────────────┐ │
│ │ [头像] 张三 · 男 · 72岁         │ │
│ │ 最后更新：10:12                │ │
│ │  [查看档案]  [地图]             │ │
│ └────────────────────────────┘ │
│                                  │
│ ┌─进行中任务（0-1 张）──────────┐ │
│ │ 状态：[ACTIVE] 张三 走失       │ │
│ │ 12:08 创建 · 17 条线索         │ │
│ │  [进入任务]                   │ │
│ └────────────────────────────┘ │
│                                  │
│ ┌─快捷操作─────────────────────┐ │
│ │ [一键求助]  [扫码/查标签]      │ │
│ │ [AI 助手]   [发起工单]         │ │
│ └────────────────────────────┘ │
│                                  │
│ ┌─最新消息（最多 3 条）──────────┐ │
│ │ • 监护邀请：李四请求加入…      │ │
│ │ • 物资工单 O-213 已发货        │ │
│ │                  [查看全部]   │ │
│ └────────────────────────────┘ │
├──────────────────────────────────┤
│ [首页•] [地图] [消息] [我的]       │
└──────────────────────────────────┘
```

- **数据来源**：

  | 区块 | API | 备注 |
  | :--- | :--- | :--- |
  | 当前患者 | `GET /api/v1/patients`（当前用户可访问） + `DataStore.lastPatientId` | 多患者时用底部选择器切换 |
  | 进行中任务 | `GET /api/v1/rescue/tasks?state=ACTIVE,SUSTAINED&patient_id={id}` | 理论上同患者仅 1 条 |
  | 快捷操作 | 客户端本地 | — |
  | 最新消息 | `GET /api/v1/notifications/inbox?page_size=3` | 徽标由 WS 推 |

- **交互**：

  | 触发 | 动作 |
  | :--- | :--- |
  | 点患者卡片 | `PatientDetail(id)` |
  | 点【查看档案】 | 同上 |
  | 点【地图】 | `TaskMap(activeTaskId)`；无活任务则 `MapTab` |
  | 点任务卡片 | `TaskDetail(id)` |
  | 点【一键求助】 | 有进行中任务 → `TaskDetail`；否 → `TaskCreate(patient_id)` |
  | 点【扫码/查标签】 | 走扫码（`MH-PUB-01`）；扫到自家标签 → `TagManager` |
  | 点【AI 助手】 | `AiSessionList` 或直接新建 `AiChat` |
  | 点【发起工单】 | `OrderCreate` |
  | 点消息行 | 对应详情 |

- **实时订阅**：`task` + `notify` 两个 sink；`home.summary` 来自派生投影，不单独拉。
- **i18n keys**：`screen.home.title`, `home.card.patient.updated_at`, `home.btn.sos`, `home.btn.scan`, `home.btn.ai`, `home.btn.order`

### 17.2 MH-PAT-01 患者档案详情

- **路由**：`PatientDetail/{patient_id}`
- **业务目标**：展示患者完整档案：基础信息 / 外观描述 / 监护列表 / 标签 / 围栏 / 病史。
- **交互线框**：

```
┌──────────────────────────────────┐
│ ← 返回           张三 档案  ✎ 编辑 │
├──────────────────────────────────┤
│ [头像] 男 · 72岁 · 1952-05-08      │
│ 身份证：1101*********008           │
│                                  │
│ ─ 外观描述 ─────────────────── │
│ 身高 168cm · 体重 62kg · 右手拄拐 │
│ 发色：花白；常戴黑框眼镜            │
│                                  │
│ ─ 围栏（已开启）──── [前往设置]  │
│ 中心：朝阳公园；半径：500m          │
│                                  │
│ ─ 监护人（3）──── [管理]         │
│ 👤 张三（主 Primary）               │
│ 👤 李四；👤 王五                    │
│                                  │
│ ─ 关联标签（2）── [管理]         │
│ [TAG-00123 ACTIVE]                │
│ [TAG-00124 LOST]                  │
│                                  │
│ [⚡ 一键求助]                      │
└──────────────────────────────────┘
```

- **API**：`GET /api/v1/patients/{patient_id}`（基线已聚合 profile + guardians + tags + fence + missing_pending 等字段；基线未单列 `/full` BFF 变体）
- **异常**：`E_PRO_4040` 无权限；`E_PRO_4041` 患者不存在 → 跳错误页。
- **一键求助**：
  - 有进行中任务 → 跳 `TaskDetail`；
  - 无 → 跳 `TaskCreate(patient_id)`；
  - `missing_pending == true` → 先弹 `MH-PAT-04` 模态确认。

### 17.3 MH-PAT-02 患者档案新建 / 编辑

- **路由**：`PatientEdit/{patient_id?}`（null 为新建）
- **字段与校验**：

  | 字段 | 规则 | 错误 |
  | :--- | :--- | :--- |
  | `name` | 1–32 字 | `E_PRO_4201` |
  | `gender` | MALE/FEMALE/UNKNOWN | `E_PRO_4202` |
  | `birthday` | yyyy-MM-dd，不得晚于今天 | `E_PRO_4203` |
  | `id_card` | 合法大陆 18 位，可空 | `E_PRO_4204` |
  | `avatar` | JPEG/PNG ≤ 5MB | 客户端校验 |
  | `appearance` | 高度/体重/标志性特征 | `E_PRO_4205` |

- **接口**：
  - 新建：`POST /api/v1/patients`（返回 `patient_id`）
  - 资料：`PUT /api/v1/patients/{patient_id}/profile`
  - 外观：`PUT /api/v1/patients/{patient_id}/appearance`

- **幂等**：写接口必带 `X-Request-Id`。
- **头像**：走 `POST /api/v1/media/upload-sign?scene=USER_AVATAR`（API §3.8.3.1）拿到 OSS presigned URL → 客户端 `PUT` 直传 → 将返回的 `public_url` 作为 `avatar_url` 提交。

### 17.4 MH-PAT-03 围栏设置

- **路由**：`PatientFence/{patient_id}`
- **业务目标**：设置中心坐标 + 半径 + 是否启用。

- **交互线框**：

```
┌──────────────────────────────────┐
│ ← 返回                围栏设置    │
├──────────────────────────────────┤
│  [ 地图区域（拖动选中心点）]       │
│                                  │
│  中心：116.480,39.936              │
│  半径：[—— ●————] 500 m            │
│                                  │
│  [ □ 启用家庭围栏告警 ]            │
│                                  │
│  说明：越界超过 3 分钟会推送提醒    │
│                                  │
│  [  保  存  ]                     │
└──────────────────────────────────┘
```

- **接口**：`PUT /api/v1/patients/{patient_id}/fence`

  ```json
  { "enabled": true, "center": { "lng": 116.480, "lat": 39.936 }, "radius_m": 500 }
  ```

- **校验**：启用时 `center` 与 `radius_m (50 ≤ r ≤ 5000)` 必填；未填 → `E_PRO_4221`。
- **权限**：需 `ACCESS_FINE_LOCATION`；无权限仅允许手动输入坐标。
- **坐标系**：高德返回 GCJ-02（§25）；客户端不做转换，直接上送。

### 17.5 MH-PAT-04 误记未寻回确认

- **路由**：`PatientMissingPending/{patient_id}`（通常为模态）
- **业务目标**：患者被系统或监护人标记"疑似未寻回"，发起人需要二次确认。

- **交互**：

  | 选择 | API |
  | :--- | :--- |
  | 【确认未寻回，发起任务】 | `POST /api/v1/patients/{patient_id}/missing-pending/confirm` body `{action: "CONFIRM_MISSING"}` + 服务端自动创建 `rescue_task` |
  | 【误记，撤销】 | `POST /api/v1/patients/{patient_id}/missing-pending/confirm` body `{action: "CONFIRM_SAFE", reason: "..."}` |
  | 【稍后再说】 | 关闭模态（24h 内再次提醒） |

- **错误**：`E_PRO_4092` 该患者已有进行中任务 → 跳 `TaskDetail`。
- **大字模式**：按钮高度 `64dp`，双按钮上下排布。

---

## 18. 监护 + 标签域

### 18.1 MH-GUA-01 监护管理中心

- **路由**：`GuardianHub/{patient_id}`
- **业务目标**：查看/管理该患者的所有监护关系与主监护。

- **交互线框**：

```
┌──────────────────────────────────┐
│ ← 返回        监护人管理           │
├──────────────────────────────────┤
│ ┌─主监护─────────────────────┐  │
│ │ 👤 张三（我）      Primary ★  │  │
│ │  [转移主监护权…]             │  │
│ └──────────────────────────┘  │
│                                  │
│ ┌─其他监护（2）─────────────┐   │
│ │ 👤 李四（配偶）   [ 移除 ]   │   │
│ │ 👤 王五（子）    [ 移除 ]   │   │
│ └──────────────────────────┘  │
│                                  │
│ ┌─待确认邀请（1）────────────┐   │
│ │ 赵六 · 1h 前邀请  [ 撤回 ]  │   │
│ └──────────────────────────┘  │
│                                  │
│ [ + 发出邀请 ]                    │
└──────────────────────────────────┘
```

- **API**：
  - 档案聚合拉取：`GET /api/v1/patients/{patient_id}`（响应内 `guardians[]`、`pending_invitations[]`、`active_primary_transfer` 字段一次性返回；基线未单列 `/guardians`、`/invitations`、`/primary-transfer/active` 端点）

- **按钮级权限**（§13.5）：
  - 【转移主监护】仅 `primary_guardian_user_id == me.user_id` 可见
  - 【移除】自己不能移除自己；非主监护不能移除他人（`E_PRO_4031`）

- **实时**：WS `guardian.*` 事件即时更新列表。

### 18.2 MH-GUA-02 发出监护邀请

- **路由**：`GuardianInvite/{patient_id}`
- **字段**：被邀请人 `username` 或 `email`；`relation`（配偶/子/女/其他）；`note`（可选备注 ≤ 64 字）。
- **接口**：`POST /api/v1/patients/{patient_id}/guardians/invitations`

  ```json
  { "patient_id": "pat_…", "invitee_key": "lisi@x.com", "invitee_key_type": "email", "relation": "SPOUSE", "note": "家庭成员" }
  ```

- **失败**：
  - `E_PRO_4093` 该用户已是监护人
  - `E_PRO_4094` 当前患者邀请数已达上限（SRS BR-007）
  - `E_PRO_4092` 存在进行中转移
- **成功**：返回 `invitation_id`，Snackbar 提示"邀请已发出，等待对方确认"。

### 18.3 MH-GUA-03 接受 / 拒绝邀请

- **触发**：消息中心或推送跳转 → 模态弹窗或独立页 `GuardianInviteAccept/{invitation_id}`
- **API**：`POST /api/v1/patients/{patient_id}/guardians/invitations/{invitation_id}/respond`，body `{action: "ACCEPT" | "REJECT" | "CANCEL"}`
- **拒绝必填 reason (5–128 字)**；接受后刷新监护列表。
- **错误**：`E_PRO_4095` 邀请已过期（`INVITATION_EXPIRED_HOURS=72`）。

### 18.4 MH-GUA-04 主监护转移发起

- **路由**：`GuardianTransferInit/{patient_id}`
- **选择**：在现有监护列表中选择一位接收人；输入 `reason` (5–256 字)。
- **接口**：`POST /api/v1/patients/{patient_id}/guardians/primary-transfer`

  ```json
  { "target_user_id": "usr_…", "reason": "因长期出差…" }
  ```

- **错误**：`E_PRO_4096` 已有转移进行中；`E_PRO_4097` 目标不是当前监护人。

### 18.5 MH-GUA-05 主监护转移确认 / 拒绝 / 撤销

- **路由**：`GuardianTransferConfirm/{patient_id}/{transfer_id}`
- **动作矩阵**（对齐 §9.5.2）：

  | 操作者 | 动作 | API |
  | :--- | :--- | :--- |
  | 目标受方 | 确认 / 拒绝 | `POST /api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/respond`（body `{action: "CONFIRM" \| "REJECT"}`） |
  | 发起方 | 撤回 | `POST /api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/cancel` |
  | 目标受方 | 拒绝（reason ≥ 5 字） | `POST …/reject` |
  | 发起方 | 撤销（PENDING 内） | `POST …/cancel` |

- **结果提示**：确认成功 → 刷新 `PatientDetail.primary_guardian`；拒绝 / 撤销 → 状态变为 `REJECTED` / `CANCELLED`，保留记录。
- **超时**：`PRIMARY_TRANSFER_EXPIRES_HOURS=48`，服务端自动置 `EXPIRED`，客户端 WS 接收后更新。

### 18.6 MH-GUA-06 移除监护

- **入口**：`GuardianHub` 中点【移除】。
- **API**：`DELETE /api/v1/patients/{patient_id}/guardians/{user_id}` + 请求体 `{ "reason": "…" }`
- **风控**：必须二次确认弹窗；仅主监护可操作；主监护不得移除自己（需走转移流程）。
- **错误**：`E_PRO_4032` 不可移除主监护；`E_PRO_4033` 不是监护人。

### 18.7 MH-TAG-01 标签列表

- **路由**：`TagManager/{patient_id}`
- **业务目标**：展示该患者名下所有标签及其状态。
- **API**：从 `GET /api/v1/patients/{patient_id}` 响应中读取 `tags[]` 字段（基线未单列 `/tags` 子端点）
- **列表项**：

```
┌──────────────────────────────────┐
│ TAG-00123  BOUND  (主监护)        │
│ 绑定于 2025-01-02  [ 挂失 ][ 二维码 ]│
├──────────────────────────────────┤
│ TAG-00124  LOST                  │
│ 挂失于 2025-01-10   （已冻结）     │
├──────────────────────────────────┤
│ TAG-00125  VOID  （终态，只读）    │
└──────────────────────────────────┘
                        [ + 绑定新标签 ]
```

- **按钮级权限**（§9.5.3）：`LOST` / `VOID` 状态禁用绑定；`UNBOUND` / `ALLOCATED` 隐藏挂失。

### 18.8 MH-TAG-02 标签绑定

- **路由**：`TagBind/{patient_id}`
- **步骤**：扫码获取 `tag_code` → 展示确认 → 绑定。
- **接口**：`POST /api/v1/tags/{tag_code}/bind`

  ```json
  { "patient_id": "pat_…", "owner_guardian_user_id": "usr_me" }
  ```

- **错误**：
  - `E_TAG_4041` 标签不存在
  - `E_TAG_4091` 已被绑定（展示原持有人姓名脱敏）
  - `E_TAG_4092` 标签状态为 `VOID`

### 18.9 MH-TAG-03 挂失确认

- **路由**：`TagLossConfirm/{tag_code}`
- **业务目标**：持有人发现标签丢失，挂失以防滥用。

- **接口**：`POST /api/v1/tags/{tag_code}/loss/confirm`

  ```json
  { "lost_at": "2025-01-10T12:00:00Z", "note": "在公园走丢" }
  ```

- **后果**：挂失后 `/r/{resource_token}` 扫描立即显示"已挂失"；匿名 submit 被拒（`E_CLUE_4012`）。

---

## 19. 任务 + 线索域

### 19.1 MH-TASK-01 创建任务

- **路由**：`TaskCreate/{patient_id}`
- **业务目标**：快速发起寻回任务。

- **交互线框**：

```
┌──────────────────────────────────┐
│ ← 返回         发起寻回            │
├──────────────────────────────────┤
│ 患者：张三（已锁定）                │
│ 最后见到：[日期时间选择器]           │
│ 最后位置：[ 选择 / 当前位置 ]        │
│ 穿着 / 携带：[ 多行输入 128 字 ]     │
│ 伴随情况：[ 单独 / 有人陪同 ]        │
│                                  │
│ [          立即发起（主色）  ]      │
└──────────────────────────────────┘
```

- **API**：`POST /api/v1/rescue/tasks`

  ```json
  {
    "patient_id": "pat_…",
    "last_seen_at": "2025-01-10T12:00:00Z",
    "last_location": { "lng": 116.48, "lat": 39.93 },
    "clothing": "黑色羽绒服、灰色毛线帽",
    "companion": "ALONE"
  }
  ```

- **成功**：返回 `task_id` → 跳 `TaskDetail(task_id)`，并推 WS `task.created`。
- **失败**：
  - `E_TASK_4091` 该患者已有进行中任务 → 跳已存在 `TaskDetail`
  - `E_PRO_4040` 非监护人
- **离线**：不允许离线创建（需服务端即时校验）；网络异常重试。
- **幂等**：`X-Request-Id` 必带；连续点击按钮会话期内复用同一 ID。

### 19.2 MH-TASK-02 任务详情

- **路由**：`TaskDetail/{task_id}`
- **业务目标**：一屏呈现任务状态、时间线、线索摘要、地图入口、操作按钮。

- **交互线框**：

```
┌──────────────────────────────────┐
│ ← 返回      任务 #T-88023    ⋮    │
├──────────────────────────────────┤
│ [ACTIVE] 已发起 02:14             │
│ 患者：张三    [查看档案]            │
│ 最后见到：10:12 朝阳公园             │
│                                  │
│ ─ 实时动态 ────────────────── │
│ [ 地图预览 + 最新线索 Marker ]      │
│  [ 打开地图 ]                     │
│                                  │
│ ─ 时间线（最新在上）───────────  │
│ • 12:08 线索 C-211 待审（家属）     │
│ • 11:55 线索 C-210 已采纳（扫码）    │
│ • 11:30 任务创建                  │
│                                  │
│ ─ 线索（17 条，3 待审）─ [全部] │
│  [线索卡片略]                     │
│                                  │
│ [  关闭任务  ]  [  长期维持  ]     │
└──────────────────────────────────┘
```

- **API**：`GET /api/v1/rescue/tasks/{task_id}/full`（聚合 task + timeline + clues + alerts + 已参与志愿者数）
- **实时**：订阅 `task` + `clue`；WS 事件触发增量更新。
- **动作矩阵**（对齐 §9.5.1）：

  | 当前状态 | 关闭 | 长期维持 |
  | :--- | :---: | :---: |
  | `CREATED` / `ACTIVE` | 允许 | 允许（`SUSTAINED`） |
  | `SUSTAINED` | 允许（仅 `RESOLVED`） | 禁用 |
  | `RESOLVED` / `FALSE_ALARM` | 禁用 | 禁用 |

- **溢出菜单 `⋮`**：分享任务海报（`MH-AI-05`）；

### 19.3 MH-TASK-03 关闭任务

- **路由**：`TaskClose/{task_id}`（模态）
- **选择**：

  ```
  ┌──────────────────────────────────┐
  │ 关闭任务                         │
  ├──────────────────────────────────┤
  │ ○ 已寻回（RESOLVED）              │
  │ ○ 误报（FALSE_ALARM）             │
  │   原因：[ 5–256 字必填 ]          │
  │                                  │
  │  [ 取消 ]     [ 提交 ]            │
  └──────────────────────────────────┘
  ```

- **API**：`POST /api/v1/rescue/tasks/{task_id}/close`

  ```json
  { "mode": "FALSE_ALARM", "reason": "家人电话联系到了", "version": 7 }
  ```

- **幂等 + 乐观并发**：`version` 字段用于版本守卫；`E_TASK_4091` 版本冲突 → `MhVersionGuardDialog`。
- **失败**：`E_TASK_4221` reason 不符合；`E_TASK_4093` SUSTAINED 不可 FALSE_ALARM。

### 19.4 MH-TASK-04 任务地图 / 轨迹

- **路由**：`TaskMap/{task_id}`
- **图层**：
  - Marker：任务起点、采纳线索点（绿色）、待审（橙色）、已拒（灰色）；
  - Polyline：按时间戳连线（仅 `APPROVED` 采纳的线索）；
  - 热力：> 5 个线索时自动开启 `HeatmapLayer`；
  - 围栏：虚线圆（若启用）。

- **API**：
  - 初始：`GET /api/v1/rescue/tasks/{task_id}/trajectory/latest`
  - 增量：`GET /api/v1/rescue/tasks/{task_id}/trajectory/latest?after_version={v}`（同一端点携带版本游标；基线未提供 `/trajectory` 根路径变体）
  - 实时：WS `clue.*` 推送点位后直接入图。

- **交互**：

  | 手势 | 动作 |
  | :--- | :--- |
  | 点 Marker | 弹底部卡片 → 线索详情 |
  | 双指缩放 | 默认地图行为 |
  | 点右下【定位】 | 回到"全部线索 + 最后见到点"的 boundingBox |
  | 点右上【图层】 | 切换热力 / 路线 / 原始点 |

- **性能**：Marker 上限 500；超过用聚合（高德 `MarkerClusterer`）。

### 19.5 MH-TASK-05 长期维持

- **路由**：`TaskSustained/{task_id}`（模态）
- **业务目标**：任务开启 `TASK_SUSTAINED_DAYS=7` 后系统提示；用户可转 `SUSTAINED`。
- **API**：`POST /api/v1/rescue/tasks/{task_id}/sustained`

  ```json
  { "enabled": true, "reason": "家属希望继续关注但不高频告警", "version": 12 }
  ```

- **效果**：开启后停止推送高频提醒，仅在关键线索（APPROVED）时告警；UI Chip 从 ACTIVE → SUSTAINED。
- **关闭**：仅可关闭为 `RESOLVED`。

### 19.6 MH-CLUE-02 家属线索提交

- **路由**：`ClueFamilySubmit/{task_id}`
- **表单**：

  | 字段 | 说明 |
  | :--- | :--- |
  | 照片 | 1–3 张，≤ 5MB / 张 |
  | 位置 | GPS 自动 + 可拖动微调 |
  | 时间 | 默认当前，可改（但不可晚于当前） |
  | 备注 | ≤ 256 字 |

- **接口**：
  1. 照片先调用 `POST /api/v1/media/upload-sign?scene=CLUE_PHOTO`（API §3.8.3.1）获取 OSS presigned URL，再由客户端 `PUT` 直传，取返回 `public_url`
  2. 主体 `POST /api/v1/clues/report`

  ```json
  {
    "task_id": "tsk_…",
    "source": "FAMILY",
    "photos": ["https://cdn/…1.jpg"],
    "location": { "lng": 116.48, "lat": 39.93 },
    "reported_at": "2025-01-10T12:10:00Z",
    "note": "老人在超市附近被监控拍到"
  }
  ```

- **离线**：写入 `outbox`，网络恢复由 `OutboxReplayWorker` 重放（§12.5）。
- **失败**：`E_CLUE_4040` 任务不存在；`E_CLUE_4092` 已在非 ACTIVE 状态。

### 19.7 MH-CLUE-01 线索详情

- **路由**：`ClueDetail/{clue_id}`
- **内容**：照片、位置、时间、上报人脱敏（家属显示姓名、公众显示"爱心网友"）、审核状态、审核理由（如 `APPROVED` 或 `REJECTED`）、相邻线索（时间临近）。
- **API**：`GET /api/v1/clues?clue_id={clue_id}`（基线未开放单条 GET，使用列表接口按 `clue_id` 过滤返回单条）
- **家属可见**：`review_state ∈ {PENDING, APPROVED}`；`REJECTED` 仅供 Web 管理员可见（家属侧过滤展示"拒绝线索"折叠区 + 拒绝原因脱敏）。

### 19.8 WebSocket Ticket 获取

- 所有地图 / 任务详情页启动时：`POST /api/v1/ws/ticket`（capabilities）→ 通过 `MhWebSocketClient.connect()` 建连（§11.2）。
- 同一进程内单例；切换任务时仅发 `{ "type":"subscribe", "domain":"task", "task_id":"…" }` 子订阅帧。

---

## 20. 匿名公共域（扫码链路）

> 公共域是**未登录态**可进入的链路，家属端 App 同样内置扫码入口；**所有公共端接口无 JWT，仅携带 `X-Anonymous-Token`**（由 `AnonymousTokenInterceptor` 管理）。

### 20.1 MH-PUB-01 扫码落地

- **路由**：`PublicScan/{resource_token}`（支持深链 `https://r.mshj.example.com/{token}`）
- **入口**：
  1. 扫码相机捕获二维码 URL → 匹配本 App App Links → 打开该页；
  2. 家属工作台【扫码/查标签】按钮 → 打开 `CameraX` 扫码预览；
  3. 外部浏览器跳转 → 引导下载或继续 H5。

- **接口**：`GET /r/{resource_token}` → 返回：

  ```json
  {
    "code": "ok",
    "data": {
      "resource_type": "TAG",
      "tag_state": "BOUND",
      "patient": { "display_name": "张先生", "avatar_url": "…", "appearance": "…" },
      "emergency_contact": { "masked_phone": "138****5678", "display_name": "张女士" },
      "anonymous_token": "atk_…",
      "rate_limit": { "window_s": 60, "remaining": 5 }
    }
  }
  ```

- **`AnonymousTokenInterceptor`** 写入 `atk_…` 后续公开接口携带。
- **状态分支**：

  | `tag_state` | 展示 |
  | :--- | :--- |
  | `BOUND` | 正常流程；显示【我是路人，想帮他】→ `PublicReport` |
  | `LOST` | 顶部红色告警「该患者已挂失」+ 仅提供一键联系紧急联系人 |
  | `VOID` | 「此标签已作废」+ 提示走【手动输入 TagCode】 |
  | 其他 | 错误页 |

- **敏感数据**：紧急联系人手机号**必须**脱敏（`138****5678`）；点击【拨打】→ `Intent.ACTION_DIAL`（非 `ACTION_CALL`）。

### 20.2 MH-PUB-02 手动输入 TagCode

- **路由**：`PublicManualEntry`
- **业务目标**：路人拿到印着 `TAG-xxxxx` 的标签但无法扫码。
- **接口**：`POST /api/v1/public/clues/manual-entry`

  ```json
  { "tag_code": "TAG-00123" }
  // 响应同 §20.1，返回 anonymous_token 与 patient 摘要
  ```

- **失败**：
  - `E_TAG_4041` 标签不存在
  - `E_TAG_4092` VOID
  - `E_GOV_4291` 限流（`PUB_MANUAL_RATE_LIMIT=10/min`）→ 倒计时按钮

### 20.3 MH-PUB-03 匿名上报线索

- **路由**：`PublicReport?mode={SIGHT|DROPOFF}&anonymous_token={atk}`
- **业务目标**：
  - `SIGHT` 目击上报：拍照 + 当前位置；
  - `DROPOFF` 交接上报：患者在我身边，等待家属（附电话联系按钮）。

- **交互线框**：

```
┌──────────────────────────────────┐
│ ← 返回         帮助家属            │
├──────────────────────────────────┤
│  模式： ◉ 目击   ○ 交接中          │
│  [+ 添加照片（1–3张）]             │
│  位置：自动 / 手动拖动              │
│  时间：默认当前                    │
│  联系方式（选填）：                │
│  [ 备注（≤ 128 字）]                │
│                                  │
│  [  提  交（主色）  ]              │
│  * 您的信息将被匿名化处理          │
└──────────────────────────────────┘
```

- **接口**：`POST /api/v1/clues/report`（匿名）

  ```json
  {
    "anonymous_token": "atk_…",
    "mode": "SIGHT",
    "photos": ["https://cdn/…"],
    "location": { "lng": 116.48, "lat": 39.93 },
    "reported_at": "2025-01-10T12:10:00Z",
    "contact": { "phone": "13800000000" },
    "note": "在超市门口看到一位老人"
  }
  ```

- **幂等**：`X-Request-Id` 必带；重复提交同 ID 不生成重复线索。
- **限流**：`PUB_REPORT_RATE_LIMIT=3/min/token`；429 倒计时。
- **失败**：
  - `E_CLUE_4012` 匿名凭据失效 → 回 `PublicScan` 重新扫码；
  - `E_TAG_4092` 标签 VOID；
  - `E_CLUE_4221` 缺照片或位置。

### 20.4 MH-PUB-04 紧急联系人提示

- **路由**：`PublicEmergency/{resource_token}`（可内嵌在 `PublicReport` 顶部）
- **接口**：复用 `GET /r/{resource_token}` 响应 `emergency_contact` 字段。
- **动作**：
  - 【拨打紧急联系人】→ `Intent.ACTION_DIAL`；
  - 【发站内消息】→ 需登录，跳 `AuthLogin` 后进入通知中心。
- **脱敏**：号码中间 4 位 `****`；完整号码仅服务端保留，客户端**严禁**展示。

### 20.5 MH-PUB-05 上报成功回执

- **路由**：`PublicReceipt/{clue_id}`
- **展示**：
  - 提交成功图示 + 线索编号前 8 位（如 `C-2a3b4c5d…`）；
  - 说明：「感谢您的帮助，家属已收到提醒」；
  - 按钮：【继续上报新线索】 / 【返回首页】；
  - 推广：【了解码上回家】→ 跳应用介绍页（H5）。
- **无后续接口**：纯本地页面；不展示任何家属隐私信息。

### 20.6 公共域统一约束

1. 公共域任何页面 **不得**出现家属登录态信息（token / 用户昵称 / 角色菜单）。
2. 公共域网络请求统一走 `AnonymousTokenInterceptor`；**不**注入 `Authorization`。
3. `anonymous_token` 在前台有效期 15min；失效必须回到扫码。
4. 崩溃上报 / 日志中**不得**记录 `anonymous_token`、紧急联系人号码、照片原图路径。
5. 相机预览页面必须有【手动输入 TagCode】降级入口（§20.2）。
6. 公共域页面保留 `MhBottomNavBar` 隐藏，采用全屏布局；返回可直接退出 App 或回上一路径。

---

## 21. 物资工单域

> 家属端仅涉及"申领 + 签收 + 取消"；**管理员专属**动作 `APPROVE` / `SHIP` / `RESOLVE_EXCEPTION` **不得**出现任何入口或 UI 状态。

### 21.1 MH-ORD-01 工单列表

- **路由**：`OrderList`
- **Tabs**：
  - 进行中（`SUBMITTED` / `APPROVED` / `SHIPPING` / `SHIPPED` / `EXCEPTION`）
  - 已完成（`RECEIVED` / `CANCELLED` / `REJECTED`）
- **API**：`GET /api/v1/material/orders?state=…&cursor=…&page_size=20`
- **项视图**：工单号、患者、物资类型、数量、状态 chip、更新时间、右侧快捷动作（签收 / 取消 / 详情）。
- **实时**：WS `material` 推送 `order.state.changed` 事件 → 增量刷新该项。
- **下拉刷新 + 分页**：支持 `PullToRefresh` + `LazyPagingItems`。

### 21.2 MH-ORD-02 工单详情

- **路由**：`OrderDetail/{order_id}`
- **API**：`GET /api/v1/material/orders?order_id={order_id}`（基线未开放单条 GET，使用列表接口按 `order_id` 过滤返回单条）
- **布局**：

```
┌──────────────────────────────────┐
│ ← 返回         工单 #O-213   [状态]│
├──────────────────────────────────┤
│ 申领患者：张三                     │
│ 申领人：我                          │
│ 物资：[SmartTag × 2, 袖标 × 1]      │
│ 收件：朝阳区…（脱敏后 3 位）        │
│ 备注：…                            │
│                                  │
│ ─ 状态时间线 ─────────────── │
│ • SHIPPED   2025-01-10 09:21     │
│ • APPROVED  2025-01-09 18:00     │
│ • SUBMITTED 2025-01-09 17:50     │
│                                  │
│ ─ 物流 ──────────────────── │
│ 顺丰：SF1234567890    [ 复制 ]    │
│                                  │
│ [ 确认签收 ]   [ 取消工单 ]        │
└──────────────────────────────────┘
```

- **动作矩阵**（对齐 §9.5.4）：

  | 状态 | 可见按钮 |
  | :--- | :--- |
  | `SUBMITTED` | 【取消工单】 |
  | `APPROVED` / `SHIPPING` | 无（只读） |
  | `SHIPPED` | 【确认签收】 |
  | `RECEIVED` / `CANCELLED` / `REJECTED` | 无 |
  | `EXCEPTION` | 【联系客服】（跳 AI 助手） |

### 21.3 MH-ORD-03 发起工单

- **路由**：`OrderCreate`
- **表单**：

  | 字段 | 校验 |
  | :--- | :--- |
  | 患者 | 从当前可访问患者列表选择 |
  | 物资项（可多选） | 每项数量 1–10，总项数 ≤ 5 |
  | 收件地址 | 省市区 + 详细地址（5–128 字） |
  | 收件人姓名 | 1–32 字 |
  | 收件人电话 | 大陆手机号正则 |
  | 备注 | ≤ 256 字，可选 |

- **接口**：`POST /api/v1/material/orders`

  ```json
  {
    "patient_id": "pat_…",
    "items": [{ "sku": "SMART_TAG", "quantity": 2 }, { "sku": "ARMBAND", "quantity": 1 }],
    "shipping": {
      "recipient_name": "张女士",
      "recipient_phone": "13800000000",
      "address": "北京市朝阳区…"
    },
    "note": "优先发顺丰"
  }
  ```

- **失败**：
  - `E_MAT_4221` 字段缺失
  - `E_MAT_4092` 该患者 30 日内已申领上限（`MATERIAL_ORDER_LIMIT`）
  - `E_MAT_4093` 某物资库存不足（展示剩余量）
- **成功**：返回 `order_id`，跳 `OrderDetail`。
- **幂等**：必带 `X-Request-Id`。

### 21.4 MH-ORD-04 签收 / 取消

- **入口**：`OrderDetail` 按钮或列表项快捷菜单。
- **确认签收**：
  - `POST /api/v1/material/orders/{order_id}/receive`

    ```json
    { "received_at": "2025-01-11T09:00:00Z", "note": "东西都收到，谢谢" }
    ```

  - 失败：`E_MAT_4091` 非 `SHIPPED` 状态不允许签收。
- **取消**：
  - `POST /api/v1/material/orders/{order_id}/cancel`

    ```json
    { "reason": "地址填错，请重新下单" }
    ```

  - 失败：`E_MAT_4092` 非 `SUBMITTED` 状态不可取消。
- **异常**：
  - `EXCEPTION` 状态禁用签收 / 取消；【联系客服】跳 `AiChat`，带上 `order_id` 作为会话 context。

### 21.5 家属端物资域排除清单

| API | 归属 | 客户端处理 |
| :--- | :--- | :--- |
| `POST /api/v1/material/orders/{id}/approve` | 管理员 | 不暴露入口 |
| `POST /api/v1/material/orders/{id}/ship` | 管理员 | 不暴露入口 |
| `POST /api/v1/material/orders/{id}/resolve-exception` | 管理员 | 不暴露入口 |
| `GET /api/v1/material/skus`（库存一览） | 管理员 | 基线 API V2.0 未提供家属端 SKU 查询端点；下单时使用基线 `material.order` 请求体中的 `tag_type` 枚举（`QR_CODE` / `NFC`） |

### 21.6 性能与体验

1. 列表 `minHeight = 88dp`，大字模式 `104dp`；卡片圆角 `12dp`。
2. 长地址使用 `Text(maxLines = 2, overflow = Ellipsis)`；详情展开为完整文本。
3. 复制物流单号走 `ClipboardManager.setPrimaryClip`，并 Snackbar"已复制"。
4. 列表与详情共享 Room 缓存（`OrderEntity`），离线可读。

---

## 22. 通知 + AI 域

### 22.1 MH-NOTI-01 消息中心

- **路由**：`NotificationTab`
- **分区**：
  - "待处理"（`action_required = true`，如监护邀请、主监护转移、线索待审 - 家属角度看仅"待你确认的动作"）
  - "任务 / 线索"
  - "物资工单"
  - "系统 / 安全"

- **API**：
  - 列表：`GET /api/v1/notifications/inbox?category=&cursor=&page_size=20`
  - 单条已读：`POST /api/v1/notifications/{notification_id}/read`
  - 全部已读：前端遍历未读列表循环调用单条 `/read`（基线未提供批量已读端点）
- **项视图**：图标（按 category）、标题、内容前 60 字、时间（相对时间"2 分钟前"）、未读小点。
- **实时**：WS `notify.new` 直接插入列表顶；`notify.read` 同步已读态；`BottomNavBar` 徽标走 `unread_count` 派生。
- **点击跳转**：根据 `deep_link` 字段直接路由（白名单校验，防止非法 scheme）。

- **大字模式**：每条卡片高 `96dp`；标题 `18sp` / 内容 `16sp`；"全部已读"按钮 `56dp` 高置顶。

### 22.2 AI 域总览（对齐 API §3.7 + SRS FR-AI-\*）

AI 域包含五种界面产出：
1. 会话列表 `MH-AI-01`
2. 对话（LUI 主入口）`MH-AI-02`
3. 工具意图确认卡 `MH-AI-03`
4. 记忆笔记 `MH-AI-04`
5. 海报生成 `MH-AI-05`

**LUI（Language User Interface）** 语义为"以自然语言驱动系统操作"，与"大字易读模式"属**正交**能力：大字模式是 A11y 视觉增强；LUI 是交互范式。AI 对话页为 LUI 主入口。

### 22.3 MH-AI-01 AI 会话列表

- **路由**：`AiSessionList`
- **API**：`GET /api/v1/ai/sessions`（API §3.8.1.1，服务端列表）+ 本地 Room 缓存（离线展示）；进入会话时调用 `POST /api/v1/ai/sessions` 创建或续接。
- **项**：会话标题（首轮提问摘要）、更新时间、未读徽标、置顶标记；删除调用 `DELETE /api/v1/ai/sessions/{id}`（API §3.8.1.3，服务端软归档 + 本地标记 `ARCHIVED`）。
- **操作**：【+ 新建会话】 → `POST /api/v1/ai/sessions` → 跳 `AiChat(session_id)`。

### 22.4 MH-AI-02 AI 对话（LUI 主入口）

- **路由**：`AiChat/{session_id}`
- **业务目标**：用户以自然语言咨询、查询、发起系统操作；AI 返回文本 / Tool Intent 卡片 / 资源引用。

- **交互线框**：

```
┌──────────────────────────────────┐
│ ← 返回   码上回家助手   ⋮ 反馈   │
├──────────────────────────────────┤
│ 🤖  您好，我是「码上回家助手」…    │
│                                  │
│ 👤  查一下张三今天的位置记录        │
│                                  │
│ 🤖  张三最后见到 10:12 朝阳公园…   │
│    [引用：任务 #T-88023]           │
│                                  │
│ 🤖  [🔧 Tool Intent: 关闭任务]     │
│    目标：T-88023                  │
│    模式：RESOLVED                 │
│                                  │
│    [ 确 认 执 行 ] [ 取消 ]       │
│                                  │
├──────────────────────────────────┤
│ [🎤] [输入消息…]          [发送]  │
└──────────────────────────────────┘
```

- **接口**：
  - 发送消息：`POST /api/v1/ai/sessions/{session_id}/messages`（SSE 流式，§11.4）
  - 续传 / 取消：**基线未独立列出**；前端通过 SSE 连接自身的重连 / abort 机制处理；重连后若 token 流未完成，由客户端发起新一轮 `POST …/messages`

- **SSE 事件类型**：

  | type | 作用 | UI |
  | :--- | :--- | :--- |
  | `message.delta` | 文本增量 | 流式打字机 |
  | `message.tool_intent` | 工具意图卡 | `AiIntentConfirm` |
  | `message.reference` | 引用资源 | 卡片下方 Chip |
  | `message.final` | 结束标记 | 停止打字效果 |
  | `message.error` | 错误 | 红字提示 + 重试按钮 |

- **语音输入**：麦克风按钮走 `MediaRecorder`；录音 ≤ 60s；**本地**不转写（调用 `/api/v1/ai/sessions/{id}/messages` 上传 `audio_url`，由服务端 ASR）。无 `RECORD_AUDIO` 权限时隐藏。

### 22.5 MH-AI-03 工具意图确认（Tool Intent）

- **展示时机**：SSE `message.tool_intent` 事件。
- **动作分层**（对齐 API §3.7.5）：

  | action class | 家属端行为 | 举例 |
  | :--- | :--- | :--- |
  | **A0 读取** | 不需确认，自动展示结果 | 查询患者档案、任务状态 |
  | **A1 建议** | 展示建议卡；用户可选是否跳转对应页面 | 推荐发起任务 |
  | **A2 写入（低风险）** | 单按钮【确认执行】 | 新建记忆笔记、标记线索备注 |
  | **A3 写入（高风险）** | 双按钮【确认执行】【取消】 + 二次确认弹窗 | 关闭任务、转移主监护、取消工单 |

- **接口**：`POST /api/v1/ai/sessions/{session_id}/intents/{intent_id}/confirm`

  ```json
  { "decision": "CONFIRM", "version": 7, "override": false }
  ```

- **错误**：
  - `E_AI_4091` 状态已变更（任务已关闭等）→ `MhVersionGuardDialog`
  - `E_AI_4092` 会话 intent 已过期（默认 5min）→ 提示"请重新询问"
- **大字模式**：Intent 卡内部每个 action 按钮高 `56dp`；风险动作使用 `MaterialTheme.colorScheme.error` 描边。

### 22.6 MH-AI-04 AI 记忆笔记

- **路由**：`AiMemoryNotes/{patient_id}`
- **业务目标**：用户维护"给 AI 用于个性化回答的长期记忆"。
- **API**：
  - 列表：`GET /api/v1/ai/memory-notes?patient_id=…`
  - 新增：`POST /api/v1/ai/memory-notes`
  - 更新：`PUT /api/v1/ai/memory-notes/{note_id}`
  - 删除：`DELETE /api/v1/ai/memory-notes/{note_id}`

- **项字段**：
  - `title`（1–32 字）
  - `content`（≤ 512 字）
  - `tags`（多选：饮食 / 作息 / 病史 / 情绪 / 家庭）
  - `effective`（开关）：关闭时 AI 不使用该条。
- **风控**：每位患者最多 50 条；超出 `E_AI_4293`；返回剩余额度。

### 22.7 MH-AI-05 AI 海报生成

- **路由**：`AiPoster/{task_id}`（从任务详情 `⋮` 进入）
- **API**：`POST /api/v1/ai/poster`

  ```json
  {
    "task_id": "tsk_…",
    "template": "STANDARD",   // STANDARD / MINI / LARGE
    "channels": ["WECHAT", "POSTER_A4"]
  }
  ```

- **响应**：返回 `poster_url`（多张）；`trace_id`。
- **交互**：轮播预览 → 【下载】保存到相册（需 `WRITE_MEDIA_IMAGES` API < 29 / Scoped Storage）；【分享】调用系统 `Intent.ACTION_SEND`。
- **限流**：`AI_POSTER_RATE_LIMIT=5/hour`；429 按倒计时（§10.6）。

### 22.8 MH-AI-06 AI 反馈

- **入口**：`AiChat` 右上 `⋮` → 【反馈本轮回答】。
- **API**：`POST /api/v1/ai/sessions/{session_id}/feedback`

  ```json
  { "session_id": "sess_…", "message_id": "msg_…", "rating": "GOOD", "tags": ["准确","有用"], "comment": "…" }
  ```

- **UX**：4 星或以下 → 展开文本域与标签（最多 3 个）。

### 22.9 AI 域统一约束

1. **服务端权威（HC-02）**：所有写入动作必须经服务端校验；客户端**不得**根据 AI 文本自行改库。
2. **版本守卫**：Intent 卡 `version` 与目标对象 `version` 不一致时必须阻断并刷新。
3. **隐私合规**：发送给 AI 的 `patient_id` / `task_id` / `memory_notes` 由后端解析鉴权；客户端**禁止**发送身份证号、完整手机号等敏感明文。
4. **Remote kill switch**：`DataStore.AI_ON == false` 时禁用 AI Tab 与快捷入口，展示"AI 助手维护中"。
5. **离线**：AI 接口全部在线；离线时入口置灰并提示"需联网"。

---

## 23. API 覆盖矩阵

> 矩阵 A：**页面 → API**（逐条列出消费）；矩阵 B：**API → 页面**（确保家属端接口 100% 覆盖）。路径 / 方法严格对齐 `v2/API_V2.0.md`。

### 23.1 矩阵 A：页面 → API

| 页面 | 消费 API（方法 + 路径） |
| :--- | :--- |
| MH-SPL-01 | `GET /api/v1/users/me`（静默） |
| MH-AUTH-01 | `POST /api/v1/auth/login` |
| MH-AUTH-02 | `POST /api/v1/auth/register` |
| MH-AUTH-03 | `POST /api/v1/auth/password-reset/request` |
| MH-AUTH-04 | `POST /api/v1/auth/password-reset/confirm` |
| MH-AUTH-05 | `PUT /api/v1/users/me/password` |
| MH-HOME-01 | `GET /api/v1/patients` · `GET /api/v1/rescue/tasks?state=ACTIVE,SUSTAINED` · `GET /api/v1/notifications/inbox?page_size=3`（前端并行聚合；基线未提供 `/home/summary` BFF） |
| MH-PAT-01 | `GET /api/v1/patients/{patient_id}`（响应内嵌 `guardians[] / appearance / fence / tags[] / pending_invitations[] / active_primary_transfer`） |
| MH-PAT-02 | `POST /api/v1/patients` · `PUT /api/v1/patients/{patient_id}/profile` · `PUT /api/v1/patients/{patient_id}/appearance`（图片直传见 §14 附件上传，以 BDD 为准） |
| MH-PAT-03 | `PUT /api/v1/patients/{patient_id}/fence` |
| MH-PAT-04 | `POST /api/v1/patients/{patient_id}/missing-pending/confirm`（body `action ∈ {CONFIRM_MISSING, CONFIRM_SAFE}`；无独立 cancel 端点） |
| MH-PAT-05 | `DELETE /api/v1/patients/{patient_id}` |
| MH-GUA-01 | `GET /api/v1/patients/{patient_id}`（从聚合响应读取 `guardians[]` / `pending_invitations[]` / `active_primary_transfer`） |
| MH-GUA-02 | `POST /api/v1/patients/{patient_id}/guardians/invitations` |
| MH-GUA-03 | `POST /api/v1/patients/{patient_id}/guardians/invitations/{invitation_id}/respond`（body `{action: "ACCEPT"｜"REJECT"｜"CANCEL"}`） |
| MH-GUA-04 | `POST /api/v1/patients/{patient_id}/guardians/primary-transfer` |
| MH-GUA-05 | `POST /api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/respond`（body `{action: "CONFIRM"｜"REJECT"}`） · `POST /api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/cancel`（发起方撤回） |
| MH-GUA-06 | `DELETE /api/v1/patients/{patient_id}/guardians/{user_id}` |
| MH-TAG-01 | 读取 `GET /api/v1/patients/{patient_id}` 响应 `tags[]` |
| MH-TAG-02 | `POST /api/v1/tags/{tag_code}/bind` |
| MH-TAG-03 | `POST /api/v1/tags/{tag_code}/loss/confirm` |
| MH-TASK-00 | `GET /api/v1/rescue/tasks` |
| MH-TASK-01 | `POST /api/v1/rescue/tasks` |
| MH-TASK-02 | `GET /api/v1/rescue/tasks/{task_id}/full` · `GET /api/v1/rescue/tasks/{task_id}/snapshot`（局部刷新） · `POST /api/v1/ws/ticket` |
| MH-TASK-03 | `POST /api/v1/rescue/tasks/{task_id}/close` |
| MH-TASK-04 | `GET /api/v1/rescue/tasks/{task_id}/trajectory/latest`（支持 `?after_version=` 增量拉取） |
| MH-TASK-05 | `POST /api/v1/rescue/tasks/{task_id}/sustained` |
| MH-CLUE-01 | `GET /api/v1/clues?task_id={task_id}&clue_id={clue_id}`（基线无单条 GET，用列表接口按 `clue_id` 过滤） |
| MH-CLUE-02 | `POST /api/v1/clues/report`（登录态家属上报，JWT 注入） |
| MH-PUB-01 | `GET /r/{resource_token}` |
| MH-PUB-02 | `POST /api/v1/public/clues/manual-entry` |
| MH-PUB-03 | `POST /api/v1/clues/report`（匿名） |
| MH-PUB-04 | 复用 MH-PUB-01 |
| MH-PUB-05 | —— |
| MH-ORD-01 | `GET /api/v1/material/orders?state=&cursor=&page_size=` |
| MH-ORD-02 | `GET /api/v1/material/orders?order_id={order_id}` |
| MH-ORD-03 | `POST /api/v1/material/orders` |
| MH-ORD-04 | `POST /api/v1/material/orders/{order_id}/receive` · `POST /api/v1/material/orders/{order_id}/cancel` |
| MH-NOTI-01 | `GET /api/v1/notifications/inbox` · `POST /api/v1/notifications/{notification_id}/read` · `POST /api/v1/notifications/read-all`（§3.8.4.1 全部已读） |
| MH-AI-01 | `GET /api/v1/ai/sessions`（§3.8.1.1）· `POST /api/v1/ai/sessions`（创建/续接）· `DELETE /api/v1/ai/sessions/{id}`（§3.8.1.3） |
| MH-AI-02 | `POST /api/v1/ai/sessions/{session_id}/messages`（SSE 流式）· `GET /api/v1/ai/sessions/{id}/messages`（§3.8.1.4 续传）· `POST .../messages/{mid}/cancel`（§3.8.1.5） |
| MH-AI-03 | `POST /api/v1/ai/sessions/{session_id}/intents/{intent_id}/confirm` |
| MH-AI-05 | `POST /api/v1/ai/poster` |
| MH-AI-06 | `POST /api/v1/ai/sessions/{session_id}/feedback` |
| MH-ME-01 | `GET /api/v1/users/me` |
| MH-ME-02 | `GET /api/v1/users/me`（只读；基线未开放 PUT /users/me） |
| MH-ME-03 | —— |
| MH-ME-04 | —— |
| MH-ME-05 | `GET /api/v1/meta/version?platform=ANDROID`（§3.8.6.1 强升策略） |
| MH-ME-06 | `POST /api/v1/users/me/push-tokens`（§3.8.5.1）· `DELETE /api/v1/users/me/push-tokens/{id}`（§3.8.5.2）· `POST /api/v1/auth/logout`（§3.8.7.1） |
| MH-ERR-01 | —— |

### 23.2 矩阵 B：API → 页面

| 方法 | 路径 | 消费页面 |
| :--- | :--- | :--- |
| POST | `/api/v1/auth/register` | MH-AUTH-02 |
| POST | `/api/v1/auth/login` | MH-AUTH-01 |
| POST | `/api/v1/auth/token/refresh` | 网络层 `AuthInterceptor`（全局） |
| POST | `/api/v1/auth/password-reset/request` | MH-AUTH-03 |
| POST | `/api/v1/auth/password-reset/confirm` | MH-AUTH-04 |
| GET | `/api/v1/users/me` | MH-SPL-01, MH-ME-01, MH-ME-02 |
| PUT | `/api/v1/users/me/password` | MH-AUTH-05 |
| GET | `/api/v1/patients` | MH-HOME-01 |
| POST | `/api/v1/patients` | MH-PAT-02 |
| GET | `/api/v1/patients/{patient_id}` | MH-PAT-01, MH-GUA-01, MH-TAG-01 |
| DELETE | `/api/v1/patients/{patient_id}` | MH-PAT-05 |
| PUT | `/api/v1/patients/{patient_id}/profile` | MH-PAT-02 |
| PUT | `/api/v1/patients/{patient_id}/appearance` | MH-PAT-02 |
| PUT | `/api/v1/patients/{patient_id}/fence` | MH-PAT-03 |
| POST | `/api/v1/patients/{patient_id}/missing-pending/confirm` | MH-PAT-04 |
| POST | `/api/v1/patients/{patient_id}/guardians/invitations` | MH-GUA-02 |
| POST | `/api/v1/patients/{patient_id}/guardians/invitations/{invitation_id}/respond` | MH-GUA-03 |
| POST | `/api/v1/patients/{patient_id}/guardians/primary-transfer` | MH-GUA-04 |
| POST | `/api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/respond` | MH-GUA-05 |
| POST | `/api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/cancel` | MH-GUA-05 |
| DELETE | `/api/v1/patients/{patient_id}/guardians/{user_id}` | MH-GUA-06 |
| POST | `/api/v1/tags/{tag_code}/bind` | MH-TAG-02 |
| POST | `/api/v1/tags/{tag_code}/loss/confirm` | MH-TAG-03 |
| GET | `/api/v1/rescue/tasks` | MH-TASK-00, MH-HOME-01 |
| POST | `/api/v1/rescue/tasks` | MH-TASK-01 |
| GET | `/api/v1/rescue/tasks/{task_id}/full` | MH-TASK-02 |
| GET | `/api/v1/rescue/tasks/{task_id}/snapshot` | MH-TASK-02（局部刷新） |
| POST | `/api/v1/rescue/tasks/{task_id}/close` | MH-TASK-03 |
| POST | `/api/v1/rescue/tasks/{task_id}/sustained` | MH-TASK-05 |
| GET | `/api/v1/rescue/tasks/{task_id}/trajectory/latest` | MH-TASK-04 |
| GET | `/api/v1/clues` | MH-CLUE-01 |
| POST | `/api/v1/clues/report` | MH-CLUE-02, MH-PUB-03 |
| GET | `/r/{resource_token}` | MH-PUB-01, MH-PUB-04 |
| POST | `/api/v1/public/clues/manual-entry` | MH-PUB-02 |
| GET | `/api/v1/material/orders` | MH-ORD-01, MH-ORD-02 |
| POST | `/api/v1/material/orders` | MH-ORD-03 |
| POST | `/api/v1/material/orders/{order_id}/receive` | MH-ORD-04 |
| POST | `/api/v1/material/orders/{order_id}/cancel` | MH-ORD-04 |
| GET | `/api/v1/notifications/inbox` | MH-HOME-01, MH-NOTI-01 |
| POST | `/api/v1/notifications/{notification_id}/read` | MH-NOTI-01 |
| POST | `/api/v1/ws/ticket` | MH-TASK-02（及所有实时消费页） |
| POST | `/api/v1/ai/sessions` | MH-AI-01, MH-AI-02 |
| POST | `/api/v1/ai/sessions/{session_id}/messages` | MH-AI-02 |
| POST | `/api/v1/ai/sessions/{session_id}/intents/{intent_id}/confirm` | MH-AI-03 |
| POST | `/api/v1/ai/sessions/{session_id}/feedback` | MH-AI-06 |
| POST | `/api/v1/ai/poster` | MH-AI-05 |
| GET | `/api/v1/ai/sessions` | MH-AI-01 |
| GET | `/api/v1/ai/sessions/{id}` | MH-AI-01（详情确认） |
| DELETE | `/api/v1/ai/sessions/{id}` | MH-AI-01 |
| GET | `/api/v1/ai/sessions/{id}/messages` | MH-AI-02（续传/多端同步） |
| POST | `/api/v1/ai/sessions/{id}/messages/{mid}/cancel` | MH-AI-02 |
| POST | `/api/v1/media/upload-sign` | MH-PAT-02（头像）、MH-CLUE-02（线索照片）、MH-AI-05（海报背景） |
| POST | `/api/v1/notifications/read-all` | MH-NOTI-01 |
| POST | `/api/v1/users/me/push-tokens` | MH-ME-06（启动 / FCM token refresh） |
| DELETE | `/api/v1/users/me/push-tokens/{id}` | MH-ME-06（退出 / 授权关闭） |
| GET | `/api/v1/meta/version` | MH-SPL-01（启动强升检查）、MH-ME-05 |
| POST | `/api/v1/auth/logout` | MH-ME-06（退出） |

> **说明**：API V2.1 增量端点（§3.8）已全部覆盖家属端业务场景；前一版手册登记的 `/media/upload` / `/users/me/push-tokens` / AI 会话管理 / `/meta/version` / `/notifications/read-all` 跨文档差距已随 API V2.1 市场化，本手册不再维护独立 gap 清单。

### 23.3 管理员专属接口（显式排除）

| API | 家属端处理 |
| :--- | :--- |
| `POST /api/v1/admin/**`（审核 / 强制操作 / DEAD replay） | 不消费 |
| `POST /api/v1/material/orders/{id}/approve` | 不消费 |
| `POST /api/v1/material/orders/{id}/ship` | 不消费 |
| `POST /api/v1/material/orders/{id}/resolve-exception` | 不消费 |
| `GET /api/v1/admin/users` | 不消费 |
| `GET /api/v1/admin/patients` | 不消费 |
| `GET /api/v1/material/skus`（完整库存） | 不消费；使用 `/public` 版 |

### 23.4 家属端覆盖率自检

1. API §3（业务接口章节）中所有 `role ∈ {FAMILY, PUBLIC}` 的接口均在矩阵 B 存在消费页面。
2. 新增后端接口时，必须在 PR 的变更说明中更新本矩阵，否则 CI 的 `api-coverage-check` 脚本阻断合并。
3. 矩阵 B 缺失消费 → 在"需求待办"清单中立项（见 §26 发布流程）。

---

## 24. 错误码统一映射

> 错误码 → i18n key → UI 行为；对齐 `v2/API_V2.0.md` §2 错误体系。未列出的错误码统一走 `error.unknown`"发生错误，请稍后重试（trace=xxxxxxxx）"。

### 24.1 行为分类

| 行为 | 说明 |
| :--- | :--- |
| `T`（Toast） | Snackbar 一次性提示 |
| `D`（Dialog） | 阻塞式弹窗，需点确认 |
| `P`（Page） | 跳转错误页 |
| `B`（Button state） | 触发按钮本地态（倒计时 / 置灰） |
| `J`（Jump） | 自动跳转页面（如跳登录） |
| `V`（VersionGuard） | 弹 `MhVersionGuardDialog` |
| `R`（Retry） | 自动/手动重试 |

### 24.2 映射表

| 错误码 | i18n key | UI 行为 | 备注 |
| :--- | :--- | :--- | :--- |
| `E_AUTH_4001` | `error.E_AUTH_4001` | T | 用户名格式不合法 |
| `E_AUTH_4002` | `error.E_AUTH_4002` | T | 密码格式不合法 |
| `E_AUTH_4003` | `error.E_AUTH_4003` | T | 邮箱格式不合法 |
| `E_AUTH_4011` | `error.E_AUTH_4011` | T | 用户名或密码错误 |
| `E_AUTH_4012` | `error.E_AUTH_4012` | D + J(AuthLogin) | 重置 token 失效 |
| `E_AUTH_4013` | `error.E_AUTH_4013` | T | 旧密码错 |
| `E_AUTH_4091` | `error.E_AUTH_4091` | T | 用户名已占用 |
| `E_AUTH_4092` | `error.E_AUTH_4092` | T | 邮箱已占用 |
| `E_GOV_4011` | `error.E_GOV_4011` | D + J(AuthLogin) | 会话过期 |
| `E_GOV_4030` | `error.E_GOV_4030` | D | 权限不足 |
| `E_GOV_4031` | `error.E_GOV_4031` | D + J(AuthLogin) | 账号已封禁 |
| `E_GOV_4032` | `error.E_GOV_4032` | D | 账号未激活 |
| `E_GOV_4291` | `error.E_GOV_4291` | B（倒计时） | 限流，按 `Retry-After` |
| `E_GOV_5001` | `error.E_GOV_5001` | T + R | 服务繁忙 |
| `E_PRO_4030` | `error.E_PRO_4030` | D | 无访问该患者权限 |
| `E_PRO_4031` | `error.E_PRO_4031` | D | 仅主监护可操作 |
| `E_PRO_4032` | `error.E_PRO_4032` | D | 不可移除主监护 |
| `E_PRO_4033` | `error.E_PRO_4033` | T | 不是监护人 |
| `E_PRO_4034` | `error.E_PRO_4034` | T | 转移已不处于可撤销状态 |
| `E_PRO_4041` | `error.E_PRO_4041` | P(ErrorPage) | 患者不存在 |
| `E_PRO_4091` | `error.E_PRO_4091` | V | 档案版本冲突 |
| `E_PRO_4092` | `error.E_PRO_4092` | D + J | 该患者已有进行中任务 / 转移 |
| `E_PRO_4093` | `error.E_PRO_4093` | T | 该用户已是监护人 |
| `E_PRO_4094` | `error.E_PRO_4094` | D | 监护人数 / 邀请数达上限 |
| `E_PRO_4095` | `error.E_PRO_4095` | D | 邀请已过期 |
| `E_PRO_4096` | `error.E_PRO_4096` | T | 已有转移进行中 |
| `E_PRO_4097` | `error.E_PRO_4097` | T | 目标不是监护人 |
| `E_PRO_4201`~`E_PRO_4205` | `error.E_PRO_420x` | T（字段下） | 档案字段校验 |
| `E_PRO_4221` | `error.E_PRO_4221` | T | 围栏配置不完整 |
| `E_TAG_4041` | `error.E_TAG_4041` | T | 标签不存在 |
| `E_TAG_4091` | `error.E_TAG_4091` | D | 已被绑定 |
| `E_TAG_4092` | `error.E_TAG_4092` | D | 标签作废 |
| `E_TASK_4040` | `error.E_TASK_4040` | P | 任务不存在 |
| `E_TASK_4091` | `error.E_TASK_4091` | D + J | 该患者已有进行中任务 |
| `E_TASK_4092` | `error.E_TASK_4092` | D | 任务已非 ACTIVE（线索拒收） |
| `E_TASK_4093` | `error.E_TASK_4093` | D | SUSTAINED 不可 FALSE_ALARM |
| `E_TASK_4094` | `error.E_TASK_4094` | V | 任务版本冲突 |
| `E_TASK_4221` | `error.E_TASK_4221` | T | FALSE_ALARM 原因 5–256 字 |
| `E_CLUE_4012` | `error.E_CLUE_4012` | D + J(PublicScan) | 匿名凭据失效 |
| `E_CLUE_4040` | `error.E_CLUE_4040` | P | 线索 / 任务不存在 |
| `E_CLUE_4092` | `error.E_CLUE_4092` | D | 任务已非 ACTIVE |
| `E_CLUE_4221` | `error.E_CLUE_4221` | T | 缺照片 / 位置 |
| `E_MAT_4041` | `error.E_MAT_4041` | P | 工单不存在 |
| `E_MAT_4091` | `error.E_MAT_4091` | D | 非 SHIPPED 不可签收 |
| `E_MAT_4092` | `error.E_MAT_4092` | D | 非 SUBMITTED 不可取消；或 30 日申领超限 |
| `E_MAT_4093` | `error.E_MAT_4093` | D | 库存不足 |
| `E_MAT_4221` | `error.E_MAT_4221` | T | 字段缺失 |
| `E_AI_4033` | `error.E_AI_4033` | P | 会话无访问权 |
| `E_AI_4091` | `error.E_AI_4091` | V | 目标对象已变更 |
| `E_AI_4092` | `error.E_AI_4092` | T | Intent 过期 |
| `E_AI_4292` | `error.E_AI_4292` | D + B | 用户 AI 额度耗尽 |
| `E_AI_4293` | `error.E_AI_4293` | D | 患者 AI 额度耗尽 |
| `E_NOTI_4041` | `error.E_NOTI_4041` | T | 通知不存在 / 已撤回 |
| `E_NET_TIMEOUT` | `error.E_NET_TIMEOUT` | T + R | 网络超时 |
| `E_NET_UNKNOWN` | `error.E_NET_UNKNOWN` | T + R | 未知网络错误；展示 trace_id 前 8 位 |
| `E_NET_NO_CONNECT` | `error.E_NET_NO_CONNECT` | T | 请检查网络连接 |
| `E_CONSISTENCY_GAP` | `error.E_CONSISTENCY_GAP` | T | 内部：版本跳洞后自愈提示，仅调试可见 |

### 24.3 统一处理入口

```kotlin
fun handleDomainException(e: DomainException, ctx: UiErrorContext) {
    val policy = ErrorPolicyRegistry[e.code] ?: DefaultPolicy
    when (policy.behavior) {
        ErrorBehavior.TOAST      -> ctx.snackbar(policy.messageRes, e.trace.take(8))
        ErrorBehavior.DIALOG     -> ctx.dialog(policy.messageRes, e.trace.take(8))
        ErrorBehavior.PAGE       -> ctx.navigate(MhRoute.ErrorPage(policy.messageRes, e.trace))
        ErrorBehavior.VERSION    -> ctx.showVersionGuard { policy.onRefresh(ctx) }
        ErrorBehavior.AUTH_JUMP  -> { ctx.invalidateSession(); ctx.navigate(MhRoute.AuthLogin) }
        ErrorBehavior.COUNTDOWN  -> ctx.countdown(e.retryAfterSeconds ?: 60)
        ErrorBehavior.RETRY      -> ctx.retryLast()
    }
}
```

### 24.4 错误页（MH-ERR-01）通用规范

1. 顶部图示 + 主文案（i18n）+ 辅助说明；
2. 底部按钮【重试】 / 【返回首页】；
3. 调试模式追加 `trace_id` 全文可复制；正式包仅前 8 位；
4. 不得展示服务端原始 stack trace；
5. 空态 / 无网络 / 403 / 404 走同一容器 + 不同 `ErrorType`。

---

## 25. 性能、安全、可观测

### 25.1 性能预算

| 指标 | 预算 | 采集方式 |
| :--- | :--- | :--- |
| 冷启动时间（非首次安装） | ≤ 2.0s | `AppStartup` + Baseline Profile，Firebase Performance `app_start` 追踪 |
| 首页 TTI（Time-to-Interactive） | ≤ 1.5s | `reportFullyDrawn()` 埋点 |
| 首页交互响应（按钮点击 → 视觉反馈） | ≤ 100ms | Jank Stats |
| WS 首帧时延（登录成功 → 收到首事件） | ≤ 1.5s | 自定义埋点 |
| SSE 首字符时延 | ≤ 1.0s | AI 自定义埋点 |
| 列表滑动掉帧率 | ≤ 1% (jank > 50ms) | `JankStats` |
| APK 体积 | Debug ≤ 80 MB；Release ≤ 30 MB | CI 脚本门禁 |
| 运行内存（首页稳定态） | ≤ 200 MB | `MemoryInfo` 采样 |
| 后台电量（1h 静默） | < 1.5% | 内部机测 |

### 25.2 性能优化手段

1. **Baseline Profile**（`:baselineprofile` 模块）：覆盖冷启动 + 首页 + 任务详情关键路径。
2. **Compose 稳定性**：数据类显式 `@Immutable` / `@Stable`；ViewModel 暴露 `StateFlow<UiState>` 而非大 map。
3. **图片**：Coil `memoryCachePolicy = ENABLED` / `diskCachePolicy = ENABLED`；缩略图走后端裁切参数（`?w=600`）。
4. **列表**：`LazyColumn` + `key = { it.id }`；避免整页 `remember` 重新计算。
5. **地图**：
   - 关闭不必要的 UI（比例尺、指南针按需打开）；
   - Marker 聚合（`MarkerClusterer`）；
   - 切换页面必须 `onPause()` / `onDestroy()` 释放 `MapView`。
6. **DI**：Hilt 避免过多 `@Singleton`；大对象（`MhWebSocketClient` / `MhDatabase`）单例，其他 `@ViewModelScoped` / `@ActivityScoped`。
7. **数据库**：`Room` 大查询用 `Paging3`；避免 `observeAsState` 订阅整表。
8. **Crash 防御**：`WindowRecomposer` 出错重建；顶层 `CoroutineExceptionHandler` 上报不吞异常。

### 25.3 安全基线

| 项 | 方案 |
| :--- | :--- |
| 证书固定（Pinning） | `CertificatePinner` 钉扎 `api.mshj.example.com` 两个 SHA-256（当前 + 备份）；过期提前 30 天滚动 |
| 敏感数据存储 | JWT / refresh：`EncryptedSharedPreferences`；不进 SharedPreferences / DataStore 明文 |
| Room 加密 | Release 使用 SQLCipher（`net.zetetic:sqlcipher-android`）+ keystore-backed passphrase |
| 反调试 | Release 关闭 `isDebuggable`，网络拦截器拒绝连接若检测到 `Proxy` + `debug` 标志 |
| ProGuard / R8 | 开启 `isMinifyEnabled`；保留 `@Serializable` / Retrofit 接口 / Hilt 相关 keep 规则 |
| 网络明文 | `usesCleartextTraffic=false`；`network_security_config.xml` 仅允许 HTTPS |
| 剪贴板 | 敏感内容（JWT / 邀请码）禁止复制；复制后 60s 自动清空 |
| 日志脱敏 | Timber `Tree` 过滤 `Authorization`、`access_token`、身份证号、完整手机号 |
| Screen capture | 登录页、密码页、AI 对话页调用 `FLAG_SECURE` 阻止截屏（Release 开） |
| Intent 合法性 | 深链参数校验 + 白名单；仅 `mshj://` 与官方 `https://r.mshj.example.com` |
| WebView | **默认禁用**；若必须：禁 JavaScript、禁文件访问、配 `safeBrowsing` |

`AndroidManifest` 关键配置：

```xml
<application
    android:networkSecurityConfig="@xml/network_security_config"
    android:usesCleartextTraffic="false"
    android:allowBackup="false"
    android:dataExtractionRules="@xml/data_extraction_rules"
    android:fullBackupContent="false"
    android:name=".MhApplication">
    …
</application>
```

### 25.4 隐私最小化

1. 日志 / Crashlytics 不上报 `patient_id` 明文（哈希前 8 位 + `trace_id`）。
2. 相机 / 相册仅在提交瞬间获取文件；Exif 信息走服务端清洗。
3. 匿名上报链路 **禁止**记录用户昵称、JWT、设备 IMEI；`X-Anonymous-Token` 日志中掩码。
4. 用户"隐私中心"提供【导出我的数据】（调用 `POST /api/v1/users/me/data-export`）和【注销账号】（`DELETE /api/v1/users/me`）入口（若 API 已就绪）。

### 25.5 可观测（HC-Trace）

| 维度 | 工具 | 关键字段 |
| :--- | :--- | :--- |
| 崩溃 | Firebase Crashlytics | `trace_id`, `user_id_hash`, `app_version`, `device` |
| 性能 | Firebase Performance | `trace_id`, `net_latency`, `ttfb`, `ssl_time` |
| 网络日志 | OkHttp `EventListener` + 采样 1/100 上报 | `method`, `path`, `code`, `trace_id`, `latency_ms` |
| 业务埋点 | 自研（走 `POST /api/v1/telemetry/events`） | `event`, `session_id`, `trace_id`, `props` |
| 用户反馈 | 我的 → 反馈 | 自动附 `trace_id` + app/device meta |

调试模式：`MhTraceBadge` 全局展示；Release 仅错误页显示 `trace_id` 前 8 位。

### 25.6 测试策略

| 层 | 工具 | 覆盖 |
| :--- | :--- | :--- |
| 单元 | JUnit5 + Turbine + MockK | Reducer、UseCase、Mapper、拦截器 |
| 网络契约 | MockWebServer | 响应外壳、外壳错误、分页、SSE 事件 |
| 数据库 | Room `MigrationTestHelper` | 每次迁移 |
| UI 组件 | Paparazzi（截图） | `core-designsystem` 全量 + 关键页 |
| 集成 UI | Compose-Test + Espresso | 关键流程（登录、任务关闭、线索提交、AI Intent） |
| 无障碍 | `AccessibilityChecks.enable()` | CI 强制 |
| 稳定性 | Macrobenchmark + Baseline Profile | 启动 + 首页 + 任务详情 |

覆盖率门禁：domain 与 data 层行覆盖 ≥ 70%；feature 层 ≥ 40%；冷启动跑 5 次中位数 ≤ 预算 × 1.1。

---

## 26. 构建、发布、回滚

### 26.1 构建矩阵

| 渠道 | `applicationIdSuffix` | `versionNameSuffix` | baseUrl | WS |
| :--- | :--- | :--- | :--- | :--- |
| `dev`（开发） | `.dev` | `-dev` | `https://api-dev.mshj.example.com/` | `wss://ws-dev.mshj.example.com/` |
| `staging`（预发） | `.stg` | `-stg` | `https://api-stg.mshj.example.com/` | `wss://ws-stg.mshj.example.com/` |
| `prod`（生产） | —— | —— | `https://api.mshj.example.com/` | `wss://ws.mshj.example.com/` |

各渠道独立签名 keystore；`prod` 仅 CI 机持有。

### 26.2 版本号策略

- `versionName` 语义化：`MAJOR.MINOR.PATCH[-channel]`，例如 `2.0.0` / `2.0.1-stg`。
- `versionCode` 采用 `MAJOR*10_000_000 + MINOR*100_000 + PATCH*100 + build_no`。
- 重大 API 变更 → MAJOR；新功能 → MINOR；热修 → PATCH。
- 灰度版本通过"应用内差异"区分，不改版本号。

### 26.3 CI 流水线（GitHub Actions 示意）

```yaml
jobs:
  lint:
    steps:
      - run: ./gradlew detekt ktlintCheck
      - run: ./gradlew lintRelease
  test:
    steps:
      - run: ./gradlew testDebugUnitTest
      - run: ./gradlew paparazziVerifyRelease
      - run: ./gradlew connectedCheck    # 仅在模拟器机器
  build:
    steps:
      - run: ./gradlew assembleDevDebug assembleStgRelease assembleProdRelease
      - run: ./gradlew bundleProdRelease
      - run: python scripts/api-coverage-check.py       # §23.4
      - run: python scripts/i18n-parity-check.py        # §7.7
      - run: python scripts/apk-size-gate.py --max 30MB
  deploy-internal:
    needs: [lint, test, build]
    steps:
      - uses: r0adkll/upload-google-play@v1
        with:
          track: internal
          packageName: com.mshj.family
          releaseFiles: app/build/outputs/bundle/prodRelease/app-prod-release.aab
          mappingFile: app/build/outputs/mapping/prodRelease/mapping.txt
```

### 26.4 灰度与回滚

| 阶段 | 比例 | 触发下一阶段 |
| :--- | :--- | :--- |
| 内部测试 | 公司内 | 观察 24h 无 P0 崩溃 |
| 开放测试（封闭） | 100 人种子家属 | 观察 48h 崩溃率 < 0.3% |
| Staged rollout 1 | 5% | 观察 24h |
| Staged rollout 2 | 20% | 观察 24h |
| Staged rollout 3 | 50% | 观察 48h |
| 全量 | 100% | —— |

**回滚决策指标**：

- 崩溃率 > 1.0%（任一渠道）
- ANR 率 > 0.5%
- 任务关闭成功率下降 > 5pt
- AI SSE 首字符 p95 > 5s（AI 域）
- 客户投诉上升（连续 2 小时每小时 > 10 条）

**回滚手段**：

1. Google Play 停止 staged rollout 并回滚到上一稳定版本；
2. 通过远程配置（`/api/v1/meta/version` 返回 `force_min_version`）强制老版本升级失败的回滚包；
3. Remote kill switch（`DataStore.REALTIME_ON` / `AI_ON`）立即禁用 WS 或 AI，缓解单域事故而无需发版。

### 26.5 ProGuard / R8 关键规则

```proguard
# Retrofit
-keepattributes Signature, *Annotation*
-keep,allowobfuscation interface <1>

# Kotlinx Serialization
-if @kotlinx.serialization.Serializable class **
-keepclassmembers class <1> { *** Companion; }
-keepclasseswithmembers class ** { kotlinx.serialization.KSerializer serializer(...); }

# Hilt
-keep class dagger.hilt.** { *; }
-keep class * extends dagger.hilt.internal.GeneratedComponent

# Compose
-keep class androidx.compose.runtime.** { *; }

# OkHttp Platform
-dontwarn org.conscrypt.**
-dontwarn org.bouncycastle.**
-dontwarn org.openjsse.**

# High Map SDK
-keep class com.amap.** { *; }
-keep class com.autonavi.** { *; }

# Baseline Profile
-keep class androidx.profileinstaller.** { *; }
```

### 26.6 Mapping.txt 上报

1. 每次 Release 构建自动上传 `mapping.txt` 至 Crashlytics / Bugly。
2. mapping 与 `versionCode` 一一对应存档，留存 12 个月。

### 26.7 热修与受控发布

1. 本项目**不使用**私有热修框架（合规风险）。
2. 关键配置（远程开关、错误码文案、AI 开关）通过后端下发 `GET /api/v1/meta/client-config` 实时覆盖，App 启动时拉取 + 缓存 1h。
3. 强制升级：`GET /api/v1/meta/version` 返回 `min_supported_version_code`；客户端低于则弹不可关闭升级对话框。

### 26.8 构建产物清单

| 产物 | 用途 |
| :--- | :--- |
| `app-prod-release.aab` | Google Play 上架 |
| `app-prod-release.apk` | 官网分发（无 GMS 机型） |
| `mapping.txt` | 反混淆 |
| `baseline-profile.txt` | 启动优化 |
| `sources.jar`（可选） | 内部调试 |
| `native-debug-symbols.zip` | NDK 堆栈（若引入） |

---

## 27. 一致性自检

在手册落地或每次基线升级后，必须完成下列自检（CI + Code Review 双把关）：

| # | 检查项 | 依据 | 状态 | 说明 |
| :---: | :--- | :--- | :---: | :--- |
| 1 | 所有 API 路径 / 方法 / 请求体 / 响应外壳与 `v2/API_V2.0.md` 逐字对齐 | HC-API-Align | ✅ | §23 矩阵 A/B 覆盖；`api-coverage-check.py` CI 门禁 |
| 2 | 所有 ID 字段按 `String` 承载，不出现 `Long`/`Int` | HC-API-Align | ✅ | §9.1 DTO 规范 + detekt 自定义规则 |
| 3 | 技术栈严格锁定（Kotlin + Compose + Hilt + Retrofit + Room + …），禁用 Flutter / RN / XML 主布局 | HC-Stack | ✅ | §3.1 版本矩阵 + §4 多模块 |
| 4 | 主色 `#F97316` + 辅色 `#0EA5E9` 显式出现在 `Theme.kt`；亮/暗双态 | HC-Brand | ✅ | §5 `MhLightColors` / `MhDarkColors` |
| 5 | `strings_*.xml` 中英双语键齐全；无硬编码中文 | HC-I18n | ✅ | §7 + `i18n-parity-check.py` |
| 6 | **大字易读模式**为每个家属主线页面声明适配 | HC-A11y | ✅ | §6.4 + 各页 `大字模式` 段落 |
| 7 | 运行时权限齐全（定位 / 相机 / 相册 / 通知 / 麦克风）且均带降级策略 | HC-Privacy | ✅ | §13 |
| 8 | 离线能力：档案 / 任务 / 通知 / 工单可读 + Outbox 写重放 | HC-Offline | ✅ | §12.5 + §11.6 |
| 9 | 矩阵 B 100% 覆盖家属端后端接口；管理员接口显式排除 | HC-Coverage | ✅ | §23 |
| 10 | 禁用短信，所有触达走 WS + FCM + 站内；不声明 `SEND_SMS` | HC-Channel / HC-No-SMS | ✅ | §11 + §13.6 Manifest |
| 11 | 推送 **定向** 下发；禁止全量广播；WS ticket 与 role 校验 | HC-Realtime | ✅ | §11.2 / §11.5 |
| 12 | `X-Trace-Id` / `X-Request-Id` 全链路透传；`X-User-Id` / `X-User-Role` 禁传 | HC-Trace / HC-08 | ✅ | §10.2 拦截器 + `ReservedHeaderGuardInterceptor` |
| 13 | 401 自动刷新 + 单次 `Mutex` 守卫；二次 401 清会话 | HC-API-Align | ✅ | §10.2.3 / §16.7 |
| 14 | 429 按 `Retry-After` 倒计时；不盲目重试 | HC-API-Align | ✅ | §10.6 `MhCountdownButton` |
| 15 | Outbox 重放复用 `X-Request-Id` 保证幂等 | HC-API-Align | ✅ | §12.5 |
| 16 | 服务端权威；UiState 仅存快照 + 派生投影 | HC-02 | ✅ | §9.4 / §9.5 |
| 17 | Tool Intent 写操作必须二次确认（A2/A3），版本守卫 | HC-02 | ✅ | §22.5 + `MhVersionGuard` |
| 18 | LUI（自然语言界面）与大字模式语义正交，互不冲突 | 术语表 | ✅ | §22.2 显式区分 |
| 19 | 非 FAMILY 角色登录家属端被拦截 | HC-Auth | ✅ | §13.5 / §16.2 |
| 20 | 公共域（扫码）无 JWT、仅 `X-Anonymous-Token`；敏感字段脱敏 | HC-Privacy | ✅ | §20 |
| 21 | 证书固定 + 敏感存储加密 + 登录页 `FLAG_SECURE` | 安全基线 | ✅ | §25.3 |
| 22 | Crashlytics / 性能埋点不记录 PII / token 明文 | HC-Privacy | ✅ | §25.4 / §25.5 |
| 23 | V2.1 边界明确：管理员专属接口（审批 / 强制操作 / DEAD replay）不在家属端实现 | HC-Coverage 边界 | ✅ | §15.2 / §23.3 |
| 24 | 性能预算可度量（冷启动 / TTI / APK / 内存） | NFR | ✅ | §25.1 + CI 门禁 |
| 25 | Staged rollout + 远程 kill switch + 强制升级三位一体 | 发布 | ✅ | §26.4 / §26.7 |

如任意检查项为 ❌ 或 ⚠ ，本手册与对应代码不得发版。

---

## 28. V2.0 总结表格（对比 V1.1 变更全景）

> 本节对照原 `android_handbook.md`（V1.1）给出总览变更；供 Review 与迁移参考。

| 维度 | V1.1（原 `android_handbook.md`） | V2.0（本手册） | 变更原因 / 依据 |
| :--- | :--- | :--- | :--- |
| **基线** | SRS_simplify.md（旧） | `v2/SRS.md` + `v2/SADD_V2.0.md` + `v2/LLD_V2.0.md` + `v2/DBD.md` + `v2/API_V2.0.md` + `v2/BDD` + `CHANGELOG_V2.1` | V2 版本基线统一 |
| **系统名称** | 未统一 | 「**码上回家**」 | SRS 术语表 |
| **技术栈** | Kotlin + XML + Compose 混合，RxJava 可选 | **纯 Compose** + Kotlin 1.9.25/JDK 17 + Hilt 2.52 + Retrofit 2.11 + OkHttp 4.12 + Room 2.6.1 + DataStore + Coroutines/Flow | HC-Stack 锁定 |
| **最低/目标 API** | minSdk 24 / target 33 | **minSdk 26 / target 35 / compileSdk 35** | 统一到 V2 基线 |
| **主题色** | 医疗蓝 `#0B6B8A` | **品牌橙 `#F97316` + 辅助蓝 `#0EA5E9`**，亮/暗双态 | HC-Brand / web_admin V2 统一视觉 |
| **国际化** | 仅中文，部分硬编码 | 中英双语，键齐全 + CI 校验 + 大字模式 × 主题 × 语言三轴覆盖 | HC-I18n |
| **无障碍** | 零散声明 | **大字易读模式** + TalkBack + WCAG AAA 对比度 + 编译期 `contentDescription` 校验 | HC-A11y |
| **LUI 概念** | 混淆为"大字大按钮" | 明确 = **Language UI（自然语言交互）**；大字模式独立 | SRS 术语表权威 |
| **分层架构** | MVP / MVVM 混用 | **Clean Architecture + MVI**，UiState / UiIntent / UiEffect 强制 | HC-02 + LLD |
| **模块划分** | 单 `app` 模块 | **多模块**：`app` / `core-*` / `feature-*` / `baselineprofile`，严禁 feature 交叉依赖 | 可维护 + 构建速度 |
| **网络层** | 单一 Retrofit，无响应外壳统一 | 统一 `ApiResponseCallAdapter` + 四拦截器（Trace / RequestId / Auth / Anonymous） + `ReservedHeaderGuard` | HC-API-Align / HC-08 |
| **ID 类型** | 部分 `Long` | **全部 `String`** | API §1.3 硬约束 |
| **实时通道** | 仅轮询 | **WS + SSE + FCM + WorkManager** 混合 + 版本守卫 + Outbox 补偿 | HC-Realtime / HC-05 |
| **推送策略** | 全量轮询 + 有短信备份 | **定向推送**（FCM + 厂商）+ 站内 + WS；**禁止** SMS | HC-No-SMS / HC-Channel |
| **本地存储** | SharedPreferences 明文存 token | `EncryptedSharedPreferences` + Room（可 SQLCipher） + DataStore + Outbox | HC-Privacy |
| **权限与隐私** | 仅 Manifest 声明 | 运行时流程图 + 隐私同意首启 + 按钮级角色控制 + 分场景降级 | HC-Privacy / HC-Auth |
| **页面总数** | ~ 18 页，分域模糊 | **47 页**，编号 `MH-*` 分 12 个域，全量规格 | 覆盖 SRS 全量 |
| **API 覆盖** | 未系统化 | **矩阵 A/B** 双向，CI 脚本校验 100% 家属域覆盖 | HC-Coverage |
| **错误码映射** | 仅通用文案 | 全量 i18n key + 7 种行为 + `MhVersionGuardDialog` | API §2 |
| **任务状态机** | 枚举散在代码 | `CREATED/ACTIVE/RESOLVED/FALSE_ALARM/SUSTAINED + UNKNOWN` + 客户端守卫矩阵 | §9.5.1 |
| **监护转移** | 无 | `PENDING_CONFIRM/ACCEPTED/REJECTED/CANCELLED/EXPIRED` 完整流程 + 撤销 | FR-PRO-005/006 |
| **标签生命周期** | 仅绑定 | `UNBOUND/ALLOCATED/BOUND/LOST/VOID` 全流程 + 挂失确认 | DBD.tag |
| **物资工单** | 无家属端 | 家属仅 `CREATE/CANCEL/RECEIVE`；管理员动作显式排除 | API §3.4 |
| **匿名公共域** | 零散 | `MH-PUB-01~05` 成套：扫码 / 手动 / 上报 / 紧急 / 回执；`X-Anonymous-Token` 独立链路 | FR-PUB-* |
| **AI 域** | 简单聊天 | 会话列表 + LUI 对话 + Tool Intent 卡（A0/A1/A2/A3 分级）+ 记忆笔记 + 海报 + 反馈；SSE 流式；版本守卫 | FR-AI-* |
| **地图 SDK** | 未明确 | **高德 GCJ-02**；Marker 聚合；热力图；围栏；释放规范 | API §1.9 |
| **性能预算** | 未量化 | 冷启动 ≤ 2s / TTI ≤ 1.5s / APK ≤ 30MB / 内存 ≤ 200MB；Baseline Profile + JankStats + CI 门禁 | NFR |
| **安全基线** | 基础混淆 | 证书固定 + SQLCipher 可选 + 明文禁用 + `FLAG_SECURE` + 日志脱敏 + Intent 白名单 | §25.3 |
| **可观测** | 仅 Crashlytics | Crashlytics + Firebase Performance + OkHttp 采样 + 业务埋点 + 用户反馈附 `trace_id` | HC-Trace |
| **发布策略** | 全量发布 | 内测 → 封闭 → staged 5/20/50/100 + 回滚指标 + 远程 kill switch + 强制升级 | 发布规范 |
| **CI 门禁** | 仅单测 | detekt/ktlint/lint + unit/UI/截图 + i18n-parity + api-coverage + apk-size + migration-test | 可落地工程化 |
| **V2.1 边界** | 未区分 | 管理员接口显式 §15.2/§23.3 排除；家属端不再实现 | CHANGELOG V2.1 |
| **自检清单** | 无 | §27 共 **25** 项，CI + Review 双把关 | —— |

### 28.1 交付状态

- 本手册 `v2/android_handbook_V2.0.md` 为**家属端 Android V2.0 开发基线**；
- 与 `v2/web_admin_handbook_V2.0.md`、`v2/API_V2.0.md`、`v2/DBD.md`、`v2/SADD_V2.0.md`、`v2/LLD_V2.0.md`、`v2/backend_handbook_v2.md` 共同构成 V2 交付集合；
- 后续调整必须通过 PR + 本手册 §27 自检；任何与基线冲突以基线为准。

— END —
