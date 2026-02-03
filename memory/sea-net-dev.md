# sea-net-desktop-v2 - Development Memory

## Project Overview
- **Type:** Desktop Application (Build from scratch)
- **Reference:** sea-net-desktop-v1 (features only, no code reuse)
- **Goal:** Build improved version with better architecture & UX
- **Timeline:** 5-7 days (Target: 10 February 2026)

---

## Tech Stack (Confirmed)

### Frontend
- **Framework:** React 18+ + TypeScript
- **Build Tool:** Vite
- **UI Components:** shadcn/ui (Radix UI primitives + Tailwind)
- **State Management:**
  - Zustand (lightweight client state)
  - TanStack Query (React Query) for server state, caching, sync
- **Forms:** React Hook Form + Zod (validation)
- **Styling:** Tailwind CSS
- **Icons:** lucide-react
- **Date Handling:** date-fns

### Backend
- **Framework:** NestJS 10+ + TypeScript
- **ORM:** Prisma
- **Database:** PostgreSQL
- **Real-time:** Socket.io (optional, if needed)
- **Background Jobs:** Bull Queue (optional, if needed)
- **Email:** SendGrid/Nodemailer (optional, if needed)

### Desktop
- **Framework:** Tauri v2

### DevOps
- **Package Manager:** Bun (faster than npm/pnpm)
- **CI/CD:** GitHub Actions

---

## Architecture Decisions

### v1 Architecture Issues (Identified by PM)
- Code structure unclear (no documentation)
- Tight coupling between components
- Lack of type safety in some areas
- No proper state management strategy
- Limited error handling
- No offline support
- Minimal testing

### v2 Architecture Improvements

#### 1. Layered Architecture
```
┌─────────────────────────────────────────┐
│         Desktop Layer (Tauri)          │
│  - Native windows                       │
│  - System tray                          │
│  - File system access                   │
│  - Auto-updater                         │
└─────────────────────────────────────────┘
                    ↓ IPC
┌─────────────────────────────────────────┐
│      Frontend Layer (React)            │
│  - UI Components (shadcn/ui)            │
│  - View Logic                           │
│  - Client State (Zustand)               │
│  - Server State (TanStack Query)        │
└─────────────────────────────────────────┘
                    ↓ HTTP/WebSocket
┌─────────────────────────────────────────┐
│       Backend Layer (NestJS)            │
│  - Controllers (API endpoints)          │
│  - Services (Business logic)            │
│  - Repositories (Data access)           │
│  - Guards (Auth, permissions)           │
└─────────────────────────────────────────┘
                    ↓ Prisma
┌─────────────────────────────────────────┐
│      Database Layer (PostgreSQL)        │
│  - Tables (Users, Customers, Bookings)  │
│  - Relationships                        │
│  - Indexes                              │
└─────────────────────────────────────────┘
```

#### 2. Feature-Based Module Structure
```
src/
├── frontend/
│   ├── modules/
│   │   ├── auth/
│   │   ├── customers/
│   │   ├── bookings/
│   │   ├── shipments/
│   │   ├── tasks/
│   │   ├── documents/
│   │   └── dashboard/
│   ├── shared/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── stores/
│   │   ├── api/
│   │   ├── utils/
│   │   └── types/
│   └── app/
│
├── backend/
│   ├── modules/
│   │   ├── auth/
│   │   ├── customers/
│   │   ├── bookings/
│   │   ├── shipments/
│   │   ├── tasks/
│   │   ├── documents/
│   │   └── dashboard/
│   ├── shared/
│   │   ├── guards/
│   │   ├── decorators/
│   │   ├── interceptors/
│   │   ├── filters/
│   │   ├── pipes/
│   │   └── utils/
│   └── config/
│
└── desktop/
    └── src-tauri/
```

#### 3. Data Flow
```
User Action
    ↓
React Component (View)
    ↓
Zustand Store (Client State) ←→ React Hook Form (Form State)
    ↓
TanStack Query (Server State)
    ↓
HTTP Request (fetch/axios)
    ↓
NestJS Controller
    ↓
NestJS Service (Business Logic)
    ↓
Prisma Repository (Data Access)
    ↓
PostgreSQL Database
    ↓
Response (JSON)
    ↓
TanStack Query Cache Update
    ↓
React Component Re-render
```

