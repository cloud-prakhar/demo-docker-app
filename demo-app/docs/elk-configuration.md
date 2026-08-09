# ELK Integration — Configuration Guide

> This document covers everything about the log shipping pipeline: how Filebeat, Logstash, and the Docker Compose network wiring work, how to set up credentials and certificates securely, and how to troubleshoot the full pipeline.

---

## Where to Run These Commands

This project spans two directory levels:

| Location | Path | What lives here |
|---|---|---|
| Repo root | `~/git-repos/demo-docker-app/` | `README.md`, both apps |
| **App dir** ← run commands here | `~/git-repos/demo-docker-app/demo-app/` | `docker-compose.yml`, `.env`, `elk/` (filebeat, logstash, certs) |

**Rule of thumb:**
- Run every `docker compose …` command and every command with a **relative path** (`.env`, `elk/…`) from the **app dir**. Compose only finds `docker-compose.yml` and auto-loads `.env` from the directory you run it in.
- Commands that target Docker globally — `docker ps`, `docker exec`, `docker cp` — work from **any directory** (marked below where relevant).
- The `curl https://localhost:9200/…` commands in this guide pass `--cacert elk/certs/http_ca.crt` and read the password from `.env`, so **both are relative paths — run them from the app dir too.**

Unless a command is explicitly marked "from any directory", assume you have first run:

```bash
cd ~/git-repos/demo-docker-app/demo-app
```

> All `elk/…` and `.env` paths in this guide are written **relative to the app dir**, not the repo root.

---

## How the Pipeline Works

Before touching any config file, understand the complete journey a log takes:

```
┌─────────────────────────────────────────────────────────────┐
│  This repo (demo-docker-app)                                │
│                                                             │
│  ┌──────────┐  JSON stdout   ┌──────────┐  port 5044       │
│  │Flask App │ ─────────────▶ │ Filebeat │ ──────────────┐  │
│  │  :5000   │                │          │               │  │
│  └──────────┘                └──────────┘               │  │
│                                                          │  │
│                              ┌──────────┐  ◀────────────┘  │
│                              │ Logstash │                   │
│                              │  :5044   │                   │
│                              └────┬─────┘                   │
└───────────────────────────────────┼─────────────────────────┘
                                    │ HTTPS :9200
                   ─────────────────┼────────────────────────
                   External ELK     │
                   ┌────────────────▼───────────────────────┐
                   │                                        │
                   │  ┌───────────────┐  ┌───────────────┐ │
                   │  │ Elasticsearch │  │    Kibana     │ │
                   │  │   es01 :9200  │  │  kib01 :5601  │ │
                   │  └───────────────┘  └───────────────┘ │
                   │                                        │
                   └────────────────────────────────────────┘
```

**The key architectural point:** Elasticsearch and Kibana are **not** part of this repo. They run as a separate standalone ELK stack (`es01` and `kib01`). This repo only contains the app and the log shippers (Filebeat + Logstash). Logstash bridges between the two by being on both Docker networks.

---

## Prerequisites

Before starting anything in this repo, the external ELK stack must already be running:

```bash
# Both of these must show "Up"
docker ps | grep -E "es01|kib01"
```

If they are not running, start them first:
```bash
docker start es01
sleep 20
docker start kib01
```

