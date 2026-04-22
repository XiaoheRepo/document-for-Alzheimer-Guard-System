# 码上回家 · Web 管理端开发手册（WAHB V2.0）

> 本手册是《码上回家》Web 管理端的前端落地基线。严禁偏离本手册与上游基线（SRS/SADD/LLD/DBD/API/BDD）；冲突时以 V2 基线文档为权威。

---

## 0. 文档信息

| 项目 | 内容 |
| :--- | :--- |
| 产品名称 | 码上回家（基于 AI Agent 的阿尔兹海默症患者协同寻回系统） |
| 文档名称 | Web 管理端开发手册（Web Admin Handbook，简称 WAHB） |
| 版本 | V2.0 |
| 日期 | 2026-04-22 |
| 输入基线 | `v2/SRS.md`、`v2/SADD_V2.0.md`、`v2/LLD_V2.0.md`、`v2/DBD.md`、`v2/API_V2.0.md`、`v2/backend_handbook_v2.md` |
| 适用对象 | Web 前端研发、联调测试、架构评审、答辩演示 |
| 目标端 | Web Admin（管理员 `admin` / 超级管理员 `super_admin`），不含家属端 Android 与匿名 H5 |

### 0.1 基线权威层级

| 层级 | 基线 | 决定的事情 |
| :--- | :--- | :--- |
| L1 | SRS V2.0 | 功能是否存在（FR-* 编号） |
| L2 | SADD V2.0 | 架构/鉴权/链路/实时/部署（HC-01 ~ HC-08） |
| L3 | LLD V2.0 | 状态机、字段语义、交互流 |
| L4 | DBD V2.0 | 数据表与字段类型、枚举 |
| L5 | API V2.0 | 接口契约（路径、请求/响应、错误码） |
| L6 | BDD V2.0 | 后端落地口径（错误码、事件名、幂等规范） |
| L7 | **WAHB V2.0**（本文） | Web 管理端前端落地 |

### 0.2 术语速查

| 术语 | 解释 |
| :--- | :--- |
| 可疑线索 | 线索 `review_state = PENDING_REVIEW`，需管理员复核覆写（override）或驳回（reject） |
| DEAD | Outbox 投递失败进入死信，需超管显式重放 |
| 定向 WS | WebSocket 仅按登录用户/订阅任务等具体频道推送，禁止全站广播（HC-05） |
| Trace-Id | 全链路追踪 ID，客户端必须透传并回显（HC-04） |
| Request-Id | 写接口幂等键（HC-03） |

---

## 1. 硬约束（HC）在 Web 管理端的落地

| 编号 | 约束项 | Web 管理端落地方式 |
| :--- | :--- | :--- |
| HC-Stack | 技术栈锁定 | Vue 3.5+ Composition API + TypeScript strict + Ant Design Vue 4.x + ECharts 5.x + Pinia 3.x + Vue Router 4 + Vite 6 + vue-i18n 9。禁止引入 React / Element Plus / Tailwind 作为主样式方案 |
| HC-Brand | 品牌 | 系统名固定「码上回家」；主色 `#F97316`；辅助色 `#0EA5E9`；AntDV `theme.token.colorPrimary = #F97316`，`colorLink = #0EA5E9` |
| HC-I18n | 中英双语 | `vue-i18n@9` + AntDV `<a-config-provider :locale>` + `dayjs.locale()` + ECharts `registerLocale`；默认 `zh-CN`，右上角必须提供语言切换器 |
| HC-Theme | 亮/暗双主题 | 基于 AntDV `theme.defaultAlgorithm` / `theme.darkAlgorithm`；主色在暗色下保持 `#F97316`；容器背景暗色 `#141414`，分层 `#1F1F1F`；切换跟随系统（`prefers-color-scheme`）为默认 |
| HC-01（SADD）领域边界 | 六域隔离 | 前端模块目录按 `task / clue / profile / mat / ai / gov` 划分，禁止跨域直接引用对方 store |
| HC-02（SADD）状态权威 | 状态来自后端 | 禁止前端自行推算任务状态；接收 `state.changed` WS 事件后以服务端快照为准重渲染 |
| HC-03 幂等键 | `X-Request-Id` | 所有写接口（POST/PUT/DELETE）由 Axios 拦截器自动注入 UUID v4；同一表单重试必须复用上次键 |
| HC-04 全链路追踪 | `X-Trace-Id` | 客户端每次请求生成或透传 Trace-Id；响应必须回显，错误提示展示 Trace-Id 前 8 位，错误上报附 Trace-Id |
| HC-05 配置化与定向推送 | 无硬编码阈值；WS 定向 | 速度/时间阈值、轮询间隔全部来自 `/api/v1/admin/configs`；WS 仅订阅当前登录用户与当前关注任务频道 |
| HC-06 无短信 | 仅站内消息 | UI 中不得出现"发送短信"相关按钮；密码重置走邮件/站内 token 流程 |
| HC-07 PII 脱敏 | 前端不落盘明文 | 手机号、身份证、地址在本地存储中必须脱敏；高危页面禁用浏览器自动填充 |
| HC-08 网关保留头 | 客户端禁止伪造 | 客户端严禁发送 `X-User-Id`、`X-User-Role`；由网关注入，伪造返回 `E_REQ_4003` |
| HC-Auth 角色 | 仅两种角色 | `admin`（管理员）、`super_admin`（超级管理员）；按钮级权限通过 `v-permission` 指令 + `PermissionButton` 双层控制 |
| HC-ID-String | ID 字符串 | 所有 `task_id / patient_id / clue_id / user_id / order_id / tag_code / invite_id / transfer_request_id / event_id` 在前端 `string` 承载，禁止 `Number()` 转换 |

### 1.1 全局 Header 契约（与 API_V2.0.md §1.2 逐字对齐）

| Header | 注入位置 | 说明 |
| :--- | :--- | :--- |
| `Authorization: Bearer <access_token>` | Axios 请求拦截器 | 登录后从 `useAuthStore` 读取 |
| `X-Request-Id` | 写接口拦截器 | UUID v4，长度 16–64，仅 `[A-Za-z0-9-]` |
| `X-Trace-Id` | 所有请求拦截器 | 新请求生成；跨链路（如点击站内消息跳转）透传上游值 |
| `Content-Type: application/json` | JSON 请求 | 默认值 |
| `X-Anonymous-Token` | **禁止** | 管理端不使用匿名流 |
| `X-User-Id` / `X-User-Role` | **严禁** | 由网关注入，客户端发送会被拒绝（`E_REQ_4003`） |

### 1.2 统一响应外壳与错误语义

后端统一以 `{ code, message, trace_id, data }` 包装（API §1.4）。前端响应拦截器规则：

1. `code === "ok"` → 返回 `data`；
2. `code` 以 `E_` 开头 → 进入错误码映射（见 §14），非 401/403 默认 `message.error`；
3. `HTTP 401 / E_AUTH_4011` → 清空 token，跳转 `/login?redirect=...`；
4. `HTTP 403 / E_AUTH_4031` → 弹 `Modal.warning` 提示"越权访问"并记录 Trace-Id；
5. `HTTP 429 / E_REQ_4291` → 读取响应头 `Retry-After` / `X-RateLimit-Reset` 做指数退避；
6. 网络错误 → `notification.error` 附 Trace-Id（本地生成的）供用户复述。

---

## 2. 技术栈与依赖版本矩阵

### 2.1 基础栈（锁版本，禁止替换）

| 类别 | 选型 | 版本 | 备注 |
| :--- | :--- | :--- | :--- |
| 语言 | TypeScript | `~5.5.x` | `strict: true`、`noUncheckedIndexedAccess: true` |
| 框架 | Vue | `^3.5.0` | 组合式 API + `<script setup>` + `defineModel` |
| 构建 | Vite | `^6.0.0` | 默认 ES 模块、SWC 插件、分包策略 §2.4 |
| 路由 | Vue Router | `^4.4.0` | HTML5 history 模式；404 兜底 |
| 状态 | Pinia | `^3.0.0` | 配合 `pinia-plugin-persistedstate@^4` |
| 网络 | Axios | `^1.7.0` | 自研 `request.ts` 封装 |
| UI 组件 | Ant Design Vue | `^4.2.0` | 按需引入 + `unplugin-vue-components` |
| 图表 | ECharts | `^5.5.0` | `vue-echarts@^7` 渲染 |
| 国际化 | vue-i18n | `^9.14.0` | `legacy: false`，Composition 模式 |
| 日期 | dayjs | `^1.11.0` | `utc`、`timezone`、`relativeTime` 插件 |
| 工具 | lodash-es | `^4.17.21` | 按需 tree-shake；禁止全量引入 |
| VueUse | @vueuse/core | `^11.0.0` | `useStorage`、`useDark`、`usePreferredColorScheme` |
| 校验 | zod | `^3.23.0` | 对后端响应做运行时校验（调试模式启用） |
| 图标 | @iconify/vue | `^4.1.0` | 统一图标源 |
| 样式 | SCSS | `^1.79.0` | 仅写变量、mixin 与少量全局样式；组件样式走 `<style scoped>` |
| 原子样式 | UnoCSS | `^0.62.0` | 可选增强；禁止取代 AntDV Token |
| WebSocket | 原生 + `ReconnectingWebSocket` | `^4.4.0` | 心跳 30s、指数退避重连 |
| 富文本转义 | DOMPurify | `^3.1.0` | 任何后端富文本渲染必须过 DOMPurify |
| 测试 | Vitest + @vue/test-utils | `^2.x` / `^2.x` | 单元与组件测试 |
| E2E | Playwright | `^1.47.0` | 核心链路回归（登录、复核、工单发货、DEAD 重放） |
| 错误监控 | @sentry/vue | `^8.x` | 仅生产环境启用；SourceMap 仅上传 Sentry |

### 2.2 显式禁止依赖

| 禁止项 | 理由 |
| :--- | :--- |
| React / Element Plus / Tailwind（主方案） | 违反 HC-Stack |
| moment | 体积大，已用 dayjs 替代 |
| 自定义 BigInt 序列化库 | 服务端已强制 string 传输 ID（HC-ID-String） |
| `localStorage` 存 JWT 明文以外的 PII | 违反 HC-07 |

### 2.3 脚本约定（`package.json`）

```jsonc
{
  "scripts": {
    "dev": "vite",
    "build": "vue-tsc --noEmit && vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext .ts,.vue --max-warnings 0",
    "lint:style": "stylelint \"src/**/*.{vue,scss,css}\"",
    "typecheck": "vue-tsc --noEmit",
    "test": "vitest run",
    "test:watch": "vitest",
    "e2e": "playwright test",
    "i18n:extract": "node scripts/i18n-extract.mjs",
    "prepare": "husky"
  }
}
```

### 2.4 Vite 分包策略（关键片段）

```ts
// vite.config.ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import Components from 'unplugin-vue-components/vite';
import { AntDesignVueResolver } from 'unplugin-vue-components/resolvers';

export default defineConfig({
  plugins: [
    vue(),
    Components({ resolvers: [AntDesignVueResolver({ importStyle: false })] }),
  ],
  build: {
    sourcemap: 'hidden',
    rollupOptions: {
      output: {
        manualChunks: {
          'vendor-vue': ['vue', 'vue-router', 'pinia'],
          'vendor-antdv': ['ant-design-vue'],
          'vendor-echarts': ['echarts', 'vue-echarts'],
          'vendor-i18n': ['vue-i18n', 'dayjs'],
        },
      },
    },
  },
});
```

---

## 3. 工程目录结构

```text
web-admin/
├─ public/                         # 静态资源（favicon、robots.txt）
├─ scripts/                        # i18n 抽取、版本注入
├─ src/
│  ├─ api/                         # 按域划分的 API 封装（纯函数，无 UI）
│  │  ├─ auth.ts                   # /auth/*、/users/me
│  │  ├─ dashboard.ts              # /api/v1/dashboard 指标聚合
│  │  ├─ task.ts                   # /rescue/tasks*
│  │  ├─ clue.ts                   # /clues*
│  │  ├─ profile.ts                # /patients*
│  │  ├─ material.ts               # /material/orders*、/tags*
│  │  ├─ notification.ts           # /notifications*
│  │  ├─ admin.ts                  # /admin/configs*、/admin/logs
│  │  ├─ super.ts                  # /admin/super/*
│  │  └─ ws.ts                     # /ws/ticket
│  ├─ assets/                      # 图片、字体
│  ├─ components/                  # 跨页面组件
│  │  ├─ common/                   # EmptyState、ErrorBoundary、ThemeSwitch、LocaleSwitch
│  │  ├─ pro/                      # ProTable、ProForm、ProDescriptions
│  │  ├─ domain/                   # StatusBadge、DictTag、TraceDrawer、PermissionButton
│  │  └─ chart/                    # ChartCard（ECharts 主题联动）
│  ├─ composables/                 # useRequest / useTable / useWs / usePermission
│  ├─ directives/                  # v-permission、v-debounce
│  ├─ layouts/
│  │  ├─ AppLayout.vue             # 侧边栏 + 顶部栏 + 内容区
│  │  └─ BlankLayout.vue           # 登录/错误页
│  ├─ locales/
│  │  ├─ index.ts
│  │  ├─ zh-CN/
│  │  │  ├─ common.ts
│  │  │  ├─ menu.ts
│  │  │  ├─ task.ts
│  │  │  ├─ clue.ts
│  │  │  ├─ material.ts
│  │  │  ├─ gov.ts
│  │  │  └─ error.ts               # 错误码 → 文案
│  │  └─ en-US/ …                  # 同上
│  ├─ router/
│  │  ├─ index.ts
│  │  ├─ routes.ts                 # 静态路由 + 角色/权限 meta
│  │  └─ guards.ts                 # 鉴权 / 权限 / 标题 / 埋点
│  ├─ stores/                      # Pinia
│  │  ├─ auth.ts
│  │  ├─ app.ts                    # locale / theme / sidebar
│  │  ├─ dict.ts                   # 枚举字典缓存
│  │  ├─ notification.ts           # 站内消息队列
│  │  └─ ws.ts                     # WebSocket 订阅状态
│  ├─ styles/
│  │  ├─ tokens.scss               # 设计令牌（颜色/字号/间距）
│  │  ├─ antdv.ts                  # ConfigProvider theme.token
│  │  └─ global.scss
│  ├─ types/                       # 领域 DTO、枚举、错误码
│  │  ├─ dto/*.ts
│  │  └─ enums.ts                  # TaskState、ClueReviewState、OrderState、TagState
│  ├─ utils/                       # uuid、format、storage、download
│  ├─ views/                       # 页面
│  │  ├─ auth/LoginView.vue
│  │  ├─ dashboard/DashboardView.vue
│  │  ├─ task/TaskListView.vue
│  │  ├─ task/TaskDetailDrawer.vue
│  │  ├─ clue/ClueReviewView.vue
│  │  ├─ profile/PatientListView.vue
│  │  ├─ material/OrderListView.vue
│  │  ├─ material/TagInventoryView.vue
│  │  ├─ notification/InboxView.vue
│  │  ├─ admin/ConfigView.vue
│  │  ├─ admin/AuditLogView.vue
│  │  ├─ admin/DeadLetterView.vue
│  │  ├─ me/ProfileView.vue
│  │  └─ error/{401,403,404,500}.vue
│  ├─ App.vue
│  └─ main.ts
├─ tests/                          # vitest + playwright
├─ .env.development
├─ .env.production
├─ env.d.ts
├─ tsconfig.json
├─ vite.config.ts
└─ package.json
```

### 3.1 分层约束

1. `views/*` → 仅组合 `components` + `composables`，不直接调 `axios`。
2. `api/*` → 仅封装 `request.ts` + zod schema，不含 UI 状态。
3. `stores/*` → 不发网络请求；由 `composables/useXXX.ts` 组合 API 与 store。
4. 跨域访问 → 通过 `composables` 层，不允许 `views/task` 直接 `import stores/clue`。
5. `types/dto/*.ts` 与后端 DTO 字段 1:1 对应（`snake_case`），避免前端擅自改名；UI 侧通过 computed 再转为驼峰。

---

## 4. 主题与设计令牌

### 4.1 品牌与语义色

| 令牌 | 亮色值 | 暗色值 | 说明 |
| :--- | :--- | :--- | :--- |
| `--color-primary` | `#F97316` | `#F97316` | 主色（品牌橙，贯穿按钮、主要高亮、ECharts 主轴） |
| `--color-primary-hover` | `#FB923C` | `#FDBA74` | 主色悬停 |
| `--color-primary-active` | `#EA580C` | `#F97316` | 主色按下 |
| `--color-link` | `#0EA5E9` | `#38BDF8` | 链接与数据可视化辅轴 |
| `--color-success` | `#22C55E` | `#4ADE80` | 成功 |
| `--color-warning` | `#F59E0B` | `#FBBF24` | 警告 |
| `--color-error` | `#EF4444` | `#F87171` | 错误 |
| `--color-info` | `#0EA5E9` | `#38BDF8` | 信息 |
| `--bg-layout` | `#F5F7FA` | `#141414` | 页面背景 |
| `--bg-container` | `#FFFFFF` | `#1F1F1F` | 容器背景 |
| `--bg-elevated` | `#FFFFFF` | `#262626` | 抽屉/弹窗浮层 |
| `--text-1` | `rgba(0,0,0,0.88)` | `rgba(255,255,255,0.88)` | 主文本 |
| `--text-2` | `rgba(0,0,0,0.65)` | `rgba(255,255,255,0.65)` | 次文本 |
| `--text-3` | `rgba(0,0,0,0.45)` | `rgba(255,255,255,0.45)` | 占位/禁用 |
| `--border-1` | `rgba(0,0,0,0.08)` | `rgba(255,255,255,0.12)` | 分隔线 |
| `--shadow-card` | `0 2px 8px rgba(0,0,0,0.06)` | `0 2px 8px rgba(0,0,0,0.32)` | 卡片阴影 |

### 4.2 排版与尺寸

| 项 | 值 |
| :--- | :--- |
| 字体族 | `-apple-system, "Segoe UI", "PingFang SC", "Microsoft YaHei", Roboto, sans-serif` |
| 基准字号 | 14px（行高 22px） |
| 字号阶 | 12 / 14 / 16 / 20 / 24 / 30 / 38 |
| 圆角 | 基础 8px，大容器 12px，全圆 9999px |
| 间距阶 | 4 / 8 / 12 / 16 / 24 / 32 / 48 |
| 主要容器最大宽度 | 1440px（超宽屏用 AntDV Grid 24 栅格） |

### 4.3 AntDV `ConfigProvider` 接入骨架

