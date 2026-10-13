# Holst CLI

Control [Holst](https://holst.so) boards from your terminal or AI agent:
manage workspaces and boards, read content, add notes, text, shapes and arrows,
and download backups.

## Install

macOS Apple Silicon and Linux x86-64. Python, Node and Yjs are included.

```sh
brew install AntonLisovoy/tap/holst-cli
```

Or:

```sh
curl -LsSf https://raw.githubusercontent.com/AntonLisovoy/holst-cli/main/install.sh | sh
```

The installer checks SHA-256 and installs to `~/.local/bin`. Add that directory
to `PATH` if prompted. Run it again to update; with Homebrew use
`brew update && brew upgrade holst-cli`.

## Start

```sh
holst --version
holst auth login
holst --help
```

`holst auth login` opens Holst in your browser; press "Open in the app"
there to finish. Requires a Holst account. Sign-in stores both the session cookie and token;
a personal access token cannot manage boards.

## Agent skill

```sh
npx skills add AntonLisovoy/holst-cli
```

## Use it from an AI agent

```sh
holst skill install
```

You pick the agents (Claude Code, Cursor, Codex, Copilot, Gemini, or the universal `.agents/skills` layout) and whether the skill goes into this directory or your user account. One canonical copy is written and every other agent gets a link to it, so a later `holst skill install` refreshes them all. Non-interactive: `holst skill install -a claude-code -a cursor --project -y`.

## MCP server

`holst mcp` lets an agent work with boards through typed tools instead of shell commands: 18 read tools (workspaces, boards, board contents in pages and text windows, templates, `doctor`) and 5 write tools (create, rename and copy boards; create and change objects). There are no delete tools. Use the skill when the agent should run `holst` commands itself; use MCP when its client supports MCP servers and you want typed calls.

Claude Code:

```sh
claude mcp add holst -- holst mcp
```

Claude Desktop (`claude_desktop_config.json`) and Cursor (`.cursor/mcp.json`). GUI apps do not inherit your shell's `PATH`, so give the absolute path from `which holst`:

```json
{
  "mcpServers": {
    "holst": {
      "command": "/opt/homebrew/bin/holst",
      "args": ["mcp"]
    }
  }
}
```

Codex (`~/.codex/config.toml`):

```toml
[mcp_servers.holst]
command = "/opt/homebrew/bin/holst"
args = ["mcp"]
```

Read-only: `holst mcp --read-only` hides the write tools. Root flags go before `mcp`, `--read-only` after it: `holst --base-url https://holst.example mcp --read-only`. The server logs to stderr, which your client shows as the server log. Board contents need Node and yjs for live data; `holst doctor` checks them.

## Keep results small

Global flags go before the command. `--select` keeps only the fields you name, `--out` writes the result to a file and prints a one-line summary:

```sh
holst --select boards.id,boards.name board list
holst --out boards.json board list
```

## Shell completion

```sh
holst completion zsh >> ~/.zshrc
```

Also `bash` and `fish`. Homebrew installs completions for all three automatically. Inside the REPL, Tab completes commands and options.

## Diagnostics

```sh
holst doctor
```

Checks the CLI version, the state directory, credentials, Node, yjs and whether the Holst API answers, and prints a hint for each problem. `--offline` skips the network checks; the exit code is 1 when a check fails.

[User guide](docs/guide.md) · [Command reference](docs/reference.md) · [Releases](https://github.com/AntonLisovoy/holst-cli/releases) · [Русский](README.ru.md)

This repository contains release binaries, documentation and the agent skill.
Source code is maintained separately. This is an unofficial client for Holst.
