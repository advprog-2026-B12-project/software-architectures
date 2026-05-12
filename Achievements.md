# Achievements Module

Bagian ini memaparkan dokumentasi arsitektur spesifik untuk modul **Achievements** yang dikerjakan secara individu.

## 1. Component / Container Diagram
Diagram berikut menunjukkan bagaimana modul Achievements menerima *request* dari Frontend maupun modul lain (seperti Quiz dan Auth untuk men-trigger *event*), serta memproses logika gamifikasi dan berinteraksi dengan basis data.

```mermaid
graph TB
    subgraph monolith ["Modul Achievements (Spring Boot)"]
        direction TB
        CTRL["AchievementController<br/>DailyMissionController<br/><i>(REST Endpoint)</i>"]
        SVC["AchievementServiceImpl<br/>DailyMissionServiceImpl<br/><i>(Business Logic)</i>"]
        REPO["AchievementRepository<br/>UserAchievementRepository<br/>DailyMissionRepository<br/>UserDailyMissionRepository<br/><i>(Data Access)</i>"]
    end
    
    FE["🌐 Next.js Frontend"]
    DB[("🗄️ PostgreSQL Database")]
    Other["🔄 Other Modules<br/><i>(Quiz, Auth)</i>"]
    
    FE -->|"GET /api/achievements<br/>REST API"| CTRL
    Other -->|"POST /api/achievements/trigger<br/>Event Triggers"| CTRL
    CTRL -->|"Method Call"| SVC
    SVC -->|"Method Call"| REPO
    REPO -->|"JDBC / JPA"| DB
    
    style monolith fill:#FDF6E3,color:#000,stroke:#D97706
    style CTRL fill:#FDE68A,color:#000,stroke:#D97706
    style SVC fill:#FCD34D,color:#000,stroke:#D97706
    style REPO fill:#FBBF24,color:#000,stroke:#B45309
    style DB fill:#3ECF8E,color:#fff,stroke:#22C55E
    style FE fill:#0EA5E9,color:#fff,stroke:#0284C7
    style Other fill:#9CA3AF,color:#fff,stroke:#4B5563
```

## 2. Code Diagrams (Class Diagrams)

Berikut adalah beberapa *Code Diagram* (Class Diagram) yang membedah arsitektur internal dari kode di modul Achievements.

### 2.1 Model & Entities Diagram
Diagram ini memetakan entitas basis data yang digunakan untuk menyimpan master data *achievement* dan *daily mission*, serta tabel *mapping* ke *user*.

```mermaid
classDiagram
    class Achievement {
        -UUID id
        -String name
        -String description
        -String iconUrl
        -int targetScore
        -String eventTrigger
    }
    class UserAchievement {
        -UUID id
        -User user
        -Achievement achievement
        -int currentScore
        -boolean isCompleted
        -LocalDateTime completedAt
    }
    class DailyMission {
        -UUID id
        -String title
        -String description
        -int targetAmount
        -int rewardPoints
    }
    class UserDailyMission {
        -UUID id
        -User user
        -DailyMission mission
        -int currentAmount
        -boolean isCompleted
        -LocalDate date
    }
    
    UserAchievement --> Achievement : "Belongs to"
    UserDailyMission --> DailyMission : "Belongs to"
```

### 2.2 Repository Diagram
Diagram ini menunjukkan pemanfaatan Spring Data JPA untuk abstraksi operasi basis data.

```mermaid
classDiagram
    class AchievementRepository {
        <<interface>>
        +Optional~Achievement~ findByName(String name)
    }
    class UserAchievementRepository {
        <<interface>>
        +List~UserAchievement~ findByUser(User user)
        +Optional~UserAchievement~ findByUserAndAchievement(User, Achievement)
    }
    class DailyMissionRepository {
        <<interface>>
        +List~DailyMission~ findAll()
    }
    class UserDailyMissionRepository {
        <<interface>>
        +List~UserDailyMission~ findByUserAndDate(User user, LocalDate date)
    }
    
    AchievementRepository --|> JpaRepository
    UserAchievementRepository --|> JpaRepository
    DailyMissionRepository --|> JpaRepository
    UserDailyMissionRepository --|> JpaRepository
```

### 2.3 Service Layer Diagram
Bagian utama tempat berjalannya proses gamifikasi, pengecekan *progress*, dan validasi pembukaan suatu *achievement*.

```mermaid
classDiagram
    class AchievementService {
        <<interface>>
        +List~AchievementResponse~ getAllAchievements()
        +AchievementResponse createAchievement(AchievementRequest)
        +EventTriggerResponse processEvent(EventTriggerRequest)
        +List~AchievementProgressResponse~ getUserProgress(UUID userId)
    }
    class DailyMissionService {
        <<interface>>
        +List~DailyMissionResponse~ getDailyMissions()
        +DailyMissionResponse createDailyMission(DailyMissionRequest)
        +void checkMissionProgress(UUID userId, String eventType)
    }
    class AchievementServiceImpl {
        -AchievementRepository achievementRepo
        -UserAchievementRepository userAchievementRepo
    }
    class DailyMissionServiceImpl {
        -DailyMissionRepository missionRepo
        -UserDailyMissionRepository userMissionRepo
    }
    
    AchievementServiceImpl ..|> AchievementService
    DailyMissionServiceImpl ..|> DailyMissionService
```

### 2.4 Controller & DTO Diagram
Diagram untuk lapisan *presentation* (API endpoint) dan *Data Transfer Object* yang membatasi data yang dikirim dan diterima oleh API.

```mermaid
classDiagram
    class AchievementController {
        -AchievementService achievementService
        +ResponseEntity getAllAchievements()
        +ResponseEntity createAchievement()
        +ResponseEntity triggerEvent()
        +ResponseEntity getUserProgress()
    }
    class DailyMissionController {
        -DailyMissionService dailyMissionService
        +ResponseEntity getDailyMissions()
        +ResponseEntity createDailyMission()
    }
    
    class EventTriggerRequest {
        +UUID userId
        +String eventType
        +int amount
    }
    class AchievementResponse {
        +UUID id
        +String name
        +String description
        +String iconUrl
    }
    
    AchievementController --> AchievementService : "Calls"
    DailyMissionController --> DailyMissionService : "Calls"
    AchievementController ..> EventTriggerRequest : "Uses"
    AchievementController ..> AchievementResponse : "Uses"
```

## 3. Penjelasan Refleksi Arsitektur Individu

Secara keseluruhan, modul **Achievements** mengadopsi _Layered Architecture_ (Controller, Service, Repository) yang sangat umum dalam Spring Boot. Pemisahan antarmuka (interface) di level Service (`AchievementService` dan `DailyMissionService`) memungkinkan pengujian (*unit testing*) dengan struktur _Mocking_ yang lebih mudah serta menjaga kelenturan seandainya implementasi dari validasi kondisi *achievement* berubah sewaktu-waktu.

Pada arsitektur saat ini (Monolith), modul-modul lain (seperti _Quiz_ dan _Auth_) dapat men-trigger *gamification events* melalui REST endpoint (`/trigger`) atau pemanggilan internal langsung jika ada _direct coupling_. Namun, seiring dengan rekomendasi _Risk Storming_, desain ini sudah cukup siap dimigrasikan menuju _Event-Driven Architecture_ (EDA). Nantinya `AchievementController` tidak lagi dipanggil melalui REST internal antar-modul, melainkan digantikan dengan **Event Subscriber** yang mendengarkan `quiz.completed` atau `user.registered` dari _Message Broker_.
