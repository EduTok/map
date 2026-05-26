# 🏛️ EduTok Complete System Architecture

This document provides a production-grade map of the entire EduTok platform. It covers the frontend layers, backend services, **PostgreSQL** database schemas, cryptographic implementations, user roles, APIs, and the operational flows between the Student/Teacher portal and the **Escolas Panel**.

---

## 1. High-Level Component & Crypto Architecture
This diagram outlines the complete technology stack, specific encryption algorithms, and how the React frontend interfaces with the Express Node.js Server and PostgreSQL persistence layer.

```mermaid
graph TD
    %% Base styling
    classDef client fill:#3b82f6,color:#ffffff,stroke:#2563eb,stroke-width:2px,rx:8px,ry:8px
    classDef server fill:#0ea5e9,color:#ffffff,stroke:#0284c7,stroke-width:2px,rx:8px,ry:8px
    classDef storage fill:#8b5cf6,color:#ffffff,stroke:#6d28d9,stroke-width:2px,rx:8px,ry:8px
    classDef cache fill:#10b981,color:#ffffff,stroke:#059669,stroke-width:2px,rx:8px,ry:8px
    classDef security fill:#ef4444,color:#ffffff,stroke:#b91c1c,stroke-width:2px,rx:8px,ry:8px
    classDef boundary fill:none,stroke:#cbd5e1,stroke-width:2px,stroke-dasharray: 5 5,rx:12px,ry:12px

    subgraph Frontend["Frontend (Client Portal)"]
        React["React + Vite + Wouter"]:::client
        ReactQuery["React Query (State)"]:::client
        Tailwind["Vanilla CSS + Lucide Icons"]:::client
        IndexedDB[("IndexedDB (Offline Cache)")]:::cache
    end

    subgraph Backend["Backend (Express Node.js Server)"]
        API["Express.js Server (/api/local/*)"]:::server
        WSEfeed["WebSocket Server (/ws/efeed)"]:::server
        WSExt["WebSocket Extension Bridge (/ws/extension)"]:::server
        dbStore["dbStore (In-Memory Collection Cache)"]:::server
        
        subgraph Crypto["Cryptography (Node 'crypto')"]
            AES["AES-256-GCM Encryption"]:::security
            SHA["SHA-256 Password Hashing"]:::security
            GenID["Crypto Random Bytes ID"]:::security
        end
    end

    subgraph Persistence["Persistence (PostgreSQL Database)"]
        Postgres[("PostgreSQL Database Store")]:::storage
    end

    React -- "REST (JSON API)" --> API
    React <--"Realtime Updates / Chat"--> WSEfeed
    React -- "Offline Files Storage" --> IndexedDB

    API -- "Sensitive Fields" --> AES
    dbStore -- "Autosave Sync (Interval)" --> Postgres
    API -- "Auth Verification" --> SHA

    API -- "SQL Queries / Collection Save" --> dbStore
    WSEfeed -- "Broadcasting events" --> API
    WSExt -- "API Key Bridge (apiKey)" --> API
    
    class Frontend,Backend,Persistence,Crypto boundary
```

---

## 2. Operational Integration Flow: Central Portal & Escolas Panel
The relationship between the administrative **Escolas Panel** (School Management) and the Student/Teacher/Parent panels. Because both panels consume the same backend API and share a single PostgreSQL persistent store, any administrative change flows instantly down to the portal.

```mermaid
sequenceDiagram
    autonumber
    participant Admin as 🏫 Escolas Panel (Diretor/Coordenador)
    participant Server as 🖥️ Central Node.js Server
    participant DB as 🐘 PostgreSQL Database
    participant WS as 📡 WebSocket Broadcaster
    participant Teacher as 👨‍🏫 Teacher Portal
    participant Student as 📚 Student/Parent Portal

    %% Class Creation & Assignment Flow
    Note over Admin, Student: 1. Operational Automation: Turma & Teacher Configuration
    Admin->>Server: POST /api/local/classes (turma, schedules, teacher assignments)
    Server->>DB: Save to PostgreSQL
    Server->>WS: Broadcast event ("calendar:event:created" / "attendance:updated")
    WS-->>Teacher: Active classes and subjects list dynamically updated
    WS-->>Student: Student calendar instantly populates with weekly lessons & schedules

    %% Grades Submission Flow
    Note over Teacher, Student: 2. Academic Performance Flow
    Teacher->>Server: POST /api/local/grades (studentId, subject, grade, term)
    Server->>DB: Save & recalculate averages (AES-256-GCM Encrypted)
    Server->>WS: Broadcast Event ("grade:created")
    WS-->>Student: Student dashboard & Parent notifications update instantly

    %% Attendance Flow (Subject and Slot Scoped)
    Note over Teacher, Student: 3. Daily Attendance Flow (Time & Subject Scoped)
    Teacher->>Server: POST /api/local/attendance (marks slot-specific presence)
    Server->>DB: Save daily attendance & increment student absences
    Teacher->>Server: POST /api/local/attendance/finalize (subject + date finalization)
    Server->>DB: Record finalized status
    Server->>WS: Broadcast event ("attendance:updated")
    WS-->>Student: Calendar icons instantly toggle from "Sem Registro" to "Presente" / "Faltou"
```