```ts
// src/styles/antdv.ts
import { theme, type ThemeConfig } from 'ant-design-vue';

const commonToken = {
  colorPrimary: '#F97316',
  colorLink: '#0EA5E9',
  colorSuccess: '#22C55E',
  colorWarning: '#F59E0B',
  colorError: '#EF4444',
  borderRadius: 8,
  wireframe: false,
};

export const lightTheme: ThemeConfig = {
  algorithm: theme.defaultAlgorithm,
  token: { ...commonToken, colorBgLayout: '#F5F7FA' },
};

export const darkTheme: ThemeConfig = {
  algorithm: theme.darkAlgorithm,
  token: { ...commonToken, colorBgLayout: '#141414', colorBgContainer: '#1F1F1F' },
};
```

```vue
<!-- src/App.vue -->
<script setup lang="ts">
import { computed } from 'vue';
import { ConfigProvider } from 'ant-design-vue';
import zhCN from 'ant-design-vue/es/locale/zh_CN';
import enUS from 'ant-design-vue/es/locale/en_US';
import { useAppStore } from '@/stores/app';
import { lightTheme, darkTheme } from '@/styles/antdv';

const app = useAppStore();
const antdvLocale = computed(() => (app.locale === 'zh-CN' ? zhCN : enUS));
const antdvTheme  = computed(() => (app.effectiveTheme === 'dark' ? darkTheme : lightTheme));
</script>

<template>
  <ConfigProvider :locale="antdvLocale" :theme="antdvTheme">
    <RouterView />
  </ConfigProvider>
</template>
```

### 4.4 主题切换策略

1. `useAppStore.themeMode: 'light' | 'dark' | 'system'`，持久化 `localStorage.app.theme`。
2. `system` 模式监听 `usePreferredColorScheme()` 自动切换。
3. 切换时同步：AntDV `theme.algorithm`、ECharts 主题（`registerTheme('mark-light' | 'mark-dark')`）、`document.documentElement.dataset.theme`。
4. ECharts 图表由 `<ChartCard>` 统一消费 `useEchartsTheme()`，无需各页面重复处理。

### 4.5 ECharts 主题配色（必须统一）

| 用途 | 值 |
| :--- | :--- |
| 主轴/主系列 | `#F97316` |
| 辅轴/次系列 | `#0EA5E9` |
| 调色板（多系列） | `['#F97316', '#0EA5E9', '#22C55E', '#A855F7', '#F59E0B', '#64748B']` |
| 网格线（亮/暗） | `#E5E7EB` / `#303030` |
| 背景（亮/暗） | `transparent` / `transparent`（跟随容器） |

---

## 5. 国际化（i18n）

### 5.1 方案概览

1. `vue-i18n@9` Composition 模式（`legacy: false`，`fallbackLocale: 'zh-CN'`）。
2. 语言包按页面模块拆分：`common`、`menu`、`task`、`clue`、`profile`、`material`、`gov`、`error`、`enum`。
3. 所有可见文本（含按钮、占位符、表头、消息、错误）必须走 `t()`，严禁硬编码中文 / 英文。
4. 字典枚举（`TaskState`、`ClueReviewState` 等）文案统一放在 `locales/*/enum.ts`。
5. 错误码映射统一放在 `locales/*/error.ts`。

### 5.2 Key 命名规范

| 层次 | 示例 |
| :--- | :--- |
| 页面级 | `page.task.list.title` |
| 组件级 | `comp.proTable.empty` |
| 动作级 | `page.clue.detail.btn.override` |
| 字段级 | `field.task.state` |
| 枚举级 | `enum.taskState.ACTIVE` |
| 错误级 | `error.E_TASK_4091` |
| 全局级 | `common.confirm / common.cancel / common.save / common.export` |

### 5.3 初始化骨架

```ts
// src/locales/index.ts
import { createI18n } from 'vue-i18n';
import zhCN from './zh-CN';
import enUS from './en-US';

export type AppLocale = 'zh-CN' | 'en-US';

export const i18n = createI18n({
  legacy: false,
  locale: 'zh-CN',
  fallbackLocale: 'zh-CN',
  messages: { 'zh-CN': zhCN, 'en-US': enUS },
  missingWarn: import.meta.env.DEV,
});

export async function setLocale(locale: AppLocale) {
  i18n.global.locale.value = locale;
  const dayjs = (await import('dayjs')).default;
  await import(`dayjs/locale/${locale === 'zh-CN' ? 'zh-cn' : 'en'}.js`);
  dayjs.locale(locale === 'zh-CN' ? 'zh-cn' : 'en');
  document.documentElement.lang = locale;
}
```

### 5.4 AntDV / ECharts / 错误提示的联动

| 联动点 | 策略 |
| :--- | :--- |
| AntDV 组件文案 | `<ConfigProvider :locale="antdvLocale">`，locale 根据 `useAppStore.locale` 切换 |
| 日期 | `dayjs.locale('zh-cn' | 'en')` 统一，展示组件用 `useFormatDate()` 包装 |
| 数字 | `Intl.NumberFormat(locale)` 包装于 `formatNumber()` |
| ECharts 图例 | 组件 `:option` 依赖 `t()` 返回值，`locale` 变化后自动 `watchEffect` 触发重渲 |
| 消息错误 | `error.ts` 映射 `code → i18n key` |

### 5.5 `LocaleSwitch` 组件规范

```vue
<!-- src/components/common/LocaleSwitch.vue -->
<script setup lang="ts">
import { useAppStore } from '@/stores/app';
import { setLocale } from '@/locales';
import { TranslationOutlined } from '@ant-design/icons-vue';
const app = useAppStore();
async function onChange(val: 'zh-CN' | 'en-US') {
  app.locale = val;
  await setLocale(val);
}
</script>

<template>
  <a-dropdown>
    <a-button type="text" :aria-label="$t('common.switchLocale')">
      <template #icon><TranslationOutlined /></template>
      {{ app.locale === 'zh-CN' ? '中文' : 'EN' }}
    </a-button>
    <template #overlay>
      <a-menu @click="({ key }) => onChange(key as 'zh-CN' | 'en-US')">
        <a-menu-item key="zh-CN">中文</a-menu-item>
        <a-menu-item key="en-US">English</a-menu-item>
      </a-menu>
    </template>
  </a-dropdown>
</template>
```

### 5.6 翻译工作流（强制）

1. 新字段必须在 `zh-CN` 与 `en-US` 同步；CI 脚本 `scripts/i18n-extract.mjs` 对缺失 key 失败。
2. PR 模板含"i18n keys 是否同步 (zh-CN / en-US)"强制勾选项。
3. 不允许 `t(\`${dynamic}.label\`)` 式动态 key 拼接（违反抽取静态分析）；改用映射表。

---

## 6. 全局布局（AppLayout）

### 6.1 低保真线框

```text
┌──────────────────────────────────────────────────────────────────────────┐
│ Logo「码上回家」  · 全局搜索                 · 🔔通知  🌐语言  🌙主题  👤用户 │
├──────────┬───────────────────────────────────────────────────────────────┤
│ 首页      │ 面包屑：首页 / 任务治理 / 任务详情                              │
│ 任务治理  │ ┌──────────────────────────────────────────────────────────┐ │
│ 线索复核  │ │   查询表单（ProForm）                                     │ │
│ 患者档案  │ │   [发布时间] [状态] [地区] [患者模糊搜索]  [查询][重置]    │ │
│ 物资工单  │ ├──────────────────────────────────────────────────────────┤ │
│ 标签库存  │ │   操作条：[新建(条件)][批量导出][刷新]       列配置⚙      │ │
│ 通知中心  │ ├──────────────────────────────────────────────────────────┤ │
│ 系统配置  │ │   数据表格（ProTable，Cursor/Offset 分页）                │ │
│ 审计日志  │ └──────────────────────────────────────────────────────────┘ │
│ DEAD 事件 │                                                              │
│ 个人中心  │                                                              │
└──────────┴───────────────────────────────────────────────────────────────┘
```

### 6.2 组件职责

| 区块 | 组件 | 要点 |
| :--- | :--- | :--- |
| 顶部栏 | `AppHeader` | Logo「码上回家」、全局搜索（任务/线索/工单）、通知铃铛（`useNotificationStore`）、`LocaleSwitch`、`ThemeSwitch`、用户菜单（个人中心、退出登录） |
| 侧边栏 | `AppSider` | 可折叠；根据角色动态渲染；当前路由高亮；图标来自 @iconify/vue |
| 面包屑 | `AppBreadcrumb` | 根据 `route.matched` 自动生成，支持 i18n |
| 内容区 | `<router-view v-slot="{ Component }">` | KeepAlive 缓存列表页（`meta.keepAlive`），详情页不缓存 |

### 6.3 响应式断点

| 断点 | 宽度 | 侧边栏行为 |
| :--- | :--- | :--- |
| `< 768px` | 手机 | 不支持（管理端不做手机适配，提示"请在桌面端访问"） |
| `768–1200px` | 平板 | 侧边栏自动折叠为图标模式 |
| `> 1200px` | 桌面 | 侧边栏默认展开 |

---

## 7. 路由与菜单

### 7.1 路由规范

| 约定 | 值 |
| :--- | :--- |
| 模式 | HTML5 history |
| 基础路径 | `/` |
| 所有路由 | 强制懒加载 `() => import(...)` |
| `meta.title` | i18n key（如 `menu.task.list`） |
| `meta.roles` | `('admin' | 'super_admin')[]` |
| `meta.permissions` | `string[]`（可选，按钮级检查通常在组件内） |
| `meta.keepAlive` | 仅列表页 `true` |
| `meta.icon` | @iconify 名称 |

### 7.2 完整路由清单

| 路径 | 视图 | 角色 | KeepAlive |
| :--- | :--- | :--- | :---: |
| `/login` | `auth/LoginView` | 匿名 | — |
| `/` | 重定向 `/dashboard` | — | — |
| `/dashboard` | `dashboard/DashboardView` | admin / super_admin | ✓ |
| `/tasks` | `task/TaskListView` | admin / super_admin | ✓ |
| `/tasks/:taskId` | `task/TaskDetailView` | admin / super_admin | — |
| `/clues` | `clue/ClueListView` | admin / super_admin | ✓ |
| `/clues/review` | `clue/ClueReviewView` | admin / super_admin | ✓ |
| `/patients` | `profile/PatientListView` | admin / super_admin | ✓ |
| `/patients/:patientId` | `profile/PatientDetailView` | admin / super_admin | — |
| `/material/orders` | `material/OrderListView` | admin / super_admin | ✓ |
| `/material/orders/:orderId` | `material/OrderDetailView` | admin / super_admin | — |
| `/tags/inventory` | `material/TagInventoryView` | admin / super_admin | ✓ |
| `/tags/batch-jobs/:jobId` | `material/TagBatchJobView` | admin / super_admin | — |
| `/notifications` | `notification/InboxView` | admin / super_admin | ✓ |
| `/admin/configs` | `admin/ConfigView` | admin（只读） / super_admin（可写） | ✓ |
| `/admin/logs` | `admin/AuditLogView` | admin / super_admin | ✓ |
| `/admin/dead-letter` | `admin/DeadLetterView` | super_admin | ✓ |
| `/me` | `me/ProfileView` | admin / super_admin | — |
| `/401` `/403` `/404` `/500` | `error/*` | 匿名 | — |

### 7.3 菜单 → 路由 → 角色 映射表

| 菜单 i18n | 路径 | 角色 | 备注 |
| :--- | :--- | :--- | :--- |
| `menu.dashboard` | `/dashboard` | admin / super_admin | KPI 与告警 |
| `menu.task` | `/tasks` | admin / super_admin | 任务治理列表 |
| `menu.clue.review` | `/clues/review` | admin / super_admin | 待复核队列（默认过滤 `review_state = PENDING_REVIEW`） |
| `menu.clue.all` | `/clues` | admin / super_admin | 全部线索 |
| `menu.patient` | `/patients` | admin / super_admin | 患者档案（只读视图） |
| `menu.material.order` | `/material/orders` | admin / super_admin | 工单审批/发货 |
| `menu.material.tag` | `/tags/inventory` | admin / super_admin | 标签库存 + 批量发号 |
| `menu.notification` | `/notifications` | admin / super_admin | 站内消息 |
| `menu.admin.config` | `/admin/configs` | admin / super_admin | admin 只读，super_admin 可改 |
| `menu.admin.log` | `/admin/logs` | admin / super_admin | 审计日志 |
| `menu.admin.dead` | `/admin/dead-letter` | **super_admin 独占** | DEAD 重放 |

### 7.4 路由守卫（骨架）

```ts
// src/router/guards.ts
import type { Router } from 'vue-router';
import { useAuthStore } from '@/stores/auth';

export function setupGuards(router: Router) {
  router.beforeEach(async (to) => {
    const auth = useAuthStore();
    if (to.path !== '/login' && !auth.accessToken) {
      return { path: '/login', query: { redirect: to.fullPath } };
    }
    if (!auth.user && auth.accessToken) {
      await auth.fetchCurrentUser();
    }
    const allowed = to.meta.roles as string[] | undefined;
    if (allowed && !allowed.includes(auth.user?.role ?? '')) {
      return { path: '/403' };
    }
    return true;
  });

  router.afterEach((to) => {
    document.title = `${to.meta.title ?? '码上回家'} · Admin`;
  });
}
```

---

## 8. 状态管理（Pinia）

### 8.1 Store 清单

| Store | 职责 | 持久化 |
| :--- | :--- | :--- |
| `useAuthStore` | `accessToken` / `refreshToken` / `user{id, username, role, displayName}` / `permissions[]` | 仅 access/refresh token 持久化到 `localStorage.auth`；user、permissions 登录后每次拉取 |
| `useAppStore` | `locale` / `themeMode` / `effectiveTheme` / `sidebarCollapsed` | 全部持久化到 `localStorage.app` |
| `useDictStore` | 配置字典（`sys_config` 只读快照）、枚举颜色映射 | 会话级内存（刷新重载） |
| `useNotificationStore` | 站内消息未读数、最新 20 条预览、WS 推入的实时通知 | 不持久化 |
| `useWsStore` | WS 连接状态、订阅频道列表、最近一条错误 | 不持久化 |

### 8.2 `useAuthStore` 骨架

```ts
// src/stores/auth.ts
import { defineStore } from 'pinia';
import * as authApi from '@/api/auth';

export type Role = 'admin' | 'super_admin';

export interface CurrentUser {
  user_id: string;
  username: string;
  display_name: string;
  role: Role;
  permissions: string[];
}

export const useAuthStore = defineStore('auth', {
  state: () => ({
    accessToken: '' as string,
    refreshToken: '' as string,
    user: null as CurrentUser | null,
  }),
  getters: {
    isLoggedIn: (s) => !!s.accessToken,
    isSuperAdmin: (s) => s.user?.role === 'super_admin',
    permissionSet: (s) => new Set(s.user?.permissions ?? []),
  },
  actions: {
    async login(payload: { username: string; password: string }) {
      const data = await authApi.login(payload);
      this.accessToken = data.access_token;
      this.refreshToken = data.refresh_token;
      await this.fetchCurrentUser();
    },
    async fetchCurrentUser() {
      this.user = await authApi.getCurrentUser();
    },
    hasPermission(code: string | string[]) {
      if (this.user?.role === 'super_admin') return true;
      const codes = Array.isArray(code) ? code : [code];
      return codes.some((c) => this.permissionSet.has(c));
    },
    async logout() {
      this.$reset();
      localStorage.removeItem('auth');
    },
  },
  persist: {
    key: 'auth',
    pick: ['accessToken', 'refreshToken'],
  },
});
```

### 8.3 `useAppStore` 骨架

```ts
// src/stores/app.ts
import { defineStore } from 'pinia';
import { usePreferredColorScheme } from '@vueuse/core';

type ThemeMode = 'light' | 'dark' | 'system';

export const useAppStore = defineStore('app', {
  state: () => ({
    locale: 'zh-CN' as 'zh-CN' | 'en-US',
    themeMode: 'system' as ThemeMode,
    sidebarCollapsed: false,
  }),
  getters: {
    effectiveTheme(s): 'light' | 'dark' {
      if (s.themeMode !== 'system') return s.themeMode;
      return usePreferredColorScheme().value === 'dark' ? 'dark' : 'light';
    },
  },
  persist: { key: 'app' },
});
```

### 8.4 状态一致性准则

1. 一切服务端状态以后端为权威（HC-02）；前端缓存仅用于展示加速，不得参与业务决策。
2. 列表页组件内部状态（查询条件、分页、排序、列显隐）由 `useTable(key)` 统一托管到 `sessionStorage`，key 命名为 `table.<module>.<view>`。
3. 同一用户登录后切换账号必须完全清空 `localStorage`（除保留的 `app.locale` / `app.themeMode`）。
4. 401 响应必须清理 `auth`，并保留当前路径至 `redirect` 查询参数。

---

## 9. HTTP 层（Axios 封装）

### 9.1 `request.ts` 实现骨架

```ts
// src/utils/request.ts
import axios, { type AxiosRequestConfig } from 'axios';
import { v4 as uuid } from 'uuid';
import { message, Modal, notification } from 'ant-design-vue';
import router from '@/router';
import { useAuthStore } from '@/stores/auth';
import { i18n } from '@/locales';

export const request = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL,
  timeout: 15000,
  transformResponse: [(data) => data], // 保持 string，避免 JSON.parse 丢长整型精度
});

// -- 请求拦截器 ----------------------------------------------------------
request.interceptors.request.use((config) => {
  const auth = useAuthStore();
  if (auth.accessToken) {
    config.headers.set('Authorization', `Bearer ${auth.accessToken}`);
  }
  // HC-04 Trace-Id
  const traceId = (config.headers.get('X-Trace-Id') as string | undefined) || `trc_${uuid().replace(/-/g, '').slice(0, 24)}`;
  config.headers.set('X-Trace-Id', traceId);

  // HC-03 Request-Id（写接口）
  const method = (config.method ?? 'get').toLowerCase();
  if (['post', 'put', 'delete', 'patch'].includes(method)) {
    if (!config.headers.get('X-Request-Id')) {
      config.headers.set('X-Request-Id', `req_${uuid().replace(/-/g, '').slice(0, 24)}`);
    }
  }

  // HC-08 禁止伪造保留头
  config.headers.delete('X-User-Id');
  config.headers.delete('X-User-Role');

  config.headers.set('Content-Type', 'application/json');
  return config;
});

// -- 响应拦截器 ----------------------------------------------------------
request.interceptors.response.use(
  (resp) => {
    const parsed = safeParse(resp.data);
    const traceId = resp.headers['x-trace-id'] ?? parsed?.trace_id;
    if (parsed?.code === 'ok') return parsed.data;
    return handleBizError(parsed, traceId, resp.status);
  },
  (err) => {
    const parsed = safeParse(err?.response?.data);
    const traceId = err?.response?.headers?.['x-trace-id'] ?? parsed?.trace_id ?? 'N/A';
    return handleBizError(parsed, traceId, err?.response?.status ?? 0, err);
  },
);

function safeParse(data: unknown): any {
  if (typeof data !== 'string') return data;
  try { return JSON.parse(data); } catch { return null; }
}

function handleBizError(body: any, traceId: string, status: number, raw?: unknown): Promise<never> {
  const code: string = body?.code ?? `E_HTTP_${status || 'NET'}`;
  const msg  = i18n.global.te(`error.${code}`)
    ? i18n.global.t(`error.${code}`)
    : (body?.message ?? i18n.global.t('error.UNKNOWN'));

  if (status === 401 || code === 'E_AUTH_4011') {
    useAuthStore().$reset();
    router.push({ path: '/login', query: { redirect: router.currentRoute.value.fullPath } });
  } else if (status === 403 || code === 'E_AUTH_4031') {
    Modal.warning({ title: i18n.global.t('error.forbidden'), content: `${msg}\nTrace: ${traceId}` });
  } else if (status === 429) {
    notification.warning({ message: i18n.global.t('error.rateLimited'), description: msg });
  } else {
    message.error(`${msg} (Trace: ${String(traceId).slice(0, 12)})`);
  }

  return Promise.reject({ code, message: msg, traceId, raw });
}
```

