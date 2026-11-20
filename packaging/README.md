# Packaging and release assets

These files are reusable release templates. They do not identify an active public download service. Replace placeholder URLs and checksums before publishing.

| Path | Purpose |
| --- | --- |
| [PRIVATE_SERVER_LAYOUT.md](PRIVATE_SERVER_LAYOUT.md) | Self-hosted artifact layouts and installation examples. |
| [latest.example.json](latest.example.json) | Template for a release manifest; replace every checksum placeholder. |
| [RELEASE_NOTES_TEMPLATE.md](RELEASE_NOTES_TEMPLATE.md) | Template for future release notes. |
| [RELEASE_NOTES_0.2.5.md](RELEASE_NOTES_0.2.5.md) | Historical release notes for 0.2.5. |
| [RELEASE_NOTES_0.2.6.md](RELEASE_NOTES_0.2.6.md) | Historical release notes for 0.2.6. |
| [site/index.html](site/index.html) | Static release-page template. |
| [systemd/hpc-agent-mcp.service](systemd/hpc-agent-mcp.service) | User service template for the MCP HTTP server. |

## Build from the public source

Run from the project root:

```bash
uv build
```

This creates a wheel and source distribution under `dist/`, which is ignored by Git. Build fresh artifacts from the cleaned source; do not reuse packages from a private checkout. Docker consumes the wheel from `dist/`; see the [Docker guide](../docs/DOCKER.md).

Keep the version in `pyproject.toml`, Docker defaults, and the release manifest consistent when releasing. Preserve version numbers in historical release notes.

Installation supports `HPC_AGENT_WHEEL` for an explicit wheel path/URL, `HPC_AGENT_LATEST_URL` for a manifest URL, or `HPC_AGENT_REPO_BASE` for a configured artifact service. If authentication is required, use `HPC_AGENT_REPO_USERNAME` and `HPC_AGENT_REPO_TOKEN` locally; do not put credentials into release files.

Local `.env` files, SSH keys, job records, logs, and cluster data are not release assets. Windows `:Zone.Identifier` files are download metadata and should not be distributed.
