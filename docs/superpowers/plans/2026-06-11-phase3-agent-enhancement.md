# Phase 3: Agent Enhancement Implementation Plan

**Goal:** Agent Tool Calling (control apps via chat) + Memory Management (editable profiles with version history + slide toggle activation).

---

## Backend Tasks

### Task 24: Agent Tool Calling engine
Update `agent/chat.go` and `agent/tools.go` to register app tools and support function calling.
- Define Tool struct with Name, Description, Parameters (JSON), Handler
- Register calendar create/list, bills create/list/stats, passwords list as tools
- Update chat handler to detect tool calls and execute them
- Return tool results to LLM for final response

### Task 25: Agent Memory models + API
Create `model/memory.go` with `AgentMemory` and `MemoryVersion` GORM models.
Create `service/memory.go` and `handler/memory.go` with CRUD + version management:
- GET/POST/PUT/DELETE /api/memories
- PUT /api/memories/:id/activate (mutual exclusion)
- GET/PUT /api/memories/:id/versions/current (save moves old current→backup)
- PUT /api/memories/:id/versions/custom
- PUT /api/memories/:id/restore (from backup)

## Frontend Tasks

### Task 26: Memory List page
Create `src/agent-settings/MemoryList.tsx` - vertical card list with:
- Each card: icon, name, preview, last modified date
- Active card: blue border + glow + "使用中" badge
- Slide toggle (mutual exclusion, only 1 active)
- "+ 新建记忆" button

### Task 27: Memory Editor page
Create `src/agent-settings/MemoryEditor.tsx` - editor with:
- 3 version tabs: Current / Backup / Custom
- Text editor for system prompt content
- Bottom bar: restore from backup, save to custom, save changes, close

### Task 28: Agent settings integration
Update AgentPanel to open MemoryList in center area when 🧠 clicked.
Wire WorkspacePage to support agent settings panels alongside apps.

---

## Verification
- Type "帮我记录一笔午餐30元" → Agent creates bill via Tool Calling
- Type "我明天有什么日程" → Agent lists calendar events
- Open memory management → switch between profiles → edit and save
- Slide toggle activates different memory → Agent personality changes
