# Component diagram

```mermaid
graph TB
    subgraph api["⚙️ Backend API - Spring Boot Monolith"]

        subgraph controllerLayer["Controller Layer"]
            clanCtrl["ClanController
            Spring RestController
            Manajemen klan, keanggotaan,
            dan detail klan"]

            leagueCtrl["LeagueController
            Spring RestController
            Leaderboard divisi
            dan reset season liga"]
        end

        subgraph serviceLayer["Service Layer"]
            clanSvc["ClanService
            Spring Service
            Pembuatan klan, validasi
            join/leave, integrasi repo"]

            leagueSvc["LeagueService
            Spring Service
            Perhitungan skor kolektif
            dan rotasi divisi antar season"]
        end

        subgraph specialLayer["Special Components"]
            resolver["ClanScoreProviderResolver
            Spring Component
            Strategy Pattern: memilih logika
            perhitungan skor per divisi
            Bronze / Silver / Gold / Diamond"]

            handler["GlobalExceptionHandler
            RestControllerAdvice
            Memetakan exception ke format
            API response yang konsisten
            HTTP 400 / 403 / 404"]
        end

        subgraph repoLayer["Repository Layer · JPA"]
            clanRepo[("ClanRepository
            JPA Interface
            Abstraksi akses data
            entitas Clan")]

            memberRepo[("ClanMemberRepository
            JPA Interface
            Abstraksi akses data
            entitas ClanMember")]
        end

    end

    db[("PostgreSQL Database
    Relational Database
    Tabel: clans, clan_members")]

    %% Controller → Service
    clanCtrl   -->|"Menggunakan"| clanSvc
    leagueCtrl -->|"Menggunakan"| leagueSvc

    %% Controller → Exception Handler
    clanCtrl   -->|"Error via Advice"| handler
    leagueCtrl -->|"Error via Advice"| handler

    %% ClanService → Repositories
    clanSvc -->|"CRUD klan"| clanRepo
    clanSvc -->|"CRUD anggota"| memberRepo

    %% LeagueService → Resolver & Repositories
    leagueSvc -->|"Meminta provider skor"| resolver
    leagueSvc -->|"Membaca data klan"| clanRepo
    leagueSvc -->|"Membaca data anggota"| memberRepo

    %% Repositories → Database
    clanRepo   -->|"SQL/JDBC"| db
    memberRepo -->|"SQL/JDBC"| db
```

### Justifikasi
Diagram komponen ini merupakan dekomposisi dari container Backend API. Modul Clan dirancang dengan prinsip Separation of Concerns, di mana ClanService berfokus pada manajemen entitas klan, sementara LeagueService menangani fitur kompetitif. Implementasi ClanScoreProviderResolver menunjukkan penggunaan Strategy Pattern untuk menghitung skor secara dinamis. Seluruh komunikasi data keluar masuk modul difasilitasi oleh folder dto untuk menjaga enkapsulasi entitas database, dan manajemen error dipusatkan pada GlobalExceptionHandler.

# Code Diagram

## Class Diagram - DTO Pattern & Core Domain
```mermaid
graph TB
    subgraph api[" Backend API - Clan Module"]

        subgraph controllers["Controller Layer"]
            clanCtrl["ClanController
            Spring RestController
            Menerima request dan mengembalikan
            ClanResponse / ClanMemberResponse"]
        end

        subgraph dtos["DTO Layer"]
            subgraph request["Request DTOs"]
                createReq["CreateClanRequest
                DTO / Record
                userId, name 3-30 char,
                description max 200"]

                joinReq["JoinClanRequest
                DTO / Record
                userId [NotNull]"]
            end

            subgraph response["Response DTOs"]
                clanResp["ClanResponse
                DTO / Record
                id, name, division,
                memberCount, leaderUserId"]

                memberResp["ClanMemberResponse
                DTO / Record
                userId, role"]
            end
        end

        subgraph services["Service Layer"]
            clanSvc["ClanService
            Spring Service
            Mengelola logika bisnis:
            buat klan, join, leave, validasi role"]
        end

        subgraph entities["Entity Layer"]
            clanEntity["Clan
            JPA Entity
            id, name, description,
            leaderUserId, division, createdAt"]

            memberEntity["ClanMember
            JPA Entity
            id, userId, clanId FK,
            role LEADER/MEMBER, joinedAt"]
        end

        subgraph error["Error Handling"]
            exHandler["GlobalExceptionHandler
            RestControllerAdvice
            ClanNotFoundException 404
            UserAlreadyInClan 400
            Unauthorized 403"]
        end

    end

    clanCtrl -->|"Consumes"| createReq
    clanCtrl -->|"Consumes"| joinReq
    clanCtrl -->|"Produces"| clanResp
    clanCtrl -->|"Produces"| memberResp
    clanCtrl -->|"Delegates to"| clanSvc
    clanSvc  -->|"Manages via JPA"| clanEntity
    clanSvc  -->|"Manages via JPA"| memberEntity
    clanCtrl -->|"Errors handled by"| exHandler
    clanSvc  -->|"Throws to"| exHandler
```

