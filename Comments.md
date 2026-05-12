## Component Diagram - Discussion Module

Menunjukkan komponen internal Discussion Module dan cara interaksinya dengan modul lain.

```mermaid
graph TB
    subgraph external ["External Actors"]
        Client["Next.js Frontend
        :3000"]
        AuthModule["Auth Module
        UserRepository
        Menyediakan data User"]
        QuizModule["Quiz Module
        ReadingRepository
        Menyediakan data Reading"]
    end

    subgraph discussionmodule ["Discussion Module"]
        direction TB

        subgraph controllers ["Controllers"]
            CC["CommentController
            /api/v1/readings/{readingId}/comments
            GET, POST, PUT, DELETE
            /api/v1/admin/comments/{commentId}"]
            RC["CommentReactionController
            /api/v1/comments/{commentId}/reactions
            POST, DELETE"]
        end

        subgraph services ["Services"]
            CS["CommentService
            interface"]
            CSI["CommentServiceImpl
            createComment()
            replyToComment()
            getCommentsByReadingId()
            updateComment()
            softDeleteComment()
            adminDeleteComment()"]
            CRS["CommentReactionService
            interface"]
            CRSI["CommentReactionServiceImpl
            addOrUpdateReaction()
            removeReaction()
            getReactionCounts()
            getUserReaction()"]
        end

        subgraph repositories ["Repositories"]
            CR["CommentRepository
            JpaRepository
            findByReadingIdOrderByCreatedAtAsc()
            findByReadingIdAndParentIsNull()
            findByParentId()"]
            CRR["CommentReactionRepository
            JpaRepository
            findByCommentId()
            findByCommentIdAndUserId()
            deleteByCommentIdAndUserId()"]
        end

        subgraph entities ["Entities"]
            CE["Comment
            Entity
            id, readingId, authorId
            parent, content
            deleted, deletedBy, deletedAt
            createdAt, updatedAt, editedAt"]
            CRE["CommentReaction
            Entity
            id, comment, userId
            reactionType, createdAt"]
            RT["ReactionType
            Enum
            UPVOTE, DOWNVOTE, FIRE
            THINKING, CLAP, SURPRISED, LOVE"]
        end

        subgraph dtos ["DTOs"]
            CREQ["CommentRequest
            content: String max 5000"]
            CRES["CommentResponse
            id, readingId, authorId, parentId
            content, deleted, timestamps
            replies, reactionCounts, myReaction"]
            RREQ["ReactionRequest
            reactionType: ReactionType"]
        end
    end

    DB[("PostgreSQL
    Supabase")]

    Client -->|"REST JSON / JWT"| CC
    Client -->|"REST JSON / JWT"| RC

    CC -->|"delegates to"| CS
    RC -->|"delegates to"| CRS
    CS -.-|"implements"| CSI
    CRS -.-|"implements"| CRSI

    CSI -->|"CRUD komentar"| CR
    CSI -->|"resolveUser()"| AuthModule
    CSI -->|"existsById(readingId)"| QuizModule
    CSI -->|"getReactionCounts(), getUserReaction()"| CRSI
    CRSI -->|"CRUD reaksi"| CRR
    CRSI -->|"resolveUser()"| AuthModule
    CRSI -->|"findById(commentId)"| CR

    CR -->|"JPA / JDBC"| DB
    CRR -->|"JPA / JDBC"| DB

    style discussionmodule fill:#1E1B4B,color:#C7D2FE,stroke:#10B981
    style controllers fill:#064E3B,color:#D1FAE5,stroke:#10B981
    style services fill:#064E3B,color:#D1FAE5,stroke:#10B981
    style repositories fill:#1E3A5F,color:#BAE6FD,stroke:#0EA5E9
    style entities fill:#14532D,color:#BBF7D0,stroke:#22C55E
    style dtos fill:#422006,color:#FDE68A,stroke:#F59E0B
    style CC fill:#10B981,color:#fff,stroke:#059669
    style RC fill:#10B981,color:#fff,stroke:#059669
    style CSI fill:#34D399,color:#fff,stroke:#059669
    style CRSI fill:#34D399,color:#fff,stroke:#059669
    style CR fill:#0EA5E9,color:#fff,stroke:#0284C7
    style CRR fill:#0EA5E9,color:#fff,stroke:#0284C7
    style DB fill:#3ECF8E,color:#fff,stroke:#22C55E
```

