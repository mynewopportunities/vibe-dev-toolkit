# System Architecture: News Synthesizer

## Overview

The News Synthesizer is a privacy-focused, local RSS news processing application that transforms raw news feeds into personalized audio broadcasts using advanced LLM inference capabilities. The system integrates llama.cpp with the mlabonne_gemma-3-27b-it-abliterated-IQ4_XS.gguf model to provide offline news synthesis with interactive persona control.

**Core Pipeline:** RSS Feeds → Metadata Generation → RAG Synthesis → Persona Composition → Text-to-Speech → Audio Output

## System Architecture Diagram

```mermaid
graph TB
    subgraph "Data Sources"
        RSS[RSS Feeds<br/>YAML Configuration]
        MODEL[LLM Model<br/>GGUF File]
    end

    subgraph "Backend Processing (Python)"
        FETCH[RSS Fetcher<br/>Async Processing]
        METADATA[Metadata Generator<br/>LLM Inference]
        RAG[RAG Synthesizer<br/>Context Retrieval]
        PERSONA[Persona Composer<br/>Natural Language Generation]
        TTS[Text-to-Speech<br/>Edge-TTS Integration]
        AUDIO[Audio Queue<br/>Threading]
    end

    subgraph "Frontend UI (Next.js)"
        FEEDS[Feed Management<br/>/feeds]
        ARTICLES[Article Browser<br/>/articles]
        SYNTHESIZE[Synthesis Interface<br/>/synthesize]
        COMPOSE[Composition Control<br/>/compose]
        TTS_UI[Audio Playback<br/>/tts]
        CHAT[Persona Chat<br/>/chat]
    end

    subgraph "Database"
        SQLITE[(SQLite<br/>Local Caching & Persistence)]
    end

    RSS --> FETCH
    MODEL --> METADATA
    FETCH --> METADATA
    METADATA --> SQLITE
    METADATA --> RAG
    RAG --> PERSONA
    PERSONA --> TTS
    TTS --> AUDIO

    SQLITE --> FEEDS
    SQLITE --> ARTICLES

    subgraph "API Mappings"
        direction LR
        API_FEEDS[GET/POST /api/v1/feeds]
        API_ARTICLES[GET/POST /api/v1/articles]
        API_METADATA[GET /api/v1/metadata/{id}]
        API_SYNTHESIZE[POST /api/v1/synthesize]
        API_COMPOSE[POST /api/v1/compose]
        API_TTS[POST /api/v1/tts]
        API_PERSONAS[GET/POST /api/v1/personas]
        API_CHAT[POST /api/v1/chat]

        FEEDS --> API_FEEDS
        ARTICLES --> API_ARTICLES
        SYNTHESIZE --> API_SYNTHESIZE
        COMPOSE --> API_COMPOSE
        TTS_UI --> API_TTS
        CHAT --> API_CHAT
    end
```

## Technology Stack

### Backend Infrastructure
- **Framework:** FastAPI (planned/planning phase)
- **Language:** Python 3.8+
- **LLM Integration:** llama.cpp with GGUF model files
- **Model:** mlabonne_gemma-3-27b-it-abliterated-IQ4_XS.gguf (13B parameters, IQ4_XS quantization)
- **RSS Processing:** feedparser, newspaper3k for content extraction
- **Text-to-Speech:** Edge-TTS (Microsoft Edge voices)
- **NLP Processing:** spaCy, scikit-learn, sentence-transformers
- **Async Processing:** aiohttp, asyncio for concurrent RSS fetching
- **Data Persistence:** SQLite with local file storage
- **Configuration:** PyYAML for feeds and persona configurations

### Frontend Interface
- **Framework:** Next.js 13+ with TypeScript
- **Styling:** Tailwind CSS (from components.json)
- **UI Components:** shadcn/ui component library
- **State Management:** React hooks and Context API
- **API Integration:** Fetch API with error handling
- **Audio Playback:** HTML5 Audio API with media controls
- **Responsive Design:** Mobile-first approach for accessibility

### Hardware Requirements
- **Minimum:** 8GB RAM, AVX2-compatible CPU, 10GB storage
- **Recommended:** NVIDIA GPU (8GB+ VRAM), 16GB+ RAM, SSD storage
- **LLM Acceleration:** GPU-accelerated inference via CUDA or similar

## Component Breakdown

### Backend Components

#### 1. RSS Feed Processor
**Location:** `backend/src/feeds/`
**Function:** Async fetching and parsing of RSS feeds configured in `feeds.yaml`
- **Input:** YAML feed configuration with URLs and metadata
- **Output:** Raw article data with timestamps and source info
- **Mapping:** Supports /api/v1/feeds (GET: retrieve feeds, POST: add feed)

