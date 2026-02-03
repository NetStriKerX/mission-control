# sea-net-desktop-v2 - Architecture Diagrams

## Mermaid Diagrams

### 1. High-Level System Architecture
```mermaid
graph TB
    subgraph "Desktop App"
        subgraph Frontend["Frontend Layer"]
            UI["UI Components<br/>(shadcn/ui)"]
            State["State Management<br/>(Zustand + TanStack Query)"]
            Forms["React Hook Form + Zod"]
            Auth["Auth Module"]
            Customers["Customers Module"]
            Bookings["Bookings Module"]
            Shipments["Shipments Module"]
            Tasks["Tasks Module"]
            Docs["Documents Module"]
            Dash["Dashboard Module"]
        end

        subgraph Desktop["Desktop Integration"]
            FS["File System"]
            Notify["Notifications"]
            Updater["Auto-updater"]
        end

        UI --> State
        State --> Auth
        State --> Customers
        State --> Bookings
        State --> Shipments
        State --> Tasks
        State --> Docs
        State --> Dash

        State --> Forms
        Forms --> State
        State --> FS
        State --> Notify
        State --> Updater
    end

    subgraph "Backend Server"
        subgraph API["API Layer"]
            AuthCtrl["Auth Controller"]
            CustCtrl["Customers Controller"]
            BookCtrl["Bookings Controller"]
            ShipCtrl["Shipments Controller"]
            TaskCtrl["Tasks Controller"]
            DocCtrl["Documents Controller"]
            TempCtrl["Templates Controller"]
            DashCtrl["Dashboard Controller"]
        end

        subgraph Services["Business Logic"]
            AuthSvc["Auth Service"]
            CustSvc["Customers Service"]
            BookSvc["Bookings Service"]
            ShipSvc["Shipments Service"]
            TaskSvc["Tasks Service"]
            DocSvc["Documents Service"]
            TempSvc["Templates Service"]
            DashSvc["Dashboard Service"]
        end

        subgraph Data["Data Access"]
            Prisma["Prisma Client"]
            Repos["Repositories"]
        end

        AuthCtrl --> AuthSvc
        CustCtrl --> CustSvc
        BookCtrl --> BookSvc
        ShipCtrl --> ShipSvc
        TaskCtrl --> TaskSvc
        DocCtrl --> DocSvc
        TempCtrl --> TempSvc
        DashCtrl --> DashSvc

        AuthSvc --> Prisma
        CustSvc --> Prisma
        BookSvc --> Prisma
        ShipSvc --> Prisma
        TaskSvc --> Prisma
        DocSvc --> Prisma
        TempSvc --> Prisma
        DashSvc --> Prisma

        Prisma --> Repos
    end

    subgraph "Database"
        PG[(PostgreSQL)]
    end

    Auth -->|HTTP| AuthCtrl
    Customers -->|HTTP| CustCtrl
    Bookings -->|HTTP| BookCtrl
    Shipments -->|HTTP| ShipCtrl
    Tasks -->|HTTP| TaskCtrl
    Docs -->|HTTP| DocCtrl
    Dash -->|HTTP| DashCtrl

    FS -->|IPC| State

    Repos -->|SQL| PG
```

### 2. Authentication Flow
```mermaid
sequenceDiagram
    participant User as User
    participant UI as React UI
    participant Store as Zustand Auth Store
    participant API as Backend API
    participant DB as PostgreSQL

    User->>UI: Enter credentials
    UI->>Store: Call login(email, password)
    Store->>API: POST /api/auth/login
    API->>DB: Validate user
    DB-->>API: User data
    API->>API: Generate JWT tokens
    API-->>Store: { accessToken, refreshToken }
    Store->>Store: Store tokens
    Store->>UI: Redirect to dashboard
    UI->>UI: Show welcome message
```

