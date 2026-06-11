# ChoexManager 系统构建流程记录

> 按实施顺序详细记录每一步的操作、关键指令与核心实现代码。

## 概览

| 阶段 | 名称 | 任务数 | 耗时约 |
|---|---|---|---|
| 一 | 框架搭建 | Task 1-14 | 前后端骨架、认证、Agent 对话 |
| 二 | 核心应用 | Task 15-23 | 日程/记账/密码簿/浏览器 CRUD |
| 三 | Agent 增强 | Task 24-28 | Tool Calling、记忆管理 |

共涉及三个 Git 仓库：
- **choex-manager** — 设计文档与实施计划
- **choex-server** — Go + Gin 后端
- **choex-web** — React + Vite 前端

---

## 环境准备

### 安装基础软件

```bash
# MySQL 9.6
brew install mysql
brew services start mysql

# 设置 root 密码并创建数据库
mysql -u root -e "ALTER USER 'root'@'localhost' IDENTIFIED BY 'choex2025';"
mysql -u root -pchoex2025 -e "CREATE DATABASE IF NOT EXISTS choex_manager CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# Redis 8.8
brew install redis
brew services start redis

# 验证
redis-cli PING        # 应返回 PONG
mysql -u root -pchoex2025 -e "SELECT 1;"  # 应返回 1

# GitHub CLI
brew install gh
gh auth login
```

### 技术选型

| 层级 | 技术 | 选型依据 |
|---|---|---|
| 后端框架 | Go + Gin | 字节跳动内部 Go 服务主流框架 |
| 前端框架 | React + Vite + TypeScript | 字节前端标准技术栈 |
| 状态管理 | Redux Toolkit | 大厂 React 项目标配 |
| 数据库 | MySQL 9.6 + Redis 8.8 | 互联网标配 |
| ORM | GORM | Go 生态最成熟的 ORM |
| 鉴权 | JWT (HS256) + bcrypt | 无状态认证 + 密码安全哈希 |
| LLM | DeepSeek API (openai-go SDK) | 已实现流式/非流式封装 |
| 加密 | AES-256-GCM | 密码簿密码加密存储 |

---

## 阶段一：框架搭建 (Task 1-14)

### Task 1: 创建 choex-server 仓库骨干

**操作：** 创建 Go 项目目录结构，复制已有的 LLM 客户端代码。

```bash
mkdir -p ~/ClaudeProjects/choex-server/internal/{config,database,middleware,model,handler,service,router,agent}
cp -r ~/ClaudeProjects/choex-manager/llm ~/ClaudeProjects/choex-server/
```

**核心文件：** `internal/config/config.go` — 环境变量配置加载器

```go
func Load() *Config {
    return &Config{
        ServerPort:    getEnv("SERVER_PORT", "8080"),
        DBHost:        getEnv("DB_HOST", "localhost"),
        JWTSecret:     getEnv("JWT_SECRET", "dev-secret"),
        EncryptionKey: getEnv("ENCRYPTION_KEY", "choex2025-32byte-secret-key!!!"),
        // ...
    }
}
func (c *Config) DSN() string {
    return c.DBUser + ":" + c.DBPassword + "@tcp(" + c.DBHost + ":" + c.DBPort + ")/" + c.DBName + "?charset=utf8mb4&parseTime=True&loc=Local"
}
```

**添加依赖：**

```bash
cd ~/ClaudeProjects/choex-server
go get github.com/gin-gonic/gin gorm.io/gorm gorm.io/driver/mysql \
  github.com/golang-jwt/jwt/v5 github.com/redis/go-redis/v9 \
  golang.org/x/crypto/bcrypt
go mod tidy
```

**推送：**

```bash
gh repo create choex2025-ops/choex-server --public --source=. --remote=origin --push
```

### Task 2: 安装 MySQL + Redis

（见上方环境准备部分）

### Task 3: 数据库连接与 User 模型

**核心文件：** `internal/database/database.go` — GORM + Redis 初始化

```go
func Init(cfg *config.Config) {
    DB, _ = gorm.Open(mysql.Open(cfg.DSN()), &gorm.Config{
        Logger: logger.Default.LogMode(logger.Info),
    })
    Redis = redis.NewClient(&redis.Options{Addr: cfg.RedisAddr()})
}
```

**User 模型：** `internal/model/user.go`

