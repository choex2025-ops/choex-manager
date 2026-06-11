# ChoexManager 设计规格说明书

> 个人生活管家 Web 应用 — 多用户、Agent 驱动、可扩展

## 1. 概述

ChoexManager 是一个**多用户 Web 应用**，提供日程管理、记账、密码簿等生活工具，核心交互通过 Agent 对话完成。Agent 具备主动学习能力，能从用户行为中提取模式并记忆，同时支持用户手动编辑和切换记忆人格。

### 1.1 设计目标

- 搭建可扩展的框架，未来能方便地添加新应用模块
- Agent 作为核心交互入口，支持 Tool Calling 调度各应用
- 多用户注册登录，数据隔离
- 内存记忆可编辑、可回退、可切换

### 1.2 技术栈

| 层 | 选型 | 说明 |
|---|---|---|
| 前端 | React + Vite + TypeScript | SPA，组件化开发 |
| 后端 | Go + Gin | REST API + SSE 流式 |
| 数据库 | MySQL | 持久化存储 |
| 缓存 | Redis | Session/JWT 黑名单/Agent 上下文/限流 |
| LLM | DeepSeek API | 已实现流式/非流式对话封装 |

---

## 2. 页面与 UI 布局

### 2.1 登录/注册页

- 居中卡片式布局
- 邮箱/用户名 + 密码输入
- 登录 / 注册切换
- JWT Token 认证，存入 localStorage

### 2.2 主工作台

```
┌──────┬──────────────────────────────┬───────────┐
│      │                              │  Agent    │
│ 左   │         中间区域              │  对话面板  │
│ 侧   │        (动态内容)             │           │
│ 应   │                              │  🧠 ⚙    │
│ 用   │                              │           │
│ 栏   │                              │  对话     │
│      │                              │  消息     │
│      │                              │  列表     │
├──────┴──────────────────────────────┴───────────┤
│              底部全局输入栏                       │
└────────────────────────────────────────────────┘
```

**三栏职责：**

| 区域 | 说明 |
|---|---|
| 左侧应用栏 (64px) | 竖向图标列表：📅日程、💳记账、🔐密码簿、🌐浏览器、+ 添加、⚙ 系统设置 |
| 中间动态区 | 欢迎页 / 选中的应用页面 / Agent 设置页面 |
| 右侧 Agent 面板 (320px) | 对话消息列表 + 顶部 🧠记忆管理 ⚙更多设置 |
| 底部输入栏 | 全局 AI 输入，发送后输出到 Agent 对话面板 |

### 2.3 欢迎页（默认页）

- ChoexManager 品牌展示（Logo + 标题 + "个人生活管家"）
- 提示"点击左侧应用开始使用"

### 2.4 应用切换

- 点击左侧应用图标 → 中间区域切换为对应应用内容
- 点击 Agent 面板设置图标(🧠) → 中间区域打开记忆管理
- 未选择时显示欢迎页

### 2.5 浏览器应用

- 地址栏 + iframe 内嵌网页视图
- 两种触发：手动输入 URL / Agent 识别 open_url 意图后自动打开
- 后端代理 `/proxy?url=<encoded>` 绕过 iframe 限制

### 2.6 记忆管理

**入口**：右键 Agent 对话栏顶部 🧠 图标

**列表页**：
- 竖向排列的记忆条卡片
- 每条显示：图标、名称、内容预览、最后修改时间
- 激活态：蓝色边框 + 光晕 + "使用中"标签 + 蓝色滑动开关
- 未激活态：暗色背景 + 灰色滑动开关
- 滑动开关互斥：同一时刻仅 1 条激活，激活一条则其余自动关闭

**编辑页**（点击某条进入）：
- 三个版本标签：当前版本 / 备份版本 / 自定义版本
- 文本编辑器（编辑 Agent 系统提示词/记忆）
- 操作栏：从备份恢复、存入自定义版、保存修改、关闭
- 保存流程：旧当前版 → 备份版，新内容 → 当前版；自定义版不受影响
- **每条记忆独立拥有三个版本槽位**

**切换逻辑**：
- 滑动激活一条记忆 → Agent 立即使用该记忆的"当前版本"
- 右侧对话面板 Agent 提示切换成功

---

## 3. 系统架构

### 3.1 分层架构

