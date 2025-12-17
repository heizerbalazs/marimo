# marimo Codebase Guide for AI Assistants

This document provides a comprehensive guide to the marimo codebase for AI assistants contributing to the project.

## Overview

**marimo** is a reactive Python notebook that's reproducible, git-friendly, and deployable as scripts or apps. It combines a Python backend (web server + runtime) with a TypeScript/React frontend to provide an interactive notebook experience.

- **Repository**: https://github.com/marimo-team/marimo
- **Version**: 0.18.4
- **Python**: 3.10+
- **Node**: 20+
- **Package Manager**: pnpm 10.24.0+

## Technology Stack

### Backend (Python)
- **Web Framework**: Starlette (ASGI) + Uvicorn server
- **Serialization**: msgspec (high-performance JSON)
- **CLI**: Click
- **Code Analysis**: Jedi (completion), Python AST
- **Markdown**: markdown + pymdown-extensions
- **Package Management**: uv, hatch
- **Testing**: pytest
- **Linting**: Ruff (linter + formatter)
- **Type Checking**: MyPy (strict mode)
- **SQL**: DuckDB, SQLglot, narwhals (DataFrame abstraction)
- **AI**: OpenAI, Anthropic, Google GenAI, MCP (Model Context Protocol)

### Frontend (TypeScript/React)
- **Framework**: React 19.2.0
- **Build Tool**: Vite (via rolldown-vite)
- **Language**: TypeScript
- **Linting/Formatting**: Biome 2.3.8
- **Monorepo**: pnpm workspaces + Turbo
- **Testing**: Vitest (unit), Playwright (E2E)
- **UI Components**: Radix UI + Tailwind CSS 4.1.17
- **Code Editor**: CodeMirror 6
- **State Management**: Jotai 2.15.1
- **WebSocket**: PartySocket
- **Visualization**: Vega/Vega-Lite, Plotly, Altair
- **Data Grid**: Glide Data Grid, TanStack React Table
- **CRDT**: Loro (collaborative editing)

## Directory Structure

```
marimo/
├── marimo/                    # Backend Python package
│   ├── _ast/                 # AST parsing, compilation, cell management
│   ├── _runtime/             # Execution engine, dataflow, reactivity
│   ├── _server/              # Web server, API endpoints, sessions
│   ├── _messaging/           # Frontend-backend protocol
│   ├── _plugins/             # UI elements and output plugins
│   ├── _cli/                 # CLI commands
│   ├── _ai/                  # AI assistant features
│   ├── _config/              # Configuration management
│   ├── _data/                # Data handling utilities
│   ├── _lint/                # Linting engine
│   └── _static/              # Built frontend assets (generated)
│
├── frontend/                  # Frontend React application
│   ├── src/
│   │   ├── core/             # Core app state, WebSocket, kernel
│   │   ├── components/       # React components
│   │   ├── plugins/          # Plugin implementations
│   │   ├── hooks/            # Custom React hooks
│   │   └── utils/            # Utilities
│   ├── e2e/                  # Playwright E2E tests
│   └── package.json
│
├── packages/                  # Shared npm packages (monorepo)
│   ├── llm-info/             # LLM provider definitions
│   ├── smart-cells/          # Smart cell templates
│   ├── lsp/                  # Language Server Protocol
│   └── openapi/              # OpenAPI schema generation
│
├── tests/                     # Python tests (mirrors marimo/)
├── docs/                      # Documentation (MkDocs)
├── examples/                  # Example notebooks
├── development_docs/          # Developer guides
├── Makefile                  # Development commands
├── pyproject.toml            # Python project config
└── package.json              # Root npm config
```

### Key Backend Modules

| Module | Purpose |
|--------|---------|
| `marimo/_ast/` | Parses notebook files, compiles cells to Python, manages cell dependencies |
| `marimo/_runtime/` | Executes cells, tracks dataflow graph, handles reactivity |
| `marimo/_server/` | ASGI app, REST API, WebSocket sessions, file routing |
| `marimo/_messaging/` | Message protocol between frontend/backend (Op classes) |
| `marimo/_plugins/ui/` | Interactive UI elements (slider, button, table, etc.) |
| `marimo/_cli/` | CLI commands (edit, run, convert, export) |
| `marimo/_ai/` | AI assistant, LLM providers, tool registry |

### Key Frontend Modules

| Module | Purpose |
|--------|---------|
| `frontend/src/core/` | App state, WebSocket connection, kernel communication |
| `frontend/src/components/editor/` | Code editor UI, cell management |
| `frontend/src/plugins/` | UI plugin implementations (maps to backend plugins) |
| `frontend/src/core/state/` | Jotai atoms (state management) |
| `frontend/src/core/kernel/` | Message handlers, runtime state tracking |

