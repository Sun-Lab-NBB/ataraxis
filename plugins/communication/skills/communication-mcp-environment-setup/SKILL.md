---
name: communication-mcp-environment-setup
description: >-
  Diagnoses and resolves ataraxis-communication-interface MCP server connectivity issues. Covers environment
  verification, command availability, Python version checks, dependency validation, and conda/pip/uv
  environment configuration. Use when the axci (communication-interface) MCP tools are unavailable, when the
  axci server fails to start, when the user reports communication-interface connection issues, or when
  starting a session that requires the axci MCP tools.
user-invocable: false
---

# MCP environment setup

Diagnoses and resolves ataraxis-communication-interface MCP server connectivity and environment configuration issues.

---

## Scope

**Covers:**
- Verifying the ataraxis-communication-interface MCP server is reachable and functional
- Diagnosing why the `axci` command is unavailable
- Checking Python version compatibility
- Validating ataraxis-communication-interface package installation and dependencies
- Environment-specific guidance for conda, pip, and uv workflows
- The external MQTT broker prerequisite, which no installation method provides
- Handing the user a CLI command when the server cannot be restored in-session

**Does not cover:**
- MCP tool usage for microcontroller hardware interaction (see `/microcontroller-setup`)
- MCP tool usage for extraction configuration management (see `/extraction-configuration`)
- MCP tool usage for log data processing (see `/log-processing`, `/log-processing-results`)
- ataraxis-communication-interface package development or contribution workflows

---

## Architecture

ataraxis-communication-interface provides a single MCP server accessed through the `axci` CLI entry point defined in
`pyproject.toml`:

```toml
[project.scripts]
axci = "ataraxis_communication_interface.interfaces.cli:axci_cli"
```

| Server                             | CLI command | Purpose                                                              |
|------------------------------------|-------------|----------------------------------------------------------------------|
| `ataraxis-communication-interface` | `axci mcp`  | Microcontroller discovery, manifest management, log processing tools |

The server accepts `-t`/`--transport`, a `click.Choice` restricted to exactly two values:

| Transport         | Behavior when launched by hand                                                              |
|-------------------|---------------------------------------------------------------------------------------------|
| `stdio` (default) | Carries the JSON-RPC stream over stdout and calls `console.disable()`, so it prints NOTHING |
| `streamable-http` | Serves over HTTP and echoes `Starting AXCI MCP server with streamable-http transport...`    |

**Note:** `run_server()` in `mcp_server.py` also accepts a third value, `sse`, but the CLI `Choice` rejects it, so `sse`
is unreachable through `axci mcp`. Never pass `-t sse`, and never tell the user to.

The communication plugin's `plugin.json` configures the Claude assistant to launch the server automatically:

```json
{
  "mcpServers": {
    "ataraxis-communication-interface": {
      "command": "axci",
      "args": ["mcp"]
    }
  }
}
```

The `axci` command must be on PATH when the Claude assistant starts. This means the Python environment where
ataraxis-communication-interface is installed must be active before launching the assistant.

### Dual-distribution model

The ataraxis communication plugin's Claude integration is split across two distribution channels:

| Component                               | Distributed via                              | What it provides                                                      |
|-----------------------------------------|----------------------------------------------|-----------------------------------------------------------------------|
| Skills (`/microcontroller-setup`, etc.) | ataraxis communication plugin                | Skill files that guide agents through workflows                       |
| MCP server registrations                | ataraxis communication plugin                | Plugin entries that tell the Claude assistant how to start the server |
| MCP server code (`axci mcp`)            | ataraxis-communication-interface pip package | The actual CLI command and server implementation                      |

Installing the plugin alone registers the MCP server and makes skills available, but the server will fail to start
because the `axci` CLI command is not present. The pip package must also be installed in the active Python environment
for the MCP server to function.

This is the most common cause of MCP failures after initial setup. Either the plugin is installed and the pip package is
not, or the pip package sits in a different Python environment than the one active when the Claude assistant launches.

### MQTT broker prerequisite

