# ChoexManager

个人生活管家 — 多用户 Web 应用，以 Agent 对话为核心交互方式，集成日程管理、记账、密码簿等生活工具。

## 项目仓库

| 仓库 | 说明 | 地址 |
|---|---|---|
| **choex-server** | Go + Gin 后端 API 服务 | https://github.com/choex2025-ops/choex-server |
| **choex-web** | React + Vite 前端 SPA | https://github.com/choex2025-ops/choex-web |
| **choex-manager** | 设计文档与实施计划 | https://github.com/choex2025-ops/choex-manager |

## 系统架构

```
┌─────────────────────────────────────────────────────┐
│                    React 前端 (Vite)                  │
│  ┌──────┬──────────────────────────┬───────────┐    │
│  │ 侧栏  │       中间内容区          │ Agent面板 │    │
│  │ 📅日程│   日程/记账/密码/浏览器    │ 对话+🧠记忆 │    │
│  │ 💳记账│                          │           │    │
│  │ 🔐密码│                          │           │    │
│  │ 🌐浏览│                          │           │    │
│  ├──────┴──────────────────────────┴───────────┤    │
│  │              底部 AI 输入栏                    │    │
│  └──────────────────────────────────────────────┘    │
│         REST API / SSE 流式                         │
└────────────────────┬────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────┐
│                  Gin 后端                             │
│  Auth 层  │  App 路由  │  Agent 引擎  │  LLM 适配    │
│   JWT    │ events    │ Tool Calling │ DeepSeek API │
│  bcrypt  │ bills     │ 工具调度      │  流式 SSE    │
│          │ passwords │ 记忆管理      │              │
│          │ memories  │ 主动建议      │              │
│          │ proxy     │              │              │
│              GORM + go-redis                        │
└────────────────────┬────────────────────────────────┘
                     │
┌────────────────────▼────────────────────────────────┐
│              MySQL 9.6 + Redis 8.8                   │
│  users │ events │ bills │ passwords │ conversations  │
│        │ agent_memories │ memory_versions             │
└─────────────────────────────────────────────────────┘
```

## 技术栈

| 层 | 技术 | 说明 |
|---|---|---|
| 前端框架 | React 19 + Vite + TypeScript | 字节前端标准技术栈 |
| 状态管理 | Redux Toolkit | 大厂 React 项目标配 |
| 后端框架 | Go + Gin | 高性能 REST API |
| ORM | GORM | Go 最成熟的 ORM |
| 数据库 | MySQL 9.6 | 持久化存储 |
| 缓存 | Redis 8.8 | JWT 黑名单 / Agent 上下文 / 限流 |
| 鉴权 | JWT (HS256) + bcrypt | 无状态认证 |
| 加密 | AES-256-GCM | 密码簿密码加密 |
| LLM | DeepSeek API (openai-go) | 流式/非流式对话 |

## 功能概览

### 用户系统
- 多用户注册/登录（邮箱 + 密码）
- JWT 鉴权，24 小时有效期
- 数据完全隔离（所有查询带 user_id 过滤）

### 核心应用

| 应用 | 功能 |
|---|---|
| 📅 日程管理 | 创建/查看/更新/删除日程，支持时间、地点、颜色标签 |
| 💳 记账 | 收支记录，月度统计（按分类汇总），日期筛选 |
| 🔐 密码簿 | AES-256-GCM 加密存储，点击查看解密，分类管理 |
| 🌐 浏览器 | 内嵌 iframe 网页，后端代理绕过 X-Frame-Options 限制 |

### Agent 对话
- **SSE 流式回复**：逐 token 实时显示
- **Tool Calling**：自然语言控制应用
  - "帮我记一笔午餐30元" → 自动创建记账
  - "我明天有什么日程" → 列出日程
  - "帮我创建一个周五下午3点的会议" → 创建日程
- **记忆管理**：可编辑的 Agent 人格文件
  - 多记忆文件支持（如"日常助手""工作模式"）
  - 三版本管理：当前版 / 备份版（自动）/ 自定义版
  - 滑动开关互斥激活
  - 保存时自动备份旧版本，可一键恢复

### 扩展机制
添加新应用只需：
1. **后端**：实现模型 + Service + Handler，注册到 router
2. **前端**：在 `src/apps/` 下创建组件，在 WorkspacePage 中注册
3. 框架自动处理路由、鉴权、侧边栏发现

## 快速启动

### 前置条件

- Go 1.25+
- MySQL 9+ 运行中
- Redis 8+ 运行中
- Node.js 20+

### 后端

```bash
cd choex-server
cp .env.example .env
# 编辑 .env，填入 DEEPSEEK_API_KEY

DB_PASSWORD=choex2025 DB_USER=root go run cmd/server/main.go
# 服务运行在 http://localhost:8080
```

