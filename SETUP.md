# Setup Guide — HIPAA ML Setup & Multi-Cloud Inference Installation

This guide covers everything needed to install and run the **multi-cloud inference installation** on macOS, Linux, Windows (WSL2), and Chromebook. After completing these steps you will have a local three-node demo environment running HIPAA ML routing with a live dashboard.

---

## Prerequisites

### All platforms

- **Python 3.11+**
- **[Ollama](https://ollama.com)** — runs local LLM inference
- **Git**
- **Redis** (optional — caching is disabled gracefully if unavailable)

### macOS

```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

brew install python@3.11 git redis
brew install --cask ollama
```

### Linux (Ubuntu / Debian)

```bash
sudo apt update
sudo apt install -y python3.11 python3.11-venv python3-pip git curl redis-server

# Install Ollama
curl -fsSL https://ollama.com/install.sh | sh
```

### Windows (WSL2 recommended)

1. Install [WSL2](https://learn.microsoft.com/en-us/windows/wsl/install) with Ubuntu:
   ```powershell
   wsl --install
   ```
2. Open the Ubuntu terminal and follow the **Linux** instructions above.
3. Install [Ollama for Windows](https://ollama.com/download/windows) natively, or run it inside WSL2.

> Native Windows (PowerShell without WSL) is not supported — the demo scripts use bash.

### Chromebook (Linux via Crostini)

1. Enable Linux: **Settings → Advanced → Developers → Linux development environment → Turn On**
2. Open the Linux terminal and follow the **Linux (Ubuntu / Debian)** instructions above.

---

## Setup

### 1. Clone and install dependencies

```bash
git clone https://github.com/PreethiAndichamy342/inference-placement-engine.git
cd inference-placement-engine
pip install -r requirements.txt
```

### 2. Pull the demo model

```bash
ollama pull tinyllama
```

### 3. (Optional) Start Redis

Caching is disabled gracefully if Redis is unavailable, but enabling it gives you routing result caching for non-PHI requests.

**macOS:**
```bash
brew services start redis
```

**Linux / Chromebook:**
```bash
sudo service redis-server start
```

**Windows (WSL2):**
```bash
sudo service redis-server start
```

### 4. Start three simulated cloud environments

```bash
bash scripts/start_demo.sh
```

Expected output:
```
[aws]    PID XXXX on port 11434
[gcp]    PID XXXX on port 11435
[onprem] PID XXXX on port 11436
Model: tinyllama:latest available on all instances.
```

### 5. Start the placement engine

```bash
export PATH="$HOME/.local/bin:$PATH"
uvicorn src.api.main:app --reload --port 8000
```

### 6. Verify everything is running

```bash
curl -s http://localhost:8000/health | python3 -m json.tool
```

Expected:
```json
{
  "status": "ok",
  "healthy_server_count": 3,
  "total_server_count": 3,
  "checked_at": "..."
}
```

Browse the interactive API docs at **http://localhost:8000/docs**  
Browse the live dashboard at **http://localhost:8000/dashboard**

### 7. Stop demo instances when done

```bash
bash scripts/stop_demo.sh
```

---

## Troubleshooting

### `curl: (7) Failed to connect to localhost port 8000`

The API server is not running. Start it with:

```bash
export PATH="$HOME/.local/bin:$PATH"
uvicorn src.api.main:app --reload --port 8000
```

---

### `healthy_server_count: 0` or backends showing unavailable

The demo Ollama instances are not running. Start them:

```bash
bash scripts/start_demo.sh
```

If you see `Demo already running`, stop first:

```bash
bash scripts/stop_demo.sh && bash scripts/start_demo.sh
```

---

### `ModuleNotFoundError` on startup

Dependencies are not installed. Run:

```bash
pip install -r requirements.txt
```

If you have multiple Python versions, ensure you're using Python 3.11+:

```bash
python3 --version
python3 -m pip install -r requirements.txt
```

---

### `ollama pull tinyllama` hangs or fails

- Ensure Ollama is running: open the Ollama app (macOS/Windows) or run `ollama serve` in a separate terminal (Linux).
- Check your internet connection — the model download is ~600 MB.

---

### Redis warnings in server logs

```
cache: Redis unavailable at localhost:6379 — caching disabled
```

This is not an error — caching is disabled gracefully. To enable it, start Redis:

```bash
# macOS
brew services start redis

# Linux / Chromebook / WSL2
sudo service redis-server start
```

---

### Port already in use (`[Errno 98] Address already in use`)

Another process is using port 8000 or one of the Ollama ports (11434–11436). Find and stop it:

```bash
# Find what's using port 8000
lsof -i :8000

# Kill by PID
kill <PID>
```

---

### On Chromebook: `bash: uvicorn: command not found`

Add the local bin directory to your PATH:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

Add this line to `~/.bashrc` to make it permanent.
