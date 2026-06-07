# Flask App — Configuration Guide

> This document covers everything about the Flask application itself: what it does, how it is structured, how logging works, how the Docker image is built, and how to run and extend it.

---

## Where to Run These Commands

This project spans two directory levels:

| Location | Path | What lives here |
|---|---|---|
| Repo root | `~/git-repos/demo-docker-app/` | `README.md`, both apps |
| **App dir** ← run commands here | `~/git-repos/demo-docker-app/demo-app/` | `docker-compose.yml`, `.env`, `Dockerfile`, `elk/` |

**Rule of thumb:**
- Run every `docker compose …` command and every command with a **relative path** (`.env`, `elk/…`, `.`) from the **app dir**. Compose only finds `docker-compose.yml` and auto-loads `.env` from the directory you run it in.
- Commands that target Docker globally or Elasticsearch directly — `docker ps`, `docker images`, `docker rmi`, `docker exec`, `docker cp`, and any `curl https://localhost:9200/…` — work from **any directory**.

Unless a command is explicitly marked "from any directory", assume you have first run:

```bash
cd ~/git-repos/demo-docker-app/demo-app
```

---

## What This App Does

This is a lightweight demo web application built with **Flask** (a Python web framework). It has two HTTP endpoints and is specifically designed to emit **structured JSON logs** to stdout — which makes it easy for log shippers like Filebeat to collect and parse them.

It is intentionally simple. The purpose is not the app itself, but demonstrating how a real containerised app integrates with a logging pipeline.

---

## Project Structure

```
demo-app/
├── app.py                  ← The entire Flask application
├── Dockerfile              ← Instructions to build the Docker image
├── requirements.txt        ← Python package dependencies
├── docker-compose.yml      ← Starts the app + log pipeline services
├── .env                    ← Local secrets (NOT committed to git)
└── elk/
    ├── certs/              ← SSL certificate (NOT committed to git)
    ├── filebeat/
    │   └── filebeat.yml
    └── logstash/
        └── pipeline/
            └── logstash.conf
```

> Files marked "NOT committed to git" are listed in `.gitignore` for security. See [elk-configuration.md](./elk-configuration.md) for details.

---

## The Application — `app.py`

```python
import logging
import sys
from flask import Flask, jsonify, request
from pythonjsonlogger import jsonlogger

app = Flask(__name__)
```

**What this does:**
- `logging` — Python's built-in logging module
- `sys` — gives us access to `stdout` (standard output, i.e. the terminal/console)
- `Flask` — the web framework
- `jsonify` — converts Python dicts into JSON HTTP responses
- `request` — lets us read incoming HTTP request details (method, path, IP)
- `jsonlogger` — third-party library that formats log lines as JSON instead of plain text

---

### JSON Logging Setup

```python
logger = logging.getLogger()
handler = logging.StreamHandler(sys.stdout)
formatter = jsonlogger.JsonFormatter(
    fmt="%(asctime)s %(name)s %(levelname)s %(message)s",
    datefmt="%Y-%m-%dT%H:%M:%S"
)
handler.setFormatter(formatter)
logger.addHandler(handler)
logger.setLevel(logging.INFO)
```

**Line by line:**

| Line | What it does |
|---|---|
| `logging.getLogger()` | Gets the root logger — the top-level logger that all other loggers inherit from |
| `StreamHandler(sys.stdout)` | Creates a handler that writes log lines to stdout (the terminal). Docker captures stdout automatically |
| `JsonFormatter(fmt=...)` | Formats each log line as a JSON object. `%(asctime)s` = timestamp, `%(levelname)s` = INFO/ERROR etc |
| `logger.setLevel(logging.INFO)` | Only log messages at INFO level or above (INFO, WARNING, ERROR, CRITICAL). DEBUG is ignored |

**Why JSON?**
Plain text logs like `[2026-03-27 10:00:01] INFO - Health check` are hard to search. JSON logs like `{"level": "INFO", "message": "Health check", "endpoint": "/health"}` can be queried field by field in Kibana.

---

### Request Logging Hook

```python
@app.before_request
def log_request():
    app.logger.info(
        "Incoming request",
        extra={"method": request.method, "path": request.path, "remote_addr": request.remote_addr}
    )
```

**What this does:**
- `@app.before_request` — Flask runs this function automatically before EVERY request, regardless of which route handles it
- Logs the HTTP method (GET, POST), the path (/health, /), and the caller's IP address
- `extra={...}` — adds these as top-level fields in the JSON log output alongside `message`

This means every single HTTP request is logged automatically. You do not need to add logging to every route manually.

---

### Application Routes

```python
@app.route("/")
def home():
    app.logger.info("Home endpoint accessed", extra={"endpoint": "/"})
    return "<h1>Hello from Docker!</h1><p>Your lightweight Flask app is running.</p>"
```

```python
@app.route("/health")
def health():
    app.logger.info("Health check", extra={"endpoint": "/health", "status": "ok"})
    return jsonify({"status": "ok"})
```

| Route | Method | Returns | Purpose |
|---|---|---|---|
| `/` | GET | HTML page | Home page — confirms the app is running |
| `/health` | GET | `{"status": "ok"}` | Health check — used by monitoring tools and load balancers to verify the app is alive |

---

### What a Log Line Looks Like

When a request hits `/health`, the app emits this JSON on stdout:

```json
{
  "asctime": "2026-03-27T10:00:00",
  "name": "flask.app",
  "levelname": "INFO",
  "message": "Incoming request",
  "method": "GET",
  "path": "/health",
  "remote_addr": "172.18.0.1"
}
```

Followed immediately by:

