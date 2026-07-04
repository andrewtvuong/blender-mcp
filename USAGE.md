# Usage

This is a security-hardened fork of [ahujasid/blender-mcp](https://github.com/ahujasid/blender-mcp). Follow **this file** for setup — the install instructions in the README are inherited from upstream and point at the PyPI package, which is not this code.

## How this fork differs from upstream

| | Upstream | This fork |
|---|---|---|
| Telemetry | Prompts, code, and screenshots uploaded to a Supabase backend (consent defaults to on) | Removed entirely — no telemetry code, no `httpx` dependency, nothing phones home |
| Addon socket | Unauthenticated; any local process can send commands (including `execute_code`) | Per-session shared-secret auth required on every command; bind pinned to `127.0.0.1` |
| Arbitrary code execution | Always available to the connected LLM | Off by default; opt-in per scene via an addon toggle with a UI warning |

## Requirements

- Blender 3.0+
- [uv](https://docs.astral.sh/uv/) (`brew install uv` on macOS)
- An MCP client: Claude Code, Claude Desktop, Cursor, etc.

## One-time setup

### 1. Install the Blender addon

1. Download `addon.py` **from this repository** (not upstream).
2. If you previously installed the upstream addon, remove it first: Blender → Edit → Preferences → Add-ons → find "Blender MCP" → Remove.
3. Blender → Edit → Preferences → Add-ons → Install… → select `addon.py` → enable the checkbox next to **Interface: Blender MCP**.

The addon starts its socket server automatically when Blender loads (configurable via the *Auto-Start Server* option). On start it:

- listens on `127.0.0.1:9876` (port configurable in the panel), and
- writes a random per-session auth token to `~/.blender-mcp/auth_token` (mode `0600`).

### 2. Connect your MCP client

> **Do not use `uvx blender-mcp`** (the bare PyPI package). That installs the upstream server, which contains the telemetry and does not send the auth token — this fork's addon will reject it with `Unauthorized`.

**Claude Code:**

```bash
claude mcp add --scope user blender -- uvx --from git+https://github.com/andrewtvuong/blender-mcp@main blender-mcp
```

(`--scope user` makes it available in every directory; omit it to register for the current project only.)

**Claude Desktop** — edit the config file (macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "blender": {
      "command": "uvx",
      "args": ["--from", "git+https://github.com/andrewtvuong/blender-mcp@main", "blender-mcp"]
    }
  }
}
```

**Running from a local clone** (picks up local edits immediately):

```json
{
  "mcpServers": {
    "blender": {
      "command": "uv",
      "args": ["run", "--directory", "/path/to/blender-mcp", "blender-mcp"]
    }
  }
}
```

## Daily use

1. Open Blender first — the addon's server starts with it.
2. Start your MCP client. In Claude Code, `/mcp` should show `blender` connected with 22 tools.
3. Ask for what you want: scene inspection, viewport screenshots, PolyHaven/Sketchfab assets, and Hyper3D/Hunyuan3D generation all work through structured tools with no code execution involved.

If you restart Blender mid-session, the auth token rotates. The MCP server reads the token file fresh on every command, so it recovers on its own — if a command errors right at the restart boundary, just retry.

## Arbitrary code execution (`execute_blender_code`)

Disabled by default. Calls to it return an error telling the LLM it is switched off.

To enable: press **N** in the 3D viewport → **BlenderMCP** tab → tick **Allow arbitrary code execution**.

Understand what this means before enabling it: the tool passes LLM-written Python to `exec()` inside Blender. That is **full Python on this machine as your user** — file access, network, subprocesses — not just Blender API calls. `exec()` cannot be sandboxed in-process, so the toggle is a consent gate, not a sandbox. Recommended practice:

- Enable it only for the session that needs it, and turn it off afterwards.
- Never blanket-approve the tool in your MCP client. In Claude Code, leave `mcp__blender__execute_blender_code` on ask-every-time and read the code in each prompt. Approve-per-call is your defense against prompt injection (e.g. hostile instructions hidden in content the model reads).
- If you use it heavily, run Blender under a separate low-privilege OS user or a VM.

## How the socket auth works

- On server start the addon generates a random 64-hex-char token (`secrets.token_hex(32)`) and writes it to `~/.blender-mcp/auth_token` (`0600`, directory `0700`).
- Every command arriving on the socket must carry the token; it is checked with a constant-time comparison before any dispatch. Missing/wrong tokens get `Unauthorized: missing or invalid auth token`.
- The MCP server reads the file per command, so the two processes need no coordination beyond the filesystem.
- The token is deleted when the server stops; a new one is generated per session.

This closes the upstream hole where **any** process on your machine could connect to `localhost:9876` and drive Blender (including code execution) without going through the MCP client at all. It does not protect against other processes running *as your user* — they can read the token file — but that boundary is unenforceable from userspace anyway; the token's job is to stop unprivileged/other-user and sandboxed processes, and accidental exposure.

## Configuration reference

| Setting | Where | Default |
|---|---|---|
| Addon listen port | BlenderMCP panel → *Port* | `9876` |
| Auto-start server with Blender | BlenderMCP panel → *Auto-Start Server* | on |
| Allow arbitrary code execution | BlenderMCP panel | **off** |
| PolyHaven / Hyper3D / Sketchfab / Hunyuan3D | BlenderMCP panel toggles | off |
| MCP server → Blender host | `BLENDER_HOST` env var | `127.0.0.1` |
| MCP server → Blender port | `BLENDER_PORT` env var | `9876` |

If you change the addon port, set `BLENDER_PORT` to match in your MCP client's server config (`"env": {"BLENDER_PORT": "9877"}`).

## Troubleshooting

- **"Could not read Blender auth token … Is the BlenderMCP addon server running?"** — Blender isn't open, the addon isn't enabled, or the server was stopped from the panel. Open Blender / start the server and retry.
- **"Unauthorized: missing or invalid auth token"** — usually a stale token after a Blender restart; retry the command. If it persists, you're running the upstream PyPI server instead of this fork — fix your client config (see setup above).
- **Connection refused on 9876** — server not running, or the port was changed in the panel without setting `BLENDER_PORT`.
- **"Code execution is disabled"** — expected. Enable the toggle only if you actually want the LLM running Python on your machine (see above).

## Staying in sync with upstream

This fork changes the socket protocol (the `auth` field), so blindly merging upstream will conflict and can silently reintroduce telemetry. Treat upstream as a source to cherry-pick from deliberately, and re-review anything touching `addon.py`'s command handling or `server.py`'s tool definitions.