#### 4. Authentication & Authorization
```
JWT Token (Access + Refresh)
    ↓
Guards (NestJS)
    ↓
Role-Based Access Control (RBAC)
    ↓
Resource Guards (Ownership check)
```

#### 5. State Management Strategy
- **Client State:** Zustand (UI state, modals, themes, user session)
- **Server State:** TanStack Query (API data, caching, synchronization)
- **Form State:** React Hook Form (form values, validation)
- **URL State:** React Router (search params, location state)

---

## Database Schema Design (Draft)

### Core Tables

#### Users
```prisma
model User {
  id            String    @id @default(cuid())
  email         String    @unique
  passwordHash  String
  name          String
  role          UserRole  @default(STAFF)
  isActive      Boolean   @default(true)
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  lastLoginAt   DateTime?

  // Relations
  createdCustomers Customer[] @relation("CreatedBy")
  createdBookings  Booking[]   @relation("BookingCreatedBy")
  assignedTasks    Task[]
  auditLogs        AuditLog[]

  @@map("users")
}

enum UserRole {
  ADMIN
  STAFF
  CUSTOMER
}
```

#### Customers
```prisma
model Customer {
  id        String   @id @default(cuid())
  name      String
  email     String?
  phone     String?
  address   String?
  notes     String?
  isActive  Boolean  @default(true)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  // Relations
  createdBy    User      @relation("CreatedBy", fields: [createdById], references: [id])
  createdById  String
  bookings     Booking[]

  @@map("customers")
}
```

#### Bookings
```prisma
model Booking {
  id          String         @id @default(cuid())
  bookingNo   String         @unique
  customerId  String
  carrier     String
  vessel      String
  voyage      String
  pol         String  // Port of Loading
  pod         String  // Port of Discharge
  etd         DateTime
  containers  Container[]

  // Relations
  customer    Customer    @relation(fields: [customerId], references: [id])
  createdBy   User        @relation("BookingCreatedBy", fields: [createdById], references: [id])
  createdById String
  shipments   Shipment[]
  documents   Document[]

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("bookings")
}
```

#### Containers
```prisma
model Container {
  id        String   @id @default(cuid())
  bookingId String
  containerNo String
  type      String
  size      String
  weight    Float?

  booking   Booking  @relation(fields: [bookingId], references: [id])

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("containers")
}
```

#### Shipments
```prisma
model Shipment {
  id          String          @id @default(cuid())
  bookingId   String
  status      ShipmentStatus  @default(PENDING)
  notes       String?
  currentLoc  String?         // Current location

  // Relations
  booking     Booking         @relation(fields: [bookingId], references: [id])
  tracking    TrackingEvent[]

  createdAt   DateTime        @default(now())
  updatedAt   DateTime        @updatedAt

  @@map("shipments")
}

enum ShipmentStatus {
  PENDING
  BOOKED
  IN_TRANSIT
  CUSTOMS
  DELIVERING
  DELIVERED
  CANCELLED
}
```

#### Tracking Events
```prisma
model TrackingEvent {
  id          String   @id @default(cuid())
  shipmentId  String
  status      String
  location    String
  notes       String?
  timestamp   DateTime @default(now())

  shipment    Shipment @relation(fields: [shipmentId], references: [id])

  @@map("tracking_events")
}
```

#### Tasks
```prisma
model Task {
  id          String     @id @default(cuid())
  title       String
  description String?
  priority    TaskPriority @default(MEDIUM)
  status      TaskStatus   @default(PENDING)
  dueDate     DateTime?
  assignedTo  String?    // User ID

  // Relations
  assignedUser User?     @relation(fields: [assignedTo], references: [id])

  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt

  @@map("tasks")
}

enum TaskPriority {
  LOW
  MEDIUM
  HIGH
  URGENT
}

enum TaskStatus {
  PENDING
  IN_PROGRESS
  COMPLETED
  CANCELLED
}
```