#### 2. Metadata Generator
**Location:** `backend/src/nlp/metadata.py` (planned)
**Function:** LLM-powered extraction of article metadata
- **Input:** Raw article text
- **Output:** Structured metadata (summary, sentiment, keywords, topics)
- **Model Integration:** llama.cpp inference with custom prompts
- **Mapping:** /api/v1/metadata/{article_id} (GET)

#### 3. RAG Synthesizer
**Location:** `backend/src/nlp/rag.py` (planned)
**Function:** Retrieval-augmented generation for context synthesis
- **Input:** Query + article metadata collection
- **Output:** Synthesized context for composition
- **Mapping:** POST /api/v1/synthesize

#### 4. Persona Composer
**Location:** `backend/src/personas/` (core logic planned)
**Function:** LL-Driven composition using configurable personas
- **Input:** Synthesized context + persona configuration
- **Output:** Personalized news segment text
- **Configuration:** YAML-based persona definitions in `personas/` directory
- **Mapping:** POST /api/v1/compose, GET/POST /api/v1/personas

#### 5. Text-to-Speech Engine
**Location:** `backend/src/audio/tts.py` (planned)
**Function:** Conversion of composed text to audio
- **Input:** Text segments with optional voice parameters
- **Output:** Audio files/cache for playback
- **Technology:** Microsoft Edge TTS voices
- **Mapping:** POST /api/v1/tts

#### 6. Chat Interface Processor
**Location:** `backend/src/chat/` (planned)
**Function:** Interactive persona parameter adjustments
- **Input:** Natural language user messages
- **Output:** Modified persona configurations
- **Mapping:** POST /api/v1/chat

#### 7. Audio Playback System
**Location:** `backend/src/audio/player.py` (implemented)
**Function:** Threaded audio queue management and playback
- **Input:** Audio file queue
- **Output:** Continuous audio broadcast
- **Integration:** Core loop in generator.py

### Frontend Components

#### 1. Feed Management (/feeds)
**Location:** `frontend/src/app/feeds/`
**Components:** FeedList, FeedAdder, FeedEditor
- **API Calls:** GET/POST /api/v1/feeds
- **Features:** Add/remove RSS sources, configure fetch settings

#### 2. Article Browser (/articles)
**Location:** `frontend/src/app/articles/`
**Components:** ArticleGrid, ArticleDetails, MetadataDisplay
- **API Calls:** GET /api/v1/articles, GET /api/v1/metadata/{id}
- **Features:** Browse processed articles with metadata

#### 3. Context Synthesis (/synthesize)
**Location:** `frontend/src/app/synthesize/`
**Components:** SynthesisForm, ContextPreview, QueryBuilder
- **API Calls:** POST /api/v1/synthesize
- **Features:** Query input, synthesized context display

#### 4. News Composition (/compose)
**Location:** `frontend/src/app/compose/`
**Components:** PersonaSelector, CompositionEditor, SegmentPreview
- **API Calls:** GET /api/v1/personas, POST /api/v1/compose
- **Features:** Persona selection, rendering controls

#### 5. Audio Playback (/tts)
**Location:** `frontend/src/app/tts/`
**Components:** AudioPlayer, PlaybackControls, VoiceSelector
- **API Calls:** POST /api/v1/tts
- **Features:** Audio generation, playback controls, seek/speed

#### 6. Persona Chat (/chat)
**Location:** `frontend/src/app/chat/`
**Components:** ChatPanel, MessageList, PersonaParameters
- **API Calls:** POST /api/v1/chat, GET/POST /api/v1/personas
- **Features:** Real-time persona adjustments

## API Endpoint Mappings

### Backend Endpoints (FastAPI - Planned)

| Endpoint | Method | Purpose | Frontend Mapping | Data Flow |
|----------|--------|---------|------------------|-----------|
| `/api/v1/feeds` | GET | Retrieve configured RSS feeds | `/feeds` (list/display) | Config → UI |
| `/api/v1/feeds` | POST | Add new feed source | `/feeds` (add form) | UI → Config |
| `/api/v1/articles` | GET | Get processed articles list | `/articles` (filter/browse) | SQLite → UI |
| `/api/v1/articles` | POST | Trigger batch processing | `/articles` (refresh button) | RSS → Processing |
| `/api/v1/metadata/{article_id}` | GET | Retrieve article metadata | `/articles` (details view) | SQLite → UI |
| `/api/v1/synthesize` | POST | Generate RAG context | `/synthesize` (query form) | LLM → Context |
| `/api/v1/compose` | POST | Create persona-driven segment | `/compose` (composition form) | Context → Text |
| `/api/v1/tts` | POST | Request audio generation | `/tts` (generate button) | Text → Audio |
| `/api/v1/personas` | GET | List available personas | `/compose`, `/chat` (selectors) | YAML → UI |
| `/api/v1/personas` | POST | Create/modify persona | `/chat` (parameter adjustments) | UI → YAML |
| `/api/v1/chat` | POST | Send persona adjustment message | `/chat` (message input) | NL Input → Config |

