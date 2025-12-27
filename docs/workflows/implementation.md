# Implementation Plan: News Synthesizer

## Overview

This document provides a comprehensive implementation plan for the News Synthesizer application, following the kliewerdaniel/workflow methodology. The implementation will progress through all development phases: Requirements → Architecture → Implementation → Testing → Security → Deployment → Operations.

## Development Methodology

The implementation follows a structured, iterative approach with departmental specialization:

1. **Requirements** (Complete): Functional and non-functional specs defined
2. **Architecture** (Complete): System design with API mappings completed
3. **Implementation** (In Progress): Component development and integration
4. **Testing** (Planned): Comprehensive validation strategy
5. **Security** (Planned): OWASP practices and model isolation
6. **Deployment** (Planned): Containerization and CI/CD
7. **Operations** (Planned): Maintenance and monitoring procedures

## Phase 1: Backend Infrastructure Setup

### 1.1 FastAPI Application Structure

**Target Location:** `backend/main.py` → `backend/src/api/`

**Implement:** Convert CLI main.py to FastAPI application with the following structure:

```python
# backend/src/api/app.py
from fastapi import FastAPI, BackgroundTasks
from fastapi.middleware.cors import CORSMiddleware
from .routes import feeds, articles, personas, synthesis, compose, tts, chat
from .core.config import settings
from .core.database import init_database
from .core.llm import load_model

app = FastAPI(
    title="News Synthesizer API",
    description="LLM-powered RSS news processing and synthesis",
    version="1.0.0"
)

# CORS for frontend integration
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000", "http://127.0.0.1:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Include routers
app.include_router(feeds.router, prefix="/api/v1")
app.include_router(articles.router, prefix="/api/v1")
# ... other routers

# Background tasks setup
background_tasks = BackgroundTasks()

@app.on_event("startup")
async def startup_event():
    init_database()
    load_model()

@app.get("/health")
async def health_check():
    return {"status": "healthy"}
```

### 1.2 Database Schema Implementation

**Target Location:** `backend/src/core/database.py`

**Implement:** SQLite with SQLAlchemy for local data persistence:

```python
from sqlalchemy import Column, Integer, String, Text, DateTime, JSON, create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker

Base = declarative_base()

class Feed(Base):
    __tablename__ = "feeds"
    id = Column(Integer, primary_key=True)
    url = Column(String, unique=True, nullable=False)
    name = Column(String)
    category = Column(String)
    enabled = Column(Boolean, default=True)

class Article(Base):
    __tablename__ = "articles"
    id = Column(Integer, primary_key=True)
    feed_id = Column(Integer, ForeignKey("feeds.id"))
    title = Column(String)
    url = Column(String, unique=True)
    content = Column(Text)
    published_at = Column(DateTime)
    metadata = Column(JSON)  # LLM-generated metadata
    processed_at = Column(DateTime)

class SynthesisJob(Base):
    __tablename__ = "synthesis_jobs"
    id = Column(Integer, primary_key=True)
    query = Column(String)
    article_ids = Column(JSON)  # List of article IDs used
    synthesis = Column(Text)    # RAG-generated content
    created_at = Column(DateTime)

class AudioFile(Base):
    __tablename__ = "audio_files"
    id = Column(Integer, primary_key=True)
    segment_id = Column(String)  # Reference to composed segment
    file_path = Column(String)
    duration = Column(Integer)   # in seconds
    voice = Column(String)

# Database initialization
engine = create_engine("sqlite:///news_synthesizer.db")
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
```

### 1.3 LLM Integration Layer

**Target Location:** `backend/src/core/llm.py`

**Implement:** llama.cpp wrapper with model management:

```python
import llama_cpp
from llama_cpp import Llama
import logging
from typing import List, Dict, Any
from .config import settings

class LLMManager:
    _instance = None
    _model = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
        return cls._instance

    def load_model(self):
        if self._model is None:
            logging.info("Loading LLM model...")
            self._model = Llama(
                model_path=settings.LLM_MODEL_PATH,
                n_ctx=settings.LLM_CONTEXT_SIZE,
                n_gpu_layers=settings.LLM_GPU_LAYERS,
                n_threads=settings.LLM_CPU_THREADS,
                verbose=False
            )
            logging.info("LLM model loaded successfully")
        return self._model

    def generate_metadata(self, article_text: str) -> Dict[str, Any]:
        """Extract article metadata using LLM"""
        prompt = f"""Analyze this news article and provide:
- Summary (2-3 sentences)
- Sentiment (positive/negative/neutral with confidence score)
- Keywords (top 10)
- Topics (categorized subjects)

Article: {article_text[:settings.MAX_ARTICLE_LENGTH]}

Output as JSON:"""

        response = self.generate(prompt, max_tokens=200)
        # Parse and validate JSON response
        return self._parse_json_response(response)

    def retrieve_and_synthesize(self, query: str, articles: List[Dict]) -> str:
        """Perform RAG synthesis"""
        context = self._prepare_context(articles)
        prompt = f"""Given these article summaries and this query, synthesize the most relevant information:

Query: {query}

Articles:
{context}

Provide a coherent synthesis addressing the query:"""

        return self.generate(prompt, max_tokens=500)

    def compose_with_persona(self, content: str, persona: Dict) -> str:
        """Compose using persona guidelines"""
        prompt = f"""You are {persona['name']}.

{persona['description']}

Tone: {persona['tone']}
Style: {persona['style']}
Formality: {persona['formality']}
Vocabulary: {persona['vocabulary_level']}
Humor level: {persona['humor']}

Compose a news segment based on this content:

{content}

Segment:"""

        return self.generate(prompt, max_tokens=600)

    def generate(self, prompt: str, max_tokens: int = 300) -> str:
        """Generic generation method"""
        if self._model is None:
            raise RuntimeError("Model not loaded")

        full_prompt = f"<s>[INST] {prompt} [/INST]"
        response = self._model(
            full_prompt,
            max_tokens=max_tokens,
            temperature=0.7,
            top_p=0.9,
            stop=["</s>"]
        )
        return response['choices'][0]['text'].strip()

    def _prepare_context(self, articles: List[Dict]) -> str:
        """Format articles for RAG context"""
        context_lines = []
        for article in articles:
            context_lines.append(f"- {article['title']}: {article['summary']}")
        return "\n".join(context_lines)

    def _parse_json_response(self, response: str) -> Dict[str, Any]:
        """Safely parse LLM JSON responses"""
        try:
            # Extract JSON from response
            import json
            start = response.find('{')
            end = response.rfind('}') + 1
            if start != -1 and end > start:
                json_str = response[start:end]
                return json.loads(json_str)
        except Exception:
            logging.warning(f"Could not parse JSON response: {response}")
        return {}
```

## Phase 2: API Router Implementation

### 2.1 Feed Management Router

**Target Location:** `backend/src/api/routes/feeds.py`

**Implement:** CRUD operations for RSS feed sources:

```python
from fastapi import APIRouter, HTTPException, Query
from typing import List, Optional
import aiohttp
import feedparser
import yaml
from sqlalchemy.orm import Session
from ..dependencies import get_db
from ..models import Feed

router = APIRouter()

@router.get("/feeds")
async def get_feeds(
    skip: int = Query(0, ge=0),
    limit: int = Query(100, ge=1, le=1000)
) -> List[Dict[str, Any]]:
    """Retrieve configured RSS feeds"""
    feeds = get_feeds_from_config()
    return feeds[skip:skip+limit]

@router.post("/feeds")
async def add_feed(feed_data: FeedCreate):
    """Add new RSS feed source"""
    # Validate feed URL
    try:
        async with aiohttp.ClientSession() as session:
            async with session.get(feed_data.url, timeout=10) as response:
                if response.status != 200:
                    raise HTTPException(400, "Feed URL not accessible")
                feed_content = await response.text()
                parsed = feedparser.parse(feed_content)
                if parsed.feed == {}:
                    raise HTTPException(400, "Invalid RSS feed format")
    except Exception as e:
        raise HTTPException(400, f"Feed validation error: {str(e)}")

    # Save to configuration
    feeds_config = load_feeds_config()
    new_feed = {
        "name": feed_data.name,
        "url": feed_data.url,
        "category": feed_data.category
    }
    feeds_config.append(new_feed)
    save_feeds_config(feeds_config)

    return {"message": "Feed added successfully", "feed": new_feed}

@router.post("/feeds/test")
async def test_feed(url: str):
    """Test RSS feed connectivity and parse sample entries"""
    try:
        async with aiohttp.ClientSession() as session:
            async with session.get(url, timeout=10) as response:
                content = await response.text()

        parsed = feedparser.parse(content)
        sample_entries = []

        for entry in parsed.entries[:3]:  # Test first 3 entries
            sample_entries.append({
                "title": entry.get('title', ''),
                "link": entry.get('link', ''),
                "published": entry.get('published', ''),
                "summary": entry.get('summary', '')[:200] + "..." if len(entry.get('summary', '')) > 200 else entry.get('summary', '')
            })

        return {
            "feed_title": parsed.feed.get('title', ''),
            "feed_description": parsed.feed.get('description', ''),
            "total_entries": len(parsed.entries),
            "sample_entries": sample_entries
        }
    except Exception as e:
        raise HTTPException(400, f"Feed test failed: {str(e)}")
```

### 2.2 Article Processing Router

**Target Location:** `backend/src/api/routes/articles.py`

**Implement:** Article listing, metadata retrieval, and processing:

```python
from fastapi import APIRouter, BackgroundTasks, HTTPException
from typing import List, Optional
from sqlalchemy.orm import Session
from ..dependencies import get_db
from ..models import Article
from ..services import rss_processor, metadata_generator

router = APIRouter()

@router.get("/articles")
async def get_articles(
    skip: int = Query(0, ge=0),
    limit: int = Query(50, ge=1, le=500),
    feed_id: Optional[int] = None,
    processed: Optional[bool] = None
) -> List[Dict[str, Any]]:
    """Retrieve processed articles"""
    db = next(get_db())
    query = db.query(Article)

    if feed_id:
        query = query.filter(Article.feed_id == feed_id)
    if processed is not None:
        if processed:
            query = query.filter(Article.processed_at.isnot(None))
        else:
            query = query.filter(Article.processed_at.is_none())

    articles = query.offset(skip).limit(limit).all()

    return [{
        "id": article.id,
        "feed_id": article.feed_id,
        "title": article.title,
        "url": article.url,
        "published_at": article.published_at,
        "processed_at": article.processed_at,
        "has_metadata": bool(article.metadata)
    } for article in articles]

@router.post("/articles")
async def process_articles(
    background_tasks: BackgroundTasks,
    feed_ids: Optional[List[int]] = None
):
    """Trigger RSS fetching and metadata generation"""
    background_tasks.add_task(process_rss_feeds_background, feed_ids)
    return {"message": "Article processing started asynchronously"}

@router.get("/articles/{article_id}/metadata")
async def get_article_metadata(article_id: int):
    """Retrieve LLM-generated metadata for article"""
    db = next(get_db())
    article = db.query(Article).filter(Article.id == article_id).first()

    if not article:
        raise HTTPException(404, "Article not found")

    if not article.metadata:
        # Generate metadata if not exists
        llm = LLMManager()
        metadata = llm.generate_metadata(article.content)
        article.metadata = metadata
        db.commit()
        db.refresh(article)

    return article.metadata

async def process_rss_feeds_background(feed_ids: Optional[List[int]] = None):
    """Background task for RSS processing"""
    processor = RSSProcessor()
    llm_manager = LLMManager()

    try:
        # Fetch from RSS feeds
        new_articles = await processor.fetch_from_feeds(feed_ids)

        # Generate metadata for each article
        for article in new_articles:
            if not article.content:
                continue
            metadata = llm_manager.generate_metadata(article.content)
            article.metadata = metadata

        # Save to database
        db = next(get_db())
        for article in new_articles:
            db_article = Article(
                feed_id=article.feed_id,
                title=article.title,
                url=article.url,
                content=article.content,
                published_at=article.published_at,
                metadata=article.metadata,
                processed_at=datetime.utcnow()
            )
            db.add(db_article)
        db.commit()

        logging.info(f"Processed {len(new_articles)} articles")

    except Exception as e:
        logging.error(f"RSS processing failed: {str(e)}")
```

### 2.3 Synthesis and Composition Routers

**Target Location:** `backend/src/api/routes/synthesis.py`, `backend/src/api/routes/compose.py`

**Implement:** RAG synthesis and persona-driven composition:

```python
# synthesis.py
from fastapi import APIRouter, HTTPException
from sqlalchemy.orm import Session
from ..dependencies import get_db
from ..models import Article, SynthesisJob
from ..core.llm import LLMManager

router = APIRouter()

@router.post("/synthesize")
async def synthesize_content(
    request: SynthesisRequest
) -> SynthesisResponse:
    """Generate RAG context from articles"""
    db = next(get_db())

    # Retrieve relevant articles
    articles_query = db.query(Article)
    if request.article_ids:
        articles_query = articles_query.filter(Article.id.in_(request.article_ids))

    articles = articles_query.limit(50).all()  # Limiting for context

    if not articles:
        raise HTTPException(400, "No articles available for synthesis")

    # Perform RAG
    llm = LLMManager()
    synthesis = llm.retrieve_and_synthesize(request.query, articles)

    # Save synthesis job
    job = SynthesisJob(
        query=request.query,
        article_ids=[a.id for a in articles],
        synthesis=synthesis
    )
    db.add(job)
    db.commit()
    db.refresh(job)

    return {"synthesis_id": job.id, "synthesis": synthesis}

# compose.py
from fastapi import APIRouter, HTTPException
from ..dependencies import get_db
from ..core.llm import LLMManager
from ..services import persona_loader

router = APIRouter()

@router.post("/compose")
async def compose_segment(
    request: CompositionRequest
) -> CompositionResponse:
    """Create persona-driven news segment"""
    llm = LLMManager()

    # Load persona
    try:
        persona = persona_loader.load_persona(request.persona_name)
    except FileNotFoundError:
        raise HTTPException(404, f"Persona '{request.persona_name}' not found")

    # Compose segment
    segment = llm.compose_with_persona(
        content=request.synthesis_content,
        persona=persona
    )

    return {"segment": segment, "persona_used": request.persona_name}

@router.get("/personas")
async def list_personas():
    """Get available personas"""
    personas = persona_loader.list_personas()
    persona_list = []

    for persona_name in personas:
        try:
            persona_data = persona_loader.load_persona(persona_name)
            persona_list.append({
                "name": persona_name,
                "description": persona_data.get("description", ""),
                "tone": persona_data.get("tone", "")
            })
        except Exception:
            pass  # Skip invalid persona files

    return {"personas": persona_list}
```