```
React 前端 (Vite + React Router)
        │
        │ REST / SSE
        ↓
Gin 后端
  ├── Auth 层 (JWT 鉴权中间件)
  ├── App 路由 (/api/calendar, /api/bills, /api/passwords)
  ├── Agent 引擎 (意图解析、工具调度、主动建议)
  ├── LLM 适配 (DeepSeek API, 流式 SSE)
  └── 数据层 (GORM + go-redis)
        │
        ↓
   MySQL + Redis
```

### 3.2 扩展机制

**Go 后端 App 接口：**

```go
type App interface {
    Name()  string   // "calendar"
    Icon()  string   // "📅"
    Routes() []Route // API 路由注册
    Tools()  []Tool  // Agent 工具注册
}
```

新应用实现此接口并注册到 Router，框架自动加载路由和 Agent 工具。

**React 前端约定目录：**

```
src/apps/
  calendar/
    index.tsx    // 主组件
    types.ts     // 类型定义
  bills/
  passwords/
  browser/
```

侧边栏自动发现 `src/apps/*` 目录，路由懒加载。

---

## 4. 数据库设计

### 4.1 MySQL 表结构

#### users

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT PK | 主键 |
| username | VARCHAR(64) | 用户名 |
| email | VARCHAR(128) UNIQUE | 邮箱 |
| password_hash | VARCHAR(256) | bcrypt 哈希 |
| created_at | DATETIME | 注册时间 |
| updated_at | DATETIME | 更新时间 |

#### events（日程）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT PK | 主键 |
| user_id | BIGINT FK | 关联用户 |
| title | VARCHAR(256) | 标题 |
| description | TEXT | 描述 |
| location | VARCHAR(256) | 地点 |
| start_time | DATETIME | 开始时间 |
| end_time | DATETIME | 结束时间 |
| all_day | TINYINT | 全天事件 |
| color | VARCHAR(16) | 标签颜色 |
| created_at / updated_at | DATETIME | 时间戳 |

#### bills（记账）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT PK | 主键 |
| user_id | BIGINT FK | 关联用户 |
| amount | DECIMAL(10,2) | 金额 |
| type | ENUM('income','expense') | 收入/支出 |
| category | VARCHAR(64) | 分类：餐饮/交通/购物/娱乐/其他 |
| note | VARCHAR(256) | 备注 |
| bill_date | DATE | 日期 |
| created_at / updated_at | DATETIME | 时间戳 |

#### passwords（密码簿）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT PK | 主键 |
| user_id | BIGINT FK | 关联用户 |
| title | VARCHAR(128) | 条目名称 |
| url | VARCHAR(512) | 网站 URL |
| username | VARCHAR(128) | 账号 |
| encrypted_password | TEXT | AES-256 加密密码 |
| note | VARCHAR(256) | 备注 |
| category | VARCHAR(64) | 分类 |
| created_at / updated_at | DATETIME | 时间戳 |

#### conversations（Agent 对话）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT PK | 主键 |
| user_id | BIGINT FK | 关联用户 |
| title | VARCHAR(128) | 对话标题 |
| created_at / updated_at | DATETIME | 时间戳 |

#### messages（对话消息）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT PK | 主键 |
| conversation_id | BIGINT FK | 关联对话 |
| role | ENUM('user','assistant') | 角色 |
| content | TEXT | 消息内容 |
| tool_calls | JSON | Agent 调用的工具记录 |
| created_at | DATETIME | 时间戳 |

#### agent_memories（Agent 记忆文件）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT PK | 主键 |
| user_id | BIGINT FK | 关联用户 |
| name | VARCHAR(64) | 记忆名称（如"日常助手"） |
| icon | VARCHAR(16) | 图标 emoji |
| is_active | TINYINT | 是否激活（每个用户仅 1 条为 1） |
| created_at / updated_at | DATETIME | 时间戳 |

#### agent_memory_versions（记忆版本）

| 字段 | 类型 | 说明 |
|---|---|---|
| id | BIGINT PK | 主键 |
| memory_id | BIGINT FK | 关联记忆文件 |
| version_type | ENUM('current','backup','custom') | 版本类型 |
| content | TEXT | 记忆内容（系统提示词） |
| created_at | DATETIME | 版本创建时间 |

### 4.2 Redis 缓存方案

| 用途 | Key 格式 | TTL |
|---|---|---|
| JWT 黑名单 | `jwt_blacklist:{jti}` | Token 有效期 |
| Agent 对话上下文 | `ctx:{user_id}:{conv_id}` | 30 分钟 |
| API 限流 | `rate:{user_id}:{api}` | 滑动窗口 |

---

## 5. API 路由设计