### Justifikasi
Diagram ini menunjukkan penerapan Data Transfer Object (DTO) untuk menjaga enkapsulasi entitas database (Clan dan ClanMember). Dengan menggunakan DTO, modul memastikan bahwa perubahan pada struktur database tidak akan merusak kontrak API dengan frontend. Relasi komposisi antara Clan dan ClanMember memastikan integritas data dalam hubungan satu-ke-banyak.

## Class Diagram - League Strategy Pattern

```mermaid
graph TB
    subgraph api["Backend API - League Module"]
        leagueSvc["LeagueService
        Spring Service
        Mengorkestrasi kalkulasi skor musiman,
        promosi, dan demosi divisi antar season"]

        resolver["ClanScoreProviderResolver
        Spring Component
        Menerima nama divisi dan mengembalikan
        ClanScoreProvider yang sesuai via providerMap"]

        iface["ClanScoreProvider
        Java Interface
        Kontrak kalkulasi skor: getDivision(): String
        dan calculateScore(List~MemberStat~): double"]

        bronze["BronzeScoreProvider
        Spring Component
        Kalkulasi dasar berbasis totalScore"]

        silver["SilverScoreProvider
        Spring Component
        totalScore + bobot accuracy"]

        gold["GoldScoreProvider
        Spring Component
        Weighted formula dengan multiplier"]

        diamond["DiamondScoreProvider
        Spring Component
        ELO-style formula dengan winRate"]

        memberStat["MemberStat
        Java Record
        userId, totalScore: int,
        accuracy: double, winRate: double"]
    end

    leagueSvc -->|"Memanggil resolve(division)"| resolver
    resolver -->|"Mengembalikan implementasi via providerMap"| iface

    bronze -->|"implements"| iface
    silver -->|"implements"| iface
    gold -->|"implements"| iface
    diamond -->|"implements"| iface

    iface -->|"Memproses List~MemberStat~"| memberStat
```

### Justifikasi
Implementasi Strategy Pattern digunakan untuk menangani kalkulasi skor yang berbeda di tiap divisi (Bronze hingga Diamond). Dengan menggunakan ClanScoreProviderResolver, sistem dapat menentukan algoritma perhitungan secara dinamis pada saat runtime. Desain ini mematuhi Open-Closed Principle, di mana divisi baru dapat ditambahkan hanya dengan membuat class provider baru tanpa mengubah kode logic yang sudah ada.

