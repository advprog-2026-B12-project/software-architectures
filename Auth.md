## Component Diagram

Menunjukkan komponen internal Authentication Module dan cara interaksinya.

```mermaid
graph TB
    subgraph external ["External Actors"]
        Client["🌐 Next.js Frontend<br/><i>:3000</i>"]
        GoogleAPI["🔐 Google OAuth 2.0<br/><i>googleapis.com</i>"]
        OtherModules["📦 Other Modules<br/><i>Comment, Clan, Quiz,<br/>Achievement</i>"]
    end

    subgraph authmodule ["Authentication Module"]
        direction TB

        subgraph controllers ["Controllers"]
            AC["🎮 AuthController<br/><i>/api/auth/**</i><br/><br/>POST /register<br/>POST /login<br/>POST /google"]
            UC["🎮 UserController<br/><i>/api/users/**</i><br/><br/>PUT /profile<br/>DELETE /account"]
        end

        subgraph services ["Services"]
            AS["⚙️ AuthService<br/><i>interface</i>"]
            ASI["⚙️ AuthServiceImpl<br/><i>register()<br/>login()<br/>googleLogin()</i>"]
            US["⚙️ UserService<br/><i>interface</i>"]
            USI["⚙️ UserServiceImpl<br/><i>updateProfile()<br/>deleteAccount()</i>"]
        end

        subgraph security ["Security Layer"]
            JWT["🔑 JwtService<br/><i>generateToken()<br/>extractUsername()<br/>isTokenValid()</i>"]
            FILTER["🛡️ JwtAuthenticationFilter<br/><i>doFilterInternal()<br/>Extracts Bearer token<br/>Sets SecurityContext</i>"]
            SC["⚙️ SecurityConfig<br/><i>SecurityFilterChain<br/>PasswordEncoder<br/>CORS, CSRF, Stateless</i>"]
        end

        subgraph repository ["Repository"]
            UR["🗄️ UserRepository<br/><i>JpaRepository</i><br/><br/>findByUsername()<br/>findByEmail()<br/>existsByEmail()"]
        end

        subgraph config ["Configuration"]
            GC["⚙️ GoogleConfig<br/><i>GoogleIdTokenVerifier<br/>bean</i>"]
        end

        subgraph model ["Model"]
            USER["📋 User<br/><i>Entity</i>"]
            ROLE["📋 Role<br/><i>Enum<br/>PELAJAR, ADMIN</i>"]
        end

        subgraph dtos ["DTOs"]
            REG["RegisterRequest"]
            LR["LoginRequest"]
            LRES["LoginResponse"]
            GSR["GoogleSsoResult"]
            GTR["GoogleTokenRequest"]
            UDTO["UserDto"]
            UPR["UpdateProfileRequest"]
        end
    end

    DB[("🗄️ PostgreSQL<br/><i>Supabase</i>")]

    Client -->|"POST /api/auth/register<br/>POST /api/auth/login<br/>POST /api/auth/google"| AC
    Client -->|"PUT /api/users/profile<br/>DELETE /api/users/account"| UC
    Client -.->|"Setiap request melewati filter"| FILTER

    AC -->|"delegates to"| AS
    UC -->|"delegates to"| US
    AS -.-|"implements"| ASI
    US -.-|"implements"| USI

    ASI -->|"hash/verify password"| SC
    ASI -->|"generate JWT"| JWT
    ASI -->|"verify Google token"| GC
    ASI -->|"CRUD users"| UR
    USI -->|"hash/verify password"| SC
    USI -->|"CRUD users"| UR

    FILTER -->|"extract & validate JWT"| JWT
    FILTER -->|"load user by username"| UR

    GC -->|"verify ID token via HTTPS"| GoogleAPI

    UR -->|"JPA / JDBC"| DB

    OtherModules -->|"imports UserRepository<br/>(direct coupling)"| UR

    style authmodule fill:#1E1B4B,color:#C7D2FE,stroke:#6366F1
    style controllers fill:#312E81,color:#C7D2FE,stroke:#6366F1
    style services fill:#312E81,color:#C7D2FE,stroke:#6366F1
    style security fill:#4C1D95,color:#DDD6FE,stroke:#8B5CF6
    style repository fill:#1E3A5F,color:#BAE6FD,stroke:#0EA5E9
    style config fill:#3B0764,color:#E9D5FF,stroke:#A855F7
    style model fill:#14532D,color:#BBF7D0,stroke:#22C55E
    style dtos fill:#422006,color:#FDE68A,stroke:#F59E0B
    style AC fill:#6366F1,color:#fff,stroke:#4F46E5
    style UC fill:#6366F1,color:#fff,stroke:#4F46E5
    style ASI fill:#8B5CF6,color:#fff,stroke:#7C3AED
    style USI fill:#8B5CF6,color:#fff,stroke:#7C3AED
    style JWT fill:#F59E0B,color:#fff,stroke:#D97706
    style FILTER fill:#EF4444,color:#fff,stroke:#DC2626
    style UR fill:#0EA5E9,color:#fff,stroke:#0284C7
    style DB fill:#3ECF8E,color:#fff,stroke:#22C55E
```