#### Documents
```prisma
model Document {
  id          String        @id @default(cuid())
  bookingId   String?
  name        String
  fileName    String
  filePath    String
  fileType    String
  fileSize    Int

  // Relations
  booking     Booking?      @relation(fields: [bookingId], references: [id])

  createdAt   DateTime      @default(now())

  @@map("documents")
}
```

#### Templates
```prisma
model Template {
  id          String   @id @default(cuid())
  name        String
  type        String
  fileName    String
  description String?

  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt

  @@map("templates")
}
```

#### Audit Logs
```prisma
model AuditLog {
  id          String       @id @default(cuid())
  userId      String
  action      AuditAction
  entity      String
  entityId    String?
  details     Json?
  ipAddress   String?
  userAgent   String?

  // Relations
  user        User         @relation(fields: [userId], references: [id])

  createdAt   DateTime     @default(now())

  @@map("audit_logs")
}

enum AuditAction {
  LOGIN
  LOGOUT
  CREATE_CUSTOMER
  UPDATE_CUSTOMER
  DELETE_CUSTOMER
  CREATE_BOOKING
  UPDATE_BOOKING
  DELETE_BOOKING
  UPDATE_SHIPMENT_STATUS
  CREATE_TASK
  UPDATE_TASK
  DELETE_TASK
  UPLOAD_DOCUMENT
  DELETE_DOCUMENT
  CREATE_TEMPLATE
  UPDATE_TEMPLATE
  DELETE_TEMPLATE
  EXPORT_DATA
  IMPORT_DATA
  UPDATE_SETTINGS
}
```

---

## API Design (Draft)

### Authentication
```
POST   /api/auth/login
POST   /api/auth/logout
POST   /api/auth/refresh
GET    /api/auth/me
```

### Customers
```
GET    /api/customers          // List with pagination, search, filter
GET    /api/customers/:id      // Get single
POST   /api/customers           // Create
PUT    /api/customers/:id      // Update
DELETE /api/customers/:id      // Delete
```

### Bookings
```
GET    /api/bookings           // List with pagination, search, filter
GET    /api/bookings/:id       // Get single
POST   /api/bookings           // Create
PUT    /api/bookings/:id       // Update
DELETE /api/bookings/:id       // Delete
GET    /api/bookings/:id/shipments
```

### Shipments
```
GET    /api/shipments          // List with pagination, search, filter
GET    /api/shipments/:id      // Get single
PUT    /api/shipments/:id      // Update (change status, location)
GET    /api/shipments/:id/tracking
```

### Tasks
```
GET    /api/tasks              // List with pagination, search, filter
GET    /api/tasks/:id          // Get single
POST   /api/tasks              // Create
PUT    /api/tasks/:id          // Update
DELETE /api/tasks/:id          // Delete
POST   /api/tasks/:id/assign   // Assign to user
```

### Documents
```
GET    /api/documents          // List
POST   /api/documents          // Upload
GET    /api/documents/:id      // Download
DELETE /api/documents/:id      // Delete
```

### Templates
```
GET    /api/templates          // List
POST   /api/templates          // Create
PUT    /api/templates/:id      // Update
DELETE /api/templates/:id      // Delete
```

### Dashboard
```
GET    /api/dashboard/stats    // Statistics (bookings, shipments, tasks)
GET    /api/dashboard/recent   // Recent activity
```

### Export
```
GET    /api/export/bookings    // Export bookings to Excel/PDF
GET    /api/export/customers   // Export customers to Excel
GET    /api/export/shipments   // Export shipments to Excel
```

---

## Architecture Diagrams

