# sdlab-platform

Shared plumbing for the [System Design Lab](https://jpm-dev-14.github.io/sdlab-blog/): versioned base images, Compose stacks, observability, and load and chaos tooling, published to GHCR. Each build repo pins one platform version and pulls these artifacts; nothing is copied or synced between repos.

> **Status:** Phase 0, building v0.1.0.

## What it publishes

| Artifact | What it carries | How a build uses it |
| --- | --- | --- |
| `ghcr.io/jpm-dev-14/sdlab-python` | Python, uv, non-root user, `sdlab-core` and the common clients preinstalled, logging and tracing defaults, health check | `FROM ghcr.io/jpm-dev-14/sdlab-python:${SDLAB_VERSION}` |
| `ghcr.io/jpm-dev-14/sdlab-dev` | `sdlab-python` plus ruff, mypy, the shared configs and a `check` script | Devcontainer image; CI runs `docker run … sdlab-dev check` |
| `ghcr.io/jpm-dev-14/sdlab-grafana`, `sdlab-prometheus` | Standard dashboards; Prometheus with the remote-write receiver (k6) and Docker service discovery on `sdlab.scrape=true` | Part of the observability stack |
| `ghcr.io/jpm-dev-14/sdlab-k6`, `sdlab-loadgen`, `sdlab-chaos` | k6 with the Kafka extension; async event generators and the results exporter; Toxiproxy presets and kill/pause scripts | `make load`, `make chaos` |
| `ghcr.io/jpm-dev-14/sdlab-stack-<component>` | Compose OCI artifacts: postgres, redis, mongo, kafka, flink, observability, toxiproxy | `include: - oci://ghcr.io/jpm-dev-14/sdlab-stack-redis:${SDLAB_VERSION}` |

## Versioning

A git tag `vX.Y.Z` publishes every image and stack as `X.Y.Z` for amd64 and arm64. Versions are immutable: a broken release is fixed by a new version, never by re-pushing a tag. Changes are listed in [CHANGELOG.md](CHANGELOG.md).

## Requirements

Docker with Compose 2.34 or later (remote OCI includes).

## Layout

```
python/sdlab-core/   shared library; depends on the common clients
images/              python, dev, grafana, prometheus, k6, loadgen, chaos
stacks/              postgres, redis, mongo, kafka, flink, observability, toxiproxy
infra/               scripts that rent cloud VMs for benchmark runs
docs/adr/            platform decisions
```

## Scope

Only plumbing lives here. Design logic (rate-limiting algorithms, locking, idempotency, ledgers, matching) stays in the build repos, and code is promoted into the platform only once a second or third build needs it.

## License

[MIT](LICENSE)