---

## 3. User Roles & Access Hierarchy
EduTok utilizes five strict user roles, granting or restricting access to specific application panels.

```mermaid
graph TD
    classDef roleAdmin fill:#334155,color:#fff,stroke:#1e293b,stroke-width:2px,rx:50px,ry:50px
    classDef roleProf fill:#8b5cf6,color:#fff,stroke:#6d28d9,stroke-width:2px,rx:8px,ry:8px
    classDef roleResp fill:#f59e0b,color:#fff,stroke:#d97706,stroke-width:2px,rx:8px,ry:8px
    classDef roleAluno fill:#10b981,color:#fff,stroke:#059669,stroke-width:2px,rx:8px,ry:8px
    classDef panel fill:#f8fafc,color:#334155,stroke:#cbd5e1,stroke-width:2px,rx:8px,ry:8px

    Secretaria(("Secretaria de Educação")):::roleAdmin
    Diretor["Diretor / Escola Manager"]:::roleAdmin

    Prof["Professor"]:::roleProf
    Resp["Responsável (Parent)"]:::roleResp
    Aluno["Aluno (Student)"]:::roleAluno

    PanelSec["Painel Central da Rede"]:::panel
    PanelEsc["Escolas Panel (Gestão)"]:::panel
    PanelT["Painel de Frequência & Notas"]:::panel
    PanelR["Boletim & Acompanhamento"]:::panel
    PanelA["Dashboard & Eduna AI Assistant"]:::panel
    PanelC["Realtime Chat Room"]:::panel

    Secretaria -->|"Manage All Schools"| PanelSec
    Diretor -->|"Manage Single School"| PanelEsc
    
    Prof -->|"Take Attendance / Grades"| PanelT
    Resp -->|"Toggle Multiple Siblings"| PanelR
    Aluno -->|"Own Performance & Eduna"| PanelA

    Prof -->|"Group Moderator"| PanelC
    Resp -->|"Contact Assigned Teachers (Requires Approval)"| PanelC
    Aluno -->|"Group & Direct Messenger"| PanelC
```

---

## 4. Production PostgreSQL Database Schema (ERD)
An Entity-Relationship Diagram mapping the core relational tables in the PostgreSQL database.

```mermaid
erDiagram
    USERS {
        string id PK "Unique UID / Hash ID"
        string role "student, teacher, parent, admin"
        string name "Full Name"
        string email "AES-256-GCM Encrypted"
        string hashed_password "SHA-256 Hashed"
        string schoolId "FK -> SCHOOLS.id"
        string schoolIds "Array of schoolIds (for teachers)"
        string gradeLevel "Class/Turma name (for students)"
    }
    SCHOOLS {
        string id PK "Unique identifier (cep-based)"
        string name "School Name"
        string address "Full address details"
        jsonb workingDays "Working days array [1,2,3,4,5]"
    }
    CLASSES {
        string id PK "Random bytes generated UID"
        string name "Turma name (e.g., 3 Reg 2)"
        string schoolId "FK -> SCHOOLS.id"
        jsonb teachers "Array of {teacherId, subjects[]}"
        jsonb schedule "Array of {dayOfWeek, startTime, endTime, subject}"
        string students "Array of studentIds"
    }
    ATTENDANCE {
        string id PK "Generated unique identifier"
        string studentId "FK -> USERS.id"
        string turmaId "Turma/grade identifier"
        string schoolId "FK -> SCHOOLS.id"
        string date "YYYY-MM-DD"
        string status "present | absent"
        string subject "Lesson subject"
        int createdAt "Epoch timestamp"
    }
    FINALIZED_ATTENDANCE {
        string schoolId PK "FK -> SCHOOLS.id"
        string turmaId PK "Turma identifier"
        string date PK "YYYY-MM-DD"
        string subject PK "Subject name"
        string teacherId "FK -> USERS.id"
        int finalizedAt "Epoch timestamp"
    }
    GRADES {
        string id PK "Generated unique identifier"
        string studentId "FK -> USERS.id"
        string subject "Subject name"
        float grade "Assigned score"
        string term "Bimestre (1st, 2nd, etc)"
        string teacherId "FK -> USERS.id"
        string schoolId "FK -> SCHOOLS.id"
        int createdAt "Epoch timestamp"
    }
    MESSAGES {
        string id PK "Generated message UID"
        string roomId "Room or turma identifier"
        string senderId "FK -> USERS.id"
        string content "Message text content"
        jsonb attachment "AES-256-GCM encrypted metadata + path"
        int createdAt "Epoch timestamp"
    }

    SCHOOLS ||--o{ USERS : "houses"
    SCHOOLS ||--o{ CLASSES : "contains"
    USERS ||--o{ ATTENDANCE : "registers presence"
    SCHOOLS ||--o{ ATTENDANCE : "manages daily presence"
    CLASSES ||--o{ ATTENDANCE : "measures attendance"
    USERS ||--o{ GRADES : "receives marks"
    USERS ||--o{ MESSAGES : "sends"
```