### 5.1 认证

| 方法 | 路径 | 说明 | 鉴权 |
|---|---|---|---|
| POST | `/api/auth/register` | 注册 | 否 |
| POST | `/api/auth/login` | 登录 | 否 |
| POST | `/api/auth/logout` | 退出 | 是 |

### 5.2 日程

| 方法 | 路径 | 说明 | 鉴权 |
|---|---|---|---|
| GET | `/api/events` | 日程列表 | 是 |
| POST | `/api/events` | 创建日程 | 是 |
| PUT | `/api/events/:id` | 更新日程 | 是 |
| DELETE | `/api/events/:id` | 删除日程 | 是 |

### 5.3 记账

| 方法 | 路径 | 说明 | 鉴权 |
|---|---|---|---|
| GET | `/api/bills` | 账单列表 | 是 |
| POST | `/api/bills` | 创建账单 | 是 |
| DELETE | `/api/bills/:id` | 删除账单 | 是 |
| GET | `/api/bills/stats` | 月度统计 | 是 |

### 5.4 密码簿

| 方法 | 路径 | 说明 | 鉴权 |
|---|---|---|---|
| GET | `/api/passwords` | 密码列表(不含明文) | 是 |
| POST | `/api/passwords` | 创建密码条目 | 是 |
| GET | `/api/passwords/:id` | 查看密码(解密) | 是 |
| PUT | `/api/passwords/:id` | 更新密码条目 | 是 |
| DELETE | `/api/passwords/:id` | 删除密码条目 | 是 |

### 5.5 Agent 对话

| 方法 | 路径 | 说明 | 鉴权 |
|---|---|---|---|
| POST | `/api/agent/chat` | 发送消息(SSE 流式) | 是 |
| GET | `/api/agent/conversations` | 对话列表 | 是 |
| GET | `/api/agent/conversations/:id/messages` | 消息历史 | 是 |

### 5.6 记忆管理

| 方法 | 路径 | 说明 | 鉴权 |
|---|---|---|---|
| GET | `/api/memories` | 记忆文件列表 | 是 |
| POST | `/api/memories` | 创建记忆文件 | 是 |
| PUT | `/api/memories/:id` | 更新记忆文件信息 | 是 |
| DELETE | `/api/memories/:id` | 删除记忆文件 | 是 |
| PUT | `/api/memories/:id/activate` | 激活记忆(互斥) | 是 |
| GET | `/api/memories/:id/versions` | 获取三版本内容 | 是 |
| PUT | `/api/memories/:id/versions/current` | 保存当前版本 | 是 |
| PUT | `/api/memories/:id/versions/custom` | 保存自定义版本 | 是 |
| PUT | `/api/memories/:id/restore` | 从备份恢复 | 是 |

### 5.7 浏览器代理

| 方法 | 路径 | 说明 | 鉴权 |
|---|---|---|---|
| GET | `/proxy?url=<encoded>` | 代理访问网页 | 是 |

---

## 6. Agent 设计

### 6.1 Tool Calling 流程

```
用户输入: "帮我记一笔午餐30元"
    ↓
Agent 解析意图 → intent: create_bill, params: {amount:30, category:"午餐"}
    ↓
调用账单 API → 写入数据库 → 返回结果
    ↓
Agent 回复: "已记录：午餐 ¥30.00，本月餐饮累计 ¥850"
```

### 6.2 工具注册

每个 App 通过 `Tools() []Tool` 方法注册 Agent 可调用的工具：

```go
type Tool struct {
    Name        string      // "create_event"
    Description string      // "创建一个新日程"
    Parameters  JSONSchema  // 参数定义
    Handler     ToolFunc    // 执行函数
}
```

Agent 引擎收集所有已注册 App 的工具，组装为 LLM 的 tools 参数发送。

### 6.3 主动建议

- Agent 后台定期分析用户行为模式
- 提取到的模式进入记忆文件的**审核队列**
- 用户查看记忆编辑页时看到待审核项，确认后写入当前版本
- 建议示例："检测到连续3天上午查看日程 → 建议记忆：用户偏好上午处理重要事务"

### 6.4 记忆生效方式

- Agent 每次处理用户输入时，读取当前激活记忆的"当前版本"作为 system prompt
- 切换记忆文件后立即生效，无需重新登录
- 编辑保存后即时生效

---

## 7. 项目目录结构

### 7.1 Go 后端 `choex-server/`

