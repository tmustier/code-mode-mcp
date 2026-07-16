# Code Mode MCP

Use JavaScript to discover and compose MCP tools through one agent-agnostic stdio server.

Code Mode exposes one model-facing tool, `exec`. A program can search configured MCP servers, inspect exact schemas, call tools and reduce intermediate results. Upstream schemas stay out of the model's initial context.

## Choose Code Mode by task shape

Use Code Mode for stages that involve:

- large tool catalogs
- repeated calls followed by filtering or aggregation
- long deterministic chains
- large intermediate results that the model does not need to inspect

Keep direct tools available for short tasks, semantic decisions, approvals, errors and rich results. A workflow can switch between direct tools and Code Mode at each stage.

The [direct tools and Code Mode benchmark](docs/benchmarks/2026-07-direct-tools-and-code-mode.md) explains this recommendation.

## How it works

```text
MCP client
  └─ exec({ code })
      └─ code-mode-mcp
          ├─ search and describe upstream tools
          ├─ call tools from JavaScript
          └─ return selected results
```

Code Mode supports:

- stdio, Streamable HTTP and legacy SSE upstream servers
- bearer authentication and OAuth
- cancellation, progress, elicitation, sampling, roots and logging
- text, image, audio, resource and structured results
- explicit JSON-only session state held in memory

Code Mode does not store tool results, screenshots, console output or intermediate values automatically.

## Requirements

You need:

- Node.js 22 or newer
- an MCP client that can launch a stdio server

## Install

Install version 0.4.0 from npm:

```bash
npm install --global @tmustier/code-mode-mcp@0.4.0
code-mode-mcp --help
```

## Configure upstream servers

Create `~/.config/code-mode-mcp/mcp.json`:

```json
{
  "mcpServers": {
    "local": {
      "command": "node",
      "args": ["/absolute/path/to/server.js"]
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

The check excludes commands, arguments, headers, tokens and environment values from its summary.

See [configuration](docs/configuration.md) for transports, environment expansion, OAuth and config lookup.

## Add Code Mode to an MCP client

Add the outer Code Mode server to your client's MCP configuration:

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

Keep the upstream config separate. Do not configure Code Mode as its own upstream server.

## Use the `exec` tool

The `code` field contains a JavaScript async function body:

```json
{
  "code": "return search('app screenshot accessibility', { limit: 5 });",
  "session_id": "optional-session",
  "timeout_ms": 120000,
  "max_output_chars": 51200
}
```

The execution context provides:

- `search()` for ranked tool discovery
- `describe()` for exact schemas
- `tools.<name>(args)` and `call(name, args)` for tool calls
- `ALL_TOOLS` and `ALL_SERVERS` for bounded custom discovery
- `text()`, `image()` and `emit()` for output selection
- `store()`, `load()` and `clearStore()` for in-memory JSON state
- `signal` for cancellation

For example:

```js
const apps = await tools.mcp__computer_use__list_apps({});
const selected = ["Calculator", "TextEdit"];
const states = await Promise.all(
  selected.map(app => tools.mcp__computer_use__get_app_state({ app }))
);

return states.map((state, index) => ({
  app: selected[index],
  text: state.content.find(block => block.type === "text")?.text.slice(0, 500)
}));
```

Code Mode preserves a complete MCP `CallToolResult` when you return it. Filter or aggregate large results inside the program to keep model context small.

See the [`exec` API](docs/exec-api.md) for discovery, canonical tool names, output limits and session state.

## Understand host authority

Code passed to `exec` has the same authority as the Node.js process. It can access the filesystem, network, environment, processes and child processes.

`node:vm` controls execution and interrupts synchronous loops. It is not a security sandbox. Run Code Mode inside the operating system, container, account and credential boundary you want the agent to have.

See the [security model](docs/SECURITY.md) before using Code Mode with sensitive systems.

## Documentation

- [configuration](docs/configuration.md)
- [`exec` API](docs/exec-api.md)
- [architecture](docs/ARCHITECTURE.md)
- [security model](docs/SECURITY.md)
- [architecture decision record](docs/adr/0001-standalone-code-mode-mcp.md)
- [direct tools and Code Mode benchmark](docs/benchmarks/2026-07-direct-tools-and-code-mode.md)

## Develop

```bash
npm ci
npm run check
npm test
npm run prepublishOnly
npm pack --dry-run
```
