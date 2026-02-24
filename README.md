# Local AI Automation Stack

**n8n + Ollama + GPT-OSS**

This repository contains the configuration to run a fully local AI automation pipeline. By combining these three tools, you can create complex workflows that utilize Large Language Models (LLMs) without sending data to external APIs.

## The Stack

| Component | Purpose                                                              | Local URL              |
| --------- | -------------------------------------------------------------------- | ---------------------- |
| n8n       | Low-code workflow automation to connect apps and logic.              | http://localhost:5678  |
| Ollama    | Local engine to run models like Llama 3, Mistral, or Phi-3.          | http://localhost:11435 |
| GPT-OSS   | Open-source interface/bridge to manage and interact with local LLMs. | http://localhost:3000  |

## Getting Started

1. Prerequisites

- **Docker & Docker Compose**: The easiest way to run these services in sync.
- **RAM**: Minimum 16GB recommended for running 7B+ parameter models smoothly.
- **Local Ollama (Optional)**: If you have the native Ollama app running on your host, it will use port 11434. The Docker version is mapped to 11435 to avoid conflicts.
- **GPU (Optional)**: NVIDIA GPU with CUDA support for significantly faster inference.

2. Basic Architecture

Your local setup will communicate over your internal Docker network:

- n8n → talks to → Ollama API (Port 11434)
- GPT-OSS → talks to → Ollama API

3. Quick Start (Docker Compose)

Create a `docker-compose.yml` file and include the following services.

The following configuration ensures that the containerized n8n has the correct permissions and networking access to both the Docker Ollama instance and your host machine.

```yaml
services:
  ollama:
    image: ollama/ollama
    container_name: ollama
    ports:
      - "11435:11434"
    volumes:
      - ./ollama_data:/root/.ollama

  n8n:
    image: n8nio/n8n
    container_name: n8n
    # user: "${USER_ID}:${GROUP_ID}"
    user: "1000:1000"
    restart: always
    extra_hosts:
      - "host.docker.internal:host-gateway"
    deploy:
      resources:
        limits:
          memory: 1G
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=0.0.0.0
      - WEBHOOK_URL=http://localhost:5678/
      - OLLAMA_URL=${OLLAMA_URL:-http://host.docker.internal:11434}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_BLOCK_FS_WRITE_ACCESS=false
      - N8N_DEFAULT_BINARY_DATA_MODE=filesystem
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=false
      - N8N_RESTRICT_FILE_ACCESS_TO=/data;/home/node/.n8n-files
      - N8N_ALLOWED_FILESYSTEM_PATHS=/data,/home/node/.n8n-files
    volumes:
      - ./n8n_data:/home/node/.n8n
      - ./n8n_files:/data
    depends_on:
      - ollama

  gpt-oss:
    image: ghcr.io/open-webui/open-webui:main
    container_name: open-webui
    ports:
      - "3000:8080"
    environment:
      - OLLAMA_BASE_URL=http://ollama:11434
    volumes:
      - ./open-webui:/app/backend/data
    depends_on:
      - ollama
```

> [!WARNING]
> This process may take some time
> wait 30 to 1 minute, just to be sure

> [!IMPORTANT]
> Remember to add a volume to persist your data
> The docker compose file already has it, but you can create it manually
> `docker volume create n8n_data`

## Configuration Steps

**Step 1: Fix Permissions**

Before starting the containers, you must ensure the local folders are writable by the Docker user (ID 1000). Run this in your terminal:

To avoid `EACCES: permission denied` errors, create the data directories and set the correct ownership before starting the containers:

```sh
mkdir -p n8n_data n8n_files ollama_data open-webui

sudo chown -R 1000:1000 ./n8n_data ./n8n_files ./ollama_data ./open-webui
chmod -R 775 ./n8n_data ./n8n_files ./ollama_data ./open-webui
```

**Step 2: Pull your Models**

Once Ollama is running, you need to download a model to use. Open your terminal and run:

```bash
# Run docker compose
docker compose up -d
docker compose down

# Pull model
docker exec -it ollama ollama pull llama3
```

**Step 3: Connect n8n to Ollama**

Open n8n at `http://localhost:5678`. When adding an Ollama node, use the following Base URL depending on which instance you want to use:

- To use the Docker Ollama: `http://ollama:11434`
- To use your Native Ollama App: `http://host.docker.internal:11434`

**Step 4: Access GPT-OSS**

1. Navigate to `http://localhost:3000`.
2. Configure the connection to point to the Ollama container.
3. Use this interface for testing prompts before embedding them into n8n workflows.

## Usage Examples

- **Email Summarizer**: Fetch unread emails via n8n → Send text to Ollama → Post summary to Slack.
- **Local Document QA**: Upload PDFs to a folder → n8n parses text → Ollama answers questions based on content.
- **Content Calendar**: Use GPT-OSS to brainstorm ideas → n8n schedules them into a local database.

## Troubleshooting

**Permission Denied (EACCES)**

If n8n logs show `permission denied, open '/home/node/.n8n/config'`, it means the folder ownership is still set to `root`. Repeat the `chown` command in **Step 1**.

**Mismatching Encryption Keys**

If you see an "Encryption Key Mismatch" error, n8n is protecting your data from a previous installation. To reset:

1. Stop the containers: `docker compose down`
2. Delete the config file: `rm ./n8n_data/config`
3. Restart: `docker compose up -d`

**Localhost Networking**

Inside the Docker network, containers cannot reach each other via `localhost`. Always use the service name (e.g., `http://ollama:11434`) or `host.docker.internal` to reach your actual computer.

⚠️ Important Notes

- Performance: If things feel slow, try using smaller models (e.g., `phi3 or mistral:7b-instruct-v0.2-q4_K_M`).
- Networking: Inside Docker, use the service name (`http://ollama:11434`) instead of `localhost`.