### Ringkasan Interaksi Komponen

| From | To | Interaksi | Protokol |
|---|---|---|---|
| Frontend | AuthController | `POST /api/auth/{register,login,google}` | REST/JSON |
| Frontend | UserController | `PUT /api/users/profile`, `DELETE /api/users/account` | REST/JSON (JWT required) |
| Setiap request | JwtAuthenticationFilter | Intercept sebelum controller | Servlet Filter |
| JwtAuthenticationFilter | JwtService | `extractUsername()`, `isTokenValid()` | Method call |
| JwtAuthenticationFilter | UserRepository | `findByUsername()` untuk load user | Method call |
| AuthController | AuthServiceImpl | `register()`, `login()`, `googleLogin()` | Method call |
| UserController | UserServiceImpl | `updateProfile()`, `deleteAccount()` | Method call |
| AuthServiceImpl | UserRepository | `findByUsername()`, `findByEmail()`, `existsByEmail()`, `save()` | JPA |
| AuthServiceImpl | JwtService | `generateToken(claims, username)` | Method call |
| AuthServiceImpl | GoogleIdTokenVerifier | `verify(idToken)` | HTTPS ke Google |
| AuthServiceImpl | PasswordEncoder | `encode()`, `matches()` | Method call |
| UserServiceImpl | UserRepository | `findByUsername()`, `save()`, `delete()` | JPA |
| UserServiceImpl | PasswordEncoder | `matches()`, `encode()` | Method call |
| Other Modules | UserRepository | Direct import (tight coupling) | JPA |

---

## Class Diagram - AuthService

```mermaid
classDiagram
    class AuthService {
        <<interface>>
        +register(RegisterRequest request) User
        +login(String usernameOrEmail, String password) LoginResponse
        +googleLogin(String idToken) GoogleSsoResult
    }

    class AuthServiceImpl {
        -UserRepository userRepository
        -PasswordEncoder passwordEncoder
        -JwtService jwtService
        -GoogleIdTokenVerifier googleIdTokenVerifier
        +register(RegisterRequest request) User
        +login(String usernameOrEmail, String password) LoginResponse
        +googleLogin(String idToken) GoogleSsoResult
        -buildUserDto(User user) UserDto
    }

    class UserRepository {
        <<interface>>
        +findByUsername(String username) Optional~User~
        +findByEmail(String email) Optional~User~
        +existsByEmail(String email) boolean
        +save(User user) User
        +delete(User user) void
    }

    class JwtService {
        -String secretKey
        -long jwtExpiration
        +extractUsername(String token) String
        +extractClaim(String token, Function claimsResolver) T
        +generateToken(String username) String
        +generateToken(Map~String,Object~ extraClaims, String username) String
        +isTokenValid(String token, String username) boolean
        -buildToken(Map~String,Object~ extraClaims, String username, long expiration) String
        -extractAllClaims(String token) Claims
        -getSignInKey() Key
    }

    class PasswordEncoder {
        <<interface>>
        +encode(CharSequence rawPassword) String
        +matches(CharSequence rawPassword, String encodedPassword) boolean
    }

    class GoogleIdTokenVerifier {
        +verify(String idTokenString) GoogleIdToken
    }

    class RegisterRequest {
        -String username
        -String email
        -String displayName
        -String password
    }

    class LoginResponse {
        -String message
        -String token
        -UserDto user
    }

    class GoogleSsoResult {
        -boolean needsRegistration
        -String message
        -String token
        -UserDto user
        -String email
        -String googleName
    }

    class UserDto {
        -UUID userId
        -String username
        -String displayName
        -String email
        -Role role
    }

    AuthService <|.. AuthServiceImpl : implements
    AuthServiceImpl --> UserRepository : uses
    AuthServiceImpl --> JwtService : uses
    AuthServiceImpl --> PasswordEncoder : uses
    AuthServiceImpl --> GoogleIdTokenVerifier : uses
    AuthServiceImpl ..> RegisterRequest : receives
    AuthServiceImpl ..> LoginResponse : returns
    AuthServiceImpl ..> GoogleSsoResult : returns
    AuthServiceImpl ..> UserDto : creates
```