### Ringkasan Interaksi Komponen

| From | To | Interaksi | Keterangan |
|---|---|---|---|
| Frontend | CommentController | `GET/POST/PUT/DELETE /api/v1/readings/{readingId}/comments/**` | REST/JSON + JWT |
| Frontend | CommentReactionController | `POST/DELETE /api/v1/comments/{commentId}/reactions` | REST/JSON + JWT |
| CommentController | CommentServiceImpl | Delegasi semua operasi komentar | Method call |
| CommentReactionController | CommentReactionServiceImpl | Delegasi semua operasi reaksi | Method call |
| CommentServiceImpl | UserRepository | `resolveUser(username)` - ambil data author | Direct import (coupling) |
| CommentServiceImpl | ReadingRepository | `existsById(readingId)` - validasi reading | Direct import (coupling) |
| CommentServiceImpl | CommentReactionServiceImpl | `getReactionCounts()`, `getUserReaction()` saat build tree | Method call |
| CommentReactionServiceImpl | UserRepository | `resolveUser(username)` | Direct import (coupling) |
| CommentReactionServiceImpl | CommentRepository | `findById(commentId)` - validasi komentar ada | Method call |

---

## Class Diagram - Entities

```mermaid
classDiagram
    class Comment {
        <<Entity>>
        -UUID id
        -UUID readingId
        -UUID authorId
        -Comment parent
        -String content
        -boolean deleted
        -UUID deletedBy
        -LocalDateTime deletedAt
        -LocalDateTime createdAt
        -LocalDateTime updatedAt
        -LocalDateTime editedAt
    }

    class CommentReaction {
        <<Entity>>
        -UUID id
        -Comment comment
        -UUID userId
        -ReactionType reactionType
        -LocalDateTime createdAt
    }

    class ReactionType {
        <<Enumeration>>
        UPVOTE
        DOWNVOTE
        FIRE
        THINKING
        CLAP
        SURPRISED
        LOVE
    }

    class CommentRepository {
        <<interface>>
        +findByReadingIdAndParentIsNull(UUID readingId) List~Comment~
        +findByParentId(UUID parentId) List~Comment~
        +findByReadingIdOrderByCreatedAtAsc(UUID readingId) List~Comment~
    }

    class CommentReactionRepository {
        <<interface>>
        +findByCommentId(UUID commentId) List~CommentReaction~
        +findByCommentIdAndUserId(UUID commentId, UUID userId) Optional~CommentReaction~
        +existsByCommentIdAndUserId(UUID commentId, UUID userId) boolean
        +deleteByCommentIdAndUserId(UUID commentId, UUID userId) void
    }

    Comment --> Comment : parent (self-referential ManyToOne)
    CommentReaction --> Comment : ManyToOne
    CommentReaction --> ReactionType : has
    CommentRepository ..> Comment : manages
    CommentReactionRepository ..> CommentReaction : manages
```

---

## Class Diagram - Service Layer

