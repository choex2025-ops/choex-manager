# Phase 1: Framework Setup Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the skeleton of ChoexManager — user auth, workspace layout, and basic agent chat — wired end-to-end and deployable locally.

**Architecture:** Go Gin REST API server with JWT auth, MySQL persistence, and Redis caching. React SPA frontend with Redux Toolkit state management, talking to the backend via fetch/SSE.

**Tech Stack:** Go 1.25 + Gin + GORM + JWT | React 18 + Vite + TypeScript + Redux Toolkit + React Router | MySQL 8 + Redis

**Repos:** `github.com/choex2025-ops/choex-server` (Go backend) + `github.com/choex2025-ops/choex-web` (React frontend)

---

## File Structure

### choex-server (new repo)

```
choex-server/
├── cmd/server/main.go              # Entry point, wire up server
├── internal/
│   ├── config/config.go            # Load env vars + config struct
│   ├── database/
│   │   └── database.go             # MySQL + Redis connection init
│   ├── middleware/
│   │   ├── auth.go                 # JWT auth middleware
│   │   └── cors.go                 # CORS middleware
│   ├── model/user.go               # GORM user model
│   ├── handler/auth.go             # Register + Login handlers
│   ├── service/auth.go             # Auth business logic (hash, JWT)
│   ├── router/router.go            # Route registration
│   └── agent/
│       ├── chat.go                 # Agent chat handler (SSE)
│       └── handler.go              # Agent HTTP handler
├── llm/deepseek.go                 # DeepSeek client (copied from existing)
├── llm/deepseek_test.go            # DeepSeek tests (copied)
├── go.mod
├── go.sum
├── Makefile
└── .env.example
```

### choex-web (new repo)

```
choex-web/
├── src/
│   ├── main.tsx                    # Entry point
│   ├── App.tsx                     # Router + global providers
│   ├── pages/
│   │   ├── LoginPage.tsx           # Login/Register page
│   │   └── WorkspacePage.tsx       # Main workspace layout
│   ├── components/
│   │   ├── Layout/
│   │   │   ├── Sidebar.tsx         # Left app icon bar
│   │   │   ├── AgentPanel.tsx      # Right agent chat panel
│   │   │   └── InputBar.tsx        # Bottom input bar
│   │   └── Welcome.tsx             # Default welcome page
│   ├── store/
│   │   ├── index.ts                # Redux store config
│   │   ├── authSlice.ts            # Auth state + actions
│   │   └── chatSlice.ts            # Agent chat state
│   ├── api/
│   │   ├── client.ts               # HTTP client wrapper
│   │   ├── auth.ts                 # Auth API calls
│   │   └── agent.ts                # Agent chat API calls
│   ├── hooks/
│   │   ├── useAuth.ts              # Auth hook
│   │   └── useSSE.ts               # SSE streaming hook
│   └── types/
│       └── index.ts                # Shared types
├── index.html
├── package.json
├── tsconfig.json
├── vite.config.ts
└── .env.example
```

---

## Task 1: Set up choex-server repo structure

**Files:**
- Create: `choex-server/go.mod`, `choex-server/cmd/server/main.go`, `choex-server/internal/config/config.go`, `choex-server/Makefile`, `choex-server/.env.example`

Copy existing `llm/` and `go.mod` from `choex-manager` into `choex-server/`, then extend.

- [ ] **Step 1: Create repo and copy existing LLM code**

```bash
mkdir -p ~/ClaudeProjects/choex-server/internal/{config,database,middleware,model,handler,service,router,agent}
cp -r ~/ClaudeProjects/choex-manager/llm ~/ClaudeProjects/choex-server/
cp ~/ClaudeProjects/choex-manager/go.mod ~/ClaudeProjects/choex-server/
```

Update `go.mod` module path:

```go
module github.com/choex2025-ops/choex-server

go 1.25.4
```

- [ ] **Step 2: Create .env.example**

```
# choex-server/.env.example
SERVER_PORT=8080
DB_HOST=localhost
DB_PORT=3306
DB_USER=root
DB_PASSWORD=yourpassword
DB_NAME=choex_manager
REDIS_HOST=localhost
REDIS_PORT=6379
JWT_SECRET=change-me-to-a-random-string
DEEPSEEK_API_KEY=your-api-key
DEEPSEEK_BASE_URL=https://api.deepseek.com/
DEEPSEEK_MODEL=deepseek-v4-pro
```

- [ ] **Step 3: Create config loader**

```go
// choex-server/internal/config/config.go
package config

import (
	"os"
)

type Config struct {
	ServerPort  string
	DBHost      string
	DBPort      string
	DBUser      string
	DBPassword  string
	DBName      string
	RedisHost   string
	RedisPort   string
	JWTSecret   string
	DeepSeekKey string
}

func Load() *Config {
	return &Config{
		ServerPort:  getEnv("SERVER_PORT", "8080"),
		DBHost:      getEnv("DB_HOST", "localhost"),
		DBPort:      getEnv("DB_PORT", "3306"),
		DBUser:      getEnv("DB_USER", "root"),
		DBPassword:  getEnv("DB_PASSWORD", ""),
		DBName:      getEnv("DB_NAME", "choex_manager"),
		RedisHost:   getEnv("REDIS_HOST", "localhost"),
		RedisPort:   getEnv("REDIS_PORT", "6379"),
		JWTSecret:   getEnv("JWT_SECRET", "dev-secret"),
		DeepSeekKey: os.Getenv("DEEPSEEK_API_KEY"),
	}
}

func (c *Config) DSN() string {
	return c.DBUser + ":" + c.DBPassword + "@tcp(" + c.DBHost + ":" + c.DBPort + ")/" + c.DBName + "?charset=utf8mb4&parseTime=True&loc=Local"
}

func (c *Config) RedisAddr() string {
	return c.RedisHost + ":" + c.RedisPort
}

func getEnv(key, defaultVal string) string {
	if v := os.Getenv(key); v != "" {
		return v
	}
	return defaultVal
}
```

- [ ] **Step 4: Create main.go skeleton**

```go
// choex-server/cmd/server/main.go
package main

import (
	"log"
	"github.com/choex2025-ops/choex-server/internal/config"
)

func main() {
	cfg := config.Load()
	log.Printf("Starting server on :%s", cfg.ServerPort)
	// Database, middleware, routes will be wired in later tasks
}
```

- [ ] **Step 5: Create Makefile**

```makefile
# choex-server/Makefile
.PHONY: run build test

run:
	go run cmd/server/main.go

build:
	go build -o bin/server cmd/server/main.go

test:
	go test ./... -v

lint:
	golangci-lint run ./...
```

- [ ] **Step 6: Add dependencies**

```bash
cd ~/ClaudeProjects/choex-server
go get github.com/gin-gonic/gin
go get gorm.io/gorm
go get gorm.io/driver/mysql
go get github.com/golang-jwt/jwt/v5
go get github.com/redis/go-redis/v9
go get golang.org/x/crypto/bcrypt
go mod tidy
```

- [ ] **Step 7: Init git and push**

