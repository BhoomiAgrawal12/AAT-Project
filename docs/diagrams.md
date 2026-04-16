# GymTally — UML Diagrams

> Rendered automatically by GitHub. If viewing locally, paste into [mermaid.live](https://mermaid.live).

---

## 1. Class Diagram

```mermaid
classDiagram
    direction LR

    class FastAPI_App {
        +startup()
        +POST /api/auth/login()
        +POST /api/auth/signup()
        +GET /api/auth/me()
        +POST /api/ingest()
        +POST /api/session/start()
        +POST /api/session/end()
        +GET /api/session/id()
        +GET /api/sessions()
        +GET /api/live()
        +POST /api/query()
        +POST /api/query/stream()
        +POST /api/collect/start()
        +POST /api/collect/stop()
        +GET /api/collect/status()
        +GET /api/health()
    }

    class Classifier {
        -model_path: Path
        -model: KerasModel
        -mock: bool
        +_load()
        +_normalise(window) ndarray
        +predict(window) tuple~int, float, list~
    }

    class RepCounter {
        -_lock: Lock
        -session_id: int
        -current_exercise: str
        -last_active_exercise: str
        -state: str
        -reps: int
        -hi_conf: int
        -total: int
        -set_id: int
        -in_valley: bool
        -last_rep_time: float
        -_pending_exercise: str
        -_pending_count: int
        +SWITCH_THRESHOLD: int
        +start_session(user) int
        +end_session() int
        +update(exercise, confidence) dict
        +snapshot() dict
        -_reset_set_state()
        -_open_set_locked(conn, exercise)
        -_close_set_locked(conn)
    }

    class Collector {
        -csv_path: Path
        -_lock: Lock
        -_label: str
        -_counts: dict
        +start(label) dict
        +stop() dict
        +status() dict
        +maybe_append(ax, ay, az)
        -_ensure_header()
    }

    class PerDeviceRateLimiter {
        -max: int
        -_lock: Lock
        -_buckets: dict
        +check(key) bool
    }

    class ESP32_Firmware {
        +WIFI_SSID: string
        +WIFI_PASS: string
        +SERVER_URL: string
        +DEVICE_ID: string
        +WINDOW_SIZE: int = 50
        +WINDOW_STRIDE: int = 25
        +SAMPLE_HZ: int = 50
        +setup()
        +loop()
        -adxlInit()
        -adxlReadXYZ() bool
        -wifiConnect()
        -buildPayload() String
        -sendWindow()
    }

    class ADXL345 {
        +I2C_ADDR: 0x53
        +SDA: GPIO21
        +SCL: GPIO22
        +RANGE: +-4g
        +RATE: 50Hz
        +readXYZ() int16 x3
    }

    class ReactApp {
        +AuthPage
        +LiveCounter
        +SessionHistory
        +FloatingChat
    }

    class AuthPage {
        -tab: login | signup
        -username: string
        -password: string
        +onAuthenticated(username)
    }

    class LiveCounter {
        -state: LiveState
        -err: string
        +onSessionChange(id)
        -polls /api/live every 500ms
    }

    class SessionHistory {
        -sessions: list
        -selected: int
        -summary: SessionSummary
        +computeAnalytics() Analytics
    }

    class FloatingChat {
        -messages: Msg[]
        -busy: bool
        +send(question)
        +stop()
        -streams /api/query/stream
    }

    class MongoDB {
        +users: Collection
        +sessions: Collection
        +sets: Collection
        +counters: Collection
    }

    class Ollama {
        +model: llama3.2
        +/api/chat
        +/api/generate
    }

    FastAPI_App --> Classifier : loads model
    FastAPI_App --> RepCounter : counts reps
    FastAPI_App --> Collector : collects data
    FastAPI_App --> PerDeviceRateLimiter : rate limits /api/ingest
    FastAPI_App --> MongoDB : persists users, sessions, sets
    FastAPI_App --> Ollama : NLP queries

    ESP32_Firmware --> ADXL345 : I2C reads
    ESP32_Firmware --> FastAPI_App : POST /api/ingest

    ReactApp --> AuthPage
    ReactApp --> LiveCounter
    ReactApp --> SessionHistory
    ReactApp --> FloatingChat
    ReactApp --> FastAPI_App : REST + ndjson stream

    RepCounter --> MongoDB : CRUD sessions & sets
    Classifier ..> RepCounter : exercise label + confidence
```

---

## 2. Sequence Diagram — Real-time Rep Counting Flow

```mermaid
sequenceDiagram
    participant ESP as ESP32 + ADXL345
    participant BE as FastAPI Backend
    participant CLS as Classifier (CNN-LSTM)
    participant RC as RepCounter
    participant DB as MongoDB
    participant FE as React Frontend

    Note over ESP: Samples accel at 50Hz<br/>Buffers 50 samples (1s window)<br/>Slides by 25 samples (0.5s)

    loop Every 0.5s (stride = 25 samples)
        ESP->>+BE: POST /api/ingest {device_id, ax[50], ay[50], az[50]}
        BE->>BE: Rate limit check (100/min/device)
        BE->>+CLS: predict(window)
        CLS->>CLS: z-score normalize per channel
        CLS->>CLS: model.predict(window)
        CLS-->>-BE: (class_idx, confidence, probabilities)

        alt exercise == "other"
            BE->>RC: update("other", conf)
            RC->>RC: Suppress in live snapshot<br/>(keep last known exercise)
        else exercise is curl/squat/rest
            BE->>+RC: update(exercise, confidence)

            alt New exercise != current (e.g. squat during curls)
                RC->>RC: Increment pending_count
                alt pending_count >= SWITCH_THRESHOLD (3)
                    RC->>DB: Close current set
                    RC->>DB: Open new set
                else pending_count < 3
                    RC->>RC: Ignore (noise suppression)
                end
            end

            alt Valley detection (rest or low confidence)
                RC->>RC: state = rest, in_valley = true
            else High confidence same exercise after valley
                RC->>RC: reps += 1
                RC->>DB: Update set (reps, form_score)
            end

            RC-->>-BE: {exercise, confidence, reps, form_score, session_id}
        end

        BE-->>-ESP: 200 OK + JSON response
        Note over ESP: Prints to Serial Monitor
    end

    loop Every 500ms
        FE->>+BE: GET /api/live (Authorization: Bearer JWT)
        BE->>RC: snapshot()
        RC-->>BE: {exercise, confidence, reps, form_score, session_id}
        BE-->>-FE: JSON
        FE->>FE: Update LiveCounter UI
    end
```

---

## 3. Sequence Diagram — Authentication + Session Lifecycle

```mermaid
sequenceDiagram
    participant U as User (Browser)
    participant FE as React Frontend
    participant BE as FastAPI Backend
    participant DB as MongoDB

    U->>FE: Open http://localhost:5173
    FE->>FE: Check localStorage for JWT
    alt No JWT
        FE->>U: Show AuthPage (Login/Signup)
    end

    U->>FE: Enter username + password
    FE->>+BE: POST /api/auth/signup {username, password}
    BE->>BE: Validate (3-32 chars, pw >= 6)
    BE->>BE: hash_password(PBKDF2-SHA256, 200k iters)
    BE->>DB: Insert into users collection
    BE->>BE: Generate JWT (HS256, 24h TTL)
    BE-->>-FE: {token, expires_in, username}
    FE->>FE: Store JWT + username in localStorage
    FE->>U: Show Dashboard

    U->>FE: Click "Start Session"
    FE->>+BE: POST /api/session/start (Bearer JWT)
    BE->>DB: Insert session {id, user, started_at}
    BE-->>-FE: {session_id}
    FE->>FE: Enable live tracking

    Note over U: User exercises with ESP32...<br/>Reps counted automatically

    U->>FE: Click "End Session"
    FE->>+BE: POST /api/session/end (Bearer JWT)
    BE->>DB: Close active set (form_score, reps)
    BE->>DB: Update session ended_at
    BE-->>-FE: {session_id}

    U->>FE: Switch to History tab
    FE->>+BE: GET /api/sessions (Bearer JWT)
    BE->>DB: Find sessions where user = current
    BE-->>-FE: [{id, user, started_at, ended_at}, ...]

    U->>FE: Click session #N
    FE->>+BE: GET /api/session/N (Bearer JWT)
    BE->>DB: Find session + all sets
    BE->>BE: Compute reps_per_exercise, form_per_exercise
    BE-->>-FE: Full summary with sets[]
    FE->>FE: Render analytics (charts, grades, trends)
```

---

## 4. Sequence Diagram — NLP Chat (Streaming)

```mermaid
sequenceDiagram
    participant U as User
    participant FC as FloatingChat
    participant BE as FastAPI Backend
    participant OL as Ollama (local LLM)
    participant DB as MongoDB

    U->>FC: Click chat bubble (bottom-right)
    FC->>U: Show chat panel with greeting

    U->>FC: Type "how many curls did I do today?"
    FC->>FC: Show user bubble + thinking dots

    FC->>+BE: POST /api/query/stream {question} (Bearer JWT)
    BE->>DB: Fetch last 10 sessions + sets for user
    BE->>BE: _context_summary() → prose brief
    Note over BE: "User: admin<br/>Today's reps: curl:12, squat:8<br/>Avg form: curl 91%, squat 78%"

    BE->>+OL: POST /api/generate {model, system, prompt, stream:true}
    Note over OL: llama3.2:1b processes

    loop Token streaming
        OL-->>BE: {"response": "You"}
        BE-->>FC: {"type":"token","content":"You"}
        FC->>FC: Append to bubble, show caret

        OL-->>BE: {"response": " did"}
        BE-->>FC: {"type":"token","content":" did"}

        OL-->>BE: {"response": " 12 curls today!"}
        BE-->>FC: {"type":"token","content":" 12 curls today!"}
    end

    OL-->>-BE: {"done": true}
    BE-->>-FC: {"type":"done"}
    FC->>FC: Remove caret, mark message done
    FC->>U: "You did 12 curls today!"
```

---

## 5. Class Diagram — Data Model (MongoDB)

```mermaid
classDiagram
    direction TB

    class users {
        +username: string PK
        +password_hash: string
        +created_at: float
    }

    class sessions {
        +id: int PK
        +user: string FK
        +started_at: float
        +ended_at: float?
    }

    class sets {
        +id: int PK
        +session_id: int FK
        +exercise: string
        +reps: int
        +form_score: float
        +hi_conf: int
        +total: int
        +started_at: float
        +ended_at: float?
    }

    class counters {
        +_id: string PK
        +seq: int
    }

    users "1" --> "*" sessions : user
    sessions "1" --> "*" sets : session_id
    counters ..> sessions : auto-increment id
    counters ..> sets : auto-increment id

    note for sets "form_score = hi_conf / total\nhi_conf = windows with confidence > 0.85"
    note for users "password_hash format:\npbkdf2_sha256$200000$salt_hex$digest_hex"
```

---

## 6. Component Diagram — Frontend

```mermaid
graph TB
    subgraph React App
        APP[App.tsx]
        AUTH[AuthPage.tsx]
        LC[LiveCounter.tsx]
        SH[SessionHistory.tsx]
        FC[FloatingChat.tsx]
        API[api.ts]
    end

    APP --> AUTH
    APP --> LC
    APP --> SH
    APP --> FC

    AUTH --> API
    LC --> API
    SH --> API
    FC --> API

    API -->|mock mode| MOCK[In-memory fake data]
    API -->|live mode| NGINX[nginx :5173]
    NGINX -->|/api/*| BACKEND[FastAPI :8001]

    SH --> RECHARTS[Recharts<br/>BarChart, PieChart, LineChart]
    FC --> NDJSON[ReadableStream<br/>ndjson parser]

    style MOCK fill:#374151,stroke:#6b7280
    style NGINX fill:#1e3a5f,stroke:#60a5fa
    style BACKEND fill:#1a3329,stroke:#22c55e
```