```mermaid
classDiagram
    class CommentService {
        <<interface>>
        +createComment(String username, UUID readingId, CommentRequest request) CommentResponse
        +replyToComment(String username, UUID readingId, UUID parentCommentId, CommentRequest request) CommentResponse
        +getCommentsByReadingId(UUID readingId, String username) List~CommentResponse~
        +updateComment(String username, UUID readingId, UUID commentId, CommentRequest request) CommentResponse
        +softDeleteComment(String username, UUID readingId, UUID commentId) void
        +adminDeleteComment(String username, UUID commentId) void
    }

    class CommentServiceImpl {
        -CommentRepository commentRepository
        -UserRepository userRepository
        -ReadingRepository readingRepository
        -CommentReactionService reactionService
        +createComment(String username, UUID readingId, CommentRequest request) CommentResponse
        +replyToComment(String username, UUID readingId, UUID parentCommentId, CommentRequest request) CommentResponse
        +getCommentsByReadingId(UUID readingId, String username) List~CommentResponse~
        +updateComment(String username, UUID readingId, UUID commentId, CommentRequest request) CommentResponse
        +softDeleteComment(String username, UUID readingId, UUID commentId) void
        +adminDeleteComment(String username, UUID commentId) void
        -assembleTree(List~Comment~ all, String username) List~CommentResponse~
        -buildNode(Comment comment, Map childrenByParent, String username) CommentResponse
        -validateContent(CommentRequest request) void
        -resolveUser(String username) User
        -newBaseComment(UUID readingId, UUID authorId, String content) Comment
        -loadActiveCommentForReading(UUID commentId, UUID readingId) Comment
        -requireOwnership(Comment comment, User user) void
    }

    class CommentReactionService {
        <<interface>>
        +addOrUpdateReaction(String username, UUID commentId, ReactionRequest request) void
        +removeReaction(String username, UUID commentId) void
        +getReactionCounts(UUID commentId) Map~ReactionType, Integer~
        +getUserReaction(String username, UUID commentId) ReactionType
    }

    class CommentReactionServiceImpl {
        -CommentReactionRepository reactionRepository
        -CommentRepository commentRepository
        -UserRepository userRepository
        +addOrUpdateReaction(String username, UUID commentId, ReactionRequest request) void
        +removeReaction(String username, UUID commentId) void
        +getReactionCounts(UUID commentId) Map~ReactionType, Integer~
        +getUserReaction(String username, UUID commentId) ReactionType
        -resolveUser(String username) User
    }

    class CommentRequest {
        -String content
    }

    class CommentResponse {
        -UUID id
        -UUID readingId
        -UUID authorId
        -UUID parentId
        -String content
        -boolean deleted
        -LocalDateTime createdAt
        -LocalDateTime updatedAt
        -LocalDateTime editedAt
        -List~CommentResponse~ replies
        -Map~ReactionType, Integer~ reactionCounts
        -ReactionType myReaction
        +fromEntity(Comment comment, List~CommentResponse~ replies)$ CommentResponse
    }

    class ReactionRequest {
        -ReactionType reactionType
    }

    CommentService <|.. CommentServiceImpl : implements
    CommentReactionService <|.. CommentReactionServiceImpl : implements
    CommentServiceImpl --> CommentReactionService : uses
    CommentServiceImpl ..> CommentRequest : receives
    CommentServiceImpl ..> CommentResponse : returns
    CommentReactionServiceImpl ..> ReactionRequest : receives
```

---

## Sequence Diagram - Create Comment dan Reply Flow

```mermaid
sequenceDiagram
    actor User as Pengguna (Reader)
    participant FE as Next.js Frontend
    participant CC as CommentController
    participant CS as CommentServiceImpl
    participant UR as UserRepository
    participant RR as ReadingRepository
    participant CR as CommentRepository
    participant DB as PostgreSQL

    User->>FE: Tulis komentar pada halaman reading
    FE->>CC: POST /api/v1/readings/{readingId}/comments { "content": "Bagus sekali!" }

    CC->>CS: createComment(username, readingId, request)

    CS->>CS: validateContent(request)

    CS->>UR: findByUsername(username)
    UR->>DB: SELECT * FROM users WHERE username = ?
    DB-->>UR: User record
    UR-->>CS: Optional~User~ (present)

    CS->>RR: existsById(readingId)
    RR->>DB: SELECT COUNT(*) FROM readings WHERE id = ?
    DB-->>RR: true
    RR-->>CS: true

    CS->>CS: newBaseComment(readingId, authorId, content)
    CS->>CR: save(comment)
    CR->>DB: INSERT INTO comments ...
    DB-->>CR: Comment (saved)
    CR-->>CS: Comment

    CS-->>CC: CommentResponse
    CC-->>FE: 201 Created { id, readingId, authorId, content, ... }
    FE-->>User: Komentar tampil di halaman

    Note over User, DB: Alur Reply ke Komentar

    User->>FE: Klik Reply pada komentar yang ada
    FE->>CC: POST /api/v1/readings/{readingId}/comments/{parentId}/replies { "content": "Setuju!" }

    CC->>CS: replyToComment(username, readingId, parentId, request)
    CS->>CS: validateContent(request)
    CS->>UR: findByUsername(username)
    UR-->>CS: User

    CS->>CR: findById(parentId)
    CR->>DB: SELECT * FROM comments WHERE id = ?
    DB-->>CR: Comment (parent)
    CR-->>CS: Optional~Comment~ (present)

    CS->>CS: Cek parent.isDeleted() dan parent.readingId == readingId

    CS->>CS: newBaseComment(readingId, authorId, content)
    CS->>CS: reply.setParent(parent)
    CS->>CR: save(reply)
    CR->>DB: INSERT INTO comments (parent_id = parentId) ...
    DB-->>CR: Comment (saved)
    CR-->>CS: Comment

    CS-->>CC: CommentResponse (dengan parentId terisi)
    CC-->>FE: 201 Created { id, parentId, content, ... }
    FE-->>User: Reply tampil di bawah komentar induk
```

