# V2.1 增量说明 — 用户治理 & 管理员患者档案

> 本次增量回应两项基线缺陷：
> 1. **缺少后台用户管理**：用户列表、启/禁用、查看/修改、删除；角色隔离（`admin` 仅管 `FAMILY`，`SUPER_ADMIN` 不可禁/删/降）。
> 2. **管理员无法查看全局患者档案**、**无法执行行政级主监护权转移**。

本次修改从基线文档开始全局推进，同时同步前端手册。未改动 SRS 正文，新增需求编号在下文以 **FR-GOV-011 / 012 / 013 / 014** 与 **FR-PRO-011 / 012** 显式标注，后续任何实现均需回写 SRS。

---

## 1. 变更文件一览

| 文件 | 变更类型 | 章节 |
| :--- | :--- | :--- |
| [v2/API_V2.0.md](API_V2.0.md) | 扩充 | §2.4（新错误码）、§2.7（新错误码）、§3.3.0 概览表 + §3.3.15–§3.3.17、§3.6.0 概览表 + §3.6.15–§3.6.20 |
| [v2/LLD_V2.0.md](LLD_V2.0.md) | 扩充 | §5.3.8–§5.3.10（管理员患者 API 契约与流程）、§8.3.8–§8.3.13（用户治理 API 契约与流程） |
| [v2/backend_handbook_v2.md](backend_handbook_v2.md) | 扩充 | §12.1 Action 白名单追加 10 条；新增 §24「后台用户治理与管理员患者访问」完整实现规范 |
| [v2/web_admin_handbook_V2.0.md](web_admin_handbook_V2.0.md) | 扩充 | §13 页面清单追加 P-14a/b、P-15a/b；§14 追加 P-14、P-15 详细规格；§15.1/§15.2 矩阵更新；§23 总结回写 15 个页面组；移除 §13 旧说明"API V2.0 未提供 /admin/users"的豁免注脚 |
| v2/CHANGELOG_V2.1.md | 新建 | 本文件 |

> 未修改：SRS.md、SADD_V2.0.md、DBD.md —— 本次新增的字段与状态**完全基于既有 `sys_user` / `guardian_relation` / `patient_profile` 结构**，未引入新表、未修改 DDL；因此 DBD 无需改动。`sys_user.status` 的三值枚举（`ACTIVE / DISABLED / DEACTIVATED`）在 V2 原 DDL 已具备。

---

## 2. 新增 API 清单（9 条）

| # | 方法 | 路径 | 角色 | 确认等级 | 简述 |
| :---: | :--- | :--- | :--- | :--- | :--- |
| 1 | GET | `/api/v1/admin/users` | ADMIN / SUPER_ADMIN | — | 用户列表（ADMIN 仅见 FAMILY） |
| 2 | GET | `/api/v1/admin/users/{user_id}` | ADMIN / SUPER_ADMIN | — | 用户详情 |
| 3 | PUT | `/api/v1/admin/users/{user_id}` | ADMIN / SUPER_ADMIN | CONFIRM_2 | 修改资料 / 角色（角色仅 SUPER_ADMIN 可改） |
| 4 | POST | `/api/v1/admin/users/{user_id}/disable` | ADMIN / SUPER_ADMIN | CONFIRM_2 | 禁用；SUPER_ADMIN 永不可禁 |
| 5 | POST | `/api/v1/admin/users/{user_id}/enable` | ADMIN / SUPER_ADMIN | CONFIRM_1 | 启用 |
| 6 | DELETE | `/api/v1/admin/users/{user_id}` | ADMIN / SUPER_ADMIN | CONFIRM_3 | 注销；SUPER_ADMIN 永不可删 |
| 7 | GET | `/api/v1/admin/patients` | ADMIN / SUPER_ADMIN | — | 全局患者列表（PII 脱敏） |
| 8 | GET | `/api/v1/admin/patients/{patient_id}` | ADMIN / SUPER_ADMIN | — | 档案详情 |
| 9 | POST | `/api/v1/admin/patients/{patient_id}/guardians/force-transfer` | **SUPER_ADMIN 独占** | CONFIRM_3 | 行政强制转移主监护 |

## 3. 新增错误码

| code | HTTP | 含义 |
| :--- | :---: | :--- |
| `E_USR_4004` | 400 | `role` 枚举非法 |
| `E_USR_4005` | 400 | `email`/`phone`/`nickname` 格式非法 |
| `E_USR_4032` | 403 | 普通管理员禁止操作非 `FAMILY` 账号 |
| `E_USR_4033` | 403 | 禁止操作 `SUPER_ADMIN`（禁/删/降级） |
| `E_USR_4034` | 403 | 禁止操作自身 |
| `E_USR_4035` | 403 | 角色变更仅 `SUPER_ADMIN` 可执行 |
| `E_USR_4092` | 409 | 仍持有主监护关系，禁止删除 |
| `E_USR_4093` | 409 | 仍有未终态任务 / 工单，禁止删除 |
| `E_PRO_4035` | 403 | 强制转移目标非 `FAMILY` 或非 `ACTIVE` |
| `E_PRO_4036` | 403 | 管理员只读访问受限 |
| `E_PRO_4046` | 404 | 强制转移目标用户不存在 |

