## System Context Diagram

System context menunjukkan Yomu dan aktor/sistem eksternal yang berinteraksi dengannya.

```mermaid
graph TB
    subgraph ext [" "]
        Reader["👤 Reader / Learner<br/><i>Reads texts, takes quizzes,<br/>joins clans, earns achievements</i>"]
        Admin["👤 Admin<br/><i>Manages readings, quizzes,<br/>achievements, daily missions</i>"]
        Google["🔐 Google OAuth<br/><i>Identity Provider</i>"]
        Supabase["🗄️ Supabase PostgreSQL<br/><i>Managed Database (Staging/Prod)</i>"]
    end

    Yomu["📚 Yomu Platform<br/><i>Gamified Reading &amp; Literacy<br/>Learning System</i>"]

    Reader -->|"REST API<br/>(HTTPS + JWT)"| Yomu
    Admin -->|"REST API<br/>(HTTPS + JWT)"| Yomu
    Yomu -->|"Verify ID Token"| Google
    Yomu -->|"JDBC / JPA"| Supabase

    style Yomu fill:#4F46E5,color:#fff,stroke:#3730A3
    style Reader fill:#10B981,color:#fff,stroke:#059669
    style Admin fill:#F59E0B,color:#fff,stroke:#D97706
    style Google fill:#EA4335,color:#fff,stroke:#B91C1C
    style Supabase fill:#3ECF8E,color:#fff,stroke:#22C55E
```

| Element | Type | Description |
|---|---|---|
| **Reader / Learner** | Person | Pengguna utama yang membaca teks, mengerjakan kuis, bergabung dengan clan, dan melihat leaderboard |
| **Admin** | Person | Mengelola konten berupa bacaan, kuis, achievements, dan daily missions |
| **Yomu Platform** | Software System | Platform membaca gamifikasi yang sedang dianalisis |
| **Google OAuth** | External System | Identity provider SSO untuk login Google |
| **Supabase PostgreSQL** | External System | PostgreSQL terkelola yang digunakan di staging/production |

---

## Container Diagram

Menunjukkan container internal dan bagaimana kelima modul hidup dalam satu unit yang dapat di-deploy.

```mermaid
graph TB
    Reader["👤 Reader"]
    Admin["👤 Admin"]
    FE["🌐 Next.js Frontend<br/><i>:3000</i>"]

    subgraph monolith ["Spring Boot Monolith (:8080)"]
        direction TB
        AUTH["🔐 Auth Module<br/><i>Register, Login,<br/>Google SSO, JWT</i>"]
        SEC["🛡️ Security Layer<br/><i>JwtAuthFilter,<br/>JwtService,<br/>SecurityConfig</i>"]
        QUIZ["📖 Reading &amp; Quiz<br/><i>Readings, Questions,<br/>Options, QuizAttempts</i>"]
        ACH["🏆 Achievements<br/><i>Achievements,<br/>Daily Missions</i>"]
        CLAN["⚔️ Social &amp; League<br/><i>Clans, Members,<br/>Leaderboard, Seasons</i>"]
        COMMENT["💬 Discussion<br/><i>Nested Comments,<br/>Reactions</i>"]
        USER["👤 User Profile<br/><i>Update profile,<br/>Delete account</i>"]
    end

    DB[("🗄️ PostgreSQL<br/><i>Single shared DB</i>")]
    Google["🔐 Google OAuth"]

    Reader --> FE
    Admin --> FE
    FE -->|"REST/JSON"| monolith

    AUTH --> SEC
    AUTH -->|"verify token"| Google
    COMMENT -.->|"imports UserRepository"| AUTH
    COMMENT -.->|"imports ReadingRepository"| QUIZ
    USER -.->|"imports UserRepository"| AUTH
    CLAN -.->|"MemberStatProvider"| QUIZ

    AUTH --> DB
    QUIZ --> DB
    ACH --> DB
    CLAN --> DB
    COMMENT --> DB
    USER --> DB

    style monolith fill:#1E1B4B,color:#C7D2FE,stroke:#6366F1
    style AUTH fill:#6366F1,color:#fff,stroke:#4F46E5
    style SEC fill:#8B5CF6,color:#fff,stroke:#7C3AED
    style QUIZ fill:#06B6D4,color:#fff,stroke:#0891B2
    style ACH fill:#F59E0B,color:#fff,stroke:#D97706
    style CLAN fill:#EF4444,color:#fff,stroke:#DC2626
    style COMMENT fill:#10B981,color:#fff,stroke:#059669
    style USER fill:#EC4899,color:#fff,stroke:#DB2777
    style DB fill:#3ECF8E,color:#fff,stroke:#22C55E
    style FE fill:#0EA5E9,color:#fff,stroke:#0284C7
```

