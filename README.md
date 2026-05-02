<img width="900" height="1747" alt="Home" src="https://github.com/user-attachments/assets/154f3e67-e11d-4d41-b1ba-3792c425c413" />
# amiphi

**A friend who knows your health and works for your well-being.**

ami-phi is a multimodal AI health companion. Patients speak in Hindi (or English), upload prescriptions and lab reports, and get back warm, plain-language explanations grounded in their own longitudinal health history. The system reads prescriptions and lab reports with medical vision models, talks back in natural Hindi, and remembers everything across conversations.


This repository is the working prototype. It's the evolution of an earlier project called *Medi-Bridge*. The architecture has since been rebuilt around `phi_context`, a structured longitudinal memory layer that is the central commitment of the system.

---

### Screenshots

<!-- Drop PNGs into docs/screenshots/ and they'll render here.
     Suggested files: home.png, voice.png, rx-scan.png, lab.png, dietlens.png, wellbeing.png -->

| Home | Voice chat (Hindi) | Prescription scan |
|:---:|:---:|:---:|
| ![Home](docs/screenshots/<img width="853" height="1844" alt="amitittle" src="https://github.com/user-attachments/assets/1d69b903-ce59-40bb-a112-ecc96099a480" />
home.png) | ![Voice](docs/screenshots/voice.png) | ![Rx Scan](docs/screenshots/rx-scan.png) |

| Lab report | DietLens | Well Being |
|:---:|:---:|:---:|
| ![Lab](docs/screenshots/lab.png) | ![DietLens](docs/screenshots/dietlens.png) | ![Wellbeing](docs/screenshots/wellbeing.png) |

> **Note:** This repository is published as a showcase of the architecture. It's not packaged for self-contained local installation — running it requires API keys for Anthropic, Google AI Studio, Sarvam, and a Vertex AI project with a MedGemma endpoint. If you want a code walkthrough or live demo, get in touch via the email at the bottom.

---

## What it does

- **ami** — a conversational layer (voice or text) that explains medications, lab values, dietary patterns, and emotional concerns in the language the user speaks
- **phi** — a structured longitudinal memory store that learns from every prescription scan, lab upload, food-plate analysis, and conversation, and feeds back into every subsequent reasoning turn
- **Prescription scanner** — photograph a prescription; MedGemma 4B extracts medicines, dosages, and inferred conditions as structured data
- **Lab report analysis** — upload a blood test (PDF or image); MedGemma 1.5 reads each test, computes trends against prior values, and explains what changed
- **DietLens** — photograph a meal; Gemini 2.5 Flash analyses nutritional content, cross-references active conditions and medications, and suggests alternatives
- **Well Being** — a separate emotional-health module with crisis routing that bypasses the LLM entirely on self-harm signals and routes to Indian helplines
- **Hindi voice** — full ASR + TTS + translation pipeline via Sarvam AI, with codemix support for the Hinglish people actually speak

---

## Architecture

### The core loop

```
user speaks (Hindi or English)
        ↓
Sarvam Saaras v3 transcribes (codemix, hi-IN)
        ↓
phi_context loaded from Redis (Postgres on miss)
   conditions + medications + lab_timeline + diet_profile + conv_memory
        ↓
Claude Sonnet 4.6 reasons over query + full patient context
        ↓
Sarvam Mayura translates response EN → Hindi
        ↓
Sarvam Bulbul v3 (Priya for health, Kavya for wellbeing) speaks
        ↓
phi_context updated with conv_memory event
        ↓
Redis cache invalidated
```

Non-voice modalities slot into the same loop. A prescription scan emits an `rx_scanned` event that updates conditions and medications; a lab report emits `lab_analysed` that updates lab_timeline; a food-plate scan emits `diet_updated`. All events flow through the same merge layer.

### phi_context — the memory layer

A single Postgres table with one row per user and five typed JSONB blobs:

```sql
CREATE TABLE phi_context (
    user_id       VARCHAR PRIMARY KEY,
    conditions    JSONB DEFAULT '[]',
    medications   JSONB DEFAULT '[]',
    lab_timeline  JSONB DEFAULT '[]',
    diet_profile  JSONB DEFAULT '{}',
    conv_memory   JSONB DEFAULT '[]',
    updated_at    TIMESTAMP DEFAULT NOW()
);
```

Cached in Redis with 24h TTL and merge-on-write invalidation. Rendered into a compact natural-language string and injected into Claude's system prompt at every turn — context is present from the start of generation, not retrieved mid-reasoning.

### Multi-model orchestration via LangGraph

Four model families speak through one agent graph:

| Layer | Model | Role |
|---|---|---|
| Clinical reasoning | Claude Sonnet 4.6 (Anthropic) | Conversational reasoning, conv_memory summarisation |
| Medical vision (primary) | MedGemma 4B (Google, Vertex AI) | Prescription OCR, food-plate ID |
| Lab extraction | MedGemma 1.5 4B (Google, Vertex AI) | Lab report extraction |
| Medical vision (fallback) | Gemini 2.5 Flash (Google AI Studio) | Silent fallback on MedGemma timeout/error; primary for DietLens |
| Hindi ASR | Sarvam Saaras v3 | Speech-to-text, codemix |
| Hindi TTS | Sarvam Bulbul v3 (Priya, Kavya) | Text-to-speech, two voices |
| Translation | Sarvam Mayura v1 | EN → HI (formal) |

Every external model call has an 8-second timeout and a typed fallback. A single model failure never breaks the conversation loop.

### Safety isolation

`wellbeing_context` is a separate store from `phi_context`. The two never merge. Three reasons:

1. Privacy — emotional disclosures and medical data live under different user expectations
2. Reasoning interference — emotional content drifts clinical reasoning measurably
3. Crisis routing requires bypassing the generative model entirely