---

## Class Diagram - User Entity

```mermaid
classDiagram
    class User {
        <<Entity>>
        -UUID id
        -String username
        -String email
        -String displayName
        -String password
        -Role role
        +getName() String
    }

    class Role {
        <<Enumeration>>
        PELAJAR
        ADMIN
    }

    class Principal {
        <<interface>>
        +getName() String
    }

    class JpaRepository~User, UUID~ {
        <<interface>>
        +save(User entity) User
        +findById(UUID id) Optional~User~
        +findAll() List~User~
        +delete(User entity) void
    }

    class UserRepository {
        <<interface>>
        +findByUsername(String username) Optional~User~
        +findByEmail(String email) Optional~User~
        +existsByEmail(String email) boolean
    }

    class RegisterRequest {
        -String username
        -String email
        -String displayName
        -String password
    }

    class UpdateProfileRequest {
        -String displayName
        -String username
        -String oldPassword
        -String newPassword
    }

    class UserDto {
        -UUID userId
        -String username
        -String displayName
        -String email
        -Role role
    }

    class LoginResponse {
        -String message
        -String token
        -UserDto user
    }

    class GoogleSsoResult {
        -boolean needsRegistration
        -String message
        -String token
        -UserDto user
        -String email
        -String googleName
    }

    Principal <|.. User : implements
    User --> Role : has
    JpaRepository <|-- UserRepository : extends
    UserRepository ..> User : manages

    RegisterRequest ..> User : "creates via AuthService"
    UpdateProfileRequest ..> User : "updates via UserService"
    User ..> UserDto : "mapped to"
    UserDto --* LoginResponse : embedded
    UserDto --* GoogleSsoResult : embedded
```

---

## Sequence Diagram - Google SSO Login Flow

```mermaid
sequenceDiagram
    actor User as Pengguna (Browser)
    participant FE as Next.js Frontend
    participant AC as AuthController
    participant AS as AuthServiceImpl
    participant GV as GoogleIdTokenVerifier
    participant Google as Google OAuth 2.0
    participant UR as UserRepository
    participant JWT as JwtService
    participant DB as PostgreSQL

    User->>FE: Klik "Login with Google"
    FE->>Google: Buka Google OAuth consent screen
    Google-->>FE: Kembalikan Google ID Token

    FE->>AC: POST /api/auth/google { "token": "eyJhb..." }
    AC->>AS: googleLogin(idToken)

    AS->>GV: verify(idToken)
    GV->>Google: Verifikasi tanda tangan token (HTTPS)
    Google-->>GV: Token payload

    alt Token tidak valid atau null
        GV-->>AS: return null
        AS-->>AC: throw IllegalArgumentException "Token Google tidak valid!"
        AC-->>FE: 400 Bad Request
        FE-->>User: Tampilkan pesan error
    else Token valid
        GV-->>AS: GoogleIdToken (payload)
        AS->>AS: Ekstrak email dan nama dari payload

        AS->>UR: findByEmail(email)
        UR->>DB: SELECT * FROM users WHERE email = ?

        alt User sudah ada di DB
            DB-->>UR: User record
            UR-->>AS: Optional User (present)

            AS->>JWT: generateToken({"role": "ROLE_PELAJAR"}, username)
            JWT-->>AS: JWT string "eyJ..."

            AS->>AS: buildUserDto(user)
            AS-->>AC: GoogleSsoResult {needsRegistration: false, token: "eyJ...", user: UserDto}

            AC-->>FE: 200 OK { "message": "Login berhasil", "token": "eyJ...", "user": {...} }
            FE->>FE: Simpan JWT di localStorage
            FE-->>User: Redirect ke Dashboard

        else User TIDAK ada di DB
            DB-->>UR: empty
            UR-->>AS: Optional User (empty)

            AS-->>AC: GoogleSsoResult {needsRegistration: true, email: "...", googleName: "..."}
            AC-->>FE: 200 OK { "needsRegistration": true, "email": "...", "googleName": "..." }
            FE-->>User: Tampilkan form registrasi (email dan nama sudah terisi)
        end
    end
```

