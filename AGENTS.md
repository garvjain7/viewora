# Working guidance for AI agents

- Be direct. Do not sugarcoat findings, and do not present assumptions as facts.
- Treat `docs/` as a starting point. A statement marked open, proposed, or provisional is not a final decision.
- Before making a consequential architecture or product choice, inspect the existing docs and code, state the uncertainty, and ask the owner when it cannot be resolved from context.
- Prefer the smallest implementation slice that satisfies an agreed requirement. Do not scaffold deferred domains just because they appear in a future plan.
- Keep secrets out of the repository. `.env` is local-only; update `.env.example` with variable names and safe placeholders only.
- Keep `schema.sql` generated from the migration-managed database; never hand-edit the snapshot.
- Do not add Docker unless the owner asks for it or a concrete development need is agreed.
# Initial scaffold

- The backend runtime and core libraries follow the current stack note in the project context. Keep the Python requirement at 3.13.
- Three.js and MediaPipe Tasks Vision belong in `packages/tryon`; `<model-viewer>` belongs in `packages/viewer`. Keep both lazy-loaded by the storefront integration.
- Use the provided stack as the current working direction, not as an irreversible commitment. Status-tagged provisional or deferred choices can change when inputs or measured triggers change.
- Docker Compose is intentionally not included in this initial local scaffold. Add it only when the owner chooses to start the VPS prototype or needs it for local test containers.