---

## 5. Ephemeral Media & WhatsApp-Style 10-Day Retention Flow
This sequence diagram details how media (images/documents/videos) are uploaded, securely encrypted using **AES-256-GCM**, cached offline in the client, and purged automatically on the central server after 10 days to guarantee privacy and fast recovery.

```mermaid
sequenceDiagram
    autonumber
    participant Sender as 📱 Sender App (React)
    participant Auth as 🔒 Server Cryptography
    participant Server as 🖥️ Express Node.js Server
    participant WS as 📡 WebSocket Gateway
    participant DB as 🐘 PostgreSQL (JSONB)
    participant Receiver as 📱 Receiver App (React)
    participant IndexedDB as 💾 Receiver Local IndexedDB

    %% Upload & Encryption Flow
    Sender->>Server: POST /messages + binary upload
    Server->>Auth: Encrypt file & payload with AES-256-GCM
    Server->>DB: INSERT INTO messages (attachment data, key metadata)
    
    %% Realtime WebSocket Delivery
    Server->>WS: broadcast("chat:message", {id, encrypted_attachment})
    WS-->>Receiver: Receive WebSocket message
    Receiver->>IndexedDB: Decrypt and cache media in local persistent browser database

    %% Server Media Purging (10-Day TTL)
    Note over Server, DB: Server TTL: Files older than 10 days are auto-deleted
    Server->>DB: DELETE /api/media/purge (files > 10 days old)
    Server->>DB: UPDATE messages SET attachment = {"purged": true}
    
    %% Offline Sync & Retrieval
    Note over Receiver, Server: User reconnects after 5 days
    Receiver->>Server: GET /api/messages (sync)
    Server-->>Receiver: Return messages (Attachments still intact on server)
    Receiver->>IndexedDB: Decrypt and save locally

    Note over Receiver, Server: User reconnects after 12 days
    Receiver->>Server: GET /api/messages (sync)
    Server-->>Receiver: Return messages (attachment.purged = true)
    Receiver->>IndexedDB: Query local offline cache
    alt File exists in local IndexedDB
        Receiver->>Receiver: Load and display media locally from cache
    else File missing (not cached)
        Receiver->>Receiver: Show overlay "Mídia expirada (Mais de 10 dias)"
    end
```

---

## 6. EduGen & AI Tutor Scoring Pipeline (Eduzão Leaderboard)
The gamification loop: how student AI interactions dynamically feed the municipal scoreboard.

```mermaid
sequenceDiagram
    autonumber
    participant Aluno as 📱 Aluno App (React)
    participant Tutor as 🤖 EduGen AI API
    participant AI as 🧠 Eduna LLM Context Engine
    participant Score as 🏆 Eduzão Score Engine
    participant DB as 🐘 PostgreSQL Database

    Aluno->>Tutor: Request personalized exercises (Subject + Topic)
    Tutor->>DB: Fetch student profile, grades history & weaknesses (AES-256-GCM Decrypted)
    Tutor->>AI: Generate customized pedagogical prompt & content
    AI-->>Tutor: Deliver 5 highly targeted questions + explanations
    Tutor-->>Aluno: Display dynamically generated exercise
    
    Aluno->>Tutor: Submit solved answers
    Tutor->>AI: Evaluate answers & generate detailed explanation
    AI-->>Tutor: Delivery correction results
    Tutor->>DB: Save progress to edugen_sessions
    
    %% Leaderboard update
    Tutor->>Score: Trigger points event (Answers correct: +50 XP)
    Score->>DB: UPDATE user.totalScore = totalScore + 50
    Score-->>Aluno: Broadcast leaderboard socket update ("Subiu no Ranking!")
```