---

## Sequence Diagram - JWT Authentication Flow

### Part 1: Login dan Penerbitan Token

```mermaid
sequenceDiagram
    actor User as Pengguna
    participant FE as Next.js Frontend
    participant AC as AuthController
    participant AS as AuthServiceImpl
    participant UR as UserRepository
    participant PE as PasswordEncoder (BCrypt)
    participant JWT as JwtService
    participant DB as PostgreSQL

    User->>FE: Masukkan username/email + password
    FE->>AC: POST /api/auth/login { "username": "john", "password": "secret123" }

    AC->>AS: login("john", "secret123")
    AS->>AS: Cek apakah input mengandung "@"

    alt Mengandung "@" → login dengan email
        AS->>UR: findByEmail("john@mail.com")
    else Tidak mengandung "@" → login dengan username
        AS->>UR: findByUsername("john")
    end

    UR->>DB: SELECT * FROM users WHERE ...
    DB-->>UR: User record (hashed password)
    UR-->>AS: Optional User

    AS->>PE: matches("secret123", "$2a$10$hashed...")
    PE-->>AS: true

    AS->>JWT: generateToken({"role": "ROLE_PELAJAR"}, "john")
    JWT->>JWT: buildToken() - set subject, claims, issuedAt, expiration, sign HS256
    JWT-->>AS: "eyJhbGciOi..."

    AS->>AS: buildUserDto(user)
    AS-->>AC: LoginResponse {message, token, user}

    AC-->>FE: 200 OK { "message": "Login berhasil", "token": "eyJhbGciOi...", "user": {...} }
    FE->>FE: localStorage.setItem("token", jwt)
    FE-->>User: Redirect ke Dashboard
```

### Part 2: Mengakses Protected Endpoint

```mermaid
sequenceDiagram
    actor User as Pengguna
    participant FE as Next.js Frontend
    participant SC as SecurityFilterChain
    participant FILTER as JwtAuthenticationFilter
    participant JWT as JwtService
    participant UR as UserRepository
    participant DB as PostgreSQL
    participant CTX as SecurityContextHolder
    participant CTRL as Protected Controller

    User->>FE: Klik "Update Profile"
    FE->>SC: PUT /api/users/profile Authorization: Bearer eyJhbGciOi...

    SC->>FILTER: doFilterInternal(request)
    FILTER->>FILTER: Ekstrak Authorization header
    FILTER->>FILTER: Cek startsWith("Bearer ")

    alt Tidak ada Bearer token
        FILTER->>SC: filterChain.doFilter() → continue
        SC-->>FE: 403 Forbidden
    else Bearer token ditemukan
        FILTER->>FILTER: jwt = header.substring(7)

        FILTER->>JWT: extractUsername(jwt)
        JWT->>JWT: Parse JWT dengan signing key
        JWT-->>FILTER: "john"

        FILTER->>UR: findByUsername("john")
        UR->>DB: SELECT * FROM users WHERE username = 'john'
        DB-->>UR: User record
        UR-->>FILTER: Optional User (present)

        FILTER->>JWT: isTokenValid(jwt, "john")
        JWT-->>FILTER: true

        FILTER->>FILTER: Buat UsernamePasswordAuthenticationToken
        FILTER->>CTX: setAuthentication(authToken)
        FILTER->>SC: filterChain.doFilter() → continue

        SC->>SC: authorizeHttpRequests check /api/users/** → authenticated
        SC->>CTRL: Forward request ke controller

        CTRL->>CTX: getAuthentication().getName()
        CTX-->>CTRL: "john"

        CTRL-->>FE: 200 OK { "message": "Profil berhasil diperbarui" }
        FE-->>User: Tampilkan notifikasi sukses
    end
```