### 3. Booking Creation Flow
```mermaid
sequenceDiagram
    participant User as User
    participant Form as React Hook Form
    participant Query as TanStack Query
    participant API as Backend API
    participant Service as BookingService
    participant DB as Prisma/PostgreSQL

    User->>Form: Fill booking form
    Form->>Form: Validate with Zod
    Form->>Query: POST /api/bookings
    Query->>API: Create booking
    API->>Service: create(bookingData)
    Service->>DB: Start transaction
    DB->>DB: Create Booking
    DB->>DB: Create Containers
    DB->>DB: Create Shipment
    DB->>DB: Create AuditLog
    DB-->>Service: Success
    Service-->>API: Created booking
    API-->>Query: Booking data
    Query->>Query: Update cache
    Query-->>Form: Success
    Form->>User: Redirect to detail view
```

### 4. Database Schema Relationships
```mermaid
erDiagram
    User ||--o{ Customer : "creates"
    User ||--o{ Booking : "creates"
    User ||--o{ Task : "assigned"
    User ||--o{ AuditLog : "performs"

    Customer ||--o{ Booking : "has"

    Booking ||--o{ Container : "contains"
    Booking ||--o{ Shipment : "has"
    Booking ||--o{ Document : "has"

    Shipment ||--o{ TrackingEvent : "tracks"

    User {
        string id PK
        string email UK
        string passwordHash
        string name
        UserRole role
        boolean isActive
        datetime createdAt
        datetime updatedAt
        datetime lastLoginAt
    }

    Customer {
        string id PK
        string name
        string email
        string phone
        string address
        string notes
        boolean isActive
        datetime createdAt
        datetime updatedAt
        string createdById FK
    }

    Booking {
        string id PK
        string bookingNo UK
        string customerId FK
        string carrier
        string vessel
        string voyage
        string pol
        string pod
        datetime etd
        datetime createdAt
        datetime updatedAt
        string createdById FK
    }

    Container {
        string id PK
        string bookingId FK
        string containerNo
        string type
        string size
        float weight
        datetime createdAt
        datetime updatedAt
    }

    Shipment {
        string id PK
        string bookingId FK
        ShipmentStatus status
        string notes
        string currentLoc
        datetime createdAt
        datetime updatedAt
    }

    TrackingEvent {
        string id PK
        string shipmentId FK
        string status
        string location
        string notes
        datetime timestamp
    }

    Task {
        string id PK
        string title
        string description
        TaskPriority priority
        TaskStatus status
        datetime dueDate
        string assignedTo FK
        datetime createdAt
        datetime updatedAt
    }

    Document {
        string id PK
        string bookingId FK
        string name
        string fileName
        string filePath
        string fileType
        int fileSize
        datetime createdAt
    }

    Template {
        string id PK
        string name
        string type
        string fileName
        string description
        datetime createdAt
        datetime updatedAt
    }

    AuditLog {
        string id PK
        string userId FK
        AuditAction action
        string entity
        string entityId
        json details
        string ipAddress
        string userAgent
        datetime createdAt
    }
```

### 5. State Management Architecture
```mermaid
graph LR
    subgraph "UI Components"
        C1["CustomerList"]
        C2["BookingForm"]
        C3["TaskBoard"]
        C4["Dashboard"]
    end

    subgraph "Client State (Zustand)"
        AuthStore["Auth Store<br/>- user<br/>- token<br/>- isAuthenticated"]
        ThemeStore["Theme Store<br/>- theme<br/>- sidebar"]
        ModalStore["Modal Store<br/>- isOpen<br/>- content"]
        NotificationStore["Notification Store<br/>- toasts"]
    end

    subgraph "Form State (React Hook Form)"
        F1["Customer Form"]
        F2["Booking Form"]
        F3["Task Form"]
    end

    subgraph "Server State (TanStack Query)"
        Q1["useCustomers"]
        Q2["useBookings"]
        Q3["useShipments"]
        Q4["useTasks"]
        Q5["useDashboard"]
    end

    subgraph "API Layer"
        API["REST API<br/>(NestJS)"]
    end

    C1 --> AuthStore
    C2 --> AuthStore
    C3 --> AuthStore
    C4 --> AuthStore

    C1 --> ThemeStore
    C2 --> ThemeStore
    C3 --> ThemeStore
    C4 --> ThemeStore

    C1 --> ModalStore
    C2 --> ModalStore
    C3 --> ModalStore
    C4 --> ModalStore

    C1 --> NotificationStore
    C2 --> NotificationStore
    C3 --> NotificationStore
    C4 --> NotificationStore

    C1 --> Q1
    C2 --> Q2
    C3 --> Q4
    C4 --> Q5

    F1 --> C2
    F2 --> C2
    F3 --> C3

    Q1 --> API
    Q2 --> API
    Q3 --> API
    Q4 --> API
    Q5 --> API
```