```bash
cd ~/ClaudeProjects/choex-server
git init
cp ~/ClaudeProjects/choex-manager/.gitignore .gitignore
git add -A
git commit -m "feat: init choex-server repo with config and skeleton"
gh repo create choex2025-ops/choex-server --public --source=. --remote=origin --push
```

---

## Task 2: Install and configure MySQL + Redis

**Files:**
- None (system-level install)

- [ ] **Step 1: Install MySQL**

```bash
brew install mysql
```

Start MySQL and set root password:

```bash
brew services start mysql
mysql_secure_installation
```

Follow prompts: set root password, remove anonymous users, disallow remote root login, remove test database, reload privileges.

- [ ] **Step 2: Create database**

```bash
mysql -u root -p -e "CREATE DATABASE IF NOT EXISTS choex_manager CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

- [ ] **Step 3: Install Redis**

```bash
brew install redis
brew services start redis
```

- [ ] **Step 4: Verify both running**

```bash
redis-cli PING
# Expected: PONG

mysql -u root -p -e "SELECT 1;"
# Expected: 1
```

---

## Task 3: Database connection and User model

**Files:**
- Create: `choex-server/internal/database/database.go`
- Create: `choex-server/internal/model/user.go`

- [ ] **Step 1: Create database connection**

```go
// choex-server/internal/database/database.go
package database

import (
	"context"
	"log"

	"gorm.io/driver/mysql"
	"gorm.io/gorm"
	"gorm.io/gorm/logger"

	"github.com/choex2025-ops/choex-server/internal/config"
	"github.com/redis/go-redis/v9"
)

var (
	DB    *gorm.DB
	Redis *redis.Client
)

func Init(cfg *config.Config) {
	var err error
	DB, err = gorm.Open(mysql.Open(cfg.DSN()), &gorm.Config{
		Logger: logger.Default.LogMode(logger.Info),
	})
	if err != nil {
		log.Fatalf("Failed to connect to MySQL: %v", err)
	}

	Redis = redis.NewClient(&redis.Options{
		Addr: cfg.RedisAddr(),
	})
	if err := Redis.Ping(context.Background()).Err(); err != nil {
		log.Fatalf("Failed to connect to Redis: %v", err)
	}

	log.Println("Database and Redis connected")
}
```

- [ ] **Step 2: Create User model**

```go
// choex-server/internal/model/user.go
package model

import "time"

type User struct {
	ID           uint64    `gorm:"primaryKey;autoIncrement" json:"id"`
	Username     string    `gorm:"size:64;not null" json:"username"`
	Email        string    `gorm:"size:128;uniqueIndex;not null" json:"email"`
	PasswordHash string    `gorm:"size:256;not null" json:"-"`
	CreatedAt    time.Time `json:"created_at"`
	UpdatedAt    time.Time `json:"updated_at"`
}

func (User) TableName() string {
	return "users"
}
```

- [ ] **Step 3: Auto-migrate on startup**

Update `cmd/server/main.go` to initialize DB and run migration:

```go
// choex-server/cmd/server/main.go
package main

import (
	"log"

	"github.com/choex2025-ops/choex-server/internal/config"
	"github.com/choex2025-ops/choex-server/internal/database"
	"github.com/choex2025-ops/choex-server/internal/model"
)

func main() {
	cfg := config.Load()

	database.Init(cfg)
	if err := database.DB.AutoMigrate(&model.User{}); err != nil {
		log.Fatalf("Migration failed: %v", err)
	}

	log.Printf("Server ready on :%s", cfg.ServerPort)
}
```

- [ ] **Step 4: Verify migration**

```bash
cd ~/ClaudeProjects/choex-server
go run cmd/server/main.go
# Expected: logs showing DB connected and migration ran

mysql -u root -p choex_manager -e "DESCRIBE users;"
# Expected: shows users table columns
```

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add MySQL/Redis connection and User model with auto-migration"
git push
```

---

## Task 4: Auth service (JWT + bcrypt)

**Files:**
- Create: `choex-server/internal/service/auth.go`

- [ ] **Step 1: Create auth service**

```go
// choex-server/internal/service/auth.go
package service

import (
	"errors"
	"time"

	"github.com/golang-jwt/jwt/v5"
	"golang.org/x/crypto/bcrypt"

	"github.com/choex2025-ops/choex-server/internal/config"
)

var (
	ErrInvalidCredentials = errors.New("invalid credentials")
	ErrUserExists         = errors.New("user already exists")
)

type AuthService struct {
	cfg *config.Config
}

func NewAuthService(cfg *config.Config) *AuthService {
	return &AuthService{cfg: cfg}
}

type Claims struct {
	UserID uint64 `json:"user_id"`
	Email  string `json:"email"`
	jwt.RegisteredClaims
}

func (s *AuthService) HashPassword(password string) (string, error) {
	bytes, err := bcrypt.GenerateFromPassword([]byte(password), bcrypt.DefaultCost)
	return string(bytes), err
}

func (s *AuthService) CheckPassword(password, hash string) bool {
	return bcrypt.CompareHashAndPassword([]byte(hash), []byte(password)) == nil
}

func (s *AuthService) GenerateToken(userID uint64, email string) (string, error) {
	claims := &Claims{
		UserID: userID,
		Email:  email,
		RegisteredClaims: jwt.RegisteredClaims{
			ExpiresAt: jwt.NewNumericDate(time.Now().Add(24 * time.Hour)),
			IssuedAt:  jwt.NewNumericDate(time.Now()),
		},
	}
	token := jwt.NewWithClaims(jwt.SigningMethodHS256, claims)
	return token.SignedString([]byte(s.cfg.JWTSecret))
}

func (s *AuthService) ParseToken(tokenString string) (*Claims, error) {
	token, err := jwt.ParseWithClaims(tokenString, &Claims{},
		func(token *jwt.Token) (any, error) {
			return []byte(s.cfg.JWTSecret), nil
		},
	)
	if err != nil || !token.Valid {
		return nil, ErrInvalidCredentials
	}
	claims, ok := token.Claims.(*Claims)
	if !ok {
		return nil, ErrInvalidCredentials
	}
	return claims, nil
}
```

- [ ] **Step 2: Commit**

```bash
git add -A
git commit -m "feat: add JWT and bcrypt auth service"
git push
```

---

## Task 5: Auth handler (Register + Login)

**Files:**
- Create: `choex-server/internal/handler/auth.go`

- [ ] **Step 1: Create auth handler**