---

# Bonus - Security Layer

## Component Diagram - Security Layer Internal Structure

```mermaid
graph TB
    subgraph clients ["Incoming HTTP Requests"]
        PUB["🌐 Public Request<br/><i>POST /api/auth/login<br/>POST /api/auth/register<br/>POST /api/auth/google<br/>OPTIONS /**</i>"]
        PROT["🔒 Protected Request<br/><i>PUT /api/users/profile<br/>DELETE /api/users/account<br/>GET /api/quiz/**<br/>GET /api/clan/**<br/>etc.</i>"]
    end

    subgraph securitylayer ["Security Layer"]
        direction TB

        subgraph filterchain ["SecurityFilterChain (configured by SecurityConfig)"]
            direction TB
            CORS["🌍 CorsFilter<br/><i>CorsConfig</i><br/><br/>allowedOrigins:<br/>localhost:3000<br/>yomu-frontend-neon.vercel.app<br/><br/>allowedMethods:<br/>GET, POST, PUT, DELETE, OPTIONS<br/>allowCredentials: true"]

            CSRF["🚫 CSRF<br/><i>disabled</i>"]

            SESSION["📦 Session Management<br/><i>STATELESS<br/>(no HttpSession)</i>"]

            FILTER["🛡️ JwtAuthenticationFilter<br/><i>extends OncePerRequestFilter</i><br/><br/>1. Extract Authorization header<br/>2. Parse Bearer token<br/>3. Validate via JwtService<br/>4. Load user from DB<br/>5. Set SecurityContext"]

            AUTHZ["🔐 Authorization Rules<br/><i>authorizeHttpRequests</i><br/><br/>OPTIONS /** → permitAll<br/>/api/auth/** → permitAll<br/>anyRequest → authenticated"]
        end

        JWT["🔑 JwtService<br/><i>@Service</i><br/><br/>generateToken()<br/>extractUsername()<br/>isTokenValid()<br/><br/>Algorithm: HS256<br/>Key: Base64-decoded secret<br/>Library: io.jsonwebtoken (JJWT)"]

        SC["⚙️ SecurityConfig<br/><i>@Configuration</i><br/><br/>Beans:<br/>SecurityFilterChain<br/>PasswordEncoder (BCrypt)"]

        PE["🔒 PasswordEncoder<br/><i>BCryptPasswordEncoder</i><br/><br/>encode(raw) → hash<br/>matches(raw, hash) → bool"]
    end

    subgraph dependencies ["Module Dependencies"]
        UR["🗄️ UserRepository<br/><i>findByUsername()</i>"]
        ASI["⚙️ AuthServiceImpl<br/><i>Uses JwtService to<br/>generate tokens<br/>Uses PasswordEncoder<br/>to hash/verify</i>"]
        USI["⚙️ UserServiceImpl<br/><i>Uses PasswordEncoder<br/>to hash/verify</i>"]
    end

    subgraph controllers ["Protected Controllers"]
        UC["🎮 UserController"]
        QC["🎮 QuizController"]
        CC["🎮 ClanController"]
        CMC["🎮 CommentController"]
        AchC["🎮 AchievementController"]
    end

    DB[("🗄️ PostgreSQL")]
    CTX["📋 SecurityContextHolder<br/><i>Stores Authentication<br/>for current thread</i>"]

    PUB -->|"HTTP"| CORS
    PROT -->|"HTTP"| CORS
    CORS --> CSRF
    CSRF --> SESSION
    SESSION --> FILTER
    FILTER --> AUTHZ

    FILTER -->|"extractUsername()<br/>isTokenValid()"| JWT
    FILTER -->|"findByUsername()"| UR
    FILTER -->|"setAuthentication()"| CTX
    UR --> DB

    AUTHZ -->|"/api/auth/** → pass through<br/>(no auth needed)"| ASI
    AUTHZ -->|"authenticated → forward to controller"| controllers

    ASI -->|"generateToken()"| JWT
    ASI -->|"encode() / matches()"| PE
    USI -->|"encode() / matches()"| PE

    SC -.->|"registers before<br/>UsernamePasswordAuthenticationFilter"| FILTER
    SC -.->|"creates bean"| PE

    controllers -->|"getAuthentication()<br/>.getName()"| CTX

    style securitylayer fill:#1E1B4B,color:#C7D2FE,stroke:#6366F1
    style filterchain fill:#312E81,color:#C7D2FE,stroke:#6366F1
    style CORS fill:#0EA5E9,color:#fff,stroke:#0284C7
    style CSRF fill:#64748B,color:#fff,stroke:#475569
    style SESSION fill:#64748B,color:#fff,stroke:#475569
    style FILTER fill:#EF4444,color:#fff,stroke:#DC2626
    style AUTHZ fill:#F59E0B,color:#fff,stroke:#D97706
    style JWT fill:#8B5CF6,color:#fff,stroke:#7C3AED
    style SC fill:#6366F1,color:#fff,stroke:#4F46E5
    style PE fill:#10B981,color:#fff,stroke:#059669
    style CTX fill:#EC4899,color:#fff,stroke:#DB2777
    style UR fill:#0EA5E9,color:#fff,stroke:#0284C7
```