```go
type User struct {
    ID           uint64    `gorm:"primaryKey;autoIncrement" json:"id"`
    Username     string    `gorm:"size:64;not null" json:"username"`
    Email        string    `gorm:"size:128;uniqueIndex;not null" json:"email"`
    PasswordHash string    `gorm:"size:256;not null" json:"-"`
}
```

**入口 main.go 中自动迁移：**

```go
database.DB.AutoMigrate(&model.User{})
```

### Task 4-5: 认证服务与 Handler

**JWT Token 生成：** `internal/service/auth.go`

```go
func (s *AuthService) GenerateToken(userID uint64, email string) (string, error) {
    claims := &Claims{
        UserID: userID, Email: email,
        RegisteredClaims: jwt.RegisteredClaims{
            ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
        },
    }
    token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
    return token.SignedString([]byte(s.cfg.JWTSecret))
}
```

**密码哈希：**

```go
func (s *AuthService) HashPassword(password string) (string, error) {
    bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
    return string(bytes), err
}
```

**注册/登录 Handler：** `internal/handler/auth.go`
- `POST /api/auth/register` — 校验邮箱唯一 → bcrypt 哈希密码 → 创建用户 → 返回 JWT
- `POST /api/auth/login` — 查询用户 → bcrypt 验证 → 返回 JWT

### Task 6: JWT + CORS 中间件

```go
// internal/middleware/auth.go
func AuthRequired(svc *service.AuthService) gin.HandlerFunc {
    return func(c *gin.Context) {
        header := c.GetHeader("Authorization")
        token := strings.TrimPrefix(header, "Bearer ")
        claims, _ := svc.ParseToken(token)
        c.Set("user_id", claims.UserID)
        c.Next()
    }
}

// internal/middleware/cors.go
func CORS() gin.HandlerFunc {
    return func(c *gin.Context) {
        c.Header("Access-Control-Allow-Origin", "http://localhost:5173")
        // ...
    }
}
```

### Task 7: 路由注册与服务启动

**路由树：** `internal/router/router.go` — 所有 API 端点的注册中心

```go
func Setup(cfg *config.Config) *gin.Engine {
    r := gin.Default()
    r.Use(middleware.CORS())

    api := r.Group("/api")
    api.Group("/auth").POST("/register", ...).POST("/login", ...)
    api.Group("").Use(middleware.AuthRequired(authSvc)).POST("/agent/chat", ...)
    // ...
}
```

**验证：**

```bash
# 启动服务
DB_PASSWORD=choex2025 DB_USER=root go run cmd/server/main.go

# 测试注册
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"test","email":"test@example.com","password":"123456"}'
# 响应: {"token":"eyJ...","user":{"id":1,...}}

# 测试登录
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"123456"}'
```

### Task 8: 搭建 choex-web 仓库

```bash
cd ~/ClaudeProjects
npm create vite@latest choex-web -- --template react-ts
cd choex-web
npm install react-router-dom @reduxjs/toolkit react-redux
gh repo create choex2025-ops/choex-web --public --source=. --remote=origin --push
```

**Vite 代理配置：** `vite.config.ts`

```typescript
export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/api': { target: 'http://localhost:8080', changeOrigin: true },
    },
  },
})
```

### Task 9: Redux Store + Auth Slice

**状态管理：** `src/store/authSlice.ts`

```typescript
export const register = createAsyncThunk('auth/register', async (data, { rejectWithValue }) => {
  try { return await authApi.register(data); }
  catch (e) { return rejectWithValue((e as Error).message); }
});

const authSlice = createSlice({
  name: 'auth',
  initialState: { token: localStorage.getItem('token'), user: null, loading: false },
  reducers: {
    logout(state) { state.token = null; state.user = null; localStorage.removeItem('token'); }
  },
  extraReducers: (builder) => {
    builder.addCase(register.fulfilled, (state, action) => {
      state.token = action.payload.token;
      state.user = action.payload.user;
      localStorage.setItem('token', action.payload.token);
    });
  },
});
```

**API 客户端：** `src/api/client.ts`

```typescript
async function request<T>(path: string, options: RequestInit = {}): Promise<T> {
  const token = localStorage.getItem('token');
  const headers = { 'Content-Type': 'application/json', ...(options.headers as Record<string,string>) };
  if (token) headers['Authorization'] = `Bearer ${token}`;
  const res = await fetch(`/api${path}`, { ...options, headers });
  if (!res.ok) throw new Error((await res.json().catch(()=>({error:'Request failed'}))).error);
  return res.json();
}
```

