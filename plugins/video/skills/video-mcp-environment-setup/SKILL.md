---
name: video-mcp-environment-setup
description: >-
  Diagnoses and resolves ataraxis-video-system MCP server connectivity issues. Covers environment
  verification, command availability, Python version checks, dependency validation, and conda/pip/uv
  environment configuration. Use when the axvs (video-system) MCP tools are unavailable, when the axvs server
  fails to start, when the user reports video-system connection issues, or when starting a session that
  requires the axvs MCP tools.
user-invocable: false
---

# MCP environment setup

Diagnoses and resolves ataraxis-video-system MCP server connectivity and environment configuration issues.

---

## Scope

**Covers:**
- Verifying the ataraxis-video-system MCP server is reachable and functional
- Diagnosing why the `axvs` command is unavailable
- Checking Python version compatibility
- Validating ataraxis-video-system package installation and dependencies
- Environment-specific guidance for conda, pip, and uv workflows
- The `--help` exemption to the ban on invoking `axvs`, which this skill owns
- Handing the user a CLI command when the server cannot be restored in-session

**Does not cover:**
- MCP tool usage for camera discovery, configuration, and hardware interaction (see `/camera-setup`)
- MCP tool usage for output verification and archive assembly (see `/post-recording`)
- MCP tool usage for log data processing (see `/log-processing`, `/log-processing-results`)
- The `axvs` command surface and its option reference (see `/cli-reference`)
- ataraxis-video-system package development or contribution workflows

**Handoff rules:** Once the server is reachable, invoke the skill that owns the blocked work and resume there.

---

## Architecture

ataraxis-video-system provides a single MCP server accessed through the `axvs` CLI entry point defined in
`pyproject.toml`:

```toml
[project.scripts]
axvs = "ataraxis_video_system.interfaces.cli:axvs_cli"
```

| Server                  | CLI command | Purpose                                                          |
|-------------------------|-------------|------------------------------------------------------------------|
| `ataraxis-video-system` | `axvs mcp`  | Camera discovery, video session management, log processing tools |

The server accepts a `-t` / `--transport` option restricted to exactly two values, `stdio` (the default) and
`streamable-http`. The underlying `run_server()` function also admits `sse`, but the CLI rejects it, so never run or
recommend `axvs mcp -t sse`. The plugin manifest `.claude-plugin/plugin.json` configures the Claude assistant to launch
the server automatically via its `mcpServers` block:

```json
{
  "mcpServers": {
    "ataraxis-video-system": {
      "command": "axvs",
      "args": ["mcp"]
    }
  }
}
```

The `axvs` command must be on PATH when the Claude assistant starts. This means the Python environment where
ataraxis-video-system is installed must be active before launching the assistant.

### Dual-distribution model

The ataraxis video plugin's Claude integration is split across two distribution channels:

| Component                          | Distributed via                   | What it provides                                                                          |
|------------------------------------|-----------------------------------|-------------------------------------------------------------------------------------------|
| Skills (`/camera-interface`, etc.) | ataraxis video plugin             | Skill files that guide agents through workflows                                           |
| MCP server registrations           | ataraxis video plugin             | `plugin.json` `mcpServers` entries that tell the Claude assistant how to start the server |
| MCP server code (`axvs mcp`)       | ataraxis-video-system pip package | The actual CLI command and server implementation                                          |

Installing the plugin alone registers the MCP server and makes skills available, but the server will fail to start
because the `axvs` CLI command is not present. The pip package must also be installed in the active Python environment
for the MCP server to function.

This is the most common cause of MCP failures after initial setup. Either the plugin is installed but the pip package is
not, or the pip package is installed in a different Python environment than the one active when the Claude assistant
launches.

---

## Diagnostic workflow

You MUST follow these steps in order when MCP tools are unavailable or the server fails to start.

### Step 1: Check MCP server status

Use the `/mcp` slash command or inspect available tools to determine whether the ataraxis-video-system MCP server is
connected. If connected, the issue is not environmental. Investigate tool-specific errors instead.

### Step 2: Verify command availability

```bash
which axvs
```

If the command is not found, proceed to step 3. If found, skip to step 4.

### Step 3: Identify the environment type and resolve

Run these commands to determine the user's environment setup:

```bash
echo "CONDA_PREFIX: ${CONDA_PREFIX:-not set}"
echo "VIRTUAL_ENV: ${VIRTUAL_ENV:-not set}"
python --version
pip list 2>/dev/null | grep ataraxis-video-system
```

Based on the output, guide the user through the appropriate resolution:

**Conda environment (CONDA_PREFIX is set but ataraxis-video-system is missing):**

Instruct the user to install ataraxis-video-system into the active environment:

```bash
pip install ataraxis-video-system
```

Or if using uv within conda:

```bash
uv pip install ataraxis-video-system
```

**Conda environment not activated (CONDA_PREFIX is not set, but conda is available):**

The user needs to activate their environment before launching the Claude assistant. Instruct the user to exit the
assistant and run:

```bash
mamba activate <environment-name>
claude
```

You MUST explain that the Claude assistant inherits the shell environment at launch time. Activating a conda environment
after the assistant has started does not make the `axvs` command available to MCP server subprocesses.

**Virtual environment (VIRTUAL_ENV is set but ataraxis-video-system is missing):**

```bash
pip install ataraxis-video-system
```

**No environment active (both CONDA_PREFIX and VIRTUAL_ENV are unset):**

The user is running in the system Python. If ataraxis-video-system is installed globally, `which axvs` would have
succeeded. Instruct the user to either activate their environment or install ataraxis-video-system into an accessible
location.

### Step 4: Verify Python version compatibility

```bash
python --version
```

ataraxis-video-system requires Python `>=3.12,<3.15`. If the Python version does not match, instruct the user to create
or activate an environment with a compatible version.

### Step 5: Verify package integrity

```bash
axvs --help
```

**This skill owns the one exemption to the ban on invoking `axvs`.** `axvs --help` and `axvs COMMAND --help` may be run:
they are read-only, start no server, touch no hardware, and report the installed build rather than a documented snapshot
of it. Use them to smoke-test the install and to settle any question about a command's real options. No other `axvs`
invocation is exempt. Always use the long form, since the CLI leaves Click's `help_option_names` at its `["--help"]`
default, so `-h` is never a help alias, and on `axvs run` it is bound to `--height`.

If the command fails with an import error, a dependency is missing or broken. Run:

```bash
pip check 2>&1 | head -20
```

`pip check` takes no package argument and verifies every installed distribution, so ignore unrelated package lines and
report only failures involving ataraxis-video-system or one of its dependencies.

### Step 6: Hand-launch the server as a smoke test

`axvs --help` proves the package imports. It does NOT prove the MCP server starts. If steps 2 through 5 all pass and the
tools are still unavailable, **have the user launch the server by hand**. Starting a server is not covered by the
`--help` exemption, so print the command rather than running it:

```bash
axvs mcp -t streamable-http
```

Use `streamable-http`, NOT the `stdio` default. Under `stdio` the server calls `console.disable()` so that no library
output corrupts the JSON-RPC stream on stdout, which means a healthy server prints nothing and is indistinguishable from
a hung one. Under `streamable-http` a healthy server echoes `Starting AXVS MCP server with streamable-http transport...`
and then blocks, serving HTTP until interrupted.

| Observed                                  | Meaning                                                                |
|-------------------------------------------|------------------------------------------------------------------------|
| The startup line, then the process blocks | The server is healthy. The fault is in how the assistant launches it   |
| An import or dependency traceback         | Return to step 5 and repair the installation                           |
| `Error: Invalid value for '-t'`           | The transport was misspelled. Only `stdio` and `streamable-http` exist |

### Step 7: Restart the Claude assistant

After the user resolves the environment issue, they must restart the Claude assistant for the MCP server to pick up the
changes. The ataraxis video plugin will automatically configure the server on the next session.

---

## Common issues and resolutions

