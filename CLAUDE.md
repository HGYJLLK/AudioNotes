# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AudioNotes is a Chinese audio/video transcription and note-taking system that combines FunASR (speech recognition) with Ollama-based LLM summarization. The application uses Chainlit v1.3+ for the web interface and PostgreSQL for data persistence.

**Note**: This codebase has been migrated from an older Chainlit version (0.7.x) to v1.3+. Key migration changes include:
- Replaced `ThreadDict` with `Dict[str, Any]` from typing
- Updated `BaseStorageClient` import path to `chainlit.data.storage_clients`
- Ensured compatibility with modern Chainlit data layer APIs

## Development Commands

### Local Development
```bash
# Setup environment (requires Python 3.10)
conda create -n AudioNotes python=3.10 -y
conda activate AudioNotes
pip install -r requirements.txt

# Configure environment
cp .env.example .env
# Edit .env with database credentials and Ollama configuration

# Run development server (accessible at http://localhost:8000)
chainlit run main.py

# Run with auto-reload
chainlit run main.py -w
```

### Docker Deployment
```bash
# Start all services (accessible at http://localhost:15433)
docker-compose up

# Start in detached mode
docker-compose up -d

# View logs
docker-compose logs -f webui

# Stop services
docker-compose down
```

## Prerequisites

1. **Ollama**: Must be installed and running locally
   - Download from https://ollama.com/download
   - Pull required model: `ollama pull qwen2:7b`
   - Default endpoint: http://localhost:11434/v1

2. **PostgreSQL**: Required for data persistence
   - Docker deployment includes PostgreSQL service
   - Local development requires accessible PostgreSQL instance

## Architecture

### Application Flow

1. **Chat Initialization** (`on_chat_start` in [main.py](main.py)):
   - User uploads audio/video file (up to 10GB)
   - File is transcribed using FunASR
   - Transcript is sent to Ollama for summarization into structured notes
   - Both transcript and summary are streamed to the UI

2. **Audio Chat** (`on_audio_chunk`, `on_audio_end` in [main.py](main.py)):
   - Real-time audio recording from browser
   - Audio chunks buffered in session
   - On completion, audio is transcribed and processed through chat flow

3. **Text Chat** (`on_message`, `chat` in [main.py](main.py)):
   - User asks questions about the transcribed content
   - Chat history is sent to Ollama with system prompt restricting answers to transcribed content
   - Responses are streamed back to user

### Core Components

**[main.py](main.py)**
- Chainlit application entry point
- Defines chat lifecycle handlers (`on_chat_start`, `on_message`, `on_audio_chunk`, etc.)
- Implements password authentication (default: admin/admin)
- Orchestrates ASR and LLM services

**[app/services/asr_funasr.py](app/services/asr_funasr.py)**
- FunASR model wrapper using lazy initialization
- Model loads on first transcription request (not at startup)
- Uses paraformer-zh, fsmn-vad, ct-punc, and cam++ models from ModelScope
- Supports two output formats:
  - `txt`: Plain text transcription (default)
  - `srt`: Subtitle format with timestamps and speaker identification
- Method: `funasr.transcribe(audio_file, output_type="txt")`

**[app/services/ollama.py](app/services/ollama.py)**
- OpenAI-compatible client for Ollama API
- Streaming chat completions with callback support
- Configuration via environment variables (OLLAMA_BASE_URL, OLLAMA_MODEL, OLLAMA_API_KEY)
- Method: `await chat_with_ollama(messages, callback=async_function)`

**[app/services/data_layer.py](app/services/data_layer.py)**
- PostgreSQL database initialization and management
- Creates database if not exists
- Defines Chainlit-specific tables: users, threads, steps, elements, feedbacks
- Custom StorageClient saves file uploads to local `storage/uploads/` directory
- Must call `data_layer.init()` before starting Chainlit app

**[app/utils/utils.py](app/utils/utils.py)**
- File path utilities for storage directories
- `storage_dir()`: Returns path to storage directory (created at project root)
- `upload_dir()`: Returns path to uploads directory, creates if needed
- `root_dir()`: Returns project root directory

### Data Storage

- **Database**: PostgreSQL stores user sessions, chat threads, messages, and feedback
- **File uploads**: Stored in `storage/uploads/` with UUID filenames
- **Logs**: Written to `storage/logs/log.log` with 500 MB rotation
- **Model cache**: FunASR models downloaded to `~/.cache/modelscope` (or `./modelscope` in Docker)

### Environment Configuration

All configuration is managed through environment variables (see [.env.example](.env.example)):

- `USERNAME`, `PASSWORD`: Login credentials (default: admin/admin)
- `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB`, `POSTGRES_HOST`: Database connection
- `OLLAMA_BASE_URL`, `OLLAMA_MODEL`, `OLLAMA_API_KEY`: LLM configuration

## Important Implementation Notes

### FunASR Model Loading
The FunASR model uses lazy initialization - it only loads when `transcribe()` is first called, not during module import. This prevents long startup delays but means the first transcription will be slower.

### Ollama Integration
The application uses Ollama's OpenAI-compatible API. When running in Docker, Ollama must be accessible at `host.docker.internal:11434`. For local development, use `localhost:11434`.

### Chat Context
The chat system maintains full conversation history through Chainlit's `cl.chat_context.to_openai()`. The system prompt constrains the LLM to only answer based on the transcribed and summarized content, preventing hallucination.

### Streaming Responses
Both transcription results and LLM responses are streamed token-by-token to provide real-time feedback to users. Use `msg.stream_token()` for incremental updates and `msg.send()` to finalize.

### Database Initialization
The application automatically creates the PostgreSQL database and required tables on startup via `data_layer.init()` in [main.py](main.py). No manual schema setup is required.

## Common Issues

- **First transcription is slow**: FunASR model loads on first use, which can take 30+ seconds
- **Ollama connection fails in Docker**: Ensure Ollama is running on host and `extra_hosts` is configured in docker-compose.yml
- **Database connection fails**: Verify PostgreSQL credentials match between .env and docker-compose.yml
- **File upload size limits**: Default limit is 10GB, configurable in `max_size_mb` parameter in [main.py:46](main.py#L46)