### Request Processing Pipeline (Urutan Eksekusi)

| Step | Komponen | Yang Terjadi |
|---|---|---|
| **1** | CorsFilter (CorsConfig) | Memeriksa header `Origin` terhadap allowed origins dan menambahkan CORS response headers |
| **2** | CSRF | Dinonaktifkan karena API bersifat stateless |
| **3** | Session Management | `STATELESS`, Spring tidak pernah membuat atau menggunakan `HttpSession` |
| **4** | JwtAuthenticationFilter | Mengekstrak Bearer token, memvalidasi, memuat user, dan menetapkan `SecurityContext` |
| **5** | Authorization Rules | Memeriksa apakah endpoint bersifat publik atau memerlukan autentikasi |
| **6** | Controller | Memproses request bisnis jika sudah terotorisasi |

---

## Class Diagram - JwtService

```mermaid
classDiagram
    class JwtService {
        <<Service>>
        -String secretKey
        -long jwtExpiration
        +extractUsername(String token) String
        +extractClaim~T~(String token, Function~Claims,T~ claimsResolver) T
        +generateToken(String username) String
        +generateToken(Map~String,Object~ extraClaims, String username) String
        +isTokenValid(String token, String username) boolean
        -buildToken(Map~String,Object~ extraClaims, String username, long expiration) String
        -extractAllClaims(String token) Claims
        -getSignInKey() Key
    }

    class Claims {
        <<interface>>
        +getSubject() String
        +getExpiration() Date
        +getIssuedAt() Date
        +get(String key) Object
    }

    class Jwts {
        <<utility>>
        +builder()$ JwtBuilder
        +parserBuilder()$ JwtParserBuilder
    }

    class JwtBuilder {
        +setClaims(Map claims) JwtBuilder
        +setSubject(String sub) JwtBuilder
        +setIssuedAt(Date iat) JwtBuilder
        +setExpiration(Date exp) JwtBuilder
        +signWith(Key key, SignatureAlgorithm alg) JwtBuilder
        +compact() String
    }

    class JwtParserBuilder {
        +setSigningKey(Key key) JwtParserBuilder
        +build() JwtParser
    }

    class SignatureAlgorithm {
        <<enumeration>>
        HS256
        HS384
        HS512
        RS256
    }

    class Keys {
        <<utility>>
        +hmacShaKeyFor(byte[] bytes)$ SecretKey
    }

    class Decoders {
        <<utility>>
        +BASE64$ Decoder
    }

    class Key {
        <<interface>>
        +getAlgorithm() String
        +getEncoded() byte[]
    }

    JwtService ..> Jwts : uses builder & parser
    JwtService ..> Claims : extracts from token
    JwtService ..> SignatureAlgorithm : HS256
    JwtService ..> Keys : creates HMAC key
    JwtService ..> Decoders : decodes Base64 secret
    JwtService ..> Key : signs & verifies
    Jwts ..> JwtBuilder : creates
    Jwts ..> JwtParserBuilder : creates
```

### Tanggung Jawab Method