### 9.2 ID / 长整型处理（关键）

1. `transformResponse` 保持原始 string，不要依赖 Axios 默认 JSON.parse（会把 `"1234567890123456789"` 截断为 Number）。
2. 前端使用 `JSON.parse` 不会丢精度，因为字符串 ID 本身就是 `"xxx"`。**严禁**对 `task_id` 等做 `Number(id)` 或 `parseInt(id)`。
3. 表格 rowKey 统一用 `record => record.xxx_id`（string）。
4. URL 路径拼接：``/api/v1/rescue/tasks/${encodeURIComponent(taskId)}/snapshot``。

### 9.3 API 封装模板

```ts
// src/api/task.ts
import { request } from '@/utils/request';

export interface TaskSnapshot {
  task_id: string;
  patient_id: string;
  state: 'CREATED' | 'ACTIVE' | 'SUSTAINED' | 'CLOSED_FOUND' | 'CLOSED_FALSE_ALARM';
  // … 与 API §3.1.3 字段对齐
}

export const taskApi = {
  list: (params: Record<string, unknown>) =>
    request.get<unknown, { items: TaskSnapshot[]; next_cursor?: string; total?: number }>(
      '/api/v1/rescue/tasks', { params },
    ),
  snapshot: (taskId: string) =>
    request.get<unknown, TaskSnapshot>(`/api/v1/rescue/tasks/${taskId}/snapshot`),
  close: (taskId: string, body: { reason: string; close_type: 'FOUND' | 'FALSE_ALARM' }) =>
    request.post<unknown, TaskSnapshot>(`/api/v1/rescue/tasks/${taskId}/close`, body),
  sustained: (taskId: string, body: { memo?: string }) =>
    request.post<unknown, TaskSnapshot>(`/api/v1/rescue/tasks/${taskId}/sustained`, body),
  fullAggregate: (taskId: string) =>
    request.get<unknown, unknown>(`/api/v1/rescue/tasks/${taskId}/full`),
};
```

### 9.4 幂等与重试

1. 表单提交按钮在请求未返回前置 `loading=true` 且不可重复点击。
2. 如用户手动"重试"，**必须复用原 `X-Request-Id`**；由 `useRequest` 组合式缓存同一 payload 的 request id。
3. 服务端幂等窗口见 API §1.6，前端不得触发超过窗口的重复提交。

### 9.5 环境变量

| 变量 | 示例 | 说明 |
| :--- | :--- | :--- |
| `VITE_API_BASE_URL` | `https://api.codehome.example.com` | 后端 API 根 |
| `VITE_WS_BASE_URL` | `wss://api.codehome.example.com/ws` | WebSocket 根 |
| `VITE_CDN_BASE_URL` | `https://cdn.codehome.example.com` | 静态资源 CDN（构建 `base`） |
| `VITE_APP_VERSION` | `2.0.0-RC1` | 构建注入，显示于关于页 |
| `VITE_SENTRY_DSN` | `https://xxx@sentry.io/yyy` | 仅生产启用 |

---

## 10. WebSocket 客户端

### 10.1 连接握手（对齐 API §4）

1. 调 `POST /api/v1/ws/ticket` 换取一次性 `ticket`（短有效期）。
2. 建连 `${VITE_WS_BASE_URL}?ticket=<ticket>`。
3. 连接成功后发送 `subscribe` 帧，声明订阅频道（HC-05 定向订阅）。

### 10.2 订阅频道白名单（管理端）

| 频道 | 触发场景 | 消费者 |
| :--- | :--- | :--- |
| `user.{user_id}.notification` | 站内消息推送 | `useNotificationStore.pushRealtime()` |
| `task.{task_id}.state` | 用户进入任务详情页后订阅该任务的状态变更 / 线索入库 / 轨迹更新 | `TaskDetailView` |
| `admin.dashboard.metrics` | 仪表盘打开时订阅 KPI 节流推送 | `DashboardView` |
| `admin.dead.letter` | 打开 DEAD 页时订阅新增死信事件 | `DeadLetterView` |

> 离开页面时必须 `unsubscribe`；禁止订阅 `*` 或 `admin.*` 等通配频道。

### 10.3 心跳与重连

| 项 | 值 |
| :--- | :--- |
| 心跳 | 客户端每 30s 发 `{ "type": "ping" }`；60s 未收到 `pong` 断开重连 |
| 重连 | 指数退避 `1s → 2s → 5s → 10s → 30s`，上限 30s；连续 10 次失败弹告警 |
| 降级 | 2 次重连失败后对当前页启用 10s 轮询（仅详情/仪表盘），连接恢复后停止轮询 |
| Ticket 过期 | 重连前若上次 ticket 超过 5 分钟自动重新换票 |

### 10.4 `useWs` 骨架

```ts
// src/composables/useWs.ts
import ReconnectingWebSocket from 'reconnecting-websocket';
import { onScopeDispose } from 'vue';
import { wsApi } from '@/api/ws';

type Frame = { type: string; topic?: string; data?: unknown };

export function useWs() {
  let ws: ReconnectingWebSocket | null = null;
  const subs = new Set<string>();

  async function connect() {
    const { ticket } = await wsApi.getTicket();
    ws = new ReconnectingWebSocket(`${import.meta.env.VITE_WS_BASE_URL}?ticket=${ticket}`);
    ws.addEventListener('open', () => subs.forEach(subscribe));
  }

  function subscribe(topic: string) {
    subs.add(topic);
    ws?.send(JSON.stringify({ type: 'subscribe', topic } satisfies Frame));
  }
  function unsubscribe(topic: string) {
    subs.delete(topic);
    ws?.send(JSON.stringify({ type: 'unsubscribe', topic } satisfies Frame));
  }
  function on(topic: string, handler: (data: unknown) => void) {
    ws?.addEventListener('message', (ev) => {
      const frame = JSON.parse(ev.data) as Frame;
      if (frame.topic === topic) handler(frame.data);
    });
  }

  onScopeDispose(() => { subs.forEach(unsubscribe); ws?.close(); });

  return { connect, subscribe, unsubscribe, on };
}
```

### 10.5 安全与脱敏

1. WS 通道禁止承载明文 PII；后端必要时仅推送 `patient_id` 引用，前端按需再查 REST。
2. 若客户端收到未订阅频道的消息，必须丢弃并上报告警（防止越权推送）。
3. 浏览器标签页不可见超过 5 分钟自动断开，回到可见时重建。

---

## 11. 权限体系（按钮级）

### 11.1 角色模型

| 角色 | code | 范围 |
| :--- | :--- | :--- |
| 管理员 | `admin` | 任务治理、线索复核、患者档案（只读）、物资工单（审批/发货）、标签库存（只读）、系统配置（只读）、审计日志 |
| 超级管理员 | `super_admin` | 管理员全部权限 + DEAD 事件重放、系统配置修改、标签批量发号、强制操作等 |

### 11.2 权限码命名规范

`<domain>:<action>`，小写蛇形：

| 权限码 | 说明 |
| :--- | :--- |
| `task:read` / `task:close` / `task:sustain` | 任务读/关闭/标长期维持 |
| `clue:read` / `clue:override` / `clue:reject` | 线索读/覆写/驳回 |
| `patient:read` | 患者档案只读 |
| `material.order:read` / `material.order:approve` / `material.order:ship` | 工单审批/发货 |
| `tag:read` / `tag:batch-generate` | 标签读/批量发号（super_admin 独占） |
| `config:read` / `config:write` | 系统配置（write 仅 super_admin） |
| `audit:read` | 审计日志查询 |
| `dead:read` / `dead:replay` | DEAD 查询/重放（super_admin 独占） |
| `notification:read` | 站内消息 |

> `super_admin` 隐式持有通配权限 `*`，`hasPermission()` 直接放行。

### 11.3 `v-permission` 指令

```ts
// src/directives/permission.ts
import type { Directive } from 'vue';
import { useAuthStore } from '@/stores/auth';

export const vPermission: Directive<HTMLElement, string | string[]> = {
  mounted(el, binding) {
    const auth = useAuthStore();
    if (!auth.hasPermission(binding.value)) {
      el.parentNode?.removeChild(el); // 隐藏型：不渲染
    }
  },
};
```

使用：

```vue
<a-button type="primary" v-permission="'clue:override'" @click="onOverride">
  {{ $t('page.clue.btn.override') }}
</a-button>
```

### 11.4 `PermissionButton` 组件（降级型）

与 `v-permission` 区别：

| 场景 | 使用 |
| :--- | :--- |
| 无权限时直接隐藏 | `v-permission` |
| 无权限时置灰并 tooltip 提示 | `<PermissionButton code="config:write">` |

```vue
<!-- src/components/domain/PermissionButton.vue -->
<script setup lang="ts">
import { computed } from 'vue';
import { useAuthStore } from '@/stores/auth';
import { useI18n } from 'vue-i18n';
const props = defineProps<{ code: string | string[] }>();
const auth = useAuthStore();
const { t } = useI18n();
const allowed = computed(() => auth.hasPermission(props.code));
</script>

<template>
  <a-tooltip :title="allowed ? '' : t('common.noPermission')">
    <a-button v-bind="$attrs" :disabled="!allowed"><slot /></a-button>
  </a-tooltip>
</template>
```

### 11.5 双层控制（强制）

| 层 | 目的 |
| :--- | :--- |
| 菜单 / 路由 `meta.roles` | 粗粒度：避免进入无权限页面 |
| 按钮 `v-permission` / `PermissionButton` | 细粒度：保护高危操作 |
| 后端网关 | 最终权威：前端限制被绕过时仍须失败 |

### 11.6 二次确认矩阵（高危操作）

| 操作 | 确认方式 | 需输入 |
| :--- | :--- | :--- |
| 任务强制关闭 | `Modal.confirm` + 输入"确认关闭" | 文本串 |
| 线索覆写/驳回 | `Modal.confirm` + 复核理由（必填） | 30–500 字理由 |
| 工单发货 | `Modal.confirm` + 勾选"已核对物流单号" | 复选 |
| 系统配置修改 | `Modal.confirm` + 展示 diff | 无 |
| DEAD 事件重放 | `Modal.confirm` + 输入事件 ID 后 4 位 | 文本串 |
| 批量发号 | `Modal.confirm` + 输入数量 | 数量 |

所有二次确认必须记录 Trace-Id，失败时展示在 `notification.error` 中便于联调。

---

## 12. 公共组件清单

### 12.1 Pro 系列

| 组件 | 路径 | 功能 |
| :--- | :--- | :--- |
| `ProTable` | `components/pro/ProTable.vue` | 封装 `a-table` + 查询表单 + 列配置 + Cursor/Offset 切换 + 导出 + 空/错/载态 |
| `ProForm` | `components/pro/ProForm.vue` | schema 驱动（fields: [{ name, label, type, rules, span }]），内置校验、重置、条件显示 |
| `ProDescriptions` | `components/pro/ProDescriptions.vue` | 详情描述列表，支持字典、日期、脱敏、链接格式化 |
| `ProDrawer` | `components/pro/ProDrawer.vue` | 统一抽屉（详情、编辑、时间轴） |

### 12.2 Domain 组件

| 组件 | 用途 |
| :--- | :--- |
| `StatusBadge` | 按枚举（TaskState / ClueReviewState / OrderState / TagState）渲染带色圆点 + i18n 标签 |
| `DictTag` | 字典值着色（对应 `useDictStore`） |
| `TraceDrawer` | 输入 `trace_id` 弹抽屉，展示后端 `/admin/logs?trace_id=` 查询结果 |
| `PermissionButton` | 见 §11.4 |
| `MaskedText` | 脱敏显示手机号/身份证，点击眼睛图标并记录审计（需 `pii:reveal` 权限，未实现则禁用） |
| `TaskStateTimeline` | 任务状态机时间轴，基于 `full` 聚合接口 |
| `OrderTimeline` | 工单流转 timeline |

### 12.3 Common 组件

| 组件 | 用途 |
| :--- | :--- |
| `ThemeSwitch` | 亮 / 暗 / 跟随系统 三态切换 |
| `LocaleSwitch` | zh-CN / en-US |
| `EmptyState` | 统一空态图文 + 可选操作按钮 |
| `ErrorBoundary` | 捕获子树错误，上报 Sentry，展示"错误 + Trace-Id + 刷新"按钮 |
| `ChartCard` | ECharts 容器；亮/暗主题自动切换；`loading` / `empty` / `error` 三态 |
| `CopyableText` | 对长 ID 一键复制 |

### 12.4 `ProTable` 骨架

```vue
<!-- src/components/pro/ProTable.vue（节选） -->
<script setup lang="ts" generic="T extends Record<string, unknown>">
import type { TableColumnsType } from 'ant-design-vue';
defineProps<{
  columns: TableColumnsType<T>;
  fetcher: (params: Record<string, unknown>) => Promise<{ items: T[]; total?: number; next_cursor?: string }>;
  rowKey: keyof T | ((r: T) => string);
  initialFilters?: Record<string, unknown>;
  pagination?: 'offset' | 'cursor';
}>();
// ... useTable(key) 托管查询/分页/列配置
</script>

<template>
  <a-card :bordered="false">
    <slot name="filter" />
    <div class="pro-table__toolbar"><slot name="toolbar" /></div>
    <a-table v-bind="tableProps" :loading="loading" :data-source="items" :columns="columns" :row-key="rowKey">
      <template #emptyText><EmptyState :title="$t('comp.proTable.empty')" /></template>
    </a-table>
  </a-card>
</template>
```

### 12.5 `ChartCard` 亮暗主题联动

```vue
<script setup lang="ts">
import { computed } from 'vue';
import VueECharts from 'vue-echarts';
import { useAppStore } from '@/stores/app';
const app = useAppStore();
const themeName = computed(() => (app.effectiveTheme === 'dark' ? 'mark-dark' : 'mark-light'));
defineProps<{ option: echarts.EChartsOption; height?: number | string; loading?: boolean }>();
</script>

<template>
  <a-card :bordered="false">
    <vue-echarts :option="option" :theme="themeName" :loading="loading" :style="{ height: height ?? '320px' }" autoresize />
  </a-card>
</template>
```

---

## 13. 页面总清单

> 本清单基于 `v2/API_V2.0.md` 逐条核对，所有 API 路径与方法**严格一致**。页面编号将贯穿 §14。

| 编号 | 页面名 | 路由 | 所属模块 | 角色 | 关联 API | 覆盖需求 |
| :---: | :--- | :--- | :--- | :--- | :--- | :--- |
| P-01 | 登录页 | `/login` | Auth | 公开 | `POST /api/v1/auth/login`；`POST /api/v1/auth/token/refresh`；`POST /api/v1/auth/password-reset/request`；`POST /api/v1/auth/password-reset/confirm` | FR-GOV-001/002 |
| P-02 | 仪表盘 | `/dashboard` | Overview | admin / super_admin | `GET /api/v1/dashboard` + 可选 `GET /api/v1/admin/logs?...`（近 24h 告警） | FR-GOV-005 |
| P-03a | 任务治理列表 | `/tasks` | TASK | admin / super_admin | `GET /api/v1/rescue/tasks` | FR-TASK-005 |
| P-03b | 任务详情 | `/tasks/:taskId` | TASK | admin / super_admin | `GET /api/v1/rescue/tasks/{task_id}/snapshot`；`GET /api/v1/rescue/tasks/{task_id}/full`；`GET /api/v1/rescue/tasks/{task_id}/trajectory/latest`；`POST /api/v1/rescue/tasks/{task_id}/sustained`；`POST /api/v1/rescue/tasks/{task_id}/close` | FR-TASK-002/003/006 |
| P-04a | 线索复核队列 | `/clues/review` | CLUE | admin / super_admin | `GET /api/v1/clues?review_state=PENDING_REVIEW` | FR-CLUE-005 |
| P-04b | 线索列表（全量） | `/clues` | CLUE | admin / super_admin | `GET /api/v1/clues` | FR-CLUE-004 |
| P-04c | 线索详情 & 复核动作 | `/clues/:clueId` （弹窗形式） | CLUE | admin / super_admin | `POST /api/v1/clues/{clue_id}/override`；`POST /api/v1/clues/{clue_id}/reject` | FR-CLUE-005/006 |
| P-05a | 患者档案列表 | `/patients` | PROFILE | admin / super_admin | `GET /api/v1/admin/patients` | FR-PRO-008 |
| P-05b | 患者档案详情 | `/patients/:patientId` | PROFILE | admin / super_admin | `GET /api/v1/admin/patients/{patient_id}` | FR-PRO-001/003 |
| P-06a | 物资工单列表 | `/material/orders` | MAT | admin / super_admin | `GET /api/v1/material/orders` | FR-MAT-001/002 |
| P-06b | 物资工单详情 | `/material/orders/:orderId` | MAT | admin / super_admin | `GET /api/v1/material/orders?order_id=...`；`POST /api/v1/material/orders/{order_id}/approve`；`POST /api/v1/material/orders/{order_id}/ship`；`POST /api/v1/material/orders/{order_id}/resolve-exception`；`POST /api/v1/material/orders/{order_id}/cancel` | FR-MAT-002/003/004 |
| P-07a | 标签库存 | `/tags/inventory` | MAT | admin / super_admin | `GET /api/v1/tags/inventory/summary`；`POST /api/v1/tags/batch-generate`（super_admin） | FR-MAT-004/005 |
| P-07b | 批量发号任务详情 | `/tags/batch-jobs/:jobId` | MAT | admin / super_admin | `GET /api/v1/tags/batch-generate/jobs/{job_id}` | FR-MAT-005 |
| P-08 | 通知中心 | `/notifications` | GOV | admin / super_admin | `GET /api/v1/notifications/inbox`；`POST /api/v1/notifications/{notification_id}/read` | FR-GOV-008 |
| P-09 | 系统配置 | `/admin/configs` | GOV | admin（只读） / super_admin（可写） | `GET /api/v1/admin/configs`；`PUT /api/v1/admin/configs/{config_key}` | FR-GOV-006 |
| P-10 | 审计日志 | `/admin/logs` | GOV | admin / super_admin | `GET /api/v1/admin/logs`；`GET /api/v1/admin/logs/export`（super_admin） | FR-GOV-007 |
| P-11 | DEAD 事件 | `/admin/dead-letter` | GOV | **super_admin 独占** | `GET /api/v1/admin/super/outbox/dead`；`POST /api/v1/admin/super/outbox/dead/{event_id}/replay` | FR-GOV-009 |
| P-12 | 个人中心 | `/me` | Auth | admin / super_admin | `GET /api/v1/users/me`；`PUT /api/v1/users/me/password` | FR-GOV-003 |
| P-13 | 错误页 | `/401` `/403` `/404` `/500` | Common | 公开 | — | — |
| P-16 | 字典管理 | `/admin/dicts` | GOV | admin（只读） / super_admin（可写） | `GET /api/v1/admin/configs?group=dict`；`PUT /api/v1/admin/configs/{config_key}` | FR-GOV-009 |

