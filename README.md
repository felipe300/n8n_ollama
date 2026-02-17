# Local AI Automation Stack

**n8n + Ollama + GPT-OSS**

This repository contains the configuration to run a fully local AI automation pipeline. By combining these three tools, you can create complex workflows that utilize Large Language Models (LLMs) without sending data to external APIs.

## The Stack

| Component | Purpose                                                              |
| --------- | -------------------------------------------------------------------- |
| n8n       | Low-code workflow automation to connect apps and logic.              |
| Ollama    | Local engine to run models like Llama 3, Mistral, or Phi-3.          |
| GPT-OSS   | Open-source interface/bridge to manage and interact with local LLMs. |

## Getting Started

1. Prerequisites

- **Docker & Docker Compose**: The easiest way to run these services in sync.
- **RAM**: Minimum 16GB recommended for running 7B+ parameter models smoothly.
- **GPU (Optional)**: NVIDIA GPU with CUDA support for significantly faster inference.

2. Basic Architecture

Your local setup will communicate over your internal Docker network:

- n8n → talks to → Ollama API (Port 11434)
- GPT-OSS → talks to → Ollama API

3. Quick Start (Docker Compose)

Create a `docker-compose.yml` file and include the following services:

```yaml
services:
  ollama:
    image: ollama/ollama
    container_name: ollama
    ports:
      - "11434:11434"
    volumes:
      - ./ollama_data:/root/.ollama

  n8n:
    image: n8nio/n8n
    container_name: n8n
    ports:
      - "5678:5678"
    environment:
      - N8N_HOST=localhost
    volumes:
      - ./n8n_data:/home/node/.n8n
    depends_on:
      - ollama

  gpt-oss:
    image: [relevant-gpt-oss-image]
    ports:
      - "3000:3000"
    environment:
      - OLLAMA_URL=http://ollama:11434
```

> [!IMPORTANT]
> Remember to add a volume to persist your data
> The docker-compose file already has it, but you can create it manually
> `docker volume create n8n_data`

## Configuration Steps

**Step 1: Pull your Models**

Once Ollama is running, you need to download a model to use. Open your terminal and run:

```bash
docker exec -it ollama ollama run llama3
```

**Step 2: Connect n8n to Ollama**

1. Open n8n at `http://localhost:5678`.
2. Add an AI Agent or Ollama node.
3. Set the Base URL to `http://ollama:11434`.
4. Enter the model name you pulled (e.g., `llama3`).

**Step 3: Access GPT-OSS**

1. Navigate to `http://localhost:3000`.
2. Configure the connection to point to the Ollama container.
3. Use this interface for testing prompts before embedding them into n8n workflows.

## Usage Examples

- **Email Summarizer**: Fetch unread emails via n8n → Send text to Ollama → Post summary to Slack.
- **Local Document QA**: Upload PDFs to a folder → n8n parses text → Ollama answers questions based on content.
- **Content Calendar**: Use GPT-OSS to brainstorm ideas → n8n schedules them into a local database.

⚠️ Important Notes

- Performance: If things feel slow, try using smaller models (e.g., `phi3 or mistral:7b-instruct-v0.2-q4_K_M`).
- Networking: Inside Docker, use the service name (`http://ollama:11434`) instead of localhost.
