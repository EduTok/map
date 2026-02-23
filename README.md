# 🏛️ EduTok Complete System Architecture

This document provides a highly detailed map of the entire EduTok platform. It covers the frontend layers, backend services, **PostgreSQL** database schemas, cryptographic implementations, user roles, APIs, and the Ephemeral Media Chat system.

---

## 1. High-Level Component & Crypto Architecture
This diagram outlines the complete technology stack, specific encryption algorithms, and how the React frontend interfaces with the Express Node.js Server and PostgreSQL layer.

```mermaid
graph TD
    %% Base styling
    classDef client fill:#3b82f6,color:#ffffff,stroke:#2563eb,stroke-width:2px,rx:8px,ry:8px
    classDef server fill:#0ea5e9,color:#ffffff,stroke:#0284c7,stroke-width:2px,rx:8px,ry:8px
    classDef storage fill:#8b5cf6,color:#ffffff,stroke:#6d28d9,stroke-width:2px,rx:8px,ry:8px
    classDef cache fill:#10b981,color:#ffffff,stroke:#059669,stroke-width:2px,rx:8px,ry:8px
    classDef security fill:#ef4444,color:#ffffff,stroke:#b91c1c,stroke-width:2px,rx:8px,ry:8px
    classDef boundary fill:none,stroke:#cbd5e1,stroke-width:2px,stroke-dasharray: 5 5,rx:12px,ry:12px

    subgraph Frontend["Frontend (Client Layer)"]
        React["React + Vite + Wouter"]:::client
        Tailwind["TailwindCSS / Lucide"]:::client
        IndexedDB[("IndexedDB (Media Cache)")]:::cache
    end

    subgraph Backend["Backend (Node.js/Express Server)"]
        API["Express.js REST API"]:::server
        WS["WebSocket Server (Realtime)"]:::server
        MemStore[("In-Memory Caches")]:::server
        
        subgraph Crypto["Cryptography (Node 'crypto')"]
            AES["AES-256-CBC Encryption"]:::security
            SHA["SHA-256 Hashing"]:::security
            UUID["Crypto.randomUUID"]:::security
        end
    end

    subgraph Persistence["Persistence (PostgreSQL DB)"]
        Postgres[("PostgreSQL Relational DB")]:::storage
    end

    React -- "REST (Polling & Data)" --> API
    React <--"Bi-directional Chat (WS)"--> WS
    React -- "Store/Retrieve Media" --> IndexedDB

    API -- "Data + Keys" --> AES
    AES -- "Encrypted Payloads" --> MemStore
    API -- "Passwords/Tokens" --> SHA

    API -- "INSERT / UPDATE / SELECT" --> Postgres
    WS -- "Read Local State" --> MemStore
    MemStore -- "Syncs to Main DB" --> Postgres
    
    class Frontend,Backend,Persistence,Crypto boundary
```

---

## 2. User Roles & Access Hierarchy
EduTok utilizes three strict user roles, granting or restricting access to specific application panels.

```mermaid
graph TD
    classDef roleAdmin fill:#334155,color:#fff,stroke:#1e293b,stroke-width:2px,rx:50px,ry:50px
    classDef roleProf fill:#8b5cf6,color:#fff,stroke:#6d28d9,stroke-width:2px,rx:8px,ry:8px
    classDef roleResp fill:#f59e0b,color:#fff,stroke:#d97706,stroke-width:2px,rx:8px,ry:8px
    classDef roleAluno fill:#10b981,color:#fff,stroke:#059669,stroke-width:2px,rx:8px,ry:8px
    classDef panel fill:#f8fafc,color:#334155,stroke:#cbd5e1,stroke-width:2px,rx:8px,ry:8px

    Admin(("System Admin")):::roleAdmin

    Prof["Professor"]:::roleProf
    Resp["Responsável (Parent)"]:::roleResp
    Aluno["Aluno (Student)"]:::roleAluno

    PanelT["Painel do Professor"]:::panel
    PanelR["Painel do Responsável"]:::panel
    PanelA["Painel do Aluno"]:::panel
    PanelC["Chat / Turmas"]:::panel
    PanelG["EduGen / AI Tutor"]:::panel
    PanelL["Eduzão / Leaderboard"]:::panel

    Prof -->|"Full Access"| PanelT
    Resp -->|"View Only Child"| PanelR
    Aluno -->|"Own Data"| PanelA

    Prof -->|"Moderation"| PanelC
    Resp -->|"Contact Teachers"| PanelC
    Aluno -->|"Group & DM"| PanelC

    Aluno -->|"Studies"| PanelG
    Aluno -->|"Gamification"| PanelL
```