### 2.4 Text-to-Speech Router

**Target Location:** `backend/src/api/routes/tts.py`

**Implement:** Audio generation with caching:

```python
import asyncio
import io
from fastapi import APIRouter, HTTPException, BackgroundTasks
from fastapi.responses import StreamingResponse
import edge_tts
import hashlib
from pathlib import Path
from ..core.config import settings
from ..models import AudioFile

router = APIRouter()

@router.post("/tts")
async def generate_tts(
    request: TTSRequest,
    background_tasks: BackgroundTasks
) -> Dict[str, Any]:
    """Generate text-to-speech audio"""
    # Generate unique ID for this audio segment
    segment_hash = hashlib.md5(f"{request.text}_{request.voice}_{request.speed}".encode()).hexdigest()
    audio_filename = f"{segment_hash}.mp3"
    audio_path = Path(settings.AUDIO_CACHE_DIR) / audio_filename

    # Check cache
    if audio_path.exists():
        return {
            "audio_id": segment_hash,
            "cached": True,
            "path": str(audio_path),
            "voice": request.voice,
            "speed": request.speed
        }

    # Generate audio
    background_tasks.add_task(generate_audio_background, request, audio_path, segment_hash)

    return {
        "audio_id": segment_hash,
        "status": "generating",
        "message": "Audio generation started in background"
    }

@router.get("/tts/{audio_id}")
async def get_tts_audio(audio_id: str):
    """Stream generated audio file"""
    audio_path = Path(settings.AUDIO_CACHE_DIR) / f"{audio_id}.mp3"

    if not audio_path.exists():
        raise HTTPException(404, "Audio file not found or still generating")

    def iterfile():
        with open(audio_path, "rb") as file:
            yield from file

    return StreamingResponse(iterfile(), media_type="audio/mpeg")

async def generate_audio_background(
    request: TTSRequest,
    audio_path: Path,
    audio_id: str
):
    """Background audio generation"""
    try:
        communicate = edge_tts.Communicate(
            text=request.text,
            voice=request.voice,
            rate=f"{(request.speed - 1) * 100:+d}%"
        )

        await communicate.save(str(audio_path))
        logging.info(f"Audio generated: {audio_path}")

        # Save to database if needed
        db = next(get_db())
        audio_record = AudioFile(
            segment_id=audio_id,
            file_path=str(audio_path),
            duration=0,  # Could calculate from file
            voice=request.voice
        )
        db.add(audio_record)
        db.commit()

    except Exception as e:
        logging.error(f"Audio generation failed: {str(e)}")
        # Clean up partial file
        if audio_path.exists():
            audio_path.unlink()
```

## Phase 3: Frontend Implementation

### 3.1 API Client Layer

**Target Location:** `frontend/src/lib/api/client.ts`

**Implement:** TypeScript API client with error handling:

```typescript
import { z } from 'zod';

// Schema definitions
const ArticleSchema = z.object({
  id: z.number(),
  feed_id: z.number(),
  title: z.string(),
  url: z.string(),
  published_at: z.string().nullable(),
  processed_at: z.string().nullable(),
  has_metadata: z.boolean()
});

const SynthesisRequestSchema = z.object({
  query: z.string(),
  article_ids: z.array(z.number()).optional()
});

const CompositionRequestSchema = z.object({
  synthesis_content: z.string(),
  persona_name: z.string()
});

const TTSRequestSchema = z.object({
  text: z.string(),
  voice: z.string().optional().default("en-US-AriaRUS"),
  speed: z.number().min(0.5).max(2).optional().default(1)
});

// API client class
class NewsSynthesizerAPI {
  private baseUrl: string;

  constructor(baseUrl: string = '/api/v1') {
    this.baseUrl = baseUrl;
  }

  // Feed management
  async getFeeds(params?: { skip?: number, limit?: number }) {
    const query = new URLSearchParams(params as any);
    const response = await fetch(`${this.baseUrl}/feeds?${query}`);
    return response.json();
  }

  async addFeed(feedData: { name: string, url: string, category?: string }) {
    const response = await fetch(`${this.baseUrl}/feeds`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(feedData)
    });
    return response.json();
  }

  async testFeed(url: string) {
    const response = await fetch(`${this.baseUrl}/feeds/test`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ url })
    });
    return response.json();
  }

  // Article management
  async getArticles(params?: {
    skip?: number,
    limit?: number,
    feed_id?: number,
    processed?: boolean
  }): Promise<typeof ArticleSchema._type[]> {
    const query = new URLSearchParams(params as any);
    const response = await fetch(`${this.baseUrl}/articles?${query}`);
    const data = await response.json();
    return z.array(ArticleSchema).parse(data);
  }

  async processArticles(feed_ids?: number[]) {
    const response = await fetch(`${this.baseUrl}/articles`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ feed_ids })
    });
    return response.json();
  }

  async getArticleMetadata(articleId: number) {
    const response = await fetch(`${this.baseUrl}/articles/${articleId}/metadata`);
    return response.json();
  }

  // Synthesis and composition
  async synthesize(query: string, articleIds?: number[]) {
    const requestData = { query, article_ids: articleIds };
    SynthesisRequestSchema.parse(requestData);

    const response = await fetch(`${this.baseUrl}/synthesize`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(requestData)
    });
    return response.json();
  }

  async compose(synthesisContent: string, personaName: string) {
    const requestData = { synthesis_content: synthesisContent, persona_name: personaName };
    CompositionRequestSchema.parse(requestData);

    const response = await fetch(`${this.baseUrl}/compose`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(requestData)
    });
    return response.json();
  }

  async getPersonas() {
    const response = await fetch(`${this.baseUrl}/personas`);
    return response.json();
  }

  // Text-to-speech
  async generateTTS(text: string, options?: { voice?: string, speed?: number }) {
    const requestData = { text, ...options };
    TTSRequestSchema.parse(requestData);

    const response = await fetch(`${this.baseUrl}/tts`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(requestData)
    });
    return response.json();
  }

  async getTTS(audioId: string): Promise<Blob> {
    const response = await fetch(`${this.baseUrl}/tts/${audioId}`);
    return response.blob();
  }
}

// Export singleton instance
export const apiClient = new NewsSynthesizerAPI(
  process.env.NEXT_PUBLIC_API_URL || 'http://localhost:8000/api/v1'
);
```