```go
// choex-server/internal/handler/auth.go
package handler

import (
	"net/http"

	"github.com/gin-gonic/gin"

	"github.com/choex2025-ops/choex-server/internal/database"
	"github.com/choex2025-ops/choex-server/internal/model"
	"github.com/choex2025-ops/choex-server/internal/service"
)

type AuthHandler struct {
	svc *service.AuthService
}

func NewAuthHandler(svc *service.AuthService) *AuthHandler {
	return &AuthHandler{svc: svc}
}

type registerRequest struct {
	Username string `json:"username" binding:"required,min=2,max=64"`
	Email    string `json:"email" binding:"required,email"`
	Password string `json:"password" binding:"required,min=6"`
}

type loginRequest struct {
	Email    string `json:"email" binding:"required,email"`
	Password string `json:"password" binding:"required"`
}

type authResponse struct {
	Token string `json:"token"`
	User  struct {
		ID       uint64 `json:"id"`
		Username string `json:"username"`
		Email    string `json:"email"`
	} `json:"user"`
}

func (h *AuthHandler) Register(c *gin.Context) {
	var req registerRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	// Check if user exists
	var existing model.User
	if err := database.DB.Where("email = ?", req.Email).First(&existing).Error; err == nil {
		c.JSON(http.StatusConflict, gin.H{"error": "email already registered"})
		return
	}

	hash, err := h.svc.HashPassword(req.Password)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to hash password"})
		return
	}

	user := model.User{
		Username:     req.Username,
		Email:        req.Email,
		PasswordHash: hash,
	}
	if err := database.DB.Create(&user).Error; err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to create user"})
		return
	}

	token, err := h.svc.GenerateToken(user.ID, user.Email)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to generate token"})
		return
	}

	var resp authResponse
	resp.Token = token
	resp.User.ID = user.ID
	resp.User.Username = user.Username
	resp.User.Email = user.Email
	c.JSON(http.StatusCreated, resp)
}

func (h *AuthHandler) Login(c *gin.Context) {
	var req loginRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	var user model.User
	if err := database.DB.Where("email = ?", req.Email).First(&user).Error; err != nil {
		c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid email or password"})
		return
	}

	if !h.svc.CheckPassword(req.Password, user.PasswordHash) {
		c.JSON(http.StatusUnauthorized, gin.H{"error": "invalid email or password"})
		return
	}

	token, err := h.svc.GenerateToken(user.ID, user.Email)
	if err != nil {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "failed to generate token"})
		return
	}

	var resp authResponse
	resp.Token = token
	resp.User.ID = user.ID
	resp.User.Username = user.Username
	resp.User.Email = user.Email
	c.JSON(http.StatusOK, resp)
}
```

- [ ] **Step 2: Commit**

```bash
git add -A
git commit -m "feat: add register and login handlers"
git push
```

---

## Task 6: JWT middleware + CORS middleware

**Files:**
- Create: `choex-server/internal/middleware/auth.go`
- Create: `choex-server/internal/middleware/cors.go`

- [ ] **Step 1: Create JWT middleware**

```go
// choex-server/internal/middleware/auth.go
package middleware

import (
	"net/http"
	"strings"

	"github.com/gin-gonic/gin"

	"github.com/choex2025-ops/choex-server/internal/service"
)

func AuthRequired(svc *service.AuthService) gin.HandlerFunc {
	return func(c *gin.Context) {
		header := c.GetHeader("Authorization")
		if header == "" || !strings.HasPrefix(header, "Bearer ") {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "missing authorization header"})
			return
		}

		token := strings.TrimPrefix(header, "Bearer ")
		claims, err := svc.ParseToken(token)
		if err != nil {
			c.AbortWithStatusJSON(http.StatusUnauthorized, gin.H{"error": "invalid token"})
			return
		}

		c.Set("user_id", claims.UserID)
		c.Set("email", claims.Email)
		c.Next()
	}
}
```

- [ ] **Step 2: Create CORS middleware**

```go
// choex-server/internal/middleware/cors.go
package middleware

import (
	"net/http"

	"github.com/gin-gonic/gin"
)

func CORS() gin.HandlerFunc {
	return func(c *gin.Context) {
		c.Header("Access-Control-Allow-Origin", "http://localhost:5173")
		c.Header("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, OPTIONS")
		c.Header("Access-Control-Allow-Headers", "Content-Type, Authorization")
		c.Header("Access-Control-Allow-Credentials", "true")

		if c.Request.Method == http.MethodOptions {
			c.AbortWithStatus(http.StatusNoContent)
			return
		}
		c.Next()
	}
}
```

- [ ] **Step 3: Commit**

```bash
git add -A
git commit -m "feat: add JWT auth middleware and CORS middleware"
git push
```

---

## Task 7: Router setup and server startup

**Files:**
- Create: `choex-server/internal/router/router.go`
- Modify: `choex-server/cmd/server/main.go`

- [ ] **Step 1: Create router**

```go
// choex-server/internal/router/router.go
package router

import (
	"github.com/gin-gonic/gin"

	"github.com/choex2025-ops/choex-server/internal/config"
	"github.com/choex2025-ops/choex-server/internal/handler"
	"github.com/choex2025-ops/choex-server/internal/middleware"
	"github.com/choex2025-ops/choex-server/internal/service"
)

func Setup(cfg *config.Config) *gin.Engine {
	r := gin.Default()
	r.Use(middleware.CORS())

	authSvc := service.NewAuthService(cfg)
	authHandler := handler.NewAuthHandler(authSvc)

	api := r.Group("/api")
	{
		auth := api.Group("/auth")
		{
			auth.POST("/register", authHandler.Register)
			auth.POST("/login", authHandler.Login)
		}
	}

	return r
}
```

- [ ] **Step 2: Update main.go to start server**

```go
// choex-server/cmd/server/main.go
package main

import (
	"log"

	"github.com/choex2025-ops/choex-server/internal/config"
	"github.com/choex2025-ops/choex-server/internal/database"
	"github.com/choex2025-ops/choex-server/internal/model"
	"github.com/choex2025-ops/choex-server/internal/router"
)

func main() {
	cfg := config.Load()

	database.Init(cfg)
	if err := database.DB.AutoMigrate(&model.User{}); err != nil {
		log.Fatalf("Migration failed: %v", err)
	}

	r := router.Setup(cfg)
	log.Printf("Server running on :%s", cfg.ServerPort)
	if err := r.Run(":" + cfg.ServerPort); err != nil {
		log.Fatalf("Failed to start server: %v", err)
	}
}
```

- [ ] **Step 3: Test server starts**

```bash
cd ~/ClaudeProjects/choex-server
go run cmd/server/main.go
# Expected: "Server running on :8080"
```

- [ ] **Step 4: Test register with curl**

```bash
curl -X POST http://localhost:8080/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"username":"test","email":"test@example.com","password":"123456"}'
# Expected: {"token":"...","user":{"id":1,"username":"test","email":"test@example.com"}}
```

- [ ] **Step 5: Test login with curl**

```bash
curl -X POST http://localhost:8080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"123456"}'
# Expected: {"token":"...","user":{"id":1,"username":"test","email":"test@example.com"}}
```

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: wire up router, middleware, and server startup with auth endpoints"
git push
```

---

## Task 8: Set up choex-web repo (React + Vite)

**Files:**
- Create: React project via Vite scaffolding

- [ ] **Step 1: Scaffold React project**

```bash
cd ~/ClaudeProjects
npm create vite@latest choex-web -- --template react-ts
cd choex-web
npm install
```

- [ ] **Step 2: Install dependencies**

```bash
npm install react-router-dom @reduxjs/toolkit react-redux
npm install -D @types/react @types/react-dom
```

- [ ] **Step 3: Create .env.example**

```
# choex-web/.env.example
VITE_API_BASE=http://localhost:8080/api
```

- [ ] **Step 4: Replace vite.config.ts with proxy setup**

```typescript
// choex-web/vite.config.ts
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: {
      '/api': {
        target: 'http://localhost:8080',
        changeOrigin: true,
      },
    },
  },
})
```

- [ ] **Step 5: Init git and push**

```bash
cd ~/ClaudeProjects/choex-web
git init
cp ~/ClaudeProjects/choex-server/.gitignore .gitignore
git add -A
git commit -m "feat: scaffold React+Vite+TS project"
gh repo create choex2025-ops/choex-web --public --source=. --remote=origin --push
```

---

## Task 9: Redux store + auth slice

**Files:**
- Create: `choex-web/src/store/index.ts`
- Create: `choex-web/src/store/authSlice.ts`
- Create: `choex-web/src/types/index.ts`
- Create: `choex-web/src/api/client.ts`
- Create: `choex-web/src/api/auth.ts`

- [ ] **Step 1: Create shared types**

```typescript
// choex-web/src/types/index.ts
export interface User {
  id: number;
  username: string;
  email: string;
}

