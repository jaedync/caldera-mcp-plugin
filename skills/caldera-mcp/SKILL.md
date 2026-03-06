---
name: caldera-mcp
description: >
  How to effectively use Caldera MCP tools for Ignition SCADA exploration, debugging, diagnostics,
  and development planning. Use this skill whenever the user is working with an Ignition gateway
  through MCP tools — debugging views, exploring project structure, investigating database or tag
  issues, planning new features, reading scripts/views/queries, running diagnostic Jython scripts,
  or discussing Ignition Perspective development. Trigger when you see Caldera MCP tools like
  read_view, execute_script, browse_tags, search_components, list_views, get_component,
  read_script, run_named_query, screenshot_view, etc. Even if the user doesn't mention "Caldera"
  explicitly, if they're interacting with an Ignition gateway through MCP, this skill applies.
---

# Caldera MCP — Exploration, Debugging & Planning Skill

Caldera MCP gives you read access to a live Ignition gateway: its views, scripts, tags, database
queries, component schemas, and runtime state. Your job is to help experienced Ignition developers
explore, debug, diagnose, and plan — not to build things end-to-end. Think of yourself as a
senior colleague who can instantly search the entire project, run test queries, trace data flows,
and propose solutions grounded in what actually exists on the gateway.

## Communication Style

**Be direct. Lead with findings, not preamble.** When the user asks a question, answer it first.
Don't open with gateway health summaries, environment descriptions, or status reports unless
the user specifically asked for them. Get to the point — these are experienced developers who
want answers, not orientation briefings.

## Environment Awareness

**This tool should only be pointed at development servers.** On your first interaction:

1. Silently call `get_gateway_health` to orient yourself — note the gateway version, bridge
   status, safety settings, and project list for your own context. Do NOT dump this information
   to the user unless they ask for it or something is wrong.

2. **Check what environment you're connected to.**
   - **QA server**: Proceed with significant caution. Default to read-only operations even if
     writes are enabled. If the user asks you to write/modify resources, confirm with them
     first: "This is a QA server — are you sure you want me to modify [resource] directly?
     I can prepare the changes and let you review them first." Always validate changes with
     `validate_view_structure` before writing. Never make bulk changes on QA.
   - **Production server**: **Stop immediately.** Do NOT run any tools against production —
     not even read-only ones. Caldera MCP should not be connected to production gateways.
     Flag this to the user, recommend they disconnect, and offer to help via Script Console
     scripts they can run manually. Develop and test those scripts against dev first.

3. Check if read-only mode is active (the health check tells you). Writes will be blocked if
   it is — and that's usually intentional. **Never suggest enabling writes, toggling write
   mode, or changing safety settings.** The user controls read-only mode through the Caldera
   dashboard — not through you. If a task requires writes and they're disabled, tell the user
   what you would change and let them decide whether to enable writes themselves.

## How to Explore a Gateway

The most valuable thing you can do is help the user understand what's on their gateway.

### Project Structure

Start broad, then drill down:

```
get_project_overview(project)     → Resource counts: views, scripts, queries, UDTs, events
list_views(project)               → All view paths — scan for patterns, naming conventions
list_scripts(project)             → Project library scripts
list_named_queries(project)       → All named queries
list_udts(project)                → User Defined Types
list_gateway_events(project)      → Timer, startup, shutdown, tag change events
```

When the user asks "where is X?" or "how does Y work?", these are your starting points.
Cross-reference: use `find_view_usage(project, view_path)` to trace where a view is embedded.

### Reading Resources

Read things before guessing about them:

```
read_view(project, view_path)                    → View JSON (summary by default for large views)
read_view(project, view_path, summary=false)     → Full JSON (careful with huge views)
get_component(project, view_path, "root/flex/label_0")  → One specific component
read_script(project, script_path)                → Project library script source
read_named_query(project, query_path)            → Query definition, parameters, SQL
read_udt(project, udt_path)                      → UDT definition with tag structure
read_gateway_event(project, event_name)          → Event handler script
read_page(project, page_path)                    → Page configuration
```

For large views, prefer `get_component` to read specific subtrees rather than pulling the
entire view JSON. The summary mode gives you the component tree structure so you can identify
the path you need, then drill in.

### Exploring Tags

Tag paths always include the provider in brackets:

```
browse_tags("[default]")                  → Top-level tag tree
browse_tags("[default]Motors")            → Drill into a folder
read_tags(["[default]Motor1/Speed", "[default]Motor1/Current"])  → Current values + quality
```