| Method | Visibilitas | Tujuan |
|---|---|---|
| `extractUsername(token)` | public | Mengekstrak claim `sub` (subject) yaitu username |
| `extractClaim(token, resolver)` | public | Generic extractor claim menggunakan `Function<Claims, T>` |
| `generateToken(username)` | public | Membuat JWT tanpa extra claims |
| `generateToken(extraClaims, username)` | public | Membuat JWT dengan custom claims seperti `{role: ROLE_PELAJAR}` |
| `isTokenValid(token, username)` | public | Memeriksa apakah username yang diekstrak cocok dengan username yang diberikan |
| `buildToken(extraClaims, username, exp)` | private | Core builder: menetapkan claims, subject, iat, exp, dan menandatangani dengan HS256 |
| `extractAllClaims(token)` | private | Mem-parse JWT string menjadi objek `Claims` |
| `getSignInKey()` | private | Mendekode Base64 `secretKey` menjadi objek HMAC `Key` |

---

## Class Diagram - JwtAuthenticationFilter

```mermaid
classDiagram
    class OncePerRequestFilter {
        <<abstract>>
        #doFilterInternal(HttpServletRequest, HttpServletResponse, FilterChain)*
    }

    class JwtAuthenticationFilter {
        <<Component>>
        -JwtService jwtService
        -UserRepository userRepository
        +JwtAuthenticationFilter(JwtService, UserRepository)
        #doFilterInternal(HttpServletRequest request, HttpServletResponse response, FilterChain filterChain) void
    }

    class JwtService {
        <<Service>>
        +extractUsername(String token) String
        +isTokenValid(String token, String username) boolean
    }

    class UserRepository {
        <<Repository>>
        +findByUsername(String username) Optional~User~
    }

    class User {
        <<Entity>>
        -UUID id
        -String username
        -String email
        -String displayName
        -String password
        -Role role
        +getName() String
    }

    class SecurityContextHolder {
        <<utility>>
        +getContext()$ SecurityContext
    }

    class SecurityContext {
        +getAuthentication() Authentication
        +setAuthentication(Authentication auth) void
    }

    class UsernamePasswordAuthenticationToken {
        -Object principal
        -Object credentials
        -Collection authorities
        +setDetails(Object details) void
    }

    class SimpleGrantedAuthority {
        -String role
        +SimpleGrantedAuthority(String role)
    }

    class HttpServletRequest {
        <<interface>>
        +getHeader(String name) String
    }

    OncePerRequestFilter <|-- JwtAuthenticationFilter : extends
    JwtAuthenticationFilter --> JwtService : delegates token ops
    JwtAuthenticationFilter --> UserRepository : loads user
    JwtAuthenticationFilter ..> SecurityContextHolder : sets authentication
    JwtAuthenticationFilter ..> UsernamePasswordAuthenticationToken : creates
    JwtAuthenticationFilter ..> SimpleGrantedAuthority : "ROLE_" + user.getRole()
    SecurityContextHolder ..> SecurityContext : provides
    UsernamePasswordAuthenticationToken ..> User : principal
    UserRepository ..> User : returns
```

Filter execution flow:
1. Baca header `Authorization`
2. Jika tidak ada atau tidak dimulai dengan `Bearer ` → lewati
3. `jwt = header.substring(7)`
4. `username = jwtService.extractUsername(jwt)`
5. Jika username tidak null dan belum ada auth di context: load user dari repository, validasi token, set SecurityContext
6. Exception apapun akan di-catch secara diam-diam (request tetap unauthenticated)
7. Selalu panggil `filterChain.doFilter()`

---

## Sequence Diagram - Expired Token Request Flow

