# CLAUDE.md — Reflex Examples Repository

## Overview

This repository contains a collection of example applications built with [Reflex](https://reflex.dev), a Python full-stack web framework. Each subdirectory is a self-contained Reflex app demonstrating different features, patterns, and use cases.

## Repository Structure

```
reflex-examples/
├── basic_crud/          # CRUD app with REST API integration
├── clock/               # Real-time clock display
├── counter/             # Simple counter (has integration tests)
├── customer_data_app/   # Customer management with SQLite
├── dalle/               # DALL-E image generation via OpenAI
├── form-designer/       # Multi-page form builder with auth & DB migrations
├── github-stats/        # GitHub statistics fetcher
├── gpt/                 # GPT chat with user accounts and DB
├── lorem-stream/        # Background tasks & streaming text
├── nba/                 # Data dashboard with Plotly charts
├── quiz/                # Interactive quiz app
├── sales/               # Sales CRM with OpenAI integration
├── snakegame/           # Snake game in Reflex
├── todo/                # Classic todo list
├── translator/          # Language translation
├── twitter/             # Twitter-like social app (multi-page)
├── upload/              # File upload handling
└── chakra_apps/         # Legacy apps using Chakra UI (older Reflex API)
```

> **Note:** `chakra_apps/` contains older examples using the deprecated Chakra UI component library. New examples should use the current Radix-based Reflex components.

## Standard App Structure

Every Reflex example follows this layout:

```
example_name/
├── assets/              # Static files (images, fonts, etc.)
├── example_name/        # Python package: app source code
│   ├── __init__.py
│   └── example_name.py  # Main app file (or split across multiple modules)
├── requirements.txt     # Runtime dependencies (must include `reflex`)
├── rxconfig.py          # Reflex configuration
└── tests/               # Integration tests (optional)
    └── test_*.py
```

For apps with tests, there must also be a `requirements-dev.txt` with test-only dependencies (e.g., `pytest`, `selenium`).

### Multi-file Apps

Larger examples split code across files:

```
complex_app/
└── complex_app/
    ├── __init__.py
    ├── complex_app.py   # Entry point: creates rx.App, calls app.add_page()
    ├── state.py         # State classes
    ├── models.py        # rx.Model database models
    ├── components/      # Reusable UI components
    │   ├── __init__.py
    │   └── navbar.py
    ├── pages/           # Page components
    │   ├── __init__.py
    │   └── home.py
    └── style.py         # Shared styles/constants
```

## Key Reflex Conventions

### App Entry Point (`rxconfig.py`)

Every app requires `rxconfig.py` at its root:

```python
import reflex as rx

config = rx.Config(
    app_name="my_app",  # Must match the Python package directory name
)
```

### State Management

All state lives in classes that inherit from `rx.State`. State vars are class-level attributes; event handlers are methods:

```python
class State(rx.State):
    count: int = 0          # Reactive state var
    items: list[str] = []   # Supports complex types

    def increment(self):    # Event handler (sync)
        self.count += 1

    @rx.background          # Background/async event handler
    async def fetch_data(self):
        async with self:    # Must acquire lock to mutate state
            self.items = await some_async_call()

    @rx.var                 # Computed var (derived from state)
    def total(self) -> int:
        return len(self.items)
```

### UI Components

Components are plain Python functions returning `rx.Component`:

```python
def index() -> rx.Component:
    return rx.center(
        rx.vstack(
            rx.heading("Hello"),
            rx.button("Click", on_click=State.increment),
        )
    )

app = rx.App()
app.add_page(index, title="My App")
```

Use `rx.foreach` to iterate over state vars in the UI (not Python `for` loops):

```python
rx.foreach(State.items, lambda item: rx.text(item))
```

### Database Models

Use `rx.Model` (built on SQLModel) for database-backed models:

```python
class Customer(rx.Model, table=True):
    name: str
    email: str
    phone: str

# In event handlers:
with rx.session() as session:
    customers = session.exec(select(Customer)).all()
    session.add(Customer(name="Alice", email="alice@example.com", phone="555-1234"))
    session.commit()
```

For apps with complex schemas, use Alembic for migrations (see `form-designer/` or `customer_data_app/`).

### Async / Background Tasks

For long-running operations, use `@rx.background`:

```python
@rx.background
async def stream_data(self):
    async with self:            # Acquire state lock before mutating
        self.loading = True
    for chunk in get_chunks():
        await asyncio.sleep(0)
        async with self:
            self.result += chunk
    async with self:
        self.loading = False
```

### Environment Variables

API keys and secrets are read from environment variables — never hardcoded:

```python
import os
client = openai.OpenAI(api_key=os.getenv("OPENAI_API_KEY"))
```

When testing locally, set the needed env var before running `reflex run`. The CI workflow sets `OPENAI_API_KEY="dummy"` for export checks.

## Development Workflow

### Running an App

```bash
cd <example_name>
python -m venv venv
source venv/bin/activate      # Windows: venv\Scripts\activate
pip install -r requirements.txt
reflex init                   # First run only — initializes .web/ directory
reflex run                    # Starts dev server at http://localhost:3000
```

### Adding a New Example

1. Create a new directory with the snake_case app name.
2. Create the standard structure (see above).
3. `requirements.txt` **must** contain `reflex` as a top-level dependency (the CI check enforces this with an exact match on `^reflex`).
4. Run `reflex init && reflex export` to verify the app exports cleanly.
5. If adding integration tests, also create `requirements-dev.txt`.

### Running Tests

Only a few examples have integration tests (currently `counter/`). Tests use `reflex.testing.AppHarness` with Selenium:

```bash
cd counter
python -m venv venv
source venv/bin/activate
pip install -r requirements.txt -r requirements-dev.txt
reflex init
pytest tests -vv
```

## CI/CD

Two GitHub Actions workflows run on PRs to `main`:

| Workflow | File | What it does |
|---|---|---|
| `check-export` | `.github/workflows/check_export.yml` | Runs `reflex init && reflex export` for every example (except `chakra_apps`). Verifies `frontend.zip` and `backend.zip` are valid. |
| `app-harness` | `.github/workflows/app_harness.yml` | Finds all `tests/` directories, installs deps, and runs `pytest tests -vv`. Requires `requirements-dev.txt` to be present. |

**CI requirements for every new example:**
- `requirements.txt` must exist and contain a line starting with `reflex`
- `reflex export` must complete without errors
- If a `tests/` directory exists, `requirements-dev.txt` must also exist

## Important Notes

- **`.web/` directory**: Generated by `reflex init` — never commit it (already in `.gitignore`).
- **Database files (`*.db`)**: Also gitignored.
- **`chakra_apps/`**: Legacy — skipped by the `check-export` CI. Do not add new apps here.
- **`app_name` in `rxconfig.py`** must exactly match the Python package subdirectory name.
- The `TELEMETRY_ENABLED=false` env var is set in CI to disable Reflex telemetry.
