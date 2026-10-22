# GROWI — fully on-premise docker-compose stack

This stack runs the entire [GROWI](https://growi.org/) wiki on your own infrastructure with **no external/cloud
dependencies**: the app, MongoDB (as a single-node replica set), Elasticsearch (full-text search), and the PDF
export converter all run as local containers, with attachments stored on local disk.

## Services

| Service         | Purpose                                              |
|-----------------|-------------------------------------------------------|
| `app`           | GROWI application (published on host port `3010`)      |
| `mongo`         | MongoDB 8.2, single-node replica set `rs0`             |
| `elasticsearch` | Elasticsearch 9.x, full-text search index              |
| `pdf-converter` | Converts pages to PDF for bulk export                  |

All data is persisted in named Docker volumes (`growi_data`, `mongo_db`, `mongo_configdb`, `es_data`,
`page_bulk_export_tmp`), so your wiki content survives container restarts/upgrades.

> **Upgrading an existing production GROWI (e.g. 7.2.9) to this version?** See
> [MIGRATION.md](./MIGRATION.md) for the step-by-step plan — converting standalone MongoDB to a replica
> set, migrating Elasticsearch off v7, and preserving your existing uploaded files without data loss.

## Before you start

1. **Change `PASSWORD_SEED`** in [docker-compose.yml](./docker-compose.yml) (`app` service) to a random secret
   string. Set it once before the first boot — do not change it afterwards, or existing password hashes will
   become invalid.
2. By default the app listens on all interfaces at host port `3010` (mapped to the container's internal port
   `3000`, since `3000` was already in use by another process on this host). Change the host-side port in
   `app.ports` if you need a different one, and/or prefix it with `127.0.0.1:` to restrict access to localhost
   only (e.g. `127.0.0.1:3010:3000`).
3. (Linux hosts only) Elasticsearch typically requires a higher `vm.max_map_count`:
   ```bash
   sudo sysctl -w vm.max_map_count=262144
   ```
   This isn't needed on Docker Desktop for Windows/Mac.

## Start

```powershell
docker compose up -d
```

Then open http://localhost:3010 — if you're redirected to `/installer`, GROWI started correctly. Follow the
on-screen wizard to create the first administrator account.

## Stop

```powershell
docker compose down
```

(Add `-v` to also delete all data volumes — use with care.)

## AI feature, powered by local Ollama

GROWI's built-in AI assistant only ships native support for `openai` / `anthropic` / `google` / `azure-openai`
as providers — there's no dedicated "Ollama" option. However, GROWI's `openai` provider is built on the Vercel
AI SDK, which automatically falls back to the plain `OPENAI_BASE_URL` environment variable whenever GROWI
doesn't hard-code a base URL (which it doesn't). Since Ollama exposes an OpenAI-compatible API, we redirect
GROWI's "OpenAI" traffic to your local Ollama server instead of the real OpenAI cloud:

```yaml
environment:
  - AI_ENABLED=true
  - AI_PROVIDERS={"openai":{"enabled":true}}
  - AI_PROVIDER_API_KEYS={"openai":"ollama"}            # any non-empty placeholder — Ollama ignores it
  - AI_ALLOWED_MODELS=[{"provider":"openai","modelId":"gemma4:latest","isDefault":true,"providerOptions":{"openai":{"store":false}}}]
  - OPENAI_BASE_URL=http://host.docker.internal:11434/v1
```

Requirements:

- Ollama must be running on your host machine (`ollama serve`, or the Ollama desktop app) with the model already
  pulled, e.g. `ollama pull gemma4:latest`.
- `host.docker.internal` lets the container reach the host; the `extra_hosts: host-gateway` entry makes this work
  on Linux too (Docker Desktop for Windows/Mac already provides it natively).
- Set `modelId` in `AI_ALLOWED_MODELS` to match the exact tag shown by `ollama list` (e.g. `gemma4:latest`,
  `gemma4:26b`).
- **`providerOptions.openai.store: false` is required.** The AI SDK GROWI uses defaults to OpenAI's stateful
  Responses API (`store: true`), which replaces earlier conversation turns with `item_reference` pointers that
  only OpenAI's own servers can resolve. Ollama's `/v1/responses` is stateless and rejects those references with
  `input[N]: unknown input item type: "item_reference"` — this surfaces in the GROWI chat UI as "An error
  occurred" as soon as a second turn/tool-referenced message is sent. Setting `store: false` makes the SDK resend
  full conversation content every turn instead, which Ollama can actually use — verified directly against Ollama's
  `/v1/responses` endpoint, reproducing both the crash (with `store` unset/`true`) and the fix (`store: false`,
  which correctly preserved multi-turn context in testing).
- This was verified end-to-end: the `app` container can reach `http://host.docker.internal:11434/v1/chat/completions`
  and `/v1/responses`, and gets real, context-correct completions back from the local model.

To disable the AI feature again, remove/comment out the `AI_*` and `OPENAI_BASE_URL` lines.

## Notes

- File uploads are stored **locally** (`FILE_UPLOAD=local`), inside the `growi_data` volume — no S3/GCS/Azure
  Blob account needed.
- Uncomment `MATHJAX`, `PLANTUML_URI`, or `FORCE_WIKI_MODE` env vars in `docker-compose.yml` as needed.
- To increase Elasticsearch memory, raise `ES_JAVA_OPTS` (e.g. `-Xms1g -Xmx1g`) if your host has the RAM.
- To upgrade GROWI later: `docker compose pull`, then `docker compose up -d`.
- This configuration is based on the official [growilabs/growi-docker-compose](https://github.com/growilabs/growi-docker-compose) project.