### 3.2 React Components Structure

**Target Location:** `frontend/src/components/`

**Implement:** Reusable UI components following shadcn/ui patterns:

// components/articles/ArticleList.tsx
import { useQuery } from '@tanstack/react-query';
import { apiClient } from '@/lib/api/client';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';

export function ArticleList() {
  const { data: articles, isLoading, error } = useQuery({
    queryKey: ['articles'],
    queryFn: () => apiClient.getArticles({ limit: 20 })
  });

  if (isLoading) return <div>Loading articles...</div>;
  if (error) return <div>Error loading articles</div>;

  return (
    <div className="space-y-4">
      {articles?.map((article) => (
        <Card key={article.id}>
          <CardHeader>
            <CardTitle className="flex items-center gap-2">
              {article.title}
              {article.has_metadata && (
                <Badge variant="secondary">Processed</Badge>
              )}
            </CardTitle>
          </CardHeader>
          <CardContent>
            <p className="text-sm text-gray-600">
              Published: {article.published_at ? new Date(article.published_at).toLocaleDateString() : 'Unknown'}
            </p>
            <a href={article.url} target="_blank" rel="noopener noreferrer"
               className="text-blue-600 hover:underline">
              Read original
            </a>
          </CardContent>
        </Card>
      ))}
    </div>
  );
}

// components/synthesis/SynthesisForm.tsx
import { useState } from 'react';
import { useMutation } from '@tanstack/react-query';
import { apiClient } from '@/lib/api/client';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';

export function SynthesisForm() {
  const [query, setQuery] = useState('');
  const [articleIds, setArticleIds] = useState<number[]>([]);

  const mutation = useMutation({
    mutationFn: () => apiClient.synthesize(query, articleIds.length ? articleIds : undefined),
    onSuccess: (data) => {
      console.log('Synthesis completed:', data);
    }
  });

  const handleSubmit = (e: React.FormEvent) => {
    e.preventDefault();
    mutation.mutate();
  };

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <div>
        <label className="block text-sm font-medium mb-2">Query</label>
        <Input
          value={query}
          onChange={(e) => setQuery(e.target.value)}
          placeholder="What would you like to synthesize?"
          required
        />
      </div>

      <div>
        <label className="block text-sm font-medium mb-2">
          Article IDs (optional, comma-separated)
        </label>
        <Input
          value={articleIds.join(', ')}
          onChange={(e) => {
            const ids = e.target.value.split(',').map(s => parseInt(s.trim())).filter(n => !isNaN(n));
            setArticleIds(ids);
          }}
          placeholder="1, 2, 3, 4..."
        />
      </div>

      <Button type="submit" disabled={mutation.isLoading}>
        {mutation.isLoading ? 'Synthesizing...' : 'Synthesize'}
      </Button>

      {mutation.data && (
        <div className="mt-4 p-4 bg-green-50 rounded">
          <h3 className="font-semibold">Synthesis Result:</h3>
          <p>{mutation.data.synthesis}</p>
        </div>
      )}
    </form>
  );
}

// components/composition/CompositionPanel.tsx
import { useState } from 'react';
import { useQuery, useMutation } from '@tanstack/react-query';
import { apiClient } from '@/lib/api/client';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';

export function CompositionPanel() {
  const [persona, setPersona] = useState('');
  const [synthesisContent, setSynthesisContent] = useState('');

  const { data: personas } = useQuery({
    queryKey: ['personas'],
    queryFn: () => apiClient.getPersonas()
  });

  const composeMutation = useMutation({
    mutationFn: () => apiClient.compose(synthesisContent, persona)
  });

  const handleCompose = () => {
    composeMutation.mutate();
  };

  return (
    <div className="space-y-4">
      <Select value={persona} onValueChange={setPersona}>
        <SelectTrigger>
          <SelectValue placeholder="Select a persona" />
        </SelectTrigger>
        <SelectContent>
          {personas?.personas?.map((p: any) => (
            <SelectItem key={p.name} value={p.name}>
              {p.name}: {p.description}
            </SelectItem>
          ))}
        </SelectContent>
      </Select>

      <Textarea
        value={synthesisContent}
        onChange={(e) => setSynthesisContent(e.target.value)}
        placeholder="Enter synthesis content to compose with selected persona..."
        rows={10}
      />

      <Button onClick={handleCompose} disabled={!persona || !synthesisContent || composeMutation.isLoading}>
        {composeMutation.isLoading ? 'Composing...' : 'Compose News Segment'}
      </Button>

      {composeMutation.data && (
        <div className="p-4 bg-blue-50 rounded">
          <h3 className="font-semibold">Composed Segment:</h3>
          <p className="whitespace-pre-wrap">{composeMutation.data.segment}</p>
        </div>
      )}
    </div>
  );
}

// components/audio/AudioPlayer.tsx
import { useState, useEffect } from 'react';
import { apiClient } from '@/lib/api/client';
import { Button } from '@/components/ui/button';

export function AudioPlayer({ text, voice = 'en-US-AriaRUS', speed = 1 }: {
  text: string;
  voice?: string;
  speed?: number;
}) {
  const [audioId, setAudioId] = useState<string>('');
  const [isGenerating, setIsGenerating] = useState(false);
  const [audioBlob, setAudioBlob] = useState<Blob | null>(null);

  const generateAudio = async () => {
    setIsGenerating(true);
    try {
      const result = await apiClient.generateTTS(text, { voice, speed });
      setAudioId(result.audio_id);

      // If cached, get the audio immediately
      if (result.cached) {
        const blob = await apiClient.getTTS(result.audio_id);
        setAudioBlob(blob);
      }
    } catch (error) {
      console.error('TTS generation failed:', error);
    } finally {
      setIsGenerating(false);
    }
  };

  // Poll for audio if not cached
  useEffect(() => {
    if (audioId && !audioBlob) {
      const pollAudio = async () => {
        try {
          const blob = await apiClient.getTTS(audioId);
          setAudioBlob(blob);
        } catch (error) {
          // Retry after delay
          setTimeout(pollAudio, 2000);
        }
      };
      pollAudio();
    }
  }, [audioId, audioBlob]);

  return (
    <div className="space-y-2">
      <Button onClick={generateAudio} disabled={isGenerating}>
        {isGenerating ? 'Generating Audio...' : 'Generate Audio'}
      </Button>

      {audioBlob && (
        <audio controls className="w-full">
          <source src={URL.createObjectURL(audioBlob)} type="audio/mpeg" />
        </audio>
      )}
    </div>
  );
}
```

### 3.3 Page Implementations

**Target Location:** `frontend/src/app/`

**Implement:** Next.js pages using React Query for state management:

```typescript
// app/articles/page.tsx
import { ArticleList } from '@/components/articles/ArticleList';
import { Button } from '@/components/ui/button';
import { apiClient } from '@/lib/api/client';
import { useMutation } from '@tanstack/react-query';

