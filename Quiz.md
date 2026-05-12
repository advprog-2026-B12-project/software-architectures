# Quiz & Reading Module — Individual Deliverable (feat/quiz)

## Component Diagram

```mermaid
graph TB
        subgraph external ["External Actors"]
        Reader["👤 Reader / Learner<br/><i>Reads passages,<br/>takes quizzes,<br/>views results</i>"]
        Admin["👤 Admin<br/><i>Creates readings,<br/>questions, and options</i>"]
        DiscussionModule["💬 Discussion Forum Module<br/><i>Comments linked<br/>to readings</i>"]
        OtherModules["📦 Other Modules<br/><i>Clan / Achievements<br/>see quiz result</i>"]
    end

    subgraph quizmodule ["Reading & Quiz Module"]
        direction TB

        subgraph controllers ["Controllers"]
            QC["🎮 QuizController<br/><i>/api/quiz/**</i><br/><br/>GET /{readingId}<br/>POST /submit<br/>GET /all<br/>GET /status/{userId}/{readingId}"]

            RC["🎮 ReadingController<br/><i>/api/readings/**</i><br/><br/>GET /<br/>GET /{id}"]

            ARC["🎮 AdminReadingController<br/><i>/api/admin/readings/**</i><br/><br/>POST /<br/>PUT /{id}<br/>DELETE /{id}"]

            AQC["🎮 AdminQuestionController<br/><i>/api/admin/questions/**</i><br/><br/>POST /<br/>PUT /{id}<br/>DELETE /{id}"]

            AOC["🎮 AdminOptionController<br/><i>/api/admin/options/**</i><br/><br/>POST /<br/>PUT /{id}<br/>DELETE /{id}"]
        end

        subgraph services ["Services"]
            QS["⚙️ QuizService<br/><i>interface</i>"]
            QSI["⚙️ QuizServiceImpl<br/><i>submit()<br/>calculateScore()<br/>preventDuplicateAttempt()</i>"]

            RS["⚙️ ReadingService<br/><i>interface</i>"]
            RSI["⚙️ ReadingServiceImpl<br/><i>findAll()<br/>findById()<br/>createReading()</i>"]

            QUES["⚙️ QuestionService<br/><i>interface</i>"]
            QUESI["⚙️ QuestionServiceImpl<br/><i>createQuestion()<br/>updateQuestion()<br/>deleteQuestion()</i>"]

            OS["⚙️ OptionService<br/><i>interface</i>"]
            OSI["⚙️ OptionServiceImpl<br/><i>createOption()<br/>updateOption()<br/>deleteOption()</i>"]
        end

        subgraph repositories ["Repositories"]
            RR["🗄️ ReadingRepository<br/><i>JpaRepository</i>"]
            QR["🗄️ QuestionRepository<br/><i>JpaRepository</i>"]
            OR["🗄️ OptionRepository<br/><i>JpaRepository</i>"]
            QAR["🗄️ QuizAttemptRepository<br/><i>JpaRepository</i><br/><br/>existsByUserIdAndReadingId()"]
        end

        subgraph models ["Models / Entities"]
            READING["📋 Reading"]
            QUESTION["📋 Question"]
            OPTION["📋 Option"]
            ATTEMPT["📋 QuizAttempt"]
        end

        subgraph dtos ["DTOs"]
            RRQ["ReadingRequest"]
            RRS["ReadingResponse"]

            QRQ["QuestionRequest"]
            QRS["QuestionResponse"]

            ORQ["OptionRequest"]
            ORS["OptionResponse"]

            QSUB["QuizSubmitRequest"]
            QRES["QuizResponse"]
            QRR["QuizResultResponse"]
        end
    end

    DB[("🗄️ PostgreSQL<br/><i>Database</i>")]

    Reader -->|"GET quiz<br/>POST submit answers"| QC
    Reader -->|"GET readings"| RC

    Admin -->|"Manage readings"| ARC
    Admin -->|"Manage questions"| AQC
    Admin -->|"Manage options"| AOC

    QC -->|"delegates to"| QS
    RC -->|"delegates to"| RS
    ARC -->|"delegates to"| RS
    AQC -->|"delegates to"| QUES
    AOC -->|"delegates to"| OS

    QS -.->|"implements"| QSI
    RS -.->|"implements"| RSI
    QUES -.->|"implements"| QUESI
    OS -.->|"implements"| OSI

    QSI -->|"load reading"| RR
    QSI -->|"check duplicate attempt"| QAR
    QSI -->|"save attempt"| QAR

    RSI -->|"CRUD readings"| RR

    QUESI -->|"CRUD questions"| QR
    QUESI -->|"validate parent reading"| RR

    OSI -->|"CRUD options"| OR
    OSI -->|"validate parent question"| QR

    RR -->|"JPA / JDBC"| DB
    QR -->|"JPA / JDBC"| DB
    OR -->|"JPA / JDBC"| DB
    QAR -->|"JPA / JDBC"| DB

    RR -.->|"manages"| READING
    QR -.->|"manages"| QUESTION
    OR -.->|"manages"| OPTION
    QAR -.->|"manages"| ATTEMPT

    READING -->|"contains"| QUESTION
    QUESTION -->|"contains"| OPTION
    ATTEMPT -->|"references"| READING

    DiscussionModule -.->|"imports ReadingRepository"| RR
    OtherModules -.->|"planned event:<br/>quiz.completed"| QSI

    style quizmodule fill:#1E1B4B,color:#C7D2FE,stroke:#6366F1

    style controllers fill:#312E81,color:#C7D2FE,stroke:#6366F1
    style services fill:#312E81,color:#C7D2FE,stroke:#6366F1
    style repositories fill:#1E3A5F,color:#BAE6FD,stroke:#0EA5E9
    style models fill:#14532D,color:#BBF7D0,stroke:#22C55E
    style dtos fill:#422006,color:#FDE68A,stroke:#F59E0B

    style QC fill:#06B6D4,color:#fff,stroke:#0891B2
    style RC fill:#06B6D4,color:#fff,stroke:#0891B2
    style ARC fill:#EF4444,color:#fff,stroke:#DC2626
    style AQC fill:#EF4444,color:#fff,stroke:#DC2626
    style AOC fill:#EF4444,color:#fff,stroke:#DC2626

    style QSI fill:#8B5CF6,color:#fff,stroke:#7C3AED
    style RSI fill:#8B5CF6,color:#fff,stroke:#7C3AED
    style QUESI fill:#8B5CF6,color:#fff,stroke:#7C3AED
    style OSI fill:#8B5CF6,color:#fff,stroke:#7C3AED

    style RR fill:#0EA5E9,color:#fff,stroke:#0284C7
    style QR fill:#0EA5E9,color:#fff,stroke:#0284C7
    style OR fill:#0EA5E9,color:#fff,stroke:#0284C7
    style QAR fill:#0EA5E9,color:#fff,stroke:#0284C7

    style DB fill:#3ECF8E,color:#fff,stroke:#22C55E
```
### Ringkasan Interaksi Komponen
| From                      | To                      | Interaksi                                            | Protokol            |
| ------------------------- | ----------------------- | ---------------------------------------------------- | ------------------- |
| Reader Frontend           | QuizController          | `GET /api/quiz/{readingId}`, `POST /api/quiz/submit` | REST/JSON           |
| Reader Frontend           | ReadingController       | `GET /api/readings/**`                               | REST/JSON           |
| Admin Frontend            | AdminReadingController  | CRUD readings                                        | REST/JSON           |
| Admin Frontend            | AdminQuestionController | CRUD questions                                       | REST/JSON           |
| Admin Frontend            | AdminOptionController   | CRUD options                                         | REST/JSON           |
| QuizController            | QuizServiceImpl         | `submit()`, `getQuizStatus()`                        | Method call         |
| ReadingController         | ReadingServiceImpl      | `findAll()`, `findById()`                            | Method call         |
| AdminReadingController    | ReadingServiceImpl      | `create()`, `update()`, `delete()`                   | Method call         |
| AdminQuestionController   | QuestionServiceImpl     | `createQuestion()`, `updateQuestion()`               | Method call         |
| AdminOptionController     | OptionServiceImpl       | `createOption()`, `updateOption()`                   | Method call         |
| QuizServiceImpl           | QuizAttemptRepository   | `existsByUserIdAndReadingId()`, `save()`             | JPA                 |
| QuizServiceImpl           | ReadingRepository       | `findById()`                                         | JPA                 |
| QuestionServiceImpl       | QuestionRepository      | CRUD operations                                      | JPA                 |
| OptionServiceImpl         | OptionRepository        | CRUD operations                                      | JPA                 |
| Semua Repository          | PostgreSQL              | Persist entities                                     | JDBC / JPA          |

