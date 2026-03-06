# Caldera MCP Plugin for Claude Code

A Claude Code plugin that connects to your Ignition SCADA gateway through [Caldera MCP](https://github.com/caldera-mcp/caldera-mcp), providing 110+ tools for exploration, debugging, diagnostics, and development planning.

## What's included

- **MCP Server Connection** -- Automatically connects to your running Caldera MCP server via native HTTP transport
- **Domain Skill** -- Teaches Claude how to effectively use Caldera MCP tools, including:
  - Perspective view debugging (common pitfalls, binding issues, transform chains)
  - Jython 2.7 scripting rules for `execute_script`
  - Gateway exploration workflows
  - ISA-101/HMI design guidance integration

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

By default, the plugin connects to `http://localhost:8765/mcp`. If your Caldera MCP server runs on a different host or port, edit `.mcp.json` in the plugin directory:

```json
{
  "mcpServers": {
    "caldera-mcp": {
      "type": "http",
      "url": "http://your-host:your-port/mcp"
    }
  }
}
```

## How it works

Claude Code natively supports HTTP Streamable transport, so the plugin connects directly to your Caldera MCP server -- no intermediary packages required.

```
Claude Code  <--HTTP-->  Caldera MCP Server  <-->  Ignition Gateway
```

## Available tools (via Caldera MCP)

The MCP server provides 110+ tools across these categories:

| Category | Examples |
|----------|----------|
| Views | `read_view`, `list_views`, `get_component`, `search_components` |
| Tags | `browse_tags`, `read_tags`, `write_tags` |
| Scripts | `execute_script`, `script_session_start/eval/end`, `read_script` |
| Database | `run_named_query`, `list_named_queries` |
| Schemas | `get_component_schema`, `get_binding_schema`, `get_expression_reference` |
| Design | `get_design_guidance`, `search_icons` |
| Visual | `screenshot_view`, `get_view_console_errors` |
| Gateway | `get_gateway_health`, `get_gateway_diagnostics` |

## License

MIT
