### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                     React Native Mobile App                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │ Auth Screen  │  │ Teacher Dash │  │ Student Dash │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              Offline Storage Layer                         │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐       │  │
│  │  │ SQLite DB   │  │ AsyncStorage│  │ File System │       │  │
│  │  │(Materials,  │  │(User Prefs) │  │(PDFs Cache) │       │  │
│  │  │ Quizzes,    │  │             │  │             │       │  │
│  │  │ Progress)   │  │             │  │             │       │  │
│  │  └─────────────┘  └─────────────┘  └─────────────┘       │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │              LLM Service (Online/Offline)                  │  │
│  │  ┌─────────────┐              ┌─────────────┐            │  │
│  │  │ Ollama API  │ ←(Online)→   │ Tiny Model  │            │  │
│  │  │(Good Model) │              │ (Bundled)   │            │  │
│  │  └─────────────┘              └─────────────┘            │  │
│  └──────────────────────────────────────────────────────────┘  │
│                           │                                      │
│                           │ HTTPS/REST API                       │
└───────────────────────────┼──────────────────────────────────────┘
                             │
                ┌────────────┴────────────┐
                │                         │
        [Online] │                 [Offline]
                │                         │
┌──────────────────────────┐   ┌──────────────────────────┐
│    Express Backend API   │   │   Works Completely        │
│  ┌────────────────────┐  │   │   Offline (No Internet)  │
│  │  Auth Middleware   │  │   └──────────────────────────┘
│  └────────────────────┘  │
│  ┌────────────────────┐  │
│  │  Route Handlers    │  │
│  │  - Classroom       │  │
│  │  - Materials       │  │
│  │  - Quiz            │  │
│  │  - Chat            │  │
│  └────────────────────┘  │
│  ┌────────────────────┐  │
│  │  Business Logic    │  │
│  │  - PDF Service     │  │
│  │  - Embedding Svc   │  │
│  │  - Quiz Generator  │  │
│  │  - Chatbot Service │  │
│  │  - S3 Service      │  │
│  │  - LLM Service     │  │
│  └────────────────────┘  │
└──────────────────────────┘
           │
    ┌──────┼──────┬──────────────┐
    │      │      │              │
┌─────────┴─┐ ┌──┴──────────┐ ┌─┴──────────┐
│ MongoDB   │ │ AWS S3      │ │ Ollama     │
│ (Atlas)   │ │ (File       │ │ API Server │
│           │ │ Storage)    │ │ (Optional) │
│ - Users   │ │             │ │            │
│ - Classroom│ │ - PDFs     │ │ - LLM      │
│ - Materials│ │ - Images   │ │   Inference│
│ - Progress │ │            │ │            │
│ - Embeddings│            │ │            │
│   (Vector) │             │ │            │
│            │             │ │            │
│ Vector     │             │ │            │
│ Search     │             │ │            │
│ Index      │             │ │            │
└────────────┘ └───────────┘ └────────────┘
```

### Data Flow Architecture

#### 1. Teacher Uploads Material Flow

```
Teacher → Upload PDF → Backend API
                          │
                          ├→ S3 Upload (File Storage)
                          ├→ PDF Processing (Extract Text)
                          ├→ Text Chunking (500-1000 tokens)
                          ├→ Generate Embeddings
                          └→ Store in MongoDB (Embeddings Collection)
                                 │
                                 └→ Index for Vector Search
```

#### 2. Student Downloads Material Flow

```
Student → Request Material → Backend API
                              │
                              ├→ Get S3 Presigned URL
                              └→ Download to Device
                                     │
                                     └→ Store in SQLite + File System
                                            │
                                            └→ Available Offline
```

#### 3. Quiz Generation Flow

```
Student → Select Material → Request Quiz Generation
                              │
                    ┌─────────┴─────────┐
                    │                   │
            [Online Detected]    [Offline Detected]
                    │                   │
                    ▼                   ▼
              Ollama API          Tiny Bundled Model
              (Good Model)        (Quantized Model)
                    │                   │
                    └─────────┬─────────┘
                              │
                              ▼
                    Generate MCQ Questions
                              │
                              ▼
                    Store in SQLite (Local)
                              │
                    └→ Can be taken offline
```

#### 4. Chatbot Query Flow

```
Student → Select Material → Ask Question
                              │
                    ┌─────────┴─────────┐
                    │                   │
            [Online]              [Offline]
                    │                   │
                    ▼                   ▼
            Query MongoDB        Query Local SQLite
            Vector Search        (Pre-synced Embeddings)
                    │                   │
                    ▼                   ▼
            Get Top 3-5 Chunks  Get Top 3-5 Chunks
            (Cosine Similarity) (Local Cosine Search)
                    │                   │
                    ▼                   ▼
            Format Context      Format Context
                    │                   │
        ┌───────────┴───────────┐
        │                       │
    [Online]                [Offline]
        │                       │
        ▼                       ▼
    Ollama API            Tiny Model
    (Good Response)      (Basic Response)
        │                       │
        └───────────┬───────────┘
                    │
                    ▼
            Return Answer to Student
```

#### 5. Offline Sync Flow

```
App detects Internet → Sync Service
                          │
                          ├→ Upload Queued Actions
                          │  (Material Uploads, Progress)
                          │
                          ├→ Download New Materials
                          │  (Check for updates)
                          │
                          └→ Sync Progress
                              (Quiz Scores, Completion)# xyz
