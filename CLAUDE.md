# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

newsletter2paper is a three-tier application that converts RSS newsletters into formatted PDFs. It consists of:

1. **FastAPI Backend** (`newsletter2paper/`) - Python REST API for managing issues, publications, and PDF generation
2. **Go PDF Service** (`pdf-maker/`) - Standalone Go service for fetching articles and generating PDFs
3. **Next.js Frontend** (`ui/`) - React-based web interface for user interaction

## Development Setup

### Running the Full Stack (Docker)

```bash
docker-compose up --build
```

This starts:
- FastAPI on http://localhost:8000
- Next.js on http://localhost:3000
- Go PDF service (runs as a background container)

### Running Services Individually

**FastAPI Backend:**
```bash
cd newsletter2paper
uvicorn main:app --reload --port 8000
```

**Next.js Frontend:**
```bash
cd ui
npm run dev
```

**Go PDF Service:**
```bash
cd pdf-maker
go run cmd/makepdf/main.go --urls "https://example.com/article" --output test.pdf
```

### Environment Variables

Copy `.env.example` to `.env` and configure:
- `SUPABASE_URL` - Supabase project URL (required)
- `SUPABASE_KEY` - Supabase anon or service key (required)
- `GO_PDF_CONTAINER` - Docker container name for PDF service (default: pdf-maker)

## Testing

**Python Tests:**
```bash
cd newsletter2paper
pytest
```

**Next.js Tests:**
```bash
cd ui
npm test           # Run all tests
npm run test:watch # Watch mode
```

**Linting:**
```bash
cd ui
npm run lint
```

## Architecture

### Service Communication Flow

1. **Frontend → FastAPI:** User configures issues and publications via Next.js UI
2. **FastAPI → Supabase:** Stores issues, publications, and article metadata
3. **FastAPI → Go PDF Service:** Calls Go CLI via `docker exec` to generate PDFs
4. **Go PDF Service → External URLs:** Fetches article content from RSS URLs
5. **Go PDF Service → FastAPI:** Returns generated PDF via shared volume
6. **FastAPI → Supabase Storage:** Uploads PDF to object storage and returns URL

### Key Components

**FastAPI Routers** (`newsletter2paper/routers/`):
- `issues.py` - CRUD for issues and issue-publication associations
- `publications.py` - Publication search and management
- `articles.py` - Article fetching from RSS feeds
- `rss.py` - RSS feed discovery and parsing
- `pdf.py` - PDF generation orchestration

**Go PDF Service** (`pdf-maker/`):
- `cmd/makepdf/main.go` - CLI entry point for PDF generation
- `cmd/fetcharticle/main.go` - Standalone article fetcher
- `internal/fetch/` - Concurrent article fetching with image download
- `internal/pdf/` - PDF generation using WeasyPrint-compatible HTML/CSS
- `internal/clean/` - HTML sanitization and processing

**Next.js Pages** (`ui/app/`):
- `page.js` - Main application interface
- `components/` - React components for UI elements
- `contexts/` - React contexts for state management (publications, config, auth)
- `utils/` - API utilities and helper functions

### Data Model

**Database Tables** (Supabase PostgreSQL):
- `users` - User accounts
- `issues` - Newsletter configurations (title, format, frequency)
- `publications` - RSS feeds and metadata
- `articles` - Fetched article content
- `issue_publications` - Many-to-many with per-publication settings (e.g., `remove_images`)
- `user_issues` - Many-to-many between users and issues

**Key Relationships:**
- Issues contain multiple Publications via `issue_publications`
- Publications have Articles fetched from RSS
- Issues belong to Users via `user_issues`

### PDF Generation Process

The FastAPI service calls the Go PDF CLI via `services/go_pdf_service.py`:

1. Prepares JSON payload with articles and issue metadata
2. Writes JSON to shared volume (`/shared` in Docker)
3. Executes Go binary via `docker exec` on pdf-maker container
4. Go service:
   - Fetches article content if not provided
   - Downloads and processes images
   - Generates HTML from template (newspaper or essay layout)
   - Converts HTML to PDF using WeasyPrint
5. FastAPI reads PDF from shared volume and uploads to Supabase storage

### Layout Types

- **newspaper** - Multi-column layout with masonry-style article arrangement
- **essay** - Single-column reading format optimized for long-form content

Layout type is specified per-issue and passed to the Go PDF generator via the `layout_type` field in the JSON payload.

## Common Patterns

### Adding a New API Endpoint

1. Define Pydantic request/response models in the router file
2. Create router function with appropriate HTTP method decorator
3. Use `DatabaseService` dependency for Supabase operations
4. Include the router in `main.py`
5. Handle errors with HTTPException

### Adding a New Frontend Component

1. Create component in `ui/app/components/`
2. Use Material-UI components for consistent styling
3. Access global state via contexts (`useAuth`, `useSelectedPublications`, `useNewsletterConfig`)
4. API calls use `authApi.js` utilities for authentication

### Database Changes

Supabase handles migrations through their dashboard. For local development:
1. Update SQLModel models in `newsletter2paper/models/`
2. Apply schema changes in Supabase dashboard
3. Update affected routers and services

## Important Notes

- The Go PDF service runs as a sidecar container in Docker, kept alive with `tail -f /dev/null` so FastAPI can call it via `docker exec`
- Shared volumes (`/shared`) facilitate file transfer between FastAPI and Go services
- The UI uses localStorage for guest users and Supabase for authenticated users
- Article content can be provided directly or fetched by URL (Go service handles both)
- Images can be removed globally (per-issue) or per-publication via `remove_images` flag
