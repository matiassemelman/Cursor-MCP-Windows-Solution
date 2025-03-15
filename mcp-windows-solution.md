# MCP Servers Solution for Windows

## Context

The Model Context Protocol (MCP) provides new tools for the Cursor Agent. MCP servers like Firecrawl-MCP (for web scraping) or GitHub-MCP (for GitHub interaction) present specific execution issues on Windows systems.

## Preliminary Steps

Before configuring MCP servers, make sure to complete the following steps:

1. Activate early access in Cursor Settings -> Beta -> Update Frequency. This gives us access to the latest update that introduces: MCP: Added global server configuration with ~/.cursor/mcp.json and support for environment variables.

2. Update to version 0.47.5

## Problem

MCP servers don't remain active on Windows, showing errors such as "Client closed" shortly after startup or "Authentication Failed: Bad credentials" when trying to use the tools.

### Initial Setup

The initial configuration attempts to run the server directly:

```json
{
  "mcpServers": {
    "firecrawl-mcp": {
      "command": "cmd",
      "args": [
        "/k",
        "set",
        "FIRECRAWL_API_KEY=your_api_key",
        "&&",
        "npx",
        "-y",
        "firecrawl-mcp"
      ]
    }
  }
}
```

This configuration leads to issues like "Client closed" or unrecognized credentials.

## Problem Analysis

### Differences Between Windows and Unix Systems

In Windows systems:
1. When a parent process (cmd.exe) terminates, it typically closes all child processes
2. The `/c` flag in cmd.exe executes the command and then immediately closes the window
3. Node.js on Windows depends more on the process that invokes it to stay alive
4. Environment variables configured with `set` within a cmd process aren't properly available to the MCP server

In Unix systems:
1. Child processes can continue running even after the parent terminates (fork behavior)
2. Background process management and environment variables are more native

### Root Causes Identified

1. In Windows, commands like `npx` and `npm` are actually batch scripts (`npx.cmd` and `npm.cmd`)
2. Batch scripts require an interpreter (cmd.exe) to run
3. The process doesn't stay alive independently
4. Credentials (API keys, tokens) aren't properly passed to the MCP server process

## Implemented Solution

The complete solution has two parts:

1. Separate the MCP server process using the Windows `start` command
2. Provide credentials through the `env` object in the JSON configuration

### For Firecrawl-MCP:

```json
{
  "mcpServers": {
    "firecrawl-mcp": {
      "command": "cmd.exe",
      "args": [
        "/c",
        "start",
        "/min",
        "cmd",
        "/k",
        "title Firecrawl-MCP Server && echo Starting server... && npx -y firecrawl-mcp"
      ],
      "env": {
        "FIRECRAWL_API_KEY": "your_api_key"
      }
    }
  }
}
```

### For GitHub-MCP:

```json
{
  "mcpServers": {
    "github": {
      "command": "cmd.exe",
      "args": [
        "/c",
        "start",
        "/min",
        "cmd",
        "/k",
        "title GitHub-MCP Server && echo Starting GitHub MCP server... && npx -y @modelcontextprotocol/server-github"
      ],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "your_github_token"
      }
    }
  }
}
```

### Solution Explanation

- `cmd.exe /c`: Executes the command and terminates
- `start /min`: Starts a new process in a minimized window
- `cmd /k`: Runs the command in the new window and keeps the window open
- `title ...`: Sets a descriptive title for the window
- `echo Starting server...`: Displays an informative message
- `npx -y ...`: Runs the MCP server
- `"env": { ... }`: Provides credentials as environment variables to the server process

This configuration allows the server to run in a separate process that doesn't depend on Cursor's original process, and correctly provides credentials to the MCP server.

## Additional Recommendations

**Debugging**: To facilitate error identification, you can temporarily remove the `/min` flag or redirect output to a log file:
   ```
   ... && npx -y firecrawl-mcp > firecrawl.log 2>&1
   ```

## References

- [Cursor forum post about npx issues in MCP](https://forum.cursor.com/t/npx-command-is-not-working-on-mcp-windows-and-macos/53486/6)
- [Model Context Protocol Documentation](https://cursor.sh/docs/mcp)
- [MCP Servers Repository](https://github.com/modelcontextprotocol/servers)