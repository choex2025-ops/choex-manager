# 开发者画像与通用提示词

> 基于 ChoexManager 项目的开发习惯分析与提炼

---

## 一、开发习惯总结

### 1.1 流程偏好

| 阶段 | 习惯 | 表现 |
|---|---|---|
| **设计先行** | 在写任何代码之前，先做完整的设计讨论 | 本项目用了约 30 轮对话完成设计确认（UI 线框图、架构图、数据库 schema）后才开始编码 |
| **视觉验证** | 偏好看到可视化的线框图/架构图 | 多次使用 visual companion 展示 UI 布局、架构全景、交互流程，逐页确认后推进 |
| **选型务实** | 以目标公司/团队的技术栈为导向 | 选择 React + Gin + MySQL 是因为"字节跳动用这些" |
| **分阶段执行** | 将大目标拆成多个独立可交付的阶段 | 本项目分 3 个 phase、共 28 个任务，每个 phase 结束都有可用产物 |
| **文档驱动** | 每个阶段都有计划文档，完成有总结文档 | specs/plans/BUILD_RECORD/README/USER_GUIDE 五层文档体系 |
| **多仓库分离** | 前后端分仓库，设计文档单独仓库 | choex-server / choex-web / choex-manager 三个独立 repo |

### 1.2 技术偏好

| 维度 | 偏好 | 本项目实例 |
|---|---|---|
| 后端语言 | Go | Gin + GORM + JWT |
| 前端框架 | React + TypeScript | Vite + Redux Toolkit |
| 状态管理 | Redux Toolkit（大厂标准） | authSlice + chatSlice |
| 数据库 | MySQL + Redis | 6 张表 + Redis 缓存预留 |
| UI 风格 | 深色主题 + 暗色面板 | #0d1117 底色 + #161b22 面板 |
| 认证 | JWT + bcrypt | HS256, 24 小时过期 |
| API 通信 | SSE 流式 | Agent 对话实时输出 |
| 版本控制 | GitHub + gh CLI | 三仓库通过 gh repo create 创建 |

### 1.3 交互设计偏好

- **三栏布局**：左侧应用导航 + 中间内容区 + 右侧助手面板
- **Agent 驱动**：操作入口可通过自然语言完成，不依赖传统表单
- **可编辑记忆**：Agent 人格/知识可手动编辑且支持版本回退
- **滑动开关**：偏好移动端风格的 toggle 交互（互斥激活）
- **内嵌浏览器**：希望在应用内直接查看网页，减少窗口切换

### 1.4 文档习惯

| 文档 | 用途 | 风格 |
|---|---|---|
| 设计规格说明书 | 记录完整设计决策 | 结构化、表格化 |
| 实施计划 | 逐任务拆分，含完整代码 | 步骤级粒度，无 TBD |
| 构建流程记录 | 按时间线复盘 | 命令 + 代码 + 说明 |
| README | 项目门面 | 图文并茂，架构图 |
| 使用说明 | 终端用户手册 | 场景化，FAQ |

---

## 二、不足与改进建议

### 2.1 测试缺失

**现状：** 仅有 LLM 客户端的 9 个单元测试（从旧项目复制），Phase 2-3 的服务层和 Handler 层没有任何测试。

**影响：** 无法保证回归安全性，重构或新增功能可能引入 bug。

**建议：**
```
- 每个 service 至少写 2 个表驱动测试（正常路径 + 异常路径）
- Handler 使用 httptest 写集成测试
- 前端组件至少写 1 个渲染测试（React Testing Library）
- CI 中加入 go test 和 npx tsc --noEmit
```

### 2.2 环境配置硬编码

**现状：** 多处出现硬编码值：
- `DB_PASSWORD=choex2025` 出现在文档、启动命令中
- `ENCRYPTION_KEY` 默认值 `choex2025-32byte-secret-key!!!` 可在代码中看到
- `JWT_SECRET` 默认值 `dev-secret`

**影响：** 容易误将开发密钥带入生产环境。

**建议：**
```
- 启动时检测 JWT_SECRET / ENCRYPTION_KEY 是否为默认值，若是则打 WARNING 日志
- .env.example 中不提供看似真实的默认值，用 PLACEHOLDER 标记
- 启动命令统一为 make run，不再手动传环境变量
```

### 2.3 前端错误处理不充分

**现状：** API 调用普遍使用 `try/catch {}` 空 catch 块（静默失败）。用户发起操作失败时看不到任何提示。

**影响：** 创建日程失败、记账失败等情况用户完全无感知。

**建议：**
```
- 在 API client 层统一处理错误，通过 Redux 全局 error slice 展示 toast
- 或者在每个组件中加入 error state 并用红色提示条展示
- catch 块中至少 console.error 以便调试
```

### 2.4 缺少 Loading 状态

**现状：** Agent 对话有 `streaming` 状态显示"思考中..."，但四大应用（日程/记账/密码簿/浏览器）在数据加载时没有任何 loading 指示。

**建议：**
```
- 添加全局 Loading 组件或骨架屏
- 每个应用组件加入 loading state
```

### 2.5 安全加固空间

**现状：**
- 密码簿的密码通过 API 返回明文（`/api/passwords/:id`），传输层依赖 HTTPS（本地开发无 HTTPS）
- 没有请求频率限制的实际实现（Redis 预留了但未启用）
- JWT 没有刷新机制，24 小时后必须重新登录
- 没有登出时 Token 加入黑名单的实现

