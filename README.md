
# ReactRanger

A React mentor that answers from the current react.dev docs and clearly marks legacy patterns.

**Status:** Day 1 foundation (FastAPI skeleton, quality gates). Live URL from Day 5.

## Who it's for

Frontend developer at a Chennai agency migrating a class-component app to modern React.

## The problem

Developers maintaining older React applications often encounter outdated patterns in tutorials, examples and existing code. Finding the correct modern replacement can be confusing, especially when older approaches still appear in online answers. ReactRanger aims to solve this problem by grounding its answers in the current official React documentation. It also identifies legacy patterns and explains how to replace them with modern React practices.

## Data sources

react.dev repository content (Learn + Reference MDX), React blog release posts (18, 19), React changelog.

- https://github.com/reactjs/react.dev
- https://github.com/facebook/react/blob/main/CHANGELOG.md
- Terms: react.dev content CC BY 4.0; React code MIT.

## Hardest problem (my interview story)

Old vs new React: retrieval often pulls legacy pages because they match the question's words. Answers must follow current docs, label legacy patterns, and give the modern replacement with a citation.

## Signature feature

Legacy detector: paste a component and see which lines use legacy patterns, each with the modern replacement and doc link.

## Target metric

Modern-pattern correctness of 90%+ on 40 questions; legacy patterns always labelled.

## Run it locally

```bash
uv sync
cp .env.example .env   # Windows: copy .env.example .env
uv run uvicorn reactranger.api.main:create_app --factory --reload
```

Then open http://localhost:8000/docs

## Checks

```bash
uv run pytest
uv run ruff check .
```

## Run locally in one command

You need Docker Desktop running. From a fresh clone:

```bash
docker compose up --build
```

This starts Postgres (with pgvector), Redis, runs the migrations and loads the sample data once, then starts the API.

- API docs: http://localhost:8000/docs
- Liveness: http://localhost:8000/health
- Readiness (checks the database): http://localhost:8000/ready

Stop with `Ctrl+C`, then `docker compose down`. Add `-v` to also delete the database volume.

## Develop and test

| Task | Mac / Linux | Windows |
|---|---|---|
| Run the API with reload | `make dev` | `uv run poe dev` |
| Unit tests (no network) | `make test` | `uv run poe test` |
| Integration tests (needs `docker compose up`) | `make test-integration` | `uv run poe test-integration` |
| Coverage report | `make cov` | `uv run poe cov` |
| Lint and format check | `make lint` | `uv run poe lint` |