### High-Level System Architecture
```
┌─────────────────────────────────────────────────────────────────┐
│                        Desktop App                             │
│                   (Tauri + React + Vite)                       │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                   Frontend Layer                        │  │
│  │  ┌───────────────────────────────────────────────────┐ │  │
│  │  │  UI Components (shadcn/ui)                       │ │  │
│  │  │  - Tables, Forms, Cards, Modals, etc.            │ │  │
│  │  └───────────────────────────────────────────────────┘ │  │
│  │  ┌───────────────────────────────────────────────────┐ │  │
│  │  │  State Management                                │ │  │
│  │  │  - Zustand (UI state)                            │ │  │
│  │  │  - TanStack Query (server state)                 │ │  │
│  │  │  - React Hook Form (form state)                  │ │  │
│  │  └───────────────────────────────────────────────────┘ │  │
│  │  ┌───────────────────────────────────────────────────┐ │  │
│  │  │  Feature Modules                                  │ │  │
│  │  │  - Auth, Customers, Bookings, Shipments, Tasks   │ │  │
│  │  └───────────────────────────────────────────────────┘ │  │
│  └─────────────────────────────────────────────────────────┘  │
│                            ↓ HTTP/Tauri IPC                    │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                   Desktop Integration                    │  │
│  │  - File system access                                   │  │
│  │  - System notifications                                 │  │
│  │  - Auto-updater                                         │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                            ↓ REST API
┌─────────────────────────────────────────────────────────────────┐
│                      Backend Server                            │
│                      (NestJS + Prisma)                         │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                   API Layer                             │  │
│  │  - Controllers (routing)                                │  │
│  │  - Guards (auth, RBAC)                                   │  │
│  │  - Pipes (validation)                                    │  │
│  └─────────────────────────────────────────────────────────┘  │
│                            ↓                                    │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                   Business Logic Layer                   │  │
│  │  - Services (domain logic)                              │  │
│  │  - Event handlers                                       │  │
│  │  - Background jobs (Bull)                               │  │
│  └─────────────────────────────────────────────────────────┘  │
│                            ↓                                    │
│  ┌─────────────────────────────────────────────────────────┐  │
│  │                   Data Access Layer                     │  │
│  │  - Prisma Client                                        │  │
│  │  - Repositories                                          │  │
│  └─────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                            ↓ SQL
┌─────────────────────────────────────────────────────────────────┐
│                   Database (PostgreSQL)                        │
│  - Users, Customers, Bookings, Shipments, Tasks, Documents    │
│  - Templates, Audit Logs                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Authentication Flow
```
User enters credentials
    ↓
React Component
    ↓
POST /api/auth/login
    ↓
NestJS AuthController
    ↓
AuthService.validateUser()
    ↓
Compare with database (Prisma)
    ↓
If valid:
    - Generate JWT Access Token (15 min)
    - Generate JWT Refresh Token (7 days)
    - Store refresh token hash in DB
    ↓
Return { accessToken, refreshToken }
    ↓
Store in Zustand auth store
    ↓
Set axios Authorization header
    ↓
Redirect to dashboard
```

### Booking Creation Flow
```
User fills booking form
    ↓
React Hook Form (validation with Zod)
    ↓
POST /api/bookings
    ↓
NestJS BookingController
    ↓
AuthGuard (check token)
    ↓
BookingService.create()
    ↓
Transaction:
    - Create Booking record
    - Create Container records
    - Create Shipment record (status: PENDING)
    - Create AuditLog
    ↓
Return created booking
    ↓
TanStack Query updates cache
    ↓
Redirect to booking detail view
```

### Real-time Updates (Optional - WebSocket)
```
┌──────────────────┐         WebSocket          ┌──────────────────┐
│   Client (React) │ ←───────────────────────→ │  NestJS Server   │
│  - Socket.io     │                            │  - Socket.io     │
└──────────────────┘                            └────────┬─────────┘
                                                       │
                                               ┌───────▼────────┐
                                               │   Events        │
                                               │ - shipment:updated
                                               │ - task:assigned
                                               │ - notification  │
                                               └────────────────┘