Common providers: `[default]` for user tags, `[System]` for gateway system tags.
If you're unsure of the structure, start with `browse_tags("[default]")` and navigate from there.

## execute_script: Your Most Powerful Exploration Tool

`execute_script` runs Jython 2.7 on the gateway with full access to the Ignition scripting API.
This is your Swiss Army knife for troubleshooting — use it to query databases, inspect tags,
test expressions, trace data flows, and validate hypotheses about how the system behaves.

### Probing Database Issues

When the user is debugging data problems, write and run queries directly:

```python
execute_script("""
# Test a named query's underlying logic
results = system.db.runNamedQuery("project", "Equipment/GetAlarms", {"areaId": 42})
_result = {
    "rowCount": results.rowCount,
    "columns": list(results.columnNames),
    "sample": [
        {col: results.getValueAt(r, col) for col in results.columnNames}
        for r in range(min(5, results.rowCount))
    ]
}
""")
```

```python
execute_script("""
# Run raw SQL to investigate a data issue
from java.lang import String
ds = system.db.runQuery("SELECT TOP 10 * FROM alarm_events ORDER BY eventtime DESC", "MyDB")
_result = {
    "columns": list(ds.columnNames),
    "rows": [
        {col: str(ds.getValueAt(r, col)) for col in ds.columnNames}
        for r in range(ds.rowCount)
    ]
}
""")
```

### Inspecting Tag Configuration and Values

```python
execute_script("""
# Deep-inspect a tag's configuration, not just its value
import json
config = system.tag.getConfiguration("[default]Motors/Motor1", True)
tag_list = list(config)
if tag_list:
    tag = tag_list[0]
    _result = {
        "name": str(tag.get("name", "")),
        "tagType": str(tag.get("tagType", "")),
        "valueSource": str(tag.get("valueSource", "")),
        "dataType": str(tag.get("dataType", "")),
        "opcServer": str(tag.get("opcServer", "")),
        "opcItemPath": str(tag.get("opcItemPath", ""))
    }
else:
    _result = {"error": "Tag not found"}
""")
```

### Testing Expressions and Transforms

```python
execute_script("""
# Test an expression before putting it in a binding
result = system.tag.readBlocking(["[default]Motor1/Speed"])[0]
speed = result.value

# Replicate what an expression binding would do
status = "Running" if speed > 0 else "Stopped"
color = "#22C55E" if speed > 0 else "#EF4444"

_result = {"speed": speed, "quality": str(result.quality), "status": status, "color": color}
""")
```

### Script Sessions for Iterative Investigation

When you're chasing a bug and need to build up context step by step:

```
script_session_start()                        → Returns session_id
script_session_eval(session_id, code_step_1)  → Run first probe, inspect result
script_session_eval(session_id, code_step_2)  → Dig deeper based on findings
script_session_eval(session_id, code_step_3)  → Variables persist across calls
script_session_end(session_id)                → Clean up when done
```

Sessions are ideal when you don't know what you're looking for yet — you can explore
interactively, keeping state between steps. They timeout after 1 hour.

### Jython 2.7 — Critical Syntax Rules

Scripts run in **Jython 2.7**, not Python 3. You WILL break scripts if you use Python 3 syntax.
Review every script you write against this list before running it:

**String formatting — NO f-strings:**
```python
# WRONG — SyntaxError in Jython 2.7
msg = f"Speed: {speed} RPM"
# RIGHT
msg = "Speed: {} RPM".format(speed)
msg = "Speed: %s RPM" % speed
```

**Dict merging — NO unpacking:**
```python
# WRONG — SyntaxError in Jython 2.7
combined = {**defaults, **overrides}
# RIGHT
combined = dict(defaults.items() + overrides.items())
```

**No walrus operator, no type hints, no dataclasses:**
```python
# WRONG
if (n := len(items)) > 10:
def getStatus(path: str) -> dict:
# RIGHT
n = len(items)
if n > 10:
def getStatus(path):
```

**Integer division:**
```python
# WRONG — returns 2, not 2.5
result = 5 / 2
# RIGHT
result = 5.0 / 2
```