```
choex-server/
├── cmd/server/main.go          # 入口
├── internal/
│   ├── config/config.go        # 配置加载
│   ├── handler/                # HTTP 处理器
│   │   ├── auth.go
│   │   ├── calendar.go
│   │   ├── bill.go
│   │   ├── password.go
│   │   ├── agent.go
│   │   └── memory.go
│   ├── middleware/
│   │   ├── auth.go             # JWT 鉴权
│   │   └── ratelimit.go        # 限流
│   ├── model/                  # GORM 模型
│   │   ├── user.go
│   │   ├── event.go
│   │   ├── bill.go
│   │   ├── password.go
│   │   ├── conversation.go
│   │   └── memory.go
│   ├── service/                # 业务逻辑
│   │   ├── calendar.go
│   │   ├── bill.go
│   │   ├── password.go
│   │   └── memory.go
│   ├── agent/                  # Agent 引擎
│   │   ├── engine.go           # 意图解析与调度
│   │   ├── tools.go            # 工具注册与管理
│   │   ├── chat.go             # 对话管理
│   │   └── proactive.go        # 主动学习与建议
│   ├── llm/
│   │   └── deepseek.go         # DeepSeek 客户端（已有）
│   ├── app/
│   │   └── app.go              # App 接口定义
│   └── router/router.go        # 路由注册
├── migrations/                  # 数据库迁移
├── go.mod
└── go.sum
```

### 7.2 React 前端 `choex-web/`

```
choex-web/
├── src/
│   ├── pages/
│   │   ├── Login.tsx
│   │   └── Workspace.tsx       # 主工作台（layout 壳）
│   ├── components/
│   │   ├── Sidebar.tsx         # 左侧应用栏
│   │   ├── AgentChat.tsx       # 右侧 Agent 对话面板
│   │   ├── InputBar.tsx        # 底部全局输入栏
│   │   └── Welcome.tsx         # 默认欢迎页
│   ├── apps/                   # 应用模块
│   │   ├── calendar/
│   │   │   ├── index.tsx       # 日程主组件
│   │   │   └── types.ts
│   │   ├── bills/
│   │   │   ├── index.tsx
│   │   │   └── types.ts
│   │   ├── passwords/
│   │   │   ├── index.tsx
│   │   │   └── types.ts
│   │   └── browser/
│   │       └── index.tsx
│   ├── agent-settings/         # Agent 设置页
│   │   ├── MemoryList.tsx      # 记忆列表
│   │   ├── MemoryEditor.tsx    # 记忆编辑
│   │   └── types.ts
│   ├── hooks/                  # 自定义 Hooks
│   ├── api/                    # API 调用封装
│   ├── store/                  # 状态管理
│   ├── App.tsx
│   └── main.tsx
├── package.json
└── vite.config.ts
```

---

## 8. 开发阶段规划

### 阶段一：框架搭建
- 后端项目骨架 (Gin + GORM + JWT)
- 前端项目骨架 (React + Vite + Router)
- 登录注册 + JWT 鉴权
- 主工作台三栏布局（壳）
- Agent 对话面板（基础对话 + SSE 流式）

### 阶段二：核心应用
- 日程管理 CRUD（前后端）
- 记账 CRUD + 统计（前后端）
- 密码簿 CRUD + 加密（前后端）
- 浏览器应用（内嵌 + 代理）

### 阶段三：Agent 增强
- Agent Tool Calling（调度应用 API）
- 记忆管理（文件 + 版本 + 滑动开关）
- 主动学习与建议
- Agent 上下文优化

### 阶段四：扩展与完善
- 应用注册机制完善
- 系统设置页
- 性能优化与缓存

---

## 9. 设计决策记录

| 决策 | 选择 | 理由 |
|---|---|---|
| 运行形态 | 纯 Web 应用 | 跨平台、用户描述布局天然适配 |
| 架构模式 | Go API + React SPA 分离 | 职责清晰、各自技术栈最优 |
| 前端框架 | React + Vite | 字节技术栈、生态丰富 |
| 后端框架 | Gin | 高性能、字节 Go 服务主流 |
| 数据库 | MySQL + Redis | 互联网标配、多用户支持 |
| Agent 模式 | 主动型 Tool Calling | 操作+咨询+建议全覆盖 |
| 扩展方式 | 接口+约定目录 | 最小侵入、插拔式注册 |
| 记忆版本 | 三槽位(当前/备份/自定义) | 支持回退+人格切换 |
| 记忆管理入口 | Agent 面板顶部 | Agent 相关设置集中管理 |