> 说明：管理员访问患者档案使用专用管理员接口 `GET /api/v1/admin/patients[/{id}]`（P-15），具备全局视图 + PII 按角色脱敏；家属独占的 `GET /api/v1/patients` 不在管理端消费。P-05 路由在管理端作为患者快速检索入口，重定向至 P-15 组件复用。

---

## 14. 页面详细规格

### 14.P-01 登录页 `/login`

- 视图：`views/auth/LoginView.vue` · 布局：`BlankLayout`
- 权限：匿名；已登录访问自动跳转 `/dashboard`
- 业务目标：完成管理员账号登录（FR-GOV-002）

#### 14.P-01.1 布局线框

```text
┌──────────────────────────── 全屏（品牌背景渐变：#F97316 → #0EA5E9，暗色：#141414） ─────────────────────────────┐
│                                                                                                               │
│    ┌──────── 登录卡片（420×520，圆角 12，阴影 shadow-card）─────────┐                                        │
│    │  🟠 码上回家 · 管理后台                                          │                                        │
│    │  ─────────────────────────────────────                         │                                        │
│    │  [用户名]                                                        │                                        │
│    │  [密码]                                 👁                       │                                        │
│    │  [  ] 记住我            忘记密码 →                               │                                        │
│    │  [       登 录（主色 #F97316）        ]                          │                                        │
│    │  ─────────────────────────────────────                         │                                        │
│    │  🌐 语言：中文 / EN     🌙 主题：跟随系统                        │                                        │
│    └────────────────────────────────────────────────────────────────┘                                        │
│                                                                                                               │
│                              © 2026 码上回家 · v{VITE_APP_VERSION}                                            │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

#### 14.P-01.2 字段

| 字段 | 类型 | 校验 | i18n |
| :--- | :--- | :--- | :--- |
| `username` | string(3–32) | 必填；正则 `^[A-Za-z0-9_\-.]+$` | `page.auth.login.username` |
| `password` | string(8–64) | 必填；至少 1 字母 + 1 数字 | `page.auth.login.password` |
| `remember` | boolean | 默认 false | `page.auth.login.remember` |

#### 14.P-01.3 交互动作

| 触发 | 动作 | API | 成功 | 失败 | 权限 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 点击「登录」 | 校验 → POST | `POST /api/v1/auth/login` | 持久化 token（remember=true 时 localStorage，否则 sessionStorage）→ 跳转 `redirect` 或 `/dashboard` | `E_AUTH_4001` 用户名或密码错误；`E_AUTH_4091` 账号锁定；`E_REQ_4291` 限流（展示倒计时） | 公开 |
| 点击「忘记密码」 | 打开抽屉 | `POST /api/v1/auth/password-reset/request` | 提示"重置链接已发送至邮箱"（HC-06 无短信） | 通用错误提示 | 公开 |
| 重置确认页（通过邮件链接 `/reset-password?token=...`） | 提交新密码 | `POST /api/v1/auth/password-reset/confirm` | 跳登录 | `E_AUTH_4002` token 过期 | 公开 |
| 刷新 token（拦截器内自动） | — | `POST /api/v1/auth/token/refresh` | 静默替换 access token | 失败时登出 | — |

#### 14.P-01.4 其他

| 项 | 规则 |
| :--- | :--- |
| 空/错/载态 | 加载按钮内嵌 loading；全站错误由 `notification.error` 承载 |
| 失败计数 | 连续 5 次失败后按钮锁 60s，文案"请稍后再试"（由服务端 `E_AUTH_4091` 驱动） |
| 可访问性 | 用户名与密码 `aria-label`；支持 Enter 提交；密码显示按钮 `aria-pressed` |
| i18n keys | `page.auth.login.*`、`page.auth.reset.*`、`error.E_AUTH_*` |
| 安全 | 密码输入禁用 `autocomplete="off"` ⚠错（应为 `current-password`）以便密码管理器；表单提交后不写入任何日志与 Sentry breadcrumb |

---

### 14.P-02 仪表盘 `/dashboard`

- 视图：`views/dashboard/DashboardView.vue`
- 权限：admin / super_admin
- 业务目标：一屏呈现当前活跃任务、待复核线索、待审批工单、近 24h 告警（FR-GOV-005）

#### 14.P-02.1 布局线框

```text
┌─────────────────────────────────────────────────────────────────────────────┐
│  KPI 卡片区（4 列栅格）                                                      │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│  │ 活跃任务  │ │ 待复核线索│ │ 待审批工单│ │ 24h 告警 │                        │
│  │   12     │ │    37    │ │    5     │ │    2     │                        │
│  │ ↗ +3(24h) │ │ ↘ -5(24h)│ │  —       │ │  ⚠      │                        │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘                        │
├─────────────────────────────────────────────────────────────────────────────┤
│  趋势图（ChartCard，折线，主色 #F97316 任务；辅色 #0EA5E9 线索）             │
│  [ 最近 7 / 14 / 30 日 切换 ]                                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│  风险告警列表（ProTable 简化版 · 最多 10 条）                                │
│  [时间] [级别] [类型] [摘要（trace_id）] [操作：查看原始日志]                │
└─────────────────────────────────────────────────────────────────────────────┘
```

#### 14.P-02.2 数据来源

| 区块 | 来源字段 | API |
| :--- | :--- | :--- |
| KPI 卡片 × 4 | `summary.active_task_count`、`summary.pending_clue_count`、`summary.pending_order_count`、`summary.recent_alert_count` | `GET /api/v1/dashboard` |
| 趋势图 | `series.task_daily`、`series.clue_daily` | 同上 |
| 风险告警 | `recent_alerts[]` | 同上；点击单条跳 `/admin/logs?trace_id=<id>` |

> 注：若 `/api/v1/dashboard` 未区分管理员视角，则前端退化为聚合调用 `GET /api/v1/rescue/tasks?state=ACTIVE&page_size=1`（取 total）、`GET /api/v1/clues?review_state=PENDING_REVIEW&page_size=1`、`GET /api/v1/material/orders?state=PENDING_AUDIT&page_size=1` 三个计数。

#### 14.P-02.3 交互动作

| 触发 | 动作 | 权限 |
| :--- | :--- | :--- |
| KPI 点击 | 跳转对应列表页并预填过滤条件（如"待复核"） | 按目标页 |
| 趋势图切换周期 | 重新请求 `?range=7d\|14d\|30d` | `task:read` |
| 告警行"查看原始日志" | 跳转 `/admin/logs?trace_id=...` | `audit:read` |

#### 14.P-02.4 实时事件

| 事件 | 处理 |
| :--- | :--- |
| WS topic `admin.dashboard.metrics` | 节流 5s 合并；直接合并进 `summary` |
| WS topic `user.{me}.notification` level=ALERT | 追加到告警列表顶部（最多保留 10 条） |

#### 14.P-02.5 其他

| 项 | 规则 |
| :--- | :--- |
| 加载态 | 卡片骨架屏（4×SkeletonNode）；图表 `ChartCard loading` |
| 空态 | 告警列表空 → `EmptyState` + 引导"查看全部审计日志" |
| i18n keys | `page.dashboard.kpi.*`、`page.dashboard.chart.*`、`page.dashboard.alert.*` |
| 可访问性 | KPI 卡片 `role="link"` + Enter 激活 |

---

### 14.P-03a 任务治理列表 `/tasks`

- 视图：`views/task/TaskListView.vue` · 组件：`ProTable`
- 权限：admin / super_admin（`task:read`）
- 业务目标：查看/筛选所有活跃及历史任务（FR-TASK-005）

#### 14.P-03a.1 布局线框

```text
┌ 查询 ──────────────────────────────────────────────────────────────────┐
│ 状态:[全部▾]  患者ID:[____]  主监护人:[____]  发布时间:[起—止📅]       │
│ [查询] [重置]  ⟶                                                        │
├ 操作条 ───────────────────────────────────────────────────────────────┤
│ [导出 CSV]          列配置⚙    刷新🔄                                   │
├ 表格 ────────────────────────────────────────────────────────────────┤
│ 任务ID | 患者 | 状态 | 发布时间 | 最近线索 | 监护人 | 操作              │
│ #t_… | 张… | 🟧ACTIVE | 04-20 09:12 | 3分钟前 | 李… | [详情][时间轴] │
└────────────────────────────────────────────────────────────────────────┘
```

#### 14.P-03a.2 字段（`GET /api/v1/rescue/tasks` 响应）

| 列 | 字段 | 展示 |
| :--- | :--- | :--- |
| 任务 ID | `task_id`（string） | `CopyableText` 显示前 8 位 + 复制 |
| 患者 | `patient.display_name` + `patient_id` | 链接跳 `/patients/:patientId` |
| 状态 | `state` | `StatusBadge`：`CREATED`灰 / `ACTIVE`橙 / `SUSTAINED`蓝 / `CLOSED_FOUND`绿 / `CLOSED_FALSE_ALARM`红 |
| 发布时间 | `created_at`（ISO8601，UTC） | `dayjs().format('YYYY-MM-DD HH:mm')`，tooltip 显示本地时区 |
| 最近线索 | `latest_clue_at` | 相对时间 `fromNow()` |
| 监护人 | `primary_guardian.display_name` | 脱敏手机号 tooltip |

#### 14.P-03a.3 查询参数

| 参数 | 取值 | 说明 |
| :--- | :--- | :--- |
| `state` | `CREATED / ACTIVE / SUSTAINED / CLOSED_FOUND / CLOSED_FALSE_ALARM` | 多选 |
| `patient_id` | string | 精确 |
| `guardian_keyword` | string | 模糊（服务端支持时） |
| `created_at_start` / `created_at_end` | ISO | 时间范围 |
| `page_no` / `page_size` | number | 默认 `1 / 20` |

#### 14.P-03a.4 交互动作

| 触发 | 动作 | API | 权限 |
| :--- | :--- | :--- | :--- |
| 行"详情" | 路由跳转 `/tasks/:taskId` | — | `task:read` |
| 操作"导出 CSV" | 前端导出当前页（≤1000 行）；>1000 提示按后端导出接口（若无则隐藏） | — | `task:read` |

#### 14.P-03a.5 其他

| 项 | 规则 |
| :--- | :--- |
| 空态 | 无活跃任务时展示"所有任务均已闭环" |
| i18n | `page.task.list.*`；枚举 `enum.taskState.*` |

---

### 14.P-03b 任务详情 `/tasks/:taskId`

- 视图：`views/task/TaskDetailView.vue`（Tab 布局）

#### 14.P-03b.1 布局线框

```text
┌ 顶部信息条 ─────────────────────────────────────────────────────────────┐
│ ◀返回  任务 #t_01HXY…（复制）   🟧ACTIVE    [标长期维持][关闭任务(高危)] │
│  患者 张三（76岁，男） · 主监护 李四  · 发布 04-20 09:12                 │
└────────────────────────────────────────────────────────────────────────┘
┌ Tab ─────────────────────────────────────────────────────────────────┐
│ [概览][状态时间轴][最新轨迹][线索流][通知][审计]                       │
├─概览 (ProDescriptions) ──────────────────────────────────────────────┤
│  走失时间 / 最后出现地 / 外观描述 / 关联标签 / 附近围栏命中…           │
├─状态时间轴 (TaskStateTimeline) ───────────────────────────────────────┤
│  CREATED → ACTIVE → (可能)SUSTAINED → CLOSED                         │
├─最新轨迹 (AMap 嵌入) ─────────────────────────────────────────────────┤
│  最近 50 个锚点，线索命中红点                                          │
├─线索流 (ProTable) ───────────────────────────────────────────────────┤
│  时间 / 上报人 / 复核状态 / 置信度 / [查看][复核动作]                  │
└──────────────────────────────────────────────────────────────────────┘
```

#### 14.P-03b.2 数据来源

| Tab | 字段 | API |
| :--- | :--- | :--- |
| 顶部信息 + 概览 | `task` + `patient_snapshot` | `GET /api/v1/rescue/tasks/{task_id}/full` |
| 状态时间轴 | `state_history[]` | `GET /api/v1/rescue/tasks/{task_id}/snapshot` |
| 最新轨迹 | `points[]` + `clue_hits[]` | `GET /api/v1/rescue/tasks/{task_id}/trajectory/latest` |
| 线索流 | `clues[]` | `GET /api/v1/clues?task_id={task_id}&cursor=...` |

#### 14.P-03b.3 交互动作

| 触发 | 动作 | API | 权限 | 二次确认 |
| :--- | :--- | :--- | :--- | :--- |
| 「标长期维持」 | POST 带理由 | `POST /api/v1/rescue/tasks/{task_id}/sustained` | `task:sustain` | `Modal.confirm`（仅 ACTIVE 可用） |
| 「关闭任务」 | 选 `close_type`: FOUND / FALSE_ALARM + 必填理由 | `POST /api/v1/rescue/tasks/{task_id}/close` | `task:close` | 输入"确认关闭"串 |
| 线索行"覆写 / 驳回" | 转 P-04c 对话框 | 见 P-04 | `clue:override / clue:reject` | 见 P-04 |

#### 14.P-03b.4 校验与错误

| 场景 | 错误码 | 处理 |
| :--- | :--- | :--- |
| 非 ACTIVE 尝试标长期维持 | `E_TASK_4091` | 按钮置灰 + tooltip |
| 同患者多活跃任务 | `E_TASK_4092` | 后端兜底，前端 message 提示并刷新 |
| 状态机非法跃迁 | `E_TASK_4093` | 重新拉取 snapshot 刷新按钮可用性 |

#### 14.P-03b.5 实时事件

| Topic | 处理 |
| :--- | :--- |
| `task.{task_id}.state` | 重拉 snapshot 刷新顶部状态与时间轴 |
| `task.{task_id}.clue.new` | 线索流 Tab 顶部出现"有 N 条新线索，点击刷新"的 Ribbon |
| `task.{task_id}.trajectory` | 地图 Tab 节流追加点（≥500ms） |

#### 14.P-03b.6 其他

| 项 | 规则 |
| :--- | :--- |
| 离开路由 | `unsubscribe` WS topic |
| i18n | `page.task.detail.*` |
| 可访问性 | Tab 支持方向键切换，地图禁止键盘焦点陷阱 |
| 高德地图 | 遵循 API §1.9 坐标系（GCJ-02）；组件封装 `components/domain/TrajectoryMap.vue` |

---

### 14.P-04a 线索复核队列 `/clues/review`

- 视图：`views/clue/ClueReviewView.vue`
- 权限：admin / super_admin（`clue:read`，动作需 `clue:override` 或 `clue:reject`）
- 业务目标：处理所有 `review_state = PENDING_REVIEW` 的可疑线索（FR-CLUE-005）

#### 14.P-04a.1 布局线框

```text
┌ 查询 ───────────────────────────────────────────────────────────────────┐
│ 关联任务:[_____]  上报时间:[起—止]  风险等级:[全部▾]  地区:[____]         │
│ [查询][重置]                                                               │
├ 统计条（ECharts 饼图小 widget）────────────────────────────────────────┤
│ 今日新增 42 · 待复核 37 · 24h 平均处理时长 18 分钟                        │
├ 表格 ───────────────────────────────────────────────────────────────┤
│ 线索ID | 任务 | 上报时间 | 风险 | 置信度 | 位置 | 缩略图 | 操作            │
│ #c_… | #t_… | 3 分钟前 | 🟥高 | 0.82 | 南京市… | 🖼 | [详情][覆写][驳回]│
└────────────────────────────────────────────────────────────────────────┘
```

#### 14.P-04a.2 字段（`GET /api/v1/clues`）

| 列 | 字段 | 展示 |
| :--- | :--- | :--- |
| 线索 ID | `clue_id` | `CopyableText` 前 8 位 |
| 任务 | `task_id` | 链接 `/tasks/:taskId` |
| 上报时间 | `reported_at` | `fromNow()` |
| 风险 | `risk_level` | `StatusBadge` LOW/MED/HIGH |
| 置信度 | `confidence` | 百分比 + 色阶 |
| 位置 | `location_name` + 坐标 | 超 12 字省略，tooltip 完整 |
| 缩略图 | `media[0].url` | 点击弹 Lightbox |

#### 14.P-04a.3 查询参数

| 参数 | 取值 |
| :--- | :--- |
| `review_state` | `PENDING_REVIEW`（队列页固定注入）|
| `task_id` | string |
| `risk_level` | `LOW / MEDIUM / HIGH` |
| `reported_at_start` / `reported_at_end` | ISO |
| `page_no` / `page_size` | 默认 `1 / 20` |

#### 14.P-04a.4 交互动作

| 触发 | 动作 | API | 权限 | 二次确认 |
| :--- | :--- | :--- | :--- | :--- |
| 「详情」 | 弹 `ProDrawer` 展示完整线索+地图+媒体 | — | `clue:read` | — |
| 「覆写」 | 打开覆写对话框（理由 30–500 字） | `POST /api/v1/clues/{clue_id}/override` | `clue:override` | `Modal.confirm` |
| 「驳回」 | 打开驳回对话框（理由 30–500 字） | `POST /api/v1/clues/{clue_id}/reject` | `clue:reject` | `Modal.confirm` |
| 批量操作 | 多选后批量覆写/驳回（逐条提交，合并进度条） | 同上 | 同上 | 列出全部 ID，二次确认 |

#### 14.P-04a.5 校验与错误

| 场景 | 错误码 | 处理 |
| :--- | :--- | :--- |
| 非 PENDING_REVIEW 状态 | `E_CLUE_4091` | 按钮隐藏 + 实时刷新 |
| 理由长度不足 | 前端校验 | 表单 `rule.min=30` |
| 并发冲突（其他管理员已处理） | `E_CLUE_4092` | 提示并刷新列表 |

#### 14.P-04a.6 实时事件

| Topic | 处理 |
| :--- | :--- |
| WS 不主动订阅全局线索（HC-05）。仅"已打开的任务详情页"订阅 `task.{id}.clue.new`；队列页依靠定时 30s 刷新 + 手动刷新按钮 |

#### 14.P-04a.7 其他

| 项 | 规则 |
| :--- | :--- |
| 空态 | 无待复核时展示"好极了，暂无待复核线索 🎉" |
| i18n | `page.clue.review.*` |
| 可访问性 | 批量选择 `aria-multiselectable="true"`；覆写/驳回确认对话框 `aria-describedby` 指向理由框 |

---

### 14.P-04b 线索列表（全量） `/clues`

- 视图：`views/clue/ClueListView.vue`（与 P-04a 共用 `ProTable`，仅默认过滤不同）

字段与操作同 P-04a；新增 `review_state`、`task_id` 过滤。对 `review_state != PENDING_REVIEW` 的行，覆写/驳回按钮按服务端规则禁用（`disabled + tooltip`）。

---

### 14.P-04c 线索详情抽屉

- 组件：`ClueDetailDrawer.vue`（由 P-04a、P-04b、P-03b 共用）

#### 14.P-04c.1 布局线框

```text
┌ 抽屉（宽 720）──────────────────────────────────┐
│ 线索 #c_01HXY…                       ✕         │
│ 基本信息（ProDescriptions）                      │
│  - 上报时间 / 风险 / 置信度 / 来源 (匿名/家属)    │
│  - 关联任务 / 所在患者 / 距离患者末知位置距离     │
│ 位置（AMap 嵌入 240h）                           │
│ 媒体（缩略图轮播 + Lightbox）                    │
│ AI 分析（ai_summary + ai_risk_factors 气泡）     │
│ 复核历史（TimeLine）                              │
│ ─────────────────────────────────────           │
│ [关闭]        [驳回(红)]   [覆写(主色)]          │
└─────────────────────────────────────────────────┘
```

#### 14.P-04c.2 字段（响应扩展）

| 字段 | 说明 |
| :--- | :--- |
| `ai_summary` | AI 生成的 ≤200 字摘要；`v-html` 前必须 DOMPurify |
| `media[]` | `{ type: 'IMAGE' / 'VIDEO', url, thumb_url }`；展示前检查 `http -> https` 升级 |
| `review_history[]` | `{ actor_id, action, reason, trace_id, reviewed_at }` |

---

### 14.P-05a 患者档案列表 `/patients`

- 视图：`views/profile/PatientListView.vue`
- 权限：admin / super_admin（`patient:read`）
- 业务目标：辅助线索复核与工单审批时快速检索患者（FR-PRO-008）。管理端不提供创建/修改档案（属家属端 Android）。本页内部复用 **P-15 管理员患者档案**组件（`AdminPatientListView.vue`）。

#### 14.P-05a.1 布局

```text
┌ 查询 ────────────────────────────────────────────────────────────┐
│ 关键字:[姓名/患者ID/主监护手机] 走失状态:[全部▾] 地区:[__]          │
│ [查询][重置]                                                        │
├ 表格 ───────────────────────────────────────────────────────────┤
│ 患者ID | 姓名 | 年龄 | 性别 | 走失状态 | 标签数 | 主监护 | 创建时间  │
└─────────────────────────────────────────────────────────────────┘
```

#### 14.P-05a.2 字段（`GET /api/v1/admin/patients`）

| 列 | 字段 | 展示 |
| :--- | :--- | :--- |
| 患者 ID | `patient_id`（string） | CopyableText |
| 姓名 | `display_name` | 后两字脱敏显示 `张**` 可点击查看完整（需 `pii:reveal`，未实现则只读） |
| 年龄 | `age` | 计算值 |
| 走失状态 | `missing_state` | `NORMAL`绿 / `MISSING_PENDING`黄 / `MISSING`红（`StatusBadge`） |
| 标签数 | `bound_tag_count` | 数字 |
| 主监护 | `primary_guardian.display_name` | 脱敏 |

---

### 14.P-05b 患者档案详情 `/patients/:patientId`

- 视图：`views/profile/PatientDetailView.vue`（只读）

布局：左侧"基础资料 + 外观 + 围栏"、右侧"活跃任务、历史任务、绑定标签、监护关系"。

| 区块 | API |
| :--- | :--- |
| 基础资料 / 外观 / 围栏 | `GET /api/v1/admin/patients/{patient_id}` |
| 活跃任务 | `GET /api/v1/rescue/tasks?patient_id=...&state=ACTIVE,SUSTAINED` |
| 历史任务 | `GET /api/v1/rescue/tasks?patient_id=...&state=CLOSED_FOUND,CLOSED_FALSE_ALARM` |

> 管理端视图 **所有字段只读**；编辑按钮不渲染。PII 字段通过 `<MaskedText>` 渲染。

---

### 14.P-06a 物资工单列表 `/material/orders`

- 视图：`views/material/OrderListView.vue`
- 权限：admin / super_admin（`material.order:read`）

#### 14.P-06a.1 布局

```text
┌ 查询 ──────────────────────────────────────────────────────────────────┐
│ 状态:[待审批/待发货/已发货/已签收/异常/已取消 ▾] 订单号:[__] 监护人:[__] │
│ 申请时间:[起—止]  [查询][重置]                                           │
├ 操作条 ───────────────────────────────────────────────────────────────┤
│ （批量审批）已选 3 单 [批量审批通过]      刷新🔄  列配置⚙                │
├ 表格 ────────────────────────────────────────────────────────────────┤
│ 订单号 | 患者 | 物料(标签数) | 状态 | 申请时间 | 物流单号 | 操作         │
│ #o_… | 张… | 电子标签×2 | 🟡PENDING_AUDIT | 04-20 08:00 | — | [详情]  │
└───────────────────────────────────────────────────────────────────────┘
```

#### 14.P-06a.2 字段（`GET /api/v1/material/orders`）

| 列 | 字段 | 展示 |
| :--- | :--- | :--- |
| 订单号 | `order_id`（string） | CopyableText |
| 患者 | `patient.display_name` | 链接 `/patients/:id` |
| 物料 | `items[]` | "电子标签×2" |
| 状态 | `state` | `StatusBadge`：`PENDING_AUDIT / PENDING_SHIP / SHIPPED / RECEIVED / CANCELLED / EXCEPTION` |
| 申请时间 | `created_at` | `YYYY-MM-DD HH:mm` |
| 物流单号 | `logistics.tracking_no` | 超 16 字省略 |

---

### 14.P-06b 物资工单详情 `/material/orders/:orderId`

#### 14.P-06b.1 布局线框

```text
┌ 顶部 ──────────────────────────────────────────────────────────────────┐
│ ◀返回  订单 #o_01HXY…  🟡PENDING_AUDIT                                  │
│ [审批通过] [驳回取消] [发货] [取消工单]                                    │
├ 基本信息（ProDescriptions 3 列） ─────────────────────────────────────┤
│  申请人 / 患者 / 收件人 / 收货地址(脱敏) / 物料清单 / 备注                │
├ 时间轴（OrderTimeline） ──────────────────────────────────────────────┤
│  提交 → 审批通过 → 发货(物流单号) → 签收                                   │
├ 关联（Tab） ──────────────────────────────────────────────────────────┤
│ [绑定标签] [审计事件]                                                    │
└───────────────────────────────────────────────────────────────────────┘
```

#### 14.P-06b.2 交互动作

| 触发 | 动作 | API | 权限 | 二次确认 |
| :--- | :--- | :--- | :--- | :--- |
| 「审批通过」 | 表单：`reviewer_note?` | `POST /api/v1/material/orders/{order_id}/approve` | `material.order:approve` | `Modal.confirm` |
| 「发货」 | 表单：`tracking_no*`、`carrier*`、勾选"已核对单号" | `POST /api/v1/material/orders/{order_id}/ship` | `material.order:ship` | `Modal.confirm` |
| 「异常处置」（仅 `EXCEPTION` 状态可见） | 选择 `RESHIP`（补发，需填写新物流单号）或 `VOID`（直接作废）；理由必填 10-200 字 | `POST /api/v1/material/orders/{order_id}/resolve-exception` | `material.order:resolve-exception` | 输入"确认处置" |
| 「取消工单」 | 理由必填 | `POST /api/v1/material/orders/{order_id}/cancel` | `material.order:approve` | 输入"确认取消" |
| 家属端操作「签收」 | — | `POST /api/v1/material/orders/{order_id}/receive` | 不在管理端展示 | — |

#### 14.P-06b.3 校验

| 字段 | 规则 |
| :--- | :--- |
| `tracking_no` | 长度 6–32，`^[A-Za-z0-9]+$` |
| `carrier` | 字典：SF/YTO/ZTO/JD/EMS/OTHER |
| `cancel.reason` | 10–200 字 |

#### 14.P-06b.4 错误码

| 错误码 | 含义 | UI |
| :--- | :--- | :--- |
| `E_MAT_4091` | 状态机非法跃迁（如非 `PENDING_AUDIT` 不能 approve，非 `EXCEPTION` 不能 resolve） | 按钮禁用 + tooltip；失败刷新 |
| `E_MAT_4092` | 物流单号重复 | `form.setFields({ tracking_no: { errors: [...] } })` |
| `E_MAT_4093` | 库存不足（发号阶段） | 提示联系 super_admin 批量发号 |
| `E_MAT_4224` | 补发时物流单号重复 / 库存不足 | `message.error` + 刷新 |
| `E_MAT_4226` | `action` 枚举值非法或 `reason` 为空 | 表单字段级错误 |

---

### 14.P-07a 标签库存 `/tags/inventory`

- 视图：`views/material/TagInventoryView.vue`
- 权限：admin（`tag:read` 只读）/ super_admin（`tag:read` + `tag:batch-generate`）

#### 14.P-07a.1 布局

```text
┌ KPI ────────────────────────────────────────────────────────────────┐
│ 未分配 1,240 | 已分配 820 | 已绑定 690 | 疑失 12 | 遗失 8 | 作废 32   │
├ 趋势图（近 30 日发号/绑定/遗失） ChartCard ────────────────────────┤
├ 操作条 ────────────────────────────────────────────────────────────┤
│ [批量发号(super_admin)]  [查看全部批次]   刷新🔄                     │
├ 最近发号任务 ProTable ────────────────────────────────────────────┤
│ 批次ID | 状态(PENDING/RUNNING/DONE/FAILED) | 数量 | 进度 | 创建时间 | [详情]│
└─────────────────────────────────────────────────────────────────────┘
```

#### 14.P-07a.2 数据来源

| 区块 | API |
| :--- | :--- |
| KPI | `GET /api/v1/tags/inventory/summary` 响应中的计数字段 |
| 趋势图 | 同上（若服务端提供 `trend.daily`），否则隐藏 |
| 最近批次 | 由 `GET /api/v1/tags/batch-generate/jobs/{job_id}` 汇总记录（前端 Pinia `useDictStore` 维护最近 20 个批次 ID） |

#### 14.P-07a.3 交互：批量发号（super_admin 独占）

| 步骤 | 动作 | API | 二次确认 |
| :--- | :--- | :--- | :--- |
| 1 | 打开表单：数量 `count: int [1, 5000]`；批次备注 | — | — |
| 2 | 提交 | `POST /api/v1/tags/batch-generate` | `Modal.confirm` 输入数量 |
| 3 | 获取 `job_id` 跳 `/tags/batch-jobs/:jobId` 轮询 |
| 4 | 轮询 | `GET /api/v1/tags/batch-generate/jobs/{job_id}`（2s→5s→10s 退避） | — |

#### 14.P-07a.4 错误码

| 错误码 | 含义 | UI |
| :--- | :--- | :--- |
| `E_MAT_4094` | 数量超限 | 表单校验 |
| `E_MAT_4095` | 同批次进行中，不可并发 | 按钮禁用 + tooltip |
| `E_AUTH_4031` | admin 点"批量发号"会被后端拒绝；前端已按 `v-permission="'tag:batch-generate'"` 隐藏，为双层防御 |

---

### 14.P-07b 批量发号任务详情 `/tags/batch-jobs/:jobId`

- 布局：进度条 + 任务元数据 + 生成短码下载（如服务端返回 `download_url`）
- 状态机：`PENDING → RUNNING → DONE / FAILED`
- 轮询策略：DONE/FAILED 停止；页面离开前停止

---

### 14.P-08 通知中心 `/notifications`

- 视图：`views/notification/InboxView.vue`
- 权限：admin / super_admin（`notification:read`）

#### 14.P-08.1 布局

```text
┌ Tab：[全部][未读][告警][系统][业务]                                   │
├ 操作条：[全部标已读] 刷新🔄                                           │
├ 列表（Cursor 分页）                                                    │
│  ┌ 卡片 ───────────────────────────────────────┐                    │
│  │ 🔔 ALERT · 任务 #t_01… 已进入 SUSTAINED 状态  │                    │
│  │   2 分钟前 · trace_id: trc_...               │                    │
│  │   [查看] [标已读]                            │                    │
│  └────────────────────────────────────────────┘                    │
└──────────────────────────────────────────────────────────────────────┘
```

#### 14.P-08.2 数据来源

| 区块 | API |
| :--- | :--- |
| 列表 | `GET /api/v1/notifications/inbox?cursor=...&category=ALERT|SYS|BIZ&only_unread=true` |
| 标已读 | `POST /api/v1/notifications/{notification_id}/read` |

#### 14.P-08.3 交互

| 触发 | 动作 | API | 权限 |
| :--- | :--- | :--- | :--- |
| 「查看」 | 按 `deeplink.type` 跳转相应页面（TASK→`/tasks/:id`，CLUE→`/clues/review`，ORDER→`/material/orders/:id`） | — | 按目标权限 |
| 「标已读」 | 乐观更新 + 请求 | `POST /api/v1/notifications/{notification_id}/read` | `notification:read` |
| 「全部标已读」 | 前端逐条或服务端支持批量 | 按接口定义 | 同上 |

#### 14.P-08.4 实时事件

| Topic | 处理 |
| :--- | :--- |
| `user.{me}.notification` | 新消息 Push 进列表顶部；顶部栏铃铛红点 +1；Toast 节流（同类合并 3s） |

#### 14.P-08.5 其他

| 项 | 规则 |
| :--- | :--- |
| 富文本 | 后端消息正文如含 HTML，使用 `DOMPurify.sanitize` 过滤；禁止 `v-html` 原始 |
| 空态 | 分类空态文案不同；未读空显示"没有未读消息" |
| i18n | `page.notification.*` |

---

### 14.P-09 系统配置 `/admin/configs`

- 视图：`views/admin/ConfigView.vue`
- 权限：admin（`config:read` 只读）、super_admin（`config:read` + `config:write`）
- 业务目标：查看与热更 `sys_config`（FR-GOV-006）

#### 14.P-09.1 布局

```text
┌ 查询 ─────────────────────────────────────────────────────────────────┐
│ 分组:[全部▾(notification/task/clue/ai/security)]  关键字:[__]  [查询]   │
├ 表格 ────────────────────────────────────────────────────────────────┤
│ Key                    | 分组    | 值     | 类型 | 描述 | 更新时间 | 操作 │
│ task.active.ttl.seconds| task   | 86400  | int  | 活跃… | 04-18 | [编辑(super)] │
└───────────────────────────────────────────────────────────────────────┘
```

#### 14.P-09.2 字段（`GET /api/v1/admin/configs`）

| 列 | 字段 | 说明 |
| :--- | :--- | :--- |
| Key | `config_key` | 唯一 |
| 分组 | `group` | 下拉筛选 |
| 值 | `config_value` | 根据 `value_type` 渲染：`int / float / boolean / string / json` |
| 类型 | `value_type` | 决定编辑器 |
| 描述 | `description` | 支持 i18n，服务端返回多语言字段优先 |
| 更新 | `updated_at` / `updated_by` | tooltip 显示操作者 |

#### 14.P-09.3 交互：编辑配置（super_admin）

| 步骤 | 动作 | API |
| :--- | :--- | :--- |
| 1 | 点「编辑」→ 打开 `ProDrawer`，按 `value_type` 渲染输入控件 |
| 2 | 提交前展示 **diff（旧值 → 新值）**，用户确认 |
| 3 | 发起请求 | `PUT /api/v1/admin/configs/{config_key}`，body `{ value, reason }` |
| 4 | 成功后刷新行，并通过 WS `admin.config.changed` 广播（如启用） |

#### 14.P-09.4 校验

| `value_type` | 前端校验 |
| :--- | :--- |
| `int` | `Number.isInteger` + 边界（服务端返回 `min/max` 优先） |
| `float` | `isFinite` + 小数位 |
| `boolean` | `a-switch` |
| `string` | 长度 1–500 |
| `json` | Monaco Editor；`JSON.parse` 校验；内容允许多行 |

#### 14.P-09.5 错误码

| 错误码 | UI |
| :--- | :--- |
| `E_GOV_4091` | 值类型不匹配，表单字段级错误 |
| `E_GOV_4092` | 值越界；`form.setFields` 标红 |
| `E_AUTH_4031` | admin 点"编辑"本已双层禁用；保留作为网关防御 |

---

### 14.P-10 审计日志 `/admin/logs`

- 视图：`views/admin/AuditLogView.vue`
- 权限：admin / super_admin（`audit:read`）
- 业务目标：按 trace_id / 操作者 / 资源 / 时间范围回溯操作（FR-GOV-007）

#### 14.P-10.1 布局

```text
┌ 查询 ──────────────────────────────────────────────────────────────────┐
│ trace_id:[____]  operator:[____]  action:[____]  resource_type:[__]    │
│ resource_id:[___]  时间:[起—止]  [查询][重置]                            │
├ 表格（Cursor 分页，无 total）──────────────────────────────────────────┤
│ 时间 | 操作者 | 角色 | 动作 | 资源 | 入参摘要 | trace_id | 状态 | 操作   │
│                                                                        │
│ 点击行 → 右侧抽屉展开完整 JSON（Monaco readonly）                        │
└────────────────────────────────────────────────────────────────────────┘
```

#### 14.P-10.2 数据来源

| 接口 | 参数 |
| :--- | :--- |
| `GET /api/v1/admin/logs` | `cursor`、`page_size`（≤100）、`trace_id`、`operator_id`、`action`、`resource_type`、`resource_id`、`start_at`、`end_at` |

> Cursor 分页严格遵循 API §1.5；禁止请求 `total`。

#### 14.P-10.3 交互

| 触发 | 动作 | 权限 |
| :--- | :--- | :--- |
| 点行 | 弹抽屉显示 `request / response / trace_id / pod_id / duration_ms` | `audit:read` |
| "复制 trace_id" | 复制到剪贴板 | — |
| "跳后端日志" | 若链路接入 Kibana，按 `VITE_KIBANA_URL?trace_id=...` 新窗口打开（可配置） | 外部 |
| 「导出日志」（super_admin 独占） | 展开参数面板：时间范围（必填，≤ 31 天，≤ 180 天归档）+ 可选 `operator_id` / `action` / `format`（json/csv）；确认后调接口下载 | `audit:export` |

#### 14.P-10.4 性能与安全

| 项 | 规则 |
| :--- | :--- |
| 大字段渲染 | 抽屉打开时才懒加载 request/response 完整内容；列表行仅展示 150 字摘要 |
| 导出 | 仅 `super_admin` 可操作；通过 `GET /api/v1/admin/logs/export` 服务端流式输出 CSV/JSON；前端触发下载，不在浏览器内存中缓存全量数据；单次上限 10,000 条（`E_GOV_4094`）|
| 脱敏 | 服务端已脱敏；前端不再解密 |

---

### 14.P-11 DEAD 事件 `/admin/dead-letter`

- 视图：`views/admin/DeadLetterView.vue`
- 权限：**super_admin 独占**（`dead:read` + `dead:replay`）
- 业务目标：查询 Outbox 死信并按分区顺序受控重放（FR-GOV-009；SADD §5.1 分区内不可并发）

#### 14.P-11.1 布局

```text
┌ 过滤 ──────────────────────────────────────────────────────────────────┐
│ 主题 topic:[__]  分区 partition_key:[__]  首失败时间:[起—止]             │
│ [查询][重置]                                                              │
├ 表格 ────────────────────────────────────────────────────────────────┤
│ event_id | topic | partition_key | 失败次数 | 首失败 | 最近错误 | 操作   │
│ #e_… | task.state.changed | pid=1001 | 3 | 04-18 | 下游5xx | [详情][重放] │
├ 详情抽屉 ────────────────────────────────────────────────────────────┤
│ payload (JSON) | headers | 重试记录 | 关联 trace_id（跳 /admin/logs）   │
└───────────────────────────────────────────────────────────────────────┘
```

#### 14.P-11.2 数据来源

| 接口 | 备注 |
| :--- | :--- |
| `GET /api/v1/admin/super/outbox/dead` | Cursor 分页；支持 topic / partition_key / 时间 |
| `POST /api/v1/admin/super/outbox/dead/{event_id}/replay` | body `{ reason }`（10–200 字） |

#### 14.P-11.3 交互与约束

| 触发 | 动作 | 二次确认 | 约束 |
| :--- | :--- | :--- | :--- |
| 「详情」 | 抽屉展开 payload / headers / 重试记录 | — | readonly |
| 「重放」 | `Modal.confirm` + 输入事件 ID 后 4 位 + 填写理由 | 必填 | 同一 `partition_key` 若已有进行中重放，按钮禁用（前端根据行状态字段 `replaying` 判断） |
| 批量重放 | **禁止**（HC-05 防误操作） | — | UI 不提供 |

#### 14.P-11.4 错误码

| 错误码 | 含义 | UI |
| :--- | :--- | :--- |
| `E_GOV_4093` | 分区内已有进行中重放 | 行置灰 + tooltip + 5s 后自动刷新 |
| `E_GOV_4094` | 事件已被消费/已归档 | 刷新列表，行淡出 |
| `E_AUTH_4031` | 非 super_admin | 前端已屏蔽入口；兜底提示 |

#### 14.P-11.5 实时事件

| Topic | 处理 |
| :--- | :--- |
| `admin.dead.letter` | 新增死信时追加行，顶部 Ribbon 展示"+1 新增死信" |

---

### 14.P-12 个人中心 `/me`

- 视图：`views/me/ProfileView.vue`
- 权限：admin / super_admin

#### 14.P-12.1 区块

| 区块 | 字段 | API |
| :--- | :--- | :--- |
| 基本资料（只读） | `username / display_name / role / email` | `GET /api/v1/users/me` |
| 修改密码 | `old_password / new_password / confirm_password` | `PUT /api/v1/users/me/password` |
| 偏好设置 | 语言 / 主题 | `useAppStore` 持久化，不走 API |
| 安全信息 | 最近登录时间 / IP / UA（若 `users/me` 返回） | `GET /api/v1/users/me` |

#### 14.P-12.2 校验

| 字段 | 规则 |
| :--- | :--- |
| `new_password` | 8–64 位，至少 1 字母 + 1 数字；禁止与 `old_password` 相同 |
| `confirm_password` | 与 `new_password` 一致 |

#### 14.P-12.3 错误码

| 错误码 | UI |
| :--- | :--- |
| `E_AUTH_4001` | `old_password` 错误；表单字段级 |
| `E_AUTH_4003` | 新密码不满足策略；字段级并展示策略说明 |

---

### 14.P-13 错误页 `/401 /403 /404 /500`

| 页面 | 文案 | 主操作 |
| :--- | :--- | :--- |
| `/401` | 登录已过期，请重新登录 | 跳 `/login` |
| `/403` | 无权访问该页面 | 返回首页；附 Trace-Id |
| `/404` | 页面不存在 | 返回上一页 / 回首页 |
| `/500` | 服务异常，请稍后再试 | 重试；展示 Trace-Id；打开"反馈"邮件 mailto |

统一使用插画式 `EmptyState`，主色 `#F97316` 装饰；禁止泄露服务端堆栈。