```mermaid
sequenceDiagram
    actor User as Pengguna (Browser)
    participant FE as Next.js Frontend
    participant CORS as CorsFilter
    participant FILTER as JwtAuthenticationFilter
    participant JWT as JwtService
    participant JJWT as JJWT Library
    participant AUTHZ as Authorization Rules
    participant CTRL as Protected Controller

    User->>FE: Klik aksi (misal: update profil)
    FE->>FE: Baca JWT dari localStorage (token sudah lama)

    FE->>CORS: PUT /api/users/profile Authorization: Bearer eyJ...[EXPIRED]

    CORS->>CORS: Periksa Origin header → allowed
    CORS->>FILTER: Pass ke filter berikutnya

    FILTER->>FILTER: Ekstrak Authorization header
    FILTER->>FILTER: startsWith("Bearer ") → true, ambil jwt

    FILTER->>JWT: extractUsername(jwt)
    JWT->>JWT: extractAllClaims(jwt)
    JWT->>JJWT: parserBuilder().setSigningKey(key).build().parseClaimsJws(jwt)

    Note over JJWT: JJWT mem-parse token dan memeriksa<br/>claim "exp". exp lebih kecil dari waktu<br/>sekarang, token sudah expired.

    JJWT--xJWT: throw ExpiredJwtException

    JWT--xFILTER: Exception merambat ke filter

    Note over FILTER: catch (Exception e) — SecurityContext<br/>tetap kosong, tidak ada Authentication.

    FILTER->>AUTHZ: filterChain.doFilter() — lanjutkan pipeline

    AUTHZ->>AUTHZ: Periksa /api/users/profile → butuh authenticated()<br/>SecurityContext kosong → NOT authenticated

    AUTHZ-->>FE: 403 Forbidden

    FE->>FE: Deteksi respons 403
    FE->>FE: Hapus localStorage
    FE-->>User: Redirect ke halaman login
```

### Catatan Penting

| Step | Yang Terjadi |
|---|---|
| Parsing token | JJWT secara internal memeriksa claim `exp` saat `parseClaimsJws()` dipanggil |
| Exception catch | `catch (Exception e)` menelan `ExpiredJwtException` secara diam-diam |
| SecurityContext kosong | `setAuthentication()` tidak pernah dipanggil, request dianggap unauthenticated |
| Respons 403 | Spring Security menolak akses unauthenticated ke endpoint yang dilindungi |

Catatan: Implementasi saat ini mengembalikan 403 Forbidden untuk token yang expired. Perbaikan yang disarankan adalah menangkap `ExpiredJwtException` secara spesifik dan mengembalikan 401 Unauthorized dengan pesan yang jelas, sehingga frontend dapat memicu alur token refresh.

---

## State Diagram - JWT Token Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Created : generateToken() dipanggil

    Created --> Valid : iat dan exp ditetapkan

    Valid --> Valid : Token berhasil divalidasi

    Valid --> Expired : now melebihi exp

    Valid --> InvalidSignature : Signature tidak cocok

    Valid --> Malformed : Format JWT tidak valid

    Valid --> UserDeleted : User tidak ditemukan di DB

    Expired --> Rejected : 403 Forbidden

    InvalidSignature --> Rejected : 403 Forbidden

    Malformed --> Rejected : 403 Forbidden

    UserDeleted --> Rejected : 403 Forbidden

    Rejected --> [*] : Request ditolak

    state Valid {
        [*] --> CheckingBearer
        CheckingBearer --> ExtractingUsername : Bearer prefix ditemukan
        ExtractingUsername --> LoadingUser : extractUsername() berhasil
        LoadingUser --> ValidatingToken : findByUsername() berhasil
        ValidatingToken --> SettingContext : isTokenValid() true
        SettingContext --> Authenticated : setAuthentication() dipanggil
        Authenticated --> [*] : Lanjut ke controller
    }
```

### Ringkasan Transisi State Token

| State | Trigger | Exception | HTTP Result |
|---|---|---|---|
| **Created** | `AuthServiceImpl.login()` atau `googleLogin()` memanggil `generateToken()` | Tidak ada | 200 + token di response body |
| **Valid** | `now < exp` AND signature cocok AND user ada di DB | Tidak ada | Request diproses normal |
| **Expired** | `now > exp` (JJWT memeriksa saat `parseClaimsJws`) | `ExpiredJwtException` | 403 Forbidden |
| **Invalid Signature** | Token dimanipulasi atau `jwt.secret` salah | `SignatureException` | 403 Forbidden |
| **Malformed** | String token bukan format JWT valid | `MalformedJwtException` | 403 Forbidden |
| **User Deleted** | Token valid tapi user tidak ada lagi di DB | Tidak ada | 403 Forbidden |
| **Blacklisted** | Belum diimplementasikan, membutuhkan token blacklist store | Tidak ada | Belum ada |