# Component diagram

@startuml
!include https://raw.githubusercontent.com/plantuml-office/C4-PlantUML/master/C4_Component.puml

LAYOUT_WITH_LEGEND()

title Component Diagram - Clan & League Module

Container_Boundary(api, "Backend API") {
    
    ' Controllers
    Component(clanCtrl, "Clan Controller", "Spring Boot RestController", "Menangani endpoint untuk manajemen klan, keanggotaan, dan detail klan.")
    Component(leagueCtrl, "League Controller", "Spring Boot RestController", "Menangani endpoint untuk leaderboard divisi dan reset season liga.")
    
    ' Services
    Component(clanSvc, "Clan Service", "Spring Service", "Mengelola logika bisnis klan seperti pembuatan klan, validasi join/leave, dan integrasi repositori.")
    Component(leagueSvc, "League Service", "Spring Service", "Mengelola perhitungan skor klan secara kolektif dan rotasi divisi antar season.")
    
    ' Special Components
    Component(resolver, "Clan Score Resolver", "Spring Component", "Menggunakan Strategy Pattern untuk memilih logika perhitungan skor berdasarkan divisi klan.")
    Component(handler, "Global Exception Handler", "RestControllerAdvice", "Menangani dan memetakan exception modul klan ke dalam format API response yang konsisten.")
    
    ' Repositories
    ComponentDb(clanRepo, "Clan Repository", "JPA Interface", "Abstraksi akses data untuk entitas Clan.")
    ComponentDb(memberRepo, "Clan Member Repository", "JPA Interface", "Abstraksi akses data untuk entitas ClanMember.")
}

ContainerDb(db, "PostgreSQL Database", "Relational Database", "Menyimpan tabel clans dan clan_members.")

' Relationships
Rel(clanCtrl, clanSvc, "Menggunakan")
Rel(leagueCtrl, leagueSvc, "Menggunakan")

Rel(clanSvc, clanRepo, "CRUD klan")
Rel(clanSvc, memberRepo, "CRUD anggota")

Rel(leagueSvc, resolver, "Meminta provider skor")
Rel(leagueSvc, clanRepo, "Membaca data klan")
Rel(leagueSvc, memberRepo, "Membaca data anggota")

Rel(clanRepo, db, "SQL/JDBC")
Rel(memberRepo, db, "SQL/JDBC")

Rel(clanCtrl, handler, "Menangkap error via Advice")
Rel(leagueCtrl, handler, "Menangkap error via Advice")

@enduml

### Justifikasi
Diagram komponen ini merupakan dekomposisi dari container Backend API. Modul Clan dirancang dengan prinsip Separation of Concerns, di mana ClanService berfokus pada manajemen entitas klan, sementara LeagueService menangani fitur kompetitif. Implementasi ClanScoreProviderResolver menunjukkan penggunaan Strategy Pattern untuk menghitung skor secara dinamis. Seluruh komunikasi data keluar masuk modul difasilitasi oleh folder dto untuk menjaga enkapsulasi entitas database, dan manajemen error dipusatkan pada GlobalExceptionHandler.

# Code Diagram

## Class Diagram - DTO Pattern & Core Domain
@startuml
title Class Diagram: Core Domain & DTO Pattern

package "entity" {
    class Clan <<Entity>> {
        - Long id
        - String name
        - String description
        - Long leaderUserId
        - String division
        - Instant createdAt
    }
    class ClanMember <<Entity>> {
        - Long id
        - Long userId
        - Role role
        - Instant joinedAt
    }
    enum Role {
        LEADER
        MEMBER
    }
}

package "dto" {
    class CreateClanRequest {
        + Long userId
        + String name
        + String description
    }
    class ClanResponse {
        + Long id
        + String name
        + String division
        + long memberCount
    }
    class ClanMemberResponse {
        + Long userId
        + Role role
    }
}

Clan "1" *-- "many" ClanMember : contains
ClanMember +-- Role
ClanController ..> CreateClanRequest : consumes
ClanController ..> ClanResponse : produces
ClanService ..> Clan : manages
@enduml

### Justifikasi
Diagram ini menunjukkan penerapan Data Transfer Object (DTO) untuk menjaga enkapsulasi entitas database (Clan dan ClanMember). Dengan menggunakan DTO, modul memastikan bahwa perubahan pada struktur database tidak akan merusak kontrak API dengan frontend. Relasi komposisi antara Clan dan ClanMember memastikan integritas data dalam hubungan satu-ke-banyak.