---

## Sequence Diagram - Get Comments dengan Tree Assembly dan Reactions

```mermaid
sequenceDiagram
    actor User as Pengguna
    participant FE as Next.js Frontend
    participant CC as CommentController
    participant CS as CommentServiceImpl
    participant CRS as CommentReactionServiceImpl
    participant CR as CommentRepository
    participant CRR as CommentReactionRepository
    participant DB as PostgreSQL

    User->>FE: Buka halaman reading
    FE->>CC: GET /api/v1/readings/{readingId}/comments

    CC->>CS: getCommentsByReadingId(readingId, username)

    CS->>CR: findByReadingIdOrderByCreatedAtAsc(readingId)
    CR->>DB: SELECT * FROM comments WHERE reading_id = ? ORDER BY created_at ASC
    DB-->>CR: List~Comment~ (flat, semua level)
    CR-->>CS: List~Comment~

    CS->>CS: assembleTree(all, username)
    Note over CS: Kelompokkan comment berdasarkan parent_id
    Note over CS: Top-level comments = parent IS NULL
    Note over CS: Bangun tree rekursif via buildNode()

    loop Untuk setiap comment node
        CS->>CRS: getReactionCounts(commentId)
        CRS->>CRR: findByCommentId(commentId)
        CRR->>DB: SELECT * FROM comment_reactions WHERE comment_id = ?
        DB-->>CRR: List~CommentReaction~
        CRR-->>CRS: List~CommentReaction~
        CRS->>CRS: Hitung Map(ReactionType -> count)
        CRS-->>CS: Map~ReactionType, Integer~

        CS->>CRS: getUserReaction(username, commentId)
        CRS->>CRR: findByCommentIdAndUserId(commentId, userId)
        CRR->>DB: SELECT * FROM comment_reactions WHERE comment_id = ? AND user_id = ?
        DB-->>CRR: Optional~CommentReaction~
        CRR-->>CRS: Optional~CommentReaction~
        CRS-->>CS: ReactionType (atau null)

        CS->>CS: Set reactionCounts dan myReaction pada CommentResponse
    end

    CS-->>CC: List~CommentResponse~ (tree structure)
    CC-->>FE: 200 OK [ { id, content, replies: [...], reactionCounts: {...}, myReaction: "UPVOTE" } ]
    FE-->>User: Tampilkan thread komentar bersarang dengan reaksi
```

---

## Sequence Diagram - Reaction Flow (Add, Update, Remove)