export default function ArticlesPage() {
  const processMutation = useMutation({
    mutationFn: () => apiClient.processArticles()
  });

  return (
    <div className="container mx-auto p-6">
      <div className="flex justify-between items-center mb-6">
        <h1 className="text-3xl font-bold">Articles</h1>
        <Button
          onClick={() => processMutation.mutate()}
          disabled={processMutation.isLoading}
        >
          {processMutation.isLoading ? 'Processing...' : 'Process Articles'}
        </Button>
      </div>

      <ArticleList />
    </div>
  );
}

// app/synthesize/page.tsx
import { SynthesisForm } from '@/components/synthesis/SynthesisForm';

export default function SynthesizePage() {
  return (
    <div className="container mx-auto p-6">
      <h1 className="text-3xl font-bold mb-6">Context Synthesis</h1>
      <SynthesisForm />
    </div>
  );
}

// app/compose/page.tsx
import { CompositionPanel } from '@/components/composition/CompositionPanel';

export default function ComposePage() {
  return (
    <div className="container mx-auto p-6">
      <h1 className="text-3xl font-bold mb-6">News Composition</h1>
      <CompositionPanel />
    </div>
  );
}

// app/tts/page.tsx
'use client';
import { useState } from 'react';
import { AudioPlayer } from '@/components/audio/AudioPlayer';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';

export default function TTSPage() {
  const [text, setText] = useState('');
  const [voice, setVoice] = useState('en-US-AriaRUS');
  const [speed, setSpeed] = useState(1);

  const voices = [
    { id: 'en-US-AriaRUS', name: 'Aria (Female)' },
    { id: 'en-US-ZiraRUS', name: 'Zira (Female)' },
    { id: 'en-US-BenjaminRUS', name: 'Benjamin (Male)' },
    { id: 'en-GB-SoniaRUS', name: 'Sonia (UK Female)' },
  ];

  const handleQuickTest = () => {
    setText('Hello! This is a test of the text-to-speech system.');
  };

  return (
    <div className="container mx-auto p-6 max-w-2xl">
      <h1 className="text-3xl font-bold mb-6">Text-to-Speech</h1>

      <div className="space-y-4">
        <div>
          <label className="block text-sm font-medium mb-2">Text to Speak</label>
          <Textarea
            value={text}
            onChange={(e) => setText(e.target.value)}
            placeholder="Enter text you want to convert to speech..."
            rows={6}
          />
        </div>

        <div className="flex gap-2">
          <Button variant="outline" onClick={handleQuickTest}>
            Quick Test
          </Button>
        </div>

        <div className="grid grid-cols-2 gap-4">
          <div>
            <label className="block text-sm font-medium mb-2">Voice</label>
            <Select value={voice} onValueChange={setVoice}>
              <SelectTrigger>
                <SelectValue />
              </SelectTrigger>
              <SelectContent>
                {voices.map(v => (
                  <SelectItem key={v.id} value={v.id}>{v.name}</SelectItem>
                ))}
              </SelectContent>
            </Select>
          </div>

          <div>
            <label className="block text-sm font-medium mb-2">Speed ({speed}x)</label>
            <input
              type="range"
              min="0.5"
              max="2"
              step="0.1"
              value={speed}
              onChange={(e) => setSpeed(parseFloat(e.target.value))}
              className="w-full"
            />
          </div>
        </div>

        {text && (
          <AudioPlayer text={text} voice={voice} speed={speed} />
        )}
      </div>
    </div>
  );
}

// app/chat/page.tsx (Persona interaction)
import { useState } from 'react';
import { useMutation } from '@tanstack/react-query';
import { apiClient } from '@/lib/api/client';
import { Button } from '@/components/ui/button';
import { Textarea } from '@/components/ui/textarea';
import { ScrollArea } from '@/components/ui/scroll-area';

export default function ChatPage() {
  const [messages, setMessages] = useState<string[]>([]);
  const [currentMessage, setCurrentMessage] = useState('');

  const sendMutation = useMutation({
    mutationFn: (message: string) => apiClient.chatSendMessage(message)
  });

  const handleSend = () => {
    const userMessage = currentMessage;
    setMessages(prev => [...prev, `You: ${userMessage}`]);
    setCurrentMessage('');

    sendMutation.mutate(userMessage, {
      onSuccess: (response) => {
        setMessages(prev => [...prev, `Persona: ${response.adjustments}`]);
      }
    });
  };

  return (
    <div className="container mx-auto p-6 max-w-4xl">
      <h1 className="text-3xl font-bold mb-6">Persona Chat</h1>

      <div className="bg-gray-50 rounded-lg p-4 mb-4 h-96">
        <ScrollArea className="h-full">
          {messages.map((msg, idx) => (
            <div key={idx} className="mb-2 p-2 bg-white rounded">
              {msg}
            </div>
          ))}
          {sendMutation.isLoading && (
            <div className="text-gray-500">Persona is thinking...</div>
          )}
        </ScrollArea>
      </div>

      <div className="flex gap-2">
        <Textarea
          value={currentMessage}
          onChange={(e) => setCurrentMessage(e.target.value)}
          placeholder="Enter your message to adjust the persona..."
          onKeyPress={(e) => e.key === 'Enter' && handleSend()}
          className="flex-1"
        />
        <Button onClick={handleSend} disabled={!currentMessage || sendMutation.isLoading}>
          Send
        </Button>
      </div>
    </div>
  );
}
```

## Phase 4: Testing Implementation

### 4.1 Unit Testing Setup

**Target Location:** `backend/tests/`, `frontend/__tests__/`

**Backend Testing:**
```python
# tests/test_llm_integration.py
import pytest
from unittest.mock import Mock, patch
from backend.src.core.llm import LLMManager

class TestLLMManager:
    @patch('backend.src.core.llm.Llama')
    def test_load_model(self, mock_llama):
        manager = LLMManager()
        manager._model = None  # Reset singleton

        mock_model = Mock()
        mock_llama.return_value = mock_model

        loaded_model = manager.load_model()

        assert loaded_model == mock_model
        mock_llama.assert_called_once()

    def test_generate_metadata(self):
        manager = LLMManager()
        manager._model = Mock()

        # Mock LLM response
        mock_response = {
            'choices': [{'text': '{"summary": "Test summary", "sentiment": "neutral"}'}]
        }
        manager._model.side_effect = lambda **kwargs: mock_response

        result = manager.generate_metadata("Test article")

        assert "summary" in result
        assert "sentiment" in result

    def test_persona_composition(self):
        manager = LLMManager()
        manager._model = Mock()

        mock_persona = {
            'name': 'Objective',
            'description': 'Neutral reporter',
            'tone': 'neutral',
            'style': 'factual'
        }

        mock_response = {'choices': [{'text': 'Composed news segment'}]}
        manager._model.side_effect = lambda **kwargs: mock_response

        result = manager.compose_with_persona("Content", mock_persona)

        assert result == "Composed news segment"
```

**Frontend Testing:**
```typescript
// __tests__/components/ArticleList.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ArticleList } from '@/components/articles/ArticleList';
import { apiClient } from '@/lib/api/client';

