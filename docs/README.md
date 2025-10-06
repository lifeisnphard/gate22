# Gate22 Documentation

This directory contains the comprehensive documentation for Gate22, built with [MkDocs](https://www.mkdocs.org/) and the [Material theme](https://squidfunk.github.io/mkdocs-material/).

## Documentation Structure

```
docs/
├── index.md                    # Documentation homepage
├── getting-started.md          # Setup and installation guide
├── architecture/              
│   ├── overview.md             # System architecture overview
│   └── authentication.md       # Authentication & authorization
├── backend/
│   └── overview.md             # Backend development guide
├── frontend/
│   └── overview.md             # Frontend development guide
├── mcp-servers/
│   └── overview.md             # MCP servers guide
├── api-reference.md            # API documentation
├── deployment.md               # Deployment guide
└── contributing.md             # Contributing guidelines
```

## Building the Documentation

### Prerequisites

Install the required dependencies:

```bash
pip install -r requirements-docs.txt
```

### Local Development

Run the documentation server locally:

```bash
mkdocs serve
```

Then open your browser to [http://localhost:8000](http://localhost:8000).

The server will automatically reload when you make changes to the documentation files.

### Building Static Site

Build the static documentation site:

```bash
mkdocs build
```

The built site will be in the `site/` directory.

### Strict Mode

Build with strict mode to catch warnings as errors:

```bash
mkdocs build --strict
```

## Documentation Guidelines

### Writing Style

- Use clear, concise language
- Write in present tense
- Use active voice
- Include code examples where appropriate
- Add diagrams to explain complex concepts

### Markdown Extensions

We use several Markdown extensions for enhanced functionality:

#### Code Blocks

````markdown
```python
def example():
    return "Hello, World!"
```
````

#### Admonitions

```markdown
!!! note
    This is a note admonition.

!!! warning
    This is a warning admonition.

!!! tip
    This is a tip admonition.
```

#### Tabs

```markdown
=== "Python"
    ```python
    print("Hello, World!")
    ```

=== "JavaScript"
    ```javascript
    console.log("Hello, World!");
    ```
```

#### Tables

```markdown
| Column 1 | Column 2 |
|----------|----------|
| Value 1  | Value 2  |
```

### Adding New Pages

1. Create a new Markdown file in the appropriate directory
2. Add the page to the navigation in `mkdocs.yml`:

```yaml
nav:
  - Section Name:
    - path/to/page.md
```

3. Test locally with `mkdocs serve`
4. Submit a pull request

## Deployment

Documentation is automatically deployed to GitHub Pages when changes are pushed to the `main` branch.

The deployment workflow is defined in `.github/workflows/deploy-docs.yml`.

### Manual Deployment

To manually deploy:

```bash
mkdocs gh-deploy
```

This will build the documentation and push it to the `gh-pages` branch.

## Live Documentation

The live documentation is available at:
https://lifeisnphard.github.io/gate22/

## Contributing

Contributions to the documentation are welcome! Please see our [Contributing Guide](contributing.md) for more information.

### What to Document

- New features and capabilities
- API changes
- Configuration options
- Common use cases and examples
- Troubleshooting guides
- Best practices

### Review Process

Documentation changes follow the same review process as code:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test locally
5. Submit a pull request

## Need Help?

- **Discord**: [Join our community](https://discord.com/invite/UU2XAnfHJh)
- **GitHub Issues**: [Report documentation issues](https://github.com/lifeisnphard/gate22/issues)

## License

The documentation is licensed under the same [Apache License 2.0](../LICENSE) as the Gate22 project.
