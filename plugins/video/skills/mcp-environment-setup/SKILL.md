---
name: mcp-environment-setup
description: >-
  Diagnoses and resolves ataraxis-video-system MCP server connectivity issues. Covers environment
  verification, command availability, Python version checks, dependency validation, and conda/pip/uv
  environment configuration. Use when MCP tools are unavailable, when the server fails to start,
  when the user reports connection issues, or when starting a session that requires MCP tools.
user-invocable: true
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

**Does not cover:**
- MCP tool usage for camera hardware interaction (see `/camera-setup`)
- MCP tool usage for camera discovery and configuration (see `/camera-setup`)
- MCP tool usage for log data processing (see `/log-processing`, `/log-processing-results`)
- ataraxis-video-system package development or contribution workflows

---

## Architecture

ataraxis-video-system provides a single MCP server accessed through the `axvs` CLI entry point defined in
`pyproject.toml`:

```toml
[project.scripts]
axvs = "ataraxis_video_system.cli:axvs_cli"
```

| Server                  | CLI command | Purpose                                                          |
|-------------------------|-------------|------------------------------------------------------------------|
| `ataraxis-video-system` | `axvs mcp`  | Camera discovery, video session management, log processing tools |

The server accepts a `--transport` option (defaults to `stdio`). The project's `.mcp.json` configures the
Claude assistant to launch the server automatically:

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

The `axvs` command must be on PATH when the Claude assistant starts. This means the Python environment
where ataraxis-video-system is installed must be active before launching the assistant.

### Dual-distribution model

The ataraxis video plugin's Claude integration is split across two distribution channels:

| Component                          | Distributed via                   | What it provides                                                           |
|------------------------------------|-----------------------------------|----------------------------------------------------------------------------|
| Skills (`/camera-interface`, etc.) | ataraxis video plugin             | Skill files that guide agents through workflows                            |
| MCP server registrations           | ataraxis video plugin             | `.mcp.json` entries that tell the Claude assistant how to start the server |
| MCP server code (`axvs mcp`)       | ataraxis-video-system pip package | The actual CLI command and server implementation                           |

Installing the plugin alone registers the MCP server and makes skills available, but the server will fail to
start because the `axvs` CLI command is not present. The pip package must also be installed in the active
Python environment for the MCP server to function.

This is the most common cause of MCP failures after initial setup: the plugin is installed but the pip
package is not, or the pip package is installed in a different Python environment than the one active when
the Claude assistant launches.

---

## Diagnostic workflow

You MUST follow these steps in order when MCP tools are unavailable or the server fails to start.

### Step 1: Check MCP server status

Use the `/mcp` slash command or inspect available tools to determine whether the ataraxis-video-system MCP
server is connected. If connected, the issue is not environmental — investigate tool-specific errors instead.

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

The user has an active conda environment but ataraxis-video-system is not installed in it. Instruct the user
to install ataraxis-video-system into the active environment:

```bash
pip install ataraxis-video-system
```

Or if using uv within conda:

```bash
uv pip install ataraxis-video-system
```

**Conda environment not activated (CONDA_PREFIX is not set, but conda is available):**

The user needs to activate their environment before launching the Claude assistant. Instruct the user to
exit the assistant and run:

```bash
mamba activate <environment-name>
claude
```

You MUST explain that the Claude assistant inherits the shell environment at launch time. Activating a
conda environment after the assistant has started does not make the `axvs` command available to MCP server
subprocesses.

**Virtual environment (VIRTUAL_ENV is set but ataraxis-video-system is missing):**

```bash
pip install ataraxis-video-system
```

**No environment active (both CONDA_PREFIX and VIRTUAL_ENV are unset):**

The user is running in the system Python. If ataraxis-video-system is installed globally, `which axvs` would
have succeeded. Instruct the user to either activate their environment or install ataraxis-video-system into
an accessible location.

### Step 4: Verify Python version compatibility

```bash
python --version
```

ataraxis-video-system requires Python `>=3.12,<3.15`. If the Python version does not match, inform the user
that their environment has an incompatible Python version, and they need to create or activate an environment
with a compatible version.

### Step 5: Verify package integrity

```bash
axvs --help
```

If the command fails with an import error, a dependency is missing or broken. Run:

```bash
pip check ataraxis-video-system 2>&1 | head -20
```

Report any missing or incompatible dependencies to the user.

### Step 6: Restart the MCP server

After the user resolves the environment issue, they must restart the Claude assistant for the MCP server to
pick up the changes. The ataraxis video plugin will automatically configure the server on the next session.

---

## Common issues and resolutions

| Symptom                                 | Cause                                  | Resolution                                                    |
|-----------------------------------------|----------------------------------------|---------------------------------------------------------------|
| `axvs: command not found`               | Environment not activated              | Activate conda/venv, then restart the Claude assistant        |
| `axvs: command not found`               | ataraxis-video-system not installed    | `pip install ataraxis-video-system` in the active environment |
| Import error on `axvs mcp`              | Missing or incompatible dependency     | `pip install --force-reinstall ataraxis-video-system`         |
| Python version mismatch                 | Wrong environment activated            | Activate environment with Python >=3.12,<3.15                 |
| MCP server starts but tools are missing | Outdated ataraxis-video-system version | `pip install --upgrade ataraxis-video-system`                 |
| MCP server connected but tools fail     | Not an environment issue               | Check tool-specific error messages                            |
| Skills available but MCP tools missing  | Plugin installed without pip package   | `pip install ataraxis-video-system` in the active environment |

---

## Related skills

| Skill                     | Relationship                                                                 |
|---------------------------|------------------------------------------------------------------------------|
| `/camera-setup`           | Requires the ataraxis-video-system MCP server for all tool interactions      |
| `/camera-interface`       | Requires the ataraxis-video-system MCP server for API verification           |
| `/post-recording`         | Requires the MCP server for archive assembly and video validation tools      |
| `/log-input-format`       | References MCP server tools for archive discovery and assembly               |
| `/log-processing`         | Requires the ataraxis-video-system MCP server for batch log processing tools |
| `/log-processing-results` | Requires the ataraxis-video-system MCP server for frame statistics tools     |
| `/pipeline`               | Orchestrates all phases that depend on MCP server connectivity               |

---

## Proactive behavior

You SHOULD proactively invoke this skill when:
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
- [ ] Informed user that the Claude assistant must be restarted after environment changes
```