Crisis-classified inputs are routed deterministically to a safety template referencing iCall (9152987821) and the Vandrevala Foundation (1860-2662-345). The LLM is never invoked for crisis turns.

---

## Tech stack

### Backend
| | |
|---|---|
| Framework | FastAPI + Python 3.11 + Uvicorn |
| Orchestration | LangGraph multi-agent state machine |
| Clinical reasoning | Anthropic Claude Sonnet 4.6 |
| Medical vision | Google MedGemma 4B / 1.5 (Vertex AI dedicated endpoint) |
| Vision fallback | Google Gemini 2.5 Flash |
| Voice | Sarvam Saaras v3 (ASR), Bulbul v3 (TTS), Mayura v1 (translation) |
| Memory | Postgres 15 (JSONB) + Redis 7 (24h TTL cache) |
| RAG | ChromaDB + LangChain (1,589-chunk OpenFDA drug corpus + Indian nutrition corpus) |

### Mobile
| | |
|---|---|
| Framework | React Native + Expo (TypeScript) |
| Navigation | React Navigation stack |
| State | Zustand |
| Camera | expo-camera |
| Audio | expo-av |
| Markdown | react-native-markdown-display |

### Web
| | |
|---|---|
| Framework | React + Vite |
| Hosting | Vercel |

### Infrastructure
Docker Compose for Postgres 15, Redis 7, ChromaDB. MedGemma served via Vertex AI dedicated endpoint (g2-standard-8 + 1× NVIDIA L4). Production deploy to Railway and Firebase Auth (phone OTP) on the immediate roadmap.

---

## API surface

| Method | Path | Description |
|---|---|---|
| GET | `/health` | Health check |
| POST | `/voice/text` | Text chat with Claude + phi_context |
| POST | `/voice/audio` | Voice chat (Sarvam ASR → Claude → Bulbul TTS) |
| POST | `/prescriptions/scan` | MedGemma OCR → phi_context |
| POST | `/reports/analyse` | MedGemma lab extraction → phi_context |
| POST | `/reports/explain-test` | Explain a single lab value |
| POST | `/nutrition/chat` | Nutrition AI chat |
| POST | `/nutrition/meal-plan` | 7-day meal plan |
| POST | `/dietlens/scan` | Gemini 2.5 Flash food-plate analysis |
| POST | `/dietlens/chat` | ami conversation grounded in DietLens result |
| POST | `/wellbeing/chat` | Therapy session (emotional classifier + safety router) |
| POST | `/wellbeing/session/start` | Start wellbeing session |
| POST | `/wellbeing/session/end` | End session, write summary |

Interactive OpenAPI docs at `/docs` when the backend is running.

---

## Repository layout

```
medi-bridge/
├── backend/
│   ├── agents/
│   │   ├── asr_agent.py                  # Sarvam Saaras
│   │   ├── tts_agent.py                  # Sarvam Bulbul
│   │   ├── orchestrator.py               # LangGraph state machine
│   │   ├── enhanced_reasoning_agent.py   # Claude with phi_context
│   │   └── rag_agent.py                  # ChromaDB retrieval
│   ├── routers/
│   │   ├── voice.py                      # Text + audio chat
│   │   ├── prescriptions.py              # Rx scan → phi_event
│   │   ├── reports.py                    # Lab analysis → phi_event
│   │   ├── dietlens.py                   # Food-plate analysis
│   │   ├── wellbeing.py                  # Emotional health module
│   │   └── nutrition.py                  # Meal plans
│   ├── services/
│   │   ├── claude.py                     # Anthropic client
│   │   ├── medical_vision_service.py     # MedGemma + Gemini fallback
│   │   ├── medgemma_service.py           # Prescription + food plate
│   │   ├── medgemma_lab_service.py       # Lab extraction
│   │   ├── dietlens_service.py           # Gemini food intelligence
│   │   ├── sarvam_service.py             # Voice + translation
│   │   ├── emotional_classifier.py       # Wellbeing intent classifier
│   │   ├── safety_router.py              # Crisis bypass
│   │   ├── translation_service.py        # Mayura
│   │   └── phi_context.py                # Memory layer
│   ├── db/                               # Postgres + Redis clients
│   └── main.py
├── mobile/
│   └── src/
│       ├── screens/                      # Voice, Lab, Rx, DietLens, WellBeing, etc.
│       ├── components/
│       ├── hooks/
│       ├── services/api.ts
│       ├── store/                        # Zustand
│       └── theme/                        # Brand tokens (amber/coral/calm-blue)
├── rag/
│   └── ingest/                           # OpenFDA drug corpus, nutrition corpus
└── infra/
    └── docker-compose.yml                # Postgres 15, Redis 7, ChromaDB
```

---

## Status

Working prototype. Backend, mobile (8 screens), and web are live in development.

| Functional today | Scaffolded / planned |
|---|---|
| Voice chat (Hindi + English, codemix) | Proactive ami calls (WebSocket) |
| Prescription scanning + extraction | HR-facing aggregate dashboard |
| Lab report analysis with trends | Firebase phone-OTP auth |
| DietLens food-plate analysis | Production deploy on Railway |
| Well Being module + crisis safety router | ABDM / ABHA integration |
| phi_context longitudinal memory | Longitudinal user study |
| ChromaDB RAG over drug + nutrition corpora | |
| Multi-model fallback orchestration | |

---

## Acknowledgements

Built on Anthropic's Claude, Google's MedGemma and Gemini families (via Vertex AI and AI Studio), and Sarvam AI's voice stack.

---

## Contact

[hello@amiphi.one](mailto:hello@amiphi.one)

---

## License

MIT
