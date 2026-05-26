# 🏛️ Arquitetura Completa do Sistema EduTok

Este documento apresenta um mapa técnico de nível profissional de toda a plataforma **EduTok**. Ele cobre as camadas de frontend, serviços backend, esquemas do banco de dados **PostgreSQL**, implementações criptográficas, papéis de usuários, APIs e os fluxos operacionais entre o portal de Alunos/Professores e o **Painel das Escolas**.

---

## 1. Arquitetura de Alto Nível dos Componentes & Criptografia

Este diagrama demonstra toda a stack tecnológica da plataforma, os algoritmos específicos de criptografia e como o frontend em React se comunica com o servidor Express Node.js e com a camada de persistência PostgreSQL.

```mermaid
graph TD
    %% Base styling
    classDef client fill:#3b82f6,color:#ffffff,stroke:#2563eb,stroke-width:2px,rx:8px,ry:8px
    classDef server fill:#0ea5e9,color:#ffffff,stroke:#0284c7,stroke-width:2px,rx:8px,ry:8px
    classDef storage fill:#8b5cf6,color:#ffffff,stroke:#6d28d9,stroke-width:2px,rx:8px,ry:8px
    classDef cache fill:#10b981,color:#ffffff,stroke:#059669,stroke-width:2px,rx:8px,ry:8px
    classDef security fill:#ef4444,color:#ffffff,stroke:#b91c1c,stroke-width:2px,rx:8px,ry:8px
    classDef boundary fill:none,stroke:#cbd5e1,stroke-width:2px,stroke-dasharray: 5 5,rx:12px,ry:12px

    subgraph Frontend["Frontend (Portal do Cliente)"]
        React["React + Vite + Wouter"]:::client
        ReactQuery["React Query (Gerenciamento de Estado)"]:::client
        Tailwind["CSS Vanilla + Ícones Lucide"]:::client
        IndexedDB[("IndexedDB (Cache Offline)")]:::cache
    end

    subgraph Backend["Backend (Servidor Express Node.js)"]
        API["Servidor Express.js (/api/local/*)"]:::server
        WSEfeed["Servidor WebSocket (/ws/efeed)"]:::server
        WSExt["Bridge WebSocket da Extensão (/ws/extension)"]:::server
        dbStore["dbStore (Cache de Coleções em Memória)"]:::server
        
        subgraph Crypto["Criptografia (Node 'crypto')"]
            AES["Criptografia AES-256-GCM"]:::security
            SHA["Hash de Senhas SHA-256"]:::security
            GenID["IDs Gerados com Crypto Random Bytes"]:::security
        end
    end

    subgraph Persistence["Persistência (Banco PostgreSQL)"]
        Postgres[("Banco de Dados PostgreSQL")]:::storage
    end

    React -- "REST (JSON API)" --> API
    React <--"Atualizações em Tempo Real / Chat"--> WSEfeed
    React -- "Armazenamento Offline de Arquivos" --> IndexedDB

    API -- "Campos Sensíveis" --> AES
    dbStore -- "Sincronização Automática (Intervalo)" --> Postgres
    API -- "Verificação de Autenticação" --> SHA

    API -- "Consultas SQL / Persistência" --> dbStore
    WSEfeed -- "Transmissão de Eventos" --> API
    WSExt -- "Bridge por API Key (apiKey)" --> API
    
    class Frontend,Backend,Persistence,Crypto boundary
```

---

## 2. Fluxo de Integração Operacional: Portal Central & Painel das Escolas

A relação entre o **Painel das Escolas** (gestão administrativa) e os painéis de alunos, professores e responsáveis. Como todos os ambientes utilizam a mesma API backend e compartilham o mesmo banco PostgreSQL, qualquer alteração administrativa é refletida instantaneamente nos portais.

