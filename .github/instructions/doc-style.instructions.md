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
- UML 图一律使用 **PlantUML** 代码块（```plantuml），禁止使用 Mermaid
- ID 字段在文档中标注为 `string` 类型传输

## 核心术语表

| 术语 | 说明 |
|------|------|
| 路人发现者 | 匿名主体，通过扫码或兜底上报线索 |
| 主监护人 | 对患者拥有最高管理权限的家属成员 |
| 短码（Short Code） | 物理标签上的简短业务标识，支持手动兜底输入 |
| 走失状态 | `NORMAL` / `MISSING` 枚举 |
| 误报 | 任务关闭类型，表示寻回事件不成立 |
| 存疑线索 | 被判为高风险无效、需管理员复核的线索 |
| 标签状态 | `未绑定` → `已分配` → `已绑定` → `挂失` → `作废` |

## 状态枚举速查

| 域 | 状态值 |
|----|--------|
| 走失状态 | `NORMAL`, `MISSING` |
| 任务状态 | `PENDING`, `ACTIVE`, `CLOSING`, `CLOSED` |
| 线索状态 | `PENDING`, `CONFIRMED`, `REJECTED`, `SUSPECT` |
| 标签状态 | `UNBOUND`, `ASSIGNED`, `BINDABLE`, `BOUND`, `LOST`, `REVOKED` |

## 硬约束提醒

文档中涉及以下内容时须确保一致：

- HC-01：TASK 域唯一权威，AI 仅建议
- HC-02：状态变更必须 Outbox
- HC-03：写接口幂等 `request_id`
- HC-04：链路 `trace_id` 全透传
- HC-05：WebSocket 定向下发
- HC-06：禁用短信，仅推送 + 站内通知
