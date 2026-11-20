# HPC Agent documentation

Start with the [project README](../README.md) for installation and an English overview. Most detailed guides below are currently in Chinese.

## Guides

| Document | Purpose |
| --- | --- |
| [User guide](USER_GUIDE.md) | Configuration, everyday use, Slurm and VASP workflows, diagnostics, and TUI commands. |
| [Docker deployment](DOCKER.md) | Building and running the optional container image. |
| [MCP overview](mcp_docs/MCP.md) | Tools, resources, structured arguments, execution switches, and smoke tests. |
| [MCP clients](mcp_docs/MCP_CLIENTS.md) | Connecting compatible AI and MCP clients. |
| [ChatGPT Web integration](mcp_docs/MCP_CHATGPT_WEB.md) | HTTPS tunnel connection steps. |
| [MCP deployment](mcp_docs/MCP_DEPLOYMENT.md) | HTTP server, health checks, systemd, logs, and auditing. |
| [MCP app metadata](mcp_docs/MCP_CHATGPT_APP_METADATA.md) | Client-facing descriptions and example prompts. |
| [External MCP tools](mcp_docs/EXTERNAL_MCP_INJECTION.md) | Connecting external servers and injecting allowlisted tools. |
| [Skills architecture](skills_architecture.md) | Skill registry, execution adapters, and intent routing. |
| [External skills](EXTERNAL_SKILLS.md) | External skills, handler templates, and debugging. |
| [Live test checklist](LIVE_TEST_CHECKLIST.md) | Validation against a real HPC environment. |
| [VASP test templates](VASP_TEST_TEMPLATES.md) | Input templates, resource suggestions, and acceptance criteria. |
| [Error library maintenance](../data/errors/README.md) | Case drafting, generic rules, and redaction. |
| [Packaging and releases](../packaging/README.md) | Build commands, release templates, and artifact organization. |

## Suggested reading order

1. Install from source using the project README and configure your local `.env`.
2. Read the user guide for the workflows you need.
3. Use VASP test templates when preparing smoke tests; use the live checklist when you are ready to submit real jobs.
4. For external clients, read the MCP overview and the relevant client/deployment guide.
5. For extensions, read the skills architecture, external skills guide, or external MCP guide.

## Common entry points

```bash
# TUI from a source checkout
uv run python app.py

# Installed TUI and MCP server
hpc-agent
hpc-agent-mcp
hpc-agent-mcp --transport streamable-http --host 127.0.0.1 --port 8000 --path /mcp
curl http://127.0.0.1:8000/health

# External MCP tools
hpc-agent-mcp-client doctor
hpc-agent-mcp-client list-tools

# Local checks; add --live-hpc only to run a real submission test
hpc-agent-check
```

In the TUI, use `/help`, `/help job`, `/help vasp`, `/help skill`, `/doctor`, `/resources`, `/skill list`, and `/skill test all`. Use `/clear` or `/clear all` to clear the conversation, and `/exit` to quit.

## Maintenance

Keep the root README focused on the project overview, setup, and core workflows. Keep this page as an index instead of duplicating detailed guides. Register new documentation here and update `pyproject.toml` if it must be included in installed distributions.
