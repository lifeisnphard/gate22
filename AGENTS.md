# AGENTS.md

This document provides coding agents with detailed context about the Gate22 repository structure, build processes, testing procedures, and conventions.

## Project Overview

Gate22 is an open-source MCP (Model Context Protocol) Gateway and Control Plane for AI governance.

- **Backend**: Python/FastAPI-based service that powers the control plane, MCP hosting, and Virtual MCP servers
- **Frontend**: Next.js 15 TypeScript application for the management portal

## Repository Structure

```
gate22/
├── backend/          # Python backend services
│   ├── aci/         # Main application package
│   │   ├── alembic/        # Database migrations
│   │   ├── cli/            # Internal Admin CLI commands
│   │   ├── common/         # Shared utilities
│   │   ├── control_plane/  # FastAPI service for the control plane
│   │   ├── mcp/            # FastAPI service for the MCP servers (bundle)
│   │   └── virtual_mcp/    # FastAPI service for the Virtual MCP servers
│   ├── mcp_servers/         # MCP server definitions
│   ├── virtual_mcp_servers/ # Virtual MCP server definitions
│   ├── pyproject.toml       # Python dependencies
│   └── compose.yml          # Docker Compose configuration
├── frontend/        # Next.js frontend application
│   ├── src/
│   │   ├── app/            # Next.js App Router pages
│   │   ├── features/       # Feature-based modules
│   │   ├── components/     # Shared UI components
│   │   └── lib/            # Utilities and shared logic
│   └── package.json        # Node.js dependencies
└── .pre-commit-config.yaml # Pre-commit hooks configuration
```

## Backend

### Code Style & Quality

We follow strict code quality standards:

- **Formatting & Linting**: We use `ruff` for code formatting and linting
- **Type Checking**: We use `mypy` for static type checking
- **Import Linting**: We use `lint-imports` for import order validation
- **Pre-commit Hooks**: Install with `pre-commit install`

### Development Setup

1. **Prerequisites**:
   - Python 3.12+
   - Docker and Docker Compose
   - `uv` package manager

2. **Install dependencies**:
   ```bash
   cd backend
   uv sync
   source .venv/bin/activate
   ```

3. **Install pre-commit hooks**:
   ```bash
   pre-commit install
   ```

4. **Environment variables**:
   ```bash
   cp .env.example .env.local
   ```
   Set the following in `.env.local`:
   - `CONTROL_PLANE_OPENAI_API_KEY`
   - `MCP_OPENAI_API_KEY`
   - `CLI_OPENAI_API_KEY`

5. **Start services**:
   ```bash
   docker compose up --build
   ```

### Testing

Run tests in ephemeral container:
```bash
docker compose exec test-runner pytest
```

### Database Migrations

- Check for changes: `docker compose exec runner alembic check`
- Generate migration: `docker compose exec runner alembic revision --autogenerate -m "description"`
- Apply migration: `docker compose exec runner alembic upgrade head`
- Revert migration: `docker compose exec runner alembic downgrade -1`

## Frontend

### Core Stack

- **Framework**: Next.js 15 with App Router
- **Language**: TypeScript (strict mode)
- **Styling**: Tailwind CSS v4
- **UI Components**: Radix UI primitives + shadcn/ui components
- **State Management**: Zustand + React Query (TanStack Query)
- **Forms**: React Hook Form + Zod validation
- **Testing**: Vitest with MSW for API mocking

### Development Setup

1. **Install dependencies**:
   ```bash
   cd frontend
   pnpm install
   ```

2. **Configure environment**:
   ```bash
   cp .env.example .env
   ```

3. **Start development server**:
   ```bash
   pnpm dev
   ```

### Development Scripts

- `pnpm dev` - Start development server
- `pnpm build` - Build for production
- `pnpm lint` - Run ESLint
- `pnpm format` - Format code with Prettier

### Conventions

- API functions should be placed in feature-specific `api/` folders or `src/lib/api/`
- Use TypeScript strict mode
- Follow existing component patterns and naming conventions
- Prefer composition over inheritance

## Pre-commit Hooks

The repository uses pre-commit hooks for code quality:

### General
- Trailing whitespace removal
- End-of-file fixer
- YAML/JSON validation
- Large file checks
- Merge conflict detection

### Frontend
- Format check: `pnpm run format:check`
- Lint check: `pnpm run lint`
- Build check: `pnpm run build`

### Backend
- Format: `uv run ruff format .`
- Lint: `uv run ruff check . --fix`
- Prettier (JSON): Format JSON files
- Type check: `uv run mypy .`
- Import linter: `uv run lint-imports`
