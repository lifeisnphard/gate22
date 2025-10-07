# GitHub Copilot Instructions for Gate22

This document provides GitHub Copilot with instructions for working effectively in the Gate22 repository.

## Project Overview

Gate22 is an open-source MCP (Model Context Protocol) Gateway and Control Plane for AI governance. The project consists of:

- **Backend**: Python/FastAPI-based service (control plane, MCP hosting, Virtual MCP servers)
- **Frontend**: Next.js 15 TypeScript application (management portal)

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

## Development Guidelines

### Backend (Python)

#### Prerequisites
- Python 3.12+
- Docker and Docker Compose
- `uv` package manager

#### Code Quality Standards
- **Formatting**: Use `ruff` for formatting (`uv run ruff format .`)
- **Linting**: Use `ruff` for linting (`uv run ruff check . --fix`)
- **Type Checking**: Use `mypy` for static type checking (`uv run mypy .`)
- **Import Linting**: Use `lint-imports` (`uv run lint-imports`)
- **Pre-commit Hooks**: Always install with `pre-commit install`

#### Setup Steps
1. Navigate to `backend/` directory
2. Run `uv sync` to install dependencies
3. Activate virtual environment: `source .venv/bin/activate`
4. Install pre-commit hooks: `pre-commit install`
5. Copy `.env.example` to `.env.local` and configure required API keys
6. Start services: `docker compose up --build`

#### Testing
- Run tests: `docker compose exec test-runner pytest`
- Tests run in ephemeral containers with automatic migrations

#### Database Migrations
- Check for changes: `docker compose exec runner alembic check`
- Generate migration: `docker compose exec runner alembic revision --autogenerate -m "description"`
- Apply migration: `docker compose exec runner alembic upgrade head`
- Revert migration: `docker compose exec runner alembic downgrade -1`

### Frontend (Next.js/TypeScript)

#### Core Stack
- **Framework**: Next.js 15 with App Router
- **Language**: TypeScript (strict mode)
- **Styling**: Tailwind CSS v4
- **UI Components**: Radix UI primitives + shadcn/ui components
- **State Management**: Zustand + React Query (TanStack Query)
- **Forms**: React Hook Form + Zod validation
- **Testing**: Vitest with MSW for API mocking

#### Directory Conventions
- **`/src/app/`**: Next.js App Router pages
  - `(auth)/`: Authentication routes
  - `(dashboard)/`: Main app routes
  - `(landing)/`: Public landing page
- **`/src/features/`**: Feature-based modules (each contains api/, components/, hooks/, types/, store/)
- **`/src/components/`**: Shared components (ui/, ui-extensions/, layout/, providers/)
- **`/src/lib/`**: Utilities and shared API logic

#### Development Commands
- Start dev server: `pnpm dev`
- Build for production: `pnpm build`
- Format code: `pnpm format`
- Run linters: `pnpm lint`

#### Coding Conventions
- API functions go in feature-specific `api/` folders or `src/lib/api/`
- Use TypeScript strict mode
- Follow existing component patterns and naming conventions
- Prefer composition over inheritance

## Pre-commit Hooks

The repository uses pre-commit hooks to enforce code quality. Hooks run automatically on commit and include:

### General Checks
- Trailing whitespace removal
- End-of-file fixer
- YAML/JSON validation
- Large file checks
- Merge conflict detection

### Frontend Hooks
- Format check: `pnpm run format:check`
- Lint check: `pnpm run lint`
- Build check: `pnpm run build`

### Backend Hooks
- Format: `uv run ruff format .`
- Lint: `uv run ruff check . --fix`
- Prettier (JSON): Format JSON files
- Type check: `uv run mypy .`
- Import linter: `uv run lint-imports`

## Contributing

1. Read [CODE_OF_CONDUCT.md](../CODE_OF_CONDUCT.md)
2. Sign the [Contributor License Agreement (CLA)](https://bit.ly/4iTXI10)
3. Follow the [CONTRIBUTING.md](../CONTRIBUTING.md) guide
4. For large changes, discuss with the team on [Discord](https://discord.com/invite/UU2XAnfHJh) first

## Common Tasks

### Adding a New Backend Feature
1. Create new modules in appropriate `aci/` subdirectory
2. Update database models if needed
3. Generate migrations with alembic
4. Add tests in corresponding `tests/` directory
5. Update API documentation

### Adding a New Frontend Feature
1. Create feature directory in `src/features/`
2. Add API functions in feature's `api/` folder
3. Create components in feature's `components/` folder
4. Add types in feature's `types/` folder
5. Update routing in `src/app/`

### Updating Dependencies
- Backend: Update `pyproject.toml`, then run `uv sync`
- Frontend: Update `package.json`, then run `pnpm install`

## Resources

- **Documentation**: [aci.dev/docs](https://www.aci.dev/products/mcp-gateway)
- **Community**: [Discord](https://discord.com/invite/UU2XAnfHJh)
- **License**: Apache 2.0