export interface AuthState {
  token: string | null;
  user: User | null;
  loading: boolean;
}

export interface AuthResponse {
  token: string;
  user: User;
}

export interface RegisterRequest {
  username: string;
  email: string;
  password: string;
}

export interface LoginRequest {
  email: string;
  password: string;
}
```

- [ ] **Step 2: Create API client**

```typescript
// choex-web/src/api/client.ts
const BASE_URL = '/api';

async function request<T>(path: string, options: RequestInit = {}): Promise<T> {
  const token = localStorage.getItem('token');
  const headers: Record<string, string> = {
    'Content-Type': 'application/json',
    ...(options.headers as Record<string, string>),
  };
  if (token) {
    headers['Authorization'] = `Bearer ${token}`;
  }

  const res = await fetch(`${BASE_URL}${path}`, { ...options, headers });
  if (!res.ok) {
    const error = await res.json().catch(() => ({ error: 'Request failed' }));
    throw new Error(error.error || `HTTP ${res.status}`);
  }
  return res.json();
}

export const api = {
  get: <T>(path: string) => request<T>(path),
  post: <T>(path: string, body: unknown) =>
    request<T>(path, { method: 'POST', body: JSON.stringify(body) }),
  put: <T>(path: string, body: unknown) =>
    request<T>(path, { method: 'PUT', body: JSON.stringify(body) }),
  del: <T>(path: string) => request<T>(path, { method: 'DELETE' }),
};
```

- [ ] **Step 3: Create auth API**

```typescript
// choex-web/src/api/auth.ts
import { api } from './client';
import type { AuthResponse, RegisterRequest, LoginRequest } from '../types';

export const authApi = {
  register: (data: RegisterRequest) => api.post<AuthResponse>('/auth/register', data),
  login: (data: LoginRequest) => api.post<AuthResponse>('/auth/login', data),
};
```

- [ ] **Step 4: Create auth slice**

```typescript
// choex-web/src/store/authSlice.ts
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';
import { authApi } from '../api/auth';
import type { AuthState, User, RegisterRequest, LoginRequest } from '../types';

const initialState: AuthState = {
  token: localStorage.getItem('token'),
  user: null,
  loading: false,
};

export const register = createAsyncThunk(
  'auth/register',
  async (data: RegisterRequest, { rejectWithValue }) => {
    try {
      return await authApi.register(data);
    } catch (e) {
      return rejectWithValue((e as Error).message);
    }
  }
);

export const login = createAsyncThunk(
  'auth/login',
  async (data: LoginRequest, { rejectWithValue }) => {
    try {
      return await authApi.login(data);
    } catch (e) {
      return rejectWithValue((e as Error).message);
    }
  }
);

const authSlice = createSlice({
  name: 'auth',
  initialState,
  reducers: {
    logout(state) {
      state.token = null;
      state.user = null;
      localStorage.removeItem('token');
    },
  },
  extraReducers: (builder) => {
    const handleAuthFulfilled = (state: AuthState, action: { payload: { token: string; user: User } }) => {
      state.token = action.payload.token;
      state.user = action.payload.user;
      state.loading = false;
      localStorage.setItem('token', action.payload.token);
    };
    const handleAuthPending = (state: AuthState) => {
      state.loading = true;
    };
    const handleAuthRejected = (state: AuthState) => {
      state.loading = false;
    };

    builder
      .addCase(register.fulfilled, handleAuthFulfilled)
      .addCase(register.pending, handleAuthPending)
      .addCase(register.rejected, handleAuthRejected)
      .addCase(login.fulfilled, handleAuthFulfilled)
      .addCase(login.pending, handleAuthPending)
      .addCase(login.rejected, handleAuthRejected);
  },
});

export const { logout } = authSlice.actions;
export default authSlice.reducer;
```

- [ ] **Step 5: Create Redux store**

```typescript
// choex-web/src/store/index.ts
import { configureStore } from '@reduxjs/toolkit';
import authReducer from './authSlice';

