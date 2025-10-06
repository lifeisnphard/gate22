# Getting Started

This guide will help you set up Gate22 for local development.

## Prerequisites

### Backend Requirements

- **Docker** and **Docker Compose** (v2.0+)
- **Python** 3.12+ (for local development without Docker)
- **PostgreSQL** 15+ with pgvector extension
- **uv** package manager (optional, for faster dependency installation)

### Frontend Requirements

- **Node.js** 18+ (LTS recommended)
- **npm** or **pnpm** or **yarn**

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/lifeisnphard/gate22.git
cd gate22
```

### 2. Set Up the Backend

#### Using Docker (Recommended)

1. Navigate to the backend directory:

```bash
cd backend
```

2. Create environment configuration:

```bash
cp .env.example .env.local
```

3. Update `.env.local` with your configuration:

```env
# Database
POSTGRES_USER=postgres
POSTGRES_PASSWORD=your_secure_password
POSTGRES_DB=gate22

# API Keys (Optional for development)
OPENAI_API_KEY=your_openai_key_here

# Application
ENVIRONMENT=local
LOG_LEVEL=INFO
```

4. Start the services with Docker Compose:

```bash
docker compose up -d
```

5. Run database migrations:

```bash
docker compose exec runner alembic upgrade head
```

6. Insert initial data (MCP servers and tools):

```bash
# Insert remote MCP servers
for server_dir in ./mcp_servers/*/; do
  server_file="${server_dir}server.json"
  tools_file="${server_dir}tools.json"
  docker compose exec runner dotenv run python -m aci.cli mcp upsert-server --server-file "$server_file"
  docker compose exec runner dotenv run python -m aci.cli mcp upsert-tools --tools-file "$tools_file"
done

# Insert virtual MCP servers
for server_dir in ./virtual_mcp_servers/*/; do
  server_file="${server_dir}server.json"
  tools_file="${server_dir}tools.json"
  docker compose exec runner dotenv run python -m aci.cli virtual-mcp upsert-server --server-file "$server_file"
  docker compose exec runner dotenv run python -m aci.cli virtual-mcp upsert-tools --tools-file "$tools_file"
done
```

7. Access the API documentation:

- Control Plane API: [http://localhost:8000/v1/control-plane/docs](http://localhost:8000/v1/control-plane/docs)
- MCP Service API: [http://localhost:8001/v1/mcp/docs](http://localhost:8001/v1/mcp/docs)
- Virtual MCP Service API: [http://localhost:8002/v1/virtual/docs](http://localhost:8002/v1/virtual/docs)

### 3. Set Up the Frontend

1. Navigate to the frontend directory:

```bash
cd frontend
```

2. Install dependencies:

```bash
npm install
# or
pnpm install
# or
yarn install
```

3. Create environment configuration:

```bash
cp .env.example .env.local
```

4. Update `.env.local`:

```env
NEXT_PUBLIC_API_URL=http://localhost:8000
NEXT_PUBLIC_APP_URL=http://localhost:3000
NEXT_PUBLIC_ENVIRONMENT=development
```

5. Start the development server:

```bash
npm run dev
# or
pnpm dev
# or
yarn dev
```

6. Open your browser and navigate to [http://localhost:3000](http://localhost:3000)

## Verification

### Backend Health Checks

Check that all services are running:

```bash
# Control Plane
curl http://localhost:8000/v1/control-plane/health

# MCP Service
curl http://localhost:8001/v1/mcp/health

# Virtual MCP Service
curl http://localhost:8002/v1/virtual/health
```

### Database Connection

You can connect to the database using a GUI client (e.g., Beekeeper Studio, pgAdmin):

- **Host**: localhost
- **Port**: 5432
- **Database**: gate22
- **User**: postgres
- **Password**: (from your .env.local file)

## Development Workflow

### Running Tests

#### Backend Tests

```bash
cd backend
docker compose exec runner pytest
```

#### Frontend Tests

```bash
cd frontend
npm run test
# or
npm run test:coverage
```

### Code Quality

#### Backend Linting

```bash
cd backend
docker compose exec runner ruff check .
docker compose exec runner ruff format .
```

#### Frontend Linting

```bash
cd frontend
npm run lint
npm run format
```

## Next Steps

- [Understand the Architecture](architecture/overview.md)
- [Explore Backend Development](backend/overview.md)
- [Explore Frontend Development](frontend/overview.md)
- [Learn About MCP Servers](mcp-servers/overview.md)

## Troubleshooting

### Port Conflicts

If you encounter port conflicts, you can modify the ports in `docker-compose.yml`:

```yaml
services:
  control_plane:
    ports:
      - "8000:8000"  # Change the first port to an available one
```

### Database Migration Issues

If migrations fail, try resetting the database:

```bash
docker compose down -v
docker compose up -d
docker compose exec runner alembic upgrade head
```

### Frontend Build Issues

Clear the Next.js cache and node_modules:

```bash
rm -rf .next node_modules
npm install
npm run dev
```

## Need Help?

- Join our [Discord community](https://discord.com/invite/UU2XAnfHJh)
- Check the [GitHub Issues](https://github.com/lifeisnphard/gate22/issues)
- Read the [Contributing Guide](https://github.com/lifeisnphard/gate22/blob/main/CONTRIBUTING.md)