## Sequence Diagram - Join Clan Flow
```mermaid
sequenceDiagram
    actor User
    participant CC as ClanController<br/>@RestController
    participant CS as ClanService<br/>@Service
    participant CMR as ClanMemberRepository<br/>@Repository
    participant CR as ClanRepository<br/>@Repository
    participant EH as GlobalExceptionHandler<br/>@RestControllerAdvice

    User->>CC: POST /clans/{clanId}/join<br/>body: JoinClanRequest { userId }
    activate CC

    CC->>CS: joinClan(userId, clanId)
    activate CS

    Note right of CS: Validasi 1: cek apakah<br/>user sudah punya klan

    CS->>CMR: existsByUserId(userId)
    activate CMR
    CMR-->>CS: boolean
    deactivate CMR

    alt User already has a clan
        CS->>EH: throw UserAlreadyInClanException
        activate EH
        EH-->>CC: ResponseEntity<br/>HTTP 400 Bad Request<br/>{ "error": "User already in a clan" }
        deactivate EH
        CC-->>User: HTTP 400 Bad Request

    else User is eligible
        Note right of CS: Validasi 2: pastikan<br/>klan dengan clanId ada

        CS->>CR: findById(clanId)
        activate CR
        CR-->>CS: Optional~Clan~
        deactivate CR

        alt Clan not found
            CS->>EH: throw ClanNotFoundException
            activate EH
            EH-->>CC: ResponseEntity<br/>HTTP 404 Not Found<br/>{ "error": "Clan not found" }
            deactivate EH
            CC-->>User: HTTP 404 Not Found

        else Clan exists
            Note right of CS: Buat entitas ClanMember baru<br/>dengan role default: MEMBER

            CS->>CMR: save(new ClanMember)<br/>{ userId, clanId, role: MEMBER, joinedAt: now() }
            activate CMR
            CMR-->>CS: ClanMember (savedEntity)
            deactivate CMR

            CS-->>CC: ClanMemberResponse<br/>{ userId, role: MEMBER }
            CC-->>User: HTTP 200 OK<br/>{ userId, role: MEMBER }
        end
    end

    deactivate CS
    deactivate CC
```

### Justifikasi
Diagram ini menggambarkan alur kerja dinamis dan validasi bisnis saat pengguna mencoba bergabung dengan klan. Proses ini melibatkan pengecekan status keanggotaan pada ClanMemberRepository sebelum melakukan persistensi data. Hal ini menunjukkan koordinasi antar komponen yang sinkron untuk memastikan seorang pengguna tidak dapat memiliki lebih dari satu klan secara bersamaan.

## Sequence Diagram - Exception Handling Architecture

```mermaid
classDiagram
    class GlobalExceptionHandler {
        <<RestControllerAdvice>>
        +handleClanNotFound(ClanNotFoundException) ResponseEntity~ErrorResponse~
        +handleUserAlreadyInClan(UserAlreadyInClanException) ResponseEntity~ErrorResponse~
        +handleUserNotInClan(UserNotInClanException) ResponseEntity~ErrorResponse~
        +handleForbidden(UnauthorizedClanActionException) ResponseEntity~ErrorResponse~
        -buildErrorResponse(int status, String message) ErrorResponse
    }

    class ErrorResponse {
        <<Record>>
        +int status
        +String error
        +String message
        +Instant timestamp
    }

    class ClanBaseException {
        <<abstract>>
        -String message
        +ClanBaseException(String message)
    }

    class ClanNotFoundException {
        <<RuntimeException>>
        +ClanNotFoundException(Long clanId)
    }

    class UserAlreadyInClanException {
        <<RuntimeException>>
        +UserAlreadyInClanException(Long userId)
    }

    class UserNotInClanException {
        <<RuntimeException>>
        +UserNotInClanException(Long userId)
    }

    class UnauthorizedClanActionException {
        <<RuntimeException>>
        +UnauthorizedClanActionException(Long userId)
    }

    ClanBaseException <|-- ClanNotFoundException : extends
    ClanBaseException <|-- UserAlreadyInClanException : extends
    ClanBaseException <|-- UserNotInClanException : extends
    ClanBaseException <|-- UnauthorizedClanActionException : extends

    GlobalExceptionHandler ..> ClanNotFoundException : catches HTTP 404
    GlobalExceptionHandler ..> UserAlreadyInClanException : catches HTTP 400
    GlobalExceptionHandler ..> UserNotInClanException : catches HTTP 400
    GlobalExceptionHandler ..> UnauthorizedClanActionException : catches HTTP 403
    GlobalExceptionHandler ..> ErrorResponse : produces
```

### Justifikasi
Menunjukkan arsitektur penanganan error terpusat menggunakan @RestControllerAdvice. Dengan menangkap custom exceptions (seperti ClanNotFoundException) secara global, modul ini memisahkan logika penanganan error dari logika bisnis utama. Hal ini meningkatkan keterbacaan kode dan memastikan klien API selalu menerima respon HTTP yang konsisten (404, 400, atau 403).