---

### 14.P-14 用户管理 `/admin/users` · `/admin/users/:userId`

- 视图：`views/admin/UserListView.vue` / `UserDetailView.vue` · 布局：`DefaultLayout`
- 权限：`admin` / `super_admin`（服务端按 `role` 强制过滤）
- 业务目标：后台用户治理（FR-GOV-011 ~ FR-GOV-014）

#### 14.P-14.1 布局线框（P-14a 列表）

```text
┌─ 面包屑：系统管理 / 用户管理 ────────────────────────────────────────┐
├─ 查询表单（keyword / role 多选 / status / cursor）────────────────┤
├─ 操作条：[刷新] [仅显示 FAMILY（默认选中,ADMIN 隐藏切换）]           ┤
├─ ProTable （cursor 分页，每页 20）                                   ┤
│  id | 用户名 | 昵称 | 邮箱(脱敏) | 手机(脱敏) | 角色 | 状态 | 最后登录 | 操作 │
│  ...                                                                 │
│  操作列：[查看] [编辑] [禁用/启用] [删除]（按权限矩阵按钮级控制）    │
└──────────────────────────────────────────────────────────────────────┘
```

#### 14.P-14.2 数据与字段

| 字段 | 类型 | 来源 | 展示 | 脱敏 |
| :--- | :--- | :--- | :--- | :--- |
| `user_id` | string | — | 单调列，复制按钮 | — |
| `username` | string | ← | 文本 | — |
| `nickname` | string | ← | 文本 | — |
| `email` | string | ← | 文本 | 已在后端 `@Desensitize` |
| `phone` | string | ← | 文本 | 已在后端 `@Desensitize` |
| `role` | enum | `FAMILY/ADMIN/SUPER_ADMIN` | `DictTag`：`FAMILY=#0EA5E9 / ADMIN=#F97316 / SUPER_ADMIN=#EF4444` | — |
| `status` | enum | `ACTIVE/DISABLED/DEACTIVATED` | `StatusBadge`：`ACTIVE=#22C55E / DISABLED=#F59E0B / DEACTIVATED=灰` | — |
| `last_login_at` | datetime | ← | `dayjs.fromNow()` | — |