```json
{
  "asctime": "2026-03-27T10:00:00",
  "name": "flask.app",
  "levelname": "INFO",
  "message": "Health check",
  "endpoint": "/health",
  "status": "ok"
}
```

Two log events per request: one from the `before_request` hook (generic), one from the route handler (route-specific).

---

## Dependencies — `requirements.txt`

```
flask==3.1.0
python-json-logger==2.0.7
```

| Package | Version | Purpose |
|---|---|---|
| `flask` | 3.1.0 | The web framework — handles HTTP routing, request/response |
| `python-json-logger` | 2.0.7 | Formats Python log output as JSON instead of plain text |

**Why pin exact versions?**
Pinning to exact versions (`==`) ensures every developer and every deployment runs the exact same code. A newer version of a library might behave differently or break things silently.

---

## The Dockerfile

```dockerfile
FROM python:3.13-alpine

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

EXPOSE 5000

CMD ["python", "app.py"]
```

**Line by line:**

| Instruction | What it does |
|---|---|
| `FROM python:3.13-alpine` | Start from the official Python 3.13 image built on Alpine Linux. Alpine is tiny (~5 MB) vs the standard image (~900 MB) |
| `WORKDIR /app` | All subsequent commands run inside the `/app` directory inside the container |
| `COPY requirements.txt .` | Copy only the requirements file first — before copying the app code |
| `RUN pip install ...` | Install dependencies. `--no-cache-dir` skips the pip cache to keep the image smaller |
| `COPY app.py .` | Copy the actual application code |
| `EXPOSE 5000` | Document that this container listens on port 5000 (does not actually open the port — that happens in docker-compose) |
| `CMD ["python", "app.py"]` | The command that runs when the container starts |

**Why copy `requirements.txt` before `app.py`?**
Docker builds images in layers. If `app.py` changes but `requirements.txt` does not, Docker reuses the cached layer for `pip install` and only rebuilds from `COPY app.py .` onwards. This makes rebuilds much faster.

---

## Running the App

### With the full log pipeline (recommended)

```bash
# from the app dir: ~/git-repos/demo-docker-app/demo-app
docker compose up -d --build
```

See [elk-configuration.md](./elk-configuration.md) for full setup including the `.env` file and cert.

### Standalone — app only (no log shipping)

If you just want to run the Flask app without any ELK services (run from the app dir, where the `Dockerfile` lives):

```bash
docker build -t flask-demo-app .
docker run -p 5000:5000 flask-demo-app
```

Then test it:
```bash
curl http://localhost:5000/
curl http://localhost:5000/health
```

---

## Useful Commands

### Rebuild after a code change

```bash
# Rebuild only the app image and restart the container
docker compose up -d --build app
```

### View live app logs

```bash
docker compose logs -f app
```

### Open a shell inside the running container

```bash
docker exec -it demo-app-app-1 sh
```

Useful for debugging — you can inspect files, check environment variables, or run Python interactively.

### Check which Python packages are installed

```bash
docker exec demo-app-app-1 pip list
```

---

## How to Add a New Route

1. Add a new route in `app.py`:

```python
@app.route("/version")
def version():
    app.logger.info("Version endpoint accessed", extra={"endpoint": "/version"})
    return jsonify({"version": "1.0.0", "app": "flask-demo-app"})
```

2. Rebuild and restart:

```bash
docker compose up -d --build app
```

3. Test:

```bash
curl http://localhost:5000/version
```

The new route's logs will automatically flow into Elasticsearch via Filebeat and Logstash — no changes needed to the log pipeline.

---

## How to Add New Log Fields

To add a custom field to a log line, pass it in the `extra` dict:

```python
app.logger.info(
    "Something happened",
    extra={"user_id": "abc123", "action": "login", "result": "success"}
)
```

This produces:
```json
{
  "levelname": "INFO",
  "message": "Something happened",
  "user_id": "abc123",
  "action": "login",
  "result": "success"
}
```

These fields are then searchable in Kibana immediately — no Logstash pipeline changes needed.

---

## Log Levels

Python has five log levels. The app is set to `INFO` meaning INFO and above are captured:

| Level | When to use | Example |
|---|---|---|
| `DEBUG` | Detailed internal state (not captured by default) | `Connecting to database at 172.0.0.1` |
| `INFO` | Normal operations | `Health check`, `User logged in` |
| `WARNING` | Something unexpected but not breaking | `Config file missing, using defaults` |
| `ERROR` | Something failed | `Database connection refused` |
| `CRITICAL` | App cannot continue | `Out of disk space, shutting down` |

To log at different levels:
```python
app.logger.debug("Debug message")
app.logger.info("Info message")
app.logger.warning("Warning message")
app.logger.error("Error message")
app.logger.critical("Critical message")
```

---

## Troubleshooting

### App container exits immediately

```bash
docker compose logs app
```

Common causes:
- Python syntax error in `app.py` — fix the error and rebuild
- Missing dependency — add it to `requirements.txt` and rebuild

### Port 5000 is already in use

```bash
# Find what is using port 5000
sudo lsof -i :5000

# Change the port in docker-compose.yml ports section:
# "5001:5000"  (host:container)
```

### App returns 404 for a route

- Check the route decorator matches exactly: `@app.route("/your-path")`
- Flask routes are case-sensitive: `/Health` ≠ `/health`
- Rebuild after any code change: `docker compose up -d --build app`

### Logs are not JSON formatted

Check that `python-json-logger` is installed inside the container:
```bash
docker exec demo-app-app-1 pip show python-json-logger
```

If missing, it means the image was built before `requirements.txt` was updated. Force a full rebuild:
```bash
docker compose build --no-cache app
docker compose up -d app
```

---

*For ELK integration configuration, see [elk-configuration.md](./elk-configuration.md)*