```mermaid
sequenceDiagram
    autonumber
    participant Admin as 🏫 Painel das Escolas (Diretor/Coordenação)
    participant Server as 🖥️ Servidor Central Node.js
    participant DB as 🐘 Banco PostgreSQL
    participant WS as 📡 Broadcast WebSocket
    participant Teacher as 👨‍🏫 Portal do Professor
    participant Student as 📚 Portal do Aluno/Responsável

    %% Class Creation & Assignment Flow
    Note over Admin, Student: 1. Automação Operacional: Configuração de Turmas & Professores
    Admin->>Server: POST /api/local/classes (turma, horários, professores)
    Server->>DB: Salvar no PostgreSQL
    Server->>WS: Disparar evento ("calendar:event:created" / "attendance:updated")
    WS-->>Teacher: Lista de turmas e matérias atualizada dinamicamente
    WS-->>Student: Calendário do aluno preenchido instantaneamente com horários e aulas

    %% Grades Submission Flow
    Note over Teacher, Student: 2. Fluxo de Desempenho Acadêmico
    Teacher->>Server: POST /api/local/grades (studentId, subject, grade, term)
    Server->>DB: Salvar e recalcular médias (Criptografado com AES-256-GCM)
    Server->>WS: Disparar evento ("grade:created")
    WS-->>Student: Dashboard do aluno e notificações dos responsáveis atualizadas instantaneamente

    %% Attendance Flow
    Note over Teacher, Student: 3. Fluxo Diário de Frequência (Por Horário e Disciplina)
    Teacher->>Server: POST /api/local/attendance (registro de presença por aula)
    Server->>DB: Salvar frequência diária e incrementar faltas
    Teacher->>Server: POST /api/local/attendance/finalize (finalização da chamada)
    Server->>DB: Registrar status finalizado
    Server->>WS: Disparar evento ("attendance:updated")
    WS-->>Student: Ícones do calendário alternam instantaneamente entre "Sem Registro", "Presente" ou "Faltou"
```

---

## 3. Papéis de Usuário & Hierarquia de Acesso

O EduTok utiliza cinco níveis rigorosos de permissões, garantindo que cada usuário acesse apenas os recursos adequados ao seu papel.

```mermaid
graph TD
    classDef roleAdmin fill:#334155,color:#fff,stroke:#1e293b,stroke-width:2px,rx:50px,ry:50px
    classDef roleProf fill:#8b5cf6,color:#fff,stroke:#6d28d9,stroke-width:2px,rx:8px,ry:8px
    classDef roleResp fill:#f59e0b,color:#fff,stroke:#d97706,stroke-width:2px,rx:8px,ry:8px
    classDef roleAluno fill:#10b981,color:#fff,stroke:#059669,stroke-width:2px,rx:8px,ry:8px
    classDef panel fill:#f8fafc,color:#334155,stroke:#cbd5e1,stroke-width:2px,rx:8px,ry:8px

    Secretaria(("Secretaria de Educação")):::roleAdmin
    Diretor["Diretor / Gestor Escolar"]:::roleAdmin

    Prof["Professor"]:::roleProf
    Resp["Responsável"]:::roleResp
    Aluno["Aluno"]:::roleAluno

    PanelSec["Painel Central da Rede"]:::panel
    PanelEsc["Painel das Escolas (Gestão)"]:::panel
    PanelT["Painel de Frequência & Notas"]:::panel
    PanelR["Boletim & Acompanhamento"]:::panel
    PanelA["Dashboard & Assistente IA Eduna"]:::panel
    PanelC["Chat em Tempo Real"]:::panel

    Secretaria -->|"Gerencia Todas as Escolas"| PanelSec
    Diretor -->|"Gerencia Uma Escola"| PanelEsc
    
    Prof -->|"Lança Frequência & Notas"| PanelT
    Resp -->|"Alterna Entre Filhos"| PanelR
    Aluno -->|"Desempenho & Eduna"| PanelA

    Prof -->|"Moderador do Grupo"| PanelC
    Resp -->|"Contato com Professores (com aprovação)"| PanelC
    Aluno -->|"Mensagens em Grupo & Diretas"| PanelC
```

---

## 4. Estrutura do Banco PostgreSQL (ERD)

Diagrama Entidade-Relacionamento representando as principais tabelas relacionais do banco PostgreSQL.

```mermaid
erDiagram
    USERS {
        string id PK "UID único / ID hash"
        string role "student, teacher, parent, admin"
        string name "Nome completo"
        string email "Criptografado com AES-256-GCM"
        string hashed_password "Hash SHA-256"
        string schoolId "FK -> SCHOOLS.id"
        string schoolIds "Array de schoolIds (professores)"
        string gradeLevel "Nome da turma (alunos)"
    }

    SCHOOLS {
        string id PK "Identificador único"
        string name "Nome da escola"
        string address "Endereço completo"
        jsonb workingDays "Dias letivos [1,2,3,4,5]"
    }

    CLASSES {
        string id PK "UID gerado aleatoriamente"
        string name "Nome da turma"
        string schoolId "FK -> SCHOOLS.id"
        jsonb teachers "Array de {teacherId, subjects[]}"
        jsonb schedule "Array de horários e disciplinas"
        string students "Array de studentIds"
    }

    ATTENDANCE {
        string id PK "Identificador único"
        string studentId "FK -> USERS.id"
        string turmaId "Identificador da turma"
        string schoolId "FK -> SCHOOLS.id"
        string date "YYYY-MM-DD"
        string status "present | absent"
        string subject "Disciplina"
        int createdAt "Timestamp Epoch"
    }

    FINALIZED_ATTENDANCE {
        string schoolId PK "FK -> SCHOOLS.id"
        string turmaId PK "Identificador da turma"
        string date PK "YYYY-MM-DD"
        string subject PK "Disciplina"
        string teacherId "FK -> USERS.id"
        int finalizedAt "Timestamp Epoch"
    }

    GRADES {
        string id PK "Identificador único"
        string studentId "FK -> USERS.id"
        string subject "Disciplina"
        float grade "Nota atribuída"
        string term "Bimestre"
        string teacherId "FK -> USERS.id"
        string schoolId "FK -> SCHOOLS.id"
        int createdAt "Timestamp Epoch"
    }

    MESSAGES {
        string id PK "UID da mensagem"
        string roomId "Sala ou turma"
        string senderId "FK -> USERS.id"
        string content "Conteúdo da mensagem"
        jsonb attachment "Metadados criptografados AES-256-GCM"
        int createdAt "Timestamp Epoch"
    }

    SCHOOLS ||--o{ USERS : "contém"
    SCHOOLS ||--o{ CLASSES : "possui"
    USERS ||--o{ ATTENDANCE : "registra presença"
    SCHOOLS ||--o{ ATTENDANCE : "gerencia frequência"
    CLASSES ||--o{ ATTENDANCE : "mede presença"
    USERS ||--o{ GRADES : "recebe notas"
    USERS ||--o{ MESSAGES : "envia"
```