---

## Class Diagram - Entity

## Class Diagram - Entities

```mermaid
classDiagram
    class Reading {
        <<Entity>>
        -UUID id
        -String title
        -String content
        -String category
        -String difficulty
        -String thumbnailUrl
        -Integer estimatedMinutes
    }

    class Question {
        <<Entity>>
        -UUID id
        -Reading reading
        -String questionText
        -Integer questionOrder
    }

    class Option {
        <<Entity>>
        -UUID id
        -Question question
        -String optionText
        -boolean correct
    }

    class QuizAttempt {
        <<Entity>>
        -UUID id
        -UUID userId
        -Reading reading
        -Integer score
        -Integer totalQuestions
        -boolean passed
        -LocalDateTime submittedAt
    }

    class ReadingRepository {
        <<interface>>
        +findAll() List~Reading~
        +findById(UUID id) Optional~Reading~
    }

    class QuestionRepository {
        <<interface>>
        +findByReadingId(UUID readingId) List~Question~
        +findById(UUID id) Optional~Question~
    }

    class OptionRepository {
        <<interface>>
        +findByQuestionId(UUID questionId) List~Option~
        +findById(UUID id) Optional~Option~
    }

    class QuizAttemptRepository {
        <<interface>>
        +findByUserId(UUID userId) List~QuizAttempt~
        +existsByUserIdAndReadingId(UUID userId, UUID readingId) boolean
    }

    Reading --> Question : OneToMany
    Question --> Reading : ManyToOne

    Question --> Option : OneToMany
    Option --> Question : ManyToOne

    QuizAttempt --> Reading : ManyToOne

    ReadingRepository ..> Reading : manages
    QuestionRepository ..> Question : manages
    OptionRepository ..> Option : manages
    QuizAttemptRepository ..> QuizAttempt : manages
```