## 4. 授权矩阵（核心）

### 4.1 用户治理

| 当前角色 | 目标 `role=FAMILY` | 目标 `role=ADMIN` | 目标 `role=SUPER_ADMIN` | 目标=自身 |
| :--- | :--- | :--- | :--- | :--- |
| `ADMIN` | 查看/编辑/禁用/启用/删除 | ❌ `E_USR_4032` | ❌ `E_USR_4033` | ❌ `E_USR_4034` |
| `SUPER_ADMIN` | 全部 | 全部 | 仅查看；`role` 不可降级 `E_USR_4033` | 仅改 `nickname/email/phone`；`role`/禁/删全拒 `E_USR_4034` |

### 4.2 管理员患者档案

| 当前角色 | 列表 | 详情 | 强制转移 |
| :--- | :---: | :---: | :---: |
| `ADMIN` | ✅ 只读脱敏 | ✅ 只读脱敏 | ❌ `E_AUTH_4031` |
| `SUPER_ADMIN` | ✅ | ✅ | ✅ 需 `CONFIRM_3` |

## 5. 新增领域事件（Outbox）

| topic | 发布方 | 消费方 | 触发 |
| :--- | :--- | :--- | :--- |
| `user.disabled` | `auth-service` | `notify-service` | 禁用成功 |
| `user.enabled` | `auth-service` | `notify-service` | 启用成功 |
| `user.deactivated` | `auth-service` | `notify-service` | 注销成功 |
| `user.role.changed` | `auth-service` | `notify-service` + `gateway-security` | 角色变更 |
| `patient.primary_guardian.force_transferred` | `profile-service` | `notify-service` | 强制转移成功 |

## 6. 前端（WAHB）新增页面

| 页面 | 路由 | 角色 | 关键能力 |
| :--- | :--- | :--- | :--- |
| P-14a | `/admin/users` | admin / super_admin | 列表 · cursor 分页 · 角色过滤 |
| P-14b | `/admin/users/:userId` | admin / super_admin | 详情 · 编辑 · 启/禁用 · 三次确认删除 |
| P-15a | `/admin/patients` | admin / super_admin | 全局列表 · PII 脱敏 · banner 警示 |
| P-15b | `/admin/patients/:patientId` | admin / super_admin（强制转移仅 super_admin） | 只读详情 · Tabs · 强制转移抽屉 |

> 前端按钮级权限码：`user:list`/`user:read`/`user:update`/`user:role:update`/`user:disable`/`user:enable`/`user:deactivate`；`admin.patient:list`/`admin.patient:read`/`admin.patient:force-transfer`。

## 7. 跨文档一致性核验

| 检查项 | 结论 |
| :--- | :---: |
| API 路径与 LLD §5.3 / §8.3 描述完全一致 | ✅ |
| BDD §24 与 API §3.3.15–§3.3.17、§3.6.15–§3.6.20 字段对齐 | ✅ |
| WAHB P-14/P-15 所用 API 与 API_V2.0.md §3 严格一致 | ✅ |
| 错误码全链路一致（API §2 ↔ LLD ↔ BDD ↔ WAHB §16） | ✅ |
| HC-06（无短信）：所有用户管理通知走站内消息 / 邮件 | ✅ |
| HC-07（PII 脱敏）：管理员患者接口响应强制 `@Desensitize` | ✅ |
| HC-Coverage（管理端 100% 覆盖）：矩阵 B 已补齐 9 条新 API | ✅ |
| 角色仅 `admin/super_admin`（管理域）与 `FAMILY`（业务域） | ✅ |
| 高危操作均定义 `CONFIRM_` 等级与 `risk_level` | ✅ |
| 长 ID 仍以 string 传输 | ✅ |

## 8. 开发落地建议

1. 后端按 [backend_handbook_v2.md](backend_handbook_v2.md) §24 实现 `AdminUserController` / `AdminPatientController` 与 `@AdminRoleGuard` 切面；
2. 测试先行：补齐 §24.8 六项测试清单；
3. 前端按 WAHB §14.P-14 / §14.P-15 实现，按钮级权限码加入 `useAuthStore.permissions`；
4. 首次灰度仅开放 `SUPER_ADMIN` 账号试用，观察 7 天审计日志再放量 `ADMIN`；
5. 生产发布前后端联调必测：授权矩阵 12 组合（详见 BDD §24.8 #1）+ JWT 吊销路径（disable / deactivate / role change）。

---

**V2.1 变更负责人留白；本增量自合并之日起生效，旧版 WAHB §13 注脚已废弃。**