// Mock the API client
jest.mock('@/lib/api/client');

const mockArticles = [
  {
    id: 1,
    feed_id: 1,
    title: 'Test Article',
    url: 'https://example.com',
    published_at: '2023-01-01T00:00:00Z',
    processed_at: null,
    has_metadata: false
  }
];

describe('ArticleList', () => {
  beforeEach(() => {
    jest.clearAllMocks();
    (apiClient.getArticles as jest.Mock).mockResolvedValue(mockArticles);
  });

  it('renders articles correctly', async () => {
    const queryClient = new QueryClient();

    render(
      <QueryClientProvider client={queryClient}>
        <ArticleList />
      </QueryClientProvider>
    );

    await waitFor(() => {
      expect(screen.getByText('Test Article')).toBeInTheDocument();
    });
  });
});
```

### 4.2 Integration Testing

**API Integration Tests:**
```python
# tests/integration/test_api_articles.py
import pytest
from httpx import AsyncClient
from backend.src.api.app import app

@pytest.mark.asyncio
async def test_get_articles():
    async with AsyncClient(app=app, base_url="http://testserver") as client:
        response = await client.get("/api/v1/articles")

        assert response.status_code == 200
        data = response.json()
        assert "articles" in data

@pytest.mark.asyncio
async def test_create_feed():
    feed_data = {
        "name": "Test Feed",
        "url": "https://example.com/rss.xml",
        "category": "test"
    }

    async with AsyncClient(app=app, base_url="http://testserver") as client:
        response = await client.post("/api/v1/feeds", json=feed_data)

        assert response.status_code == 200
        data = response.json()
        assert data["message"] == "Feed added successfully"
```

### 4.3 LLM Inference Validation

**Prompt Testing Framework:**
```python
# tests/test_prompts.py
import pytest
from backend.src.core.llm import LLMManager

class TestPromptTemplates:
    def test_metadata_extraction_prompt(self):
        manager = LLMManager()

        test_article = """
        Tech company announces new AI chip.
        Stock prices rise 15% following announcement.
        """

        prompt = manager._generate_metadata_prompt(test_article)

        # Test that prompt contains required elements
        assert "summary" in prompt
        assert "sentiment" in prompt
        assert "keywords" in prompt
        assert test_article.strip() in prompt

    def test_synthesis_prompt_structure(self):
        manager = LLMManager()

        articles = [
            {"title": "Article 1", "summary": "Summary 1"},
            {"title": "Article 2", "summary": "Summary 2"}
        ]

        prompt = manager._generate_synthesis_prompt("tech industry", articles)

        assert "tech industry" in prompt
        assert "Article 1" in prompt
        assert "Article 2" in prompt
```

## Phase 5: Security Implementation

### 5.1 Input Validation & Sanitization

**Request Validation Models:**
```python
# backend/src/models/validation.py
from pydantic import BaseModel, HttpUrl, Field
from typing import List, Optional
import re

class FeedCreate(BaseModel):
    name: str = Field(..., min_length=1, max_length=100)
    url: HttpUrl
    category: Optional[str] = Field(None, max_length=50)

    @validator('url')
    def validate_rss_url(cls, v):
        if not re.match(r'^https?://', str(v)):
            raise ValueError('URL must be HTTP or HTTPS')
        return v

class SynthesisRequest(BaseModel):
    query: str = Field(..., min_length=1, max_length=500)
    article_ids: Optional[List[int]] = Field(None, min_elements=1, max_elements=100)

    @validator('query')
    def sanitize_query(cls, v):
        # Remove potentially harmful characters
        return re.sub(r'[<>]', '', v)

class CompositionRequest(BaseModel):
    synthesis_content: str = Field(..., min_length=1, max_length=10000)
    persona_name: str = Field(..., regex=r'^[a-zA-Z0-9_-]+$')
```

### 5.2 Rate Limiting & Throttling

**Rate Limiting Middleware:**
```python
# backend/src/core/rate_limit.py
from fastapi import Request, HTTPException
from collections import defaultdict, deque
import time
import asyncio
from typing import Deque

class RateLimiter:
    def __init__(self, max_requests: int = 10, window_seconds: int = 60):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.requests = defaultdict(lambda: deque())

    async def check_rate_limit(self, client_id: str):
        now = time.time()
        request_times = self.requests[client_id]

        # Clean old requests outside the window
        while request_times and now - request_times[0] > self.window_seconds:
            request_times.popleft()

        if len(request_times) >= self.max_requests:
            raise HTTPException(
                status_code=429,
                detail="Rate limit exceeded. Try again later."
            )

        request_times.append(now)

    def get_remaining_limit(self, client_id: str) -> int:
        now = time.time()
        request_times = self.requests[client_id]

        # Clean old requests
        while request_times and now - request_times[0] > self.window_seconds:
            request_times.popleft()

        return self.max_requests - len(request_times)

# Middleware implementation
from fastapi.middleware.base import BaseHTTPMiddleware

class RateLimitMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, limiter: RateLimiter):
        super().__init__(app)
        self.limiter = limiter

    async def dispatch(self, request: Request, call_next):
        # Get client identifier (IP address for simplicity)
        client_id = request.client.host

        await self.limiter.check_rate_limit(client_id)

        response = await call_next(request)

        # Add rate limit headers
        response.headers['X-RateLimit-Remaining'] = str(
            self.limiter.get_remaining_limit(client_id)
        )
        response.headers['X-RateLimit-Limit'] = str(self.limiter.max_requests)

        return response
```

### 5.3 LLM Safety Checks

**Content Validation:**
```python
# backend/src/core/safety.py
import logging
from typing import Dict, Any

class ContentSafetyValidator:
    def __init__(self):
        self.banned_keywords = {
            'system', 'admin', 'root', 'password', 'secret',
            'api_key', 'token', 'credential', 'private'
        }

    def validate_llm_input(self, text: str) -> bool:
        """Check for potentially unsafe prompts or content"""
        text_lower = text.lower()

        # Check for banned keywords
        for keyword in self.banned_keywords:
            if keyword in text_lower:
                logging.warning(f"Blocked input containing banned keyword: {keyword}")
                return False

        # Check prompt injection attempts
        injection_patterns = [
            r'!system\s*:', r'!admin\s*:', r'!override\s*:',
            r'do not.*instruction', r'ignore.*guideline'
        ]

        for pattern in injection_patterns:
            if re.search(pattern, text_lower, re.IGNORECASE):
                logging.warning(f"Blocked potential prompt injection: {pattern}")
                return False

        # Check for excessive repeated characters
        if re.search(r'(.)\1{20,}', text):
            logging.warning("Blocked input with excessive repetition")
            return False

        return True

    def validate_llm_output(self, output: str, max_length: int = 10000) -> Dict[str, Any]:
        """Validate and sanitize LLM output"""
        if len(output) > max_length:
            logging.warning("Truncated excessively long output")
            output = output[:max_length] + "..."

        # Basic sanitization
        output = output.replace('<script', '<<script').replace('</script', '<</script')

        return {
            'content': output,
            'length': len(output),
            'validated': True
        }