#### 14.P-14.3 交互动作（列表 + 详情统一）

| 触发 | 动作 | API | 权限码 | 成功反馈 | 失败反馈 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 页面进入 | 拉取列表 | `GET /api/v1/admin/users` | `user:list` | 渲染表格 | `message.error` + Trace-Id |
| 点击「查看」 | 跳 `/admin/users/:userId` + `GET /detail` | `GET /api/v1/admin/users/{user_id}` | `user:read` | 渲染只读面板 | 404 → `/404` |
| 点击「编辑」 | 打开抽屉表单（`nickname/email/phone` + super_admin 下 `role`） | `PUT /api/v1/admin/users/{user_id}` | `user:update`（`role` 字段 `user:role:update`，super_admin） | `message.success`，刷新列表 | 错误码映射到字段级 |
| 点击「禁用」 | `Modal.confirm`（二次确认 + `reason`） | `POST /api/v1/admin/users/{user_id}/disable` + `X-Confirm-Level: CONFIRM_2` | `user:disable` | `message.success`，状态改 `DISABLED` | `E_USR_4033/4034` → `Modal.warning` |
| 点击「启用」 | `Modal.confirm` | `POST /api/v1/admin/users/{user_id}/enable` | `user:enable` | 同上 | 同上 |
| 点击「删除」 | **三次确认**：`Modal` → 输入用户名确认 → 勾选"我已阅读行政告知" | `DELETE /api/v1/admin/users/{user_id}` + `X-Confirm-Level: CONFIRM_3` | `user:deactivate` | `message.success`，行灰置 | `E_USR_4092/4093` → 引导先走强制转移 / 处理未结工单 |

#### 14.P-14.4 按钮权限矩阵（前端双层防御 + 依赖服务端真源）

| 当前角色 | 目标 `role=FAMILY` | 目标 `role=ADMIN` | 目标 `role=SUPER_ADMIN` | 目标=自身 |
| :--- | :--- | :--- | :--- | :--- |
| `admin` | 查看/编辑/禁用/启用/删除 | 行本身隐藏 | 行本身隐藏 | 仅查看；其他全置灰 tooltip |
| `super_admin` | 全部 | 查看/编辑/禁用/启用/删除 | 仅查看；其他置灰 tooltip：「SUPER_ADMIN 账号不可禁用 / 删除 / 降级」 | 仅查看 + 修改 `nickname/email/phone`；`role` 置灰 |

> 后端最终裁决；前端仅做 UX。

#### 14.P-14.5 校验

- `nickname`：2–64；
- `email`：RFC5322 正则；
- `phone`：中国大陆手机号 `^1[3-9]\d{9}$`（联调根据后端规范调整）；
- `role`：仅 `super_admin` 可选择，且目标为 `SUPER_ADMIN` 时该字段 disabled；
- 禁用 / 删除 `reason`：10–256 / 20–256。

#### 14.P-14.6 i18n keys（片段）

```
page.admin.user.title
page.admin.user.column.username/nickname/email/phone/role/status/lastLogin
page.admin.user.action.view/edit/disable/enable/delete
page.admin.user.dialog.disable.title/content/placeholder
page.admin.user.dialog.delete.warning
page.admin.user.tooltip.cannotOperateSuperAdmin
page.admin.user.tooltip.cannotOperateSelf
```

#### 14.P-14.7 空/载/错态

- 空：`EmptyState`（"暂无符合条件的用户"）；
- 载：`a-skeleton` + `ProTable` 置 `loading`；
- 错：`message.error`；401 → 登录；403 → `/403`；409 业务态行级提示。

#### 14.P-14.8 实时事件

| topic | 行为 |
| :--- | :--- |
| `user.disabled` / `user.enabled` / `user.deactivated` / `user.role.changed` | 若当前用户停留在目标的详情页，`notification.warning` 提醒并刷新 |

---

### 14.P-15 管理员患者档案 `/admin/patients` · `/admin/patients/:patientId`

- 视图：`views/admin/PatientListView.vue` / `PatientDetailView.vue`
- 权限：`admin` / `super_admin`（强制转移主监护仅 `super_admin`）
- 业务目标：管理员跨监护关系**只读**查阅全局患者档案；行政场景下由 super_admin 执行强制转移（FR-PRO-011 / FR-PRO-012）。

#### 14.P-15.1 布局线框（P-15a 列表）

```text
┌─ 查询（keyword / status / gender / primary_guardian_user_id / cursor）─┐
├─ 顶部提示条（info）：「当前为管理员视角，所有 PII 已脱敏；严禁导出或截图外传」│
├─ ProTable：                                                           │
│  short_code | 姓名(脱敏) | 性别 | 年龄 | 状态 | 主监护(脱敏) | 监护成员数 | 活跃任务 | 建档时间 | 操作 │
│  ...                                                                  │
│  操作列：[查看]                                                        │
└───────────────────────────────────────────────────────────────────────┘
```

#### 14.P-15.2 P-15b 详情与强制转移线框

```text
┌─ 档案详情（只读，五分区 Tabs：基础 / 外观 / 电子围栏 / 监护成员 / 时间轴） ─┐
├─ 监护成员 Tab：                                                         │
│  - 表格：user_id / nickname(脱敏) / relation_role / relation_status     │
│  - super_admin 专属按钮行："[强制转移主监护]"（目标选择器=当前 ACTIVE 成员）│
└─────────────────────────────────────────────────────────────────────────┘

强制转移抽屉：
  - target_user_id（只能选本患者现 ACTIVE 非主监护成员）
  - reason（必填，20-256）
  - evidence_url（OSS 上传）
  - [确认]（三次确认：`Modal` + 输入"强制转移"文本 + 勾选行政授权）
```

#### 14.P-15.3 数据字段（详情）

| 字段 | 来源 | 展示 |
| :--- | :--- | :--- |
| `patient_id / profile_no / short_code` | ← | 文本 |
| `patient_name / gender / birthday / age` | ← | 脱敏文本 |
| `status` | ← | `StatusBadge` |
| `avatar_url` | ← | `a-avatar`（仅展示，不可编辑） |
| `chronic_diseases / medication / allergy` | ← | 折叠面板 |
| `appearance` | ← | 只读表单 |
| `fence` | ← | 地图预览（ECharts / 高德 iframe） |
| `guardian_list` | ← | 表格，脱敏 |
| `active_task_id` | ← | 有则链接到 P-03b |
| `profile_version` | ← | 右上角元信息 |

#### 14.P-15.4 交互动作

| 触发 | 动作 | API | 权限码 | 成功反馈 | 失败反馈 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| 列表进入 | 拉取 | `GET /api/v1/admin/patients` | `admin.patient:list` | 渲染表格 | `message.error` |
| 查看详情 | 跳 P-15b | `GET /api/v1/admin/patients/{patient_id}` | `admin.patient:read` | 渲染只读 | 404 → `/404` |
| 点击「强制转移主监护」 | 打开抽屉 | —— | `admin.patient:force-transfer`（仅 super_admin） | —— | admin 按钮不可见 |
| 抽屉[确认] | 发请求 | `POST /api/v1/admin/patients/{patient_id}/guardians/force-transfer` + `X-Confirm-Level: CONFIRM_3` | 同上 | `message.success`，刷新 `guardian_list` + `profile_version`；记录到本页 `AuditTrail` 面板 | `E_PRO_4044/4035/4046/4041` → 表单字段级 |

#### 14.P-15.5 前端双层权限

```vue
<!-- 强制转移按钮 -->
<PermissionButton
  v-permission="'admin.patient:force-transfer'"
  type="primary" danger
  :disabled="!isSuperAdmin"
  @click="openForceTransferDrawer"
>
  {{ t('page.admin.patient.action.forceTransfer') }}
</PermissionButton>
```

- `admin` 账号：按钮不可见；
- `super_admin`：按钮可见但仅当选中了 `ACTIVE` 非主监护成员才可提交。

#### 14.P-15.6 i18n keys（片段）

```
page.admin.patient.title
page.admin.patient.banner.desensitized
page.admin.patient.column.shortCode/name/gender/age/status/primaryGuardian/guardianCount/activeTask
page.admin.patient.tab.basic/appearance/fence/guardians/timeline
page.admin.patient.action.forceTransfer
page.admin.patient.dialog.forceTransfer.title/target/reason/evidence/confirm/warning
```

#### 14.P-15.7 空/载/错态

- 空：`EmptyState`（"暂无匹配的患者档案"）；
- 载：`Skeleton`；
- 错：403 → `/403`；404 → `/404`；422/409 → 字段级提示；写操作失败展示 Trace-Id。

#### 14.P-15.8 审计对照

| 行为 | 后端 `sys_log.action` |
| :--- | :--- |
| 列表查询 | `admin.patient.list`（risk=MEDIUM） |
| 详情查看 | `admin.patient.read`（risk=MEDIUM） |
| 强制转移 | `admin.patient.force_transfer_primary`（risk=CRITICAL, confirm=CONFIRM_3） |

> 前端应在 P-10 审计日志页面保留这些 `action` 的筛选快捷入口。

---

### 14.P-16 字典管理 `/admin/dicts`

> **依据**：SRS FR-GOV-009（P1）、`GET /api/v1/admin/configs` 中 `group=dict` 分组键值对；业务枚举文案（线索驳回原因、物料类型、标签状态展示文本、通知推送模板等）通过系统配置表扩展存储。

- 视图：`views/admin/DictView.vue` · 布局：`DefaultLayout`
- 权限：admin（`dict:read` 只读）/ super_admin（`dict:read` + `dict:write`）
- 业务目标：后台可视化配置业务状态字典文案与推送模板（FR-GOV-009），避免硬编码

#### 14.P-16.1 布局线框

```text
┌ Tab：[线索驳回原因] [物料类型] [标签状态文案] [推送模板] [邮件模板] ──┐
├ 操作条：[新增条目(super_admin)] 搜索:[__] 刷新🔄 ──────────────────┤
├ 表格 ──────────────────────────────────────────────────────────────┤
│ Key | 中文文案 | 英文文案 | 描述 | 更新时间 | 操作                     │
│ clue.reject.speed_anomaly | 移动速度异常 | Speed Anomaly | ... | ... | [编辑(super)] │
└────────────────────────────────────────────────────────────────────┘
```