### 6. Module Structure
```mermaid
graph TB
    subgraph "src/"
        subgraph "frontend/"
            subgraph "modules/"
                AuthM["auth/"]
                CustM["customers/"]
                BookM["bookings/"]
                ShipM["shipments/"]
                TaskM["tasks/"]
                DocM["documents/"]
                DashM["dashboard/"]
            end

            subgraph "shared/"
                Comp["components/"]
                Hooks["hooks/"]
                Stores["stores/"]
                API["api/"]
                Utils["utils/"]
                Types["types/"]
            end

            App["app/"]
        end

        subgraph "backend/"
            subgraph "modules/"
                AuthN["auth/"]
                CustN["customers/"]
                BookN["bookings/"]
                ShipN["shipments/"]
                TaskN["tasks/"]
                DocN["documents/"]
                TempN["templates/"]
                DashN["dashboard/"]
            end

            subgraph "shared/"
                Guards["guards/"]
                Decorators["decorators/"]
                Interceptors["interceptors/"]
                Filters["filters/"]
                Pipes["pipes/"]
                Utils["utils/"]
            end

            Config["config/"]
        end

        subgraph "desktop/"
            Tauri["src-tauri/"]
        end
    end

    AuthM --> AuthN
    CustM --> CustN
    BookM --> BookN
    ShipM --> ShipN
    TaskM --> TaskN
    DocM --> DocN
    DashM --> DashN
```

### 7. API Request Flow
```mermaid
graph TB
    Start([User Action]) --> UI[React Component]
    UI --> Check{Need Form?}
    Check -->|Yes| Form[React Hook Form + Zod]
    Check -->|No| Direct[TanStack Query]
    Form --> Valid{Valid?}
    Valid -->|No| UI
    Valid -->|Yes| Direct
    Direct --> Request[HTTP Request]
    Request --> Router[React Router]
    Router --> Axios[Axios Interceptor]
    Axios --> AddToken{Has Token?}
    AddToken -->|Yes| AddHeader[Add Authorization]
    AddToken -->|No| Request
    AddHeader --> Request
    Request --> Backend[NestJS API]
    Backend --> AuthCheck{Auth Required?}
    AuthCheck -->|Yes| Guard[AuthGuard]
    AuthCheck -->|No| Controller
    Guard --> ValidToken{Valid?}
    ValidToken -->|No| Error[401 Unauthorized]
    ValidToken -->|Yes| Controller
    Controller --> Service[Service Layer]
    Service --> Repo[Repository]
    Repo --> DB[(PostgreSQL)]
    DB --> Repo
    Repo --> Service
    Service --> Controller
    Controller --> Response[JSON Response]
    Error --> UI
    Response --> Axios
    Axios --> Query[TanStack Query]
    Query --> Cache[Update Cache]
    Cache --> UI
```