export const store = configureStore({
  reducer: {
    auth: authReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

- [ ] **Step 6: Commit**

```bash
git add -A
git commit -m "feat: add Redux store, auth slice, API client"
git push
```

---

## Task 10: Login page

**Files:**
- Create: `choex-web/src/pages/LoginPage.tsx`
- Modify: `choex-web/src/App.tsx`
- Modify: `choex-web/src/main.tsx`

- [ ] **Step 1: Create LoginPage component**

```tsx
// choex-web/src/pages/LoginPage.tsx
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { useAppDispatch } from '../hooks/useAppDispatch';
import { register as registerAction, login as loginAction } from '../store/authSlice';

export default function LoginPage() {
  const [isRegister, setIsRegister] = useState(false);
  const [username, setUsername] = useState('');
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const dispatch = useAppDispatch();
  const navigate = useNavigate();

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError('');

    try {
      if (isRegister) {
        await dispatch(registerAction({ username, email, password })).unwrap();
      } else {
        await dispatch(loginAction({ email, password })).unwrap();
      }
      navigate('/workspace');
    } catch (err) {
      setError((err as string) || 'Operation failed');
    }
  };

  return (
    <div style={styles.container}>
      <div style={styles.card}>
        <div style={styles.header}>
          <div style={styles.logo}>☀</div>
          <h1 style={styles.title}>ChoexManager</h1>
          <p style={styles.subtitle}>个人生活管家</p>
        </div>

        <form onSubmit={handleSubmit} style={styles.form}>
          {isRegister && (
            <input
              style={styles.input}
              type="text"
              placeholder="用户名"
              value={username}
              onChange={(e) => setUsername(e.target.value)}
              required
            />
          )}
          <input
            style={styles.input}
            type="email"
            placeholder="邮箱"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
            required
          />
          <input
            style={styles.input}
            type="password"
            placeholder="密码"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
            required
          />
          {error && <p style={styles.error}>{error}</p>}
          <button style={styles.button} type="submit">
            {isRegister ? '注 册' : '登 录'}
          </button>
        </form>

        <p style={styles.switch}>
          {isRegister ? '已有账号？' : '还没有账号？'}
          <span style={styles.link} onClick={() => setIsRegister(!isRegister)}>
            {isRegister ? '去登录' : '立即注册'}
          </span>
        </p>
      </div>
    </div>
  );
}

const styles: Record<string, React.CSSProperties> = {
  container: {
    display: 'flex',
    alignItems: 'center',
    justifyContent: 'center',
    minHeight: '100vh',
    background: '#0d1117',
  },
  card: {
    background: '#161b22',
    border: '1px solid #30363d',
    borderRadius: 12,
    padding: '40px 48px',
    width: 380,
  },
  header: {
    textAlign: 'center',
    marginBottom: 32,
  },
  logo: {
    width: 48,
    height: 48,
    background: '#1f6feb',
    borderRadius: 12,
    margin: '0 auto 16px',
    display: 'flex',
    alignItems: 'center',
    justifyContent: 'center',
    fontSize: 24,
  },
  title: {
    fontSize: 18,
    fontWeight: 600,
    color: '#e6edf3',
    margin: 0,
  },
  subtitle: {
    fontSize: 12,
    color: '#8b949e',
    marginTop: 4,
  },
  form: {
    display: 'flex',
    flexDirection: 'column',
    gap: 12,
  },
  input: {
    background: '#0d1117',
    border: '1px solid #30363d',
    borderRadius: 6,
    padding: '10px 12px',
    color: '#e6edf3',
    fontSize: 14,
    outline: 'none',
    width: '100%',
    boxSizing: 'border-box' as const,
  },
  button: {
    background: '#1f6feb',
    color: 'white',
    border: 'none',
    borderRadius: 6,
    padding: '10px 0',
    fontSize: 14,
    fontWeight: 600,
    cursor: 'pointer',
    width: '100%',
    marginTop: 4,
  },
  error: {
    color: '#f85149',
    fontSize: 12,
    margin: 0,
  },
  switch: {
    textAlign: 'center',
    fontSize: 12,
    color: '#8b949e',
    marginTop: 16,
  },
  link: {
    color: '#58a6ff',
    cursor: 'pointer',
    marginLeft: 4,
  },
};
```

- [ ] **Step 2: Create typed hooks**

```typescript
// choex-web/src/hooks/useAppDispatch.ts
import { useDispatch, useSelector } from 'react-redux';
import type { RootState, AppDispatch } from '../store';

export const useAppDispatch = useDispatch.withTypes<AppDispatch>();
export const useAppSelector = useSelector.withTypes<RootState>();
```

- [ ] **Step 3: Update main.tsx with Redux Provider**

```tsx
// choex-web/src/main.tsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { Provider } from 'react-redux';
import { BrowserRouter } from 'react-router-dom';
import { store } from './store';
import App from './App';
import './index.css';

ReactDOM.createRoot(document.getElementById('root')!).render(
  <React.StrictMode>
    <Provider store={store}>
      <BrowserRouter>
        <App />
      </BrowserRouter>
    </Provider>
  </React.StrictMode>
);
```

- [ ] **Step 4: Update App.tsx with routing**

```tsx
// choex-web/src/App.tsx
import { Routes, Route, Navigate } from 'react-router-dom';
import LoginPage from './pages/LoginPage';

export default function App() {
  return (
    <Routes>
      <Route path="/login" element={<LoginPage />} />
      <Route path="*" element={<Navigate to="/login" replace />} />
    </Routes>
  );
}
```

- [ ] **Step 5: Add global CSS reset**

```css
/* choex-web/src/index.css */
*, *::before, *::after {
  box-sizing: border-box;
  margin: 0;
  padding: 0;
}
body {
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
  background: #0d1117;
  color: #e6edf3;
  -webkit-font-smoothing: antialiased;
}
input {
  font-family: inherit;
}
```

- [ ] **Step 6: Verify login page renders**

```bash
cd ~/ClaudeProjects/choex-web
npm run dev
# Open http://localhost:5173
# Should show login page with register/login toggle
```

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: add login page with register flow and Redux auth"
git push
```

---

## Task 11: Workspace layout (shell)

**Files:**
- Create: `choex-web/src/pages/WorkspacePage.tsx`
- Create: `choex-web/src/components/Layout/Sidebar.tsx`
- Create: `choex-web/src/components/Layout/AgentPanel.tsx`
- Create: `choex-web/src/components/Layout/InputBar.tsx`
- Create: `choex-web/src/components/Welcome.tsx`
- Modify: `choex-web/src/App.tsx`

- [ ] **Step 1: Create Welcome component**

```tsx
// choex-web/src/components/Welcome.tsx
export default function Welcome() {
  return (
    <div style={{
      flex: 1,
      display: 'flex',
      flexDirection: 'column',
      alignItems: 'center',
      justifyContent: 'center',
      background: '#0d1117',
    }}>
      <div style={styles.icon}>☀</div>
      <h1 style={styles.title}>ChoexManager</h1>
      <p style={styles.subtitle}>个人生活管家</p>
      <p style={styles.hint}>点击左侧应用开始使用</p>
    </div>
  );
}

const styles: Record<string, React.CSSProperties> = {
  icon: {
    width: 80,
    height: 80,
    background: 'linear-gradient(135deg, #1f6feb, #58a6ff)',
    borderRadius: 20,
    display: 'flex',
    alignItems: 'center',
    justifyContent: 'center',
    fontSize: 36,
    marginBottom: 24,
  },
  title: {
    fontSize: 28,
    fontWeight: 700,
    color: '#e6edf3',
    marginBottom: 8,
  },
  subtitle: {
    fontSize: 14,
    color: '#8b949e',
  },
  hint: {
    marginTop: 32,
    fontSize: 12,
    color: '#484f58',
  },
};
```

- [ ] **Step 2: Create Sidebar**

```tsx
// choex-web/src/components/Layout/Sidebar.tsx
import type { ReactNode } from 'react';

interface AppItem {
  icon: string;
  label: string;
  active: boolean;
  onClick: () => void;
}

interface SidebarProps {
  apps: AppItem[];
}

export default function Sidebar({ apps }: SidebarProps) {
  return (
    <div style={styles.sidebar}>
      {apps.map((app, i) => (
        <div
          key={i}
          onClick={app.onClick}
          title={app.label}
          style={{
            ...styles.item,
            ...(app.active ? styles.itemActive : {}),
          }}
        >
          {app.icon}
        </div>
      ))}
      <div style={styles.spacer} />
      <div style={styles.item} title="设置">⚙</div>
    </div>
  );
}

const styles: Record<string, React.CSSProperties> = {
  sidebar: {
    width: 64,
    background: '#161b22',
    borderRight: '1px solid #30363d',
    display: 'flex',
    flexDirection: 'column',
    alignItems: 'center',
    paddingTop: 16,
    gap: 12,
    height: '100%',
  },
  item: {
    width: 44,
    height: 44,
    background: '#21262d',
    borderRadius: 10,
    display: 'flex',
    alignItems: 'center',
    justifyContent: 'center',
    fontSize: 18,
    color: '#8b949e',
    cursor: 'pointer',
    userSelect: 'none',
  },
  itemActive: {
    background: '#1f6feb',
    color: 'white',
    boxShadow: '0 0 12px rgba(31,111,235,0.27)',
  },
  spacer: {
    flex: 1,
  },
};
```

- [ ] **Step 3: Create AgentPanel (placeholder)**

```tsx
// choex-web/src/components/Layout/AgentPanel.tsx
export default function AgentPanel() {
  return (
    <div style={{
      width: 320,
      background: '#161b22',
      borderLeft: '1px solid #30363d',
      display: 'flex',
      flexDirection: 'column',
    }}>
      <div style={{
        padding: '10px 14px',
        borderBottom: '1px solid #30363d',
        display: 'flex',
        alignItems: 'center',
        gap: 8,
      }}>
        <span style={{ color: '#e6edf3', fontWeight: 600, fontSize: 12 }}>Agent</span>
        <div style={{ marginLeft: 'auto', display: 'flex', gap: 4 }}>
          <span style={styles.iconBtn} title="记忆管理">🧠</span>
          <span style={styles.iconBtn} title="更多设置">⚙</span>
        </div>
      </div>
      <div style={{ flex: 1, padding: 16 }}>
        <div style={styles.message}>
          <div style={styles.avatar}>🤖</div>
          <div style={styles.bubble}>
            你好！我是你的生活管家。需要什么帮助吗？
          </div>
        </div>
      </div>
    </div>
  );
}

const styles: Record<string, React.CSSProperties> = {
  iconBtn: {
    width: 28,
    height: 28,
    background: '#21262d',
    borderRadius: 6,
    textAlign: 'center',
    lineHeight: '28px',
    fontSize: 13,
    color: '#8b949e',
    cursor: 'pointer',
  },
  message: {
    display: 'flex',
    gap: 8,
    alignItems: 'flex-start',
  },
  avatar: {
    width: 28,
    height: 28,
    background: '#1f6feb',
    borderRadius: 6,
    flexShrink: 0,
    textAlign: 'center',
    lineHeight: '28px',
    fontSize: 12,
  },
  bubble: {
    background: '#21262d',
    borderRadius: 8,
    padding: '8px 12px',
    fontSize: 12,
    color: '#e6edf3',
  },
};
```

- [ ] **Step 4: Create InputBar**

```tsx
// choex-web/src/components/Layout/InputBar.tsx
import { useState } from 'react';

export default function InputBar() {
  const [text, setText] = useState('');

  const handleSend = () => {
    if (!text.trim()) return;
    // Connected to agent in a later task
    setText('');
  };

  return (
    <div style={{
      padding: '12px 20px',
      background: '#161b22',
      borderTop: '1px solid #30363d',
      display: 'flex',
      gap: 8,
      alignItems: 'center',
    }}>
      <input
        style={{
          flex: 1,
          background: '#0d1117',
          border: '1px solid #30363d',
          borderRadius: 6,
          padding: '8px 12px',
          color: '#e6edf3',
          fontSize: 13,
          outline: 'none',
        }}
        placeholder="输入指令，如'帮我记一笔午餐30元'..."
        value={text}
        onChange={(e) => setText(e.target.value)}
        onKeyDown={(e) => e.key === 'Enter' && handleSend()}
      />
      <button
        style={{
          background: '#1f6feb',
          color: 'white',
          border: 'none',
          padding: '8px 20px',
          borderRadius: 6,
          fontSize: 13,
          cursor: 'pointer',
        }}
        onClick={handleSend}
      >
        发送
      </button>
    </div>
  );
}
```

- [ ] **Step 5: Create WorkspacePage**

```tsx
// choex-web/src/pages/WorkspacePage.tsx
import { useState } from 'react';
import Sidebar from '../components/Layout/Sidebar';
import AgentPanel from '../components/Layout/AgentPanel';
import InputBar from '../components/Layout/InputBar';
import Welcome from '../components/Welcome';

const APPS = [
  { icon: '📅', label: '日程' },
  { icon: '💳', label: '记账' },
  { icon: '🔐', label: '密码簿' },
  { icon: '🌐', label: '浏览器' },
];

export default function WorkspacePage() {
  const [activeApp, setActiveApp] = useState<string | null>(null);

  const appItems = APPS.map((app) => ({
    ...app,
    active: activeApp === app.label,
    onClick: () => setActiveApp(activeApp === app.label ? null : app.label),
  }));

  return (
    <div style={{ display: 'flex', flexDirection: 'column', height: '100vh' }}>
      <div style={{ display: 'flex', flex: 1, overflow: 'hidden' }}>
        <Sidebar apps={appItems} />
        <Welcome />
        <AgentPanel />
      </div>
      <InputBar />
    </div>
  );
}
```

- [ ] **Step 6: Update App.tsx with workspace route**

```tsx
// choex-web/src/App.tsx
import { Routes, Route, Navigate } from 'react-router-dom';
import LoginPage from './pages/LoginPage';
import WorkspacePage from './pages/WorkspacePage';

export default function App() {
  return (
    <Routes>
      <Route path="/login" element={<LoginPage />} />
      <Route path="/workspace" element={<WorkspacePage />} />
      <Route path="*" element={<Navigate to="/login" replace />} />
    </Routes>
  );
}
```

- [ ] **Step 7: Verify workspace renders**

```bash
cd ~/ClaudeProjects/choex-web
npm run dev
# Login → navigate to /workspace
# Expected: Three-panel layout with welcome page, sidebar, agent panel, input bar
```

- [ ] **Step 8: Commit**

```bash
git add -A
git commit -m "feat: add workspace page with three-panel layout"
git push
```

---

## Task 12: Auth guard (redirect unauthenticated users)

**Files:**
- Create: `choex-web/src/hooks/useAuth.ts`
- Create: `choex-web/src/components/ProtectedRoute.tsx`
- Modify: `choex-web/src/App.tsx`

- [ ] **Step 1: Create useAuth hook**

```typescript
// choex-web/src/hooks/useAuth.ts
export function useAuth() {
  const token = localStorage.getItem('token');
  return { isAuthenticated: !!token, token };
}
```

- [ ] **Step 2: Create ProtectedRoute**

```tsx
// choex-web/src/components/ProtectedRoute.tsx
import { Navigate } from 'react-router-dom';
import { useAuth } from '../hooks/useAuth';

export default function ProtectedRoute({ children }: { children: React.ReactNode }) {
  const { isAuthenticated } = useAuth();
  if (!isAuthenticated) return <Navigate to="/login" replace />;
  return <>{children}</>;
}
```

- [ ] **Step 3: Update App.tsx with guard**

```tsx
// choex-web/src/App.tsx
import { Routes, Route, Navigate } from 'react-router-dom';
import LoginPage from './pages/LoginPage';
import WorkspacePage from './pages/WorkspacePage';
import ProtectedRoute from './components/ProtectedRoute';

export default function App() {
  return (
    <Routes>
      <Route path="/login" element={<LoginPage />} />
      <Route path="/workspace" element={
        <ProtectedRoute>
          <WorkspacePage />
        </ProtectedRoute>
      } />
      <Route path="*" element={<Navigate to="/login" replace />} />
    </Routes>
  );
}
```

- [ ] **Step 4: Commit**

```bash
git add -A
git commit -m "feat: add auth guard for protected routes"
git push
```

---

## Task 13: Agent chat with SSE streaming

**Files:**
- Create: `choex-server/internal/agent/handler.go`
- Create: `choex-server/internal/agent/chat.go`
- Modify: `choex-server/internal/router/router.go`

- [ ] **Step 1: Create agent SSE chat handler**

```go
// choex-server/internal/agent/handler.go
package agent

import (
	"encoding/json"
	"net/http"

	"github.com/gin-gonic/gin"

	"github.com/choex2025-ops/choex-server/internal/config"
)

type AgentHandler struct {
	svc *ChatService
}

func NewAgentHandler(cfg *config.Config) *AgentHandler {
	return &AgentHandler{svc: NewChatService(cfg)}
}

type chatRequest struct {
	Message string `json:"message" binding:"required"`
}

func (h *AgentHandler) Chat(c *gin.Context) {
	var req chatRequest
	if err := c.ShouldBindJSON(&req); err != nil {
		c.JSON(http.StatusBadRequest, gin.H{"error": err.Error()})
		return
	}

	userID := c.GetUint64("user_id")

	c.Writer.Header().Set("Content-Type", "text/event-stream")
	c.Writer.Header().Set("Cache-Control", "no-cache")
	c.Writer.Header().Set("Connection", "keep-alive")
	c.Writer.WriteHeader(http.StatusOK)

	flusher, ok := c.Writer.(http.Flusher)
	if !ok {
		c.JSON(http.StatusInternalServerError, gin.H{"error": "streaming not supported"})
		return
	}

	ch, err := h.svc.SendMessage(c.Request.Context(), userID, req.Message)
	if err != nil {
		data, _ := json.Marshal(gin.H{"error": err.Error()})
		c.Writer.Write([]byte("data: " + string(data) + "\n\n"))
		flusher.Flush()
		return
	}

	for res := range ch {
		if res.Err != nil {
			data, _ := json.Marshal(gin.H{"error": res.Err.Error()})
			c.Writer.Write([]byte("data: " + string(data) + "\n\n"))
			flusher.Flush()
			return
		}
		data, _ := json.Marshal(gin.H{"content": res.Content})
		c.Writer.Write([]byte("data: " + string(data) + "\n\n"))
		flusher.Flush()
	}

	data, _ := json.Marshal(gin.H{"done": true})
	c.Writer.Write([]byte("data: " + string(data) + "\n\n"))
	flusher.Flush()
}
```

- [ ] **Step 2: Create chat service (wraps existing LLM client)**

```go
// choex-server/internal/agent/chat.go
package agent

import (
	"context"

	"github.com/choex2025-ops/choex-server/internal/config"
	"github.com/choex2025-ops/choex-server/llm"
)

type ChatService struct {
	cfg *config.Config
}

func NewChatService(cfg *config.Config) *ChatService {
	return &ChatService{cfg: cfg}
}

type StreamChunk struct {
	Content string
	Err     error
}

func (s *ChatService) SendMessage(ctx context.Context, userID uint64, message string) (<-chan StreamChunk, error) {
	systemPrompt := "你是 ChoexManager 的个人生活管家。你可以帮助用户管理日程、记账、查询密码簿。请用温和、简洁的语言回复。"

	_, rawCh, err := llm.CallDeepSeek(systemPrompt, []llm.Message{
		{Role: "user", Content: message},
	}, true)

	if err != nil {
		return nil, err
	}

	ch := make(chan StreamChunk)
	go func() {
		defer close(ch)
		for res := range rawCh {
			ch <- StreamChunk{Content: res.Content, Err: res.Err}
		}
	}()

	return ch, nil
}
```

- [ ] **Step 3: Register agent route in router**

```go
// Add to choex-server/internal/router/router.go

import (
	// ... existing imports
	"github.com/choex2025-ops/choex-server/internal/agent"
)

// Inside Setup(), after the auth routes, add:
agentHandler := agent.NewAgentHandler(cfg)

protected := api.Group("")
protected.Use(middleware.AuthRequired(authSvc))
{
	protected.POST("/agent/chat", agentHandler.Chat)
}
```

Full updated `Setup()` function:

```go
func Setup(cfg *config.Config) *gin.Engine {
	r := gin.Default()
	r.Use(middleware.CORS())

	authSvc := service.NewAuthService(cfg)
	authHandler := handler.NewAuthHandler(authSvc)
	agentHandler := agent.NewAgentHandler(cfg)

	api := r.Group("/api")
	{
		auth := api.Group("/auth")
		{
			auth.POST("/register", authHandler.Register)
			auth.POST("/login", authHandler.Login)
		}

		protected := api.Group("")
		protected.Use(middleware.AuthRequired(authSvc))
		{
			protected.POST("/agent/chat", agentHandler.Chat)
		}
	}

	return r
}
```

- [ ] **Step 4: Test agent chat**

```bash
# Start server
cd ~/ClaudeProjects/choex-server
# Ensure DEEPSEEK_API_KEY env var is set
go run cmd/server/main.go

# In another terminal, test SSE chat
curl -X POST http://localhost:8080/api/agent/chat \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token-from-login>" \
  -d '{"message":"你好"}'
# Expected: SSE stream with content chunks
```

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: add agent chat endpoint with SSE streaming"
git push
```

---

## Task 14: Frontend SSE hook and chat integration

**Files:**
- Create: `choex-web/src/hooks/useSSE.ts`
- Create: `choex-web/src/store/chatSlice.ts`
- Modify: `choex-web/src/store/index.ts`
- Modify: `choex-web/src/components/Layout/AgentPanel.tsx`
- Modify: `choex-web/src/components/Layout/InputBar.tsx`
- Modify: `choex-web/src/pages/WorkspacePage.tsx`

- [ ] **Step 1: Create SSE hook**

```typescript
// choex-web/src/hooks/useSSE.ts
import { useCallback } from 'react';

interface SSECallbacks {
  onToken: (content: string) => void;
  onDone: () => void;
  onError: (error: string) => void;
}

export function useSSE() {
  const connect = useCallback(
    (url: string, body: unknown, callbacks: SSECallbacks) => {
      const token = localStorage.getItem('token');

      fetch(url, {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          Authorization: `Bearer ${token}`,
        },
        body: JSON.stringify(body),
      }).then(async (response) => {
        if (!response.ok) {
          const err = await response.json().catch(() => ({ error: 'SSE failed' }));
          callbacks.onError(err.error || `HTTP ${response.status}`);
          return;
        }

        const reader = response.body?.getReader();
        if (!reader) {
          callbacks.onError('No response body');
          return;
        }

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
            const jsonStr = line.slice(6);
            try {
              const data = JSON.parse(jsonStr);
              if (data.content) {
                callbacks.onToken(data.content);
              } else if (data.done) {
                callbacks.onDone();
              } else if (data.error) {
                callbacks.onError(data.error);
              }
            } catch {
              // Skip unparseable lines
            }
          }
        }
      }).catch((err) => {
        callbacks.onError(err.message);
      });
    },
    []
  );

  return { connect };
}
```

- [ ] **Step 2: Create chat slice**

```typescript
// choex-web/src/store/chatSlice.ts
import { createSlice, type PayloadAction } from '@reduxjs/toolkit';

interface Message {
  role: 'user' | 'assistant';
  content: string;
}

interface ChatState {
  messages: Message[];
  streaming: boolean;
}

const initialState: ChatState = {
  messages: [
    { role: 'assistant', content: '你好！我是你的生活管家。需要什么帮助吗？' },
  ],
  streaming: false,
};

const chatSlice = createSlice({
  name: 'chat',
  initialState,
  reducers: {
    addMessage(state, action: PayloadAction<Message>) {
      state.messages.push(action.payload);
    },
    appendToLastMessage(state, action: PayloadAction<string>) {
      const last = state.messages[state.messages.length - 1];
      if (last && last.role === 'assistant') {
        last.content += action.payload;
      }
    },
    setStreaming(state, action: PayloadAction<boolean>) {
      state.streaming = action.payload;
    },
  },
});

export const { addMessage, appendToLastMessage, setStreaming } = chatSlice.actions;
export default chatSlice.reducer;
```

- [ ] **Step 3: Register chat slice in store**

```typescript
// choex-web/src/store/index.ts
import { configureStore } from '@reduxjs/toolkit';
import authReducer from './authSlice';
import chatReducer from './chatSlice';

export const store = configureStore({
  reducer: {
    auth: authReducer,
    chat: chatReducer,
  },
});

export type RootState = ReturnType<typeof store.getState>;
export type AppDispatch = typeof store.dispatch;
```

- [ ] **Step 4: Update AgentPanel with chat state**

```tsx
// choex-web/src/components/Layout/AgentPanel.tsx
import { useAppSelector } from '../../hooks/useAppDispatch';

export default function AgentPanel() {
  const { messages, streaming } = useAppSelector((s) => s.chat);

  return (
    <div style={{
      width: 320,
      background: '#161b22',
      borderLeft: '1px solid #30363d',
      display: 'flex',
      flexDirection: 'column',
    }}>
      <div style={{
        padding: '10px 14px',
        borderBottom: '1px solid #30363d',
        display: 'flex',
        alignItems: 'center',
        gap: 8,
      }}>
        <span style={{ color: '#e6edf3', fontWeight: 600, fontSize: 12 }}>Agent</span>
        <div style={{ marginLeft: 'auto', display: 'flex', gap: 4 }}>
          <span style={styles.iconBtn} title="记忆管理">🧠</span>
          <span style={styles.iconBtn} title="更多设置">⚙</span>
        </div>
      </div>

      <div style={{ flex: 1, padding: 16, display: 'flex', flexDirection: 'column', gap: 12, overflow: 'auto' }}>
        {messages.map((msg, i) => (
          <div key={i} style={{
            display: 'flex',
            gap: 8,
            flexDirection: msg.role === 'user' ? 'row-reverse' : 'row',
          }}>
            <div style={{
              ...styles.avatar,
              background: msg.role === 'user' ? '#30363d' : '#1f6feb',
            }}>
              {msg.role === 'user' ? '👤' : '🤖'}
            </div>
            <div style={{
              ...styles.bubble,
              background: msg.role === 'user' ? 'rgba(31,111,235,0.15)' : '#21262d',
              border: msg.role === 'user' ? '1px solid rgba(31,111,235,0.27)' : 'none',
            }}>
              {msg.content}
            </div>
          </div>
        ))}
        {streaming && (
          <div style={{ display: 'flex', gap: 8 }}>
            <div style={{ ...styles.avatar, background: '#1f6feb' }}>🤖</div>
            <div style={{ ...styles.bubble, background: '#21262d', fontStyle: 'italic', color: '#8b949e' }}>
              思考中...
            </div>
          </div>
        )}
      </div>
    </div>
  );
}

const styles: Record<string, React.CSSProperties> = {
  iconBtn: {
    width: 28,
    height: 28,
    background: '#21262d',
    borderRadius: 6,
    textAlign: 'center',
    lineHeight: '28px',
    fontSize: 13,
    color: '#8b949e',
    cursor: 'pointer',
  },
  avatar: {
    width: 28,
    height: 28,
    borderRadius: 6,
    flexShrink: 0,
    textAlign: 'center',
    lineHeight: '28px',
    fontSize: 12,
  },
  bubble: {
    borderRadius: 8,
    padding: '8px 12px',
    fontSize: 12,
    color: '#e6edf3',
    maxWidth: 220,
    wordBreak: 'break-word' as const,
  },
};
```

- [ ] **Step 5: Update InputBar with SSE send logic**

```tsx
// choex-web/src/components/Layout/InputBar.tsx
import { useState } from 'react';
import { useAppDispatch } from '../../hooks/useAppDispatch';
import { addMessage, appendToLastMessage, setStreaming } from '../../store/chatSlice';
import { useSSE } from '../../hooks/useSSE';

export default function InputBar() {
  const [text, setText] = useState('');
  const dispatch = useAppDispatch();
  const { connect } = useSSE();

  const handleSend = () => {
    if (!text.trim()) return;

    dispatch(addMessage({ role: 'user', content: text }));
    dispatch(addMessage({ role: 'assistant', content: '' }));
    dispatch(setStreaming(true));

    connect('/api/agent/chat', { message: text }, {
      onToken: (content) => {
        dispatch(appendToLastMessage(content));
      },
      onDone: () => {
        dispatch(setStreaming(false));
      },
      onError: (error) => {
        dispatch(appendToLastMessage(`[错误: ${error}]`));
        dispatch(setStreaming(false));
      },
    });

    setText('');
  };

  return (
    <div style={{
      padding: '12px 20px',
      background: '#161b22',
      borderTop: '1px solid #30363d',
      display: 'flex',
      gap: 8,
      alignItems: 'center',
    }}>
      <input
        style={{
          flex: 1,
          background: '#0d1117',
          border: '1px solid #30363d',
          borderRadius: 6,
          padding: '8px 12px',
          color: '#e6edf3',
          fontSize: 13,
          outline: 'none',
        }}
        placeholder="输入指令，如'帮我记一笔午餐30元'..."
        value={text}
        onChange={(e) => setText(e.target.value)}
        onKeyDown={(e) => e.key === 'Enter' && handleSend()}
      />
      <button
        style={{
          background: '#1f6feb',
          color: 'white',
          border: 'none',
          padding: '8px 20px',
          borderRadius: 6,
          fontSize: 13,
          cursor: 'pointer',
        }}
        onClick={handleSend}
      >
        发送
      </button>
    </div>
  );
}
```

- [ ] **Step 6: Verify full chat flow**

```bash
# Terminal 1: Start backend
cd ~/ClaudeProjects/choex-server
go run cmd/server/main.go

# Terminal 2: Start frontend
cd ~/ClaudeProjects/choex-web
npm run dev

# Open http://localhost:5173
# Register → navigate to /workspace → type in input bar → send
# Expected: Agent responds with streaming text in the right panel
```

- [ ] **Step 7: Commit**

```bash
git add -A
git commit -m "feat: integrate SSE chat streaming in frontend"
git push
```

---

## End-to-End Verification

After all tasks complete, verify the full flow:

1. Open `http://localhost:5173` → Login page renders
2. Register a new user → Workspace page loads
3. Workspace shows: left sidebar, welcome page (center), Agent panel (right), input bar (bottom)
4. Type message in input bar → Agent responds with streaming text in right panel
5. Clicking sidebar items toggles active state (app content rendered in later phases)
6. Refresh page → stays on workspace (auth token persisted)
