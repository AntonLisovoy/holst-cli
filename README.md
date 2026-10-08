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
holst auth login --email you@example.com
holst --help
```

Requires a Holst account. Sign-in stores both the session cookie and token;
a personal access token cannot manage boards.

## Agent skill

```sh
npx skills add AntonLisovoy/holst-cli
```

[User guide](docs/guide.md) · [Releases](https://github.com/AntonLisovoy/holst-cli/releases) · [Русский](README.ru.md)

This repository contains release binaries, documentation and the agent skill.
Source code is maintained separately. This is an unofficial client for Holst.
