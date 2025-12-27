# Deployment Strategy for News Synthesizer

## Deployment Philosophy
The application requires careful deployment due to local LLM model dependency, GPU requirements, and real-time processing needs. Focus on isolated environments, reliable model loading, and scalable RSS processing.

## Infrastructure Requirements

### Hardware Requirements
- **Minimum:** CPU with AVX2, 8GB RAM, 10GB storage
- **Recommended:** NVIDIA GPU (RTX 30-series+), 16GB+ RAM, SSD storage
- **Local Models:** GPU acceleration for llama.cpp inference

### Software Environment
- **Backend:** Python 3.8+, FastAPI, llama.cpp compiled with GPU support
- **Frontend:** Node.js 18+, Next.js build tools
- **Model:** Pre-downloaded GGUF file (mlabonne_gemma-3-27b-it-abliterated-IQ4_XS.gguf)

## Deployment Options

### 1. Local Development
- **Setup:**
  ```bash
  cd backend
  python -m venv venv
  source venv/bin/activate
  pip install -r requirements.txt
  cp llama.cpp path/to/compiled/llamacpp
  ```
- **Frontend:** `npm install && npm run dev`
- **Backend:** `uvicorn main:app --reload`

### 2. Docker Containerization
- **Backend Dockerfile:**
  ```dockerfile
  FROM python:3.11-slim
  # Install system dependencies for llama.cpp
  RUN apt-get update && apt-get install -y cmake build-essential
  # Copy llama.cpp and compile
  COPY llama.cpp /llama.cpp
  RUN cd /llama.cpp && mkdir build && cd build && cmake .. && make
  # Install Python dependencies
  COPY requirements.txt .
  RUN pip install -r requirements.txt
  # Copy model file
  COPY mlabonne_gemma-3-27b-it-abliterated-IQ4_XS.gguf /models/
  COPY . .
  CMD ["uvicorn", "main:app", "--host", "0.0.0.0"]
  ```
- **Frontend Dockerfile:**
  ```dockerfile
  FROM node:18-alpine
  COPY package*.json .
  RUN npm ci
  COPY . .
  RUN npm run build
  CMD ["npm", "start"]
  ```
- **Docker Compose:**
  ```yaml
  version: '3.8'
  services:
    backend:
      build: ./backend
      ports:
        - "8000:8000"
      volumes:
        - ./backend:/app
    frontend:
      build: ./frontend
      ports:
        - "3000:3000"
      environment:
        - NEXT_PUBLIC_API_URL=http://backend:8000
  ```

### 3. Production Deployment
- **Cloud Options:** AWS EC2 with GPU instances, Google Cloud Compute Engine
- **Cost Considerations:** GPU instances ($0.50+/hour), focus on spot instances
- **Scaling:** Horizontal scaling for multiple users, model loading optimization

## CI/CD Pipeline

### GitHub Actions Configuration
```yaml
name: CI/CD
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: pip install -r backend/requirements.txt
      - name: Run tests
        run: cd backend && pytest
  deploy:
    needs: test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to server
        # Add deployment steps (Docker build and push, etc.)
```

### Automated Pipelines
- **Build Step:** Compile llama.cpp if needed
- **Test Step:** Run unit/integration tests
- **Security Scan:** Dependency and code analysis
- **Deploy Step:** Docker image push to registry
- **Rollback:** Automated rollback on deployment failures

## Environment Configuration

### Environment Variables
- **Backend:**
  - `LLAMA_CPP_MODEL_PATH`: Path to .gguf file
  - `DATABASE_URL`: SQLite file location
  - `RSS_FETCH_CONCURRENCY`: Number of concurrent feed fetches
- **Frontend:**
  - `NEXT_PUBLIC_API_BASE_URL`: Backend API URL
  - `NEXT_PUBLIC_TTS_VOICE`: Default