---

## Class Diagram — QuizService

```mermaid
classDiagram
    class QuizService {
        <<interface>>
        +submit(QuizSubmitRequest request) QuizResultResponse
    }

    class QuizServiceImpl {
        -ReadingRepository readingRepository
        -QuizAttemptRepository quizAttemptRepository
        +submit(QuizSubmitRequest request) QuizResultResponse
    }

    class QuizSubmitRequest {
        -UUID userId
        -UUID readingId
        -Map~UUID,UUID~ answers
    }

    class QuizResultResponse {
        -int score
        -int totalQuestions
        -boolean passed
        -String message
    }

    class ReadingRepository {
        <<interface>>
        +findById(UUID id) Optional~Reading~
        +findAll() List~Reading~
    }

    class QuizAttemptRepository {
        <<interface>>
        +existsByUserIdAndReadingId(UUID userId, UUID readingId) boolean
        +save(QuizAttempt attempt) QuizAttempt
    }

    QuizService <|.. QuizServiceImpl : implements

    QuizServiceImpl --> ReadingRepository : uses
    QuizServiceImpl --> QuizAttemptRepository : uses

    QuizServiceImpl ..> QuizSubmitRequest : receives
    QuizServiceImpl ..> QuizResultResponse : returns
```

---

## Class Diagram — QuizController

```mermaid
classDiagram
    class QuizController {
        -ReadingService readingService
        -QuizService quizService
        -ReadingRepository readingRepository
        -QuizAttemptRepository quizAttemptRepository
        +getQuiz(UUID readingId) Reading
        +submit(QuizSubmitRequest request) ResponseEntity
        +getAll() List~Reading~
        +getQuizStatus(UUID userId, UUID readingId) Map~String,Boolean~
    }

    class ReadingService {
        <<interface>>
        +findById(UUID id) Reading
    }

    class QuizService {
        <<interface>>
        +submit(QuizSubmitRequest request) QuizResultResponse
    }

    class ReadingRepository {
        <<interface>>
        +findAll() List~Reading~
    }

    class QuizAttemptRepository {
        <<interface>>
        +existsByUserIdAndReadingId(UUID userId, UUID readingId) boolean
    }

    class Reading {
        -UUID id
        -String title
        -String content
    }

    class QuizSubmitRequest {
        -UUID userId
        -UUID readingId
        -Map~UUID,UUID~ answers
    }

    class QuizResultResponse {
        -int score
        -int totalQuestions
        -boolean passed
    }

    QuizController --> ReadingService : uses
    QuizController --> QuizService : uses
    QuizController --> ReadingRepository : uses
    QuizController --> QuizAttemptRepository : uses

    QuizController ..> Reading : returns
    QuizController ..> QuizSubmitRequest : receives
    QuizController ..> QuizResultResponse : returns
```

