# Hermes Agent — Portainer Dev Swarm

A self-hosted, multi-language **AI agent swarm** powered by [Hermes Agent](https://github.com/nousresearch/hermes-agent), deployable via Portainer as a Docker Compose stack.

Each agent gets an isolated sandbox container pre-loaded with **Angular**, **PHP + Composer**, **Rust**, **Node 22**, and **Python 3.11** — ready for real development tasks out of the box.

---

## Table of Contents

- [Architecture](#architecture)
- [Prerequisites](#prerequisites)
- [Repository Structure](#repository-structure)
- [Deployment](#deployment)
- [First-Time Setup](#first-time-setup)
- [Using the Agent Swarm](#using-the-agent-swarm)
- [Configuration](#configuration)
- [Updating](#updating)
- [Troubleshooting](#troubleshooting)

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  Portainer Stack: hermes-dev-swarm                  │
│                                                     │
│  ┌──────────────────────┐                           │
│  │  hermes              │  Main agent container     │
│  │  (hermes-agent)      │  Interactive / API        │
│  │                      │  Port 8080 (gateway)      │
│  └────────┬─────────────┘                           │
│           │ spawns via Docker socket                │
│           ▼                                         │
│  ┌──────────────────────┐                           │
│  │  hermes-terminal-dev │  Sandbox image            │
│  │  (per-task)          │  Angular · PHP · Rust     │
│  │                      │  Node 22 · Python 3.11    │
│  └──────────────────────┘                           │
│                                                     │
│  Volume: hermes_data  →  /root/.hermes              │
│  (skills, memory, sessions, config)                 │
└─────────────────────────────────────────────────────┘
```

The `hermes` container orchestrates agents and delegates tasks to isolated `hermes-terminal-dev` sandbox containers spawned on demand via the Docker socket.

---

## Prerequisites

| Requirement | Notes |
|---|---|
| Docker Engine 24+ | With Compose v2 plugin |
| Portainer CE/BE 2.19+ | Community or Business edition |
| Git repository access | Stack must be deployed via Portainer Git integration |
| LLM API key | OpenRouter (recommended) or Anthropic |

> **Important:** Portainer's web editor does not support local `build:` contexts. You must deploy this stack using **Portainer → Stacks → Add stack → Repository** and point it at this Git repo.

---

## Repository Structure

```
.
├── docker-compose.yml          # Portainer stack definition
├── Dockerfile.hermes           # Main agent image
├── Dockerfile.terminal-dev     # Sandbox image (Angular, PHP, Rust, etc.)
├── entrypoint.sh               # Hermes container entrypoint
└── README.md
```

---

## Deployment

### 1. Fork or clone this repository

Host it somewhere Portainer can reach: GitHub, GitLab, Gitea, or a private registry.

### 2. Add the stack in Portainer

1. Open **Portainer → Stacks → Add stack**
2. Select **Repository** as the build method
3. Fill in:
   - **Name:** `hermes-dev-swarm`
   - **Repository URL:** your repo URL
   - **Compose path:** `docker-compose.yml`
4. Optionally add environment variables (see [Configuration](#configuration))
5. Click **Deploy the stack**

Portainer will build both images and start the containers. The first build takes several minutes (Rust toolchain install is the slowest step).

### 3. Verify containers are running

In Portainer, both containers should show as **running**:
- `hermes-dev-swarm`
- `hermes-terminal-dev`

---

## First-Time Setup

Open a **Console** into the `hermes-dev-swarm` container (Portainer → Containers → `hermes-dev-swarm` → Console), then run:

```bash
# 1. Run interactive setup wizard
hermes setup

# 2. Point Hermes at the Docker backend
hermes config set terminal.backend docker
hermes config set terminal.docker_image hermes-terminal-dev:latest

# 3. Add your LLM API key (if not set via environment variable)
echo "OPENROUTER_API_KEY=sk-or-..." >> ~/.hermes/.env
# or
echo "ANTHROPIC_API_KEY=sk-ant-..." >> ~/.hermes/.env

# 4. Launch the agent
hermes
```

---

## Using the Agent Swarm

Once `hermes` is running, you can delegate tasks to specialized subagents. Each subagent spawns in its own sandboxed container with the full dev toolchain available.

```
/delegate Angular frontend developer
/delegate PHP Laravel backend
/delegate Rust performance module
/delegate Python data pipeline
```

Example session:

```
> /delegate Angular frontend developer
  ↳ Task: Build a login form component with reactive forms and validation

> /delegate PHP Laravel backend
  ↳ Task: Create a REST API endpoint for user authentication with JWT

> /delegate Rust performance module
  ↳ Task: Write a zero-copy CSV parser returning parsed rows as JSON
```

Each agent has access to `ng`, `php`, `composer`, `rustc`, `cargo`, `node`, `npm`, `python`, `pip`, `git`, and standard build tools.

---

## Configuration

### Environment Variables

Set these in Portainer under **Stack → Environment variables** or in `~/.hermes/.env` inside the container.

| Variable | Description |
|---|---|
| `OPENROUTER_API_KEY` | API key for OpenRouter (recommended LLM provider) |
| `ANTHROPIC_API_KEY` | API key for Anthropic Claude directly |
| `TZ` | Timezone (default: `UTC`) |

### Hermes Config

Inside the container, configuration lives at `~/.hermes/config.toml` and can also be managed via:

```bash
hermes config list
hermes config set <key> <value>
hermes config get <key>
```

### Ports

| Port | Purpose | Default |
|---|---|---|
| `8080` | Web gateway (Telegram, Discord, HTTP API) | Disabled until configured |

---

## Updating

To pull the latest Hermes Agent from GitHub and rebuild:

1. In Portainer, go to **Stacks → hermes-dev-swarm**
2. Click **Pull and redeploy**

This re-runs `git clone` + `uv pip install` inside the build, picking up any upstream changes. Your `hermes_data` volume (skills, memory, sessions) is preserved across redeployments.

---

## Troubleshooting

### Build fails: `repository does not exist` / pull denied
Make sure there is **no `image:` tag** on either service in `docker-compose.yml`. An `image:` field on a service with `build:` causes Docker to attempt a pull from Docker Hub.

### Build fails: `can't find = in "#"`
Inline comments on `ENV` lines (e.g. `ENV PATH=... # comment`) are not valid in Dockerfiles. Comments must be on their own line.

### Container exits immediately
The `hermes` container uses `entrypoint: ["tail", "-f", "/dev/null"]` to stay alive for interactive use. If overridden accidentally, restore this in `docker-compose.yml`.

### Docker socket permission denied
The Docker socket is mounted without `:ro` so Hermes can spawn sandbox containers. If you see permission errors, ensure the container runs as root or that the `docker` group is configured correctly on the host.

### Rust / cargo not found at runtime
The `ENV PATH="/root/.cargo/bin:${PATH}"` line in `Dockerfile.terminal-dev` must appear **after** the `rustup` install `RUN` step and on its own line without trailing comments.

---

## Security Notes

- The Docker socket (`/var/run/docker.sock`) gives the container significant host access. Run this stack only on trusted infrastructure.
- Store API keys using **Portainer Secrets** or environment variables rather than hardcoding them in the compose file.
- Consider network-isolating the stack if exposing port 8080 publicly.