### Data Flow Patterns

#### RSS Processing Pipeline
1. **Fetch Phase:** `/api/v1/feeds` (GET) → RSS sources retrieved
2. **Processing Phase:** `/api/v1/articles` (POST) → Article batch processing triggered
3. **Storage Phase:** Metadata stored in SQLite via internal processing

#### Composition Pipeline
1. **Context Generation:** `/synthesize` (POST) → RAG synthesis from articles
2. **Persona Application:** `/compose` (POST) → Style application with selected persona
3. **Audio Generation:** `/tts` (POST) → Text-to-speech conversion

#### Interactive Adjustment
1. **Persona Selection:** `/personas` (GET) → Available configurations loaded
2. **Chat Modification:** `/chat` (POST) → Natural languageupdates to persona
3. **Parameter Persistence:** `/personas` (POST) → Updated configurations stored

## Persona System Architecture

### Persona Configuration Files
**Location:** `backend/personas/*.yaml`
**Structure:**
- **Basic Attributes:** name, description, tone, style, bias, formality
- **Language Characteristics:** vocabulary_level, humor, perspective, emotional_expression
- **Content Focus:** intellectual_focus, moral_positioning, cultural_context
- **Expression Style:** rhetorical_style, argumentation_method, use_of_metaphor
- **Technical Aspects:** epistemic_disclosure, certainty_expression, value_system

### Runtime Persona Management
- **File System Integration:** Persona files loaded at startup via PyYAML
- **Dynamic Modification:** Chat interface enables parameter adjustments
- **Persistence:** Modified personas saved to YAML files
- **Concurrency:** Thread-safe access for multi-user scenarios

## Security and Performance Considerations

### Security Architecture
- **Local Processing:** All LLM inference occurs offline, no data transmission
- **Model Isolation:** GGUF models stored locally with filesystem protection
- **Input Sanitization:** YAML configuration validation and RSS content filtering
- **API Security:** Planned JWT authentication for session management
- **Rate Limiting:** Built-in throttling for chat and synthesis endpoints

### Performance Optimization
- **GPU Acceleration:** CUDA-optimized llama.cpp for inference speed (4-8x faster)
- **Memory Management:** Efficient model loading and context caching
- **Concurrent Processing:** Async RSS fetching with configurable concurrency limits
- **Data Caching:** SQLite optimization for metadata storage and retrieval
- **Audio Streaming:** Background threading for uninterrupted audio playback

### Scalability Considerations
- **Batch Processing:** Multiple article handling in single inference calls
- **Model Sharing:** Single model instance with concurrent safe access
- **Resource Limiting:** System monitoring with automatic throttling
- **Storage Optimization:** Deduplication and compression for news archives

## Deployment Architecture

### Local Development
- **Backend:** Uvicorn ASGI server with hot reload
- **Frontend:** Next.js development server with fast refresh
- **Integration:** Direct API calls to localhost backend
- **Debugging:** Comprehensive logging and model inference monitoring

### Docker Containerization
- **Backend Container:** Python slim base with llama.cpp compilation
- **Frontend Container:** Node Alpine with Next.js production build
- **Model Management:** GGUF files mounted from host or volume
- **Networking:** Internal service communication via Docker networks

### Production Considerations
- **GPU Provisioning:** Cloud GPU instances (AWS/GCP) with spot pricing
- **Monitoring:** LLM inference latency and memory usage tracking
- **Backup Strategy:** SQLite database file versioning
- **Update Management:** Model file versioning and hot-swapping capability

## Future Enhancements

### Planned API Extensions
- **Article Filtering:** Advanced search and filtering endpoints
- **Template System:** Custom composition templates beyond personas
- **Multi-Model Support:** Additional GGUF model compatibility
- **Streaming Audio:** Real-time TTS streaming for immediate playback
- **History Tracking:** User interaction and preference learning

### Analytics Integration
- **Usage Metrics:** Endpoint usage statistics for optimization
- **Quality Assessment:** LLM output quality scoring and management
- **Performance Database:** Historical processing time tracking

This architecture provides a solid foundation for privacy-focused, AI-driven news synthesis with extensible persona controls and comprehensive API integrations between backend processing and frontend interactions.