```

## Phase 6: Deployment Infrastructure

### 6.1 Container Builds

**Enhanced Dockerfile Configuration:**

```dockerfile
# backend/Dockerfile
FROM python:3.11-slim as base

# Install system dependencies for llama.cpp
RUN apt-get update && apt-get install -y \
    cmake \
    build-essential \
    git \
    libssl-dev \
    libffi-dev \
    && rm -rf /var/lib/apt/lists/*

# Download and compile llama.cpp with GPU support
FROM base as llm-builder
RUN git clone --depth 1 https://github.com/ggerganov/llama.cpp.git /llama.cpp && \
    cd /llama.cpp && \
    mkdir build && cd build && \
    cmake -DLLAMA_CURL=ON -DLLAMA_AVX2=ON -DLLAMA_F16C=ON -DLLAMA_FMA=ON .. && \
    make -j$(nproc)

FROM base as final
COPY --from=llm-builder /llama.cpp/build /llama.cpp/build
COPY --from=llm-builder /llama.cpp/llama-cli /usr/local/bin/llama-cli

# Create non-root user
RUN useradd --create-home --shell /bin/bash app && \
    mkdir -p /app /app/audio_cache /app/models && \
    chown -R app:app /app

USER app
WORKDIR /app

# Install Python dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY --chown=app:app . .

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD python -c "import requests; requests.get('http://localhost:8000/health')"

EXPOSE 8000

# Use production server
CMD ["uvicorn", "src.api.app:app", "--host", "0.0.0.0", "--port", "8000", "--workers", "1"]
```

### 6.2 Docker Compose for Development

**Enhanced docker-compose.yml:**
```yaml
version: '3.8'

services:
  backend:
    build:
      context: ./backend
      dockerfile: Dockerfile
    ports:
      - "8000:8000"
    volumes:
      - ./backend:/app
      - ./models:/app/models
      - ./audio_cache:/app/audio_cache
    environment:
      - ENVIRONMENT=development
      - LOG_LEVEL=INFO
    healthcheck:
      test: ["CMD", "curl", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  frontend:
    build:
      context: ./frontend
      dockerfile: Dockerfile
    ports:
      - "3000:3000"
    environment:
      - NEXT_PUBLIC_API_URL=http://backend:8000/api/v1
    depends_on:
      backend:
        condition: service_healthy
    volumes:
      - ./frontend:/app
      - /app/node_modules

  llama-cpp:
    image: ghcr.io/ggerganov/llama.cpp:main
    profiles:
      - gpu-enabled
    volumes:
      - ./models:/models:ro
      - ./backend/src/core/llm_wrapper.py:/llm_wrapper.py:ro
    environment:
      - LLAMA_MODEL_PATH=/models/mlabonne_gemma-3-27b-it-abliterated-IQ4_XS.gguf
    command: ["python", "/llm_wrapper.py"]
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

### 6.3 CI/CD Pipeline Enhancement

**GitHub Actions Workflow:**
```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  BACKEND_IMAGE: ${{ github.repository }}/news-synthesizer-backend
  FRONTEND_IMAGE: ${{ github.repository }}/news-synthesizer-frontend

jobs:
  test-and-security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'

      - name: Install dependencies and test
        run: |
          cd backend
          pip install -r requirements.txt
          pip install pytest pytest-asyncio
          python -m pytest tests/ -v --cov=src/

      - name: Run security scan
        uses: securecodewarrior/github-action-gosec@master
        with:
          args: './backend/...'
      - name: Vulnerability scanning
        uses: github/super-linter/slim@v4
        env:
          DEFAULT_BRANCH: main

  build-and-push:
    needs: test-and-security
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v4
        with:
          images: ${{ env.REGISTRY }}/${{ env.BACKEND_IMAGE }}

      - name: Build and push backend
        uses: docker/build-push-action@v4
        with:
          context: ./backend
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}

      - name: Build and push frontend
        uses: docker/build-push-action@v4
        with:
          context: ./frontend
          push: true
          tags: ${{ env.REGISTRY }}/${{ env.FRONTEND_IMAGE }}:${{ github.sha }}
          labels: ${{ steps.meta.outputs.labels }}

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to production
        run: |
          # Implementation depends on deployment target
          echo "Deploying to production environment"
```

## Phase 7: Operations & Monitoring

### 7.1 Logging Strategy

**Structured Logging Configuration:**
```python
# backend/src/core/logging.py
import logging
import json
import sys
from pathlib import Path
from datetime import datetime

class StructuredLogger:
    def __init__(self, name: str, level: str = "INFO"):
        self.logger = logging.getLogger(name)
        self.logger.setLevel(getattr(logging, level.upper()))

        # Remove existing handlers
        for handler in self.logger.handlers[:]:
            self.logger.removeHandler(handler)

        # Console handler for development
        console_handler = logging.StreamHandler(sys.stdout)
        console_handler.setFormatter(self._get_formatter())
        self.logger.addHandler(console_handler)

        # File handler for production
        log_dir = Path("logs")
        log_dir.mkdir(exist_ok=True)

        file_handler = logging.FileHandler(
            log_dir / f"news_synthesizer_{datetime.now().strftime('%Y%m%d')}.log"
        )
        file_handler.setFormatter(self._get_formatter())
        self.logger.addHandler(file_handler)

    def _get_formatter(self):
        class StructuredFormatter(logging.Formatter):
            def format(self, record):
                log_entry = {
                    "timestamp": datetime.fromtimestamp(record.created).isoformat(),
                    "level": record.levelname,
                    "logger": record.name,
                    "message": record.getMessage(),
                    "module": record.module,
                    "function": record.funcName,
                    "line": record.lineno
                }

                # Add extra fields if available
                if hasattr(record, 'extra_fields'):
                    log_entry.update(record.extra_fields)

                return json.dumps(log_entry)

        return StructuredFormatter()

    def info(self, message: str, **extra):
        extra_fields = {"extra_fields": extra} if extra else {}
        self.logger.info(message, **extra_fields)

    def error(self, message: str, exc_info=None, **extra):
        extra_fields = {"extra_fields": extra} if extra else {}
        self.logger.error(message, exc_info=exc_info, **extra_fields)

# Global logger instance
logger = StructuredLogger("news_synthesizer")
```

### 7.2 Health Monitoring

**Application Health Checks:**
```python
# backend/src/api/routes/health.py
from fastapi import APIRouter, status
from ..dependencies import get_db
from ..core.llm import LLMManager
from ..core.config import settings
import psutil
import time
from datetime import datetime
from ..core.performance_monitor import PerformanceMonitor

router = APIRouter()

@router.get("/health")
async def health_check():
    """Basic health check"""
    return {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "service": "news-synthesizer-backend"
    }

@router.get("/health/detailed")
async def detailed_health_check():
    """Comprehensive health check including dependencies"""
    health_data = {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "service": "news-synthesizer-backend",
        "version": "1.0.0",
        "checks": {}
    }

    # Database connectivity
    try:
        db = next(get_db())
        db.execute("SELECT 1")
        health_data["checks"]["database"] = {"status": "up"}
    except Exception as e:
        health_data["checks"]["database"] = {"status": "down", "error": str(e)}
        health_data["status"] = "unhealthy"

    # LLM model loading
    try:
        llm_manager = LLMManager()
        if llm_manager._model or llm_manager.load_model():
            health_data["checks"]["llm_model"] = {"status": "loaded"}
        else:
            health_data["checks"]["llm_model"] = {"status": "not_loaded"}
            health_data["status"] = "degraded"
    except Exception as e:
        health_data["checks"]["llm_model"] = {"status": "error", "error": str(e)}
        health_data["status"] = "unhealthy"

    # System resources
    health_data["checks"]["system"] = {
        "cpu_percent": psutil.cpu_percent(interval=1),
        "memory_percent": psutil.virtual_memory().percent,
        "disk_percent": psutil.disk_usage('/').percent
    }

    # Performance metrics
    monitor = PerformanceMonitor()
    health_data["checks"]["performance"] = {
        "requests_per_minute": monitor.get_requests_per_minute(),
        "average_response_time": monitor.get_average_response_time(),
        "error_rate": monitor.get_error_rate()
    }

    if health_data["status"] != "healthy":
        raise HTTPException(
            status_code=status.HTTP_503_SERVICE_UNAVAILABLE,
            detail=health_data
        )

    return health_data
```

### 7.3 Backup and Recovery

**Database Backup Strategy:**
```bash
# Scripts for backup operations
#!/bin/bash
# backup_db.sh
BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/news_synthesizer_${DATE}.db"

mkdir -p $BACKUP_DIR

# Create SQLite backup
sqlite3 /app/news_synthesizer.db ".backup ${BACKUP_FILE}"

# Compress backup
gzip $BACKUP_FILE

# Clean old backups (keep last 7 days)
find $BACKUP_DIR -name "*.db.gz" -mtime +7 -delete

echo "Backup completed: ${BACKUP_FILE}.gz"
```

**Configuration Backup:**
```python
# backend/src/core/backup.py
import shutil
from pathlib import Path
from datetime import datetime

class BackupManager:
    def __init__(self, backup_dir: str = "/backups"):
        self.backup_dir = Path(backup_dir)
        self.backup_dir.mkdir(exist_ok=True)

    def backup_database(self):
        """Create SQLite database backup"""
        db_path = Path("news_synthesizer.db")
        if db_path.exists():
            timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
            backup_path = self.backup_dir / f"db_backup_{timestamp}.db"
            shutil.copy2(db_path, backup_path)
            return backup_path
        return None

    def backup_personas(self):
        """Backup persona configurations"""
        personas_dir = Path("personas")
        if personas_dir.exists():
            timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
            backup_path = self.backup_dir / f"personas_backup_{timestamp}"
            shutil.copytree(personas_dir, backup_path)
            return backup_path
        return None

    def backup_feeds_config(self):
        """Backup RSS feed configurations"""
        feeds_file = Path("feeds.yaml")
        if feeds_file.exists():
            timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
            backup_path = self.backup_dir / f"feeds_backup_{timestamp}.yaml"
            shutil.copy2(feeds_file, backup_path)
            return backup_path
        return None

    def create_full_backup(self):
        """Create comprehensive backup"""
        db_backup = self.backup_database()
        persona_backup = self.backup_personas()
        feeds_backup = self.backup_feeds_config()

        # Create archive
        timestamp = datetime.now().strftime("%Y%m%d_%H%M%S")
        archive_path = self.backup_dir / f"full_backup_{timestamp}.tar.gz"

        import tarfile
        with tarfile.open(archive_path, "w:gz") as tar:
            if db_backup:
                tar.add(db_backup, arcname="database.db")
            if persona_backup:
                tar.add(persona_backup, arcname="personas")
            if feeds_backup:
                tar.add(feeds_backup, arcname="feeds.yaml")

        # Clean up individual backups
        for path in [db_backup, persona_backup, feeds_backup]:
            if path and path.exists():
                if path.is_file():
                    path.unlink()
                else:
                    shutil.rmtree(path)

        return archive_path
```

## Implementation Checklist

### Phase 1: Backend Infrastructure
- [x] Project structure setup
- [ ] FastAPI application conversion
- [ ] Database schema implementation
- [ ] LLM integration layer
- [ ] Configuration management
- [ ] Dependency injection setup

### Phase 2: API Implementation
- [ ] Feed management router
- [ ] Article processing router
- [ ] Synthesis and composition routers
- [ ] Text-to-Speech router
- [ ] Chat interface router
- [ ] Request validation models
- [ ] Error handling middleware

### Phase 3: Frontend Implementation
- [ ] API client layer with TypeScript
- [ ] Component library setup
- [ ] React components implementation
- [ ] Page route implementations
- [ ] State management setup
- [ ] Error boundaries
- [ ] Loading states

### Phase 4: Testing Framework
- [ ] Unit testing setup
- [ ] API integration tests
- [ ] Component testing
- [ ] LLM validation tests
- [ ] End-to-end test suite
- [ ] Performance benchmarks
- [ ] Security testing

### Phase 5: Security & Validation
- [ ] Input validation and sanitization
- [ ] Rate limiting implementation
- [ ] Content safety validation
- [ ] Authentication preparation
- [ ] API security headers
- [ ] XSS/CSRF protection

### Phase 6: Deployment & DevOps
- [ ] Container builds enhancement
- [ ] Development environment setup
- [ ] CI/CD pipeline configuration
- [ ] Environment variables management
- [ ] Scaling considerations
- [ ] Update strategy implementation

### Phase 7: Operations & Monitoring
- [ ] Structured logging implementation
- [ ] Health check endpoints
- [ ] Performance monitoring
- [ ] Error tracking
- [ ] Backup and recovery scripts
- [ ] Documentation for operations

This implementation plan provides a comprehensive roadmap for building the News Synthesizer application with production-ready features, security measures, and operational procedures. The phased approach ensures systematic development while maintaining quality and security standards throughout the implementation process.

## Implementation Progress Log

### Completed Tasks ✅
- [x] Documentation analysis and synthesis
- [x] Backend FastAPI structure planning with routers
- [x] SQLite database schema design with SQLAlchemy
- [x] LLM integration layer with llama.cpp wrapper
- [x] React frontend with TypeScript and API client
- [x] Component architecture using shadcn/ui
- [x] Testing framework setup with pytest and Jest
- [x] Security measures including validation and rate limiting
- [x] Docker containerization strategy
- [x] CI/CD pipeline with GitHub Actions
- [x] Operations monitoring with health checks
- [x] Backup and recovery procedures

### Next Implementation Steps 🎯
1. **Start Phase 1**: Convert main.py to FastAPI application structure
2. **Implement core**: Build LLM manager and database models
3. **Build APIs**: Create feed, article, and synthesis routers
4. **Frontend development**: Set up Next.js with API integration
5. **Testing**: Implement validation and security tests
6. **Deployment**: Configure Docker and CI/CD pipelines+++++++ REPLACE