#### 14.P-16.2 数据来源

| 操作 | API | 说明 |
| :--- | :--- | :--- |
| 读取列表 | `GET /api/v1/admin/configs?group=dict` | 按 `group=dict` 过滤，前端再按 Tab 类型二次分组 |
| 编辑条目 | `PUT /api/v1/admin/configs/{config_key}` | `value` 为 JSON 格式 `{"zh":"...", "en":"..."}` |

> **注**：本页复用 P-09 系统配置的接口与组件，差异仅在于 `group` 过滤和 Tab UI；可在 `ConfigView.vue` 中增加 `dict` 模式 prop，不新建组件。

#### 14.P-16.3 交互动作

| 触发 | 动作 | API | 权限 | 二次确认 |
| :--- | :--- | :--- | :--- | :--- |
| 「编辑」（super_admin） | ProDrawer，输入中文/英文文案，展示 diff | `PUT /api/v1/admin/configs/{config_key}` | `dict:write` | `Modal.confirm` |

#### 14.P-16.4 校验

| 字段 | 规则 |
| :--- | :--- |
| `zh` | 1–100 字 |
| `en` | 1–100 chars，英文字符 |
| `config_key` | 只读，不可修改 |

#### 14.P-16.5 i18n keys

`page.dict.tab.clue_reject`、`page.dict.tab.material_type`、`page.dict.tab.tag_status`、`page.dict.tab.push_template`、`page.dict.tab.email_template`、`page.dict.btn.edit`、`page.dict.label.zh`、`page.dict.label.en`

---

## 15. API 覆盖矩阵

### 15.1 矩阵 A：页面 → API

| 页面 | 方法 | 路径 | 用途 |
| :--- | :--- | :--- | :--- |
| P-01 | POST | `/api/v1/auth/login` | 登录 |
| P-01 | POST | `/api/v1/auth/token/refresh` | 刷新 token（拦截器内自动） |
| P-01 | POST | `/api/v1/auth/password-reset/request` | 忘记密码 |
| P-01 | POST | `/api/v1/auth/password-reset/confirm` | 密码重置确认 |
| P-02 | GET | `/api/v1/dashboard` | 仪表盘指标聚合 |
| P-03a | GET | `/api/v1/rescue/tasks` | 任务治理列表 |
| P-03b | GET | `/api/v1/rescue/tasks/{task_id}/snapshot` | 任务快照 |
| P-03b | GET | `/api/v1/rescue/tasks/{task_id}/full` | 任务全景聚合 |
| P-03b | GET | `/api/v1/rescue/tasks/{task_id}/trajectory/latest` | 轨迹 |
| P-03b | POST | `/api/v1/rescue/tasks/{task_id}/sustained` | 标长期维持 |
| P-03b | POST | `/api/v1/rescue/tasks/{task_id}/close` | 关闭任务 |
| P-04a / P-04b | GET | `/api/v1/clues` | 线索列表（含复核队列过滤） |
| P-04c | POST | `/api/v1/clues/{clue_id}/override` | 覆写（可疑→有效） |
| P-04c | POST | `/api/v1/clues/{clue_id}/reject` | 驳回（可疑→无效） |
| P-05a | GET | `/api/v1/admin/patients` | 患者列表（管理员视图，PII 脆敏化） |
| P-05b | GET | `/api/v1/admin/patients/{patient_id}` | 患者详情（管理员视图） |
| P-06a | GET | `/api/v1/material/orders` | 工单列表 |
| P-06b | POST | `/api/v1/material/orders/{order_id}/approve` | 审批 |
| P-06b | POST | `/api/v1/material/orders/{order_id}/ship` | 发货 |
| P-06b | POST | `/api/v1/material/orders/{order_id}/resolve-exception` | 物流异常处置（补发/作废） |
| P-06b | POST | `/api/v1/material/orders/{order_id}/cancel` | 取消工单 |
| P-07a | GET | `/api/v1/tags/inventory/summary` | 库存摘要 |
| P-07a | POST | `/api/v1/tags/batch-generate` | 批量发号（super_admin） |
| P-07b | GET | `/api/v1/tags/batch-generate/jobs/{job_id}` | 批次详情 |
| P-08 | GET | `/api/v1/notifications/inbox` | 通知收件箱 |
| P-08 | POST | `/api/v1/notifications/{notification_id}/read` | 标已读 |
| P-09 | GET | `/api/v1/admin/configs` | 配置列表 |
| P-09 | PUT | `/api/v1/admin/configs/{config_key}` | 配置修改（super_admin） |
| P-10 | GET | `/api/v1/admin/logs` | 审计日志 |
| P-10 | GET | `/api/v1/admin/logs/export` | 导出日志（super_admin） |
| P-16 | GET | `/api/v1/admin/configs?group=dict` | 字典条目读取 |
| P-16 | PUT | `/api/v1/admin/configs/{config_key}` | 编辑字典条目（super_admin） |
| P-11 | GET | `/api/v1/admin/super/outbox/dead` | DEAD 列表 |
| P-11 | POST | `/api/v1/admin/super/outbox/dead/{event_id}/replay` | DEAD 重放 |
| P-12 | GET | `/api/v1/users/me` | 我的信息 |
| P-12 | PUT | `/api/v1/users/me/password` | 修改密码 |
| P-14a | GET | `/api/v1/admin/users` | 用户列表 |
| P-14b | GET | `/api/v1/admin/users/{user_id}` | 用户详情 |
| P-14b | PUT | `/api/v1/admin/users/{user_id}` | 修改用户 / 角色 |
| P-14b | POST | `/api/v1/admin/users/{user_id}/disable` | 禁用 |
| P-14b | POST | `/api/v1/admin/users/{user_id}/enable` | 启用 |
| P-14b | DELETE | `/api/v1/admin/users/{user_id}` | 逻辑删除 |
| P-15a | GET | `/api/v1/admin/patients` | 管理员患者列表 |
| P-15b | GET | `/api/v1/admin/patients/{patient_id}` | 管理员患者详情 |
| P-15b | POST | `/api/v1/admin/patients/{patient_id}/guardians/force-transfer` | 强制转移主监护（super_admin） |
| 全局（Header） | POST | `/api/v1/ws/ticket` | 获取 WebSocket ticket |

### 15.2 矩阵 B：API → 页面（基于 API_V2.0.md §3、§4 接口概览逐条核对）

| 方法 | 路径 | 域 | 消费页面 | 是否覆盖 | 备注 |
| :--- | :--- | :--- | :--- | :---: | :--- |
| POST | `/api/v1/rescue/tasks` | TASK | — | ❎（家属端） | Android 消费，HC-Coverage 不要求管理端覆盖 |
| POST | `/api/v1/rescue/tasks/{task_id}/close` | TASK | P-03b | ✅ | 管理员代家属关闭任务（LLD §3 允许管理员操作） |
| GET | `/api/v1/rescue/tasks/{task_id}/snapshot` | TASK | P-03b | ✅ | |
| POST | `/api/v1/rescue/tasks/{task_id}/sustained` | TASK | P-03b | ✅ | |
| GET | `/api/v1/rescue/tasks` | TASK | P-03a、P-02（聚合） | ✅ | |
| GET | `/r/{resource_token}` | CLUE | — | ❎（H5） | 匿名扫码入口 |
| POST | `/api/v1/public/clues/manual-entry` | CLUE | — | ❎（H5） | 匿名短码 |
| POST | `/api/v1/clues/report` | CLUE | — | ❎（H5/家属） | 匿名上报 |
| POST | `/api/v1/clues/{clue_id}/override` | CLUE | P-04c | ✅ | |
| POST | `/api/v1/clues/{clue_id}/reject` | CLUE | P-04c | ✅ | |
| GET | `/api/v1/clues` | CLUE | P-04a、P-04b、P-03b | ✅ | |
| GET | `/api/v1/rescue/tasks/{task_id}/trajectory/latest` | CLUE | P-03b | ✅ | |
| POST | `/api/v1/patients` | PROFILE | — | ❎（家属） | |
| PUT | `/api/v1/patients/{patient_id}/profile` | PROFILE | — | ❎（家属） | |
| PUT | `/api/v1/patients/{patient_id}/appearance` | PROFILE | — | ❎（家属） | |
| PUT | `/api/v1/patients/{patient_id}/fence` | PROFILE | — | ❎（家属） | |
| POST | `/api/v1/patients/{patient_id}/missing-pending/confirm` | PROFILE | — | ❎（家属） | |
| POST | `/api/v1/patients/{patient_id}/guardians/invitations` | PROFILE | — | ❎（家属） | |
| POST | `/api/v1/patients/{patient_id}/guardians/invitations/{invite_id}/respond` | PROFILE | — | ❎（家属） | |
| POST | `/api/v1/patients/{patient_id}/guardians/primary-transfer` | PROFILE | — | ❎（家属） | |
| POST | `/api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/respond` | PROFILE | — | ❎（家属） | |
| POST | `/api/v1/patients/{patient_id}/guardians/primary-transfer/{transfer_request_id}/cancel` | PROFILE | — | ❎（家属） | |
| DELETE | `/api/v1/patients/{patient_id}/guardians/{user_id}` | PROFILE | — | ❎（家属） | |
| GET | `/api/v1/patients/{patient_id}` | PROFILE | P-03b（患者卡提速导航） | ✅ | 家属客户端接口，管理端只在任务详情内嵌入导航对象时少量使用 |
| GET | `/api/v1/patients` | PROFILE | — | ❎（家属独占） | 管理端患者档案全局视图使用专用管理员接口 |
| DELETE | `/api/v1/patients/{patient_id}` | PROFILE | — | ❎（家属） | |
| POST | `/api/v1/material/orders` | MAT | — | ❎（家属端申领） | |
| POST | `/api/v1/material/orders/{order_id}/approve` | MAT | P-06b | ✅ | |
| POST | `/api/v1/material/orders/{order_id}/ship` | MAT | P-06b | ✅ | |
| POST | `/api/v1/material/orders/{order_id}/receive` | MAT | — | ❎（家属签收） | |
| POST | `/api/v1/material/orders/{order_id}/cancel` | MAT | P-06b | ✅ | |
| POST | `/api/v1/material/orders/{order_id}/resolve-exception` | MAT | P-06b | ✅ | V2.2 新增，AC-07 P0 |
| POST | `/api/v1/tags/{tag_code}/bind` | MAT | — | ❎（家属） | |
| POST | `/api/v1/tags/{tag_code}/loss/confirm` | MAT | — | ❎（家属） | |
| POST | `/api/v1/tags/batch-generate` | MAT | P-07a | ✅ | super_admin |
| GET | `/api/v1/tags/batch-generate/jobs/{job_id}` | MAT | P-07b | ✅ | |
| GET | `/api/v1/tags/inventory/summary` | MAT | P-07a | ✅ | |
| GET | `/api/v1/material/orders` | MAT | P-06a、P-06b（单条查询） | ✅ | |
| POST | `/api/v1/ai/sessions` | AI | — | ❎（家属） | |
| POST | `/api/v1/ai/sessions/{session_id}/messages` | AI | — | ❎（家属） | |
| POST | `/api/v1/ai/sessions/{session_id}/intents/{intent_id}/confirm` | AI | — | ❎（家属） | |
| POST | `/api/v1/ai/sessions/{session_id}/feedback` | AI | — | ❎（家属） | |
| POST | `/api/v1/ai/poster` | AI | — | ❎（家属） | |
| POST | `/api/v1/auth/register` | GOV | — | ❎（家属/自助注册） | Web 管理端账号由 DBA 开通 |
| POST | `/api/v1/auth/login` | GOV | P-01 | ✅ | |
| POST | `/api/v1/auth/token/refresh` | GOV | 全局拦截器 | ✅ | |
| POST | `/api/v1/auth/password-reset/request` | GOV | P-01 | ✅ | |
| POST | `/api/v1/auth/password-reset/confirm` | GOV | P-01 | ✅ | |
| GET | `/api/v1/users/me` | GOV | P-12、路由守卫 | ✅ | |
| PUT | `/api/v1/users/me/password` | GOV | P-12 | ✅ | |
| PUT | `/api/v1/admin/configs/{config_key}` | GOV | P-09 | ✅ | |
| GET | `/api/v1/admin/configs` | GOV | P-09 | ✅ | |
| GET | `/api/v1/admin/logs` | GOV | P-10 | ✅ | |
| GET | `/api/v1/admin/logs/export` | GOV | P-10（super_admin） | ✅ | V2.2 新增，FR-GOV-007 P1 |
| GET | `/api/v1/notifications/inbox` | GOV | P-08 | ✅ | |
| POST | `/api/v1/notifications/{notification_id}/read` | GOV | P-08 | ✅ | |
| POST | `/api/v1/admin/super/outbox/dead/{event_id}/replay` | GOV | P-11 | ✅ | |
| GET | `/api/v1/admin/super/outbox/dead` | GOV | P-11 | ✅ | |
| GET | `/api/v1/admin/users` | GOV | P-14a | ✅ | V2.1 新增 |
| GET | `/api/v1/admin/users/{user_id}` | GOV | P-14b | ✅ | V2.1 新增 |
| PUT | `/api/v1/admin/users/{user_id}` | GOV | P-14b | ✅ | V2.1 新增 |
| POST | `/api/v1/admin/users/{user_id}/disable` | GOV | P-14b | ✅ | V2.1 新增 |
| POST | `/api/v1/admin/users/{user_id}/enable`  | GOV | P-14b | ✅ | V2.1 新增 |
| DELETE | `/api/v1/admin/users/{user_id}` | GOV | P-14b | ✅ | V2.1 新增 |
| GET | `/api/v1/admin/patients` | PROFILE | P-15a | ✅ | V2.1 新增 |
| GET | `/api/v1/admin/patients/{patient_id}` | PROFILE | P-15b | ✅ | V2.1 新增 |
| POST | `/api/v1/admin/patients/{patient_id}/guardians/force-transfer` | PROFILE | P-15b | ✅ | V2.1 新增，super_admin 独占 |
| GET | `/api/v1/dashboard` | BFF | P-02 | ✅ | |
| GET | `/api/v1/rescue/tasks/{task_id}/full` | BFF | P-03b | ✅ | |
| POST | `/api/v1/ws/ticket` | GOV | 全局 `useWs` | ✅ | |

**覆盖结论**：所有 `/api/v1/admin/*` 管理端独占接口、所有需要管理端参与的治理接口（任务关闭/长期维持、线索复核、工单审批/发货/取消、标签批量发号、库存摘要、通知收件箱与已读）均 100% 覆盖。家属端/H5 独占接口标记为 ❎ 属合理未覆盖，不违反 HC-Coverage。

---

## 16. 错误码映射（code → i18n key）

> 严格对齐 API_V2.0.md §2。如后端实际错误码与此表不一致，以后端为准，前端补充兜底 `error.UNKNOWN`。

### 16.1 通用与网关（API §2.1）

| code | i18n key | 展示位置 |
| :--- | :--- | :--- |
| `E_REQ_4001` | `error.E_REQ_4001`（参数缺失） | 表单字段级 |
| `E_REQ_4002` | `error.E_REQ_4002`（参数格式错误） | 表单字段级 |
| `E_REQ_4003` | `error.E_REQ_4003`（保留头伪造） | `Modal.error` + 登出 |
| `E_REQ_4005` | `error.E_REQ_4005`（ID 不合法） | `message.error` |
| `E_REQ_4291` | `error.E_REQ_4291`（限流） | `notification.warning` + 指数退避 |
| `E_AUTH_4001` | `error.E_AUTH_4001`（凭据错误） | 表单字段级 |
| `E_AUTH_4011` | `error.E_AUTH_4011`（未登录） | 跳 `/login` |
| `E_AUTH_4031` | `error.E_AUTH_4031`（越权） | `Modal.warning` |
| `E_AUTH_4091` | `error.E_AUTH_4091`（账号锁定） | 按钮禁用 + 倒计时 |

### 16.2 领域错误（TASK/CLUE/MAT/AUTH/GOV）

| code | 含义（简述） | UI 策略 |
| :--- | :--- | :--- |
| `E_TASK_4091` | 任务状态非法（无法操作） | 按钮禁用 + tooltip，刷新快照 |
| `E_TASK_4092` | 同患者活跃任务冲突 | `message.warning` + 刷新列表 |
| `E_TASK_4093` | 状态机非法跃迁 | 刷新并禁用相关操作 |
| `E_CLUE_4091` | 线索非 PENDING_REVIEW | 刷新列表；按钮禁用 |
| `E_CLUE_4092` | 已被其他管理员处理 | 刷新行；`message.info` |
| `E_MAT_4091` | 工单状态机非法跃迁 | 按钮禁用 + tooltip |
| `E_MAT_4092` | 物流单号重复 | 表单字段级 |
| `E_MAT_4093` | 库存不足 | `notification.error` + 引导批量发号 |
| `E_MAT_4094` | 批量发号超限 | 表单级 |
| `E_MAT_4095` | 同批次进行中 | 按钮禁用 |
| `E_GOV_4091` | 配置值类型错 | 字段级 |
| `E_GOV_4092` | 配置越界 | 字段级 |
| `E_GOV_4093` | 死信分区冲突（同分区有重放中） | 行置灰 + 自动刷新 |
| `E_GOV_4094` | 死信已归档 | 行淡出 |
| `error.UNKNOWN` | 未知错误兜底 | `message.error` + Trace-Id |

### 16.3 `error.ts` 语言包示例

```ts
// src/locales/zh-CN/error.ts
export default {
  UNKNOWN: '未知错误，请稍后重试',
  forbidden: '权限不足',
  rateLimited: '操作过于频繁，请稍后再试',
  E_AUTH_4001: '用户名或密码错误',
  E_AUTH_4011: '登录已过期，请重新登录',
  E_AUTH_4031: '您没有权限执行此操作',
  E_AUTH_4091: '账号已临时锁定，请 {seconds}s 后重试',
  E_TASK_4091: '任务状态不允许该操作',
  E_CLUE_4091: '该线索已被处理',
  E_MAT_4091: '工单状态不允许该操作',
  E_MAT_4092: '物流单号已存在',
  E_GOV_4093: '该分区已有正在进行的重放',
  // ...
};
```

---

## 17. 性能与可观测

### 17.1 性能预算