### Ketergantungan Antar Modul (Direct Coupling)

| From | To | Jenis Coupling | Bukti |
|---|---|---|---|
| Comment → Auth | Import `UserRepository` | Akses DB langsung | CommentServiceImpl.java |
| Comment → Quiz | Import `ReadingRepository` | Akses DB langsung | CommentServiceImpl.java |
| User → Auth | Import `UserRepository` | Akses DB langsung | UserServiceImpl.java |
| Clan/League → Quiz | `MemberStatProvider` | Interface (loose) | LeagueService.java |
| Quiz → Achievements | Belum terhubung | Event yang direncanakan | Quiz completion seharusnya memicu progress achievement |

---

## Deployment Diagram

```mermaid
graph TB
    subgraph dev ["Developer Machine"]
        IDE["💻 IDE"]
        DockerLocal["🐳 Docker Compose<br/><i>PostgreSQL :5432</i>"]
        Boot["☕ gradlew bootRun<br/><i>:8080</i>"]
        NextDev["🌐 npm run dev<br/><i>:3000</i>"]
    end

    subgraph github ["GitHub"]
        Repo["📦 Repository<br/><i>advprog-2026-B12-project/<br/>yomu-backend</i>"]
        CI["🔄 CI Pipeline<br/><i>ci.yaml — test, JaCoCo,<br/>SonarQube</i>"]
        CD["🚀 CD Pipeline<br/><i>cd.yaml — bootJar<br/>deploy.yml — Docker push</i>"]
        GHCR["📦 GHCR<br/><i>Docker Image Registry</i>"]
    end

    subgraph aws ["AWS EC2 (Production)"]
        EC2["🖥️ EC2 Instance"]
        subgraph containers ["Docker Compose on EC2"]
            AppContainer["📚 yomu-backend<br/><i>eclipse-temurin:21-jre-alpine<br/>:8080</i>"]
        end
    end

    subgraph supabase ["Supabase Cloud"]
        ProdDB[("🗄️ PostgreSQL<br/><i>Production DB</i>")]
        StagingDB[("🗄️ PostgreSQL<br/><i>Staging DB</i>")]
    end

    IDE -->|"git push"| Repo
    Repo -->|"trigger"| CI
    CI -->|"on success"| CD
    CD -->|"docker push"| GHCR
    CD -->|"SSH deploy"| EC2
    GHCR -->|"docker pull"| EC2
    AppContainer -->|"JDBC"| ProdDB
    AppContainer -->|"JDBC"| StagingDB
    Boot -->|"JDBC"| DockerLocal

    style aws fill:#FF9900,color:#fff,stroke:#E68A00
    style github fill:#24292F,color:#fff,stroke:#1B1F23
    style supabase fill:#3ECF8E,color:#fff,stroke:#22C55E
    style dev fill:#1E293B,color:#E2E8F0,stroke:#475569
```

| Layer | Teknologi | Detail |
|---|---|---|
| **Runtime** | Eclipse Temurin 21 (JRE Alpine) | Multi-stage Docker build |
| **Container Orchestration** | Docker Compose | Single EC2 instance |
| **CI** | GitHub Actions | JUnit, JaCoCo (80%), SonarQube |
| **CD** | GitHub Actions + SSH | Docker image → GHCR → EC2 pull |
| **Database** | Supabase PostgreSQL | Instansi staging/prod terpisah |
| **Frontend** | Next.js (repo terpisah) | Di-deploy secara independen |

---

## Future Container Diagram — Event-Driven Architecture (EDA)

