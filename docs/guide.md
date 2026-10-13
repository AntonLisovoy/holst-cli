# holst-cli

A stateful command-line interface to **[Holst](https://holst.so)** — a hosted
collaborative whiteboard (a Russian alternative to Miro). Built so that AI
agents can drive the product without a browser.

## What "the real software" means here

Holst has no local installation. This CLI does **not** re-implement boards: it
speaks the same HTTP API the official web client at `https://app.holst.so` uses,
and reads board contents by decoding the same CRDT snapshots the app restores
from. The Holst service is a hard dependency — without credentials the CLI fails
with instructions rather than degrading to a fake local mode.

## Requirements

- macOS Apple Silicon or Linux x86-64 for prebuilt releases
- Python 3.10+ only when installing from source
- A Holst account
- Network access to `https://app.holst.so` (or your on-premise host)
- Node 21+ plus the `yjs` package for source installations; prebuilt releases include both, to read boards live and to write to them
  (see [How board contents work](#how-board-contents-work))

## Installation

Prebuilt releases include Python, Node and Yjs:

```bash
brew install AntonLisovoy/tap/holst-cli
```

Or install into `~/.local/bin` without Homebrew:

```bash
curl -LsSf https://raw.githubusercontent.com/AntonLisovoy/holst-cli/main/install.sh | sh
holst --version
```

Run the same command to update. Add `~/.local/bin` to your `PATH` if needed.
For a specific version, use `sh -s -- 1.0.0` instead of `sh`.

For development from a source checkout:

```bash
uv sync --dev
uv run holst --help
```

Dependencies install automatically. Source installations additionally need Node
21+ and `npm install --prefix ~/.config/holst-cli/node yjs`.

## Authentication - read this first

Two things to know before you start.

**A personal access token will not work.** The Holst UI issues PATs with a single
scope, `organization:audit_events_read` - they cannot list or create boards.

**A token alone is not enough either.** Verified against the live service: Holst
checks the session cookie *and* the bearer token together.

| Credential sent to `GET /gapi/current_user` | Result |
|---|---|
| Bearer token alone | 401 |
| Cookie alone | 401 |
| **Cookie + token** | **200** |

Signing in stores both halves, so this is usually invisible - but it explains why
`auth login --token <TOKEN>` by itself fails, and why the value in `localStorage`
(a CSRF token, not the session token) authenticates nothing.

```bash
# Recommended: email and password. The password is never stored;
# the resulting token and cookie are.
holst auth login --email you@example.com

# Non-interactive, where there is no terminal for a hidden prompt:
printf '%s' "$PASSWORD" | holst auth login --email you@example.com --password-stdin

# Pair like the desktop app does
holst auth pair

# Or copy both halves from the browser: DevTools -> Network -> any /gapi/
# request -> its Cookie and Authorization request headers
holst auth login --cookie "<COOKIE>"
holst auth login --token <TOKEN> --no-verify
```

Not sure what your account allows, or why a login failed?

```bash
holst auth methods --email you@example.com   # permitted sign-in methods
holst auth diagnose                          # which credential form works
```

The token is stored in `~/.config/holst-cli/credentials.json` with mode
`0600` and is never printed in full. `HOLST_TOKEN` overrides the stored value.

Verify it:

```bash
holst auth status --verify
holst auth whoami
```

## Usage

### Interactive (default)

```bash
holst
```

Starts a REPL with history, a status prompt, and the packaged `SKILL.md` path in
the banner. `help` lists the command groups; `exit` leaves.

### One-shot commands

```bash
# Workspaces
holst workspace list
holst workspace use <WORKSPACE_ID>
holst workspace tree

# Boards
holst board list
holst board list --search retro
holst board create --name "Sprint 42 Retro" --use
holst board info
holst board rename "Sprint 42 Retro (archived)"
holst board duplicate --name "Sprint 43 Retro"
holst board move --into <FOLDER_ID>              # into a folder; --root takes it out
holst board url
holst board members
holst board recent --limit 10                     # newest first; --cursor continues
holst board favorites --add <BOARD_ID>
holst board delete --board <BOARD_ID>            # to trash
holst board restore <BOARD_ID>                   # back from trash
holst board delete --board <BOARD_ID> --permanently

# Reading board contents
holst content summary
holst content objects --type sticker
holst content text
holst content raw -o board.json
holst content objects --snapshot-only     # skip the live session

# Writing board contents
holst content add-frame --ratio 16:9 --name "Sprint 43"
holst content add-sticker --text "Ship the harness" --color yellow4
holst content add-sticker --text "Retry-safe" --id <UUID>   # same id again adds nothing
holst content add-sticker --text "Inside the frame" --frame <FRAME_ID>
holst content add-text --text "Heading" --x 40 --y 40
holst content add-shape --kind flowchart-decision --text "Ship?"
holst content add-card --text "Decision record" --color blue3
holst content add-arrow --from-object <ID> --to-object <ID>
holst content shapes                       # the 62 shape kinds
holst content apply --file objects.json     # many at once

# Changing and removing them
holst content update --object <ID> --text "Reworded" --color red6
holst content update --object <ID> --x 400 --y 200 --to-front
holst content update --object <ID> --frame <FRAME_ID>
holst content delete --object <ID>          # a frame takes its contents
holst content delete --object <ID> --dry-run   # print the plan, write nothing

# Snapshots
holst backup list
holst backup download board.snapshot

# Templates
holst template list --scope standard
holst template categories

# Session
holst session status
holst session history
```

### JSON mode for agents

Every command accepts `--json` and emits a uniform envelope:

```bash
holst --json board list
```

```json
{ "ok": true, "boards": [ ... ], "count": 12, "workspace_id": "..." }
```

Failures are equally machine-readable and exit non-zero:

```json
{ "ok": false, "error": "…", "error_type": "auth_error" }
```

`error_type` is one of `auth_error`, `not_found`, `api_error`, `config_error`,
`invalid_input`, `invalid_select`, `io_error`, `skill_install_error`, `doctor_failed`.

## Agents, output and diagnostics

### Installing the skill into an AI agent

`holst skill install` writes the bundled `SKILL.md` where your agent looks for
skills. With no flags it asks which agents and whether the skill belongs to this
directory or to your user account.

| Agent | Project path | Global path |
|---|---|---|
| `claude-code` | `.claude/skills` | `~/.claude/skills` |
| `cursor` | `.agents/skills` | `~/.cursor/skills` |
| `codex` | `.agents/skills` | `~/.codex/skills` |
| `copilot` | `.agents/skills` | `~/.copilot/skills` |
| `gemini` | `.agents/skills` | `~/.gemini/skills` |
| `universal` | `.agents/skills` | `~/.config/agents/skills` |

Each path gets a `holst-cli/SKILL.md` below it. One canonical copy is written in
the universal layout for the chosen scope; every other agent directory gets a
relative link to it (computed from real paths, so a symlinked `~/.claude`
works). When a link cannot be made, the file is copied and a warning says so.

Flags that skip the picker:

- `-a` / `--agent`: an agent to install for; repeat it for several.
- `-g` / `--global` or `--project`: the scope.
- `--dir DIRECTORY`: install into exactly this directory; bypasses the agent
  layout and makes no links.
- `-y`: do not prompt, take the defaults (the universal layout in this directory).

`--json` never prompts: without `-a`, `-g`/`--project` or `-y` it fails with
`invalid_input` naming those flags.

```bash
holst skill install -a claude-code -a cursor --project -y
holst skill path
```

### Keeping output small: `--select` and `--out`

Global flags go before the subcommand. Both imply `--json`.

`--select PATHS` takes comma-separated dotted paths and keeps only those fields:

- lists are walked and the shape is kept, so `boards.id` on a list of boards gives
  `{"ok": true, "boards": [{"id": "b1"}, {"id": "b2"}]}`;
- a path that does not exist gives `null`, never an error, so a `null` may be a
  typo;
- `ok` is always kept; `error_type` and `error` are kept only when `ok` is false;
- an empty path (`--select ""` or `a,,b`) fails with `invalid_select`.

```bash
holst --select boards.id,boards.name board list
```

`--out FILE` writes the whole JSON result to the file and prints one line:

```json
{ "ok": true, "output": "/absolute/path/boards.json", "bytes": 1843 }
```

When a command fails (an error envelope), no file is written and the envelope is
printed as usual (a file that cannot be written is `io_error`). The exception is
a failing `doctor` report: it is still written to the file, and the summary line
carries `"ok": false`, `error_type` and `error`. `--out` cannot be combined with
`content raw --output` (`invalid_input`), `backup download` refuses it, and it
is ignored with a warning for `completion` and the REPL. Combine it with
`--select` to store a trimmed result.

```bash
holst --out boards.json board list
```

### Shell completion

```bash
holst completion bash > ~/.local/share/bash-completion/completions/holst
holst completion zsh > "${fpath[1]}/_holst"
holst completion fish > ~/.config/fish/completions/holst.fish
```

Homebrew installs all three for you. Inside the REPL, Tab completes commands
and options, and history is kept in `repl_history` in the state directory.
`--json` and `--base-url` given when the REPL starts apply to every line;
`--select` and `--out` there are ignored with a warning.

### Diagnostics: `holst doctor`

`holst doctor` runs the checks below in order and prints a table with a hint
under each problem. The exit code is 1 when a check fails. `--offline` skips
the network checks; `--timeout SECONDS` (default 10) bounds each request.
Credentials, tokens and cookies are always masked.

| Check | ok | warn | fail |
|---|---|---|---|
| `version` | CLI version, Python, platform, install kind | - | - |
| `state_dir` | exists and is writable | does not exist yet | exists and is not writable |
| `credentials` | token and cookie both present | only one is present, or the file is readable by others | neither is present |
| `base_url` | value and where it came from | - | - |
| `node` | Node 21 or newer with `WebSocket` | missing or too old: `content` commands and live edits are unavailable | - |
| `yjs` | the `yjs` package loads | not found or not loadable | - |
| `scripts` | `hud.js` and `convert.js` are present and load with yjs | - | a script is missing or does not load |
| `api` | the host answered any HTTP response | - | no answer, or timeout |
| `auth` | the stored credentials identify a user | - | rejected (401/403) or any other error |

`api` and `auth` are skipped with `--offline`; `auth` is also skipped when
`credentials` or `api` failed, and `scripts` when Node or yjs is not usable. In
`--json` mode the report is
`{"ok": ..., "checks": [...]}`, and a failing report carries
`"error_type": "doctor_failed"` and an `error` summary.

```bash
holst doctor --offline
holst --json doctor
```

### Debug output: `--debug`

`holst --debug <command>`, or `HOLST_DEBUG=1` in the environment, writes a
trace to stderr: one line per HTTP request (method, host, path, status, time to
headers, retry, size), per board-session protocol message (direction, type, document,
`left`, update id, Simple or Link, bytes), per request the Node helper makes
(origin, path, whether credentials were attached, status, bytes) and per phase
(connect, sync, plan, ack, finish), each with the time since start. stdout is
unchanged, so `--json` output stays valid. Under `holst mcp` the variable works
the same way and the trace lands in the server log.

Tokens and cookies are masked wherever they appear (URL-encoded too), every
query value is masked (the keys stay), another document's name on the socket
is shown as `<other>`, and no header, request or frame body is printed. The
trace still names hosts, your board ids and paths: read it before you share
it. An error that quotes the server shows at most 400 characters of it, with
credentials masked, with or without `--debug`; server text longer than 65,536 characters
is replaced by `<omitted: N characters>` instead of being quoted.

```bash
holst --debug content summary
HOLST_DEBUG=1 holst --json board list > boards.json
```

### MCP server: `holst mcp`

`holst mcp` serves Holst to an AI agent over the Model Context Protocol (stdio). Client setup is in the storefront README, section "MCP server". `holst mcp --read-only` lists only the read tools.

| Tool | Does |
|---|---|
| `auth_whoami` | The signed-in account |
| `workspace_list`, `workspace_get`, `workspace_tree` | Workspaces and their folders and boards |
| `board_list` | Boards in a workspace, paginated, optional name filter |
| `board_get`, `board_recent`, `board_members`, `board_comments`, `board_sharing` | One board with its url; recent boards; members; comment thread metadata; sharing settings |
| `template_list`, `template_categories` | Templates |
| `content_summary`, `content_objects`, `content_object_get`, `content_text` | Board contents: counts, objects (paginated), one object with its full text, all text |
| `content_shapes` | Shape kinds, colours, frame ratios, arrow types (offline) |
| `doctor` | The `holst doctor` report |
| `board_create`, `board_rename`, `board_duplicate` | Write: boards |
| `content_create`, `content_update` | Write: up to 100 objects per call, all or nothing; items take `id`, and `dry_run: true` returns the plan without writing |

Results stay small: `board_list`, `board_recent` and `content_objects` are paginated and return `{items, next_cursor}` (50 by default, 100 at most); the other lists (workspaces, the workspace tree, members, comment threads, templates) come whole, since they grow with the account rather than with board content. Texts come in windows (`offset`, `max_chars`, then `text_next_offset`); `raw` payloads appear only where asked for, with long strings cut. Board contents are cached for 30 s per board (`refresh: true` reloads) and every content result carries `source`: live or snapshot, its age, and why it is stale.

A failed call returns an error result whose text starts with its type: `auth_error`, `not_found`, `api_error`, `holst_error`, `hud_error`, `object_not_found`, `write_error`, `write_unconfirmed`, `config_error`, `invalid_input`, `read_only` or `internal_error`. The CLI's `--json` uses the same names, plus `invalid_select`, `io_error`, `no_input` and `skill_install_error`. The `doctor` tool reports failed checks inside a normal result (`ok: false`, `error_type: doctor_failed`), not as a tool error.

## How board contents work

Holst stores board contents as a Yjs CRDT document; there is no "GET board as
JSON" endpoint, and no create-object endpoint either. The CLI reads two ways and
writes one way.

**Reading** prefers the live document and falls back to a snapshot:

```
live      wss://app.holst.so/hud/ws   ->  join, sync, decode
snapshot  GET /gapi/backup/list       ->  snapshot keys
          GET /gapi/backup/download   ->  binary Yjs update -> pycrdt
```

Snapshots are written about every ten minutes, so they can lag the board, and a
board nobody has opened has none at all. When a read falls back to one, the CLI
says so and why. `--snapshot-only` forces that route; `--key` implies it.

**Writing** joins the board's live document, applies every operation in one Yjs
transaction, and waits for the server to acknowledge the update. Nothing is
reported as written until that acknowledgement arrives.

Six object types can be created: `frame`, `sticker`, `text`, `shape`, `card` and
`arrow`. Everything else on a board is readable but not writable -- see
[Limitations](#limitations).

`content apply` takes a JSON list. An entry with no `op` creates an object of
its `type`, with the remaining keys matching that `add-*` command's options; an
entry with `"op": "update"` or `"op": "delete"` names an existing `id`:

```json
[
  {"type": "frame", "frame_id": "3f1c9d20-0000-4000-8000-000000000001",
   "ratio": "16:9", "name": "Sprint 43", "x": 0, "y": 0},
  {"type": "sticker", "text": "Ship it", "color": "yellow4", "x": 40, "y": 120,
   "parent_id": "3f1c9d20-0000-4000-8000-000000000001"},
  {"type": "shape", "kind": "flowchart-decision", "text": "Ready?", "x": 500, "y": 120},
  {"type": "arrow", "start": {"x": 260, "y": 200}, "end": {"x": 500, "y": 200}},

  {"op": "update", "id": "…", "text": "Reworded", "color": "red6"},
  {"op": "delete", "id": "…"}
]
```

Ids are not allocated by the server, so giving the frame an explicit `frame_id`
lets the notes name it as their `parent_id` in the same file. Omit it and the
frame gets a generated id, which only exists after the write. Any create entry
may carry an `"id"` (the MCP items take `id` too; `frame_id` is the same key
for a frame), and every `add-*` command takes `--id`. If the board already
holds that id with the same type, the entry is skipped; with another type, or
twice in one file, the whole plan is refused. Re-running a write with the same
ids is therefore safe.

`--dry-run` (on every `content add-*`, `update`, `delete` and `apply`; `dry_run`
on the MCP tools) builds the plan against the live board and prints what would
be created, changed and removed, and writes nothing.

If any id in the file is wrong -- an edit or delete naming an object that is not
there, or a `parent_id` or arrow end naming an object that is not, or one the
same plan deletes -- **nothing** is written and the error names the id. A file
may name a frame it creates further down. The references are checked again in
the write session; a board that changed under the plan makes the helper refuse it (its
internal exit 9) and nothing is written. The command then rebuilds the plan once
against the board as it is. The rebuilt plan may fail validation (`invalid_input`,
for example a target that has vanished) with no further write; a second stale-plan
refusal is reported as `object_not_found` (the CLI exits 1).

**Editing** rewrites the fields named and leaves the rest alone; `--text`
retypes the body in place, so anyone with that board open sees the change
without reloading. **Deleting** a frame takes its contents with it, as it does
in Holst; `--keep-children` leaves them behind, moved to the enclosing frame
(or the top level), and an arrow end stuck to a deleted object becomes a free
point approximately where it was drawn. If the connection drops after the update was sent,
the command fails with `error_type` `write_unconfirmed` (helper exit 10, CLI exit 1; tell cases apart by `error_type`): the board may have applied it, and
`--json` lists the `created`, `updated` and `deleted` ids; re-run with the same
ids. Neither can be undone from the
CLI: a board's undo history belongs to the browser session that made the change.

Colours are Holst palette tokens -- one of `white`, `amber`, `blue`, `brown`,
`crimson`, `cyan`, `grass`, `gray`, `green`, `indigo`, `jade`, `lime`, `mint`,
`orange`, `pink`, `plum`, `purple`, `red`, `sky`, `teal`, `tomato`, `violet`,
`yellow` followed by a shade `1`-`12` -- or a hex value, except on a card, which
takes tokens only. Frame presets are `16:9`, `4:3`, `1:1`, `a4-portrait` and
`a4-landscape`; pass `--width`/`--height` for a custom size. Shape kinds are
listed by `content shapes` -- 19 basic, 25 flowchart and 18 BPMN.

One thing to know when reading back what you wrote: a text object's type on the
wire is `simple-text`, so `content objects --type simple-text` is what filters
for it. `add-text` and `"type": "text"` in an apply file both accept the
friendlier name.

**Both live reads and all writes need Node 21+ and the reference `yjs` package** (included in prebuilt releases),
because Holst's `hud` boards speak the Yjs v2 encoding that `pycrdt` does not.
Install it once:

```bash
npm install --prefix ~/.config/holst-cli/node yjs
```

Without it, reads fall back to snapshots and writes fail with that exact install
command -- never silently. A v2 update decodes as *empty*, not as an error, under
a v1 reader, so "this board is empty" is never reported without checking.
Override the binary with `HOLST_NODE` and the package location with
`HOLST_YJS_PATH`.

Writing is implemented for the `hud` provider, which is what every current board
uses. A `hocuspocus` board is refused with a message naming the reason rather
than written to blindly; its contents remain readable.

## Examples

Two runnable scripts in `examples/`, both shaped the way an agent would drive
the CLI — `--json` in, ids out:

| Script | What it does |
|---|---|
| `read_board.sh [query]` | Find a board by name and print its object counts and every string on it |
| `write_board.sh [name]` | Create a board, put a 16:9 frame and five notes on it, read them back, print the URL |
| `diagram_board.sh [name]` | Draw a flowchart of shapes and arrows, relabel one node, delete another, read it back |

## Configuration

| Variable | Purpose | Default |
|---|---|---|
| `HOLST_TOKEN` | Session token, overrides the stored file | — |
| `HOLST_COOKIE` | Session cookie; Holst SaaS needs it together with the token | — |
| `HOLST_NODE` | Node binary for board contents and live edits | `node` on `PATH` |
| `HOLST_YJS_PATH` | Directory that directly contains the `yjs` package (normally a `node_modules` directory) | searched in order: this variable; `<executable dir>/runtime/node_modules` (prebuilt binary installs only); `<state dir>/node/node_modules`; `NODE_PATH`; the global npm root |
| `HOLST_BASE_URL` | Service host (on-premise installs) | `https://app.holst.so` |
| `HOLST_AUTH_SCHEME` | Token header form: `bearer`, `token`, `raw`, `x-auth-token` | `bearer` |
| `HOLST_CLI_STATE_DIR` | Session/credentials directory | `~/.config/holst-cli` |
| `HOLST_LIVE_TESTS` | Set to `1` to enable the live test suite | unset |
| `HOLST_DEBUG` | Set to `1` (or pass `--debug`) to trace HTTP requests, board-session frames and timings to stderr, redacted | unset |

## Running the tests

```bash
# from the repository root
pip install -e ".[test]"

# Offline suite: unit tests, CRDT pipeline, local HTTP contract, CLI subprocess
HOLST_CLI_FORCE_INSTALLED=1 python3 -m pytest holst_cli/tests/ -v -s

# Live suite (creates and deletes boards in YOUR workspace)
HOLST_LIVE_TESTS=1 HOLST_CLI_FORCE_INSTALLED=1 \
  python3 -m pytest holst_cli/tests/test_full_e2e.py -v -s -k Live
```

See `tests/TEST.md` for the plan, results, and known coverage gaps.

## Limitations

- **Six object types can be created**: frames, sticky notes, text, shapes, cards
  and arrows. The rest of Holst's catalogue -- tables, images, kanban boards,
  mind maps, code blocks, dice and the others -- decodes and lists fine, and any
  of them can be moved, resized, restacked, reparented and deleted, but there is
  no builder to create one and no way to set its colour or text.
- **Editing covers position, size, colour, text, parent and stacking.** Rotation,
  fonts, arrow labels and per-type styling are not exposed; `content raw` shows
  the fields that exist on any given object. An arrow is the exception to
  moving: it follows its endpoints, and re-routing one means deleting it and
  drawing another.
- **Writing needs the `hud` provider**, and Node with the `yjs` package. A
  `hocuspocus` board is refused explicitly.
- **No server-side image export.** Holst renders PNG/SVG/PDF in the browser, so
  there is no endpoint to ask for a rendered board.
- **Unofficial.** Holst publishes no public API reference; the endpoints used
  here were read from the official web client. They may change without notice.
  Response fields are read defensively and the raw payload is always preserved.
  When something does break, `docs/WHEN_THE_API_CHANGES.md` is the recovery
  procedure: how to identify which layer drifted, how to read the current
  bundle with `tools/bundle_search.py`, and where each fact lives in the code.

## Attribution

Not affiliated with or endorsed by Holst.
