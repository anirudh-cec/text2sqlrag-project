# Local Docker Development Guide
## Multi-Source RAG + Text-to-SQL System

**Last Updated:** 2026-01-25
**Purpose:** Build and run the application locally using Docker for development and testing
**Dockerfile:** `Dockerfile` (local development, not `Dockerfile.lambda` for AWS)

---

## Table of Contents

1. [Quick Start](#1-quick-start)
2. [Prerequisites](#2-prerequisites)
3. [Environment Configuration](#3-environment-configuration)
4. [Building the Docker Image](#4-building-the-docker-image)
5. [Running the Container](#5-running-the-container)
6. [Testing the Local Deployment](#6-testing-the-local-deployment)
7. [Development Workflow](#7-development-workflow)
8. [Docker Compose Usage](#8-docker-compose-usage)
9. [Troubleshooting](#9-troubleshooting)
10. [Cleanup Commands](#10-cleanup-commands)
11. [Advanced Usage](#11-advanced-usage)

---

## 1. Quick Start

For experienced users who want to get started immediately:

```bash
# Clone repository (if not already done)
cd /path/to/text2sqlrag-project

# Create .env file
cp .env.example .env
# Edit .env with your API keys (OpenAI, Pinecone, Database URL)

# Build and run with Docker Compose (RECOMMENDED)
docker-compose up --build

# OR build and run manually
docker build -t rag-text-to-sql:local .
docker run -p 8000:8000 --env-file .env rag-text-to-sql:local

# Test the deployment
curl http://localhost:8000/health
```

Open your browser: http://localhost:8000/docs (Swagger UI)

---

## 2. Prerequisites

### Required Software

| Software | Minimum Version | Installation |
|----------|----------------|--------------|
| **Docker** | 20.10+ | [Install Docker](https://docs.docker.com/get-docker/) |
| **Docker Compose** | 2.0+ | Included with Docker Desktop |
| **Git** | 2.0+ | [Install Git](https://git-scm.com/downloads) |
| **curl** or **Postman** | Latest | For API testing |

### Required External Services

You need API keys from these services:

- **OpenAI** - For embeddings and LLM ([Get API key](https://platform.openai.com/api-keys))
- **Pinecone** - For vector storage ([Create account](https://www.pinecone.io/))
- **Supabase** - For PostgreSQL database ([Create project](https://supabase.com/))
- **(Optional)** Upstash Redis - For query caching ([Get credentials](https://upstash.com/))
- **(Optional)** OPIK - For LLM monitoring ([Create account](https://www.comet.com/opik))

### System Requirements

- **RAM:** Minimum 8 GB (16 GB recommended)
- **Disk Space:** Minimum 10 GB free
- **CPU:** 2+ cores (4+ recommended)
- **OS:** Linux, macOS, or Windows with WSL2

### Verify Docker Installation

```bash
# Check Docker version
docker --version
# Expected: Docker version 20.10.0 or higher

# Check Docker Compose version
docker-compose --version
# Expected: Docker Compose version v2.0.0 or higher

# Test Docker is running
docker run hello-world
# Expected: "Hello from Docker!" message
```

---

## 3. Environment Configuration

### Step 1: Create .env File

```bash
# Copy the example environment file
cp .env.example .env
```

### Step 2: Edit .env File

Open `.env` in your favorite editor and configure:

```bash
# Use your preferred editor
nano .env
# OR
vim .env
# OR
code .env  # VS Code
```

### Step 3: Configure Required Variables

**Minimum required configuration:**

```env
# OpenAI Configuration (REQUIRED)
OPENAI_API_KEY=sk-proj-your-actual-openai-key-here

# Pinecone Configuration (REQUIRED)
PINECONE_API_KEY=your-actual-pinecone-key-here
PINECONE_ENVIRONMENT=us-east-1-aws
PINECONE_INDEX_NAME=rag-documents

# Supabase PostgreSQL Configuration (REQUIRED)
DATABASE_URL=postgresql://postgres.PROJECT-REF:PASSWORD@aws-1-us-east-1.pooler.supabase.com:5432/postgres

# Storage Backend (for local development)
STORAGE_BACKEND=local

# Application Settings
ENVIRONMENT=development
```

**Optional configuration for advanced features:**

```env
# Upstash Redis (Query Caching - Optional)
UPSTASH_REDIS_URL=https://your-redis.upstash.io
UPSTASH_REDIS_TOKEN=your-token-here

# OPIK Monitoring (Optional)
OPIK_API_KEY=your-opik-key-here
OPIK_PROJECT_NAME=Multi-Source RAG

# Vanna SQL Configuration (Optional - uses defaults)
VANNA_MODEL=gpt-4o
VANNA_TEMPERATURE=0.0
VANNA_PINECONE_INDEX=vanna-sql-training
```

### Step 4: Verify .env File

```bash
# Check that .env has been created
ls -la .env

# View .env contents (be careful not to share!)
cat .env
```

⚠️ **IMPORTANT:** Never commit `.env` to Git! It's already in `.gitignore`.

---

## 4. Building the Docker Image

### Method 1: Build with Docker Command (Manual)

```bash
# Build the image with a tag
docker build -t rag-text-to-sql:local .

# Build with verbose output (see each step)
docker build -t rag-text-to-sql:local --progress=plain .

# Build without cache (clean build)
docker build -t rag-text-to-sql:local --no-cache .
```

**Build time:** 5-10 minutes (first build), 1-2 minutes (subsequent builds with cache)

**Expected output:**
```
[+] Building 342.5s (17/17) FINISHED
 => [internal] load build definition from Dockerfile
 => => transferring dockerfile: 1.23kB
 => [internal] load .dockerignore
 => [base 1/2] FROM python:3.12-slim
 => [base 2/2] RUN apt-get update && apt-get install -y ...
 => [builder 1/3] WORKDIR /app
 => [builder 2/3] COPY requirements.txt .
 => [builder 3/3] RUN pip install --user ...
 => [runtime 1/5] WORKDIR /app
 => [runtime 2/5] COPY --from=builder /root/.local /root/.local
 => [runtime 3/5] COPY app/ ./app/
 => [runtime 4/5] COPY data/sql/ ./data/sql/
 => [runtime 5/5] RUN mkdir -p data/uploads
 => exporting to image
 => => naming to docker.io/library/rag-text-to-sql:local

✅ Successfully built rag-text-to-sql:local
```

### Method 2: Build with Docker Compose (Recommended)

```bash
# Build the image using docker-compose
docker-compose build

# Build without cache
docker-compose build --no-cache

# Build with specific service (if you have multiple services)
docker-compose build app
```

### Verify the Built Image

```bash
# List Docker images
docker images | grep rag-text-to-sql

# Expected output:
# rag-text-to-sql   local    abc123def456   2 minutes ago   1.2GB
```

### Understanding the Dockerfile Stages

The `Dockerfile` uses a **multi-stage build** for optimization:

1. **Stage 1 (base):** System dependencies (libmagic, poppler, tesseract, OpenCV)
2. **Stage 2 (builder):** Python dependencies installation
3. **Stage 3 (runtime):** Final image with application code

**Benefits:**
- Smaller final image size (~1.2 GB vs ~2.5 GB single-stage)
- Faster builds with layer caching
- No build tools in production image

---

## 5. Running the Container

### Method 1: Run with Docker Command

**Basic run (foreground):**
```bash
docker run -p 8000:8000 --env-file .env rag-text-to-sql:local
```

**Run in background (detached mode):**
```bash
docker run -d \
  --name rag-app \
  -p 8000:8000 \
  --env-file .env \
  rag-text-to-sql:local
```

**Run with volume mounts (persist uploads):**
```bash
docker run -d \
  --name rag-app \
  -p 8000:8000 \
  --env-file .env \
  -v $(pwd)/data/uploads:/app/data/uploads \
  -v $(pwd)/data/cached_chunks:/app/data/cached_chunks \
  rag-text-to-sql:local
```

**Run with custom port:**
```bash
docker run -d \
  --name rag-app \
  -p 3000:8000 \
  --env-file .env \
  rag-text-to-sql:local

# Access at http://localhost:3000
```

**Run with environment variables (without .env file):**
```bash
docker run -d \
  --name rag-app \
  -p 8000:8000 \
  -e OPENAI_API_KEY="sk-proj-..." \
  -e PINECONE_API_KEY="..." \
  -e PINECONE_ENVIRONMENT="us-east-1-aws" \
  -e PINECONE_INDEX_NAME="rag-documents" \
  -e DATABASE_URL="postgresql://..." \
  -e STORAGE_BACKEND="local" \
  -e ENVIRONMENT="development" \
  rag-text-to-sql:local
```

### Method 2: Run with Docker Compose (Recommended)

**Start the application:**
```bash
# Start in foreground (see logs in terminal)
docker-compose up

# Start in background (detached)
docker-compose up -d

# Start and rebuild
docker-compose up --build

# Start with specific service
docker-compose up app
```

### Verify Container is Running

```bash
# List running containers
docker ps

# Expected output:
# CONTAINER ID   IMAGE                      STATUS         PORTS
# abc123def456   rag-text-to-sql:latest    Up 2 minutes   0.0.0.0:8000->8000/tcp

# Check container health
docker ps --format "table {{.Names}}\t{{.Status}}"

# View container logs
docker logs rag-app

# Follow logs in real-time
docker logs -f rag-app
```

### Container Startup Time

- **Cold start (first time):** 30-60 seconds
- **Warm start (subsequent runs):** 5-10 seconds

You'll know it's ready when you see:
```
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     Started reloader process [1] using StatReload
INFO:     Started server process [7]
INFO:     Waiting for application startup.
INFO:     ✓ Document RAG services initialized!
INFO:     ✓ Text-to-SQL service initialized!
INFO:     ✓ Cache service initialized!
INFO:     Application startup complete.
```

---

## 6. Testing the Local Deployment

### Test 1: Health Check Endpoint

```bash
# Check if application is running
curl http://localhost:8000/health

# Expected response:
{
  "status": "healthy",
  "service": "Multi-Source RAG + Text-to-SQL API",
  "timestamp": "2026-01-25T10:30:00.123456",
  "version": "0.1.0",
  "services": {
    "embedding_service": true,
    "vector_service": true,
    "rag_service": true,
    "sql_service": true,
    "query_cache": true
  }
}
```

### Test 2: Info Endpoint

```bash
# Get application information
curl http://localhost:8000/info | jq '.'

# Expected: Full application info with features and endpoints
```

### Test 3: Upload a Test Document

```bash
# Create a test document
cat > test-document.txt <<EOF
Company Policies:
- Remote work: 3 days per week allowed
- Vacation: 15 days per year for full-time employees
- Support hours: Monday-Friday 9 AM - 6 PM EST
EOF

# Upload the document
curl -X POST http://localhost:8000/upload \
  -F "file=@test-document.txt" | jq '.'

# Expected response:
{
  "status": "success",
  "filename": "test-document.txt",
  "document_id": "abc123...",
  "chunks_created": 1,
  "storage_backend": "local",
  "processing_time_ms": 1234
}
```

### Test 4: Query Documents

```bash
# Query the uploaded document
curl -X POST http://localhost:8000/query/documents \
  -H "Content-Type: application/json" \
  -d '{
    "query": "How many vacation days do full-time employees get?",
    "top_k": 3
  }' | jq '.'

# Expected: Answer about 15 days vacation
```

### Test 5: SQL Query

```bash
# Query the database
curl -X POST http://localhost:8000/query \
  -H "Content-Type: application/json" \
  -d '{
    "query": "How many customers do we have?",
    "auto_approve_sql": true
  }' | jq '.'

# Expected: SQL results with customer count
```

### Test 6: Interactive API Documentation

Open your browser and navigate to:

- **Swagger UI:** http://localhost:8000/docs
- **ReDoc:** http://localhost:8000/redoc

Try the interactive API explorer:
1. Click on an endpoint (e.g., `/health`)
2. Click "Try it out"
3. Click "Execute"
4. View the response

---

## 7. Development Workflow

### Hot Reload Development

For development with auto-reload on code changes:

**Option 1: Mount code as volume (recommended)**

```bash
docker run -d \
  --name rag-app-dev \
  -p 8000:8000 \
  --env-file .env \
  -v $(pwd)/app:/app/app \
  -v $(pwd)/data:/app/data \
  rag-text-to-sql:local
```

Now when you edit files in `app/`, the container will automatically reload!

**Option 2: Run uvicorn with reload flag**

Edit the Dockerfile `CMD` line temporarily:
```dockerfile
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]
```

Then rebuild:
```bash
docker-compose build
docker-compose up
```

### Typical Development Flow

```bash
# 1. Start container in background
docker-compose up -d

# 2. Watch logs in one terminal
docker-compose logs -f

# 3. Edit code in your IDE
code app/main.py

# 4. Test changes (auto-reloaded if using volumes)
curl http://localhost:8000/health

# 5. Check logs for errors
docker-compose logs --tail=50

# 6. Repeat steps 3-5

# 7. Stop container when done
docker-compose down
```

### Running Tests Inside Container

```bash
# Execute pytest inside running container
docker exec rag-app pytest tests/ -v

# Run specific test file
docker exec rag-app pytest tests/test_routes.py -v

# Run tests with coverage
docker exec rag-app pytest tests/ --cov=app --cov-report=html

# Run evaluation script
docker exec rag-app python evaluate.py
```

### Accessing Container Shell

```bash
# Get a bash shell inside the container
docker exec -it rag-app bash

# Once inside, you can:
# - Run Python REPL: python
# - Check installed packages: pip list
# - View files: ls -la
# - Run manual tests: curl http://localhost:8000/health
# - Exit: exit
```

### Viewing Application Logs

```bash
# View last 100 lines
docker logs rag-app --tail=100

# Follow logs in real-time
docker logs -f rag-app

# View logs with timestamps
docker logs -t rag-app

# View logs from last 10 minutes
docker logs --since 10m rag-app

# Search logs for errors
docker logs rag-app 2>&1 | grep -i "error"
```

---

## 8. Docker Compose Usage

### Basic Commands

```bash
# Start services (build if needed)
docker-compose up

# Start in background
docker-compose up -d

# Stop services (keeps containers)
docker-compose stop

# Stop and remove containers
docker-compose down

# Stop, remove containers, and remove volumes
docker-compose down -v

# Restart services
docker-compose restart

# View logs
docker-compose logs

# Follow logs
docker-compose logs -f

# View logs for specific service
docker-compose logs app

# Build images
docker-compose build

# Build without cache
docker-compose build --no-cache

# Pull latest images
docker-compose pull
```

### Service Management

```bash
# List running services
docker-compose ps

# Stop specific service
docker-compose stop app

# Start specific service
docker-compose start app

# Restart specific service
docker-compose restart app

# Remove stopped containers
docker-compose rm

# Scale services (if supported)
docker-compose up -d --scale app=3
```

### Debugging with Docker Compose

```bash
# Run a command in the service
docker-compose exec app bash

# Run command without starting service
docker-compose run app python --version

# View resource usage
docker-compose top

# Check configuration
docker-compose config

# Validate docker-compose.yml
docker-compose config --quiet && echo "Valid" || echo "Invalid"
```

---

## 9. Troubleshooting

### Problem 1: Port Already in Use

**Error:** `Bind for 0.0.0.0:8000 failed: port is already allocated`

**Solution:**
```bash
# Find process using port 8000
lsof -i :8000
# OR on Linux
sudo netstat -tulpn | grep :8000

# Kill the process
kill -9 <PID>

# OR use a different port
docker run -p 8001:8000 --env-file .env rag-text-to-sql:local
```

### Problem 2: Services Show as `false` in Health Check

**Error:** Health check returns `"embedding_service": false, ...`

**Solution:**
```bash
# Check container logs for errors
docker logs rag-app --tail=50

# Verify environment variables
docker exec rag-app env | grep -E "OPENAI|PINECONE|DATABASE"

# Check if API keys are correct
docker exec rag-app python -c "import os; print('OpenAI key:', os.getenv('OPENAI_API_KEY', 'NOT SET')[:15])"

# Restart with correct .env file
docker-compose down
docker-compose up -d
```

### Problem 3: Container Exits Immediately

**Error:** Container starts then immediately stops

**Solution:**
```bash
# Check exit code and logs
docker ps -a | grep rag
docker logs rag-app

# Common causes:
# - Missing .env file
# - Invalid environment variables
# - Python syntax errors in code

# Run in foreground to see errors
docker-compose up
```

### Problem 4: Out of Memory

**Error:** Container killed by OOM (Out of Memory)

**Solution:**
```bash
# Check Docker memory limits
docker stats rag-app

# Increase Docker Desktop memory:
# Docker Desktop → Settings → Resources → Memory → 8 GB+

# OR limit container memory
docker run -d \
  --name rag-app \
  -p 8000:8000 \
  --env-file .env \
  --memory="4g" \
  --memory-swap="6g" \
  rag-text-to-sql:local
```

### Problem 5: Cannot Connect to Database

**Error:** `could not connect to server` in logs

**Solution:**
```bash
# Check DATABASE_URL format
docker exec rag-app env | grep DATABASE_URL

# Test database connection from container
docker exec rag-app python -c "
import os
from sqlalchemy import create_engine
engine = create_engine(os.getenv('DATABASE_URL'))
print('Connection successful!' if engine.connect() else 'Failed')
"

# Ensure using Session Pooler (IPv4) for Supabase:
# ✅ postgres://...@aws-1-us-east-1.pooler.supabase.com:5432/postgres
# ❌ postgres://...@db.PROJECT-REF.supabase.co:5432/postgres
```

### Problem 6: Slow Container Startup

**Issue:** Container takes 2+ minutes to start

**Solutions:**
```bash
# Check if downloading models/data
docker logs -f rag-app

# Increase timeout in docker-compose.yml
healthcheck:
  start_period: 120s

# Pre-cache models locally (mount as volume)
mkdir -p ~/.cache/huggingface
docker run -v ~/.cache/huggingface:/root/.cache/huggingface ...
```

### Problem 7: Permission Denied Errors

**Error:** `Permission denied` when accessing files

**Solution:**
```bash
# Fix file permissions on host
chmod -R 755 data/uploads
chmod -R 755 data/cached_chunks

# OR run container as root (not recommended for production)
docker run --user root ...

# Check container user
docker exec rag-app whoami
docker exec rag-app id
```

### Problem 8: Docker Build Fails

**Error:** Build errors during `docker build`

**Solutions:**
```bash
# Build with verbose output
docker build -t rag-text-to-sql:local --progress=plain --no-cache .

# Check Docker disk space
docker system df

# Clean up to free space
docker system prune -a

# Check specific error in build log:
# - Python package conflicts → Update requirements.txt
# - System package not found → Update apt-get install line
# - Network timeout → Retry build
```

---

## 10. Cleanup Commands

### Stop and Remove Container

```bash
# Stop container
docker stop rag-app

# Remove container
docker rm rag-app

# Stop and remove in one command
docker rm -f rag-app

# Using docker-compose
docker-compose down
```

### Remove Docker Images

```bash
# Remove specific image
docker rmi rag-text-to-sql:local

# Remove all RAG images
docker images | grep rag-text-to-sql | awk '{print $3}' | xargs docker rmi

# Remove unused images
docker image prune

# Remove all unused images (be careful!)
docker image prune -a
```

### Clean Up Volumes

```bash
# List volumes
docker volume ls

# Remove specific volume
docker volume rm volume_name

# Remove all unused volumes
docker volume prune

# Remove volumes created by docker-compose
docker-compose down -v
```

### Complete Cleanup

```bash
# Remove everything (containers, images, volumes, networks)
docker-compose down -v --rmi all

# System-wide cleanup
docker system prune -a --volumes

# Check freed space
docker system df
```

---

## 11. Advanced Usage

### Build for Different Platforms

```bash
# Build for ARM64 (Apple M1/M2)
docker build --platform linux/arm64 -t rag-text-to-sql:arm64 .

# Build for x86-64 (Intel/AMD)
docker build --platform linux/amd64 -t rag-text-to-sql:amd64 .

# Multi-platform build (requires buildx)
docker buildx build \
  --platform linux/amd64,linux/arm64 \
  -t rag-text-to-sql:multi \
  --push .
```

### Export and Import Images

```bash
# Save image to tar file
docker save rag-text-to-sql:local -o rag-app.tar

# Load image from tar file
docker load -i rag-app.tar

# Transfer to another machine
scp rag-app.tar user@remote:/path/to/
```

### Resource Limits

```bash
# Run with CPU and memory limits
docker run -d \
  --name rag-app \
  -p 8000:8000 \
  --env-file .env \
  --cpus="2.0" \
  --memory="4g" \
  --memory-swap="6g" \
  rag-text-to-sql:local

# View resource usage
docker stats rag-app
```

### Network Configuration

```bash
# Create custom network
docker network create rag-network

# Run container on custom network
docker run -d \
  --name rag-app \
  --network rag-network \
  -p 8000:8000 \
  --env-file .env \
  rag-text-to-sql:local

# Connect additional containers
docker run -d \
  --name postgres \
  --network rag-network \
  postgres:15
```

### Debugging with Docker Exec

```bash
# Python REPL inside container
docker exec -it rag-app python

# Test imports
docker exec rag-app python -c "import app; print(app.__file__)"

# Check environment
docker exec rag-app env | sort

# Check disk usage
docker exec rag-app df -h

# Check process list
docker exec rag-app ps aux

# Network test
docker exec rag-app curl -s http://localhost:8000/health
```

### Using Docker with Makefile

Create a `Makefile` for convenience:

```makefile
.PHONY: build run stop clean logs shell test

build:
	docker-compose build

run:
	docker-compose up -d

stop:
	docker-compose down

clean:
	docker-compose down -v --rmi all

logs:
	docker-compose logs -f

shell:
	docker exec -it rag-text-to-sql bash

test:
	docker exec rag-text-to-sql pytest tests/ -v

health:
	curl http://localhost:8000/health | jq '.'

restart: stop run
```

Then use:
```bash
make build
make run
make logs
make shell
make test
```

---

## 📋 Quick Reference

### Essential Commands Cheatsheet

```bash
# Build and run (quick start)
docker-compose up --build -d

# View logs
docker-compose logs -f

# Stop everything
docker-compose down

# Restart
docker-compose restart

# Check health
curl http://localhost:8000/health

# Access shell
docker exec -it rag-text-to-sql bash

# Run tests
docker exec rag-text-to-sql pytest tests/ -v

# Clean up everything
docker-compose down -v --rmi all
docker system prune -a
```

### Port Mappings

| Service | Container Port | Host Port | URL |
|---------|---------------|-----------|-----|
| FastAPI App | 8000 | 8000 | http://localhost:8000 |
| API Docs (Swagger) | 8000 | 8000 | http://localhost:8000/docs |
| API Docs (ReDoc) | 8000 | 8000 | http://localhost:8000/redoc |

### Volume Mappings

| Host Path | Container Path | Purpose |
|-----------|---------------|---------|
| `./data/uploads` | `/app/data/uploads` | Uploaded documents |
| `./data/cached_chunks` | `/app/data/cached_chunks` | Embeddings cache |

### Environment Variables

See `.env.example` for complete list. Key variables:

| Variable | Required | Example |
|----------|----------|---------|
| `OPENAI_API_KEY` | ✅ Yes | `sk-proj-...` |
| `PINECONE_API_KEY` | ✅ Yes | `pcsk_...` |
| `DATABASE_URL` | ✅ Yes | `postgres://...` |
| `STORAGE_BACKEND` | No | `local` (default for dev) |
| `ENVIRONMENT` | No | `development` |

---

## 🎯 Common Use Cases

### Use Case 1: Testing Before AWS Deployment

```bash
# Build and test locally
docker-compose up --build

# Run integration tests
docker exec rag-text-to-sql pytest tests/ -v

# If tests pass, deploy to AWS
git push origin main  # Triggers GitHub Actions deployment
```

### Use Case 2: Development with Hot Reload

```bash
# Mount code directory for auto-reload
docker run -d \
  -p 8000:8000 \
  --env-file .env \
  -v $(pwd)/app:/app/app \
  --name rag-dev \
  rag-text-to-sql:local

# Edit code, see changes instantly
code app/main.py

# Watch logs
docker logs -f rag-dev
```

### Use Case 3: Debugging Production Issues

```bash
# Build with exact production environment
docker build -t rag-text-to-sql:debug .

# Run with production-like settings
docker run -d \
  -p 8000:8000 \
  -e ENVIRONMENT=production \
  -e STORAGE_BACKEND=s3 \
  --env-file .env.production \
  rag-text-to-sql:debug

# Access logs and shell
docker logs -f <container-id>
docker exec -it <container-id> bash
```

---

## 📚 Additional Resources

- **Docker Documentation:** https://docs.docker.com/
- **Docker Compose Documentation:** https://docs.docker.com/compose/
- **FastAPI Documentation:** https://fastapi.tiangolo.com/
- **Project README:** `../README.md`
- **AWS Deployment Guide:** `./new_deploy.md`

---

**Document Version:** 1.0
**Last Updated:** 2026-01-25
**Maintained By:** Development Team
**Questions?** Check the troubleshooting section or open an issue on GitHub