```mermaid
graph TB
    Reader["👤 Reader"]
    Admin["👤 Admin"]
    FE["🌐 Next.js Frontend"]
    APIGW["🚪 API Gateway<br/><i>Rate limiting, routing,<br/>JWT validation</i>"]
    Google["🔐 Google OAuth"]

    subgraph broker ["Message Broker (RabbitMQ/Kafka)"]
        direction LR
        Q1["📨 quiz.completed"]
        Q2["📨 user.registered"]
        Q3["📨 reading.finished"]
        Q4["📨 season.reset"]
        Q5["📨 achievement.unlocked"]
    end

    subgraph services ["Microservices"]
        direction TB
        AUTH_SVC["🔐 Auth Service<br/><i>Own DB</i>"]
        QUIZ_SVC["📖 Reading &amp; Quiz Service<br/><i>Own DB</i>"]
        ACH_SVC["🏆 Achievement Service<br/><i>Own DB</i>"]
        CLAN_SVC["⚔️ Social &amp; League Service<br/><i>Own DB</i>"]
        COMMENT_SVC["💬 Discussion Service<br/><i>Own DB</i>"]
        USER_SVC["👤 User Service<br/><i>Own DB</i>"]
    end

    Reader --> FE
    Admin --> FE
    FE -->|HTTPS| APIGW
    APIGW --> AUTH_SVC
    APIGW --> QUIZ_SVC
    APIGW --> ACH_SVC
    APIGW --> CLAN_SVC
    APIGW --> COMMENT_SVC
    APIGW --> USER_SVC
    AUTH_SVC -->|verify| Google

    QUIZ_SVC -->|"publish"| Q1
    QUIZ_SVC -->|"publish"| Q3
    AUTH_SVC -->|"publish"| Q2
    CLAN_SVC -->|"publish"| Q4
    ACH_SVC -->|"publish"| Q5

    Q1 -->|"subscribe"| ACH_SVC
    Q1 -->|"subscribe"| CLAN_SVC
    Q3 -->|"subscribe"| ACH_SVC
    Q2 -->|"subscribe"| ACH_SVC
    Q4 -->|"subscribe"| CLAN_SVC
    Q5 -->|"subscribe"| USER_SVC

    style broker fill:#7C3AED,color:#fff,stroke:#6D28D9
    style services fill:#1E293B,color:#E2E8F0,stroke:#475569
    style APIGW fill:#F97316,color:#fff,stroke:#EA580C
    style AUTH_SVC fill:#6366F1,color:#fff,stroke:#4F46E5
    style QUIZ_SVC fill:#06B6D4,color:#fff,stroke:#0891B2
    style ACH_SVC fill:#F59E0B,color:#fff,stroke:#D97706
    style CLAN_SVC fill:#EF4444,color:#fff,stroke:#DC2626
    style COMMENT_SVC fill:#10B981,color:#fff,stroke:#059669
    style USER_SVC fill:#EC4899,color:#fff,stroke:#DB2777
```

### Key Events dalam EDA

| Event | Publisher | Subscribers | Payload |
|---|---|---|---|
| `quiz.completed` | Quiz Service | Achievement Service, Clan/League Service | `{userId, readingId, score, total, timestamp}` |
| `reading.finished` | Quiz Service | Achievement Service | `{userId, readingId, timestamp}` |
| `user.registered` | Auth Service | Achievement Service | `{userId, username, timestamp}` |
| `season.reset` | Clan Service (scheduled) | Clan Service, League Service | `{seasonId, timestamp}` |
| `achievement.unlocked` | Achievement Service | User Service (notifications) | `{userId, achievementId, name, timestamp}` |

### Database per Service

| Service | Database | Tabel Utama |
|---|---|---|
| Auth Service | `auth_db` | `users`, `roles` |
| Quiz Service | `quiz_db` | `readings`, `questions`, `options`, `quiz_attempts` |
| Achievement Service | `achievement_db` | `achievements`, `user_achievements`, `daily_missions`, `user_daily_missions` |
| Clan Service | `clan_db` | `clans`, `clan_members` |
| Comment Service | `comment_db` | `comments`, `comment_reactions` |
| User Service | `user_db` | `user_profiles` |
