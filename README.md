# pupsourcing-website

Documentation website for the [pupsourcing/core](https://github.com/pupsourcing/core) project - Clean Event Sourcing library written in Go.

## Overview

This repository hosts the documentation website for pupsourcing, built with [MkDocs](https://www.mkdocs.org/) and the [GitBook theme](https://github.com/GitbookIO/theme-default).

## Local Development

### Prerequisites

- Python 3.8+
- pip

### Setup

1. Install dependencies:
```bash
pip install mkdocs mkdocs-gitbook
```

2. Run the development server:
```bash
mkdocs serve
```

3. Open your browser to [http://localhost:8000](http://localhost:8000)

### Building the Site

Build the static site:
```bash
mkdocs build
```

The built site will be in the `site/` directory.

## Documentation Structure

- `mkdocs.yml` - MkDocs configuration
- `docs/` - Documentation content
  - `index.md` - Landing page with event sourcing introduction
  - `getting-started.md` - Installation and quick start
  - `core-concepts.md` - Event sourcing fundamentals
  - `adapters.md` - Database adapter documentation
  - `consumers.md` - Consumers and projections guide
  - `deployment.md` - Worker-first deployment and operations
  - And more...

## Deployment

The site can be deployed to GitHub Pages or any static hosting service.

### GitHub Pages

```bash
mkdocs gh-deploy
```

This builds the site and pushes it to the `gh-pages` branch.

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test locally with `mkdocs serve`
5. Submit a pull request

## License

MIT License - see the [pupsourcing LICENSE](https://github.com/pupsourcing/core/blob/main/LICENSE)
