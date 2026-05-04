# MCP Server Configuration Guide

This guide documents where to configure stdio MCP servers for different Claude environments, based on our experience setting up the remarkable-mcp server.

## Overview

Each Claude environment uses a different configuration method and location:

| Environment | Configuration Method | Scope Options |
|------------|---------------------|---------------|
| Claude Code CLI | Command-based or `.claude.json` | Global or project-local |
| Claude Desktop (macOS) | JSON config file | Global only |
| VS Code Extension | Workspace config file | Workspace only |

## 1. Claude Code CLI

### Method 1: CLI Commands (Recommended)

Use the `claude mcp` commands to manage servers:

```bash
# Add a server (project-local scope)
claude mcp add remarkable -- /path/to/.venv/bin/python -m remarkable_mcp.cli --ssh

# Add a server (global scope - available in all projects)
claude mcp add remarkable -s global -- /path/to/.venv/bin/python -m remarkable_mcp.cli --ssh

# List configured servers
claude mcp list

# Get server details
claude mcp get remarkable

# Remove a server
claude mcp remove remarkable -s local
```

**Configuration Storage:**
- **Local (project):** `.claude.json` in the project directory
- **Global:** `~/.config/claude/config.json` or similar (varies by system)

### Method 2: JSON Config (Alternative)

For complex configurations, use `add-json`:

```bash
claude mcp add-json remarkable '{"type":"stdio","command":"/path/to/python","args":["-m","module.name","--flag"],"env":{}}'
```

### Important Notes

- **Project-local scope:** MCP server only available when running `claude` from within that project directory
- **Global scope:** MCP server available from any directory
- Configuration is stored in `.claude.json` for local or global config files for global

## 2. Claude Desktop (macOS)

### Configuration File Location

```
~/Library/Application Support/Claude/claude_desktop_config.json
```

### Format

```json
{
  "preferences": {
    "quickEntryDictationShortcut": "capslock",
    "localAgentModeTrustedFolders": [
      "/path/to/trusted/folder"
    ]
  },
  "mcpServers": {
    "remarkable": {
      "type": "stdio",
      "command": "/absolute/path/to/.venv/bin/python",
      "args": [
        "-m",
        "remarkable_mcp.cli",
        "--ssh"
      ],
      "env": {}
    }
  }
}
```

### Important Notes

- **Must use absolute paths** - relative paths won't work
- **Restart required** - Must restart Claude Desktop app after editing
- **Sandboxing issues** - macOS app is sandboxed and may not access all filesystem locations (e.g., iCloud Drive paths can be problematic)
- Edit this file manually or via command line tools

## 3. VS Code (Claude Code Extension)

### Configuration File Location

```
.vscode/mcp.json
```

(In the workspace root directory)

### Format

```json
{
  "mcpServers": {
    "remarkable": {
      "type": "stdio",
      "command": "/absolute/path/to/.venv/bin/python",
      "args": [
        "-m",
        "remarkable_mcp.cli",
        "--ssh"
      ],
      "env": {}
    }
  }
}
```

### Important Notes

- **Must use absolute paths** - relative paths won't work
- **Window reload required** - Use `Cmd+Shift+P` → "Developer: Reload Window" after editing
- **Workspace-specific** - Each workspace needs its own configuration
- Can add to `.gitignore` if you don't want to commit to version control

## Common Pitfalls & Solutions

### Issue: `uv run` Doesn't Work

**Problem:**
```json
{
  "command": "uv",
  "args": ["run", "--directory", "/path/to/project", "package-name", "--flag"]
}
```

This fails because `uv run` creates a temporary environment that may not have the package installed.

**Solution:**
Use the virtual environment's Python directly:
```json
{
  "command": "/path/to/project/.venv/bin/python",
  "args": ["-m", "package.module", "--flag"]
}
```

### Issue: Module Not Found

**Problem:**
```
ModuleNotFoundError: No module named 'package_name'
```

**Solution:**
1. Ensure the package is installed in editable mode:
   ```bash
   cd /path/to/project
   uv pip install -e .
   ```

2. Use `python -m module.name` instead of the script executable:
   ```json
   {
     "command": "/path/to/.venv/bin/python",
     "args": ["-m", "package.cli"]
   }
   ```

### Issue: Path with Spaces

**Problem:**
Paths with spaces (e.g., "Mobile Documents") cause issues.

**Solution:**
- Always use the full absolute path as a single string in the `command` field
- Don't try to escape or quote paths in JSON config files
- The JSON parser handles spaces correctly when properly formatted

Example:
```json
{
  "command": "/Users/name/Library/Mobile Documents/path/to/python"
}
```

## Verification Steps

### Claude Code CLI
```bash
cd /path/to/project  # If using local scope
claude mcp list
# Should show: ✓ Connected
```

### Claude Desktop
1. Restart the app
2. Start a conversation
3. Look for MCP server tools in the available tools list

### VS Code
1. Reload the window (`Cmd+Shift+P` → "Developer: Reload Window")
2. Check the Claude Code extension status
3. MCP tools should appear in tool suggestions

## Example: remarkable-mcp Server

This is the working configuration for the remarkable-mcp server:

```json
{
  "type": "stdio",
  "command": "/Users/mark/Library/Mobile Documents/com~apple~CloudDocs/Documents/CursorProjects/remarkable-mcp/.venv/bin/python",
  "args": [
    "-m",
    "remarkable_mcp.cli",
    "--ssh"
  ],
  "env": {}
}
```

Key points:
- Uses `.venv/bin/python` directly (not `uv run`)
- Uses `-m` flag to run as module
- Absolute path to Python interpreter
- Path includes spaces ("Mobile Documents") - works fine in JSON

## Troubleshooting

1. **Check server status:**
   ```bash
   claude mcp list  # For CLI
   ```

2. **Test manual startup:**
   ```bash
   /path/to/.venv/bin/python -m package.cli
   # Should start the MCP server (will wait for JSON-RPC input)
   ```

3. **Check logs:**
   - Claude Code CLI: `~/.claude/debug/`
   - Claude Desktop: Console.app (filter by "Claude")
   - VS Code: Developer Console (`Help` → `Toggle Developer Tools`)

4. **Verify Python can import the module:**
   ```bash
   /path/to/.venv/bin/python -c "import package; print(package.__file__)"
   ```

## Summary

- **Claude Code CLI**: Use `claude mcp add` commands, works from project directory (local) or anywhere (global)
- **Claude Desktop**: Edit `~/Library/Application Support/Claude/claude_desktop_config.json`, restart app
- **VS Code**: Edit `.vscode/mcp.json`, reload window
- **Always use absolute paths** and `python -m` for reliability
- **Avoid `uv run`** - use direct Python interpreter from venv instead