**Other Jython 2.7 traps:**
- `_result` is how you return values — assign your output dict to `_result`
- `print` is a statement (but `print("x")` works in both 2 and 3)
- No `pathlib`, `typing`, `enum` (use string constants), `collections.OrderedDict` works
- `str` is bytes, `unicode` is text — but for MCP scripts this rarely matters
- List comprehensions work. Dict comprehensions work. Set comprehensions work.
- `try/except Exception as e:` works (not `except Exception, e:` — that's Python 2.5 style)

Use `validate_script(code)` to syntax-check before running if you're unsure.

### Bridge Execution Context

`execute_script` runs through the WebDev bridge, which has some differences from the
Ignition Script Console:

- **Project library imports don't resolve.** You cannot `from myproject.utils import helper`.
  Instead, inline the logic you need or use `system.util.getProjectName()` to verify context.
- **Return values via `_result`.** The bridge captures the `_result` variable — this is the
  ONLY way to get structured data back. `print()` output goes to stdout but isn't returned
  in the tool result.
- **Timeouts.** Long-running scripts may timeout. For heavy queries, limit your result set
  (e.g., `SELECT TOP 100`) and paginate if needed.
- **No GUI access.** `system.gui.*`, `system.nav.*`, and `system.perspective.openPopup` are
  not available in the bridge context. Stick to `system.tag.*`, `system.db.*`, `system.date.*`,
  `system.alarm.*`, and `system.util.*`.
- **Named queries need the 3-argument form.** In gateway scope (which the bridge runs in),
  `system.db.runNamedQuery("queryPath", params)` fails or returns empty. You need the 3-arg
  form: `system.db.runNamedQuery("projectName", "queryPath", params)` — the project name
  is required because there's no implicit project context in gateway scope.

## Providing Scripts for QA/Production

When the user needs to debug something in QA or production (where Caldera MCP shouldn't be
connected), develop and test the script against dev first, then provide it formatted for
Ignition's Script Console:

```python
# Script Console version — paste this into the Ignition Designer Script Console
# Investigates: [describe what this checks]
results = system.db.runNamedQuery("project", "path/to/query", {"param": value})
for row in range(results.rowCount):
    print("{}: {}".format(
        results.getValueAt(row, "name"),
        results.getValueAt(row, "status")
    ))
```

Test the logic on dev via `execute_script`, then hand the user a clean version they can
paste into the Script Console on the target environment. This keeps production safe while
still helping them debug.

## Debugging Views

### Tracing Binding Issues

When a view isn't displaying data correctly:

1. `read_view(project, view_path)` — Check the view structure and bindings
2. `get_component(project, view_path, component_path)` — Zoom into the broken component
3. Look at its bindings: what tags, expressions, or queries is it bound to?
4. Use `execute_script` or `read_tags` to check if the underlying data source is returning
   what's expected
5. Use `get_view_console_errors(project, view_path)` to capture runtime JS errors and
   binding failures (requires Playwright)
6. `screenshot_view(project, view_path)` to see what it actually looks like

### Understanding Component Schemas

When the user is confused about why a component isn't behaving as expected:

```
search_components(query="table")              → Find the exact type name
get_component_schema("ia.display.table")      → Full property schema, events, examples
get_binding_schema("tag")                     → How tag bindings work
get_expression_reference(function_name="if")  → Expression language reference
```

### Common Perspective Pitfalls

These trip up even experienced developers. When debugging a view, check for ALL of these —
they cause silent failures with no error messages:

**Flex containers — style vs props:**
Layout CSS goes in `props.style`, not directly in `props`. This is the #1 Perspective trap.
```json
// WRONG — silently ignored, container defaults to row
{"props": {"flexDirection": "column"}}
// RIGHT — CSS properties go in style
{"props": {"style": {"flexDirection": "column"}}}
```
The component also has a native `props.direction` property, but CSS in `props.style` is the
standard Perspective convention. If you see `flexDirection` as a direct prop, it's a bug.

**Embedded view params — paramDirection:**
The TARGET view must have `paramDirection: "input"` on each parameter it accepts from a parent,
or parent-provided values are silently ignored. You can diagnose this from the JSON (`propConfig`
will be empty or missing `paramDirection`), but the fix is done in the Perspective Designer —
right-click each parameter in the view's params and set its direction. This isn't something
you edit in JSON directly.

**Dropdown options — object format required:**
Dropdown options MUST be an array of `{value, label}` objects. Plain string arrays render
but bind incorrectly — the selected "value" will be the display text rather than a
meaningful identifier, and option-dependent logic breaks.
```json
// WRONG — renders but value binding is broken
"options": ["Manual", "Auto", "Off"]
// RIGHT — proper value/label separation
"options": [
  {"value": 0, "label": "Manual"},
  {"value": 1, "label": "Auto"},
  {"value": 2, "label": "Off"}
]
```

**Tag paths — provider brackets required:**
Tag paths MUST include the provider in brackets. Without it, Ignition returns
`Bad_NotFound` with `"Tag provider '' not found"` — the binding silently fails.
```
WRONG: WaterTreatment/ClearWell/Level
RIGHT: [default]WaterTreatment/ClearWell/Level
```

**Tag bindings — config.tagPath not config.path:**
In view JSON, tag binding configurations use `config.tagPath`, not `config.path`. If you
see `config.path` in a tag binding, the binding will resolve to null with no error. The
binding schema shows `tagPath` as the correct key.

**Table column render property:**
Table columns use `render` for cell rendering configuration (e.g., progress bars, toggles),
not `renderer` or `cellRenderer`. Check `get_component_schema("ia.display.table")` for the
exact format.

Use `validate_view_structure(view_json)` — it checks for many of these automatically.

### Transform Chaining — {value} Changes Meaning

When a binding has multiple transforms, each transform's `{value}` is the **output of the
previous transform**, not the original binding value. This causes subtle bugs:

```
Tag binding: [default]ClearWell/Temperature → 18.7 (Celsius)
Transform 1 (expression): {value} * 9.0 / 5.0 + 32    → 65.66 (Fahrenheit)
Transform 2 (expression): if({value} > 50, 'HIGH', 'NORMAL')
                          → {value} is 65.66 (F), NOT 18.7 (C)!
                          → Shows 'HIGH' even though 18.7°C is normal
```

If a threshold check is wrong after a unit conversion transform, check whether the threshold
accounts for the converted units. Either adjust the threshold in the later transform or
restructure to do the comparison before the conversion.

### Expression Bindings and Tag References

In Perspective expression bindings (`type: "expr"`), you MUST use the `tag()` function to
read tag values. A tag path as a string literal is just a string — it doesn't read the tag:

```
WRONG — this is a string literal, not a tag read:
  concat('Level: ', '[default]WaterTreatment/ClearWell/Level', '%')
  → Shows: "Level: [default]WaterTreatment/ClearWell/Level%"

RIGHT — tag() function reads the actual tag value:
  concat('Level: ', tag('[default]WaterTreatment/ClearWell/Level'), '%')
  → Shows: "Level: 72.3%"
```

This is different from tag bindings (`type: "tag"`) where the tag path is in the config and
`{value}` refers to the tag's current value.

### Cross-View Communication

View parameters (`view.params`) only flow **parent → child** through embedded views. They
do NOT work for:
- Direct page navigation (typing a URL or using nav actions)
- Sibling views at the same level
- Communication between docked views and the main page

When a parameterized view works embedded but shows defaults when navigated to directly, the
params aren't being provided because there's no parent. Solutions:

1. **URL parameters** — Pass params in the URL: `/view-path?pumpPath=...`. Configure the
   page's URL params in the page configuration.
2. **Session custom properties** — Use `session.custom.selectedPump` to share data across
   all views in a session. Set via script: `self.session.custom.selectedPump = value`
3. **Page params** — Configure default param values in the page configuration
4. **Detect standalone mode** — Check if params are empty and show a selector/default view

## Planning New Features

When the user has a user story or feature request and wants to plan implementation:

1. **Explore what exists** — Use `get_project_overview`, `list_views`, `list_scripts` to
   understand the current project structure, naming conventions, and patterns already in use.

2. **Find similar implementations** — If they want to build a new motor detail screen, search
   for existing equipment views: `list_views(project)` and look for patterns. Read a few
   representative views to understand the team's conventions.

3. **Check available components** — `search_components` and `get_component_schema` to
   identify the right components for the job. `get_design_guidance(topic)` for ISA-101/HMI
   best practices on the relevant topic (color, layout, alarms, navigation, etc.).

4. **Verify data availability** — Use `browse_tags`, `read_tags`, `list_named_queries`,
   and `execute_script` to confirm the data sources the feature needs actually exist
   and return the expected shape.

5. **Propose a plan** — Based on what you've found: suggest view hierarchy, component
   choices, data binding strategy, and any new tags/queries/scripts that would be needed.
   Ground every recommendation in what you actually observed on the gateway.

## Schema and Design Reference

These tools are your reference library — use them to give well-informed recommendations:

- `search_components(query, category)` — Find component types by keyword or category
- `get_component_schema(type)` — Full property schema with events and examples
- `get_binding_schema(type)` — Tag, property, expression, or query binding reference
- `get_expression_reference(function_name, category)` — Ignition expression language docs
- `get_design_guidance(topic)` — ISA-101/EEMUA 201 HMI design guidance (14 topics)
- `search_icons(query, library)` — Perspective icon search across all libraries

## Visual Tools

When Playwright is available:

- `screenshot_view(project, view_path)` — See what a view actually looks like
- `get_view_console_errors(project, view_path)` — JS errors and binding failures
- `screenshot_gateway_page("/web/status")` — Gateway status pages
- `get_gateway_diagnostics()` — CPU, memory, threads (no browser needed)
- `list_perspective_sessions()` — Who's connected and what they're looking at

## Safety Awareness

Even in read-mostly mode, understand the safety layers:

- **Auto-backups**: Before any write, the previous version is saved automatically
- **Auto-snapshots**: Before significant writes, a project snapshot is taken (15 min cooldown)
- **Audit logging**: Every tool call is recorded to SQLite — useful for reviewing what changed
- **Write-pause**: The dashboard can pause all writes; the health check shows this state

### Write Operations Checklist

If writes are enabled and the user asks you to modify something, follow this sequence:

1. **Read the current state first** — never write blind
2. **Validate your changes** with `validate_view_structure` before writing
3. **Use surgical edits** — prefer `update_component` or `write_view` in `patch` mode over
   full view rewrites. Component-level edits reduce the risk of accidentally clobbering
   other parts of the view.
4. **For bulk operations**, take a manual `snapshot_project` first
5. **Verify after writing** — re-read the resource to confirm your changes landed correctly

On QA servers, always confirm with the user before any write. On dev, proceed but still
follow the checklist.

## API-Generated Tools (8.3+ Only)

Ignition 8.3 gateways have ~45 additional tools generated from the OpenAPI spec, covering
gateway management (projects, modules, backups, licensing, logs). These are off by default
(they add context bloat) — enable via the dashboard settings if needed for specific gateway
administration tasks. For day-to-day exploration and debugging, you won't need them.

## ClickUp Integration

If a ClickUp MCP server is connected, use it as a **final report**, not a progress log.

### Commenting on Tasks

When you finish a task or subtask, leave **one** comment summarizing the outcome. The comment
should be self-contained — readable by someone who only has access to ClickUp, not your
local environment.

**What to include:**
- What was accomplished (specific outcome, not process)
- Which Ignition resources were modified and why (view paths, tag paths, script paths, query paths)
- Any relevant findings or decisions made during the work

**What NOT to include:**
- References to local files, paths on your machine, or conversation transcripts
- Step-by-step progress updates ("first I read the view, then I checked the tags...")
- Tool names or MCP internals ("I used `read_view` to investigate...")
- Anything that requires access to the developer's local environment to understand

**Good example:**
> Fixed the `high_level` label binding in `Settings/AlarmConfig` (water-treatment project).
> The tag path was missing the `[default]` provider prefix — changed from
> `WaterTreatment/ClearWell/HighLevelAlarm` to `[default]WaterTreatment/ClearWell/HighLevelAlarm`.
> Also corrected the binding config key from `config.path` to `config.tagPath`.
> Both issues caused the binding to silently fail with no error.

**Bad example:**
> Completed the task. Reference `/Users/dev/caldera-mcp-skill-workspace/iteration-4/eval-10/response.md`
> for details. Used read_view and get_component to diagnose the issue.

If the user asks you to write something specific to the task description or a comment, do
exactly what they ask. But for your own summary comments, keep them targeted and self-contained.

## Effectiveness Tips

1. **Read before you guess.** The gateway has the answers — read the view, read the script,
   browse the tags, run the query. Don't speculate when you can verify.

2. **Use execute_script aggressively for debugging.** It's the fastest way to test a theory
   about why something isn't working. Write a quick probe, run it, see what comes back.

3. **Batch tag reads.** `read_tags` takes an array — one call with 20 paths is far faster
   than 20 individual calls.

4. **Use view summaries for orientation.** `read_view` defaults to summary mode for large
   views. Get the tree structure first, then use `get_component` to drill into specific parts.

5. **Script sessions for complex investigations.** When you need to query, transform, and
   cross-reference data across multiple steps, a session lets you build up state without
   losing context between calls.

6. **Ground your recommendations.** When proposing a solution or design, cite what you found
   on the gateway: "I see you're using flex repeaters with embedded views for equipment
   screens in /Equipment/* — I'd suggest the same pattern here."
