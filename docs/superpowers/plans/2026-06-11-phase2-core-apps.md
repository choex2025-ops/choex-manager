# Phase 2: Core Applications Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development.

**Goal:** Implement Calendar, Bills, Passwords, and Browser apps — full CRUD with frontend components, backend APIs, Agent Tool Calling registration.

**Tech Stack:** Same as Phase 1 — Go/Gin + React/Redux + MySQL

---

## File Structure (new files only)

### choex-server additions

```
choex-server/internal/
├── model/
│   ├── event.go           # Event GORM model
│   ├── bill.go            # Bill GORM model
│   ├── password.go        # Password GORM model
├── service/
│   ├── calendar.go        # Calendar business logic
│   ├── bill.go            # Bill business logic
│   ├── password.go        # Password business logic (AES encrypt/decrypt)
├── handler/
│   ├── calendar.go        # Calendar HTTP handlers
│   ├── bill.go            # Bill HTTP handlers
│   ├── password.go        # Password HTTP handlers
│   └── proxy.go           # Browser proxy handler
├── app/
│   └── app.go             # App interface definition
```

### choex-web additions

```
choex-web/src/
├── apps/
│   ├── calendar/
│   │   ├── index.tsx      # Calendar main component
│   │   └── types.ts       # Calendar types
│   ├── bills/
│   │   ├── index.tsx      # Bills main component
│   │   └── types.ts       # Bills types
│   ├── passwords/
│   │   ├── index.tsx      # Passwords main component
│   │   └── types.ts       # Passwords types
│   └── browser/
│       └── index.tsx      # Browser component
```

---

## Task 15: Calendar backend (model + service + handler)

**Files:**
- Create: `choex-server/internal/model/event.go`
- Create: `choex-server/internal/service/calendar.go`
- Create: `choex-server/internal/handler/calendar.go`
- Modify: `choex-server/cmd/server/main.go` (add migration)
- Modify: `choex-server/internal/router/router.go`

Create Event model:

```go
// model/event.go
package model

import "time"

type Event struct {
    ID          uint64    `gorm:"primaryKey;autoIncrement" json:"id"`
    UserID      uint64    `gorm:"index;not null" json:"user_id"`
    Title       string    `gorm:"size:256;not null" json:"title"`
    Description string    `gorm:"type:text" json:"description"`
    Location    string    `gorm:"size:256" json:"location"`
    StartTime   time.Time `gorm:"not null" json:"start_time"`
    EndTime     time.Time `json:"end_time"`
    AllDay      bool      `gorm:"default:false" json:"all_day"`
    Color       string    `gorm:"size:16" json:"color"`
    CreatedAt   time.Time `json:"created_at"`
    UpdatedAt   time.Time `json:"updated_at"`
}
```

Create CalendarService with CRUD operations. Create CalendarHandler with:
- `GET /api/events` — list current user's events
- `POST /api/events` — create event
- `PUT /api/events/:id` — update event
- `DELETE /api/events/:id` — delete event

All routes protected by auth middleware. Each handler extracts `user_id` from context.

## Task 16: Bills backend (model + service + handler)

Create Bill model with fields: id, user_id, amount, type (income/expense), category, note, bill_date.
Create BillService and BillHandler with:
- `GET /api/bills` — list with optional date range query
- `POST /api/bills` — create
- `DELETE /api/bills/:id` — delete
- `GET /api/bills/stats` — monthly summary (group by category, total income/expense)

## Task 17: Passwords backend (model + service + handler)

Create Password model with AES-256 encryption for the password field.
Create PasswordService with encrypt/decrypt using a server key from config.
Create PasswordHandler with:
- `GET /api/passwords` — list (without plaintext)
- `POST /api/passwords` — create (encrypt on write)
- `GET /api/passwords/:id` — get with decrypted password
- `PUT /api/passwords/:id` — update
- `DELETE /api/passwords/:id` — delete

## Task 18: App interface + router registration

Create `internal/app/app.go` with the `App` interface:

```go
type Route struct {
    Method  string
    Path    string
    Handler gin.HandlerFunc
}
type App interface {
    Name() string
    Icon() string
    Routes() []Route
}
```

Create a registry in router that apps register to. Wire calendar, bills, passwords handlers through this interface.

## Task 19: Calendar frontend

Create `src/apps/calendar/index.tsx` with:
- Event list view (today's events + upcoming)
- "New Event" button → inline form
- Display: title, time, location, color tag

## Task 20: Bills frontend

Create `src/apps/bills/index.tsx` with:
- Summary bar (monthly income, expense, balance)
- Bill list with amount, category, date
- "Add Bill" form
- Category filter

## Task 21: Passwords frontend

Create `src/apps/passwords/index.tsx` with:
- Password list (title, url, username, masked password)
- "Show password" toggle (calls GET /api/passwords/:id for decrypted view)
- "Add Password" form
- Category grouping

## Task 22: Browser frontend + proxy

Create `src/apps/browser/index.tsx` with address bar + iframe.
Create `choex-server/internal/handler/proxy.go` with `/proxy?url=...` endpoint that fetches and serves external page content.

## Task 23: Workspace integration

Update `WorkspacePage.tsx` to dynamically render the selected app component instead of always showing Welcome. The active app state drives which component mounts.

---

## Verification

- Register, login → workspace loads
- Click 📅 → Calendar app renders with event list
- Click 💳 → Bills app renders with summary
- Click 🔐 → Passwords app renders with list
- Click 🌐 → Browser opens with address bar
- Agent can reference app data in conversation (Tool Calling wired in Phase 3)