---

## 3. PostgreSQL Database Schema (ERD)
An Entity-Relationship Diagram mapping the core relational tables in the PostgreSQL database.

```mermaid
erDiagram
    USERS {
        uuid id PK
        string role "aluno, professor, responsavel"
        string hashed_password
        jsonb lgpd_consent
        jsonb preferences
    }
    GRADE_GROUPS {
        string grade PK
        string name
    }
    MESSAGES {
        uuid id PK
        string grade FK
        uuid sender_uid FK
        text content
        jsonb attachment "AES Encrypted Base64"
        timestamp created_at
    }
    EDUGEN_SESSIONS {
        uuid session_id PK
        uuid student_uid FK
        jsonb progress
        timestamp created_at
    }
    LEADERBOARD {
        uuid user_uid PK
        int total_score
    }
    FCM_TOKENS {
        uuid user_uid PK
        string token
    }
    EDU_API_KEYS {
        string key_id PK
        uuid user_uid FK
        jsonb analytics
        jsonb logs
    }

    USERS ||--o{ MESSAGES : "sends"
    GRADE_GROUPS ||--o{ MESSAGES : "contains"
    USERS ||--o{ EDUGEN_SESSIONS : "participates"
    USERS ||--|| LEADERBOARD : "has score in"
    USERS ||--o{ FCM_TOKENS : "registers"
    USERS ||--o{ EDU_API_KEYS : "owns"
```

---

## 4. Complete Application Flow: Ephemeral Media & Chat
This details the step-by-step lifecycle of sending media (images/documents/videos), distributing it, and securely pruning it via the WhatsApp-style 10-day retention model to support offline delivery.

```mermaid
sequenceDiagram
    autonumber
    participant Sender as 📱 Sender App
    participant Auth as 🔒 Auth/Crypto
    participant Server as 🖥️ Express Server
    participant WS as 📡 WebSocket Router
    participant DB as 🐘 PostgreSQL
    participant Receiver as 📱 Receiver App
    participant IndexedDB as 💾 Receiver IndexedDB

    %% Sending Flow
    Sender->>Server: POST /messages + base64 media
    Server->>Auth: SHA-256 Hash IDs / AES Encrypt
    Server->>DB: INSERT INTO messages (content, attachment)
    
    %% Realtime Delivery
    Server->>WS: broadcast("chat:message", full_payload)
    WS-->>Receiver: Receive message via WebSocket
    Receiver->>IndexedDB: Save massive base64 media locally
    Receiver-->>Sender: Display Message + Render Medias

    %% 10-Day Retention Model 
    Note over Server, DB: Server TTL Cap explicitly set to 10 Days
    Server->>Server: setTimeout(864000000ms) executed
    Server->>DB: UPDATE messages SET attachment = '__purged__'
    
    %% Offline Recovery Flow
    Note over Receiver, Server: Offline User returns (e.g., 5 days later)
    Receiver->>Server: GET /api/messages
    Server-->>Receiver: Messages (Media intact on server)
    Receiver->>IndexedDB: Cache locally -> View Message

    Note over Receiver, Server: Offline User returns (e.g., 12 days later)
    Receiver->>Server: GET /api/messages
    Server-->>Receiver: Messages (attachment.data = "__purged__")
    Receiver->>IndexedDB: Check IndexedDB Cache
    alt Not in Cache
        Receiver->>Receiver: Show "Mídia expirada" placeholder overlay
    end
```

---

## 5. EduGen (AI Module) & Leaderboard Flow
The flow of how students interact with the AI Tutor and how actions generate points for the Eduzão gamification ladder.

```mermaid
sequenceDiagram
    autonumber
    participant Aluno as 📱 Aluno App
    participant EduGen as 🤖 EduGen API
    participant AI as 🧠 AI / LLM Engine
    participant Leaderboard as 🏆 Eduzão Score Engine
    participant DB as 🐘 PostgreSQL

    Aluno->>EduGen: Start AI Session
    EduGen->>DB: INSERT INTO edugen_sessions
    Aluno->>EduGen: Answer Questions / Submit Prompts
    EduGen->>AI: Generate Content / Analyze Answers
    AI-->>EduGen: Response
    EduGen->>DB: UPDATE edugen_sessions (progress)
    
    %% Scoring
    EduGen->>Leaderboard: Trigger Points Event (e.g., +50 XP)
    Leaderboard->>DB: UPDATE leaderboard SET total_score = total_score + 50
    Leaderboard-->>Aluno: Notification / UI Update "Subiu de Nível!"
```