| Symptom                                                  | Cause                                  | Resolution                                                    |
|----------------------------------------------------------|----------------------------------------|---------------------------------------------------------------|
| `axvs: command not found`                                | Environment not activated              | Activate conda/venv, then restart the Claude assistant        |
| `axvs: command not found`                                | ataraxis-video-system not installed    | `pip install ataraxis-video-system` in the active environment |
| Import error on `axvs mcp`                               | Missing or incompatible dependency     | `pip install --force-reinstall ataraxis-video-system`         |
| Python version mismatch                                  | Wrong environment activated            | Activate environment with Python >=3.12,<3.15                 |
| MCP server starts but tools are missing                  | Outdated ataraxis-video-system version | `pip install --upgrade ataraxis-video-system`                 |
| MCP server connected but tools fail                      | Not an environment issue               | Check tool-specific error messages                            |
| Skills available but MCP tools missing                   | Plugin installed without pip package   | `pip install ataraxis-video-system` in the active environment |
| GenICam/CTI tools report Unsupported on macOS            | GenICam runtime never installed        | Expected, not a fault. See GenICam runtime availability below |
| GenICam/CTI tools report Unsupported on Linux or Windows | Damaged installation                   | `pip install --force-reinstall ataraxis-video-system`         |

### GenICam runtime availability

The GenICam camera runtime ships as the `harvesters` and `genicam` distributions, which ataraxis-video-system declares
with a `sys_platform != 'darwin'` marker. They install automatically on Linux and Windows, and never on macOS, because
the `genicam` distribution publishes no macOS wheel for some of the Python versions the library supports, so the marker
excludes the runtime there rather than offering it on a subset of them. The library guards the import, so an absent
runtime produces no import error.

Branch on the host platform whenever `check_runtime_requirements_tool` reports `CTI: Unsupported` or
`get_cti_status_tool` reports `CTI: Unavailable.`:

- **macOS**: permanent and by design. Do not prescribe a reinstall. Steer the user to the `opencv` camera
  interface, or to a Linux or Windows host for GenICam cameras. Every other feature, including video encoding
  and log processing, works normally there. The only other macOS restriction is the live frame display, which
  VideoSystem disables because macOS forbids GUI updates outside the main thread.
- **Linux or Windows**: the runtime installs alongside the library there, so its absence means a damaged
  installation. `pip install --force-reinstall ataraxis-video-system` restores it.

For the tool-level consequences of an absent runtime, see `/camera-setup`.

---

## Fallback: hand the user a CLI command

When the server cannot be restored in this session, the user cannot restart the assistant right now, or the fix needs an
environment change that only takes effect at the next launch, the work is not necessarily blocked. Most `axvs` commands
run entirely without the MCP server.

**The ban on invoking `axvs` stays absolute.** You MUST NOT run those commands yourself, even though the shell is
available and they would work. Print the exact command, tell the user to run it, and ask them to paste the output back.
`--help` remains the sole exemption (see step 5).

`/cli-reference` is canonical for the whole `axvs` surface, so invoke it for the capabilities that have a CLI path, the
command to hand the user for each one, and the tools that stay blocked until the server is back.

---

## Related skills

| Skill                     | Relationship                                                                 |
|---------------------------|------------------------------------------------------------------------------|
| `/camera-setup`           | Requires the ataraxis-video-system MCP server for all tool interactions      |
| `/camera-interface`       | Requires the ataraxis-video-system MCP server for API verification           |
| `/post-recording`         | Requires the MCP server for archive assembly and video validation tools      |
| `/cli-reference`          | Owns the `axvs` surface, and the command to hand a user when MCP stays down  |
| `/log-input-format`       | References MCP server tools for archive discovery and assembly               |
| `/log-processing`         | Requires the ataraxis-video-system MCP server for batch log processing tools |
| `/log-processing-results` | Requires the ataraxis-video-system MCP server for frame statistics tools     |
| `/pipeline`               | Orchestrates all phases that depend on MCP server connectivity               |

---

## Proactive behavior

You should proactively invoke this skill when:
- A session begins and MCP tools from the ataraxis-video-system server are expected but unavailable
- Any ataraxis-video-system MCP tool call fails with a connection or server error
- The user mentions issues with MCP server connectivity or environment setup

---

## Verification checklist

```text
MCP Environment Setup:
- [ ] Checked MCP server connection status (ataraxis-video-system)
- [ ] Verified 'axvs' command is on PATH (which axvs)
- [ ] Confirmed Python version matches >=3.12,<3.15
- [ ] Identified environment type (conda, venv, system)
- [ ] Provided environment-specific resolution steps
- [ ] Smoke-tested a hand launch with 'axvs mcp -t streamable-http' if steps 2-5 all passed
- [ ] Invoked no 'axvs' command other than '--help'
- [ ] Handed the user a CLI command for any tool blocked by a server that stays down
- [ ] Informed user that the Claude assistant must be restarted after environment changes
```