---

## Sequence Diagram - Admin Creates a Quiz

```mermaid
sequenceDiagram
    actor Admin as Admin
    participant FE as Next.js Frontend
    participant ARC as AdminReadingController
    participant RS as ReadingServiceImpl
    participant RR as ReadingRepository
    participant AQC as AdminQuestionController
    participant QUES as QuestionServiceImpl
    participant QR as QuestionRepository
    participant AOC as AdminOptionController
    participant OS as OptionServiceImpl
    participant OR as OptionRepository
    participant DB as PostgreSQL

    Admin->>FE: Open "Create Quiz" page
    FE-->>Admin: Display empty reading/question form

    Note over Admin,FE: Step 1 — Create Reading

    Admin->>FE: Fill reading title and content
    FE->>ARC: POST /api/admin/readings

    ARC->>RS: createReading(readingRequest)
    RS->>RR: save(reading)
    RR->>DB: INSERT INTO readings ...
    DB-->>RR: Saved Reading
    RR-->>RS: Reading entity
    RS-->>ARC: ReadingResponse
    ARC-->>FE: 201 Created + readingId

    FE-->>Admin: Reading successfully created

    Note over Admin,FE: Step 2 — Create Questions

    loop For each question
        Admin->>FE: Add question text
        FE->>AQC: POST /api/admin/questions

        AQC->>QUES: createQuestion(questionRequest)

        QUES->>RR: findById(readingId)
        RR->>DB: SELECT * FROM readings WHERE id=?
        DB-->>RR: Reading record
        RR-->>QUES: Reading entity

        QUES->>QR: save(question)
        QR->>DB: INSERT INTO questions ...
        DB-->>QR: Saved Question
        QR-->>QUES: Question entity

        QUES-->>AQC: QuestionResponse
        AQC-->>FE: 201 Created + questionId

        Note over Admin,FE: Step 3 — Create Options

        loop For each option
            Admin->>FE: Add option text + correctness
            FE->>AOC: POST /api/admin/options

            AOC->>OS: createOption(optionRequest)

            OS->>QR: findById(questionId)
            QR->>DB: SELECT * FROM questions WHERE id=?
            DB-->>QR: Question record
            QR-->>OS: Question entity

            OS->>OR: save(option)
            OR->>DB: INSERT INTO options ...
            DB-->>OR: Saved Option
            OR-->>OS: Option entity

            OS-->>AOC: OptionResponse
            AOC-->>FE: 201 Created
        end
    end

    FE-->>Admin: Quiz successfully created
```

---

## Sequence Diagram - Quiz Submission Flow

```mermaid
sequenceDiagram
    actor User as Reader / Learner
    participant FE as Next.js Frontend
    participant QC as QuizController
    participant QS as QuizServiceImpl
    participant QAR as QuizAttemptRepository
    participant RR as ReadingRepository
    participant DB as PostgreSQL

    User->>FE: Answer quiz questions
    User->>FE: Click "Submit Quiz"

    FE->>QC: POST /api/quiz/submit

    QC->>QS: submit(quizSubmitRequest)

    QS->>QAR: existsByUserIdAndReadingId(userId, readingId)
    QAR->>DB: SELECT EXISTS(...)
    DB-->>QAR: true / false
    QAR-->>QS: duplicate status

    alt User already completed quiz
        QS-->>QC: throw IllegalStateException("Quiz already completed")
        QC-->>FE: 409 Conflict
        FE-->>User: Display duplicate attempt error

    else First submission
        QS->>RR: findById(readingId)
        RR->>DB: SELECT reading + questions + options
        DB-->>RR: Reading entity
        RR-->>QS: Reading

        QS->>QS: calculateScore()

        QS->>QAR: save(quizAttempt)
        QAR->>DB: INSERT INTO quiz_attempts ...
        DB-->>QAR: Saved attempt

        QAR-->>QS: QuizAttempt entity

        QS-->>QC: QuizResultResponse
        QC-->>FE: 200 OK + score/result

        FE-->>User: Display quiz result
    end
```