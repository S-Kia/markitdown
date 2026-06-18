# Deploying MarkItDown on Docker

MarkItDown (Microsoft) converts files — PDF, Office docs, HTML, images, audio, etc. — into Markdown. The repo ships two Docker-deployable pieces:

1. **CLI image** (root `Dockerfile`) — a one-shot converter that reads a file on stdin and writes Markdown to stdout.
2. **MCP server image** (`packages/markitdown-mcp/Dockerfile`) — a long-running server exposing a `convert_to_markdown(uri)` tool over STDIO or HTTP/SSE. This is the "deploy as a service" option.

Run everything below from this `markitdown/` folder. Both were verified to build and run correctly (package installs cleanly, both entrypoints work).

## Prerequisites

- Docker Desktop (or Docker Engine) installed and running. Check with `docker --version` and `docker info`.

## Option A — CLI converter image

```bash
# build
docker build -t markitdown:latest .

# convert a file (stdin -> stdout)
docker run --rm -i markitdown:latest < ~/your-file.pdf > output.md
```

Add `--build-arg INSTALL_GIT=true` if you need plugins installed from git.

## Option B — MCP server as a service (recommended for "deploy")

Using the included `docker-compose.yml`:

```bash
docker compose up -d --build      # build + start in the background
docker compose logs -f            # watch logs
docker compose down               # stop
```

The server listens on `http://127.0.0.1:3001` (MCP Streamable HTTP + SSE). Put any files you want to convert into `./data` on the host — they show up at `/workdir` inside the container, so a host file `./data/example.pdf` is referenced as `file:///workdir/example.pdf`.

Or without compose, straight from the package folder:

```bash
docker build -t markitdown-mcp:latest ./packages/markitdown-mcp

# STDIO (e.g. for Claude Desktop):
docker run -it --rm markitdown-mcp:latest

# HTTP server with a mounted data dir:
docker run -it --rm -p 127.0.0.1:3001:3001 -v "$PWD/data:/workdir" \
  markitdown-mcp:latest --http --host 0.0.0.0 --port 3001
```

### Use from Claude Desktop

Add to `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "markitdown": {
      "command": "docker",
      "args": ["run", "--rm", "-i", "markitdown-mcp:latest"]
    }
  }
}
```

## Security note

The MCP server is meant for local use. The compose file publishes the port only to `127.0.0.1`. Do not expose it to other interfaces (`0.0.0.0` on the host) unless you understand the implications — see the project README's security section.
