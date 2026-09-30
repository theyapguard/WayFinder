# WayFinder — System Architecture

WayFinder is a hands-free indoor navigation system for blind and visually impaired
students. This document explains how the pieces fit together and how a
voice request becomes turn-by-turn audio guidance.

## High-level Architecture

```mermaid
flowchart TB
    subgraph Student["🧑‍🦯 Student (Voice-First, Hands-Free)"]
        MIC["🎙️ Wake word: 'Hey WayFinder'"]
        SPK["🔊 Audio guidance (TTS)"]
    end

    subgraph App["📱 Flutter App (Android)"]
        VA["Voice Assistant Service\nPorcupine wake word + STT + GPT-4 NLU"]
        CAM["Camera Service\n8 frames @ localization, 10-15 FPS @ nav"]
        POS["Position Localization\nRAG retrieval + VLM verification"]
        NAV["Real-Time Navigation\nWaypoint embedding matching + compass"]
        GUIDE["Navigation Guide\nTurn-by-turn instruction engine"]
        EMERG["🚨 Emergency Exit Routing"]
    end

    subgraph Vision["🧠 Vision AI Server (FastAPI + PyTorch, GPU)"]
        DINO["DINOv2\n768-dim scene embeddings"]
        YOLO["YOLOv8-seg\npeople & object detection"]
        SD["Stable Diffusion 2.0\nobstacle/people inpainting"]
    end

    subgraph Cloud["☁️ Supabase (PostgreSQL + Auth + Storage)"]
        DB[("maps · map_nodes · navigation_paths\npath_waypoints · place_embeddings")]
        AUTH["Auth (students & admins)"]
        STORE["Storage: reference images, floor plans"]
    end

    subgraph Admin["🛠️ Admin Mode (Sighted Staff)"]
        MAPS["Map & node management"]
        REC["Record paths → waypoints + embeddings"]
    end

    MIC --> VA
    VA --> POS
    VA --> NAV
    VA --> EMERG
    CAM --> POS
    CAM --> NAV
    POS --> DINO
    NAV --> DINO
    CAM --> YOLO
    YOLO --> SD
    DINO --> POS
    DINO --> NAV
    POS --> GUIDE
    NAV --> GUIDE
    GUIDE --> SPK
    VA --> SPK
    POS <--> DB
    NAV <--> STORE
    EMERG <--> DB
    Admin --> DB
    AUTH --- App
    VA -. "Verify location (optional)" .-> GPT["GPT-4 Vision\nconfidence + reasoning"]
```

## Request Lifecycle

```mermaid
sequenceDiagram
    autonumber
    participant S as Student
    participant A as Flutter App
    participant V as Vision Server (GPU)
    participant C as Supabase

    S->>A: "Hey WayFinder, take me to chemistry"
    A->>A: Wake word to STT to GPT-4 intent parse
    A->>V: POST /embed (8 camera frames)
    V->>V: YOLOv8 then Stable Diffusion clean then DINOv2
    V-->>A: 768-dim embeddings (clean scene)
    A->>C: Cosine similarity vs place_embeddings
    C-->>A: Top matches (sim > 0.80)
    A->>S: Audio: "You are at the Main Entrance"
    A->>C: Load route waypoints + start node
    loop Every frame (10-15 FPS)
        A->>V: POST /embed (live frame)
        V-->>A: Live embedding
        A->>A: Compare vs current waypoint (dynamic threshold)
        alt Match (turn point)
            A->>S: Audio: "Turn left"
            A->>A: Advance to next waypoint
        else No match for 30 frames
            A->>A: Enter recovery (capture 3 frames, relocalize)
        end
    end
    A->>S: Audio: "You have arrived. Chemistry is on your right."
```

## The Three Layers of the System

### 1. Perception — "Where am I, and what's around me?"
- The camera captures frames; **YOLOv8-seg** finds people and carried objects.
- **Stable Diffusion 2.0** inpaints them out so a temporary crowd doesn't break matching.
- **DINOv2** turns the cleaned frame into a **768-dimensional embedding** — a compact,
  semantic fingerprint of the scene that is robust to lighting, angle, and time of day.

### 2. Localization — "Which classroom am I standing at?"
- **RAG (Retrieval-Augmented Generation)** over stored location embeddings:
  - **Retrieve:** cosine similarity search against `place_embeddings` (threshold > 0.80).
  - **Augment:** attach reference images and metadata for the top matches.
  - **Generate:** **GPT-4 Vision** verifies the candidate and returns confidence + reasoning.

### 3. Guidance — "How do I get to my class?"
- Recorded paths store **waypoints** (embedding + compass heading + turn type).
- During navigation the live embedding is compared to the current waypoint; when
  similarity crosses a **dynamic threshold** (0.87 clean, 0.75 crowded, 0.90 before turns),
  the waypoint instruction is spoken and the guide advances.
- A **compass** orients the student before movement starts. Loss of match for 30 frames
  triggers automatic recovery.

## Why It Works Without Infrastructure

| Traditional indoor navigation | WayFinder |
|-------------------------------|-----------|
| Bluetooth beacons installed per room | Not required — camera only |
| Precise floor-plan / Wi-Fi fingerprinting | Not required — recorded walk-through |
| Dedicated hardware for each student | Not required — any Android phone |
| GPS (doesn't work indoors) | Replaced by visual place recognition |

## Component Reference

| Layer | File(s) |
|-------|---------|
| Wake word & voice | `lib/services/voice_assistant_service.dart` |
| Localization (RAG) | `lib/services/position_localization_service.dart`, `lib/services/dinov2_service.dart` |
| Navigation loop | `lib/services/real_time_navigation_service.dart`, `lib/services/navigation_guide.dart` |
| Path recording | `lib/services/continuous_path_recorder.dart` |
| Backend / auth | `lib/services/supabase_service.dart` |
| Vision server | `scripts/dinov2_http_gateway.py` |
| Data model | `database_scheme.sql` |