## Development Setup

### Prerequisites

Install [pixi](https://github.com/prefix-dev/pixi) for environment management, or use `hatch` directly.

### Initial Setup

```bash
# Using pixi (recommended)
pixi shell
make fe && make py
make dev

# Without pixi
hatch shell
make fe && make py
make dev
```

This starts:
- Backend server on port 2718
- Frontend dev server on port 3000

### Optional: Pre-commit Hooks

```bash
uvx pre-commit install
# or
pixi run pre-commit install
```

## Development Workflow

### Common Make Commands

| Command | Purpose |
|---------|---------|
| `make help` | Show all available commands |
| `make install-all` | First-time setup (install all deps) |
| `make fe` | Build frontend assets |
| `make py` | Install Python deps in editable mode |
| `make dev` | Start dev servers (backend + frontend) |
| `make check` | Run all checks (lint, typecheck) |
| `make test` | Run all tests |
| `make fe-check` | Lint + typecheck frontend |
| `make fe-test` | Run frontend tests |
| `make py-check` | Lint + typecheck + format Python |
| `make py-test` | Run Python tests |
| `make e2e` | Run E2E tests |
| `make docs` | Build documentation |
| `make storybook` | Start Storybook |

### Frontend Development

```bash
# In frontend/
pnpm dev                      # Hot reload dev server
pnpm build:watch              # Production build with watch
pnpm test                     # Run unit tests
pnpm playwright test --ui     # Run E2E tests interactively
pnpm storybook                # Component development
pnpm lint                     # Lint TypeScript
pnpm typecheck                # Type check
```

### Python Development

```bash
# Run specific test
hatch run +py=3.13 test:test tests/_ast/

# Run changed tests only
hatch run +py=3.13 test:test --picked

# Run with optional dependencies
hatch run +py=3.13 test-optional:test tests/_ast/

# Lint and format
hatch run lint
hatch run format

# Type check
hatch run typecheck:check
```

### Running marimo in Development Mode

```bash
# Backend with auto-reload
marimo -d edit --no-token

# Frontend with hot reload
cd frontend && pnpm dev

# Then run backend in headless mode
marimo edit --headless --no-token
```

## Architecture

### Frontend-Backend Communication

**WebSocket Protocol:**
1. Frontend connects via WebSocket to backend
2. Backend sends `Op` (Operation) messages containing execution results, outputs, state updates
3. Frontend sends requests for cell execution, UI changes, completions
4. Messages serialized via `msgspec` (JSON)

**Message Flow:**
```
User Action → Frontend → WebSocket → Backend Session Manager
                                          ↓
                                    Runtime Execution
                                          ↓
                                    Serialize Op Message
                                          ↓
                             WebSocket → Frontend → Update UI
```

**Key Files:**
- Backend: `marimo/_messaging/ops.py` (all message types)
- Frontend: `frontend/src/core/websocket/useMarimoWebSocket.tsx`
- Frontend: `frontend/src/core/kernel/handlers.ts` (message handlers)

### Reactivity Model

**Cell Dependency Graph:**
- AST parsing identifies variable dependencies between cells
- Runtime maintains directed acyclic graph (DAG)
- When a cell executes, dependent cells automatically re-run
- UI element changes trigger re-execution of cells using them

**Implementation:**
- Backend: `marimo/_runtime/dataflow.py` (dependency tracking)
- Backend: `marimo/_runtime/runtime.py` (execution orchestration)
- Frontend: `frontend/src/core/cells/cells.ts` (cell state management)

### Plugin Architecture

**Backend (`marimo/_plugins/ui/_core/ui_element.py`):**
```python
class UIElement(ABC, Generic[S, T]):
    """Base class for UI elements
    S: Frontend type (what user sees)
    T: Python type (what code uses)
    """
    def _convert_value(self, value: S) -> T:
        """Transform frontend value to Python value"""
```

**Frontend (`frontend/src/plugins/plugins.ts`):**
- Registry maps component names → React components
- Plugins receive: initial value, onChange handler, args
- On change, sends `SetUIElementValueRequest` to backend

### State Management

**Backend:**
- `RuntimeContext`: Thread-local context per session
- `Session`: Wraps kernel + WebSocket connection
- `CellManager`: Tracks cell graph and execution state

**Frontend (Jotai atoms):**
- `notebookAtom`: Current notebook structure
- `cellsAtom`: Cell states and outputs
- `settingsAtom`: User preferences
- `sessionAtom`: Backend session info

## Code Style & Conventions

### Python

**Style:**
- **Line length**: 79 characters
- **Formatter**: Ruff
- **Linter**: Ruff (strict)
- **Type checking**: MyPy strict mode
- **Imports**: Absolute imports only (no relative)
- **Future imports**: `from __future__ import annotations` (required)

**Key Rules:**
- All public functions must have type annotations
- Use `ruff format` for formatting
- Use `ruff check --fix` for auto-fixable lints
- No `print()` statements (use logging)
- Prefer specific exceptions over `Exception`
- Avoid mutable default arguments

**Before committing:**
```bash
make py-check
```

### TypeScript/React

**Style:**
- **Formatter/Linter**: Biome
- **Conventions**: React 19 patterns, hooks
- **State**: Jotai atoms (not Redux, Context for everything)

**Before committing:**
```bash
make fe-check
```

### Documentation

- Python: Google-style docstrings
- TypeScript: JSDoc comments
- Markdown: Files in `docs/` (MkDocs)

## Testing

### Python Tests

**Location**: `tests/` (mirrors `marimo/` structure)

**Framework**: pytest

**Run tests:**
```bash
make py-test                  # All tests
hatch run test:test tests/_ast/  # Specific module
```

**Writing tests:**
- Mirror the structure in `tests/`
- Use `pytest` fixtures
- Use `inline-snapshot` for snapshot testing
- Mock WebSocket connections for server tests

### Frontend Tests

**Unit Tests**: Vitest
```bash
cd frontend && pnpm test
```

**E2E Tests**: Playwright
```bash
make e2e
# or interactively
cd frontend && pnpm playwright test --ui
```

**Writing E2E tests:**
- Best practices: https://playwright.dev/docs/best-practices
- Build frontend first: `make fe`
- Tests in `frontend/e2e/`

### Test Coverage

- Python: `pytest-codecov`
- Frontend: Vitest coverage

## Common Development Tasks

### Adding a New UI Element

**Backend:**
1. Create class in `marimo/_plugins/ui/_impl/`:
```python
from marimo._plugins.ui._core.ui_element import UIElement

class MyElement(UIElement[str, str]):
    def __init__(self, value: str = ""):
        super().__init__(
            component_name="marimo-my-element",
            initial_value=value,
        )
```

2. Export from `marimo/_plugins/ui/__init__.py`
3. Add tests in `tests/_plugins/ui/_impl/`

**Frontend:**
1. Create component in `frontend/src/plugins/impl/`:
```tsx
export const MyElementPlugin: React.FC<IPluginProps<string>> = ({
  value,
  setValue,
}) => {
  return <input value={value} onChange={(e) => setValue(e.target.value)} />;
};
```

2. Register in `frontend/src/plugins/plugins.ts`:
```tsx
{
  "marimo-my-element": MyElementPlugin,
}
```

3. Add Storybook story in `frontend/src/plugins/impl/__stories__/`

### Adding a New CLI Command

1. Create command in `marimo/_cli/` (or submodule)
2. Register in `marimo/_cli/cli.py`:
```python
@click.command()
def my_command():
    """Description of command"""
    pass

# Add to CLI group
cli.add_command(my_command)
```

### Adding a New Lint Rule

1. Create rule in `marimo/_lint/rules/`
2. Register in `marimo/_lint/linter.py`
3. Add tests in `tests/_lint/`

### Adding AI Tools

1. Create tool in `marimo/_ai/_tools/tools/`
2. Register in `marimo/_ai/_tools/tools_registry.py`
3. Implement MCP protocol if needed

### Modifying the Protocol

**When adding new message types:**
1. Add Op class in `marimo/_messaging/ops.py`
2. Add handler in `frontend/src/core/kernel/handlers.ts`
3. Update TypeScript types in `frontend/src/core/kernel/messages.ts`

**Run codegen after changes:**
```bash
make fe-codegen
```

### Building for Production

```bash
# Build frontend
make fe

# Build Python wheel
make wheel

# Output: dist/marimo-*.whl
```

## Git Workflow

### Branch Naming

- Feature: `feature/description`
- Fix: `fix/description`
- For this task: `claude/add-claude-documentation-i1jB5`

### Commits

**Style:**
- Use clear, descriptive commit messages
- Focus on "why" rather than "what"
- Follow conventional commits (optional but nice)

**Examples:**
```
Add slider UI element to support range selection

Fix race condition in cell execution
Cells were executing out of order when multiple UI elements
changed simultaneously. Added lock to ensure sequential execution.

Update documentation for SQL cells
```

**Creating commits:**
```bash
git status
git diff
git add <files>
git commit -m "Your message"
```

### Pull Requests

**Before submitting:**
1. Run `make check` (lint + typecheck)
2. Run `make test`
3. Sign the CLA (comment: `I have read the CLA Document and I hereby sign the CLA`)

**PR Template:**
- Title: Clear, concise description
- Body: Summary of changes, test plan
- Link related issues

**Labels:**
- `test-all`: Run all tests (not just changed files)

## Important Files Reference

### Configuration

| File | Purpose |
|------|---------|
| `pyproject.toml` | Python project metadata, dependencies, tool configs |
| `package.json` | Root npm scripts, monorepo config |
| `frontend/package.json` | Frontend dependencies |
| `pnpm-workspace.yaml` | pnpm monorepo workspace definition |
| `Makefile` | Development command shortcuts |
| `biome.jsonc` | Frontend linting/formatting rules |
| `tsconfig.json` | TypeScript configuration |
| `frontend/vite.config.mts` | Vite build configuration |
| `mkdocs.yml` | Documentation site configuration |

### Core Architecture

| File | Purpose |
|------|---------|
| `marimo/_ast/app.py` | Notebook app definition |
| `marimo/_ast/compiler.py` | Cell code compilation |
| `marimo/_runtime/runtime.py` | Cell execution orchestrator |
| `marimo/_server/asgi.py` | ASGI app setup |
| `marimo/_server/sessions.py` | Session management |
| `marimo/_messaging/ops.py` | All message types |
| `marimo/_plugins/ui/_core/ui_element.py` | UI element base class |
| `frontend/src/core/MarimoApp.tsx` | Root React component |
| `frontend/src/core/websocket/useMarimoWebSocket.tsx` | WebSocket hook |
| `frontend/src/core/state/jotai.ts` | State atoms |

## Debugging

### Backend

**Enable debug mode:**
```bash
marimo -d edit
```

**Tracing:**
- OpenTelemetry support (see `marimo/_tracer/`)
- Set environment variable for traces

**Logs:**
- Backend logs to stdout/stderr
- WebSocket messages in debug mode

### Frontend

**Dev tools:**
- React DevTools
- Browser console
- Network tab (WebSocket frames)

**State inspection:**
- Jotai DevTools (install separately)
- `console.log` in components

### Common Issues

**Frontend not updating:**
- Check WebSocket connection
- Verify message handler registered
- Check Jotai atom updates

**Cells not executing:**
- Check dependency graph
- Look for circular dependencies
- Verify runtime state

**Build failures:**
- Clear node_modules: `rm -rf node_modules && pnpm install`
- Clear Python cache: `find . -type d -name __pycache__ -exec rm -rf {} +`
- Rebuild: `make fe && make py`

## Resources

- **Documentation**: https://docs.marimo.io
- **Contributing Guide**: `/CONTRIBUTING.md`
- **Discord**: https://marimo.io/discord
- **GitHub Issues**: https://github.com/marimo-team/marimo/issues
- **Development Docs**: `/development_docs/`

## Best Practices for AI Assistants

### When Making Changes

1. **Read before editing**: Always read files before modifying them
2. **Understand context**: Check related files and tests
3. **Follow patterns**: Match existing code style and patterns
4. **Test thoroughly**: Write tests for new functionality
5. **Update docs**: Update documentation when adding features
6. **Small commits**: Make focused, incremental changes

### Code Quality

- **Type safety**: Add type annotations (Python) and proper TypeScript types
- **Error handling**: Handle errors gracefully, use specific exceptions
- **Performance**: Consider performance implications (esp. in runtime/rendering)
- **Security**: Avoid XSS, SQL injection, command injection
- **Accessibility**: Follow Radix UI patterns for accessible components

### Communication

- **Clear intent**: Explain what and why in commit messages
- **Ask questions**: If requirements are unclear, ask before implementing
- **Document decisions**: Comment on non-obvious code choices

### Specific to marimo

- **Reactivity first**: Design with reactive execution in mind
- **No hidden state**: Avoid global state, use RuntimeContext
- **Backwards compatibility**: Consider impact on existing notebooks
- **Plugin system**: Prefer extending via plugins over core changes
- **WebSocket protocol**: Be careful with protocol changes (versioning)

## Codebase Statistics

- **Backend**: ~50k+ lines of Python
- **Frontend**: ~100k+ lines of TypeScript/React
- **Tests**: ~30k+ lines
- **Supported Python versions**: 3.10, 3.11, 3.12, 3.13, 3.14
- **License**: Apache 2.0
- **Community**: NumFOCUS affiliated project

---

**Last Updated**: 2025-12-17
**Version**: 0.18.4

For questions or clarifications, see the [Contributing Guide](CONTRIBUTING.md) or reach out on [Discord](https://marimo.io/discord).
