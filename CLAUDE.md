# sdlab-platform

Shared plumbing for the System Design Lab, published to GHCR (`ghcr.io/jpm-dev-14/*`) as versioned images and Compose stacks. Build repos pin one version and pull artifacts; nothing syncs by file. Lab-wide method and rules: the workspace `../CLAUDE.md` and `../.claude/rules/platform.md`.

## Scope (the guardrail)

Only plumbing: base images, `sdlab-core` (settings, logging, metrics, tracing, health, client setup), Compose stacks, observability, load and chaos tooling, CI, VM scripts. Never design logic (rate-limit algorithms, locks, idempotency, ledgers, matching, quorums). Code is promoted here only when a second or third build needs it; name those builds in the commit body.

## Contracts the builds rely on

- **Versioning:** git tag `vX.Y.Z` publishes every image and stack as `X.Y.Z`, multi-arch (amd64 + arm64). Immutable: a bad release gets a new version. Release with `/platform-release X.Y.Z`; every change goes in `CHANGELOG.md` under Unreleased first.
- **sdlab-python:** `WORKDIR /app`, non-root, `UV_PROJECT_ENVIRONMENT` pointing at the image's venv, `sdlab-core` preinstalled from `git+https://github.com/jpm-dev-14/sdlab-platform@vX.Y.Z#subdirectory=python/sdlab-core`, the same URL builds declare, so `uv sync` sees it as installed. `sdlab-core` depends on the common clients so they appear in every build's lockfile (`uv sync` removes anything not locked). Keep the image lean.
- **sdlab-dev:** sdlab-python plus ruff and mypy as standalone tools with the shared configs, and a `check` script (ruff format --check, ruff check, mypy --strict, the build's pytest). Used as the devcontainer image and the CI entry point.
- **Prometheus and Grafana:** remote-write receiver on (k6 → Prometheus); Docker service discovery scrapes containers labeled `sdlab.scrape=true`; the standard dashboard puts client latency next to server CPU, pools, Redis, Postgres and Kafka lag.
- **Results:** the `sdlab-loadgen` exporter writes the lab results schema (workspace `.claude/skills/measure/results-schema.md`). Changing that shape is a breaking change.
- **Memory:** every stack sets `mem_limit`s and documents them. A typical build (postgres + redis + observability + 2 services + load generator) must fit in about 4.5 GB on the 8 GB dev laptop.

## Layout

```
python/sdlab-core/
images/{python,dev,grafana,prometheus,k6,loadgen,chaos}/
stacks/{postgres,redis,mongo,kafka,flink,observability,toxiproxy}/
infra/   docs/adr/   .github/workflows/
```

## Phase 0: v0.1.0 (keep it minimal; builds 1 and 2 grow it)

Deliverable: sdlab-python, sdlab-dev, sdlab-grafana, sdlab-prometheus, sdlab-k6; the postgres, redis and observability stacks; the template wired to them; a hello-world build running under load with its dashboard; blog post 0.

1. ADRs in `docs/adr/` for the decisions below.
2. `python/sdlab-core`, `images/python`, `images/dev`, with tests.
3. `stacks/postgres`, `stacks/redis`, `images/prometheus`, `images/grafana`, `stacks/observability`.
4. `images/k6` and the results exporter (smoke profile end to end).
5. Release workflow: buildx multi-arch, a smoke test per image, `docker compose publish --resolve-image-digests`; make the packages public.
6. Template wired to v0.1.0 plus a hello-world build; prove `make up` works on a clean machine with nothing but Docker.

Decisions to record, each ADR with options, choice and why:

- Python version for sdlab-python (3.14 is current; confirm arm64 wheels for every client). PyFlink runs in Flink's own image, so it doesn't constrain this.
- Clients: Postgres (asyncpg vs. psycopg 3), Redis (redis-py asyncio), Mongo (PyMongo async API), Kafka (confluent-kafka vs. aiokafka).
- Service stack: FastAPI + uvicorn, pydantic-settings, logging (structlog vs. stdlib JSON), prometheus-client, OpenTelemetry now or later.
- Tests: pytest with pytest-asyncio or anyio, Hypothesis; the ruff rule set; mypy strict settings.
- Demo tooling: pnpm; ESLint + Prettier vs. Biome. Postgres migration tool.
- Tool pinning: exact ruff and mypy versions in sdlab-dev, mirrored locally with `uv tool install ruff==<v>` and `mypy==<v>` so the editor hook agrees with CI.

## Verify early (risks)

- `include: - oci://…:${SDLAB_VERSION}` resolves with interpolation from `.env`.
- `docker compose publish` may reject the Prometheus Docker-socket bind mount. If it does, pick another discovery mechanism and record it in an ADR.
- New GHCR packages may default to private; make each one public so `make up` needs no login.
- arm64 builds under QEMU can be slow on hosted runners; consider native arm64 runners.
- Local GHCR pushes need `gh auth refresh -h github.com -s write:packages`; CI uses `GITHUB_TOKEN`.

## Commands

Defined in Phase 0; `make help` lists them once the Makefile exists.