### Task 10-12: 登录页 + 工作台布局 + 路由守卫

**登录页组件：** `src/pages/LoginPage.tsx`
- 深色卡片式布局，表单含用户名/邮箱/密码
- 注册/登录模式切换
- 登录成功后导航到 `/workspace`

**工作台布局：** `src/pages/WorkspacePage.tsx`
- 左侧 64px 图标栏（Sidebar）
- 中间动态内容区（Welcome 或 App 组件）
- 右侧 320px Agent 对话面板
- 底部全局输入栏

**路由守卫：** `src/components/ProtectedRoute.tsx`
- 检查 localStorage 中的 token
- 未登录重定向到 `/login`

### Task 13: Agent SSE 流式对话后端

**核心文件：** `internal/agent/handler.go`

```go
func (h *AgentHandler) Chat(c *gin.Context) {
    c.Writer.Header().Set("Content-Type", "text/event-stream")
    c.Writer.Header().Set("Cache-Control", "no-cache")
    c.Writer.WriteHeader(http.StatusOK)

    flusher := c.Writer.(http.Flusher)
    userID := c.GetUint64("user_id")

    ch, _ := h.svc.SendMessage(c.Request.Context(), userID, req.Message)
    for res := range ch {
        data, _ := json.Marshal(gin.H{"content": res.Content})
        c.Writer.Write([]byte("data: " + string(data) + "\n\n"))
        flusher.Flush()
    }
    c.Writer.Write([]byte(`data: {"done":true}` + "\n\n"))
    flusher.Flush()
}
```

**LLM 服务：** 复用已有的 `llm/deepseek.go`（DeepSeek API 流式/非流式封装）。

### Task 14: 前端 SSE 聊天集成

**SSE Hook：** `src/hooks/useSSE.ts`

```typescript
export function useSSE() {
  const connect = useCallback((url: string, body: unknown, callbacks: SSECallbacks) => {
    fetch(url, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json', Authorization: `Bearer ${token}` },
      body: JSON.stringify(body),
    }).then(async (response) => {
      const reader = response.body?.getReader();
      const decoder = new TextDecoder();
      let buffer = '';
      while (true) {
        const { done, value } = await reader.read();
        if (done) break;
        buffer += decoder.decode(value, { stream: true });
        const lines = buffer.split('\n');
        buffer = lines.pop() || '';
        for (const line of lines) {
          if (!line.startsWith('data: ')) continue;
          const data = JSON.parse(line.slice(6));
          if (data.content) callbacks.onToken(data.content);
          else if (data.done) callbacks.onDone();
        }
      }
    });
  }, []);
  return { connect };
}
```

**Chat Slice：** `src/store/chatSlice.ts`

```typescript
const chatSlice = createSlice({
  name: 'chat',
  initialState: {
    messages: [{ role: 'assistant', content: '你好！我是你的生活管家。' }],
    streaming: false,
  },
  reducers: {
    addMessage(state, action) { state.messages.push(action.payload); },
    appendToLastMessage(state, action) {
      state.messages[state.messages.length - 1].content += action.payload;
    },
    setStreaming(state, action) { state.streaming = action.payload; },
  },
});
```

**验证全流程：**

```bash
# 终端1: 启动后端
cd ~/ClaudeProjects/choex-server
DB_PASSWORD=choex2025 DB_USER=root DEEPSEEK_API_KEY=sk-xxx go run cmd/server/main.go

# 终端2: 启动前端
cd ~/ClaudeProjects/choex-web
npm run dev
# 浏览器打开 http://localhost:5173
# 注册 → 工作台 → 输入消息 → Agent 流式回复
```

---

## 阶段二：核心应用 (Task 15-23)

### Task 15-17: 三大应用后端

**数据模型（3 张表）：**

| 模型 | 表 | 关键字段 |
|---|---|---|
| Event | events | title, start_time, end_time, color |
| Bill | bills | amount, type (income/expense), category, bill_date |
| Password | passwords | title, url, username, encrypted_password (AES-256) |

**服务层模式（以账单为例）：**

