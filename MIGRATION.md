# Migrating an existing GROWI 7.2.9 (docker-compose) to v8 (this project)

This guide is for upgrading a **production** GROWI 7.2.9 instance — official Docker image,
`growi-docker-compose` style deployment, **standalone MongoDB**, **local-disk file storage** — to the
version used in this project (`growilabs/growi:8`, MongoDB replica set, Elasticsearch 9.x). It assumes you
want to keep your existing MongoDB data and uploaded files, not start fresh.

Read this whole document before starting. Do the upgrade during a maintenance window — it requires stopping
the app, and some steps are not safely reversible without a restore from backup.

## 0. Confirm your current versions first

Don't guess — these determine which steps below actually apply to you.

```bash
# MongoDB version (must already be >= 6.0 on 7.2.9; if not, see "MongoDB too old" below)
docker exec <your-mongo-container> mongosh --quiet --eval "db.version()"

# Elasticsearch version — v8.0 removes support for Elasticsearch v7 entirely
docker exec <your-es-container> curl -s http://localhost:9200 | grep number
```

## 1. The three hard requirements for v8

| Requirement | Why | Typical 7.2.9 situation |
|---|---|---|
| MongoDB must run as a **replica set** (single-node is fine) | v8 uses MongoDB change streams, which only work on a replica set | Standalone → **must convert in place** |
| Elasticsearch must be **v8 or v9** | v7 support removed entirely in v8.0; starting v8 against ES7 logs an error and search stays broken | If on ES7 → **must migrate, fresh index recommended** (don't try to carry over the old index) |
| GROWI AI / Knowledge Assistants | All legacy AI settings and config are replaced by the new agent-based system (`AI_ENABLED`/`AI_PROVIDERS`/`AI_ALLOWED_MODELS`) | **Does not carry over** — must be reconfigured from scratch after the upgrade |

Local-disk file storage (`FILE_UPLOAD=local`) is unaffected by the v8 upgrade — it's just a Docker volume. As
long as you don't rename or delete that volume, your uploaded files survive untouched.

## 2. Back up everything before touching anything

```bash
# Full MongoDB dump
docker exec <your-mongo-container> mongodump --archive=/tmp/backup.gz --gzip
docker cp <your-mongo-container>:/tmp/backup.gz ./growi-backup-$(date +%F).gz

# Your uploaded files volume (replace with your actual volume name)
docker run --rm -v <your_growi_data_volume>:/data -v ${PWD}:/backup alpine \
  tar czf /backup/growi-files-$(date +%F).tar.gz -C /data .

# Your current docker-compose.yml and any .env file
cp docker-compose.yml docker-compose.yml.bak-pre-v8
```

Keep these backups off the server if possible, and don't delete them until v8 has been running cleanly for a
while.

## 3. Maintenance window: stop the app (keep MongoDB running)

```bash
docker compose stop app
```

## 4. Convert MongoDB standalone → single-node replica set, in place

Edit your **current** `docker-compose.yml` `mongo` service to add the replica-set flag — keep the same image,
same volumes, same data:

```yaml
command: ["--replSet", "rs0", "--bind_ip_all"]
```

Recreate just that container (no data is touched, only the startup flag changes):

```bash
docker compose up -d mongo
docker exec -it <your-mongo-container> mongosh --eval \
  "rs.initiate({_id:'rs0', members:[{_id:0, host:'mongo:27017'}]})"
docker exec -it <your-mongo-container> mongosh --eval "rs.status().myState"   # expect 1 (PRIMARY)
```

## 5. Replace Elasticsearch entirely (only if you're on ES7 — skip if already 8.x/9.x)

Don't try to carry over an ES7 index into ES9; create a fresh one and let GROWI rebuild it from MongoDB
(MongoDB is the source of truth for page content, so nothing is lost):

```bash
docker compose rm -sf elasticsearch
docker volume rm <your_es_data_volume_name>   # old ES7 index data — leaving it behind can break v8 startup
```

## 6. Update `docker-compose.yml` to match this project

Use this project's [docker-compose.yml](./docker-compose.yml) as your reference and apply the equivalent
changes to your **existing** file (don't just copy ours wholesale — see the warnings below):

- `image: growilabs/growi:7...` → `growilabs/growi:8`
- `MONGO_URI=mongodb://mongo:27017/growi` → `...?replicaSet=rs0`
- Elasticsearch image → `docker.elastic.co/elasticsearch/elasticsearch:9.3.3`, plus mount the
  [elasticsearch/v9/config](./elasticsearch/v9/config) files (copy them from this project)
- Add the new `pdf-converter` service and its `BULK_EXPORT_PDF_CONVERTER_URI` env var

### ⚠️ Two things you must NOT copy from this project's file

1. **Volume names.** Keep your existing volume names exactly as they are (whatever you currently have for
   the Mongo data dir and the GROWI data/uploads dir). If you copy this project's volume names
   (`growi_data`, `mongo_db`, etc.) and they differ from your production names, Docker will create brand-new
   empty volumes and your existing data/files will appear to have vanished (they haven't — they're just
   orphaned under the old volume name).
2. **`PASSWORD_SEED`.** Keep your existing production value. Do **not** use this project's placeholder
   (`changeme`). Changing `PASSWORD_SEED` invalidates every existing user's password/session hash.

## 7. Bring it up

```bash
docker compose up -d
```

This pulls GROWI 8 and (if replaced) a fresh Elasticsearch 9, and starts them against your now-replica-set
MongoDB and your existing file-storage volume.

## 8. Verify, then rebuild the search index

1. Confirm login works, pages render, and attached images/files load (this proves the volume mapped
   correctly).
2. Admin → **Elasticsearch Management** → **Rebuild page data index** (populates the new/empty index from
   MongoDB).
3. If you previously set `LOCAL_STRATEGY_ENABLED` or `SAML_ENABLED`, check `/login` in a private window and
   confirm ID/Pass and SAML auth are enabled/disabled as expected.
4. Reconfigure AI: Admin → **AI Settings**, or the `AI_ENABLED` / `AI_PROVIDERS` / `AI_PROVIDER_API_KEYS` /
   `AI_ALLOWED_MODELS` env vars (see the "AI feature, powered by local Ollama" section in
   [README.md](./README.md) for the exact scheme and the `store:false` gotcha if backing it with a local
   model instead of real OpenAI).

## 9. Rollback plan

Nothing in steps 3–5 deletes your MongoDB data or uploaded files, so you can revert:

1. Set `image:` back to `growilabs/growi:7.2.9`.
2. Leave MongoDB as a replica set — older GROWI tolerates a replica set fine, it just doesn't require one.
3. Restore the old Elasticsearch container/volume if you kept it, or just let 7.2.9 rebuild against a fresh
   same-major-version ES.
4. If anything is still broken, restore from the `mongodump` archive and the file-storage tarball taken in
   step 2 — that's the last-resort safety net.

## Operational note: no shell in the container (v7.5+)

From GROWI v7.5 onward, the official Docker image is a Docker Hardened Image (DHI) with no `sh`/`bash` and no
package manager. If any of your current operational scripts rely on `docker exec -it <growi-container> sh`
for debugging, that stops working after this upgrade — plan an alternative (e.g. exec into the Mongo/ES
containers instead, which still have shells, or inspect via logs/admin UI).
