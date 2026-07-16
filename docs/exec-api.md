# Use the `exec` API

The Code Mode MCP server exposes one tool, `exec`. Its `code` field contains a JavaScript async function body.

## Input

```json
{
  "code": "return search('app screenshot accessibility', { limit: 5 });",
  "session_id": "optional-session",
  "timeout_ms": 120000,
  "max_output_chars": 51200
}
```

Pass raw JavaScript. Do not wrap it as JSON-encoded source or a Markdown code fence.

## Available globals

Each execution can use:

- `search(query, options?)` for ranked discovery
- `describe(name)` for one exact tool schema
- `tools.<name>(args)` for a known tool
- `call(name, args)` for a name chosen at runtime
- `ALL_TOOLS` for the complete frozen catalog
- `ALL_SERVERS` for server status and tool counts
- `text(value)`, `image(value, detail?)` and `emit(block)` for output selection
- `store(key, value)`, `load(key)` and `clearStore(key?)` for JSON state
- `signal` for cancellation

Host APIs such as `process`, `require()`, dynamic `import()`, `fetch()`, filesystem and child processes are also available. Read the [security model](SECURITY.md) before running Code Mode with sensitive systems.

## Discover tools

Search the catalog with a short description:

```js
return search("app screenshot accessibility", { limit: 5 });
```

Search ranks tool names, descriptions, titles, server names and top-level input properties. It returns compact results without schemas.

`search()` returns up to 10 results by default. You can set `server` and `limit`; the maximum limit is 50.

Search several phrasings when the upstream vocabulary is uncertain:

```js
const queries = ["send Teams message", "post chat message", "reply channel"];
const hits = queries.flatMap(query =>
  search(query, { server: "teams", limit: 5 })
);

return [...new Map(hits.map(hit => [hit.name, hit])).values()].slice(0, 10);
```

Inspect one exact schema:

```js
return describe("mcp__computer_use__get_app_state");
```

Inspect a small candidate set:

```js
return search("send Teams message", { limit: 3 })
  .map(hit => describe(hit.name));
```

Rephrase an empty search before deciding that a capability is absent. You can also inspect a bounded part of `ALL_TOOLS`:

```js
return ALL_TOOLS
  .filter(tool => /message|chat|channel/i.test(
    `${tool.name} ${tool.description}`
  ))
  .slice(0, 30);
```

`ALL_SERVERS` contains `{ server, status, toolCount, error? }` summaries. Use it to check server availability and decide whether direct enumeration is practical.

## Use canonical tool names

Each upstream `(server, tool)` pair has one deterministic callable name.

Safe short names use this form:

```text
mcp__<server>__<tool>
```

Unsafe, ambiguous or long names use a sanitised prefix and 16 hexadecimal characters. The suffix comes from SHA-256 over the server name, a NUL byte and the raw tool name.

Canonical names are:

- safe for supported model providers
- valid JavaScript properties
- independent of catalog order
- no longer than 64 characters

Raw upstream names stay in searchable metadata. `call()` and `describe()` accept unambiguous legacy aliases without adding duplicate functions to `tools`.

Hosts can import the naming implementation from `@tmustier/code-mode-mcp/naming`. They can import `createCatalogSearchPage()` from the package root to share ranked discovery with a bounded result page and exact total count.

## Compose calls

Call a known tool through `tools`:

```js
const apps = await tools.mcp__computer_use__list_apps({});
const selected = ["Calculator", "TextEdit"];
const states = await Promise.all(
  selected.map(app =>
    tools.mcp__computer_use__get_app_state({ app })
  )
);

return states.map((state, index) => ({
  app: selected[index],
  text: state.content.find(block => block.type === "text")?.text.slice(0, 500)
}));
```

Use `call(name, args)` when discovery determines the name at runtime.

Code Mode supports loops, branching and parallel calls. Filter and aggregate intermediate results before returning them to the model.

## Return rich output

Return a complete MCP `CallToolResult` to preserve its content blocks, `structuredContent`, `isError` and `_meta`:

```js
return await tools.mcp__computer_use__get_app_state({ app: "Calculator" });
```

Select specific output when an intermediate result is large:

```js
const result = await tools.mcp__computer_use__get_app_state({
  app: "Calculator"
});

text("Current Calculator state");
image(result.content.find(block => block.type === "image"), "original");
```

`console.log()` and related methods are captured and returned. They are not written to MCP standard output.

## Understand output limits

Code Mode bounds returned text in memory. Oversized JSON becomes a valid truncation envelope instead of malformed, character-sliced JSON.

Catalog-shaped arrays keep up to 30 compact entries or 5 detailed schema entries. The result includes the total, omitted count and a filtering hint.

Code Mode does not spill full output to disk. Return a smaller projection when possible.

## Store session state

Store JSON data in process memory:

```js
store("cursor", { page: 2 });
return load("cursor");
```

Use `clearStore()` to remove it. State disappears when the Code Mode server exits. The default `session_id` is `"default"`.
