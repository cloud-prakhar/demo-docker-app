# Demo App — Teardown and Cleanup Guide

> This guide walks you through stopping and completely removing everything created by this project — containers, images, networks, local credential files, and Elasticsearch data. Steps are organised from least destructive to most destructive so you can stop at whichever level you need.

---

## Where to Run These Commands

Run **everything in this guide from the app dir**, unless a step is marked "from any directory":

```bash
cd ~/git-repos/demo-docker-app/demo-app
```

- `docker compose …` and any relative path (`.env`, `elk/…`) **require** the app dir.
- `docker ps`, `docker images`, `docker rmi`, `docker network ls`, `docker cp`, and `curl https://localhost:9200/…` work **from any directory**.

> Paths like `.env` and `elk/certs/http_ca.crt` below are relative to the app dir. (Do **not** prefix them with `demo-app/` once you have `cd`-ed in — that path won't exist.)

---

## Before You Start — Know What This App Creates

This project creates the following resources on your machine. This is what cleanup targets:

| Resource | Name | Type |
|---|---|---|
| Container | `demo-app-app-1` | Flask app |
| Container | `demo-app-filebeat-1` | Log collector |
| Container | `demo-app-logstash-1` | Log processor |
| Docker image | `demo-app-app:latest` | Built locally from `Dockerfile` |
| Docker image | `filebeat:9.3.2` | Downloaded (~430 MB) |
| Docker image | `logstash:9.3.2` | Downloaded (~877 MB) |
| Docker network | `demo-app_elk` | Internal bridge network |
| Local file | `demo-app/.env` | Contains your Elasticsearch password |
| Local file | `demo-app/elk/certs/http_ca.crt` | TLS certificate copied from `es01` |
| Elasticsearch index | `flask-app-YYYY.MM.DD` | Log data stored in your ELK stack |
| Kibana Data View | `flask-app-*` | Created manually in Kibana UI (if you made one) |

> **What this guide does NOT touch:** The external ELK stack (`es01`, `kib01`, `elastic` network). Those are managed separately — see the [elk-setup teardown guide](https://github.com/cloud-prakhar/elk-setup/blob/main/elk-stack-complete-teardown.md).

---

## Check What Is Currently Running

Always look before you clean up:

```bash
# Containers from this project
docker ps -a | grep demo-app

# Network created by this project
docker network ls | grep demo-app

# Images used by this project
docker images | grep -E "demo-app|logstash|filebeat"

# Elasticsearch index created by this project (from any directory)
curl --cacert ~/ELK/http_ca.crt \
  -u "elastic:<your-elastic-password>" \
  "https://localhost:9200/_cat/indices/flask-app-*?v"

# Local credential and cert files (from the app dir)
ls -la .env elk/certs/http_ca.crt 2>/dev/null
```

---

## Level 1 — Stop the Services (Reversible)

> **Use this when:** You want to free up CPU and memory but plan to run the app again later. All data and configuration is preserved.

```bash
cd ~/git-repos/demo-docker-app/demo-app
docker compose stop
```

**What happens:**
- All three containers (`app`, `logstash`, `filebeat`) are gracefully stopped
- Containers still exist — you can restart them instantly
- No data, images, networks, or files are deleted

**To bring everything back:**
```bash
docker compose start
```

---

## Level 2 — Stop and Remove Containers

> **Use this when:** You want a clean slate for the containers but want to keep the images (so next startup is fast — no re-download needed).

```bash
cd ~/git-repos/demo-docker-app/demo-app
docker compose down
```

**What happens:**
- Stops all three containers
- Removes all three containers
- Removes the `demo-app_elk` internal network
- Docker images are **kept** — next `docker compose up` reuses them

**Verify containers are gone:**
```bash
docker ps -a | grep demo-app
# Should return nothing
```

**To start fresh:**
```bash
docker compose up -d
```

---

## Level 3 — Remove the App Image (Locally Built)

> **Use this when:** You want to force a full rebuild of the Flask app image from scratch — e.g. after a major Dockerfile change, or to reclaim disk space (~58 MB).

First, make sure containers are stopped and removed (Level 2):
```bash
cd ~/git-repos/demo-docker-app/demo-app
docker compose down
```

Then remove the locally built app image:
```bash
docker rmi demo-app-app
```

**What happens:**
- The `demo-app-app:latest` image is deleted from your machine
- Next `docker compose up --build` will rebuild it from the `Dockerfile`
- The Logstash and Filebeat images are **not** affected

**Verify:**
```bash
docker images | grep demo-app
# Should return nothing
```

---

## Level 4 — Remove Downloaded Images (Frees ~1.3 GB)

> **Use this when:** You want to free up significant disk space. Logstash (~877 MB) and Filebeat (~430 MB) will need to be re-downloaded next time.

First ensure containers are stopped and removed (Level 2):
```bash
cd ~/git-repos/demo-docker-app/demo-app
docker compose down
```

Remove the pulled images:
```bash
docker rmi docker.elastic.co/logstash/logstash:9.3.2
docker rmi docker.elastic.co/beats/filebeat:9.3.2
```

**Verify:**
```bash
docker images | grep -E "logstash|filebeat"
# Should return nothing
```

> **Note:** Removing these images does not affect the external ELK stack (`es01`, `kib01`) — those use different images (`elasticsearch` and `kibana`).

---

## Level 5 — Remove Local Credential and Certificate Files

> **Use this when:** You are done with the project and want to remove sensitive files from your machine.

```bash
# from the app dir

# Remove the password file
rm .env

# Remove the TLS certificate
rm -rf elk/certs/
```

**What happens:**
- `.env` is deleted — Logstash will fail to start until you recreate it
- The CA cert is deleted — Logstash will fail to connect to Elasticsearch until you recopy it
- These files are **never in git**, so deleting them only affects your local machine

**To recreate them when needed:**
```bash
# from the app dir

# Re-copy the cert
mkdir -p elk/certs
docker cp es01:/usr/share/elasticsearch/config/certs/http_ca.crt \
  elk/certs/http_ca.crt

# Re-create the .env
echo "ELASTIC_PASSWORD=<your-elastic-password>" > .env
```

---

## Level 6 — Delete Elasticsearch Log Data

> **Use this when:** You want to remove all log data this app shipped into Elasticsearch. This only deletes the `flask-app-*` indices — it does not affect Elasticsearch itself or any other indices.

```bash
# Delete all flask-app indices (all dates)
curl --cacert ~/ELK/http_ca.crt \
  -u "elastic:<your-elastic-password>" \
  -X DELETE "https://localhost:9200/flask-app-*"
```

**Expected response:**
```json
{"acknowledged": true}
```

**Delete a specific date's index only:**
```bash
curl --cacert ~/ELK/http_ca.crt \
  -u "elastic:<your-elastic-password>" \
  -X DELETE "https://localhost:9200/flask-app-2026.03.27"
```

**Verify the indices are gone:**
```bash
curl --cacert ~/ELK/http_ca.crt \
  -u "elastic:<your-elastic-password>" \
  "https://localhost:9200/_cat/indices/flask-app-*?v"
# Should return only the header row with no data rows
```

---

## Level 7 — Remove Kibana Data View and Dashboard

> **Use this when:** You created a Kibana Data View (`flask-app-*`) or dashboard during the practical and want to remove them cleanly.

### Delete the Data View

1. Open `http://localhost:5601`
2. Click the hamburger menu (top-left) → **Management** → **Stack Management**
3. Under **Kibana**, click **Data Views**
4. Find `flask-app-*` (or whatever name you gave it)
5. Click the **trash icon** on the right → **Delete**

### Delete the Dashboard

1. Click hamburger menu → **Dashboard**
2. Find your dashboard (e.g. `Flask App Overview`)
3. Click the three-dot menu (`⋮`) next to it → **Delete**
4. Confirm deletion

---

## Complete Cleanup — All Levels at Once

Run all commands below **in order** to remove everything this project created:

```bash
# Step 1: Stop and remove all containers and the internal network
cd ~/git-repos/demo-docker-app/demo-app
docker compose down

# Step 2: Remove all images (locally built + pulled)
docker rmi demo-app-app
docker rmi docker.elastic.co/logstash/logstash:9.3.2
docker rmi docker.elastic.co/beats/filebeat:9.3.2

# Step 3: Remove local credential and cert files (still in the app dir from Step 1)
rm -f .env
rm -rf elk/certs/

# Step 4: Delete Elasticsearch log data (from any directory)
curl --cacert ~/ELK/http_ca.crt \
  -u "elastic:<your-elastic-password>" \
  -X DELETE "https://localhost:9200/flask-app-*"
```

Then clean up Kibana manually (Level 7 above).

---

## Verify Everything Is Removed

Run this after cleanup to confirm nothing is left behind:

```bash
# run from the app dir (the "Local files" check uses relative paths)
echo "=== Containers ===" && \
  docker ps -a | grep demo-app || echo "None"

echo "=== Images ===" && \
  docker images | grep -E "demo-app|logstash|filebeat" || echo "None"

echo "=== Networks ===" && \
  docker network ls | grep demo-app || echo "None"

echo "=== Local files ===" && \
  ls .env elk/certs/http_ca.crt 2>/dev/null || echo "None"

echo "=== Elasticsearch indices ===" && \
  curl -s --cacert ~/ELK/http_ca.crt \
    -u "elastic:<your-elastic-password>" \
    "https://localhost:9200/_cat/indices/flask-app-*?v" | grep flask || echo "None"
```

All sections should print `None`.

---

## Cleanup Decision Guide

Not sure which level you need? Use this:

```
Do you want to run the app again soon?
├── YES → Level 1 (stop only) or Level 2 (stop + remove containers)
└── NO  → Continue below

Do you want to free up disk space?
├── YES → Level 3 + Level 4 (removes ~1.4 GB of images)
└── NO  → Skip image removal

Are you done with this project entirely?
├── YES → Run Complete Cleanup (all levels)
└── NO  → Remove only what you need
```

---

## What Cleanup Does NOT Do

Be aware of what is intentionally left untouched:

| Left untouched | Reason |
|---|---|
| `es01` and `kib01` containers | These are your external ELK stack — other projects may depend on them |
| `elastic` Docker network | Created by the ELK stack, not by this project |
| Other Elasticsearch indices | Only `flask-app-*` indices are deleted — nothing else is touched |
| The cloned git repository | Your code stays on disk. Run `rm -rf ~/git-repos/demo-docker-app` to remove it entirely |

---

## Troubleshooting Cleanup

### `docker compose down` fails with "network not found"

The network was already removed. Safe to ignore, or run:
```bash
docker rm -f demo-app-app-1 demo-app-logstash-1 demo-app-filebeat-1 2>/dev/null
```

### `docker rmi` fails with "image is being used by a stopped container"

A stopped (not running) container still holds a reference to the image. Remove the containers first:
```bash
docker compose down
docker rmi demo-app-app
```

### Elasticsearch DELETE returns `index_not_found_exception`

The index does not exist — either logs never reached Elasticsearch, or it was already deleted. This is not an error; the cleanup is already complete for that index.

### `.env` or cert file already missing

If `rm` says "No such file or directory", the file was already cleaned up. Nothing to worry about.

---

*For app configuration details, see [app-configuration.md](./app-configuration.md)*
*For ELK integration details, see [elk-configuration.md](./elk-configuration.md)*
*For ELK stack teardown, see the [elk-setup teardown guide](https://github.com/cloud-prakhar/elk-setup/blob/main/elk-stack-complete-teardown.md)*
