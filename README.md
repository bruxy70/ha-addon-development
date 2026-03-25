# ha-addon-development

A Claude Code plugin for developing Home Assistant apps (formerly add-ons) — full-stack web applications that run inside the HA ecosystem. Covers Python/FastAPI backends, React frontends, HA app packaging, and the complete development workflow.

## Skills

- **ha-app-packaging** — HA app structure, config.yaml schema, Dockerfile patterns, ingress, Supervisor API, data persistence, options schema, translations, publishing to HACS.
- **python-backend** — Python/FastAPI patterns for HA apps. SQLite with SQLAlchemy, JWT authentication, API key management, Pydantic validation, ingress base path handling, migrations with Alembic, graceful shutdown.
- **react-frontend** — React frontend patterns for HA apps. Ingress base path handling, React Query for server state, dark/light theming, responsive design, error boundaries, build configuration with Vite.

## Agents

### Coders
- **backend-coder** — Python/FastAPI developer. API routes, database models, authentication, migrations, service layers.
- **frontend-coder** — React developer. Components, state management, API integration, theming, responsive layouts.

### Cross-Cutting
- **architect** — Full-stack architecture. Database schema, API design, security patterns, deployment strategy.
- **planner** — Research and planning. Codebase analysis, task decomposition, implementation planning.

### Quality
- **test-writer** — Creates pytest and Jest tests for backend and frontend.
- **test-runner** — Runs tests and validates code changes systematically.
- **security-auditor** — Reviews auth, input validation, data protection, and HA-specific security.
- **design-review** — Reviews UI quality, responsiveness, accessibility, and console health.

## Installation

```
/plugin marketplace add bruxy70/ha-addon-development
/plugin install ha-addon-development@bruxy70-ha-addon-development
```

## License

MIT
