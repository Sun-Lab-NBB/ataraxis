---
name: mcp-environment-setup
description: >-
  Diagnoses and resolves MCP server connectivity issues for Ataraxis library servers bundled with the
  implementation plugin. Covers environment verification, command availability, Python version checks,
  and dependency validation. Use when MCP tools are unavailable, when a server fails to start, when
  the user reports connection issues, or when starting a session that requires MCP tools.
---

# MCP Environment Setup

Diagnoses and resolves MCP server connectivity and environment configuration issues for Ataraxis library servers
distributed by the implementation plugin.

---

## Scope

**Covers:**
- Verifying that Ataraxis MCP servers are reachable and functional
- Diagnosing why a CLI command (`axvs`, etc.) is unavailable
- Checking Python version compatibility
- Validating package installation and dependencies
- Environment-specific guidance for conda, pip, and uv workflows

**Does not cover:**
- MCP tool usage for hardware interaction (see `/camera-interface` and other implementation skills)
- Ataraxis library development or contribution workflows

---

## Architecture

The implementation plugin bundles MCP server registrations for Ataraxis hardware libraries. Each library provides its
own MCP server through a CLI entry point:

| Server                   | CLI command  | pip package                    | Purpose                                      |
|--------------------------|--------------|--------------------------------|----------------------------------------------|
| `ataraxis-video-system`  | `axvs mcp`  | `ataraxis-video-system`        | Camera discovery, video session management   |

Additional servers will be added as axci and axmc libraries gain MCP support.

Each server accepts a `--transport` option (defaults to `stdio`). The plugin's `plugin.json` configures Claude Code to
launch all registered servers automatically when the plugin is enabled.

### Dual-distribution model

The implementation plugin's Claude Code integration is split across two distribution channels:

| Component                              | Distributed via          | What it provides                                                   |
|----------------------------------------|--------------------------|--------------------------------------------------------------------|
| Skills (`/camera-interface`, etc.)     | implementation plugin    | Skill files that guide agents through workflows                    |
| MCP server registrations               | implementation plugin    | `mcpServers` entries that tell Claude Code how to start the servers |
| MCP server code (`axvs mcp`, etc.)    | pip packages             | The actual CLI commands and server implementations                 |

Installing the plugin alone registers the MCP servers and makes skills available, but the servers will fail to start
because the CLI commands are not present. The pip package for each library must also be installed in the active Python
environment for its MCP server to function.

This is the most common cause of MCP failures after initial setup: the plugin is installed but the pip package is not,
or the pip package is installed in a different Python environment than the one active when Claude Code launches.

---

## Server registry

Use this table to map an unavailable MCP server to its CLI command and pip package. When diagnosing a specific server,
substitute the correct command and package name throughout the diagnostic workflow.

| MCP server name          | CLI command | pip package                 | Version constraint |
|--------------------------|-------------|-----------------------------|--------------------|
| `ataraxis-video-system`  | `axvs`      | `ataraxis-video-system`     | `>=3.0.0`          |

---

## Diagnostic workflow

You MUST follow these steps in order when MCP tools are unavailable or a server fails to start. Apply these steps to
whichever server is affected, substituting the correct CLI command from the server registry above.

### Step 1: Check MCP server status

Use the `/mcp` slash command or inspect available tools to determine whether the affected MCP server is connected. If
connected, the issue is not environmental — investigate tool-specific errors instead.

### Step 2: Verify command availability

```bash
which <command>
```

For example, for the ataraxis-video-system server:

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
pip list 2>/dev/null | grep <package-name>
```

Based on the output, guide the user through the appropriate resolution:

**Conda environment (CONDA_PREFIX is set but package is missing):**

The user has an active conda environment but the package is not installed in it:

```bash
pip install <package-name>
```

Or if using uv within conda:

```bash
uv pip install <package-name>
```

**Conda environment not activated (CONDA_PREFIX is not set, but conda is available):**

The user needs to activate their environment before launching Claude Code. Instruct the user to exit Claude Code and
run:

```bash
mamba activate <environment-name>
claude
```

You MUST explain that Claude Code inherits the shell environment at launch time. Activating a conda environment after
Claude Code has started does not make CLI commands available to MCP server subprocesses.

**Virtual environment (VIRTUAL_ENV is set but package is missing):**

```bash
pip install <package-name>
```

**No environment active (both CONDA_PREFIX and VIRTUAL_ENV are unset):**

The user is running in the system Python. If the package is installed globally, `which <command>` would have succeeded.
Instruct the user to either activate their environment or install the package into an accessible location.

### Step 4: Verify Python version compatibility

```bash
python --version
```

Check the pip package's Python version constraint. If the Python version does not match, inform the user that their
environment has an incompatible Python version, and they need to create or activate an environment with a compatible
version.

To check the required Python version for a specific package:

```bash
pip show <package-name> | grep Requires-Python
```

### Step 5: Verify package integrity

```bash
<command> --help
```

If the command fails with an import error, a dependency is missing or broken. Run:

```bash
pip check <package-name> 2>&1 | head -20
```

Report any missing or incompatible dependencies to the user.

### Step 6: Restart the MCP server

After the user resolves the environment issue, they must restart Claude Code for the MCP servers to pick up the
changes. The plugin's `mcpServers` configuration will automatically start the servers on the next session.

---

## Common issues and resolutions

| Symptom                                 | Cause                                | Resolution                                              |
|-----------------------------------------|--------------------------------------|---------------------------------------------------------|
| `<command>: command not found`          | Environment not activated            | Activate conda/venv, then restart Claude Code           |
| `<command>: command not found`          | Package not installed                | `pip install <package>` in the active environment       |
| Import error on `<command> mcp`         | Missing or incompatible dependency   | `pip install --force-reinstall <package>`               |
| Python version mismatch                 | Wrong environment activated          | Activate environment with compatible Python version     |
| MCP server starts but tools are missing | Outdated package version             | `pip install --upgrade <package>`                       |
| MCP server connected but tools fail     | Not an environment issue             | Check tool-specific error messages                      |
| Skills available but MCP tools missing  | Plugin installed without pip package | `pip install <package>` in the active environment       |

---

## Related skills

| Skill                | Relationship                                                        |
|----------------------|---------------------------------------------------------------------|
| `/camera-interface`  | Requires the ataraxis-video-system MCP server for API verification  |
| `/camera-setup`      | Requires the ataraxis-video-system MCP server for all tool interactions |

---

## Proactive behavior

You SHOULD proactively invoke this skill when:
- A session begins and MCP tools from an Ataraxis library server are expected but unavailable
- Any Ataraxis library MCP tool call fails with a connection or server error
- The user mentions issues with MCP server connectivity or environment setup

---

## Verification checklist

```text
MCP Environment Setup:
- [ ] Identified which MCP server(s) are affected using /mcp or tool inspection
- [ ] Looked up affected server in the server registry table
- [ ] Verified CLI command is on PATH (which <command>)
- [ ] Confirmed Python version is compatible with the package
- [ ] Identified environment type (conda, venv, system)
- [ ] Provided environment-specific resolution steps
- [ ] Informed user that Claude Code must be restarted after environment changes
```
