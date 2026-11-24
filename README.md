# System One Demo

Companion code for the talk "The Two-Tier AI Stack - Decision and Reasoning" at [SG Agentic Nights: November](https://globalai.community/e/xirlve2e) (Global AI Community Singapore, Tue Nov 24, 2026).

The idea: a small System 1 decision model answers focused questions (which team, how urgent, automate or hand to a human) in a few hundred milliseconds, and plain code acts on its answers. A general LLM (System 2) steps in only when System 1 escalates.

This guide gets the core demo running on your own machine: the same four support tickets triaged by System 1, then by System 2. Allow about 15 minutes, plus one large model download. No n8n or Docker experience needed.

## What you need

| Tool | What it does here | Port |
| --- | --- | --- |
| [Docker](https://docs.docker.com/get-docker/) | Runs n8n, the workflow tool the demos are built in | 5678 |
| [ollaya](https://ollaya.dev) with the `laya:en` model | System 1, the decision model | 11435 |
| [Ollama](https://ollama.com) | System 2, a general LLM (several GB download - use good wifi) | 11434 |

Then clone this repo:

```bash
git clone https://github.com/whatevergeek/system-one-demo.git
cd system-one-demo
```

## Get started

### 1. Start n8n

```bash
cp .env.example .env
docker compose up -d
```

Open http://localhost:5678 and create an owner account. It's a local login for your own n8n only.

### 2. Start System 1 (ollaya)

Install ollaya from https://ollaya.dev and check that the `laya:en` model is available:

```bash
ollaya list
```

### 3. Start System 2 (Ollama)

Install Ollama from https://ollama.com, then pull the model for your machine:

```bash
ollama pull gemma4:12b-mlx   # Apple Silicon Mac
ollama pull gemma4:12b       # Windows, Linux or Intel Mac
```

### 4. Import the workflows

In n8n, create a new workflow, open the **...** menu in the canvas header, choose **Import from File**, and import these three files from `workflows/`, in this order:

1. `s2-config.json` - holds the System 2 model name
2. `system1-ticket-triage.json`
3. `system2-ticket-triage.json`

Not on an Apple Silicon Mac? Open the **S2 Config** workflow, change the value in its **S2 Model Name** node to `gemma4:12b`, and save.

### 5. Run it

1. Open **System 1 (Laya) Ticket Triage** and click **Execute workflow**. Double-click the last node, **Route Decision**, to see each ticket's team, confidence, urgency, route, output tokens and latency in milliseconds.
2. Open **System 2 (Ollama LLM) Ticket Triage** and run it the same way. The first run can take a few minutes while Ollama loads the model; later runs are much faster.

Same tickets, same routing rule: System 1 answers in milliseconds, System 2 in seconds. That contrast is the point of the talk.

## New to n8n?

n8n is a visual workflow tool. Each JSON file in `workflows/` imports as a canvas of connected steps called nodes. **Execute workflow** runs it; double-clicking a node shows its settings and the data that went in and out.

## Next steps

- [MORE_DEMOS.md](MORE_DEMOS.md) - the chat front, the five-questions pipeline (classify, route, rank, gate, choose), and using a different System 2 model or an OpenAI-compatible endpoint.
- [REPO_DETAILS.md](REPO_DETAILS.md) - every file in the repo, System 1 vs System 2 evaluations over labelled tickets, bulk import from the terminal, and the fallback plan.
- [DEMO.md](DEMO.md) - the talk's demo-day runbook.

## Troubleshooting

- **The first System 2 run is very slow.** Ollama loads the model into memory on first use and drops it after a few idle minutes. Wait it out (the workflows allow 300 s).
- **System 2 fails or uses the wrong model.** The System 2 workflow reads the model name from `S2 Config`; if it can't, it quietly falls back to `gemma4:12b-mlx`. Double-click its **Get S2 Model** node: if it shows an error, pick `S2 Config` from the node's workflow list (import `s2-config.json` first if it isn't there).
- **Linux: the workflows can't reach ollaya or Ollama.** n8n runs in a container and reaches the two model services on your machine through `host.docker.internal`. Docker Desktop on Mac and Windows provides it; on Linux, add `extra_hosts: ["host.docker.internal:host-gateway"]` to the n8n service in `docker-compose.yml`.
