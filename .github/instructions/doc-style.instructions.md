---
description: "编辑项目文档时自动注入排版规范和术语表。Use when: editing markdown, document formatting, 文档排版"
applyTo: "**/*.md"
---

# 文档排版与术语规范

编辑本项目 `.md` 文档时，遵守以下规范。

## 排版规则

- 中英文之间加一个半角空格：`阿尔兹海默症 Alzheimer`
- 关键术语、状态枚举、字段名、事件名使用行内代码样式：`MISSING`、`task_id`、`ClueSubmittedEvent`
- 表格优于纯文本列表表达结构化信息
- UML 图一律使用 Mermaid
- ID 字段在文档中标注为 `string` 类型传输

## 核心术语表

| 术语 | 说明 |
|------|------|
| 路人发现者 | 匿名主体，通过扫码或兜底上报线索 |
| 主监护人 | 对患者拥有最高管理权限的家属成员 |
| 短码（Short Code） | 物理标签上的简短业务标识（6 位），支持手动兜底输入 |
| 走失状态（`lost_status`） | `NORMAL`（正常）/ `MISSING_PENDING`（疑似走失）/ `MISSING`（走失中）三态枚举 |
| 误报 | 任务关闭类型之一，表示本次走失事件不成立 |
| 存疑线索 | 被系统判为高风险无效、需管理员复核的线索 |
| 标签状态 | `UNBOUND` → `ALLOCATED` → `BOUND` → `SUSPECTED_LOST` → `LOST` → `VOIDED` |
| LUI | 基于自然语言的人机交互界面 |
| 间接线索 | 通过扫描海报、转发链接等非现场直接目击方式上报的线索 |
| 核心轨迹集 | 经防漂移算法过滤与时空逻辑校验后的有效位置坐标时间序列 |
| 设备指纹 | 采集匿名路人设备软硬件特征生成的全局唯一 Hash 标识串 |

## 状态枚举速查

| 域 | 状态值 |
|----|--------|
| 患者走失状态 | `NORMAL`, `MISSING_PENDING`, `MISSING` |
| 寻回任务状态 | `CREATED`, `ACTIVE`, `SUSTAINED`, `CLOSED_FOUND`, `CLOSED_FALSE_ALARM` |
| 线索研判状态 | `SUBMITTED`, `PENDING`, `PENDING_REVIEW`, `VALID`, `INVALID` |
| 防走失标签状态 | `UNBOUND`, `ALLOCATED`, `BOUND`, `SUSPECTED_LOST`, `LOST`, `VOIDED` |
| 物资工单状态 | `PENDING_AUDIT`, `PENDING_SHIP`, `SHIPPED`, `RECEIVED`, `EXCEPTION`, `REJECTED`, `CANCELLED`, `VOIDED` |
| 监护权请求状态 | `CREATED`, `PENDING_CONFIRM`, `COMPLETED`, `REJECTED`, `REVOKED`, `EXPIRED` |

## 硬约束提醒（HC-01 ~ HC-08）

文档中涉及以下内容时须确保一致：

- HC-01：`TASK` 域唯一权威 — AI 仅生成建议，严禁绕过 LUI 确认逻辑直接改写状态
- HC-02：变更原子性 — 核心状态变更必须采用 Local Transaction + Outbox Pattern
- HC-03：接口幂等性 — 写接口必须支持 `X-Request-Id` 去重
- HC-04：全链路追踪 — 必须透传 `X-Trace-Id`，日志、内部调用、异步事件均须包含
- HC-05：动态配置化 — 严禁硬编码，围栏半径、防漂移阈值、AI Token 限制等通过配置中心下发
- HC-06：匿名风险隔离 — 匿名入口必须执行"设备指纹 + 频率 + 地理位置"校验
- HC-07：隐私脱敏规范 — PII 展示强制脱敏，路人端照片叠加时间戳水印
- HC-08：通信约束 — 仅支持站内通知、应用推送（极光推送）及 WebSocket 定向下发，通知网关层预留短信服务接口