```

---

## Key Architecture Decisions (Log)

### 1. State Management
**Decision:** Use Zustand + TanStack Query
**Reason:**
- Zustand is lightweight (small bundle)
- TanStack Query handles server state, caching, and synchronization automatically
- Separation of concerns: client state vs server state
**Date:** 2026-02-03

### 2. Form Handling
**Decision:** React Hook Form + Zod
**Reason:**
- Minimal re-renders
- Excellent performance
- Type-safe validation with Zod
**Date:** 2026-02-03

### 3. UI Components
**Decision:** shadcn/ui
**Reason:**
- Modern, customizable components
- Based on Radix UI (accessible primitives)
- Copy-paste components (full control)
- Tailwind-based (consistent styling)
**Date:** 2026-02-03

### 4. Database ORM
**Decision:** Prisma
**Reason:**
- Excellent TypeScript support
- Auto-generated types
- Great migration system
- Easy to use
**Date:** 2026-02-03

### 5. API Communication
**Decision:** REST API with TanStack Query
**Reason:**
- Simpler than GraphQL for this use case
- TanStack Query handles caching, loading states, error states
**Date:** 2026-02-03

### 6. Module Structure
**Decision:** Feature-based modules
**Reason:**
- Better code organization
- Easier to maintain
- Better scalability
**Date:** 2026-02-03

---

## TODOs

### Architecture & Design
- [x] Review v1 features
- [x] Design v2 architecture improvements
- [x] Create architecture diagrams
- [x] Write architecture documentation
- [x] Design database schema
- [x] Design API endpoints
- [ ] Refine architecture based on feedback

### Project Setup (Next)
- [ ] Initialize Tauri + React + Vite project
- [ ] Setup shadcn/ui
- [ ] Setup Tailwind CSS
- [ ] Setup Zustand stores
- [ ] Setup TanStack Query
- [ ] Setup React Hook Form + Zod
- [ ] Setup NestJS backend
- [ ] Setup Prisma + PostgreSQL
- [ ] Setup ESLint, Prettier
- [ ] Setup GitHub Actions (CI/CD)

### Implementation (After Setup)
- [ ] Implement authentication (login, register, logout)
- [ ] Implement user management
- [ ] Implement customer management
- [ ] Implement booking system
- [ ] Implement shipment tracking
- [ ] Implement task management
- [ ] Implement document management
- [ ] Implement template manager
- [ ] Implement dashboard
- [ ] Implement export features
- [ ] Implement dark mode
- [ ] Implement keyboard shortcuts

---

## Notes & Thoughts

### Potential Improvements Over v1
1. **Better Type Safety:** Full TypeScript coverage, no `any` types
2. **State Management:** Proper separation of concerns (Zustand + TanStack Query)
3. **Error Handling:** Global error boundary, user-friendly error messages
4. **Loading States:** Consistent loading indicators everywhere
5. **Form Validation:** Real-time validation with clear error messages
6. **Accessibility:** WCAG AA compliance (via shadcn/ui components)
7. **Performance:** Code splitting, lazy loading, memoization
8. **Testing:** Unit tests, integration tests, E2E tests
9. **Documentation:** Clear architecture docs, API docs, component docs
10. **Developer Experience:** Hot reload, fast builds, consistent code style

### Risks & Mitigations
1. **Timeline Risk (5-7 days):**
   - Mitigation: Focus on core features first, defer nice-to-haves
   - Mitigation: Use proven libraries (shadcn/ui, Prisma, etc.)

2. **Complex Features (Real-time, Offline mode):**
   - Mitigation: Implement in later sprints if time permits
   - Mitigation: Use simpler alternatives (polling instead of WebSocket)

3. **Desktop Integration (Tauri v2):**
   - Mitigation: Use Tauri v2 (more stable than v1)
   - Mitigation: Start with basic desktop features, add advanced later

---

## Sprint 1 Day 1 - Work Summary (2026-02-03 15:00)

### ✅ Completed
- Review sea-net-v1 features and architecture
- Design v2 architecture improvements
- Create architecture diagrams (8 Mermaid diagrams)
- Write architecture documentation
- Design complete database schema (Prisma)
- Design API endpoints (40+ endpoints)

### 📄 Deliverables
- `/memory/sea-net-dev.md` - Architecture documentation (20,760 bytes)
- `/memory/sea-net-diagrams.md` - Architecture diagrams (18,081 bytes)

### 🎯 Next Steps (Sprint 1 Day 2)
- Initialize Tauri + React + Vite project
- Setup shadcn/ui components
- Setup NestJS backend
- Setup Prisma + PostgreSQL
- Implement authentication
- Implement user management

---

*Last updated: 2026-02-03 15:00*
*Author: Senior Fullstack AI Agent*