### 8. File System Structure
```
sea-net-desktop-v2/
├── frontend/                    # React + Vite
│   ├── src/
│   │   ├── modules/
│   │   │   ├── auth/
│   │   │   │   ├── components/
│   │   │   │   ├── hooks/
│   │   │   │   ├── types.ts
│   │   │   │   └── index.ts
│   │   │   ├── customers/
│   │   │   │   ├── components/
│   │   │   │   │   ├── CustomerList.tsx
│   │   │   │   │   ├── CustomerForm.tsx
│   │   │   │   │   └── CustomerDetail.tsx
│   │   │   │   ├── hooks/
│   │   │   │   │   ├── useCustomers.ts
│   │   │   │   │   └── useCustomer.ts
│   │   │   │   ├── types.ts
│   │   │   │   └── index.ts
│   │   │   ├── bookings/
│   │   │   │   ├── components/
│   │   │   │   │   ├── BookingList.tsx
│   │   │   │   │   ├── BookingForm.tsx
│   │   │   │   │   ├── BookingDetail.tsx
│   │   │   │   │   └── ContainerForm.tsx
│   │   │   │   ├── hooks/
│   │   │   │   │   ├── useBookings.ts
│   │   │   │   │   └── useBooking.ts
│   │   │   │   ├── types.ts
│   │   │   │   └── index.ts
│   │   │   ├── shipments/
│   │   │   │   ├── components/
│   │   │   │   │   ├── ShipmentList.tsx
│   │   │   │   │   ├── ShipmentDetail.tsx
│   │   │   │   │   ├── TrackingTimeline.tsx
│   │   │   │   │   └── StatusUpdateForm.tsx
│   │   │   │   ├── hooks/
│   │   │   │   │   ├── useShipments.ts
│   │   │   │   │   └── useShipment.ts
│   │   │   │   ├── types.ts
│   │   │   │   └── index.ts
│   │   │   ├── tasks/
│   │   │   │   ├── components/
│   │   │   │   │   ├── TaskBoard.tsx
│   │   │   │   │   ├── TaskForm.tsx
│   │   │   │   │   └── TaskCard.tsx
│   │   │   │   ├── hooks/
│   │   │   │   │   ├── useTasks.ts
│   │   │   │   │   └── useTask.ts
│   │   │   │   ├── types.ts
│   │   │   │   └── index.ts
│   │   │   ├── documents/
│   │   │   │   ├── components/
│   │   │   │   │   ├── DocumentList.tsx
│   │   │   │   │   ├── UploadForm.tsx
│   │   │   │   │   └── DocumentPreview.tsx
│   │   │   │   ├── hooks/
│   │   │   │   │   ├── useDocuments.ts
│   │   │   │   │   └── useDocument.ts
│   │   │   │   ├── types.ts
│   │   │   │   └── index.ts
│   │   │   ├── templates/
│   │   │   │   ├── components/
│   │   │   │   │   ├── TemplateList.tsx
│   │   │   │   │   ├── TemplateForm.tsx
│   │   │   │   │   └── TemplatePreview.tsx
│   │   │   │   ├── hooks/
│   │   │   │   │   ├── useTemplates.ts
│   │   │   │   │   └── useTemplate.ts
│   │   │   │   ├── types.ts
│   │   │   │   └── index.ts
│   │   │   └── dashboard/
│   │   │       ├── components/
│   │   │       │   ├── StatsCards.tsx
│   │   │       │   ├── RecentActivity.tsx
│   │   │       │   ├── BookingChart.tsx
│   │   │       │   └── ShipmentStatusChart.tsx
│   │   │       ├── hooks/
│   │   │       │   ├── useDashboard.ts
│   │   │       │   └── useStats.ts
│   │   │       ├── types.ts
│   │   │       └── index.ts
│   │   ├── shared/
│   │   │   ├── components/
│   │   │   │   ├── ui/           # shadcn/ui components
│   │   │   │   ├── layout/
│   │   │   │   │   ├── Header.tsx
│   │   │   │   │   ├── Sidebar.tsx
│   │   │   │   │   └── Layout.tsx
│   │   │   │   └── common/
│   │   │   │       ├── Button.tsx
│   │   │   │       ├── Input.tsx
│   │   │   │       ├── DataTable.tsx
│   │   │   │       ├── SearchInput.tsx
│   │   │   │       ├── LoadingSpinner.tsx
│   │   │   │       └── ErrorBoundary.tsx
│   │   │   ├── hooks/
│   │   │   │   ├── useAuth.ts
│   │   │   │   ├── useToast.ts
│   │   │   │   ├── useModal.ts
│   │   │   │   └── useKeyboardShortcuts.ts
│   │   │   ├── stores/
│   │   │   │   ├── authStore.ts
│   │   │   │   ├── themeStore.ts
│   │   │   │   ├── modalStore.ts
│   │   │   │   └── notificationStore.ts
│   │   │   ├── api/
│   │   │   │   ├── axios.ts
│   │   │   │   ├── endpoints.ts
│   │   │   │   └── client.ts
│   │   │   ├── utils/
│   │   │   │   ├── date.ts
│   │   │   │   ├── format.ts
│   │   │   │   ├── validation.ts
│   │   │   │   └── export.ts
│   │   │   └── types/
│   │   │       ├── global.ts
│   │   │       ├── api.ts
│   │   │       └── pagination.ts
│   │   └── app/
│   │       ├── App.tsx
│   │       ├── main.tsx
│   │       ├── router.tsx
│   │       └── styles.css
│   ├── index.html
│   ├── vite.config.ts
│   ├── tailwind.config.js
│   └── package.json
│
├── backend/                     # NestJS
│   ├── src/
│   │   ├── modules/
│   │   │   ├── auth/
│   │   │   │   ├── auth.controller.ts
│   │   │   │   ├── auth.service.ts
│   │   │   │   ├── auth.module.ts
│   │   │   │   ├── dto/
│   │   │   │   ├── guards/
│   │   │   │   └── strategies/
│   │   │   ├── customers/
│   │   │   │   ├── customers.controller.ts
│   │   │   │   ├── customers.service.ts
│   │   │   │   ├── customers.module.ts
│   │   │   │   └── dto/
│   │   │   ├── bookings/
│   │   │   │   ├── bookings.controller.ts
│   │   │   │   ├── bookings.service.ts
│   │   │   │   ├── bookings.module.ts
│   │   │   │   └── dto/
│   │   │   ├── shipments/
│   │   │   │   ├── shipments.controller.ts
│   │   │   │   ├── shipments.service.ts
│   │   │   │   ├── shipments.module.ts
│   │   │   │   └── dto/
│   │   │   ├── tasks/
│   │   │   │   ├── tasks.controller.ts
│   │   │   │   ├── tasks.service.ts
│   │   │   │   ├── tasks.module.ts
│   │   │   │   └── dto/
│   │   │   ├── documents/
│   │   │   │   ├── documents.controller.ts
│   │   │   │   ├── documents.service.ts
│   │   │   │   ├── documents.module.ts
│   │   │   │   └── dto/
│   │   │   ├── templates/
│   │   │   │   ├── templates.controller.ts
│   │   │   │   ├── templates.service.ts
│   │   │   │   ├── templates.module.ts
│   │   │   │   └── dto/
│   │   │   └── dashboard/
│   │   │       ├── dashboard.controller.ts
│   │   │       ├── dashboard.service.ts
│   │   │       ├── dashboard.module.ts
│   │   │       └── dto/
│   │   ├── shared/
│   │   │   ├── guards/
│   │   │   │   ├── jwt-auth.guard.ts
│   │   │   │   ├── roles.guard.ts
│   │   │   │   └── resource.guard.ts
│   │   │   ├── decorators/
│   │   │   │   ├── roles.decorator.ts
│   │   │   │   └── public.decorator.ts
│   │   │   ├── interceptors/
│   │   │   │   ├── logging.interceptor.ts
│   │   │   │   └── transform.interceptor.ts
│   │   │   ├── filters/
│   │   │   │   ├── all-exceptions.filter.ts
│   │   │   │   └── validation-exception.filter.ts
│   │   │   ├── pipes/
│   │   │   │   └── validation.pipe.ts
│   │   │   └── utils/
│   │   ├── config/
│   │   │   ├── database.config.ts
│   │   │   ├── jwt.config.ts
│   │   │   └── app.config.ts
│   │   ├── app.module.ts
│   │   └── main.ts
│   ├── prisma/
│   │   ├── schema.prisma
│   │   └── migrations/
│   ├── nest-cli.json
│   └── package.json
│
├── desktop/                     # Tauri
│   ├── src-tauri/
│   │   ├── src/
│   │   │   ├── lib.rs
│   │   │   └── main.rs
│   │   ├── Cargo.toml
│   │   ├── tauri.conf.json
│   │   └── icons/
│   └── src/
│
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── cd.yml
│
├── README.md
├── .gitignore
└── docker-compose.yml
```

---

*Generated: 2026-02-03*
*Author: Senior Fullstack AI Agent*