```go
// internal/service/bill.go
type BillStats struct {
    TotalIncome  float64            `json:"total_income"`
    TotalExpense float64            `json:"total_expense"`
    ByCategory   map[string]float64 `json:"by_category"`
}

func (s *BillService) Stats(userID uint64, yearMonth string) (*BillStats, error) {
    var bills []model.Bill
    database.DB.Where("user_id = ? AND bill_date LIKE ?", userID, yearMonth+"%").Find(&bills)
    stats := &BillStats{ByCategory: make(map[string]float64)}
    for _, b := range bills {
        if b.Type == "income" { stats.TotalIncome += b.Amount }
        else { stats.TotalExpense += b.Amount; stats.ByCategory[b.Category] += b.Amount }
    }
    return stats, nil
}
```

### Task 17: AES-256 密码加密

**核心实现：** `internal/service/password.go`

```go
func (s *PasswordService) Encrypt(plaintext string) (string, error) {
    block, _ := aes.NewCipher(s.key)
    aesGCM, _ := cipher.NewGCM(block)
    nonce := make([]byte, aesGCM.NonceSize())
    io.ReadFull(rand.Reader, nonce)
    ciphertext := aesGCM.Seal(nonce, nonce, []byte(plaintext), nil)
    return base64.StdEncoding.EncodeToString(ciphertext), nil
}

func (s *PasswordService) decrypt(encoded string) (string, error) {
    ciphertext, _ := base64.StdEncoding.DecodeString(encoded)
    block, _ := aes.NewCipher(s.key)
    aesGCM, _ := cipher.NewGCM(block)
    nonceSize := aesGCM.NonceSize()
    nonce, ciphertext := ciphertext[:nonceSize], ciphertext[nonceSize:]
    plaintext, _ := aesGCM.Open(nil, nonce, ciphertext, nil)
    return string(plaintext), nil
}
```

**加密密钥：** 从环境变量 `ENCRYPTION_KEY` 读取，不足 32 字节时自动补零到 32 字节。

### Task 18-22: 前端应用 + 浏览器代理

**四个应用组件：**
- `src/apps/calendar/index.tsx` — 日程列表 + 新建表单 + 时间选择
- `src/apps/bills/index.tsx` — 月度统计栏 + 收支表单 + 分类筛选
- `src/apps/passwords/index.tsx` — 列表 + 点击查看密码（调用解密 API）
- `src/apps/browser/index.tsx` — 地址栏 + iframe + 代理加载

**浏览器代理后端：** `internal/handler/proxy.go`

```go
func ProxyHandler(c *gin.Context) {
    targetURL := c.Query("url")
    resp, _ := http.Get(targetURL)
    defer resp.Body.Close()
    for k, v := range resp.Header {
        c.Header(k, v[0])
    }
    io.Copy(c.Writer, resp.Body)
}
```

### Task 23: 工作台集成

```tsx
// WorkspacePage.tsx - React.lazy 懒加载应用
const CalendarApp = React.lazy(() => import('../apps/calendar'));
// ...

const ActiveComponent = APPS.find(a => a.label === activeApp)?.component;
{ActiveComponent ? (
  <React.Suspense fallback={<div>加载中...</div>}>
    <ActiveComponent />
  </React.Suspense>
) : <Welcome />}
```

**路由注册（后端）：** 最终路由树共 30+ 端点

```
/api/auth/register       POST
/api/auth/login          POST
/api/agent/chat          POST  (SSE)
/api/events              GET/POST
/api/events/:id          PUT/DELETE
/api/bills               GET/POST
/api/bills/:id           DELETE
/api/bills/stats         GET
/api/passwords           GET/POST
/api/passwords/:id       GET/PUT/DELETE
/api/memories            GET/POST
/api/memories/:id        DELETE
/api/memories/:id/activate      PUT
/api/memories/:id/versions      GET
/api/memories/:id/versions/:type PUT
/api/memories/:id/restore       PUT
/api/proxy               GET
```

---

## 阶段三：Agent 增强 (Task 24-28)

### Task 24: Agent Tool Calling 引擎

**设计思路：** 在系统提示词中告知 LLM 可用工具及 JSON 调用格式。LLM 返回 `{"tool":"...","args":{...}}` 时，服务端解析并执行对应操作，再将结果送回 LLM 生成自然语言回复。

**核心实现：** `internal/agent/tools.go`

```go
func toolCallSystemPrompt() string {
    return `你是 ChoexManager 个人生活管家。你可以使用以下工具帮助用户：