```mermaid
sequenceDiagram
    actor User as Pengguna
    participant FE as Next.js Frontend
    participant RC as CommentReactionController
    participant CRS as CommentReactionServiceImpl
    participant UR as UserRepository
    participant CR as CommentRepository
    participant CRR as CommentReactionRepository
    participant DB as PostgreSQL

    Note over User, DB: Menambah atau Mengubah Reaksi

    User->>FE: Klik reaksi pada komentar (misal: UPVOTE)
    FE->>RC: POST /api/v1/comments/{commentId}/reactions { "reactionType": "UPVOTE" }

    RC->>CRS: addOrUpdateReaction(username, commentId, request)

    CRS->>UR: findByUsername(username)
    UR->>DB: SELECT * FROM users WHERE username = ?
    DB-->>UR: User
    UR-->>CRS: User

    CRS->>CR: findById(commentId)
    CR->>DB: SELECT * FROM comments WHERE id = ?
    DB-->>CR: Comment
    CR-->>CRS: Optional~Comment~ (present)

    CRS->>CRR: findByCommentIdAndUserId(commentId, userId)
    CRR->>DB: SELECT * FROM comment_reactions WHERE comment_id = ? AND user_id = ?
    DB-->>CRR: Optional~CommentReaction~

    alt Reaksi sudah ada (update)
        CRR-->>CRS: Optional (present)
        CRS->>CRS: existing.setReactionType(UPVOTE)
        Note over CRS: Tidak perlu save() eksplisit, JPA dirty checking
    else Reaksi belum ada (insert)
        CRR-->>CRS: Optional (empty)
        CRS->>CRS: new CommentReaction(comment, userId, UPVOTE, now)
        CRS->>CRR: save(reaction)
        CRR->>DB: INSERT INTO comment_reactions ...
        DB-->>CRR: CommentReaction (saved)
    end

    CRS-->>RC: void
    RC-->>FE: 201 Created (no body)
    FE-->>User: Tampilkan reaksi terupdate

    Note over User, DB: Menghapus Reaksi

    User->>FE: Klik reaksi yang sama untuk membatalkan
    FE->>RC: DELETE /api/v1/comments/{commentId}/reactions

    RC->>CRS: removeReaction(username, commentId)
    CRS->>UR: findByUsername(username)
    UR-->>CRS: User

    CRS->>CRR: findByCommentIdAndUserId(commentId, userId)
    CRR->>DB: SELECT * FROM comment_reactions WHERE comment_id = ? AND user_id = ?
    DB-->>CRR: Optional~CommentReaction~

    alt Reaksi ada
        CRR-->>CRS: Optional (present)
        CRS->>CRR: delete(reaction)
        CRR->>DB: DELETE FROM comment_reactions WHERE id = ?
        DB-->>CRR: ok
    else Reaksi tidak ada
        CRR-->>CRS: Optional (empty)
        Note over CRS: ifPresent tidak terpenuhi, tidak ada aksi
    end

    CRS-->>RC: void
    RC-->>FE: 204 No Content
    FE-->>User: Reaksi dihapus
```

---

## Sequence Diagram - Soft Delete dan Admin Delete Flow

```mermaid
sequenceDiagram
    actor User as Pengguna (Reader)
    actor Admin as Admin
    participant CC as CommentController
    participant CS as CommentServiceImpl
    participant UR as UserRepository
    participant CR as CommentRepository
    participant DB as PostgreSQL

    Note over User, DB: Soft Delete oleh Pemilik Komentar

    User->>CC: DELETE /api/v1/readings/{readingId}/comments/{commentId}

    CC->>CS: softDeleteComment(username, readingId, commentId)

    CS->>UR: findByUsername(username)
    UR->>DB: SELECT * FROM users WHERE username = ?
    DB-->>UR: User
    UR-->>CS: User (currentUser)

    CS->>CR: findById(commentId)
    CR->>DB: SELECT * FROM comments WHERE id = ?
    DB-->>CR: Comment
    CR-->>CS: Optional~Comment~

    CS->>CS: loadActiveCommentForReading(): cek isDeleted() dan readingId cocok
    CS->>CS: requireOwnership(): cek authorId == currentUser.id

    alt Bukan pemilik komentar
        CS-->>CC: throw AccessDeniedException
        CC-->>User: 403 Forbidden
    else Pemilik komentar
        CS->>CS: existing.setDeleted(true), setDeletedBy(userId), setDeletedAt(now)
        CS->>CR: save(existing)
        CR->>DB: UPDATE comments SET is_deleted=true, deleted_by=?, deleted_at=? WHERE id=?
        DB-->>CR: Comment (updated)
        CS-->>CC: void
        CC-->>User: 204 No Content
    end

    Note over Admin, DB: Admin Delete (tanpa cek kepemilikan)

    Admin->>CC: DELETE /api/v1/admin/comments/{commentId}
    Note over CC: @PreAuthorize("hasRole('ADMIN')")

    CC->>CS: adminDeleteComment(username, commentId)
    CS->>UR: findByUsername(username)
    UR-->>CS: User (admin)

    CS->>CR: findById(commentId)
    CR->>DB: SELECT * FROM comments WHERE id = ?
    DB-->>CR: Comment
    CR-->>CS: Optional~Comment~

    alt Komentar sudah dihapus
        CS-->>CC: throw IllegalArgumentException "Komentar sudah dihapus!"
        CC-->>Admin: 400 Bad Request
    else Komentar aktif
        CS->>CS: existing.setDeleted(true), setDeletedBy(adminId), setDeletedAt(now)
        CS->>CR: save(existing)
        CR->>DB: UPDATE comments SET is_deleted=true, deleted_by=?, deleted_at=? WHERE id=?
        DB-->>CR: ok
        CS-->>CC: void
        CC-->>Admin: 204 No Content
    end
```
