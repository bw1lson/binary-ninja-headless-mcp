# Binary Ninja Headless MCP

A headless [Binary Ninja](https://binary.ninja/) server that speaks [MCP](https://modelcontextprotocol.io/) (Model Context Protocol), giving AI agents full access to deep reverse-engineering workflows — disassembly, IL, patching, types, xrefs, and more — without a GUI.

Designed to run in the same Docker container as the agent runtime. No sidecars, no extra services.

This entire project---code, tests, and documentation---is 100% vibe coded.

## Why

Existing Binary Ninja MCP servers are either GUI-bound or expose a limited tool surface. This server is headless-only and designed for agent-driven workflows in sandboxed VM/container environments: the agent gets full control over the analysis system, automating large parts of reverse engineering while you interactively discuss and steer the process.

The goal is an interface where agents can inspect, refine, and extend an analysis over time — updating types, symbols, and metadata, improving the analysis database incrementally, applying patches and iterating safely with undo/redo, and running custom scripts when a workflow needs something bespoke.

## Features

- **181 tools** across 36 feature groups: analysis, disassembly, IL, patching, undo/redo, types, workflows, memory, search, xrefs, scripting, and more.
- **Read-only by default** with safe mutation workflows (undo/redo, transactions).
- **Scripting access** via `binja_eval` and `binja_call` for anything the tool catalog doesn't cover.
- **Stdio and TCP transports.**
- **Complete command-line client** (`binja_cli` / `binja-cli`) with schema-derived
  flags, JSON batches, offline discovery, and managed persistent servers.
- **Zero runtime dependencies** beyond Binary Ninja itself.
- **Fake backend mode** for CI and development without a Binary Ninja license.

## Prerequisites

- Python `3.11+`
- A [Binary Ninja](https://binary.ninja/) installation with a headless-capable license and the `binaryninja` Python module importable in your runtime (for real analysis)
- For CI/development without Binary Ninja, use fake backend mode

## Installation

```bash
git clone https://github.com/mrphrazer/binary-ninja-headless-mcp.git
cd binary-ninja-headless-mcp
pip install .
```

Or install directly from the repo root without cloning:

```bash
pip install git+https://github.com/mrphrazer/binary-ninja-headless-mcp.git
```

## Quick Start

### Command-line client

The CLI exposes every MCP tool through the same backend. Its usage follows
Ghidra's `ghidra_cli`. See the [complete CLI guide and generated tool reference](BINJA_CLI.md)
for arguments, defaults, sessions, batches, scripting, output, and troubleshooting.

```bash
binja_cli list --names-only             # Works without Binary Ninja
binja_cli describe session_open
binja_cli server start                  # Requires licensed Binary Ninja
SID=$(binja_cli --field session_id call session_open --path /bin/ls)
binja_cli --session-id "$SID" call binary_functions --limit 5
binja_cli call session_close --session-id "$SID"
binja_cli server stop
```

Use `python3 -m binary_ninja_headless_mcp.binja_cli` from a source checkout.
For a license-free client check, run `binja_cli --fake-backend call health_ping`.
Managed sessions persist across commands; in-process sessions last one invocation.
The managed default is `127.0.0.1:8766`. TCP exposes powerful analysis and Python
execution capabilities without authentication; keep it accessible only to trusted clients.

### MCP server

Stdio transport (default):

```bash
python3 binary_ninja_headless_mcp.py
```

TCP transport:

```bash
python3 binary_ninja_headless_mcp.py --transport tcp --host 127.0.0.1 --port 8765
```

Fake backend mode (no Binary Ninja required):

```bash
python3 binary_ninja_headless_mcp.py --fake-backend
```

## Use With AI Agents

This server speaks standard MCP over `stdio` (default) or `tcp`, so any MCP-capable agent host can use it.

### Claude Code

```bash
claude mcp add binary_ninja_headless_mcp -- python3 /path/to/binary-ninja-headless-mcp/binary_ninja_headless_mcp.py
```

Or add it to your project's `.mcp.json`:

```json
{
  "mcpServers": {
    "binary_ninja_headless_mcp": {
      "command": "python3",
      "args": ["binary_ninja_headless_mcp.py"],
      "cwd": "/path/to/binary-ninja-headless-mcp"
    }
  }
}
```

### Codex

```bash
codex mcp add binary_ninja_headless_mcp -- python3 binary_ninja_headless_mcp.py
```

### Generic MCP Host

- Register a server named `binary_ninja_headless_mcp`.
- Use command `python3` with args `["binary_ninja_headless_mcp.py"]` when `cwd` is the repo root, or use an absolute script path in `args`.
- Set `cwd` to the repo path if you want relative paths like `samples/ls` to resolve correctly.
- Use stdio transport unless your host requires TCP.
- For fake mode (no Binary Ninja installed), append `--fake-backend`.
- Verify connectivity by calling `health_ping`, then `session_open`.

## Docker Co-Location Pattern

Recommended deployment model: run the agent process and this MCP server in the same container image.

Example baseline:

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY . /app
RUN python -m pip install --upgrade pip && pip install ruff pytest
CMD ["python3", "binary_ninja_headless_mcp.py"]
```

If you need real Binary Ninja analysis in-container, add your Binary Ninja runtime + license setup in this same image and start the agent with this MCP server configured.

## MCP Methods

- `initialize`
- `ping`
- `tools/list`
- `tools/call`
- `shutdown`

`tools/list` behavior:

- Without explicit pagination params, returns the full tool catalog.
- If `offset` or `limit` is provided, uses paginated output (`offset=0`, `limit=50` default in paged mode).
- Supports filtering via:
  - `prefix` (for example `binary.`)
  - `query` (substring match against tool name/description)
- Returns pagination metadata: `offset`, `limit`, `total`, `has_more`.
- When a page is truncated (`has_more=true`), includes `next_offset` and a `notice` hint.

Tool call response behavior:

- `structuredContent` is the canonical full payload.
- `content[0].text` is a compact summary string (not full JSON duplication).
- This split is intentional to keep context usage smaller while still exposing full machine-readable data.

## Quality And Testing

This repository is well tested and has enforced quality gates.

The [CLI verification report](CLI_VERIFICATION.md) records native coverage of all
181 tools across both architectures, transports, and argument styles, along with
reproduction commands, installed-package checks, lifecycle evidence, and tested
runtime boundaries. The [CLI guide](BINJA_CLI.md) documents everyday usage.

- Test suite: run `pytest --collect-only -q` for the current collected test count.
- CI workflow enforces:
  - `ruff format --check .`
  - `ruff check .`
  - `pytest`
- CI uses `BINARY_NINJA_HEADLESS_MCP_FAKE_BACKEND=1` so checks run without requiring Binary Ninja installation.
- Additional structural tests verify tool registry consistency and backend reachability.

## Context Controls

- Read-only mode is the default for opened sessions (`read_only=true`).
- `binary_basic_blocks_at` and `function_basic_blocks` are paginated (`offset`/`limit`).
- `memory_read` has a hard response cap: `length <= 65536`.

## Limitations

- Enterprise APIs are currently not covered.
- Debugger APIs are currently not covered.

## Security Model

- MCP communication (`stdio`/`tcp`) is unauthenticated by default.
- The server exposes arbitrary scripting via `binja_eval` and broad API access via `binja_call`.
- This is by design for trusted, containerized agent environments.
- Do not expose this server directly to untrusted users or networks.

## Local Dev Workflow

```bash
ruff format --check .
ruff check .
BINARY_NINJA_HEADLESS_MCP_FAKE_BACKEND=1 pytest -q
```

## Feature Fuzzer

Use the built-in MCP feature fuzzer to exercise a broad tool surface against `samples/ls`.

Real Binary Ninja backend:

```bash
python3 -m binary_ninja_headless_mcp.fuzzer --binary samples/ls --iterations 120 --seed 1337
```

Fake backend smoke run:

```bash
python3 -m binary_ninja_headless_mcp.fuzzer --binary samples/ls --fake-backend --allow-tool-errors --iterations 20
```

Write a JSON coverage report:

```bash
python3 -m binary_ninja_headless_mcp.fuzzer --binary samples/ls --report-json /tmp/mcp-fuzzer-report.json
```

Useful flags:

- `--min-success-tools N`: exits non-zero if fewer than `N` tools succeeded.
- `--verbose`: print each tool call while fuzzing.
- `--update-analysis`: open the seed session with `update_analysis=true`.

## Analysis and session lifecycle

`session_open`, `session_open_bytes`, and `session_open_existing` with
`update_analysis=true` return only after analysis finishes. `analysis_update_and_wait`
has the same synchronous contract. Neither switches to a task or partial result
on a timer. Sessions are registered before analysis; an analysis failure or
cancellation returns a tool error containing the retained `session_id` in both
structured data and compact text, so another call can inspect, retry, or close it.

`analysis_update` now returns a tracked task, just like `task_analysis_update`.
Poll `task_status` and fetch `task_result` when `result_ready` is true. All terminal
states (`completed`, `failed`, `cancelled`) have a result envelope; failed tasks
include `error`. A pending result request is a tool error. This is a deliberate
change from the earlier untracked `analysis_update` response.

Only one managed analysis may be queued, running, or cancelling per session.
Different sessions can analyze concurrently. `analysis_status` adds `status`,
start/completion timestamps, `last_analysis_task_id`, and `last_analysis_error` to
the existing native `state`, `progress`, `info`, and `is_aborted` fields. These
managed fields are compatible with Ghidra's lifecycle vocabulary; the native
progress fields remain Binary Ninja specific. `has_log` is false: there is no
fabricated Ghidra-style analysis log. Native updates triggered by edits or raw API
calls are visible in native progress but are not separate managed tasks.

Cancelling a queued task removes its analysis reservation without aborting the
view. Cancelling running analysis requests a native abort and retains ownership
until both analysis and abort finish. Once terminal, the session can analyze
again; the workflow suspended by our abort is resumed. Cancelling an old terminal
task has no effect on a later run. Running search tasks are best effort and drain
normally if already started.

An explicit `analysis_set_hold` or disabled workflow is preserved. Analysis returns
an actionable error while held or disabled; a normal native wait return only counts
as completion when the engine reports idle and is not aborted. Release the hold or
enable the workflow before retrying.
Explicit native control changes invalidate an overlapping managed run even if a
later control restores an idle state. New analysis requests are rejected while
control calls are still in flight. A later explicit workflow disable also
takes precedence over the automatic resume associated with an earlier cancellation.

Binary Ninja can mark interrupted initial analysis as finished before completing
function discovery. On retry, the server re-requests linear sweep when the view's
`analysis.linearSweep.autorun` setting permits it, preserving its configured
analysis mode. This recovery is retained until a managed analysis completes.
Native heuristics can still produce a different function set after interruption
in reduced analysis modes. Reopen the original input for an independent fresh
analysis when comparing results; completion means the requested native pass
finished, not proof that every possible function was discovered.

`loader_rebase` requires exclusive use of the session. It rejects active analysis
and other admitted operations; new native calls are rejected during replacement,
while status/list calls use cached metadata. Close waits for replacement to finish.
A rebased view keeps ownership of shared native file metadata, and view-specific
caches are reset. Raw evaluation also protects every session exposed in its context
from concurrent close for the duration of the evaluation.
Use the dedicated lifecycle/control tools for cancellation and rebasing; arbitrary
native calls can bypass managed coordination.

`session_close` prevents new operations, cancels work, and waits for admitted MCP
calls and background tasks before closing the view. Reopened upload sessions share
ownership of their temporary backing file; it is deleted after the last dependent
session closes, so closing a parent does not break its children.
This ownership is established before a reopened session becomes visible, including
while its initial analysis is still running.
If draining or aborting exceeds 30 seconds, it returns an error, retains the
resources in `closing` state, and permits a later `session_close` retry.
`analysis_status`, `session_list`, and task metadata remain available. Shutdown
uses one shared 30-second drain deadline; EOF and transport errors also perform
cleanup. The line-delimited stdio transport processes requests sequentially;
use asynchronous tasks for polling on that connection. A second TCP connection
can inspect or cancel a synchronous call on the first connection.

## Lifecycle verification

The dedicated stateful driver checks strict results and clean EOF shutdown over
an actual stdio subprocess, and fails on unexpected errors or timeouts:

```bash
python3 -m binary_ninja_headless_mcp.lifecycle_fuzzer \
  --binary samples/ls --seed 1337 --transitions 100 \
  --output-dir /tmp/binja-lifecycle-1337
BINJA_NATIVE_TEST_BINARY="$PWD/samples/ls" \
  python3 -m pytest tests/test_transport_lifecycle.py
```

Use a fresh output directory per run. The driver preserves the input, writes a
checkpoint database, incremental request/response trace, progress, stderr, input
hash, engine version, and final report. `--timeout` sets the external response/task
watchdog (default 300 seconds); a timeout fails verification. To run additional
seeds on a large binary, `--analysis-database /path/to/checkpoint.bndb` reuses a
completed analysis in a private copy and records the database hash.
`--load-options '{"analysis.limits.cacheSize":64}'` passes explicit native settings
and records them in the report; neither driver changes product defaults. No native
license is needed for `--fake-backend` smoke runs or deterministic lifecycle race tests.

The broad feature fuzzer excludes plugin execution and repository modification
from live runs; these actions have tests using doubles. `--trace-jsonl` preserves
calls incrementally. It reports an unavailable debug parser explicitly. Its default
exit status fails on all unexpected tool errors; `--allow-tool-errors` is for
exploratory diagnostics and is not a correctness gate. Run it on a disposable copy
when exercising writes. The feature fuzzer also accepts `--load-options` and
`--analysis-database`: it copies the database privately and separately probes raw
byte loading without repeating initial analysis.

## Feature Catalog

The server currently exposes `181` tools across `36` feature groups.

### analysis
- `analysis_status`: Get managed lifecycle status/timestamps/task/error plus native state and progress.
- `analysis_progress`: Get analysis progress snapshot.
- `analysis_update`: Start tracked analysis; returns task_id. Rejects overlapping runs on this session.
- `analysis_update_and_wait`: Wait synchronously until analysis finishes; no timed partial-success response.
- `analysis_abort`: Abort analysis.
- `analysis_set_hold`: Hold/release analysis queue.

### annotation
- `annotation_rename_function`: Rename a function.
- `annotation_rename_symbol`: Rename symbol at address.
- `annotation_undefine_symbol`: Undefine user symbol at address.
- `annotation_define_symbol`: Define symbol at address.
- `annotation_rename_data_var`: Rename data variable.
- `annotation_define_data_var`: Define data variable.
- `annotation_undefine_data_var`: Undefine data variable.
- `annotation_set_comment`: Set comment at address.
- `annotation_get_comment`: Get comment at address.
- `annotation_add_tag`: Add user data tag at address.
- `annotation_get_tags`: Get tags at address.

### arch
- `arch_info`: Get architecture and platform metadata.
- `arch_disasm_bytes`: Disassemble bytes with selected architecture.
- `arch_assemble`: Assemble instruction text with selected architecture.

### baseaddr
- `baseaddr_detect`: Run base-address detection.
- `baseaddr_reasons`: Get base-address detection reasons.
- `baseaddr_abort`: Abort base-address detection.

### binary
- `binary_summary`: Get binary/session summary.
- `binary_save`: Save the current binary view to a file path.
- `binary_functions`: List functions with pagination.
- `binary_strings`: List discovered strings with pagination.
- `binary_search_text`: Search raw text/bytes in a session.
- `binary_sections`: List sections with pagination.
- `binary_segments`: List segments with pagination.
- `binary_symbols`: List symbols with pagination.
- `binary_data_vars`: List data variables with pagination.
- `binary_get_function_at`: Find function by address.
- `binary_get_function_disassembly_at`: Get full disassembly for function containing an address.
- `binary_get_function_il_at`: Get full IL for function containing an address.
- `binary_functions_at`: List functions at an address.
- `binary_basic_blocks_at`: List basic blocks at an address with pagination.

### binja
- `binja_info`: Return Binary Ninja version/install info.
- `binja_call`: Generic API bridge: call `bn.*` or `bv.*` target path.
- `binja_eval`: Evaluate Python code with `bn`, `sessions`, and optional `bv`.

### data
- `data_typed_at`: Get typed data variable at an address.

### database
- `database_create_bndb`: Create .bndb from session.
- `database_save_auto_snapshot`: Save auto snapshot.
- `database_info`: Get database status for session.
- `database_snapshots`: List database snapshots.
- `database_read_global`: Read database global string key.
- `database_write_global`: Write database global string key.

### debug
- `debug_parsers`: List debug info parsers valid for this view.
- `debug_parse_and_apply`: Parse debug info and apply it to the view.

### disasm
- `disasm_linear`: Get linear disassembly lines.
- `disasm_function`: Get full disassembly for function containing an address.
- `disasm_range`: Address-range disassembly lines.

### external
- `external_library_add`: Add external library.
- `external_library_list`: List external libraries.
- `external_library_remove`: Remove external library.
- `external_location_add`: Add external location mapping.
- `external_location_get`: Get external location mapping.
- `external_location_remove`: Remove external location mapping.

### function
- `function_basic_blocks`: List basic blocks in a function with pagination.
- `function_callers`: Callers of a function.
- `function_callees`: Callees of a function.
- `function_variables`: List function variables.
- `function_var_refs`: List variable references in MLIL/HLIL.
- `function_var_refs_from`: List variable references originating from an address.
- `function_ssa_var_def_use`: Get SSA variable definition and uses.
- `function_ssa_memory_def_use`: Get SSA memory definition and uses by memory version.
- `function_metadata_store`: Store function metadata by key.
- `function_metadata_query`: Query function metadata by key.
- `function_metadata_remove`: Remove function metadata by key.

### health
- `health_ping`: Health check.

### il
- `il_function`: IL function listing.
- `il_instruction_by_addr`: Get IL instruction by source address.
- `il_address_to_index`: Map address to IL index/indices.
- `il_index_to_address`: Map IL index to source address.
- `il_rewrite_capabilities`: List IL rewrite support for one function and IL level.
- `il_rewrite_noop_replace`: Perform no-op IL expression replacement.
- `il_rewrite_translate_identity`: Translate IL with identity mapping callback.

### loader
- `loader_rebase`: Rebase BinaryView.
- `loader_load_settings_types`: List loader settings type names.
- `loader_load_settings_get`: Get loader settings values.
- `loader_load_settings_set`: Set one loader setting value.

### memory
- `memory_read`: Read bytes from the view (`length <= 65536`).
- `memory_write`: Write bytes (hex) to the view.
- `memory_insert`: Insert bytes (hex) into the view.
- `memory_remove`: Remove bytes from the view.
- `memory_reader_read`: Read integer values via BinaryReader.
- `memory_writer_write`: Write integer values via BinaryWriter.

### mcp
- `mcp_response_format`: Explain tool result fields (`structuredContent` full payload, `content[0].text` summary).

### metadata
- `metadata_store`: Store metadata by key.
- `metadata_query`: Query metadata by key.
- `metadata_remove`: Remove metadata by key.

### patch
- `patch_assemble`: Assemble and patch instruction bytes at address.
- `patch_status`: Inspect patch availability at address.
- `patch_convert_to_nop`: Patch instruction to NOP when supported.
- `patch_always_branch`: Patch conditional branch to always branch when supported.
- `patch_never_branch`: Patch conditional branch to never branch when supported.
- `patch_invert_branch`: Patch conditional branch by inversion when supported.
- `patch_skip_and_return_value`: Patch instruction to skip and return value when supported.

### plugin
- `plugin_valid_commands`: List context-valid plugin commands.
- `plugin_execute`: Execute a context-valid plugin command.

### plugin_repo
- `plugin_repo_status`: List plugin repositories and plugin states.
- `plugin_repo_check_updates`: Check plugin repository updates.
- `plugin_repo_plugin_action`: Run install/uninstall/enable/disable action on repository plugin.

### project
- `project_create`: Create project.
- `project_open`: Open project.
- `project_close`: Close tracked project.
- `project_list`: List project folders/files.
- `project_create_folder`: Create project folder.
- `project_create_file`: Create project file from base64 data.
- `project_metadata_store`: Store project metadata.
- `project_metadata_query`: Query project metadata.
- `project_metadata_remove`: Remove project metadata.

### search
- `search_data`: Search for raw byte patterns (hex string).
- `search_next_text`: Find next text match.
- `search_all_text`: Find all text matches in range (regex optional).
- `search_next_data`: Find next data/byte-pattern match.
- `search_all_data`: Find all data/byte-pattern matches in range.
- `search_next_constant`: Find next constant occurrence.
- `search_all_constant`: Find all constant occurrences in range.

### section
- `section_add_user`: Add user section.
- `section_remove_user`: Remove user section.

### segment
- `segment_add_user`: Add user segment.
- `segment_remove_user`: Remove user segment.

### session
- `session_open`: Open a binary and create a session.
- `session_open_bytes`: Open a binary session from base64-encoded bytes.
- `session_open_existing`: Open another session from an existing session's file.
- `session_close`: Close one open session.
- `session_list`: List open sessions.
- `session_mode`: Get session safety/determinism mode.
- `session_set_mode`: Update session safety/determinism mode.

### task
- `task_analysis_update`: Alias of analysis_update: start tracked analysis and return task_id.
- `task_search_text`: Start async search task.
- `task_status`: Get task status.
- `task_result`: Get a completed, failed, or cancelled result envelope; pending tasks return an error.
- `task_cancel`: Request cancellation. Terminal tasks are unchanged; running analysis drains before reuse.

### transform
- `transform_inspect`: Inspect/process transform extraction pipeline. For a session input, the root context is retained in the selected contexts so temporary transform cleanup preserves the session's view.

### type
- `type_parse_string`: Parse a single type string.
- `type_parse_declarations`: Parse C declarations for types/variables/functions.
- `type_define_user`: Define user type from type source.
- `type_rename`: Rename a type.
- `type_undefine_user`: Undefine a user type.
- `type_import_library_type`: Import type from type library.
- `type_import_library_object`: Import object type from type library.
- `type_export_to_library`: Export type into a type library.

### type_archive
- `type_archive_create`: Create and optionally attach a type archive.
- `type_archive_open`: Open and optionally attach a type archive.
- `type_archive_list`: List attached type archives.
- `type_archive_get`: Get one tracked type archive.
- `type_archive_pull`: Pull types from a type archive.
- `type_archive_push`: Push types to a type archive.
- `type_archive_references`: Query archive incoming/outgoing references for one type.

### type_library
- `type_library_create`: Create and optionally attach a type library.
- `type_library_load`: Load and optionally attach a type library.
- `type_library_list`: List type libraries attached to the view.
- `type_library_get`: Get one tracked type library.

### uidf
- `uidf_parse_possible_value`: Parse user-informed possible value set string.
- `uidf_set_user_var_value`: Set function user variable value.
- `uidf_clear_user_var_value`: Clear function user variable value.
- `uidf_list_user_var_values`: List all user variable values for a function.

### undo
- `undo_begin`: Begin undo transaction.
- `undo_commit`: Commit undo transaction.
- `undo_revert`: Revert undo transaction.
- `undo_undo`: Perform undo.
- `undo_redo`: Perform redo.

### value
- `value_reg`: Get register value at/after an address.
- `value_stack`: Get stack contents at/after an address.
- `value_possible`: Get IL possible value set at an address.
- `value_flags_at`: Get lifted IL flag read/write state at an address.

### workflow
- `workflow_list`: List registered workflows.
- `workflow_describe`: Describe workflow topology and settings.
- `workflow_clone`: Clone workflow.
- `workflow_insert`: Insert activities before an activity.
- `workflow_insert_after`: Insert activities after an activity.
- `workflow_remove`: Remove workflow activity.
- `workflow_graph`: Summarize workflow graph.
- `workflow_machine_status`: Get workflow machine status.
- `workflow_machine_control`: Control workflow machine runtime.

### xref
- `xref_code_refs_to`: Code references to an address.
- `xref_code_refs_from`: Code references from an address.
- `xref_data_refs_to`: Data references to an address.
- `xref_data_refs_from`: Data references from an address.

## Contact

For more information, contact Tim Blazytko ([@mr_phrazer](https://x.com/mr_phrazer)).