可用工具：
1. list_events - 列出用户的日程列表
2. create_event - 创建新日程。参数: title, start_time, end_time, description, location
3. list_bills - 列出用户的记账记录
4. create_bill - 创建记账记录。参数: amount, type(expense/income), category, note

当你需要调用工具时，以以下JSON格式回复，不要有任何其他内容：
{"tool":"工具名","args":{...参数...}}

收到工具结果后，用自然语言向用户解释结果。`
}

func processAgentChat(cfg *config.Config, userID uint64, history []llm.Message, userMsg string) (<-chan llm.StreamResult, error) {
    // 第一步：非流式调用 LLM，检测是否需要工具调用
    content, _, _ := llm.CallDeepSeek(toolCallSystemPrompt(), history, false)

    var toolReq struct {
        Tool string         `json:"tool"`
        Args map[string]any `json:"args"`
    }
    if json.Unmarshal([]byte(content), &toolReq) == nil && toolReq.Tool != "" {
        // 执行工具调用
        result := executeTool(toolReq.Tool, userID, ...)

        // 将结果送回 LLM 生成最终回复（流式）
        newHistory := append(history,
            llm.Message{Role: "user", Content: userMsg},
            llm.Message{Role: "user", Content: "工具返回结果: " + result},
        )
        return llm.CallDeepSeek(systemPrompt, newHistory, true)
    }

    // 无需工具调用：直接流式返回
    ch := make(chan llm.StreamResult, 1)
    ch <- llm.StreamResult{Content: content}
    return ch, nil
}
```

**执行工具调用的 executeTool 函数：**
- `list_events` → 查询数据库 events 表 → 返回 JSON
- `create_event` → 解析参数 → GORM 插入 → 返回确认
- `list_bills` → 查询 bills 表 → 返回 JSON
- `create_bill` → 解析参数 → GORM 插入（自动填入当日日期）→ 返回确认

### Task 25: 记忆管理后端

**数据模型：** `internal/model/memory.go`

```go
type AgentMemory struct {
    ID       uint64 `gorm:"primaryKey;autoIncrement"`
    UserID   uint64 `gorm:"index;not null"`
    Name     string `gorm:"size:64;not null"`
    Icon     string `gorm:"size:16"`
    IsActive bool   `gorm:"default:false"`
}

type MemoryVersion struct {
    ID          uint64 `gorm:"primaryKey;autoIncrement"`
    MemoryID    uint64 `gorm:"index;not null"`
    VersionType string `gorm:"size:16;not null"` // current, backup, custom
    Content     string `gorm:"type:text"`
}
```

**核心业务逻辑：** `internal/service/memory.go`

```go
// 激活互斥：同一用户同时只能有 1 条激活记忆
func (s *MemoryService) Activate(id uint64, userID uint64) error {
    return database.DB.Transaction(func(tx *gorm.DB) error {
        tx.Model(&model.AgentMemory{}).Where("user_id = ?", userID).Update("is_active", false)
        return tx.Model(&model.AgentMemory{}).Where("id = ?", id).Update("is_active", true).Error
    })
}

// 保存当前版本时自动备份
func (s *MemoryService) SaveVersion(memoryID uint64, versionType string, content string) error {
    if versionType == "current" {
        var current model.MemoryVersion
        // 将旧的 current 移到 backup
        if err := DB.Where("memory_id = ? AND version_type = ?", memoryID, "current").First(&current); err == nil {
            DB.Where("memory_id = ? AND version_type = ?", memoryID, "backup").Delete(&model.MemoryVersion{})
            DB.Create(&model.MemoryVersion{
                MemoryID: memoryID, VersionType: "backup", Content: current.Content,
            })
        }
    }
    // 写入新版本
    // ...
}
```

### Task 26-28: 前端记忆管理 UI

**记忆列表页：** `src/agent-settings/MemoryList.tsx`
- 竖向卡片列表，每条显示图标/名称/激活状态
- 滑动开关组件（蓝色=激活，灰色=未激活，点击时互斥）
- "使用中" 标签 + "+ 新建记忆" 按钮

**记忆编辑器：** `src/agent-settings/MemoryEditor.tsx`
- 三个版本标签页：当前版本 / 备份版本 / 自定义版本
- 文本编辑器（系统提示词内容）
- 操作栏：从备份恢复、存入自定义版、保存修改、关闭