### 前端

```bash
cd choex-web
npm install
npm run dev
# 开发服务器运行在 http://localhost:5173
```

### 使用

1. 打开 http://localhost:5173
2. 注册账号 → 进入工作台
3. 点击左侧图标使用各应用
4. 在底部输入框与 Agent 对话
5. 点击右侧面板 🧠 管理 Agent 记忆

## API 一览

### 认证
| 方法 | 路径 | 鉴权 |
|---|---|---|
| POST | `/api/auth/register` | 否 |
| POST | `/api/auth/login` | 否 |

### Agent
| 方法 | 路径 | 鉴权 |
|---|---|---|
| POST | `/api/agent/chat` | 是 |

### 日程
| 方法 | 路径 |
|---|---|
| GET/POST | `/api/events` |
| PUT/DELETE | `/api/events/:id` |

### 记账
| 方法 | 路径 |
|---|---|
| GET/POST | `/api/bills` |
| DELETE | `/api/bills/:id` |
| GET | `/api/bills/stats` |

### 密码簿
| 方法 | 路径 |
|---|---|
| GET/POST | `/api/passwords` |
| GET/PUT/DELETE | `/api/passwords/:id` |

### 记忆管理
| 方法 | 路径 |
|---|---|
| GET/POST | `/api/memories` |
| DELETE | `/api/memories/:id` |
| PUT | `/api/memories/:id/activate` |
| GET/PUT | `/api/memories/:id/versions` |
| PUT | `/api/memories/:id/restore` |

### 浏览器
| 方法 | 路径 |
|---|---|
| GET | `/api/proxy?url=...` |

## 数据库设计

### MySQL 表

| 表 | 说明 | 关键字段 |
|---|---|---|
| users | 用户 | id, username, email, password_hash |
| events | 日程 | user_id, title, start_time, end_time, color |
| bills | 账单 | user_id, amount, type, category, bill_date |
| passwords | 密码 | user_id, title, encrypted_password |
| agent_memories | 记忆文件 | user_id, name, icon, is_active |
| memory_versions | 记忆版本 | memory_id, version_type, content |

### Redis 缓存

| 用途 | Key 格式 | TTL |
|---|---|---|
| JWT 黑名单 | `jwt_blacklist:{jti}` | Token 有效期 |
| Agent 上下文 | `ctx:{user_id}:{conv_id}` | 30 分钟 |
| API 限流 | `rate:{user_id}:{api}` | 滑动窗口 |

## 项目结构

```
choex-manager/          # 设计文档仓库
├── docs/
│   ├── BUILD_RECORD.md              # 构建流程记录
│   ├── superpowers/
│   │   ├── specs/                   # 设计规格说明书
│   │   └── plans/                   # 三阶段实施计划
│   └── ...
└── README.md

choex-server/           # Go 后端
├── cmd/server/main.go               # 入口
├── internal/
│   ├── agent/          # Agent 引擎
│   ├── config/         # 配置加载
│   ├── database/       # MySQL + Redis 连接
│   ├── handler/        # HTTP 处理器（6 模块）
│   ├── middleware/      # JWT + CORS 中间件
│   ├── model/          # GORM 数据模型（6 表）
│   ├── router/         # 路由注册中心
│   └── service/        # 业务逻辑层（5 模块）
├── llm/                # DeepSeek API 客户端
└── Makefile

choex-web/              # React 前端
├── src/
│   ├── agent-settings/ # 记忆管理 UI
│   ├── api/            # API 调用层
│   ├── apps/           # 应用模块（4 个）
│   ├── components/     # 布局组件
│   ├── hooks/          # 自定义 Hooks（SSE/Auth）
│   ├── pages/          # 页面组件
│   └── store/          # Redux 状态管理
```

## 文档

- [设计规格说明书](docs/superpowers/specs/2026-06-11-choex-manager-design.md) — 完整的系统设计
- [构建流程记录](docs/BUILD_RECORD.md) — 按步骤记录的构建过程与核心代码
- [阶段一计划](docs/superpowers/plans/2026-06-11-phase1-framework.md)
- [阶段二计划](docs/superpowers/plans/2026-06-11-phase2-core-apps.md)
- [阶段三计划](docs/superpowers/plans/2026-06-11-phase3-agent-enhancement.md)

## 后续规划

- [ ] 阶段四：扩展与完善 — 应用注册机制、系统设置、性能优化
- [ ] Agent 主动学习 — 从用户行为中提取模式并建议记忆更新
- [ ] 移动端适配
- [ ] Docker 容器化部署
- [ ] CI/CD 流水线