For ELK stack setup instructions, refer to the [elk-setup repository](https://github.com/cloud-prakhar/elk-setup).

---

## Security — What Must Never Be in Git

This project handles credentials and TLS certificates. The following files are **excluded from git** via `.gitignore` and must **never be committed**:

| File | Why it must stay local |
|---|---|
| `demo-app/.env` | Contains the `elastic` user password in plain text |
| `demo-app/elk/certs/http_ca.crt` | TLS Certificate Authority cert from your Elasticsearch instance. Every instance generates its own unique cert — committing it would expose your specific setup and is meaningless to anyone else |

**How to verify these are not tracked** (works from any directory inside the repo):
```bash
git ls-files | grep -E "\.env|certs"
```

This should return **nothing**. If it returns any result, those files are tracked and must be removed immediately (from the app dir):
```bash
git rm --cached .env
git rm --cached elk/certs/http_ca.crt
git commit -m "remove sensitive files from tracking"
```

> **Careful: `.gitignore` protects the `elk/certs/` directory, not the cert filename.** The ignore rules are `demo-app/elk/certs/`, `notes-app/elk/certs/`, and `elk/certs/` — a cert copied anywhere else is **not** ignored. In particular a stray `demo-app/elk/http_ca.crt` (one directory too high) shows up as an untracked file and would be swept in by `git add -A`.
>
> Check for misplaced certs before committing:
> ```bash
> # from the repo root — lists any cert outside an ignored certs/ directory
> git status --porcelain | grep -E "\.crt|\.key|\.pem"
> ```
> Anything listed here is unprotected. Move it into `elk/certs/` or delete it.

---

## One-Time Setup

Every developer or machine running this project must perform these steps once locally. They are **not** stored in git.

### Step 1 — Get the CA Certificate

Logstash needs to trust the HTTPS connection to Elasticsearch. The certificate lives inside the running `es01` container. Copy it into the project:

```bash
# from the app dir: ~/git-repos/demo-docker-app/demo-app

# Create the certs folder
mkdir -p elk/certs

# Copy the cert out of the es01 container (docker cp works from any directory)
docker cp es01:/usr/share/elasticsearch/config/certs/http_ca.crt \
  elk/certs/http_ca.crt
```

**What this cert is:**
It is a Certificate Authority (CA) file — specifically, the CA that signed Elasticsearch's own HTTPS certificate. By giving it to Logstash, we are saying "trust any certificate signed by this CA". Without it, Logstash would refuse the HTTPS connection.

**Why it is not in git:**
Every Elasticsearch installation generates its own unique CA. The cert in your `es01` container is specific to your machine. It would be wrong (and useless) for someone else cloning this repo.

> **Re-copy this cert whenever `es01` is recreated.** The CA is generated when the container is first created. If `es01` is ever removed and recreated (not merely stopped and started), it generates a **brand new CA** and your local copy silently becomes stale. Symptoms: `curl` fails with `SSL certificate problem: self-signed certificate in certificate chain`, and Elasticsearch logs `SSLProtocolException` / `unknown CA`. The fix is simply to run the `docker cp` above again.
>
> Verify your local copy actually matches the running container:
> ```bash
> diff <(docker exec es01 cat /usr/share/elasticsearch/config/certs/http_ca.crt) \
>      elk/certs/http_ca.crt && echo "cert matches" || echo "STALE — re-copy it"
> ```

> **Put the cert in `elk/certs/`, not `elk/`.** `docker-compose.yml` mounts `./elk/certs/http_ca.crt`, and `.gitignore` only excludes `elk/certs/`. A copy left at `elk/http_ca.crt` is unused **and not ignored by git** — a `git add -A` would commit your TLS cert. If you have a stray copy there, delete it:
> ```bash
> ls elk/http_ca.crt 2>/dev/null && rm elk/http_ca.crt
> ```

### Step 2 — Create the `.env` File

Create `.env` (in the app dir, next to `docker-compose.yml`) with your Elasticsearch password:

```bash
# from the app dir: ~/git-repos/demo-docker-app/demo-app
cat > .env << 'EOF'
# Password for the elastic superuser in your running ELK stack
# Logstash uses this to authenticate when sending logs to Elasticsearch
# To get/reset this password:
#   echo "y" | docker exec -i es01 elasticsearch-reset-password -u elastic -s
ELASTIC_PASSWORD=<your-elastic-password>
EOF
```

Replace `<your-elastic-password>` with your actual password.

**How Docker Compose uses this file:**
Docker Compose automatically reads `.env` in the same directory as `docker-compose.yml`. The value of `ELASTIC_PASSWORD` is then injected into the Logstash container via the `environment` section, where Logstash reads it as `${ELASTIC_PASSWORD}` in the pipeline config.

**Important notes on passwords:**
- Use only alphanumeric characters (`A-Z`, `a-z`, `0-9`) in the password — special characters like `*`, `+`, `=` can break Logstash's variable expansion. **`=` is especially bad**: `.env` is parsed as `KEY=VALUE`, so a password containing `=` gets truncated at the first one
- If you need to set a clean password, use the Elasticsearch API directly:

```bash
curl --cacert elk/certs/http_ca.crt \
  -u "elastic:<current-password>" \
  -X POST "https://localhost:9200/_security/user/elastic/_password" \
  -H 'Content-Type: application/json' \
  -d '{"password":"YourNewCleanPassword"}'
```

**If you do not know the current password** (for example `es01` was recreated, so the value in `.env` no longer works and returns `401`), you cannot recover it — Elasticsearch only stores a hash. Reset it, then immediately set a clean alphanumeric value:

```bash
# from the app dir. Generates a random temporary password (-b = batch, no prompt)
TMP=$(docker exec -i es01 elasticsearch-reset-password -u elastic -s -b | tr -d '\r\n')

# Replace it with a clean alphanumeric password
NEW=DemoElastic2026Pass
curl -s --cacert elk/certs/http_ca.crt -u "elastic:$TMP" \
  -X POST "https://localhost:9200/_security/user/elastic/_password" \
  -H 'Content-Type: application/json' \
  -d "{\"password\":\"$NEW\"}"

# Write it to .env and verify
printf 'ELASTIC_PASSWORD=%s\n' "$NEW" > .env
curl -s --cacert elk/certs/http_ca.crt -u "elastic:$NEW" https://localhost:9200/_cluster/health
```

> **Is resetting `elastic` safe?** Yes for Kibana — `kib01` authenticates as the separate `kibana_system` user with its own password, so it is unaffected. Check before you reset if you are unsure:
> ```bash
> docker inspect kib01 --format '{{range .Config.Env}}{{println .}}{{end}}' | grep ELASTICSEARCH_USERNAME
> ```
> Do check whether any *other* project on your machine (for example `notes-app`) has the old `elastic` password in its own `.env` — those will need updating too.

---

## Docker Compose — `docker-compose.yml`

### Services Overview

This compose file starts **three services**. Elasticsearch and Kibana are intentionally absent — they are external.

```
Services in this file:       External (already running):
─────────────────────        ───────────────────────────
app      (Flask app)         es01   (Elasticsearch)
logstash (log processor)     kib01  (Kibana)
filebeat (log collector)
```

### The `app` Service

```yaml
app:
  build: .
  ports:
    - "5000:5000"
  logging:
    driver: "json-file"
    options:
      max-size: "10m"
      max-file: "3"
  labels:
    - "co.elastic.logs/enabled=true"
    - "co.elastic.logs/json.keys_under_root=true"
    - "co.elastic.logs/json.overwrite_keys=true"
  networks:
    - elk
```

| Setting | Purpose |
|---|---|
| `build: .` | Build the image from the `Dockerfile` in the current directory |
| `ports: 5000:5000` | Map container port 5000 to host port 5000 (format is `host:container`) |
| `logging.driver: json-file` | Store container logs as JSON files on the host at `/var/lib/docker/containers/` — Filebeat reads these files |
| `max-size: 10m` | Rotate log files when they reach 10 MB |
| `max-file: 3` | Keep only the last 3 rotated files — prevents disk fill |
| `co.elastic.logs/enabled: true` | **The opt-in label** — Filebeat only collects logs from containers with this label |
| `co.elastic.logs/json.keys_under_root: true` | Tell Filebeat the log content is JSON and to promote fields to root level |

### The `logstash` Service

```yaml
logstash:
  image: docker.elastic.co/logstash/logstash:9.3.2
  volumes:
    - ./elk/logstash/pipeline:/usr/share/logstash/pipeline:ro
    - ./elk/certs/http_ca.crt:/usr/share/logstash/certs/http_ca.crt:ro
  ports:
    - "5044:5044"
  environment:
    - LS_JAVA_OPTS=-Xms256m -Xmx256m
    - xpack.monitoring.enabled=false
    - ELASTIC_PASSWORD=${ELASTIC_PASSWORD}
  networks:
    - elk
    - elastic
```

| Setting | Purpose |
|---|---|
| `image: logstash:9.3.2` | Must match the version of your running Elasticsearch |
| `./elk/logstash/pipeline:/usr/share/logstash/pipeline:ro` | Mount the pipeline config into the container. `:ro` = read-only (the container cannot modify it) |
| `./elk/certs/http_ca.crt:...:ro` | Mount the CA cert into the container so Logstash can trust `es01`'s HTTPS cert |
| `LS_JAVA_OPTS=-Xms256m -Xmx256m` | Set Java heap memory: min 256 MB, max 256 MB. Keeps Logstash from consuming too much RAM |
| `xpack.monitoring.enabled=false` | Disable Logstash's built-in monitoring to avoid extra noise and connections |
| `ELASTIC_PASSWORD=${ELASTIC_PASSWORD}` | Read `ELASTIC_PASSWORD` from the host environment (populated from `.env`) and inject it into the container |
| `networks: [elk, elastic]` | **Logstash is on both networks.** `elk` = receives from Filebeat. `elastic` = sends to `es01` |

### The `filebeat` Service

```yaml
filebeat:
  image: docker.elastic.co/beats/filebeat:9.3.2
  user: root
  command: ["--strict.perms=false"]
  volumes:
    - ./elk/filebeat/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro
    - /var/lib/docker/containers:/var/lib/docker/containers:ro
    - /var/run/docker.sock:/var/run/docker.sock:ro
  depends_on:
    - logstash
  networks:
    - elk
```

| Setting | Purpose |
|---|---|
| `user: root` | Filebeat needs root to read Docker's container log files and the Docker socket |
| `command: ["--strict.perms=false"]` | **Disables Filebeat's config-ownership check.** By default Filebeat refuses to start unless `filebeat.yml` is owned by the user running it (root). The bind-mounted file is owned by your host user, not root, so without this flag Filebeat exits with `config file must be owned by the user identifier (uid=0) or root`. See the troubleshooting entry below |
| `/var/lib/docker/containers:ro` | Mounts the host directory where Docker stores all container log files. Filebeat reads here |
| `/var/run/docker.sock:ro` | Mounts the Docker socket — Filebeat uses this to discover running containers and read their metadata (name, image, labels) |
| `depends_on: logstash` | Docker starts Logstash before Filebeat (so Filebeat does not try to connect before Logstash is ready) |

### Networks

```yaml
networks:
  elk:
    driver: bridge   # Internal — app, filebeat, logstash communicate here

  elastic:
    external: true   # External — already exists, we join it to reach es01
```

**Why two networks?**

```
[app] ──── elk network ────▶ [filebeat] ──── elk network ────▶ [logstash]
                                                                     │
                                                              elastic network
                                                                     │
                                                                  [es01]
```

- `elk` is an internal bridge network created by this compose file. It allows `app`, `filebeat`, and `logstash` to communicate using service names (`logstash:5044`).
- `elastic` is the external network where `es01` and `kib01` live. Logstash joins it so it can reach `es01` by name.

**`external: true` means:** "This network already exists. Do not create it. Just connect to it."

---

## Filebeat Configuration — `elk/filebeat/filebeat.yml`

```yaml
filebeat.autodiscover:
  providers:
    - type: docker
      hints.enabled: true
      # Without this, hints-based autodiscover falls back to a default config that
      # collects EVERY container's logs, bypassing the opt-in template below.
      hints.default_config:
        enabled: false
      # Only collect logs from containers that have the label co.elastic.logs/enabled=true
      templates:
        - condition:
            equals:
              docker.container.labels.co.elastic.logs/enabled: "true"
          config:
            # The `container` input was removed in 9.x; filestream with the
            # container parser is the supported replacement.
            - type: filestream
              id: container-${data.docker.container.id}
              paths:
                - /var/lib/docker/containers/${data.docker.container.id}/*.log
              parsers:
                - container:
                    stream: all
                    format: docker
              # No ndjson parser here on purpose: the app's JSON must reach
              # Logstash intact inside `message`, because logstash.conf parses it
              # and renames the fields (level, endpoint, http_method, client_ip...).
              # Decoding it here would leave `message` as plain text and the
              # Logstash `if [message] =~ /^\s*\{/` filter would never match.

processors:
  - add_docker_metadata:
      host: "unix:///var/run/docker.sock"
  - add_host_metadata: ~

output.logstash:
  hosts: ["logstash:5044"]

logging.level: warning
```

### Section by Section

**`autodiscover`**

Filebeat does not have a fixed list of containers to watch. Instead it uses Docker autodiscovery — it monitors the Docker daemon for container events and automatically starts or stops log collection as containers come and go.

**`hints.enabled: true`**

Tells Filebeat to read `co.elastic.logs/*` labels on each container to decide how to handle its logs.

**`hints.default_config.enabled: false` — do not remove this**

This is the line that actually makes collection opt-in. With `hints.enabled: true` alone, Filebeat applies a **default config to every container it discovers**, and the `templates.condition` below is simply an *additional* rule — not a restriction. The result is that logs from every container on the host (Grafana, Prometheus, `es01`, even Logstash itself) get shipped into `flask-app-*`.

That is not just noise. It caused a real outage during setup: the flood of unrelated logs pushed Logstash past its 256 MB heap and it died with `java.lang.OutOfMemoryError: Java heap space`. See the troubleshooting entries below.

**`templates.condition`**

Only collect logs from containers where the label `co.elastic.logs/enabled` equals `"true"`. Containers without this label are ignored. Combined with the line above, this is the **opt-in mechanism** — you control which containers contribute logs.

**`type: filestream` and the `container` parser**

The `container` input type used by older versions of this config **was removed in Filebeat 9.x**. The supported replacement is a `filestream` input with a `container` parser, which understands Docker's `{"log":"...","stream":"stdout","time":"..."}` wrapper format.

`id:` is required for `filestream` inputs and must be unique per input — hence `container-${data.docker.container.id}`.

**Why there is deliberately no JSON parsing here**

It is tempting to add an `ndjson` parser (or the old `json.keys_under_root: true`) so Filebeat decodes the app's JSON itself. **Do not.** The Logstash pipeline in this project is what parses the JSON and renames the fields, and it keys off the raw JSON still being present:

```
if [message] =~ /^\s*\{/ { ... }
```

If Filebeat decodes the JSON first, `message` arrives at Logstash as plain text, that condition never matches, and **none of the renames fire** — documents land in Elasticsearch with only `service` and no `level`, `endpoint`, `http_method`, or `client_ip`. Parse the JSON in exactly one place: Logstash.

**`processors`**

Processors run on every event before it is sent:
- `add_docker_metadata` — attaches container name, image, labels, and ID to each log event. This is how you know which container a log came from when viewing in Kibana.
- `add_host_metadata` — attaches the hostname of the machine running Filebeat.

**`output.logstash`**

Send all collected log events to Logstash at `logstash:5044`. The service name `logstash` resolves to the Logstash container because both are on the same `elk` Docker network.

**`logging.level: warning`**

Only print Filebeat's own internal logs at WARNING level or above. This prevents Filebeat's operational messages from cluttering `docker logs filebeat`.

---

## Logstash Pipeline — `elk/logstash/pipeline/logstash.conf`

A Logstash pipeline has three sections: **input** (where data comes from), **filter** (how to process it), **output** (where to send it).

### Input

```
input {
  beats {
    port => 5044
  }
}
```

Listen on port 5044 for connections from Filebeat. The `beats` input understands the Beats protocol — a lightweight binary protocol that Filebeat uses to ship logs reliably.

### Filter

```
filter {
  if [message] =~ /^\s*\{/ {
    json {
      source  => "message"
      target  => "log"
    }
    mutate {
      rename => {
        "[log][levelname]"   => "level"
        "[log][message]"     => "log_message"
        "[log][name]"        => "logger"
        "[log][asctime]"     => "timestamp"
        "[log][endpoint]"    => "endpoint"
        "[log][method]"      => "http_method"
        "[log][remote_addr]" => "client_ip"
        "[log][status]"      => "response_status"
      }
    }
  }

  mutate {
    add_field => { "service" => "flask-demo-app" }
  }

  if [type] == "log" and ![message] {
    drop {}
  }
}
```

**`if [message] =~ /^\s*\{/`**
Only process events where the message looks like JSON (starts with `{`). This prevents Logstash from crashing on non-JSON lines (startup messages, warnings, etc.).

**`json { source => "message" target => "log" }`**
Parse the JSON string in `message` and store the resulting fields under a temporary key called `log`. So `{"levelname": "INFO"}` becomes `[log][levelname] = "INFO"`.

**`mutate { rename => {...} }`**
Promote fields from `[log][fieldname]` to top-level fields with cleaner names. This makes them immediately visible in Kibana's field list without having to expand nested objects.

| Original field | Renamed to | Why |
|---|---|---|
| `[log][levelname]` | `level` | Shorter, standard name |
| `[log][message]` | `log_message` | Avoids collision with Logstash's built-in `message` field |
| `[log][asctime]` | `timestamp` | Cleaner name |
| `[log][endpoint]` | `endpoint` | Promote to top level for easy filtering |
| `[log][method]` | `http_method` | Clarifies it is HTTP-specific |
| `[log][remote_addr]` | `client_ip` | More descriptive |

**`add_field { "service" => "flask-demo-app" }`**
Tags every event with the service name. Useful when you have multiple apps sending logs to the same Elasticsearch — you can filter by `service: "flask-demo-app"`.

**`drop {}`**
Silently discard Filebeat internal heartbeat events that have no real log content.

### Output

```
output {
  elasticsearch {
    hosts    => ["https://es01:9200"]
    user     => "elastic"
    password => "${ELASTIC_PASSWORD}"
    ssl_enabled                 => true
    ssl_certificate_authorities => ["/usr/share/logstash/certs/http_ca.crt"]
    ssl_verification_mode       => "none"
    index    => "flask-app-%{+YYYY.MM.dd}"
  }
}
```

| Setting | Purpose |
|---|---|
| `hosts: ["https://es01:9200"]` | Connect to `es01` on the `elastic` Docker network over HTTPS |
| `user / password` | Authenticate with Elasticsearch. Password is read from the `ELASTIC_PASSWORD` environment variable — **never hardcoded** |
| `ssl_enabled: true` | Use HTTPS (encrypted connection). Required because `es01` enforces HTTPS |
| `ssl_certificate_authorities` | Path inside the Logstash container to the CA cert we mounted. This is what allows Logstash to trust `es01`'s certificate |
| `ssl_verification_mode: none` | Skip hostname verification. Required because `es01`'s certificate was issued to the container ID, not to the hostname `es01`. In production this should be `full` with a properly issued cert |
| `index: flask-app-%{+YYYY.MM.dd}` | Store logs in daily indices: `flask-app-2026.03.27`, `flask-app-2026.03.28`, etc. Daily indices make it easy to delete old data and keep searches fast |

---

## Starting the Stack

```bash
# from the app dir: ~/git-repos/demo-docker-app/demo-app

# First time: pull images and build the app
docker compose up -d --build

# Subsequent starts
docker compose up -d
```

### Expected startup order

Docker Compose respects `depends_on`, so services start in this order:
1. `app` and `logstash` start together
2. `filebeat` starts after `logstash` is up

Logstash itself takes 30–60 seconds to fully initialise its pipeline. Filebeat will retry the connection automatically.

### Verify the pipeline is running

```bash
# from the app dir (steps 1–3); step 4 works from any directory

# 1. All three containers are up (look for "Up", not "Exited")
docker compose ps

# 2. Logstash pipeline started successfully
docker compose logs logstash | grep "Pipelines running"

# 3. Filebeat started without a config/permission error
#    NOTE: filebeat.yml sets logging.level: warning, so a healthy Filebeat is
#    SILENT — `docker compose logs filebeat` is normally empty. That is good.
#    Any output here usually means an error. The real proof Filebeat works is
#    step 4 below (documents arriving in Elasticsearch).
docker compose logs filebeat        # expect: no output when healthy

# 4. Logs are reaching Elasticsearch
curl --cacert elk/certs/http_ca.crt \
  -u "elastic:$(grep ELASTIC_PASSWORD .env | cut -d= -f2)" \
  "https://localhost:9200/_cat/indices/flask-app-*?v"
# A row for flask-app-YYYY.MM.dd with docs.count > 0 means the full pipeline works.
# If you see only the header row, generate traffic first: curl http://localhost:5000/
```

### Verify the pipeline is running *correctly*

Documents arriving is necessary but not sufficient — they must be the **right** documents, with the fields the Logstash pipeline is supposed to add. Run both of these checks after any change to `filebeat.yml` or `logstash.conf`.

```bash
PW=$(grep ELASTIC_PASSWORD .env | cut -d= -f2)

# 5. Only the Flask app is being collected — expect ONE bucket: demo-app-app-1
#    Scoped to the last 10 minutes on purpose: documents indexed before a config
#    fix are never rewritten, so an all-time query would still show old noise.
curl -s --cacert elk/certs/http_ca.crt -u "elastic:$PW" \
  -H 'Content-Type: application/json' "https://localhost:9200/flask-app-*/_search" \
  -d '{"size":0,"query":{"range":{"@timestamp":{"gte":"now-10m"}}},
       "aggs":{"c":{"terms":{"field":"container.name.keyword","size":20}}}}'

# 6. The Logstash field renames actually fired
curl -s --cacert elk/certs/http_ca.crt -u "elastic:$PW" \
  -H 'Content-Type: application/json' "https://localhost:9200/flask-app-*/_search" \
  -d '{"size":1,"query":{"exists":{"field":"client_ip"}},"sort":[{"@timestamp":"desc"}],
       "_source":["service","level","logger","log_message","endpoint","http_method","client_ip"]}'
```

A healthy result for check 6 looks like:

```json
{
  "service": "flask-demo-app",
  "level": "INFO",
  "logger": "app",
  "log_message": "Incoming request",
  "http_method": "GET",
  "client_ip": "172.20.0.1"
}
```

If check 5 returns extra containers, or check 6 returns zero hits, go to the matching troubleshooting entry below — the pipeline is running but misconfigured.

> Both checks only reflect documents indexed **after** a config change. Generate fresh traffic and wait ~15 seconds before trusting the result; already-indexed documents are never rewritten.

---

## Stopping and Restarting

```bash
# Stop the app stack only (ELK stack keeps running)
docker compose down

# Restart a single service after a config change
docker compose restart logstash
docker compose restart filebeat

# Start ELK stack first, then the app
docker start es01 && sleep 20 && docker start kib01
docker compose up -d
```

---

## Updating the Elasticsearch Password

If you reset the `elastic` password in Elasticsearch, update it here too:

1. Set the new password in Elasticsearch:
```bash
curl --cacert elk/certs/http_ca.crt \
  -u "elastic:<current-password>" \
  -X POST "https://localhost:9200/_security/user/elastic/_password" \
  -H 'Content-Type: application/json' \
  -d '{"password":"YourNewPassword"}'
```

2. Update `.env` (in the app dir):
```
ELASTIC_PASSWORD=YourNewPassword
```

3. Restart Logstash to pick up the new value:
```bash
docker compose up -d --force-recreate logstash
```

---

## Troubleshooting

### Logstash: `401 Unauthorized` connecting to Elasticsearch

**What it means:** The password Logstash is using does not match what Elasticsearch expects.

**Diagnose:**
```bash
docker compose logs logstash | grep "401"
```

> **If `es01` was recently recreated**, this is expected and the old password is unrecoverable — Elasticsearch stores only a hash. Skip straight to the reset flow in [Updating the Elasticsearch Password](#updating-the-elasticsearch-password). Expect a stale CA cert at the same time, since both are generated when the container is created.

**Fix:**
1. Verify the `.env` file has the correct password
2. Verify the password has no special characters — `=` is the worst offender, since `.env` parses `KEY=VALUE` and truncates the value at the first extra `=`
3. Test the password manually (from any directory):
```bash
curl --cacert elk/certs/http_ca.crt \
  -u "elastic:PASTE_REAL_PASSWORD_HERE" \
  https://localhost:9200
```
If this returns JSON with `cluster_name`, the password is correct. If it returns a 401, reset the password (see section above).

> **Gotcha:** `<your-elastic-password>` in these docs is a **placeholder**. Do not type the angle brackets — `-u "elastic:<m2W…>"` sends the literal `<…>` characters and fails with `security_exception`. To avoid quoting issues entirely, omit the password and let curl prompt for it:
> ```bash
> curl --cacert elk/certs/http_ca.crt -u elastic https://localhost:9200
> ```
> or read it straight from `.env` (run from the app dir):
> ```bash
> curl --cacert elk/certs/http_ca.crt \
>   -u "elastic:$(grep ELASTIC_PASSWORD .env | cut -d= -f2)" \
>   "https://localhost:9200/_cat/indices/flask-app-*?v"
> ```

4. Check Docker Compose is reading `.env` correctly — the container must see the variable:
```bash
docker exec demo-app-logstash-1 env | grep ELASTIC_PASSWORD
```

---

### Logstash: `ELASTIC_PASSWORD is not defined`

**What it means:** Logstash started before the environment variable was available.

**Fix:**
```bash
# from the app dir — verify .env exists and has the variable
grep ELASTIC_PASSWORD .env

# Recreate logstash to force a fresh read of .env
docker compose up -d --force-recreate logstash
```

---

### Logstash: `network elastic not found`

**What it means:** The external `elastic` Docker network does not exist — the ELK stack is not running.

**Fix:**
```bash
# Start the ELK stack first
docker start es01
sleep 20
docker start kib01

# Then start this app stack
docker compose up -d
```

---

### curl / Logstash: `self-signed certificate in certificate chain`

**Full symptom:**
```
* TLSv1.3 (OUT), TLS alert, unknown CA (560):
* SSL certificate problem: self-signed certificate in certificate chain
```
and in `docker logs es01`:
```
SSLProtocolException: Unexpected exception ... caught exception while handling client http traffic
```

**What it means:** Your local `elk/certs/http_ca.crt` does not match the CA currently inside `es01`. This is almost always because `es01` was **removed and recreated**, which regenerates its CA. Stopping and starting `es01` does *not* do this.

**Fix — re-copy the cert:**
```bash
docker cp es01:/usr/share/elasticsearch/config/certs/http_ca.crt elk/certs/http_ca.crt
docker compose restart logstash
```

Confirm it matches:
```bash
diff <(docker exec es01 cat /usr/share/elasticsearch/config/certs/http_ca.crt) \
     elk/certs/http_ca.crt && echo "cert matches"
```

> A recreated `es01` also means a **new `elastic` password**, so expect a `401` right after fixing the cert. See the password reset flow above.

---

### Logstash: `java.lang.OutOfMemoryError: Java heap space`

**Full symptom:**
```
[FATAL][org.logstash.Logstash] uncaught error (in thread ...)
java.lang.OutOfMemoryError: Java heap space
```

**What it means:** Logstash runs with a deliberately small 256 MB heap (`LS_JAVA_OPTS=-Xms256m -Xmx256m`). Under normal load from one Flask app that is plenty. Running out of heap almost always means Logstash is being fed **far more than the app's logs**.

**Diagnose — is Filebeat over-collecting?** This is the usual cause:
```bash
grep -A2 "hints.enabled" elk/filebeat/filebeat.yml
# must show: hints.default_config: / enabled: false
```
Then confirm which containers are actually in the index (see the next entry).

**Fix:** correct `filebeat.yml` as described in the Filebeat section, then recreate:
```bash
docker compose up -d --force-recreate filebeat
```

Verify it has stopped recurring:
```bash
docker inspect demo-app-logstash-1 --format 'RestartCount={{.RestartCount}} Status={{.State.Status}}'
docker compose logs logstash --since 10m | grep -c OutOfMemoryError    # expect 0
```

Only if the app genuinely produces this much log volume should you raise the heap in `docker-compose.yml`:
```yaml
- LS_JAVA_OPTS=-Xms512m -Xmx512m
```

---

### `flask-app-*` contains logs from unrelated containers

**Symptom:** Searching the index in Kibana shows events from `grafana`, `prometheus`, `es01`, or `demo-app-logstash-1` — not just the Flask app.

**What it means:** `hints.default_config.enabled: false` is missing from `filebeat.yml`. See the Filebeat section above.

**Diagnose — break recent traffic down by container:**
```bash
curl -s --cacert elk/certs/http_ca.crt \
  -u "elastic:$(grep ELASTIC_PASSWORD .env | cut -d= -f2)" \
  -H 'Content-Type: application/json' \
  "https://localhost:9200/flask-app-*/_search" \
  -d '{"size":0,"query":{"range":{"@timestamp":{"gte":"now-10m"}}},
       "aggs":{"c":{"terms":{"field":"container.name.keyword","size":20}}}}'
```
A healthy pipeline returns **only** `demo-app-app-1`.

> **Always time-scope this query.** Dropping the `range` filter aggregates over all history, so documents collected *before* you fixed the config keep showing up and make a healthy pipeline look broken. Old documents are never retroactively corrected — the only way to remove them is the cleanup below.
>
> If the query returns `"total": 0`, the app has simply been idle. Generate traffic, wait ~15 seconds, and re-run.

**Fix:** add the missing line, recreate Filebeat, then re-run the aggregation filtered to events since the fix to confirm.

**Cleaning up already-indexed noise** (optional — this deletes data, so check the count first):
```bash
PW=$(grep ELASTIC_PASSWORD .env | cut -d= -f2)

# 1. Count what would be deleted
curl -s --cacert elk/certs/http_ca.crt -u "elastic:$PW" \
  -H 'Content-Type: application/json' "https://localhost:9200/flask-app-*/_count" \
  -d '{"query":{"bool":{"must_not":{"term":{"container.name.keyword":"demo-app-app-1"}}}}}'

# 2. If the number looks right, delete them
curl -s --cacert elk/certs/http_ca.crt -u "elastic:$PW" \
  -H 'Content-Type: application/json' -X POST "https://localhost:9200/flask-app-*/_delete_by_query" \
  -d '{"query":{"bool":{"must_not":{"term":{"container.name.keyword":"demo-app-app-1"}}}}}'
```

---

### Documents arrive but have no `level`, `endpoint`, or `client_ip` fields

**Symptom:** The index fills up and `docs.count` climbs, but documents contain only `service: flask-demo-app` plus Docker metadata. None of the renamed fields from the Logstash pipeline appear, so Kibana has nothing useful to filter on.

**What it means:** Something decoded the app's JSON **before** Logstash saw it. The Logstash filter is gated on `if [message] =~ /^\s*\{/` — it only fires when `message` still holds the raw JSON string. If Filebeat decoded it first, `message` is plain text and every rename is skipped.

**Diagnose — look at what `message` actually contains:**
```bash
curl -s --cacert elk/certs/http_ca.crt \
  -u "elastic:$(grep ELASTIC_PASSWORD .env | cut -d= -f2)" \
  -H 'Content-Type: application/json' "https://localhost:9200/flask-app-*/_search" \
  -d '{"size":1,"sort":[{"@timestamp":"desc"}],"_source":["message","level","endpoint","asctime","levelname"]}'
```
- `message` starts with `{` and `level` is present → **healthy**
- `message` is plain text and you see `levelname`/`asctime` at the root instead of `level` → Filebeat decoded the JSON

**Fix:** remove any `ndjson` parser (or legacy `json.keys_under_root` / `json.message_key` settings) from `elk/filebeat/filebeat.yml`. Keep only the `container` parser, then:
```bash
docker compose up -d --force-recreate filebeat
```
Generate fresh traffic and re-check — existing documents are not retroactively fixed.

---

### Filebeat: `type container is not supported` / input fails to start

**What it means:** The `container` **input type** was removed in Filebeat 9.x. A config written for 7.x/8.x will not start.

**Fix:** use a `filestream` input with a `container` **parser** instead — see the Filebeat configuration section above for the exact block. Note that `filestream` inputs also require a unique `id`.

---

### Filebeat: no logs reaching Logstash

**Step 1 — Check the Flask app container has the required label:**
```bash
docker inspect demo-app-app-1 | grep -A 10 "Labels"
```
Must include `co.elastic.logs/enabled: true`. If not, check the `labels:` section in `docker-compose.yml`.

**Step 2 — Check Filebeat can reach the Docker socket:**
```bash
docker compose logs filebeat | grep -i "error\|docker"
```

**Step 3 — Check Filebeat is actually running (not crash-looping):**
```bash
docker compose ps -a | grep filebeat
```
If it shows `Exited`, read the exit reason — Filebeat prints the error even at `warning` level:
```bash
docker compose logs filebeat | tail -20
```
See the two entries below for the most common crash causes.

---

### Filebeat: `config file must be owned by the user identifier (uid=0) or root`

**What it means:** Filebeat refuses to load `filebeat.yml` because the bind-mounted file is owned by your host user, not by root (the user inside the container). Filebeat exits immediately with status 1.

**Fix:** the compose file disables this check via `command: ["--strict.perms=false"]` on the `filebeat` service. Confirm it is present, then recreate:
```bash
grep strict.perms docker-compose.yml   # should print: command: ["--strict.perms=false"]
docker compose up -d --force-recreate filebeat
```
(Alternative: `sudo chown root:root elk/filebeat/filebeat.yml` — but this is brittle, as any `git checkout` or editor save resets the owner.)

---

### Filebeat: container starts but ships nothing / `flask-app-*` index never appears

**What it means:** A common trap is an **empty or truncated `filebeat.yml`** (0 bytes). Filebeat may start but collect nothing, so Logstash never creates the index.

**Diagnose:**
```bash
wc -c elk/filebeat/filebeat.yml      # 0 bytes = the config is empty/broken
```

**Fix — restore the committed config:**
```bash
# from the app dir
git checkout HEAD -- elk/filebeat/filebeat.yml
wc -c elk/filebeat/filebeat.yml      # ~1.5 KB is healthy; 0 bytes is broken
docker compose up -d --force-recreate filebeat
```

> **Do not treat `git checkout` as a guaranteed fix here.** It restores whatever is committed, which may itself be an older config — for example one still using the `container` input type that Filebeat 9.x removed, or one missing `hints.default_config.enabled: false`. After restoring, compare the file against the reference config in the Filebeat section above before concluding it is correct.

---

### No logs appearing in the Elasticsearch index

```bash
# Check whether the index exists at all
curl --cacert elk/certs/http_ca.crt \
  -u "elastic:<your-elastic-password>" \
  "https://localhost:9200/_cat/indices/flask-app-*?v"
```

- **Index exists, `docs.count` is 0** — Logstash connected but no events have been processed. Generate traffic on the Flask app and wait 10–15 seconds.
- **Index does not exist** — Logstash has not successfully written anything. Check Logstash logs for errors.
- **`index_not_found_exception`** — Normal if no logs have been sent yet. Generate traffic first.

Generate test traffic:
```bash
for i in {1..5}; do
  curl -s http://localhost:5000/health > /dev/null
  curl -s http://localhost:5000/ > /dev/null
done
echo "Traffic generated — wait 15 seconds then check the index"
```

---

### CA certificate errors in Logstash

**Symptom in logs:**
```
SSL certificate problem: unable to get local issuer certificate
```

**What it means:** The cert file at `elk/certs/http_ca.crt` is missing, empty, or does not match `es01`'s current CA.

**Fix:**
```bash
# from the app dir — re-copy the cert from the running es01 container
docker cp es01:/usr/share/elasticsearch/config/certs/http_ca.crt \
  elk/certs/http_ca.crt

# Verify it has content
head -3 elk/certs/http_ca.crt
# Should start with: -----BEGIN CERTIFICATE-----

# Restart Logstash
docker compose restart logstash
```

> If the file is present and well-formed but the error persists, the cert is **stale rather than missing** — `es01` was recreated and generated a new CA. See [`self-signed certificate in certificate chain`](#curl--logstash-self-signed-certificate-in-certificate-chain) above for the `diff` check that confirms this, and note that a recreated `es01` also invalidates the password in `.env`.

---

## Quick Reference — All Ports

| Service | Port | Protocol | Direction |
|---|---|---|---|
| Flask App | 5000 | HTTP | Inbound (your browser / curl) |
| Logstash | 5044 | Beats protocol | Inbound from Filebeat only |
| Elasticsearch (`es01`) | 9200 | HTTPS | Logstash → es01 |
| Kibana (`kib01`) | 5601 | HTTP | Your browser |

---

## Quick Reference — All Files

| File | Committed to git | Contains |
|---|---|---|
| `app.py` | Yes | Flask application code |
| `Dockerfile` | Yes | Image build instructions |
| `requirements.txt` | Yes | Python dependencies |
| `docker-compose.yml` | Yes | Service definitions |
| `.gitignore` | Yes | List of excluded files |
| `elk/filebeat/filebeat.yml` | Yes | Filebeat collection config |
| `elk/logstash/pipeline/logstash.conf` | Yes | Logstash pipeline config |
| `docs/kibana-setup.md` | Yes | Data View setup, queries, dashboards |
| `demo-app/.env` | **No** | Elasticsearch password |
| `demo-app/elk/certs/http_ca.crt` | **No** | TLS certificate (machine-specific) |

> Only `elk/certs/` is git-ignored. A cert saved anywhere else — for example `elk/http_ca.crt` — is **not** protected. See the security section above.

---

## Next Step — View the Logs in Kibana

Once the checks above pass, the logs are in Elasticsearch but Kibana cannot display them until a **Data View** exists for `flask-app-*`. See [kibana-setup.md](./kibana-setup.md) for that and for the field reference, KQL queries, and dashboards.

Quick version:

```bash
curl -s -u "elastic:$(grep ELASTIC_PASSWORD .env | cut -d= -f2)" \
  -X POST "http://localhost:5601/api/data_views/data_view" \
  -H 'kbn-xsrf: true' -H 'Content-Type: application/json' \
  -d '{"data_view":{"title":"flask-app-*","name":"Flask App Logs","timeFieldName":"@timestamp"}}'
```

---

*For Flask application details, see [app-configuration.md](./app-configuration.md)*
*For Kibana Data Views, queries, and dashboards, see [kibana-setup.md](./kibana-setup.md)*
*For ELK Stack installation, see the [elk-setup repository](https://github.com/cloud-prakhar/elk-setup)*