**工作台集成：**
- 右侧 Agent 面板顶部 🧠 图标 → 点击切换记忆列表
- 记忆列表点击某条 → 打开编辑器
- 左侧应用切换不受影响

---

## 最终项目结构

### choex-server (Go 后端)

```
choex-server/
├── cmd/server/main.go           # 入口：加载配置、初始化DB、启动Gin
├── internal/
│   ├── agent/
│   │   ├── handler.go           # Agent SSE流式对话Handler
│   │   └── tools.go             # Tool Calling引擎 + 工具执行
│   ├── config/config.go         # 环境变量配置
│   ├── database/database.go     # MySQL + Redis连接
│   ├── handler/                 # HTTP处理器
│   │   ├── auth.go              # 注册/登录
│   │   ├── calendar.go          # 日程CRUD
│   │   ├── bill.go              # 账单CRUD + 统计
│   │   ├── password.go          # 密码簿CRUD（AES加密）
│   │   ├── memory.go            # 记忆管理CRUD
│   │   └── proxy.go             # 浏览器代理
│   ├── middleware/
│   │   ├── auth.go              # JWT鉴权中间件
│   │   └── cors.go              # CORS中间件
│   ├── model/                   # GORM数据模型（6张表）
│   ├── router/router.go         # 路由注册中心
│   └── service/                 # 业务逻辑层
├── llm/                         # DeepSeek API客户端（流式/非流式）
├── Makefile
└── .env.example
```

### choex-web (React 前端)

```
choex-web/
├── src/
│   ├── agent-settings/          # Agent 设置页
│   │   ├── MemoryList.tsx       # 记忆列表（滑动开关）
│   │   └── MemoryEditor.tsx     # 记忆编辑器（3版本标签）
│   ├── api/
│   │   ├── client.ts            # fetch封装（自动带JWT）
│   │   └── auth.ts              # 认证API
│   ├── apps/                    # 应用模块（懒加载）
│   │   ├── calendar/            # 日程管理
│   │   ├── bills/               # 记账
│   │   ├── passwords/           # 密码簿
│   │   └── browser/             # 嵌入式浏览器
│   ├── components/Layout/       # 布局组件
│   │   ├── Sidebar.tsx          # 左侧应用图标栏
│   │   ├── AgentPanel.tsx       # 右侧Agent对话面板
│   │   └── InputBar.tsx         # 底部全局输入栏
│   ├── hooks/
│   │   ├── useSSE.ts            # SSE流式接收Hook
│   │   └── useAuth.ts           # 认证状态Hook
│   ├── pages/
│   │   ├── LoginPage.tsx        # 登录/注册页
│   │   └── WorkspacePage.tsx    # 主工作台（路由+状态管理）
│   └── store/                   # Redux状态管理
│       ├── authSlice.ts         # 认证状态
│       └── chatSlice.ts         # 聊天消息状态
```

### choex-manager (设计文档)

```
choex-manager/
├── docs/
│   ├── BUILD_RECORD.md          # 本文档：构建流程记录
│   └── superpowers/
│       ├── specs/                # 设计规格说明书
│       └── plans/                # 三阶段实施计划
└── llm/                         # 共享LLM客户端（已整合到server）
```

---

## 启动方式

```bash
# 1. 确保 MySQL + Redis 运行
brew services list | grep -E "mysql|redis"

# 2. 启动后端
cd ~/ClaudeProjects/choex-server
DEEPSEEK_API_KEY=sk-your-key go run cmd/server/main.go

# 3. 启动前端
cd ~/ClaudeProjects/choex-web
npm run dev

# 4. 打开 http://localhost:5173
```

## 测试 API

```bash
# 注册
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"demo","email":"demo@test.com","password":"123456"}'

# 登录（获取 token）
TOKEN=$(curl -s -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"demo@test.com","password":"123456"}' | jq -r '.token')

# 创建日程
curl -X POST http://localhost:8080/api/events \
  -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" \
  -d '{"title":"测试会议","start_time":"2026-06-12T14:00:00Z","end_time":"2026-06-12T15:00:00Z"}'

# Agent 对话（SSE 流式）
curl -X POST http://localhost:8080/api/agent/chat \
  -H "Content-Type: application/json" -H "Authorization: Bearer $TOKEN" \
  -d '{"message":"帮我记一笔午餐30元"}' --no-buffer
```
