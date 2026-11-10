# Viewora

Viewora is an early planning and scaffold repository for a QR-first product experience with merchant and shopper flows, product pages, 3D/AR assets, AI-assisted discovery, and commerce. **Nothing in the planning docs should be treated as a fixed final decision** unless the owner explicitly confirms it. Current stack direction is in [docs/stack.md](docs/stack.md); unresolved questions are in [docs/open-questions.md](docs/open-questions.md).

## Repository layout

- `apps/`: planned shopper storefront (SSR candidate) and merchant dashboard (SPA candidate).
- `packages/`: shared UI, tokens, viewer, try-on, and generated API client.
- `backend/`: Python 3.13 backend scaffold and dependency environment. Most modules, API/worker entrypoints, migrations, and tests are not implemented yet.
- `docs/`: working product, requirements, architecture, stack, data-model, API-convention, glossary, and open-question notes.
- `infra/`, `tools/`: deployment configuration and repository tooling as they are added.

## Current provider direction (provisional)

- PostgreSQL 15+; Neon is the current provider direction, subject to checking extension, connection, backup, region, and plan requirements.
- Cloudflare R2 and Cloudflare CDN for product photos, GLB/glTF, USDZ, and derived assets.
- Render Redis-compatible service is the current direction to evaluate. Its free Key Value offering is temporary; do not depend on it for durable carts, job queues, or events without confirming its persistence and recovery behavior.
- Redis uses, queue technology, and whether cache/state roles need separate physical services remain open. See [docs/stack.md](docs/stack.md).

The initial local scaffold intentionally has no Docker Compose. Python is the backend runtime. Node is the planned storefront SSR runtime if that choice remains; it also builds the frontend workspace. The dashboard is planned as a static SPA. The repository currently has no implemented web app or running API.

## Secrets

Copy `.env.example` to `.env` and fill only the values needed by the feature being developed. Never commit `.env`, credentials, or production secrets.
## Python environment

The backend uses a local virtual environment at `backend/.venv/`. `uv sync` manages it from `backend/pyproject.toml`, with dependency versions recorded in `backend/uv.lock`.

From the repository root:

```powershell
uv sync --project backend
.\backend\.venv\Scripts\Activate.ps1
```

To leave the environment, run `deactivate`. The `.venv` directory is local-only and is ignored by Git. You can also run project commands without activating it with `uv run --project backend <command>`.


