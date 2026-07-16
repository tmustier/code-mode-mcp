# Configure Code Mode

Code Mode reads a separate list of upstream MCP servers. This keeps it independent from the outer MCP client and prevents recursive self-registration.

## Create the config file

Create `~/.config/code-mode-mcp/mcp.json`:

```json
{
  "settings": {
    "executionTimeoutMs": 120000,
    "requestTimeoutMs": 120000
  },
  "mcpServers": {
    "local": {
      "command": "node",
      "args": ["/absolute/path/to/server.js"],
      "requestTimeoutMs": 180000
    },
    "remote": {
      "url": "https://example.com/mcp",
      "auth": "bearer",
      "bearerTokenEnv": "EXAMPLE_MCP_TOKEN"
    }
  }
}
```

Check the file without starting MCP:

```bash
code-mode-mcp --check-config \
  --config ~/.config/code-mode-mcp/mcp.json
```

The check returns server names, transports and authentication types. It excludes commands, arguments, headers, tokens and environment values.

## Choose the config path

You can pass an exact path with `--config`.

Without `--config`, Code Mode uses the first file it finds:

1. `$CODE_MODE_MCP_CONFIG`
2. `./.code-mode-mcp.json`
3. `~/.config/code-mode-mcp/mcp.json`
4. `~/.config/pi-code-mode-mcp/mcp.json`

`PI_CODE_MODE_MCP_CONFIG` and `PI_CODE_MODE_MCP_HOME` remain as compatibility fallbacks.

## Configure a server

Each entry in `mcpServers` must define one connection type:

- `command` for a stdio process
- `url` for a remote server

A stdio server can also set `args`, `env` and `cwd`.

A URL server can set `transport`, `headers` and authentication. Code Mode tries Streamable HTTP first and falls back to SSE. Set `transport` to `"streamable-http"` or `"sse"` to require one transport.

Strings support `${VAR}` and exact `$env:VAR` environment expansion. Relative `cwd` and `settings.stateDir` paths resolve from the config file's directory.

## Configure OAuth

Set `auth` to `"oauth"` for an OAuth server:

```json
{
  "mcpServers": {
    "linear": {
      "url": "https://mcp.example.com/mcp",
      "auth": "oauth",
      "oauth": {
        "grantType": "authorization_code",
        "scope": "read write"
      }
    }
  }
}
```

For authorization-code OAuth, Code Mode starts a loopback callback and sends the authorization URL through MCP URL elicitation. The outer client decides whether to open it. Code Mode returns `cancel` when interaction is unavailable.

Code Mode stores OAuth tokens, dynamic client registration, PKCE verifiers and discovery metadata in mode-0600 files under `settings.stateDir`. The default directory is `~/.config/code-mode-mcp`.

Code Mode does not store tool results in this directory.

## Add Code Mode to the outer client

Configure Code Mode as a stdio MCP server:

```json
{
  "mcpServers": {
    "code-mode": {
      "command": "npx",
      "args": [
        "-y",
        "@tmustier/code-mode-mcp@0.4.0",
        "--config",
        "/Users/you/.config/code-mode-mcp/mcp.json"
      ]
    }
  }
}
```

Keep the outer and upstream configurations separate. Do not add Code Mode to its own upstream file.

## Configure Pi

Pi can use Code Mode through `pi-mcp-adapter`. Add it to `~/.pi/agent/mcp.json` or `.pi/mcp.json`:

```json
{
  "mcpServers": {
    "code-mode": {
      "command": "npx",
      "args": [
        "-y",
        "@tmustier/code-mode-mcp@0.4.0",
        "--config",
        "/Users/you/.config/code-mode-mcp/mcp.json"
      ],
      "lifecycle": "lazy",
      "requestTimeoutMs": 180000,
      "directTools": ["exec"]
    }
  }
}
```

Restart or reload Pi after changing its MCP configuration. Pi's normal direct tools remain available alongside Code Mode.