---

## 5. Fluxo de Retenção de Mídia Temporária (10 Dias)

Este diagrama mostra como imagens, documentos e vídeos são enviados, criptografados com **AES-256-GCM**, armazenados offline no navegador e removidos automaticamente do servidor após 10 dias.

```mermaid
sequenceDiagram
    autonumber
    participant Sender as 📱 Aplicativo do Remetente (React)
    participant Auth as 🔒 Criptografia do Servidor
    participant Server as 🖥️ Servidor Express Node.js
    participant WS as 📡 Gateway WebSocket
    participant DB as 🐘 PostgreSQL (JSONB)
    participant Receiver as 📱 Aplicativo do Destinatário (React)
    participant IndexedDB as 💾 IndexedDB Local

    %% Upload & Encryption Flow
    Sender->>Server: POST /messages + upload binário
    Server->>Auth: Criptografar arquivo e payload com AES-256-GCM
    Server->>DB: INSERT INTO messages (dados anexados)

    %% Realtime Delivery
    Server->>WS: broadcast("chat:message", {id, encrypted_attachment})
    WS-->>Receiver: Receber mensagem WebSocket
    Receiver->>IndexedDB: Descriptografar e salvar mídia localmente

    %% Server Purge
    Note over Server, DB: TTL do servidor: arquivos acima de 10 dias são removidos
    Server->>DB: DELETE /api/media/purge (arquivos antigos)
    Server->>DB: UPDATE messages SET attachment = {"purged": true}

    %% Offline Sync
    Note over Receiver, Server: Usuário reconecta após 5 dias
    Receiver->>Server: GET /api/messages (sincronização)
    Server-->>Receiver: Retorna mensagens com anexos disponíveis
    Receiver->>IndexedDB: Salvar localmente

    Note over Receiver, Server: Usuário reconecta após 12 dias
    Receiver->>Server: GET /api/messages (sincronização)
    Server-->>Receiver: Retorna mensagens (attachment.purged = true)
    Receiver->>IndexedDB: Consultar cache offline

    alt Arquivo existe localmente
        Receiver->>Receiver: Carregar mídia do cache local
    else Arquivo ausente
        Receiver->>Receiver: Mostrar aviso "Mídia expirada (Mais de 10 dias)"
    end
```

---

## 6. Pipeline do EduGen & Tutor Inteligente

Fluxo operacional do sistema de geração de exercícios personalizados com IA da plataforma EduTok.

```mermaid
sequenceDiagram
    autonumber
    participant Aluno as 📱 Aplicativo do Aluno (React)
    participant Tutor as 🤖 API IA EduGen
    participant AI as 🧠 Motor de Contexto Eduna LLM
    participant DB as 🐘 Banco PostgreSQL

    Aluno->>Tutor: Solicitar exercícios personalizados (matéria + tópico)
    Tutor->>DB: Buscar perfil do aluno, histórico de notas e dificuldades (AES-256-GCM Descriptografado)
    Tutor->>AI: Gerar prompt pedagógico personalizado
    AI-->>Tutor: Retornar perguntas e explicações detalhadas
    Tutor-->>Aluno: Exibir exercício gerado dinamicamente

    Aluno->>Tutor: Enviar respostas resolvidas
    Tutor->>AI: Avaliar respostas e gerar correções detalhadas
    AI-->>Tutor: Retornar resultado da correção
    Tutor->>DB: Salvar progresso em edugen_sessions
```