An MQTT broker is an external service the user installs separately. It is NOT a pip dependency, and no installation
method pulls it in. The library is validated against a locally running
[mosquitto](https://mosquitto.org/) broker, version **2.1.2**.

The broker is required only for sending and receiving data over MQTT. The MCP server starts without one, and every
non-MQTT tool works without one. `check_mqtt_broker_tool` defaults to `127.0.0.1:1883`.

**Note:** a "not reachable" result from `check_mqtt_broker_tool` is NOT an MCP server fault. Do not run this skill's
diagnostic workflow for it, and do not prescribe a reinstall. Tell the user to install and start a broker instead.

---

## Diagnostic workflow

You MUST follow these steps in order when MCP tools are unavailable or the server fails to start.

### Guard: stop if this is an ataraxis-communication-interface checkout

Run this BEFORE any other step, and before prescribing any `pip install`:

```bash
grep -l 'name = "ataraxis-communication-interface"' \
  "$(git rev-parse --show-toplevel 2>/dev/null || pwd)/pyproject.toml" 2>/dev/null
```

If it prints a path, the working directory is inside a source checkout of the library itself. You MUST stop the
diagnostic workflow here and hand off to the user. Every resolution below installs the PyPI wheel, which would shadow
the user's working tree and silently serve stale code to the MCP server.

Tell the user that the developer install path is the one in the repository's `Developers` section. That path is `tox -e
create` to build the `axci_dev` mamba environment, followed by `tox -e install` to install the checkout into it. Then
stop. Development and contribution workflows are outside this skill's scope.

### Step 1: Check MCP server status

Use the `/mcp` slash command or inspect available tools to determine whether the ataraxis-communication-interface MCP
server is connected. If connected, the issue is not environmental, investigate tool-specific errors instead.

### Step 2: Verify command availability

```bash
which axci
```

If the command is not found, proceed to step 3. If found, skip to step 4.

### Step 3: Identify the environment type and resolve

Run these commands to determine the user's environment setup:

```bash
echo "CONDA_PREFIX: ${CONDA_PREFIX:-not set}"
echo "VIRTUAL_ENV: ${VIRTUAL_ENV:-not set}"
python --version
pip list 2>/dev/null | grep ataraxis-communication-interface
```

Based on the output, guide the user through the appropriate resolution:

**Conda environment (CONDA_PREFIX is set but ataraxis-communication-interface is missing):**

The user has an active conda environment but ataraxis-communication-interface is not installed in it. Instruct the user
to install ataraxis-communication-interface into the active environment:

```bash
pip install ataraxis-communication-interface
```

Or if using uv within conda:

```bash
uv pip install ataraxis-communication-interface
```

**Conda environment not activated (CONDA_PREFIX is not set, but conda is available):**

The user needs to activate their environment before launching the Claude assistant. Instruct the user to exit the
assistant and run:

```bash
mamba activate <environment-name>
claude
```

You MUST explain that the Claude assistant inherits the shell environment at launch time. Activating a conda environment
after the assistant has started does not make the `axci` command available to MCP server subprocesses.

**Virtual environment (VIRTUAL_ENV is set but ataraxis-communication-interface is missing):**

```bash
pip install ataraxis-communication-interface
```

**No environment active (both CONDA_PREFIX and VIRTUAL_ENV are unset):**

The user is running in the system Python. If ataraxis-communication-interface is installed globally, `which axci` would
have succeeded. Instruct the user to either activate their environment or install ataraxis-communication-interface into
an accessible location.

### Step 4: Verify Python version compatibility

```bash
python --version
```

ataraxis-communication-interface requires Python `>=3.12,<3.15`. If the Python version does not match, inform the user
that their environment has an incompatible Python version, and they need to create or activate an environment with a
compatible version.

### Step 5: Verify package integrity

```bash
axci --help
```

**Note:** the sibling skills forbid agents from invoking `axci`. `--help` is the ONE exemption to that rule. It is
read-only, it starts no server and touches no hardware, and it reports the installed build rather than a documented
snapshot of it, so it never drifts. Use it to smoke-test the install and to settle any question about a command's real
options. The exemption covers `axci --help` and `axci COMMAND --help` only, and no other `axci` invocation. Always use
the long form: the CLI leaves Click's `help_option_names` at its `["--help"]` default, so `-h` is never a help alias,
and on `axci mqtt` it is bound to `--host`.

If the command fails with an import error, a dependency is missing or broken. Run:

```bash
pip check 2>&1 | head -20
```

`pip check` accepts no package argument and reports the whole environment, so ignore lines about unrelated packages.
Report any missing or incompatible dependency involving ataraxis-communication-interface or one of its dependencies to
the user.

### Step 6: Hand-launch the server as a smoke test

`axci --help` proves the package imports. It does NOT prove the MCP server starts. If steps 2 through 5 all pass and the
tools are still unavailable, have the user launch the server by hand:

```bash
axci mcp -t streamable-http
```

Use `streamable-http`, NOT the `stdio` default. Under `stdio` the server calls `console.disable()` and prints absolutely
nothing, so a healthy server is indistinguishable from a hung one. Under `streamable-http` a healthy server echoes
`Starting AXCI MCP server with streamable-http transport...` and then blocks, serving HTTP until interrupted.

| Observed                              | Meaning                                                                   |
|---------------------------------------|---------------------------------------------------------------------------|
| Startup line, then the process blocks | The server is healthy. The fault is in the assistant's launch environment |
| Traceback instead of the startup line | A broken dependency. Return to step 5                                     |
| Exits immediately with no output      | A crash during startup. Capture stderr and report it                      |

This is a foreground process. Tell the user to interrupt it with Ctrl+C once they have read the result.

### Step 7: Restart the MCP server

After the user resolves the environment issue, they must restart the Claude assistant for the MCP server to pick up the
changes. The ataraxis communication plugin will automatically configure the server on the next session.

---

## Fallback: hand the user a CLI command

When the server cannot be restored in this session, the user cannot restart the assistant right now, or the fix needs an
environment change that only takes effect at the next launch, the work is not necessarily blocked. Several `axci` CLI
commands run entirely without the MCP server.

**The ban on invoking `axci` stays absolute.** You MUST NOT run those commands yourself, even though the shell is
available and they would work. Print the exact command, tell the user to run it, and ask them to paste the output back.
`--help` remains the sole exemption (see step 5).

`/cli-reference` is canonical for the whole `axci` surface, so invoke it for the capabilities that have a CLI path, the
command to hand the user for each one, and the tools that stay blocked until the server is back.

---

## Common issues and resolutions

| Symptom                                 | Resolved by                                              |
|-----------------------------------------|----------------------------------------------------------|
| `axci: command not found`               | Steps 2 and 3                                            |
| Skills available but MCP tools missing  | Step 3, under the dual-distribution model                |
| Python version mismatch                 | Step 4                                                   |
| Import error on `axci mcp`              | Step 5                                                   |
| `axci mcp` prints nothing, appears hung | Step 6                                                   |
| MCP server connected but tools fail     | Step 1                                                   |
| MCP server starts but tools are missing | `pip install --upgrade ataraxis-communication-interface` |
| MQTT broker reported unreachable        | The MQTT broker prerequisite section above               |
| Stale code served from a source clone   | The checkout guard at the top of the workflow            |

---

## Related skills

| Skill                        | Relationship                                                                  |
|------------------------------|-------------------------------------------------------------------------------|
| `/microcontroller-setup`     | Requires the MCP server for hardware discovery and manifest management        |
| `/microcontroller-interface` | Requires the MCP server for API verification                                  |
| `/extraction-configuration`  | Requires the MCP server for config read/write/validate tools                  |
| `/log-processing`            | Requires the MCP server for batch log processing tools                        |
| `/log-processing-results`    | Requires the MCP server for output verification and event query tools         |
| `/log-input-format`          | References MCP server tools for archive discovery and assembly                |
| `/cli-reference`             | Canonical reference for every `axci` command handed to a user in the fallback |
| `/pipeline`                  | Orchestrates all phases that depend on MCP server connectivity                |

---

## Proactive behavior

You should proactively invoke this skill when:
- A session begins and MCP tools from the ataraxis-communication-interface server are expected but unavailable
- Any ataraxis-communication-interface MCP tool call fails with a connection or server error
- The user mentions issues with MCP server connectivity or environment setup

---

## Verification checklist

```text
MCP Environment Setup:
- [ ] Ran the checkout guard before prescribing any install, and handed off if it matched
- [ ] Checked MCP server connection status (ataraxis-communication-interface)
- [ ] Verified 'axci' command is on PATH (which axci)
- [ ] Confirmed Python version matches >=3.12,<3.15
- [ ] Identified environment type (conda, venv, system)
- [ ] Provided environment-specific resolution steps
- [ ] Smoke-tested a hand launch with 'axci mcp -t streamable-http' if steps 2-5 all passed
- [ ] Invoked no 'axci' command other than '--help'
- [ ] Handed the user a CLI command for any tool blocked by a server that stays down
- [ ] Informed user that the Claude assistant must be restarted after environment changes
```
