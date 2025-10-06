# Contributing to Gate22

Thank you for your interest in contributing to Gate22! This document provides guidelines and instructions for contributing to the project.

## Code of Conduct

Please read and follow our [Code of Conduct](https://github.com/lifeisnphard/gate22/blob/main/CODE_OF_CONDUCT.md).

## Ways to Contribute

### 1. Report Bugs

Found a bug? Please open an issue on GitHub with:
- Clear description of the bug
- Steps to reproduce
- Expected vs actual behavior
- Environment details (OS, versions, etc.)
- Screenshots if applicable

### 2. Suggest Features

Have an idea? Open a feature request with:
- Clear description of the feature
- Use case and benefits
- Potential implementation approach
- Any relevant examples or mockups

### 3. Contribute Code

We welcome code contributions! See the development workflow below.

### 4. Improve Documentation

Documentation improvements are always appreciated:
- Fix typos or unclear explanations
- Add examples or tutorials
- Improve API documentation
- Translate documentation

### 5. Add MCP Servers

Expand the MCP server library:
- Add new remote MCP server definitions
- Add virtual MCP servers for popular APIs
- Improve existing server configurations

## Development Workflow

### 1. Fork and Clone

```bash
# Fork the repository on GitHub
# Then clone your fork
git clone https://github.com/YOUR_USERNAME/gate22.git
cd gate22
```

### 2. Set Up Development Environment

See [Getting Started](getting-started.md) for detailed setup instructions.

### 3. Create a Branch

```bash
git checkout -b feature/your-feature-name
# or
git checkout -b fix/bug-description
```

Branch naming conventions:
- `feature/` - New features
- `fix/` - Bug fixes
- `docs/` - Documentation changes
- `refactor/` - Code refactoring
- `test/` - Test improvements

### 4. Make Changes

Follow the coding standards for each component:

**Backend (Python):**
- Follow PEP 8 style guide
- Use type hints
- Write docstrings for functions/classes
- Keep functions focused and small

**Frontend (TypeScript):**
- Use TypeScript strictly (avoid `any`)
- Follow existing component patterns
- Write meaningful variable names
- Add JSDoc comments for complex logic

### 5. Write Tests

- Add tests for new features
- Ensure existing tests pass
- Aim for good coverage

```bash
# Backend tests
cd backend
docker compose exec runner pytest

# Frontend tests
cd frontend
npm run test
```

### 6. Run Linters

```bash
# Backend
cd backend
docker compose exec runner ruff check .
docker compose exec runner ruff format .

# Frontend
cd frontend
npm run lint
npm run format
```

### 7. Commit Changes

Write clear, descriptive commit messages:

```bash
git add .
git commit -m "feat: add support for new authentication method"
```

Commit message format:
```
<type>(<scope>): <subject>

<body>

<footer>
```

Types:
- `feat` - New feature
- `fix` - Bug fix
- `docs` - Documentation
- `style` - Code style changes
- `refactor` - Code refactoring
- `test` - Test changes
- `chore` - Build/tooling changes

Examples:
```
feat(mcp): add WebSocket transport support

Add WebSocket transport for real-time MCP communication.
Includes connection handling, message framing, and error recovery.

Closes #123
```

### 8. Push and Create Pull Request

```bash
git push origin feature/your-feature-name
```

Then create a Pull Request on GitHub with:
- Clear title and description
- Reference related issues
- Include screenshots for UI changes
- List any breaking changes

## Pull Request Guidelines

### PR Checklist

- [ ] Code follows project style guidelines
- [ ] Tests added/updated and passing
- [ ] Documentation updated
- [ ] Commit messages are clear
- [ ] No merge conflicts
- [ ] Linked to related issue(s)

### PR Review Process

1. Automated checks run (tests, linting)
2. Code review by maintainers
3. Address feedback
4. Approval and merge

## Coding Standards

### Python (Backend)

```python
# Good
async def create_mcp_server(
    session: AsyncSession,
    data: MCPServerCreate,
) -> MCPServer:
    """
    Create a new MCP server.
    
    Args:
        session: Database session
        data: Server creation data
        
    Returns:
        Created MCP server
        
    Raises:
        ValueError: If server name already exists
    """
    existing = await get_by_name(session, data.name)
    if existing:
        raise ValueError(f"Server {data.name} already exists")
    
    server = MCPServer(**data.model_dump())
    session.add(server)
    await session.commit()
    
    return server
```

### TypeScript (Frontend)

```typescript
// Good
interface MCPServerCardProps {
  server: MCPServerPublic;
  onClick?: (server: MCPServerPublic) => void;
}

export function MCPServerCard({ server, onClick }: MCPServerCardProps) {
  const handleClick = useCallback(() => {
    onClick?.(server);
  }, [server, onClick]);
  
  return (
    <Card onClick={handleClick} className="cursor-pointer hover:shadow-lg">
      <CardHeader>
        <CardTitle>{server.name}</CardTitle>
        <CardDescription>{server.description}</CardDescription>
      </CardHeader>
    </Card>
  );
}
```

## Project Structure

Understand the project structure before contributing:

```
gate22/
├── backend/           # Python/FastAPI backend
│   ├── aci/          # Main application
│   ├── mcp_servers/  # MCP server definitions
│   └── tests/        # Backend tests
├── frontend/         # Next.js frontend
│   ├── src/          # Source code
│   └── tests/        # Frontend tests
└── docs/             # Documentation
```

## Testing Guidelines

### Backend Tests

```python
# Test example
import pytest
from httpx import AsyncClient

@pytest.mark.asyncio
async def test_create_mcp_server(
    client: AsyncClient,
    auth_headers: dict,
):
    response = await client.post(
        "/v1/control-plane/mcp-servers",
        headers=auth_headers,
        json={
            "name": "TEST_SERVER",
            "url": "https://test.example.com",
            "description": "Test server",
        }
    )
    
    assert response.status_code == 201
    data = response.json()
    assert data["name"] == "TEST_SERVER"
```

### Frontend Tests

```typescript
// Test example
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { MCPServerCard } from '../MCPServerCard';

describe('MCPServerCard', () => {
  const mockServer = {
    id: '123',
    name: 'Test Server',
    description: 'Test description',
  };
  
  it('calls onClick when clicked', async () => {
    const handleClick = vi.fn();
    render(<MCPServerCard server={mockServer} onClick={handleClick} />);
    
    await userEvent.click(screen.getByText('Test Server'));
    expect(handleClick).toHaveBeenCalledWith(mockServer);
  });
});
```

## Adding MCP Servers

### Adding a Remote MCP Server

1. Create directory in `backend/mcp_servers/`:
```bash
mkdir backend/mcp_servers/my_server
```

2. Create `server.json`:
```json
{
  "name": "MY_SERVER",
  "url": "https://mcp.example.com",
  "transport_type": "streamable_http",
  "description": "My MCP server",
  "logo": "https://example.com/logo.svg",
  "categories": ["Category"],
  "auth_configs": [...]
}
```

3. (Optional) Create `tools.json` for static tools

4. Add tests

5. Update documentation

### Adding a Virtual MCP Server

1. Create directory in `backend/virtual_mcp_servers/`:
```bash
mkdir backend/virtual_mcp_servers/my_api
```

2. Create `server.json`:
```json
{
  "name": "MY_API",
  "description": "My API service"
}
```

3. Create `tools.json` with REST API definitions

4. Add tests

5. Update documentation

## Documentation

### Writing Documentation

- Use clear, concise language
- Include code examples
- Add diagrams where helpful
- Keep examples up-to-date

### Building Docs Locally

```bash
# Install MkDocs
pip install mkdocs-material

# Serve docs locally
mkdocs serve

# Build docs
mkdocs build
```

## Release Process

Releases are managed by maintainers:

1. Update version numbers
2. Update CHANGELOG
3. Create release tag
4. Build and push Docker images
5. Deploy to production
6. Create GitHub release

## Getting Help

- **Discord**: [Join our community](https://discord.com/invite/UU2XAnfHJh)
- **GitHub Issues**: [Report bugs or ask questions](https://github.com/lifeisnphard/gate22/issues)
- **Documentation**: [Read the docs](https://lifeisnphard.github.io/gate22/)

## Recognition

Contributors are recognized in:
- GitHub contributors page
- Release notes
- Project README

Thank you for contributing to Gate22! 🚀

For the full contributing guide, see [CONTRIBUTING.md](https://github.com/lifeisnphard/gate22/blob/main/CONTRIBUTING.md).