## Class Diagram - League Strategy Pattern
@startuml
title Class Diagram: League Score Strategy Pattern

interface ClanScoreProvider <<Interface>> {
    + getDivision(): String
    + calculateScore(List<MemberStat>): double
}

class BronzeScoreProvider implements ClanScoreProvider
class SilverScoreProvider implements ClanScoreProvider
class GoldScoreProvider implements ClanScoreProvider
class DiamondScoreProvider implements ClanScoreProvider

class ClanScoreProviderResolver {
    - providerMap: Map<String, ClanScoreProvider>
    + resolve(String division): ClanScoreProvider
}

record MemberStat {
    Long userId
    int totalScore
    double accuracy
}

LeagueService --> ClanScoreProviderResolver : uses
ClanScoreProviderResolver o-- ClanScoreProvider : manages
ClanScoreProvider ..> MemberStat : processes
@enduml

### Justifikasi
Implementasi Strategy Pattern digunakan untuk menangani kalkulasi skor yang berbeda di tiap divisi (Bronze hingga Diamond). Dengan menggunakan ClanScoreProviderResolver, sistem dapat menentukan algoritma perhitungan secara dinamis pada saat runtime. Desain ini mematuhi Open-Closed Principle, di mana divisi baru dapat ditambahkan hanya dengan membuat class provider baru tanpa mengubah kode logic yang sudah ada.

## Sequence Diagram - Join Clan Flow
@startuml
title Sequence Diagram: Join Clan Workflow

actor User
participant ClanController
participant ClanService
participant ClanMemberRepository
participant ClanRepository

User -> ClanController : joinClan(clanId, JoinClanRequest)
activate ClanController

ClanController -> ClanService : joinClan(userId, clanId)
activate ClanService

ClanService -> ClanMemberRepository : existsByUserId(userId)
alt User already has a clan
    ClanService --[#red]> ClanController : throw UserAlreadyInClanException
else User is eligible
    ClanService -> ClanRepository : findById(clanId)
    ClanRepository --> ClanService : Optional<Clan>
    
    ClanService -> ClanMemberRepository : save(new ClanMember)
    activate ClanMemberRepository
    ClanMemberRepository --> ClanService : savedEntity
    deactivate ClanMemberRepository
    
    ClanService --> ClanController : ClanMemberResponse
end

deactivate ClanService
ClanController --> User : 200 OK / Error Response
deactivate ClanController
@enduml

### Justifikasi
Diagram ini menggambarkan alur kerja dinamis dan validasi bisnis saat pengguna mencoba bergabung dengan klan. Proses ini melibatkan pengecekan status keanggotaan pada ClanMemberRepository sebelum melakukan persistensi data. Hal ini menunjukkan koordinasi antar komponen yang sinkron untuk memastikan seorang pengguna tidak dapat memiliki lebih dari satu klan secara bersamaan.

## Sequence Diagram - Exception Handling Architecture
@startuml
title Class Diagram: Centralized Exception Handling

class GlobalExceptionHandler <<RestControllerAdvice>> {
    + handleClanNotFound(ClanNotFoundException): ResponseEntity
    + handleBadRequest(RuntimeException): ResponseEntity
    + handleForbidden(UnauthorizedClanActionException): ResponseEntity
}

package "exceptions" {
    class ClanNotFoundException <<RuntimeException>>
    class UserAlreadyInClanException <<RuntimeException>>
    class UserNotInClanException <<RuntimeException>>
    class UnauthorizedClanActionException <<RuntimeException>>
}

GlobalExceptionHandler ..> ClanNotFoundException : catches
GlobalExceptionHandler ..> UserAlreadyInClanException : catches
GlobalExceptionHandler ..> UserNotInClanException : catches
GlobalExceptionHandler ..> UnauthorizedClanActionException : catches

note right of GlobalExceptionHandler
  Maps custom exceptions to 
  proper HTTP Status Codes 
  (404, 400, 403)
end note
@enduml

### Justifikasi
Menunjukkan arsitektur penanganan error terpusat menggunakan @RestControllerAdvice. Dengan menangkap custom exceptions (seperti ClanNotFoundException) secara global, modul ini memisahkan logika penanganan error dari logika bisnis utama. Hal ini meningkatkan keterbacaan kode dan memastikan klien API selalu menerima respon HTTP yang konsisten (404, 400, atau 403).