**建议：**
```
- 生产环境必须启用 HTTPS
- 实现 Redis 限流中间件
- 添加 JWT refresh token 机制
- 实现登出时 Token 失效
```

### 2.6 跳过 TDD 承诺

**现状：** Phase 1 计划中声明了"TDD. Frequent commits."，但实际执行时全部跳过测试环节。

**建议：**
```
要么坚持 TDD，要么在计划中明确声明测试阶段独立处理。
避免计划与实际执行不一致造成的信任损耗。
```

### 2.7 缺少 CI/CD

**现状：** 完全没有自动化流水线。

**建议：**
```
GitHub Actions 最小配置：
- check: go build + go test + npx tsc --noEmit
- lint: golangci-lint + eslint
构建通过才能合并 PR
```

---

## 三、通用项目启动提示词

以下提示词可以直接用于启动新项目，已经内置了你的偏好和工作流。

### 3.1 完整版（推荐新项目使用）

```
我想启动一个新项目：[一句话描述项目目标]。

请按以下流程推进：

**阶段 0：项目设置**
1. 帮我做技术选型（参考以下偏好）：
   - 后端：Go + Gin + GORM + MySQL + Redis + JWT
   - 前端：React + Vite + TypeScript + Redux Toolkit
   - UI 风格：深色主题（#0d1117 底色）
2. 创建 GitHub 仓库（我用户名 choex2025-ops）
   - 前后端分仓库：[name]-server / [name]-web
   - 如果规格文档较多，单独 [name]-docs 仓库

**阶段 1：设计讨论（先不要写代码）**
1. 用 visual companion 展示 UI 线框图
2. 确认后展示：系统架构图、数据库 schema、API 路由设计
3. 逐页确认直到全部通过
4. 写出设计规格说明书，我审核后进入实现

**阶段 2：实施计划**
1. 按功能模块拆分独立可交付的 phase
2. 每个 phase 拆成 5-15 个 bite-sized 任务
3. 计划中每个任务必须包含：精确文件路径、完整代码、验证步骤
4. 用 checkbox 格式追踪进度

**阶段 3：执行**
1. 按任务顺序执行，每完成一个任务提交一次
2. 后端先做骨架（config → model → service → handler → router → main）
3. 前端后做（store → API → 页面 → 布局 → 集成）
4. 每个 phase 结束做端到端验证

**阶段 4：文档**
1. 构建流程记录（按时间线，含命令和关键代码）
2. README（架构图 + API 一览 + 快速启动）
3. 使用说明（面向终端用户的场景化手册）

**其他偏好：**
- 我喜欢先看设计再动手，设计确认前不要写任何代码
- 复杂交互请用 visual companion 展示线框图
- 如果有多种方案，列出优劣并给我推荐
- 记忆管理/配置类功能我喜欢版本控制和可回退机制
```

### 3.2 精简版（小型项目或快速迭代用）

```
我要做一个 [描述]。请按以下流程：

1. 先和我讨论 2-3 个关键设计决策（技术栈已定：
   后端 Go+Gin+MySQL+JWT，前端 React+Vite+Redux Toolkit，
   深色主题 UI，前后端分仓库）
2. 确认后写实施计划（逐任务含代码和路径，无 TBD）
3. 按任务顺序执行，每任务提交一次
4. 完成后写 README 和构建记录

偏好：
- 设计前不动代码
- 重要交互先画线框图确认
- 分阶段交付，每阶段产出可用物
```

### 3.3 功能追加版（在已有项目上加新功能）

```
我要在现有项目上加 [描述新功能]。当前项目：
- 后端：Go+Gin（repo: choex2025-ops/[name]-server）
- 前端：React+Vite+Redux（repo: choex2025-ops/[name]-web）
- 认证：JWT，所有 API 通过 AuthRequired 中间件

请：
1. 确认新功能对现有架构的影响（是否新增表/路由/组件）
2. 写实施计划
3. 执行并更新文档
```

---

## 四、个人开发 checklist

每次启动新项目时，可以用这个清单自查：

```
□ 设计确认了吗？（UI 线框图 + 架构图 + 数据库 schema）
□ 技术选型和"大厂对齐"了吗？
□ 写了实施计划吗？（含完整代码，无 TBD）
□ 仓库创建了吗？（前后端分仓库）
□ MySQL + Redis 装好了吗？
□ 环境变量放 .env.example 了吗？（不要硬编码）
□ 每个 service 有测试吗？
□ 前端错误处理不是空 catch 吗？
□ 写了 README 和使用说明吗？
□ 构建流程记录了吗？
```

---

## 五、风格速查

| 元素 | 你的偏好 |
|---|---|
| 代码缩进 | Tab |
| 文件夹命名 | kebab-case (agent-settings) |
| Go 包命名 | 单数、小写 (handler, service, model) |
| React 组件 | PascalCase, default export |
| TypeScript 类型 | interface 优先于 type |
| Git 提交信息 | feat:/fix:/docs: 前缀，中文描述可选 |
| 暗色主题 | bg #0d1117, panel #161b22, border #30363d |
| 主色 | #1f6feb (蓝) |
| 成功色 | #3fb950 (绿) |
| 错误色 | #f85149 (红) |
| 字体 | system-ui / -apple-system |