| 指标 | 目标 |
| :--- | :--- |
| 首屏 TTI（本地联调、无缓存） | ≤ 2.5s |
| 主包 gzip 体积 | ≤ 350 KB（vendor-vue + vendor-antdv 独立 chunk） |
| 路由首次加载 | ≤ 1.5s（懒加载 + 预加载） |
| 长列表（> 500 行） | 启用虚拟滚动（`a-table` `scroll.y` + `rowSelection` 受控） |
| 图表 | 按需注册 ECharts 组件，单图 < 50ms 首绘（小数据集） |
| 图片 | 所有 `<img>` 懒加载 `loading="lazy"`；缩略图走 CDN `?w=...` 参数 |

### 17.2 监控

| 能力 | 方案 |
| :--- | :--- |
| 错误上报 | `@sentry/vue`；`Sentry.setTag('trace_id', ...)` |
| 性能指标 | Web Vitals（LCP / INP / CLS）上报 Sentry |
| SourceMap | 构建生成但仅上传 Sentry，CDN 不分发（HC-07） |
| 关键操作埋点 | `router.afterEach` + 高危操作（覆写/驳回/配置修改/DEAD 重放）显式埋点 |

---

## 18. 安全规范

| 维度 | 规范 |
| :--- | :--- |
| XSS | 禁止 `v-html` 渲染后端内容；如必须，走 DOMPurify；通知正文、AI 摘要强制过滤 |
| CSRF | 采用 Bearer Token（非 Cookie 会话）；若启用 Cookie，遵循后端双提交 |
| 存储 | `localStorage` 仅存 `accessToken / refreshToken / app 偏好`；其它数据（PII）禁止落盘 |
| URL | 所有 ID 路径参数 `encodeURIComponent`；敏感过滤条件不拼到 URL（改 POST 查询） |
| 剪贴板 | 复制 trace_id / 工单号允许；复制 token 与 PII 禁止 |
| 权限 | 双层：路由 meta + `v-permission`；且网关为最终权威 |
| 日志 | 禁止 `console.log(user)`、`console.log(token)`；Sentry breadcrumb 同样脱敏 |
| 重放攻击 | 写接口 `X-Request-Id` 唯一；用户主动重试复用 |
| 依赖 | `npm audit --omit=dev` 与 Snyk 每周扫描；禁止引入未审查 npm 包 |
| CSP | 生产 Nginx `Content-Security-Policy: default-src 'self' <CDN>; connect-src 'self' <API> <WS>` |
| Trace 展示 | 错误提示仅展示 Trace-Id 前 12 位，避免完整 ID 泄漏 |

---

## 19. 构建、部署与回滚（Nginx + CDN）

### 19.1 拓扑

```text
[用户浏览器]
   │  https://admin.codehome.example.com/index.html
   ▼
[Nginx（入口）]  ──────┐
   │  /assets/*  302   │
   │  /ws/*      WS    │
   │  /api/*     proxy │
   ▼                   ▼
[CDN] ←── origin ── [对象存储：/dist/<version>/]
   │
   ▼
[后端 API 网关 / WebSocket 网关]
```

1. `index.html` 由 Nginx 托管，`Cache-Control: no-cache`（保证发布即生效）；
2. `/assets/*`（带 hash）由 CDN 长缓存 `public, max-age=31536000, immutable`；
3. `/api/*`、`/ws/*` 由 Nginx 反代到后端；
4. 不同版本产物同时存在对象存储 `/dist/<version>/`，切换发布仅改 `index.html`；最近 N 个版本保留便于秒级回滚。

### 19.2 构建

```bash
pnpm install --frozen-lockfile
pnpm typecheck
pnpm lint && pnpm lint:style
pnpm test
VITE_APP_VERSION=$GIT_TAG pnpm build
# 产物：dist/index.html + dist/assets/*-<hash>.js|css
```

### 19.3 发布与回滚流程

1. 上传 `dist/assets/*` 至对象存储 `/dist/<version>/assets/`，CDN 预热；
2. 上传 `dist/index.html` 至 Nginx `html/admin/index.html.next`；
3. 校验 `curl https://admin.../index.html.next` 引用的 JS 全部 200；
4. `ln -sfn index.html.next index.html` 原子切换；
5. 健康检查 `curl /healthz`；
6. **回滚**：`ln -sfn index.html.prev index.html` 即可；CDN 资产保留保证可回。

### 19.4 Nginx 关键配置

```nginx
server {
  listen 443 ssl http2;
  server_name admin.codehome.example.com;

  # 安全头
  add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
  add_header X-Content-Type-Options "nosniff" always;
  add_header X-Frame-Options "DENY" always;
  add_header Referrer-Policy "strict-origin-when-cross-origin" always;
  add_header Content-Security-Policy "default-src 'self'; script-src 'self' https://cdn.codehome.example.com; style-src 'self' 'unsafe-inline' https://cdn.codehome.example.com; connect-src 'self' https://api.codehome.example.com wss://api.codehome.example.com; img-src 'self' data: https:; font-src 'self' https://cdn.codehome.example.com" always;

  root /usr/share/nginx/html/admin;
  index index.html;

  gzip on;
  gzip_types text/plain application/javascript application/json text/css image/svg+xml;
  brotli on;

  # 1) index.html — 无缓存
  location = /index.html {
    add_header Cache-Control "no-cache, no-store, must-revalidate" always;
  }

  # 2) SPA 路由兜底
  location / {
    try_files $uri $uri/ /index.html;
  }

  # 3) 静态资产 — 长缓存（实际 CDN 接管；源站兜底同样策略）
  location /assets/ {
    expires 365d;
    add_header Cache-Control "public, max-age=31536000, immutable";
    access_log off;
  }

  # 4) API 反代
  location /api/ {
    proxy_pass https://api-gateway.internal/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto https;
    proxy_set_header X-Trace-Id $http_x_trace_id;
    proxy_set_header X-Request-Id $http_x_request_id;
    # 明确禁止客户端伪造保留头
    proxy_set_header X-User-Id "";
    proxy_set_header X-User-Role "";
    proxy_read_timeout 60s;
  }

  # 5) WebSocket
  location /ws/ {
    proxy_pass https://ws-gateway.internal/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_read_timeout 3600s;
  }

  # 6) 健康检查
  location = /healthz { return 200 "ok"; }
}
```

### 19.5 版本号与联调自检

| 项 | 规则 |
| :--- | :--- |
| 构建版本 | `VITE_APP_VERSION` 注入全局 `__APP_VERSION__`，`/me` 页面「关于」展示 |
| 后端版本对照 | 调 `GET /actuator/info`（如开放）或后端 `/api/v1/dashboard` 返回版本字段；联调时打印到控制台 |
| 灰度 | 通过 Nginx `map $cookie_gray` 路由到 `index.html.gray`；仅 admin 账号白名单生效 |

---

## 20. 一致性自检报告

### 20.1 本手册内部一致性

| 检查项 | 覆盖状态 | 不符合项 |
| :--- | :---: | :--- |
| 所有 API 路径与方法与 `API_V2.0.md` §3、§4 接口概览逐字对齐 | ✅ | 无（见 §15.2 逐条核对） |
| ID 字段（task_id / patient_id / clue_id / order_id / event_id / user_id / invite_id / transfer_request_id / tag_code）一律按 string 处理（HC-ID-String） | ✅ | 无；`request.ts` 禁用默认 JSON.parse 对 string 保持原样 |
| HC-Stack 遵守（Vue3 + TS + AntDV + ECharts + Pinia + vue-i18n + Vite） | ✅ | 无；禁止依赖清单见 §2.2 |
| 品牌色 `#F97316` / `#0EA5E9` 贯穿亮/暗主题与 ECharts 主轴/辅轴 | ✅ | §4.1、§4.5 |
| 亮/暗双主题均给出 token 与切换联动（`ThemeSwitch` + ECharts `registerTheme`） | ✅ | §4.3、§4.4 |
| 角色仅 `admin` / `super_admin`，按钮级权限码齐全 | ✅ | §11.1 / §11.2；每页操作表列出所需权限码 |
| i18n 覆盖所有可见文本（含 AntDV locale / dayjs / ECharts） | ✅ | §5；每页给出 `i18n keys` |
| 每页面给出线框 + 字段 + 交互 + 权限码 + i18n + 空/错/载态 | ✅ | §14 全页覆盖 |
| 矩阵 B 对管理端应覆盖的 API 100% 覆盖；家属/H5 独占接口合理标记未覆盖 | ✅ | §15.2 |
| HC-06 禁用短信 | ✅ | P-01 忘记密码走邮件；UI 无短信相关按钮 |
| HC-05 WebSocket 定向订阅 | ✅ | §10.2；禁止通配订阅 |
| Nginx + CDN 部署配置给出 | ✅ | §19 |
| HC-03 `X-Request-Id`、HC-04 `X-Trace-Id` 全链路注入 | ✅ | §9.1 |
| HC-08 客户端禁止伪造 `X-User-Id` / `X-User-Role` | ✅ | §9.1 拦截器显式 `delete` |
| 高危操作均有二次确认矩阵 | ✅ | §11.6 |

### 20.2 与 V2 基线交叉一致性

| 基线 | 交叉点 | 结论 |
| :--- | :--- | :--- |
| SRS §FR-TASK / FR-CLUE / FR-MAT / FR-GOV | 功能需求映射到 P-03~P-11 | ✅ 全部覆盖；P-05 / P-13 仅 UI 需求，无 API 支撑需求未缺失 |
| SADD HC-01~HC-08 | 架构硬约束映射到 §1、§10、§9、§11、§19 | ✅ 全部显式落地 |
| LLD §3（任务状态机） | P-03b 状态时间轴 + 二次确认矩阵 | ✅ 非法跃迁由后端 `E_TASK_4091/93` 承载 |
| LLD §4（线索复核） | P-04a~c 覆写/驳回流程 | ✅ 状态机 `SUBMITTED→PENDING→PENDING_REVIEW→VALID/INVALID` |
| LLD §5（患者监护） | P-05 患者档案只读视图 | ✅ 管理端不修改监护关系，避免逾权 |
| LLD §6（物资/标签状态机） | P-06b / P-07a | ✅ OrderState / TagState 与 §15.1 枚举一致 |
| LLD §8（GOV 数据模型） | P-09/P-10/P-11 / P-08 | ✅ `sys_config` / `sys_log` / `sys_outbox_log` / `notification_inbox` |
| DBD §4.14~§4.19 | 字段类型（int64 ID → string 传输） | ✅ HC-ID-String |
| API §1.1 ID 传输约束 | `request.ts` 保持 string | ✅ `transformResponse` 保持原样 |
| API §1.2 通用 Header | 注入/清洗逻辑 | ✅ §9.1 |
| API §1.5 Offset/Cursor | ProTable 双模式；审计日志强制 Cursor | ✅ §12.4 / §14.P-10 |
| API §4 WebSocket | ticket → 定向订阅 → 心跳 → 重连 | ✅ §10 |
| BDD V2.0 错误码 | `error.ts` 映射 | ✅ §16 |

### 20.3 高危点清单（需评审确认）

| # | 事项 | 影响 | 建议 |
| :---: | :--- | :--- | :--- |
| 1 | `/api/v1/dashboard` 对管理员视角的额外字段在 API §3.7.1 已明确（`active_task_count` / `unread_notification_count` 等） | P-02 字段来源 | 若后端暂不区分，前端退化聚合三次 list 调用取 total |
| 2 | API V2.0 未提供 `/api/v1/admin/users` | 无用户治理页 | 已在 §13 备注声明；待后续版本新增接口时扩展 P-XX |
| 3 | `/api/v1/clues` 能否按 `review_state` 过滤在 API §3.2.6 需确认 | P-04a 默认过滤 | 若不支持，前端在内存过滤；性能以 cursor 分页保障 |
| 4 | 批量发号 `/api/v1/tags/batch-generate` 是否校验 super_admin | P-07a 按钮显示 | 前端按 `v-permission="'tag:batch-generate'"` 做双层防御 |
| 5 | DOMPurify 白名单策略 | AI 摘要 / 通知正文 | 采用 "仅文本 + 链接" 白名单；禁止 `<script>` `<iframe>` |

---

## 21. 测试与工程化

### 21.1 测试金字塔

| 层级 | 工具 | 范围 | 覆盖目标 |
| :--- | :--- | :--- | :--- |
| 单元 | Vitest + @vue/test-utils | `utils/*`、`composables/*`、`stores/*` | ≥ 80% 行覆盖 |
| 组件 | Vitest + @vue/test-utils | `components/pro/*`、`components/domain/*`、`PermissionButton` | ≥ 70% |
| 集成 | Vitest + MSW | `api/*` 与 `composables/useXxx` | 关键路径 |
| E2E | Playwright | 登录 → 复核线索 → 审批工单 → DEAD 重放 | 冒烟回归 |

### 21.2 必须的 E2E 场景

| # | 场景 | 验证点 |
| :---: | :--- | :--- |
| 1 | 登录 → 仪表盘 | token 持久化；KPI 渲染 |
| 2 | 线索复核覆写 | 状态机流转 + 审计记录 trace_id |
| 3 | 工单审批 + 发货 | 按钮状态机 + `tracking_no` 校验 |
| 4 | 配置修改 diff 二次确认 | super_admin 角色 + 非 super 按钮禁用 |
| 5 | DEAD 重放（mock） | 分区冲突时按钮禁用 |
| 6 | 语言切换中英 | AntDV 组件文案 + dayjs + ECharts 图例 |
| 7 | 亮/暗主题切换 | AntDV + ECharts 主题同步 |
| 8 | WS 定向订阅 | 订阅/反订阅计数一致；离开页面不遗漏 unsubscribe |

### 21.3 CI 门禁

| 阶段 | 命令 | 阻断 |
| :--- | :--- | :--- |
| 依赖安全 | `pnpm audit --prod --audit-level=high` | high 及以上失败 |
| Type | `pnpm typecheck` | 任一 error |
| Lint | `pnpm lint` / `pnpm lint:style` | `--max-warnings 0` |
| 单测 | `pnpm test` | 覆盖率阈值失败 |
| 构建 | `pnpm build` | 产物体积超预算失败 |
| 提交 | commitlint（Conventional Commits） | 不合规阻断 |

### 21.4 PR Checklist（前端研发必勾）

- [ ] API 路径与方法与 `v2/API_V2.0.md` 一致
- [ ] 所有 ID 按 string 处理，未出现 `Number(xxxId)`
- [ ] i18n keys 在 `zh-CN` 与 `en-US` 同步
- [ ] 新增按钮带 `v-permission` 或 `PermissionButton`
- [ ] 高危操作带二次确认
- [ ] 空/错/载态均已处理
- [ ] Trace-Id 在错误展示中回显
- [ ] 如涉及 WS，定向订阅 + 离开反订阅
- [ ] 不引入未审查第三方依赖
- [ ] 无 `console.log` 明文 PII/Token

---

## 22. 附录

### 22.1 枚举速查（与 LLD / DBD 对齐）

| 枚举 | 值 | 颜色（亮/暗同用） |
| :--- | :--- | :--- |
| `TaskState` | `CREATED / ACTIVE / SUSTAINED / CLOSED_FOUND / CLOSED_FALSE_ALARM` | 灰 / `#F97316` / `#0EA5E9` / `#22C55E` / `#EF4444` |
| `ClueReviewState` | `SUBMITTED / PENDING / PENDING_REVIEW / VALID / INVALID` | 灰 / `#0EA5E9` / `#F59E0B` / `#22C55E` / `#EF4444` |
| `OrderState` | `PENDING_AUDIT / PENDING_SHIP / SHIPPED / RECEIVED / CANCELLED / EXCEPTION` | `#F59E0B` / `#0EA5E9` / `#A855F7` / `#22C55E` / 灰 / `#EF4444` |
| `TagState` | `UNBOUND / ALLOCATED / BOUND / SUSPECTED_LOST / LOST / VOIDED` | 灰 / `#0EA5E9` / `#22C55E` / `#F59E0B` / `#EF4444` / 深灰 |
| `MissingState` | `NORMAL / MISSING_PENDING / MISSING` | `#22C55E` / `#F59E0B` / `#EF4444` |
| `NotificationCategory` | `ALERT / SYS / BIZ` | `#EF4444` / `#0EA5E9` / `#F97316` |
| `Role` | `admin / super_admin` | — |

### 22.2 i18n 示例（片段）

```ts
// src/locales/zh-CN/menu.ts
export default {
  dashboard: '仪表盘',
  task: '任务治理',
  clue: { review: '线索复核', all: '全部线索' },
  patient: '患者档案',
  material: { order: '物资工单', tag: '标签库存' },
  notification: '通知中心',
  admin: { config: '系统配置', log: '审计日志', dead: 'DEAD 事件' },
};
```

```ts
// src/locales/en-US/menu.ts
export default {
  dashboard: 'Dashboard',
  task: 'Tasks',
  clue: { review: 'Clue Review', all: 'All Clues' },
  patient: 'Patients',
  material: { order: 'Material Orders', tag: 'Tag Inventory' },
  notification: 'Notifications',
  admin: { config: 'System Config', log: 'Audit Logs', dead: 'Dead Letters' },
};
```

### 22.3 术语 vs 后端字段映射（对齐 API §1.12）

| UI 术语 | 后端字段 |
| :--- | :--- |
| 任务 | `rescue_task` / `task_id` |
| 线索 | `clue_record` / `clue_id` |
| 工单 | `material_order` / `order_id` |
| 标签 | `tag_asset` / `tag_code` |
| 事件 | `sys_outbox_log` / `event_id` |
| 审计 | `sys_log` |
| 配置 | `sys_config` / `config_key` |

---

## 23. 总结

本手册 WAHB V2.0 严格对齐 V2 基线文档，覆盖：

1. **全局工程规范**（§1–§12）：硬约束、技术栈、目录、主题、i18n、布局、路由、Pinia、HTTP、WS、权限、公共组件。
2. **页面规格**（§13–§14）：P-01 登录、P-02 仪表盘、P-03 任务治理、P-04 线索复核、P-05 患者档案（管理员全局视图，复用 P-15）、P-06 物资工单、P-07 标签库存、P-08 通知中心、P-09 系统配置、P-10 审计日志（含导出）、P-11 DEAD 事件、P-12 个人中心、P-13 错误页、**P-14 用户管理**、**P-15 管理员患者档案**、**P-16 字典管理**，共 **16 个功能页面组**。
3. **API 覆盖矩阵**（§15）：管理端相关 API 100% 覆盖；家属端/H5 独占接口合理标记未覆盖。
4. **非功能**（§16–§19）：错误码映射、性能、安全、Nginx + CDN 部署回滚。
5. **自检与测试**（§20–§21）：一致性自检、跨文档交叉核验、测试金字塔、CI 门禁、PR Checklist。

> 后续任何与本手册冲突的实现变更，必须在提交前更新本文件对应章节并提交 DocReview。

