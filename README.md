# Caldera MCP Plugin for Claude Code

A Claude Code plugin that connects to your Ignition SCADA gateway through [Caldera MCP](https://github.com/caldera-mcp/caldera-mcp), providing 71+ tools for exploration, debugging, diagnostics, and development planning.

## What's included

- **MCP Server Connection** -- Automatically connects to your running Caldera MCP server via `mcp-remote` over HTTPS
- **5 Domain Skills** -- Workflow-specific guidance that loads on demand via progressive disclosure:

| Skill | Triggers when | What it provides |
|-------|--------------|-----------------|
| `exploring` | Browsing project structure, listing resources, reading views/scripts/tags | Project navigation workflows, tag exploration, search patterns |
| `debugging-views` | View not displaying correctly, broken bindings, layout issues | Binding trace workflow, silent failure catalog, transform chain debugging |
| `writing-jython` | Running scripts on gateway, probing databases, testing expressions | Jython 2.7 syntax rules, bridge execution context, script session management |
| `planning` | Feature requests, building new screens, component selection | Existing pattern analysis, component schemas, ISA-101/HMI design guidance |
| `safety-writes` | Any write/delete operation (auto-activates) | Write checklist, environment classification, backup awareness |

- **Shared Reference Files** -- Detailed domain knowledge that loads only when needed:
  - Jython 2.7 syntax rules (no f-strings, no walrus operator, integer division)
  - Perspective silent failure catalog (9 common pitfalls)
  - Binding type reference (tag, expression, property, query)
  - Bridge execution context (limitations vs Script Console)
  - Common tool sequences by task type

## Prerequisites

- [Claude Code](https://code.claude.com) v1.0.33+
- [Caldera MCP server](https://github.com/caldera-mcp/caldera-mcp) running on your Ignition gateway

## Install

### From marketplace

```bash
/plugin marketplace add jaedync/caldera-mcp-plugin
/plugin install caldera-mcp@caldera-mcp
```

### Local development

```bash
claude --plugin-dir ./caldera-mcp-plugin
```

## Configuration

By default, the plugin connects to `https://localhost:8766/mcp` (Caldera MCP's HTTPS port) via `mcp-remote`. If your server runs on a different host or port, edit `plugins/caldera-mcp/.mcp.json`:

```json
{
  "mcpServers": {
    "caldera-mcp": {
      "command": "npx",
      "args": ["-y", "mcp-remote", "https://your-host:your-port/mcp"],
      "env": {
        "NODE_TLS_REJECT_UNAUTHORIZED": "0"
      }
    }
  }
}
```

## How it works

The plugin uses `mcp-remote` (auto-installed via npx) to connect to your Caldera MCP server over HTTPS via stdio transport.

```
Claude  <--stdio-->  mcp-remote  <--HTTPS-->  Caldera MCP Server  <-->  Ignition Gateway
```

## Available tools (via Caldera MCP)

The MCP server provides 71+ tools across these categories:

| Category | Examples |
|----------|----------|
| Views | `read_view`, `list_views`, `read_component`, `search_components` |
| Tags | `browse_tags`, `read_tags`, `write_tags` |
| Scripts | `execute_script`, `script_session_start/eval/end`, `read_script` |
| Database | `run_named_query`, `list_named_queries` |
| Schemas | `get_component_schema`, `get_binding_schema`, `get_expression_reference` |
| Design | `get_design_guidance`, `search_icons` |
| Visual | `screenshot_view`, `get_view_console_errors` |
| Gateway | `get_gateway_health`, `get_gateway_diagnostics` |

## License

MIT
