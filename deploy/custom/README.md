# Custom Firecrawl deployment

Production deployment owned by the versioned `custom-v2.11.192` branch of the Firecrawl fork. It replaces the former `firecrawl-deploy` repository and the standalone `searxng-deploy` stack.

## Pinned application versions

| Component | Version | Immutable identity |
|---|---|---|
| Firecrawl | `2.11.192` | source `6ff68d3b029089aed837a881d76616c2b52d711b`; image digest `sha256:4d24b41634f63b470e69c9bc34e75e05fbc3ce52eee39b5b5d38fc628b2fd51b` |
| SearXNG | `2026.8.4+c63835bd2` | revision `c63835bd2a5133b30b3752a20eac6b443a918f41`; image digest `sha256:f4c8e59de166ed71f6380c0847c312ca51f0d41996e31d0559163b6b09ecde52` |

Firecrawl uses its release image rather than a local source build. SearXNG is an image dependency only; no SearXNG source checkout is maintained. Auxiliary images are also digest-pinned in `compose.yaml`.

## Topology

- Firecrawl API: `http://127.0.0.1:3002`
- Shared SearXNG API/UI: `http://127.0.0.1:8080`
- Internal state: Redis, RabbitMQ, NUQ PostgreSQL
- Rendering: Playwright service
- LLM/embedding: LiteLLM over `litellm_default`
- Cross-stack access: `firecrawl` network

Firecrawl, 9Router, and OpenWebUI use the same SearXNG service at `http://searxng:8080/search`. Only loopback ports are published.

The Compose project remains `firecrawl`, preserving these named volumes during migration:

- `firecrawl_postgres`
- `firecrawl_rabbitmq`
- `firecrawl_searxng-cache`

Never run `docker compose down -v` during normal operations.

## Configuration

Copy `.env.example` to `.env`, set all required secrets, and restrict the file to mode `0600`. `.env` is ignored by the repository-wide `.gitignore`.

`searxng/settings.yml` is intentionally minimal and inherits the pinned image defaults. JSON output is enabled for Firecrawl, 9Router, and OpenWebUI. Limiting is disabled because the service is private and only reachable over loopback or Docker networks.

## Operations

Start or reconcile the stack from this directory:

```bash
cd /home/vinh/firecrawl/deploy/custom
docker network inspect firecrawl >/dev/null 2>&1 || docker network create firecrawl
docker compose config --quiet
docker compose up -d --wait
```

Inspect status and logs:

```bash
docker compose ps
docker compose logs --tail=100 api playwright searxng
```

Stop while retaining all data:

```bash
docker compose down
```

## Smoke tests

```bash
curl -fsS http://127.0.0.1:3002/ | jq -e '.message | contains("Firecrawl API")'
curl -fsS 'http://127.0.0.1:8080/search?q=Firecrawl&format=json' | jq -e '.results | type == "array"'
curl -fsS -X POST http://127.0.0.1:3002/v2/search \
  -H 'Content-Type: application/json' \
  -d '{"query":"Firecrawl official documentation","limit":3,"sources":["web"]}' \
  | jq -e '.success == true and (.data.web | length) > 0'
```

Also verify SearXNG from the 9Router and OpenWebUI containers over Docker DNS after any network change.

## Upgrade policy

1. Fast-forward fork `main` to upstream.
2. Identify the latest Firecrawl release tag and matching image build SHA.
3. Pull the latest SearXNG image and record its OCI version, revision, and digest.
4. Create a new `custom-v<firecrawl-version>` branch from the release tag.
5. Update all pins, render Compose, recreate, and run health/search/scrape probes.
6. Keep the previous custom branch and image digests as the rollback target.

Do not merge versioned deployment branches back into upstream-following `main`.
