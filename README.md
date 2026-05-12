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
        CI["🔄 CI Pipeline<br/><i>ci.yaml - test, JaCoCo,<br/>SonarQube</i>"]
        CD["🚀 CD Pipeline<br/><i>cd.yaml - bootJar<br/>deploy.yml - Docker push</i>"]
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

## Future Container Diagram - Event-Driven Architecture (EDA)

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

---

## Risk Storming - Monolith di Bawah Skala Besar

Risk Storming mengevaluasi apa yang bisa salah ketika Yomu mendapatkan popularitas besar (100.000+ pengguna concurrent).

### Risk Map

```mermaid
graph LR
    subgraph high ["🔴 HIGH RISK"]
        R1["Single Point of Failure<br/><i>Satu crash mematikan semua modul</i>"]
        R2["Database Bottleneck<br/><i>Semua 5 modul berbagi<br/>satu PostgreSQL</i>"]
        R3["Tidak Bisa Scale Independen<br/><i>Beban Quiz tidak sama dengan beban Clan</i>"]
        R4["Tight Coupling<br/><i>Comment mengimport repository<br/>Quiz dan Auth secara langsung</i>"]
    end

    subgraph med ["🟡 MEDIUM RISK"]
        R5["Risiko Deployment<br/><i>Setiap perubahan me-redeploy<br/>seluruh monolith</i>"]
        R6["Cascade Failures<br/><i>Query leaderboard lambat<br/>memblokir submit kuis</i>"]
        R7["Season Reset Locks<br/><i>triggerSeasonReset() menahan<br/>transaksi DB yang panjang</i>"]
    end

    subgraph low ["🟢 LOW RISK"]
        R8["Single EC2 Instance<br/><i>Tidak ada redundansi horizontal</i>"]
        R9["Event Wiring yang Hilang<br/><i>Quiz ke Achievement belum<br/>terhubung</i>"]
    end

    style high fill:#FEE2E2,color:#991B1B,stroke:#EF4444
    style med fill:#FEF3C7,color:#92400E,stroke:#F59E0B
    style low fill:#DCFCE7,color:#166534,stroke:#22C55E
```

### Analisis Risiko Detail

| # | Risiko | Severity | Dampak | Area |
|---|---|---|---|---|
| **R1** | **Single Point of Failure** | Tinggi | Jika JVM crash atau OOM, semua fitur mati sekaligus | Availability |
| **R2** | **Shared Database Bottleneck** | Tinggi | Semua modul berebut connection pool PostgreSQL yang sama. Query N+1 leaderboard akan menghabiskan koneksi untuk submit kuis | Performance |
| **R3** | **Tidak Bisa Scale Independen** | Tinggi | Saat peak hour, traffic kuis bisa 10x traffic clan, tapi tidak bisa scale pemrosesan kuis saja | Scalability |
| **R4** | **Tight Cross-Module Coupling** | Tinggi | `CommentServiceImpl` langsung mengimport `UserRepository` dan `ReadingRepository` dari modul lain. Perubahan model Auth/Quiz merusak Comments | Maintainability |
| **R5** | **Deployment Berisiko** | Sedang | Bug fix di Comments membutuhkan redeploy seluruh aplikasi, berisiko regresi Auth/Quiz | Reliability |
| **R6** | **Cascade Failures** | Sedang | `LeagueService.getLeaderboardByDivision()` yang lambat menghabiskan thread pool dan memblokir request yang tidak terkait | Performance |
| **R7** | **Transaksi Panjang saat Season Reset** | Sedang | `triggerSeasonReset()` memproses semua 4 divisi dalam satu `@Transactional`, mengunci baris di seluruh clan | Data Integrity |
| **R8** | **Single EC2, Tanpa Redundansi** | Rendah | Saat ini masih bisa diterima untuk proyek mahasiswa, tapi fatal pada skala besar | Availability |
| **R9** | **Quiz→Achievement Belum Terhubung** | Rendah | `processEvent()` sudah ada tapi quiz completion belum memanggilnya | Correctness |

---

## Bagaimana EDA Menyelesaikan Risiko yang Teridentifikasi

| Risiko | Solusi EDA |
|---|---|
| **R1 - Single Point of Failure** | Setiap service berjalan independen. Jika Achievement Service crash, Quiz dan Auth tetap berjalan. Event yang belum diproses disimpan di broker dan di-replay saat recovery. |
| **R2 - Database Bottleneck** | Database per service menghilangkan contention. Query leaderboard pada `clan_db` tidak bisa mengganggu submit kuis pada `quiz_db`. |
| **R3 - Tidak Bisa Scale Independen** | Quiz Service bisa scale ke 10 replika saat peak hour sementara Clan Service tetap di 2 replika. |
| **R4 - Tight Coupling** | Modul hanya berkomunikasi lewat events. Comment Service subscribe ke `user.registered` untuk menyimpan cache lokal, bukan mengimport `UserRepository` secara langsung. |
| **R5 - Deployment Berisiko** | Setiap service di-deploy independen. Bug fix di Comments hanya men-deploy Comment Service saja. |
| **R6 - Cascade Failures** | Pemrosesan event secara asynchronous via broker berarti kalkulasi leaderboard yang lambat tidak bisa memblokir submit kuis. |
| **R7 - Transaksi Panjang Season Reset** | Season reset mempublish event `season.reset`. Pemrosesan dipecah menjadi transaksi kecil independen per clan. |
| **R8 - Single EC2** | Microservices secara natural di-deploy di banyak node dengan container orchestration. |
| **R9 - Quiz→Achievement Belum Terhubung** | EDA membuat ini mudah: Quiz Service publish `quiz.completed`; Achievement Service subscribe dan memanggil `processEvent()`. |

### Ringkasan: Monolith vs EDA

```mermaid
graph LR
    subgraph mono ["Saat Ini: Monolith"]
        direction TB
        M1["Satu proses = satu failure domain"]
        M2["Shared DB = contention"]
        M3["Scale semua atau tidak sama sekali"]
        M4["Direct imports = tight coupling"]
        M5["Full redeploy setiap perubahan"]
    end

    subgraph eda ["Masa Depan: EDA"]
        direction TB
        E1["Failure domain terisolasi"]
        E2["DB per service = tidak ada contention"]
        E3["Scale setiap service secara independen"]
        E4["Events = loose coupling"]
        E5["Deployment independen"]
    end

    mono -->|"migrasi"| eda

    style mono fill:#FEE2E2,color:#991B1B,stroke:#EF4444
    style eda fill:#DCFCE7,color:#166534,stroke:#22C55E
```

Strategi migrasi yang direkomendasikan adalah Strangler Fig Pattern. Ekstrak satu modul sekaligus, mulai dari Achievements karena sudah memiliki `processEvent()` yang didesain untuk konsumsi event, lalu perkenalkan broker di samping monolith dan secara bertahap alihkan traffic ke service baru.

## Individual Diagram

| [Ahmad Faiq Fawwaz Abdussalam](Auth.md) | [Sabina Maritza Moenzil](Quiz.md) | [Julius Albert Wirayuda](Achievements.md) | [Muhammad Hariz Albaari](Comments.md) | [Fadhil Daffa Putra Irawan](Clans.md) |
| -- |-----------------------------------| -- | -- | -- |
